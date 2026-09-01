# Tournament Calculator

`tools/tournament-calculator.html` — fully self-contained (inline
`<style>` + `<script>`), no server dependency. Two independent bodies of
work have shipped against it: dual-meet UI fixes, and turning it into an
installable, offline-capable PWA.

## Dual-meet mode

Dual-meet categories originally exposed **three** inputs — separate `Club A
pairs` / `Club B pairs` fields plus a shared `Number of brackets` field that
dual mode inherited from standard mode's default (4), so a new dual category
opened showing 4 brackets instead of the sensible default of 1.

Fixed by:

- Collapsing the two club-pair inputs into a single **Pairs per club**
  field (default 5) that writes the same value to both `teamsA`/`teamsB`
  internally. The internal split is kept — not collapsed to one field —
  because the match math (`calcDualCategory`, `dualGroupStage`) already
  handles asymmetric club sizes correctly, the CSV schema's `teams_a`/
  `teams_b` columns stay stable, and asymmetry remains reachable via CSV
  import (a plan saved before this change, or one hand-edited, can still
  produce unequal counts — the "club pair counts differ" note that surfaces
  that case was kept for exactly this reason).
- A **separate `groupDual` field**, rather than reusing standard mode's
  `group`. The format toggle is global and flips every category at once;
  overwriting a shared field would destroy a carefully-set standard-mode
  bracket layout the instant someone peeked at dual mode, with no undo. Two
  fields means toggling formats back and forth is lossless. Standard mode's
  `group` default (4) is untouched.

## PWA (installable, offline)

**The only page with offline support.** [Control
Center](control-center.md) later became installable too, but with no
service worker — the calculator remains the only page where *offline* is
both useful and safe: it has no live data (nothing it renders can go
stale, so caching can never show a wrong score — the exact failure mode
that rules out full PWA treatment for a live-data page like Control
Center or any event page), it's genuinely useful without a network
(schedule planning happens at venues, on hotel wifi), and it's evergreen
(not tied to an event lifecycle that would later strand a service
worker). The event templates, the scoresheet generator, and archived
events remain deliberately non-installable altogether.

**Manifest scope is the page path, not the directory**
(`/tools/tournament-calculator.html`) — its own manifest, not the shared
site-wide one (`/assets/favicons/site.webmanifest`, left with an empty
`name` so it can't make anything installable on its own; Control Center's
manifest is a third, separate file for the same reason). Two independent
guards contain the worker's blast radius to this one page: the manifest
scope, and a fetch handler that **passes through anything it doesn't
explicitly own** rather than a catch-all cache-first branch — critical,
since `scoresheet-generator.html` shares the `/tools/` directory and posts
multipart uploads that must never be served from or captured by cache.

**Caching strategy is split by resource type**, not a single blanket rule:
the HTML document itself is network-first with a cache fallback (so a
pushed fix lands on the *next* launch, not the one after — this codebase's
edit-and-commit hot-fix model would otherwise trap installed users on a
stale build); fonts, the pinned SheetJS export library, and static assets
are cache-first. The install-time precache is split the same way — the
core shell is cached atomically (one failed URL fails the whole install),
third-party assets (fonts, SheetJS) are cached opportunistically
(`Promise.allSettled`), so a CDN blip costs only the export button, not the
ability to install at all.

**Persistence was already built, just inert.** The page's `save()`/`load()`
functions already wired up every field to a `window.storage` API — but
that's a claude.ai artifacts runtime API, undefined on GitHub Pages, so both
functions silently no-op in production. The fix was a small `localStorage`
fallback (keeping the `window.storage` branch, in case the page is ever
opened as an artifact again) — not a rewrite. One accepted side effect:
this changes behavior for every visitor, not just installed ones — the
calculator now remembers state across visits in a plain browser tab too.

---
**Features:** [Tournament Calculator usage](../features/tournament-calculator.md)
**Specs:** `specs/calculator-dual-meet-spec.md` and `specs/calculator-pwa-spec.md` in the site repo.
