# Control Center

One operator console (`tools/control-center.html`) covering every registered
event, replacing what used to be a separate `match-control.html` copy
per event — four copies at last count, ~13,700 lines total, each carrying
its own ~106-line hand-duplicated configuration block. The console carries
**no per-event configuration of its own**: it picks an event from
`event-data/config/events.json` (see [event registry
schema](event-data-config.md)), reads that event's registry entry, and
renders.

> Renamed from "Match Control" to "Control Center" after the fact — the
> file, the URL (`/tools/control-center`), and every reference were moved
> together; `tools/match-control.html` survives only as a redirect stub so
> an already-bookmarked link keeps working. `specs/match-control-console-spec.md`
> and `specs/awards-podium-tab-spec.md` in the site repo, both written
> before the rename, still use the old name throughout — treat them as a
> historical record, not out of date documentation.

## Config resolution

Selecting an event resolves three things fresh, every time:

- `CURRENT_EVENT_KEY` / `EVENTS_REGISTRY` entry
- `CURRENT_TYPE` — `"dual-meet"` or `"standard"`, **required and explicit**,
  never inferred from the data. Guessing from team-code shape works most of
  the time and fails silently; an unrecognized/missing `type` shows a
  visible configuration error instead.
- `DISPLAY` — the optional `{ divisions, events, clubs }` label maps. Absent
  is fine; the console just shows raw codes (`LIWD`, `PNF`) instead of
  friendly labels.

Everything else the console needs maps onto that: the category set is the
second code segment (`DIVISIONS` × `EVENTS`), stage (RR/QF/SF/Bronze/Final)
comes from generic regexes in `STAGE_META` — not per-event — and the day
label comes straight from the published snapshot's own `label` field.

**Mission Control is event-agnostic.** Its dependency surface is just
`EVENT_KEY` — it touches none of `DIVISIONS`, `EVENTS`, `CLUBS`,
`STAGE_META`, `parseCode`, or `divisionLabel`. That's what makes it safe to
share unmodified across every event type.

## The tabs

**Live Matches, Match Finder, and Standings** have no club logos anywhere
(no `.score-logo`/`.live-team-logo`/`.cs-logo` treatment); the console
**ignores `isLive` and always shows live data** (`computeDayIsLive()` is
hardcoded `true`), so an operator can preview a day before it's public;
and labels degrade to raw codes when `display` config is absent rather
than erroring.

**Standings layout is chosen by `type`.** `dual-meet` gets per-club columns
inside each category with a shared club win-total summary bar and the
cross-club Bronze/Final shown as one block spanning both clubs (see
`CROSS_CLUB_STAGE_KEYS`); `standard` gets flat category cards with a
desktop toggle bar and a mobile category-search filter.

**Awards** is a fifth tab, added later — full derivation and export
architecture in its own section below.

**Mission Control** — see [Mission Control usage](../features/control-center.md#mission-control)
for what it does; scoped to whichever event is selected. The go-live states are `auto`
(live 4 hours before the day's earliest scheduled match time, computed live
from the synced `Schedule` column — no hour to configure), `true` (force
live), and `false` (Live/Standings hidden, scores suppressed on Tournament
Hub only — this console's own tabs stay live regardless, so an operator can
verify a fix before un-hiding). The connection-check line also surfaces the
currently-cached sync config's short SHA (see [sync
pipeline](sync-pipeline.md) § Observability), so an operator can see at a
glance whether a just-committed `events.json` change has actually taken
effect yet.

Each facility row in the Facility Sync Status list links directly to that
facility's actual Google Sheet (`docs.google.com/spreadsheets/d/<sheetId>/edit`).
`FACILITIES` (resolved per day-select from `EVENTS_REGISTRY`) carries
`sheetId` alongside `name` for exactly this — previously dropped, since
nothing needed it. No new exposure: sheet IDs are already public in the
same `config/events.json` fetch the console already makes (see [event
registry schema](event-data-config.md) § Sheet IDs are effectively
public). The link is omitted for a facility with no `sheetId` yet, the
same "not set up" condition the sync pipeline itself skips.

