# Spec — Event site templates

Create two reusable event-site templates in the `sage-match-control.github.io`
repo, so a new tournament site is a copy-and-fill job rather than a
copy-an-old-event-and-hunt-for-hardcoded-strings job.

| Template | Based on | For |
| --- | --- | --- |
| `standard-tournament-template` | `events/archives/bkl-cup-2026/` | Multi-day, multi-venue, open-entry tournaments |
| `dual-meet-template` | bkl structure + `events/archives/ppa-x-club-2600-dual-meet.html` event model | Two clubs facing off, usually one day, one venue |

**The deliverable is the templates, not an event.** No real event is created by
this task. §9 is the runbook a later session follows to stand one up.

Everything below was derived by reading the three source pages
(`bkl-cup-2026/index.html` 2752 lines, `match-control.html` 3167,
`ppa-x-club-2600-dual-meet.html` 1590). Line references are to those archived
copies; script-relative line numbers are offsets within the `<script>` block.

**Settled up front** (§3): data comes from the Cloud Run snapshot pipeline, not
a direct browser fetch of Google Sheets; snapshots go to one generic data repo
with a folder per event; `bracket-generator.html` ships in both templates.

---

## 1. Placement

```
_templates/
  CLAUDE.md                        <- how to instantiate (§9)
  standard-tournament-template/
    index.html
    match-control.html
    bracket-generator.html
  dual-meet-template/
    index.html
    match-control.html
    bracket-generator.html
```

`_templates/` at the **repo root**, not under `events/`.

Templates must not be served as pages. This repo has no `.nojekyll`, so GitHub
Pages runs Jekyll over it, and Jekyll excludes top-level directories beginning
with `_` from the published site. That is the mechanism keeping `_templates/`
unpublished.

