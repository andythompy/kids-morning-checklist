# Calendar Setup Instructions

The "What's On?" column reads events from a shared Google Calendar (default name: **Thompson**).

Because this app is a static page on GitHub Pages, it **cannot talk to Google Calendar directly** — that would require putting OAuth credentials in `index.html`, which would expose them to anyone who views the page. Instead, the page calls a small **server-side proxy** that you own, which holds the Google credentials and returns event JSON.

The easiest (and free) way to host that proxy is **Google Apps Script**, because it can read your own calendars without any extra OAuth configuration. This guide walks through that path.

> Until you complete this setup, the calendar column will simply show **"Calendar not set up yet."** — the rest of the app (checklists, weather) works fine without it.

---

## Step 1: Make sure the calendar is shared with you

1. The shared calendar must be visible in [Google Calendar](https://calendar.google.com/) under **My calendars** or **Other calendars** for the Google account you'll use to create the Apps Script.
2. Note the exact display name (default expected by the app: `Thompson`). It must match `CONFIG.calendar.calendarName` in `index.html`.

## Step 2: Create the Apps Script

1. Go to <https://script.google.com/> and click **New project**.
2. Replace the contents of `Code.gs` with the script below.
3. Click the **Save** icon and give the project a name (e.g. `Morning Checklist Calendar Proxy`).

```javascript
// Returns events from a named Google Calendar as JSON.
// Required query params: timeMin, timeMax, calendarName
// Example: ?timeMin=2026-05-10T00:00:00Z&timeMax=2026-05-13T00:00:00Z&calendarName=Thompson
function doGet(e) {
  try {
    const params = (e && e.parameter) || {};
    const calendarName = params.calendarName;
    const timeMin = params.timeMin ? new Date(params.timeMin) : null;
    const timeMax = params.timeMax ? new Date(params.timeMax) : null;

    if (!calendarName || !timeMin || !timeMax) {
      return jsonResponse({ error: 'Missing timeMin, timeMax or calendarName' });
    }

    const matches = CalendarApp.getCalendarsByName(calendarName);
    if (matches.length === 0) {
      return jsonResponse({ error: 'Calendar not found: ' + calendarName });
    }

    const events = matches[0].getEvents(timeMin, timeMax).map(function (event) {
      const allDay = event.isAllDayEvent();
      return {
        title: event.getTitle(),
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

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `Calendar not set up yet.` | `CONFIG.calendar.endpoint` is still `""` |
| `Calendar could not be loaded.` + 401/403 in console | Web app deployed as "Only myself" — redeploy with **Who has access: Anyone** |
| `Calendar could not be loaded.` + 404 in console | `calendarName` doesn't match the calendar in Google Calendar exactly (case/spacing matters) |
| Events show on the wrong day | Apps Script returns ISO timestamps in UTC; the app renders them in the browser's local timezone — make sure the Nest Hub is set to your timezone |
