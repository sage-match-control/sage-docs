# Spec — Mission Control layout, shared sign-in, and attendance marks

> **Status: in progress.** Written 2026-11-21 against the site's `main` that day
> (`tools/control-center.html`, `lib/v1/`, `_tests/`) and `sage-tools-api` 3.8.0, and agreed with the
> owner from two mockups. Three phases, **all three to merge before the UPAP 3rd Annual (22 November
> 2026)** by the owner's decision (§1 D11). **Built on 2026-11-21, all three phases, on the
> `mission-control-layout` branch of the site repo (one commit per phase, each passing `npm run verify`),
> of `sage-docs` and of the workspace repo; not pushed or merged.** The owner's checks (§15.2) and the merge are
> left. See the [Divergences](#divergences) at the end for where the build departs from the text below.
> Every decision is settled (§1).

- **Phase 1:** Mission Control rearranged into cards that use a desktop screen's width, with a link
  to the Scoresheet Generator; and the operator's sign-in shared between the browser's open Control
  Center tabs.
- **Phase 2:** in Control Center, a mark on every player who is not checked in yet, in Match
  Finder, Live Matches and Standings, a count per match, and a count per venue.
- **Phase 3:** the same marks on the schedule board, in a staff view that only a browser with a
  signed-in Control Center tab can see.

---

## 0. Read this first

This spec is written for an implementer with no other context. Build the phases in order, each one
finished and checked (§14) before the next. Within a phase, do the sections in the order written.
Finish as §15 says.

### 0.1 The system in one paragraph

S.A.G.E. runs pickleball tournaments. Each tournament day has one Google Sheet per venue
("facility workbook"). A Cloud Run service, `sage-tools-api`, publishes each day's matches and
standings as a JSON snapshot that every page reads. The site (`sage-match-control.github.io`) is
static HTML and native ES modules: no build step, no framework. Its shared code is the **engine**,
`lib/v1/`, in layers: `platform.js`, `domain/` (pure rules), `data/` (fetching, sockets, storage),
`views/` (HTML builders) and `apps/` (whole pages). **Control Center** (`tools/control-center.html`,
one file, about 4,850 lines) is the operators' console; its first tab, **Mission Control**, is where
an operator signs in. Each event has a public **Tournament Hub** (`events/<key>/index.html`) and a
**schedule board** (`events/<key>/schedule.html`, the venue wall display, `lib/v1/apps/schedule-board.js`),
both thin shells over the engine. Events with **attendance** have an `ATTENDANCE` tab in each
workbook: one row per person, with a `present` tick box; staff tick people in from Control Center's
Attendance tab or a desk page. The API serves that list to signed-in operators
(`GET /v3/days/{day}/facilities/{facility}/people`), and a Cloudflare Worker sends a data-free
"nudge" over a WebSocket when someone is ticked, so open pages re-read it.

### 0.2 Where things are

| Path (from `D:\Personal\SAGE`) | What |
| --- | --- |
| `CLAUDE.md` | the workspace rules. Read the `sage-match-control.github.io` section |
| `sage-match-control.github.io/tools/control-center.html` | Control Center |
| `sage-match-control.github.io/lib/v1/` | the engine. `_templates/CLAUDE.md` §8 has its rules in brief; `sage-docs/docs/technical/site-engine.md` in full |
| `sage-match-control.github.io/lib/v1/views/ticket.js`, `live-matches.js`, `standings.js` | the builders of Match Finder's tickets, Live Matches' rows and Standings' tables, shared with the Hub |
| `sage-match-control.github.io/lib/v1/views/attendance-view.js` | the Attendance tab's list; its `fetchPeople` is the model for Phase 2's reader |
| `sage-match-control.github.io/lib/v1/data/attendance-signal.js` | the attendance nudge socket |
| `sage-match-control.github.io/lib/v1/domain/attendance.js` | `parseAttendanceCsv` (the fixture reader) |
| `sage-match-control.github.io/lib/v1/apps/schedule-board.js`, `lib/v1/css/schedule-board.css` | the schedule board |
| `sage-match-control.github.io/tools/scoresheet-generator.html` | read only: it already accepts `?day=<day key>` (§0.4) |
| `sage-tools-api/src/attendance/domain/personKey.mjs` | the server's one definition of "the same person" |
| `sage-match-control.github.io/_tests/` | unit tests (`unit/`) and the comparison harness (`engine/`). `npm run verify` runs both |
| `sage-match-control.github.io/_fixtures/` | local fixture data for `?fixture=<name>` on localhost |
| `sage-docs/docs/` | the docs (mkdocs) |
| `.claude/launch.json` (workspace root) | dev servers; `site-http-server` serves the site on port 8000 |

### 0.3 Hard rules

1. **Branches, never pushes.** Work on a branch named `mission-control-layout` in
   `sage-match-control.github.io` and in `sage-docs`, and commit there, at least one commit per
   phase, in phase order. Each phase's last commit must pass `npm run verify` on its own, so the
   owner can merge phase by phase. Never push or merge. The owner does that (§15.2).
2. **`lib/v1` changes are compatible only** (`_templates/CLAUDE.md` §8). You may add a file, an
   option or a CSS rule. You may never remove or rename an export, a CSS class a page uses, or a
   theme property. With the new options absent, every existing function returns byte-for-byte
   what it returns today.
3. **No new export of an existing engine file is imported anywhere.** A browser may serve an
   engine file from its cache for up to 10 minutes after a push, so a module that imports a name
   its cached neighbour doesn't have yet breaks the whole page. Every new function another file
   imports goes in a **new** file (§7, §8, §6). New *options* on existing functions are fine: an
   old cached copy ignores an option it doesn't know.
4. **Nothing in `views/` or `domain/` names an operator feature** (the `guards.test.mjs` rule):
   the view hooks are neutral (`markPlayer`, `matchNote`), and the attendance words live in
   Control Center, `data/` and `apps/`.
5. **The public pages show nothing new.** The Hub never passes the new options. The board shows
   marks only in its staff view with a sign-in (§11). Printing never shows them.
6. **Do not change:** `sage-tools-api`, `event-data`, any Apps Script, the live Worker, the
   snapshot format, `events.json`'s shape, the Attendance tab's own list, the desk and scorer
   pages, team events' views (`views/teams.js`), any event shell (`events/*/index.html`,
   `events/*/schedule.html`), or anything under `events/archives/`.
7. **Never call the production API** and never sign in for real. Check with the fixtures and a
   fake token (§14).
8. **Never weaken or delete a test** to make one pass. The test changes are the ones §12 lists.
9. **Write docs in the present tense** (workspace `CLAUDE.md`).

### 0.4 Facts checked against the code (2026-11-21)

**Mission Control (Phase 1)**

- Mission Control is `#organizerResults` > `.organizer-panel` (markup around line 1266). The
  panel's style is `.organizer-panel{ … max-width:480px; margin:0 auto; }` (around line 407). The
  page column is `.wrap{max-width:920px; padding:0 20px 90px}`, so the panel can be up to 880px wide.
- Its markup today, top to bottom: `#organizerDayLabel`; `#orgLoginForm` (username, then a row of
  password and **Sign in**); `#orgAuthStatus`; `#orgSignOutBtn`; `#orgAuthHint`; a divider;
  **Facility sync status** `#orgStatusBox`; `#orgResyncBtn`; the CSV switch row `#orgCsvRow`;
  `#orgFacilityResyncGroup` > `#orgFacilityRow`; `#orgResultBox`; `#orgCheckConnBtn`;
  `#orgConnResultBox`; a divider; **Public site status** (`#goLiveCurrent`, `#goLiveSignInHint`,
  `.golive-buttons` with `data-live` buttons, a hint); a divider; **Sync method**
  (`#syncMethodCurrent`, `#syncMethodButtons` with `data-push` buttons, a hint); a divider;
  `#scorerLinksSection` (ending in its own divider); **Public pages** (a hint and `#orgScheduleRow`).
- The script shows and hides these by `id` with `style.display`: `syncActionAvailability()`
  (around line 3257) toggles `#orgResyncBtn`, `#orgFacilityResyncGroup` (only with more than one
  facility), `.golive-buttons`, `#orgCsvRow` and `#goLiveSignInHint`; `renderAuthStatus()` (around
  line 3806) toggles `#orgSignOutBtn` and `#orgLoginForm`; `renderSyncMethod()` toggles
  `#syncMethodButtons`; `renderScorerLinks()` toggles `#scorerLinksSection`, `#scorerSwitchRow`
  and `#scorerLinkBtn`.
- `goLiveButtonsEl` is `document.querySelector('.golive-buttons')` (around line 3160): the
  **first** element with that class. `#syncMethodButtons` has the same class.
- `renderScheduleLauncher()` (around line 3301) fills `#orgScheduleRow` with **Open schedule**
  (`<pagesPath>schedule`) and **Open Tournament Hub** (`<pagesPath>`), both `target="_blank"`. It is
  called once, from `selectEvent()` (around line 4718), **before** that function sets
  `currentDayIndex = null`, and never when the day changes.
- `pagesPathOf(eventKey)` returns a root-absolute folder with a trailing slash
  (`/events/upap-2026/`). `escapeHtml` exists in the page's script. `selectDay(idx)` (around line
  1911) sets `currentDayIndex = idx`, then calls `renderScorerLinks()`, and later sets `FACILITIES`
  and calls `renderFacilityControls()`. `DAYS[i]` has `key` and `label`. `currentDay()` (line 1544)
  returns the selected day or null.
- The Scoresheet Generator reads `?day=`: it finds the event whose `events.json` entry has that day
  (an archived one too), selects it and presets the day (`scoresheet-generator.html` around line
  827). GitHub Pages and the harness's server both serve `/tools/scoresheet-generator` without
  `.html`.
- `.organizer-text-input + .organizer-text-input{margin-top:8px}` exists, for stacked inputs.
- Nothing outside `tools/control-center.html` uses the classes `organizer-schedule-btn` or
  `organizer-panel`, or the ids above.
- The page's `localStorage` keys are `sage.lastSelection`, `sage.search` and `sage.facilityEnds`,
  each read and written inside `try`/`catch`.
- These labels are quoted by the docs and the operator checklists, and **must not change**:
  **Facility sync status**, **Resync this day now**, **Or resync just one facility**, **Use CSV
  export fallback instead of Sheets API**, **Check connection**, **Public site status**, **Auto**,
  **Force live**, **Force hidden**, **Sync method**, **Live push + GitHub**, **GitHub only**,
  **Scorer links**, **Accepting**, **Stopped**, **Issue scorer link**, **Open schedule**, **Open
  Tournament Hub**, **Sign in**, **Sign out**. Toasts say "Sign in at the top of Mission Control
  first", so sign-in stays at the top.

**Sign-in (Phase 1)**

