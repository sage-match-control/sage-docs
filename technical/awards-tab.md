# Awards tab

A fifth tab on `tools/match-control.html` showing each category's podium and
exporting it as SAGE-branded PNGs. Entirely **read-only and derived** — it
writes nothing, needs no auth, and adds no new sync surface or
`sage-tools-api` dependency. Everything it shows comes from `MATCHES` (the
day's parsed match rows) and `STANDINGS`, already loaded by the console for
the other tabs.

## Podium derivation

`buildPodiums()` is the single pure function everything else consumes:
`(MATCHES, STANDINGS) -> [{ category, source, gold, silver, bronze,
bronzeWalkover, warning }]`, ordered to match Standings' own category order.

For each category:

1. **Find the decisive Final.** Among all matches tagged `F` for that
   category (there can be more than one, for a twice-to-beat bracket — see
   below), the decisive one is the **resolved instance with the highest
   instance number**, where *resolved* means played, or decided by a bye
   walkover. A category with no `F` match at all (pure round robin) falls
   back to its top-3 standings rows instead, tagged `source: 'standings'`.
2. **Find the decisive Bronze**, the same way.
3. **Gold/silver** = winner/loser of the decisive Final. **Bronze** = winner
   of the decisive Bronze.
4. Anything that can't be resolved cleanly — a tied score, or both sides of
   a match reading `BYE` — produces a `warning` naming the match number and
   **suppresses the whole category's podium** rather than guessing. Nothing
   here ever fabricates a winner.

### Twice-to-beat resolution

A bracket can run a Final as `F(1)`/`F(2)` — the second match only happens
if the first goes the challenger's way. `matchInstanceOf()` reads the
trailing `(1)`/`(2)` off the team code. The decisive-match search always
prefers the **highest resolved instance**: if only `F(1)` is played, its
result stands; once `F(2)` is also played, it supersedes `F(1)`'s result
entirely.

### Byes, and why they're detected the way they are

A twice-to-beat bracket often has no Bronze match to actually *play* — third
place is already determined by the bracket structure. Rather than special-
case that in code, it's encoded in the data: the operator types `BYE` into
the opponent's team code or either player-name cell, making the real side a
winner by walkover. `matchByeSide(m)` checks **all three cells** (both team
codes, both pairs of player names) — deliberately, since the sheet
autofills these by formula, so whichever cell the operator overrides has to
be the one that's checked. A bye match is decisive **without scores** —
`rowsToMatches()` sets `played` from the score columns, so a bye match has
`played === false`, and the podium derivation explicitly does not gate on
`played` when a bye side is present.

A bye-decided category shows a **Walkover** tag on that medalist instead of
a score, and the bye side is never rendered as a person anywhere.

### Byes must not surface where a match looks playable

A bye is never a real match, so it's filtered out of every surface that
presents match data as something to watch or play, not just the podium:

| Surface | What's filtered |
| --- | --- |
| Standings | Any row whose team code/player1 is `BYE`; if a category's *decisive* Bronze was bye-decided, the whole Bronze block is dropped (not just that row) — a one-sided "Bronze Battle" showing one team with no opponent is worse than showing nothing |
| Live Matches | Bye matches excluded from court grouping entirely |
| Match Finder | A team's own bye match never appears in their schedule, and can never be flagged "Next Up" |
| Team index / autocomplete | `BYE` is never indexed as a searchable name |

## Overall Champion (dual-meet only)

Deliberately **the same metric as Standings' own club win-total
summary** — total wins across Round Robin rows only, per club — not a
podium-derived count (e.g. gold medals), so the two tabs can never disagree
about who's ahead. A tie is shown explicitly as "Tied," never resolved by an
invented tiebreak.

## Image export — Canvas 2D, no library

`tools/match-control.html` has zero JS dependencies (Google Fonts is the
only external resource), so the export draws directly with the Canvas 2D
API rather than pulling in `html2canvas` or a similar library. Cost: the
card design lives in drawing code, not CSS.

**Per-category card:** fixed 1080×1350 logical canvas, rendered at 2× (with
a fallback to 1× if that would exceed a safe canvas-area budget — iOS
Safari caps total canvas area near 16.7M pixels). Each medal row fills its
whole box in the medal tint once decided (dark navy text — checked for
contrast: ~7.3:1 on gold, ~8.9:1 on silver, ~4.7:1 on bronze); an undecided
placing stays on the plain navy panel, so the row's own color signals
decided-vs-not.

**Whole-tournament sheet:** every category in a grid (column count picked
by category count, to keep the image from becoming an ever-taller single
column) rather than one row per category. Nothing is ever truncated —
category names, player names, and club labels all word-wrap; `wrapCanvasText`
force-breaks at the character level for the rare single "word" (a long
hyphenated surname, say) that's wider than its column on its own, so no
line can ever overflow its box.

**The measure-then-draw pattern.** Both the per-card and whole-tournament
drawing functions take a `draw` boolean and run through the *exact same*
layout logic either way — called once with `draw: false` (nothing painted,
just wrapping/line-counting to get a height), then again with `draw: true`
at the real position once the canvas is correctly sized. This is what
makes it possible for content to be genuinely dynamic-height (wrap to
however many lines it needs) without measuring and drawing ever disagreeing
about where something lands — they're driven by identical code, not two
hand-synced implementations.

Fonts are explicitly `document.fonts.load()`-ed and awaited before the
first stroke — `ctx.font` silently falls back to a system font if the
family hasn't been requested by *some* DOM node yet, with no error to
catch, and this is the single most likely way a cold-load export comes out
wrong while the on-screen tab looks fine.

---
**Features:** [Awards tab usage](../features/match-control.md#awards)
**Spec:** `specs/awards-podium-tab-spec.md` in the site repo (full derivation rules, acceptance checklist).
