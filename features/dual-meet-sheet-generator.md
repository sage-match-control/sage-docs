# Dual Meet Sheet Generator

Builds a dual meet's event workbook — every category tab, ready to score —
from a plan you exported out of the Tournament Calculator. It replaces the
old routine of duplicating last event's spreadsheet and find-replacing every
team code by hand.

**Dual meets only.** A standard-tournament version is planned; for now the
generator rejects a standard-format plan rather than approximating one.

## What it does for you

Given a plan with seven categories, it creates seven tabs, each with:

- Both clubs' pair blocks, with the win/loss/score/quotient formulas already
  wired to the scoring functions.
- The head-to-head score grid for each block.
- The Bronze and Final blocks, with formulas that fill in the finalists
  automatically once the group stage finishes.
- The name-entry scaffold — the numbered STEP 1 / 2 / 3 columns where you
  paste your roster.

It also fills in the `Variables`, `Title`, and `Reference for Players` tabs
from the same plan, so the event name, date, courts, and match duration are
all consistent without being typed three times.

## How to use it

1. Build the plan in the [Tournament Calculator](tournament-calculator.md)
   and switch the format to **Dual meet**.
2. Click **Copy plan & open generator**. This copies the plan to your
   clipboard and opens the master workbook's copy dialog.
3. Click **Make a copy**. Work in *your copy* — never the master.
4. In your copy, choose **SAGE → Generate event tabs** from the menu.
5. Paste the plan into the sidebar (Ctrl+V, or Cmd+V on Mac), add the venue
   name, and click **Generate**.

The first time you run it, Google will ask you to authorize the script. That
is expected — it is the workbook's own script asking for permission to write
to the workbook.

## What it does *not* do

**A generated workbook is not ready to run an event yet.** Two things are
still hand work:

- **Player names.** The plan has no roster in it, so the name columns come
  out empty for you to paste into. The generator says so when it finishes.
- **The schedule.** The `SCHEDULE` tab — and the `CSV`, `STANDINGSCSV`, and
  `Court Control` tabs that read from it — are not generated. Until you build
  those, they still hold whatever event the master workbook was copied from,
  so scores will not compute and the live sync will publish the wrong thing.

Treat the generated workbook as a well-formed starting point, not a finished
one.

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
