# Bracket Generator

`tools/bracket-generator.html` — fully self-contained (inline `<style>` +
`<script>`), no build step, no dependency beyond two Google Fonts
stylesheets, and no network call at run time. It replaced a copy of the same
file duplicated into both event-site templates (and drifting between them);
see the [spec](../specs/bracket-generator-spec.md) for why it moved.

## The deal

`dealBrackets()` shuffles the pairs, then deals them round-robin into however
many buckets were requested, so bracket sizes differ by at most one. The
requested bracket count clamps to the pair count — you can't ask for more
brackets than you have pairs.

## One palette source

The page's `:root` block is the only place its colors are declared, screen
**and** export alike — including the four bracket-badge fills/texts
(`--badge-0-fill`/`-text` through `--badge-3-fill`/`-text`). The PNG export is
hand-drawn to a `<canvas>`, which takes color strings rather than CSS
variables, so `exportAsImage()` resolves those same tokens at draw time via a
small `cssVar()` helper (`getComputedStyle` + a shipped-value fallback)
instead of keeping a second, hand-copied palette. Re-skinning the tool is a
single `:root` edit that re-skins the exported image too, with nothing to
keep in step by hand.

Two of the four badge fills — `--badge-2-fill` and `--badge-3-fill` — ship
**darkened** from their nearest palette tokens (`--green-dark` and a stock
purple), because both land under the 4.5:1 contrast floor against their label
text at their natural values. A re-skin that touches either one has to
re-check that contrast rather than restoring the "clean" palette value.

## The event name

Optional, and remembered per browser in `localStorage`
(`sage-bracket-event-name`) — set on `change`, read back on load, wrapped in
`try/catch` since some private-browsing modes throw on access. A `?event=`
query parameter prefills it and overrides whatever's stored, then persists
that value for the next visit with no parameter needed. It's captured into
state at draw time (alongside the category), not read live at export time, so
editing it after a draw doesn't retroactively change what that draw exports.

It reaches three sinks and is safe by construction at each: `textContent` for
the on-page result line, `ctx.fillText` for the canvas (which takes a string,
not markup), and `slugify()` for the filename. It's never interpolated into
an `innerHTML` string.

## Replaced a per-event copy

Four near-identical copies of this file used to exist — two event-site
templates (kept byte-identical by a documented-but-manual convention) plus
one live event's own copy that had already drifted to a stale, pre-S.A.G.E.
palette. The live event's copy is now a redirect stub to
`/tools/bracket-generator.html` (preserving `?event=` and any hash), matching
the precedent `tools/match-control.html` set when Control Center was renamed.
An archived event's own copy is left exactly as it was — it's frozen, and a
redirect there would point at a tool that no longer says that event's name.

---
**Features:** [bracket generator](../features/bracket-generator.md)