- The token is `{ token, expiresAt }` (`expiresAt` a number, ms since epoch: `AuthService.mjs`
  line 55) under the key `sage.authToken` (`AUTH_TOKEN_STORAGE_KEY`, line 1502).
  `authTokenStore()` (around line 3316) is `sessionStorage` in a browser tab and `localStorage` when
  installed as an app. `loadAuthToken()`, `saveAuthToken(token, expiresAt)` and `clearAuthToken()`
  follow it; `currentAuthToken()` returns the token string when unexpired, else clears it and calls
  `sessionExpired()`.
- `signIn()` (around line 3832) calls `saveAuthToken(body.token, body.expiresAt)` then
  `renderAuthStatus()`. The **Sign out** handler (around line 3862) calls `clearAuthToken()`,
  `renderAuthStatus()` and a toast. `renderAuthStatus()` redraws everything that depends on
  sign-in. It is first called at line 3869, at load.
- So a new browser tab starts signed out: `sessionStorage` belongs to one tab, and the Mission
  Control links open with `rel="noopener"`, which gives the new tab an empty one.
- Nothing in the site uses `BroadcastChannel` today.

**Attendance (Phase 2)**

- An event has attendance when its `events.json` entry's `attendance` is `"console"` or
  `"desks"`: `attConsoleSetting()` (line 4182). `attConsoleSignedIn()` (line 4208) is
  `!!FIXTURE || !!currentAuthToken()`.
- `GET /v3/days/{day}/facilities/{facility}/people` (operator token) answers `Person[]`:
  `{ key, player, teams: string[], categories: string[], present: boolean, timeIn, withdrawn: boolean, shirt }`.
  `key` is the server's `personKey(name)`: `String(name ?? "").normalize("NFD")`, accents
  (U+0300–U+036F) removed, runs of whitespace made one space, trimmed, lower-cased. The client never
  computes keys today; it reads them.
- `views/attendance-view.js` exports `fetchPeople({ apiBase, token, day, facility })` (line 25). A
  `data/` file may not import from `views/` (`guards.test.mjs`, `ALLOWED`), so Phase 2's reader
  keeps its own copy of those twelve lines (§8). `domain/attendance.js` exports
  `parseAttendanceCsv(text)`, and the fixture files are
  `/_fixtures/<event>/attendance-<facility name, lower-cased, spaces as ->-<fixture>.csv`
  (`attendance-view.js` around line 325).
- `data/attendance-signal.js` exports `createAttendanceSignal({ baseUrl, enabled, event, day, onChange, onStatus })`
  → `{ isOpen(), close() }`. `onChange(facility)` is already coalesced (one call per facility per
  2 s, and one more 11 s after the last nudge).
- A match in `MODEL.matches` has `t1p1`, `t1p2`, `t2p1`, `t2p2` (names, or `'TBD'`), `t1`, `t2`
  (codes), `played` (both scores in), `facility` and `num`. `MODEL.unneededGames` is a `Set` of
  series games that will never be played.
- Match Finder: `createFinder({ … ticket: { decorate: ticketDecorate } … })` (line 1744) spreads
  `ticket` into every `ticketHTML` call; `consoleTicketHTML()` (line 2124) calls
  `Ticket.ticketHTML(m, MODEL, { teamCode, isNext, decorate: ticketDecorate })`.
  `ticketDecorate(m)` (line 2045) returns `{ className?, attrs?, metaHTML? }`; `metaHTML` ends the
  ticket's meta row. `ticketHTML` writes players as `<span class="p">…</span>` (two per side).
- Live Matches: `drawLiveMatches()` (line 2397) calls
  `Live.renderLiveMatches(MODEL, { boardEl, metaEl, extrasEl, facilityExtras, liveRow, emptyMatchupHTML, courtCell })`.
  `liveTableRowsHTML` (`live-matches.js` line 80) writes players as `<span class="p">` inside
  `teamCellHTML`, and the category line as `<td class="live-cat-cell" colspan="3">…</td>`.
  `facilityExtras: name => facilityProgressCardHTML(name, progressForFacility(name))`;
  `facilityProgressCardHTML(name, info)` (line 2357) returns one `.fp-card` with `.fp-line` rows.
- Standings: `createStandings({ … })` (line 1769). In `standings.js`, `pairCell(s, model)`
  (`ticket.js` line 27) writes a Round Robin row's names as two bare `<span>`s;
  `stageTablesHTML(rows, model, { … })` (line 174) calls it in both table layouts and calls
  `stageBlockHTML(key, rows, model)` (line 137) for playoff stages, which calls
  `matchupHTML(pair, model)` → `matchupSideHTML(s, side)` (`<span class="p">`). A standard event
  reaches it through `categoryCardHTML(cat, rows, model, { … })` (line 330), called from
  `createStandings` with `cardOptions()` (around line 618); a dual meet through
  `dualMeetStandingsHTML(model, options)` → `categoryColumnHTML(…, model, options)` →
  `clubSubsectionHTML(clubCode, rows, model, options)` → `stageTablesHTML(rows, model, options)`, and
  `sharedStagesHTML(rows, model)` (line 383) → `stageBlockHTML`. The bracket chart
  (`views/bracket.js`) and full tree are separate and stay unchanged.
- A team event's Match Finder, Live Matches rows and Standings are `views/teams.js` (Control
  Center passes `delegate`, `liveRow` and calls `teams.renderStandings()`): Phase 2 does not mark
  them (D18).
- The theme contract (`_tests/unit/theme-contract.test.mjs`): engine CSS reads only listed custom
  properties. In Control Center `--amber` is green (`--green-dark`), so the not-in colour is the
  literal `#B3541A`, which Control Center already uses for warnings.

**The schedule board (Phase 3)**

- Every `events/<key>/schedule.html` and template shell calls
  `mountScheduleBoard({ eventKey, dayKey, liveBaseUrl, categoryColors, … })` and links
  `/lib/v1/css/schedule-board.css`. `mountScheduleBoard` rejects unknown settings
  (`KNOWN_SETTINGS`), so Phase 3 adds none: it reads `?staff=1`, as it already reads `?compact=1`.
- The board has no sign-in. Its cards come from `cellHTML(m)` (line 142), names from
  `nameHTML(n, codeFallback)` as `<span class="fo-name">`; a board match has `p1a`, `p1b`, `p2a`,
  `p2b`, `s1`, `s2` (null until scored) and `num`. `gridHTML(shown, rows)` builds the grid for the
  screen **and** for print (`buildPrintPages()`, line 240).
- `SNAPSHOT.facilities[].name` lists the day's facilities once a load succeeds (`loadSchedule`,
  line 429).

**The harness**

- `_tests/engine/cases.mjs` screenshots Control Center's views signed out
  (`cc/<event>/<view>/<state>/<phone|desktop>`) and, for `piggleball-2026`, signed in
  (`cc-signed-in/…`, views `mission-control`, `attendance`, `score-entry-save`, `live`,
  `court-set`), with a fake token in `sessionStorage` and a `cloudRun` mock whose `/people` reply is
  `parseAttendanceCsv` of `_fixtures/attendance-demo-2026/attendance-main-pre.csv`, whose names are
  **not** piggleball's. Phone is 375×812, desktop 1280×800.
- Board cases are `scheduleCases()` (`std-schedule`, `dm-schedule`, `team-schedule`), with views
  as `[name, query, steps]` (`['compact', '?compact=1', []]`, `['print', '', [printMedia()]]`).
- `_tests/helpers/browser.mjs` `openPage(browser, url, options)` seeds `storage` with
  `context.addInitScript` before the page's scripts run; `_tests/helpers/runCase.mjs` passes a
  case's fields to it.
- Harness snapshots (`_tests/fixtures/<event>/…json`, `loadSnapshot(event)` in
  `_tests/helpers/fixtures.mjs`) have `facilities[].matchesCsv` with the columns `team1Player1`,
  `team1Player2`, `team2Player1`, `team2Player2`; `parseCsv(text)` in the same helper parses it.

---

## 1. Decisions (settled with the owner, 2026-11-21)

| # | Decision |
| --- | --- |
| **Phase 1: layout** | |
| D1 | **Cards, in this order:** (1) a header card: the day label and sign-in; (2) **Facility sync status** with both resync controls; (3) **Public site status**; (4) **Scorer links** (only when the event has score entry, as today); (5) **Pages and tools**: launch tiles; (6) **Troubleshooting**: Check connection, Sync method and the CSV fallback switch |
| D2 | **Desktop (900px and wider):** two columns inside the 880px column. The header card, Pages and tools, and Troubleshooting span both; Facility sync status is the wider left column (about 59%); Public site status and Scorer links stack on the right. **Narrower:** one column, the D1 order, at most 480px wide and centred |
| D3 | **Troubleshooting is a `<details>` closed by default.** This browser remembers whether it was left open (`localStorage` `sage.troubleshootingOpen`). It shows signed in or out; its contents already handle signed out |
| D4 | **Check connection moves into Troubleshooting**, with its result box |
| D5 | **The CSV fallback switch moves into Troubleshooting**, with a one-line hint. It still affects only the resync buttons, and still hides when signed out |
| D6 | **Open scoresheet generator** tile: `/tools/scoresheet-generator?day=<day key>`, new tab; with no day, `/tools/scoresheet-generator`. Tiles: **Open schedule** (dark, primary), **Open Tournament Hub**, **Open scoresheet generator**, each with a one-line subtitle; Phase 3 adds **Open schedule (staff view)**. The label **Public pages** becomes **Pages and tools**. No "New" badge |
| D7 | **Shorter help text**, exactly as §3 gives it, for: the sign-in hint (now hidden once signed in), Public site status, Sync method, Scorer links; the Public pages paragraph goes. Status lines the script writes are unchanged |
| D8 | **Resync one facility stays a row of buttons**, shown only for more than one facility; it wraps |
| D9 | **Labels stay** (§0.4's list) |
| D10 | **Harness differences are accepted as owner-approved presentation changes** (§12) |
| D11 | **All three phases merge before UPAP** (22 November 2026), an exception to the site's rule of never merging within 3 days of an event, decided by the owner on 2026-11-21. The owner merges; the implementer builds on the branch. Because UPAP runs on these pages, §14's checks are not optional |
| **Phase 1: sign-in** | |
| D12 | **A Control Center tab shares its sign-in with this browser's other open Control Center tabs**, over a same-origin `BroadcastChannel`. A new tab with no sign-in asks; an open, signed-in tab answers; the new tab stores the answer in its own storage (`authTokenStore()`, unchanged). Signing in in one tab signs in the others that are open. **Sign out** in one tab signs out every open tab in this browser, with a toast "Signed out in another tab." there. Other devices, other browsers and private windows are untouched: the channel never leaves this browser profile. Closing every tab still ends a browser-tab sign-in, as today; the installed app keeps `localStorage`, as today |
| D13 | **Nothing goes in a URL**, ever. The channel carries `{ token, expiresAt }` only, never the username or password |
| **Phase 2: attendance marks** | |
| D14 | **A not-in player** is a name whose key is on the day's attendance roster (any facility), not withdrawn, and not present at any facility. **Marked**: the name greyed with a small hollow amber dot before it. A checked-in player, a withdrawn one, and a name not on the roster (TBD, BYE, a typo) get **nothing** |
| D15 | **Only unplayed matches are marked** (and not a series game that is never played). A played match shows as today |
| D16 | **Where:** Match Finder's tickets (each name, and a chip **N not in** at the end of the ticket's meta row); Live Matches' rows (each name, and the chip after the category and round); Standings' list view (each name in the Round Robin tables and the playoff matchup cards; a Round Robin row has no match, so its names are marked whenever the person is not in). The bracket chart and full tree are unchanged |
| D17 | **Per venue:** each Live Matches facility card gains a line **Still to check in N** (non-withdrawn people at that facility not yet in), once that facility's list has loaded, and none when N is 0 |
| D18 | **Standard and dual-meet events.** A team event's Match Finder, Live Matches rows and Standings come from `views/teams.js` and get no marks in this spec; its facility cards still get the D17 line |
| D19 | **Who sees it:** Control Center only, signed in (or a local fixture), on an event with attendance. The Hub never passes the options. Signing out removes every mark at once |
| D20 | **Matching names:** the site gets its own copy of the server's `personKey` (`attendanceKey`, in a new domain file), with a parity test that runs both on the same names and fails if they ever differ |
| D21 | **Loading:** one reader per Control Center page for the selected day: every facility's list when the day is picked, signed in; one facility again on each attendance nudge; everything again when the page comes back into view or the socket reconnects. No polling. A failed read keeps the last list and says nothing (the Attendance tab is where errors show) |
| **Phase 3: the board's staff view** | |
| D22 | **Entry:** a fourth tile, **Open schedule (staff view)**, shown when signed in and the event has attendance, opening `<pagesPath>schedule?staff=1` in a new tab |
| D23 | **Sign-in:** the board asks this browser's open Control Center tabs for the sign-in over D12's channel. It keeps it in memory only, never answers other pages, and drops it on a sign-out message |
| D24 | **No answer means the public board, as usual**, with nothing added: no notice, no badge. So does `?staff=1` on any other device or browser |
| D25 | **With a sign-in:** a bold **STAFF VIEW · ATTENDANCE** badge in the board's top bar the whole time, the D14 marks on each unplayed card's names, and a **N not in** tag in the card's top line |
| D26 | **Never printed:** the board's PDF and printed sheets are exactly as today, staff view or not |
| D27 | **Not remembered:** the board without `?staff=1` is always the public board. The board keeps `?staff=1` in its URL when it rewrites it (court and venue filters), so a refresh stays in staff view |

