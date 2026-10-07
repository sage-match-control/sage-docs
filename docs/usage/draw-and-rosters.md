# 3. Draw and fill rosters

**Who:** organizer · **When:** as soon as entries close (7–10 days before) ·
**You need:** the generated workbooks from [step 2](build-the-workbooks.md), and
each category's list of pairs

The workbook says how many brackets each category has and how
many pairs go in each. This step draws the brackets, puts the names in the
workbook, and checks that they show.

## Run a draw people can trust

For a draw people are watching, use the Bracket Generator's **Verifiable Draw**:

1. Put the pairs up on screen **before** anyone supplies a seed. Once the list is
   visible it can't be quietly changed, and the seed doesn't exist yet, so
   nothing can be tuned to suit it.
2. **Ask the room for the seed.** Any word or number, ideally called out by a
   player rather than by you. Type it in and say it out loud.
3. Draw. The seconds of shuffling on screen are a reveal; the seed already decided
   the answer.

The full ceremony, and how a player re-checks their own pair on any SHA-256 site,
is [Running a draw people can trust](../features/bracket-generator.md#running-a-draw-people-can-trust).
If you must re-draw, say so out loud, fix the list and ask for a new seed. The
exports carry a draw number, so a second attempt shows as `Draw #2`.

## Steps (standard tournament and dual meet)

Draw one category at a time.

1. On the category's tab in the workbook, choose **SAGE → Open Bracket Generator**.
   It opens the tool with the event name and that category already filled in.
2. Paste that category's pairs, one per line. Set the **bracket count** to the one
   the workbook used. Add **keep-apart groups** if some pairs must not meet in the
   round robin.
3. Run the draw.
4. **Save both exports:** the **image**, which is what you post and print, and the
   **text file**, which is the one you work from next. The text file also carries
   the seed and every pair's fingerprint, so the draw can be re-checked months
   later. Put them in the event's `BRACKETS` folder (see
   [step 2](build-the-workbooks.md#file-it)).
5. Repeat for every category. Eight categories means eight images and eight text
   files.

Then put the names in the workbook, which depends on the format.

### Standard tournament: import the text files

1. In the workbook choose **SAGE → Import bracket draws**.
2. Drop in **every category's text file at once** and import.

For each file it fills the category tab's **`STEP 1 · NAMES`** with the pairs in
draw order and **`STEP 3 · RANDOMIZED CODES`** with its codes. It records the seed,
the draw number and the filename beside the roster, so you can tell months later
which draw a tab came from.

It matches each file to a tab by its category line, and asks you to pick a tab for
any file it can't place. It checks every file before writing anything. A draw with
the wrong number of pairs, the wrong number of brackets or the wrong bracket sizes
is refused by name with both shapes reported, and nothing is written. A tab that
already has names is refused unless you tick **Replace existing names**, which is
also how a re-draw gets applied.

Do this **in every venue's workbook** for the categories that venue runs. The
menu item stays after a successful import, so you can run it again.

### Dual meet: paste the names, then shuffle

A dual meet's draw is a per-club roster blind, so there is nothing to import.

1. Paste each club's players into its **`STEP 1 · NAMES`** column.
2. On that tab choose **SAGE → Shuffle roster codes**. It draws each club's
   **`STEP 3 · RANDOMIZED CODES`** independently from the codes in `STEP 2`.

`STEP 3` ships blank on purpose: filling it in order maps every pair to its own
slot and undoes the blinding. The menu item refuses a tab with no roster, and asks
before replacing codes that are already there, because reshuffling a live workbook
re-points every pair.

### Team: type the `Teams` tab

A team event has no bracket draw to import. The organizer decides the groups.

1. Type or paste each team's name and its players (with level and gender) into
   **`Teams`**. Captains can change it through the day.
2. Lineups are entered on `MatchUps` later, as captains hand them in. See
   [the team workbook](../features/team-workbook.md).

## Check it worked

- On every category tab, column **B** shows **names**. A tab still showing codes
  instead of names means the roster and the codes haven't met.
- No category tab shows an error.
- Every category has a saved image and a saved text file (standard and dual meet).
- A team workbook: `STANDINGSCSV` lists the right team names, and the `Teams` tab
  lists every player.

## If it doesn't

- **Import refused a file.** The message names the file and gives both shapes. Redraw
  with the right bracket count, or fix the pair count.
- **A file can't be placed.** Pick its tab from the dropdown. A file left unassigned
  is skipped, not guessed.
- **Shuffle refused the tab.** The tab has no roster. Paste the names first.
- **Names do not appear.** Check that `STEP 1` and `STEP 3` are both filled and
  that the tab is the right category.

See [9. When something's wrong](troubleshooting.md).

## Where the formats differ

- **Standard:** import the draw text files. The qualifier draw is separate, made
  after the round robin on the day ([step 8](run-the-day.md#playoff-hand-offs)).
- **Dual meet:** paste, then shuffle in the sheet. No file to import.
- **Team:** type `Teams`. Lineups come later.

**Next:** [4. Register and build the site](register-and-build-the-site.md).
