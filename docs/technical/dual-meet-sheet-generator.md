# Dual Meet Sheet Generator

`sage-tools-api/scripts/sheet-generator.gs` — bound Apps Script that builds a
dual meet event's whole workbook from a Tournament Calculator CSV: every
category tab, `SCHEDULE`, and the four readout tabs the live sync and the
operator both depend on. Implements
[`dual-meet-sheet-generator-spec.md`](../specs/dual-meet-sheet-generator-spec.md)
Phase 1 (category tabs),
[`dual-meet-schedule-generator-spec.md`](../specs/dual-meet-schedule-generator-spec.md)
Phase 2 (`SCHEDULE`), and
[`dual-meet-readouts-generator-spec.md`](../specs/dual-meet-readouts-generator-spec.md)
Phase 3 (`Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`) — all three
built and in use.

**It lives in the API repo but is not part of the API.** It runs inside a
Google Sheet, talks to no server, and ships by being pasted into the
workbook's own script project. Changing it is not a deploy and does not bump
`package.json` — nothing about what Cloud Run is running has changed. Same
arrangement as `sheets-sync.gs`.

## Why bound Apps Script and not a Cloud Run endpoint

The category tabs (and, since Phase 3, the readout tabs too) don't run on
native Sheets formulas. They run on a library of Named Functions —
`GETTOTALWINS`, `SORTBYWINS`, `GETSCOREAGAINSTPAIR`, `STACKBLOCKS`,
`MATCHTIME`, and friends — that exist only inside the workbook and are in no
repo. They're workbook-level `LAMBDA` definitions under
`Data -> Named functions`, not container-bound Apps Script custom functions
(an easy thing to assume, since the generator itself *is* bound Apps
Script — but the two are unrelated features living in the same workbook).
Generating a *blank* spreadsheet would produce tabs full of `#NAME?`.
The library itself — every definition, its dependency graph, and the traps
that come with editing it — is written up in
[The Named Function library](named-function-library.md).

So generation has to start from a copy of a master workbook that already
carries those functions. That single constraint rules out the Sheets API
approach, which would additionally have needed a service account, Drive
scopes, file-ownership and sharing handling, and a bet on `files.copy`
preserving a bound script project. The bound-script route needs no new
infrastructure and no new credentials.

## The template contract

The master workbook holds a hidden `_CATEGORY_TEMPLATE` tab that is
**exactly the `n=1` instance of the generator's own layout function** —
`computeLayout(1, 1, 1, false)`. One group pair per club, one pair-vs-pair
per playoff block.

This is the load-bearing coupling in the whole design. The generator reads
prototype cells at fixed positions in that template and copies their
formatting onto every block it lays down, which is what carries the cell
borders and each club's colours. If the layout constants change, the template
has to be rebuilt to match — the file's header comment spells out the exact
row numbers so the code and the template can't silently drift.

Formatting is *copied from named prototype cells*, not set property by
property. Copying a whole cell format carries fill, borders, font weight and
font colour together; setting properties individually silently drops the cell
borders that separate every square of the score grids.

## Shape of a generated tab

Row geometry is computed, never hardcoded — everything scales with `n`, the
pairs per club per bracket:

- **Group blocks**, `2n` rows each (two rows per pair), one per club per
  bracket. At two brackets, blocks are grouped **by club** — club A's bracket
  1 and 2 back to back, then club B's.
- **Score grids** beside each block, `n × n`, condensing each pair's two rows
  into one grid row.
- **Bronze and Final** blocks, each two entrants, with their own smaller
  self-referential score grid.
- **Semi blocks**, only under `po_format: semis`.
- **The roster scaffold**, columns `AA:AG` for club A and `AP:AV` for club B —
  both on the *same rows*, beside club A's bracket-1 block.

That last point is why blocks are grouped by club. The roster lists all of a
club's pairs, which needs more vertical room than any single bracket block
has. Grouping a club's brackets together lets the roster span that club's own
blocks, so no block ever needs padding rows to make room — every block is
exactly `2n` rows. Interleaving the clubs bracket-by-bracket instead would
require padding club A's first block, leaving a visible run of blank rows
between the two clubs.

## Playoff feeders

The formula that fills in who reaches the final, gated on the group stage
being complete:

```
=IF(SUM($J$4:$J$11)=16,INDEX(SORTBYWINS($A$4:$H$11),1),"-")
```

`16` is `n²` computed from the pair count, not a constant — every pair meets
all `n` of the opposing club's pairs, so a complete block is `n²` matches.
Column `J` is matches-played. The feeder sits on the entrant's *second* row
with the first row mirroring it (`I26` = `=I27`), which is why column `I` is
left unmerged while `D:H` are merged.

### Two brackets, "straight to final by record"

Under `po_format: aggregate` there are no semi matches at all. Each club's two
bracket winners are compared directly — most wins, then best quotient. This
needed a different shape because `SORTBYWINS` ranks one contiguous `A:H`
block and no single range spans a club's two brackets, so it's an explicit
two-way comparison with `LET` binding each winner once.

Two traps worth knowing if you touch it:

