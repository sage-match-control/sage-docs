# Control Center

**Used in:** [4. Register and build the site](../usage/register-and-build-the-site.md) ·
[5. Connect the workbooks](../usage/connect-the-workbooks.md) ·
[7. Rehearse](../usage/rehearse.md) · [8. Run the day](../usage/run-the-day.md)

Control Center is the operator's console for running a tournament day —
one page, covering every registered event, always showing live data (unlike
Tournament Hub, it never hides scores while previewing a day before it's
publicly live). Open it, pick an event and a day, and the tabs appear, in
this order: **Mission Control** (where it opens), **Awards**, **Attendance**,
**Match Finder**, **Live Matches**, **Standings** and **Teams**. **Attendance**
shows only for an event with attendance turned on, and **Teams** only for a
team event whose rosters are published.

No sign-in is needed to view Awards, Match Finder, Live Matches or
Standings — they're read-only.
Only the actions inside Mission Control (resyncing, forcing the public site
live or hidden), marking people in on the Attendance tab and entering scores in
Match Finder (or on Standings, from a team event's matchup cards or a playoff bracket card) need an operator to sign in. Signed out, the Attendance tab asks you to
sign in instead of showing the list, and no match in Match Finder can be scored.

**Installable** — from Chrome's install prompt on Android, or "Add to Home
Screen" on iOS Safari — so an operator's device can launch straight into
it with its own icon, no browser chrome or address bar around it. This is
just a shortcut to the same live page; there's no offline mode, so it
still needs a network connection to do anything, same as opening it in a
regular tab.

## Mission Control

The operator-only control panel — the one part of Control Center that needs
signing in, and the tab it opens on. It is a set of cards. On a phone they stack in one column, in
the order below. On a desktop screen, **Facility sync status** sits on the left, with **Public site
status** and **Scorer links** stacked beside it on the right, and the sign-in card, **Pages and tools**
and **Troubleshooting** span the full width. A card's explanation sits behind the **i** beside its title
(**Public site status**, **Scorer links**, and in Troubleshooting **Sync method** and the CSV fallback): hover over
it, or reach it with the keyboard, on a computer; tap it on a phone, and tap anywhere else to close it.

- **Sign in** with the shared operator username/password, in the card at the top beside the day.
  This exchanges your password for a session token that lasts 12 hours — the password itself is
  never stored anywhere. In a browser tab, closing the tab signs you out.
  Installed on a phone's home screen, Control Center stays signed in until the
  12 hours run out or you press **Sign out**, even if the phone closes the app
  in the background. The status line reads **Signed in until 3:30 PM**, with
  the date added (**3:30 AM, Tue, Oct 6**) when the sign-in ends on a later day. The note about
  how the password is handled shows only while you are signed out.

  **Other tabs in the same browser share the sign-in.** A Control Center tab opened while another
  one is signed in picks the sign-in up from it within a second, with nothing to type; signing in
  in one tab signs in the others that are open; and **Sign out** in one tab signs every open tab
  in that browser out, with the message **Signed out in another tab.** in the others. Only this
  browser profile is affected: another device, another browser or a private window stays signed
  out until you sign in there. Closing every tab still ends a browser-tab sign-in.
- **Facility sync status** — whether each venue's spreadsheet is syncing
  cleanly, and when it last succeeded. Each row has a **Google Sheet**
  link that opens that facility's actual Google Sheet for this event, in
  a new tab — for checking or correcting a score straight at the source
  without having to go find it in Drive. Whether it opens editable
  depends on that sheet's own Google sharing settings, not on Control
  Center.
  When a venue's latest sync came from an edit, its row also reads
  "edit→sync <n>s" — how long that edit took to reach the published data.
  It is a diagnostic for operators and is hidden when the figure is not a
  plausible single edit (a full resync, for instance).
  Under each venue, a second line repeats its matches done and estimated
  finish (or, once it's done, its actual end) from Live Matches, and adds
  "stale — may read late" when that venue's data is old enough to make the
  estimate unreliable.
  Below the venues, a **Live updates** line reads **push connected** when
  this page is receiving scores the moment they are published, or **polling
  GitHub (push not connected)** when it is not. In the second case updates
  still arrive, just 30–60 seconds slower.

  **Resync this day now**, right under the venues, pulls a fresh copy from
  the spreadsheet(s) immediately, instead of waiting for the next automatic
  sync. On a day with more than one venue, **Or resync just one facility**
  shows one button per venue and does the same for that venue; the buttons wrap onto more
  than one row when they do not fit. If the normal way of reading the sheet fails, the **Use CSV
  export fallback** switch under **Troubleshooting** switches either one to Google's CSV export.
  The result appears right below, one line per venue: **Synced**, **This
  attempt failed. Still showing its previous data**, or **Failed. Nothing
  published for it yet**, then whether it was pushed to open pages or went
  through GitHub, and how long it took.
- **Public site status** — the kill switch for Tournament Hub's Live
  Matches/Standings (this console's own tabs are unaffected and always show
  live data, for preview). Three states: `Auto` (goes live automatically
  ~4 hours before the day's first scheduled match), `Force live`, or
  `Force hidden` — for quietly correcting a bad score before anyone sees it.
- **Scorer links** — for an event with score entry turned on: the switch that
  accepts or stops scorer links, and **Issue scorer link**. See [Scorer
  links](#scorer-links) below.
- **Pages and tools** — a row of tiles, each opening in a new tab so the console stays
  where it is: **Open schedule** (the venue's schedule board); for an event with attendance, when you are
  signed in, **Open schedule (staff view)**, the same board with who is not in yet marked (see the schedule
  board's [staff view](schedule-board.md#staff-view)); **Open Tournament Hub** (the event's public page, as
  players and spectators see it right now); **Open scoresheet generator**, which opens the [Scoresheet
  Generator](scoresheet-generator.md) on the selected day and follows the day when you change it; and
  **Open prize draw**, which opens the [Prize draw](prize-draw.md) on the selected event and day; and
  **Open clock**, which opens the full-screen [Clock](clock.md) with the event's name above the time.
  Every tile is a raised button with an ↗ (it opens a new tab); **Open schedule** is the dark one. On a
  desktop they share one row, however many there are; on a phone they stack.
- **Troubleshooting** — closed until you open it, and this browser remembers whether you left it
  open. It holds what you reach for when something is wrong:
    - **Check connection** — confirms Cloud Run is reachable and shows its
      version. When you're signed in it adds a **Live push** line saying whether
      Cloud Run is publishing scores to the live push service (**on**, **off**,
      **switched off in Control Center**, or **not available** on an older
      version). Signed out, the line asks you to sign
      in. This is about Cloud Run's side; the **Live updates** line under Facility
      sync status is about this page's own connection. The result appears
      right under the button, one line each for the response time, the version,
      which copy of the event registry Cloud Run is using, and live push.
    - **Sync method** — an emergency switch between **Live push + GitHub** (the
      normal way: scores reach open pages in a few seconds) and **GitHub only**
      (every score goes through GitHub, so pages catch up within 30–60 seconds). Use it if live
      updates misbehave: pages stop updating, or
      show stale scores. Switching to GitHub only asks you to confirm, and either
      way it takes effect within about a minute. It shows the current method, and it
      is hidden until you sign in. If Cloud Run has no live push configured at all,
      it says so and there is nothing to switch.
    - **Use CSV export fallback instead of Sheets API** — changes how the resync buttons read the
      sheets. Turn it on only when a resync fails through the Sheets API. It is hidden until you
      sign in.

Quick outcomes, like signing in or out, switching the public site or the
sync method, or "pick a day first", appear as a short message at the top
of the screen, wherever you are on the page. Errors stay until you close
them or for about nine seconds. Messages from Cloud Run, Google or GitHub are
shown in plain words, never as raw data.

The result boxes (resync, Check connection) have a close button, and they
clear themselves when you change the event, the day or the tab, so a result
never sits under a different day. A resync that finishes after you've moved
on reports as a short message at the top instead.

### Scorer links

For an event whose `events.json` entry has a `scoreEntry` setting, Mission
Control has a **Scorer links** section. A scorer link opens the event's
[scorer page](scorer-page.md), where staff enter scores for any venue of one day
and do nothing else; they never see Control Center.

- **Accepting / Stopped** is a toggle: on is **Accepting**, off is **Stopped**.
  **Accepting** lets scorer links save scores and be issued. **Stopped** makes every
  scorer save fail and refuses new
  links, within about a minute, while operators can still enter scores in Match
  Finder. Links issued earlier work again if the switch goes back to
  **Accepting** before they expire. The switch only moves between those two
  states; it never turns score entry on or off, which is a setting in
  `events.json`. Until you sign in, the section shows the current state and nothing to
  press. The toggle and **Issue scorer link** are also off, with a note, when the event
  has no scorer page (`events/<key>/scorer.html` isn't on the site), since a link would
  open nothing; add the page first.
- **Issue scorer link for Oct 3** opens a dialog that makes a link for the selected
  day. It covers every venue of that day and is valid for 24 hours from when it is
  issued, not until midnight. It can't be issued once the day is over (from 6:00 AM
  the morning after), or while the switch reads **Stopped**.
- In the dialog, two optional fields, **Issued to** and **Note** (at most 40
  characters each), say who the link is for: a name, and for example the courts
  they cover. **Issue link** issues it; **Cancel** issues nothing. See
  [Who a link was issued to](#who-a-link-was-issued-to).
- The link appears with **Copy**, **Share** (on devices that support it),
  **Show QR** and **Valid until Mon, Oct 5, 8:30 AM (24 hours)**, in Manila
  time. When it was issued to someone, the title names them. Send it, or have the
  scorers scan the QR code.

A scorer page that is already open finds out about a stop when a save is refused
("Scorer links are stopped for this event. Ask the operator."); one that is
reloaded says so as soon as it loads.

#### Who a link was issued to

Both **Issue scorer link** and **Issue desk link** open a dialog with two optional
text fields: **Issued to** (a name) and **Note** (for example `Gate A` or
`Courts 3–4`). Each is at most 40 characters. **Issue link** issues the link with
whatever is filled in, or with neither. Both are signed into the link, so the holder
can't change them. The dialog opens empty every time, so the next link isn't labelled
with the last one's name. If a link can't be issued, the dialog says why and stays open
with what was typed, to try again.

They show up in three places:

- the page the link opens, after the day: ` · Day 1 · Ana (Gate A)`;
- the `markedBy` column of the workbook's `ATTENDANCE` tab, for a desk link: see
  [event attendance](event-attendance.md#who-marked-someone-in);
- the note on both score cells of a match, for a scorer link: see
  [score entry](score-entry.md#who-entered-a-score).

The label reads `Ana (Gate A)` with both, `Ana` with only a name, `Desk link (Gate A)`
or `Scorer link (Courts 3–4)` with only a note, and `Desk link` or `Scorer link` with
neither. An operator's own marks and scores in Control Center read `Control Center`.

The label says who the link was **issued to**, not who typed. Anyone holding a
forwarded link writes under its label, so give each person or desk their own link.
A link issued with neither field works the same way, with no label.

## Awards

The podium tab: once a category's Final and Bronze matches are scored, this
tab shows who won gold, silver, and bronze — derived automatically from the
match data, nothing to type in separately. Before those matches are played,
every placing just reads **Pending**; nothing is ever guessed.

**Overall Champion** *(dual-meet events only)* — a banner at the top naming
whichever club has more Round Robin wins overall, with the runner-up's win
count shown underneath. If the two clubs are tied, it says so rather than
picking one.

**Exporting for the ceremony** — every category card has an **Export image**
button that downloads a SAGE-branded PNG of that category's podium, ready to
hand to an emcee or post on social media. A long pair of names goes onto two
lines, one player each, in smaller type if it has to, so no name runs off the
picture. An **Export whole tournament**
button at the top produces a single combined image covering every category
at once, arranged as a grid so it stays a reasonable shape regardless of how
many categories the event has.

**Team events** have one podium, **Team Championship**. Gold is the winner of
the Final matchup, silver its loser, and bronze the winner of the Bronze
matchup, each decided on total points (or, for an event scored on games won, on
matches won and then total points). A placing reads **Pending** until its
matchup is finished. Each medal shows the team name with every player who
played for that team listed beneath, A–Z. If the Final or Bronze matchup ends
level on points, the card shows a warning naming the matchup's lowest match
number and leaves all three placings Pending. **Export image** and **Export
whole tournament** put each team's name on the picture with its players
beneath. On the single-podium image the player list shrinks to fit its row,
and a roster too long for even the smallest size ends in "+N more".

A few things the Awards tab is careful about:

- A bronze decided by walkover (no bronze match actually needed to be
  played) goes to that pair like any other bronze, with no score and no
  label. That holds whether the workbook schedules a Bronze match against a
  **BYE** or only lists the round robin's #3 against a **BYE** in the bronze
  slots with no match at all.
- A twice-to-beat Final is won by the #1 pair winning once or the #2 pair
  winning twice, and a best-of-3 by whoever wins two. Gold waits until one of
  them has: a #2 pair that wins game 1 of a twice-to-beat has forced game 2,
  not won. A game that is no longer needed doesn't count as a match left on
  the facility's progress card either.
- If a match's scores look wrong (e.g. tied, which shouldn't be possible), the
  card shows a warning naming the match number instead of guessing a winner.
- A category with no playoff bracket at all (pure round robin) falls back to
  its top-three standings, tagged **By standings** so it's clear where that
  podium came from. All three placings read **Pending** until every
  round-robin match in that category has a score — before then the top three
  are only the current leaders.

## Attendance

Staff check-in, for an event whose entry in `events.json` has an
`attendance` setting. The tab lists everyone playing at each venue that day,
grouped by category (or by team, for a team event), with a switch per person.
Flip it to mark someone in; it records the time. Flipping it off asks **Mark … not
present?** first, so a stray click can't clear a check-in. Above the list:

- the number of people in at each venue
- **Update roster**, to bring the list up to date with the workbook now
  instead of waiting for the next sync
- **Issue desk link**, for an event set to `"desks"`: a link for one day that
  lets desk staff mark people in from their own phones, with **Copy**,
  **Share** and **Show QR**. It opens the same dialog of optional **Issued to** and
  **Note** fields as [scorer links](#who-a-link-was-issued-to)
- **Needs attention**: names that might be the same person written two ways,
  people listed twice in one category, and **Show withdrawn**

Full reference: [event attendance](event-attendance.md).

## Match Finder

Identical to Tournament Hub's Match Finder — search a pair's name, see
their full day of matches with Live/Next Up flags and scores, under the same
summary card: the next match (or **Playing now** with the court), category,
round-robin rank with W, L, PF, PA and Q, and where the pair is in the playoffs. Each ticket carries
its match number as a dark tab in its top-left corner, and its category beside the
round. Useful for an
operator fielding "where's my match" questions without having to also pull up
Tournament Hub itself. Before a search it lists every match of the day in
match-number order (a team event: its team chips, then every matchup,
ordered by its lowest match number). A team event's matchup cards show their
matches with a **Hide matches** button, as on the Hub.

The search box sits just above the list, not in the banner, and on a phone it
stays pinned to the top of the screen while you scroll. **Find matches** closes
the suggestions, as Enter and picking one do. Control Center never scrolls the
page after a search, as Tournament Hub does on a phone.

Control Center's search also takes a **match number**: type `42` or `#42`
to see just that match (for a team event, its matchup card narrowed to that
match). A number that isn't on the day says so.

### Who is not in yet

For an event with [attendance](event-attendance.md) turned on, a signed-in operator sees who has not
checked in. A player who is on the day's attendance list, has not withdrawn and is not yet in at any
venue shows greyed, with a small hollow amber dot before their name. A checked-in player, a withdrawn
one, and a name that is not on the list (**TBD**, **BYE**, a misspelling) show nothing. Only a match
still to be played is marked, and a series game that will never be played is not. An unplayed match with
anyone not in ends its line with a chip, **2 not in**, counting the different players. Ticking someone in
on the Attendance tab, or on a desk page, clears their marks within a few seconds, and signing out removes
every mark. The Tournament Hub never shows them, and the [schedule board](schedule-board.md#staff-view) has the same
marks only in its staff view. A team event's Match Finder has no marks.

### Entering a score

For an event with score entry turned on, a signed-in operator can enter, correct
or clear a match's score from Match Finder (and, for a team event, from the
matchup cards on Standings) without opening the workbook. See
[Score entry](score-entry.md).

## Live Matches

A board of every court currently in use, grouped by venue if the event has
more than one. Each court shows the match number (a dark tab at the top-left of
the match), the category and round of whatever's playing there, both teams'
names, and the live score — or "No match playing" for an idle court. This is
the same view a wall-mounted screen at the venue would show.

A match stays on its court until someone sets the next one, so a court whose match
has both scores in reads **Done**: a check mark on an amber court cell in place of
the pulsing dot, **Done** after the round, the names greyed and the final score kept
dark. The count above the board, and each venue's, keeps these courts apart:
**2 courts in play · 1 done**. (The Tournament Hub shows the same **Done** rows, but
counts a done court as in play.) A series game that is no longer needed reads
**Done** the same way.

For an event with attendance, a signed-in operator also sees [who is not in
yet](#who-is-not-in-yet) here: the names greyed with the amber dot, and the **N not in** chip after the
category and round. Each venue's progress card gains a line, **Still to check in N**, counting the people
on that venue's list who have not withdrawn and are not in yet; it is left out when everyone is in, and
until that venue's list has loaded. A team event's court rows have no name marks, but its venue cards
still carry the **Still to check in** line.

### Set match

For a signed-in operator at an event whose `scoreEntry` is set (the same setting as
[Score entry](score-entry.md)), each court's own cell, the number at the left of its row, is a
button. A small **✎ Set match** (an idle court), **✎ Change** (a court with a match on it) or
**✎ Next match** (a court whose match is **Done**) sits under the number. It puts a match on that court, replaces it, or clears the court, without
opening the venue's workbook. It writes the same `Court Control` cell an operator types into, so
the two can be mixed freely.

The dialog has three steps:

1. **Pick.** The court's title (with the venue when the event has several), and what is on it
   now. Under **Next up** are the next three matches at that venue that are not played, not on a
   court and not a BYE, in schedule order and with this court's own scheduled matches first within a
   time. Choose one, or type a match number. A number that is not at this venue, is a BYE, is already
   on this court, or is on another court (**Clear that court first**) shows why and cannot be
   reviewed. **Clear court** is offered when a match is on the court.
2. **Review.** What will happen, with warnings the operator may go ahead past: the match already has
   a score, it is scheduled for another court, one of its players is on a court now (at this venue
   or another), its players or lineup are not decided yet, or the match it replaces is not scored yet
   and will show as not started.
3. **Save.** The match goes on the court and the Hub, the board and Control Center show it live
   within a few seconds. If someone changed the court in the workbook while the dialog was open, it
   shows what the sheet now holds and offers **Keep the sheet's** or **Replace with yours**.

Moving a match from one court to another is two actions: clear the old court, then set the new
one. The sheet refuses what a warning cannot cover: a court cell holding a formula or text, a match
that is not in `SCHEDULE`, a BYE, and a match already on another court. Only a signed-in operator can
set a court; a scorer link cannot. Technical detail:
[Control Center § Court control](../technical/control-center.md#court-control).

Above the courts, a card per venue shows how many matches are done, how many
are left, how many are in play (on a court right now, so not counted as left;
done, left and in play add up to the venue's total), and — on the day itself — an estimated finish time and how far
ahead of or behind schedule that venue is. The estimate is the operators'
own formula: matches left plus idle court slots left, times the scheduled
match length, divided by the number of courts. It never counts less than one
match length per match still queued on the busiest court, and it keeps
working past midnight until 6:00 AM. If a venue's sheet stops syncing, the
card says so, because no new scores means the estimate drifts later on its own.

Once every match at a venue has both scores in, its card reads **All matches
done** with the **actual end** time, and underneath, the scheduled end and
how early or late the venue finished (for example "1 h 15 min early"). The actual end is recorded once,
the moment the last score first arrives, and every phone and laptop reads the
same time. A later **Resync this day now** doesn't move it. Clearing a score
reopens the venue, and it gets a new actual end when the score goes back in.

## Standings

Win/loss records and rankings by category. For a **dual meet**, standings are
split by club within each category (with a shared win-count summary at the
top), and the Bronze/Final rounds — since those pit one club against the
other — are shown as a single head-to-head block rather than nested under
either club.

Each round-robin table shows W, L, **PF** (points for), **PA** (points
against) and Q (quotient, PF ÷ PA), the same columns as the public event
page. PF and PA add up the pair's played matches; a match against a BYE
counts for neither.

Round-robin pairs are ranked by wins, then head-to-head among pairs level
on wins, then quotient — the same order as the public event page. The
Awards tab's standings fallback uses it too. The workbook's playoff feeders
don't: they rank by wins then quotient, so on a head-to-head tie the pair
the sheet advances can differ from the one listed first here.

Each round-robin table's heading carries a status pill, **In progress · 6/10** (six of the table's
ten round-robin meetings are played) or **Final**, and a pair level on wins with another carries a
pill under its names saying how the table placed it: **Tie · won head-to-head 11–5**,
**3-way tie · on quotient**, or a yellow **Tied** when nothing separates the pair. Control Center
shows them whether or not the public day is live, since its Standings always show the scores. At a team
event the status pill is in each bracket's title, and the tie pills are not drawn. The words are the
public event page's: see [Tournament Hub](tournament-hub.md).

For an event with attendance, a signed-in operator sees the same greyed names with the amber dot in the
list view: in the round-robin tables, and in the playoff matchup cards. A round-robin row is not a match,
so its names are marked whenever the person is not in. The bracket chart and the full tree are unchanged,
and a team event's standings carry no marks.

A pair whose names aren't in the sheet yet is listed under its team code
(`ND_1`…) in the round-robin tables, so the standings are complete before the
rosters are. Undecided playoff spots read **TBD**, and the Awards tab never
puts a code on the podium.

When a category's round robin is split into brackets, each bracket gets its
own small table under a colored **Bracket 1**, **Bracket 2**… label, two
side by side, so parallel pools read as pools. On desktop, the **1 col**
button in that category's header stacks its brackets in a single, normal-width
column instead (**2 cols** puts them back); the choice lasts until the page is
reloaded. On a desktop screen every
category is its own column in one row; scroll sideways for the rest. A long
player name is never cut off — its table scrolls sideways instead.

At a **standard** event, a category with two or more playoff rounds has a
**List | Bracket** switch under its title (the same one as on the
[Tournament Hub](tournament-hub.md); the choice is remembered in the browser).
**Bracket** draws the playoffs as a chart: rounds side by side, a line from
each match to the seat its winner fills, and the Bronze under the Final. An
empty seat reads **Winner of QF-2**, **Loser of SF-1**, the event's own label
for a seat the round robin fills, or **Qualifier**; once the names are in, a
seat the round robin filled also shows where the pair finished (**Br 2 · #1**).
A screen too narrow for the chart shows the same chart, lines included, in a window
you swipe sideways, with round chips to jump to a round; **Full tree**, under the
chart on a tablet as well as a phone, opens the whole chart to pinch and drag. Here a card with a scheduled match opens
the score dialog when clicked, in the chart and in Full tree, for a signed-in
operator at an event with score entry on (see [Score entry](score-entry.md)).
The Hub's cards are not clickable.

At a **team event**, Standings has three parts, the same as on the public
event page: a table per bracket ranked by points scored, then quotient, then
head-to-head points, then pair wins (with the eight quarterfinalists the
operators entered in the workbook marked **Advances**), the playoff matchups
— Quarterfinals, Semifinals, Final, Bronze — as full cards, and each bracket's matchups behind a collapsible heading. A
matchup is won on total points across its four matches, or, for an event whose
`teamScoring` is `"games"`, on matches won and then total points, with the table
showing W, L, PF, PA and Quotient, ranked by the event's tiebreakers (a tie only a
playoff can settle reads **Tied**), and each matchup card each side's PF, PA and
quotient — see
[Tournament Hub § Team events](tournament-hub.md#team-events). Live Matches
names both teams on each court and shows the running matchup score, and Match
Finder searches by team or player.

If a match's team code doesn't match any configured category, a visible
warning banner names it rather than silently lumping it into an "Other"
bucket — so a data problem in the spreadsheet gets noticed instead of hidden.

## Teams

For a team event: every team's roster, the same as the event page's Teams tab.
Cards start collapsed and open to show each player's level and gender. Match
Finder shows a team's roster with its matchups, and finds a player by name from
the roster before their lineup is in. A player's result shows their level and
gender beside their name.

### Lineup check

An event whose settings list its lineup rules, as the Millennium Invitational's
do, gets a **Lineup check** box above the rosters. It lists every lineup that
breaks a rule, by match number, and each such match carries the same warning
on its line in Match Finder and in the score dialog:

- a player in two pairs of the same round, or a full round that leaves a
  rostered player out (everyone plays once a round, and a tiebreaker's pair may repeat a player);
- a player who isn't on the team's roster;
- **Men's High** without two men, or without an advanced player;
- **Mixed** without one man and one woman, or with the team's captain (the
  first player in the team's roster);
- **Women's** with a man in it. **Genderless** has no restriction.

The organiser can let a named player into a pair the rule would keep them out
of (at the Millennium Invitational, Gian may play Women's); that player draws no
warning there.

Gender and level come from the workbook's `Teams` tab, so a check waits until
the roster has them, and the box says so. A warning never stops a score or a
lineup: an injured player's emergency replacement may break a rule on purpose.
Only operators see it; the event's public pages show no warnings. Each warning
also sits on its match on every matchup card: Match Finder's and the
Standings tab's playoff cards and bracket matchups. A matchup card whose
matches have any stands out with a red edge and header and a badge naming the
pairs, such as **⚠ 7 lineup issues · MD, WD, XD**, so it shows even while the
card's matches are folded away. Each of those pairs' match lines is tinted, with
a red bar and a red pair chip.

For an event that holds lineups back until each round (the Millennium
Invitational), Control Center always shows every lineup, and a matchup card whose
lineup is in but not yet shown to the public reads **Lineup hidden**. Ticking the
matchup's **Show lineup** box in the workbook's `MatchUps` tab, then **Resync this
day now**, shows it on the Hub and the board, and the
badge goes once the sync publishes it.

---
**Technical:** [Control Center architecture, incl. Awards tab internals](../technical/control-center.md) · [sync pipeline](../technical/sync-pipeline.md) · [auth](../technical/auth.md) · [event attendance](../technical/event-attendance.md) · [scorer page](../technical/scorer-page.md)
