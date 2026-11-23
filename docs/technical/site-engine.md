# Site engine

The site's pages share their code as JavaScript modules under
`sage-match-control.github.io/lib/v1/`, the **engine**. Each event page is a
**shell** (its markup, its colours and one small settings script), and
`tools/control-center.html` imports the same modules for every view it shares
with an event page. There is no build step, no framework and no runtime
dependency beyond what a page loads from Google Fonts and, for Control Center's
desk-link QR code, cdnjs. Pushing to GitHub Pages deploys it.

## Layout

```
lib/v1/
  platform.js      constants: the Pages address, the Cloud Run URL, fetch timeout, poll intervals, fixtureName()
  domain/          the rules, pure functions with no DOM and no clock
    csv  byes  series  deciders  matches  standings  codes  bracket  brackets  golive  progress
    teams  lineup-rules  awards  event-pages  attendance  attendance-marks  schedule-grid  scoresheet-names  courts  model
  data/            the outside world
    registry  snapshot  live-channel  attendance-signal  attendance-status  operator-session  api  tokens
  views/           functions that return HTML strings, plus small components
    html  scroll  reveal  ticket  finder  live-matches  standings  bracket  bracket-full  teams  team-mark
    score-dialog  court-dialog  attendance-view
  apps/            one per kind of page, wiring data, model and views to the page's markup
    hub  schedule-board  scorer  attendance-desk
  css/             ticket  finder  live-matches  standings  teams  score-dialog  court-dialog  attendance
                   hub  schedule-board  scorer  attendance-desk
_tests/            never published (Jekyll skips folders that start with "_")
```

Pages import the engine by root-absolute path (`/lib/v1/apps/hub.js`), for the
same reason their assets are root-absolute: moving an event's folder into
`events/archives/` must not break it.

## Dependency rules

Imports point one way, enforced by `_tests/unit/guards.test.mjs`:

| Layer | May import |
| --- | --- |
| `platform.js` | nothing |
| `domain/` | other `domain/` files |
| `data/` | `platform`, `domain`, `data` |
| `views/` | `domain`, `views` |
| `apps/` | `platform`, `domain`, `data`, `views` (never another app) |

`domain/` uses no browser global: a function that needs the time takes `now`.
Nothing under `views/` or `domain/` names an operator feature, apart from
`score-dialog.js`, `court-dialog.js` and `attendance-view.js`, which are Control Center and the
scorer and desk pages' components. No file in them takes a flag that names a
page (`isConsole`, `mode: 'hub'`).

## The versioning rule

A page names the engine version in every import. Only the newest version
folder is edited, and an edit must be compatible: it may add a module, an
export, an option or a CSS class, or fix a bug, but never removes or renames an
export, a CSS class a page's markup or CSS uses, or a theme property, and never
changes an export's parameters or return shape in a way an existing caller
would notice. GitHub Pages lets a browser keep a file for up to ten minutes, so
within a version one module only starts using another module's export after
that export was published in an earlier push.

An incompatible change is a new folder, `lib/v2/`: copy `v1`, change it there,
move Control Center, the templates and every unfinished event's pages to it,
and freeze `v1`. A finished event's shell never moves version.

## The model every view reads

