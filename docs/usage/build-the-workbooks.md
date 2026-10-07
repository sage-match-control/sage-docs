# 2. Build the workbooks

**Who:** organizer · **When:** 1–2 weeks before · **You need:** the plan from
[step 1](plan-the-event.md), with **Copy plan & open generator** already clicked
(or, for a team event, the PickleDrive workbook); Google access to the masters
and to the shared drive folder **SAGE → 1. TOURNAMENTS**

The scoring workbook is where scores are typed and standings computed. An event
has **one workbook per venue per day**: a dual meet is one, a standard tournament
is one for every venue on every day, and a team event is one. This step builds
them, files them where the system can reach them, and numbers their matches.

Two rules apply to every generator:

- **Work in your copy, never the master.** A generator builds several tabs in
  place, so it can run **once per workbook**. If a run fails partway, or you want
  to change the plan, start again from a fresh copy of the master.
- **The names come in step 3, not now.** The plan has no roster. The generator
  leaves the name columns blank and tells you so when it finishes.

## Steps

### Dual meet

1. In the tab that **Copy plan & open generator** opened, click **Make a copy**.
2. In your copy, choose **SAGE → Generate event tabs**.
3. Paste the plan into the sidebar (Ctrl+V, or Cmd+V on Mac). You can instead
   drag the calculator's `.csv` onto the big box, or use **Choose file**.
4. Add the **venue name** and click **Generate**.
5. Keep the sidebar open until it says **Done**. A full-size meet takes a few
   minutes. The first time, Google asks you to authorize the script. That is
   expected.

You get every category tab, `SCHEDULE` with every match placed on a court and a
start time, the readout tabs, and an empty `ATTENDANCE` tab. The workbook renames
itself to the plan's date, title and venue, for example
`2026-09-12 PNF x BUP Dual Meet - PPC`.

### Standard tournament

Do this once for **each venue on each day**.

1. Click **Make a copy** of the Standard Tournament Master.
2. In your copy, choose **SAGE → Generate event tabs** and paste the plan.
3. **Tick the categories this venue runs.** If you exported one plan per venue,
   **Select all** ticks the lot.
4. Enter the venue's **court labels** in schedule order: `1,2,3,4` at one venue,
   `5,6,7,8,9` at the next. Their number is the venue's court count, and they are
   the court numbers the website shows.
5. Set each category's **wave**. Wave 2 plays after all of wave 1 is finished.
   Mixed doubles defaults to wave 2, so a player entered in both a same-gender and
   a mixed category is never due on two courts at once.
6. Click **Generate**, and keep the sidebar open until **Done**.

You get every category tab with its playoff ladder, a `MATCHES` tab holding every
match, an **empty** `SCHEDULE`, the readout tabs and an empty `ATTENDANCE` tab.
The workbook renames itself, for example
`2026-09-27 Pickle For Sight Tournament - PCPH ANNEX`.

### Team: the PickleDrive copy

There is no Team Tournament Master yet. A team event's workbook is made by copying
PickleDrive's and clearing its inputs, so **only an event of PickleDrive's shape
can be made this way**: 15 teams `A`–`O` in three brackets of five, four pairs per
matchup, 152 matches, one venue and 10 courts. Any other shape waits for the
master.

!!! note "Admin task — needs the team workbook and the site repo's runbook"
    The cells to clear are listed in `_templates/CLAUDE.md`, "Team events: the
    workbook", in the site repo. They are repeated below. The organizer can do the
    copy; ask the admin if a cell is unclear.

1. Open the source: the workbook **2026-10-03 PickleDrive Club One Year
   Celebration**. **Never edit it.** Choose **File → Make a copy** and name the copy
   `<date> <title> - <FACILITY>`, like a generated workbook.
2. Turn on **View → Show → Formulas**. **Clear values only, and never a cell
   holding a formula**: leave any cell that starts with `=`.