- **`LET` names cannot look like cell references.** `w1`/`w2` are read as
  cells W1/W2 and rejected outright. The names are `br1_win`/`br2_win` —
  underscores can't appear in A1 notation.
- Everything is gated on `usesSemis_()`, which mirrors the calculator's own
  rule: at 2+ brackets anything that isn't `aggregate` means `semis`; below
  two brackets `po_format` is inert and must not be read at all.

## Failure behaviour

Validation runs to completion and reports every problem at once, before
anything is written. A name collision with an existing tab aborts rather than
overwriting or renaming — a half-replaced set of tabs mid-event is the worst
available outcome.

If something fails *after* writing starts, there's no rollback: the error
names exactly which tabs made it in. Guessing past an error would produce a
workbook that looks complete and isn't.

## Verifying a change before it reaches a workbook

```bash
node scripts/verify-sheet-generator.mjs
```

Runs the real `generateEventTabs` against a mocked Sheets API and checks the
result against the specs' worked examples — four scenarios: the readouts
spec's §10 reference event, a two-bracket `semis`/`aggregate` mix, a 2-court
event, and the rerun refusal.

This is the one place the repo's "no tests" convention bends, and the reason
is the in-place rebuild. `SCHEDULE` and the four readout tabs are grown from
prototypes that only exist once, so the build is deliberately non-idempotent:
a run that dies halfway cannot be retried, it costs a fresh copy of the
master workbook. That makes the normal loop — paste into Apps Script, copy
the master, generate, read the execution log — expensive enough that finding
a bug in Node first is worth the mock.

The mock is faithful about the things that have actually caused bugs: range
bounds, sheet growth, `getLastRow`/`getLastColumn`, and `copyTo` repeating a
smaller source to fill a larger destination. It does **not** evaluate
formulas — a formula's *text* is what gets asserted — so anything that
depends on Sheets actually calculating (Named Functions resolving, spill
heights, `COUNTPAIRAT` matching a slot time) still has to be confirmed in a
real workbook.

## Tracing

Every run writes its full computed geometry, each grid's anchor, and each
generated feeder formula to the Apps Script execution log via `Logger`,
prefixed `[sage]`. Find it under **Extensions → Apps Script → Executions**.

This exists because layout bugs here are invisible in the finished artifact —
a block one row off looks like a perfectly normal spreadsheet unless you
already know which row it should have been on. Every bug found while building
this was of that kind.

## The `onOpen` it shares with `sheets-sync.gs`

The Dual Meet Master carries `sheets-sync.gs` alongside the generator, so a
copy of the Master is ready for both features from the start — no code
pasting needed for either, and no shared secret to enter: the Master carries
that too, in developer metadata rather than Script Properties precisely
because properties don't survive a copy. See
[sync pipeline](sync-pipeline.md#how-the-secret-reaches-a-workbook).

Both files declare a **byte-identical** `onOpen`
that delegates to feature-detected builders, `addGeneratorMenuItems_` here
and `addSyncMenuItems_` in `sheets-sync.gs`, each contributing only its own
menu items and its own leading separator. Two `onOpen` declarations in one
Apps Script project don't error — the last-loaded file's silently wins, and
load order isn't controllable — but which one wins can't matter here, since
both bodies are the same text. Change this file's `onOpen`, change
`sheets-sync.gs`'s to match — see
`sage-docs/docs/specs/sync-script-configuration-spec.md` §7 for the full
contract.

`Generate event tabs` appears only in a workbook that hasn't been generated
yet: `addGeneratorMenuItems_` checks the `TABS_GENERATED_FOR` Script
Property that `generateEventTabs` writes at the end of a successful run
(stamped with the spreadsheet's ID, so an inherited property from a copy
doesn't suppress the item in a workbook that genuinely needs it) and
contributes nothing once it matches. This is menu hygiene, not a safety
mechanism — `REBUILT_IN_PLACE_TABS`' pristine-shape check is what actually
refuses a second run — but it stops a live event workbook from offering an
action that can only produce an error.

For the sync half — the setup dialog, its validation, and the
spreadsheet-ID guard that keeps a copied workbook from inheriting another
facility's day key — see [Sync pipeline](sync-pipeline.md).

## What is not generated

Player names. Every other tab a generated workbook needs — category tabs,
`SCHEDULE`, `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`, `Variables`,
`Title`, `Reference for Players` — is built and wired together; the operator
pastes rosters into each category tab's name-entry columns and the workbook
is ready to run.

The calculator's plan CSV carries match *counts* only; `buildMatchList` does
the cross-product pairing and slot placement itself. See
`dual-meet-schedule-generator-spec.md` §4.

Also unexercised: the two-bracket **`semis`** path has been verified by
computation but never by generating a real tab.

## Reading the spec

[`dual-meet-sheet-generator-spec.md`](../specs/dual-meet-sheet-generator-spec.md)
is the design document, but it was written before the workbook was read
closely, and **§13 records where the two disagree** — the CODES column's row
count, the roster's position, the feeder formulas, and an entire undocumented
score grid on the playoff blocks. Read §13 before trusting a cell reference
anywhere else in it.

---
**Features:** [what the generator does for you](../features/dual-meet-sheet-generator.md)