- Do **not** add a `.nojekyll` to this repo without also finding another way to
  exclude `_templates/`. (The *data* repo needs `.nojekyll`; this one must not
  have one. Different repos — don't cross the wires.)
- Templates contain only placeholders and example values, never secrets, so if
  they do leak into the published site nothing is exposed — it is a tidiness
  boundary, not a security one. State that in `_templates/CLAUDE.md`.

`bracket-generator.html` is byte-identical in both templates. The duplication is
deliberate: it makes instantiation a single `cp -r` of one folder. Add a comment
at the top of both copies noting that a change to one must be mirrored to the
other.

---

## 2. Asset paths — fix the bug the originals have

The bkl and PPA pages were authored at `events/<slug>/` and later moved into
`events/archives/` as pure renames (commit `8e7d516`, all `R100`). Their
relative asset paths were never updated, so **in their current location they are
broken**:

- `../favicons/…` → `events/archives/favicons/` (does not exist)
- `../../assets/logo.png` → `events/assets/logo.png` (does not exist)

Templates must not inherit this. **Use root-absolute paths in both templates:**

```html
<link rel="icon" href="/events/favicons/favicon-32x32.png">
<meta property="og:image" content="/assets/logo.png">
```

This works because `sage-match-control.github.io` is a user/org Pages site
served at the domain root. (If the site is ever moved to a project-pages repo
served under `/<repo>/`, root-absolute paths break and every template needs a
prefix — note this in `_templates/CLAUDE.md`.)

Payoff: templates preview correctly in place, instantiation is
location-independent, and moving a finished event into `events/archives/` no
longer breaks its icons — the exact bug that already bit bkl, PPA, and boc.

Leave the archived originals alone; fixing them is a separate cleanup (§11).

---

## 3. Settled decisions

**D1 — Data source: Cloud Run snapshot pipeline.**
Both templates read a published JSON snapshot from GitHub Pages; Cloud Run
fetches the sheets and publishes it. PPA's direct browser-side `gviz` fetch is
**not** carried into either template. Mission Control, the partial-sync status
line, and the "last synced" timestamp all depend on this and are in scope for
both templates.

**D2 — Data repo: one generic repo named `event-data`, a folder per event.**
Not a repo per event. Snapshots live at
`sage-match-control/event-data` → `<event-key>/data/<day-key>.json`, served over
GitHub Pages at `https://sage-match-control.github.io/event-data/…`. Layout and
the `SyncConfig` restructure it implies are in §8.

Consequences: `GITHUB_REPO` stays a single env var, its value becoming
`event-data`; `GitHubPublisher` needs no change; day keys become globally unique
across events; a template's `EVENT_KEY` must equal its folder name in that repo.
Because the name is now fixed, `GHPAGES_REPO` is a **hardcoded constant** in both
templates rather than a per-event token — one less thing to fill in at
instantiation, and one less way for a new event to point at nothing.

**D3 — `bracket-generator.html` ships in both templates.**
It is fully event-agnostic (no config block, no sheet access — pairs are pasted
in by hand), so it is copied verbatim from bkl and needs no genericisation.

**D4 — Organizer page stays unlinked.** Follow bkl: `match-control.html` is not
linked from `index.html`; the URL is handed out directly.

---

## 4. The shared template contract

Both templates must expose the **same** configuration surface, so a session that
has instantiated one can instantiate the other without relearning anything. The
only intentional difference is the club dimension (§6).

### 4.1 Placeholder convention

Two kinds of value, handled differently on purpose:

- **Identity strings that must never survive instantiation** — event name, dates,
  venue, sheet IDs, URLs, club names — are `{{DOUBLE_BRACE}}`
  tokens. They are greppable, and "no `{{` remains" is a hard acceptance check.
- **Structural config that demonstrates the shape** — `DIVISIONS`, `EVENTS`,
  `FACILITIES`, `DAYS` — ships with *working example values* under a
  `// EXAMPLE — replace with this event's own` comment, so the template renders
  and the expected shape is self-evident.

Never leave a real bkl or PPA value as an example: no `bkl`, `BKL`,
`bklCup2026`, `Pickleball Central PH`, `Pampanga Paddle Aces`, `Club 2600`, and
no real sheet IDs or Cloud Run URLs anywhere in `_templates/`.

### 4.2 Token list

| Token | Used in |
| --- | --- |
| `{{EVENT_KEY}}` | config; folder name under `events/` and in the data repo; storage namespace |
| `{{EVENT_TITLE}}` | `<title>`, hero `<h1>`, footer |
| `{{EVENT_TAGLINE}}` | hero subtitle |
| `{{EVENT_HEADLINE}}` | bkl's `.domination-line` slot — optional, blank is fine |
| `{{EVENT_DATE_RANGE}}` | hero eyebrow, footer, meta description |
| `{{VENUE}}` | hero eyebrow, footer, `FACILITIES` |
| `{{QR_IMAGE}}` / `{{QR_URL}}` | QR panel image + printed short link |
| `{{CLOUD_RUN_BASE_URL}}` | `match-control.html` only |
| `{{CLUB_A_CODE}}` … `{{CLUB_B_NAME}}` | **dual-meet template only** |

### 4.3 Config block

Both templates keep bkl's convention: one commented `CONFIGURATION` region at
the top of the `<script>`, closed by an explicit `END CONFIGURATION` marker,
with nothing event-specific below it. An organizer must never need to read past
the marker.

```js
// ---- Event identity -------------------------------------------------------
// Does triple duty: browser-storage namespace, this event's folder in the
// shared data repo, and its folder under events/ in this repo. Keep all three
// identical.
const EVENT_KEY = '{{EVENT_KEY}}';

// ---- Days -----------------------------------------------------------------
// One entry per tournament day. Length drives the UI: see §7.1. Day keys must
// be globally unique across all events in the data repo — prefix them with
// something event-specific.
const DAYS = [ /* EXAMPLE — replace */ ];
const LIVE_GO_LIVE_HOUR_PH = 7;

// ---- Venues / courts ------------------------------------------------------
// Length drives the UI: see §7.2.
const FACILITIES = [ /* EXAMPLE — replace */ ];

// ---- Categories -----------------------------------------------------------
const DIVISIONS = { /* code -> { name, full } */ };
const EVENTS    = { /* code -> label */ };
// Optional overrides. Omit or leave null to fall back to the key order of
// DIVISIONS / EVENTS above, which is the normal case (§7.3).
const DIVISION_ORDER = null;
const EVENT_ORDER    = null;

// ---- Clubs ----------------------------------------------------------------
// DUAL-MEET TEMPLATE ONLY. Absent entirely from the standard template.
const CLUBS = { /* code -> { name, full } */ };

// ---- Data source ----------------------------------------------------------
// Fixed for every event — one shared repo, one folder per event, keyed by
// EVENT_KEY above. Nothing here is filled in at instantiation; if a page ever
// needs a different repo, that is a platform change, not an event setting.
const GHPAGES_OWNER  = 'sage-match-control';
const GHPAGES_REPO   = 'event-data';
const GHPAGES_BRANCH = 'main';
const snapshotUrlFor = dayKey =>
  `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${EVENT_KEY}/data/${dayKey}.json?t=${Date.now()}`;

// ---- Network --------------------------------------------------------------
const FETCH_TIMEOUT_MS = 8000;
const POLL_INTERVAL_MS = 10000;

// ---- Storage keys ---------------------------------------------------------
const STORAGE_DAY_KEY        = `${EVENT_KEY}.dayIndex`;
const STORAGE_SEARCH_KEY     = `${EVENT_KEY}.search`;
const ORG_SECRET_STORAGE_KEY = `${EVENT_KEY}.orgSecret`;  // match-control.html only
```

`STAGE_META` / `STAGE_ORDER` / `BADGE_CLASSES` carry over from bkl unchanged in
both templates.

### 4.4 Theme

Collect every event-specific colour into the single `:root` block at the top of
`<style>`, under a `/* ---------- THEME ---------- */` banner, and state in a
comment that this block is the only place to restyle an event. bkl's palette
(`--court:#C6FF00`, `--court-dark:#060B16`, …) is a reasonable neutral default
for both templates; do not carry PPA's palette into the dual-meet template
purely because that was a dual meet.

### 4.5 Required spreadsheet columns

Identical for both templates. Exact, case-sensitive:

- **Matches tab**: `matchNumber`, `teamCode1`, `team1Player1`, `team1Player2`,
  `teamCode2`, `team2Player1`, `team2Player2`, `Schedule`, `team1Score`,
  `team2Score`, `CourtAssignment`, `court`.
  `court` is the *live* court (distinct from the scheduled `CourtAssignment`)
  and is what drives the Live Matches board. The PPA sheet did not have it;
  without it the Live tab renders "No matches are currently on court." forever.
- **Standings tab**: `teamCode`, `player1`, `player2`, `wins`, `loss`,
  `quotient`, `bracket`.

Document this list in `_templates/CLAUDE.md` — it is the most common cause of a
new event silently rendering empty.

---

## 5. Template A — `standard-tournament-template`

Copy `events/archives/bkl-cup-2026/` and genericise per §7. Everything in the
table below is inherited behaviour to **keep**; it is what makes this worth
templating rather than rebuilding.

| Feature | Where it lives now (`index.html`, script-relative) |
| --- | --- |
| CONFIGURATION block with explicit END marker | L1–L168 |
| Snapshot fetch (`fetchDaySnapshot`, `snapshotUrlFor`) | L286–L307 |
| `FETCH_TIMEOUT_MS` + `AbortController` timeout | L73 |
| Polling, paused while the tab is hidden | L78 |
| Stale-response guard (`dayIndexAtStart`) in `loadLiveData` | L318+ |
| 7 AM PH go-live gating (`computeDayIsLive`, `isLive: 'auto'`) | L42–L52 |
| Partial-sync reporting (`snapshot.failedFacilities`) | in `loadLiveData` |
| Live Matches court board | L1318–L1414 |
| Standings: category toggle bar (desktop) + category autocomplete (mobile) | L855–L924 |
| Standings: scroll capture/restore across re-render | L771–L808 |
| Standings: matchup pairing (`pairUpMatchups`, `renderMatchup`) | L655–L741 |
| Twice-to-beat `(1)`/`(2)` handling (`matchInstanceOf`) | L482–L490 |
| `parseCode` memo cache + regex derived from `DIVISIONS`/`EVENTS` | L147, L448–L463 |
| Autocomplete with clear (×) button | L1118–L1192 |
| QR panel | body, `.qr-panel` |
| Day picker gating the tabs until a day is chosen | L1035–L1117 |

`match-control.html` additionally keeps: the Mission Control tab (resync,
shared secret in `sessionStorage`, connection check, per-facility buttons,
CSV-fallback checkbox — body + script L1541–L1740), and its override of
`computeDayIsLive()` to always return `true` (script L54–L56) so the organizer
page is never gated. Keep both, and keep the comment explaining why.

Team code format: `<DIVISION><EVENT>_<REST>` (e.g. `B18MD_1`, `HI40XD_SF_2`,
`B35XD_F_1_(2)`).

---

## 6. Template B — `dual-meet-template`

Start from the **same genericised bkl base** as Template A — not from the PPA
file — then port the dual-meet model onto it. The PPA page is the reference for
*what a dual meet is*, not the code to build on: it lacks the Live board,
Mission Control, polling, the stale-response guard, and the snapshot pipeline.

### 6.1 Port from PPA

| Feature | Where it lives now (`ppa-x-club-2600-dual-meet.html`) |
| --- | --- |
| `CLUBS` registry, club-prefixed team codes | L~960 |
| `parseCode` with a club segment | L~1010 |
| Club win-totals summary bar (`clubWins`, VS separator, leader highlight) | in `renderStandings` |
| Desktop standings: category column with clubs stacked inside (`renderCategoryColumn`, `renderClubSubsection`) | L~1160–1185 |
| Mobile standings: one column per club, categories inside | `renderStandings` else-branch |
| `divisionLabel()` including the club's full name | L~1030 |
| Hero copy pattern: "Club A × Club B · date · venue" eyebrow | body header |

The club-aware standings rendering **replaces** bkl's club-less category card.

### 6.2 Team code format — the fork point

- Standard: `<DIVISION><EVENT>_<REST>`
- Dual meet: `<CLUB>_<DIVISION><EVENT>_<REST>`

Build the regex the way bkl does, with the club segment added:

```js
const CODE_REGEX = new RegExp(
  `^(${Object.keys(CLUBS).join('|')})_(${Object.keys(DIVISIONS).join('|')})(${Object.keys(EVENTS).join('|')})_(.+)$`
);
```

`parseCode(code)` returns `{ club, division, event, rest }`, keeps bkl's
`parseCodeCache` memoisation, and on no match returns
`{ club: null, division: null, event: null, rest: code }`.

The added `club` is additive, so audit — don't rewrite — these call sites:
`divisionLabel`, `divisionEventLabel`, `categorySortKey`, `standingsStageKey`,
`roundKeyword`, `pairKey`, `renderStandings`, `renderCategoryColumn`,
`renderClubSubsection`.

`pairKey()` must include the club (PPA's already does) so two pairs with the
same names on opposite clubs never merge in the autocomplete index.

### 6.3 Take bkl's round parsing, not PPA's

Use bkl's `roundKeyword()` with anchored regexes (`(^|_)SF(_|\d|$)`), **not**
PPA's `roundLabel()` which uses `rest.includes('SF')`. bkl's version survives
twice-to-beat suffixes and won't false-positive on a bracket label.

---

## 7. Genericisation work

This is the substance of the task. Every item is something currently hardcoded
that must become config-driven, so that "regardless of the categories or the
number of days" actually holds.

### 7.1 Any number of days

`DAYS.length` drives the UI at **runtime**. The instantiator sets the array; they
never delete UI code.

- `length > 1` — day picker visible, tabs gated until a day is chosen,
  `STORAGE_DAY_KEY` remembered across reloads, "Jump to today"
  (`getTodayOrNextDayIndex()`, `match-control.html`) available.
- `length === 1` — picker hidden, day `0` auto-selected on load so tabs and data
  appear immediately, `STORAGE_DAY_KEY` unused, "Jump to today" hidden.

Both templates carry both paths. The dual-meet template merely *ships* with a
one-entry example `DAYS`; it is not restricted to one day.

Keep `computeDayIsLive()` and `isLive: 'auto'` in both — the 7 AM PH gate is
what hides scores and the Live/Standings tabs before an event starts, and it is
just as wanted for a one-day meet.

Also drop bkl's `STORAGE_DAY_KEY` comment about the cancelled Aug 14 day and the
`.v2` suffix — that is bkl history, meaningless in a template.

### 7.2 Any number of venues

- Replace bkl's hardcoded `facilityForCourt()` (courts 1–4 → Main, 5–9 → Annex,
  10–15 → Dreamcourts) with a lookup driven by `FACILITIES`.
- Keep `groupLiveCourtsByFacility()` and the `'Other Courts'` fallback bucket for
  unmapped courts.
- `FACILITIES.length === 1` — suppress the facility title row on the live board
  (a redundant venue name above a single table) and hide the per-facility resync
  row in `match-control.html`, which duplicates "Resync this day now".
- `match-control.html` — generate the per-facility buttons from `FACILITIES`
  instead of the hardcoded `Main` / `Annex` / `Dreamcourts` markup.

### 7.3 Any set of categories

- `CODE_REGEX` already derives from `Object.keys(DIVISIONS)` / `EVENTS` in bkl.
  Keep that, and keep the "don't hand-edit this regex" comment.
- `DIVISION_ORDER` / `EVENT_ORDER` become **optional**: when null, fall back to
  the key insertion order of `DIVISIONS` / `EVENTS`. Today bkl requires a
  hand-maintained parallel list of *full names* that silently mis-sorts if it
  drifts from `DIVISIONS`. Removing that duplication is the single biggest
  reduction in per-event setup work.
- Delete bkl's `divisionMeta()` special case (`/^B(\d+)$/` → "Beginner N+").
  It is a bkl-specific naming convention; unknown codes should fall back to
  `{ name: code, full: code }` only.
- PPA's division chips (`chipDefs = ['IMD','IWD','IXD','HIMD','HIXD']`,
  L~1290) are hardcoded. If the chips row is kept, derive it from the
  division/event combinations actually present in the loaded data. Otherwise
  drop the chips from both templates.

