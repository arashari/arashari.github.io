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

### New: `mathcat-helper/manifest.json`

```json
{
  "name": "Math Cat Helper",
  "short_name": "Math Cat",
  "description": "Solver & Puzzle Generator for Math Cat card game",
  "start_url": "/mathcat-helper/",
  "display": "standalone",
  "background_color": "#fff8f0",
  "theme_color": "#ff7043",
  "icons": [
    { "src": "icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any maskable" }
  ]
}
```

### New: `mathcat-helper/sw.js`

Cache-first service worker:

- **Install event:** Pre-caches `./`, `./manifest.json`, `./icon.svg`
- **Fetch event:** Serves from cache first; falls back to network if not cached
- **Cache name:** `mathcat-v1` (bump version to force cache refresh on updates)
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
    navigator.serviceWorker.register('sw.js');
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
| App update (bump `mathcat-v1` → `mathcat-v2`) | Old cache deleted, new files fetched |

---

## Constraints

- No build system, no npm — all vanilla
- No change to existing CSS/JS/HTML structure
- GitHub Pages serves over HTTPS — service workers work without any config
- `start_url: /mathcat-helper/` matches the GitHub Pages deployment path for `arashari.github.io`
