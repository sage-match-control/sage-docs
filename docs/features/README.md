# Features

A reference to the S.A.G.E. tools: what each one is, what it shows, and every
option it has. Each page covers one tool and carries no setup sequence. If you are
new, or you are running an event, start with the **[Usage guide](../usage/README.md)**.
It takes you from planning the event to the day of the tournament and after, and
links here for the detail. Every page here says which Usage step it comes into.

If you're looking for how something is *built*, its counterpart page in
[Technical](../technical/README.md) is linked from the bottom of each page here.

## Public pages

For players and spectators, and for the screens at the venue.

- **[Tournament Hub](tournament-hub.md)** — the public event page: Match
  Finder, live scores, live standings, the club-vs-club win count, and team
  events' matchups and rosters. Nothing to install, nothing to sign into.
- **[Schedule board](schedule-board.md)** — the venue wall display: courts as
  columns, time slots as rows, one card per match.

## Running the day

For operators, scorers and desk staff.

- **[Control Center](control-center.md)** — the operator console. Mission
  Control (sign-in, resync, the public-site switch, scorer links), Awards,
  Attendance, Match Finder, Live Matches, Standings and Teams.
- **[Score entry](score-entry.md)** — the dialog that enters, corrects or clears
  a match's score without opening the workbook.
- **[Scorer page](scorer-page.md)** — for scorer staff: enter match scores on a
  phone with a scorer link from the operator, at any venue of the day.
- **[Event attendance](event-attendance.md)** — staff check-in for any event,
  in Control Center or on a desk link from any phone. One check-in per person,
  with their arrival time.

## Scoring workbooks

The Google Sheets where scores are typed and standings are computed.

- **[The scoring workbook](scoring-workbook.md)** — the tabs every workbook
  has, which ones a person edits, and the whole **SAGE** menu.
- **[Dual Meet Sheet Generator](dual-meet-sheet-generator.md)** — turns a
  finished dual-meet plan into the event's scoring workbook.
- **[Standard Tournament Generator](standard-tournament-generator.md)** — the
  same for a standard tournament, one venue-day workbook at a time.
- **[Team workbook](team-workbook.md)** — the team event's workbook: `Teams`,
  lineups, playoff seed cells, and why a matchup is won on points.

## Planning and print tools

Standalone tools. Nothing to sign into.

- **[Tournament Calculator](tournament-calculator.md)** — plan a tournament's
  timing before the draw is even final: how many matches, how many courts, what
  time it ends.
- **[Bracket Generator](bracket-generator.md)** — paste in a category's pairs
  and draw its brackets on screen, exported as an image or a text file.
- **[Scoresheet Generator](scoresheet-generator.md)** — turn a matches CSV or a
  registered event's day into printable PDF scoresheets.

## Three kinds of event

S.A.G.E. supports three tournament shapes, and it matters which one you're
looking at:

- **Standard tournament** — an open-entry bracket event, any number of days
  and venues. Categories are just categories.
- **Dual meet** — two clubs facing off. Every category, standings, and the
  Awards tab's "Overall Champion" line are framed as Club A vs. Club B.
- **Team** — named teams meeting in matchups of four matches, won on total
  points, with a group stage and then playoffs. See the
  [team workbook](team-workbook.md).

Every page in this section notes where the formats differ.