### 7.4 Strings and branding

- Every event-specific string in the HTML body — `<title>`, meta description,
  all `og:` tags, hero eyebrow / title / tagline / headline, footer, QR caption
  and link text — becomes a `{{TOKEN}}` from §4.2.
- Keep the S.A.G.E. crown/shield SVG mark and the "Powered by S.A.G.E. Match
  Control Experts" footer line in both templates: those are the operator's
  brand, not the event's.
- Storage keys are namespaced off `EVENT_KEY` (§4.3), so a new event can never
  read or clobber another event's saved state in a shared browser.

---

## 8. Backend: multi-event `SyncConfig` (one-time)

Templates are only instantiable if the API can serve more than one event. This
work happens **once**, not per event.

> **Superseded — see `sync-config-runtime-spec.md`.** This section was
> implemented as described, then replaced: `src/sync/SyncConfig.mjs` no longer
> exists. The event/day/facility registry now lives in the `event-data` repo at
> `config/events.json`, fetched and cached at runtime by `SyncConfigStore.mjs`,
> so adding an event is a commit to that repo rather than a Cloud Run redeploy.
> The data-repo layout (§8.1), the `<event>/data/<day>.json` path, the
> globally-unique day-key rule, and the decision to keep the flat `/sync/:day`
> route all still hold exactly as written below. What changed is only *where the
> registry lives* — §8.2's `EVENTS` const and its module-load `DAY_INDEX` are
> historical.

