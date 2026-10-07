# The scoring workbook

**Used in:** [2. Build the workbooks](../usage/build-the-workbooks.md) ·
[5. Connect the workbooks](../usage/connect-the-workbooks.md) ·
[8. Run the day](../usage/run-the-day.md)

The Google Sheet where scores are typed and standings are computed. It is the
source of truth for an event: [Tournament Hub](tournament-hub.md),
[Control Center](control-center.md) and the [schedule board](schedule-board.md)
all show what the workbook says, a few seconds after it says it.

An event has **one workbook per venue per day**. Three venues across five days
is fifteen workbooks. A dual meet is one workbook. A team event is one workbook
(see [the team workbook](team-workbook.md)).

The [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) and the
[Standard Tournament Generator](standard-tournament-generator.md) build it from a
plan. This page covers what is in a built workbook, whichever generator built it.

## The tabs

Every format has these.

| Tab | What it is | Who touches it |
| --- | --- | --- |
| **Category tabs** (one per category, named by its key, e.g. `HIMD`) | The pairs, their win/loss/score formulas, the score grids and the playoff blocks. The roster goes into **`STEP 1 · NAMES`** and the draw into **`STEP 3 · RANDOMIZED CODES`**. | A person types the roster, or imports it. They leave the formulas alone. |
| **`SCHEDULE`** | Every match, numbered and placed on a court and a start time, with each category colour-coded. | A person types **scores** here. It is the tab every scoring formula reads results from. They leave the layout alone. |
| **`Court Control`** | Which match is on which court right now, with a pace-against-plan estimate. | The operator enters the match number against a court. This is what makes a match **live**. |
| **`CSV`** | One row per match: the tab the live sync publishes to the site. | Nobody. `Fill match numbers` writes its `matchNumber` column. |
| **`STANDINGSCSV`** | One row per pair's record: the second tab the live sync publishes. | Nobody. |
| **`Timeline`** | A pair-by-time-slot grid for spotting a pair double-booked. | Read only. A cell showing more than 1 is a double-booking. |
| **`ATTENDANCE`** | Staff check-in, empty until **Update roster** fills it. | The API keeps it. Use a filter view, not Sort, to rearrange it. See [event attendance](event-attendance.md). |
| **`Variables`, `Title`, `Reference for Players`** | The event name, date, courts and match duration, filled in from the plan. | Nobody. `Title` carries the event key. |

A standard tournament also has **`MATCHES`**: every category's matches laid out
side by side, from which `SCHEDULE` is packed by hand. See
[the Standard Tournament Generator](standard-tournament-generator.md#packing-the-schedule).

The sync reads exactly two tabs, `CSV` and `STANDINGSCSV`. A team event also
publishes its roster tab, `Teams`.

### Which edits are safe

- **Safe, and what the day is made of:** scores on `SCHEDULE`, match numbers in
  `Court Control`, player names in `STEP 1 · NAMES`.
- **Safe but deliberate:** pasting court blocks into `SCHEDULE` (standard
  tournaments only, with **Paste special → Formula only**).
- **Leave alone:** every formula, `CSV`, `STANDINGSCSV`, `Timeline`, the headings
  and the `Court Control` layout. A column renamed in `CSV` or `STANDINGSCSV`
  leaves the Hub empty.

## The SAGE menu

A generated workbook carries a **SAGE** menu. Which items it shows depends on
whether the workbook has been generated and whether live sync is set up. The
menu appears after the workbook is reloaded.

| Item | What it does |
| --- | --- |
| **Generate event tabs** | Builds the workbook from a plan pasted into the sidebar. **Runs once per workbook, then disappears from the menu.** A second run could only fail. |
| **Import bracket draws** | *Standard only.* Fills every category's roster and codes from the Bracket Generator's text exports. Stays in the menu so it can be run again. See [Filling the rosters from the draw](standard-tournament-generator.md#filling-the-rosters-from-the-draw). |
| **Shuffle roster codes** | *Dual meet only.* On a category tab, draws each club's `STEP 3 · RANDOMIZED CODES` independently from the codes in `STEP 2`. Refuses a tab with no roster, and asks before replacing codes that are there. Stays in the menu. See [Drawing the roster codes](dual-meet-sheet-generator.md#drawing-the-roster-codes). |
| **Open Bracket Generator** | On a category tab: opens the [Bracket Generator](bracket-generator.md) with the event name and that category already filled in. |
| **Fill match numbers** | Renumbers the matches on `SCHEDULE` from a number you give it: `0` starts at 1, `2000` starts at 2001. Only slots with a team on both sides count as matches, and empty slots show `-`. It writes the same numbers into the `CSV` tab's `matchNumber` column, which is what the website reads. It writes nothing if a slot has a team on one side only. Always present. |
| **Set up live sync** | Once per workbook: enter the **day key** and **venue (facility) name**. Setup checks both against the registry and runs a real test sync before saving. Replaced by **Live sync settings** afterwards. |
| **Live sync settings** | Shows the saved day key and venue and lets you change them. Safe to click any time. |
| **Sync now** | Sends the workbook to the site immediately and reports what it sent: the venue and day, whether open pages got it straight away, and how long it took. |
| **Pause live sync** / **Resume live sync** | Stops and restarts automatic publishing without touching the saved settings. For a large edit to a workbook that is already live. |
| **Generate Scoresheets** | Opens the [Scoresheet Generator](scoresheet-generator.md) in a new tab with this workbook's day and venue already selected. Makes no change to the workbook. |
| **Set shared secret** / **Replace shared secret** | Only on a master, and on a hand-made workbook that carries no secret. A copy of a master never shows it. |
| **Help** | Always present. A workflow refresher, this workbook's current status, what to check when something looks wrong, and the version of each SAGE script in the workbook. |

The first time any item runs, Google asks you to authorize the script. That is
expected: it is the workbook's own script asking permission to write to the
workbook.

### The run-once rule

**Generate event tabs** builds several tabs in place (`SCHEDULE`, `Court
Control`, `Timeline`, `CSV` and `STANDINGSCSV` each grow from the master's own
small starting shape), so a workbook can be generated exactly once. A run that
fails partway cannot be retried in the same workbook. Make a fresh copy of the
master and generate there. Work in your copy, never in the master.

### What a sync reports

A successful **Sync now** names the venue and day it synced, how the data was
published and how long it took. A failure reads **Sync failed** followed by the
reason, and **SAGE → Help** explains each one.
[9. When something's wrong](../usage/troubleshooting.md) lists the common ones.

---
**Features:** [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) · [Standard Tournament Generator](standard-tournament-generator.md) · [Team workbook](team-workbook.md)
**Technical:** [sync pipeline](../technical/sync-pipeline.md) · [Dual Meet Sheet Generator](../technical/dual-meet-sheet-generator.md) · [Standard Tournament Generator](../technical/standard-tournament-generator.md)