3. Clear the inputs:
   - **`Teams`:** type the new event's team names and players over the old ones.
   - **`MatchUps`:** clear every lineup. Type each playoff seed cell back to its
     seed number (the cells are listed in
     [the team workbook](../features/team-workbook.md#playoff-seed-cells)).
   - **`SCHEDULE`:** clear every typed score. A score must be a truly empty cell
     (**Delete**, not a space), because the formulas test for an empty cell.
   - **`Court Control`:** clear every match number on a court.
   - **`ATTENDANCE`:** clear rows 2 down in columns `A:G`. **Update roster** refills
     them.
   - **`Title`:** the new event's title and date.
   - **`Raffle`:** clear it.
   - **Slot times** are `SCHEDULE!B6:B`. Retype them if the event starts at another
     time and they are typed values.

The copy keeps the sync script but not its trigger. [Step 5](connect-the-workbooks.md)
creates the trigger and replaces the copied day key.

## Pack `SCHEDULE` (standard only)

A standard tournament's `SCHEDULE` starts empty. You place matches on courts by
copying from `MATCHES`, which keeps you in charge of which categories share the
courts when. Each venue of each day has its own `SCHEDULE`.

1. On `MATCHES`, select whole court blocks: both rows of the slot, starting at a
   block's left edge. The label beside each row (`RR 1`, `SF`, `F + B`) is to the
   left of the block, so it doesn't come with it.
2. Paste them onto a `SCHEDULE` slot with **Paste special → Formula only**.
3. Keep to the rules nothing checks for you:
   - no pair twice in one slot;
   - a playoff round only after the round before it has finished, with a **free slot
     after the round robin for the qualifier draw**;
   - a final and its bronze in the same slot;
   - mixed doubles after the same-gender categories.
4. Leave unused courts blank or `-`.

The generator's execution log suggests a layout that follows those rules, if you
would rather copy one than work one out. The reference is on
[the Standard Tournament Generator](../features/standard-tournament-generator.md#packing-the-schedule).

## File it

Move each workbook into the shared drive folder **SAGE → 1. TOURNAMENTS →** the
event's own folder, named `<YYYY-MM-DD> <event name>`, for example
`2026-10-03 PiggleBall Tournament`. Create the folder if it doesn't exist. It also
holds the event's `BRACKETS` folder and the schedule PDF.

**`1. TOURNAMENTS` is shared with the API's service account as Editor, and a
workbook inside it inherits that share.** This was checked on Piggleball's workbook
on 2026-10-08, so there is no separate share step. The service account is what lets
score entry and attendance write to the workbook.

Then set **General access** to **Anyone with the link: Viewer** on **every workbook,
every time**. The folder does not set it, and the sync reads the workbook with an
API key, which needs it. A workbook that loses it shows **Sync failed** later.

## Fill match numbers

For every format, in every workbook: **SAGE → Fill match numbers**.

1. Enter a base number. `0` starts at 1; `1000` starts at 1001.
2. **Each venue of a day gets its own range** (1000, 2000…), because a day's venues
   show together on the site and no two matches may share a number. Nothing checks
   the venues against each other.

It numbers every slot that has a team on both sides, and writes the same numbers
into the `CSV` tab's `matchNumber` column, which is what the website reads.

For a standard tournament, **then check `Timeline`**: its total should equal the
match count on `MATCHES`, and no cell should show more than 1. A cell showing 2 is
a pair booked twice in one slot.

## Check it worked

- Every workbook is in the event's folder, and its name reads
  `<date> <title> - <VENUE>`.
- **File → Share** on each lists **Anyone with the link: Viewer**, and the
  service account `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com`
  as Editor (inherited).
- `SCHEDULE` is full of matches, each numbered, and no number repeats across a
  day's venues.
- **SAGE → Help** reports the workbook's status and the script versions.
- A team workbook: `STANDINGSCSV` lists the new teams with zero points, `CSV` has
  152 rows with every score empty, and every quarterfinal and later row reads its
  seed (`QF-3_1`…).

## If it doesn't

- **The generator lists problems and stops.** It checks the plan before writing
  anything and lists every problem at once: the plan is the wrong format, clubs are
  uneven, a court label is repeated, no category is ticked. Fix the plan and run it
  in a **fresh copy**. See the refusals on the
  [Dual Meet](../features/dual-meet-sheet-generator.md#if-it-refuses-to-run) and
  [Standard](../features/standard-tournament-generator.md#if-it-refuses-to-run)
  generator pages.
- **A run stops partway.** The bar turns red and says why. The tabs it finished stay,
  and the copy cannot be generated again. Make a fresh copy of the master.
- **Fill match numbers writes nothing.** A slot has a team on one side only, or a
  code sits on a slot's second row. Fix the `SCHEDULE` and run it again.
- **The Share dialog does not list the service account.** The workbook is not in
  the event's folder under **1. TOURNAMENTS**. Move it there, or share it by hand as
  Editor.

See [9. When something's wrong](troubleshooting.md).

## Where the formats differ

|  | Dual meet | Standard | Team |
| --- | --- | --- | --- |
| Workbooks | one | one per venue per day | one |
| Source | Dual Meet Master | Standard Tournament Master | copy of PickleDrive's |
| `SCHEDULE` | filled by the generator | empty: **pack it** | cleared and kept |
| Plan | from the calculator | from the calculator | by hand |

**Next:** [3. Draw and fill rosters](draw-and-rosters.md).
