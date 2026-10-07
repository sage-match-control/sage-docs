# Score entry

**Used in:** [4. Register and build the site](../usage/register-and-build-the-site.md) ·
[8. Run the day](../usage/run-the-day.md)

The score dialog: enter, correct or clear one match's score without opening the
workbook. It opens from [Match Finder in Control Center](control-center.md#match-finder)
for a signed-in operator, and from the [scorer page](scorer-page.md) for scorer
staff with a scorer link. Both use the same dialog.

## Turning it on

Score entry is a setting on the event, `scoreEntry`, in
`event-data/config/events.json`:

| Setting | What it means |
| --- | --- |
| `"scoreEntry": "console"` | Signed-in operators enter scores in Match Finder. |
| `"scoreEntry": "links"` | Operators can, and scorers can with a **scorer link** from Mission Control. The event also needs its scorer page. |
| *(absent)* | Score entry is off. Nothing in Match Finder is clickable. |

Every venue's workbook must be shared with the API's service account as
**Editor**, so it can write the score. Putting the workbook in the event's
folder under **1. TOURNAMENTS** does this; see
[Usage step 2](../usage/build-the-workbooks.md#file-it). Mission Control's
**Scorer links** switch moves an event between `"links"` and `"console"`. It
never turns score entry on or off.

## Opening a match

While signed in, every match in Match Finder carries a small **✎ Enter score**
hint, or **✎ Edit score** once it has a score. Click a match, or focus it and
press Enter, to open the dialog. On the scorer page, tap a match. Signed out,
nothing is clickable.

For a team event the clickable parts are the match lines inside each matchup
card. Live Matches, Standings and Awards stay read-only. A BYE can't be scored.
In a standard or dual-meet event a match whose players aren't decided yet opens
read-only and says so. In a team event a match with no lineup can still be
scored, with a warning.

The dialog puts the people first. For a pair, the two players' names are the
bold headline of each side and the team code is a small tag beside them. For a
team, the team name leads and the players sit underneath.

## The two steps

1. **Enter.** The two sides sit in the same order as the sheet, team 1 on the
   left. Each score takes up to two digits. Enter moves from the first box to
   the second, and from the second to the next step.
2. **Review.** The winner is named in the largest text, above the score, so a
   pair of scores typed against the wrong sides shows up before anything is
   saved. A tie reads **Tied — neither side wins this match**. Anything unusual
   is listed as a warning and never refuses the save: a tie, neither side
   reaching 11, a win by one point, a score over 21, a series game that isn't
   needed because the series is already decided, or a team match with no lineup.
   Pressing Enter twice in a row on the last box does not save: the second press
   is ignored for a moment after Review appears.

**Save** writes both scores into that match's cells of the venue's workbook and
publishes the venue straight away, so pages update within a few seconds, just as
if someone had typed the score in the sheet. A message at the top of the screen
confirms it. A big workbook can be slow to answer: after a few seconds the dialog
says it is still waiting, and a save that gets no answer in two minutes says so.
Saving again is safe, because a score already in the sheet is not written twice.

## Corrections and clearing

Opening a match that already has a score shows that score in the boxes, and
Review adds **Was 11 – 9**. **Clear score**, on the first step, empties both
cells again, after a Review that reads **Clear the score of match #42**.

## When the sheet changed

If the match's cells changed since the dialog opened (someone typed into the
sheet, or another operator or scorer saved), nothing is written. The dialog
shows what the sheet reads now and what you entered, and offers **Keep the
sheet's score** or **Replace with yours**. If the match number now belongs to a
different pairing, the only choice is **Close**. While the dialog is open, a note
appears under the scores when a new snapshot shows that the sheet's score
changed. The boxes are left as they are.

## When a save fails

- **Publishing failed.** The score reached the sheet but the update could not be
  published. A warning says so. Use **Resync this day now** on Mission Control.
  A scorer is told to tell the operator.
- **The workbook is not shared.** The dialog names the service account to share
  it with.
- **The sign-in expired.** The dialog says so. Sign in on Mission Control and
  save again.
- **Scorer links are stopped.** The save is refused until the operator sets the
  switch back to **Accepting**.

[9. When something's wrong](../usage/troubleshooting.md) lists these with the
fix.

---
**Features:** [Control Center](control-center.md) · [Scorer page](scorer-page.md)
**Technical:** [Control Center § Score entry](../technical/control-center.md#score-entry)
