# Spec — the site engine: shared modules for the event pages and Control Center

> **Status: implemented and merged** into `main` of `sage-match-control.github.io`
> (merge `179217f`, 2026-10-06). Built 2026-10-06 in twelve commits (Phases 0 to
> 5; Phase 6 is this documentation) on one branch rather than one per phase,
> which the owner chose, and merged as one. Written against `7fb908e`, where no
> event is registered after 3 October 2026. The merge commit lists the cause-3
> rows. Before and after the merge, the templates were instantiated for one
> finished event per type (standard, dual meet, team) and matched the original
> pages in every view on the real published data, on production as well
> (site commits `2c2526e`, removed in `170de3b`).
>
> **Measured at Phase 5.** The six template pages are shells of 244 lines (the
> standard Hub's `index.html`), 281 (the dual-meet Hub's), 153 and 157 (the two
> schedule boards), 86 (the scorer page) and 62 (the attendance desk page).
> `tools/control-center.html` fell from 9,875 lines at `7fb908e` to 4,405. The
> engine, `lib/v1/`, is 43 files: 6,692 lines of JavaScript and 3,249 of CSS.
> `npm run verify` runs 313 unit tests and 403 comparison cases (Control Center
> over four events, both Hubs and boards, the scorer and desk pages, each at
> phone and desktop width and in three data states).
>
> **The owner's decisions, 2026-10-06.** These are settled, not open
> questions:
>
> - the Hub takes its settings from `events.json` (§4.6);
> - no merge within 3 days of an event, unless the owner overrides it
>   (§0.4);
> - a fix that reached only one page goes to both, as a category (§3.3
>   item 3);
> - CC's own features attach to a shared core through extension points,
>   never flags (§4.10);
> - the six presentation differences found before the build (§3.3 item 4,
>   §12 rows 5–10): keep both for the dual-meet desktop layout and the two
>   round-robin table differences; CC's version for the ticket's pair code
>   and the Live Matches dividers; the Hub's version for phone sizes.
>
> Every drifted function and every differing shared CSS rule at `7fb908e`
> was checked before the build, and §12 rows 1–10 record what that found. The
> only decision left for the build is a cause-5 difference that appears only
> when the harness renders a page. The implementer brings any such difference
> to the owner when it comes up.
>
> This spec is **Phase 8** of the
> [`sage-tools-api` architecture spec](../in-progress/sage-tools-api-architecture-spec.md).
> Phase 8 started as a decision record comparing three ways to stop
> hand-copying code between the site's pages: shared files, a stamping
> script, or keeping the copies under a test suite. The owner chose on
> 2026-10-06. The choice is shared JavaScript modules with a versioning rule,
> and no frontend framework. This spec replaces that record and is the plan
> to build the choice.
>
> **Supersedes, in part:** the consistency and parity layers of the
> [site test suite](../not-started/site-test-suite-spec.md), and the "carry the shared blocks
> byte-identical" steps of the
> [team tournament template](../not-started/team-tournament-template-spec.md). §9 lists what
> changes in each.

The site's pages share a great deal of code, and every shared line is copied by
hand into each page that needs it. This spec moves that code into one set of
JavaScript modules, the **engine**, at `/lib/v1/`.

- Each event page becomes a thin **shell**: its markup, its colours and a few
  settings.
- Control Center imports the same modules for every view it shares with an
  event page.
- The rules (played, BYE, series finals, standings, team results) become pure
  modules with unit tests that run in Node.

The site keeps its defining properties:

- no build step;
- no framework;
- no runtime dependency beyond what pages load today;
- deployed by pushing to GitHub Pages.

Nothing a visitor or an operator sees changes, except the few items §3.3
names.

---

## 0. Read this first (implementer orientation)

You need nothing beyond this spec and the repos to build it. Read §0–§5 before
touching any file, then build the phases of §7 in order.

### 0.1 The workspace

`D:\Personal\SAGE` holds four git repos side by side (the root `CLAUDE.md`
describes each):

| Repo | Role in this spec |
| --- | --- |
| `sage-match-control.github.io/` | **Where all the work happens.** The public site, served by GitHub Pages from `main`. Every path in this spec is relative to this repo unless it says otherwise |
| `event-data/` | Read-only here, except one sentence Phase 6 adds to `config/README.md` (§4.6). Holds `config/events.json`, the event registry, and each event's published snapshots `<event>/data/<day>.json`. The fixtures in §6 are copied from it |
| `sage-tools-api/` | Not changed. `src/sync/domain/facilityCompletion.mjs` is the server's copy of the played/BYE/series rules, which one parity test reads (§6.6). Its `node_modules` has the `puppeteer` that `_templates/hub-pubmat/` borrows; this spec does not use it |
| `sage-docs/` | Docs. Phase 6 updates them (§8) |

Node is 22.23.3 or later. The local static server for manual checks is the
`static-site` entry in `D:\Personal\SAGE\.claude\launch.json`: Python serving
the site repo on port 8123. Open `http://localhost:8123/tools/control-center`
or `http://localhost:8123/events/<key>/`. Module scripts do not load from
`file://` URLs, so always go through a server.

### 0.2 The site today, in brief

- **No build step.** Each page is one HTML file with an inline `<style>` and
  one inline classic `<script>` at the end of `<body>`. Pushing to `main`
  publishes it. GitHub Pages runs Jekyll over the repo, and Jekyll skips any
  top-level folder starting with `_`. That is why `_templates/`, `_fixtures/`
  and (in this spec) `_tests/` are never published.
- **Event pages** are made by copying a template folder into
  `events/<event-key>/` and filling it in (`_templates/CLAUDE.md` is the
  runbook):
  - `_templates/standard-tournament-template/` and
    `_templates/dual-meet-template/` each hold an `index.html` and a
    `schedule.html`. The `index.html` is the **Tournament Hub**: Match
    Finder, Live Matches and Standings for one event. The `schedule.html` is
    the **schedule board**, a venue wall display for one day.
  - `_templates/scorer/scorer.html` is the scorer page. Staff open it from a
    scorer link to enter scores.
  - `_templates/attendance/attendance.html` is the attendance desk page.
    Staff open it from a desk link to check people in.
- **`tools/control-center.html`** ("CC" below) is the operator console: one
  file of 9,875 lines that serves every registered event. It reads its
  settings from `event-data/config/events.json` at runtime and branches on
  each event's `type`: `"standard"`, `"dual-meet"` or `"team"`. It shows
  every view the Hub shows, plus Awards, Mission Control (the organizer
  tab), Attendance and score entry.
- **Data.** Every page reads one JSON snapshot per day from
  `https://sage-match-control.github.io/event-data/<event>/data/<day>.json`.
  It polls every 10 s, and also holds a WebSocket to the live Worker, which
  pushes each new snapshot. A snapshot holds `facilities[]`, and each
  facility carries `matchesCsv`, `standingsCsv` and, for team events,
  `rosterCsv`. The snapshot also carries `isLive`, `lastEditAt` and
  `completedAt` stamps.
- **Finished events stay as they are.** Their folders are never edited by this
  spec, and their pages keep their inline code with live push off
  (`LIVE_BASE_URL = ''`). The finished events are:
  - `events/pnf-x-bup-dual-meet/`
  - `events/pickle-for-sight-2026/`
  - `events/piggleball-2026/`
  - `events/pickledrive-anniversary-2026/`
  - everything under `events/archives/`

### 0.3 Words used in this spec

| Word | Means |
| --- | --- |
| **engine** | everything under `lib/v1/` |
| **shell** | an HTML page whose own script only declares settings and calls one engine `mount…` function |
| **Hub** | an event's `index.html` (Tournament Hub) |
| **CC** | `tools/control-center.html` |
| **live pages** | the pages this spec converts: CC, and the four templates' pages (`index.html`, `schedule.html` of both event templates, the scorer template, the attendance template) |
| **baseline** | the same page at the commit the current phase's branch started from |
| **visible change** | any difference in the rendered text or pixels of a page between baseline and the branch, as measured by the harness of §6 |

### 0.4 Working rules

1. **One branch per phase** in the site repo, named `engine-phase-<n>`, cut
   from `main`. Merge only when the owner says so.
2. **Never near an event.** Do not merge while any event in
   `event-data/config/events.json` has a day whose `date` is today or within
   the next 3 days. A merge publishes at once, and GitHub Pages caches each
   file for 10 minutes. Check `events.json` before every merge. The owner may
   override the 3-day window for a particular merge; only the owner can.
3. **No visible change**, except the items in §3.3 and the rows you add to
   §12. The harness of §6 is what proves it. A phase is done when the harness
   passes against that phase's baseline.
4. **Never edit** the finished events' folders (§0.2), `events/archives/`,
   `tools/` other than `control-center.html`, `field-guide.html`, or anything
   in `sage-tools-api/` or `event-data/` (except the `config/README.md` sentence of §4.6, in Phase 6).
5. **When two copies of a function disagree, record the choice.** Follow
   §5.2, and add a row to §12 for every decision.
6. **Commit messages** start with `Engine phase <n>:`. Follow the workspace
   attribution rules for co-author lines.

---

## 1. The decision

### 1.1 The options that were weighed

1. **Shared files**, loaded by each page. No build step and no copies.
2. **A stamping script** that pastes each shared block into its pages. Pages
   stay one file each, but there is a script to run and a check that it ran.
3. **Keep the copies**, and let a test suite catch drift. Nothing changes in
   the pages, but every copy is still edited by hand.

**Chosen: option 1**, widened from "three marked blocks" to all code shared
between pages (§2 shows why). It uses native ES modules and a versioning rule
(§4.3). Finished events keep their inline code.

### 1.2 Why not a frontend framework

React, Vue, Svelte and the no-build ones (Preact with `htm`, Lit) were all
considered and rejected:

- **The pages are simple to render.** Each page fetches one JSON snapshot,
  turns it into HTML strings, and re-renders when a new snapshot arrives. Page
  state is small: the event, the day, the tab and the search. A framework's
  strength, managing UI state that changes often, is not what the site lacks.
- **The real problems are copying and untested rules.** A framework fixes
  neither. Native modules fix the copying. Pure modules make the rules
  testable in Node.
- **A framework needs a build step.** That means a GitHub Actions build before
  Pages can publish, which brings in a toolchain. On 2026-10-06 a GitHub
  Actions outage already cancelled one Pages deploy.
- **It would mean rewriting about 30,000 lines** that have run real events.
  That rewrite is a bigger risk than the drift it would remove.
- **A no-build framework loaded from a CDN** adds a third-party runtime that
  venue phones depend on, for little gain over plain modules.

Revisit this choice only if CC becomes much more interactive (live editing
grids, offline work) or several people start editing the site. The `domain/`
modules carry over unchanged to any framework.

