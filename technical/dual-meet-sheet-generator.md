# Dual Meet Sheet Generator

`sage-tools-api/scripts/sheet-generator.gs` — bound Apps Script that builds a
dual meet's category tabs from a Tournament Calculator CSV. Implements
`specs/dual-meet-sheet-generator-spec.md` Phase 1.

**It lives in the API repo but is not part of the API.** It runs inside a
Google Sheet, talks to no server, and ships by being pasted into the
workbook's own script project. Changing it is not a deploy and does not bump
`package.json` — nothing about what Cloud Run is running has changed. Same
arrangement as `sheets-sync.gs`.

## Why bound Apps Script and not a Cloud Run endpoint

The category tabs don't run on native Sheets formulas. They run on
container-bound custom functions — `GETTOTALWINS`, `SORTBYWINS`,
`GETSCOREAGAINSTPAIR`, and friends — that exist only inside the workbook and
are in no repo. Generating a *blank* spreadsheet would produce tabs full of
`#NAME?`.

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

## Tracing

Every run writes its full computed geometry, each grid's anchor, and each
generated feeder formula to the Apps Script execution log via `Logger`,
prefixed `[sage]`. Find it under **Extensions → Apps Script → Executions**.

This exists because layout bugs here are invisible in the finished artifact —
a block one row off looks like a perfectly normal spreadsheet unless you
already know which row it should have been on. Every bug found while building
this was of that kind.

## Not built

`SCHEDULE` (Phase 2) and the `CSV` / `STANDINGSCSV` / `Court Control` tabs
that depend on it (Phase 3). Phase 2 is blocked on input the calculator
doesn't currently export — it computes match *counts*, not a match list — so
it needs either the pairing/court/slot assignment ported into the generator,
or the calculator taught to export the schedule. That fork should be settled
before Phase 2 starts.

Until then a generated workbook's `SCHEDULE` still holds whatever event the
master was copied from, so it is not ready to run.

Also unexercised: the two-bracket **`semis`** path has been verified by
computation but never by generating a real tab.

## Reading the spec

`specs/dual-meet-sheet-generator-spec.md` is the design document, but it was
written before the workbook was read closely, and **§13 records where the two
disagree** — the CODES column's row count, the roster's position, the feeder
formulas, and an entire undocumented score grid on the playoff blocks. Read
§13 before trusting a cell reference anywhere else in it.

---
**Features:** [what the generator does for you](../features/dual-meet-sheet-generator.md)