### 8.1 Data repo layout

```
event-data/
  .nojekyll                  <- at the root, once for the whole repo
  <event-key>/
    data/
      <day-key>.json
```

Keeping `data/` *inside* each event folder makes the new layout a pure prefix of
today's `data/<day>.json`, so `repoPathFor` grows one path segment and nothing
else about publishing changes.

Because there is still exactly one repo, `GITHUB_REPO` stays a single env var —
it just points at `event-data`. No per-day repo override, no second Cloud Run
service.

### 8.2 `src/sync/SyncConfig.mjs`

Restructure the flat `day2`..`day6` `DAYS` map into an event-keyed registry, and
derive a flat day index at module load:

```js
const EVENTS = {
  'bkl-cup-2026': { days: { day2: { label: '…', facilities: [ /* unchanged */ ] }, /* … */ } },
  // new events appended here at instantiation time
};

// dayKey -> { event, entry }, built once at load. Throw on a duplicate day key
// across events rather than letting one event publish into another's folder —
// this guard is the only thing keeping the flat /sync/:day route safe.
const DAY_INDEX = /* … */;
```

- `getDay(day)` returns `{ day, event, label, facilities }` — `event` is
  **additive**, so existing callers keep working.
- `repoPathFor(event, day)` returns `` `${event}/data/${day}.json` ``.
- `knownDays()` still returns every day key across all events, so
  `UnknownSyncDayError` keeps listing something useful.