---

## 2. The layout (Phase 1)

Desktop, signed in (Scorer links shows only for an event with score entry):

```
┌───────────────────────────────────────────────────────────────────────────┐
│ DAY 1                                   Signed in until 6:40 PM. [Sign out]│
├──────────────────────────────────────────────┬────────────────────────────┤
│ FACILITY SYNC STATUS                         │ PUBLIC SITE STATUS         │
│ ● The Foundry  Synced 12s ago  [Google Sheet] │ Auto: live since 6:00 AM   │
│     24 / 61 done · est. finish 5:10 PM       │ [Auto][Force live][Hidden] │
│ ● Live updates: push connected               │ hint                       │
│ [        RESYNC THIS DAY NOW               ] ├────────────────────────────┤
│ Or resync just one facility (2+ venues only) │ SCORER LINKS               │
│ [Venue A] [Venue B] [Venue C]                │ Accepting: …               │
│ (resync result box)                          │ (●) ACCEPTING              │
│                                              │ [Issue scorer link for …]  │
├──────────────────────────────────────────────┴────────────────────────────┤
│ PAGES AND TOOLS                                                           │
│ [OPEN SCHEDULE ↗] [OPEN TOURNAMENT HUB ↗] [OPEN SCORESHEET GENERATOR ↗]    │
│ [OPEN SCHEDULE (STAFF VIEW) ↗]   ← Phase 3, signed in + attendance only    │
├───────────────────────────────────────────────────────────────────────────┤
│ ▸ TROUBLESHOOTING   Check connection, sync method, CSV fallback           │
└───────────────────────────────────────────────────────────────────────────┘
```

Troubleshooting open, on a desktop: three columns (Check connection and its result; Sync method;
the CSV switch and its hint). On a phone they stack, each after a thin rule. Signed out, the header
card's right side is the sign-in form in one row (username, password, **Sign in**), "Not signed in."
and the hint. On a phone the day label sits above the form.

---

# Phase 1 — layout and shared sign-in

## 3. Markup (`tools/control-center.html`)

Replace the whole of `<div id="organizerResults" style="display:none;"> … </div>` (from that
opening tag to its matching closing tag, just before the `</div>` that closes `.wrap`) with the
block below. Keep the HTML comments. Every `id`, `data-*` attribute and inline
`style="display:none;"` the script toggles is the same as today.

```html
  <div id="organizerResults" style="display:none;">
    <!-- Mission Control (sage-docs/docs/specs/.../mission-control-layout-spec.md): cards,
         one column on a phone; on a desktop the sync card sits beside the public-site and
         scorer-link cards. Every id here is read by the script: keep them. -->
    <div class="organizer-panel mc-grid">

      <section class="mc-card mc-head">
        <div class="organizer-day-label" id="organizerDayLabel"></div>
        <div class="mc-auth">
          <div id="orgLoginForm">
            <label class="search-label" for="orgUsernameInput">Sign in</label>
            <div class="organizer-secret-row mc-login-row">
              <input id="orgUsernameInput" type="text" class="organizer-text-input" placeholder="Username" autocomplete="username" />
              <input id="orgPasswordInput" type="password" class="organizer-text-input" placeholder="Password" autocomplete="current-password" />
              <button type="button" id="orgSignInBtn" class="organizer-secondary-btn">Sign in</button>
            </div>
          </div>
          <div class="mc-auth-status-row">
            <p class="organizer-auth-status" id="orgAuthStatus">Not signed in.</p>
            <button type="button" id="orgSignOutBtn" class="organizer-secondary-btn" style="display:none;">Sign out</button>
          </div>
          <p class="organizer-hint" id="orgAuthHint">Your password is exchanged for a session token kept only in this browser. The password itself is never saved.</p>
        </div>
      </section>

      <section class="mc-card mc-sync">
        <label class="search-label">Facility sync status</label>
        <div id="orgStatusBox" class="organizer-status-box"></div>

        <button type="button" id="orgResyncBtn" class="organizer-primary-btn">Resync this day now</button>

        <div id="orgFacilityResyncGroup">
          <label class="search-label">Or resync just one facility</label>
          <div class="organizer-facility-row" id="orgFacilityRow"></div>
        </div>

        <div id="orgResultBox" class="organizer-result" style="display:none;"></div>
      </section>

      <div class="mc-side">
        <!-- Go-live override (spec §4.1): a kill switch as a button, not a
             config edit + redeploy. Affects the PUBLIC index.html only — this
             console's own tabs always stay live, for preview (spec §3). -->
        <section class="mc-card">
          <label class="search-label">Public site status</label>
          <div class="golive-current" id="goLiveCurrent"></div>
          <p class="organizer-hint" id="goLiveSignInHint" style="display:none; margin-top:-2px;">Sign in to change this.</p>
          <div class="golive-buttons" id="goLiveButtons">
            <button type="button" class="golive-btn" data-live="auto">Auto</button>
            <button type="button" class="golive-btn" data-live="true">Force live</button>
            <button type="button" class="golive-btn golive-btn-danger" data-live="false">Force hidden</button>
          </div>
          <p class="organizer-hint">Changes the public event page's Live Matches and Standings only. This console's own tabs always show live data, for preview.</p>
        </section>

        <!-- Scorer links (control-center-score-entry-spec.md §5.8): a link that can
             enter scores and nothing else, and a switch that stops every such link.
             The switch writes config/events.json (PUT /v3/events/{event}/score-entry). -->
        <section class="mc-card" id="scorerLinksSection" style="display:none;">
          <label class="search-label">Scorer links</label>
          <div class="golive-current" id="scorerLinksCurrent"></div>
          <div class="switch-row" id="scorerSwitchRow" style="display:none;">
            <button type="button" class="switch switch-stop" id="scorerSwitch" role="switch" aria-checked="true" aria-labelledby="scorerSwitchLabel"><span class="switch-thumb"></span></button>
            <span class="switch-label" id="scorerSwitchLabel">Accepting</span>
          </div>
          <button type="button" id="scorerLinkBtn" class="organizer-secondary-btn" style="display:none;">Issue scorer link</button>
          <div id="scorerLinkResult" class="organizer-result" style="display:none;"></div>
          <p class="organizer-hint">A scorer link opens this event's scorer page, for entering scores at any venue on that day and nothing else. It works for 24 hours, while this reads <b>Accepting</b>. <b>Stopped</b> blocks every scorer link at once; operators can still enter scores in Match Finder.</p>
        </section>
      </div>

      <!-- The schedule board is a venue wall display, deliberately unlisted from
           the public pages — this is its only entry point, operators launch it
           from here and spectators never see a link to it. Tournament Hub just
           opens the event's real public page, so an operator can see exactly
           what a player/spectator sees right now, in the same click. The
           scoresheet generator opens on this day. See renderScheduleLauncher(). -->
      <section class="mc-card mc-open">
        <label class="search-label">Pages and tools</label>
        <div class="mc-tiles" id="orgScheduleRow"></div>
      </section>

      <!-- For when something is wrong: closed by default, remembered per browser
           (sage.troubleshootingOpen). -->
      <details class="mc-card mc-trouble" id="orgTroubleshooting">
        <summary class="mc-trouble-summary">
          <span class="mc-trouble-title">Troubleshooting</span>
          <span class="mc-trouble-sub">Check connection, sync method, CSV fallback</span>
        </summary>
        <div class="mc-trouble-body">
          <div class="mc-trouble-item">
            <button type="button" id="orgCheckConnBtn" class="organizer-secondary-btn organizer-check-conn-btn">Check connection</button>
            <div id="orgConnResultBox" class="organizer-result" style="display:none;"></div>
          </div>

          <!-- Emergency switch between "live push + GitHub archive" and "GitHub
               only". Writes config/events.json (PUT /v3/settings/live-push), so it works
               even when the live Worker is the thing that is broken. -->
          <div class="mc-trouble-item">
            <label class="search-label">Sync method</label>
            <div class="golive-current" id="syncMethodCurrent"></div>
            <div class="golive-buttons" id="syncMethodButtons" style="display:none;">
              <button type="button" class="golive-btn" data-push="true">Live push + GitHub</button>
              <button type="button" class="golive-btn golive-btn-danger" data-push="false">GitHub only</button>
            </div>
            <p class="organizer-hint">For when live updates misbehave. <b>GitHub only</b> sends every score through GitHub: pages catch up within 30&ndash;60 seconds. Takes effect within about a minute.</p>
          </div>

          <div class="mc-trouble-item" id="orgCsvRow">
            <div class="switch-row">
              <button type="button" class="switch" id="orgUseCsvMethod" role="switch" aria-checked="false" aria-labelledby="orgUseCsvMethodLabel"><span class="switch-thumb"></span></button>
              <span class="switch-label plain" id="orgUseCsvMethodLabel">Use CSV export fallback instead of Sheets API</span>
            </div>
            <p class="organizer-hint">Changes how the resync buttons read the sheets. Turn it on only when a resync fails through the Sheets API.</p>
          </div>
        </div>
      </details>

    </div>
  </div>
```

