# Event spec template

> **Not a spec, and no build status.** This is the starting point for every new event's spec,
> which is why it sits at the root of `specs/` and not in a status folder. It is written from the
> seven event specs before it, Pickle for Sight to the UPAP 3rd Annual Tournament, and takes its
> shape from UPAP's, the first built in phases.

An event spec is what lets someone with **no other context** build an event's pages and register
it: every value, every edit, every check, written out. The procedure itself, which file to copy and
which token means what, is the runbook in the site repo, `_templates/CLAUDE.md`. The spec does not
repeat the runbook: it applies it to one event, with the event's own values, and adds the edits the
runbook leaves to judgement (a re-skin, the board's colours, any engine change).

There is one template for all three event types. Where they differ, the skeleton in Part 2 has a
block for each: **[Standard]**, **[Team]** and **[Dual meet]**. Keep the block for the event's type
and delete the others.

---

## Part 1 — How to write an event spec

### 1.1 Gather before you write

Write nothing until you have all of this. Every fact in the spec comes from one of these sources,
and the spec says which.

- **The workbook**, read through the Sheets API, never from memory or a screenshot of it:
  - `Title` and `Variables`: the event name, date, facility, courts.
  - `SCHEDULE` and `CSV`: every match, numbered, with its time and court; the first serve; the
    slot length; any empty court.
  - `STANDINGSCSV`: every pair or team and every playoff seat.
  - `ATTENDANCE`: that its header is there.
  - **[Team]** `Teams` and `MatchUps` as well: the team names, rosters and pair numbers.
  - The generator version, from **SAGE → Help**. A format the master's generator does not build
    yet is a workbook nobody may regenerate; the spec says so in its operator tasks.
  - The facility name exactly as the workbook spells it. Live-sync setup compares it, with case.
- **The pubmat** (the organiser's poster), if there is one: sample the palette from it, and the
  display font closest to its lettering. Measure the contrast of every pair the template uses
  (§1.4). With no pubmat the event keeps the house colours, as the Angeles City Employees
  Sportsfest did, and the spec has no re-skin.
- **The logo and the QR image.** The logo is staged untracked in `events/<event-key>/assets/`
  before the spec is written. The QR image usually follows later; the spec says what happens
  without it. The pubmat may sit beside them for reference and is **never committed**.
- **The precedent**: the last event spec of the same type. Read what it did and what its status
  line says went wrong.
- **The owner's answers.** Ask before writing the edits, not after: a changed answer moves every
  check that depends on it. The questions every event needs:

  | Question | Why it matters |
  | --- | --- |
  | The short link (`tinyurl.com/SAGEx…`) and who makes the QR image | `{{QR_URL}}`, the QR panel |
  | Attendance: desk links, console only, or none | `"attendance"`, whether `attendance.html` exists |
  | Score entry: scorer links, console only, or none | `"scoreEntry"`, whether `scorer.html` exists |
  | The category or bracket labels | `display` in `events.json`, and every Standings check |
  | Which logo, if the organiser has more than one | `{{EVENT_LOGO}}` |
  | Whether the staff pages (desk, scorer) take the event's colours or keep the house colours | Phase 3's edits. House colours unless the owner says otherwise |
  | The build order, if not the default (§1.2) | The phases |
  | The first serve, if the workbook and pubmat disagree | The go-live time and every timing check |

  **[Team]** also: how teams are ranked (`teamScoring`, `headToHeadQuotient`), the pair labels
  (`display.pairs`), the playoff seats' labels before they're filled (`display.seedLabels`), team
  colours (`display.teamColors`), and whether lineups are hidden until shown (`lineupReveal`) or
  checked (`lineupRules`).
  **[Dual meet]** also: each club's code and name, and both clubs' logos.

  The answers go in the spec's Questions section, dated.

### 1.2 Choose the phases

Build in **phases**, each its own branch, its own checks and its own push. The default is three:

| Phase | Builds | Why here |
| --- | --- | --- |
| 1 | The schedule board **and** the event's `events.json` entry | Registering the event lets the operators set up live sync and rehearse on the real board while the rest is built |
| 2 | The Tournament Hub, with its logo (and the QR image, if it is in) | The public page; the short link goes out after this |
| 3 | Everything else: the desk page, the scorer page, the dry-run checklist, the QR image | Staff pages, needed by the dry run |

The spec says what is live between phases, and what must not be used yet (Control Center's **Open
Tournament Hub** leads to the 404 page until Phase 2; **Issue desk link** waits for Phase 3).

**An engine change comes first**, as its own phase before the pages, when the event needs
something `lib/v1/` doesn't do (Ladies' Dink's tiebreaker game, lettered brackets). That phase:

- Edits `lib/v1/` **compatibly only** (`_templates/CLAUDE.md` §8): add, never remove or rename.
  One module may start using another's new export only in a later push, unless the export is in a
  module that did not exist before.
- Writes the tests first and watches them fail.
- If it touches the "played" or BYE rules, changes `sage-tools-api`'s `facilityCompletion.mjs` in
  step, with its tests, a version bump and a changelog entry, and pushes the API first.
- Lists the docs pages it changes (`technical/site-engine.md`, the features pages) and the
  workspace `CLAUDE.md` lines.

One pass instead of phases only when the owner asks for it.

### 1.3 Write every edit so it can't be misapplied

- **Anchor by text, never by line number.** Each anchor is **one line** and occurs **exactly
  once** in its file; check that in the dry run. A range is given by its first and last line and
  the number of lines between them, so a wrong match is caught.
- **Give every replacement in full**, in a fenced block, leading spaces included.
- **State each file's line endings.** The checkout mixes them, and a search across lines can miss
  text that looks identical. As of 12 November 2026:

  | Template file | Line endings |
  | --- | --- |
  | `standard-tournament-template/index.html`, `dual-meet-template/index.html` | LF |
  | `standard-tournament-template/schedule.html`, `dual-meet-template/schedule.html` | CRLF |
  | `team-tournament-template/index.html` and `schedule.html` | LF |
  | `scorer/scorer.html` | LF |
  | `attendance/attendance.html`, `dry-run-checklist-template.md` | CRLF |

  Check again when writing; the spec repeats what it found. Git stores LF either way.
- **Never `sed -i` the site checkout**: Git Bash's `sed` strips CR on this machine. A scripted
  edit is a Node script that detects the file's line ending, splits on it and joins on it.
- **Every check has its exact expected output**: `Prints nothing.`, `Prints 8.`, the full line.
  Get each number by running the command in the dry run, never by counting by hand (UPAP's §5.4
  counted a 19-line comment as 17).
- **Every phase starts with `git status --short` and the exact output it expects**, unrelated
  untracked files included, so the implementer can tell an expected file from a stray one.
- **The `events.json` entry is added by a Node script**, saved outside the repo, that refuses to
  run unless the file is in `JSON.stringify(…, null, 2)` form and the key is absent, adds the
  entry and prints the last two event keys. Before it, a `grep -c` shows the event and day keys are
  free: a duplicate day key fails the whole file's validation, and the API keeps the old copy.
- **A page cites the spec** in its comments as `sage-docs/docs/specs/.../<name>-spec.md` (the
  workspace `CLAUDE.md`'s rule).
- **A re-skin writes every colour through a `var(--token)`**, keeps the block headed **lib/v1/css's
  house colours, through this page's tokens** whole, and adds each re-skinned page to `PAGES` in
  `_tests/unit/house-colours.test.mjs` (**[Team]** a re-skinned Hub to `TEAM_HUBS` too). The token
  names stay `--navy` and `--green` whatever the colours are.

### 1.4 Write the checks

- **Contrast**, measured (WCAG), for: white on `--navy`; `--navy` on `--green`; `--ink` and
  `--ink-soft` on `--paper`; `--green-dark` on white; `--green` on white (a fill colour if under
  4.5:1). The spec's design notes list them.
- **A fixture** (Appendix A): a Node script that writes `_fixtures/<event-key>/<state>.json` and
  adds the event to `_fixtures/config.json`, shaped like the real workbook, with fixed scores so it
  prints the same line every run. It is **never committed**: the harness reads
  `_fixtures/config.json`, and an extra event changes its screenshots. Each phase removes it.
  **[Team]** writes `pre`, `mid` and `finished`; the others `finished` at least.
- **Browser checks at the widths that change the layout:** 375px (phone), **1024px** (the Hub's
  QR panel shows from 900px wide and can cover the hero's title just above that), 1280px, and
  1440px for the board. UPAP's checks skipped 1024 and missed exactly that overlap.
- **Write each check as what is seen**, with the fixture's real numbers and names, and list what is
  expected and not a bug (a 404 for `qr.png` before it's supplied).

### 1.5 Dry-run the spec before handing it over

Apply every edit exactly as written to a **detached worktree** of the site at the commit the spec
names, never to the checkout:

```bash
git -C /d/Personal/SAGE/sage-match-control.github.io worktree add --detach <scratch>/dry-run <sha>
```

In it: every anchor matched once; each phase's unit tests (record the count); the fixture's
printed line; every browser check; the `events.json` script on a **copy** of the file. Then remove
the worktree (`git worktree remove`). The status line records the date, what was run and what was
**not** (usually `npm run verify`'s screenshot comparison, commits and pushes).

If `main` has moved by the time a phase runs, re-dry-run that phase and update its counts.

### 1.6 Deadline, pushes and filing

- **The deadline is the site freeze**: no merge to the site's `main` within 3 days of the event.
  The status line names the last day a phase may land, and says that after it a push needs the
  owner's explicit go-ahead.
- **Every push and merge to `main` is the owner's yes in chat.** Each phase ends in a *Stop and
  ask* step that says what the push will publish.
- **Filing:** the spec starts in `not-started/`, named after the event
  (`upap-3rd-annual-tournament-spec.md`); the filename is unique across all four folders. It is
  added to the specs README's not-started list, `not-started/README.md`'s table and `mkdocs.yml`'s
  nav, with a MINOR docs version. It moves by the procedure in the
  [specs README](README.md#moving-a-spec-between-folders): to `in-progress/` when its first phase
  is pushed, to `implemented/` once the event has run. Each move is a MINOR docs version.
- **After the event**, the spec's §8 turns live push off (`LIVE_BASE_URL = ''` in every page);
  archiving into `events/archives/` is a later, separate step (`_templates/CLAUDE.md` §7).

### 1.7 What an event spec does not do

- Edit the workbook. That is the operators' list, §8 of the skeleton, written down so its
  constraints on the data are known.
- Hold a secret. The sync secret is read from Cloud Run by whoever manages it, never pasted into
  the spec, chat or a document.
- Tidy anything it doesn't need. "Don't rename, fix or tidy anything this spec doesn't mention"
  is one of its rules.

---

## Part 2 — The skeleton

Copy everything inside the fence into `not-started/<name>-spec.md`. Replace every `<…>`; keep the
type block that fits and delete the others; delete a section's guidance lines in *italics* once
it's written. The section numbers assume no engine phase: with one, it becomes §4 and the rest move
down by one.

~~~~markdown
# Spec — <Event name>

> **Status: not started.** Written <YYYY-MM-DD> against `sage-match-control.github.io` at
> `<sha>`, `event-data`'s `config/events.json` at `<sha>`, and the event workbook as read that day
> through the Sheets API. The owner answered the open questions on <date>; the answers are in §9.
>
> **Built in <three> phases** (§0.3): <the schedule board with the event's `events.json` entry,
> then the Tournament Hub, then everything else>. Each phase is its own branch, its own checks and
> its own push.
>
> **Dry-run on <YYYY-MM-DD>.** Every edit of <§…> was applied to a detached worktree of the site
> at `<sha>`: every anchor matched exactly once. Appendix A's fixture was built and <the pages>
> were checked in a browser at 375px, 1024px, 1280px and 1440px; the checks in <§…> are what they
> showed. The site's unit tests passed after each phase's test change (<n>, then <n>). §4.6's
> `events.json` script was run on a copy of the file and printed what §4.6 expects. Not run in the
> dry run: <`npm run verify`'s screenshot comparison, and any commit or push>.
>
> **Deadline.** The event is on **<Weekday d Month YYYY>**. The site repo takes no merge within
> 3 days of an event, so **all phases reach the site's `main` by <Weekday d Month>**. After that,
> push only with the owner's explicit go-ahead.

Build the event pages for the **<Event name>**, <a one-day doubles tournament> run by **<the
organiser>** on **<date>** at **<venue, town>**, and register the event so the live system knows it.

The pages are copies of the site's <standard / team / dual-meet> event template, filled in for this
event and <re-skinned from the organiser's pubmat: …> <or: in the house colours>. <**No shared code
changes.** or: Phase 1 makes <n> compatible engine changes first: …>

---

## 0. Briefing — read all of it before you change anything

### 0.1 What you are working in

| Thing | Where | What it is |
| --- | --- | --- |
| **The site repo** | `D:\Personal\SAGE\sage-match-control.github.io` | A GitHub Pages site: hand-written HTML and native JavaScript modules, no build step, no framework. Pushing its `main` branch publishes it within a minute or two |
| **The engine** | `lib/v1/` in the site repo | Shared JavaScript and CSS that every event page loads. **You never edit it** <unless Phase 1 says so> |
| **The template** | `_templates/<type>-template/` (`index.html`, `schedule.html`) | The starting point for the event's pages: short HTML shells. **You copy it, never edit it** |
| **The `event-data` repo** | `D:\Personal\SAGE\event-data` | The shared data store. Its `config/events.json` registers every event. Its `main` also receives automatic sync commits, so always `git pull` right before you change it |
| **The workbook** | Google Sheet `<sheet id>` | Where the operators keep the schedule and scores. You don't touch it |
| **Control Center** | `tools/control-center.html` in the site repo | The operators' console. It lists every event in `events.json` |

Words this spec uses:

- **Fixture mode**: on `localhost` only, `?fixture=<state>` makes an engine page read
  `/_fixtures/config.json` and `/_fixtures/<event-key>/<state>.json` instead of the real data.
  Appendix A writes them. **The fixture is never committed.**
- **House colours**: the site's default palette (navy `#14263C`, green `#7CB92C`, off-white
  `#F6F7F2`). A **re-skin** replaces them in a page's `:root { … }` block, and
  `_tests/unit/house-colours.test.mjs` lists the re-skinned pages it checks.

> **[Standard]**
> - **Team code**: how the workbook names a pair, `<CATEGORY>_<n>`, e.g. `<B45HMD_2>`. Playoff
>   seats are `<CATEGORY>_F_1` (Final, seat 1), `<CATEGORY>_SF_2`, `<CATEGORY>_B_1` and so on.
> - **Category token**: the part before the first `_`: a **division** then an **event** (`MD`).

> **[Team]**
> - **Team code**: `<SIDE>_<PAIR>`. `SIDE` is a team letter in the bracket stage (`A`), or
>   `<STAGE>-<SLOT>` in a playoff (`QF-3`). `PAIR` is the pair number in the matchup.
> - **Matchup**: two teams' games against each other, sharing one `matchUp` value.

> **[Dual meet]**
> - **Team code**: `<CLUB>_<DIVISION><EVENT>_<n>`, e.g. `<PPA_B18MD_1>`. `<CLUB>` is one of the
>   two club codes, `<A>` and `<B>`.

### 0.2 Rules

- **Find edits by their anchor text, never by line number.** Every anchor is a single line and
  occurs exactly once in its file. If it isn't there, or is there more than once, **stop and
  report**. Don't guess and don't improvise a different edit.
- **Keep each file's line endings.** <List each copied file: CRLF or LF.> Where an edit replaces
  several lines, it names the first and the last line of the range and how many lines it spans.
  **Never use `sed -i`**: on this machine it strips the CR from every line.
- **Replacement text is given in full** in each fenced block. Copy it exactly, including leading
  spaces. Never retype a value from memory.
- **Use Git Bash** for every command. Python is not installed; Node is.
- **Edit only what the phase names:** files in `events/<event-key>/`, the list in
  `_tests/unit/house-colours.test.mjs`, and `config/events.json` in `event-data` <and, in
  Phase 1, the engine files it lists>. Don't edit `lib/`, `tools/`, `_templates/`, `_fixtures/`
  (except by running Appendix A, which you then undo) or any other event's folder. If something
  there seems to need a change, stop and report.
- **One phase at a time.** A phase starts only when the one before it is merged into `main` and
  pushed.
- **Commit** where a phase says *Commit*, with the message it gives, followed by a blank line and
  the attribution line your session's instructions give.
- **Never push, and never merge into `main`, without the owner's yes in chat.** Each phase ends at
  a **Stop and ask** step.
- **Never commit `assets/pubmat.*`.** It is reference only.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.3 The phases

| Phase | Builds | Branch | Ends with |
| --- | --- | --- | --- |
| **1** (§4) | `events/<event-key>/schedule.html` and the event's entry in `event-data/config/events.json` | `<short>-board` | Push of the site **and** `event-data` |
| **2** (§5) | `events/<event-key>/index.html` and its logo | `<short>-hub` | Push of the site |
| **3** (§6) | <`attendance.html`, `scorer.html,`> the dry-run checklist, and the QR image if not already in | `<short>-rest` | Push of the site |

What is live in between: <after Phase 1 the event is registered and the operators can set up live
sync (§8.2); **Open Tournament Hub** leads to "Page not found" until Phase 2; **Issue desk link**
waits for Phase 3; don't hand out the short link before Phase 2>.

### 0.4 Commands you will use in every phase

Unit tests (from the site repo root): `cd _tests && npm test; cd ..`

Full check (unit tests, then the comparison of `main` against your branch):
`cd _tests && npm run verify; cd ..`. If it says a Playwright browser is missing, run
`cd _tests && npx playwright install chromium; cd ..` once.

Serve the **repo root** on `localhost` for a browser check (Ctrl+C to stop):
`npx --yes http-server -p 8000 -c-1`

---

## 1. Event facts

From <the pubmat and the event workbook (`Title`, `Variables`, `SCHEDULE`, `CSV` tabs)>.

| Fact | Value |
| --- | --- |
| Organiser | <…> |
| Event name | <…> |
| Date | <Weekday, d Month YYYY — one day> |
| Venue | <…>. One facility, **<n> courts**. The workbook's facility name is exactly **`<Facility>`** |
| First serve | <h:mm AM> |
| Matches | <n>, numbered 1–<n>, <25>-minute slots from <…> to <…> |
| Event key | `<event-key>` (the folder under `events/`, the key in `events.json`, the folder in `event-data`) |
| Day key | `<day-key>` (unique across every event in `events.json`) |
| Short link | `tinyurl.com/<SAGExEVENT>` → `https://sage-match-control.github.io/events/<event-key>/` |
| Not on the site | <entry fee, registration, …> |

> **[Standard]** A table of categories: code, the workbook's name, pairs, round robin (single,
> double, best-of-3), playoff. Then how the day runs (waves, courts per category, empty courts).

> **[Team]** The teams (code, name, bracket), the matchup shape (how many games, how a matchup is
> won), the ranking and its ties (`teamScoring`), and the playoff seeding.

> **[Dual meet]** The two clubs (code, name), the categories and how many pairs each club fields,
> and how the club score is counted.

---

## 2. Inputs

| Input | Status | Used in |
| --- | --- | --- |
| Logo | <Already in the site repo, untracked, at `events/<event-key>/assets/logo.<ext>` (<size>, <background>)>. Committed in Phase 2 | §5 |
| Pubmat | <Untracked at `events/<event-key>/assets/pubmat.<ext>`>. **Reference only: never commit it** | — |
| QR image | <To follow from the owner as `events/<event-key>/assets/qr.png`>. If it isn't there when a phase runs, carry on and report it missing. Don't generate one | §5 (if present), else §6 |
| Event workbook | `<sheet id>` | §4.6 |

> **[Dual meet]** Both clubs' logos as well, square, for `{{CLUB_A_LOGO}}` and `{{CLUB_B_LOGO}}`.

If <the logo or the pubmat> is missing when you start, stop and report.

---

## 3. Design notes (no action — the reasons behind the edits)

1. **The palette**, sampled from <the pubmat and the logo>, with each token's role and where it
   came from, and the measured contrast: white on `--navy`; `--navy` on `--green`; `--ink` and
   `--ink-soft` on `--paper`; `--green-dark` on white; `--green` on white.
2. **The font**, if any is added, and exactly where it is used.
3. **The board's colours**: <the organiser's, read off `SCHEDULE`> or <ours, because …>. Each
   category's (or bracket's) hue, and why.
4. **The labels**: what the engine shows for each code, with the owner's wording.
5. <Anything else a reader needs in order to understand an edit.>

*With no pubmat: say the event keeps the house colours, and delete points 1 and 2.*

---

## 4. Phase 1 — the schedule board and `events.json`

### 4.1 Start

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git status --short
```

This must print exactly:

```
?? events/<event-key>/
```

<and any unrelated untracked file the dry run saw, named>. Anything else: stop and report.

```bash
git checkout main
git pull
git checkout -b <short>-board
cd _tests && npm install && npm test; cd ..
```

`npm test` must pass before you change anything.

### 4.2 Copy the board and fill in its tokens

```bash
cp _templates/<type>-template/schedule.html events/<event-key>/schedule.html
```

Replace **every** occurrence of each token (their meanings: `_templates/CLAUDE.md` §3):

| Token | Replace with |
| --- | --- |
| `{{EVENT_KEY}}` | `<event-key>` |
| `{{EVENT_TITLE}}` | `<Event name>` |
| `{{SCHEDULE_DAY_KEY}}` | `<day-key>` |

> **[Dual meet]** and `{{CLUB_A_CODE}}`, `{{CLUB_B_CODE}}`, which also set `CLUB_ORDER`.

**Check:** `grep -n '{{' events/<event-key>/schedule.html` prints nothing.

### 4.3 The board's theme

*Only for a re-skin. Each edit: (a), (b), … with its anchor, its range and line count, and its
full replacement. Usually the theme comment, the `:root` colours (`--navy` through `--radius`,
eleven lines), the font link, and one rule before `</style>`. Leave the block headed
**lib/v1/css/schedule-board.css's house colours, through this page's tokens** as it is.*

### 4.4 The board's category colours

Replace the template's `CAT_META` (its comment block through the `};` that closes it) with:

```js
<// Where the colours come from (sage-docs/docs/specs/.../<name>-spec.md §3).>
const CAT_META = {
  <KEY>: { short:'<CHIP>', color:'#<hex>' },
};
```

> **[Standard]** / **[Dual meet]** One key per category token (`<DIVISION><EVENT>`).
> **[Team]** One per bracket (`G1`, or `GA` for a lettered bracket), plus `PO` for the playoffs.

**Checks**, each with its expected output: no template example key or house colour left; the
number of `short:` entries; the font name's count, if one was added.

### 4.5 Add the board to the house-colours test

*Only for a re-skin.* In `_tests/unit/house-colours.test.mjs`, find the last line of `PAGES`
<quote it> and replace it with itself and `'events/<event-key>/schedule.html',`. Run the unit tests;
they must pass, <n> more than before.

### 4.6 Register the event in `event-data`

```bash
cd /d/Personal/SAGE/event-data
git checkout main
git pull
git status --short
```

Prints nothing. Then the keys must be free:

```bash
grep -c '"<day-key>"\|"<event-key>"' config/events.json
```

Prints `0`. Save this outside the repo as `add-<short>-event.mjs`, and run it from the `event-data`
root:

```js
import fs from 'node:fs';

const path = 'config/events.json';
const raw = fs.readFileSync(path, 'utf8');
const config = JSON.parse(raw);
if (JSON.stringify(config, null, 2) !== raw) throw new Error('events.json is not in JSON.stringify(…, null, 2) form: stop and report');
if (config.events['<event-key>']) throw new Error('<event-key> is already registered: stop and report');

config.events['<event-key>'] = {
  type: '<standard | team | dual-meet>',
  title: '<Event name>',
  attendance: '<desks | console>',
  scoreEntry: '<links | console>',
  days: {
    '<day-key>': {
      label: '<Mon d>',
      date: '<YYYY-MM-DD>',
      isLive: 'auto',
      facilities: [{ name: '<Facility>', sheetId: '<sheet id>' }],
    },
  },
  display: { /* the type's block below */ },
};

fs.writeFileSync(path, JSON.stringify(config, null, 2));
console.log(Object.keys(config.events).slice(-2).join(' then '));
```

> **[Standard]** `display: { divisions: { <code>: '<label>', … }, events: { MD: "Men's Doubles", … } }`.
> The key order sets the display order. Add `playoffSeats` only if the workbook has no `seat` column.

> **[Team]** No `divisions`. Add `teamScoring` (and `headToHeadQuotient`) if not points;
> `display.pairs`, `display.seedLabels`, `display.teamColors` as the owner answered;
> `lineupReveal` and `lineupRules` if used. Their shapes: `event-data/config/README.md`.

> **[Dual meet]** `display: { divisions, events, clubs: { <A>: '<Club A name>', <B>: '<Club B name>' } }`.

It must print `<previous last event> then <event-key>`. Explain each field's meaning in a list
under it. **Check:** `git diff --stat` shows only `config/events.json`, <n> lines added and none
removed, and a `node -e` one-liner prints <the entry's type, day, facility and labels>.
*Commit* on `main`, **don't push yet**:

```bash
git add config/events.json
git commit -m "<Event name>: register <event-key> (day <day-key>, <Facility>)"
cd /d/Personal/SAGE/sage-match-control.github.io
```

### 4.7 Commit the site's Phase 1

```bash
git add events/<event-key>/schedule.html _tests/unit/house-colours.test.mjs
git status --short
git commit -m "<Event name>: schedule board<, re-skinned from the pubmat>"
```

`git status` before the commit: <the exact staged and untracked lines>.

### 4.8 Check the board and Control Center in a browser

Run Appendix A from the site repo root. It must print exactly: `<the fixture's line>`.

Board at **1440px**: `http://localhost:8000/events/<event-key>/schedule.html?fixture=finished`

- [ ] <The top bar, the title, the meta line "<MON D> · <n> MATCHES · <n> SLOTS · ALL COURTS".>
- [ ] <The court filter and the time column's first and last slot.>
- [ ] <The legend's chips, in the order categories first appear in the day.>
- [ ] <Which court shows which colour when; the cells that read OPEN; the FINAL and BRONZE tags.>

Control Center: `http://localhost:8000/tools/control-center.html?fixture=finished&event=<event-key>&day=<day-key>`

- [ ] <The header; Mission Control's facility, "<n> / <n> done · All matches done", the go-live line.>
- [ ] <Standings: every category's label, in order; no raw code; no *Unmapped category* warning.>
- [ ] <Awards: "<n> of <n> podiums complete", each podium named.>
- [ ] <Attendance: "<Facility>: No roster yet".>
- [ ] **Expected, not bugs:** <the 404s; Open Tournament Hub leading to "Page not found">.

Stop the server and remove the fixture:

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/<event-key>
git status --short
```

`git status` shows nothing under `_fixtures/`.

### 4.9 Full check

`npm run verify` must pass. A failure means something outside this phase's files changed: run
`git status` and stop and report.

### 4.10 Stop and ask: push Phase 1

Tell the owner Phase 1 is ready, with the results of §4.8 and §4.9, and that pushing publishes the
board and registers the event. On a yes:

```bash
git checkout main
git pull
git merge --ff-only <short>-board
git push
cd /d/Personal/SAGE/event-data
git pull --rebase
git push
cd /d/Personal/SAGE/sage-match-control.github.io
```

If `--ff-only` refuses or the rebase conflicts, stop and report. **Check, a minute or two later:**
<the live board's address and what it shows; Control Center lists the event>.

---

## 5. Phase 2 — the Tournament Hub

### 5.1 Start

Phase 1 must be on `main`. `git checkout main`, `git pull`, `git status --short` (<exact
output>), `git checkout -b <short>-hub`.

### 5.2 Copy the Hub and fill in its tokens

`cp _templates/<type>-template/index.html events/<event-key>/index.html`, then every token of
`_templates/CLAUDE.md` §3 for this template:

| Token | Replace with |
| --- | --- |
| `{{EVENT_KEY}}` | `<event-key>` |
| `{{EVENT_TITLE}}` | `<Event name>` |
| `{{EVENT_TAGLINE}}` | `<…>` |
| `{{EVENT_HEADLINE}}` | `<…>` (blank is fine) |
| `{{EVENT_DATE_RANGE}}` | `<Weekday, d Month YYYY>` |
| `{{VENUE}}` | `<…>` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr><SAGExEVENT>` |
| `{{EVENT_LOGO}}` | `assets/logo.<ext>` |

> **[Team]** and `{{CLUB_NAME}}` (blank takes no room).
> **[Dual meet]** and `{{CLUB_A_CODE}}`, `{{CLUB_B_CODE}}`, `{{CLUB_A_NAME}}`, `{{CLUB_B_NAME}}`,
> `{{CLUB_A_LOGO}}`, `{{CLUB_B_LOGO}}`.

A `&` in a name is `&amp;` here (`_templates/CLAUDE.md` §3). **Check:** `grep -n '{{'
events/<event-key>/index.html` prints nothing.

### 5.3 The Hub's content

*The edits to the header's text and order, the logo's alt text, the footer and the search
placeholders, each with its anchor and replacement as they read **after** §5.2, and a check per
edit.*

### 5.4 The Hub's colours and font

*Only for a re-skin: the theme comment (anchor its first line and the first line after it that
closes it; give the line count the dry run measured), the `:root` colours, any event-only tokens,
the font link.*

### 5.5 The Hub's header styling

*Only for a re-skin: the rules added before `</style>`, every colour through a `var(--…)`.
Checks: no house colour left; the count of event-only tokens; `wc -l` of the finished file.*

### 5.6 Add the Hub to the house-colours test

*Only for a re-skin:* `PAGES`, and **[Team]** `TEAM_HUBS` too. The unit tests must pass.

### 5.7 Commit the site's Phase 2

```bash
git add events/<event-key>/index.html events/<event-key>/assets/logo.<ext> _tests/unit/house-colours.test.mjs
git add events/<event-key>/assets/qr.png 2>/dev/null; true
git status --short
git commit -m "<Event name>: Tournament Hub<, re-skinned from the pubmat>"
```

`git status` before the commit: <the exact lines>. **Never stage the pubmat.**

### 5.8 Check the Hub in a browser

Build the fixture again (Appendix A), serve the site, and open
`http://localhost:8000/events/<event-key>/?fixture=<state>`.

At **375px**, **1024px** and **1280px**:

- [ ] <The header, top to bottom, exactly as it reads.>
- [ ] <At 375px: what fits on one line; no horizontal scroll
      (`document.documentElement.scrollWidth === innerWidth`).>
- [ ] At 1024px and 1280px: the QR panel at the top right shows `tinyurl.com/<SAGExEVENT>` and
      **covers none of the header's text**.
- [ ] <The tabs; the day picker or its absence; the stat numbers.>
- [ ] <The footer, line by line.>
- [ ] <Match Finder: a search, and the matches it finds.>
- [ ] <Standings at 1280px and at 375px.>
- [ ] Developer tools: no console errors; the only failed request is <`assets/qr.png`, if not
      supplied>.

Stop the server and remove the fixture as at the end of §4.8.

### 5.9 Full check, then stop and ask: push Phase 2

`npm run verify` must pass. Tell the owner Phase 2 is ready, with §5.8's results and whether
`qr.png` was included, and that pushing publishes the Hub at the short link. On a yes:
`git checkout main`, `git pull`, `git merge --ff-only <short>-hub`, `git push`. **Check** the live
Hub a minute or two later.

---

## 6. Phase 3 — <the desk page, the scorer page,> the dry-run checklist and the QR image

### 6.1 Start

Phase 2 must be on `main`. <Exact `git status --short`.> `git checkout -b <short>-rest`.

### 6.2 Copy the files and fill in their tokens

```bash
cp _templates/attendance/attendance.html events/<event-key>/attendance.html
cp _templates/scorer/scorer.html events/<event-key>/scorer.html
cp _templates/dry-run-checklist-template.md events/<event-key>/dry-run-checklist.md
```

*Keep only the pages the event's `attendance` and `scoreEntry` call for.* Tokens:
`{{EVENT_KEY}}` and `{{EVENT_TITLE}}`. <The staff pages keep the house colours (§9) / take the
palette: the same `:root` edit as §5.4(b).> <Any event-specific line for the checklist.>
**Check:** `grep -rnI '{{' events/<event-key>/` prints nothing.

### 6.3 Check the staff pages

- [ ] <The desk page: its tab title, "Ask the operator for a desk link.", its colours.>
- [ ] <The scorer page, likewise.>

### 6.4 Commit, full check, then stop and ask: push Phase 3

*Commit* (`"<Event name>: <desk page, scorer page> and dry-run checklist"`), including `qr.png`
if it is in; `npm run verify`; ask; on a yes, fast-forward `main` and push. If `qr.png` still
hasn't come, say so: it is committed and pushed on its own when it does.

---

## 7. Report back, at the end of each phase

- Which checks passed, and any that didn't, with what you saw.
- That the fixture was removed.
- Whether `qr.png` was present.
- What was pushed, and to which repo, only after the owner's yes.
- `git status --short` in every repo you touched.

---

## 8. Operator tasks — people, not the implementer

The workbook was read on <date>. These happen in Google Sheets and on the day.

### 8.1 Workbook checks before sync setup

*What the read found and what is left: the generator version (SAGE → Help); rosters; `Court
Control`; back-to-back matches and players in two categories (the organiser's call); anything the
board's colours or labels depend on.*

> **[Team]** The `Teams` roster's names, pair numbers and captains; `MatchUps`; the lineup rows.

### 8.2 Share the workbook, then set up live sync — once, after Phase 1 is pushed

1. Share it with `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**;
   set **General access** to **Restricted**.
2. **Extensions → Apps Script**: confirm `sheets-sync.gs` is there and current.
3. **SAGE → Set up live sync**: Day `<day-key>`, Venue `<Facility>` (exact), the default tabs.
   The secret only if asked, from whoever manages Cloud Run, never pasted into chat.
4. **Confirm:** a minute later `https://sage-match-control.github.io/event-data/<event-key>/data/<day-key>.json`
   lists `<Facility>` with <n> matches.

### 8.3 Go-live

<The Hub goes live 4 hours before the first serve: <time>. Control Center's override changes it.>

### 8.4 The QR panel

After `qr.png` is pushed: `node _templates/hub-pubmat/render.mjs <event-key>`, print
`_templates/hub-pubmat/out/<event-key>/qr-panel.pdf`, and scan it before mounting it.

### 8.5 Dry run, by <date>

`events/<event-key>/dry-run-checklist.md` against the real workbook, plus <this event's own
checks: anything new it is the first to use>.

### 8.6 After the event

Set `LIVE_BASE_URL = ''` in every page of `events/<event-key>/`. Leave the event in `events.json`.
Move this spec to `implemented/`.

---

## 9. Questions, answered by the owner on <date>

| Question | Answer |
| --- | --- |
| The short link | <…> |
| Attendance and score entry | <…> |
| The labels | <…> |
| The logo | <…> |
| The staff pages' colours | <…> |
| Build order | <…> |

---

## 10. Decisions, for reference

- **Event key `<event-key>`, day key `<day-key>`**: <why>.
- **`events.json` ships with the board**: registering early lets the operators set up live sync
  and rehearse on the real board while the Hub is built.
- <Every other choice a later reader might question, with its reason.>

---

## Appendix A — the fixture script

Used in §4.8 and §5.8, and never committed. Save it **outside the repo** and run it from the site
repo root. It writes `_fixtures/<event-key>/<state>.json` and adds the event to
`_fixtures/config.json`; the end of §4.8 says how to remove both.

```js
// <What the fixture is shaped like and what it writes. How to undo it.>
// usage, from the site repo root: node <path>/make-<short>-fixture.mjs
```

*Start from the precedent's script and change its shape, never its file format:*

> **[Standard]** The UPAP spec's Appendix A: the `SCHEDULE` slot by slot, `CATS`, fixed scores,
> playoff seats from the round-robin order, `matchesCsv` and `standingsCsv` with the workbook's
> columns.

> **[Team]** The Ladies' Dink spec's Appendix A: `pre`, `mid` and `finished`, a `rosterCsv`, and
> each matchup's games sharing a `matchUp`.

> **[Dual meet]** No engine-era precedent yet: start from the standard script, prefix every code
> with its club code, and add `display.clubs` to the fixture's config entry. Check every column
> against `_templates/CLAUDE.md` §4 and §5 before trusting it. This block is the thinnest in the
> template: the one dual meet so far, Pickle & Friends × 1Bataan United Picklers, was built before
> the engine, so the first engine-era dual meet's spec checks every step of it.

It must print one line, quoted in §4.8, that names the counts and every podium, so a changed
fixture shows at once.
~~~~