---

## 2. What is duplicated today

Measured at `7fb908e` by comparing each function's text across pages
(whitespace ignored). `{a, b}` means a and b are identical; separate braces
are copies that have drifted apart.

| Function | Copies |
| --- | --- |
| `parseCSV` | {CC, std index, dm index, std schedule, dm schedule, scorer} |
| `seriesGameOf` | {CC, std index, dm index, std schedule, dm schedule, scorer} |
| `unneededSeriesGames` | {CC, scorer} {std index, dm index, std schedule, dm schedule} |
| `rowsToMatches` | {CC, scorer} {std index, dm index} |
| `rowsToStandings` | {CC} {std index, dm index} |
| `createLiveChannel` (the `LIVE CHANNEL` block) | {CC, std index, dm index, std schedule, dm schedule, scorer} |
| `fetchDaySnapshot`, `fetchDaySnapshotFromPages` | {CC} {std index, dm index} {std schedule, dm schedule} {scorer} |
| `escapeHtml` | {CC, std index, dm index, scorer} |
| `createScoreDialog` (the `SCORE CLIENT` block) | {CC, scorer} |
| `createAttendanceView` (the `ATTENDANCE CLIENT` block) | {CC, attendance template} |
| `httpWords`, `messageFromJson` | {CC, scorer} |
| `friendlyApiMessage` | {CC} {scorer} |
| `ticketHTML`, `renderStageTables` | {CC} {std index} {dm index} |

That table covers the named blocks. The larger duplication is unnamed:

- **CC contains a second Tournament Hub.** The standard template's
  `index.html` defines 83 functions, and CC defines 72 of the same names.
  Only 36 are still identical. The other 36 have drifted: among them
  `rowsToMatches`, `rowsToStandings`, `parseCode`, `ticketHTML`,
  `renderStandings`, `runSearch`, `loadLiveData` and
  `computeDayIsLive`.
- **The two templates are mostly the same code.** The dual-meet and standard
  `index.html` share 68 identical functions.
- **Every event page is a full copy of its template's code**, not just its
  settings. Piggleball's `index.html` differs from its template by 179 lines:
  tokens, settings and colours. The other ~3,100 lines are template code. A
  fix made after an event is created has to be carried into that event's
  pages by hand, which is why commits since September touch 5–9 HTML files at
  once.
- **CSS is copied the same way.** The standard template's `index.html` holds
  1,329 lines of CSS and CC holds 2,638, with the ticket card, standings
  board and Live Matches table in both.

Two more copies are kept by hand today:

- **The event's settings.** A Hub's `DAYS`, `DIVISIONS` and `EVENTS`
  duplicate what `events.json` already holds. That pair is a "kept in sync by
  hand" item in the root `CLAUDE.md`.
- **The rules.** The played/BYE/series rules are in 8 site files plus the
  server's `facilityCompletion.mjs`.

---

## 3. Goals, non-goals and constraints

### 3.1 Goals

1. One copy on the site of every function used by more than one live page.
2. The rules (played, BYE, series, standings, go-live, facility progress, team
   results, awards podiums, attendance parsing) as pure modules with
   `node:test` unit tests.
3. The Hub and CC render their shared views with the same code and CSS.
4. A new event's pages are shells of about 150–250 lines: no copied
   engine code, and settings taken from `events.json` where it already holds
   them.
5. A fix to the engine reaches CC and every unfinished event's pages at once.
6. Finished events keep working unchanged, for good.

### 3.2 Non-goals

- Changing what any page shows or does (beyond §3.3).
- Moving CC's own features out of its file: Mission Control, Awards image
  export, the attendance console, the event picker. Only code CC shares with
  another page moves. §11 lists splitting CC further as a later option.
- A team-type Hub. CC keeps its team views, and they move into the engine
  because CC uses them. A team event's Hub shell is the
  [team template spec](../not-started/team-tournament-template-spec.md)'s job (§9).
- The other tools (`scoresheet-generator.html`, `tournament-calculator.html`,
  `bracket-generator.html`, `tools/index.html`) and `field-guide.html`.
- Any change to `sage-tools-api`, or to `event-data` beyond one documentation sentence (§4.6).

### 3.3 What changes on purpose

1. **The Hub reads its settings from `events.json`** (§4.6): the day list,
   facility names and the division, event and club labels. The page no longer
   declares `DAYS`, `FACILITIES`, `DIVISIONS`, `EVENTS` or `CLUBS`, apart
   from the club logos, which `events.json` does not hold. A visitor sees no
   difference. The Hub makes one extra request on load, and uses the last
   good copy of the settings if that request fails.
2. **Live pages load scripts and stylesheets from `/lib/v1/`.** A page is no
   longer one self-contained file.
3. **A fix that reached only one page reaches both** (§5.2 cause 3). The
   owner approved this as a category on 2026-10-06, so these changes need no
   further approval: the implementer records each one in §12 and lists them
   for the owner before the phase merges. Four are known (§12 rows 1–4):
   - **Series games after the decider.** The Hub greys these out as "Not
     needed" on its tickets and on the matchups in Standings, and never shows
     "Next Up" on one. CC does neither in Match Finder or Standings. After
     this spec, CC does both.
   - **BYE matches.**
     - CC leaves BYE matches out of a pair's list in Match Finder. The Hub
       leaves them out only of the all-matches list.
     - CC drops BYE rows from Standings, and drops the Bronze stage whenever
       a bye decides it (`categoryBronzeIsByeDecided`). The Hub drops Bronze
       only when a Bronze row is itself a BYE.

     After this spec, the Hub does what CC does.
   - **Match Finder's "Teams" count.** The Hub counts only codes ending in a
     bare number: real pairs, not playoff slots like `NWD_SF_1`. CC counts
     every code, slots included, so its count is too high. After this spec,
     CC counts the Hub's way.
   - **BYE entries in the pair search.** CC leaves pairs that are a BYE out of
     Match Finder's autocomplete (`rebuildTeamIndex`); the Hub lists them.
     After this spec, the Hub leaves them out too.
4. **Presentation differences the owner decided on 2026-10-06** (§5.2 cause
   5, §12 rows 5–10). Each one keeps both versions or picks one:
   - **Keep both** (no visible change). The shared core takes the Hub's
     version. CC keeps its own version, either through a named option or in
     its own `<style>`, under the banner described in §7.3:
     1. **Dual-meet Standings on desktop.** The Hub lays it out as a grid:
        one row per division, that division's events across, and leftover
        categories in a side column (`.standings-col-overflow`). CC lays it
        out as one row of 360px columns that scrolls sideways.
        `views/standings.js` takes an option
        `dualMeetDesktopLayout: 'grid' | 'row'`, default `'grid'`; CC
        passes `'row'`. Each layout's CSS goes with it, the Hub's in
        `css/standings.css` and CC's in CC's own `<style>`.
     2. **Long pair names in round-robin tables.** The Hub cuts them with
        "…" and gives the full name on hover (`table-layout:fixed`, the
        ellipsis rules, `pairCell`'s `title`). CC keeps whole names and lets
        the table scroll sideways (`.br-table-wrap`). This split is
        deliberate (site commit `ed58647`). `pairCell` keeps its `title` in
        both; CC's own `<style>` undoes the cut. `renderStageTables` always
        emits the `.br-table-wrap` wrapper, which changes nothing on the Hub,
        where nothing overflows.
     3. **Round-robin table style.** The Hub's tables are cards with rounded
        corners, a shadow and spacing; CC's are plain tables. This goes with
        item 2: CC's own `<style>` keeps the plain look.
   - **Control Center's version, for both pages** (a visible change on the
     Hub):
     4. **The pair code on a Match Finder ticket**
        (`.team-block .team-name`) is a small, muted, uppercase label
        (9.5px, weight 700, letter-spacing .06em), not 14px dark text. The
        names are what the reader looks for.
     5. **Live Matches row dividers** (`table.live-table td`) are
        `2px solid rgba(20,27,44,.4)`, not a faint 1px line.
   - **The Hub's version, for both pages** (a visible change in CC):
     6. **Phone sizes, at most 480px wide.** `.names-row` keeps
        `column-gap:8px`; `.team-block .players` is 14px;
        `.live-team-code` is 10px; `.live-team-players` is 10.5px.
5. **Every further row the implementer adds to §12 under §5.2 cause 5**,
   where the owner agrees to a change in what a page shows.

### 3.4 Constraints

- **No build step, and nothing new in the browser.** Engine files are served
  exactly as written. The only external resources a page loads stay what it
  loads today: Google Fonts, and CC's SRI-pinned `qrcode-generator` from
  cdnjs.
- **Root-absolute paths.** Pages import `/lib/v1/...`, never a relative path,
  for the same reason their assets are root-absolute: moving a folder into
  `events/archives/` must not break it (`_templates/CLAUDE.md` §6).
- **Node can import `domain/` modules.** They use no DOM, no `fetch`, no
  `localStorage` and no page globals, so `node --test` imports them directly.
- **Dev tools only under `_tests/`.** `_tests/package.json` is the only
  package file in the repo. Nothing in `lib/` needs an install.
- **No folder named `vendor`** anywhere under `lib/`. Jekyll's default
  `exclude` list drops `vendor/` paths from the published site.

---

## 4. Target design

### 4.1 Layout

