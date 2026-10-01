# Spec — site test suite (Control Center and the event pages)

> **Status: not started.** Nothing here is built. Written 2026-10-02 against
> `sage-match-control.github.io` at `89a7476` (live push switched on in the
> pages). The sibling of the
> [`sage-tools-api` test suite](sage-tools-api-test-suite-spec.md), which
> covers the Cloud Run service and stops at its HTTP boundary.
>
> **Overlaps the [automated dry run](automated-dry-run-spec.md).** §2 is the
> line between them: this suite owns every check that runs on fixtures, and
> the dry run's rendering layer calls this suite's checks instead of
> building its own.
>
> **Do not push any of this during an event, or in the days before one.**
> `_tests/` is never published, but a push to `main` still runs a Pages
> build.

Pin what Control Center and the event pages do today, in tests that run the
pages' **own code** in a real browser against fixture data, so that a change
to a page, or to one copy of a rule that exists in several pages, fails a test
instead of surfacing on event day.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec adds a `_tests/` folder, one `.gitignore` line and a pre-push hook to
`sage-match-control.github.io` (the GitHub Pages site), adds fixtures under its
existing `_fixtures/`, and changes **no page**. It reads one file from the
sibling `sage-tools-api` repo (§5.2) and changes nothing there.

**What the site is.** Hand-written, self-contained HTML. Each page is one file
with an inline `<style>` and **one classic inline `<script>`**: no modules, no
bundler, no `package.json`, no tests. Pages deploy on push to `main`. GitHub
Pages runs Jekyll over the repo (there is no `.nojekyll`, on purpose), and
Jekyll does not publish top-level folders whose names begin with `_`. That is
what keeps `_templates/` and `_fixtures/` off the live site, and it is why the
suite lives in **`_tests/`**: its `package.json`, tests and `node_modules`
are never published.

**Facts checked against the code** (each one shapes the design):

- **Page functions are globals.** Every page's script is a classic `<script>`,
  so its top-level `function` declarations are properties of `window`, and its
  top-level `const`s are visible to code evaluated in the page. A test can call
  `sideIsBye(...)` or `teamMatchupResult(...)` in the loaded page and get the
  page's real answer. No extraction or reimplementation is needed, and no
  page has to change.
- **Only some pages support `?fixture=`.** `tools/control-center.html` and
  PickleDrive's two pages define `FIXTURE`: on `localhost`/`127.0.0.1`,
  `?fixture=<name>` loads `/_fixtures/...` instead of the published data.
  Piggleball and both templates do not; they only test
  `typeof FIXTURE !== 'undefined'` to keep the live channel off. So the suite
  does **not** rely on `?fixture=`. It answers the pages' **real** data URLs
  (`https://sage-match-control.github.io/event-data/...`) from inside the
  browser (§4.3), which works for every page, needs no page change, and tests
  the real URL each page builds.
- **Templates are runnable once their tokens are filled.** Both templates'
  `index.html` and `schedule.html` contain `{{TOKEN}}` placeholders
  (`{{EVENT_KEY}}`, `{{EVENT_TITLE}}`, `{{SCHEDULE_DAY_KEY}}`, the club tokens
  in the dual meet, …; `_templates/CLAUDE.md` §3 lists them) and `// EXAMPLE`
  config arrays (`DAYS`, `FACILITIES`, …). The suite's static server fills the
  tokens with fixed test values on the fly (§4.3). Every new event is copied
  from a template, so testing the templates tests what the next event starts
  from.
