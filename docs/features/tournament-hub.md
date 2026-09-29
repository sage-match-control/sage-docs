# Tournament Hub

Every S.A.G.E. event gets its own Tournament Hub — a public page linked
from a QR code posted at the venue. No app to install, no login to
remember — open it on a phone and everything below is already running.

## For players: find your matches

Type your pair's names into **Match Finder** and see your entire day at
once — every round, every opponent, in schedule order — without scrolling
past anyone else's matches. Search by either player's name or the full pair.

Each match card ("ticket") shows:

- The scheduled time and court
- A **Live** pill and pulsing dot while the match is actually in progress on
  a court
- A **Next Up** pill on whichever of your matches is next and not yet live
- The score, once it's in
- Which round it is (pool play, or a named playoff round like Quarterfinal,
  Semifinal, Bronze, Final)

## For spectators: watch it happen

**Live scores and court assignments** update on their own while play is
underway — no refreshing, no flagging down a volunteer to ask what's
happening on Court 6.

**Live standings** — win/loss records and rankings — update the same way,
sorted by division and category, so the board on your phone is never a stale
printout. At a standard tournament, a division whose round robin is split
into brackets shows one small table per bracket, labelled **Bracket 1**,
**Bracket 2**…, side by side. On a computer screen, the **1 col** button in
that division's header stacks them in one narrower column instead, and
**2 cols** puts them back.

Within each round-robin table, pairs are ranked by **wins**, then
**head-to-head** (among pairs level on wins, whoever won the matches
between them), then **quotient**. When three or more pairs are level and
their results against each other go in a circle, quotient decides. This is
the website's ranking only: the organizer's spreadsheet picks who advances
to the playoffs by wins and then quotient, so on a head-to-head tie the
two can differ.

**Club Showdown** *(dual-meet events only)* — a running head-to-head win
count between the two clubs sits front and center on Standings, updated
after every match, with whichever club is ahead visually highlighted.

## Team events

A team event has named teams instead of pairs. Two teams meet in a
**matchup** — four matches (men's doubles, women's doubles and two mixed
doubles) between the same two teams — and **team names appear everywhere a
team does**: Standings, Match Finder, Live Matches and the schedule board.
The letter codes the organizer uses in the workbook (`A`, `B`…) are not shown here;
Control Center still shows them beside each team name.

**Reading a matchup card.** The header gives the stage (a bracket,
Quarterfinal, Semifinal, Bronze or Final), the two time slots and the courts. Under it, both team names
face each other around the **matchup score**, with the **pair count** (for
example *pairs 3–1*, the matches each team won) in small type beneath. Below
that, one row per match shows the pair (MD, WD or XD), the players on each
side and the score. A match being played right now carries a live dot and its
court. A pill marks a matchup that is *In progress*, *Final* or a *Tie*, and
the team that won a finished matchup is marked **Winner**.

**Why a matchup is won on total points.** The team with more points across
all four matches wins the matchup — not the team that wins more matches. A team
can win three matches 11–9 and lose the fourth 0–11, and still lose the
matchup 33–38. That is why the pair count is only secondary information. If
both teams finish on the same total, the matchup is a **tie** (equal points),
shown as *Tie* with no winner named.

**Standings** show a table per bracket, ranked by the organizer's
tiebreakers, in order:

1. **Points scored** across the team's bracket matches.
2. **Quotient** — points scored divided by points conceded.
3. **Head-to-head points** — among the teams still level, the points each
   scored in its matches against the others.
4. **Pair wins** — the number of matches the team won in its bracket.

Teams level on all four stay in the workbook's order. Before the first result
the tables list the teams without rank numbers. Eight teams reach the
**quarterfinals**, seeded 1–8 and paired 3 v 6, 1 v 8, 2 v 7 and 4 v 5. The
organizer decides who they are by typing their letters into the workbook
against each seed; the site marks those teams **Advances** and never works it
out for itself. Until a seed is filled in, its card reads *Seed 3 · TBD* and so
on. Below the tables come the playoff matchups — Quarterfinals, Semifinals,
Final, Bronze — then each bracket's matchups behind a collapsible heading.

**Searching by team or player.** Match Finder takes a team name or a player's
name. A team result lists every matchup that team plays, in schedule order,
with the team on the left and *Next up* on its first matchup still to be
played. A player result lists each match they play, each inside its matchup
card with only their own match showing. With nothing typed, tapping a team
in the list runs its search.

**Lineup not set.** A team captain enters each matchup's players in the
workbook before it is played. Until then its match rows read *Lineup not set*
(the schedule board says *Lineup TBD*), and a player can't be found until
their lineup is in. The same player may play a different pair in each matchup.

## At the venue: the schedule board

A wall display meant for a screen at the venue, not a phone — courts as
columns, time slots as rows, one card per match, color-coded by category.
Live matches get a highlighted ring; finished ones dim and show their score.
It's unlisted (nothing on Tournament Hub links to it) — an operator launches
it from Mission Control and hands the URL to whoever's running the venue's
screen. See [Control Center § Mission Control](control-center.md#mission-control).

When a day is played at more than one venue, a **Venue** picker at the top
narrows the board to one venue's matches and courts, so each venue's screen
shows only its own. The board also supports splitting by court range (so a
two-screen venue can show different courts on each), collapsing its header
down to just the essentials for a screen that needs every pixel, and
printing/exporting to PDF for a paper copy at the front desk. The venue and
court choices stay in the page address, so bookmark each screen's page once
and it comes back the same after a restart.

At a team event each card names both teams above their players and carries the
pair label (MD, WD or XD), and cards are coloured by bracket rather than by
category, with playoffs in their own colour.

## What you *won't* see here

Tournament Hub is read-only and always shows the current state of play —
it has no sign-in, no editing, and (before an event goes live) hides scores
and standings entirely rather than showing incomplete data. The tools
operators use to run the event — Mission Control, the Awards podium/export,
resyncing a facility — live in a separate console. See
[Control Center](control-center.md).

---
**Technical:** [Control Center architecture](../technical/control-center.md) · [schedule board](../technical/schedule-board.md) · [how live data reaches the page](../technical/sync-pipeline.md)
