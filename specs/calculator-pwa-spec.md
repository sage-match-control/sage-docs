# Spec — Tournament Calculator as a PWA

Make `tools/tournament-calculator.html` installable and offline-capable, without
affecting any other page on the site.

**Scope:** this one page. The event templates, the scoresheet generator, and the
archived events are untouched and must remain non-installable.

---

## 1. Why this page and not the others

The calculator is the only page on the site where offline is both **useful and
safe**:

- **No live data.** It computes tournament timings from inputs the user types.
  Nothing it renders can go stale, so caching it cannot show anyone a wrong
  score — the failure mode that rules PWAs out for the event pages.
- **Genuinely useful without a network.** Schedule planning happens at venues,
  on hotel wifi, in cars. The core calculation is pure client-side JS.
- **Evergreen.** Not tied to an event lifecycle, so there is no archiving step
  that would later strand a service worker.

The event pages are the opposite on all three counts. Do not generalise this
spec to them without revisiting each point.

## 2. What the page is today

| | |
|---|---|
| File | `tools/tournament-calculator.html`, ~1114 lines / ~63 KB, fully self-contained (inline `<style>` + `<script>`) |
| Manifest | Links the shared `/assets/favicons/site.webmanifest`, whose `name`/`short_name` are empty strings — nothing on the site is currently installable |
| Fonts | Google Fonts — Archivo Black, Barlow Condensed, Inter |
| SheetJS | `cdnjs.cloudflare.com/.../xlsx/0.18.5/xlsx.full.min.js` |
| Images | `/assets/logo.png` |
| State | A complete `save()` / `load()` pair exists and the boot path already restores every field — but both are gated on `window.storage`, a claude.ai artifacts API that does not exist on GitHub Pages, so **nothing persists in production today** (§8) |

**SheetJS is export-only.** It is used at four call sites, all in one function:
`aoa_to_sheet`, `book_new`, `book_append_sheet`, `writeFile`. Every calculation
in the tool works without it; only the "download breakdown as .xlsx" button
depends on it. That makes it a precache target, not a blocker.

## 3. Verify first

Short list — this project has no design-invalidating gates, but confirm these
before writing code.

| # | Check | Note |
|---|---|---|
| 1 | Chrome/Android install criteria currently in force | **Informational only — the service worker ships regardless** (decided, for offline support). This just tells you whether the automatic install prompt depends on it, which affects how users discover the install. iOS Safari installs from the manifest alone |
| 2 | `cf-cache`/CORS on the SheetJS URL | Already confirmed: cdnjs returns `Access-Control-Allow-Origin: *` and `immutable`, so it precaches as a normal (non-opaque) response |
| 3 | `.webmanifest` MIME from Pages | Already confirmed: `application/manifest+json; charset=utf-8`. No workaround needed |

## 4. Files

**New**

```
tools/tournament-calculator.webmanifest
tools/sw.js
```

**Changed**

```
tools/tournament-calculator.html   (manifest link + registration snippet + §8 localStorage fallback)
```

**Explicitly not changed:** `/assets/favicons/site.webmanifest`. It stays as-is,
icons-only with empty `name`, serving the other 16 pages. Do not "fix" its empty
name — that is what keeps every other page non-installable (§7).

## 5. Manifest — `tools/tournament-calculator.webmanifest`

```json
{
  "name": "SAGE Tourney Calculator",
  "short_name": "SAGE Calc",
  "description": "Estimate tournament duration, court load, and schedule breakdown.",
  "start_url": "/tools/tournament-calculator.html",
  "scope": "/tools/tournament-calculator.html",
  "display": "standalone",
  "theme_color": "#14263C",
  "background_color": "#F6F7F2",
  "icons": [
    { "src": "/assets/favicons/android-chrome-192x192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/assets/favicons/android-chrome-512x512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

> `name` is the full title, shown in the install prompt and app settings.
> `short_name` is the **home screen label**, which Android and iOS truncate at
> roughly 12–15 characters — "SAGE Tourney Calculator" would render as
> "SAGE Tourne…". `"SAGE Calc"` keeps the branding and fits. Change it if you
> want different wording, but keep it short.

- `theme_color` and `background_color` come from the page's own palette
  (`--navy` and `--bg`), not the shared manifest's `#ffffff` — a mismatched
  `background_color` shows as a white flash on every launch.
- `scope` is deliberately the **page path, not the directory**. Scope matching
  is a string prefix, so this keeps `scoresheet-generator.html` out of the app
  (§7). Non-directory scopes are legal but less commonly used — confirm the
  browser accepts it during §9 step 5. If it objects, fall back to `/tools/`
  and rely on the pass-through handler alone, which still contains the
  behaviour; the precise scope is defence in depth, not the only guard.
- **Icons reuse the existing site icons — decided.** Note the consequence: if
  an event page ever becomes installable too, the two would be visually
  indistinguishable on a home screen. Not a problem while this is the only PWA.

