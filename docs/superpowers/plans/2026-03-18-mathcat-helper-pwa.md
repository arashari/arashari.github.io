# Math Cat Helper PWA Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add offline-ready PWA support to `mathcat-helper/index.html` so it's installable on mobile/desktop and works fully without a network connection.

**Architecture:** Three new files alongside `index.html` (icon.svg, manifest.json, sw.js) plus four lines added to `index.html`. The service worker uses a cache-first strategy, pre-caching all app files on install and cleaning up stale caches on activate.

**Tech Stack:** Vanilla HTML/CSS/JS, Web App Manifest API, Service Worker API. No build tools, no dependencies.

---

## Chunk 1: Icon, Manifest, and Service Worker

### Task 1: Create the app icon

**Files:**
- Create: `mathcat-helper/icon.svg`

- [ ] **Step 1: Create `mathcat-helper/icon.svg`**

  Create the file with this exact content:

  ```svg
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
    <defs>
      <linearGradient id="bg" x1="0%" y1="0%" x2="100%" y2="100%">
        <stop offset="0%" style="stop-color:#ff7043"/>
        <stop offset="100%" style="stop-color:#ffa726"/>
      </linearGradient>
    </defs>
    <rect width="512" height="512" rx="96" ry="96" fill="url(#bg)"/>
    <text x="256" y="320" text-anchor="middle" font-size="280" font-family="Apple Color Emoji,Segoe UI Emoji,Noto Color Emoji,sans-serif">🐱</text>
  </svg>
  ```

- [ ] **Step 2: Verify the SVG renders correctly**

  Open `mathcat-helper/icon.svg` directly in a browser. You should see an orange gradient rounded square with a cat emoji centered in it. The cat should be large and clearly visible.

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/icon.svg
  git commit -m "feat: add PWA app icon (SVG)"
  ```

---

### Task 2: Create the web app manifest

**Files:**
- Create: `mathcat-helper/manifest.json`

- [ ] **Step 1: Create `mathcat-helper/manifest.json`**

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
      {
        "src": "icon.svg",
        "sizes": "any",
        "type": "image/svg+xml",
        "purpose": "any maskable"
      }
    ]
  }
  ```

- [ ] **Step 2: Validate the manifest JSON**

  Run:
  ```bash
  python3 -c "import json; json.load(open('mathcat-helper/manifest.json')); print('valid JSON')"
  ```
  Expected output: `valid JSON`

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/manifest.json
  git commit -m "feat: add PWA web app manifest"
  ```

---

### Task 3: Create the service worker

**Files:**
- Create: `mathcat-helper/sw.js`

- [ ] **Step 1: Create `mathcat-helper/sw.js`**

  ```js
  const CACHE_NAME = 'mathcat-v1';
  const PRECACHE_URLS = ['./', './manifest.json', './icon.svg'];

  self.addEventListener('install', event => {
    event.waitUntil(
      caches.open(CACHE_NAME).then(cache => cache.addAll(PRECACHE_URLS))
    );
  });

  self.addEventListener('activate', event => {
    event.waitUntil(
      caches.keys().then(keys =>
        Promise.all(
          keys.filter(key => key !== CACHE_NAME).map(key => caches.delete(key))
        )
      ).then(() => self.clients.claim())
    );
  });

  self.addEventListener('fetch', event => {
    event.respondWith(
      caches.match(event.request).then(cached => cached || fetch(event.request))
    );
  });
  ```

- [ ] **Step 2: Verify the JS is syntactically valid**

  Run:
  ```bash
  node --check mathcat-helper/sw.js && echo "syntax OK"
  ```
  Expected output: `syntax OK`

- [ ] **Step 3: Commit**

  ```bash
  git add mathcat-helper/sw.js
  git commit -m "feat: add service worker for offline caching"
  ```

---

### Task 4: Wire PWA into index.html

**Files:**
- Modify: `mathcat-helper/index.html`

  Add 3 tags to `<head>` (after the existing `<meta name="viewport">` tag, around line 5):
  ```html
  <link rel="manifest" href="manifest.json">
  <meta name="theme-color" content="#ff7043">
  <link rel="apple-touch-icon" href="icon.svg">
  ```

  Add SW registration before `</body>` (after line 658, before `</body>`):
  ```html
  <script>
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('sw.js')
        .catch(err => console.warn('SW registration failed:', err));
    }
  </script>
  ```

- [ ] **Step 1: Add manifest link and meta tags to `<head>`**

  In `mathcat-helper/index.html`, find:
  ```html
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>🐱 Math Cat Helper</title>
  ```

  Replace with:
  ```html
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="manifest" href="manifest.json">
  <meta name="theme-color" content="#ff7043">
  <link rel="apple-touch-icon" href="icon.svg">
  <title>🐱 Math Cat Helper</title>
  ```

- [ ] **Step 2: Add service worker registration before `</body>`**

  In `mathcat-helper/index.html`, find:
  ```html
  // ── Start ──
  init();
  </script>
  </body>
  </html>
  ```

  Replace with:
  ```html
  // ── Start ──
  init();
  </script>
  <script>
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('sw.js')
        .catch(err => console.warn('SW registration failed:', err));
    }
  </script>
  </body>
  </html>
  ```

- [ ] **Step 3: Verify the HTML is valid**

  Open `mathcat-helper/index.html` in a browser. The page should load and function identically to before — no visual changes, all interactions work.

- [ ] **Step 4: Commit**

  ```bash
  git add mathcat-helper/index.html
  git commit -m "feat: wire PWA manifest and service worker into index.html"
  ```

---

## Chunk 2: Verification

### Task 5: End-to-end PWA verification

No code changes — verification only.

- [ ] **Step 1: Serve the app locally over HTTP**

  Run:
  ```bash
  python3 -m http.server 8080 --directory .
  ```

  Open: `http://localhost:8080/mathcat-helper/`

  Note: Service workers require `localhost` or HTTPS. `http://localhost` qualifies.

- [ ] **Step 2: Verify service worker registers**

  In Chrome DevTools → Application → Service Workers:
  - You should see `sw.js` listed with status **activated and running**
  - No errors in the console

- [ ] **Step 3: Verify manifest is recognized**

  In Chrome DevTools → Application → Manifest:
  - Name: `Math Cat Helper`
  - Short name: `Math Cat`
  - Theme color: `#ff7043` (orange)
  - Icons section shows the SVG icon
  - No manifest errors

- [ ] **Step 4: Verify offline works**

  In Chrome DevTools → Network tab, check **Offline** checkbox.
  Reload `http://localhost:8080/mathcat-helper/`.
  Expected: App loads fully. Solver and generator both function. No network errors.

  Uncheck Offline when done.

- [ ] **Step 5: Verify install prompt appears**

  In Chrome, look for the install icon in the address bar (or the "Install Math Cat Helper" option in the browser menu). Clicking it should install the app in standalone mode (no browser chrome, orange title bar).

  On mobile: visit the URL → browser menu → "Add to Home Screen". The installed icon should show the cat emoji on an orange background.

- [ ] **Step 6: Verify the deployed app on GitHub Pages**

  After pushing to `main`, visit `https://arashari.github.io/mathcat-helper/` and repeat Steps 2–5. GitHub Pages serves over HTTPS so the full PWA install flow is available.

- [ ] **Step 7: Final commit if any fixes were made**

  If any issues were found and fixed during verification, commit those fixes:
  ```bash
  git add -p
  git commit -m "fix: PWA verification fixes"
  ```