- `SyncService.syncDay()` already calls `getDay(day)` — take `event` off that
  result and pass it to `repoPathFor`. That is its only change.
  `GitHubPublisher` needs none.

**Keep the route as `/sync/:day`.** Day keys are globally unique, so the event is
derivable from the key. This keeps the 15 Apps Script installs already on the
bkl spreadsheets working untouched — they build their URL as
`${CLOUD_RUN_BASE_URL}/sync/${DAY_KEY}`. A `/sync/:event/:day` route would be
marginally tidier but breaks every installed script for nothing the duplicate-key
guard doesn't already provide.

### 8.3 Sheet tab names / GIDs

`SyncConfig` assumes every facility sheet uses tabs named `CSV` (matches) and
`STANDINGSCSV` (standings), with GIDs `2136121736` / `1507029786` for the
`?method=csv` fallback. These are module constants today. **Move them to per-day
config** (defaulting to the current values) — templating means new events will
arrive with sheets that don't match. The PPA sheet already used a different
standings GID (`327842042`), so this is a real variation, not a hypothetical.

### 8.4 Apps Script

Rename `scripts/bkl-sheets-sync.gs` to `scripts/sheets-sync.gs` now that it
serves more than one event, and generalise its header comment (it currently
documents installing across "15 facility spreadsheets — 5 days x 3 facilities").
Its CONFIG block already parameterises `DAY_KEY` / `FACILITY_NAME` /
`CLOUD_RUN_BASE_URL`; the watched tab GIDs (SCHEDULE `331216315`, COURT CONTROL
`359580789`) are also per-spreadsheet and must be called out as such.