In the page's `<head>`, replace the shared manifest link:

```html
<link rel="manifest" href="/tools/tournament-calculator.webmanifest">
```

## 6. Service worker — `tools/sw.js`

### 6.1 Caching strategy

Split by resource type. This is the load-bearing decision:

| Resource | Strategy | Why |
|---|---|---|
| `tournament-calculator.html` (the document) | **Network-first, cache fallback** | The document is the whole app — all logic is inline. Network-first means a pushed fix lands on the next launch, not the one after. Offline still works via the fallback |
| Google Fonts CSS + font files | Cache-first | Effectively immutable, and a font miss is a visible layout shift |
| SheetJS (`xlsx.full.min.js`) | Cache-first | Version-pinned in the URL and served `immutable` |
| `/assets/logo.png`, icons | Cache-first | Static |
| Everything else | **Pass through untouched** | §7 |

Network-first on the document costs one round trip (~64 KB) per launch when
online. That is the right trade for a tool whose correctness matters more than
its cold-start time, and it preserves this codebase's edit-and-commit hot-fix
model rather than trapping users on a cached build.

### 6.2 Precache resilience

Precache the shell **atomically**, and the third-party assets **opportunistically**:

```js
const CACHE = 'sage-calc-v1';

const SHELL = [
  '/tools/tournament-calculator.html',
  '/assets/logo.png',
  '/assets/favicons/android-chrome-192x192.png',
  '/assets/favicons/android-chrome-512x512.png'
];

const OPTIONAL = [
  'https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js',
  'https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700&display=swap'
];

self.addEventListener('install', e => e.waitUntil((async () => {
  const cache = await caches.open(CACHE);
  await cache.addAll(SHELL);                     // atomic — fail install if this fails
  await Promise.allSettled(OPTIONAL.map(u => cache.add(u)));  // never block install
  self.skipWaiting();
})()));
```

`cache.addAll` is atomic: one failed URL fails the whole install. Putting cdnjs
or Google Fonts in that list would mean **a third-party blip prevents the app
from installing at all**. `allSettled` on the optional set avoids that; a missing
SheetJS only costs the export button, and it will be picked up on a later fetch.

Font *files* (`fonts.gstatic.com`) are referenced from inside the Google Fonts
stylesheet, so they cannot be listed upfront — cache them on first fetch via the
runtime handler instead.

### 6.3 Activation

```js
self.addEventListener('activate', e => e.waitUntil((async () => {
  const keys = await caches.keys();
  await Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)));
  await self.clients.claim();
})()));
```

`skipWaiting()` + `clients.claim()` means a new worker takes over immediately
rather than waiting for every tab to close. Bump `CACHE` on any change to the
cached set.

### 6.4 Registration

At the end of the page's inline script:

```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/tools/sw.js', { scope: '/tools/tournament-calculator.html' })
      .catch(() => { /* PWA is an enhancement; never break the page over it */ });
  });
}
```

Registration failure must be silent and non-fatal. The calculator has to keep
working in browsers with service workers disabled, in private windows, and
behind enterprise policies.

## 7. Containment — do not leak into `/tools/`

A worker at `/tools/sw.js` is *technically* scoped to `/tools/`, which contains
`scoresheet-generator.html`. Two independent guards keep the blast radius to one
page:

1. **Manifest `scope`** is the page path (§5), so navigating to the scoresheet
   generator leaves the installed app window rather than absorbing it.
2. **The fetch handler passes through anything it does not explicitly own.** Do
   not write a catch-all cache-first branch. Requests for
   `scoresheet-generator.html` must reach the network unmodified and uncached —
   it posts multipart uploads to Cloud Run and must never be served from cache.

This is worth an explicit test (§10), because a sloppy `fetch` handler is the
easy way to silently break a neighbouring tool.

The narrower `/tools/sw.js` placement is deliberate: a worker at the site root
would scope to everything, including the event pages, where cached stale data is
actively harmful.

## 8. Activate the existing persistence

**In scope — ships with this work.** Reopening an installed app with its own
icon to a blank form reads as broken in a way a browser tab does not.

This is **not** a from-scratch feature. The page already has the whole thing:

```js
async function save(){
  try{ if(window.storage) await window.storage.set('ttc-config', JSON.stringify(readForm())); }catch(e){}
}
async function load(){
  try{
    if(!window.storage) return null;
    const r=await window.storage.get('ttc-config');
    return r?JSON.parse(r.value):null;
  }catch(e){ return null; }
}
```

`save()` is called from `recalc()` (so on every input change) and `load()` from
the boot IIFE, which already restores title, date, start, courts, duration,
buffer, format, both club names, and the full `cats` array. All of it works.

The problem is only that both are gated on **`window.storage`** — a claude.ai
artifacts runtime API. On GitHub Pages it is `undefined`, so `save()` silently
no-ops, `load()` returns `null`, and boot falls through to `[defaultCat()]`.
The author's own comment says as much: *"works in claude.ai artifacts; safely
skipped elsewhere."*

