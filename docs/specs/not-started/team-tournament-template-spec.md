# Spec — Team tournament event-site template

> **Status: not started.** Nothing here is built. Revised 2026-10-06 against
> `sage-match-control.github.io` at `170de3b`, the first commit after the
> [site engine](../implemented/site-engine-spec.md) merged (`179217f`). It
> also reads `events/pickledrive-anniversary-2026/` (the prototype),
> `tools/control-center.html` and `lib/v1/`.
>
> **Every decision is made.** The owner settled Q1–Q6 on 2026-10-06 (§13).
>
> **PickleDrive's pages are the reference.** The owner builds the
> [Team Tournament Master](team-tournament-master-spec.md) from PickleDrive's
> workbook. So what `events/pickledrive-anniversary-2026/index.html` and
> `schedule.html` read and show is the contract, and the template matches
> it. Any difference the harness finds that §9 doesn't list is fixed to
> match PickleDrive's page and recorded in §17. It is not a question for
> the owner.
>
> The first version (2026-10-05, sage-docs `d967994`) planned to copy
> PickleDrive's inline code into the template, and asked to be revised once
> the engine existed. This is that revision. The engine already holds the
> team views Control Center uses, so the template is a set of shells on the
> engine, like the other two templates.
>
> Split out of [Team tournament](../implemented/pickledrive-club-anniversary-team-tournament-spec.md)
> §15. Its sibling, [Team Tournament Master](team-tournament-master-spec.md),
> is the workbook generator and is a separate piece of work.

Make `_templates/team-tournament-template/`, which gives a team tournament
**everything a standard or dual-meet event gets from `_templates/`**:

- a Tournament Hub and a schedule board;
- a scorer page and an attendance desk page that work on a team event;
- the dry-run runbook;
- the hub board's QR panel;
- the runbook steps, including an interim way to get the workbook (§7).

A new team event then becomes a copy-and-fill job like the other two. It no
longer means copying PickleDrive's 3,006-line Hub and 1,387-line board and
hunting down their event values.

**The deliverables are the template and the engine changes it needs, not an
event.** This work creates no real event.

---

## 0. Read this first (implementer orientation)

This spec and the repos are all you need. Read all of §0, then §1–§8 before
touching a file, then follow §11's steps in order. Every path is relative to
the repo the section names. Most are in `sage-match-control.github.io/`
("the site").

### 0.1 The workspace

`D:\Personal\SAGE` is itself a git repo, holding the workspace `CLAUDE.md`,
and it contains four more repos. Read that `CLAUDE.md` once; it describes
everything below.

| Repo | Role here |
| --- | --- |
| `sage-match-control.github.io/` | **Almost all the work.** The public site, served by GitHub Pages from `main`, with no build step. `lib/v1/` is the engine, `_templates/` the templates, `_tests/` the tests (never published), and `_fixtures/` the local test data (never published) |
| `event-data/` | One documentation edit (§12): `config/README.md`. `config/events.json` is the event registry every page reads; you do not edit it |
| `sage-docs/` | Documentation edits (§12), and this spec's move to `implemented/` at the end |
| `sage-tools-api/` | **Not changed.** It reads no `display` block |
| `D:\Personal\SAGE` itself | The workspace `CLAUDE.md` (§12) |

### 0.2 Words used here

| Word | Means |
| --- | --- |
| **Hub** | An event's public page, `events/<event-key>/index.html`: Match Finder, Live Matches, Standings (and for a team event, Teams) |
| **Board** | The venue wall display, `events/<event-key>/schedule.html`: courts across, time slots down, one cell per match |
| **Shell** | A page that holds only markup, a `THEME` block of CSS colours and one `<script type="module">` calling an engine `mount…` function |
| **Engine** | `lib/v1/`: `platform.js`, `domain/` (pure rules, no DOM), `data/` (registry, snapshots, live channel), `views/` (HTML builders and components), `apps/` (`mountHub`, `mountScheduleBoard`, `mountScorer`, `mountAttendanceDesk`) and `css/`. Imports point one way: platform ← domain ← data ← views ← apps |
| **Snapshot** | One day's published JSON. `{ day, label, isLive, generatedAt, facilities: [{ name, matchesCsv, standingsCsv, rosterCsv, … }] }` |
| **Registry** | `event-data/config/events.json`. `data/registry.js` reads it and `eventConfigFrom` turns one event into an `EventConfig` (`domain/model.js` documents the shape) |
| **Team code** | `<SIDE>_<PAIR>`, e.g. `A_3`, `QF-3_4`, `SF-A_2`, `Fi-J_1`. `PAIR` is the pair number in the matchup (1 = MD at PickleDrive). `SIDE` is a **base team** letter (`A`) in the bracket stage, or `<STAGE>-<SLOT>` in a playoff. `STAGE` is `QF`, `SF`, `Br` or `Fi`. `SLOT` is a **seed** number (`3`) until the organiser types a team letter over it (`A`) |
| **Matchup** | Two teams' set of matches, one per pair, won on total points. The `matchUp` column holds its key, e.g. `A v B` or `SF-1 v SF-2` |
| **Bracket** | A group in the group stage. A team's bracket number is the `bracket` column of `STANDINGSCSV` |
| **Lineup not set** | A match whose player cells are empty or equal their own team code. `domain/matches.js` turns both into `TBD` |

### 0.3 Commands

- **Node** 22.23.3 or later.
- **Tests**, from `sage-match-control.github.io/_tests/` (run `npm install`
  once):
  - `npm test` runs the unit tests: `node --test` over `unit/**`, fast.
  - `npm run compare` runs the comparison harness. Every case renders on the
    **baseline** (the merge-base with `main`, checked out to a temp folder)
    and on your working tree in Playwright's Chromium, and the text and
    screenshots are compared. It takes a long time.
    - `CASE='team-index/.*' npm run compare` runs only matching cases.
    - A failing case writes `baseline.png`, `branch.png`, `diff.png` and
      `text.diff` to `_tests/out/<case>/`.
  - `npm run verify` runs both, and must pass before every push to `main`.
  - If Chromium is missing: `npx playwright install chromium`.
- **Serving the site by hand.** Module scripts don't load from `file://`.
  - Python is **not** installed on this machine, so the `static-site`
    entry in `D:\Personal\SAGE\.claude\launch.json` does not work.
  - Instead, write a 20-line `node:http` static server in your scratchpad
    (not in a repo). It serves the site folder on port 8123, maps an
    extensionless path to `.html` and a folder to `index.html`.
  - Then open `http://localhost:8123/<path>`.
- **Fixture mode.** On `localhost` only, `?fixture=<name>` makes every
  engine page read `/_fixtures/config.json` as the registry and
  `/_fixtures/<event>/<name>.json` as the snapshot (`lib/v1/platform.js`
  `fixtureName()`). A template page under `_templates/` still has its
  `{{TOKENS}}` and won't run as it is. To check a template by hand,
  instantiate a copy (§10.4).

### 0.4 Rules

1. **The engine's versioning rule** (`_templates/CLAUDE.md` §8). Only edit
   `lib/v1/` compatibly:
   - you may add a module, an export, an optional parameter, an option
     or a CSS class, and you may fix a bug;
   - never remove or rename an export, a CSS class a page uses, or a theme
     property, and never change what an existing call returns.

   A module may start using another module's **new** export or option only
   in a **later push to `main`**, because browsers cache a file for up to
   10 minutes. That is why §8 has two pushes.
2. **Never edit a finished event's pages:** `events/pickledrive-anniversary-2026/`,
   `events/piggleball-2026/`, `events/pickle-for-sight-2026/`,
   `events/pnf-x-bup-dual-meet/`, `events/archives/`. PickleDrive's pages
   keep their own inline code and their `MXD` labels for good. Read them;
   don't change them.
3. **Never weaken or delete a test to make a change pass.** If an existing
   case changes, it is either a difference this spec lists (§9), which gets
   an `ACCEPTED` entry naming its §17 row, or a bug.
4. **Don't merge within 3 days of an event.** Check `event-data/config/events.json`
   for a day dated within 3 days; at the time of writing none is after
   2026-10-03.
