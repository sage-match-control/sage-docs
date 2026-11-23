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
    csv  byes  series  deciders  matches  pair-schedule  standings  standings-status  codes  bracket  brackets  golive  progress
    teams  lineup-rules  awards  event-pages  attendance  attendance-marks  schedule-grid  scoresheet-names  courts  model
    board-day  initial-day
  data/            the outside world
    registry  snapshot  live-channel  attendance-signal  attendance-status  operator-session  api  tokens
  views/           functions that return HTML strings, plus small components
    html  scroll  reveal  hub-bar  ticket  finder  club-bar  live-matches  standings  podium  bracket  bracket-full  teams  team-mark
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
| Match Finder | `searchHandlers`, `searchHint`, `delegate`, `onSearched()`, `introTop(model)` | the match-number search; the team event's own finder; nothing for `onSearched` (the Hub passes `revealResults`) or `introTop` (the dual-meet Hub passes the club bar; Control Center and every other event draw none) |
| Live Matches | `facilityExtras(name, model)` with `extrasEl`, `liveRow`, `emptyMatchupHTML`, `teamLogoHTML`, `courtCell(facility, court, match\|null)` → `{ className, attrs, html }` or `null`, `markPlayer(name, match)` → class, `matchNote(match)` → html, `countDone` | the facility progress cards; a team event's rows; the dual-meet Hub's club logos; the **Set match** button on each court's cell (`withCourtCell` adds it to the first `live-court-cell` of a court's row, whichever builder drew it); a class on a name, and html after a row's category and round; `true`, so a done court counts apart from the courts in play |
| Standings | `dualMeetDesktopLayout` (`'grid'` or `'row'`), `rrBracketLayout` (`'grid'` or `'column'`), `clubLogoHTML`, `unresolved`, `decorateBracketMatch(m)` → `{ className, attrs }`, `markPlayer(name, match\|null)` → class, `onPairSelect(teamCode)`, `podium`, `jumpChips` | the row layout; the dual-meet Hub's single table and logos; the set the label warning reads; the click-to-score class and attributes for a bracket card whose game has a scheduled match; a class on a name in the list view (`null` for a round-robin row, which has no match); `openPairInFinder` for `onPairSelect` (the pair in Match Finder, as the Hub does); nothing for `podium` (the Hub draws a finished category's podium; Control Center has its Awards tab) or `jumpChips` (the dual-meet Hub draws the chips on a phone) |
| team views | `decorateRow`, `expandAllClass`, `teamLetters`, `searchHint`, `onSearched()`, `podium` | the click-to-score row hook; the Teams tab's button class (Control Center); no team letters and the Hub's own search hint (a team event's Hub); nothing for `onSearched` (the Hub passes `revealResults`); nothing for `podium` (the Hub opens the Standings with the event's podium) |

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
event's `display.pairs` from `events.json`. A board whose page has the
`#dayfilter` container reads the event's days from `events.json` too, for its
**Day** picker; which day it shows is `domain/board-day.js`
([schedule board](schedule-board.md#day-picker-day)).

Which day the Hub and Control Center open on is `domain/initial-day.js`'s
`initialDayIndex({ days, savedKey, today })`: the day this browser last picked for the
event (`savedKey`, while it is still one of the event's days), else the day dated
`today`, else the day nearest `today` by date (a later day winning a tie), else the
first day; `null` only for an event with no days. `today` is `manilaIsoDate(now)`, the
Philippines' date, with `now` passed in by the caller. The Hub passes the day saved
under `<event>.dayIndex`; Control Center the event's entry in
`sage.controlCenter.dayByEvent` (`{ [event]: dayKey }`, written with every day it
loads), after an address's `?day=`. `_tests/unit/domain/initial-day.test.mjs` covers it.

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
category tag when it does. It returns `''` for a team event (whose finder is `views/teams.js`). While
the day is not live, and for a pair with no round-robin row, it is the next-match block alone in a
`.pair-summary` (and `''` when the pair has no next match); otherwise the block is the card's first
child, before `.ps-top`.

- `domain/pair-schedule.js`: `pairMatches(model, codes)` is the pair's matches (any of its codes, either
  side, a BYE side left out) in schedule order: the Schedule time in event-day minutes, a time that does
  not parse last, equal times by match number. `nextUpIndex(model, list)` is the index of the first one
  not played, not on a live court and not in `model.unneededGames`, or `-1`. `pairResultsHTML` uses both
  for its tickets and their **Next Up** pill, and `nextMatchHTML` uses them for the block.
- `nextMatchHTML(team, model)`: the navy `.ps-next` block. On a live day, a pair match with a live court
  that is not played is *Playing now* (`.ps-next--live`, the live dot, `Court ${m.liveCourt}`); otherwise
  the Next Up match (`.ps-next--next`) shows its Schedule time and, when not blank, its
  `CourtAssignment`. The opponent is the other side's two names, or, for an open side, its
  `openSeatLabel` or *To be decided*. The round is the pair's own code's `roundKeyword` through the
  same names as the playoff rows (*Round robin* with none), with ` · Game n` for a series game. The
  facility (`m.facility`) shows only when `model.day.facilities` has more than one. A day that is not
  live ignores live courts. Every value is escaped.

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
are `.pair-summary` and the `.ps-*` classes at the end of `css/finder.css`; the block's are
`.ps-next` and its `small`, `b`, `span`, `i` and `em`, in the page's `--navy`, `--white` and `--green`.
`pair-summary.test.mjs` covers each stage and the block on hand-made days, and `pair-schedule.test.mjs`
the order and the Next Up rule.

## Showing a search's results

On a phone a search's results can start below the screen, under the search box and the day's counts,
so a search can change nothing the person sees. `views/reveal.js` exports `PHONE_QUERY` (`(max-width: 899px)`, the width at which
`css/finder.css` pins the search box) and `revealResults({ resultsEl, searchPanelEl, input })`. On a
phone it blurs the box, so the keyboard closes, and scrolls the page so the results start just under
the pinned search panel (the panel's `top` offset and height plus 8 px, plus the bar's height (`--hub-bar-h`, else 54 px) while
the bar is on the page but not showing yet, which only a cached stylesheet from before the fold rules
leaves, since there the scroll brings it in and moves the panel down; instantly under *reduce motion*) and returns `true`. On a computer it does nothing and returns `false`.

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

## The bar the banner folds into

A bar pinned to the top of the screen stands in for the banner (hub-player-experience-spec §5): on a
phone always, since the banner never shows there; on a computer once the reader folds the banner with its
chevron, the schedule board's. Control Center uses the same bar. On a phone it is two rows: the logo, the
small line and the whole title, then, on a darker band, the menu button (and at a multi-day event a day pill
beside it). On a computer the Hub's is one slim row (54 px) and Control Center's two rows (the title, then
the row of the day picker, the tabs and the sync line). `views/hub-bar.js` exports `HUB_BAR_HEIGHT` (54, the
slim row, until the bar is measured), `TITLE_MIN_PX` (13), `FOLDED_KEY` (`'sage.hub.bannerFolded'`),
`titleFit`, `barHeight`, `shortDay`, `dayPillLabel` and
`createHubBar({ heroEl, tabsWrapEl, dayPickerEl, dayPickerWrapEl, syncStatusEl, panelFor, text, logoSrc, pickers, storageKey, menuLabel, computerRows })`,
which returns `{ update() }`; the last six are optional. The banner keeps the page's one set of tabs,
pickers and sync line; the bar reads them and clicks them, it never holds its own state.

- **Markup.** Made once and appended to `document.body`: `.hub-bar` (hidden), `.hub-menu-scrim` and
  `nav.hub-menu#hubMenu` (`aria-label` from `menuLabel`, default *Tournament Hub*). The bar carries a logo
  (`.hub-bar-logo`: `logoSrc`, else the first `.hero-logos` image's `src`; left out when there is none), two
  lines (`.hub-bar-meta`, `.hub-bar-title`: `text()`'s `{ meta, title }`, read on every update, else the
  hero's `.eyebrow` and `.title` with each `<br>` read as a space and whitespace collapsed, read from a copy
  since a hidden element's `innerText` runs the lines together), a row (`.hub-bar-row`), the chevron that
  brings the banner back (`.hub-bar-fold`, *Show the banner*), the phone's day pill (`.hub-bar-pday`,
  hidden: `.hub-bar-pday-text`, a caret and a `<select aria-label="Day">` laid over it, invisible) and the
  phone's button (`.hub-bar-btn`, `aria-expanded`, `aria-controls="hubMenu"`). `computerRows: 2` adds
  `hub-bar-two` to the bar. It also appends the fold chevron (`.hub-fold-btn`, *Fold the banner*, hidden) to
  the banner. The CSS shows the row and the chevrons on a computer, the pill and the button on a phone.
- **Items.** `readBarItems` reads the banner: a tab per `.view-tab` showing (not `display:none`, not
  `hidden`), the open one `active`, and none while the tab row itself is hidden (no day loaded); each of
  `pickers` (`[{ el, wrapEl?, label }]`) whose wrapper is showing, keeping its index; the day picker (its
  options without the disabled placeholder, its value) when its wrapper is showing, so only at an event with
  more than one day; the sync line when it is showing. `barRowHTML` draws the computer's row (the sync line,
  a `.hub-bar-day` label with a `<select>`, a `.hub-bar-tab` pill per tab; never the extra pickers) and
  `menuHTML` the phone's menu (a `.hub-menu-tab` per tab, a `.hub-menu-day.hub-menu-picker[data-picker]` row
  per picker, `.hub-menu-day`, `.hub-menu-sync`). `buttonLabel` names the open tab, or *Choose a day* while a
  day picker waits for a first pick, or *Menu*. Everything is escaped. The row is replaced only when its
  markup changed, so a poll that finds the same sync line leaves an open day picker alone; the menu is filled
  each time it opens.
- **The day pill** (a phone, an event with more than one day). Shown when the day picker is read and the
  stylesheet lays the pill out (`--hub-bar-fit` is set on the bar; a cached older stylesheet does not, and
  the pill stays hidden). Its text is `dayPillLabel`: the open day's label up to its first " · "
  (`shortDay`: *Nov 28 · Next Gen* reads *Nov 28*), or *Day* before one is picked. Its `<select>` holds the
  day picker's options (with a disabled *Choose a day* placeholder before a pick), rewritten only when they
  change; a choice goes through the bar's own `change` handling, as the row's and the menu's do.
- **The title.** Where the stylesheet sets `--hub-bar-fit: 1` on the bar (a phone, and `hub-bar-two` on a
  computer; the slim row sets 0 and cuts the title short with an ellipsis), `fitTitle` measures the title at
  its own size: `titleFit({ room, need, base })` keeps it (`'fits'`, a pixel of rounding allowed), shrinks it
  (`{ size }`, rounded down to 0.1 px) or, below `TITLE_MIN_PX`, wraps it (`hub-bar-title-wrap`) at its own
  size. It runs in `fit()` and once the fonts have loaded.
- **The bar's height.** `measure()` sets the bar's rounded height on `<html>` as `--hub-bar-h`, written only
  when it changes, from `fit()` and a `ResizeObserver` on the bar. `barHeight()` reads it back (54 where it
  is not set). `views/reveal.js`, `views/standings.js`, `apps/hub.js` and Control Center read the custom
  property themselves (54 where it is not set) rather than import `barHeight`, so a module never needs
  another's new export in the same push.
- **A tab** closes the menu, clicks the matching `.view-tab` (whose handler switches the view and
  remembers it) and scrolls `panelFor(view)` to just under the bar (`tabScrollTop(top, scrollY, height)`: its
  top minus the bar's measured height and 8 px, never above the page; instant under *reduce motion*). **A day**, or **a
  picker**, closes the menu, sets the banner's `<select>` and dispatches `change` on it, then scrolls to the top.
- **Folding.** The banner's chevron sets the folded choice, the bar's clears it; each stores it in
  `localStorage[storageKey]` (`'1'` or `'0'`), re-checks, scrolls to the top and moves focus to the other
  chevron. The choice is read once, when the bar is made.
- **When it shows.** With the fold rules loaded (`css/hub-bar.css` sets `--hub-bar-fold: 1` on `.hub-bar`),
  `barFolds`: always on a phone (`PHONE_QUERY` from `reveal.js`), even before a day is loaded; on a computer
  when the banner is folded and the tabs are showing, so the banner, with its pickers, is back while no day is
  loaded. The banner's chevron shows on a computer once the tabs show. A stylesheet from before the fold rules
  (a browser's cached copy) gets `barShows`, the earlier rule: the tabs are showing, the banner is out of
  view (an `IntersectionObserver` on it, threshold 0) and the bar is styled
  (`getComputedStyle(bar).position === 'fixed'`). Showing unhides the bar and puts `hub-bar-on` on `body`,
  and with the fold rules `hub-bar-folded` too; hiding reverses them and closes the menu. The banner
  chevron's `hidden` is written only when it changes, since a page may watch the banner to redraw the bar.
  A change of width re-checks.
- **Fitting the row.** On a computer the slim row keeps its width while it fits; where it doesn't the CSS
  lets it shrink, with the title keeping 120 px, and scroll sideways (Control Center's own row is as wide as
  the bar, and scrolls only where that is too narrow), and `fit()` puts `hub-bar-tight` on the bar (which hides the sync line) and scrolls the open tab
  into view. It runs on every check and on `resize`.
- **The menu** (a phone, below 900 px). The button toggles it, and opening focuses its first tab (or its
  first select). The scrim, the button and Escape close it and return focus to the button; widening the
  screen to 900 px or more closes it too.
- **Wiring.** `apps/hub.js` creates the bar once the event is known to be a Hub event and its views exist
  (a shell without `header.hero` or the tab row keeps a no-op), with `panelFor` returning the search
  panel, Live Matches, Standings or Teams panel, and calls `bar.update()` at the end of `showView`, of
  `revealLiveTabsAfterLoad`, in `selectDay` after it hides the tabs, and in `loadLiveData` after the
  sync line is written (success and error). Control Center creates it with `text()` (*Control Center* and
  the event's title), `logoSrc: '/assets/logo.png'`, its event picker in `pickers`,
  `storageKey: 'sage.controlCenter.bannerFolded'`, `menuLabel: 'Control Center'` and `computerRows: 2`, and a
  `MutationObserver` on the banner (its subtree, text and `class`, `style` and `hidden`) calls `update()` on
  any change there.
- **CSS** (`css/hub-bar.css`, which `css/hub.css` imports and Control Center links on its own). The bar is
  fixed, `z-index` 50, a wrapping flex line on the theme's `--navy`, with the accent `--green` for its small
  line and a computer's open tab. The markup is the same for every layout: the second row's items take
  `order` 10 and up, and a phone's `.hub-bar::after` (an empty item as wide as the bar, `order` 9) ends the
  title's line; the band is a gradient, the bottom 48 px (44 on a computer) in `--navy-deep` (`--navy` where a
  theme has none). On a phone the second row is the day pill (`flex:1`) and the button (`flex:3`), each 34 px
  high with 7 px above and below; alone, the button takes the row. On a computer the slim row is 54 px, no
  wrap, `--navy`; `.hub-bar-two` wraps instead, a 58 px title row (38 px logo, a 20 px title) over the 44 px
  row (`flex-basis` 100%), the sync line pushed to its right end. `--hub-bar-h` starts at 100 px on a phone and
  54 px on a computer until the script measures the bar. On a phone `header.hero` is hidden and `html body`
  has `--hub-bar-h` plus 12 px of top padding (`html body` outranks the pages' own `html,body{ padding:0 }`,
  which `hub.css` sets after its import); hiding the banner from the stylesheet rather than the script means
  it never flashes up while the page loads. On a computer `body.hub-bar-folded` hides `header.hero` and pads
  the page by `--hub-bar-h` plus 24 px. The chevrons are 32 and 30 px squares with the board's border and
  colours, the banner's in its top-left corner. While `body.hub-bar-on`, what is already pinned to the top
  moves under it: `.standings-col-head`, `.club-summary` and `.dm-pin` at `top: var(--hub-bar-h)`, and on a
  phone the Match Finder search panel 8 px below that. On a computer a Standings column scrolls by itself
  (the pages cap it at 92vh), so its `.standings-col-head` pins to the column's own top and keeps `top:0`
  there (`.standings-board-desktop .standings-col-head`); on a computer `.qr-panel` moves from `top:26px` to
  26 px under the bar. From 900 to 1099 px the slim row's sync line is hidden; below 900 px the row and
  chevrons are hidden and the menu and its scrim (`z-index` 49 and 48) shown, from `--hub-bar-h` down.

`hub-bar.test.mjs` covers the pure parts (reading the items and pickers, the markup, the button's label, the
day pill's label, how the title fits, when the bar shows with and without the fold rules, where a tab scrolls,
and `reveal.js`'s allowance, measured or not) with faked elements. The engine harness's `hub-behaviour.test.mjs` covers the bar on a page: a phone's bar always on
with no banner, its button, menu, Escape, scrim and widening (which brings a computer's banner back); a
computer's banner and chevron, folding, the choice kept across a reload, unfolding, the bar's row, the QR
panel and the 950 px sync line; a team event's four tabs; a dual meet's club bar pinned under it; a multi-day
event's day picker, in the row, in the menu and as a phone's day pill (a quarter of the row), and the day
that opens with nothing saved and with a day picked before; the two rows on a phone and Control Center's
computer, the Hub's slim row on a computer, the title whole and `--hub-bar-h` equal to the bar's height; and
Control Center's bar: its menu with the event picker, switching event from it (a long title fitted or
wrapped), folding into its own key, and seven tabs at 950 px. The harness's steps (`engine/steps.mjs`) click a tab and set a picker from script when the banner is
folded away, as the bar's menu does, so every phone case runs as before; its screenshots show the bar in
place of the banner on every phone case, and the fold chevron in every computer's banner.

## The round-robin pills

Each round-robin table says how far it has got, and each pair level on wins with another says how
it was placed (hub-player-experience-spec §6). The rules are `domain/standings-status.js`, which
imports `domain/` only; the markup is drawn by `views/standings.js` and `views/teams.js`, and only
when `model.isLive`, so nothing shows before the day is live. Control Center's model is always live.

- `tableProgress(rows, matches, type)` → `{ played, total }`. A table's meetings are the round-robin
  matches (`roundKeyword` of the code's `rest` is `null`) with a side that is a row of the table and no
  BYE side (`matchByeSide`), counted once each. A best-of-3 meeting (`bestOf` 3, grouped by
  `rrSeriesGroups`) is one meeting, played when `walkSeries(list, 3).decidedBy` is set; any other match
  is played when `m.played`. A dual meet's club table has one side of each cross-club match, so both
  clubs' tables count the same matches. `progressLabel` turns it into `{ kind, text }`, `In progress ·
  6/10` or `Final`, and `null` for a table with no meetings.
- `tieBreaks(rows, matches)` → `Map<teamCode, { size, rule, won?, score? }>`. Among the rows that have
  played (`wins + loss > 0`), the rows level on wins form a group, and `headToHeadWins` (the function
  `rankStandings` orders by) gives each row's head-to-head wins in it. `rule` is `'h2h'` for a row whose
  head-to-head wins differ from every other row's in the group, `'quotient'` when it shares them and its
  quotient differs from theirs, `'none'` when it shares both. `size` is the whole group. In a group of
  two told apart by head-to-head the entry also has `won` and `score`, the pair's own side first: the
  one match's points (`11–5`), a best-of-3 meeting's games (`2–1`), or, for two or more meetings, the
  meetings won and lost. `tieLabel` turns an entry into `{ text, tied }`. A row level with no other is
  not in the map. The pills describe `rankStandings`' order only, not the workbook's feeders.
- `teamTableProgress(matchups, group)` → `{ played, total }` for a team bracket: the group-stage
  matchups (`stage` null) of the bracket, played once `result.state` is `final` or `tie`.

`stageTablesHTML` draws `<span class="rr-status rr-status--progress|final">` in a bracket's
`.rr-bracket-label`; for a table drawn as one (no brackets, or the dual meet's `'column'` layout, one
pill for the whole table) in the Round Robin heading, which is the `.stage-caption` or, in a standard
card, inside the `.standings-section-text` title. `groupTableHTML` puts it in each team bracket's
`h3.group-title`. A tie pill is `<span class="rr-tie">` (`rr-tie--tied` for *Tied*) passed to
`pairCell`'s `after` option, which places an HTML string in the pair cell after the two names (none
by default, and none in a TBD cell); it follows the block-level name spans without a `<br>`, which
would add an empty line. Team tables get no tie pills.

`css/standings.css` styles both: `.rr-status` takes its fill from `--green-dark` and `--navy`, and the
tie pill from `--paper-dim`, `--ink` and `--line`, with *Tied* in the team events' yellow. The
stylesheet is loaded by all three templates and Control Center, so Control Center draws the pills
through the shared views with no change of its own. `_tests/unit/domain/standings-status.test.mjs`
covers the rules, including that the pills' order is `rankStandings`' on each tie case, and
`_tests/unit/views/standings-status-markup.test.mjs` the markup.

## A Standings row opens its pair

On the Hub and in Control Center a named round-robin row opens that pair in Match Finder
(hub-player-experience-spec §7).
`createStandings` takes an option `onPairSelect(teamCode)`, off by default. When it is set, the options the
builders already pass carry a boolean, `openRows`: `cardOptions()` adds it for a standard event's cards
(`categoryCardHTML` hands it to `stageTablesHTML`), and a dual meet's body is drawn with `{ ...options, openRows }`,
which `dualMeetStandingsHTML` passes down through `categoryColumnHTML` and `clubSubsectionHTML` to
`stageTablesHTML` (the shared Bronze and Final block never draws a round-robin row). Without `openRows` every
row is drawn exactly as before, so the page's old cached copies are unchanged; a team event's Standings are `views/teams.js`, which never reads it.

`stageTablesHTML` gives each round-robin row for a named pair (not `isEmptyStanding`, so not an unseeded slot, a
TBD or a pair still listed under its team code) `class="… rr-open"`, `data-team="<teamCode>"` and `tabindex="0"`,
in the grid and the `'column'` layouts alike, and ends its Q cell with
`<span class="rr-open-mark" aria-hidden="true">›</span>`, after the figure and clear of the tie pill, which is in
the pair cell. A click on a `tr.rr-open` inside the panel, or Enter with focus in one, calls
`options.onPairSelect(tr.dataset.team)`; the panel's own click handler checks for a row first, since nothing in a
row is another control.

`apps/hub.js` passes `onPairSelect` for a standard or dual-meet event. It finds the pair in `finder.pairs` whose
`codes` include the team code (a code the finder has no matches for does nothing), clicks the Match Finder tab,
puts the pair's label in the search box, sets `finder.selection`, runs the search and calls `reveal`. On a phone
`reveal` lands on the results under the pinned search box; on a computer, where it does nothing, the page scrolls
to the search panel (16 px above it, and the bar's height more while the bar shows, `body.hub-bar-on`: `--hub-bar-h`, else 54 px), so the results are in
view. It uses the finder's existing members and adds no import. Control Center passes the same kind of handler,
`openPairInFinder`, for a standard event or a dual meet: it switches with `showView('finder')`, runs the search the
same way and calls `revealResults` itself, since it passes the finder no `onSearched`.

`css/standings.css` has `tr.rr-open` (a pointer), its `:focus-visible` outline (`--green-dark`) and
`.rr-open-mark` (`--muted`, bold, 4 px from the figure). The mark is about 3 px wide and fits the Q column's
54 px, so a row's height does not change. `_tests/unit/views/standings-open-rows.test.mjs` covers the markup and
`hub-behaviour.test.mjs` taps a row at a phone and a computer size, with Enter and with the bar showing, and in
Control Center at both sizes, at a dual meet, and at a team event, which has no open rows.

## The podium

On the Hub a finished category's Standings card opens with its podium, and a team event's Standings with the event's
(hub-player-experience-spec §8). The rule is Control Center's Awards tab's, so the two never disagree: `views/podium.js`
reads `domain/awards.js` (`buildPodiums`, `buildTeamPodium`, `resolveDecisiveMatch`, `standingsBronzeWalkover`) and
`domain/series.js`'s `seriesGameOf`, and has no rule of its own beyond when to show it.

- `podiumFor(model, category)` and `teamPodiumFor(model)` → `{ gold, silver, bronze, finalLine }` or `null`. Each place is a
  display string: a pair's two names joined by ` / ` (followed by ` · <club label>` at a dual meet), or a team's name;
  `bronze` is `null` when there is none. `buildPodiums(model)` is kept per model in a `WeakMap` (a model is built once per
  snapshot), so the cards of one render share one derivation.
- **When it shows** (standard and dual meet): the entry has no `warning` (a Final or Bronze tied on points), `gold` and `silver`
  are set, and `bronze` is set or the category has no bronze at all (`resolveDecisiveMatch(model, category, 'B').any` and
  `standingsBronzeWalkover(...).walkover` are both false). A category decided by its round robin alone (`source: 'standings'`)
  shows too, once `buildPodiums` fills its places (the round robin is done), with no final line. A team event shows when
  `buildTeamPodium` has no warning, gold and silver, and a bronze unless the event has no `Br` matchup.
- **The final line** (gold's `<small>`): for `source: 'bracket'`, the decisive Final's winner's score first, *Beat <silver>
  11–7 in the final*, and with no score for a series final (`seriesGameOf(m.t1)`, whose games are not one score). A team's
  uses the `Fi` matchup's result from the winner's side: `pts1`–`pts2`, or `pairs1`–`pairs2` when `result.scoring` is
  `'games'`, and *37–35 on points* for a Final level on games, which `"2–2"` would misreport.
- `podiumHTML(p)` returns `<div class="podium">` with a `.podium-title`, and an `<ol class="podium-list">` of `<li class="podium-place
  podium-place--gold|silver|bronze">` rows, each a `.podium-rank` (1, 2, 3) and `.podium-names`; every string is escaped. No
  podium is `''`. `categoryPodiumHTML(model, rows)` reads the category from the card's rows
  (`parseCode(rows[0].teamCode, type).category`) for the two Standings builders.

`createStandings` takes `podium` (default `false`) and carries it in the card options: `categoryCardHTML` and
`categoryColumnHTML` (a dual meet's column, through `dualMeetStandingsHTML`) put the podium first in `.standings-col-body`.
`createTeams` takes it through `hooks.podium`, and `teamStandingsHTML` starts the `.team-standings` container with it. Both
import the new module; a module that did not exist before cannot be a stale cached copy, so the import is safe. `apps/hub.js` passes `podium: true` to both;
Control Center passes neither, so its Standings never draw one.

`css/standings.css` has `.podium` (a white card, 14px radius, `--card-shadow`, 12px 14px padding), `.podium-title` (Barlow
Condensed, uppercase, `--muted`), `.podium-place` (a flex row, a dashed `--line` divider between places), `.podium-rank` (a 26px
circle; gold `#F3D46B` on `#4A3500`, silver `#DADDE3` on `#2E333D`, bronze `#E3B48A` on `#4A2A0C`, the pair summary card's
medal colours) and `.podium-names` (14px 600, wrapping long names, its `small` 12px `--ink-soft`). On a computer the card sits
in its column and changes no column's width, so the 1 col / 2 cols toggle and the bracket chart are as before.
`_tests/unit/views/podium.test.mjs` covers the rule on the fixtures and on hand-made days, and
`podium-wiring.test.mjs` the options (off by default, first in the body, none on `mid`).

## The club bar

At a dual meet the Hub's Match Finder opens with the club score (hub-player-experience-spec §9).
`views/club-bar.js` exports `clubBarHTML(model, { clubLogoHTML })`: `''` unless `model.event.type` is
`'dual-meet'` and `model.isLive` (and `''` with fewer than two clubs), otherwise
`<div class="finder-club-bar">` around `clubSummaryHTML`, the win-totals bar of Standings. It groups
`model.standings` by club and orders the clubs as `dualMeetStandingsHTML` does (the event's `display.clubs` in key
order, then any other), the few lines copied rather than exported so that no existing module starts importing a new export
of another in the same push. The wins are therefore Standings' wins, and the same logos come through `clubLogoHTML`,
which `apps/hub.js` now holds in one named const for both.

`introHTML(model, { introTop })` prints `introTop(model)`'s HTML before `.stats-row`; `createFinder` takes `introTop`
(default none) and passes it to the first screen's `introHTML`. Only the first screen draws it: a search's results
replace it, and a team event's finder, which draws its own first screen through its `delegate`, never does. The Hub passes
`introTop: m => clubBarHTML(m, { clubLogoHTML })` at a dual meet and nothing elsewhere; Control Center passes none. The
middle count's label reads *Pairs* (it was *Teams*, which a pair of players is not); a team event's own counts (Matches,
Matchups, Teams) are `teams.js`'s and unchanged.

`css/finder.css` has `.finder-club-bar` (a 14px bottom margin) and `.finder-club-bar .club-summary{ position:static }`,
because the bar sticks in Standings and must scroll here. `_tests/unit/views/club-bar.test.mjs` covers the bar against
Standings', the logos, and the cases that draw none (a standard or team event, a day not live), and
`finder.test.mjs` the label and the option. A page now has two `.club-summary` elements at a dual meet (Match Finder's and
Standings'), so a selector for Standings' names its panel (`#standingsBody .club-summary`).

## Category chips

On the Hub a dual meet's Standings on a phone have a row of chips that jump to a category
(hub-player-experience-spec §10). `createStandings` takes `jumpChips` (default `false`) and `dualMeetStandingsHTML`
reads it: in the phone branch only (not `isDesktopStandings()`), the club bar and the chips are wrapped in
`<div class="dm-pin">`, before `.standings-board`, with a `<nav class="dm-jump" aria-label="Jump to a category">` of
`<button type="button" class="dm-jump-chip" data-jump-cat="<label>">`, one for each category in `sortedCats` order. A
chip's `data-jump-cat` is its column's `data-cat-key`, and its text is `${divCode} ${evCode}` from
`splitCategory(parseCode(rows[0].teamCode, type).category, display)`, or the category code when the display settings cannot
split it; both are escaped. Without the option, or on a computer, the markup is what it was, so Control Center, which does
not pass it, is unchanged. `apps/hub.js` passes `jumpChips: true` at a dual meet.

`createStandings` finds a chip's column by comparing each `.standings-col`'s `data-cat-key` with the chip's
`data-jump-cat` (a label is never built into a selector) and scrolls the page so the column starts 8 px under the pinned
row: `column top + scrollY - pinned row height - 8`, and the Hub bar's height (`--hub-bar-h`, else 54 px) less when the bar is showing there
(`hub-bar-on`), instant under *reduce motion*. A tap from the top of the page lands with the banner gone and the bar on,
though the bar is not on yet, so `hubBarShowsAt(top)` also answers yes when the target is below the banner (`header.hero`)
and the page has a `.hub-bar`, which only a stylesheet from before the fold rules needs (with them the bar is always
on on a phone). Control Center draws no chips. One passive `scroll` listener, added once when the option is
set and throttled with `requestAnimationFrame`, marks `active` the chip of the last column whose top is no further than the
8 px gap and 4 px of rounding under the pinned row's bottom, reading the DOM each time, and `render()` runs the same
marking after it redraws so a live update keeps the highlight.

`css/standings.css`, under the phone media query, has `.dm-pin` (sticky at the top, `z-index` 6, `--paper`, 6 px under),
`.dm-pin .club-summary{ position:static }` (the bar pins as part of the row), `.dm-jump` (a flex row that scrolls
sideways with no scrollbar) and `.dm-jump-chip` (Barlow Condensed 700 12px, uppercase, a white pill; `.active` is navy),
and the category heads stop pinning under the row (`.dm-pin ~ .standings-board .standings-col-head{ position:static }`).
`css/hub-bar.css` has `body.hub-bar-on .dm-pin{ top: var(--hub-bar-h) }` beside the club bar's own rule. `_tests/unit/views/standings.test.mjs`
covers the markup (the order, the labels and targets, the escaping, the markup without the option and on a computer) and
`hub-behaviour.test.mjs` taps the chips at a phone size, with the bar off and on, by hand-scrolling, and checks that a
computer and Control Center draw none.

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
