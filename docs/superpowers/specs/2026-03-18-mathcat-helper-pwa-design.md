# Math Cat Helper — PWA Design

**Date:** 2026-03-18
**Scope:** Add offline-ready PWA support + mobile install to `mathcat-helper/index.html`

---

## Goal

Make `mathcat-helper` installable as a mobile/desktop app and fully functional offline, with no change to the existing UI or behavior.

---

## Approach

SVG icon + minimal files (3 new files, 1 modified).

The app is a single self-contained HTML file with zero external dependencies. This makes PWA implementation trivial — the entire app is cached in one request.

---

## Files

### New: `mathcat-helper/icon.svg`

An SVG icon with:
- Orange gradient background (`#ff7043` → `#ffa726`), matching the app header
- 🐱 cat emoji centered, large enough to read at 192×192 and 512×512
- Rounded corners (like iOS/Android icons)

Used as the home screen icon. One SVG file covers all sizes via `sizes="any"` in the manifest.

**iOS note:** Safari ignores the manifest and requires a PNG for `apple-touch-icon`. Using an SVG here means iOS will fall back to a screenshot-based icon. This is an accepted limitation for this implementation. A future iteration could add a 180×180 PNG to improve iOS home screen appearance.

### New: `mathcat-helper/manifest.json`

```json
{
  "name": "Math Cat Helper",
  "short_name": "Math Cat",
  "description": "Solver & Puzzle Generator for Math Cat card game",
  "start_url": "/mathcat-helper/",
  "scope": "/mathcat-helper/",
  "display": "standalone",
  "background_color": "#fff8f0",
  "theme_color": "#ff7043",
  "icons": [
    { "src": "icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any maskable" }
  ]
}
```

`scope` is explicitly set to `/mathcat-helper/` (rather than relying on the default) so install behavior is predictable if the manifest path ever changes.

### New: `mathcat-helper/sw.js`

Cache-first service worker with three event handlers:

- **Install event:** Pre-caches `./`, `./manifest.json`, `./icon.svg`
- **Activate event:** Iterates `caches.keys()`, deletes any cache whose name doesn't match the current `CACHE_NAME` constant, then calls `self.clients.claim()` so the new SW takes control of open pages immediately
- **Fetch event:** Serves from cache first; falls back to network if not cached
- **Cache name:** `mathcat-v1` (bump string to force full cache refresh on updates)
- Scope is limited to `/mathcat-helper/` automatically (sw.js lives there)

### Modified: `mathcat-helper/index.html`

Add to `<head>`:
```html
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#ff7043">
<link rel="apple-touch-icon" href="icon.svg">
```

Add before `</body>`:
```html
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('sw.js')
      .catch(err => console.warn('SW registration failed:', err));
  }
</script>
```

---

## Behavior

| Scenario | Result |
|---|---|
| First visit (online) | App loads normally, SW installs, files cached |
| Subsequent visits (online) | Served from cache instantly |
| Visit while offline | Served from cache, fully functional |
| Install to home screen | Standalone mode, no browser chrome, orange theme |
| App update (bump `mathcat-v1` → `mathcat-v2`) | Old cache deleted on activate, new files fetched |

---

## Constraints

- No build system, no npm — all vanilla
- No change to existing CSS/JS/HTML structure
- This repo (`arashari.github.io`) is a **user/org GitHub Pages site** that deploys from the repo root at `https://arashari.github.io/`. The `mathcat-helper` directory is served at `https://arashari.github.io/mathcat-helper/`, making `/mathcat-helper/` the correct absolute path for `start_url` and `scope`
- GitHub Pages serves over HTTPS — service workers work without any config
- iOS Safari does not support SVG `apple-touch-icon`; the iOS home screen icon will be a screenshot fallback (accepted limitation)