- `#orgCsvRow` now wraps the switch **and** its hint, so `syncActionAvailability()` hides both
  when signed out. Its old inline margin is gone; spacing comes from §4.
- The four `organizer-divider` elements are gone.
- `#goLiveButtons` is a new `id`, used by §5.1.
- Grep first: nothing else may use the id `orgTroubleshooting` or the class prefix `mc-`.

## 4. Styles (`tools/control-center.html`, the page's `<style>`)

### 4.1 Change one existing rule

`.organizer-facility-row{ display:flex; gap:8px; }` (around line 703) becomes
`.organizer-facility-row{ display:flex; flex-wrap:wrap; gap:8px; }`, and `.organizer-facility-btn`
gets `min-width:120px;`. Leave `.organizer-panel` as it is: §4.3 overrides it.

### 4.2 Delete the launcher rules

Delete the block that starts with the comment `/* Launchers for the schedule wall display (see
#orgScheduleRow). */` (around line 666): `.organizer-schedule-row`, `.organizer-schedule-btn`,
`.organizer-schedule-btn:hover`, `.organizer-schedule-btn.primary` and
`.organizer-schedule-btn.primary:hover`. Nothing uses them after §5.2.

### 4.3 Add the Mission Control rules

Directly after the `.organizer-day-label{ … }` rule (around line 415):

```css
  /* ---------- MISSION CONTROL CARDS (mission-control-layout-spec.md) ----------
     The panel is a grid of cards: one column up to 899px (480px wide, centred,
     as the single card was), two from 900px, where the sync card sits beside
     the public-site and scorer-link cards and the rest span both columns. */
  .organizer-panel.mc-grid{
    background:none; box-shadow:none; border-radius:0; padding:0;
    max-width:480px; margin:0 auto;
    display:grid; gap:16px; grid-template-columns:minmax(0,1fr); align-items:start;
  }
  .mc-card{background:var(--white); border-radius:18px; padding:22px; box-shadow:var(--card-shadow); min-width:0;}
  .mc-side{display:grid; gap:16px; align-content:start; min-width:0;}
  .mc-card > .organizer-hint:last-child{margin-bottom:0;}
  .mc-head .organizer-day-label{margin-bottom:14px;}
  .mc-login-row .organizer-text-input + .organizer-text-input{margin-top:0;}
  .mc-login-row .organizer-secondary-btn{flex:0 0 auto;}
  .mc-auth-status-row{display:flex; align-items:center; justify-content:space-between; gap:8px 12px; flex-wrap:wrap;}
  .mc-auth-status-row .organizer-auth-status{margin:8px 0;}
  .mc-auth #orgAuthHint{margin:0;}
  .mc-sync #orgFacilityResyncGroup{margin-top:16px;}
  .mc-tiles{display:grid; grid-template-columns:minmax(0,1fr); gap:10px;}
  .mc-tile{
    display:flex; flex-direction:column; gap:3px; min-width:0;
    padding:12px 14px; border-radius:12px;
    border:1px solid rgba(20,27,44,.18); background:var(--white);
    color:var(--ink); text-decoration:none;
    transition:border-color .12s ease, background .12s ease;
  }
  .mc-tile:hover{border-color:var(--cork);}
  .mc-tile-title{font-family:'Barlow Condensed',sans-serif; font-weight:700; font-size:15px; letter-spacing:.04em; text-transform:uppercase;}
  .mc-tile-title::after{content:"\00a0\2197"; font-weight:400; opacity:.6;}
  .mc-tile-sub{font-size:12px; line-height:1.4; color:var(--muted);}
  .mc-tile.primary{background:var(--ink); border-color:var(--ink); color:var(--white);}
  .mc-tile.primary:hover{background:var(--navy-deep); border-color:var(--navy-deep);}
  .mc-tile.primary .mc-tile-sub{color:rgba(255,255,255,.78);}
  .mc-trouble{padding:0;}
  .mc-trouble-summary{
    display:flex; flex-wrap:wrap; align-items:baseline; gap:2px 10px;
    padding:18px 22px; cursor:pointer; list-style:none;
  }
  .mc-trouble-summary::-webkit-details-marker{display:none;}
  .mc-trouble-summary::before{content:"\25B8"; color:var(--muted); display:inline-block; transition:transform .12s ease;}
  .mc-trouble[open] > .mc-trouble-summary::before{transform:rotate(90deg);}
  .mc-trouble-summary:focus-visible{outline:2px solid var(--navy); outline-offset:-2px; border-radius:18px;}
  .mc-trouble-title{font-family:'Barlow Condensed',sans-serif; font-weight:700; font-size:16px; letter-spacing:.04em; text-transform:uppercase; color:var(--ink);}
  .mc-trouble-sub{font-size:12px; color:var(--muted);}
  .mc-trouble-body{padding:0 22px 22px; display:grid; grid-template-columns:minmax(0,1fr);}
  .mc-trouble-item{padding:16px 0; border-top:1px solid rgba(20,27,44,.1); min-width:0;}
  .mc-trouble-item .organizer-hint{margin-bottom:0;}
  .mc-trouble-item .switch-row{margin:0;}
  .mc-trouble .organizer-check-conn-btn{margin-top:0;}
  @media (min-width:900px){
    .organizer-panel.mc-grid{max-width:none; grid-template-columns:minmax(0,1.45fr) minmax(0,1fr);}
    .mc-head, .mc-open, .mc-trouble{grid-column:1 / -1;}
    .mc-head{display:flex; align-items:flex-start; justify-content:space-between; gap:24px;}
    .mc-head .organizer-day-label{margin:4px 0 0; text-align:left;}
    .mc-auth{flex:0 1 480px; min-width:0;}
    .mc-tiles{grid-template-columns:repeat(3, minmax(0,1fr));}
    .mc-trouble-body{grid-template-columns:repeat(3, minmax(0,1fr)); gap:0 28px;}
  }
```

With Phase 3's fourth tile the desktop tiles wrap to a second row; that is expected. If a value
here looks wrong in §14's checks, adjust it, keep the structure, and say so in your report. Change
no rule outside Mission Control.

## 5. Script: the layout (`tools/control-center.html`)

### 5.1 The go-live buttons by id

Around line 3160, `const goLiveButtonsEl = document.querySelector('.golive-buttons');` becomes
`const goLiveButtonsEl = document.getElementById('goLiveButtons');`.

### 5.2 `renderScheduleLauncher()`: the tiles

Replace the function's body (around line 3301; keep its comment, adding one line saying the third
tile opens the Scoresheet Generator on the current day):

```js
function renderScheduleLauncher(){
  if(!orgScheduleRowEl) return;
  if(!CURRENT_EVENT_KEY){ orgScheduleRowEl.innerHTML = ''; return; }
  const base = pagesPathOf(CURRENT_EVENT_KEY);
  const day = currentDayIndex !== null ? DAYS[currentDayIndex] : null;
  // The generator finds the event from the day key itself (scoresheet-generator.html, ?day=).
  const scoresheetHref = '/tools/scoresheet-generator' + (day ? `?day=${encodeURIComponent(day.key)}` : '');
  const tile = (href, title, sub, cls) =>
    `<a class="mc-tile${cls ? ' ' + cls : ''}" target="_blank" rel="noopener" href="${escapeHtml(href)}">` +
    `<span class="mc-tile-title">${title}</span><span class="mc-tile-sub">${sub}</span></a>`;
  orgScheduleRowEl.innerHTML =
    tile(`${base}schedule`, 'Open schedule', 'The venue wall display', 'primary') +
    tile(base, 'Open Tournament Hub', 'What players and spectators see') +
    tile(scoresheetHref, 'Open scoresheet generator', day ? `Printable scoresheets for ${escapeHtml(day.label)}` : 'Printable scoresheets');
}
```

(Phase 3, §11.1, adds the fourth tile to this function.)

### 5.3 Call it when the day changes

1. In `selectDay()` (around line 1911), directly after `renderScorerLinks();`, add
   `renderScheduleLauncher();`.
2. In `selectEvent()` (around line 4704; the call is around line 4718), **move** the existing
   `renderScheduleLauncher();` to directly after `currentDayIndex = null;` in the same function.

### 5.4 The sign-in hint only when signed out

Next to `const orgAuthStatusEl = …` (around line 3124) add
`const orgAuthHintEl = document.getElementById('orgAuthHint');`. In `renderAuthStatus()`, add
`if(orgAuthHintEl) orgAuthHintEl.style.display = 'none';` to the signed-in branch and
`if(orgAuthHintEl) orgAuthHintEl.style.display = '';` to the signed-out branch.

### 5.5 Remember Troubleshooting

After `const orgConnResultBox = …` (around line 4038):

```js
// Troubleshooting opens closed; this browser remembers if it was left open (mission-control-layout-spec D3).
const STORAGE_TROUBLESHOOTING_KEY = 'sage.troubleshootingOpen';
const orgTroubleshootingEl = document.getElementById('orgTroubleshooting');
if(orgTroubleshootingEl){
  try{ orgTroubleshootingEl.open = localStorage.getItem(STORAGE_TROUBLESHOOTING_KEY) === '1'; } catch(e){}
  orgTroubleshootingEl.addEventListener('toggle', () => {
    try{ localStorage.setItem(STORAGE_TROUBLESHOOTING_KEY, orgTroubleshootingEl.open ? '1' : '0'); } catch(e){}
  });
}
```

Do not change `syncActionAvailability()`.

## 6. Shared sign-in (D12, D13)

### 6.1 New file `lib/v1/data/operator-session.js`

