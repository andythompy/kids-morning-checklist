# Morning Checklist - AI Agent Instructions

## Project Architecture

**Single-File Web App** for Google Nest Hub Max (1280x800) deployed via GitHub Pages. The entire application lives in `index.html` (719 lines) with no build process.

### Core Stack
- **HTML5**: Single file architecture with embedded CSS/JS
- **Tailwind CSS**: CDN-loaded (cdn.tailwindcss.com) - no config file
- **Vanilla JavaScript**: ES6 modules with async/await patterns
- **Firebase v10.7.1**: Modular SDK (app, auth, firestore) from CDN
- **Canvas-confetti v1.9.2**: CDN-loaded for celebration effects
- **Open-Meteo API**: Free weather API (no auth required)

### Data Flow & State Management

**Firebase Firestore as Single Source of Truth**:
- Daily documents in `checklists` collection (e.g., `checklists/2025-11-03`)
- Real-time sync via `onSnapshot()` listener
- Anonymous authentication (no user accounts)
- State shape: `{ "rose-get_dressed": true, "lucy-pack_bag": false, ... }`

**State Keys Pattern**: Use `childId` (not display name) for database stability
```javascript
// Database key uses stable id: "rose-get_dressed"
getStateKey("rose", "Get dressed") → "rose-get_dressed"

// Display shows: "Rose 🌹"
CONFIG.children = [{ id: "rose", name: "Rose 🌹" }, ...]
```

**Critical**: Always use `child.id` for state operations, `child.name` for UI display. This allows emoji/display name changes without breaking existing Firebase data.

## Configuration Pattern

Edit the `CONFIG` object (~line 280) for all customizations:

```javascript
const CONFIG = {
    children: [
        { id: "rose", name: "Rose 🌹" },  // id=database key, name=display
        { id: "lucy", name: "Lucy 🪿" },
        { id: "olive", name: "Olive 🦄" }
    ],
    tasks: [{ text: "Get dressed", emoji: "👕" }, ...],
    weather: { lat: -38.1481736, lon: 144.336537 }  // Geelong, Australia
};

// Color schemes (3 entries) map to children array indices
const COLORS = [
    { bg: 'bg-white', border: 'border-purple-400', ... },  // Rose
    { bg: 'bg-white', border: 'border-blue-400', ... },    // Lucy  
    { bg: 'bg-white', border: 'border-pink-400', ... }     // Olive
];
```

## Development Workflows

### Local Testing
```powershell
# NEVER use file:// protocol - causes CORS errors
# Use Live Server extension or Python HTTP server
python -m http.server 8000
```

### Deployment
```powershell
git add index.html
git commit -m "Description"
git push  # GitHub Actions auto-deploys to Pages
```

**GitHub Actions**: `.github/workflows/deploy.yml` deploys on push to `main`. No build step - uploads entire repo.

### Firebase Setup
See `FIREBASE_SETUP.md` for initial Firebase configuration. **Must enable Anonymous Authentication** or Firestore security rules will block access.

## Key Technical Patterns

### Weather Theming
4 themed backgrounds with inline SVG data URIs:
- `weather-fine`: Sunrise gradient (22-30°C)
- `weather-hot`: Bright sun with rays (>30°C)
- `weather-cold`: Snowflakes over snow banks (<10°C)
- `weather-rain`: Dark clouds with rain lines (WMO codes 51-67, 80-82, 95-99)

Body class changes dynamically: `document.body.className = 'weather-hot'`

### Mobile Responsiveness
- **≤768px**: Horizontal scroll with `scroll-snap-type: x mandatory`
- **Columns**: `min-width: calc(100vw - 3rem)` for one column per viewport
- **Container**: `scroll-padding-left: 1rem` for left-aligned snap
- **≥769px**: Desktop 3-column layout with flexbox

### Touch Event Handling
Prevent task clicks during scrolling:
```javascript
taskElement.addEventListener('click', (e) => {
    if (e.target.closest('.tasks-scroll').dataset.scrolling !== 'true') {
        toggleTask(child.id, task.text, taskElement, column);
    }
});
```

## Common Operations

### Adding a New Child
1. Add to `CONFIG.children`: `{ id: "newid", name: "Name" }`
2. Add color scheme to `COLORS` array (3 total)
3. Firebase data auto-creates on first interaction

### Adding a New Task
Add to `CONFIG.tasks` array - applies to all children automatically

### Modifying Display Names
Only change `name` field in `CONFIG.children`. Never change `id` (breaks existing Firebase data).

### Testing Weather Themes
Uncomment test UI (~line 240), or call: `testWeather('hot', 35)`

## Critical Constraints

- **Single File Only**: All code must live in `index.html`
- **No Build Tools**: Pure HTML/CSS/JS with CDN dependencies
- **Target Device**: Google Nest Hub Max (1280x800) - optimize for this resolution
- **Daily Reset**: State automatically scopes to `YYYY-MM-DD` document keys
- **Firebase Required**: App won't work without Firebase credentials configured

## Debugging Tips

- Check browser console for Firebase auth success: `"✓ Signed in to Firebase anonymously"`
- Firestore errors usually mean Anonymous Auth not enabled
- Mobile scroll issues: verify `scroll-snap-align: start` on columns
- Weather not loading: check coordinates in `CONFIG.weather`