```
lib/v1/
  platform.js            # site-wide constants: data and API addresses, timings, fixture mode
  domain/                # pure: no DOM, no fetch, no storage, no page globals. Node-testable
    csv.js               # parseCSV
    codes.js             # parseCode, category/club/division labels, stage metadata and ordering
    matches.js           # rowsToMatches, match indexes, instance/pair helpers
    byes.js              # BYE_RE, sideIsBye, matchByeSide, isByeStandingRow
    series.js            # SERIES_GAME_RE, seriesGameOf, seriesGroups, walkSeries, unneededSeriesGames
    standings.js         # rowsToStandings, rankStandings, headToHeadWins, isEmptyStanding, fmtQ
    golive.js            # parseScheduleTimeToMinutes, earliestScheduleMinutes, computeDayIsLive
    progress.js          # facility progress (CC's computeFacilityProgress and its pure helpers)
    teams.js             # team type: side codes, pair labels, matchup results, group ranking, rosters
    awards.js            # podium derivation, bronze walkovers, overall champion
    attendance.js        # the ATTENDANCE tab: CSV parsing, grouping, duplicate-name checks
    schedule-grid.js     # the schedule board's court/slot layout and court-range parsing
    model.js             # buildDayModel(event, day, snapshot, options): everything a view needs
  data/                  # browser I/O, no DOM rendering
    registry.js          # fetch events.json, eventConfigFrom(), last-good copy
    snapshot.js          # snapshot URL, fetch with timeout, the poller
    live-channel.js      # createLiveChannel (today's LIVE CHANNEL block, parameterised)
    api.js               # Cloud Run calls: fetch with timeout and token, readable error messages
    tokens.js            # decoding and storing scorer and desk link tokens
  views/                 # DOM: functions returning HTML strings, plus small components
    html.js              # escapeHtml
    scroll.js            # scroll-position capture and restore across re-renders
    ticket.js            # ticketHTML and the pair cell
    finder.js            # Match Finder: team index, autocomplete, search, intro, all-matches list
    live-matches.js      # Live Matches board
    standings.js         # standings: stage tables, standard category cards, dual-meet club columns
    teams.js             # team-type views (matchup cards, group tables, rosters)
    score-dialog.js      # createScoreDialog (today's SCORE CLIENT block)
    attendance-view.js   # createAttendanceView (today's ATTENDANCE CLIENT block)
  apps/                  # one per kind of page: wires data, model and views to a page's markup
    hub.js               # mountHub(settings)
    schedule-board.js    # mountScheduleBoard(settings)
    scorer.js            # mountScorer(settings)
    attendance-desk.js   # mountAttendanceDesk(settings)
  css/
    hub.css  ticket.css  finder.css  live-matches.css  standings.css  teams.css
    score-dialog.css  attendance.css  schedule-board.css  scorer.css  attendance-desk.css
_tests/                  # never published (Jekyll skips "_" folders)
  package.json           # "type": "module", "private": true; devDependencies pinned exactly
  unit/                  # node:test over lib/v1/domain/
  engine/                # the comparison harness of §6
  helpers/               # server, browser, fixtures (shared with the future site test suite)
  fixtures/              # snapshots and registry used by unit tests and the harness
```

The file names above are the target; split or merge a module only if one would
exceed about 800 lines or would hold unrelated things, and record that in §12.
Keep import depth shallow: an `apps/` module imports `views/`, `data/` and
`domain/`; a `views/` module imports `domain/` and `views/html.js`; nothing
imports `apps/`.

### 4.2 Dependency rules

| Files | May import | Never |
| --- | --- | --- |
| `platform.js` | nothing | — |
| `domain/*.js` | other `domain/` files | `platform.js`, `data/`, `views/`, `apps/`; any browser global (`document`, `window`, `fetch`, `localStorage`, `location`, timers) |
| `data/*.js` | `platform.js`, `domain/`, other `data/` | `views/`, `apps/`; `document` |
| `views/*.js` | `domain/`, other `views/` | `data/` (a view is handed data, it never fetches), `apps/` |
| `apps/*.js` | everything above | other `apps/` |
| a page's inline module script | `lib/v1/` | another page |

`_tests/unit/guards.test.mjs` enforces these rules. It reads each file's import
specifiers, and checks that `domain/` files mention none of the browser
globals listed. A `domain/` function that needs the time takes `now` (ms since
epoch) as a parameter.

### 4.3 The versioning rule

The engine lives at a versioned path, and a page names the version it uses in
every import: `/lib/v1/apps/hub.js`.

1. **Only the newest version folder is ever edited.** Today that is `lib/v1/`.
2. **Edits to it must be compatible.** A compatible edit:
   - may add a module, an export, an option or a CSS class;
   - may fix a bug or change a module's internals;
   - never removes or renames an export, a CSS class a page's own markup or
     CSS uses, or a CSS custom property the theme contract lists (§4.7);
   - never changes an export's parameters or the shape of what it returns, in
     a way an existing caller would notice.
3. **Mixed caching must stay safe.** GitHub Pages lets a browser keep a file
   for up to 10 minutes. So within a version, one module may only start using
   another module's export after that export has been published in an
   **earlier** push. In practice: an edit that adds an export and its first
   caller in different modules is two pushes, the export first.
4. **An incompatible change starts a new version.**
   - Copy `lib/v1/` to `lib/v2/` and make the change there.
   - Move CC, the templates and every **unfinished** event's pages to `v2`.
   - From then on `lib/v1/` is frozen: never edited again, never deleted.
     Finished events that use `v1` keep working on the code they were built
     with.
5. **A finished event's shell never moves version.** When an event finishes,
   its pages keep the version they have.

A page built on `v1` therefore gets every later compatible fix, even after its
event has finished. That is accepted: fixes reach finished events too, and
nothing that could break them is allowed in place.

### 4.4 The two shapes every view uses

Define both as JSDoc typedefs at the top of `domain/model.js`. Every module
that takes them refers to them with `import("./model.js")` type references.

```js
/**
 * One event, as every engine page sees it. Built by data/registry.js's
 * eventConfigFrom(registry, eventKey) from event-data/config/events.json.
 * @typedef {object} EventConfig
 * @property {string} key                       the event key, e.g. "piggleball-2026"
 * @property {"standard"|"dual-meet"|"team"} type
 * @property {string} title
 * @property {DayConfig[]} days                 in the order events.json lists them
 * @property {{ divisions?: Record<string,string>, events?: Record<string,string>,
 *              clubs?: Record<string,string> }} display
 * @property {"links"|"console"|undefined} scoreEntry
 * @property {string|undefined} attendance
 *
 * @typedef {object} DayConfig
 * @property {string} key                       e.g. "piggleball-day1"
 * @property {string} label                     e.g. "Oct 3"
 * @property {string|undefined} date            "YYYY-MM-DD", Asia/Manila
 * @property {string[]} facilities              facility names, in order
 */

/**
 * One day's data, derived once per snapshot and handed to every view.
 * Built by buildDayModel(event, day, snapshot, { now, alwaysLive }).
 * @typedef {object} DayModel
 * @property {EventConfig} event
 * @property {DayConfig} day
 * @property {object|null} snapshot             the raw snapshot, null before the first load
 * @property {boolean} isLive                   computeDayIsLive, or true when alwaysLive (CC)
 * @property {Match[]} matches                  every facility's matches, sorted by number
 * @property {Map<string, Match[]>} matchesByFacility
 * @property {Map<string, Match>} matchByCode
 * @property {Set<Match>} unneededGames         series games after the decider
 * @property {object[]} standings
 * @property {object|undefined} team            team-type data (matchups, ranking, roster), team events only
 */
```

`Match` is what `rowsToMatches` returns today in CC: `num`, `time`, `court`,
`liveCourt`, `matchUp`, `t1`, `t1p1`, `t1p2`, `t2`, `t2p1`, `t2p2`,
`t1Score`, `t2Score`, `played`. Add fields to `DayModel` as the views need
them, and keep the typedef current.

Today the pages keep this state in mutable globals: `MATCHES`,
`MATCH_BY_CODE`, `UNNEEDED_GAMES`, `STANDINGS`, `dayIsLive`, and in CC
`CURRENT_TYPE` and `DISPLAY`. Shared code reads those globals directly. In the
engine it receives a `DayModel` (or an `EventConfig`) as a parameter instead.
That conversion is most of the work in Phases 1–3.

### 4.5 What a shell looks like

