# Calendar Setup Instructions

The "What's On?" column reads events from a shared Google Calendar (default name: **Thompson**).

Because this app is a static page on GitHub Pages, it **cannot talk to Google Calendar directly** — that would require putting OAuth credentials in `index.html`, which would expose them to anyone who views the page. Instead, the page calls a small **server-side proxy** that you own, which holds the Google credentials and returns event JSON.

The easiest (and free) way to host that proxy is **Google Apps Script**, because it can read your own calendars without any extra OAuth configuration. This guide walks through that path.

> Until you complete this setup, the calendar column will simply show **"Calendar not set up yet."** — the rest of the app (checklists, weather) works fine without it.

> ⚠️ **Read [Security considerations](#security-considerations) before pointing this at a personal calendar.** With the default "Anyone can access" Apps Script deployment, the contents of the chosen calendar are effectively public. The recommended setup is to use a **dedicated calendar** that only contains events safe to display.

---

## Step 1: Pick (or create) a calendar to expose

The proxy will return *every* event in this calendar to anyone who hits the URL (see [Security considerations](#security-considerations) — there is no practical way to make the URL secret because it ships in the public `index.html`). So:

- **Recommended:** in Google Calendar, click **+ → Create new calendar** and make a calendar called e.g. `Thompson` or `Family Dashboard`. Add only the events you're happy to see on a kitchen display. Share it with anyone in the family who should be able to add events.
- **Not recommended:** pointing this at your main personal calendar.

The calendar must be visible in [Google Calendar](https://calendar.google.com/) under **My calendars** or **Other calendars** for the Google account you'll use to create the Apps Script. The display name must match `CONFIG.calendar.calendarName` in `index.html` exactly (default: `Thompson`).

## Step 2: Create the Apps Script

1. Go to <https://script.google.com/> and click **New project**.
2. Replace the contents of `Code.gs` with the script below. (It deliberately omits `description` and `attendees`, and only returns `location` if you leave the line in — strip that line if you don't want locations exposed.)
3. Click the **Save** icon and give the project a name (e.g. `Morning Checklist Calendar Proxy`).

```javascript
// Returns events from a named Google Calendar as JSON.
// Required query params: timeMin, timeMax, calendarName
// Optional query param:  c (shared token — see "Token gating" below)
// Example: ?timeMin=2026-05-10T00:00:00Z&timeMax=2026-05-13T00:00:00Z&calendarName=Thompson&c=k7q2x9mz
//
// ── Token gating (optional) ──────────────────────────────────────────
// Set SHARED_TOKEN to a random string (e.g. "k7q2x9mz") and bookmark
// the checklist page as  https://yoursite/?c=k7q2x9mz
// The page reads "c" from the URL at runtime and forwards it here.
// Because the token only exists in the bookmark and in this constant,
// it never appears in the public GitHub repo.
// Leave SHARED_TOKEN as "" to allow unauthenticated access (old behaviour).
var SHARED_TOKEN = "";   // ← put your random string here, or leave "" to disable

function doGet(e) {
  try {
    var params = (e && e.parameter) || {};

    // ── Token check ──────────────────────────────────────────────────
    if (SHARED_TOKEN && params.c !== SHARED_TOKEN) {
      return jsonResponse({ error: "forbidden" });
    }

    var calendarName = params.calendarName;
    var timeMin = params.timeMin ? new Date(params.timeMin) : null;
    var timeMax = params.timeMax ? new Date(params.timeMax) : null;

    if (!calendarName || !timeMin || !timeMax) {
      return jsonResponse({ error: 'Missing timeMin, timeMax or calendarName' });
    }

    var matches = CalendarApp.getCalendarsByName(calendarName);
    if (matches.length === 0) {
      return jsonResponse({ error: 'Calendar not found: ' + calendarName });
    }

    var events = matches[0].getEvents(timeMin, timeMax).map(function (event) {
      var allDay = event.isAllDayEvent();
      return {
        title: event.getTitle(),
        // Remove the next line if you don't want event locations exposed via the URL.
        location: event.getLocation() || '',
        allDay: allDay,
        startTime: event.getStartTime().toISOString(),
        endTime: event.getEndTime().toISOString()
      };
    });

    return jsonResponse({ items: events });
  } catch (err) {
    return jsonResponse({ error: String(err) });
  }
}

function jsonResponse(payload) {
  return ContentService
    .createTextOutput(JSON.stringify(payload))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## Step 3: Deploy as a Web App

1. In the Apps Script editor, click **Deploy → New deployment**.
2. Click the gear icon next to **Select type** and choose **Web app**.
3. Configure:
   - **Description**: `Morning Checklist Calendar Proxy`
   - **Execute as**: **Me** (your Google account — this is what gives the script permission to read your calendars).
   - **Who has access**: **Anyone** *(required for the static page to call it without authentication; the script only ever returns events from the named calendar, never writes)*.
4. Click **Deploy**.
5. Authorize the script when prompted. You will see a "Google hasn't verified this app" warning — this is expected for personal Apps Script projects. Click **Advanced → Go to … (unsafe)** and continue.
6. Copy the **Web app URL**. It looks like:

   ```
   https://script.google.com/macros/s/AKfycby...../exec
   ```

## Step 4: Plug the URL into the app

Open `index.html` and find the `CONFIG.calendar` block (around line 330):

```javascript
calendar: {
    endpoint: "https://script.google.com/macros/s/AKfycby...../exec", // <-- paste your URL here
    calendarName: "Thompson",                                          // must match the calendar name exactly
    daysToShow: 3                                                      // 1–7
}
```

Commit and push — GitHub Actions will redeploy to Pages automatically.

## Step 5: Verify

1. Open the deployed page. The "What's On?" column should populate within a couple of seconds.
2. If it shows "Calendar could not be loaded.", open DevTools → Console; the network response from your `/exec` URL will explain why (most often: wrong calendar name, or the deployment is set to "Only myself" instead of "Anyone").

---

## Updating the script later

Apps Script web apps are versioned. If you edit the script, you must click **Deploy → Manage deployments → ✏️ Edit → Version: New version → Deploy** to publish the change. The `/exec` URL stays the same.

## Alternative proxies

The frontend just expects JSON in either of these shapes, so any backend works:

```jsonc
// Shape A — Google Calendar Events API style (what Apps Script returns above):
{ "items": [ { "title": "...", "startTime": "...", "endTime": "...", "allDay": false, "location": "" } ] }

// Shape B — a bare array:
[ { "title": "...", "startTime": "...", "endTime": "...", "allDay": false } ]
```

So if you'd rather host the proxy as a Firebase Cloud Function, Cloudflare Worker, etc., return either shape and point `CONFIG.calendar.endpoint` at it.

## Security considerations

**Short version:** with this architecture, the contents of the chosen calendar are effectively public *unless* you enable token gating (see below). Don't point it at a calendar that contains things you wouldn't want a stranger to read.

**Why the endpoint URL alone is not secret:**

- The Apps Script web app is deployed as **"Who has access: Anyone"**. That's required, because the static GitHub Pages page has no Google identity to authenticate as.
- The `/exec` URL itself is **not secret**: it's hard-coded into `index.html`, which is in the public GitHub repo and served by GitHub Pages. Anyone can read it from page source or DevTools → Network on the Nest Hub.
- Referer/Origin checking in the script is trivially spoofable and not a real defence.

### Token gating (recommended)

You can add a lightweight shared secret that stops crawlers and casual visitors from getting calendar data:

1. **Pick a random string** (e.g. `k7q2x9mz` — don't use a dictionary word).
2. **In the Apps Script**, set `var SHARED_TOKEN = "k7q2x9mz";` at the top of `Code.gs` and redeploy (Deploy → Manage deployments → ✏️ Edit → Version: New version → Deploy).
3. **Bookmark the checklist page** on your Nest Hub as `https://yoursite/?c=k7q2x9mz`.

The page reads `c` from the URL at runtime and forwards it to the proxy. Because the token only exists in two places — the Nest Hub bookmark and the Apps Script constant — it **never appears in the public GitHub repo**. Anyone who finds your `/exec` URL in the source code still can't use it without the token.

**What this does NOT protect against:**

- Someone who can see the Nest Hub's address bar can read the token from the URL.
- If the page ever loads a third-party resource that honours Referer headers, the token could leak. A `<meta name="referrer" content="no-referrer">` tag is already included in `index.html` to prevent this.
- Browser history/sync may store the bookmarked URL (including the token) in your Google account.

**If the token leaks:** change `SHARED_TOKEN` in the Apps Script, redeploy, and update the bookmark. The old token stops working immediately.

**What the script *does* protect against (with or without the token):**

- Writing to the calendar (read-only via `getEvents`).
- Reading any other calendar on your Google account (only `getCalendarsByName(calendarName)` is called, so the only calendar exposed is the one whose name matches `CONFIG.calendar.calendarName`).
- Account compromise (the script runs as you on Google's infrastructure; your credentials are never exposed).

**Recommended mitigations, in order of effort:**

1. **Enable token gating** (above). This stops automated scraping and casual visitors.
2. **Use a dedicated calendar** (see Step 1). This shrinks the blast radius from "my whole life" to "events I deliberately put on the family dashboard".
3. **Strip sensitive fields** in the script. The example already omits descriptions and attendees; remove the `location` line too if locations are sensitive. The frontend handles a missing `location` gracefully.
4. **Avoid putting personally identifiable details in event titles** on the dashboard calendar (e.g. `"Doctor — Dr Smith re: <condition>"` → `"Appointment"`).
5. **Long-term: Firebase Cloud Function + App Check.** Replace Apps Script with a Function that requires a Firebase App Check token. Only your deployed page can mint a valid token, so random visitors to the `/exec` URL get rejected. This is the proper authenticated-proxy pattern but is significantly more setup; not covered here.

If you ever suspect the URL has been abused, go to **Deploy → Manage deployments → Archive** in the Apps Script editor; that immediately invalidates the URL. Then create a new deployment and update `CONFIG.calendar.endpoint`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `Calendar not set up yet.` | `CONFIG.calendar.endpoint` is still `""` |
| `Calendar could not be loaded.` + 401/403 in console | Web app deployed as "Only myself" — redeploy with **Who has access: Anyone** |
| `Calendar could not be loaded.` + 404 in console | `calendarName` doesn't match the calendar in Google Calendar exactly (case/spacing matters) |
| Events show on the wrong day | Apps Script returns ISO timestamps in UTC; the app renders them in the browser's local timezone — make sure the Nest Hub is set to your timezone |
