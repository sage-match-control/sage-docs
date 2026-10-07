# 7. Rehearse

**Who:** the coordinator with the operators who will run the day · **When:** the last few days, well before
the event, while it doesn't matter if something is briefly wrong · **You need:**
every earlier step done, the real workbooks and the real site, and the operator
login

The rehearsal is the same shape as the day: enter a few fake scores, confirm they
reach Control Center and the public page, mark a match live and confirm it shows as
live, then revert. It is the only step that exercises the whole chain end to end, and
the only place a case-sensitive venue name or a missing column is cheap to find. Don't
skip it, however short the time.

## Steps

### Pick your test rows

Pick three real, already-scheduled matches from the day and write down their match
numbers and team codes:

- **Match A** will simulate a **completed** match.
- **Match B** will simulate a **live** match.
- **Match C** will simulate an **unmapped category code**.

### Edit the sheet

1. **Match A:** on `SCHEDULE`, type two different scores (for example 11 and 7).
2. **Match B:** in `Court Control` (not on `SCHEDULE`), enter Match B's number against
   a court this venue uses, exactly as you would to mark a match live for real.
3. **Match C:** change one of its team codes so its division/event prefix no longer
   matches anything in the registration's `display` labels (for example tack on an
   extra letter). This is what should trigger the console's unmapped-code warning
   instead of a silent "Other".
4. **Team events — lineup and playoff slots.** Enter one matchup's lineup on
   `MatchUps`. Within about 30 seconds the site should show those players' names in
   place of *Lineup not set*. If it doesn't, check `MatchUps` is ticked in **SAGE →
   Set up live sync**. Then type a team letter into one quarterfinal seed cell on
   `MatchUps` (see [the seed cells](../features/team-workbook.md#playoff-seed-cells)).
   That quarterfinal card switches from *Seed n · TBD* to the team's name, and the
   team gets an **Advances** label. Put the seed number back afterwards.

### Check the console (operator device only)

Sign in under Mission Control, click **Resync this day now**, then check:

- [ ] **Live Matches:** Match A shows the score you set. Match B shows a live pill on
  the court you set. Every other court at that venue shows an idle placeholder, not
  blank.
- [ ] **Standings:** a visible warning banner names Match C's unmapped code. It is not
  a silent "Other" bucket.
- [ ] **Match Finder:** search a real player's name. It returns their matches in
  schedule order, with the right opponent and score state.
- [ ] **Awards:** it loads without errors and lists every category. Before any Final or
  Bronze is played, every placing reads **Pending**.
- [ ] **Public site status:** set it to **Force hidden**, then back to **Auto**. The
  console's own tabs stay visible throughout.
- [ ] **Score entry** (if the event has it): open a match in Match Finder, enter a score,
  **Save**, and see it appear in the sheet. Then clear it with **Clear score**.

Don't move on until every check passes.

### Check the public screens

Open the pages as a visitor would, with the schedule board on one screen and the Hub on
a phone:

- [ ] **Schedule board:** courts and times render. Match B's court shows as in progress.
- [ ] **Live Matches** on the Hub matches what the console showed. If **Force hidden**
  was on at any point, confirm the Hub goes hidden rather than showing scores.
- [ ] **Standings** on the Hub: the same unmapped-code warning and the same standings as
  the console.

### Check scorer links and attendance

- [ ] **Scorer links** (`"links"` events): press **Issue scorer link** under Mission
  Control, open the link on a phone, pick a venue, enter and save a test score, and
  clear it. Set **Scorer links** to **Stopped**, check that a save is refused (it takes
  about a minute), and set it back to **Accepting**.
- [ ] **Attendance:** **Update roster** reports every venue's people. Mark a test person
  in, see it appear in the workbook's `ATTENDANCE` tab, and undo it. For `"desks"`,
  **Issue desk link** and open the link on a phone.

### Clean up

- [ ] Clear both scores on Match A.
- [ ] Clear Match B from `Court Control`.
- [ ] Restore Match C's original team code.
- [ ] A team event: put every seed cell back to its seed number and clear the test
  lineup.
- [ ] **Resync this day now** once more, to publish the reverted state.

## The "Ready" checklist

Check the event against this before the day.

- [ ] The plan's finish time is one the venue will accept.
- [ ] Every category has a drawn bracket, with its image and text file saved.
- [ ] Every category tab has its roster filled and its codes drawn, and no tab shows an
  error.
- [ ] A standard tournament's `SCHEDULE` is packed and numbered in every venue's
  workbook.
- [ ] The event page loads, shows the right days and categories, and lists every venue.
- [ ] A test score typed into each venue's workbook reached Control Center and the
  public page, and has been reverted.
- [ ] Every venue reports a recent successful sync.
- [ ] With attendance: **Update roster** reports every venue's people, a test mark
  appeared in `ATTENDANCE` and has been undone, and for `"desks"` a desk link opens on a
  phone.
- [ ] With scorer links: a test link saved a score, and the stop switch refused one.
- [ ] The schedule PDF matches the site, and the scoresheets are printed.
- [ ] The hub board carries this event's QR panel, and scanning it opens this event's
  page.
- [ ] Everyone has their [handout](scorer-handout.md) and their link ready to send.

At that point the event is ready.

## If it doesn't

A failure here is the rehearsal doing its job. Find it in
[9. When something's wrong](troubleshooting.md), fix it, and repeat the check.

## Where the formats differ

- **Team:** add the lineup and seed-cell checks above.
- **Standard:** also check every venue's workbook, not just one.
- **Dual meet:** one workbook, so one pass.

**Next:** [8. Run the day](run-the-day.md).