## Installability

Installable via its own manifest, `tools/control-center.webmanifest`, the
same pattern [Tournament Calculator](tournament-calculator.md) established —
`scope` is the page path (`/tools/control-center.html`), not the `/tools/`
directory, so installing this doesn't sweep in `scoresheet-generator.html`
or the calculator itself. Icons and theme/background colors reuse the
shared site assets and palette, same as the calculator's manifest.

**Deliberately no service worker.** Unlike the calculator (whose whole
justification for offline support is that nothing it shows can go stale),
Control Center's entire value is live data — scores, sync status, court
assignments — so caching any of it risks showing an operator something
stale during a live event, exactly the failure mode the calculator's own
spec ruled out for pages like this. The manifest alone is enough for
"Add to Home Screen" / an install prompt and a standalone window; it adds
no caching and changes no runtime behavior. An installed shortcut still
needs a live connection to do anything, identical to a regular tab.

## Theme

The console is a **tool** — it lives in `tools/` beside
`scoresheet-generator.html` and `tournament-calculator.html`, which are
where the S.A.G.E. palette (navy structure, green accent, off-white paper;
Archivo Black / Barlow Condensed / Inter) originated, and it uses that
palette directly rather than porting a converted theme from the old
per-event pages. Two consequences:

- **It does not take on per-event theming.** An event may re-skin its own
  Tournament Hub, but the console looks identical regardless of which event
  is selected — it's one tool pointed at different data, and an operator
  switching events shouldn't see the furniture move.
- Contrast rules worth knowing if touching the CSS: **green is a fill
  colour, not a text colour** (`--green` on white is ~2.3:1, fails AA at any
  size — use it as a background with navy text on top, or on a navy panel).
  The masthead is a navy bezel on paper; controls sitting on it need their
  own light-on-navy overrides rather than reusing the paper-ground base
  rules.

## Awards tab

A fifth tab showing each category's podium and exporting it as
SAGE-branded PNGs. Entirely **read-only and derived** — it writes nothing,
needs no auth, and adds no new sync surface or `sage-tools-api` dependency.
Everything it shows comes from `MATCHES` (the day's parsed match rows) and
`STANDINGS`, already loaded by the console for the other tabs.

### Podium derivation

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

#### Twice-to-beat resolution

A bracket can run a Final as `F(1)`/`F(2)` — the second match only happens
if the first goes the challenger's way. `matchInstanceOf()` reads the
trailing `(1)`/`(2)` off the team code. The decisive-match search always
prefers the **highest resolved instance**: if only `F(1)` is played, its
result stands; once `F(2)` is also played, it supersedes `F(1)`'s result
entirely.

#### Byes, and why they're detected the way they are

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

#### Byes must not surface where a match looks playable

A bye is never a real match, so it's filtered out of every surface that
presents match data as something to watch or play, not just the podium:

| Surface | What's filtered |
| --- | --- |
| Standings | Any row whose team code/player1 is `BYE`; if a category's *decisive* Bronze was bye-decided, the whole Bronze block is dropped (not just that row) — a one-sided "Bronze Battle" showing one team with no opponent is worse than showing nothing |
| Live Matches | Bye matches excluded from court grouping entirely |
| Match Finder | A team's own bye match never appears in their schedule, and can never be flagged "Next Up" |
| Team index / autocomplete | `BYE` is never indexed as a searchable name |

### Overall Champion (dual-meet only)

Deliberately **the same metric as Standings' own club win-total
summary** — total wins across Round Robin rows only, per club — not a
podium-derived count (e.g. gold medals), so the two tabs can never disagree
about who's ahead. A tie is shown explicitly as "Tied," never resolved by an
invented tiebreak.

### Image export — Canvas 2D, no library

`tools/control-center.html` has zero JS dependencies (Google Fonts is the
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
**Features:** [Control Center](../features/control-center.md) · [Awards tab usage](../features/control-center.md#awards)
**Specs:** `specs/match-control-console-spec.md` and `specs/awards-podium-tab-spec.md` in the site repo (full build history, acceptance checklists — both written under the console's old name).