`data/registry.js` returns an `EventConfig` for one event, read from
`event-data/config/events.json` the way Control Center reads it: `type`
(required, never inferred), `title`, `days` (in date order, each with its
facility names), `display` (the code-to-label maps for divisions, events and
clubs; key order is display order), `scoreEntry`, `attendance`, `teamScoring` (a team event's rule: `"points"`, or
`"games"` when `events.json` says so), `scoring` (the game's own rule, `{ to, winBy, podiumWinBy }`, or `null`
without the setting; see [the scoring rule](#the-scoring-rule)) and `playoffSeats` (a standard event's labels for empty
playoff seats, `{}` without the setting; see [the bracket chart](#the-bracket-chart)). A successful
fetch is kept in `localStorage` (`sage.registry.lastGood`), and a failed one
falls back to it.

`domain/model.js` `buildDayModel(event, day, snapshot, { now, alwaysLive })`
turns one day's snapshot into a `DayModel`: every facility's matches (and each
facility's own, for the Live Matches board), the standings, the match index by
code, the games that will never be played (a series Final's games after its decider, and a team playoff
matchup's tiebreaker that the matchup's other games made unnecessary), whether the day is live
(`alwaysLive` is Control Center, which always shows scores) and, for a team
event, its `team` data. It is built once per snapshot and handed to every view.

## Extension points

Control Center adds to a shared view through an option that does nothing by
default, never a flag. Each is documented in the JSDoc of the function that
offers it.

| View | Option | What Control Center passes |
| --- | --- | --- |
| ticket | `decorate(m)` → `{ className, attrs, metaHTML }`, `teamLogoHTML(code)`, `markPlayer(name, match)` → class | the click-to-score class, attributes and hint; the dual-meet Hub passes club logos; a class on one player's name (Control Center's not-in mark) |
| Match Finder | `searchHandlers`, `searchHint`, `delegate`, `onSearched()` | the match-number search; the team event's own finder; nothing for `onSearched` (the Hub passes `revealResults`) |
| Live Matches | `facilityExtras(name, model)` with `extrasEl`, `liveRow`, `emptyMatchupHTML`, `teamLogoHTML`, `courtCell(facility, court, match\|null)` → `{ className, attrs, html }` or `null`, `markPlayer(name, match)` → class, `matchNote(match)` → html, `countDone` | the facility progress cards; a team event's rows; the dual-meet Hub's club logos; the **Set match** button on each court's cell (`withCourtCell` adds it to the first `live-court-cell` of a court's row, whichever builder drew it); a class on a name, and html after a row's category and round; `true`, so a done court counts apart from the courts in play |
| Standings | `dualMeetDesktopLayout` (`'grid'` or `'row'`), `rrBracketLayout` (`'grid'` or `'column'`), `clubLogoHTML`, `unresolved`, `decorateBracketMatch(m)` → `{ className, attrs }`, `markPlayer(name, match\|null)` → class | the row layout; the dual-meet Hub's single table and logos; the set the label warning reads; the click-to-score class and attributes for a bracket card whose game has a scheduled match; a class on a name in the list view (`null` for a round-robin row, which has no match) |
| team views | `decorateRow`, `expandAllClass`, `teamLetters`, `searchHint`, `onSearched()` | the click-to-score row hook; the Teams tab's button class (Control Center); no team letters and the Hub's own search hint (a team event's Hub); nothing for `onSearched` (the Hub passes `revealResults`) |

## Settings, the shells and fixture mode

Each `apps/` module documents its settings in its header and rejects an
unknown or missing one with a console error that names it: `mountHub({
eventKey, liveBaseUrl, clubLogos })`, `mountScheduleBoard({ eventKey, dayKey,
liveBaseUrl, categoryColors, clubOrder, type })`, `mountScorer({ eventKey,
liveBaseUrl })`, `mountAttendanceDesk({ eventKey })`. A shell keeps the
constant names `EVENT_KEY`, `LIVE_BASE_URL` and (schedule board) `DAY_KEY`,
which the harness and the runbook find by name.

`mountHub` serves all three event types. For `"team"` it builds the team views
(`views/teams.js`, the same component Control Center uses, so it has two
callers) with `teamLetters: false` and its own `searchHint`, and shows a fourth
Teams tab. The schedule board takes `type` (`'standard'`, `'dual-meet'` or
`'team'`, inferred from `clubOrder` without it); a team board also reads the
event's `display.pairs` from `events.json`.

A team event's `display.teamColors` reaches every page as `EventConfig.teamColors`
(`eventConfigFrom` keeps hex colours only, `{}` without the map). `views/team-mark.js` turns it
into marks: `teamMarkOf(teamColors, code)` gives `{ color, shape }`, the n-th team in the map
taking the n-th of `TEAM_MARK_SHAPES`; `teamMarkHTML(mark)` draws it as an inline SVG; and
`teamTintStyle(left, right)` sets `--team-a`/`--team-b` for the `.team-tint` gradient.
`views/teams.js` (`sideMarkOf(model, side)`) puts a mark before each team name in the matchup
cards, bracket tables, Live Matches rows, roster cards and Match Finder, and tints each matchup
card's headline; the board does the same from `buildScheduleData`'s `base1`/`base2`. The styles
are in `css/teams.css` and `css/schedule-board.css`.

`domain/prize-draw.js` is the prize draw's rules (entries, prizes, each spin's winners, the replay check), imported by `tools/prize-draw.html`.

`domain/scoresheet-names.js` `withTeamNames(matchesCsv, standingsCsv)` adds `teamName1` and
`teamName2` columns to a team event's matches CSV, for the Scoresheet Generator to post.

`domain/matches.js` reads an optional `lineupShown` column (`isShown`: a ticked box, `TRUE`, yes, 1, x or ✓)
into each match's `lineupShown` (`null` without the column). For an event with `lineupReveal`, `mountHub`
builds its day model with `hideUnshownLineups`, which makes every unticked match's players `TBD` in
`matches` and `matchesByFacility` alike, and the schedule board passes the same option to
`buildScheduleData`. A page can add badges to each matchup card's header through `createTeams`'s
`cardBadges(mu)`, and a note to every match line, in every view, through `rowNotes(m)` (unlike
`decorateRow`, which only the interactive cards apply: Match Finder's and Standings' matchup cards). A
Match Finder matchup card (a team result, the all-matchups list) takes `createTeams`' `collapsed` set through
`hooks.collapsed`, and carries a **Hide matches** / **Show matches** toggle (`data-tm-collapse`) that
`createTeams` keeps across the poll's re-render.

`domain/lineup-rules.js` checks a team event's lineups against its `lineupRules` in `events.json`
(`lineupRulesFrom`, `lineupIssues`): pure, and read only by Control Center.

The team rules in `domain/teams.js` take the event's `teamScoring`:
`teamMatchupResult(ms, scoring)`, `buildTeamData(matches, standings, { scoring })`
(which echoes it as `model.team.scoring` for the views) and
`teamRankBracket(rows, matches, scoring, { headToHeadQuotient })`, where the event's
`headToHeadQuotient: false` skips the head-to-head quotient below. A group-stage game whose two
codes end `_TB` is left out of `teamGameRecords` and of both quotients, and only ranks the teams
still level after everything below, by the most such games won against each other; an unplayed
one doesn't count as a group match left. With `"games"` a matchup is won on
matches won and then total points, and a bracket is ranked by matches won, then,
for two teams level, head-to-head wins and the head-to-head quotient, and for
three or more, the overall quotient (two still level going to the two-team
rules), all computed from the matches. A tie left after that keeps the sheet's
order; once every group match between the bracket's teams is played, the ranked
array carries `ties`, `[{ codes, playoff: 'game' | 'round-robin' }]`, and the
bracket table gives those teams the higher rank and a `.tie-pill` (*Tied ·
playoff game* for two teams, *Tied · playoff* for three or more); `teamGameRecords(matches)` gives each side's wins, losses and points
for and against, and the bracket table shows W, L, PF, PA and Quotient
(`fmtQ(pf / pa)`, "—" before a point is conceded). Each matchup card adds a `.tm-stats`
line under each side, its PF, PA and quotient in that matchup. Without it every
rule is the points rule, and the card has no such line.

`domain/series.js` also holds a **best-of-3 round-robin meeting** (a category planned with
`rr_best_of` 3): its three games are rows with the same two team codes, flagged `3` in the
`CSV` tab's `bestOf` column, which `rowsToMatches` and the board's `parseFacilityCsv` read as
`bestOf`. `rrSeriesGroups(matches)` collects them by facility and pair (match numbers repeat
across facilities), in match-number order and at most three, and `unneededMeetingGames` returns
the games after the one that gave a side its second win. `unneededSeriesGames` includes them in
the same set as a series Final's games after the decider. `headToHeadWins` counts such a meeting
once, for the pair that took it, so `rankStandings` breaks a tie on meetings won; W, L and the
quotient come from `STANDINGSCSV`, and the round-robin table's PF and PA (`pointsFromMatches` in
`views/standings.js`) count every game's points. The API's completion stamp applies the same rule
([Sync pipeline § Facility completion](sync-pipeline.md#facility-completion)).

`domain/deciders.js` holds a team playoff matchup's **tiebreaker game**: a match whose two team
codes both end in `_TB` (`QF-1_TB` v `QF-2_TB`), belonging to the matchup whose rows share its
`matchUp`. It is played only when the matchup's other games leave the two sides level on games won.
Once they are all played and one side has won more, `unneededDeciders(matches)` returns it, and
`unneededSeriesGames` includes it in the same set as a series Final's games after the decider, so
facility progress, the board's *Not needed* card, Match Finder's next match and the score dialog's
warning all treat it as never played. `teamMatchupResult` leaves it out of a settled matchup (a
matchup won 2–0 is final without it; at 1–1 it counts, so the matchup stays in progress until it is
scored), a matchup card does not list it, and `pairLabel` reads it as *Tiebreaker* / *TB*. A
tiebreaker row with no `matchUp` is always counted. The API's completion stamp applies the same
rule ([Sync pipeline § Facility completion](sync-pipeline.md#facility-completion)).

`domain/brackets.js` keys a team's bracket from `STANDINGSCSV`'s `bracket`: `bracketKeyOf` gives a
number (`"1"` → `1`) or one upper-case letter (`"a"` → `"A"`), and anything else (blank, `#N/A`,
`AB`) is no bracket. `compareBrackets` sorts no bracket first, then numbers by value, then letters
A–Z. A lettered bracket reads **Bracket A** wherever a numbered one reads **Bracket 1**, and the
schedule board's category for it is `GA`.

On `localhost`, `?fixture=<name>` makes the Hub and the schedule board read
`/_fixtures/` instead of the published data, as the scorer and desk pages
do (`platform.js` `fixtureName()`; inert on the live site).

## Match numbers, categories and done courts

The match number is a tab in a top-left corner, the same on every page: `.ticket-num` as the
ticket's first child (`views/ticket.js`), `.live-num` at the start of a Live Matches row's strip
(`td.live-cat-cell`, in `views/live-matches.js` and a team event's `liveRowHTML` in `views/teams.js`),
and `.tm-num` in a team match line. Both corner tabs sit absolutely, so the room beside them is
padding: the ticket's `.time-row`, and the strip's left and right padding, the same on both sides so
the centred text stays centred. The strip's padding is set by `table.live-table tr td.live-cat-cell`,
whose selector outranks the shells' own phone rules for the strip, which finished events keep; the
strip's height is fixed (24px, 22px on a phone) so an idle row keeps the height of a live one.

A ticket's category is `divisionEventLabel` of whichever side's code has one (`.tag.cat`, before the
round's tag): the category alone, as Live Matches shows it, since a dual meet's club is already on
each side's code.

A Live Matches row is **done** when its match is `played` or in `model.unneededGames`: the court cell
gets `done` and a `.live-done-mark` (✓) in place of `.live-dot`, both rows get `live-done`, and the
strip ends in `.live-done-note`. The test lives in `live-matches.js` and again in `teams.js` (a module
may only start using another's new export in a later push), and `live-matches.test.mjs` checks both.
The done colours are literals, not `--amber`, which most shells set to a green.

## The pair summary

`views/pair-summary.js` draws the card Match Finder shows above a searched pair's tickets;
`pairResultsHTML` (`views/finder.js`) puts it under the results head and leaves out the head's
category tag when it does. It returns `''` for a team event (whose finder is `views/teams.js`), while
the day is not live, and for a pair with no round-robin row.

- `roundRobinStanding(team, model)`: the pair's round-robin code (the first of `team.codes` whose
  stage is `RR`), its group (same category, club for a dual meet, and bracket label, filtered as
  `stageTablesHTML` filters), ranked with `rankStandings`, and PF and PA from `pointsFromMatches`
  (`views/standings.js`). W, L and Q are the `STANDINGSCSV` row's. `played` is whether the pair has a
  played round-robin match (the rank shows `—` until then), `done` is `categoryRoundRobinDone`.
- `playoffRounds(team, model, standing)`: one row per playoff code in `team.codes` (a pair joins a
  playoff slot's code once the sheet writes its names there), over its matches without a BYE side. A
  series final (`seriesGameOf`) is one row over every game of the series: `walkSeries` decides it,
  the result counts games from the pair's side, and an undecided series lists the games so far. A
  slot whose opponent side is blank, `TBD` or its own code is open: its label is the opponent slot's
  `from` (`matchIdText`: "W:SF-2" → "Winner of Semifinal 2") or its `seat`. A bronze taken without a
  match (`standingsBronzeWalkover`, matched by names) adds a `walkover` row.
- `pairStatus(rounds, standing, model)`: the pill, by the first case that fits: a live round, the
  Final won or lost, the Bronze won (or the walkover) or lost, the last round still to play, lost, or
  won; with no playoff round, *Did not advance* once the category's round robin is done and every
  seat of its first playoff stage is named.

The module is new, so `finder.js` importing it is safe against a browser's cached copy of an existing
module; it uses `pointsFromMatches`, an export a push earlier than the one that added it. Its styles
are `.pair-summary` and the `.ps-*` classes at the end of `css/finder.css`. `pair-summary.test.mjs`
covers each stage on hand-made days.

## Showing a search's results

On a phone the banner and the search box fill the first screen, so a search changes nothing the
person can see. `views/reveal.js` exports `PHONE_QUERY` (`(max-width: 899px)`, the width at which
`css/finder.css` pins the search box) and `revealResults({ resultsEl, searchPanelEl, input })`. On a
phone it blurs the box, so the keyboard closes, and scrolls the page so the results start just under
the pinned search panel (the panel's `top` offset and height plus 8 px; instantly under *reduce
motion*) and returns `true`. On a computer it does nothing and returns `false`.

`createFinder`'s `onSearched` option is called right after a person runs a search, in exactly
three handlers: a click on a suggestion, Enter in the box and the **Find matches** button. A data
refresh, `setQuery` (a restored search), `reset`, typing and the clear button never call it. The
**Find matches** handler also closes the suggestion list, in Control Center too. `createTeams`'s
`onSearched` is called after a team chip's search and returns `true` when it moved the page; without
it, or when it returns `false` (a computer, or Control Center), the chip scrolls to the top as it
always did. `apps/hub.js` passes `revealResults` to both, and calls it after restoring a saved
search when Match Finder is the open tab. Control Center passes nothing, so it never scrolls.

The module is new, so `hub.js` and the views importing it are safe against a browser's cached
copy of an existing module. `reveal.test.mjs` covers it and the option with stubbed browser globals, and
the engine harness's `hub-behaviour.test.mjs` opens the Hub at a real phone and computer size, which the
harness's full-page screenshots can't (they never scroll).

## The scoring rule

`domain/scoring.js` holds an event's own rule for a game, from `events.json`'s `scoring`, and the
warnings a score gets under it. `scoringFrom(raw, eventKey)` checks the setting (`to` 1–99, default
11; `winBy` 1 or 2; `podiumWinBy` 1 or 2, default `winBy`) and returns `null` without it, or with a
console warning when it is malformed. `isPodiumMatch(m, type)` is true for a final or a bronze match:
either side's code, after its category, names round `F` or `B` (`codes.js`'s `roundKeyword`, so
`B45HMD_1`, whose category starts with B, is not one). `scoringRuleFor(m, scoring, type)` gives
the `{ to, winBy }` one match plays to, and `scoreRuleWarnings(a, b, rule)` what is wrong with a score
under it: under win by 1 the winner has exactly `to`; under win by 2, with no cap, the winner has `to`
and the loser at most `to - 2`, or past that the lead is exactly 2. `views/score-dialog.js`'s
`scoreBasicWarnings(a, b, m, rule)` uses the rule when there is one (the dialog's optional
`scoringRule(m)`), keeping *Unusually high: over 21* and a group-stage tiebreaker's game to 5;
without one its warnings are those of a game to 11, win by 2. Control Center and the scorer page
pass `scoringRule`; the warnings never stop a save.

## The bracket chart

A standard category with two or more playoff rounds can draw its playoffs as a
bracket chart instead of the matchup cards. `domain/bracket.js` is the pure part:
`buildBracket(rows, model)` groups a category's playoff rows into rounds of
matches (a `Node`, with a `Game` per scheduled match and a `Seat` per side),
`seatText` says what each seat reads, `bracketShapeKey` names the category's
shape (`3x2`: three round robin brackets, two advancing from each), and
`layoutBracket` returns every card's and line's position in pixels.
`views/bracket.js` turns that into the chart (`bracketChartHTML`) and the
narrow-screen window (`roundStripHTML`: the same chart in a sideways-scrolling track under a chip per round), and `views/bracket-full.js` is Full
tree, a fixed layer with pinch, drag and double-tap whose arithmetic is
`fitScale`, `clampPan` and `zoomAbout`. Full tree draws its chart with
`bracketChartHTML`'s `fullTree` option: `FULL_TREE_DIMS`'s wider cards and taller
seats, each player of a pair on a `bk-line` of their own (`.bk--two` in
`css/standings.css`). `views/standings.js` decides which to
show: `createStandings` keeps the reader's List / Bracket choice
(`localStorage` `sage.standings.playoffView`), and `stageTablesHTML` draws the
whole chart when it is no wider than the card, else the window. Given the window's
width, `roundStripHTML` stretches the cards (`fitCardWidth`, `bracketChartHTML`'s
`cardW`) so the window holds whole rounds (`bk-rounds--fit`, snapping to each), and
`createStandings` lights the chip of every round in view. On a phone the room is the
card's body less its padding and border (`availableWidth`), one round at a time; on
the desktop layout `categoryCardHTML`'s `fixedColumn` makes it the card's own column,
360px or, for side-by-side brackets, 740px (`COLUMN_WIDTH`, `WIDE_COLUMN_WIDTH`, less
`CARD_PADDING` and `COLUMN_CHROME`), so List and Bracket draw the card at the same
width, and `roundsInView` shows one round at a time or two (a chart with no more
rounds is drawn whole, stretched). A standard category's card titles its two
sections, Round Robin and Playoffs (`.standings-section-title`, `stageTablesHTML`'s
`sectionTitles`; a dual meet keeps the plain stage caption), and the List / Bracket
switch sits at the end of the Playoffs title (`.playoffs-head`, `viewSwitch`), not
in the category's title. Either way a **Full tree** button (`fullTreeButtonHTML`) sits under it.

The sheet supplies the shape of the ladder in two `STANDINGSCSV` columns,
`match` and `from` (`rowsToStandings` reads them; see
[the generator](standard-tournament-generator.md#bracket-links)). With them a
category is **linked**: rounds, lines and "Winner of QF-2" come from the sheet,
and the seats are grouped by `match`, never by arithmetic on seat numbers. A
workbook without them is drawn by pairing the rows in order (the same rule as the
matchup cards, now `pairPlayoffRows`), with the rounds side by side, no lines and
`TBD` seats. A seat the round robin fills reads the workbook's label for it (the
row's `seat`, `STANDINGSCSV`'s `seat` column), else the event's
(`display.playoffSeats`, by category code or by shape key, in
[the config](event-data-config.md)), else **Qualifier**; once its names are on the
sheet, `sourceTag` matches them to the round robin rows by name and tags the
finish (`Br 2 · #1`). The site displays labels and never computes who qualifies.

Control Center passes `decorateBracketMatch`, so a card with a scheduled match
opens the score dialog (see [Control Center](control-center.md)). The Hub passes
nothing. The chart's CSS is in `css/standings.css` and uses only the theme
properties; the Full tree layer's class `bk-full` is on a `div`, which the rule
`div.bk-full` keeps apart from the span of the same class that holds a pair's full
names.

## The theme contract

A shell's `:root` defines the custom properties the engine's CSS reads
(`--navy`, `--green`, `--paper`, `--ink`, `--radius`, `--card-shadow` and the
rest in `_tests/unit/theme-contract.test.mjs`). A property the CSS reads with
a fallback (for example `--att-bar-bg`) is listed as optional. Where a page
differs from the shared rules, its own rules sit in a labelled block at the end
of its `<style>`.

Some tints and shadows in the engine's CSS are still literal house colours
(`rgba(124,185,44,…)` and the like): a winning side, a live row, the live dot's
pulse, the dividers, a sticky header's paper. A page with its own palette would
show them unchanged, so every template's Hub, board and scorer page restates each
of those rules in a block headed **lib/v1/css's house colours, through this
page's tokens**, writing `color-mix(in srgb, var(--token) N%, transparent)` for
`rgba(<house token>, N/100)`. That is the same colour on the house palette and
the event's own on any other. `_tests/unit/house-colours.test.mjs` fails when an
engine rule in a house colour is missing from the block of a page it lists (the
templates, and event pages re-skinned from them), or is restated as anything but
its token version.

## The comparison harness

`_tests/` runs the same page in two trees, the commit the branch started from
and the branch, on the same data, the same fixed clock and the same screen
size, and compares what each shows: the page's text and a full-page
screenshot.

```bash
cd sage-match-control.github.io/_tests
npm install
npm run verify      # the unit tests, then the comparison
npm test            # the unit tests only
npm run compare     # the comparison only
```

`CONCURRENCY=3` sets how many cases run at once and `CASE='<regex>'` selects
cases by id (for example `CASE='^cc/piggleball-2026/finder'`; on Windows Git
Bash, prefix `MSYS_NO_PATHCONV=1` if the regex starts with `/`). A case that
differs because the machine was busy (a step timed out, or a page was caught
mid-reload) is run again; a real difference is still there on the reruns.
`_tests/engine/accepted.mjs` lists the differences that are accepted on
purpose, each naming its row in the spec's reconciliations table. The unit
tests under `_tests/unit/` cover the pure modules, the views' markup
builders, the apps' settings checks, the dependency guards, the theme
contract, and that the engine's server-side twin, `facilityCompletion.mjs`
in `sage-tools-api`, agrees with the browser's played/BYE/series rules
(`parity-server.test.mjs`, which needs `sage-tools-api` beside this repo).

## Changing the engine

- **A rule** (played, BYE, series, standings, go-live): change the module in
  `domain/`, with its unit test first. There is one copy, so the Hub, Control
  Center and the scorer page all change; run `npm run verify`.
  `sage-tools-api/src/sync/domain/facilityCompletion.mjs` is the server's copy
  of the played and BYE rules; the parity test fails if the two disagree.
- **A view**: change the builder in `views/` and its CSS in `css/`, keep the
  ids and classes the pages' markup uses, and add an option rather than a
  flag when Control Center needs something the Hub does not.
- **A new view or page kind**: a new module in the right layer, a unit test
  for it, and, for a page, an `apps/` module with a shell.
- **Before pushing to `main`**: `npm run verify`, and not within three days of
  an event.

Where the Hub and Control Center still differ on purpose, the spec's
reconciliations table ([§12](../specs/implemented/site-engine-spec.md))
records each difference and the decision behind it.

---
**Features:** [Usage guide](../usage/README.md) · [The Tournament Hub](../features/tournament-hub.md)
**Spec:** [`site-engine-spec.md`](../specs/implemented/site-engine-spec.md) (the plan, the phases and the reconciliations).
