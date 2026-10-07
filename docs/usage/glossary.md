# Glossary

The terms the guide uses, each in a sentence or two, linking to where it matters.

**Attendance.** Staff check-in for an event. A person is marked in once, and that
covers every category they play that day at that venue. Set per event as
`"console"` or `"desks"`. See [event attendance](../features/event-attendance.md).

**Awards.** Control Center's podium tab. Gold, silver and bronze come from the
scored Final and Bronze matches. See [Control Center](../features/control-center.md#awards).

**BYE.** A slot with no opponent. A pair with a bye advances without a match. A
BYE can't be scored.

**Bracket.** A group of pairs who play each other in a round robin. A category can
have several. Also what the Bracket Generator draws. Not to be confused with a
team event's *bracket stage*, which is its group.

**Category.** One division × event, for example Beginner 18 Men's Doubles. A
category has its own tab in the workbook.

**Category, division and event codes.** A category's code is `<DIVISION><EVENT>`:
`B18MD` is division `B18` and event `MD`. The registration maps each code to a
label. A code the registration doesn't know shows a warning on the console.

**Club code.** The short code of a club in a dual meet, e.g. `PNF`. A dual meet's
team codes start with it.

**Court Control.** The workbook tab that says which match is on which court right
now. Entering a match number against a court makes the match **live**. See
[the scoring workbook](../features/scoring-workbook.md).

**Day key.** The identifier of one tournament day in `events.json` (e.g.
`pnf-bup-day1`). It must be **unique across every event**. Each workbook is told
its day key during **Set up live sync**.

**Desk link.** A link that lets desk staff mark people in from their phone, for
one day. See the [desk handout](desk-handout.md).

**Event key.** The identifier of the whole event (e.g. `pnf-x-bup-dual-meet`). It is
the event's folder name in the site repo and in `event-data`, its key in
`events.json`, and it appears in the workbook's `Title` tab. Choose it once and
keep it byte-identical everywhere. See [1. Plan it](plan-the-event.md#choose-the-keys).

**Facility / venue.** One place where matches are played. The registration calls it
a *facility*, the pages call it a *venue*. A day can have several. Its name is
compared **exactly, including capitals**, between the registration and the
workbook.

**Go-live.** When the Tournament Hub starts showing a day's live scores and
standings. By default (`Auto`) about 4 hours before the day's first match.
Control Center's **Public site status** can force it live or hidden.

**Hub.** See *Tournament Hub*.

**Live (match).** A match whose number is in `Court Control`. It shows a live
pill on the Hub and a ringed card on the schedule board.

**Live push.** The service that sends each published snapshot straight to open
pages, within seconds. If it is down, pages catch up through GitHub in 30–60 seconds.

**Live sync.** The workbook's script that sends the workbook to the site on every
edit. Set up once per workbook with **SAGE → Set up live sync**.

**Match number.** A match's number, unique within a day. Venues of one day each
get their own range (1000, 2000…). See [2. Build the workbooks](build-the-workbooks.md#fill-match-numbers).

**Matchup.** At a team event: four matches between the same two teams, won on
total points. See [the team workbook](../features/team-workbook.md).

**Playoff.** The rounds after the round robin: Quarterfinal, Semifinal, Bronze,
Final. See [Run the day](run-the-day.md#playoff-hand-offs).

**Qualifier draw.** A standard tournament's second draw, made after the round
robin, which places the qualifiers into playoff slots.

**Round robin.** Every pair in a bracket plays every other. Ranked by wins, then
head-to-head, then quotient on the website.

**Scorer link.** A link that lets scorer staff enter scores for any venue of one
day, for 24 hours. See the [scorer handout](scorer-handout.md).

**Seed (Bracket Generator).** The word or number that decides a draw. Anyone can
re-check a draw from it. **Seed (team event).** The number of a playoff slot,
replaced by a team letter once the organizer decides who advances.

**Service account.** The Google account the API uses to write scores and
attendance into a workbook:
`sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com`. It needs **Editor**
on every workbook of an event that uses score entry or attendance.

**Snapshot.** The published copy of a venue's workbook that every page reads.
**Sync** makes a fresh one.

**Sync.** Sending the workbook to the site. Happens on every edit, and by hand with
**Sync now** or **Resync this day now**.

**Team code.** The code of a pair in a match: `B18MD_1` at a standard event,
`PNF_B18MD_1` at a dual meet, `A_3` at a team event. See
[Adding a new event § Team code format](../technical/adding-a-new-event.md#team-code-format).

**Tournament Hub.** The public event page. See [Tournament Hub](../features/tournament-hub.md).

**Twice-to-beat.** A final in which the round robin's #1 only has to win once and #2
has to win twice. If #2 wins game 1, game 2 decides it.

**Walkover.** A match not played because one side is absent or withdrawn. The
opponent's win is entered as a score. See [Late changes](run-the-day.md#late-changes).

**Withdrawn.** On the attendance roster: a person no longer on the workbook's
roster. They keep their row and are hidden.
