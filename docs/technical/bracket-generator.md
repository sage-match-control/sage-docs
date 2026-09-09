# Bracket Generator

`tools/bracket-generator.html` — fully self-contained (inline `<style>` +
`<script>`), no build step, no dependency beyond two Google Fonts
stylesheets, and no network call at run time. It replaced a copy of the same
file duplicated into both event-site templates (and drifting between them);
see the [spec](../specs/implemented/bracket-generator-spec.md) for why it moved.

## The deal

`dealBrackets()` shuffles the pairs, then deals them round-robin into however
many buckets were requested, so bracket sizes differ by at most one. The
requested bracket count clamps to the pair count — you can't ask for more
brackets than you have pairs.

## The draw is verifiable

The ordering is not `Math.random()` — it is a deterministic function of the
seed and the pair list, so anyone can reproduce it without this tool.

For each pair, in the list as entered (trimmed, blank lines dropped):

    fingerprint = SHA-256( seed + "|" + pair )      lowercase hex

Pairs are sorted ascending by that hex string, then dealt round-robin as above.
Ties — which only arise from two identical pair strings, since identical input
hashes identically — break by the pair text, then by input position.

Three details are load-bearing for anyone re-checking a draw:

- **The separator is a single `|`, with no surrounding spaces.**
- **The seed is trimmed and uppercased before hashing** (`normalizeSeed`), and
  the field displays the normalized form, so what is on screen is what was
  hashed.
- **The sort runs on the full 64-character hash.** The 8-character form in the
  text export is display only; sorting on a truncated key would start colliding
  around 65k items.

Because each pair's fingerprint depends only on the seed and its own name, the
order of the pasted list does not affect the result — which is what lets the
export stay verifiable without recording the input order.

`shuffle()` still exists and is still Fisher-Yates, but now feeds only
`runShuffleAnimation()`'s cosmetic per-tick frames. Those are thrown away and
never exported, so they need no reproducibility.

### The seed

Typing one is optional; having one is not. A blank field generates an
8-character seed (`crypto.getRandomValues`, alphabet omitting `I`/`O`/`0`/`1`
so it survives being read aloud), fills the field with it and draws — so every
draw is reproducible whether or not anyone asked for a ceremony. Both exports
label the source, `(entered)` or `(auto)`, because only a seed supplied by a
person shows the organiser did not go looking for one they liked.

### Secure context required

`crypto.subtle` is only available in a secure context. GitHub Pages is HTTPS so
this never bites in production, but a local `file://` open can hit it. The page
**fails closed**: the draw button is disabled and the reason stated. It never
falls back to `Math.random()` while still printing a seed — an export naming a
seed it was not produced from is a false proof, which is worse than no feature.

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