**The change is to add a `localStorage` fallback to those two functions —
roughly four lines.** Keep the `window.storage` branch so the page keeps working
if it is ever opened as an artifact again:

```js
async function save(){
  try{
    const v = JSON.stringify(readForm());
    if(window.storage) await window.storage.set('ttc-config', v);
    else localStorage.setItem('ttc-config', v);
  }catch(e){}
}
async function load(){
  try{
    if(window.storage){ const r = await window.storage.get('ttc-config'); return r?JSON.parse(r.value):null; }
    const v = localStorage.getItem('ttc-config');
    return v?JSON.parse(v):null;
  }catch(e){ return null; }
}
```

The existing `try/catch` wrapping is already the right shape — it matches how
all six event pages guard their own `localStorage` access, and it keeps private
windows and storage-disabled browsers from breaking the page.

### 8.1 Two consequences, accepted deliberately

- **It changes behaviour for every visitor, not just installed users.** The
  calculator now remembers state across visits in a plain browser tab too.
  Accepted — but it is a product change, not a PWA-only one, so don't be
  surprised when someone reports the form "isn't blank any more".
- **Restored configs predate the current schema.** `load()` spreads saved
  categories straight into `cats`, so a config saved before the dual-meet change
  has no `groupDual`. It degrades correctly — both `c.groupDual||1` in the card
  and `Math.round(c.groupDual)||1` in `calcDualCategory` fall back to 1 — so no
  migration is needed now. But once persistence is live in production, future
  schema changes will need this considered rather than assumed.

## 9. Implementation order

1. Add the `localStorage` fallback to `save()` / `load()` (§8) — independent of
   everything else, and worth landing first so it gets exercised on its own
2. Add `tools/tournament-calculator.webmanifest` (§5)
3. Swap the page's `<link rel="manifest">` to it (§5)
4. Add `tools/sw.js` with the split strategy, resilient precache, and
   pass-through default (§6)
5. Add the guarded registration snippet (§6.4)
6. Verify install + offline on a real Android device, not just desktop DevTools
   — including that the non-directory `scope` (§5) is accepted

Step 1 stands alone and is separately revertible. Steps 2–3 make the page
installable on iOS Safari. Steps 4–5 add offline support and the Android
install prompt.

## 10. Acceptance checklist

**Install**

- [ ] Chrome on Android offers install; the app launches standalone with no
      browser chrome, correct icon, and no white flash on launch.
- [ ] iOS Safari "Add to Home Screen" produces the same.
- [ ] **No other page on the site became installable** — check
      `scoresheet-generator.html` and at least one archived event page.

**Offline**

- [ ] Airplane mode, launch the installed app: the page loads and calculations
      work.
- [ ] Fonts render correctly offline (no fallback-font layout shift).
- [ ] The .xlsx export works offline, having visited the page online once.
- [ ] First-ever visit with cdnjs unreachable still installs and still
      calculates; only the export button degrades.

**Updates — the one most likely to be missed**

- [ ] Edit a visible string in the HTML, commit, push. Reload the installed app
      while online: **the change appears on that launch**, not the next one.
- [ ] Old caches are deleted after a `CACHE` version bump (check DevTools →
      Application → Cache Storage).

**Persistence (§8)**

- [ ] Fill in a plan, close the tab, reopen: every field is restored — title,
      date, start, courts, duration, buffer, format, club names, and all
      categories.
- [ ] Same in the installed app: close it fully, relaunch, state survives.
- [ ] A dual-meet plan round-trips with the right `groupDual`, and a config
      saved before that field existed still loads, falling back to 1 bracket.
- [ ] Private/incognito window and a browser with site data blocked: the
      calculator still loads and works, just without restoring.
- [ ] Corrupt the stored value by hand (set `ttc-config` to `not json`) — the
      page still boots to a default category rather than throwing.

**Containment**

- [ ] `scoresheet-generator.html` loads normally with the worker active, and its
      CSV upload to Cloud Run still works — confirm the request reaches the
      network and is not served or intercepted from cache.
- [ ] Navigating from the installed app to `scoresheet-generator.html` leaves
      the app window (confirms the `scope` string is doing its job).
- [ ] Unregistering the worker and hard-reloading returns the calculator to
      normal behaviour with no errors.
- [ ] With service workers blocked (private window / disabled), the calculator
      still loads and works.

## 11. Out of scope

- Making any event page, template, or archived page a PWA. §1 explains why the
  reasoning does not transfer.
- A dedicated calculator app icon. **Decided: reuse the site icons** (§5).
- Push notifications, background sync, or any install-prompt UI. The browser's
  native prompt is sufficient.
- Self-hosting SheetJS. Precaching the pinned cdnjs URL covers the realistic
  case, since installation implies at least one online visit; vendoring ~900 KB
  into the repo for an export button is a poor trade.
- Changing `/assets/favicons/site.webmanifest` or any other page's manifest link.