5. **Work on a branch** in the site repo, `team-template`. The engine and
   the pages are two merges into `main` (§8). The docs repos take ordinary
   commits on `main` after the second merge.
6. **Commits.** One per step of §11, with a plain message saying what
   changed. End every commit message with the attribution line your
   session's instructions give. Don't push or merge without the owner's
   go-ahead.
7. **Docs** (`sage-docs`) are in the present tense: what the system does
   now, never "used to" or "was changed".
8. **Citing a spec from outside `sage-docs`** (code comments, `CLAUDE.md`
   files, READMEs): write `sage-docs/docs/specs/.../team-tournament-template-spec.md`,
   with literally `...` in place of the status folder.

### 0.5 Read these before coding

In this order:

1. `_templates/CLAUDE.md`, the instantiation runbook (§2's 14 steps and §8).
2. `_templates/standard-tournament-template/index.html` and
   `schedule.html`, the shells you start from.
3. `lib/v1/apps/hub.js` and `lib/v1/apps/schedule-board.js`, which you
   branch.
4. `lib/v1/views/teams.js`, `lib/v1/domain/teams.js` and
   `lib/v1/domain/schedule-grid.js`.
5. `tools/control-center.html`: search for `createTeams(`, `drawLiveMatches`,
   `isTeamKey` and `pairLabel(`. That is how Control Center wires the team
   views, and the Hub copies its wiring.
6. `events/pickledrive-anniversary-2026/index.html` and `schedule.html`,
   the behaviour reference.
7. `_tests/engine/cases.mjs`, `_tests/engine/compare.test.mjs`,
   `_tests/engine/accepted.mjs` and `_tests/helpers/fixtures.mjs`, the
   harness.

### 0.6 Parity with the other templates

| What a standard or dual-meet event gets | Where it comes from | A team event today | This spec |
| --- | --- | --- | --- |
| Hub, `index.html` | a template shell on `mountHub` | **None.** `mountHub` refuses a team event ("This event's type isn't set up for a Tournament Hub") | §3 |
| Board, `schedule.html` | a template shell on `mountScheduleBoard` | **None.** The board has no team branch; PickleDrive's is inline and frozen | §4 |
| Scorer page, `scorer.html` | `_templates/scorer/`, shared by every type | `scorer.js` has a team branch that has never run at an event (PickleDrive used `"console"`). It shows a seeded playoff side's raw code and has no harness case | §5.1 |
| Attendance desk page, `attendance.html` | `_templates/attendance/`, shared | Works: PickleDrive ran it | §5.2: a harness case |
| Dry-run runbook | `_templates/dry-run-checklist-template.md`, shared | PickleDrive's own copy, with team lines added by hand | §6.1 |
| Hub board QR panel | `_templates/hub-pubmat/render.mjs` reads the Hub's markup | Works if the team Hub keeps that markup | §6.2 |
| Runbook in `_templates/CLAUDE.md` | §1–§5 | "There is no template yet" | §12 |
| The workbook | generated from the Dual Meet or Standard Master | No master | Interim: a cleared copy of PickleDrive's workbook, for PickleDrive's shape only (§7). The master is [its own spec](team-tournament-master-spec.md) |
| Control Center | its type branches | serves every team event | one call site (§2.1) |

---

## 1. What exists

| Piece | Where | State |
| --- | --- | --- |
| The team rules | `lib/v1/domain/teams.js` | Pure functions with unit tests: `parseSideCode`, `sideOf`, `pairOf`, `stageOf`, `slotOf`, `isGroupTeamCode`, `baseTeamOf`, `teamNameOf`, `sideLabel` ("Seed 3 · TBD" for an unfilled slot), `groupOf`, `stageLabel`, `pairLabel`, `teamMatchupResult`, `buildTeamData`, `teamRankBracket`, rosters. `PAIRS` and `STAGES` are module constants |
| The team views | `lib/v1/views/teams.js`, `css/teams.css` | `createTeams(options)`: matchup cards, bracket tables and playoffs, roster cards (the Teams tab), the Live Matches row (`liveRow`), and the team Match Finder as a `delegate` of `createFinder`. **Only Control Center calls it** |
| The day model | `lib/v1/domain/model.js` | `buildDayModel` adds `model.team` = `{ rowByCode, matchups, advancing, roster, rosterByCode }` for a team event |
| The Hub app | `lib/v1/apps/hub.js` | Refuses a team event |
| The board app | `lib/v1/apps/schedule-board.js`, `lib/v1/domain/schedule-grid.js` | Standard and dual meet only. The type is inferred from the `clubOrder` setting. The category is a segment of the team code, and the stage comes from the code's tail (`ladderStageOf`). No team names, no `events.json` |
| The scorer app | `lib/v1/apps/scorer.js` | Has `entry.type === 'team'` branches. `sideName(code)` looks up `teamNames[side]`, built from the venue's `STANDINGSCSV` by exact `teamCode`. The dialog's sub line (`describe`) is `m.matchUp`. "Lineup not set" is a warning |
| The desk app | `lib/v1/apps/attendance-desk.js`, `domain/attendance.js` | Groups a team event's people by team |
| The prototype | `events/pickledrive-anniversary-2026/` | Inline code, pubmat theme, frozen |
| Fixtures, harness | `_tests/fixtures/config.json` (registry), `_tests/fixtures/pickledrive-anniversary-2026/pickledrive-anniversary-2026-day1.json` (the finished day, with quarterfinals) | Listed in `SNAPSHOTS` in `_tests/helpers/fixtures.mjs`. `deriveState` makes `pre` and `mid` from it |
| Fixtures, local | `_fixtures/config.json`, `_fixtures/pickledrive-anniversary-2026/{pre,finished,edge,qf-pre}.json` and `attendance-kingcourts-pre.csv` | For `?fixture=` |

PickleDrive's published day has 15 teams `A`–`O` in three brackets of five,
120 group matches (#1–120), quarterfinals `QF-3 v QF-6`, `QF-1 v QF-8`,
`QF-2 v QF-7` and `QF-4 v QF-5` (#121–136), semifinals `SF-1 v SF-2` and
`SF-3 v SF-4`, then `Br-1 v Br-2` and `Fi-1 v Fi-2`. That is 152 matches in
all, four pairs per matchup, one facility (Kingcourts) and 10 courts.

Already derived from the data, so nothing in the template sets them:

- the groups (`groupOf` reads `bracket`);
- who advances (`buildTeamData`'s `advancing`: any base team in a playoff
  slot);
- facilities and courts (`events.json` and the matches).

An unknown stage prefix shows its raw code. **The pair labels are not yet
derived:** §2.1 fixes that.

---

## 2. Engine changes shared by the pages

### 2.1 Pair labels per event (`display.pairs`)

The pair layout belongs to the event, so it moves into the registry. The
Hub, the board, the scorer and Control Center all read the registry, so
nothing is kept in step by hand.

**`events.json` shape** (documented in `event-data/config/README.md`, §12).
A team event may carry `display.pairs`:

```json
"display": {
  "pairs": {
    "1": { "full": "Men's Doubles", "short": "MD" },
    "2": { "full": "Women's Doubles", "short": "WD" },
    "3": { "full": "Mixed Doubles", "short": "XD" },
    "4": { "full": "Mixed Doubles", "short": "XD" }
  }
}
```

**`lib/v1/data/registry.js`, `eventConfigFrom`:**

- Import `numberRepeatedPairs` and `PAIRS` from `../domain/teams.js`.
- Add a `pairs` property to the returned object:
  - `numberRepeatedPairs(raw.display.pairs)` when `raw.display.pairs` is an
    object whose every key matches `/^\d+$/` and every value has
    non-empty string `full` and `short`;
  - when it is present but fails that check, `PAIRS`, plus one
    `console.warn('events.json: display.pairs for <eventKey> is malformed; using the default pair labels')`;
  - otherwise `PAIRS`.
- Add `@property {Record<string,{full:string,short:string}>} pairs` to the
  `EventConfig` typedef in `domain/model.js`.

**`lib/v1/domain/teams.js`:**

- Change `pairLabel(code, short)` to `pairLabel(code, short, pairs = PAIRS)`.
- An unknown pair number (one not in `pairs`, or no pair number) now
  returns `` `Pair ${n}` `` when the code has a pair number `n`, and `''`
  when it has none. Today it returns `''` for both.

**Callers.** Pass the event's pairs everywhere:

| File | Where | Becomes |
| --- | --- | --- |
| `lib/v1/views/teams.js` | `matchupRowHTML`: `pairLabel(m.t1, true)` | `pairLabel(m.t1, true, model.event.pairs)` |
| `lib/v1/views/teams.js` | `liveRowHTML`: `pairLabel(m.t1, false)` | `pairLabel(m.t1, false, model.event.pairs)` |
| `tools/control-center.html` | the dialog `describe`: `pairLabel(m.t1, true)` (search `[m.matchUp, pairLabel(`) | `pairLabel(m.t1, true, MODEL.event.pairs)`, in push 2 |
| `lib/v1/domain/schedule-grid.js`, `lib/v1/apps/scorer.js` | new code (§4, §5) | the event's `pairs` |

A view must tolerate an `EventConfig` without `pairs` (a cached older
`registry.js`): write `model.event.pairs || undefined`, so the default
parameter applies.

PickleDrive has no `display.pairs`, so Control Center keeps showing it as
MD, WD, XD 1, XD 2. `STAGES` stays a constant: the stage prefixes are part
of the code format.

### 2.2 Options in `lib/v1/views/teams.js`

`createTeams(options)` gains three things. Each default is what Control
Center gets today, so Control Center passes nothing new.

**`options.teamLetters`** (default `true`). With `false`, no organiser's team
letter appears anywhere in the team views. That covers these seven places,
all of which draw one today:

| # | Function | Today |
| --- | --- | --- |
| 1 | `matchupCardHTML` | `sideChipHTML(model, side)` after each team name |
| 2 | `groupTableHTML` | `<span class="chip">${teamCode}</span>` after the team name |
| 3 | `liveRowHTML` | `sideChipHTML(model, side)` after each team name |
| 4 | `teamResultHTML` | `<span class="chip">${base}</span>` in the `<h2>` |
| 5 | `autocompleteHTML` | `<span class="chip">${e.base}</span>` in a team suggestion |
| 6 | `introHTML` | `<span class="chip">${t.base}</span>` in each team chip button |
| 7 | `rosterCardHTML` | the meta line `Team ${teamCode} · N players`. With `false` it reads `N players`, as on the prototype |

How the option reaches them:

- `createTeams` builds `const hooks = { decorateRow: options.decorateRow, teamLetters: options.teamLetters !== false }`.
- Functions that already take `hooks` read `hooks.teamLetters !== false`.
  Treat a missing `hooks.teamLetters` as `true`, so a direct call from a
  unit test is unchanged.
- Functions that don't take it gain a **trailing optional** parameter:
  - `groupTableHTML(model, group, rows, hooks = {})`, passed by
    `teamStandingsHTML`;
  - `liveRowHTML(model, court, m, hooks = {})`, passed by `createTeams`'s
    `liveRow`;
  - `autocompleteHTML(matches, hooks = {})`, passed by the delegate's
    `renderAutocomplete`;
  - `rosterCardHTML(model, teamCode, { …, teamLetters = true })`;
  - `rostersHTML(model, openSet, { expandAllClass, teamLetters = true })`,
    passed by `renderRosters` and passed on to `rosterCardHTML`;
  - also the two `rosterCardHTML` calls inside `teamResultHTML` and
    `playerResultHTML`.
- The no-name fallback `Team ${teamCode}` in `rosterCardHTML` (a team whose
  workbook row has no name) stays: without a name the letter is the only
  label.

This is a presentation option like the standings view's `rrBracketLayout`
(site engine §12 rows 5 and 13), not a page flag. The guard test forbids
`isHub`, `isConsole` and `mode: 'hub'`, and nothing here uses them. The
public pages hide the letters because the
[Tournament Hub](../../features/tournament-hub.md#team-events) page says
"The letter codes the organizer uses in the workbook (`A`, `B`…) are not
shown here; Control Center still shows them".

**`options.searchHint`** (default
`'Search a team, a player or a match number above.'`, the string
`createTeams` hard-codes today as `hint`). The Hub passes
`'Search a team or a player above.'`: it has no match-number search.

**Player tags** (decided: on both pages, §13 Q1). A player's level and
gender show beside their name in a player result, as on the prototype:

- `buildTeamIndex`: when a roster row first sets `entry.rosterTeam`, also
  set `entry.level = r.level` and `entry.gender = r.gender`.
- `playerResultHTML`: the heading becomes
  `<span class="player-head"><h2>${name}</h2>${tags}</span>`.
  - `tags` is
    `<span class="roster-tags">` + one `<span class="roster-tag">` per
    non-empty value of `[entry.level ? teamLevelLabel(entry.level) : '', entry.gender]`
    + `</span>`;
  - it is `''` when both are empty.
- `css/teams.css` adds
  `.results-head .player-head{ display:flex; align-items:center; gap:8px; flex-wrap:wrap; }`.
  `.roster-tags` and `.roster-tag` already exist there.

Finally, `css/teams.css`'s header comment "Only Control Center has a team
event" becomes "Control Center and a team event's Hub".

---

## 3. The Hub

### 3.1 `_templates/team-tournament-template/index.html`

Copy `_templates/standard-tournament-template/index.html` and change only
what this list names:

1. **`<head>`.** Add `<link rel="stylesheet" href="/lib/v1/css/teams.css">`
   after `standings.css`.
   - Meta description and `og:description`: "Find your team's matchups,
     live scores, and standings."
   - Keep `<title>{{EVENT_TITLE}} — Tournament Hub</title>` exactly:
     `render.mjs` strips that suffix.
2. **`THEME` banner and `:root`:** unchanged, byte for byte.
3. **The own-touches block** (`/* ==== THIS TEMPLATE'S OWN TOUCHES … */`):
   - delete the §12 rows 5–7 and 13 rules (`.standings-board`,
     `.standings-col`, `.br-table …`, `.rr-bracket-label …`,
     `.standings-board-desktop …`), which style category standings a
     team event doesn't have;
   - keep the two §12 row 12 Live Matches phone rules;
   - add, under a comment
     `/* four tabs fit a 375px phone (the prototype's rule) */`:
     `@media (max-width:480px){ .view-tabs{gap:6px;} .view-tab{font-size:11px; padding:9px 12px;} }`.
4. **Hero:**
   - keep the QR panel, logos, tagline, `.eyebrow`, title and
     `.domination-line`, exactly as in the standard template;
   - subtitle: "Follow every team, every matchup and every court — live.".
5. **Tabs:** after the Standings button, add
   `<button class="view-tab" data-view="teams" type="button" hidden style="display:none;">Teams</button>`.
6. **Search panel:**
   - label `Search a team or a player`;
   - placeholder `e.g. a team name or a player's name`.
7. **Standings panel:** delete `#categoryToggleBar` and the whole
   `#categoryFilterWrap` block. Keep `#standingsResults` and
   `#standingsBody`.
8. After `#standingsResults`, add
   `<div id="teamsResults" style="display:none;"><div id="teamsBody"></div></div>`.
9. **Footer and the settings script:** unchanged.
   - `mountHub({ eventKey: EVENT_KEY, liveBaseUrl: LIVE_BASE_URL })`.
   - `LIVE_BASE_URL = 'wss://sage-live.sagematchcontrol.workers.dev'`, a
     literal and not a token.

The tokens are the standard set: `EVENT_KEY`, `EVENT_TITLE`,
`EVENT_TAGLINE`, `EVENT_HEADLINE`, `EVENT_DATE_RANGE`, `VENUE`,
`EVENT_LOGO`, `QR_IMAGE`, `QR_URL`. Expect about 210 lines.

### 3.2 The team branch in `lib/v1/apps/hub.js`

`mountHub`'s `KNOWN_SETTINGS` don't change.

1. **Type check:** accept `'team'` too. The refusal message stays for any
   other type.
2. `const teamEvent = EVENT_TYPE === 'team';`
3. **Teams view.** Before `createFinder`, when `teamEvent`, build:

   ```js
   teams = createTeams({
     getModel: () => MODEL, getFinder: () => finder,
     input, acList: byId('acList'), resultsEl,
     standingsEl: standingsBodyEl, rostersEl: byId('teamsBody'),
     saveSearch: saveSearchState,
     renderStandings: () => teams.renderStandings(),
     searchHint: 'Search a team or a player above.',
     teamLetters: false,
   });
   ```

   Import `createTeams` from `../views/teams.js`.
4. **Finder:** pass `delegate: teams ? teams.delegate : null` to
   `createFinder`.
5. **Standings:**
   - construct `createStandings` only when not `teamEvent`;
   - define `renderStandings()` as `teamEvent ? teams.renderStandings() : standings.render()`;
   - replace every `standings.render()` and `standings.reset()` call with
     `renderStandings()` and a matching guarded reset.
6. **Live Matches:** for a team event, `renderLiveMatches` passes
   `liveRow: teams.liveRow` and
   `emptyMatchupHTML: '<div class="live-matchup">&nbsp;</div>'`, and no
   `teamLogoHTML`.
7. **`showView(view)`:**
   - toggle `#teamsResults` with `view === 'teams'`;
   - call `teams.renderRosters()` when `view === 'teams'`;
   - guard the lookup with `byId('teamsResults')` being present, so a
     standard shell without it is unaffected.
8. **`loadLiveData`:** after `MODEL = model`, when `teamEvent`, call
   `teams.syncTab()`. If the Teams tab is the active one, call
   `teams.renderRosters()`.
   - The tab shows whenever the snapshot carries a roster, **before
     go-live too**.
   - Standings and Live Matches stay gated on `dayIsLive`, as now.
9. **`selectDay`:** hide `#teamsResults` with the other panels, and call
   `teams.reset()` beside `finder.reset()`.
10. **`revealLiveTabsAfterLoad`:** unchanged. It must not touch the Teams
    tab's visibility; `syncTab` owns that. Its saved-view rule already lands
    on `'teams'` only when that tab is showing.
11. **Restoring a saved search on the first load**, for a team event. This
    is what Control Center's `selectDay` does (search `isTeamKey` in
    `tools/control-center.html`):
    - a saved value matching `/^(team|player):/` goes through
      `teams.entryForSearchKey(saved)`;
    - if found, set `finder.selection = entry` and call
      `finder.setQuery(entry.kind === 'team' ? entry.label : entry.name)`;
    - otherwise drop it.
    - For a standard or dual-meet event, a saved `team:`/`player:` value
      is dropped too.

A standard or dual-meet event must behave exactly as before. Their harness
cases prove it (§10.2).

### 3.3 What it shows

These are the prototype's views, all in `views/teams.js`:

- **Day picker, sync status and go-live gating:** as on the other Hubs.
- **Four tabs.** The Teams tab shows whenever the event has a roster,
  before go-live too.
- **Match Finder:**
  - before a search: the stats row, team chips by bracket, then every
    matchup by its lowest match number;
  - a team result: its roster card, then its matchups with **Next up**;
  - a player result: the player's tags, roster card, and each of their
    matches on its matchup card.
- **Standings:** bracket tables with **Advances** marks, the playoff
  rounds, and each bracket's matchups behind a toggle.
- **Live Matches:** two lines per court, the match and the running matchup
  total.
- **Teams:** roster cards by bracket, with Expand all.

---

## 4. The schedule board

### 4.1 `_templates/team-tournament-template/schedule.html`

Copy `_templates/standard-tournament-template/schedule.html` and change only
the settings script:

- The comment above `CAT_META` describes the team colour key:
  - one entry per bracket, keyed `G<n>`, and one `PO` for every playoff
    match;
  - a team workbook's `SCHEDULE` tab is uncoloured, so we choose these
    colours;
  - keep each hue dark enough that navy text stays readable on its 40%
    tint;
  - marked `// EXAMPLE — replace`.
- `CAT_META` (the example; §13 Q4):

  ```js
  const CAT_META = {
    G1: { short:'BR 1',     color:'#1155CC' },
    G2: { short:'BR 2',     color:'#B45F06' },
    G3: { short:'BR 3',     color:'#741B47' },
    G4: { short:'BR 4',     color:'#38761D' },
    PO: { short:'PLAYOFFS', color:'#14263C' }
  };
  ```

  The first three are hues the standard template already ships; `G4` is a
  green and `PO` the house navy. They are examples: each event replaces
  them (`_templates/CLAUDE.md` step 6).
- The call: `mountScheduleBoard({ eventKey: EVENT_KEY, dayKey: DAY_KEY, liveBaseUrl: LIVE_BASE_URL, categoryColors: CAT_META, type: 'team' })`.

The tokens are the board's existing set: `{{EVENT_KEY}}`, `{{EVENT_TITLE}}`
and `{{SCHEDULE_DAY_KEY}}`. The `THEME` block is unchanged.

### 4.2 `lib/v1/domain/schedule-grid.js`

Add the team rows; the existing ones don't change.

- **`parseFacilityCsv(text, type, team = null)`.**
  - For `type === 'team'`, `team` is `{ rowByCode, pairs }`.
  - Read the `matchUp` column too (`idx('matchUp')`).
  - Each row is the usual row plus:
    - `matchUp`;
    - `team1: sideLabel(sideOf(c1), rowByCode)` and
      `team2: sideLabel(sideOf(c2), rowByCode)`;
    - `pair: pairLabel(c1, true, pairs)`;
    - `stage: ladderStageForTeam(c1)`;
    - `cat`, computed instead of a code segment: `'PO'` when
      `stageOf(sideOf(c1)) !== null`, else `` `G${groupOf(sideOf(c1), rowByCode)}` ``
      when that group isn't null, else `''`.
  - Leave the player fields as they are parsed now. The view decides
    "Lineup TBD" (§4.3).
- **`ladderStageForTeam(code)`**, a new export. It maps the side's prefix
  (`stageOf(sideOf(code))`): `'QF'` → `'QF'`, `'SF'` → `'SF'`,
  `'Br'` → `'B'`, `'Fi'` → `'F'`, `null` → `'RR'`, anything else → `'RR'`.
- **`buildScheduleData(snapshot, venueName, type, { pairs } = {})`.**
  - For `'team'`, first build `rowByCode` from every facility's
    `standingsCsv`: `new Map(rowsToStandings(parseCSV(f.standingsCsv || '')).map(s => [s.teamCode, s]))`
    over all facilities.
  - Pass `{ rowByCode, pairs: pairs || PAIRS }` to each `parseFacilityCsv`.
- **Imports:** `sideOf`, `stageOf`, `groupOf`, `sideLabel`, `pairLabel` and
  `PAIRS` from `./teams.js`, and `rowsToStandings` from `./standings.js`.
- Series "not needed" marking (`markUnneededGames`) runs as now; a team day
  has no series.
- Update the file's header comment: board rows for a team day also carry
  `matchUp`, `team1`, `team2`, `pair` and `stage`.

### 4.3 `lib/v1/apps/schedule-board.js`

- **Settings:** `KNOWN_SETTINGS` gains `'type'`. Document it in the header
  comment: `'standard' | 'dual-meet' | 'team'`, default inferred from
  `clubOrder` as today.
  - Any other value: `console.error('mountScheduleBoard: unknown type "<value>" (known: standard, dual-meet, team)')`,
    then return.
- `const EVENT_TYPE = type || (clubOrder ? 'dual-meet' : 'standard');`
- **Pair labels**, team only. Before the first `loadSchedule(true)`, read
  them:

  ```js
  try { pairs = eventConfigFrom(await loadRegistry({ fixture: FIXTURE }), eventKey)?.pairs; }
  catch(err){ console.warn(`mountScheduleBoard: couldn't read events.json (${err.message}); using the default pair labels`); }
  ```

  - `mountScheduleBoard` becomes `async`, which is compatible: callers
    ignore its return value.
  - The other types never fetch the registry.
  - Pass `{ pairs }` to `buildScheduleData`.
- **`cellHTML(m)`**, when `EVENT_TYPE === 'team'`:
  - the stage pill reads `STAGE_META[m.stage]`;
  - after the stage pill, add `<span class="cell-pair">${esc(m.pair)}</span>`
    when `m.pair` is set;
  - each side's `.fo-names` holds `<span class="fo-team">${esc(teamN)}</span>`,
    then the two players, or `<span class="fo-name unknown">Lineup TBD</span>`
    when the lineup isn't set. Not set means the first player is `null` or
    equals the side's own code; `domain/matches.js` uses the same rule;
  - no club tag.

  Every other cell is unchanged.
- **`meta(cat)` fallback:** for a `cat` matching `/^G(\d+)$/` that
  `categoryColors` lacks, return `{ short: 'BR ' + n, color: FALLBACK.color }`.

### 4.4 `lib/v1/css/schedule-board.css`

Add, beside the matching existing rules and in the board's fonts (Barlow
Condensed for labels, as `.cell-stage` uses):

- `.cell-pair`: the same box as `.cell-stage`, and
  `.cell-cat, .cell-stage, .cell-pair{ white-space:nowrap; }`;
- `.fo-team{ font-weight:700; font-size:12px; line-height:1.3; color:var(--navy); max-width:100%; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }`;
- inside `@media screen and (max-width: 720px)` (where `.faceoff` stacks):
  `.fo-team{ font-size:11.5px; }`;
- inside the first `@media print`: `.print-grid .cell-pair{ font-size:5.5pt; }`
  and `.print-grid .fo-team{ font-size:6.5pt; line-height:1.15; }`.

These sizes are the prototype's (`events/pickledrive-anniversary-2026/schedule.html`,
lines 239–270, 437 and 488–489), set in the board's own fonts. Use only the
board's contract properties: the `THEME CONTRACT` comment at the top of the
file, which `theme-contract.test.mjs` reads.

---

## 5. The scorer and desk pages on a team event

### 5.1 Scorer (`lib/v1/apps/scorer.js`)

The template `_templates/scorer/scorer.html` is shared and does not change.
In `scorer.js`'s team branches:

- **Names.** Replace `teamNames` with `rowByCode`, built in `applySnapshot`
  from the chosen venue's `standingsCsv`:
  `new Map(rowsToStandings(parseCSV(f.standingsCsv || '')).map(s => [s.teamCode, s]))`.
  - `sideName(code)` returns `sideLabel(sideOf(code), rowByCode)` for a
    team event.
  - So a filled playoff side (`SF-A_2`) shows its team's name, and an
    unfilled one shows "Seed 1 · TBD".
- **Pair labels.** At start, keep `eventConfigFrom(registry, EVENT_KEY)`
  beside `entry` and read its `pairs`.
- **The dialog's sub line** (`describe`), for a team event:
  `[stageLabel(sideOf(m.t1), rowByCode), pairLabel(m.t1, false, pairs)].filter(Boolean).join(' · ')`.
  For example, "Bracket 2 · Mixed Doubles 1" or "Semifinal · Men's Doubles"
  (§13 Q5).
- **Unchanged:**
  - search (it matches whatever `sideName` returns);
  - the `.sc-chip` showing the raw code under a name (scorers are staff);
  - "Lineup not set";
  - the score dialog, conflicts, expiry and court chips.

Control Center's own score dialog keeps its `matchUp · pair` sub line. Only
the scorer page changes.

### 5.2 Attendance desk

No change. A team event's desk groups people by team already. It gains a
harness case on team data (§10.2).

---

## 6. The runbook and the hub board

### 6.1 `_templates/dry-run-checklist-template.md`

Add PickleDrive's team checks, marked "(team events)", worded by role
rather than by cell (§7 gives the cells of PickleDrive's workbook):

- In the rehearsal section, after the existing first edit check:

  > - [ ] **Lineup and playoff slots (team events).** Enter one matchup's
  >   lineup on `MatchUps`. Within about 30 seconds the site shows those
  >   players' names in place of *Lineup not set*; if not, check `MatchUps`
  >   is ticked in **SAGE → Set up live sync**. Then type a team letter into
  >   one quarterfinal seed cell on `MatchUps`. That quarterfinal card
  >   switches from *Seed n · TBD* to the team's name, and the team gets an
  >   *Advances* label. Put the seed number back afterwards.

- In the during-play section:

  > - [ ] **Playoff teams (team events).** When the bracket stage ends,
  >   enter each qualifier's letter against its seed on `MatchUps`; after
  >   each playoff round, enter the next round's teams the same way. The
  >   site takes every playoff team from these cells and never picks them
  >   itself. The seed cells are listed in `_templates/CLAUDE.md` (the
  >   team workbook step).

### 6.2 The hub board's QR panel

`node _templates/hub-pubmat/render.mjs <event-key>` reads four things from
the event's `index.html`:

- `<title>`;
- `<div class="eyebrow">`;
- the `<img … alt="QR code…">`;
- `<div class="qr-link-text">`.

§3.1 keeps all four in the standard form, so it works unchanged. Check it
once (§10.4).

---

## 7. The workbook, until the Team Tournament Master exists (§13 Q6)

The other two formats generate their workbook from a master. A team event
has none yet. The interim rule:

- **Only for an event of PickleDrive's shape**, make the workbook by copying
  PickleDrive's and clearing it. That shape is:
  - 15 teams `A`–`O` in three brackets of five;
  - four pairs per matchup;
  - a group-stage round robin;
  - eight quarterfinalists, then semifinals, Bronze and Final;
  - 152 matches, one facility and 10 courts.
- The team codes are generic (`A_3`, `QF-3_4`), so copying carries over no
  other event's names. Carried-over names are why the runbook forbids
  copying for the other formats.
- **Any other shape waits for the master.**

The implementer writes this as a new subsection of `_templates/CLAUDE.md`
§2 step 8 or just before it: "**Team events: the workbook**". Its content:

1. **Source:** the live PickleDrive workbook, Drive file
   `1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA`
   ("2026-10-03 PickleDrive Club One Year Celebration"). It carries the
   optimised calculation (the hidden `StackCache` tab;
   `sage-docs/docs/specs/.../team-workbook-stack-cache-spec.md`). **Never
   edit the source:** **File → Make a copy**, named
   `<date> <title> - <FACILITY>` like a generated workbook.
2. **Its tabs:**
   - **input:** `Title`, `Teams`, `MatchUps`, `SCHEDULE`, `Court Control`,
     `ATTENDANCE`, `Raffle`;
   - **computed:** `Standings`, `FINAL RANK`, `Awards`, `CSV`,
     `STANDINGSCSV`, `MatchLookup`, `StackCache`, `Variables`,
     `Variables V2`, `Reference for Players`, `Timeline`,
     `Timeline (Individual}`, `Pairings Guide`;
   - **`Brackets`** is a leftover from another event; ignore it.
3. **Clear the inputs. Clear values only, and never a cell holding a
   formula:** turn on **View → Show → Formulas** first, and leave any cell
   that starts with `=`.
   - **`Teams`:** type the new event's team names and players over the old
     ones.
   - **`MatchUps`:**
     - clear every lineup;
     - type each playoff seed cell back to its seed number: quarterfinal
       seeds 3, 6, 1, 8, 2, 7, 4, 5 in `D604`, `D614`, `D624`, `D634`,
       `D644`, `D654`, `D664`, `D674`; semifinal seeds 1–4 in `D684`,
       `D694`, `D704`, `D714`; Bronze 1–2 in `D724`, `D734`; Final 1–2 in
       `D744`, `D754`.
   - **`SCHEDULE`:** clear every typed score. A score must be a truly empty
     cell (**Delete**, not a space), because the formulas test for an
     empty cell.
   - **`Court Control`:** clear every match number on a court.
   - **`ATTENDANCE`:** clear rows 2 down in `A:G`. **Update roster**
     refills them.
   - **`Title`:** the new event's title and date.
   - **`Raffle`:** clear it.
   - The scorers' **slot times** are `SCHEDULE!B6:B`: retype them if the
     new event starts at another time and they are typed values.
4. **Check before wiring it up:**
   - `STANDINGSCSV` lists the 15 new team names with zero points;
   - `CSV` has 152 rows with every score empty;
   - every quarterfinal and later row reads its seed (`QF-3_1` …).
5. **Then step 8 as usual.** The copy keeps the bound `sheets-sync.gs` but
   not its trigger. Run **SAGE → Set up live sync** with the new day key
   and facility; that creates the trigger and replaces the copied day key.
   Then share it with the service account (step 13).

Write the cell addresses exactly as above. They come from PickleDrive's
workbook through its runbook (`events/pickledrive-anniversary-2026/dry-run-checklist.md`),
which is the reference (see the status note). The Team Tournament Master
starts from the same workbook, so when it exists its spec replaces this
subsection with "generate it from the master", and the site needs no change.

---

## 8. The order of pushes

The versioning rule (§0.4 rule 1) needs two merges into `main`:

1. **Merge 1, the engine below the apps.** No page uses anything new yet:
   - `data/registry.js` (`pairs`) and `domain/model.js` (the typedef);
   - `domain/teams.js` (`pairLabel`, `Pair <n>`);
   - `domain/schedule-grid.js` (team rows, `ladderStageForTeam`);
   - `views/teams.js` (`teamLetters`, `searchHint`, player tags, pairs
     passed through);
   - `css/teams.css` and `css/schedule-board.css`;
   - their unit tests, the demo fixtures (§10.2), and the harness changes
     that need no new page.
2. **Wait at least 10 minutes after merge 1 is live.** Then **merge 2, the
   apps and pages:**
   - `apps/hub.js`, `apps/schedule-board.js` and `apps/scorer.js`;
   - the two template pages;
   - Control Center's `pairLabel` call;
   - the dry-run template and `_templates/CLAUDE.md`;
   - the harness cases for the new pages.

Run `npm run verify` before each merge.

---

## 9. Known differences

**From the prototype**, which the one-time check in §10.3 shows:

| # | Page | Difference | Prototype | Template |
| --- | --- | --- | --- | --- |
| 1 | both | Theme and fonts | pubmat: Montserrat, Playfair Display, Satisfy, its own palette | the S.A.G.E. house theme |
| 2 | both | Mixed doubles label | `MXD` | `XD`; an event that wants `MXD` sets `display.pairs` |
| 3 | Hub | Hero | PickleDrive's own wording and title kicker | the standard hero with tokens |
| 4 | Hub | Intro hint | "Search a team or a player above." | the same |
| 5 | Hub | The Live Matches team cell's markup | name and players directly in the cell | Control Center's wrapper (`.live-team-cell-inner` > `.live-team-text`) (§13 Q3) |
| 6 | board | Colour key hues | olive, sky, gold, forest (the pubmat's) | the §4.1 examples (§13 Q4) |
| 7 | board | Fallback chip for an unlisted bracket | `?` | `BR <n>` |

**From the baseline in existing harness cases**, each needing an
`ACCEPTED` entry (§10.2):

| # | Cases | Difference | Why |
| --- | --- | --- | --- |
| 8 | `cc/pickledrive-anniversary-2026/finder-search/*` | A player result gains the level and gender tags | §13 Q1 |
| 9 | every `cc/team-demo-2026/*` | The baseline reads the demo's three pairs with the old constant (MD, WD, "XD 1"); the branch reads `display.pairs` (MD, WD, XD). The baseline has no player tags | §2.1, §13 Q1 |
| 10 | every `scorer/team-demo-2026/*` | Playoff side names and the dialog's sub line | §5.1, §13 Q5 |

**Any other difference from PickleDrive's pages is a bug in the template or
the engine change.** Fix it to match PickleDrive's page: its text, which
views show what, and its layout where the shared CSS lacks a rule.
Exceptions:

- a fix would change Control Center or a standard or dual-meet page. Then
  put the PickleDrive behaviour behind an option whose default keeps the
  existing page, as `teamLetters` does;
- the difference is one of rows 1–7.

Record each fix in §17. None of this needs the owner.

Any other difference in an **existing** harness case (one that isn't rows
8–10) means the change broke something. Fix the change, not the case.

---

## 10. Tests

### 10.1 Unit (`_tests/unit/`)

Add cases to the existing files; write each test first and watch it fail.

- `data/data-modules.test.mjs`:
  - `eventConfigFrom` gives `pairs` from `display.pairs`, numbered for
    repeats (two XD → "XD 1", "XD 2");
  - without the map, `pairs` is `PAIRS`;
  - a malformed map (key `"x"`, or an entry without `short`) gives `PAIRS`
    and warns once.
- `domain/teams-awards-attendance-grid-model.test.mjs`:
  - `pairLabel` with and without `pairs`;
  - a three-pair layout;
  - `pairLabel('A_7', true)` is `'Pair 7'` and `pairLabel('A', true)` is
    `''`;
  - `ladderStageForTeam` for `A_1`, `QF-3_1`, `SF-A_2`, `Br-1_1`, `Fi-J_4`
    and `ZZ-1_1`;
  - `buildScheduleData(snapshot, null, 'team', { pairs })` on the demo
    fixture: `team1`/`team2` (a seed reads "Seed 4 · TBD"), `pair`,
    `stage`, and `cat` (`G1`, `G2`, `PO`);
  - the standard and dual-meet results equal what they are today.
- `views/teams.test.mjs`:
  - with default options, all seven places of §2.2 draw a letter;
  - with `teamLetters: false`, none do: no `class="chip"` in the card,
    bracket table, live row, team result, autocomplete or intro, and the
    roster card meta has no `Team A ·`;
  - `searchHint` reaches the intro;
  - a player result carries the tags;
  - pair labels follow `model.event.pairs`.
- `apps/settings.test.mjs`:
  - `mountScheduleBoard({ eventKey: 'e', dayKey: 'd', type: 'nope' })`
    logs exactly one error naming `type`;
  - `'type'` is a known setting;
  - the existing unknown-setting cases still pass.
- `engine/fixture-mode.test.mjs`: on localhost with `?fixture=finished`, an
  instantiated team Hub and board load `team-demo-2026` with no console
  error. Follow the file's existing pattern.
- The guards stay green: `guards.test.mjs` (dependency direction, no
  operator words, no page flags), `no-copies.test.mjs`,
  `theme-contract.test.mjs` and `live-pages.test.mjs`.

### 10.2 The harness

**A demo event of another shape, `team-demo-2026`** (merge 1). It is
fixture-only, so never add it to `event-data`.

| | Value |
| --- | --- |
| Title | `Team Demo (fixture only)` |
| Type, settings | `"team"`, `"scoreEntry": "links"`, `"attendance": "desks"` |
| Day | `team-demo-2026-day1`, label `Oct 10`, date `2026-10-10`, `isLive: "auto"` |
| Facility | `Demo Courts`. `sheetId` `test` in `_tests/fixtures/config.json` (Control Center's routed CSV); `FIXTURETEAMSHEETID000000000000000` in `_fixtures/config.json` (the desk case) |
| `display.pairs` | `1` MD Men's Doubles, `2` WD Women's Doubles, `3` XD Mixed Doubles |
| Teams | Bracket 1: `A` Aces, `B` Bandits, `C` Comets, `D` Dynamos. Bracket 2: `E` Eagles, `F` Falcons, `G` Giants, `H` Hawks |
| Group matchups | Per bracket, in this order: `A v B`, `C v D`, `A v C`, `B v D`, `A v D`, `B v C` (and the same with `E`–`H`). Bracket 1's six, then bracket 2's six: 12 matchups × 3 pairs = matches #1–36, pairs in order |
| Playoffs | `SF-A v SF-4` (#37–39: seed 1 already filled with `A`), `SF-2 v SF-3` (#40–42), `Br-1 v Br-2` (#43–45), `Fi-1 v Fi-2` (#46–48) |
| Times and courts | 4 courts. Match *n* is on `Court ((n−1) mod 4)+1` at `9:00 AM` + 20 min × ⌊(n−1)/4⌋. `court` (live) is empty |
| Players | Each team has 6 rostered players, 3 `M` and 3 `F`, levels `3.0`/`3.5`. Names are `<First name> <Last name>`, the first names starting with the team's letter (e.g. Aces: `Alma Reyes`, `Ana Cruz`, …), so the first player's first word is 3+ letters. Lineups are filled for every group match, except match #11 (`B v D`, pair 2), where team 2's two player cells equal its code `D_2`. In the playoffs, `SF-A` has a lineup; seeds `SF-4`, `SF-2`, `SF-3`, `Br-*` and `Fi-*` have player cells equal to their own codes |
| Scores (the `final` file) | #1–36 and #37–42 scored with any plausible pickleball scores (11–n, n < 10); #43–48 empty |
| `standingsCsv` | Header `teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket`. One row per team `A`–`H` with its points from the scores, a quotient to 5 places, and its bracket. Then `SF-A,Aces,…`, `SF-4,,0,0,-,`, `SF-2,,0,0,-,`, `SF-3,,0,0,-,`, `Br-1,,0,0,-,`, `Br-2,,0,0,-,`, `Fi-1,,0,0,-,`, `Fi-2,,0,0,-,` |
| `rosterCsv` | Header `teamCode,player,level,gender`; 48 rows |
| `matchesCsv` | The header PickleDrive's fixture uses: `matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score` |

Write the snapshot with a small script in your scratchpad, not in the repo,
and commit only its output:

- `_tests/fixtures/team-demo-2026/team-demo-2026-day1.json` (the harness),
  and a byte-identical `_fixtures/team-demo-2026/finished.json` (fixture
  mode);
- `_fixtures/team-demo-2026/attendance-demo-courts-pre.csv`, in the format
  of `_fixtures/pickledrive-anniversary-2026/attendance-kingcourts-pre.csv`,
  listing the 48 players with their team letter in `teams`, all `FALSE`
  except two.

Register the event in three places:

- `_tests/fixtures/config.json` and `_fixtures/config.json` (with their
  sheet IDs above);
- `SNAPSHOTS` in `_tests/helpers/fixtures.mjs`:
  `'team-demo-2026': { day: 'team-demo-2026-day1', file: 'team-demo-2026/team-demo-2026-day1.json' }`.

Because Control Center's cases iterate `SNAPSHOTS`, it gets
`cc/team-demo-2026/*` cases on its own.

**New pages have no baseline.** The harness renders every case on the
baseline tree, where the new templates don't exist yet. Change
`compare.test.mjs` as follows:

- when the case's file (its `path` without the query, resolved under
  `baseline.dir`) does not exist, run the case on the branch tree only;
- fail it on any page error;
- otherwise write `branch.png` and `branch.txt` to `_tests/out/<case>/`
  for review, and pass;
- log `new page, no baseline: <case id>`.

After merge 2 the baseline has the files, and these become ordinary
regression cases with no further change.

**New cases** in `_tests/engine/cases.mjs`, following the existing
generators:

- **`team-index`** (merge 2): in `hubCases`, add
  `{ label: 'team-index', path: '/_templates/team-tournament-template/index.html', events: ['pickledrive-anniversary-2026', 'team-demo-2026'] }`.
  - Its views are its own list, not the standard Hub's:
    - `finder` (no steps);
    - `finder-team`: `click('.team-chip-btn')`;
    - `finder-player`: `type('#teamInput', word)`, then
      `click('#acList .ac-item:last-child')`;
    - `live`: `tab('Live Matches')`;
    - `standings`: `tab('Standings')`;
    - `standings-bracket-open`: `tab('Standings')`, then
      `click('[data-tm-group]')`;
    - `teams`: `tab('Teams')`;
    - `teams-expanded`: `tab('Teams')`, then
      `click('[data-roster-all]')`.
  - Wrap any step whose element may be missing in a state with
    `ifPresent`/`ifVisible`, as the existing cases do.
- **`team-schedule`** (merge 2): in `scheduleCases`, add
  `{ label: 'team-schedule', path: '/_templates/team-tournament-template/schedule.html', events: [both] }`,
  with views `default`, `compact`, `print` and `courts-1-2`.
- **Tokens and settings:** `tokensFor`/`eventSettings` read PickleDrive's
  `DAYS`, `FACILITIES`, `CAT_META` and `DAY_KEY` from its finished pages.
  - Add `'pickledrive-anniversary-2026': 'events/pickledrive-anniversary-2026'`
    to `FINISHED_PAGES`.
  - For `team-demo-2026`, which has no pages, give `eventSettings` a fixed
    entry: `DAY_KEY` `team-demo-2026-day1`, `CAT_META` the §4.1 example.
- **Scorer on team data** (merge 1): in `scorerCases`, run the four
  existing views for `team-demo-2026` as well, with the registry from
  `_tests/fixtures/config.json` (already `"links"`).
- **Desk on team data** (merge 1): in `attendanceCases`, run the three
  existing views for `team-demo-2026` with the site `_fixtures/config.json`
  registry. Its mark step clicks the first `input.att-switch`, and it
  stays identical to the baseline.
  - Generalise `attendanceBackend(event, files)` so each venue is
    `{ name, sheetIdPrefix, file }`, and build `external` from that list
    instead of the two hard-coded patterns.
  - The `attendance-demo-2026` cases must still match the baseline
    exactly.
- **`ACCEPTED` entries** (`_tests/engine/accepted.mjs`) for §9 rows 8–10,
  each `kind: 'both'`, with `row` set to the string `'team §17 row <n>'`.
  - Change the log line in `compare.test.mjs` from
    `` `§12 rows ${…}` `` to `` `rows ${…}` ``, so both kinds of row read
    well.
  - Add no other entry. Any other difference is fixed, not accepted
    (§9).

Everything else that exists today is identical to the baseline: every
`std-*`, `dm-*`, `cc*`, `scorer/piggleball-2026/*` and
`attendance/attendance-demo-2026/*` case.

### 10.3 One check against the prototype (before merge 2)

Compare the template with PickleDrive's own pages: text by machine, layout
by eye.

- Add an optional case field `baselinePath`. When set, the baseline tree
  serves that path and the branch serves `path`.
- Add temporary cases `proto-index/pickledrive-anniversary-2026/<view>/<state>/<vp>`
  and `proto-schedule/…`:
  - `baselinePath` `/events/pickledrive-anniversary-2026/index.html` (or
    `schedule.html`);
  - `path` the team template, instantiated with PickleDrive's tokens;
  - the `team-index` and `team-schedule` views;
  - `root: '.wrap'` for the Hub, so the hero (§9 row 3) is not compared,
    and `root: '#grid'` for the board.
- For these cases compare **text only**: pixels differ by design (§9 row
  1). Save both screenshots side by side, and compare their layout
  yourself: what sits where, what is shown or hidden, what wraps at
  375 px. Colours and fonts don't count.
- Every text difference must be a §9 row (rows 2 and 4–7 show up here).
  Fix anything else, and any layout difference beyond the theme, to match
  PickleDrive's page (§9).
- Record the outcome in §17. Then delete these cases and `baselinePath`
  handling in the same commit, as the site engine removed its check pages.

### 10.4 By hand

1. Instantiate both pages into `events/team-check-beta/`. The
   `**/*beta*` git-ignore rule keeps it out of commits. Use:
   - `EVENT_KEY` `team-demo-2026`;
   - `SCHEDULE_DAY_KEY` `team-demo-2026-day1`;
   - the other tokens any plain text;
   - `QR_IMAGE` `/assets/logo.png`.
2. With your static server, open `/events/team-check-beta/?fixture=finished`
   and `/events/team-check-beta/schedule?fixture=finished`:
   - no console error, at 375 px and at desktop width;
   - the four tabs fit on one row at 375 px;
   - no team letter anywhere on the Hub;
   - print the board to PDF once.
3. Repeat with `EVENT_KEY` `pickledrive-anniversary-2026` and
   `?fixture=pre`, `finished`, `edge` and `qf-pre`.
4. `grep -rn '{{' events/team-check-beta/` prints nothing.
5. `node _templates/hub-pubmat/render.mjs team-check-beta` writes a QR
   panel. It needs the `sage-tools-api/node_modules` puppeteer (see that
   folder's `README.md`).
6. Delete the folder.

---

## 11. Steps

Branch `team-template` in the site repo. One commit per step.

1. Unit tests for §2.1, then `registry.js`, `model.js`'s typedef and
   `teams.js`'s `pairLabel` (§2.1).
2. Unit tests for §2.2, then `views/teams.js` and `css/teams.css` (§2.2).
3. Unit tests for §4.2, then `domain/schedule-grid.js` and
   `css/schedule-board.css` (§4.2, §4.4).
4. The demo fixtures, `SNAPSHOTS`, the generalised desk backend, the
   scorer and desk cases on team data, the "no baseline" rule, and the
   `ACCEPTED` entries for §9 rows 8–10 (§10.2). Run `npm run verify`; only
   the accepted rows may differ.
5. **Merge 1** with the owner's go-ahead. Wait 10 minutes after it is live.
6. `apps/scorer.js` (§5.1) and Control Center's `pairLabel` call (§2.1).
7. `_templates/team-tournament-template/index.html` and `apps/hub.js`
   (§3).
8. `_templates/team-tournament-template/schedule.html` and
   `apps/schedule-board.js` (§4.1, §4.3).
9. The `team-index` and `team-schedule` cases (§10.2), and the settings
   and fixture-mode unit tests (§10.1).
10. The prototype check (§10.3). Fix every difference toward PickleDrive, fill
    §17, then delete its cases.
11. The by-hand checks (§10.4).
12. The dry-run template (§6.1) and `_templates/CLAUDE.md` (§7, §12).
13. `npm run verify`, then **merge 2** with the owner's go-ahead.
14. The docs in the other repos (§12), then move this spec to
    `implemented/` by the procedure in the [specs index](../README.md),
    with §17 filled in.

---

## 12. Runbook and documentation

**`_templates/CLAUDE.md`** (site repo, step 12):

- §1, Choosing a template: add "Named teams meeting in matchups (a team
  tournament) → `team-tournament-template/`".
- §2:
  - **step 1:** `team-tournament-template` is the third choice;
  - **step 4:** a team event's labels are `display.pairs`;
  - **step 6:** for a team event, `CAT_META` is one hue per bracket
    (`G<n>`) plus `PO`, chosen by us because the team `SCHEDULE` tab is
    uncoloured. Keep the 40% tint readable;
  - **step 7:** `type: "team"`, `display.pairs` (with the example of
    §2.1), and `rosterSheetName` on a facility whose roster tab isn't
    `Teams`;
  - **the workbook:** the "Team events: the workbook" subsection of §7.
  - Steps 10–14 apply as written; the team checks are in the shared
    dry-run template.
- §3, tokens: the team template uses the standard set for `index.html`
  and the board's set for `schedule.html`.
- §5, the team code format: `<SIDE>_<PAIR>` (`A_3`, `QF-3_4`, `SF-A_2`,
  `Fi-J_1`), seeds and letters (§0.2), and that the board reads a team
  match's stage from the side's prefix.
- §5.1: nothing new is kept in sync by hand. Pair labels live only in
  `events.json`, and the colour key only in the board.

**Workspace `CLAUDE.md`** (`D:\Personal\SAGE`):

- In the `events/pickledrive-anniversary-2026/` bullet, PickleDrive is "the
  first `"team"` event, hand-built before the team template existed",
  not "the prototype the team template will be extracted from".
- `_templates/`: "the three event-site templates (`dual-meet-template/`,
  `standard-tournament-template/`, `team-tournament-template/`)".
- The team-event rules bullet under "Things that must be kept in sync by
  hand": `lib/v1/domain/teams.js` is used by Control Center, the team Hub,
  board and scorer, and pair labels come from `display.pairs`.

**`event-data/config/README.md`:** in the `display` bullet, add `pairs`,
which is team events only:

- the shape;
- the key is the pair number in a team code;
- a type that repeats is numbered;
- without it, the labels are MD, WD, XD 1, XD 2.

**sage-docs:**

- `technical/adding-a-new-event.md`: the team choice points at the
  template and at the interim workbook rule;
- `technical/site-engine.md`: `mountHub` serves all three types,
  `views/teams.js` has two callers with `teamLetters` and `searchHint`, and
  the board takes `type`;
- `technical/schedule-board.md`: the team cell (team name, pair chip,
  Lineup TBD) and the colour key;
- `technical/scorer-page.md`: team names and the sub line;
- `technical/event-data-config.md`: `display.pairs`;
- `features/tournament-hub.md`, Team events: a team event's page is a Hub
  like the others; the pair labels are the event's; a player result shows
  the player's level and gender;
- `features/control-center.md`: a player result shows the player's level
  and gender;
- `features/scorer-page.md`: what a team match shows;
- `specs/README.md`, `specs/not-started/README.md` →
  `specs/implemented/README.md`, and `mkdocs.yml`, by the move procedure.

---

## 13. Decisions

All settled by the owner on 2026-10-06. Don't revisit them.

| # | Question | Decided |
| --- | --- | --- |
| Q1 | Player level and gender tags in a player result? | **On both pages**, the Hub and Control Center (§2.2) |
| Q2 | An unknown pair number's label? | **`Pair <n>`**. A blank label hides a workbook mistake (§2.1) |
| Q3 | Control Center's Live Matches cell markup on the Hub? | **Yes.** One markup for both pages. It must still lay out as PickleDrive's does (§10.3) |
| Q4 | The board's colour key? | **The prototype's scheme:** one hue per bracket plus one for playoffs, in the §4.1 example hues |
| Q5 | The scorer's sub line? | **`<stage> · <pair>`**, not the `matchUp` key (§5.1) |
| Q6 | The workbook before the Team Tournament Master exists? | **A cleared copy of PickleDrive's workbook, for an event of PickleDrive's shape only**; any other shape waits for the master (§7) |

Also settled, in the first version: the template uses the S.A.G.E. house
theme, and the pubmat theme stays PickleDrive's.

---

## 14. Blocked on

Nothing. The engine is merged, and the team views and rules exist. The
pages read `CSV`, `STANDINGSCSV` and the roster tab in the format
PickleDrive's workbook publishes, so they need neither the Team
Tournament Master nor the calculator's team format.

The master starts from that same workbook, so it publishes the same tabs in
the same shape. When it arrives, the pages built here need no change. If the
master ever changes a published column, that is the master's spec's
concern, and the change is made in the engine like any other.

---

## 15. Out of scope

- The Team Tournament Master and the
  [calculator's team format](calculator-team-format-spec.md). §7 is the
  interim.
- Control Center beyond the `pairLabel` call and the player tags. Its own
  score dialog keeps `matchUp · pair`.
- PickleDrive's own pages, which are frozen.
- Team logos, and scoresheets for team codes.
- Any change to `sage-tools-api` or to `event-data/config/events.json`.

---

## 16. Acceptance checklist

**Implementer:**

- [ ] Merge 1:
  - `display.pairs` reaches `EventConfig.pairs`;
  - `pairLabel` takes `pairs` and returns `Pair <n>`;
  - `createTeams` takes `teamLetters` and `searchHint`, and player tags
    show;
  - `buildScheduleData` handles a team day;
  - the demo fixtures and the scorer and desk cases on team data are in;
  - `npm run verify` passes with only §9 rows 8–10 accepted.
- [ ] Merge 2:
  - `mountHub` serves a team event;
  - `mountScheduleBoard` takes `type: 'team'`;
  - the scorer names playoff sides and shows `<stage> · <pair>`;
  - Control Center passes its event's pairs.
- [ ] `_templates/team-tournament-template/` holds `index.html` and
  `schedule.html`, whose `THEME` blocks are byte-identical to the standard
  template's.
- [ ] The `team-index` and `team-schedule` cases run over both events, both
  widths and three states, with no page error.
- [ ] Every case that existed before is identical to the baseline, apart
  from §9 rows 8–10.
- [ ] The prototype check is done, its results recorded in §17, and its
  cases deleted.
- [ ] The by-hand checks pass, including the QR panel render.
- [ ] The dry-run template, `_templates/CLAUDE.md`, the workspace
  `CLAUDE.md`, `event-data/config/README.md` and the sage-docs pages of
  §12 are updated.

Nothing on this list waits for the owner, apart from the go-ahead for each
merge (§0.4 rule 6).

---

## 17. Reconciliations and divergences

The implementer fills this in:

- rows 8–10 of §9 as they land;
- one row per difference the prototype check (§10.3) finds;
- one row per place the built template departs from this spec.

| # | What | Pages | Kept | Proved by | Note |
| --- | --- | --- | --- | --- | --- |
