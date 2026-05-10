# Kids Morning Checklist

A single-page web app for our Google Nest Hub Max that shows each child's morning routine, the weather, and upcoming family calendar events. Tasks tick off in real-time and reset every day.

> **Heads up:** this repo is set up for *our* family — children's names, calendar name, location coordinates, and Firebase project are all hard-coded in `index.html`. If you fork it for your own use, see [Customising for your family](#customising-for-your-family) below.

## How it's built

- **Single file**: everything lives in [`index.html`](./index.html) — HTML, CSS (Tailwind via CDN), and JavaScript (ES modules via CDN). No build step.
- **Hosting**: GitHub Pages, deployed automatically by [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml) on every push to `main`.
- **State**: Firebase Firestore. Each day gets a document under `checklists/YYYY-MM-DD`. All devices viewing the page sync in real-time via `onSnapshot`.
- **Auth**: Firebase Anonymous Authentication. There are no user accounts; the device signs itself in anonymously and Firestore rules require `request.auth != null`.
- **Weather**: [Open‑Meteo](https://open-meteo.com/) — no API key required.
- **Calendar**: Google Calendar, fetched via a small server-side proxy that you own (see [Calendar setup](#calendar-setup)).

## Configuration map — what lives where

| Thing | Where it's configured | Is it a secret? |
|---|---|---|
| Children, tasks, per-child chores | `CONFIG.children`, `CONFIG.tasks`, `CONFIG.childSpecificTasks` in `index.html` | No |
| Weather location | `CONFIG.weather` (lat/lon) in `index.html` | No |
| Calendar proxy URL | `CONFIG.calendar.endpoint` in `index.html` | No (it's a public URL — and effectively so are the calendar contents it returns; see [`CALENDAR_SETUP.md`](./CALENDAR_SETUP.md#security-considerations)) |
| Firebase web app config | `firebaseConfig` object in `index.html` | **No — see below** |
| Firestore security rules | Set in the Firebase Console | n/a (lives in Google) |
| GitHub Actions deploy | `.github/workflows/deploy.yml` | **No secrets needed** |

### Why none of the keys in `index.html` are secrets

The Firebase **web** config (`apiKey`, `projectId`, `appId`, etc.) is **designed to be public**. It's an identifier for the project, not a credential. Anyone visiting the live page already downloads it. The security boundary is your **Firestore security rules** plus **Anonymous Auth being enabled** — both configured in the Firebase Console, both documented in [`FIREBASE_SETUP.md`](./FIREBASE_SETUP.md).

So:
- ✅ Firebase web config can safely live in the repo.
- ✅ The deploy workflow doesn't need any GitHub Actions secrets.
- ❌ A Google Calendar OAuth token / API key would be a real secret — that's why the calendar is fetched via a proxy you host (Apps Script, Firebase Function, etc.) instead of being called directly from the page.

## How deployment works

`.github/workflows/deploy.yml` runs on every push to `main` and:

1. Checks out the repo.
2. Uploads the entire repo as a GitHub Pages artifact.
3. Publishes it to Pages.

There's no build step, no environment variables, and no GitHub Actions secrets in use. If you want to verify, open the workflow file — there are no `${{ secrets.* }}` references anywhere.

To enable Pages on a fresh fork: **Settings → Pages → Build and deployment → Source: GitHub Actions**.

## First-time setup

If you're configuring this from scratch (forking, or rebuilding), do these in order:

1. **Firebase** — follow [`FIREBASE_SETUP.md`](./FIREBASE_SETUP.md). You'll create a project, enable Firestore, publish security rules, enable Anonymous Auth, and paste the resulting `firebaseConfig` into `index.html`.
2. **Calendar (optional)** — follow [`CALENDAR_SETUP.md`](./CALENDAR_SETUP.md) to deploy the Apps Script proxy and paste its URL into `CONFIG.calendar.endpoint`. **Read the [security considerations](./CALENDAR_SETUP.md#security-considerations) first** — point the proxy at a dedicated calendar, not your personal one. Until this is done, the calendar column shows *"Calendar not set up yet."* and the rest of the app works normally.
3. **Enable GitHub Pages** — Settings → Pages → Source: **GitHub Actions**.
4. **Push to `main`** — the workflow will deploy automatically.

## Customising for your family

All in `CONFIG` near the top of `index.html`:

```javascript
const CONFIG = {
    children: [
        { id: "rose",  name: "Rose 🌹"   },  // id = stable database key, name = display
        { id: "lucy",  name: "Lucy 🐻‍❄️" },
        { id: "olive", name: "Olive 🦄"  }
    ],
    tasks: [ { text: "Get dressed", emoji: "👕" }, /* ... */ ],
    childSpecificTasks: { rose: [ /* ... */ ], lucy: [ /* ... */ ], olive: [ /* ... */ ] },
    weather: { lat: -38.1481736, lon: 144.336537 },
    calendar: { endpoint: "", calendarName: "Thompson", daysToShow: 3 }
};
```

Notes:
- **Never change a child's `id`** once data exists in Firestore — it's the database key. Changing `name` (including emoji) is safe.
- The `COLORS` array immediately below `CONFIG` has one entry per child, in the same order.
- Weather coordinates: right-click on Google Maps to copy lat/lon for your location.

## Local testing

Don't open `index.html` with `file://` — Firebase and ES module imports will be blocked by CORS. Use any static server, e.g.:

```bash
python -m http.server 8000
# then visit http://localhost:8000/
```

You should see `✓ Signed in to Firebase anonymously` in the browser console.

## Files

- [`index.html`](./index.html) — the entire app
- [`FIREBASE_SETUP.md`](./FIREBASE_SETUP.md) — Firestore + Anonymous Auth setup
- [`CALENDAR_SETUP.md`](./CALENDAR_SETUP.md) — Google Apps Script calendar proxy setup
- [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml) — GitHub Pages deploy workflow
