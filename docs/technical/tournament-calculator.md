# Tournament Calculator

`tools/tournament-calculator.html` — fully self-contained (inline
`<style>` + `<script>`), no server dependency.

## Dual-meet mode

A dual-meet category exposes two inputs: **Pairs per club** (default 5) and
**Number of brackets** (`groupDual`, default 1).

- **Pairs per club writes the same value to both `teamsA` and `teamsB`.**
  The internal split is deliberately kept rather than collapsed to a single
  field: the match math (`calcDualCategory`, `dualGroupStage`) handles
  asymmetric club sizes correctly, the CSV schema's `teams_a`/`teams_b`
  columns stay stable, and asymmetry is still reachable through CSV import
  or a hand-edited plan. The "club pair counts differ" note exists to
  surface exactly that case, and must stay for it.
- **Bracket count is `groupDual`, separate from standard mode's `group`.**
  The format toggle is global and flips every category at once, so a shared
  field would destroy a carefully-set standard-mode bracket layout the
  instant someone peeked at dual mode, with no undo. Two fields make
  toggling formats lossless. Standard mode's `group` default is 4.

## PWA (installable, offline)

**The only page with offline support.** [Control
Center](control-center.md) is installable too, but has no service worker.
The calculator is the only page where *offline* is both useful and safe: it
has no live data (nothing it renders can go stale, so caching can never show
a wrong score — the exact failure mode that rules out full PWA treatment for
a live-data page like Control Center or any event page), it's genuinely
useful without a network (schedule planning happens at venues, on hotel
wifi), and it's evergreen (not tied to an event lifecycle that would strand
a service worker). The event templates, the scoresheet generator, and
archived events are deliberately non-installable altogether.

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

**Persistence works in any tab, not just an installed one.** `save()`/
`load()` persist every field to `localStorage`, so a plan survives across
visits in a plain browser tab. They also retain a `window.storage` branch —
a claude.ai artifacts runtime API that is undefined on GitHub Pages — so the
page still works if opened as an artifact.

## Dual Meet Sheet Generator handoff

Selecting the Dual meet format reveals a box under the export buttons whose
button copies the plan CSV to the clipboard and opens the master workbook's
`/copy` URL, so the operator lands in Google's "Make a copy" dialog and
pastes into the
[generator's](dual-meet-sheet-generator.md) sidebar.

Three implementation details that are easy to get wrong:

- **`buildPlanCsv()` is shared with `exportCSV()`**, so the clipboard and the
  downloaded file are byte-identical.
- **The clipboard write starts inside the click handler and is not awaited.**
  The anchor's own `target="_blank"` navigation opens the dialog; awaiting the
  clipboard promise first would push that navigation outside the user gesture
  and into the popup blocker.
- **There's an `execCommand('copy')` fallback**, because the async clipboard
  API is secure-context only. If both fail, the message points at Export CSV
  rather than failing silently.

The master's file ID is hard-coded in the anchor's `href`. A Drive file ID is
stable across folder moves, so only replacing the workbook itself breaks the
link. The show/hide becomes a swap once a standard-tournament generator
exists.

---
**Features:** [Tournament Calculator usage](../features/tournament-calculator.md)
**Specs:** [`calculator-dual-meet-spec.md`](../specs/calculator-dual-meet-spec.md) and [`calculator-pwa-spec.md`](../specs/calculator-pwa-spec.md).
