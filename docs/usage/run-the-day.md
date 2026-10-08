# 8. Run the day

**Who:** operators, with scorers and desk staff · **When:** doors open to the last
match · **You need:** the event ready ([step 7](rehearse.md)), the operator login,
the gear from [Roles, access and kit](before-you-start.md#equipment), and
[9. When something's wrong](troubleshooting.md) open in another tab

The console is where you **watch** the day. The workbook is where you **run** it. Keep
every venue's workbook open all day, and Control Center open beside it.

## Before doors open

1. Get the operator device on the venue's wifi. Keep the hotspot ready.
2. Open Control Center, select the event and today's day.
3. **Sign in** under Mission Control.
4. Click **Check connection**. It confirms everything is reachable before a single match
   depends on it. Its second line should read **Live push: on**.
5. Click **Resync this day now**. It pulls a fresh copy of the schedule and confirms
   each venue's workbook can be read. It does not prove the workbook sends its own
   updates: that only shows once the first real edit in **Court Control** appears on
   the site without a resync.
6. Confirm **Facility sync status** shows a fresh **Synced** for every venue, and that
   **Live updates** reads **push connected**.
7. Open each venue's **Google Sheet** from its row in Facility sync status, and keep it
   open all day. Scores and Court Control go there.

### Deciding when the public site goes live

Under **Public site status**: by default (**Auto**) the public site goes live on its
own, about 4 hours before the day's earliest scheduled match. No action needed. To
show it sooner, for example so players can check their schedule when they arrive, click
**Force live**.

### The venue screens

- Click **Open schedule** and put that on the wall display. If the day is split across
  venues, pick that venue's button in the **Venue** row first, then **bookmark** the
  page. The bookmark reopens straight to that venue. A venue with two screens can split
  the courts, one bookmark each.
- Click **Open Tournament Hub** for the public page. It suits a second screen showing
  Standings or Live Matches, or just to confirm it looks like what a player sees.
- Check the **hub board** is up with this event's QR panel.
- On a phone (not the console, which always shows live data), open the public page and
  confirm it looks like what a player would see.

### Attendance desks and scorer links

- **Attendance:** open the **Attendance** tab and press **Update roster** once rosters
  are final. For `"desks"`, press **Issue desk link** and send the link, or its QR code,
  to each desk, with the [desk handout](desk-handout.md). A desk link works until the end
  of that day in Manila time.
- **Scorer links:** under **Scorer links** press **Issue scorer link** and send the link,
  or its QR code, to each scorer, with the [scorer handout](scorer-handout.md). One link
  covers every venue of the day and is valid for 24 hours, so issue it that morning. It
  can't be issued once the day is over.

When you issue a desk or scorer link, fill in **Issued to** with the person's name and
**Note** with their gate or courts (`Ana`, `Gate A`; `Courts 3–4`). The workbook then shows
who did what: the `markedBy` column of `ATTENDANCE` and the note on each score cell read
`Ana (Gate A)`, not just `Desk link`. Issue one link per person or desk, and ask them not
to pass it on: the label records who the link was issued to, not who typed, so a forwarded
link writes under its first owner's name.

## During play

Three jobs run at once, as set out in
[Roles, access and kit](before-you-start.md#how-many-operators):

- **Calling matches:** Court Control, calling each match to its court, and the playoff
  qualifier draws.
- **Scores and playoff names:** entering scores, and writing in the playoff names after
  each draw.
- **Players and the organizer:** players' questions, and every conversation with the
  organizer.

The operator calling matches never also coordinates with the organizer.

### The court rhythm

The operator calling matches, at each court as matches happen, in the workbook's
**Court Control** tab:

1. As a court frees up, enter the **next** match's number against that court. This is what
   makes the match show as live on the Live Matches tab, the schedule board and the Hub.
2. When a match finishes, replace that court's entry with the **following** match number
   immediately. **Never leave a court blank.** A blank court shows as idle instead of
   telling anyone what's coming up.
3. Call the next match to its court.

The operator on scores enters each finished match's **score**.

Everything else happens on its own. Scores and court status reach the public site and the
wall display within a few seconds.

### Three ways to enter a score

All three write the same two cells of the match in the workbook. Mix them freely. Choose
by who is available, as set out in [Roles, access and kit](before-you-start.md#who-enters-scores).

| Way | How |
| --- | --- |
| **In the sheet** | Type both scores into the match on `SCHEDULE`. |
| **In Control Center** | Match Finder → click the match → [Score entry](../features/score-entry.md). Needs `scoreEntry` set. |
| **A scorer's phone** | The [scorer page](../features/scorer-page.md), from a scorer link. |

If two people change the same match, the second to save is shown what the sheet now
reads and chooses **Keep the sheet's score** or **Replace with yours**, instead of
overwriting it.

### What to watch

- **Facility sync status**, periodically. It should read **Synced** from a few seconds to
  a couple of minutes ago, continuously. If it goes stale, click **Resync this day now**
  yourself rather than waiting. A score or court entry that hasn't appeared within about
  **2 minutes** gets the same.
- **A bad score that needs a quiet fix.** Set **Public site status** to **Force hidden**,
  correct the sheet, resync, then set it back to **Auto**. This hides the Hub only.
  Control Center keeps showing everything, so you can verify the fix first.
- **Live updates misbehaving** (pages stop updating or show stale scores). Open Mission
  Control and check that **Live updates** reads **push connected** and **Check
  connection** says **Live push: on**. If the live service is the problem, switch **Sync
  method** to **GitHub only**: every score then goes through GitHub and pages catch up
  within 30–60 seconds. Switch it back when things are healthy. It takes effect within
  about a minute.
- **Scorer links that should stop** (a link sent to the wrong person, say). Set **Scorer
  links** on Mission Control to **Stopped**: within about a minute every scorer's save is
  refused, and you can still enter scores yourself in Match Finder. Set it back to
  **Accepting** when ready.
- **A player asking where their match is.** The operator on players and the organizer
  uses Match Finder, in Control Center or on the Hub.

## Playoff hand-offs

The site works out the standings. At three points a person has to hand it a decision, and
until they do, the playoff matches read **TBD**.

### Standard tournament: the qualifier draw

After the round robin, before the first playoff slot, a category's qualifiers are drawn
by lot into playoff slots. **How it is drawn varies by event, and how and by whom is the
organizer's call.** The operator calling matches runs it, and the operator on scores
writes the result into the category tab's **qualifier draw** area. The draw lists which finisher each row is waiting
for, and the slot table beside it lists every slot the draw can land on. This is why
[step 2](build-the-workbooks.md#pack-schedule-standard-only) leaves a free slot after the
round robin.

A category with two brackets has no qualifier draw. Its semifinals are a crossover (Br 1 #1
v Br 2 #2, Br 2 #1 v Br 1 #2), so the slots come filled in.

Then check that the playoff matches show the right names:

- [ ] Each playoff match on `SCHEDULE` shows the pairs the draw placed there.
- [ ] After a resync, **Live Matches**, **Standings** and Match Finder show the same names,
  not **TBD**.

### Team event: the seed cells

When the group stage ends, the operator on scores types each qualifier's team letter into
its quarterfinal seed cell on `MatchUps`. After each playoff round, enter the next round's teams the same way.
The site takes every playoff team from those cells and never picks them itself. The cells
are listed on [the team workbook](../features/team-workbook.md#playoff-seed-cells). Check
that each card changes from *Seed n · TBD* to the team's name and that the team is marked
**Advances**.

### Twice-to-beat finals

In a category settled by a twice-to-beat final, the #1 pair wins by winning once and the #2
pair by winning twice. If #1 wins game 1, game 2 is not needed. It greys out, never
shows **Next Up**, and doesn't count as a match left. Don't put it on a court. If #2 wins
game 1, game 2 is played, and gold waits for it.

## Late changes

The organizer decides each case, through the operator on players and the organizer. The
operator on scores makes the change in the workbook. The options and their effect:

- **A substitute, or a replacement for a pair that hasn't played yet.** Overwrite the names
  in `STEP 1 · NAMES`. The codes and the schedule stay.
- **A walkover.** The opponent's win is entered as a score.
- **Either way,** tell the operator calling matches, and check the change reaches the Hub.
- **A schedule change after the scoresheets are printed.** For a minor change, correct the
  slips by hand. For a major one, generate the scoresheets again and reprint
  ([step 6](printables.md#scoresheets)).

## End of day

1. Once the last Finals and Bronze matches are scored, check the **Awards** tab. Every
   category should show its medalists, or a warning naming a match if a score looks off.
2. Use **Export image** per category, or **Export whole tournament** for one combined
   image, to hand results to the emcee or post them.
3. Check **Standings** looks complete.
4. Nothing needs turning off. Once the public site has gone live, it stays that way.
5. If anything broke, note the rough time. `event-data` keeps a timestamped commit for
   every sync and every override.

## Check it worked

- Every venue shows **Synced** and recent.
- A court is never blank during play.
- The Hub, the board and the console agree.
- At the end, Awards lists every category's podium.

## If it doesn't

See [9. When something's wrong](troubleshooting.md).

## Where the formats differ

- **Standard:** the qualifier draw; one workbook per venue to keep open.
- **Dual meet:** the Awards tab opens with an **Overall Champion** banner naming the club
  with more round-robin wins.
- **Team:** seed cells and lineups; a matchup is won on total points; one podium,
  **Team Championship**.

**Next:** [After the event](after-the-event.md).