A Hub shell after Phase 5. Everything the page shows before scripts run (the
hero, the tab bar, the empty view containers) stays as markup in the page.
`_templates/hub-pubmat/render.mjs` reads the `<title>`, the hero's
`.eyebrow`, the QR `<img>` and `.qr-link-text` from it.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>{{EVENT_TITLE}} — Tournament Hub</title>
<!-- meta, favicons, Google Fonts: exactly as the template has them today -->
<link rel="stylesheet" href="/lib/v1/css/hub.css">
<style>
  /* ============================ THEME ============================
     The event's colours: the custom properties of §4.7, and nothing else
     unless the event needs its own touches below. */
  :root{ --navy:#0B2545; --green:#3BAA5C; /* … */ }
</style>
</head>
<body>
  <!-- the hero, tabs and view containers: the template's markup today, unchanged -->
<script type="module">
// ============================================================================
// SETTINGS — the only script in this page. Days, facilities and labels come
// from event-data/config/events.json (see sage-docs/docs/specs/.../site-engine-spec.md §4.6).
// ============================================================================
import { mountHub } from '/lib/v1/apps/hub.js';

const EVENT_KEY = '{{EVENT_KEY}}';
// The live Worker's address, or '' once the event has finished (live push off).
const LIVE_BASE_URL = 'wss://sage-live.sagematchcontrol.workers.dev';

mountHub({ eventKey: EVENT_KEY, liveBaseUrl: LIVE_BASE_URL });
</script>
</body>
</html>
```

A dual-meet shell also passes `clubLogos: { '<CLUB>': '<path>' }`. A schedule
board shell passes `dayKey` and `categoryColors`, today's `CAT_META`. Each
`mount…` function documents its settings in a JSDoc block, and rejects an
unknown setting with a console error that names it.

Keep the constant names `EVENT_KEY`, `LIVE_BASE_URL` and (schedule board)
`DAY_KEY` exactly. The harness (§6.3) and `_templates/CLAUDE.md` find them by
name.

### 4.6 Settings from `events.json`

`data/registry.js` exports:

- `loadRegistry({ fixture })`. It fetches `events.json` with the
  same cache-busting `?t=` and timeout CC uses today. On success it stores the
  text in `localStorage` under `sage.registry.lastGood`, and returns the parsed
  object. If the fetch fails, it returns the stored copy if there is one, and
  otherwise throws.
- `eventConfigFrom(registry, eventKey)`, which returns an `EventConfig`.
  CC's `selectEvent` builds the same thing today; use its reading of
  `events.json` (`type`, `title`, `days`, `display`, `scoreEntry`,
  `attendance`) as the definition.

The scorer and attendance pages already read `events.json`; they move onto
`loadRegistry` too. CC moves onto it in Phase 2.

**The rule this adds:** an event stays in `events.json` for as long as any page
built on the engine shows it. Removing a finished event's entry would blank its
Hub. Add that sentence to `event-data/config/README.md`. That is a
documentation change in `event-data` that Phase 6 makes; it is the one edit
that repo gets.

### 4.7 The theme contract

The engine's CSS uses only these custom properties for colour, radius and
shadow. Every shell defines them in its `:root`:

```
--navy --navy-deep --navy-mid --green --green-dark --paper --paper-dim
--line --ink --ink-soft --white --muted --amber --cork
--court --court-dark --court-line --radius --card-shadow
```

That is the list in both templates' and CC's `THEME` blocks today. The
schedule board has its own palette: list its properties at the top of
`css/schedule-board.css` in Phase 4, and treat that list as part of the
contract too. A property the engine CSS reads must be in a contract list;
`_tests/unit/theme-contract.test.mjs` greps `lib/v1/css/*.css` for `var(--…)`
and fails on any name not listed.

### 4.8 Fixture mode

Today CC, PickleDrive's Hub and the scorer and attendance pages accept
`?fixture=<name>` on `localhost` or `127.0.0.1`. The registry then comes from
`/_fixtures/config.json`, and each snapshot from
`/_fixtures/<event>/<name>.json`. The templates' Hub and schedule board do
not. `platform.js` exports `fixtureName()`, which returns the name only on
those two hosts, and the data layer uses it everywhere. So every engine page
gains fixture mode. This is a local-only addition, invisible on the live site.

### 4.9 Converting a classic script to a module script

Each live page's inline `<script>` becomes `<script type="module">`. The
differences that matter here:

- **Strict mode.** Assigning to an undeclared name throws. Before converting a
  page, run it once in the harness and fix any such assignment in the same
  commit.
- **Top-level names are no longer globals.** `window.foo`, inline `on…=`
  attributes and console debugging by global name stop working. At
  `7fb908e` no live page uses inline handlers or assigns to `window.*`, so
  nothing depends on this. Do not add either.
- **Module scripts run deferred**, after the document is parsed. The current
  scripts already sit at the end of `<body>`, so the order does not change.
- **Imports are hoisted.** Put all `import` lines at the top of the script.
- **`file://` stops working.** Module scripts are fetched with CORS, so a page
  must be opened through a server (§0.1).

### 4.10 Shared core, with Control Center adding to it

Much of the drift between the Hub and CC comes from features only CC has. The
two pages split the work like this:

- **`views/` and `domain/` hold the shared core:** what every page that
  shows a view should see, with every fix from either side (§3.3 item 3).
  They know nothing about operators. There is no mention of score entry,
  sign-in, tokens, Mission Control or the event picker anywhere under
  `views/` or `domain/`. `guards.test.mjs` enforces it (§6.6).
- **Features only CC has stay in CC's own script.** They attach to the core
  through **named extension points**: hooks and callbacks that the core
  calls and that do nothing by default.
- **Hub-only behaviour lives in `apps/hub.js`.** That covers hiding scores
  until the day is live (`DayModel.isLive`), the saved day, tab and search,
  and the day picker's gating.
- **Differences by event type stay in the core**, branching on
  `event.type`. Every type will eventually have a Hub.

Never add a flag that names a page (`mode: 'hub' | 'console'`, `isConsole`).
A flag inside the core means every new CC feature edits the module the public
Hub runs. A hook means a CC-only feature is a CC-only change. Adding a hook is
also a compatible change under §4.3.

The starting set of extension points, taken from the drift found at
`7fb908e`:

| Core function | Extension point | What CC passes |
| --- | --- | --- |
| `ticketHTML(m, model, opts)` | `opts.decorate(m) → { className?, attrs?, metaHTML? }` | the `scoreable` class, the data attributes and the score hint (today's `scoreAttrs` / `scoreHintHTML`) |
| Match Finder (`createFinder(el, opts)` or the functions it wraps) | `opts.searchHandlers: [(query, model) → html \| null]`, tried in order before the pair search | the match-number search (today's `MATCH_NUMBER_SEARCH_RE` branch and `renderMatchByNumber`) |
| Match Finder | `opts.onTicketClick(m, event)` | opening the score dialog (today's `openScoreFromFinder`) |
| Live Matches | `opts.facilityExtras(facility, model) → html` | the facility progress card (today's `facilityProgressCardHTML` / `facilityStatusProgressHTML`) |
| Standings | `opts.beforeStandings(model) → html` | the warning for categories `events.json` cannot label (today's `renderResolutionWarning`) |
| `buildDayModel(event, day, snapshot, opts)` | `opts.alwaysLive` | `true`: CC always shows scores |
| Standings (`views/standings.js`) | `opts.dualMeetDesktopLayout: 'grid' \| 'row'`, default `'grid'` | `'row'`: §3.3 item 4.1, §12 row 5. It is named for the layout, not the page, so it is allowed under the no-flag rule |

The table is the starting set, not a limit. When reconciling a function
shows another CC-only difference, add an extension point of the same kind
and a row to this table in §12. Do not add a flag. Each extension point is
documented in the JSDoc of the function that offers it. Its default (no hook
passed) is exactly what the Hub shows.

---

## 5. Where each function goes

### 5.1 The map

"Start from" is the copy to move into the module. Where the copies differ,
§5.2 decides how they are reconciled. Functions not listed stay in their page
until Phase 3 or 4 moves them with the view or app that owns them.

| Module | Functions and constants | Start from |
| --- | --- | --- |
| `domain/csv.js` | `parseCSV` | any (all identical) |
| `domain/series.js` | `SERIES_GAME_RE`, `seriesGameOf`, `seriesGroups`, `walkSeries`, `unneededSeriesGames` | CC (its `unneededSeriesGames` is built on `seriesGroups`/`walkSeries`; the Hub and schedule board copies are one 25-line function) |
| `domain/byes.js` | `BYE_RE`, `sideIsBye`, `matchByeSide`, `isByeStandingRow`; the Hub's `matchSideIsBye` | CC; replace `matchSideIsBye` with `matchByeSide` if equivalent |
| `domain/matches.js` | `rowsToMatches`, `buildMatchByCodeIndex`, `matchInstanceOf`, `isUnnamedScheduledPair`, `pairKey` | CC (`rowsToMatches` adds `matchUp`) |
| `domain/codes.js` | `parseCode`, `splitCategory`, `categoryLabel`, `clubLabel`, `divisionLabel`, `divisionEventLabel`, `buildCategoryOrderIndex`, `categorySortKey`, `roundKeyword`, `roundLabel`, `ROUND_KEYWORD_LABELS`, `standingsStageKey`, `STANDINGS_STAGE_KEY_BY_ROUND_KEYWORD`, `STAGE_META`, `STAGE_ORDER`, `CROSS_CLUB_STAGE_KEYS`, `BADGE_CLASSES` | CC (its `parseCode` splits by position and reads labels from `display`, which is what the Hub uses after §4.6; the Hub's version builds a regex from `DIVISIONS`/`EVENTS`). Each function takes the `EventConfig` or its `type`/`display` as a parameter instead of reading `CURRENT_TYPE`/`DISPLAY` |
| `domain/standings.js` | `rowsToStandings`, `rankStandings`, `headToHeadWins`, `isEmptyStanding`, `fmtQ` | CC |
| `domain/golive.js` | `parseScheduleTimeToMinutes`, `earliestScheduleMinutes` (the Hub calls it `earliestScheduleMinutesFrom`), `computeDayIsLive(day, snapshot, now, leadHours = 4)` | the Hub. CC's own `computeDayIsLive` (always true) becomes `buildDayModel`'s `alwaysLive` option. CC's `describeAutoGoLive`, `formatMinutesAsClock` and `currentConfiguredIsLive` (Mission Control's go-live control) use these and stay in CC |
| `domain/progress.js` | `eventScheduleMinutes`, `eventDayNowMinutes`, `eventDayMinutesFromISO`, `isEventDayActive`, `eventClock`, `formatGap`, `computeFacilityProgress`, `facilityActualEnd`, `facilityDataIsStale`, `facilityProgressFor`, `facilityFinishParts` | CC. The ones reading `localStorage` (`loadFacilityEnds`, `saveFacilityEnds`, `facilityEndKey`) stay in CC and pass their values in |
| `domain/teams.js` | CC's team block's pure functions: `numberRepeatedPairs`, `parseSideCode`, `sideOf`, `pairOf`, `stageOf`, `slotOf`, `isGroupTeamCode`, `baseTeamOf`, `teamNameOf`, `sideLabel`, `groupOf`, `stageLabel`, `pairLabel`, `teamMatchupResult`, `teamRowsToRoster`, `teamRankBracket`, and the data-building half of `rebuildTeamData` | CC. PickleDrive's copy stays inline and frozen (a finished event) |
| `domain/awards.js` | `nonByeCode`, `matchCategoryOf`, `matchRestOf`, `resolveDecisiveMatch`, `winnerOfMatch`, `loserOfMatch`, `medalistFromTeam`, `categoryBronzeIsByeDecided`, `standingsBronzeWalkover`, `standingsTop3ForCategory`, `categoryRoundRobinDone`, `buildPodiums`, `computeOverallChampion`, `teamMedalist`, `buildTeamPodium` | CC (only CC uses them; they move so the bronze-walkover and BYE interplay gets unit tests) |
| `domain/attendance.js` | `attParseCsv`, `attClockTime`, `attSplitList`, `parseAttendanceCsv`, `defaultCategoryLabel`, `groupForDesk`, `attLettersOnly`, `attOneEditApart`, `possibleDuplicates`, `sameNameSameCategory`, `ATTENDANCE_SHIRT_HEADERS` | the `ATTENDANCE CLIENT` block (identical in its copies) |
| `domain/schedule-grid.js` | the schedule board's pure functions: `parseCourtAssignment`, `assignCourts`, `buildScheduleData`, `chunkBalanced`, `parseCourtsParam`, `courtsToParam`, `scoreOrNull`, `nameOrNull`, `readableOn`, and `buildPrintPages` if it touches no DOM; the dual-meet board's `clubOf` | std `schedule.html`, plus `clubOf` from dm `schedule.html` |
| `domain/model.js` | `buildDayModel` (new): what `loadLiveData` computes today before rendering | CC's `loadLiveData` and the Hub's |
| `data/snapshot.js` | `snapshotUrlFor`, `fetchDaySnapshotFromPages`, `fetchDaySnapshot`, and a `createPoller({ intervalMs, onTick })` that pauses while `document.hidden`, as each page's own timer does today | CC's (four drifted copies: reconcile per §5.2) |
| `data/live-channel.js` | `createLiveChannel({ baseUrl, enabled, onSnapshot })` and `LIVE_PING_MS`, `LIVE_PONG_TIMEOUT_MS`, `LIVE_SAFETY_POLL_MS`, `LIVE_RETRY_MAX_MS` | the `LIVE CHANNEL` block (identical); `LIVE_BASE_URL` becomes the `baseUrl` setting |
| `data/registry.js` | `loadRegistry`, `eventConfigFrom` (§4.6) | CC's `configUrlFor`, and its config reading in `selectEvent` |
| `data/api.js` | `httpWords`, `HTTP_WORDS`, `messageFromJson`, `friendlyApiMessage`, a `fetchJson(url, { method, body, token, timeoutMs })` | CC (`friendlyApiMessage` drifted: reconcile) |
| `data/tokens.js` | `scorerDecode`, `scorerLoadToken`, `deskDecode`, `deskLoadToken`, and their storage keys | the scorer and attendance templates |
| `platform.js` | `GHPAGES_OWNER`, `GHPAGES_REPO`, `CLOUD_RUN_BASE_URL`, `FETCH_TIMEOUT_MS`, `POLL_INTERVAL_MS`, `ATTENDANCE_POLL_MS`, `fixtureName()` | the values every page has today (they are identical) |
| `views/html.js` | `escapeHtml` (and the schedule board's `esc`, if equivalent) | any |
| `views/scroll.js` | `SCROLLABLE_SELECTOR`, `scrollNodeIdentity`, `captureScrollPositions`, `restoreScrollPositions` | CC |
| `views/ticket.js` | `ticketHTML`, `pairCell`, `standingRowClass` | three drifted copies: reconcile |
| `views/finder.js` | `rebuildTeamIndex`, `refreshFinder`, `renderAutocomplete`, `selectAcItem`, `updateClearBtn`, `resolveTeam`, `runSearch`, `renderIntro`, `allMatchesHTML` | reconcile (§5.2). CC's `renderMatchByNumber`, `scoreAttrs`, `scoreHintHTML` and `openScoreFromFinder` stay in CC and attach through §4.10's extension points |
| `views/live-matches.js` | `courtSortKey`, `courtNumberFrom`, `buildFacilityCourtGroups`, `liveEmptyRowHTML`, `liveTableRowsHTML`, `facilityTableHTML`, `renderLiveMatches` | CC (it groups courts from the snapshot's facilities; the Hub's `buildLiveCourts`, `groupLiveCourtsByFacility` and `facilityForCourt` read courts ranges from `FACILITIES`, which §4.6 removes) |
| `views/standings.js` | `renderStageTables`, `sortStageKeys`, `renderStageBlock`, `pairUpMatchups`, `scoreForTeamCode`, `matchupSideHtml`, `matchupScoreHTML`, `matchupInstanceLabel`, `renderMatchup`, `isDesktopStandings`, `DESKTOP_STANDINGS_QUERY`, `renderStandings` and its type branches (CC's `renderDualMeetStandings`, `renderStandardStandings`, `renderClubSummary`, `renderClubSubsection`, `renderSharedStages`, `renderCategoryColumn`, `renderCategoryCard`, `renderCategoryToggleBar`, `renderCatAutocomplete`, `selectCategory`, `updateClearCatBtn`, the Hub's `renderStandingsBody`) | reconcile |
| `views/teams.js` | CC's team views: `teamRosterListHTML`, `teamRosterCardHTML`, `renderTeamRosters`, `sideChipHTML`, `teamMatchupRowHTML`, `teamMatchupCardHTML`, `teamGroupTableHTML`, `renderTeamStandings`, `teamLiveTableRowsHTML`, the team finder functions (`teamRebuildIndex` … `allMatchupsHTML`) | CC |
| `views/score-dialog.js` | `createScoreDialog` and its constants | the `SCORE CLIENT` block (identical) |
| `views/attendance-view.js` | `createAttendanceView`, `markPerson`, `attendanceCsvUrl`; `ATTENDANCE_CSS` moves to `css/attendance.css` | the `ATTENDANCE CLIENT` block (identical) |
| `apps/hub.js` | `mountHub`: the Hub's `selectDay`, day picker, `showView`, `revealLiveTabsAfterLoad`, saved state (`saveDayIndex`, `loadSavedDayIndex`, `saveSearchState`, `loadSavedSearch`, `clearSavedSearch`, `saveView`, `loadSavedView`), polling and the live channel | the templates' `index.html`. Keep the storage keys exactly (`${EVENT_KEY}.dayIndex`, `.search`, `.view`) so returning visitors keep their state |
| `apps/schedule-board.js` | `mountScheduleBoard`: the rest of `schedule.html`'s script | std `schedule.html`, with the dual-meet club styling as an option |
| `apps/scorer.js` | `mountScorer`: the rest of the scorer template's script | the scorer template |
| `apps/attendance-desk.js` | `mountAttendanceDesk`: the rest of the attendance template's script | the attendance template |

### 5.2 Reconciling copies that have drifted

At `7fb908e`, 36 functions the Hub and CC both define have drifted apart.
Most differ by a few lines. Classify each difference, line by line, by its
cause. A function can have several causes at once.

| # | Cause | Seen in (examples) | What to do |
| --- | --- | --- | --- |
| 1 | **A CC-only feature** built into the shared function: score entry, the match-number search, the event picker, CC's extra tabs | `ticketHTML`, `runSearch`, `showView`, `refreshFinder`, `renderAutocomplete`, `selectAcItem`, `resolveTeam` | Keep the core free of it. Move the CC-only lines into CC's script, attached through an extension point (§4.10) |
| 2 | **Different settings sources.** The Hub reads `FACILITIES` court ranges and its `DIVISIONS`/`EVENTS` constants; CC reads the snapshot's facilities and `events.json`'s `display` | `ticketHTML` (`facilityForCourt` vs `m.facility`), `parseCode`, `divisionLabel`, `divisionEventLabel`, `categorySortKey`, the Live Matches grouping | Take CC's version. After §4.6 both pages read the same `EventConfig` and `DayModel`, so this difference disappears |
| 3 | **A fix that reached only one page** | the four cases in §3.3 item 3 (§12 rows 1–4) | Put the fix in the core, so both pages get it. Pre-approved (§3.3 item 3): record it in §12 and in `accepted.mjs` (§6.5), and do not stop to ask |
| 4 | **A difference by event type**: CC branches on `type` where a template has only one type | `parseCode`, `renderStandings`, `rowsToStandings`, `loadLiveData` | Take CC's branches, keyed on `event.type` |
| 5 | **None of the above**: a difference in what a page shows, which neither copy explains as a feature, a fix or a type | the six already decided in §3.3 item 4 (§12 rows 5–10) | Build those as decided. For any other, stop and ask the owner: show both outputs on the same fixture, and record the answer. "Keep both" is a valid answer: the core takes the Hub's version, and CC keeps its own through a named option or its deliberate-differences block (§7.3) |

Wording, comments, variable names and wrapper markup that change nothing
visible (cause 0) need no decision. Take CC's text, unless the harness shows a
visible difference, in which case it is not cause 0.

How to tell cause 3 from cause 5: a fix makes a rule correct where it was not.
Examples are a BYE shown as a playable match, or a match after the decider
offered as "Next Up". A rendering preference is cause 5 even when one copy is
newer, for example a different layout or label for the same correct data.

Record every reconciliation in §12: the function, the copies, the cause
(1–5), what was kept, and the harness case or unit test that proves it.
Causes 3 and 5 also need a line in `_tests/engine/accepted.mjs` (§6.5),
because they change what a page shows. Before a phase merges, list its
cause-3 rows for the owner in the merge request.

Then prove each rule function moved into `domain/` behaves like the page code
it replaces. `_tests/unit/characterization/` runs both on the fixtures:

- It extracts the old function's text from the baseline page, using the
  brace-matching extractor in `_tests/helpers/extract.mjs`.
- It evaluates the old function in a `node:vm` context, with the constants it
  reads (`BYE_RE`, `SERIES_GAME_RE`, …) also extracted from the page.
- It runs the old and new versions on every fixture snapshot of §6.2 and
  asserts equal results.

A deliberate difference (a field added, or a cause-3 fix) is asserted explicitly, not left to the equality check. Once
Phase 5 has merged, delete the characterization tests: the unit tests are the
record from then on.

---

## 6. The comparison harness (Phase 0)

The harness is what makes "no visible change" checkable. It runs the same page
in two trees, the baseline and the branch, on the same data, clock and screen
size, and compares what each shows.

### 6.1 Tooling

- **Runner: `node:test`.** It is the same runner `sage-tools-api` uses.
- **Browser: Playwright,** the `playwright` library rather than the
  `@playwright/test` runner, Chromium only. Pin it to an exact version in
  `_tests/package.json`, with no `^`.
- **Image comparison:** `pixelmatch` and `pngjs`, also pinned exactly. They
  are used only to draw a diff image when two screenshots differ.

These choices match the [site test suite](../not-started/site-test-suite-spec.md) §4.1, so
that suite builds on this harness later instead of beside it. Add
`_tests/node_modules/` and `_tests/out/` to the repo's `.gitignore`.

```
_tests/
  package.json
  helpers/
    server.mjs       # startSite({ root, instantiate }) → { baseUrl, close }
    browser.mjs      # openPage(): context, clock, router, error collector
    extract.mjs      # pull a function or `const NAME = …` out of a page's text
    fixtures.mjs     # load fixtures; derive pre/mid states
    baseline.mjs     # check out the baseline commit into a temp git worktree
  fixtures/          # §6.2
  engine/
    cases.mjs        # §6.4: every page × fixture × state × view × viewport
    accepted.mjs     # §6.5
    compare.test.mjs # runs every case on both trees and compares
  unit/
  out/               # git-ignored: screenshots and diff images of failing cases
```

`package.json` scripts:

| Script | Runs |
| --- | --- |
| `npm test` | `node --test unit/` (no browser) |
| `npm run compare` | `node --test engine/` |
| `npm run verify` | both |

### 6.2 Fixtures

Copy these into `_tests/fixtures/`, unchanged:

| File | From `event-data/` | Covers |
| --- | --- | --- |
| `piggleball-2026/piggleball-day1.json` | `piggleball-2026/data/piggleball-day1.json` | standard, one facility, a series final |
| `pickle-for-sight-2026/pickle-for-sight-day1.json` | `pickle-for-sight-2026/data/pickle-for-sight-day1.json` | standard, two facilities |
| `pnf-x-bup-dual-meet/pnf-x-bup-day1.json` | `pnf-x-bup-dual-meet/data/pnf-x-bup-day1.json` | dual meet |
| `pickledrive-anniversary-2026/pickledrive-anniversary-2026-day1.json` | `pickledrive-anniversary-2026/data/pickledrive-anniversary-2026-day1.json` | team (CC only) |
| `config.json` | `config/events.json`, with only those four events kept and each `sheetId` replaced by `"test"` | the registry |

Also reuse the attendance CSVs already in `_fixtures/` (the site repo's
existing fixture folder) for the attendance cases.

`fixtures.mjs` derives three **states** from each snapshot. Every case runs in
each state:

- **`final`**: the snapshot as published.
- **`pre`**: every `team1Score`/`team2Score` and live `court` cell blanked in
  each facility's `matchesCsv`, and `completedAt` removed.
- **`mid`**: scores kept only on the lower-numbered half of matches. The two
  lowest-numbered unplayed matches get a live `court`.

`standingsCsv` is left as published in all three. The harness compares two
renderings of the same input, so the input does not need to be internally
consistent.

### 6.3 Serving the two trees

- **Baseline.** `baseline.mjs` runs `git worktree add --detach <tmp> <base>`,
  where `<base>` is `git merge-base HEAD main`. It removes the worktree when
  the run ends. The `BASE` environment variable overrides `<base>`.
- **Branch.** The working tree.

`startSite` serves a tree on `127.0.0.1`, port 0. It maps an extensionless
path to `.html`, as GitHub Pages does. It instantiates any file under
`_templates/` when serving it:

1. **Tokens.** Every `{{TOKEN}}` is replaced from the case's token table, and
   an unknown token answers `500` with its name. The token list is in
   `_templates/CLAUDE.md` §3. Image tokens point at `/assets/logo.png`.
2. **Settings.** For each of `EVENT_KEY`, `DAYS`, `FACILITIES`, `DIVISIONS`,
   `EVENTS`, `CLUBS`, `DAY_KEY` and `CAT_META` that the file declares as a
   top-level `const NAME =`, the initializer is replaced with the case's
   value, written as a JavaScript literal. The baseline templates declare all
   of them. After Phase 5 a shell declares only `EVENT_KEY` (and `DAY_KEY` on
   the schedule board), and reads the rest from the routed `events.json`.
   One case table therefore drives both trees.

The settings for each fixture event come from that event's own pages:

- `piggleball-2026`, `pickle-for-sight-2026` and `pnf-x-bup-dual-meet` each
  have an `index.html` and `schedule.html` under `events/`;
- `extract.mjs` reads their `DAYS`, `FACILITIES`, `DIVISIONS`, `EVENTS`,
  `CLUBS` and `CAT_META` once, and `cases.mjs` stores the result.

So the templates are compared on real events' data with those events' real
settings.

### 6.4 Opening a page and comparing it

`openPage(browser, url, options)` gives both trees identical conditions:

| What | How |
| --- | --- |
| Clock | `page.clock.install({ time })` before navigation, at the fixture day's `date` 13:00 +08:00, timezone `Asia/Manila`. Time moves only through the case's own steps |
| Data | `page.route` answers `https://sage-match-control.github.io/event-data/config/events.json*` with `fixtures/config.json`, and `…/event-data/<event>/data/<day>.json*` with the case's derived snapshot |
| Live Worker | `page.routeWebSocket(/sage-live/, ws => ws.close())`: the page falls back to polling, as it does when the Worker is down |
| Cloud Run | `page.route('https://sage-tools-api-*.run.app/**', …)` answers from the case's handler table; an unhandled request fails the case |
| Fonts | Google Fonts requests answer `200` with an empty body, so text renders in fallback fonts the same way in both trees |
| Anything else leaving `127.0.0.1` | fails the case with its URL |
| Motion | `page.emulateMedia({ reducedMotion: 'reduce' })`, and a style tag setting `*{transition:none!important;animation:none!important;caret-color:transparent!important}` added in both trees |
| Errors | any `pageerror` or `console.error` fails the case, unless the case lists that message |
| Storage | a fresh browser context per case |

Each **case** is `{ page, fixture, state, viewport, steps, root }`:

- `steps` is a list of actions: click a tab, type into the search box and pick
  the first suggestion, choose a standings category, open the score dialog
  for match N, fill scores, press Review, and so on.
- After the last step the harness waits for the network to go idle, then
  captures:
  - the `innerText` of `root` (default `body`);
  - a full-page PNG screenshot.

A case passes when both captures are identical between the trees. When it
fails, it writes both PNGs, a pixelmatch diff image and a text diff to
`_tests/out/<case-id>/`.

**Viewports:** phone 375 × 812 and desktop 1280 × 800 for every case.

**The cases.** `cases.mjs` generates them from the table below. Every Hub
and schedule board row runs on each of its fixtures, in all three states.

| Page | Fixtures | Views and steps |
| --- | --- | --- |
| std `index.html` | piggleball, pickle-for-sight | Match Finder (default list); a search for the first team in the autocomplete; Live Matches; Standings (default category, then the second category) |
| dm `index.html` | pnf | the same, plus the club summary in Standings |
| std `schedule.html` | piggleball, pickle-for-sight | default; `?compact=1`; `?venue=<second facility>` (pickle-for-sight only); `?courts=1-2`; print media (`page.emulateMedia({ media: 'print' })`) |
| dm `schedule.html` | pnf | default; `?compact=1`; print media |
| CC | all four, chosen in its event picker | Match Finder; a search by match number; Live Matches with facility progress; Standings; Awards; Teams (team only); Mission Control signed out; Attendance tab signed out |
| CC, signed in | piggleball | Mission Control signed in (seed the stored token `loadAuthToken` reads, per §6.4.1); Match Finder with score entry: open match 1's dialog, enter 11–5, Review, Save (routed `PUT …/score` → 200) |
| scorer template | piggleball | the list; a facility filter; open the first playable match, enter 11–9, Review, Save (routed → 200); the same save answered `409` with `current` |
| attendance template | the attendance fixture in `_fixtures/attendance-demo-2026/` | the list; mark one person (routed `PUT …/attendance` → 200); unmark them |

#### 6.4.1 Tokens for the scorer and desk pages

Neither page checks a token's signature in the browser. `scorerDecode`
(scorer template) and `deskDecode` (attendance template) only read the
payload. The harness builds `base64url(JSON.stringify({ scope, day, exp })) +
'.x'`:

- `scope: 'score-desk'` for the scorer page, or the scope `deskDecode`
  expects for the desk page (read it from that function);
- `day` set to the fixture's day key;
- `exp` an hour after the case's clock.

It passes the token the way the page's link carries it; read that from
`scorerStart` and `deskStart`. CC's signed-in state is the same idea. The
harness seeds whatever `loadAuthToken` reads, with `expiresAt` an hour after
the clock.

### 6.5 Accepted differences

`engine/accepted.mjs` exports a list of
`{ case: <case-id pattern>, kind: 'text'|'pixels'|'both', reason, row }`.
`row` is the §12 row number that records the decision. A failing case that
matches an entry is reported as **accepted** and does not fail the run. Keep
the list short. An entry is allowed only for §3.3 items 3, 4 and 5: a fix
carried to the other page (§5.2 cause 3), or an owner-approved cause-5
change.

### 6.6 Unit tests

`_tests/unit/` holds the following tests.

- **`domain/<module>.test.mjs` for each `domain/` module.** They are table
  tests using the fixtures and hand-written cases. They must cover at least:
  - the played rule: both scores present;
  - BYE by team code and by each player name, case-insensitive;
  - series finals: twice-to-beat for both seats, best-of-3, games after the
    decider marked unneeded;
  - go-live: `true`, `false`, `"auto"` before and after the 4-hour lead, no
    parseable times, no date;
  - facility progress with and without live courts;
  - the bronze walkover;
  - team matchup results and group ranking;
  - attendance duplicate-name detection.
- **`parity-server.test.mjs`.** It imports
  `../../../sage-tools-api/src/sync/domain/facilityCompletion.mjs` when that
  sibling checkout exists, and skips with a message when it does not. It runs
  the server's played/BYE/series decisions and `domain/` on the same case
  table, and asserts they agree. This pair is the one remaining hand-kept copy
  of the rules (root `CLAUDE.md`, "Things that must be kept in sync by hand").
- **`guards.test.mjs`.** The dependency rules of §4.2. It also checks that
  no file under `views/` or `domain/` mentions operator features
  (§4.10). The check fails on these words, case-insensitive, outside
  comments: `getToken`, `authToken`, `signIn`, `scoreEntry`, `scoreable`,
  `organizer`, `missionControl`, `CLOUD_RUN`. The exceptions are
  `views/score-dialog.js` and `views/attendance-view.js`, which are the
  operator components themselves. CC and the scorer and desk pages load
  them, and the Hub never does.
- **`theme-contract.test.mjs`.** §4.7.
- **`no-copies.test.mjs`.** No live page defines a function that a `lib/v1/`
  module exports. It extracts every `function NAME(` and `const NAME = (… =>`
  in the live pages' inline scripts and compares the names against the
  engine's exports. It starts in Phase 1, with an allow-list of names not yet
  moved; each phase shrinks the list, and Phase 5 empties it.

### 6.7 Phase 0 acceptance

- `npm run verify` passes on a branch with no page changes. Baseline and
  branch are the same code, so every case is equal. This proves the harness
  is deterministic. Run it three times in a row.
- Change one character of a Hub template's CSS, then of one view's text.
  `npm run compare` fails on the expected cases, with the diff images
  written. Revert both changes.
- A case that tries to reach any un-routed host fails, naming the URL.

---

## 7. Phases

Each phase is one branch and one merge, and leaves the site working. Run
`npm run verify` before every commit. The baseline is always the merge-base, so
each phase is compared with what is live.

### 7.0 Phase 0 — the harness

Build §6 in full. **No file outside `_tests/` and `.gitignore` changes.**

### 7.1 Phase 1 — `platform.js` and the `domain/` modules

1. Create `lib/v1/platform.js` and every `domain/` module of §5.1, with their
   unit tests and characterization tests (§5.2).
2. **Convert the live pages to module scripts.** Change each live page's
   `<script>` to `<script type="module">` (§4.9), and import from
   `/lib/v1/domain/` in place of its own copies.
3. **Leave each page's call sites alone for now.** Where a page calls a moved
   function and the engine version now takes `type`, `display` or `now` as a
   parameter, keep a one-line wrapper in the page. Remove the wrapper when its
   callers move in Phase 3. For example:

   ```js
   const parseCode = code => Codes.parseCode(code, CURRENT_TYPE);
   ```

4. **Pages to convert:** CC; both `index.html` and `schedule.html` templates;
   the scorer and attendance templates.

**Acceptance:**

- `npm run verify` passes, with every new §12 row recorded.
- `no-copies.test.mjs` passes with only views and apps on its allow-list.

### 7.2 Phase 2 — the `data/` modules

1. Create `data/registry.js`, `snapshot.js`, `live-channel.js`, `api.js` and
   `tokens.js`.
2. Move every live page onto them. The `LIVE CHANNEL` block is deleted from
   every live page, and each page calls `createLiveChannel({ baseUrl:
   LIVE_BASE_URL, … })`. The `LIVE_BASE_URL` constant stays in the page,
   because it is a per-page setting (§4.5).
3. Add fixture mode (§4.8) to the templates' Hub and schedule board through
   `snapshot.js`.

**Acceptance:**

- `npm run verify` passes.
- With `LIVE_BASE_URL` set, a manual check against production on
  `localhost:8123` shows the socket connecting. In the browser's network
  panel, the `wss://` request answers `101`.

### 7.3 Phase 3 — the shared views and their CSS

Move one area at a time, each as its own commit. The order:

1. `views/html.js`, `views/scroll.js` and `views/ticket.js`, with
   `css/ticket.css`;
2. `views/finder.js` and `css/finder.css`;
3. `views/live-matches.js` and `css/live-matches.css`;
4. `views/standings.js` and `css/standings.css`;
5. `views/teams.js` and `css/teams.css` (CC only);
6. `views/score-dialog.js` and `css/score-dialog.css`, deleting the
   `SCORE CLIENT` block from CC and the scorer template;
7. `views/attendance-view.js` and `css/attendance.css`, deleting the
   `ATTENDANCE CLIENT` block from CC and the attendance template.

**For each area:**

- **Build `buildDayModel` (`domain/model.js`)** in the first commit. The
  pages compute a `DayModel` once per snapshot, and the moved views read it
  instead of the globals.
- **Classify every drifted line** of the area's functions by §5.2's causes,
  before writing the shared version.
  - CC-only lines (cause 1) move into CC's script, behind the extension
    points of §4.10. Add a new extension point where the table has none.
  - Fixes (cause 3) go into the core.
  - The §3.3 item 3 cases belong to areas 1, 2 and 4 (tickets, Match Finder,
    Standings).
- **Move the CSS rules** for the area out of the Hub templates' and CC's
  `<style>` into the area's CSS file, and link it from each page. Where the
  Hub's and CC's rules for the same selector differ, reconcile them like
  functions (§5.2). Styling only CC's features need (the `scoreable` ticket,
  the score hint, facility progress) stays in CC's own `<style>`, never in
  `lib/v1/css/`.
- **Put the "keep both" overrides (§3.3 item 4, items 1–3) in one block** in
  CC's `<style>`, under a banner:
  `/* ==== DELIBERATE DIFFERENCES FROM THE HUB — site-engine-spec §12 ==== */`.
  Each rule carries a comment with its §12 row. Nothing else goes in the
  block. A rule there that §12 does not list is drift, and counts as a
  failure in review.
- **Keep each page's `THEME` block in the page.**

**Acceptance:**

- `npm run verify` passes.
- `no-copies.test.mjs` has only `apps/` names left on its allow-list.
- The words `LIVE CHANNEL`, `SCORE CLIENT` and `ATTENDANCE CLIENT` appear in
  no live page.

### 7.4 Phase 4 — the apps and the shells

1. **Create the four `apps/` modules and their CSS**, out of what is left of
   each template's script and `<style>`.
2. **Reduce each template page to a shell** (§4.5):
   - the four event-template pages: std and dm `index.html` and
     `schedule.html`;
   - the scorer and attendance templates.
3. **Remove the Hub's settings constants**, which `events.json` now supplies
   (§4.6): `DAYS`, `FACILITIES`, `DIVISIONS`, `EVENTS`, `CLUBS`,
   `DIVISION_ORDER` and `EVENT_ORDER`. Category order follows the key order
   of `display.divisions` and `display.events`, which is how CC orders them.
   The dm shell keeps `CLUB_LOGOS`. The schedule shells keep `DAY_KEY` and
   `CAT_META`; rename `CAT_META` to `categoryColors` in the settings passed to
   `mountScheduleBoard`, and keep the constant's name in the page.
4. **Update the template runbook, `_templates/CLAUDE.md`**, in the same
   commit:
   - §2 steps 3–5: fewer tokens, no config arrays to fill in, the theme still
     edited in the shell;
   - §3 tokens: the list shrinks;
   - §5.1 sync table: the `DAYS`/`FACILITIES` rows and the three block rows
     go;
   - §7: live push off is still `LIVE_BASE_URL = ''` in the shells;
   - a new section on the engine and its versioning rule.
5. **Check `_templates/hub-pubmat/` still works** against the new templates:
   - `render.mjs` reads static markup that is unchanged, so it needs nothing;
   - `capture.mjs` routes only the snapshot, and the Hub now also fetches
     `events.json` from the network, which works as it is;
   - render one board from an instantiated template to confirm.

**Acceptance:**

- `npm run verify` passes.
- Each shell is under 300 lines.
- Instantiating a template by the updated runbook, on a scratch event key
  with the piggleball fixture and `?fixture=`, gives a working Hub, schedule
  board, scorer page and desk page on `localhost:8123`. Delete the scratch
  folder afterwards and do not commit it.

### 7.5 Phase 5 — clean-up

1. **Remove what is left over.** That means every Phase 1 wrapper, the
   `no-copies` allow-list and the characterization tests.
2. **Check the dependency guard and theme contract** with no exceptions left.
3. **Measure the shells and CC.** Record in this spec's status line:
   - the shells' line counts;
   - CC's line count, which is expected to fall by about 4,000;
   - the engine's total.

**Acceptance:**

- `npm run verify` passes.
- `grep -rn "function parseCSV\|function rowsToMatches\|function createLiveChannel" tools/control-center.html _templates/`
  prints nothing.

### 7.6 Phase 6 — documentation

Update every file in §8, then move this spec to `implemented/` by the procedure
in [`../README.md`](../README.md).

---

## 8. Documentation to update (Phase 6, and the runbook in Phase 4)

| File | Change |
| --- | --- |
| root `CLAUDE.md` (`D:\Personal\SAGE`) | **`sage-match-control.github.io` section:** pages are shells plus the engine at `lib/v1/`, with the layout and the versioning rule in brief; `_tests/` exists, with `npm run verify`; the site has one package file, dev-only, in `_tests/`. **"Things that must be kept in sync by hand":** the three byte-identical block bullets go; the played/BYE/series bullet shrinks to `lib/v1/domain/` and `facilityCompletion.mjs`, checked by `parity-server.test.mjs`; the team-event bullet says CC uses `lib/v1/domain/teams.js` and PickleDrive's copy is frozen; the `DAYS` ↔ `events.json` item goes for engine pages |
| `_templates/CLAUDE.md` | done in Phase 4 (§7.4) |
| `event-data/config/README.md` | §4.6's rule: an event stays in `events.json` while an engine page shows it |
| `sage-docs/docs/technical/site-engine.md` (new) | the engine: layout, dependency rules, the versioning rule, `EventConfig`/`DayModel`, the theme contract, fixture mode, the harness and how to run it, how to add a view or change a rule |
| `sage-docs/docs/technical/README.md`, `mkdocs.yml` | index and nav for `site-engine.md` |
| `sage-docs/docs/technical/architecture.md` | the site's part of the architecture; "Things that must be kept in sync by hand" |
| `sage-docs/docs/technical/adding-a-new-event.md` | the shorter steps, no config arrays, "kept in sync by hand" |
| `sage-docs/docs/technical/control-center.md` | "The `SCORE CLIENT` block", "Theme", "Fixtures": now engine modules |
| `sage-docs/docs/technical/scorer-page.md`, `event-attendance.md`, `schedule-board.md`, `sync-pipeline.md` | any mention of the `LIVE CHANNEL`, `SCORE CLIENT` or `ATTENDANCE CLIENT` blocks or of byte-identical copies |
| `sage-docs/docs/features/preparing-an-event.md` | any step that fills in `DAYS`/`DIVISIONS`/`EVENTS` in a page |
| `sage-docs/docs/specs/README.md`, `not-started/README.md` → `implemented/README.md`, `mkdocs.yml` | this spec's status as it moves |
| [`sage-tools-api-architecture-spec.md`](../in-progress/sage-tools-api-architecture-spec.md) §6.8 and status line | Phase 8 is this spec; done when this spec is implemented |

Grep `sage-docs/docs` and both `CLAUDE.md` files for `byte-identical`,
`LIVE CHANNEL`, `SCORE CLIENT`, `ATTENDANCE CLIENT` and `self-contained`, and
fix each hit that describes a live page. Hits about finished events, the other
tools, or history stay.

---

## 9. Effect on other specs

These notes are added to each spec's status line when this spec is written.
Each spec is revised properly when this one is implemented.

- **[Site test suite](../not-started/site-test-suite-spec.md).** Its `_tests/` layout,
  Playwright choice, router and clock are the same as §6, so it builds on this
  harness.
  - Its **consistency** layer is mostly no longer needed: there are no
    byte-identical blocks left on live pages.
  - Its **parity** layer becomes `_tests/unit/`.
  - Its **page tests** remain, run against the shells.
  - Revise it after Phase 5, before building it.
- **[Team tournament template](../not-started/team-tournament-template-spec.md).** A team
  event's Hub becomes a shell on the engine, with a team branch in
  `apps/hub.js` built from the `views/teams.js` CC already uses. It does not
  copy PickleDrive's inline code or the shared blocks. Revise it after Phase 5.
- **[Automated dry run](../not-started/automated-dry-run-spec.md).** It reuses the site test
  suite's checks, and is unaffected otherwise.

---

## 10. Acceptance checklist

**Phase 0**

- [x] `_tests/` exists, `npm run verify` is deterministic across three runs,
      and an injected change fails the expected cases.

**Phase 1**

- [x] `lib/v1/platform.js` and every `domain/` module of §5.1 exist, with
      unit tests.
- [x] Every live page is a module script and imports `domain/`.
- [x] Characterization tests pass; §12 records every reconciliation.

**Phase 2**

- [x] No live page has a `LIVE CHANNEL` block; all use `data/live-channel.js`.
- [x] Every page reads the registry and snapshots through `data/`.
- [x] The templates' Hub and schedule board accept `?fixture=` on localhost.

**Phase 3**

- [x] The Hub and CC render Match Finder, tickets, Live Matches and Standings
      with `views/` and the shared CSS files.
- [x] No live page has a `SCORE CLIENT` or `ATTENDANCE CLIENT` block.
- [x] CC's own features (score entry, match-number search, facility progress,
      the label warning) attach through §4.10's extension points; no file in
      `views/` or `domain/` names an operator feature, and none takes a
      page-naming flag.
- [x] All four §3.3 item 3 fixes show on both pages and §3.3 item 4 (§12 rows 5–10) is built as decided.
- [x] Every cause-3 row is listed for the owner at merge (rows 1–4 and 23, in the build report and the merge commit).

**Phase 4**

- [x] The six template pages are shells under 300 lines.
- [x] The Hub takes its days and labels from `events.json`, with a last-good
      fallback.
- [x] The runbook instantiates a working event on fixtures (`_tests/engine/fixture-mode.test.mjs` instantiates the templates and loads them on fixtures).

**Phase 5**

- [x] No wrapper, allow-list or characterization test is left; guards are
      green.

**Phase 6**

- [x] Every file in §8 is updated; this spec is in `implemented/`.

**Always**

- [x] `npm run verify` passes on every merge, with every accepted difference
      recorded in §12.
- [x] No finished event's folder, archived page or other tool changed.
- [x] Nothing merged during an event or in the 3 days before one (nothing is merged yet).

---

## 11. Rollout, rollback and later options

- **Rollout.** Phases merge one at a time between events. After each merge:
  1. load the live Hub of the newest finished event, CC and the schedule board
     on the production site;
  2. check that each renders;
  3. check that the browser console shows no error.
- **Rollback.** `git revert` the phase's merge commit and push. Each phase
  leaves the pages working on its own, so reverting the latest phase is
  always safe. Revert later phases before earlier ones.
- **Later options, not part of this spec:**
  - moving CC's own features into `lib/v1/console/` modules;
  - `<link rel="modulepreload">` hints, if the module waterfall is ever
    measurably slow on venue Wi-Fi;
  - a team-type Hub (§9).

---

## 12. Reconciliations and divergences

The implementer fills this in as the phases land: one row per reconciled
function or CSS rule (§5.2), and one per place the built engine departs from
this spec. Rows 1–10 were found and decided before the build. Their "Proved
by" cell is filled in when the work lands.

| # | Phase | Function / rule | Copies | Cause (§5.2) | Kept | Proved by (harness case or unit test) | Owner (cause 3: listed at merge; cause 5: approved) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 3 | Series games after the decider: `ticketHTML`, `renderMatchup`, `runSearch`'s "Next Up", `UNNEEDED_GAMES` in `loadLiveData` | Hub {std, dm index} vs CC | 3 | the Hub's (greyed "Not needed"; never "Next Up"), now in CC too | `unit/views/ticket.test.mjs`, `unit/views/standings.test.mjs` (a series game after the decider is greyed out), `unit/views/finder.test.mjs` (Next Up skips it); accepted in `accepted.mjs`: `cc/piggleball-2026/finder/*/desktop` | approved as a category 2026-10-06 |
| 2 | 3 | BYE matches: `runSearch`'s pair list; `renderStageTables`/`renderSharedStages` dropping BYE rows and a bye-decided Bronze | CC vs Hub | 3 | CC's (`matchByeSide` filter; `isByeStandingRow` filter; `categoryBronzeIsByeDecided`), now on the Hub too | `unit/views/finder.test.mjs` (a BYE is not in a pair's list), `unit/views/standings.test.mjs` (a BYE row never shows), `unit/views/live-matches.test.mjs` (a BYE is not a court); no fixture case differs | approved as a category 2026-10-06 |
| 3 | 3 | Match Finder "Teams" count (`renderIntro`) | Hub vs CC | 3 | the Hub's (bare-number codes only) | `unit/views/finder.test.mjs`; accepted: `cc/{piggleball,pickle-for-sight,pnf-x-bup}/finder/*` | approved as a category 2026-10-06 |
| 4 | 3 | BYE pairs in the autocomplete (`rebuildTeamIndex`) | CC vs Hub | 3 | CC's (`sideIsBye` skip) | `unit/views/finder.test.mjs` (the pair index skips a BYE); no fixture case differs | approved as a category 2026-10-06 |
| 5 | 3 | Dual-meet desktop Standings layout (dm `renderStandings` vs CC `renderDualMeetStandings`; `.standings-board-desktop`, `.standings-col-overflow`) | dm Hub vs CC | 5 | both: option `dualMeetDesktopLayout`, `'grid'` (Hub, default) / `'row'` (CC) | `unit/views/standings.test.mjs` (both layouts); the `dm-index` and `cc` standings cases are identical to the baseline | owner 2026-10-06: keep both |
| 6 | 3 | Round-robin pair names (`.br-table`, `td.pair`, `td.pair span`, `pairCell`'s `title`, `.br-table-wrap`) | Hub vs CC | 5 | both: Hub's in shared CSS; CC's whole names and sideways scroll in CC's deliberate-differences block | `cc/*/standings/*` and `std-index`, `dm-index` standings cases, identical to the baseline | owner 2026-10-06: keep both |
| 7 | 3 | Round-robin table card style (`.br-table` radius, shadow, margin; `.br-table` sibling spacing rules) | Hub vs CC | 5 | both: Hub's in shared CSS; CC's plain tables in CC's deliberate-differences block | the same cases, identical to the baseline | owner 2026-10-06: keep both |
| 8 | 3 | Ticket pair code `.team-block .team-name` | Hub vs CC | 5 | CC's (9.5px muted uppercase label) on both | accepted: `std-index/*/finder*` (pixels); `unit/views/ticket.test.mjs` | owner 2026-10-06: CC's |
| 9 | 3 | Live Matches row divider `table.live-table td` | Hub vs CC | 5 | CC's (2px, `rgba(20,27,44,.4)`) on both | accepted: `std-index/*/live/*` (pixels) | owner 2026-10-06: CC's |
| 10 | 3 | Phone sizes at most 480px: `.names-row` gap, `.team-block .players`, `.live-team-code`, `.live-team-players` | Hub vs CC | 5 | the Hub's on both | accepted: `cc/*`, `dm-index/*` finder and live cases at phone width, and `cc-signed-in` score entry (pixels). Checked by reverting the changed values: the cases then match the baseline exactly | owner 2026-10-06: the Hub's |
| | 11 | 3 | `.names-row` base `margin-bottom` (the ticket's names row) | std Hub 12px vs dm Hub and CC 14px | 5 | both: the standard Hub's in `css/ticket.css`; the dual-meet Hub and CC keep 14px in their own block | `cc/*/finder/*`, `dm-index/*/finder/*` identical to the baseline | owner 2026-10-06: any undecided CSS difference keeps both |
| 12 | 3 | Live Matches at phone width: the match number (`.live-match-pill`), the VS or score (`.live-vs-score`), the category line (`td.live-cat-cell`); the standard Hub's `td.live-vs-cell` and `td.live-cat-cell` | std Hub vs dm Hub and CC | 5 | both: each page keeps its own rules in its own block | `*/live/*/phone` cases (identical to the baseline apart from row 10) | owner 2026-10-06: keep both |
| 13 | 3 | The dual-meet Hub's Round Robin: one table with a `Br` bracket column. CC draws the same data as a grid of bracket tables | dm Hub vs CC | 5 | both: option `rrBracketLayout`, `'grid'` (default, CC) / `'column'` (dual-meet Hub) | `unit/views/standings.test.mjs` (no fixture has sub-brackets, so no harness case shows it) | owner 2026-10-06: keep both |
| 14 | 3 | Dual-meet Hub's club logos in the ticket, the Live Matches row and the club bar | dm Hub only | 1 | options `teamLogoHTML` (ticket, Live Matches) and `clubLogoHTML` (club bar), passed by `apps/hub.js` from the shell's `CLUB_LOGOS` | `dm-index/*` cases identical to the baseline | n/a |
| 15 | 3 | Divergence: `beforeStandings(model) → html` (§4.10) is not built. The category-code warning sits outside the Standings panel, so CC passes the `unresolved` Set the views fill and draws the warning itself after `render()` | CC | 1 | `options.unresolved` | `cc` standings cases | n/a |
| 16 | 3 | Divergence: Live Matches `facilityExtras(name, model) → html` writes into a separate element, `options.extrasEl`, because CC's facility progress cards are not inside the board | CC | 1 | `facilityExtras` with `extrasEl` | `cc/*/live/*` | n/a |
| 17 | 3 | Divergence: `onTicketClick` (§4.10) is not built. A ticket's `decorate` marks its click target (the class, `data-score-*` and `role`), and CC's two delegated listeners on its results container open the dialog | CC | 1 | `decorate`, and CC's own listeners | `cc-signed-in/*/score-entry-save/*` | n/a |
| 18 | 3 | Live Matches courts: the Hubs listed every court in their configured `FACILITIES` ranges, CC lists each facility's scheduled courts from its own matches. The engine takes CC's | Hub vs CC | 2 | CC's, on both | `unit/views/live-matches.test.mjs`. It shows only where a configured range and the data disagree, which no fixture does | n/a |
| 19 | 3 | The dual-meet Hub treated a team code naming a club not in its `CLUBS` as unparsed; the engine reads it as CC does, the club being the code's first segment | dm Hub vs CC | 2 | CC's | `unit/domain/matches-standings-codes.test.mjs` | n/a |
| 20 | 3 | A team event's views, the score dialog and the attendance list moved out of CC without a change: `createTeams` takes `decorateRow` and `expandAllClass` as options (the guard forbids the operator words in `views/teams.js`); the attendance list's poll interval comes in as `pollMs` (a view imports no `platform.js`) | CC | 1 | options | `cc/pickledrive-anniversary-2026/*`, `attendance/*` | n/a |
| 21 | 3 | The attendance list's stylesheet, injected as a `<style>` at run time, is `css/attendance.css`, linked after the page's own styles. The desk page's `--att-*` overrides in its `<style>` never took effect (the injected defaults came later and won); they are kept as they were, so the page looks as before | desk page, CC | 0 | the same order as before | `attendance/*` identical to the baseline | n/a |
| 22 | 1 | Placement: `matchInstanceOf` is in `domain/codes.js` (re-exported by `matches.js`); `courtNumberFrom` and `formatMinutesAsClock` are in `domain/` (`progress.js`, `golive.js`) and `views/live-matches.js` re-exports the first; CC's `computeDayIsLive` is `consoleDayIsLive` and the schedule board's `stageOf` is `ladderStageOf`, so that no page defines what the engine exports | all | 0 | as listed | `unit/no-copies.test.mjs` | n/a |
| 23 | 1 | Scorer page: on expiry the poller is stopped and stays stopped (it could be restarted by a visibility change before); the registry error wording keeps "replied N" | scorer | 3 | the fix | `scorer/*` | approved as a category 2026-10-06 |
| 24 | 2 | `eventConfigFrom` orders a day list by date, as CC does (its typedef said listing order); a registry that fails to load falls back to the last good copy | all | 2 | CC's | `unit/data/data-modules.test.mjs` | owner 2026-10-06: date order |
| 25 | 4 | The Hub's days, facilities and labels come from `events.json`, so a Hub shows the registry's day label, not the one its page carried. A Hub whose event is missing from the registry, or whose type is not `standard` or `dual-meet`, says so in its prompt | both Hubs | 2 | the registry's | `std-index/*`, `dm-index/*` identical to the baseline (the fixtures' labels agree); `engine/fixture-mode.test.mjs` | owner 2026-10-06: settings from `events.json` |
| 26 | 6 | The wrapper allowance in `no-copies.test.mjs` (§7.1 step 3) existed from Phase 1 to Phase 5 and is gone; Control Center's adapters that add its own behaviour are named `consoleTicketHTML`, `drawLiveMatches`, `loadDaySnapshot`, `resolveFacilityEnd` and `progressForFacility` | CC | 0 | as listed | `unit/no-copies.test.mjs` | n/a |
| 27 | 0 | Harness: a step that times out under load, a page that reports its own load timing out, and a page caught in a poll's reload are run again, up to three times; a real difference is still there. The attendance desk case runs in state `pre` only (the page has no other state) | harness | 0 | as listed | `engine/harness.test.mjs` | n/a |
