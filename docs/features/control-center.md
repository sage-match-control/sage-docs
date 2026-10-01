# Control Center

Control Center is the operator's console for running a tournament day —
one page, covering every registered event, always showing live data (unlike
Tournament Hub, it never hides scores while previewing a day before it's
publicly live). Open it, pick an event and a day, and five tabs appear, in
this order: **Mission Control** (where it opens), **Awards**, **Live
Matches**, **Match Finder** and **Standings**.

No sign-in is needed to view Awards, Live Matches, Match Finder or
Standings — they're read-only.
Only the actions inside Mission Control (resyncing, forcing the public site
live or hidden) need an operator to sign in.

**Installable** — from Chrome's install prompt on Android, or "Add to Home
Screen" on iOS Safari — so an operator's device can launch straight into
it with its own icon, no browser chrome or address bar around it. This is
just a shortcut to the same live page; there's no offline mode, so it
still needs a network connection to do anything, same as opening it in a
regular tab.

## Mission Control

The operator-only control panel — the one part of Control Center that needs
signing in, and the tab it opens on. From top to bottom:

- **Sign in** with the shared operator username/password. This exchanges your
  password for a short-lived session token held only in that browser tab —
  the password itself is never stored anywhere.
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
- **Resync this day now**, right under the venues — pulls a fresh copy from
  the spreadsheet(s) immediately, instead of waiting for the next automatic
  sync. On a day with more than one venue, **Or resync just one facility**
  does the same for one venue. **Use CSV export fallback** switches either
  one to Google's CSV export, if the normal way of reading the sheet fails.
  The result appears right below, one line per venue: **Synced**, **This
  attempt failed. Still showing its previous data**, or **Failed. Nothing
  published for it yet**, then whether it was pushed to open pages or went
  through GitHub, and how long it took.
- **Check connection** — confirms Cloud Run is reachable and shows its
  version. When you're signed in it adds a **Live push** line saying whether
  Cloud Run is publishing scores to the live push service (**on**, **off**,
  **switched off in Control Center**, or **not available** on an older
  version). Signed out, the line asks you to sign
  in. This is about Cloud Run's side; the **Live updates** line under Facility
  sync status is about this page's own connection. The result appears
  right under the button, one line each for the response time, the version,
  which copy of the event registry Cloud Run is using, and live push.
- **Public site status** — the kill switch for Tournament Hub's Live
  Matches/Standings (this console's own tabs are unaffected and always show
  live data, for preview). Three states: `Auto` (goes live automatically
  ~4 hours before the day's first scheduled match), `Force live`, or
  `Force hidden` — for quietly correcting a bad score before anyone sees it.
- **Sync method** — an emergency switch between **Live push + GitHub** (the
  normal way: scores reach open pages in a few seconds) and **GitHub only**
  (every score goes through GitHub, so pages catch up within 30–60 seconds, as
  before live push). Use it if live updates misbehave: pages stop updating, or
  show stale scores. Switching to GitHub only asks you to confirm, and either
  way it takes effect within about a minute. It shows the current method, and it
  is hidden until you sign in. If Cloud Run has no live push configured at all,
  it says so and there is nothing to switch.
- **Public pages** — launchers for the venue's schedule board and the
  event's Tournament Hub, both opening in a new tab so the console stays
  where it is.

Quick outcomes, like signing in or out, switching the public site or the
sync method, or "pick a day first", appear as a short message at the bottom
of the screen, wherever you are on the page. Errors stay until you close
them or for about nine seconds. Messages from Cloud Run, Google or GitHub are
shown in plain words, never as raw data.

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
hand to an emcee or post on social media. An **Export whole tournament**
button at the top produces a single combined image covering every category
at once, arranged as a grid so it stays a reasonable shape regardless of how
many categories the event has.

**Team events** have one podium, **Team Championship**. Gold is the winner of
the Final matchup, silver its loser, and bronze the winner of the Bronze
matchup, each decided on total points. A placing reads **Pending** until its
matchup is finished. Each medal shows the team name with every player who
played for that team listed beneath, A–Z. If the Final or Bronze matchup ends
level on points, the card shows a warning naming the matchup's lowest match
number and leaves all three placings Pending. **Export image** puts the team
names on the picture, without the player lists.

A few things the Awards tab is careful about:

- A category decided by walkover (no bronze match actually needed to be
  played) shows a **Walkover** tag on that medalist instead of pretending a
  match happened.
- If a match's scores look wrong (e.g. tied, which shouldn't be possible), the
  card shows a warning naming the match number instead of guessing a winner.
- A category with no playoff bracket at all (pure round robin) falls back to
  its top-three standings, tagged **By standings** so it's clear where that
  podium came from. All three placings read **Pending** until every
  round-robin match in that category has a score — before then the top three
  are only the current leaders.

## Live Matches

A board of every court currently in use, grouped by venue if the event has
more than one. Each court shows the category and round of whatever's playing
there, both teams' names, and the live score — or "No match playing" for an
idle court. This is the same view a wall-mounted screen at the venue would
show.

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

## Match Finder

Identical to Tournament Hub's Match Finder — search a pair's name, see
their full day of matches with Live/Next Up flags and scores. Useful for an
operator fielding "where's my match" questions without having to also pull up
Tournament Hub itself.

## Standings

Win/loss records and rankings by category. For a **dual meet**, standings are
split by club within each category (with a shared win-count summary at the
top), and the Bronze/Final rounds — since those pit one club against the
other — are shown as a single head-to-head block rather than nested under
either club.

Round-robin pairs are ranked by wins, then head-to-head among pairs level
on wins, then quotient — the same order as the public event page. The
Awards tab's standings fallback uses it too. The workbook's playoff feeders
don't: they rank by wins then quotient, so on a head-to-head tie the pair
the sheet advances can differ from the one listed first here.

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

At a **team event**, Standings has three parts, the same as on the public
event page: a table per bracket ranked by points scored, then quotient, then
head-to-head points, then pair wins (with the eight quarterfinalists the
organizer entered in the workbook marked **Advances**), the playoff matchups
— Quarterfinals, Semifinals, Final, Bronze — as full cards, and each bracket's matchups behind a collapsible heading. A
matchup is won on total points across its four matches — see
[Tournament Hub § Team events](tournament-hub.md#team-events). Live Matches
names both teams on each court and shows the running matchup score, and Match
Finder searches by team or player.

If a match's team code doesn't match any configured category, a visible
warning banner names it rather than silently lumping it into an "Other"
bucket — so a data problem in the spreadsheet gets noticed instead of hidden.

---
**Technical:** [Control Center architecture, incl. Awards tab internals](../technical/control-center.md) · [sync pipeline](../technical/sync-pipeline.md) · [auth](../technical/auth.md)
