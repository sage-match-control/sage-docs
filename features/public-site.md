# The public event site

Every S.A.G.E. event gets its own page, linked from a QR code posted at the
venue. No app to install, no login to remember — open it on a phone and
everything below is already running.

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
printout.

**Club Showdown** *(dual-meet events only)* — a running head-to-head win
count between the two clubs sits front and center on Standings, updated
after every match, with whichever club is ahead visually highlighted.

## At the venue: the schedule board

A wall display meant for a screen at the venue, not a phone — courts as
columns, time slots as rows, one card per match, color-coded by category.
Live matches get a highlighted ring; finished ones dim and show their score.
It's unlisted (nothing on the public site links to it) — an operator launches
it from Mission Control and hands the URL to whoever's running the venue's
screen. See [Match Control § Mission Control](match-control.md#mission-control).

It supports splitting by court range (so a two-screen venue can show
different courts on each), collapsing its header down to just the essentials
for a screen that needs every pixel, and printing/exporting to PDF for a
paper copy at the front desk.

## Bracket Generator

A separate tool for building and exporting playoff bracket graphics — paste
in the pairs, get a clean bracket image ready to print or post, without
wrestling a spreadsheet into a diagram. Self-contained; pairs are entered by
hand, nothing is fetched live.

## What you *won't* see here

The public site is read-only and always shows the current state of play —
it has no sign-in, no editing, and (before an event goes live) hides scores
and standings entirely rather than showing incomplete data. The tools
operators use to run the event — Mission Control, the Awards podium/export,
resyncing a facility — live in a separate console. See
[Match Control](match-control.md).

---
**Technical:** [Match Control console architecture](../technical/match-control.md) · [schedule board](../technical/schedule-board.md) · [how live data reaches the page](../technical/sync-pipeline.md)
