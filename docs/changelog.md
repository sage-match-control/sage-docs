# Changelog

What changed in these docs, release by release. The other pages describe the
system as it is; the history of how the docs got there lives here.

## How the docs are versioned

The docs carry a version, `MAJOR.MINOR.PATCH`, shown in the footer of every page.
What a reader would notice decides which number goes up:

| Bump | When | Examples |
| --- | --- | --- |
| **MAJOR** | A page or nav section is removed, renamed or moved, so a bookmark or a link from another repo stops working. | Splitting a tab in two; deleting a page and sharing its content out. |
| **MINOR** | Something new to read: a new page or section, a page updated for a change in the product, or a new spec or a spec moving status folder. | A features page for a new tool; a feature's new behaviour; a spec moving to `implemented/`. |
| **PATCH** | A correction that changes nothing in the product: a wrong fact, a broken link, a typo, clearer wording. | Fixing a menu name; repointing a link. |

A spec's URL includes its status folder, so a status move is MINOR, not MAJOR.
Cite specs by filename, as the [Specs index](specs/README.md#citing-a-spec)
describes, and a status move breaks nothing.

### Releasing a version

Every change that reaches `main` carries its version, in the same commit (or the
last commit of its pull request):

1. Add an entry at the top of this page: `## [x.y.z] — YYYY-MM-DD`, dated in Manila
   time. A pull request with several changes gets one entry, with the largest bump
   among them.
2. Set the same version in `mkdocs.yml`'s `copyright` line.

The deploy workflow refuses to publish when the two disagree.

An entry groups its lines under **Added**, **Changed**, **Removed** and **Fixed**.
**History** records facts taken out of the pages because they describe the past
rather than the system: renames, earlier versions, measurements of a design that
has been replaced.

---

## [2.68.0] — 2026-11-14

### Changed

- The schedule board carries the S.A.G.E. landscape logo, in white on its header bar and in its own colours
  at the top of every printed sheet. **Features:** [Schedule board](features/schedule-board.md#pdf-and-paper).
  **Technical:** [Schedule board](technical/schedule-board.md#print-pdf-export) (the two logo files, the
  `.bezel-logo` and `.print-logo` rules, the PDF button waiting for the logo).

## [2.67.0] — 2026-11-14

### Changed

- [Court Control from Control Center](specs/in-progress/control-center-court-control-spec.md) moves from `not-started/`
  to `in-progress/`: it is built and live, and the owner's real-workbook checks are left.

## [2.66.0] — 2026-11-14

### Changed

- Court control from Control Center is built, on the `court-control` branches, awaiting the owner's
  checks (K1–K6 of its spec), so the spec stays in `not-started/`. **Features:** [Control
  Center](features/control-center.md#set-match) describes Live Matches' **Set match** / **Change**
  court cell and its dialog. **Technical:** [Control Center](technical/control-center.md#court-control)
  (the `courtCell` option, the court dialog, `fixtureSetCourt`, the `courtConflict=1` fixture flag),
  [Site engine](technical/site-engine.md) (`domain/courts.js`, `views/court-dialog.js`,
  `css/court-dialog.css`, `withCourtCell`), [API](technical/api.md) (the eighteenth `/v3` route, the
  exceptions row, five new error codes) and [Sync pipeline](technical/sync-pipeline.md#court-control-writes).
  **Usage:** [Run the day](usage/run-the-day.md#the-court-rhythm) and [Rehearse](usage/rehearse.md).
  API 3.8.0.

## [2.65.0] — 2026-11-14

### Changed

- [Court Control from Control Center](specs/in-progress/control-center-court-control-spec.md) goes from an
  outline to a build spec with every decision settled, written for a no-context build. Operators only; on
  when the event's `scoreEntry` is; the court's own cell in Live Matches is the button (**✎ Set match** /
  **✎ Change**). `PUT /v3/…/courts/{court}/match`, in a new `src/courts/` folder, finds a court's row by
  the number in column B: the team and dual-meet workbooks start their courts at row 6, not 5. The
  console's side is a `courtCell` option of `views/live-matches.js`, a new `domain/courts.js` and
  `views/court-dialog.js`, so the Hub and scorer page are unchanged. The score dialog's "put the next
  match on this court" prompt is a later phase, outlined only.

## [2.64.0] — 2026-11-14

### Changed

- [Team names on scoresheets](specs/implemented/scoresheet-team-names-spec.md) moves from `in-progress/` to
  `implemented/`: the named sheets were used at Ladies' Dink on 7 November 2026. Its §8 docs are in:
  [Scoresheet Generator](features/scoresheet-generator.md) (a team event's sheets print team names),
  [Scoresheet pipeline](technical/scoresheet-pipeline.md) (`withTeamNames`, `hasTeamNames`, quoted
  fields only for named CSVs), [Printables](usage/printables.md) (print playoff sheets after seating) and
  [Site engine](technical/site-engine.md) (`domain/scoresheet-names.js`).
- [Angeles City Employees Sportsfest](specs/implemented/acso-employees-sportsfest-spec.md) moves from
  `in-progress/` to `implemented/`: the event ran on 13 November 2026, every match played and the
  facility stamped complete. Its status line records what was built and what is not confirmed (the
  venue's Hub board print).
- [Code and access protection](specs/implemented/code-and-access-protection-spec.md) moves from
  `in-progress/` to `implemented/`, as its own status line said it would once an event built after
  §9.6–§9.9 ran with a Restricted workbook: the Angeles City Employees Sportsfest did. The owner's tasks
  of §9.4 and the settings of §2, §5, §6 and §7 stay open and are listed in its status line.
- [Double round robin, best-of-3 round robin and one-bracket finals](specs/in-progress/double-round-robin-and-finals-spec.md):
  the Standard Tournament Master runs Standard Tournament Generator 1.5.0, and the UPAP 3rd Annual
  Tournament's workbook was generated from it. The owner's scored-workbook checks and the dry run are left.

## [2.63.0] — 2026-11-14

### Changed

- [Sync pipeline § Score entry](technical/sync-pipeline.md): a score save reads `SCHEDULE` as values and
  as formulas at once, and writes the "who entered it" note alongside the publish instead of after it
  (API 3.7.1).

## [2.62.0] — 2026-11-13

### Changed

- [Tournament Hub](features/tournament-hub.md): Full tree's cards are wider than the chart's, with each
  player of a pair on a line of their own, so full names show. [Site engine](technical/site-engine.md)
  names the `fullTree` option and `FULL_TREE_DIMS`.
- [Control Center](features/control-center.md): on an exported podium image a long pair of names goes onto
  two lines, in smaller type if it has to, instead of running off the picture;
  [its technical page](technical/control-center.md) describes `fitMedalNameLines`.

## [2.61.0] — 2026-11-12

### Changed

- [Calculator team format](specs/not-started/calculator-team-format-spec.md) is revised against the
  calculator as it is now, the two team events since it was written (the Millennium Invitational and
  Ladies' Dink), and a draft plan for a two-day team event on 5–6 December. The playoff starts at a
  chosen round, from a Round of 16 down to Bronze + Final straight from the standings, and always
  ends in a Bronze and a Final. A tiebreaker game, in the playoffs only or in every matchup, adds a
  slot to each matchup or round that has one. The break before the playoffs is a checkbox, ticked at
  25 minutes. **Matches on court at once** gains a hint that matches sharing a player can't share a
  slot, and a plan whose matchups leave courts idle (3 matches 2 at a time, or a group-stage
  tiebreaker) gets an alert that the group stage could run shorter than the model's estimate. Its
  worked examples reproduce all three team events' finishes (4:05, 9:00 and 9:40 PM) and the
  December draft's Choice and Select divisions (7:50 and 2:50 PM); its Prime division rests teams
  between matchups, which the model leaves to the break. The owner answered all ten of its
  questions. It is written to be built with no other context: the calculator's 29 edits by anchor,
  a nine-case unit test of `calcTeam` against those examples, the docs changes and the browser
  checks, all dry-run on a copy of the calculator.

### Fixed

- The same spec said a team plan imported by an older copy of the calculator reports "no categories
  found". It imports as a standard plan and keeps the categories on screen.
- The same spec reused the after-10 PM alert unchanged, whose *one more court* figure is the
  standard format's arithmetic. For a team plan it now suggests the courts of one more lane, worked
  out with the team model.

## [2.60.0] — 2026-11-12

### Changed

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md) moves
  from **In progress** to **Implemented**: the event ran on 7 November 2026 and the facility was
  stamped complete; its pages are archived with live push off. The dry run and the workbook's
  Restricted sharing stay unconfirmed in its status.
- [After the event](usage/after-the-event.md) gains **Archive the event's pages, later**: moving the
  folder breaks the old address, so the short link the QR code opens is pointed at the new one the
  same day. Live push off now names the desk page too.

### Fixed

- [Control Center](technical/control-center.md) describes the console as built on the site engine.
  Its Standings, Match Finder, Teams tab, facility progress, Awards and team sections named functions
  the console no longer has; they now name the engine's (`createStandings`, `createFinder`,
  `createTeams`, `stageTablesHTML`, `buildPodiums(model)` and the rest) and say what stays in the
  page. The tab row has up to seven tabs, Awards is the second, and the page loads `qrcode-generator`
  from cdnjs for **Show QR**.
- [Event registry schema](technical/event-data-config.md): `type` is read by every engine page and,
  for rosters, by `sage-tools-api`; `archived` also turns score entry off in the API; `date` drives
  go-live, progress and both link kinds; facility workbooks are Restricted, so a published sheet ID
  opens nothing.
- [API](technical/api.md): thirteen of the seventeen `/v3` routes have a frozen twin, not sixteen; the
  people list takes a desk token too, as [Auth](technical/auth.md) now says.
- [Sync pipeline](technical/sync-pipeline.md): the site's copy of the completion rules is the engine's
  `lib/v1/domain/` with its parity test; a day's trigger runtime matches the measurements (50–60
  minutes, 105 at PickleDrive); the SAGE menu lists **Open Bracket Generator**; score entry's own
  `SheetsClient` and the lazy scoresheet import ([Scoresheet pipeline](technical/scoresheet-pipeline.md))
  are in `src/app.mjs`; `live` diagnostics carry `switch` and `active` (also
  [Deployment](technical/deployment.md)).
- [Architecture](technical/architecture.md) names `sage-docs` and the `AttendanceSignals` port.
- [Site engine](technical/site-engine.md)'s layout lists every module.
- [Schedule board](technical/schedule-board.md): a card's fourth state, **not needed**, and where its
  stage regexes live.
- Finished events' paths point at `events/archives/` in [Adding a new
  event](technical/adding-a-new-event.md), [Event attendance](technical/event-attendance.md) and the
  [implemented specs' index](specs/implemented/README.md).
- The changelog's links to the Ladies' Dink spec point at `implemented/`.

---

## [2.59.0] — 2026-11-12

### Added

- [Ranking rules](specs/not-started/ranking-rules-spec.md), a not-started spec: an event's
  standings ordered by a list of named rules in `events.json`, with blocks by group size and an
  optional restart, applied to each tied group in turn. The standard, `"points"` and `"games"`
  rankers become presets with parity tests. Display only. Its decisions are taken; it has no
  build date.

---

## [2.58.0] — 2026-11-12

### Added

- The [event spec template](specs/event-spec-template.md) gains §1.3, **several facilities, or
  several days**: each facility's workbook, match-number range and sync setup, and the board's one
  day, with the owner choosing between pushing it each evening and an engine phase that lets it
  pick its day. It also gains **Part 3**, the engine phase's skeleton to insert before the board:
  tests first, the API's half when the played or BYE rules change, the documentation, and the
  pushes, API first.

### Changed

- The event spec template's Phases 2 and 3 carry every command written out, as Phase 1 does. The
  line-endings table is replaced by `git ls-files --eol`, read when the spec is written. The
  contrast checks give each pair its WCAG line (4.5:1 body, 3:1 large text) and what a shortfall
  means. Part 1's §1.3–§1.7 become §1.4–§1.8.

---

## [2.57.0] — 2026-11-12

### Added

- The [event spec template](specs/event-spec-template.md), at the root of Specs: one template
  for every event type. Part 1 says how to write an event spec: gather the workbook, the pubmat
  and the owner's answers first; build in phases (the board with `events.json`, the Hub, the rest,
  and an engine phase first when one is needed); anchor every edit on one unique line, state each
  file's line endings and never use `sed -i`; check the Hub at 1024px as well as 375 and 1280; and
  dry-run the spec in a detached worktree. Part 2 is the skeleton, in the UPAP spec's shape, with
  Standard, Team and Dual meet blocks. The [specs index](specs/README.md#writing-an-event-spec)
  points to it.

---

## [2.56.0] — 2026-11-12

### Changed

- The [UPAP 3rd Annual Tournament spec](specs/in-progress/upap-3rd-annual-tournament-spec.md)
  moves from `not-started/` to `in-progress/`: all three phases are on the site's `main` and the
  workbook syncs. Its status line lists what is left (the QR image, the workbook checks, the dry
  run) and an open finding: the Hub's QR panel covers the right end of the title between 900 and
  about 1120px wide.

---

## [2.55.1] — 2026-11-12

### Fixed

- The [UPAP 3rd Annual Tournament spec](specs/in-progress/upap-3rd-annual-tournament-spec.md)'s
  §5.4(a) gives the theme comment it replaces as 19 lines, not 17. Its anchors and §5.5's line
  count were already right. §0.2 notes that the standard template's `index.html` is LF on disk,
  unlike the rest of the checkout.

## [2.55.0] — 2026-11-11

### Added

- [UPAP 3rd Annual Tournament](specs/in-progress/upap-3rd-annual-tournament-spec.md), in
  **Not started**: the site for the UP Alumni Pickleball Community's tournament on 22 November
  2026 at The Foundry, Cainta, eight one-bracket categories by age group and DUPR level on 4
  courts, built in three phases: the schedule board with the `events.json` entry, then the
  Tournament Hub, then the desk page and checklist. It instantiates the standard template,
  re-skins it from the pubmat (plum structure, the ball's green accent, the UPAP mark's colours
  and Comfortaa in the hero), sets the board's category colours, registers the event with five
  divisions, and lists the workbook checks.

### Changed

- The [double round robin spec](specs/in-progress/double-round-robin-and-finals-spec.md)'s D6
  names its first event, the UPAP 3rd Annual Tournament, and the in-progress table says so.

## [2.54.1] — 2026-11-10

### Fixed

- The [double round robin, best-of-3 and one-bracket finals spec](specs/in-progress/double-round-robin-and-finals-spec.md),
  the [Specs index](specs/README.md) and the in-progress folder's table say the work is built
  and live on `main`, not waiting on its branch, and list what is left: pasting the generator
  into the Standard Tournament Master, the real-workbook checks and the dry run.

## [2.54.0] — 2026-11-10

### Added

- [Tournament Calculator](features/tournament-calculator.md#how-the-round-robin-is-played),
  [Standard Tournament Generator](features/standard-tournament-generator.md#double-and-best-of-3-round-robins)
  and their technical pages describe the double round robin, best-of-3 round-robin meetings
  and the one-bracket `final` and `final_bronze` playoffs, with the `rr_legs` and `rr_best_of`
  plan columns, the `CSV` tab's score lookup by match number and its `bestOf` column, and the
  series W and L formulas.
- [Named Functions](technical/named-function-library.md#called-from-generated-formulas) says where
  the generator calls the column primitives from cells.
- The glossary gains *best-of-3 meeting*, *double round robin* and *final and bronze*.

### Changed

- [Sync pipeline](technical/sync-pipeline.md#facility-completion) and the
  [site engine](technical/site-engine.md) describe a best-of-3 round-robin meeting's game 3 after
  a 2–0 as never played; the [Hub](features/tournament-hub.md) and
  [schedule board](features/schedule-board.md) pages say it is greyed.
- [Plan the event](usage/plan-the-event.md) lists the round-robin choice and the four one-bracket
  playoffs.
- The [Standard Tournament Master spec](specs/implemented/standard-tournament-master-spec.md) lists
  the two new shapes (§5.5), their codes (§8), the new plan columns (§4) and validation (§9), and
  the `CSV` tab's score lookup and `bestOf` column (§10.5).
- [The double round robin spec](specs/in-progress/double-round-robin-and-finals-spec.md) moved from
  `not-started/` to `in-progress/`: built on a branch, not yet merged.

## [2.53.0] — 2026-11-10

### Changed

- [Double round robin, best-of-3 round robin and one-bracket finals](specs/in-progress/double-round-robin-and-finals-spec.md)
  (not started) adds best-of-3 round-robin meetings (`rr_best_of`) in any standard category,
  with the usual playoff after them: W and L count meetings won, the quotient counts every
  game's points, and game 3 after a 2–0 is not a match left. The plan covers the calculator,
  the Standard Tournament Generator 1.5.0 (series W and L formulas and a `bestOf` column on
  `CSV`), a compatible change in the site's `lib/v1/domain/`, and `sage-tools-api` 3.7.0's
  completion rule.

---

## [2.52.0] — 2026-11-10

### Changed

- [Standard Tournament Generator](features/standard-tournament-generator.md) 1.4.1: four brackets,
  top 2, cross bracket 1 with 2 and 3 with 4, a bracket's #1 and #2 still on opposite halves
  (QF-1 Br 1 #1 v Br 2 #2, QF-2 Br 3 #1 v Br 4 #2, QF-3 Br 2 #1 v Br 1 #2, QF-4 Br 4 #1 v Br 3 #2).
  Also in its [technical page](technical/standard-tournament-generator.md) and the
  [master spec](specs/implemented/standard-tournament-master-spec.md)'s §18.

### History

- Generator 1.4.0 crossed bracket 1 with 3 and 2 with 4.

---

## [2.51.0] — 2026-11-10

### Added

- [Double round robin and one-bracket finals](specs/in-progress/double-round-robin-and-finals-spec.md)
  (not started): a double round robin for a one-bracket standard category, and two more
  one-bracket playoffs, a one-game Final with a played 3 v 4 Bronze and a one-game Final with
  RR #3 taking bronze, in the Tournament Calculator and the Standard Tournament Generator
  1.5.0. Written to be built without other context.

---

## [2.50.1] — 2026-11-08

### Fixed

- [Adding a new event](technical/adding-a-new-event.md): archiving an event ends its old address, so its
  short link has to be pointed at the new one. The site's 404 page doesn't redirect.

---

## [2.50.0] — 2026-11-08

### Changed

- [Adding a new event](technical/adding-a-new-event.md) and
  [Event data config](technical/event-data-config.md): archiving an event moves its folder to
  `events/archives/` and sets `pagesArchived`.
- [Control Center](technical/control-center.md): the scorer-page check follows `pagesArchived`.
- [Site engine](technical/site-engine.md): lists `domain/event-pages`.
- [Event attendance](features/event-attendance.md): Pickle for Sight's page address.

---

## [2.49.0] — 2026-11-07

### Changed

- [Team names on scoresheets](specs/implemented/scoresheet-team-names-spec.md) moves from Not started to In
  progress: it is built and live, with its live-page checks and docs left.
- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md): the status line
  says what is live and what is left on the day of the event.
- [Specs index](specs/README.md): the Team tournament entry links its template, which is built.

---

## [2.48.0] — 2026-11-07

### Changed

- [Team Tournament Master](specs/not-started/team-tournament-master-spec.md): the Show lineup column, the
  `lineupShown` formula and spare rows, from the Millennium Invitational and Ladies' Dink.

---

## [2.47.0] — 2026-11-07

### Changed

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md): the score dialog
  warns on a playoff tiebreaker short of 11, public pages hold a lineup until it is ticked, and Control Center
  warns when a player plays both of a matchup's games.
- [Control Center](features/control-center.md) and [Event data config](technical/event-data-config.md): a
  tiebreaker's pair may repeat a player.
- [Team workbook](features/team-workbook.md): a spare row with `-` as its team code isn't a match.

---

## [2.46.0] — 2026-11-07

### Changed

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md),
  [Team workbook](features/team-workbook.md), [Tournament Hub](features/tournament-hub.md) and
  [Site engine](technical/site-engine.md): a bracket tie's game can be entered as a `_TB` row, which settles
  the tie and counts for nothing else.

---

## [2.45.0] — 2026-11-07

### Changed

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md): two teams level on
  wins at 1–1 head-to-head go straight to a side-out game to 5, with no quotient.
- [Event data config](technical/event-data-config.md), [Site engine](technical/site-engine.md) and
  [Tournament Hub](features/tournament-hub.md): an event's `headToHeadQuotient: false`.

---

## [2.44.0] — 2026-11-06

### Changed

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md): the organiser's
  revised tiebreaker process. Two teams still level play one side-out game to 5; three or more play a round
  robin to 5, then first to 3, then a timed match; a playoff tiebreaker is to 11, with the pair submitted in a
  3-pair lineup.

---

## [2.43.1] — 2026-11-06

### Changed

- [Team names on scoresheets](specs/implemented/scoresheet-team-names-spec.md): rewritten so it can be
  built with no other context, with the exact code, tests, checks and doc text, all run on a throwaway
  copy of the repos.

---

## [2.43.0] — 2026-11-06

### Added

- [Team names on scoresheets](specs/implemented/scoresheet-team-names-spec.md): a new spec, not
  started, for printing each team's name beside its pair code on a team event's scoresheets.

---

## [2.42.0] — 2026-11-06

### Changed

- [Build the workbooks](usage/build-the-workbooks.md#team-the-pickledrive-copy): a copied team
  workbook gets the current sync script pasted in before it is connected, since it keeps the source
  workbook's otherwise.
- [Troubleshooting](usage/troubleshooting.md): **Sync now** showing raw JSON, and what to do.

---

## [2.41.0] — 2026-11-01

### Changed

- [Site engine](technical/site-engine.md) (the theme contract) and
  [Adding a new event](technical/adding-a-new-event.md): the standard and dual-meet templates' Hub
  and board, and the scorer template, restate the engine's house-coloured rules through the page's
  tokens too, so any template re-skins completely from its `:root` block.

---

## [2.40.0] — 2026-11-01

### Changed

- [Site engine](technical/site-engine.md) (the theme contract) and
  [Adding a new event](technical/adding-a-new-event.md): the team template's pages restate the
  engine's house-coloured tints and shadows through the page's tokens, so a re-skinned team event
  shows its own colours on a winning side, a live row and the dividers.
- [Adding a new event](technical/adding-a-new-event.md) and
  [Build the workbooks](usage/build-the-workbooks.md#team-the-pickledrive-copy): a team event of
  another shape copies the nearest past event's workbook and is adapted by hand, with the checks
  to run before connecting it.
- [Rehearse](usage/rehearse.md) and [Run the day](usage/run-the-day.md): the team checks for lineup
  reveal, team colours and roster edits (neither a **Show lineup** tick nor a `Teams` edit syncs by
  itself), and the playoff tiebreaker row on the day.

---

## [2.39.0] — 2026-11-01

### Changed

- [Control Center](technical/control-center.md): a team event's Standings keeps to the page's column
  on a desktop screen, with its brackets two across.
- The [Ladies' Dink spec](specs/implemented/ladies-dink-club-tournament-spec.md) and the in-progress
  and specs indexes: the logo, the Standings margins and the score dialog's tiebreaker warnings (a
  tiebreaker is to 7) are built; the two open questions are answered: the organiser decides a
  three-team tie on the day (Q10), and first serve is 3:30 PM (Q11).

---

## [2.38.0] — 2026-10-31

### Changed

- [Site engine](technical/site-engine.md), [Sync pipeline](technical/sync-pipeline.md),
  [Tournament Hub](features/tournament-hub.md), [Schedule board](features/schedule-board.md) and
  [Team workbook](features/team-workbook.md): a team playoff matchup's tiebreaker game (`_TB`, shown
  as TB, not counted once the matchup is settled), lettered brackets (`Bracket A`), and the Tied
  pill's generic wording (*Tied · playoff game* / *Tied · playoff*).
- The [Ladies' Dink spec](specs/implemented/ladies-dink-club-tournament-spec.md) moves from
  `not-started/` to `in-progress/`: its engine changes, pages and `events.json` entry are pushed.

---

## [2.37.0] — 2026-10-31

### Added

- [Ladies' Dink 1st Club Tournament](specs/implemented/ladies-dink-club-tournament-spec.md), not
  started: the site for a 16-team women's team event on 7 November 2026 at the Pampanga Pickleball
  Center, and the three compatible engine changes it needs first: a playoff matchup's tiebreaker game
  (`_TB` codes, to 7) that isn't counted once the matchup's other games settle it, in the site and the
  API's completion stamp; lettered brackets; and a Tied pill whose wording fits any event. Every edit is
  an exact Find/Replace, dry-run on copies of the site and the API, with a fixture script for the
  browser checks. The owner's answers are in §15; two questions are still open, neither blocking the
  build: what settles three teams level on wins and quotient, and whether first serve is 3:00 or
  3:30 PM. §14 records the event workbook's state and what is left to fix in it.

---

## [2.36.0] — 2026-10-26

### Added

- [Lean deploy image](specs/not-started/lean-deploy-image-spec.md), not started: a hand-built
  `sage-tools-api-base` image with Node and Chromium that the API's `Dockerfile` builds `FROM`, so a
  deploy stores about 20 MB in Artifact Registry instead of 361 MB, with optional `puppeteer-core`. It
  records the repository's cleanup policy, which keeps the last deploy.

### Fixed

- [Code and access protection](specs/implemented/code-and-access-protection-spec.md) §3 describes
  the cost controls in place: Cloud Run's maximum of 10 instances and the `cost-alert` budget ($1.00 a
  month, alerts at 50%, 90% and 100%, no spend cap). It said neither existed.

---

## [2.35.0] — 2026-10-26

### Changed

- [Millennium Invitational](specs/implemented/millennium-invitational-spec.md) moves from
  **In progress** to **Implemented**: the event ran on 24 October 2026, all 120 matches were published
  and the facility was stamped complete. Its status records the day's re-skin from the newer pubmat,
  pair 1 as Men's Doubles, and the pages' live push off.
- [Playoff bracket chart](specs/implemented/standings-bracket-chart-spec.md) moves from **In progress**
  to **Implemented**: the Standard Tournament Master carries generator 1.4.0, and the Sportsfest and
  Piggleball workbooks publish their bracket links.

### Fixed

- The changelog's links to both specs, and the Standard Tournament Master spec's links to the bracket
  chart spec, point at `implemented/`. One earlier entry still linked to the bracket chart spec's
  first folder, `not-started/`.

---

## [2.34.0] — 2026-10-26

### Added

- Spec [Hub player experience](specs/not-started/hub-player-experience-spec.md): proposals P9–P17
  from a comparison with Dinktator's spectator pages (§1.2): a tab's own address, Standings opening
  on a category on phones, *Final* once a day is over, a podium on a finished category, the row
  colours and ties explained, a Standings row opening its pair's matches, and for a dual meet the
  club score on the first screen, *Pairs* for *Teams*, category jump chips and a cross-club grid.
  Decisions D8–D14, and what is not taken from Dinktator (§21.2).
- Spec [Hub player experience](specs/not-started/hub-player-experience-spec.md): a picture for every
  proposal that changes the page, today beside a mockup, drawn on the engine's template pages and
  the harness's fixture data (`images/hub-player-experience/`).

### Fixed

- Spec [Hub player experience](specs/not-started/hub-player-experience-spec.md): a standings row is
  pink at a level record too, not only a losing one. P13's tie note follows the Hub's real order,
  head-to-head wins among all the pairs level and then quotient.

---

## [2.33.0] — 2026-10-24

### Changed

- Match Finder (Hub and Control Center): a team event's matchup cards carry a **Hide matches** button.
- Control Center: a team event's matchup cards on Standings open the score dialog, as in Match Finder.

---

## [2.32.0] — 2026-10-24

### Changed

- Tournament Hub: a team matchup reads *In progress* from the moment one of its matches is on court,
  not only once a score is in.

---

## [2.31.0] — 2026-10-24

### Changed

- Millennium Invitational spec: pair 1 is Men's Doubles (any two men, of any level), the organiser's
  rule of 24 October, labelled "MD"; `advancedLevels` is empty, and the Lineup check's "Men's High"
  wording on pair 1 is expected.

---

## [2.30.0] — 2026-10-22

### Changed

- The [Standard Tournament Generator](features/standard-tournament-generator.md) 1.4.0 seats three and four
  brackets, top 2, by the organisers' rules (the qualifier scaffold's slots come written in), and writes a
  `STANDINGSCSV` `seat` column, each empty round-robin seat's label (**Bracket 1 · #1**, **Best runner-up**),
  which the bracket chart shows before `display.playoffSeats`. **Add bracket links** adds it to older
  workbooks. [The generator](technical/standard-tournament-generator.md),
  [the site engine](technical/site-engine.md), [the config](technical/event-data-config.md), the
  [Standard Tournament master spec](specs/implemented/standard-tournament-master-spec.md) (§10.1, §10.4, §18)
  and the [bracket chart spec](specs/implemented/standings-bracket-chart-spec.md) §14 say so.
- The bracket chart on the [Tournament Hub](features/tournament-hub.md): on a computer a category's column
  keeps its **1 col** / **2 cols** width in List and Bracket, and shows one or two rounds at a time to match;
  the swipe window always settles on whole rounds, its cards stretched to the card's width; **Round Robin**
  and **Playoffs** are titled sections, with the List | Bracket switch at the end of the Playoffs title; and
  **Full tree** fills a computer's screen, enlarging the chart. [The site engine](technical/site-engine.md)
  and the [bracket chart spec](specs/implemented/standings-bracket-chart-spec.md) §14 say so.

---

## [2.29.0] — 2026-10-22

### Changed

- The [playoff bracket chart](specs/implemented/standings-bracket-chart-spec.md) keeps its lines on a phone,
  in portrait too: a chart too wide for its card is the same chart in a window that scrolls sideways, under
  chips that jump to a round, instead of a strip of rounds without lines. **Full tree** is under the chart on
  every screen, tablets included. [Tournament Hub](features/tournament-hub.md),
  [Control Center](features/control-center.md) and [the site engine](technical/site-engine.md) say so.

---

## [2.28.0] — 2026-10-21

### Added

- The playoff bracket chart in the docs: [Tournament Hub](features/tournament-hub.md) and
  [Control Center](features/control-center.md) (the List | Bracket switch, empty-seat text, the round strip and
  Full tree, click-to-score), the [Standard Tournament Generator](features/standard-tournament-generator.md)
  (the `STANDINGSCSV` `match` and `from` columns and **SAGE → Add bracket links**), and, on the technical side,
  [the site engine](technical/site-engine.md#the-bracket-chart), [the generator](technical/standard-tournament-generator.md#bracket-links),
  [Control Center](technical/control-center.md) and the `display.playoffSeats` setting in
  [the config](technical/event-data-config.md).

### Changed

- [Playoff bracket chart](specs/implemented/standings-bracket-chart-spec.md) moves from **Not started** to
  **In progress**: built and tested on a branch of each repo, not yet merged. Its §14 lists where the build
  differs from the text.
- The [Standard Tournament master spec](specs/implemented/standard-tournament-master-spec.md) §10.1, §10.4 and
  §18 record the link table and the two new columns; [Entering a score](features/score-entry.md) says a
  bracket card on Standings opens the score dialog.

---

## [2.27.0] — 2026-10-21

### Changed

- [Angeles City Employees Sportsfest](specs/implemented/acso-employees-sportsfest-spec.md) moves from
  **Not started** to **In progress**: the event's pages and its `events.json` entry are pushed, and the
  workbook syncs all 96 matches with the players' names.
- [Millennium Invitational](specs/implemented/millennium-invitational-spec.md)'s status: the workbook
  syncs, so sync setup is no longer left.

---

## [2.26.0] — 2026-10-21

### Added

- [Playoff bracket chart](specs/implemented/standings-bracket-chart-spec.md), in **Not started**: a
  List / Bracket switch on each standard category with two or more playoff rounds, on the Tournament
  Hub and in Control Center. The rounds sit side by side with lines from each match to the next; a
  phone shows one round at a time and a zoomable Full tree. The Standard Tournament Generator 1.3.0
  (or its Add bracket links menu, for older workbooks) writes the ladder's shape into two new
  `STANDINGSCSV` columns, an event may label empty seats in `display.playoffSeats`, and each
  qualifier is tagged with its bracket and finish once the sheet names it.

---

## [2.25.0] — 2026-10-21

### Added

- [Angeles City Employees Sportsfest](specs/implemented/acso-employees-sportsfest-spec.md), in
  **Not started**: the site for the Angeles City Sports Office's employees' pickleball tournament on
  13 November 2026, three categories on 8 courts at Pickle Place. It instantiates the standard
  template in the house theme with the organiser's logo and category colours, registers the event
  with team codes that have no division (`"divisions": { "": "" }`), and lists the workbook fixes
  still owed.

---

## [2.24.0] — 2026-10-21

### Changed

- Control Center's lineup badge names the pairs with issues (**⚠ 7 lineup issues · MD, WD, XD**), and each
  of those pairs' match lines is tinted with a red bar and pair chip:
  [Control Center](features/control-center.md) and [Control Center, technical](technical/control-center.md).

---

## [2.23.0] — 2026-10-20

### Changed

- A team event's schedule board shows players larger than team names, and a card whose lineup isn't
  set keeps the height it will have once it is, so the board doesn't resize as lineups come in:
  [the schedule board](features/schedule-board.md) and [its technical page](technical/schedule-board.md).

---

## [2.22.0] — 2026-10-20

### Changed

- Control Center shows each lineup warning on its match in every view, the Standings tab's bracket
  matchups and playoff cards included, and a matchup card with lineup issues stands out with a red edge
  and header: [Control Center](features/control-center.md),
  [Control Center, technical](technical/control-center.md) and [the site engine](technical/site-engine.md).

---

## [2.21.0] — 2026-10-20

### Changed

- A bracket of an event scored on games won (`teamScoring: "games"`) is ranked by the Millennium
  Invitational's revised tiebreakers: two teams level on wins by head-to-head wins, then the
  head-to-head quotient; three or more by their quotient, two still level going to the two-team rules.
  A tie only a playoff can settle reads **Tied · playoff game** or **Tied · playoff round robin** once
  the round robin is over: [Tournament Hub](features/tournament-hub.md),
  [Control Center](features/control-center.md), [events.json](technical/event-data-config.md),
  [the site engine](technical/site-engine.md) and the
  [Millennium Invitational](specs/implemented/millennium-invitational-spec.md) spec.

---

## [2.20.0] — 2026-10-20

### Added

- A team event can hold its lineups back (`lineupReveal`): the Hub and the board show a match's players
  only once the published `CSV` tab's `lineupShown` column is `TRUE` for it (at the Millennium
  Invitational, a **Show lineup** tick box per matchup in `MatchUps`), and Control Center marks each such matchup
  **Lineup hidden**. Each Control Center matchup card also carries a **⚠ N lineup issues** badge:
  [Tournament Hub](features/tournament-hub.md), [Control Center](features/control-center.md),
  [Control Center, technical](technical/control-center.md), [events.json](technical/event-data-config.md),
  [the site engine](technical/site-engine.md) and the
  [Millennium Invitational](specs/implemented/millennium-invitational-spec.md) spec.

---

## [2.19.0] — 2026-10-20

### Changed

- A team event scored on games won (`teamScoring: "games"`) shows PF, PA and the quotient beside W and L
  in its bracket table, and each matchup card gives each side's PF, PA and quotient in the matchup:
  [Tournament Hub](features/tournament-hub.md), [Control Center](features/control-center.md) and
  [the site engine](technical/site-engine.md).
- A team event can name its playoff placeholders (`display.seedLabels`): the Millennium Invitational's
  Final reads *Rank 1 · TBD* v *Rank 2 · TBD* and its Bronze *Rank 3 · TBD* v *Rank 4 · TBD*:
  [Tournament Hub](features/tournament-hub.md) and [events.json](technical/event-data-config.md).

---

## [2.18.0] — 2026-10-20

### Added

- A team event's **Lineup check** in Control Center: `lineupRules` in `events.json` gives each pair
  a kind (Men's High, Mixed, Women's, Genderless), the levels that count as advanced and whether a
  captain may play Mixed, and every lineup that breaks one, or plays someone twice in a round, is
  flagged for operators: [Control Center](features/control-center.md#lineup-check),
  [Control Center, technical](technical/control-center.md), [events.json](technical/event-data-config.md),
  [the site engine](technical/site-engine.md) and the
  [Millennium Invitational](specs/implemented/millennium-invitational-spec.md) spec's day-of section.

---

## [2.17.0] — 2026-10-19

### Added

- A team event's colours: `display.teamColors` in `events.json` puts a mark (the team's colour on a
  shape of its own) before each team name and shades each matchup card between the two teams'
  colours, on the Hub, the schedule board and Control Center:
  [events.json](technical/event-data-config.md), [the site engine](technical/site-engine.md),
  [the schedule board, technical](technical/schedule-board.md),
  [Tournament Hub](features/tournament-hub.md), [the schedule board](features/schedule-board.md)
  and the usage guide's register-and-build page.

---

## [2.16.0] — 2026-10-19

### Changed

- **Issue desk link** and **Issue scorer link** open a dialog that asks for the optional
  **Issued to** and **Note**, instead of two fields that stay on the panel:
  [Who a link was issued to](features/control-center.md#who-a-link-was-issued-to),
  [Control Center, technical](technical/control-center.md) and the usage guide's
  [run-the-day](usage/run-the-day.md) page.

---

## [2.15.0] — 2026-10-18

### Changed

- [Millennium Invitational](specs/implemented/millennium-invitational-spec.md) moves from
  **Not started** to **In progress**: the engine's `teamScoring: "games"`, the event's pages and
  its `events.json` entry are built and pushed.
- The team pages describe a matchup as won on total points, or, for an event whose
  `teamScoring` is `"games"`, on games won and then total points, with the bracket ranked by games
  won, total points, head-to-head wins and head-to-head points and shown as W, L and Pts:
  [Tournament Hub](features/tournament-hub.md), [Control Center](features/control-center.md),
  [the team workbook](features/team-workbook.md), [Control Center, technical](technical/control-center.md),
  [the site engine](technical/site-engine.md), [events.json](technical/event-data-config.md) and the
  usage guide's glossary, overview and run-the-day pages.

---

## [2.14.0] — 2026-10-18

### Added

- [Millennium Invitational](specs/implemented/millennium-invitational-spec.md), in **Not started**:
  the site for Millennium Pickleball Club's 2nd Invitational, a charity team tournament on
  24 October 2026. It adds an opt-in `teamScoring: "games"` to the engine's team rules (a matchup
  won on games won; a bracket ranked by games won, total points, then head-to-head wins and points), instantiates
  the team template with the club's colours, and lists the workbook fixes still owed.

---

## [2.13.1] — 2026-10-18

### Fixed

- [Code and access protection](specs/implemented/code-and-access-protection-spec.md), its row in
  the in-progress index and the specs index: the releases and the site it built are deployed
  (`sage-tools-api` 3.4.0 contains 3.1.1–3.3.0), so deploying them is no longer listed as left.

---

## [2.13.0] — 2026-10-18

### Changed

- [Push for attendance](specs/implemented/attendance-push-spec.md) moves from **Not started** to
  **Implemented**: `sage-tools-api` 3.4.0 and the site merged on 2026-10-13, and the Worker
  deployed on 2026-10-18. Its status line records what was built, two as-built details, and that
  the owner's checks P1–P7 had not run.

---

## [2.12.0] — 2026-10-13

### Changed

- [Event attendance](technical/event-attendance.md#push) (technical): the new **Push**
  section describes the data-free nudge, the Worker's separate attendance object, when the API
  nudges, and that nothing polls; the flow diagram, the list's cache and read-load paragraphs,
  and the client paragraph follow it. It replaces the note that the live Worker is not used for
  attendance.
- [Sync pipeline](technical/sync-pipeline.md#the-worker), [API](technical/api.md),
  [Control Center](technical/control-center.md#attendance-tab): the Worker also relays
  attendance nudges on a separate object per day; a write that changes the tab nudges it; the
  Attendance tab's attendance signal and when it is closed.
- [Event attendance](features/event-attendance.md), [Desk handout](usage/desk-handout.md):
  marks made on other devices appear within a second or two; if the page says *Live updates
  are off*, press **Refresh**.
- [Run the day](usage/run-the-day.md): the **Sync method** switch does not change attendance.

---

## [2.11.0] — 2026-10-13

### Changed

- [Push for attendance](specs/implemented/attendance-push-spec.md), by the owner's decision:
  a page's read on opening, on coming back into view, on reconnecting and on **Refresh** gets
  one follow-up read 11 s later, because it may have come from another instance's cache up to
  10 s old (D8, §5.2). The implementer commits to a feature branch instead of leaving the
  work uncommitted, one site commit per push (§0.3, §10.1).

### Fixed

- [Push for attendance](specs/implemented/attendance-push-spec.md), checked again against the
  code so it can be built without other context. §0.4 gains the facts the work depends on: how
  `LivePublisher` labels its errors, that `#reconcileFacility` isn't given the event, how the
  attendance tests build the service, where Control Center's Attendance tab mounts and tears
  down, the site's import rules, and the harness closing every live socket. §3 names `smoke.mjs`'s
  one combined key and numbers the new checks. §4 says how `nudge` reuses `#request`, where the
  back-off time is kept, and adds a wiring test. §5 spells out the per-facility load sequence,
  where the *Live updates are off* line goes, what closes Control Center's signal, and which
  harness cases to accept. §6 says what the local checks need, and to report rather than
  improvise when there is no Google access. §7 names the right bullet in *Run the day*.

## [2.10.0] — 2026-10-13

### Changed

- Specs: [sage-tools-api architecture hardening](specs/implemented/sage-tools-api-architecture-spec.md)
  and [Usage guide and docs restructure](specs/implemented/docs-usage-guide-spec.md) move from
  In progress to **Implemented**, by the owner's decision. Their status lines name the checks
  not run: the fresh-master check, the production acceptance and the Cloud Build filter; and
  the newcomer read-through.

## [2.9.1] — 2026-10-12

### Fixed

- [sage-tools-api architecture hardening](specs/implemented/sage-tools-api-architecture-spec.md):
  the workspace is a GitHub repo, `sage-match-control/sage-workspace`. Phase 7 has only the
  build-trigger filter left.
- [Usage guide and docs restructure](specs/implemented/docs-usage-guide-spec.md): D5's
  site-repo fixes are done (the dry-run checklist template's labels and scorer links, and
  `_templates/CLAUDE.md` step 13). Only the newcomer read-through is left.

## [2.9.0] — 2026-10-12

### Added

- Spec: [Push for attendance](specs/implemented/attendance-push-spec.md). A check-in on one
  desk reaches the other desks and Control Center within a second or two: the live Worker
  relays a nudge that carries no data, on its own object per day, and each page re-reads
  the list from the API. Nothing polls; **Refresh** stays. Not started.

## [2.8.0] — 2026-10-12

A facility workbook is Restricted and read by the API's service account, and the
attendance list comes through the API.

### Added

- [Event attendance](technical/event-attendance.md#reading-the-list): the
  people list route (`GET /v3/days/{day}/facilities/{facility}/people`), its rules, its
  10-second cache, and [how the API reads a workbook](technical/event-attendance.md#reading-the-workbook).
- [API](technical/api.md): the route, as the seventeenth `/v3` route and the one with no
  frozen twin.
- A `LICENSE` file (all rights reserved) in this repo.

### Changed

- [Build the workbooks](usage/build-the-workbooks.md#file-it),
  [Connect the workbooks](usage/connect-the-workbooks.md),
  [Troubleshooting](usage/troubleshooting.md) and
  [Adding a new event](technical/adding-a-new-event.md): a facility workbook's General
  access is **Restricted** and the service account is Editor on every event's workbooks.
  The first check when a sync fails is the service account, not General access. The two
  masters keep *Anyone with the link: Viewer*, which **Make a copy** needs.
- [Sync pipeline](technical/sync-pipeline.md), [Deployment](technical/deployment.md) and
  [Architecture](technical/architecture.md): the fetchers read as the service account;
  `GOOGLE_SHEETS_API_KEY` is the fallback; the CSV export fallback needs a link-shared
  workbook.
- [Control Center](technical/control-center.md) and its
  [feature page](features/control-center.md), and
  [Event attendance](features/event-attendance.md): signed out, the Attendance tab asks to
  sign in instead of listing people.
- [Site engine spec](specs/implemented/site-engine-spec.md) §12: row 28, the signed-out
  Attendance tab.
- Spec: [Code and access protection](specs/implemented/code-and-access-protection-spec.md)
  moves from `not-started/` to `in-progress/`. The code, the runbook and these docs are
  built; the owner's checks, the deploy and the account settings are left. It gains §9.10,
  where the build departs from the design.

## [2.7.0] — 2026-10-12

### Changed

- Spec: [Code and access protection](specs/implemented/code-and-access-protection-spec.md)
  gains §9, facility workbooks closed to link access (the server reads as its service
  account, the desk page and Control Center read the roster through a new API route), and
  §10, two fixes to `facilityCompletion.mjs`: how it reads a score's value, and its stale
  comments. Both are full designs with the owner's decisions recorded, written to be built
  without other context. §6 now has the service account as Editor on every facility
  workbook, and signed out, Control Center's Attendance tab is to show a sign-in prompt.
  §3 (a Cloud Run cap) and §4 (sign-in rate limiting) are not done, by the owner's
  decision, and §5 keeps to what GitHub's free plan includes, with the paid options in
  §11. Still not started.

## [2.6.0] — 2026-10-11

### Added

- Spec: [Code and access protection](specs/implemented/code-and-access-protection-spec.md).
  A to-do list for protecting the code and the workbooks at little or no cost. Not started.

## [2.5.0] — 2026-10-10

### Added

- Spec: [Hub player experience](specs/not-started/hub-player-experience-spec.md). Eight
  independent proposals to make the Tournament Hub quicker on a phone, each for the owner
  to keep or drop. Not started.

## [2.4.0] — 2026-10-09

Desk and scorer links name who they were issued to, and the workbook records it.

### Added

- [Who a link was issued to](features/control-center.md#who-a-link-was-issued-to):
  the optional **Issued to** and **Note** fields when issuing a desk or scorer link.
- [Who marked someone in](features/event-attendance.md#who-marked-someone-in): the
  `markedBy` column of the `ATTENDANCE` tab.
- [Who entered a score](features/score-entry.md#who-entered-a-score): the note on both
  score cells of a match.
- [Technical](technical/auth.md#who-a-scoped-token-was-issued-to): the `to` and `note` keys
  of a scoped token, the optional body of the two `/v3` issue routes, how `markedByColumn`
  picks a column, and the one-cell widening of the attendance write allowlist.

### Changed

- The scorer and desk pages show the label after the day.
- [Run the day](usage/run-the-day.md): issue one link per person, with their name and gate or
  courts. The [scorer](usage/scorer-handout.md) and [desk](usage/desk-handout.md) handouts say
  the page shows who the link was issued to, and not to pass it on.
- Spec: [Link attribution](specs/implemented/link-attribution-spec.md) moves from Not started
  to **Implemented**.

## [2.3.0] — 2026-10-08

### Added

- Spec: [Link attribution](specs/implemented/link-attribution-spec.md). Desk and scorer
  links name who they were issued to, and the workbook records it on every mark and
  score they make. Not started.

## [2.2.1] — 2026-10-08

### Fixed

- The [scorer handout](usage/scorer-handout.md) and [desk handout](usage/desk-handout.md)
  stand alone, with no way to reach another page. They no longer end with links to the
  Features pages, and a page template, `overrides/handout.html`, drops the menu, tabs,
  search, repository and edit links, and the footer. A scorer or desk volunteer sees
  only the one page.

## [2.2.0] — 2026-10-08

The roles match how SAGE works: the organizer hires SAGE and decides, and SAGE members
do the work.

### Changed

- [Roles, access and kit](usage/before-you-start.md) defines four roles. The
  **organizer** hires SAGE and decides; they do nothing in the workbooks or the
  system. The **coordinator**, a SAGE member, prepares the event: plan, workbooks,
  draw, rosters, printing. The **developer**, a SAGE member, does everything
  technical. The **operators** are the SAGE members on site on the day, and they set
  up and run the venue screens. Scorers and desk staff can be SAGE members or the
  organizer's volunteers.
- The operators' three jobs on the day: **calling matches** (Court Control, calling
  matches to courts, the qualifier draws), **scores and playoff names**, and **players
  and the organizer**. Jobs can overlap, but the operator calling matches never also
  coordinates with the organizer. [Run the day](usage/run-the-day.md) says which job
  does each part of play.
- New section: [What SAGE needs from the organizer](usage/before-you-start.md#what-sage-needs-from-the-organizer).
- Every Usage step names the coordinator, developer or operators as its owner. *Admin
  task* boxes are **Developer task** boxes.
- On the day, an operator types the qualifier draw, team lineups, playoff seeds and
  late changes into the workbook; the organizer decides them. Features and Technical
  say the same.
- Glossary: Coordinator, Developer, Operator and Organizer.

## [2.1.1] — 2026-10-08

### Fixed

- Stray headings in the middle of paragraphs. A wrapped line starting with `#1` or
  `#2` rendered as a page title (Run the day, Glossary, Tournament Hub, Tournament
  Calculator, the Standard Tournament Master spec), and a sentence directly above a
  `---` rule rendered as a section heading (the Sync script configuration spec).
- Technical → Event attendance: *Pickle for Sight's attendance* is a section, not a
  second page title.

## [2.1.0] — 2026-10-08

Spec statuses brought in line with what is built.

### Changed

- [Usage guide and docs restructure](specs/implemented/docs-usage-guide-spec.md)
  moves from Not started to **In progress**: merged as 2.0.0, with the newcomer
  read-through and the site repo's checklist fixes left.
- [Site test suite](specs/not-started/site-test-suite-spec.md) stays Not started,
  with its status revised: the site engine built the `_tests/` runner, fixtures and
  unit tests it shares, and its page tests are not built.
- Nine implemented specs that had no status line carry one: Bracket Generator,
  Calculator dual-meet fixes, Calculator PWA, Dual Meet Schedule Generator, Event
  site templates, PNF × BUP dual meet, Scoresheet event picker, Runtime-fetched sync
  config and Sync script configuration.

## [2.0.0] — 2026-10-08

The Features & Usage tab splits in two: a **Usage** guide that trains a new
operator, and **Features** as a per-tool reference. Built from the
[usage guide spec](specs/implemented/docs-usage-guide-spec.md).

### Added

- **Usage** tab, first after Home: Start here, Roles, access and kit, Glossary,
  steps 1–9 (Plan it, Build the workbooks, Draw and fill rosters, Register and build
  the site, Connect the workbooks, Print, Rehearse, Run the day, When something's
  wrong), the scorer and desk handouts, and After the event.
- Features pages: [Schedule board](features/schedule-board.md),
  [Score entry](features/score-entry.md),
  [The scoring workbook](features/scoring-workbook.md) (every tab and the whole SAGE
  menu) and [Team workbook](features/team-workbook.md).
- A **Used in:** line at the top of every features page, linking the Usage steps
  where the tool comes in.
- This changelog, and the docs version in the footer.
- Checklists render as task lists.

### Changed

- The **Features & Usage** tab is now **Features**, a reference grouped as Public
  pages, Running the day, Scoring workbooks, and Planning and print tools.
- Control Center's sections follow the order of its tabs. *Entering a score* is on
  its own page, Score entry.
- The Dual Meet and Standard Tournament generator pages keep what each generator
  builds, its rules and its refusals. The SAGE menu moves to The scoring workbook,
  and the step-by-step to Usage step 2.
- Tournament Hub's schedule board section moves to its own page.
- The scorer page and event attendance pages are reference only. Their step-by-step
  is the scorer and desk handouts.
- [Adding a new event](technical/adding-a-new-event.md) is the admin's mechanics page
  for Usage steps 4–6, in the order register → site → workbooks, with the scorer
  page and the service-account share as steps of their own.
- Technical overview: Event attendance is listed under Backend.
- Home: four sections, Usage first.
- Usage, Features and Technical pages are written in the present tense. Their change
  history is recorded below.

### Removed

- `features/preparing-an-event.md`. Its content is in Usage steps 1–7.
- `features/running-an-event-day.md`. Its content is in Usage step 8 and After the
  event.
- The sync pipeline's *Baseline: the time-based trigger* section. Recorded below.

### Fixed

- Features overview: three kinds of event (standard, dual meet, team), not two.
- Adding a new event: both masters carry the sync script; only a hand-made workbook
  needs it pasted in.
- Technical overview: three Apps Script files live in `sage-tools-api/apps-script/`;
  event attendance is `src/attendance/`, part of the API.
- Home: `sage-tools-api` also runs attendance and score entry, and holds the live
  push Worker (on Cloudflare); there are five generator tools.

### History

- **Control Center** was first called **Match Control**. The file, the URL and every
  reference moved together; `tools/match-control.html` stays as a redirect stub. The
  [console](specs/implemented/match-control-console-spec.md) and
  [Awards tab](specs/implemented/awards-podium-tab-spec.md) specs predate the rename.
- **Pickle for Sight's attendance** is the earlier, per-workbook version: an
  `attendance.gs` web app in each workbook, one row per pair slot, no desk links and no
  Attendance tab. It was removed from `sage-tools-api`; its last version is
  `apps-script/attendance.gs` at commit `198f02c`. Every later event uses the API's
  `ATTENDANCE` tab, which the API creates in a workbook made before the generators
  added it.
- **The Bracket Generator** replaced four near-identical copies: two event-site
  templates and one live event's copy, which had drifted to a pre-S.A.G.E. palette.
  That event's copy became a redirect stub; archived events' copies are frozen. Adding
  keep-apart groups left a draw without groups byte-for-byte the same.
- **The generators** replaced duplicating the last event's workbook and
  find-replacing every category key, court number and team code by hand. The dual-meet
  generator's spec was written before the workbook was read closely, hence its §13;
  every bug found while building the generator was a block one row off, which is why it
  logs its layout.
- **The time-based sync trigger.** Before the lock-based sync, `sheets-sync.gs`
  scheduled a one-shot `runIfSettled` trigger from every edit. Measured on the
  Piggleball workbook on 1 October 2026, the delay from the last edit to the sync
  starting was 119, 70, 74, 22 and 108 s across five bursts: a median of about 74 s
  against the 3 s intended. Each `onEditInstallable` run took 1.3–3.3 s, mostly
  deleting and recreating triggers, and parallel edits raced on that and left stray
  triggers, so a workbook could sync twice. The lock-based sync replaced it; a
  non-holder run takes 0.6–2.2 s.
- **Live push** replaced polling GitHub Pages as the main delivery path. With polling
  alone, a published score took about 35–40 s to reach a viewer.
- **The PickleDrive workbook** was trimmed on 5 October 2026: the `StackCache` tab, a
  rewritten matchup family and 15 named functions removed, with every published value
  unchanged.
- **`completedAt`** is newer than the first published snapshots. A snapshot from
  before it uses the per-browser `syncedAt` fallback.
- **The team template.** PickleDrive Club One Year Celebration, the first team event,
  was hand-built as `events/pickledrive-anniversary-2026/` before the team template
  existed.
- **`sage-tools-api` 3.0.0** restructured the service. Before it, the service was a
  feature-sliced, layered monolith with hand-wired constructor injection: each service
  was coded against the concrete object it received, routes did their own auth and
  error mapping, and `Server` imported every feature. `/v1` was the partial REST
  surface of the 2.x releases.
- **Control Center's Match Finder refresh** used to re-run the search box's text, so
  pausing mid-word swapped the list for "Several pairs match" and the pinned search box
  lost its scroll. It redraws the search that was run.
- **The Tournament Calculator's legend** used to infer extra rounds from `plan.byes`,
  and said "no preliminary rounds needed" above a chart showing two.
- **The service-account share** inherited from `1. TOURNAMENTS` was confirmed on
  2026-10-08: Piggleball's workbook there lists the account as Editor.

## [1.0.0] — 2026-10-08

The docs as they stood when versioning began, at commit `cf8a4ae`: Home and the
Features & Usage, Technical and Specs tabs. Earlier changes are in the repo's git
history.