### 8.5 Known consequence for bkl

Once `GITHUB_REPO` points at `event-data`, a resync of a bkl day publishes to
`event-data/bkl-cup-2026/data/…`, which is **not** where the archived bkl
pages read from — their live data would stop updating. Harmless (the event ended
in Aug 2026 and nothing triggers a resync), but leave a comment in `SyncConfig`
saying so rather than leaving a trap for whoever revives that event. Migrating
the old published data is out of scope (§11).

---

## 9. `_templates/CLAUDE.md` — the instantiation runbook

Write this file as part of the task. It is what makes the templates usable
without re-reading this spec, and Claude Code picks it up automatically when
working under `_templates/`.

It must cover:

1. **Choosing a template** — clubs facing each other → `dual-meet-template`;
   otherwise `standard-tournament-template`. Day count and category count do
   **not** affect the choice; both templates handle any number of each.
2. **The steps**, in order:
   - `cp -r _templates/<template> events/<event-key>/`
   - rename `bracket-generator.html`'s sibling QR image placeholder
   - replace every `{{TOKEN}}` (§4.2) — then `grep -r '{{' events/<event-key>/`
     must come back empty
   - replace the `// EXAMPLE — replace` config values: `DAYS`, `FACILITIES`,
     `DIVISIONS`, `EVENTS`, and `CLUBS` for a dual meet
   - set the `:root` theme block if the event wants its own palette
   - add the event + its days to `event-data`'s `config/events.json` (see the
     §8 note — this was `SyncConfig.mjs` when this spec was written), using day
     keys prefixed with the event key
   - install `scripts/sheets-sync.gs` on each facility spreadsheet
   - create the `<event-key>/data/` folder in `event-data`