```js
// The operator's sign-in, shared between this browser's open pages of the site
// (sage-docs/docs/specs/.../mission-control-layout-spec.md §6). A BroadcastChannel
// reaches only same-origin pages in this browser profile: never another device,
// another browser or a private window. It carries { token, expiresAt } and nothing
// else; nothing here touches storage or a URL.
//
// Messages: { type: 'ask' }, { type: 'session', token, expiresAt }, { type: 'signed-out' }.

export const OPERATOR_SESSION_CHANNEL = 'sage.operator-session';

const valid = s => !!s && typeof s.token === 'string' && s.token !== ''
  && typeof s.expiresAt === 'number' && s.expiresAt > Date.now();

/**
 * @param {{ getSession?: () => ({ token: string, expiresAt: number } | null),
 *           onSession: (s: { token: string, expiresAt: number }) => void,
 *           onSignedOut: () => void,
 *           answer?: boolean }} options
 *   getSession   this page's own unexpired sign-in, or null. Read only when answer is true
 *   onSession    another page signed in, or answered an ask, with an unexpired sign-in
 *   onSignedOut  another page signed out
 *   answer       true (Control Center): answer other pages' asks. false (the board): never answer
 * @returns {{ ask(): void, announce(s): void, signedOut(): void, close(): void }}
 *   Without BroadcastChannel every method does nothing.
 */
export function createOperatorSession({ getSession = () => null, onSession, onSignedOut, answer = true }){
  if(typeof BroadcastChannel !== 'function') return { ask(){}, announce(){}, signedOut(){}, close(){} };
  const channel = new BroadcastChannel(OPERATOR_SESSION_CHANNEL);
  channel.onmessage = e => {
    const msg = e.data || {};
    if(msg.type === 'ask'){
      if(!answer) return;
      const s = getSession();
      if(valid(s)) channel.postMessage({ type: 'session', token: s.token, expiresAt: s.expiresAt });
    }else if(msg.type === 'session'){
      if(valid(msg)) onSession({ token: msg.token, expiresAt: msg.expiresAt });
    }else if(msg.type === 'signed-out'){
      onSignedOut();
    }
  };
  return {
    ask(){ channel.postMessage({ type: 'ask' }); },
    announce(s){ if(valid(s)) channel.postMessage({ type: 'session', token: s.token, expiresAt: s.expiresAt }); },
    signedOut(){ channel.postMessage({ type: 'signed-out' }); },
    close(){ channel.close(); },
  };
}
```

A `BroadcastChannel` never delivers a message to the object that sent it, so a page never hears
its own `ask`.

### 6.2 Control Center

1. Import it beside the other data imports (around line 1455):
   `import { createOperatorSession } from '/lib/v1/data/operator-session.js';`
2. Directly **before** the load-time `renderAuthStatus();` call (line 3869), add:

```js
// This browser's other open Control Center tabs share their sign-in (mission-control-layout-spec D12).
const operatorSession = createOperatorSession({
  getSession: () => (currentAuthToken() ? loadAuthToken() : null),
  onSession: s => {
    const mine = currentAuthToken() ? loadAuthToken() : null;
    if(mine && mine.expiresAt >= s.expiresAt) return;   // ours is as new or newer
    saveAuthToken(s.token, s.expiresAt);
    renderAuthStatus();
  },
  onSignedOut: () => {
    if(!loadAuthToken()) return;
    clearAuthToken();
    renderAuthStatus();
    showToast('ok', 'Signed out in another tab.', 'auth');
  },
});
```

   and directly **after** the load-time `renderAuthStatus();`, add
   `if(!currentAuthToken()) operatorSession.ask();`
3. In `signIn()`, after `saveAuthToken(body.token, body.expiresAt);`, add
   `operatorSession.announce({ token: body.token, expiresAt: body.expiresAt });`
4. In the **Sign out** handler, after `clearAuthToken();`, add `operatorSession.signedOut();`
5. Do **not** announce from `sessionExpired()` or anywhere else: each tab finds its own expiry.

`signIn` and the Sign out handler run only on a click, after the module has finished loading, so
using `operatorSession` inside them before its declaration line is safe.

---

# Phase 2 — attendance marks in Control Center

## 7. New file `lib/v1/domain/attendance-marks.js`

Pure functions (no browser globals: `guards.test.mjs`). No imports.

```js
// Who on a match or a facility's list is not checked in yet (sage-docs/docs/specs/.../mission-control-layout-spec.md §7).
// The roster's keys come from sage-tools-api (src/attendance/domain/personKey.mjs); attendanceKey is
// the same rule, kept in step by _tests/unit/parity-person-key.test.mjs.

/** A person's key: their name with accents, case and spacing ignored. The server's personKey, exactly. */
export function attendanceKey(name){
  return String(name ?? '')
    .normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .replace(/\s+/g, ' ')
    .trim()
    .toLowerCase();
}

/**
 * One lookup for the whole day, from every facility's people (Person[] arrays).
 * key -> { present, withdrawn }: present if present at any facility; withdrawn only if withdrawn at every one it is on.
 * @param {object[][]} lists
 * @returns {Map<string, { present: boolean, withdrawn: boolean }>}
 */
export function attendanceIndex(lists){
  const index = new Map();
  for(const people of lists || []){
    for(const p of people || []){
      if(!p || !p.key) continue;
      const was = index.get(p.key);
      index.set(p.key, was
        ? { present: was.present || !!p.present, withdrawn: was.withdrawn && !!p.withdrawn }
        : { present: !!p.present, withdrawn: !!p.withdrawn });
    }
  }
  return index;
}

/** True when the name is on the roster, not withdrawn and not present. A name not on it (TBD, BYE) is false. */
export function isNotIn(index, name){
  const e = index && index.get(attendanceKey(name));
  return !!e && !e.withdrawn && !e.present;
}

/** How many different players on an unplayed match are not in; 0 for a played match. */
export function matchNotInCount(m, index){
  if(!m || m.played || !index) return 0;
  const keys = new Set();
  for(const name of [m.t1p1, m.t1p2, m.t2p1, m.t2p2]){
    if(isNotIn(index, name)) keys.add(attendanceKey(name));
  }
  return keys.size;
}

/** How many different, non-withdrawn people on one facility's list are not present. */
export function facilityNotInCount(people){
  const keys = new Set();
  for(const p of people || []) if(p && p.key && !p.withdrawn && !p.present) keys.add(p.key);
  return keys.size;
}
```

## 8. New file `lib/v1/data/attendance-status.js`

Imports only `../domain/attendance.js` (`parseAttendanceCsv`, an existing export),
`../domain/attendance-marks.js` (new file) and `./attendance-signal.js` (existing export).

```js
// The day's attendance lists, kept current for pages that only show who is not in yet
// (sage-docs/docs/specs/.../mission-control-layout-spec.md §8): Control Center's marks and the
// schedule board's staff view. Read through sage-tools-api with the operator's token; re-read on
// an attendance nudge, when the page comes back into view, and when the socket reconnects.
// Nothing polls. A failed read keeps that facility's last list and says nothing.
import { parseAttendanceCsv } from '../domain/attendance.js';
import { attendanceIndex } from '../domain/attendance-marks.js';
import { createAttendanceSignal } from './attendance-signal.js';

// GET one facility's people. A copy of views/attendance-view.js's fetchPeople: data/ may not import views/.
async function readPeople({ apiBase, token, day, facility }){
  const url = `${apiBase}/v3/days/${encodeURIComponent(day)}/facilities/${encodeURIComponent(facility)}/people`;
  const res = await fetch(url, { headers: { 'Authorization': `Bearer ${token}` }, cache: 'no-store' });
  if(!res.ok){ const err = new Error(`HTTP ${res.status}`); err.status = res.status; throw err; }
  return res.json();
}

async function readFixture(event, facility, fixture){
  const slug = facility.toLowerCase().replace(/\s+/g, '-');
  const res = await fetch(`/_fixtures/${encodeURIComponent(event)}/attendance-${encodeURIComponent(slug)}-${encodeURIComponent(fixture)}.csv`, { cache: 'no-store' });
  if(res.status === 400 || res.status === 404) return [];
  if(!res.ok){ const err = new Error(`HTTP ${res.status}`); err.status = res.status; throw err; }
  return parseAttendanceCsv(await res.text());
}

/**
 * @param {{ apiBase: string, getToken: () => (string|null), event: string, day: string,
 *           facilities: string[], liveBaseUrl?: string, fixture?: string|null,
 *           onChange: () => void }} options
 *   getToken    the operator's token now, or null: nothing is read (a fixture reads without one)
 *   onChange    called after any facility's list changed what index() returns
 * @returns {{ index(): Map|null, people(facility): object[]|null, reload(): void, close(): void }}
 *   index()     attendanceIndex of every list read so far, or null before the first one arrives
 *   people(f)   that facility's last list, or null before it arrives
 */
export function createAttendanceStatus(options){
  const { apiBase, getToken, event, day, facilities, liveBaseUrl = '', fixture = null, onChange } = options;
  const lists = new Map();      // facility -> Person[]
  let index = null, closed = false, wasOpen = null;

  async function load(facility){
    const token = getToken();
    if(closed || (!token && !fixture)) return;
    try{
      const people = fixture ? await readFixture(event, facility, fixture) : await readPeople({ apiBase, token, day, facility });
      if(closed || !Array.isArray(people)) return;
      lists.set(facility, people);
      index = attendanceIndex([...lists.values()]);
      onChange();
    }catch(e){ /* keep the last list (D21) */ }
  }
  const loadAll = () => { facilities.forEach(f => { load(f); }); };

  const signal = createAttendanceSignal({
    baseUrl: liveBaseUrl, enabled: !!liveBaseUrl && !fixture, event, day,
    onChange: f => { if(facilities.includes(f)) load(f); },
    onStatus: open => { if(open && wasOpen === false) loadAll(); wasOpen = open; },
  });
  const onVisible = () => { if(document.visibilityState === 'visible') loadAll(); };
  document.addEventListener('visibilitychange', onVisible);
  loadAll();

  return {
    index: () => index,
    people: facility => lists.get(facility) || null,
    reload: loadAll,
    close(){ closed = true; signal.close(); document.removeEventListener('visibilitychange', onVisible); },
  };
}
```

## 9. Engine view hooks (compatible; the Hub never passes them)

Two new options, documented in each file's header comment beside the existing options:

- `markPlayer(name, match) → className` — a class added to one player's name, or `''`. `match` is
  the match the name is shown for, or `null` (a Round Robin standings row).
- `matchNote(match) → html` — HTML added after a Live Matches row's category and round.

With an option absent, the output must be **byte-for-byte** what it is today: build the class
attribute only when the hook returns a non-empty string.

### 9.1 `views/ticket.js`

- `ticketHTML(m, model, options)`: destructure `markPlayer` from `options` too. Write each of the
  four `<span class="p">${escapeHtml(x)}</span>` as
  `<span class="${pClass(x)}">${escapeHtml(x)}</span>`, where
  `const pClass = name => { const c = markPlayer ? markPlayer(name, m) : ''; return c ? 'p ' + c : 'p'; };`
- `pairCell(s, model, options = {})`: a third, optional parameter `{ markPlayer }`. The two name
  spans become `<span${cls(s.player1)}>` and `<span${cls(s.player2)}>`, where
  `const cls = name => { const c = options.markPlayer && name ? options.markPlayer(name, null) : ''; return c ? ` class="${c}"` : ''; };`
  (Note the leading space inside the template.) The `tbd` branches are unchanged.

### 9.2 `views/live-matches.js`

In `liveTableRowsHTML(court, m, model, options)`:

- the players in `teamCellHTML` use the same `pClass` rule as §9.1, with `options.markPlayer` and `m`;
- the category cell becomes
  `<td class="live-cat-cell" colspan="3">${cat ? escapeHtml(cat) + ' &middot; ' : ''}<span class="live-round">${escapeHtml(roundText)}</span>${options.matchNote ? options.matchNote(m) || '' : ''}</td>`.

