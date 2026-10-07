# Dual Meet Sheet Generator

**Used in:** [2. Build the workbooks](../usage/build-the-workbooks.md) ·
[3. Draw and fill rosters](../usage/draw-and-rosters.md)

Builds a dual meet's event workbook — every category tab, ready to score —
from a plan you exported out of the Tournament Calculator. There is no
spreadsheet to duplicate and no team code to find-replace by hand.

**Dual meets only.** A standard-format plan is rejected rather than
approximated; standard tournaments have their own
[Standard Tournament Generator](standard-tournament-generator.md).

## What it does for you

Given a plan with seven categories, it creates seven tabs, each with:

- Both clubs' pair blocks, with the win/loss/score/quotient formulas already
  wired to the scoring functions.
- The head-to-head score grid for each block.
- The Bronze and Final blocks, with formulas that fill in the finalists
  automatically once the group stage finishes.
- The name-entry scaffold — the numbered STEP 1 / 2 / 3 columns where you
  paste your roster.

It also builds the **`SCHEDULE`** tab — every match, numbered and placed on a
court and a start time, with each category colour-coded. That is the tab you
type scores into during the event, and the one every scoring formula reads
match results back from.

And it fills in the `Variables`, `Title`, and `Reference for Players` tabs
from the same plan, so the event name, date, courts, and match duration are
all consistent without being typed three times.

Finally, it builds the four **readout tabs**: `Court Control` (the live
who's-on-which-court board, with a pace-against-plan estimate), `Timeline`
(a pair-by-time-slot grid for spotting a pair double-booked before the event
starts), and `CSV`/`STANDINGSCSV` (the two tabs the live sync publishes to
the public event page — one row per match, one row per pair's record).
These are what make a generated workbook able to run an event end to end,
not just look like one.

It also adds an empty **ATTENDANCE** tab, ready for
[event attendance](event-attendance.md): pressing **Update roster** in Control
Center fills it in.

When it finishes, it renames the workbook itself to the plan's date and
title followed by the venue label in capitals, e.g.
`2026-09-12 PNF x BUP Dual Meet - PPC`. With no venue label the dash part is
left off. A run that fails partway leaves the name as it was.

## The sidebar

The generator runs from **SAGE → Generate event tabs** in a copy of the Dual Meet
Master, with the plan pasted into the sidebar. The steps, in order, are in
[Usage step 2](../usage/build-the-workbooks.md). The rest of the SAGE menu is on
[the scoring workbook](scoring-workbook.md#the-sage-menu).

While it runs, a progress bar under the button shows how far it has got
("Step 12 of 31", with the time so far), and the box under it lists what it is
building, each line stamped with the time since the start. A full-size meet
takes a few minutes; keep the sidebar open until it says **Done**. If a run
stops partway, the bar turns red and the sidebar says why: the tabs it finished
stay as they are, and the copy cannot be generated again, so make a fresh copy
of the master and generate there. The sidebar's foot shows the generator's
version.

The plan can be pasted (Ctrl+V, or Cmd+V on Mac), or the calculator's exported
`.csv` can be **dragged straight onto the big box** or chosen with the sidebar's
**Choose file** button. All three routes fill the same box, so you can still read
over the plan, or edit it, before generating. A file is read on your own machine
and never uploaded anywhere.

## Drawing the roster codes

A dual meet's blind is a per-club roster draw rather than a bracket draw, so
there is no draw file to import: `STEP 3 · RANDOMIZED CODES` is drawn in the
sheet.

Paste each club's players into its `STEP 1 · NAMES` column, then choose
**SAGE → Shuffle roster codes** on that tab. It reads the codes from
`STEP 2` and writes a shuffled order into `STEP 3`, drawing each club's
column independently — the two clubs are separate rosters and never mix.

`STEP 3` ships blank on purpose. Filling it in order is not a no-op: it maps
every pair to its own roster slot, which quietly undoes the blinding the
shuffle exists to provide.

The item refuses a tab that has no roster on it, and asks before replacing
codes that are already there, since reshuffling re-points every pair on the
tab. It stays in the menu after the workbook is generated, because that is
when the roster arrives.

## What it does *not* do

**Player names.** The plan has no roster in it, so the name columns come out
empty for you to paste into. The generator says so when it finishes — that
is the one piece of hand work a generated workbook still needs before it can
run an event.

Treat the generated workbook as ready to run once rosters are in and their
codes are drawn, not as a finished, already-scored event.

## If it refuses to run

The generator checks the whole plan before writing anything, and if something
is wrong it lists *every* problem at once and leaves the workbook untouched.
Common ones:

- **The plan isn't a dual meet.** Switch the calculator's format to Dual meet
  and re-export.
- **Uneven clubs.** Both clubs need the same number of pairs in a category.
- **A tab already exists** with that category's name. The generator will not
  overwrite or rename anything — delete or rename the old tabs yourself
  first. This is deliberate: a half-replaced set of tabs partway through an
  event is far worse than being told to clean up first.
- **`SCHEDULE` or a readout tab has already been built.** Unlike the category
  tabs, `SCHEDULE`, `Court Control`, `Timeline`, `CSV`, and `STANDINGSCSV`
  are each grown in place from the master's own small starting shape, so
  every one of them can only be built once. If you have already generated
  into this workbook — or a run failed partway through — start again from a
  fresh copy of the master. The generator will not try to reset any of the
  five, for the same reason as above.

## Playoff shapes

For a category split into two brackets, there are two ways the club's two
bracket winners settle who goes to the final:

- **Club semis** — the two winners play each other. Costs one extra match per
  club, and the generator lays out the semi blocks for it.
- **Straight to final by record** — no extra match. The winner with the
  better record (most wins, then best quotient) takes the final slot, the
  other takes bronze.

You choose this in the calculator; the generator follows whichever the plan
specifies.

---
**Technical:** [how the generator is built](../technical/dual-meet-sheet-generator.md)
