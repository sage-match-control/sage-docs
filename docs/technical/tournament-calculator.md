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

## Scheduling floors

Court capacity is not the only limit on how fast a phase can run — a pair
can't be on two courts at once. `seqParts()` returns the per-category slot
floor as the **max** of the capacity estimate (`ceil(matches / courts)`) and
the dependency floor, never the capacity estimate alone:

- **Round robin** (`rrRoundFloor()`) — a bracket of `n` needs `n-1` rounds
  when `n` is even and `n` when it's odd (the bye rotates). Three pairs in
  one bracket is 3 matches that all share a pair, so it needs 3 slots even
  on 4 free courts, not `ceil(3/4) = 1`.
- **Dual brackets** are a cross-product rather than a round robin, so the
  floor is the *larger* club's pair count: a 2v5 bracket is 10 matches, but
  the two-pair club plays 5 each, so 5 rounds. Taking `max(a, b)` rather
  than the bracket's combined size is what makes the asymmetric case right.
- **Brackets interleave**, so a category's floor is its biggest bracket's,
  not the sum — separate brackets share no pairs and can run concurrently.

The same `max` shape already governs playoff rounds and best-of-3/
twice-to-beat series, and `calcAll()` folds every category's floor into the
day's total via `maxSeq`. The practical consequence is that a small bracket
leaves courts idle, and the projected finish now says so.

## Playoff round naming

Round names are **positional and stay positional**: the last round is the
Final, and walking backward the structural size doubles — SF, QF, R16, R32.
A four-round ladder is `R16 · QF · SF · Final` *however few pairs are
actually in the early rounds*, so a tiered ladder can legitimately show
"Round of 16: 1 match".

This looks wrong and isn't. These names are stage identifiers, not
descriptions of field size: the standard-tournament workbook keys playoff
slots as `<KEY>_<STAGE>_<k>` with `STAGE ∈ {R32, R16, QF, SF}`, its
`STAGE_ORDER` is `['R16','QF','SF','BRONZE','FINAL']`, and Control Center
and the schedule board parse the same vocabulary. Renaming the sparse early
rounds to anything else breaks all three.
`standard-tournament-master-spec.md` §1.1 fixes this contract against a
worked example: `LI18MD` (15 pairs, 3 brackets, top 2) reduces as
**1 + 1 + 2 + 1** matches, and the workbook holds exactly two `R16` slots,
two `QF`, four `SF`, two `B` and two `F` — which is what the calculator
emits. Treat that example as the regression test for any change here.

`plan.prelims` and `plan.prelimSpots` count the rounds the lower tiers play
before the top tier enters. They drive `legendTxt()` and the chart's
`freshIndex` (which round the group winners enter at), never the labels. The
legend previously inferred "are there extra rounds?" from `plan.byes`, which
is a bye count rather than a round count, and which wildcards zero out — so
it claimed "no preliminary rounds needed" directly above a chart showing two.

## The two fill modes are different shapes, not just different counts

`bye` keeps the tiered ladder. `wc` fills the draw out to a complete bracket
— `2^ceil(log2(direct)) - direct` wildcards — so every qualifier starts in
the same round and no one byes. Measuring wildcards against the *bracket*
rather than against the ladder's short play-in round is the point: 3 brackets
× top 2 is 6 qualifiers in a draw of 8, so 2 wildcards, and the ladder
collapses from four rounds (`R16 · QF · SF · Final`) to three (`QF · SF ·
Final`). See `standard-tournament-master-spec.md` §5.4.

**The toggle's label and visibility must not depend on which fill is
active.** `plan.wcFill` (what choosing wildcards *would* add) and
`plan.tieredByes` (what the tiered ladder holds) are therefore computed the
same way in both modes — the ladder is built even in `wc` mode purely for
that count. Deriving either from the live plan is how the button came to
read "+1 wildcard" and then deliver 2, and how it could vanish mid-click:
at 2 brackets × top 4 the active `wc` plan has no byes and no wildcards, so
a live-plan test hid the control and stranded the operator in `wc` mode.
The two counts are independent — either can be zero while the other isn't
(2 brackets × top 3 has no ladder byes but 2 wildcards to add; 2 brackets ×
top 4 is the reverse) — so the control is offered when *either* is non-zero,
which is exactly when the two fills produce different shapes.

Consequences worth knowing before changing this: the two fills can differ in
round count, so anything sizing a tab from the round list must read the plan
rather than assume. `advance == 1` is unaffected — a flat bracket's
first-round byes already equal `2^ceil(log2(groups)) - groups` — so only
tiered categories move. And filling a large field is not free: 5 brackets ×
top 2 is 10 qualifiers, which fills to 16, pulling in 6 wildcards and taking
the playoff from 10 matches to 16.

`buildChart()` seeds the earliest round from `groups × (adv-1)` **plus
`plan.wc`** for the same reason: a wildcard is an extra body in the
lower-ranked pool, and omitting it left that round one occupant short of its
own slots, rendering a literal `undefined` into the draw.

## Standard mode: single-bracket format

A one-bracket category carries `soloFormat` — `'ttb'` (twice-to-beat final,
the default) or `'rr'` (no playoff, standings decide every medal).
`playoffPlan()` reads it only when `groups === 1`, and the `'rr'` plan is an
empty `rounds` array with `rrOnly: true` rather than a zero-match round, so
every consumer that maps over `rounds` produces nothing instead of a phantom
line item.

- **`rrOnly` is a separate flag from `single`, not a replacement.** Both
  single-bracket shapes still set `single: true` — that is what keeps the
  format toggle rendered in both states, so `'rr'` is reversible from the
  UI rather than a one-way door. Anything that means *twice-to-beat
  specifically* (the "counted at full length" footnote, the playoff chart)
  tests `single && !rrOnly`.
- **`soloFormat` is its own field, not folded into `poFormat`.** The two
  apply to different formats (`poFormat` is dual-only, 2+ brackets;
  `soloFormat` is standard-only, exactly 1 bracket) and the format toggle is
  global, so sharing a field would let a dual-meet setting silently rewrite
  a standard-mode one — the same losslessness argument as `group` vs.
  `groupDual` above.
- **A 2-team category is unaffected.** `calcCategory()` short-circuits to
  the best-of-3 series before bracket math runs, so no toggle appears there.

The CSV gains a trailing `solo_format` column, written only for standard
categories that actually resolve to one bracket. `sheet-generator.gs` parses
the plan CSV positionally and is dual-only, so a trailing column is inert
there; an older 11-column export imports with `soloFormat` defaulting to
`'ttb'`, preserving what those files meant when they were written.

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

## Sheet generator handoff

A box under the export buttons holds one button that copies the plan CSV to
the clipboard and opens a master workbook's `/copy` URL, so the operator
lands in Google's "Make a copy" dialog and pastes into that master's
generator sidebar. The master follows the format: the Dual Meet Master for
the [Dual Meet Sheet Generator](dual-meet-sheet-generator.md), the Standard
Tournament Master for the
[Standard Tournament Generator](standard-tournament-generator.md).
`GENERATOR_MASTERS` holds each one's file ID and help text, and
`toggleDualClubsVisibility()` swaps the link and text when the format
changes.

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

Both masters' file IDs are hard-coded in `GENERATOR_MASTERS`. A Drive file
ID is stable across folder moves, so only replacing a workbook itself breaks
its link.

---
**Features:** [Tournament Calculator usage](../features/tournament-calculator.md)
**Specs:** [`calculator-dual-meet-spec.md`](../specs/implemented/calculator-dual-meet-spec.md) and [`calculator-pwa-spec.md`](../specs/implemented/calculator-pwa-spec.md).