Add both options to the header comment. `liveEmptyRowHTML` and `options.liveRow` are unchanged.

### 9.3 `views/standings.js`

Thread `markPlayer` down to every list-view name:

1. `matchupSideHTML(s, side, options = {})`: the two `<span class="p">` use the §9.1 `pClass`
   rule with `options.markPlayer` and `options.match` (the side's game).
2. `matchupHTML(pair, model, options = {})`: pass
   `{ markPlayer: options.markPlayer, match: pair[i] ? model.matchByCode.get(pair[i].teamCode) || null : null }`
   to each `matchupSideHTML(pair[i], …)`.
3. `stageBlockHTML(key, rows, model, options = {})`: pass `options` to `matchupHTML`.
4. `stageTablesHTML`: add `markPlayer` to its destructured options; pass `{ markPlayer }` as the
   third argument of all three `pairCell(s, model)` calls, and as the fourth of
   `stageBlockHTML(key, stages.get(key), model)`.
5. `categoryCardHTML`: add `markPlayer` to its destructured options and pass it into its
   `stageTablesHTML` options.
6. `createStandings`: add `markPlayer: options.markPlayer` to what `cardOptions()` returns. The
   dual-meet path already passes the whole `options` object down to `stageTablesHTML`.
7. `sharedStagesHTML(rows, model, options = {})`: pass `options` to its `stageBlockHTML` call, and
   in `categoryColumnHTML` call `sharedStagesHTML(sharedRows, model, options)`.
8. Header comment: add `options.markPlayer(name, match|null)` to the list.

Do not touch `bracket.js`, `bracket-full.js` or the bracket chart calls.

### 9.4 `css/ticket.css`

Add at the end (Control Center loads `ticket.css`; the Hub loads it too but never emits the
classes):

```css
/* A name an operator page marks (options.markPlayer), and its count (a page's own chip):
   Control Center's attendance marks (mission-control-layout-spec §9.4). */
.att-not-in{color:#8A949A;}
.att-not-in::before{content:""; display:inline-block; width:7px; height:7px; margin-right:5px; border:1.5px solid #B3541A; border-radius:50%; vertical-align:1px;}
.att-not-in-chip{display:inline-block; margin-left:6px; padding:1px 7px; border-radius:6px; background:rgba(179,84,26,.12); color:#B3541A; font-size:11px; font-weight:700; letter-spacing:.02em; white-space:nowrap; vertical-align:1px;}
```

If a selector in `ticket.css`, `live-matches.css` or `standings.css` sets a name's colour more
specifically (check `.ticket .p`, `.live-team-players .p`, `.br-table td.pair span`,
`.mu-pair .p`), raise this rule's specificity to match (`.ticket .p.att-not-in`, …) rather than
changing theirs.

## 10. Control Center wiring

1. Imports (beside the others):
   `import { isNotIn, matchNotInCount, facilityNotInCount } from '/lib/v1/domain/attendance-marks.js';`
   and `import { createAttendanceStatus } from '/lib/v1/data/attendance-status.js';`
2. State, next to `let CURRENT_VIEW = null;` (line 1843):
   `let ATT_STATUS = null;            // the day's attendance reader (mission-control-layout-spec §10), or null`
   `let ATT_STATUS_SIGNED_IN = null;  // attConsoleSignedIn() when ATT_STATUS was last (re)started`
3. Functions (anywhere after `attConsoleSignedIn`; they are hoisted):

```js
// ---- Attendance marks (mission-control-layout-spec §10): who is not in yet, in Match Finder, Live Matches and Standings ----
function attStatusRestart(){
  if(ATT_STATUS){ ATT_STATUS.close(); ATT_STATUS = null; }
  ATT_STATUS_SIGNED_IN = attConsoleSignedIn();
  const day = currentDay();
  if(day && attConsoleSetting() && ATT_STATUS_SIGNED_IN){
    ATT_STATUS = createAttendanceStatus({
      apiBase: CLOUD_RUN_BASE_URL, getToken: currentAuthToken, event: CURRENT_EVENT_KEY, day: day.key,
      facilities: FACILITIES.map(f => f.name), liveBaseUrl: LIVE_BASE_URL, fixture: FIXTURE,
      onChange: attMarksChanged,
    });
  }
  attMarksChanged();
}
function attMarksChanged(){
  if(!MATCHES.length) return;
  if(CURRENT_VIEW === 'finder') finder.refresh();
  else if(CURRENT_VIEW === 'live') drawLiveMatches();
  else if(CURRENT_VIEW === 'standings') renderStandings();
}
function attIndex(){ return ATT_STATUS ? ATT_STATUS.index() : null; }
// markPlayer: '' for a played match or a series game never played (D15).
function attMarkPlayer(name, m){
  const index = attIndex();
  if(!index || (m && (m.played || MODEL.unneededGames.has(m)))) return '';
  return isNotIn(index, name) ? 'att-not-in' : '';
}
function attNotInChipHTML(m){
  const index = attIndex();
  if(!index || MODEL.unneededGames.has(m)) return '';
  const n = matchNotInCount(m, index);
  return n ? `<span class="att-not-in-chip">${n} not in</span>` : '';
}
```

4. Restart it:
   - in `selectDay()`, directly after `renderFacilityControls();`: `attStatusRestart();`
   - in `selectEvent()`, directly after `currentDayIndex = null;` (beside §5.3's move):
     `attStatusRestart();`
   - at the end of `renderAuthStatus()`:
     `if(attConsoleSignedIn() !== ATT_STATUS_SIGNED_IN) attStatusRestart();`
     (`renderAuthStatus` first runs at load, before a day exists; `attStatusRestart` then only
     records the state.)
5. Match Finder: `createFinder({ … ticket: { decorate: ticketDecorate, markPlayer: attMarkPlayer } … })`,
   and `markPlayer: attMarkPlayer` in `consoleTicketHTML`'s options. In `ticketDecorate(m)`, add
   `attNotInChipHTML(m)` to the end of `metaHTML` in both of its returns: the first becomes
   `const extra = warn + attNotInChipHTML(m); if(!attrs) return extra ? { metaHTML: extra } : {};`
   and the second's `metaHTML` becomes `scoreHintHTML(m) + warn + attNotInChipHTML(m)`.
6. Live Matches: in `drawLiveMatches()` add `markPlayer: team ? undefined : attMarkPlayer` and
   `matchNote: team ? undefined : attNotInChipHTML`, and change `facilityExtras` to
   `name => facilityProgressCardHTML(name, progressForFacility(name), ATT_STATUS ? ATT_STATUS.people(name) : null)`.
   `facilityProgressCardHTML(name, info, people = null)` then adds, after its second `.fp-line`,
   `${people && facilityNotInCount(people) ? `<div class="fp-line">Still to check in <b>${facilityNotInCount(people)}</b></div>` : ''}`
   (D17; for team events too).
7. Standings: `createStandings({ … markPlayer: (name, m) => (CURRENT_TYPE === 'team' ? '' : attMarkPlayer(name, m)) })`.

---

# Phase 3 — the schedule board's staff view

## 11. Staff view

### 11.1 The tile (`tools/control-center.html`)

In `renderScheduleLauncher()` (§5.2), when signed in and the event has attendance, add a fourth
tile after the scoresheet one:

```js
    + (currentAuthToken() && attConsoleSetting()
      ? tile(`${base}schedule?staff=1`, 'Open schedule (staff view)', 'Who is not in yet, for this browser only')
      : '')
```

and call `renderScheduleLauncher()` at the end of `renderAuthStatus()` too, so the tile follows
sign-in.

### 11.2 The board (`lib/v1/apps/schedule-board.js`)

1. Imports: `import { CLOUD_RUN_BASE_URL } from '../platform.js';` (add to the existing
   `fixtureName` import), `import { createOperatorSession } from '../data/operator-session.js';`,
   `import { createAttendanceStatus } from '../data/attendance-status.js';`,
   `import { isNotIn, attendanceKey } from '../domain/attendance-marks.js';` All four are new files
   or existing exports (§0.3 rule 3).
2. State, after `let compact = …`:

```js
  // Staff view (mission-control-layout-spec §11): ?staff=1, and only with a sign-in from an open Control Center
  // tab in this browser. Without one the board is the public board, unchanged (D24). Never printed (D26).
  const staffAsked = new URLSearchParams(location.search).get("staff") === "1";
  let staffToken = null;        // in memory only (D23)
  let attStatus = null;
```

3. `syncUrl()`: keep the parameter, after the `compact` line:
   `if(staffAsked) url.searchParams.set("staff", "1");` (D27).
4. Marks: change `cellHTML(m)` to `cellHTML(m, staff = null)`, where `staff` is the index or
   `null`. When `staff` is non-null and the match is not done (`!(m.s1 !== null && m.s2 !== null)`)
   and not `m.notNeeded`:
   - `nameHTML(n, codeFallback, staff)` adds ` att-not-in` to `fo-name`'s class when
     `n && isNotIn(staff, n)` (give `nameHTML` the third parameter, default `null`, and pass it from
     the four standard / dual-meet calls; team events' `teamSideNamesHTML` is unchanged, D18);
   - the card's `cell-top` gets, after `cell-stage`,
     `<span class="cell-notin">${n} not in</span>` when `n > 0`, where `n` is the number of
     different keys among `p1a, p1b, p2a, p2b` that `isNotIn` (count with `attendanceKey`, as
     `matchNotInCount` does; the board's match shape differs, so count here).
   `gridHTML(shown, rows = DATA.rows, staff = null)` passes `staff` to every `cellHTML`.
   `renderGrid()` passes `staffIndex()` (below); `buildPrintPages()` passes nothing, so print
   is unchanged (D26).
5. The badge: when staff view is active, add to the bar once:
   `document.querySelector(".bezel").classList.add("staff-view")` and insert
   `<span class="staff-badge" id="staffBadge">Staff view · attendance</span>` as the bar's first
   child; remove both when it ends.
6. Wiring, after `liveChannel.follow(eventKey, dayKey);`:

```js
  function staffIndex(){ return staffToken && attStatus ? attStatus.index() : null; }
  function staffStop(){
    staffToken = null;
    if(attStatus){ attStatus.close(); attStatus = null; }
    const bar = document.querySelector(".bezel"); if(bar) bar.classList.remove("staff-view");
    const badge = byId("staffBadge"); if(badge) badge.remove();
    if(DATA) renderGrid();
  }
  // The badge appears with the first list that loads, not before: a refused read (attendance off,
  // a bad token) leaves the board exactly the public board (D24).
  function staffBadgeOn(){
    const bar = document.querySelector(".bezel");
    if(bar && !byId("staffBadge")){
      bar.classList.add("staff-view");
      bar.insertAdjacentHTML("afterbegin", `<span class="staff-badge" id="staffBadge">Staff view &middot; attendance</span>`);
    }
  }
  function staffStart(){
    if(!staffToken || attStatus || !SNAPSHOT) return;
    attStatus = createAttendanceStatus({
      apiBase: CLOUD_RUN_BASE_URL, getToken: () => staffToken, event: eventKey, day: dayKey,
      facilities: (SNAPSHOT.facilities || []).map(f => f.name), liveBaseUrl, fixture: FIXTURE,
      onChange: () => { if(attStatus && attStatus.index()) staffBadgeOn(); if(DATA) renderGrid(); },
    });
  }
  if(staffAsked){
    const session = createOperatorSession({
      answer: false,
      onSession: s => { staffToken = s.token; staffStart(); },
      onSignedOut: staffStop,
    });
    session.ask();
  }
```

   and at the end of the success path of `loadSchedule` (after `renderBoard();`), add
   `if(staffToken && !attStatus) staffStart();` so a sign-in that arrived before the first
   snapshot starts once the facilities are known. Marks and badge both wait for the first list
   that loads; if the API refuses, `index()` stays `null` and the board is the public board.
   Item 5 above describes the badge's markup; `staffBadgeOn()` is where it is added.
7. Expiry: in `onSession`, also set a timer to `staffStop()` at `s.expiresAt`.
8. Fixture: on localhost with `?fixture=`, no Control Center tab is needed: when `FIXTURE` and
   `staffAsked`, call `onSession` yourself with `{ token: 'fixture', expiresAt: Date.now() + 3600e3 }`
   (the reader reads fixture CSVs without a token).

### 11.3 `lib/v1/css/schedule-board.css`

Add at the end. Use only custom properties already in the board's `THEME CONTRACT` block or
literals (the theme contract test checks):

```css
/* Staff view (mission-control-layout-spec §11): never in print. */
.staff-badge{font-family:'Barlow Condensed',sans-serif; font-weight:700; font-size:13px; letter-spacing:.08em; text-transform:uppercase; padding:4px 10px; border-radius:6px; background:#B3541A; color:#fff; margin-right:12px; white-space:nowrap;}
.fo-name.att-not-in{opacity:.55;}
.fo-name.att-not-in::before{content:""; display:inline-block; width:8px; height:8px; margin-right:5px; border:2px solid #B3541A; border-radius:50%; vertical-align:1px;}
.cell-notin{font-size:11px; font-weight:700; padding:1px 6px; border-radius:5px; background:#B3541A; color:#fff; white-space:nowrap;}
@media print{ .staff-badge, .cell-notin{display:none !important;} .fo-name.att-not-in{opacity:1;} .fo-name.att-not-in::before{display:none;} }
```

Check the badge and tag on the board's dark bar and its cards at the compact and full header
sizes; adjust sizes, not the structure.

---

## 12. Tests (`sage-match-control.github.io/_tests/`)

### 12.1 Unit tests (`npm test`)

- **Phase 1:** `_tests/unit/data/data-modules.test.mjs` — `createOperatorSession`: with a fake
  `BroadcastChannel` on `globalThis` (a class whose instances share a list and deliver to every
  instance but the sender), check: `ask` from one gets `session` from an answering one with a valid
  sign-in and nothing from `answer: false` or an expired one; `announce` reaches `onSession`;
  `signedOut` reaches `onSignedOut`; an expired or malformed `session` message is ignored; without
  `BroadcastChannel` every method is a no-op.
- **Phase 2:** a new `_tests/unit/domain/attendance-marks.test.mjs` covering every function in
  §7: accents, case and spacing in `attendanceKey`; present at one facility and not another;
  withdrawn at one only; TBD and BYE; a played match counts 0; one person on both sides counts
  once. `data-modules.test.mjs` — `createAttendanceStatus` with faked `fetch`, `document` and
  `WebSocket` (the file already fakes them for the attendance signal): reads every facility once
  with the bearer token, reads nothing without a token, keeps a list on a failed read, re-reads one
  facility on a nudge, `close()` stops it. `_tests/unit/views/ticket.test.mjs`,
  `live-matches.test.mjs` and `standings.test.mjs`: without the options the output equals today's
  (the existing tests already pin it: leave them as they are); with `markPlayer` the named span
  gains the class and nothing else changes; `matchNote` lands in the category cell.
- **Phase 2:** a new `_tests/unit/parity-person-key.test.mjs`, built like `parity-server.test.mjs`
  (skips when `sage-tools-api` is not beside the repo): import
  `sage-tools-api/src/attendance/domain/personKey.mjs` and `lib/v1/domain/attendance-marks.js`, and
  assert `attendanceKey(n) === personKey(n)` for at least: `'Ana Reyes'`, `'  Ana   Reyes '`,
  `'ANA reyes'`, `'José Niño'`, `'Ñino'`, `'Dan\tEmil\nDizon'`, `'Yanna Palo-Apilado'`,
  `"O'Brien"`, `''`, `null`, `undefined`, `'Zoë Ångström'`, and every name in every harness
  snapshot's `matchesCsv`.
- **Phase 3:** none beyond the above; the board is covered by the harness.

### 12.2 The harness (`npm run compare`)

1. **`_tests/helpers/browser.mjs`**: a new optional `openPage` option
   `operatorSession: { token, expiresAt } | null` (document it in the header comment). When set,
   add an init script that, on the site's origin, opens a `BroadcastChannel('sage.operator-session')`
   and answers every `{ type: 'ask' }` with
   `{ type: 'session', token, expiresAt }`. `runCase.mjs` passes `c.operatorSession`. `common()`
   in `cases.mjs` defaults it to `null`.
2. **A people reply built from the event's own players.** In `cases.mjs`, add
   `peopleFor(event)`: every distinct non-empty, non-`TBD`, non-`bye` name in
   `loadSnapshot(event).facilities[].matchesCsv` (`team1Player1`, `team1Player2`, `team2Player1`,
   `team2Player2`, via the helper's `parseCsv`), in first-seen order, as
   `{ key: attendanceKey(name), player: name, teams: [], categories: [], present: i % 2 === 0, timeIn: '', withdrawn: false, shirt: '' }`
   (import `attendanceKey` from `lib/v1/domain/attendance-marks.js`). Every second person is not in.
3. **Control Center, signed in:** in `controlCenterSignedInCases()`, the `/people` reply becomes
   `peopleFor(event)` for every view **except** `attendance` (which keeps the demo CSV, so its
   screenshots don't change). Add views
   `['mission-control-troubleshooting', [ifPresent('#orgTroubleshooting > summary', click('#orgTroubleshooting > summary'))]]`,
   `['finder', [tab('Match Finder')]]` and `['standings', [tab('Standings')]]`.
4. **The board:** in `scheduleCases()`, for `std-schedule` add views `['staff', '?staff=1', []]`
   and `['staff-print', '?staff=1', [printMedia()]]`, with
   `operatorSession: { token: 'harness-operator-token', expiresAt: Date.parse(time) + 3600e3 }` and
   `cloudRun: [{ ...attendancePeople, reply: () => ({ status: 200, body: peopleFor(event) }) }]`.
   Add `['staff-no-session', '?staff=1', []]` with neither (D24: identical to `default`).
5. **`_tests/engine/accepted.mjs`**, at the end, with a section comment naming this spec:

```js
  // ---- Mission Control layout, shared sign-in and attendance marks (sage-docs/docs/specs/.../mission-control-layout-spec.md) ----
  {
    case: /^cc(-signed-in)?\/[^/]+\/mission-control(-troubleshooting)?\//, kind: 'both', row: 'owner 2026-11-21 mission control',
    reason: 'Mission Control is cards: two columns on a desktop, Troubleshooting (Check connection, Sync method, CSV fallback) folded at the bottom, a Pages and tools row with the Scoresheet Generator (and, signed in on an event with attendance, the staff-view board), and shorter help text',
  },
  {
    case: /^cc-signed-in\/[^/]+\/(finder|live|court-set|standings|score-entry-save)\//, kind: 'both', row: 'owner 2026-11-21 attendance marks',
    reason: 'signed in on an event with attendance, a player not checked in yet is greyed with an amber ring, an unplayed match shows “N not in”, and each Live Matches facility card counts who is still to check in',
  },
  {
    case: /^std-schedule\/[^/]+\/staff\//, kind: 'both', row: 'owner 2026-11-21 board staff view',
    reason: 'with ?staff=1 and a sign-in from an open Control Center tab, the board shows a Staff view badge, rings on the names not checked in yet, and “N not in” on unplayed cards',
  },
```

   `staff-print` and `staff-no-session` are **not** accepted: they must match the baseline
   exactly (D24, D26). Phase 1's commits carry only the first entry, Phase 2's add the second,
   Phase 3's the third.

### 12.3 Run them

```bash
cd sage-match-control.github.io/_tests && npm install && npx playwright install chromium && npm run verify
```

`npm test` must pass. `npm run compare` must pass with only the entries above: any **other** case
that differs is a bug to fix, not a difference to accept. Look at the branch's screenshots of the
accepted cases (written under `_tests/out/`): phone and desktop, every state.

---

## 13. Documentation

Present tense; keep each page's structure; change only what this spec changes.

### 13.1 `sage-docs`

1. `docs/features/control-center.md`, **Mission Control**: rewrite the "From top to bottom" list
   in the D1 order (keeping each item's existing facts), with **Pages and tools** (the tiles,
   including the staff-view board) and **Troubleshooting** (closed until opened, remembered; Check
   connection, Sync method, the CSV fallback). Say that on a desktop the sync card sits beside the
   public-site and scorer-link cards. In the sign-in paragraph, add D12: a new tab in the same
   browser picks up the sign-in from an open one; signing out in one tab signs out that browser's
   other tabs; other devices are untouched.
2. Same page, **Match Finder**, **Live Matches** and **Standings**: one paragraph each on the
   not-in marks (D14–D19), and in Live Matches the **Still to check in** line.
3. `docs/features/schedule-board.md`: a **Staff view** section (D22–D27): how to open it, what it
   shows, that it needs an open, signed-in Control Center tab in the same browser, that it never
   prints, and not to cast it to a venue screen.
4. `docs/features/event-attendance.md`: one paragraph pointing to the marks in Control Center
   and the board's staff view.
5. `docs/technical/control-center.md`: in **Narrow screens**, the Mission Control grid (§4.3);
   a **Shared sign-in** section (§6: the channel, its three messages, why a `BroadcastChannel`
   leaks nothing new, and that the board never answers); an **Attendance marks** section (§7–§10:
   the key copy and its parity test, the reader, the two view hooks, what re-renders).
6. `docs/technical/site-engine.md`: the two new options (`markPlayer`, `matchNote`) in the
   views' extension points, and the three new files in the layer list.
7. `docs/technical/schedule-board.md`: the staff view (§11), its never-printed rule, and the
   `staff` URL parameter beside `courts`, `venue` and `compact`.
8. `docs/technical/auth.md`: where the operator token lives, with D12.
9. `docs/usage/run-the-day.md`, `docs/usage/troubleshooting.md`, `docs/usage/rehearse.md`,
   `docs/usage/register-and-build-the-site.md`: where a step uses **Check connection**, **Sync
   method** or **Use CSV export fallback**, say it is under **Troubleshooting** in Mission Control
   (`grep -rn "Check connection\|Sync method\|CSV export fallback" docs/usage`). In
   `run-the-day.md`'s "What to watch", one line on the not-in marks.
10. `docs/features/scoresheet-generator.md`: one sentence that **Open scoresheet generator** opens
    it on the console's current day.
11. Any page saying **Open schedule** is "at the foot of Mission Control" (`grep -rn "Open schedule" docs`):
    make it **Pages and tools** in Mission Control.
12. The workspace `CLAUDE.md`'s "Things that must be kept in sync by hand" gains one bullet: the
    person key exists twice, `sage-tools-api/src/attendance/domain/personKey.mjs` and the site's
    `lib/v1/domain/attendance-marks.js` (`attendanceKey`), and
    `_tests/unit/parity-person-key.test.mjs` fails if they differ. Commit it on a
    `mission-control-layout` branch of the workspace repo (`D:\Personal\SAGE`).
13. This spec: move it to `in-progress/` when all three phases are built (the owner's checks,
    §15.2, are left), following `docs/specs/README.md` → *Moving a spec between folders*; update
    its status line; note any change to §4's or §11.3's values in a **Divergences** section at the
    end.
14. `docs/changelog.md` and `mkdocs.yml`: one MINOR entry and version for the branch.

### 13.2 Site repo

1. `_templates/dry-run-checklist-template.md` and `events/upap-2026/dry-run-checklist.md`: the
   **Check connection** and **Use CSV export fallback instead of Sheets API** steps say to open
   **Troubleshooting** first; add a step to the attendance part: "Open **Open schedule (staff
   view)** on the operator laptop (not a venue screen) and check it shows the not-in marks". Leave
   every other event's checklist alone.
2. `_templates/CLAUDE.md` step 6: the schedule board's only links are "the `Open schedule` tiles
   in Mission Control's **Pages and tools**".

---

## 14. Checking it yourself

Start `site-http-server` from the workspace's `.claude/launch.json` (port 8000) and open
`http://localhost:8000/tools/control-center.html?fixture=pre`. `?fixture=` (localhost only) reads
`_fixtures/` instead of the published data, with the registry in `_fixtures/config.json`. Pick
**attendance-demo-2026** (standard, scorer links, two facilities, attendance, fixture `pre`), then
its day. To be signed in without the API, in the browser console:

```js
sessionStorage.setItem('sage.authToken', JSON.stringify({ token: 'x', expiresAt: Date.now() + 3600e3 }));
```

and reload. (Calls to the real API fail with a toast; that is expected.)

### 14.1 Phase 1

At **375×812**, **768×1024**, **1280×800** and **1440×900**, signed out and signed in:

1. The cards are in the D1 order; two columns at 1280 and 1440, one centred 480px column at 375
   and 768; no horizontal scroll.
2. Signed out: username, password and **Sign in** in one row; the hint shows; Enter in the
   password box still calls `signIn()`. Signed in: the form and hint are gone; status and **Sign
   out** show.
3. **Resync this day now** spans the sync card; on the two-facility day, **Or resync just one
   facility** shows two buttons.
4. **Public site status**: signed in, **Force hidden** opens its confirm dialog (cancel it). This
   proves `goLiveButtonsEl` is right.
5. **Scorer links** shows on the right on a desktop (every fixture event has `scoreEntry`).
6. **Pages and tools**: three tiles, each opening a new tab; the scoresheet tile's `href` is
   `/tools/scoresheet-generator?day=<day key>`, follows a day change, and the generator opens on
   that event and day.
7. **Troubleshooting** is closed on a fresh load, and stays as you left it across reloads. Signed
   out, Sync method asks to sign in and the CSV item is hidden; signed in, the CSV switch flips.
8. **Shared sign-in**: with tab A signed in (the console step), open tab B at the same address
   without the console step: B reads signed in within a second. Press **Sign out** in B: A signs
   out too and shows "Signed out in another tab." Then, with both signed out, run in B's console
   `new BroadcastChannel('sage.operator-session').postMessage({ type: 'session', token: 'x', expiresAt: Date.now() + 3600e3 })`
   (what a sign-in announces; the console's channel object is not the page's own, so both tabs
   receive it): both read signed in. A private window, or another browser, stays signed out
   throughout.
9. Every other tab looks and works as before.

### 14.2 Phase 2

Signed in, on attendance-demo-2026 (`_fixtures/attendance-demo-2026/attendance-*-pre.csv` have
people not present; check that some of them play on that day's matches, and if none do, say so and
test with the harness's cases instead):

1. Match Finder: a not-in player's name is greyed with an amber ring; an unplayed match with any
   shows **N not in** at the end of its meta row; a played match shows neither.
2. Live Matches: the same on each court row, the chip after the round; each facility card reads
   **Still to check in N**.
3. Standings: the marks in Round Robin tables and playoff matchup cards; the bracket chart and
   full tree are unchanged.
4. Sign out: every mark and the facility line disappear at once.
5. The Hub (`/events/attendance-demo-2026/` if it exists, or any Hub template page with
   `?fixture=`) shows no marks.
6. No new console errors.

### 14.3 Phase 3

1. Signed in, the fourth tile **Open schedule (staff view)** shows; it opens the board with
   `?staff=1`. With `?fixture=pre`, the board shows the **Staff view · attendance** badge and the
   marks on unplayed cards (§11.2 item 8).
2. Without the fixture, with a signed-in Control Center tab open in the same browser (the console
   step), the board in a new tab gets the badge once the people read succeeds. Here it fails
   against the real API, so check instead that the board stays exactly the public board, with no
   badge or notice: D24.
3. Sign out in Control Center: the board's marks and badge disappear.
4. Print preview (Ctrl+P) in staff view shows sheets without any mark, tag or badge.
5. Changing the court or venue filter keeps `staff=1` in the address.
6. The board without `?staff=1` is unchanged.

Take screenshots of 375 and 1280 for each phase, for your report.

---

## 15. Finishing

### 15.1 Report back

What you changed in each repo (the branch and its commits, phase by phase), the result of
`npm run verify` at the end of each phase, the screenshots from §14, and anything you changed from
the values in §4 or §11.3 or could not do.

### 15.2 The owner's checks (not yours)

- O1. On a real desktop and a real phone, sign in on Mission Control against the live service and
  run §14.1 items 2–8, on a past or rehearsal day.
- O2. With UPAP's real roster (the `ATTENDANCE` tab of The Foundry's workbook, nobody checked in
  yet), check Match Finder, Live Matches and Standings show the marks, then tick one person in
  from the Attendance tab and watch their marks go within a few seconds.
- O3. Open **Open schedule (staff view)** on the operator laptop: badge and marks; then the plain
  **Open schedule** in another browser: no badge, no marks.
- O4. Merge the branch phase by phase (D11), then move this spec to `implemented/`.

---

## 16. Alternatives considered (do not build these)

| Alternative | Why not |
| --- | --- |
| A dropdown for resyncing one facility | At most three venues so far; buttons are one click and exist (D8) |
| Troubleshooting always open, or opening itself on a failure | The owner chose closed by default (D3); the sync card already colours a failed facility |
| Shorter section names (Sync, Public site, Open) | The docs and checklists quote the current names in about twenty places (D9) |
| A facility preset on the scoresheet tile | The owner chose the day only (D6) |
| Keeping the sign-in in `localStorage` in a browser tab | It would outlive closing the browser on a shared venue laptop; D12 keeps "close every tab to sign out" |
| Passing the token to the board in its URL | It would land in history, screenshots and shared links (D13) |
| A "sign in to see staff view" notice on the board | The owner chose the plain public board when there is no sign-in (D24) |
| Matching attendance by team code instead of name | Playoff rows carry seat codes (`HIMD_QF_1`) the roster never has |
| A green tick on checked-in players | The owner chose to mark only who is missing (D14) |
| Marks in the bracket chart and full tree | The owner chose the list view only (D16) |
| Marks in team events' views | `views/teams.js` is a separate path; not needed for UPAP (D18) |
| Reusing the Attendance tab's list as the source | It exists only while that tab is open; the marks are needed on the other tabs |

---

## Divergences

Where the built code departs from the text above, and what the build could not check. The intent of each
section stands.

1. **§10 item 3, `attMarksChanged()` redraws Match Finder whichever tab is showing.** The text redraws it only
   while Match Finder is the tab on screen. A day loads while Mission Control is showing, and Match Finder's list is
   drawn once then and lives on while its tab is hidden, so a list drawn before the first people read arrived
   stayed unmarked until something else redrew it (the harness's `finder` view showed no marks). Live Matches and
   Standings are drawn each time their tab opens, so they are redrawn only when on screen, as written.
2. **§9.3 item 4, `stageTablesHTML` has two `pairCell` calls, not three.** The Round Robin table has one in each of
   its two layouts (§0.4 says "both table layouts"); both pass `{ markPlayer }`.
3. **§11.2 item 4, a team event's staff view marks nothing.** The text leaves `teamSideNamesHTML` unchanged (D18) but
   would still count and tag a team card. The `N not in` tag is left off team cards too, so a team card is never
   half marked.
4. **§11.2 items 6 to 8, one `staffSession` function.** The expiry timer is set in a `staffSession(s)` that both the
   channel's `onSession` and the fixture call, clears a previous timer and is cleared by `staffStop`; in fixture mode
   the board takes the fixture sign-in and does not also ask the channel.
5. **§12.2 item 4, the board's `staff` and `staff-print` cases** pass the `operatorSession` and `cloudRun` through a
   fourth element of the view tuple. `staff-print` and `staff-no-session` match the baseline exactly, as required.
6. **§14 checks not done as written.**
   - **§14.2 item 4 (sign out removes every mark)** was not shown in the browser: with `?fixture=` Control Center counts as
     signed in whatever the token (`attConsoleSignedIn()` is `!!FIXTURE || …`), so the fixture keeps its marks.
     `attStatusRestart()` runs from `renderAuthStatus()` whenever the signed-in state changes, which is what removes them.
   - **§14.2 item 2 (marks on Live Matches rows)**: the fixture's `pre` state has no match on a court. The rows and
     chips were checked in the harness's `live` views (`mid`), and the venue line in the fixture.
   - **§14.1 item 6**: the Scoresheet Generator reads the real `events.json`, which has no fixture event's day, so the
     `?day=` link was checked with a real day key (`piggleball-day1`) and the tile's `href` against the fixture's.
   - **§14.3 item 2 (a board in a new tab against the real API)** was not run: it would call the production API, which this
     build was told not to. A board with no sign-in staying the public board is the harness's `staff-no-session` case.
     The sign-out and sign-in messages, the expiry timer path and the filters keeping `?staff=1` were checked on a throw-away
     board page for the fixture event (untracked, `*beta*`).
7. **§4 and §11.3 values are as written.** On a phone the board's `N not in` tag wraps to a line of its own in the
   card's top line (that line already wraps on a narrow screen), which adds a line of height to an unplayed card there.
   The harness's screenshots show it; nothing was changed.
