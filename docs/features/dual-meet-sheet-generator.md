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

## How to use it

1. Build the plan in the [Tournament Calculator](tournament-calculator.md)
   and switch the format to **Dual meet**.
2. Click **Copy plan & open generator**. This copies the plan to your
   clipboard and opens the master workbook's copy dialog.
3. Click **Make a copy**. Work in *your copy* — never the master.
4. In your copy, choose **SAGE → Generate event tabs** from the menu.
5. Paste the plan into the sidebar (Ctrl+V, or Cmd+V on Mac), add the venue
   name, and click **Generate**.

If you would rather work from the file than the clipboard, you can **drag the
calculator's exported `.csv` straight onto the big box**, or use the
sidebar's **Choose file** button — either one fills the same box, so you can
still read over the plan, or edit it, before generating. All three routes are
equivalent; a file is read on your own machine and never uploaded anywhere.

The first time you run it, Google will ask you to authorize the script. That
is expected — it is the workbook's own script asking for permission to write
to the workbook.

A generated workbook carries the live sync script already, so the **SAGE**
menu always has both features in it: **Generate event tabs**, and — once
[live sync is set up](preparing-an-event.md) — **Generate Scoresheets**,
**Sync now**, **Pause live sync** (or **Resume live sync**), and **Live sync
settings**. Before setup it's just **Set up live sync** instead. **Help** is
always there either way, with a workflow refresher, this workbook's current
status, and what to check when something looks wrong.

**Set up live sync**/**Live sync settings** is safe and repeatable to click
any time; so is **Generate Scoresheets**, which just opens the [Scoresheet
Generator](scoresheet-generator.md) in a new tab with this workbook's day and
venue already selected and makes no changes to the spreadsheet. **Generate
event tabs** is the opposite — it runs once per workbook and then disappears
from the menu, since a second run can only fail.

## What it does *not* do

**Player names.** The plan has no roster in it, so the name columns come out
empty for you to paste into. The generator says so when it finishes — that
is the one piece of hand work a generated workbook still needs before it can
run an event.

Treat the generated workbook as ready to run once rosters are in, not as a
finished, already-scored event.

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