3. **The required spreadsheet columns** (§4.5), stated as the first thing to
   verify — it is the most common cause of a new event rendering empty.
4. **The team code format** for each template (§5, §6.2), with examples.
5. **Root-absolute asset paths** (§2) and why not to "fix" them to relative.
6. **Archiving**, later: move the folder into `events/archives/`; because paths
   are root-absolute nothing needs re-prefixing. Note that this is the step the
   old events got wrong.
7. **The `_templates/` publication boundary** (§1): no `.nojekyll` in this repo.

---

## 10. Acceptance checklist

**Both templates**

- [ ] `grep -rniE 'bkl|bklCup2026|Pickleball Central PH|Pampanga Paddle Aces|Club 2600|tinyurl' _templates/` returns nothing.
- [ ] No real sheet ID, Cloud Run URL, or QR short link anywhere in `_templates/`.
- [ ] Every `{{TOKEN}}` in §4.2 appears in the template that needs it, and every
      `{{` in the templates is listed in §4.2.
- [ ] Config block ends with an explicit `END CONFIGURATION` marker and nothing
      event-specific appears below it.
- [ ] Asset paths are root-absolute and resolve when the template is opened in
      place (§2).
- [ ] Setting `DAYS` to 1 entry hides the picker and auto-loads; setting it to 3
      shows the picker and gates the tabs — **without editing anything but the
      array** (§7.1).
- [ ] Setting `FACILITIES` to 1 entry suppresses facility headings and the
      per-facility resync row; 2+ restores both (§7.2).
- [ ] Adding a division/event to `DIVISIONS`/`EVENTS` flows through parsing,
      sorting, and display with `DIVISION_ORDER`/`EVENT_ORDER` left `null`
      (§7.3).
- [ ] All three tabs render against a hand-made snapshot JSON.
- [ ] Before the configured date at 07:00 PH, `index.html` hides scores and the
      Live/Standings tabs; `match-control.html` shows them regardless.
- [ ] Mission Control: connection check hits `/ping`, resync sends
      `X-Sync-Secret`, secret lives in `sessionStorage` only and is never written
      into the file.
- [ ] Polling stops while the tab is backgrounded and resumes on focus.
- [ ] Standings and live board keep scroll position across a poll re-render.
- [ ] Mobile (≤980px) and desktop standings layouts both render.

**`dual-meet-template` only**

- [ ] Standings show the club-vs-club win summary with the leader highlighted.
- [ ] A pair with the same names on both clubs is not merged in the autocomplete.
- [ ] Desktop shows category columns with clubs stacked inside; mobile shows one
      column per club.

**Backend**

- [ ] `snapshotUrlFor()` produces `…/event-data/<event-key>/data/<day>.json`.
- [ ] `SyncConfig` throws at module load if two events declare the same day key.
- [ ] Regression: `getDay('day2')` still resolves to the `bkl-cup-2026` event and
      `repoPathFor` yields `bkl-cup-2026/data/day2.json` — the restructure must
      not change what existing day keys mean.
- [ ] Sheet tab names/GIDs are per-day config with the current values as
      defaults (§8.3).

**Runbook**

- [ ] `_templates/CLAUDE.md` exists and covers all seven points in §9.
- [ ] Following it end-to-end against a scratch event produces a working page
      without opening this spec.

---

## 11. Out of scope

- Creating any real event. §9 is the runbook; running it is a later task.
- Migrating the published `bkl-cup-2026` data into `event-data` (§8.5).
- Fixing the broken relative asset paths in the existing archived pages (§2).
  Templates avoid the bug; retrofitting the old pages is separate.
- Generating QR PNGs or short links.
- Any change to the scoresheet half of `sage-tools-api`.
- Retiring `events/archives/ppa-x-club-2600-dual-meet.html` or the bkl folder.
  Templates are copies; the originals stay exactly as they are.
