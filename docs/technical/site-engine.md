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
    csv  byes  series  matches  standings  codes  golive  progress  teams  awards  attendance  schedule-grid  model
  data/            the outside world
    registry  snapshot  live-channel  api  tokens
  views/           functions that return HTML strings, plus small components
    html  scroll  ticket  finder  live-matches  standings  teams  score-dialog  attendance-view
  apps/            one per kind of page, wiring data, model and views to the page's markup
    hub  schedule-board  scorer  attendance-desk
  css/             ticket  finder  live-matches  standings  teams  score-dialog  attendance
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
`score-dialog.js` and `attendance-view.js`, which are Control Center and the
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
clubs; key order is display order), `scoreEntry` and `attendance`. A successful
fetch is kept in `localStorage` (`sage.registry.lastGood`), and a failed one
falls back to it.

`domain/model.js` `buildDayModel(event, day, snapshot, { now, alwaysLive })`
turns one day's snapshot into a `DayModel`: every facility's matches (and each
facility's own, for the Live Matches board), the standings, the match index by
code, the series games that will never be played, whether the day is live
(`alwaysLive` is Control Center, which always shows scores) and, for a team
event, its `team` data. It is built once per snapshot and handed to every view.

## Extension points

Control Center adds to a shared view through an option that does nothing by
default, never a flag. Each is documented in the JSDoc of the function that
offers it.

| View | Option | What Control Center passes |
| --- | --- | --- |
| ticket | `decorate(m)` → `{ className, attrs, metaHTML }`, `teamLogoHTML(code)` | the click-to-score class, attributes and hint; the dual-meet Hub passes club logos |
| Match Finder | `searchHandlers`, `searchHint`, `delegate` | the match-number search; the team event's own finder |
| Live Matches | `facilityExtras(name, model)` with `extrasEl`, `liveRow`, `emptyMatchupHTML`, `teamLogoHTML` | the facility progress cards; a team event's rows; the dual-meet Hub's club logos |
| Standings | `dualMeetDesktopLayout` (`'grid'` or `'row'`), `rrBracketLayout` (`'grid'` or `'column'`), `clubLogoHTML`, `unresolved` | the row layout; the dual-meet Hub's single table and logos; the set the label warning reads |
| team views | `decorateRow`, `expandAllClass`, `teamLetters`, `searchHint` | the click-to-score row hook; the Teams tab's button class (Control Center); no organiser's team letters and the Hub's own search hint (a team event's Hub) |

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

On `localhost`, `?fixture=<name>` makes the Hub and the schedule board read
`/_fixtures/` instead of the published data, as the scorer and desk pages
always could (`platform.js` `fixtureName()`; inert on the live site).

## The theme contract

A shell's `:root` defines the custom properties the engine's CSS reads
(`--navy`, `--green`, `--paper`, `--ink`, `--radius`, `--card-shadow` and the
rest in `_tests/unit/theme-contract.test.mjs`), and nothing else is a
colour in the engine. A property the CSS reads with a fallback (for example
`--att-bar-bg`) is listed as optional. Where a page differs from the shared
rules, its own rules sit in a labelled block at the end of its `<style>`.

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