- **Rules are copied by hand between pages.** These are the copies:

  | Rule | Copies | Listed in root `CLAUDE.md`? |
  |---|---|---|
  | `LIVE CHANNEL` block (between `// ==== LIVE CHANNEL` and `// ==== END LIVE CHANNEL ====`) | 9 files: `tools/control-center.html`, PickleDrive's and Piggleball's `index.html` + `schedule.html`, both templates' `index.html` + `schedule.html`. Byte-identical by rule | yes ("compare them with `diff`") |
  | "Played" and "BYE" | `sage-tools-api/src/sync/facilityCompletion.mjs` (`facilityIsComplete`) and the console's `rowsToMatches` (`played` = both scores present), `sideIsBye`, `matchByeSide`, `computeFacilityProgress` | yes |
  | Team-event rules | `events/pickledrive-anniversary-2026/index.html` and the console's `team type` block. 21 functions exist under the same name in both (listed in §5.2) | yes |
  | Pair labels | the console's `PAIRS` (`XD`) and PickleDrive `schedule.html`'s `PAIR_SHORT` (`MXD`), both built with `numberRepeatedPairs`. The `MXD`/`XD` difference is deliberate | yes |
  | Go-live rule | `computeDayIsLive`, `earliestScheduleMinutesFrom`, `parseScheduleTimeToMinutes` and `GO_LIVE_LEAD_HOURS = 4` in every current event's `index.html` and both templates. The console has its own `computeDayIsLive(_day)` and `parseScheduleTimeToMinutes` | **no** |
  | Shared constants | `POLL_INTERVAL_MS = 10000`, `LIVE_SAFETY_POLL_MS = 60000`, `GHPAGES_OWNER`/`GHPAGES_REPO`, `CLOUD_RUN_BASE_URL` | partly (`CLAUDE.md`'s "hard-coded constants") |

  Today the only check on any of these is a person remembering to run `diff`.

  The [attendance spec](multi-event-attendance-spec.md) adds one more
  byte-identical block, `ATTENDANCE CLIENT`, in `tools/control-center.html`,
  `_templates/attendance/attendance.html` and every `events/<key>/attendance.html`.
  It also adds a Control Center **Attendance** tab, and fixtures named
  `_fixtures/<event>/attendance-<facility>-<name>.csv`. If it has landed,
  `live-channel-identical.test.mjs` gets a sibling,
  `attendance-client-identical.test.mjs`, with the same rules. The attendance
  template joins §3's pages. The console's attendance tab and the desk page
  get page tests: the category filter and jump, a two-team person toggling in
  both places, withdrawn people hidden, and the desk-off and expired-link
  messages. The router answers the gviz `ATTENDANCE` export and the API's
  `PUT` like any other routed request.
- **Live/Hide and the sync-method switch are real writes.** The console's
  `window.confirm` prompts guard `POST /sync/:day/live` (`isLive: false`) and
  `POST /sync/live-push` (`enabled: false`). A test that reached Cloud Run would
  change production config, so the suite intercepts every request to Cloud
  Run, GitHub and the Worker, and **fails any test whose page tries to reach
  a host it did not intercept** (§4.3).
- **Time drives behaviour.** The 10 s poll, the 60 s safety poll while a live
  socket is open, the 4-hour go-live lead, facility ETAs and "stale" warnings
  all depend on the clock. The suite controls the page's clock (§4.2).

**Rules that apply to every change** (from the root `CLAUDE.md` and this repo's):

- No page changes. If a behaviour cannot be tested without changing a page,
  stop and record it in the hand-off (§10).
- Working copies use CRLF line endings. Keep them.
- `.gitignore` is a single rule today (`**/*beta*`). This spec adds exactly one
  more, `_tests/node_modules/`. `beta.html` scratch copies stay untracked and
  are never tested.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../site-test-suite-spec.md`.
- Documentation is written in the present tense.

**How to work.** One branch, `site-tests`, one commit per build step (§9). If
a test fails, the page is right until proven otherwise: this suite describes
it. Fix the test, not the page. If a behaviour looks wrong, or two copies of a
rule disagree, pin what each does today, mark it `// KNOWN DIFFERENCE`
(§6.1), and tell the owner.

---

## 1. Goals, non-goals and constraints

**Goals**

1. Every hand-copied rule in the table above is checked mechanically: the
   byte-identical ones by text, the ported ones by running both copies on the
   same inputs and comparing the answers.
2. Every current event-day page (§3) is loaded in a real browser on fixture
   data for each state an event goes through (before play, mid-play, finished,
   edge cases), and its key content is asserted, including the runbook's
   rehearsal checks.
3. The live channel's behaviour is tested against a fake Worker: a pushed
   snapshot re-renders the page; a dropped socket falls back to polling.
4. The runbook checks are a reusable module the automated dry run calls (§2).
5. The suite stays current: a page change updates its tests in the same
   commit, and a new event cannot be added without being added to the suite
   (§8).

**Non-goals**

- Changing any page, including adding `?fixture=` support, test hooks or
  `data-testid` attributes.
- Talking to any real service: GitHub, Cloud Run, the live Worker, Google
  Sheets. That is the dry run's preflight and rehearsal (§2).
- Visual regression (screenshot diffs), accessibility audits, cross-browser
  runs. Chromium only.
- The evergreen tools other than Control Center (calculator, bracket
  generator, scoresheet generator, `tools/index.html`), `field-guide.html`,
  finished events and `events/archives/`. §11 says what comes next.

**Constraint.** The suite pins today's behaviour **including known
differences between copies**. It must not "fix" a page to make a test pass.

---

## 2. Overlap with the automated dry run

The [automated dry run](automated-dry-run-spec.md) (idea only, not started)
proposes one command that runs the runbook's rehearsal in three layers. Its
middle layer and this suite are largely the same work. Building both
independently would produce two browser harnesses, two sets of fixtures, and
two copies of the runbook's checks. That is the very duplication both specs
exist to reduce.

| Dry run layer | What it does | Overlap with this suite | Who owns it |
|---|---|---|---|
| **Preflight** | `GET /ping`, `GET /sync/config`, `POST /sync/:day` against **production**; every facility recently synced; go-live state | None. This suite never touches a real service | Dry run |
| **Rendering** | Builds a snapshot from the real one with matches A/B/C applied, opens the console and public pages on localhost, asserts the runbook's §1.2–§1.3 checks | **Almost total.** Same pages, same checks, same "real page code, never a reimplementation" principle, fixture data instead of live data | **This suite.** It provides the browser harness, the runbook checks (`_tests/checks/runbook.mjs`, §5.4) and committed A/B/C fixtures per event type. The dry run's rendering layer shrinks to: build a fixture from the real snapshot, then call `runbook.mjs` on it |
| **Rehearsal** | Edits the real facility sheet in an automated browser, waits for the snapshot to change, runs the rendering checks on live data, restores | Reuses the same checks, against live data instead of a fixture | Dry run. Its spikes (Google sign-in, cell targeting, restore) are untouched by this spec |

What follows from that split:

- **One set of runbook checks.** `runbook.mjs` takes a Playwright `page`
  already pointed at a page and a scenario
  (`{ finished: <match A>, live: { match: <B>, court }, unmapped: <code C> }`),
  and asserts the runbook's lines. It does not care where the data came from,
  so the dry run passes live data and this suite passes a fixture.
- **One browser library.** This suite uses Playwright (§4.1). The dry run's
  rendering layer, built on `runbook.mjs`, uses it too. Its rehearsal layer
  still plans Puppeteer, which `sage-tools-api` already depends on. Whether
  the rehearsal moves to Playwright (which also has persistent profiles for
  the Google sign-in spike) is the dry run's decision, recorded there.
- **Fixtures.** The dry run notes "only PickleDrive has fixtures today". This
  suite commits fixtures for all three event types, including an A/B/C
  `runbook` fixture per type (§4.4). That answers the dry run's open question
  of whether fixtures are committed per event: committed per **type** here,
  generated per **event** by the dry run and discarded.
- **Order.** This suite has no blocking spikes and can be built first. The dry
  run's rendering layer then needs only a fixture generator.

The dry run spec carries a matching note.

---

## 3. Pages in scope

| Page | Type | Live channel | `?fixture=` today | Tested as |
|---|---|---|---|---|
| `tools/control-center.html` | all three | yes | yes | console, once per type |
| `events/pickledrive-anniversary-2026/index.html` | team | yes | yes | public page |
| `events/pickledrive-anniversary-2026/schedule.html` | team | yes | yes | schedule board |
| `events/piggleball-2026/index.html` | standard | yes | no | public page |
| `events/piggleball-2026/schedule.html` | standard | yes | no | schedule board |
| `_templates/standard-tournament-template/index.html`, `schedule.html` | standard | yes | no | public page, board (tokens filled) |
| `_templates/dual-meet-template/index.html`, `schedule.html` | dual meet | yes | no | public page, board (tokens filled) |

Not in scope: `events/pnf-x-bup-dual-meet/` and
`events/pickle-for-sight-2026/` carry no `LIVE CHANNEL` block, which root
`CLAUDE.md` reserves for events that haven't finished, so they are frozen.
`events/archives/`, `tools/match-control.html` (a redirect stub), the other
tools, and `beta.html` copies are out too. The page manifest (§8.2) lists every
HTML file in the repo with its status, so nothing is left out silently.

---

## 4. Strategy

### 4.1 Layers and tooling

| Layer | Folder | Needs a browser | What it checks |
|---|---|---|---|
| **Consistency** | `_tests/consistency/` | no | page **text**: byte-identical blocks, shared constants, leftover `{{TOKEN}}`s |
| **Parity** | `_tests/parity/` | yes | two or more copies of a rule, each **run** in its own page (or the server's module), give the same answers on the same inputs |
| **Pages** | `_tests/pages/` | yes | each page in §3, on each fixture, shows what it should, and its live channel and clock-driven behaviour work |

**Runner: `node:test`**, the same as `sage-tools-api` (Node 22.23.3 or later),
so the workspace has one test idiom. **Browser: Playwright** (the `playwright`
library, not `@playwright/test`'s runner), Chromium only, pinned to an exact
version in `_tests/package.json`. It is a **dev-only** dependency in a folder
Jekyll never publishes, so nothing reaches the site.

Playwright over Puppeteer, despite Puppeteer already being in `sage-tools-api`,
because three things this suite needs are built in and would otherwise be
hand-made fakes, each a place for the harness itself to be wrong:

- `page.clock` installs a controllable clock (`Date`, timers) before the page's
  script runs, so the 10 s poll and the 4-hour go-live lead are tested with
  `clock.runFor(...)` instead of real waiting.
- `page.routeWebSocket(...)` stands in for the live Worker: the test accepts
  the page's socket, sends it snapshots, and closes it on demand.
- `page.route(...)` answers HTTP requests by URL pattern, and Playwright's
  assertions wait for the page to settle instead of sleeping.

The consistency layer needs no browser and no install, so it runs on its own
(`npm run test:fast`) and lands first (§9 step 1).

### 4.2 Determinism

1. **Clock.** Every browser test installs `page.clock` at a fixed instant
   before navigation, in context timezone `Asia/Manila` (the venues'), and
   moves time only with `clock.runFor` / `clock.fastForward`. Nothing
   sleeps. A test never reads the real time.
2. **Network.** Every request leaving `127.0.0.1` is routed (§4.3). An
   unrouted request **fails the test** with its URL. Google Fonts requests
   are answered with an empty `200` so text renders in the fallback font
   without a network.
3. **Viewport.** 1280 × 800 by default. Public pages are also loaded at
   375 × 812 for the overflow check (§5.3).
4. **Errors.** Every browser test fails on any `pageerror` (an uncaught
   exception) or `console.error` from the page, unless the test expects that
   exact message. A page that throws silently on some fixture is the most
   likely event-day failure, and this rule catches it on every test for
   free.
5. **Independent.** One browser per test file, a fresh context per test, no
   state carried between tests (the pages use `localStorage` for saved
   searches and day choice; a fresh context starts empty).

### 4.3 Helpers

```
_tests/
  package.json            # "type": "module", "private": true, devDependency playwright (exact)
  helpers/
    server.mjs            # startSite(): static server over the repo root
    browser.mjs           # launch(), openPage(), the request router and error collector
    fakeWorker.mjs        # the live Worker, via page.routeWebSocket
    extract.mjs           # pulls a marked block or a `const NAME = <literal>` out of a page's text
    fixtures.mjs          # loads _fixtures/ files, builds config for an event key
  pages.mjs               # the page manifest (§8.2)
  checks/runbook.mjs      # the runbook checks (§5.4), shared with the dry run
  cases/played-bye.json   # the played/BYE case table (§5.2)
  consistency/*.test.mjs
  parity/*.test.mjs
  pages/*.test.mjs
```

**`server.mjs`.** `startSite()` serves the repo root on `127.0.0.1`, port 0,
and returns `{ baseUrl, close }`. It resolves an extensionless path to `.html`,
as GitHub Pages does (`/events/<key>/schedule` is how the console opens the
board). For any path under `/_templates/` it substitutes every `{{TOKEN}}` from
the template's entry in `pages.mjs` (fixed test values such as
`EVENT_KEY: "test-standard"`, `EVENT_TITLE: "Test Standard Event"`, and
`SCHEDULE_DAY_KEY` set to the first key in the template's example `DAYS`) and
answers `500` naming any token it has no value for. It serves bytes otherwise
unchanged. It never serves `beta` files.

**`browser.mjs`.** `openPage(browser, path, options)` creates a context
(timezone, viewport), installs the clock at `options.now`, installs the router,
navigates, and waits for the page's first render. Options:

```js
{
  now: "2026-10-03T08:00:00+08:00",   // required; there is no default clock
  config: <events.json object>,       // answers .../event-data/config/events.json
  snapshots: { "<event>/<day>": <snapshot> },   // answers .../event-data/<event>/data/<day>.json
  worker: "absent" | "refuses" | fakeWorker,    // the live channel's fate (default "refuses")
  cloudRun: { "<METHOD> <path>": handler },     // console only; see below
  viewport, expectErrors: [/.../],
}
```

The router answers `https://sage-match-control.github.io/event-data/...` from
`config` and `snapshots` (a snapshot not supplied answers `404`, as GitHub
does for a day never published), records every request in
`page.requests` (method, URL, body), and fails the test on anything else
leaving `127.0.0.1`. For the console, requests to `CLOUD_RUN_BASE_URL` go to
`cloudRun` handlers. An unhandled Cloud Run route fails the test, so no test
can reach the real API, let alone write config. The default handlers answer
`GET /ping` and `GET /sync/config` with fixed, plausible bodies, and
`POST /auth/login` with a fake token.

**`fakeWorker.mjs`.** `createFakeWorker()` returns an object that
`openPage` passes to `page.routeWebSocket` for the page's `LIVE_BASE_URL`
address. It records `connections`, and offers `push(snapshot)` (sends it to
every open socket in the message shape the `LIVE CHANNEL` block expects; the
implementer reads that shape from the block), `close(code)` and `refuse()`.
`"refuses"` (the default) closes every connection at once, so a page falls
back to polling. Its own tests in `_tests/consistency/` cannot cover it (it
needs a browser), so `pages/live-channel.test.mjs` starts with a case that
proves the fake connects and delivers on one page before any page relies on
it.

**`extract.mjs`.** `block(text, startMarker, endMarker)` returns the exact
bytes between the markers inclusive, and throws if either marker is missing
or appears twice. `constLiteral(text, name)` returns the source text of a
top-level `const <name> = <literal>;` and throws if it is absent or declared
twice.

### 4.4 Fixtures

Fixtures are snapshot JSON files in the shape `event-data` publishes. They
live under the existing `_fixtures/<event-key>/<name>.json`, so `?fixture=`
still works by hand on the pages that support it. Test events are added to
`_fixtures/config.json` beside PickleDrive's.

| Event key | Type | Pages it feeds | Fixtures |
|---|---|---|---|
| `pickledrive-anniversary-2026` | team | PickleDrive pages, console (team) | existing `pre`, `finished`, `edge`, `qf-pre`; new `mid`, `runbook` |
| `piggleball-2026` | standard | Piggleball pages, console (standard) | new `pre`, `mid`, `finished`, `edge`, `runbook` |
| `test-standard` | standard | the standard template | the same five, against the template's example `DAYS`/`FACILITIES` |
| `test-dual-meet` | dual meet | the dual-meet template, console (dual meet) | the same five, against the template's example config and test club codes |

- `pre`: scheduled, nothing played or live.
- `mid`: some matches played, some live on courts, others pending.
- `finished`: every non-BYE match scored, `completedAt` stamped.
- `edge`: a BYE by team code, by each player name, mixed-case `Bye`, both
  sides BYE (the data error), an unparseable court, a duplicate court/time,
  a blank `Schedule`.
- `runbook`: exactly the rehearsal's state. Match A finished, match B live on a
  court, match C with an unmapped category code. Each fixture's matches A/B/C
  are named in `_tests/fixtures.mjs` so `runbook.mjs` gets them from one place.

Fixtures are built **from real data shapes**. For Piggleball, start from
`event-data`'s published day file once it exists and strip it to the needed
state; until then, and for the templates, from the column list in
`_templates/CLAUDE.md` §4. A fixture never contains a real person's phone,
email or anything beyond the player names a published snapshot already
carries.

---

## 5. Coverage matrix

Every case below is required.

### 5.1 Consistency (no browser)

| File | Cases |
|---|---|
| `live-channel-identical.test.mjs` | every file containing `// ==== LIVE CHANNEL` has exactly one block (`block()` throws otherwise), and every block is byte-identical to the console's. On failure, the message names the file and the first differing line. The set of files carrying the block equals the manifest's `liveChannel: true` set (§8.2) |
| `shared-constants.test.mjs` | `POLL_INTERVAL_MS`, `LIVE_SAFETY_POLL_MS`, `GO_LIVE_LEAD_HOURS`, `GHPAGES_OWNER`, `GHPAGES_REPO` and `CLOUD_RUN_BASE_URL` have one value across every in-scope page that declares them (a page that does not declare one is fine; two values are not) |
| `no-leftover-tokens.test.mjs` | no file under `events/` (outside `archives/`) contains `{{`. This is `_templates/CLAUDE.md` §2 step 3's `grep`, automated |
| `manifest.test.mjs` | the page manifest guards (§8.2) |

### 5.2 Parity (each copy run in its own page)

| File | Cases |
|---|---|
| `played-bye.test.mjs` | `cases/played-bye.json` holds matches CSVs. It starts with every case in `sage-tools-api/scripts/verify-facility-completion.mjs`: all non-BYE scored, partial, BYE by team code, by either player name, case-insensitive `bye`, a score removed, empty CSV, header-only CSV. Each CSV is run through the console's own `parseCSV` → `rowsToMatches` → `computeFacilityProgress` in the loaded console, and the console's "complete" (`left === 0`, which is what prints "All matches done") is compared with the server's `facilityIsComplete(csv)`, imported from `../../sage-tools-api/src/sync/facilityCompletion.mjs`. If that sibling repo is absent, the server half is **skipped with a message, not failed**, and the console half still runs against the expected answers stored in the case file. **Empty and header-only CSVs**: the server says not complete; the console's `left` is `0` for both. Pin what each copy does, and what the facility card actually shows in that state (§5.3), as `// KNOWN DIFFERENCE` if they disagree, and report it |
| `team-rules.test.mjs` | PickleDrive's `index.html` and the console (team event loaded) open on the same fixture, for each of the six team fixtures. These rule functions are called in both pages with the same inputs, and the results compared as JSON: `baseTeamOf`, `groupOf`, `isGroupTeamCode`, `pairOf`, `parseSideCode`, `sideOf`, `slotOf`, `stageOf`, `stageLabel`, `teamNameOf`, `teamMatchupResult`, `teamRankBracket`, `teamCourtsLabel`, `teamTimesLabel`, `pairLabel`, `sideLabel`, `numberRepeatedPairs`. Inputs: every team code, matchup and group in the fixture. `pairLabel`/`sideLabel` compare after mapping the public page's `MXD` to `XD` (deliberate). The `*HTML` builders (`teamMatchupCardHTML`, `teamMatchupRowHTML`, `teamResultHTML`) are excluded because their markup differs by design, and so is `rebuildTeamData` (state, not a rule). A rule function listed here that is missing from either page fails the test, so a rename in one copy cannot drop it from the check |
| `pair-labels.test.mjs` | PickleDrive `schedule.html`'s `PAIR_SHORT` equals the console's `PAIRS[n].short` for every pair number, after `MXD` → `XD` |
| `go-live.test.mjs` | `computeDayIsLive(day, snapshot)` in every in-scope `index.html` (both events, both templates) gives the same answer for a table of cases: `isLive` `true`/`false` override in either direction; `auto` before, at and after the 4-hour lead before the earliest `Schedule` time; a snapshot with no parseable time; a different calendar day. The clock is set per case. The console's `computeDayIsLive(_day)` and its `parseScheduleTimeToMinutes` are compared on the same cases where their inputs overlap. Where the console's variant differs by design, pin it as `// KNOWN DIFFERENCE` with the reason read from its comment |

### 5.3 Pages (browser, real pages, routed data)

**Every page in §3, on every fixture of its type** (a loop, one `it` per page ×
fixture):

- loads with no `pageerror` and no `console.error` (§4.2 rule 4)
- renders its main content, not an empty or "error loading" state (the
  implementer reads each page for the element that proves it)
- makes no request outside the router (§4.2 rule 2)
- at 375 px wide, public pages and boards have no horizontal page scroll
  (`document.documentElement.scrollWidth <= innerWidth`)

**Console** (`pages/console.test.mjs`, once per type):

| Area | Cases |
|---|---|
| Event and day | the registry's events and days are listed from `config`; picking one loads its snapshot URL (asserted in `page.requests`) |
| Facility progress | `pre`: done 0 / left = total; `mid`: the counts from the fixture; `finished`: "All matches done", and with `completedAt` the actual end; a facility whose `syncedAt` is old on event day shows the stale warning; BYE matches are not counted |
| Live Matches, Standings, Match Finder, Awards | the runbook checks (§5.4) on the `runbook` fixture, plus: every configured court shows a row; Awards reads "Pending" before any Final/Bronze is played and names finishers on `finished` |
| Sign-in | a wrong password (router answers `401`) shows an error and no token is stored; a right one enables the operator controls |
| Resync | posts `POST /sync/<day>` to `CLOUD_RUN_BASE_URL` with the bearer token (asserted in `page.requests`), and shows the result |
| Live/Hide | choosing **false** opens the `confirm` prompt; dismissing it sends nothing; accepting it sends `POST /sync/<day>/live` with `{"isLive":false}`; **auto** and **true** send without a prompt |
| Sync method | switching to GitHub only opens its `confirm` prompt and sends `POST /sync/live-push` `{"enabled":false}` only when accepted; **Check connection** shows the router's `/ping` version and the `live` block from `/sync/config` |
| Schedule board link | **Open schedule** opens `/events/<key>/schedule` (extensionless) |

**Public pages and boards** (`pages/public.test.mjs`):

| Area | Cases |
|---|---|
| Go-live | with `isLive: "auto"`, before the 4-hour lead the page shows its not-live state and no scores; `clock.runFor` past the lead (no reload) switches to live; `isLive: false` hides it on the next poll; `true` shows it before the lead |
| Content | standings, live courts and schedule match the fixture for `mid` and `finished`; the team page shows matchup cards and group tables; the dual-meet template shows the club win summary |
| Board | courts as columns from the highest `CourtAssignment`; a live match highlighted; a duplicate court/time gets the `+n` badge; `?venue=` narrows to one facility; `?courts=` composes with it |
| Saved state | a search and the chosen day survive a reload in the same context (`localStorage`), and a fresh context starts clean |

**Live channel** (`pages/live-channel.test.mjs`, on the console and one
public page per type):

1. The fake Worker connects and delivers one snapshot (harness self-check, §4.3).
2. With the socket open, a pushed snapshot re-renders the page **without** a
   GitHub fetch (`page.requests` shows none between push and render).
3. With the socket open, GitHub is polled every `LIVE_SAFETY_POLL_MS` and not
   every `POLL_INTERVAL_MS` (`clock.runFor`, count the requests).
4. Worker refuses: the page polls GitHub every `POLL_INTERVAL_MS`.
5. Socket drops mid-session: polling resumes; a later reconnect (as the block
   implements it) stops the fast poll again.
6. A pushed snapshot older than the one shown (`publishedAt`) does not
   replace it, if the block implements that. If it does not, pin that it does
   replace it.

### 5.4 The runbook checks (`checks/runbook.mjs`)

Two exported functions, one per runbook section. Each is written against
whatever page it is handed, so the dry run can call them on live data.

```js
/** Runbook §1.2. `page` is the console, signed in, on the scenario's day. */
export async function checkConsole(page, { finished, live, unmapped }) { ... }

/** Runbook §1.3. `pages` are the board, the public page's Live tab and its Standings. */
export async function checkPublic({ board, live, standings }, { finished, live: liveMatch, unmapped }) { ... }
```

They assert, line for line, `_templates/dry-run-checklist-template.md` §1.2
and §1.3:

- Live Matches shows match A's final score; match B has a live pill on its
  court; every other configured court of that facility shows an idle
  placeholder, not a blank.
- Standings shows a warning banner naming C's unmapped code, not a silent
  "Other" bucket.
- Match Finder: searching a player from A returns their matches in schedule
  order with the right opponent and score state.
- Awards loads and lists every category, with "Pending" where nothing is
  decided.
- Board: courts and times render; B's court is in progress. Public Live
  matches the console. Public Standings shows the same warning and
  standings.

The runbook's Live/Hide line is a console-only action and is tested in §5.3,
not here: on live data it is a real write, and the dry run decides
separately whether it tests it at all. `pages/runbook.test.mjs` runs both
functions on each type's `runbook` fixture.

---

## 6. Rules for every test

1. **The page's code is the subject.** Assertions read the DOM or call the
   page's own functions. A test never reimplements a page rule to compute an
   expected value, beyond fixed expected answers written out in the test or
   case file.
2. **No real network, no real waiting.** §4.2.
3. **Pin, then report.** When two copies disagree, or a page does something
   that looks wrong, pin what it does today, mark it as below, and list it in
   the hand-off. Do not change the page.
4. **Selectors by what a person sees.** Prefer role and visible text
   (`getByRole`, `getByText`) over CSS classes, so a restyle doesn't break a
   test that isn't about style. Where a page offers neither, use the most
   stable id or class and say why in a comment.
5. **One behaviour per `it`.** `describe(<page>)`, then
   `it("<what a person sees>")`.

### 6.1 Known-difference marker

```js
// KNOWN DIFFERENCE (played/BYE): the server says a header-only CSV is not complete;
// the console's computeFacilityProgress returns left === 0. The card shows "...".
```

`grep -rn "KNOWN DIFFERENCE" _tests/` lists every disagreement between copies
the suite has found. Each one is the owner's decision: make the copies agree
(and flip the test in that commit), or record why they differ in root
`CLAUDE.md`'s "kept in sync by hand" list.

---

## 7. Proof that the net works

With the suite green, make each breakage below in a scratch working copy,
one at a time, confirm **at least one test fails**, and revert. The list goes
in the commit message.

1. Change one character inside Piggleball `schedule.html`'s `LIVE CHANNEL` block.
2. Make the console's `sideIsBye` check only the team code.
3. Make the console count a match as played with one score.
4. Change the tie-break in PickleDrive `index.html`'s `teamMatchupResult`.
5. Change `'WD'` in PickleDrive `schedule.html`'s `PAIR_SHORT`.
6. Set `GO_LIVE_LEAD_HOURS` to 2 in the standard template.
7. Remove the console's unmapped-code warning banner.
8. Drop the `confirm` before Live/Hide **false** in the console.
9. Make the live channel ignore pushed snapshots.
10. Leave `{{VENUE}}` in a page under `events/`.

---

## 8. Keeping the suite current

The same principle as the
[`sage-tools-api` test suite's §9](sage-tools-api-test-suite-spec.md#9-keeping-the-suite-current),
applied to the site.

### 8.1 The rule

**A change to an in-scope page is not finished until its tests are.** The
test change goes in the same commit. In particular:

| Change | Tests in the same commit |
|---|---|
| A page's visible behaviour | the page test that pins it, edited or added |
| One copy of a hand-copied rule (§0 table) | the other copies changed too, and the consistency or parity test passes. If the copies are meant to differ from now on, a `KNOWN DIFFERENCE` and a line in root `CLAUDE.md` |
| The `LIVE CHANNEL` block | every copy, in one commit; `live-channel-identical` is the `diff` |
| A new event instantiated from a template | its pages added to `pages.mjs`, its event to `_fixtures/config.json`, and `pre`/`mid`/`finished`/`runbook` fixtures, or a manifest entry reusing the template's fixtures when its config is the template's shape. Add this as a step in `_templates/CLAUDE.md` §2 |
| An event finishes | its pages' manifest status set to `frozen` and `liveChannel: false`, in the same commit that drops their `LIVE CHANNEL` block |
| A template change | the template's tests; an event already copied from it is not changed unless its own commit says so |
| A new page shape or snapshot field | a fixture that exercises it |

### 8.2 The guards

`_tests/pages.mjs` lists **every** `.html` file in the repo outside
`node_modules` and `beta` copies, each with a status:

```js
export const pages = {
  "tools/control-center.html": { status: "tested", type: ["dual-meet", "standard", "team"], liveChannel: true },
  "events/piggleball-2026/index.html": { status: "tested", type: "standard", event: "piggleball-2026", liveChannel: true },
  "_templates/standard-tournament-template/index.html": { status: "tested", type: "standard", event: "test-standard", liveChannel: true,
      tokens: { EVENT_KEY: "test-standard", EVENT_TITLE: "Test Standard Event", /* … */ } },
  "events/pnf-x-bup-dual-meet/index.html": { status: "frozen", reason: "finished; no LIVE CHANNEL" },
  "tools/bracket-generator.html": { status: "untested", reason: "evergreen tool; site test spec §11" },
  // … every other file
};
```

`consistency/manifest.test.mjs` fails when:

- an `.html` file exists that `pages.mjs` does not list (a new event or page
  cannot slip in untested without someone writing down why);
- `pages.mjs` lists a file that does not exist;
- the files containing the `LIVE CHANNEL` marker differ from those with
  `liveChannel: true`;
- a `tested` page has no `pages/` test that loads it (each page test file
  exports the paths it covers);
- a `tested` event page's event has no `pre`, `mid`, `finished` and `runbook`
  fixture.

### 8.3 Where the rule lives

- **Root `CLAUDE.md`**, in the `sage-match-control.github.io` section, replacing
  "No bundler, no package.json, no tests":

  > Tests live in `_tests/` (never published: Jekyll skips `_` folders):
  > `npm test --prefix _tests`. Every change to an in-scope page changes
  > `_tests/` in the same commit; a new event gets its `_tests/pages.mjs`
  > entry and fixtures. The hand-copied rules below are checked by the suite:
  > change every copy, never make a test pass by editing one side.

  The "kept in sync by hand" entries for the `LIVE CHANNEL` block, played/BYE
  and the team-event rules each gain "checked by `_tests/`". "Compare them
  with `diff`" becomes "`npm test --prefix _tests` compares them". The go-live
  rule is added to that list.
- **`.githooks/pre-push`** in the site repo runs `npm run test:fast --prefix
  _tests` (no browser, a second or two) and, if `_tests/node_modules` exists,
  the full suite. It refuses the push on failure. Setup per clone:

  ```bash
  git config core.hooksPath .githooks
  ```

- **`sage-docs/docs/technical/`**: a `site-tests.md` page (layers, commands,
  fixtures, the marker, the rule), indexed in `technical/README.md` and linked
  from `control-center.md`.

---

## 9. Build steps

**Rule.** The only files this spec touches outside `_tests/` are
`_fixtures/` (new fixtures, new events in `config.json`), `.gitignore` (one
line), `.githooks/pre-push`, `_templates/CLAUDE.md` (§8.1's new step), root
`CLAUDE.md` and the `sage-docs` pages in §8.3. **No page.** `git diff --stat`
against `main` shows no `.html` file.

1. **Consistency layer first.** `_tests/package.json` (no dependency yet),
   `extract.mjs`, `pages.mjs`, and §5.1. This lands on its own: it already
   replaces the hand `diff` and finds any drift that exists today. If it does
   (two `LIVE CHANNEL` copies differ, say), report it; do not fix the page
   in this commit.
2. Add Playwright (exact version), `.gitignore`'s `_tests/node_modules/`,
   `npm run setup` (`playwright install chromium`), and the helpers in §4.3.
   Write `live-channel.test.mjs` case 1 first: it proves the harness.
3. Fixtures (§4.4), then the parity layer (§5.2).
4. The pages layer (§5.3) and `checks/runbook.mjs` (§5.4).
5. Run the sabotage list (§7).
6. The hook, `_templates/CLAUDE.md`'s new step, root `CLAUDE.md`, the
   `sage-docs` technical page. Update the
   [automated dry run](automated-dry-run-spec.md)'s rendering layer to point
   at the built `runbook.mjs`.

`_tests/package.json` scripts:

```jsonc
{
  "test": "node --test \"consistency/**/*.test.mjs\" \"parity/**/*.test.mjs\" \"pages/**/*.test.mjs\"",
  "test:fast": "node --test \"consistency/**/*.test.mjs\"",
  "test:parity": "node --test \"parity/**/*.test.mjs\"",
  "test:pages": "node --test \"pages/**/*.test.mjs\"",
  "setup": "playwright install chromium"
}
```

---

## 10. Acceptance checklist and hand-off

- [ ] `npm test --prefix _tests` passes on unmodified pages, on Node 22.23.3 or
      later, with no network access beyond `127.0.0.1` (test it offline).
- [ ] Every case in §5 exists.
- [ ] All ten sabotage breakages were caught (§7).
- [ ] `git diff --stat` against `main` shows no `.html` file, and nothing
      outside the files listed in §9's rule.
- [ ] `pages.mjs` lists every `.html` file; the manifest guards pass and each
      was seen to fail when broken.
- [ ] The pre-push hook refuses a push with a failing consistency test.
- [ ] The site's Pages build after the merge publishes nothing under
      `_tests/` (check `https://sage-match-control.github.io/_tests/package.json`
      is `404`).
- [ ] Root `CLAUDE.md`, `_templates/CLAUDE.md`, the `sage-docs` technical page
      and the dry run spec are updated (§8.3, §9 step 6).

**Hand-off report to the owner:** test counts per layer; every
`KNOWN DIFFERENCE` found, with what each copy does; any page behaviour that
could not be tested without a page change; how long `test:fast` and the full
suite take. Then move this spec to `implemented/` per `docs/specs/README.md`.

---

## 11. Out of scope, and what comes next

- Any page change, including test hooks, `data-testid`s or `?fixture=`
  support for more pages.
- Real services. The dry run's preflight and rehearsal cover those (§2).
- Screenshot comparison, accessibility audits, browsers other than Chromium.
- Finished events and `events/archives/`.

**Next candidates**, each its own spec when wanted:

- **Tournament calculator → generator contract.** `buildPlanCsv()` produces
  the CSV the two Apps Script generators consume, and `sage-tools-api`'s
  `scripts/fixtures/` holds real plan CSVs those generators are verified
  against. A test that the calculator, given the same inputs, still produces
  CSVs of that shape would join the two harnesses.
- **Bracket generator's verifiable draw.** It is deterministic by design and
  so is well suited to pinned-output tests.
- **Moving the copied rules into shared files.** The architecture spec's
  Phase 7 decision. If it happens, the parity layer shrinks to ordinary unit
  tests of the shared files, and this suite's fixtures and page tests carry
  over unchanged.
