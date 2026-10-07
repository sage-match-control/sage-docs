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

## Keep-apart groups

`keepApartGroups` is UI state: a list of `{ name, members: [pair, ...] }`, each member
picked from the entered pairs (never typed) so it always equals the string the
draw hashes. Spec:
[keep-apart groups](../specs/implemented/bracket-generator-keep-apart-spec.md).

The fingerprints and their sort are untouched. `orderForDraw(sorted, groups)`
then builds the list the deal runs on: each group's members in fingerprint
order, group 1 first, then every pair not in a group, also in fingerprint
order. `dealBrackets` deals that list round-robin. Any run of up to
`N` consecutive pairs lands in `N` different brackets, so a group no larger
than the bracket count is always split, with no search or retry involved. With
no groups the list is the plain fingerprint order, so groups add nothing to a
draw that has none (the text export included).

Rules the code enforces:

- A group has at least 2 and at most `numBrackets` members. The bracket count
  is the capped one (never more brackets than pairs). A violation refuses the
  draw before a seed is generated or the draw counter moves.
- A member must be entered exactly once. `reconcileGroups()` drops members whose
  line was edited, removed or duplicated, on every change to the pair list, and
  the group says so.
- `groupsApart()` re-checks the finished deal and refuses it if a group shares
  a bracket. It cannot fire while `orderForDraw` is correct; it is there so a
  later edit to either cannot publish a broken draw.
- The groups are part of `inputKey`, so changing them resets the draw counter.
  Their names are not: a name is a label and never reaches the hash.
- A name (`''` means "Group N") is cleaned by `cleanGroupName`: whitespace
  collapsed, trimmed, 40 characters at most. The pencil swaps just the name area
  of the card (`refreshGroupName`), not the whole list, so a click on the
  card's other buttons is not lost when the field blurs. Enter or leaving the
  field saves, Escape cancels. Names go into the DOM only through
  `escapeHtml` or `setAttribute`. `currentGroupNames` snapshots them at draw
  time, like the category and event name.

**Text export.** With groups, the `DRAW VERIFICATION` block lists the
fingerprints group by group (`Group 1 (keep apart, 4 pairs)`, or
`Group 1 "Top seeds" (keep apart, 4 pairs)` for a named one, …, `Everyone
else`; a heading always starts with `Group <n>`, so no name can be read as the
`Seed:` or `Draw:` line the importer looks for) and the method and check instructions say groups are dealt first. The
`BRACKET` lines are never annotated: `SAGE → Import bracket draws` reads
every numbered line under a `BRACKET` heading as a pair name. The image export
only gains a count in its verification line. Result cards tag grouped pairs
`G1`, `G2`…

## Draw types

A `drawMode` radio group picks only where the seed comes from:

- `seeded` (default, labelled **Verifiable Draw**) — the typed seed, or an
  auto seed if the field is blank.
- `random` (labelled **Random Draw**) — always a fresh auto seed; the field
  is disabled and just displays it.

Both then run the same fingerprint draw below, so both are equally
verifiable. There is no unverified mode: a seed costs nothing to keep.

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

`shuffle()` is Fisher-Yates and feeds only
`runShuffleAnimation()`'s cosmetic per-tick frames. Those are thrown away and
never exported, so they need no reproducibility.

### The seed

Typing one is optional; having one is not. A blank field generates an
8-character seed (`crypto.getRandomValues`, alphabet omitting `I`/`O`/`0`/`1`
so it survives being read aloud), fills the field with it and draws — so every
draw is reproducible whether or not anyone asked for a ceremony. Both exports
label the source, `(entered)` or `(auto)`, because only a seed supplied by a
person shows whoever ran the draw did not go looking for one they liked.

The last generated seed is kept in `lastAutoSeed`. If the field still holds it
at the next seeded draw, that draw is labelled `(auto)`, not `(entered)` —
nobody typed it. Switching back to `seeded` with that seed still in the field
clears the field, so a public draw starts from an empty box.

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

## `?category=`, and why it is not remembered

`initCategoryFromQuery` prefills the category field from `?category=` and
stops there — no `localStorage` write, no fallback read, nothing carried to
the next visit. That asymmetry with `?event=` above is deliberate: an event
name is the same all day and worth remembering, while a category resurrected
on a later visit is how someone draws the wrong one.

It is what `SAGE → Open Bracket Generator` in a scoring workbook sends, along
with `?event=`, so the export's category line matches the tab it will be
imported back into. That link is built in `sheets-sync.gs`
(`showBracketGeneratorLink`) and reaches every workbook, dual meets included;
see [bracket draw name import](../specs/implemented/bracket-draw-name-import-spec.md)
§11.

## One tool, no per-event copies

The tool lives only at `/tools/bracket-generator.html`. A live event's own
bracket page is a redirect stub to it (preserving `?event=` and any hash), as
`tools/match-control.html` is for Control Center. An archived event's own copy
is frozen and left as it is: a redirect there would point at a tool that doesn't
carry that event's name.

---
**Features:** [bracket generator](../features/bracket-generator.md)
