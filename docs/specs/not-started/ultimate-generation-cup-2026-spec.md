# Spec — The Ultimate Generation Cup 2026

> **Status: not started.** Written 2026-11-23 against `sage-match-control.github.io` at
> `d92ed28`, `event-data`'s `config/events.json` at `e0eb51f5`, and both event workbooks as read
> that day through the Sheets API. The owner answered every open question the same day; the
> answers are in §10. Left to come: the short link and the QR image (§2).
>
> **Built in four phases** (§0.3): a **Day** picker on the schedule board (an engine change),
> then the event's board with its `events.json` entry, then the Tournament Hub, then everything
> else. Each phase is its own branch, its own checks and its own push.
>
> **Dry-run on 2026-11-23.** Every site edit of §4–§7 was applied to a detached worktree of the
> site at `d92ed28`: every anchor matched exactly once, and §4's engine edits, applied again to
> clean copies by their one-line anchors, gave byte-identical files. Appendix A's fixture was
> built (twice, the same output) and the board, the Hub, Control Center and the desk page were
> checked in a browser at 375px, 900px, 960px, 1024px, 1120px, 1280px and 1440px; the checks in
> §4.6, §5.8, §6.8 and §7.3 are what they showed. The site's unit tests: 889 before, 6 failing
> with §4.2's tests alone, then 904, 907 and 910 after Phases 1, 2 and 3. §5.6's `events.json`
> script was run on a copy of the file and printed what §5.6 expects. Not run in the dry run:
> `npm run verify`'s screenshot comparison, §4.7's documentation edits (their anchors were
> checked), and any commit or push.
>
> **Deadline.** The event is on **Saturday 28 and Sunday 29 November 2026**. The site repo takes
> no merge within 3 days of an event, so **all four phases reach the site's `main` by Tuesday
> 24 November**. After that, push only with the owner's explicit go-ahead.

Build the event pages for **The Ultimate Generation Cup 2026**, a two-day doubles tournament run
by the **Pampanga Paddle Aces** on **28–29 November 2026** at **Kingcourts Pickleball, San
Fernando, Pampanga**, and register the event so the live system knows it. Day 1 is the **Next
Gen** division (born 1981–2012), Day 2 the **Gen X** division (born 1960–1980).

The pages are copies of the site's standard event template, filled in for this event and
re-skinned from the organiser's pubmat: deep violet, hot pink and blue. **Phase 1 makes one
compatible engine change first:** the schedule board shows one day, fixed in its page, and this
is the first engine-era event with two; Phase 1 gives every board made from a template a **Day**
picker, as the Hub has, so nothing has to be pushed during the event.

---

## 0. Briefing — read all of it before you change anything

### 0.1 What you are working in

| Thing | Where | What it is |
| --- | --- | --- |
| **The site repo** | `D:\Personal\SAGE\sage-match-control.github.io` | A GitHub Pages site: hand-written HTML and native JavaScript modules, no build step, no framework. Pushing its `main` branch publishes it within a minute or two |
| **The engine** | `lib/v1/` in the site repo | Shared JavaScript and CSS that every event page loads. **Only Phase 1 edits it**, and only the files it names |
| **The template** | `_templates/standard-tournament-template/` (`index.html`, `schedule.html`) | The starting point for a standard event's pages: short HTML shells. Phase 1 adds one line to the three templates' `schedule.html`; after that **you copy them, never edit them** |
| **The `event-data` repo** | `D:\Personal\SAGE\event-data` | The shared data store. Its `config/events.json` registers every event. Its `main` also receives automatic sync commits, so always `git pull` right before you change it |
| **The docs repo** | `D:\Personal\SAGE\sage-docs` | The documentation. Phase 1 updates four pages there and this spec's status |
| **The workbooks** | Day 1: Google Sheet `1WQ2sksqI-xoRulvkCoUHYIhCxAmBDSbGnJhRbzGqU0k`; Day 2: `1fe5mzoPO2XoeBS54TZLjq3I8GZ0lIOS2Sss76O9RhY8` | Where the operators keep each day's schedule and scores. You don't touch them |
| **Control Center** | `tools/control-center.html` in the site repo | The operators' console. It lists every event in `events.json`. This spec doesn't change it |

Words this spec uses:

- **Team code**: how the workbook names a pair, `<CATEGORY>_<n>`, e.g. `LIMD_12`. Playoff seats
  are `<CATEGORY>_F_1` (Final, seat 1), `<CATEGORY>_SF_2`, `<CATEGORY>_QF_7`, `<CATEGORY>_R16_5`,
  `<CATEGORY>_B_1` (Bronze) and so on.
- **Category token**: the part before the first `_` (`LIMD`): a **division** (`LI`) then an
  **event** (`MD`).
- **Fixture mode**: on `localhost` only, `?fixture=<state>` makes an engine page read
  `/_fixtures/config.json` instead of the real `events.json`, and
  `/_fixtures/<event-key>/<state>.json` instead of the real published data, **whatever the day**.
  Appendix A writes one state per day: `day1-finished` and `day2-finished`. **The fixture is
  never committed.**
- **House colours**: the site's default palette (navy `#14263C`, green `#7CB92C`, off-white
  `#F6F7F2`). A **re-skin** replaces them in a page's `:root { … }` block, and
  `_tests/unit/house-colours.test.mjs` lists the re-skinned pages it checks.

### 0.2 Rules

- **Find edits by their anchor text, never by line number.** Every anchor is a single line and
  occurs exactly once in its file. If it isn't there, or is there more than once, **stop and
  report**. Don't guess and don't improvise a different edit.
- **Keep each file's line endings.** On disk in the site checkout: `lib/v1/apps/schedule-board.js`,
  `lib/v1/css/schedule-board.css`, the three templates' `schedule.html`,
  `_templates/attendance/attendance.html`, `_templates/CLAUDE.md` and
  `_tests/unit/house-colours.test.mjs` are **CRLF**; `_templates/standard-tournament-template/index.html`
  and `_templates/dry-run-checklist-template.md` are **LF**, so their copies start as LF. New files
  you create may be LF; Git stores LF either way. Where an edit replaces several lines, it names
  the first and the last line of the range and how many lines it spans. **Never use `sed -i`**:
  on this machine it strips the CR from every line. The scripted edits are Node scripts that keep
  each file's own line ending.
- **Replacement text is given in full** in each fenced block. Copy it exactly, including leading
  spaces. Never retype a value from memory.
- **Use Git Bash** for every command. Python is not installed; Node is.
- **Edit only what the phase names.** Phase 1: the engine and template files §4 lists, its two
  new test files, `_templates/CLAUDE.md` and the docs. Phases 2–4: files in
  `events/generation-cup-2026/`, the list in `_tests/unit/house-colours.test.mjs`, and
  `config/events.json` in `event-data`. Don't edit `tools/`, `_fixtures/` (except by running
  Appendix A, which you then undo) or any other event's folder. If something there seems to need
  a change, stop and report.
- **One phase at a time.** A phase starts only when the one before it is merged into `main` and
  pushed.
- **Commit** where a phase says *Commit*, with the message it gives, followed by a blank line and
  the attribution line your session's instructions give.
- **Never push, and never merge into `main`, without the owner's yes in chat.** Each phase ends at
  a **Stop and ask** step.
- **Never commit `assets/pubmat.jpg`.** It is reference only.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.3 The phases

| Phase | Builds | Branch | Ends with |
| --- | --- | --- | --- |
| **1** (§4) | The schedule board's **Day** picker: `lib/v1/domain/board-day.js` (new), `lib/v1/apps/schedule-board.js`, `lib/v1/css/schedule-board.css`, one line in each template's `schedule.html`, two test files, the docs | `gencup-engine` | Push of the site, then the docs |
| **2** (§5) | `events/generation-cup-2026/schedule.html` and the event's entry in `event-data/config/events.json` | `gencup-board` | Push of the site **and** `event-data` |
| **3** (§6) | `events/generation-cup-2026/index.html` and its logo | `gencup-hub` | Push of the site |
| **4** (§7) | `attendance.html`, the dry-run checklist, and the QR image if it has come | `gencup-rest` | Push of the site |

What is live in between:

- **After Phase 1** nothing any visitor sees changes: every existing board has no picker
  container and works as before (§4.6 checks UPAP's).
- **After Phase 2** the event is registered: Control Center lists it, the operators can set up
  both workbooks' live sync (§9.2), and **Open schedule** opens the board. **Open Tournament Hub**
  leads to "Page not found" until Phase 3. Don't hand out the short link before Phase 3.
- **After Phase 3** the public Hub is live. **Issue desk link** waits for Phase 4.
- Nothing is pushed during the event: the board opens on each event day by itself (§3.5).

### 0.4 Commands you will use in every phase

Unit tests (from the site repo root): `cd _tests && npm test; cd ..`

Full check (unit tests, then the comparison of `main` against your branch):
`cd _tests && npm run verify; cd ..`. If it says a Playwright browser is missing, run
`cd _tests && npx playwright install chromium; cd ..` once.

Serve the **repo root** on `localhost` for a browser check (Ctrl+C to stop):
`npx --yes http-server -p 8000 -c-1`

---

## 1. Event facts

From the pubmat and the two workbooks (`Title`, `Variables`, `SCHEDULE`, `CSV`, `STANDINGSCSV`,
`ATTENDANCE` tabs), read on 2026-11-23.

| Fact | Value |
| --- | --- |
| Organiser | Pampanga Paddle Aces |
| Event name | The Ultimate Generation Cup 2026 (the workbooks title each day "Paddle Aces The Ultimate Generation Cup DAY 1" / "DAY 2") |
| Dates | **Day 1: Saturday 28 November 2026, Next Gen** (born 1981–2012). **Day 2: Sunday 29 November 2026, Gen X** (born 1960–1980) |
| Venue | Kingcourts Pickleball, San Fernando, Pampanga. One facility each day; the workbooks' facility name is exactly **`Kingcourts`**. Day 1 uses **10 courts**, Day 2 **6** |
| First serve | Day 1 **9:00 AM**; Day 2 **10:00 AM** |
| Matches | Day 1: 166, numbered 1–166, 25-minute slots from 9:00 AM to 4:55 PM (20 slots). Day 2: 49, 25-minute slots from 10:00 AM to 3:25 PM (14 slots); **not yet numbered** (§9.1) |
| Event key | `generation-cup-2026` (the folder under `events/`, the key in `events.json`, the folder in `event-data`) |
| Day keys | `generation-cup-day1`, `generation-cup-day2` (unique across every event in `events.json`) |
| Short link | `tinyurl.com/SAGExGenCup` → `https://sage-match-control.github.io/events/generation-cup-2026/` |
| Generator | Both workbooks are from the Standard Tournament Master (generator 1.5.0, §9.1) |
| Not on the site | The entry fees, the prizes (medals, cash, T-shirt, food, gift items) and the raffle |

**The categories.** The pubmat advertises Beginner, Low Inter, High Inter and Advanced in each
division; the workbooks name them by level alone, and the owner keeps the workbooks' names
(§10). The division of the code is the level: `N` Novice, `LI` Low Intermediate, `HI` High
Intermediate, `A` Advanced, `O` Open. Which generation is the day's label, not the category's.

Day 1, Next Gen, 85 pairs:

| Code | The workbook's name | Pairs | Brackets (sizes) | Playoff |
| --- | --- | --- | --- | --- |
| `NMD` | Novice Men's Doubles | 13 | 4 (4, 3, 3, 3) | Quarterfinals (each bracket's top two, crossed), Semifinals, Bronze, Final |
| `NWD` | Novice Women's Doubles | 8 | 2 (4, 4) | Semifinals (top two, crossed), Bronze, Final |
| `NXD` | Novice Mixed Doubles | 12 | 3 (4, 4, 4) | The three bracket winners to the Semifinals; the best runner-up meets the winner of the 2nd- v 3rd-best runner-ups (`R16-3`) in `QF-2` for the fourth seat; Bronze, Final |
| `LIMD` | Low Intermediate Men's Doubles | 14 | 4 (4, 4, 3, 3) | As `NMD` |
| `LIWD` | Low Intermediate Women's Doubles | 9 | 2 (5, 4) | As `NWD` |
| `LIXD` | Low Intermediate Mixed Doubles | 9 | 2 (5, 4) | As `NWD` |
| `HIMD` | High Intermediate Men's Doubles | 8 | 2 (4, 4) | As `NWD` |
| `HIXD` | High Intermediate Mixed Doubles | 4 | 1 | Final, #1 v #2; #3 takes bronze without playing (`B_2` is a BYE) |
| `AMD` | Advanced Men's Doubles | 4 | 1 | As `HIXD` |
| `AXD` | Advanced Mixed Doubles | 4 | 1 | As `HIXD` |

Day 2, Gen X, 25 pairs:

| Code | The workbook's name | Pairs | Brackets (sizes) | Playoff |
| --- | --- | --- | --- | --- |
| `NMD` | Novice Men's Doubles | 4 | 1 | Final; #3 takes bronze without playing |
| `LIMD` | Low Intermediate Men's Doubles | 11 | 3 (4, 4, 3) | As Day 1's `NXD` |
| `LIXD` | Low Intermediate Mixed Doubles | 4 | 1 | As `NMD` |
| `AMD` | Advanced Men's Doubles | 3 | 1, **double round robin** (every pairing twice) | As `NMD` |
| `OWD` | Open Women's Doubles | 3 | 1, double round robin | As `NMD` |

Every match is one game. The shared engine already reads all of these playoffs, best runner-ups
and double round robins; nothing in this spec changes how they work.

**How each day runs.** Day 1 plays Novice and Low Intermediate men's and women's first (courts
1–10 from 9:00), then High Intermediate men's from 11:55, Mixed Doubles from 12:45, and the
Advanced and High Intermediate Mixed categories last; the last match, `AXD`'s Final, is at
4:55 PM on Court 6. Day 2 plays every category but Mixed from 10:00 on courts 1–6, then
`LIMD`'s ladder to 1:45 PM and `LIXD` on courts 4–5 from 2:10 PM.

---

## 2. Inputs

| Input | Status | Used in |
| --- | --- | --- |
| Logo | **Already in the site repo, untracked**, at `events/generation-cup-2026/assets/logo.jpg`: the round Pampanga Paddle Aces badge (blue-to-violet disc, green ball), 500×500, white corners. Use it as is. Committed in Phase 3 | §6 |
| Pubmat | **Already in the site repo, untracked**, at `events/generation-cup-2026/assets/pubmat.jpg` (1256×1792). **Reference only: never commit it** | — |
| QR image | **To follow from the owner** as `events/generation-cup-2026/assets/qr.png`. If it isn't there when a phase runs, carry on and report it missing. Don't generate one | §6 (if present), else §7 |
| Workbooks | Day 1 `1WQ2sksqI-xoRulvkCoUHYIhCxAmBDSbGnJhRbzGqU0k`, Day 2 `1fe5mzoPO2XoeBS54TZLjq3I8GZ0lIOS2Sss76O9RhY8` | §5.6 |

If `logo.jpg` or `pubmat.jpg` is missing when you start, stop and report.

---

## 3. Design notes (no action — the reasons behind the edits)

1. **The palette**, sampled from the pubmat, used by both pages:

   | Token | Value | Role, from |
   | --- | --- | --- |
   | `--navy` | `#2A1052` | structure (headers, buttons, the board's bar): the pubmat's purple, deepened |
   | `--navy-deep` | `#170830` | shadows |
   | `--green` | `#FF4FA3` | the accent fill (active tab, live ring): the pubmat's hot pink, lightened so navy text reads on it |
   | `--green-dark` | `#B0125F` | the accent as text |
   | `--paper` | `#F7F5FA` | page background, a cool off-white |
   | `--paper-dim` | `#EEEAF4` | panels |
   | `--line` | `#DCD5E6` | rules and borders |
   | `--ink` | `#1E1030` | body text |
   | `--ink-soft` | `#5E5470` | secondary text |
   | `--white` | `#FFFFFF` | unchanged |

   The token names stay `--navy` and `--green`: the engine's CSS reads them. Measured contrast
   (WCAG): white on `--navy` 16.2:1; `--navy` on `--green` 5.3:1; `--ink` on `--paper` 16.6:1;
   `--ink-soft` on `--paper` 6.5:1; `--green-dark` on white 6.8:1. `--green` on white is 3.0:1,
   so it is a fill only, never text on white.
2. **The Hub's hero colours**, its own tokens: `--gc-pink` `#E8197D` and `--gc-blue` `#1E6FE0`
   (the pubmat's brush strokes, as translucent bands), `--gc-violet` `#B98CFF` ("2026", 6.4:1 on
   `--navy`), and the two generation chips, white on `--gc-blue-deep` `#1A5FC8` (6.0:1) and on
   `--gc-pink-deep` `#C2126A` (5.9:1). "Ultimate" is `--green` on `--navy`, 5.3:1.
3. **The font.** Anton (Google Fonts) is the closest match to the pubmat's heavy condensed
   capitals. It is used only for the Hub's title and "2026", and the board's title.
4. **The board's colours are the workbooks' where they have them.** The generator fills each
   category's round-robin cells by level and event (Men's blue, Women's magenta, Mixed orange,
   lighter for a lower level); read off both `SCHEDULE` tabs through the Sheets API: `NMD`
   `#C9DAF8`, `NWD` `#EAD1DC`, `NXD` `#FCE5CD`, `LIMD` `#6D9EEB`, `LIWD` `#C27BA0`, `LIXD`
   `#F6B26B`, `HIMD` `#1155CC`, `HIXD` `#B45F06`. The workbooks leave `AMD`, `AXD` and `OWD`
   grey (`#CCCCCC`), so those take the generator's own Advanced shades (`#1C4587`, `#783F04`)
   and `OWD` its darkest magenta (`#4C1130`): Open sits above Advanced. The same code has the
   same colour on both days.
5. **The board's Day picker** (Phase 1). The board shows one day, its page's `DAY_KEY`. A page
   whose markup has the new container `<div class="dayfilter" id="dayfilter" hidden></div>`
   reads the event's days from `events.json` and, with two or more, shows a **Day** select in
   the bar, beside the venue and court filters. The board opens on `?day=` when it names one of
   the event's days, else on the day whose date is today, else on `DAY_KEY`. So Control
   Center's **Open schedule**, which opens the board without `?day=`, shows Day 1 on 28 November
   and Day 2 on 29 November with nobody touching it, and Day 1 on any other date. Choosing a day
   reloads the board with `?day=` and drops `?courts=` and `?venue=`, since Day 1 has 10 courts
   and Day 2 has 6. The rule is `lib/v1/domain/board-day.js`, a new pure module with its own
   tests. A board without the container (every existing one) reads no more than before and is
   unchanged; the compact header hides the picker, as it hides the court filter. Control Center
   itself is not changed.
6. **The labels** (owner's choice, §10): divisions Novice, Low Intermediate, High Intermediate,
   Advanced, Open; the engine shows `<division> <event>`, e.g. "Low Intermediate Men's Doubles".
   It finds the division by the **longest** matching prefix, so `HI` wins over nothing shorter,
   `N` only matches `N…` and `O` only `OWD`. The generation is in each day's label:
   **"Nov 28 · Next Gen"** and **"Nov 29 · Gen X"**, which the Hub's and the board's day pickers,
   the board's meta line and Control Center all show.
7. **The Hub's header**, top to bottom: the two logos, "Pampanga Paddle Aces", "The **Ultimate**
   Generation Cup" (two lines, Anton capitals), "2026" in violet, two chips saying which
   generation plays which day, then the dates and the venue. The QR panel was measured from 900
   to 1280px wide and covers none of it.

---

## 4. Phase 1 — the schedule board's Day picker

**Why:** this is a two-day event, and the board shows exactly one day, fixed by `DAY_KEY` in its
page; without a change, the board would have to be pushed on the evening of 28 November, inside
the freeze. Every change here is **compatible** (`_templates/CLAUDE.md` §8): it adds, and never
removes or renames, an export, a CSS class a page uses or a theme property, so every other
event's page keeps working when this is pushed. `schedule-board.js` uses an export of
`domain/board-day.js` in the same push, which is allowed because that module is new.

### 4.1 Start

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
```

This must print exactly:

```
?? events/archives/ladies-dink-2026/assets/pubmat.webp
?? events/generation-cup-2026/
?? events/upap-2026/assets/pubmat.jpg
```

(the first and last are unrelated untracked files; leave them alone). Anything else: stop and
report.

```bash
git checkout -b gencup-engine
cd _tests && npm install && npm test; cd ..
```

`npm test` must pass, **889 tests**, before you change anything.

### 4.2 Tests first

Create `_tests/unit/domain/board-day.test.mjs` with exactly:

```js
// domain/board-day.js: which day a schedule board shows when its event has more than one
// (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §4).
import test from 'node:test';
import assert from 'node:assert/strict';
import { boardDay, localIsoDate } from '../../../lib/v1/domain/board-day.js';

const DAYS = [
  { key: 'generation-cup-day1', label: 'Nov 28 · Next Gen', date: '2026-11-28' },
  { key: 'generation-cup-day2', label: 'Nov 29 · Gen X', date: '2026-11-29' },
];

test('a ?day= that is one of the event\'s days wins over everything', () => {
  assert.equal(boardDay({ days: DAYS, dayKey: 'generation-cup-day1', requested: 'generation-cup-day2', today: '2026-11-28' }), 'generation-cup-day2');
});

test('a ?day= the event does not have is ignored', () => {
  assert.equal(boardDay({ days: DAYS, dayKey: 'generation-cup-day1', requested: 'upap-day1', today: '2026-11-20' }), 'generation-cup-day1');
});

test('without ?day=, the day whose date is today', () => {
  assert.equal(boardDay({ days: DAYS, dayKey: 'generation-cup-day1', requested: null, today: '2026-11-29' }), 'generation-cup-day2');
});

test('without ?day= and on no event day, the page\'s own day', () => {
  assert.equal(boardDay({ days: DAYS, dayKey: 'generation-cup-day1', requested: null, today: '2026-11-23' }), 'generation-cup-day1');
});

test('with no days read (events.json failed), the page\'s own day', () => {
  assert.equal(boardDay({ days: [], dayKey: 'generation-cup-day1', requested: 'generation-cup-day2', today: '2026-11-29' }), 'generation-cup-day1');
});

test('a day without a date is never today', () => {
  assert.equal(boardDay({ days: [{ key: 'x' }, { key: 'y' }], dayKey: 'x', requested: '', today: '2026-11-29' }), 'x');
});

test('localIsoDate reads the local calendar date, zero-padded', () => {
  assert.equal(localIsoDate(new Date(2026, 10, 9, 23, 59)), '2026-11-09');
  assert.equal(localIsoDate(new Date(2026, 0, 1, 0, 0)), '2026-01-01');
});
```

Create `_tests/unit/apps/board-day-picker.test.mjs` with exactly:

```js
// The schedule board's day picker (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §4): every
// board template has its container, so a multi-day event made from a template gets the picker; the board fills
// it from events.json and leaves it hidden at a one-day event. A board without the container is unchanged.
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import path from 'node:path';
import { SITE_ROOT } from '../../helpers/paths.mjs';

const read = rel => fs.readFileSync(path.join(SITE_ROOT, rel), 'utf8');
const CONTAINER = '<div class="dayfilter" id="dayfilter" hidden></div>';

for (const rel of [
  '_templates/standard-tournament-template/schedule.html',
  '_templates/dual-meet-template/schedule.html',
  '_templates/team-tournament-template/schedule.html',
]) {
  test(`${rel} has the day picker's container, hidden, before the venue filter`, () => {
    const html = read(rel);
    const at = html.indexOf(CONTAINER);
    assert.ok(at > -1, 'the container is there');
    assert.equal(html.indexOf(CONTAINER, at + 1), -1, 'once');
    assert.ok(at < html.indexOf('id="venuefilter"'), 'before the venue filter');
  });
}

test('the board reads the day from domain/board-day.js, only for a page with the container', () => {
  const js = read('lib/v1/apps/schedule-board.js');
  assert.match(js, /import \{ boardDay, localIsoDate \} from '\.\.\/domain\/board-day\.js';/);
  assert.match(js, /const dayFilterEl = byId\("dayfilter"\);/);
  assert.match(js, /if\(teamEvent \|\| dayFilterEl\)\{/);
  // Every read of the day follows the picker's choice, not the page's DAY_KEY.
  assert.doesNotMatch(js, /fetchDaySnapshot\(eventKey, dayKey/);
  assert.doesNotMatch(js, /follow\(eventKey, dayKey\)/);
  assert.doesNotMatch(js, /day: dayKey/);
});

test('the board stylesheet hides an empty day picker and in the compact header', () => {
  const css = read('lib/v1/css/schedule-board.css');
  assert.match(css, /\.dayfilter\[hidden\]\{ display:none; \}/);
  assert.match(css, /\.bezel\.compact \.dayfilter,/);
});
```

Run the unit tests (§0.4). They must report **901 tests, 6 failing**, and only these:

```
not ok - _templates/standard-tournament-template/schedule.html has the day picker's container, hidden, before the venue filter
not ok - _templates/dual-meet-template/schedule.html has the day picker's container, hidden, before the venue filter
not ok - _templates/team-tournament-template/schedule.html has the day picker's container, hidden, before the venue filter
not ok - the board reads the day from domain/board-day.js, only for a page with the container
not ok - the board stylesheet hides an empty day picker and in the compact header
not ok - unit\domain\board-day.test.mjs
```

(the last is the whole file failing, since `board-day.js` doesn't exist yet). Any other failure:
stop and report.

### 4.3 The code

**(a) The new module.** Create `lib/v1/domain/board-day.js` with exactly:

```js
// Which day a schedule board shows, when its page offers a day picker and its event has more than one day
// (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §4). In order:
//
//   1. the page's ?day=, when it is one of the event's day keys (the picker writes it);
//   2. the event's day whose date is today, so a wall display left on overnight shows the next day's board;
//   3. the page's own DAY_KEY.
//
// With no days read (events.json failed to load), always the page's own DAY_KEY.

/**
 * @param {{ days: { key: string, date?: string }[], dayKey: string, requested?: string|null, today: string }} args
 *   days       the event's days, from events.json (registry.js's eventConfigFrom)
 *   dayKey     the page's own DAY_KEY
 *   requested  the page's ?day=, or null
 *   today      the local date, YYYY-MM-DD (localIsoDate)
 * @returns {string} a day key
 */
export function boardDay({ days, dayKey, requested = null, today }){
  if(!Array.isArray(days) || days.length === 0) return dayKey;
  if(requested && days.some(d => d.key === requested)) return requested;
  const dated = days.find(d => d.date && d.date === today);
  return dated ? dated.key : dayKey;
}

/**
 * A date's local calendar day as YYYY-MM-DD, the form events.json writes a day's `date` in.
 * @param {Date} date
 * @returns {string}
 */
export function localIsoDate(date){
  const pad = n => String(n).padStart(2, '0');
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())}`;
}
```

**(b) The board, `lib/v1/apps/schedule-board.js`** (CRLF, 535 lines before). Eight edits.

(b1) Replace the two lines from

```js
// Center's Mission Control opens it. Shows one day of an event, reads the same published snapshot the
```

through

```js
// Hub does, and needs nothing from events.json (the court count comes from the data).
```

with these three:

```js
// Center's Mission Control opens it. Shows one day of an event, reads the same published snapshot the
// Hub does, and needs nothing from events.json (the court count comes from the data), except for a team
// event's labels and, on a page whose markup has #dayfilter, the event's days for its day picker.
```

(b2) Replace the line

```js
//     dayKey           required. The one day the board shows (a day key from events.json)
```

with these two:

```js
//     dayKey           required. The day the board shows (a day key from events.json); with a day picker, the
//                      day it shows when neither ?day= nor today's date names another
```

(b3) Find the line

```js
import { readableOn, courtsToParam, chunkBalanced, clubOf, parseCourtsParam as parseCourts, buildScheduleData as buildData } from '../domain/schedule-grid.js';
```

and replace it with itself followed by:

```js
import { boardDay, localIsoDate } from '../domain/board-day.js';
```

(b4) Find the line that starts `  let pairs, teamColors, seedLabels, lineupReveal;   // a team event's pair labels`
(it is one long line ending `read from events.json before the first load (below)`) and add these
two lines after it:

```js
  // The day this board shows: the page's DAY_KEY, unless its day picker (below) chooses another.
  let shownDay = dayKey;
```

(b5) Replace the line

```js
    return Snapshot.fetchDaySnapshot(eventKey, dayKey, { liveChannel, fixture: FIXTURE });
```

with:

```js
    return Snapshot.fetchDaySnapshot(eventKey, shownDay, { liveChannel, fixture: FIXTURE });
```

(b6) Replace the six lines from

```js
  if(teamEvent){
```

through

```js
  liveChannel.follow(eventKey, dayKey);
```

(the team event's `events.json` read, a blank line, and the live channel's `follow`) with:

```js
  // A page whose markup has #dayfilter also reads the event's days there, for its day picker; a board without
  // one reads no more than it did before the picker existed.
  const dayFilterEl = byId("dayfilter");
  if(teamEvent || dayFilterEl){
    try {
      const config = eventConfigFrom(await loadRegistry({ fixture: FIXTURE }), eventKey) || {};
      if(teamEvent) ({ pairs, teamColors, seedLabels, lineupReveal } = config);
      if(dayFilterEl) renderDayFilter(dayFilterEl, config.days || []);
    }
    catch(err){ console.warn(`mountScheduleBoard: couldn't read events.json (${err.message}); using the default pair labels and the page's own day`); }
  }

  // ---------- DAY PICKER (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §4) ----------
  // Shown when the event has two days or more. The day is ?day=, else today's, else DAY_KEY (domain/board-day.js).
  // Choosing one reloads the board on it: another day has its own courts and venues, so ?courts= and ?venue= go,
  // and everything that follows a day (the live channel, the staff view) starts afresh.
  function renderDayFilter(el, days){
    shownDay = boardDay({ days, dayKey, requested: new URLSearchParams(location.search).get("day"), today: localIsoDate(new Date()) });
    if(days.length < 2) return;
    el.innerHTML = `<label class="cf-label" for="dayPicker">Day</label><select class="day-select" id="dayPicker">` +
      days.map(d => `<option value="${esc(d.key)}"${d.key === shownDay ? " selected" : ""}>${esc(d.label || d.key)}</option>`).join("") +
      `</select>`;
    el.hidden = false;
    byId("dayPicker").addEventListener("change", e => {
      const url = new URL(location.href);
      url.searchParams.set("day", e.target.value);
      url.searchParams.delete("courts");
      url.searchParams.delete("venue");
      location.assign(url);
    });
  }

  liveChannel.follow(eventKey, shownDay);
```

(b7) Replace the line

```js
      apiBase: CLOUD_RUN_BASE_URL, getToken: () => staffToken, event: eventKey, day: dayKey,
```

with:

```js
      apiBase: CLOUD_RUN_BASE_URL, getToken: () => staffToken, event: eventKey, day: shownDay,
```

(b8) **Check:** the file is now 567 lines, every one CRLF:

```bash
node -e "const t=require('fs').readFileSync('lib/v1/apps/schedule-board.js','utf8');console.log(t.split('\n').length-1,(t.match(/\r\n/g)||[]).length)"
```

Prints `567 567`.

**(c) The stylesheet, `lib/v1/css/schedule-board.css`** (CRLF, 653 lines before). Three edits.

(c1) Replace the line

```css
.bezel.compact .venuefilter,
```

with these two:

```css
.bezel.compact .dayfilter,
.bezel.compact .venuefilter,
```

(c2) Find the line

```css
.venuefilter .cf-btn{ padding:2px 9px; }
```

and replace it with itself followed by:

```css

/* ---------- DAY PICKER ----------
   Rendered only on a page whose markup has #dayfilter, and only when the event has two days or more
   (apps/schedule-board.js). Choosing a day reloads the board on it, by ?day=. Bordered like the court
   filter's buttons; the chevron is white for the navy bar. */
.dayfilter{
  display:flex; align-items:center; gap:4px;
  margin-left:18px; padding-left:16px;
  border-left:1px solid rgba(255,255,255,.22);
}

.dayfilter[hidden]{ display:none; }

.day-select{
  font-family:'Barlow Condensed',sans-serif;
  font-weight:700; font-size:12px; letter-spacing:.04em;
  padding:2px 24px 2px 9px;
  border-radius:5px; cursor:pointer;
  border:1px solid rgba(255,255,255,.28);
  background:transparent url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16"><path d="M4 6l4 4 4-4" stroke="%23FFFFFF" stroke-width="1.6" fill="none" stroke-linecap="round" stroke-linejoin="round"/></svg>') no-repeat right 7px center;
  background-size:10px;
  color:var(--white);
  appearance:none; -webkit-appearance:none; -moz-appearance:none;
}

.day-select:hover, .day-select:focus{ border-color:var(--green); outline:none; }

.day-select option{ color:var(--ink); background:var(--white); }
```

(the first line of that block is empty).

(c3) In the `@media screen and (max-width: 980px)` block, replace the line (two leading spaces)

```css
  .venuefilter,
```

with these two:

```css
  .dayfilter,
  .venuefilter,
```

The new rules use no house-colour literal (only white tints and `var(…)`), so no page has to
restate them for `house-colours.test.mjs`. **Check:** the same one-liner as (b8) on
`lib/v1/css/schedule-board.css` prints `683 683`.

**(d) The templates.** Each of the three templates' `schedule.html` gains the container, on its
own line just above the venue filter's. Save this **outside the repo** as `add-dayfilter.mjs`
and run it from the site repo root:

```js
// Adds the schedule board's day-picker container to each board template, on its own line just above the
// venue filter's, keeping each file's own line endings. Refuses to run twice.
// usage, from the site repo root: node <path>/add-dayfilter.mjs
import fs from 'node:fs';

const FILES = [
  '_templates/standard-tournament-template/schedule.html',
  '_templates/dual-meet-template/schedule.html',
  '_templates/team-tournament-template/schedule.html',
];
const ANCHOR = '  <div class="venuefilter" id="venuefilter" hidden></div>';
const ADDED = '  <div class="dayfilter" id="dayfilter" hidden></div>';

for (const file of FILES) {
  const text = fs.readFileSync(file, 'utf8');
  const eol = text.includes('\r\n') ? '\r\n' : '\n';
  const lines = text.split(eol);
  if (lines.includes(ADDED)) throw new Error(`${file} already has the day picker: stop and report`);
  const hits = lines.filter(l => l === ANCHOR).length;
  if (hits !== 1) throw new Error(`${file}: the venue filter's line occurs ${hits} times, not once: stop and report`);
  lines.splice(lines.indexOf(ANCHOR), 0, ADDED);
  fs.writeFileSync(file, lines.join(eol));
  console.log(`${file}: ${eol === '\r\n' ? 'CRLF' : 'LF'}, ${lines.length - 1} lines`);
}
```

It must print:

```
_templates/standard-tournament-template/schedule.html: CRLF, 159 lines
_templates/dual-meet-template/schedule.html: CRLF, 163 lines
_templates/team-tournament-template/schedule.html: CRLF, 146 lines
```

Run the unit tests (§0.4). They must pass, **904 tests** (the 15 new ones include the guard
tests every new `lib/v1` file gets). `git diff --stat` must show the three templates with 1 line
added each, `lib/v1/apps/schedule-board.js` with 40 added and 8 removed, and
`lib/v1/css/schedule-board.css` with 30 added.

### 4.4 `_templates/CLAUDE.md`

In `_templates/CLAUDE.md` (CRLF), replace the three lines from

```markdown
   - **`{{SCHEDULE_DAY_KEY}}`** — the board shows exactly one day. Set this
```

through

```markdown
     it at whichever day is being played; there is no day picker on the board.
```

with these five:

```markdown
   - **`{{SCHEDULE_DAY_KEY}}`** — the day the board shows. Set this to that
     day's key in `events.json` (step 7); for a multi-day event, the first
     day. At an event with two days or more the board's bar has a **Day**
     picker, filled from `events.json`, and on each event day the board opens
     on that day by its date (`lib/v1/domain/board-day.js`).
```

### 4.5 Commit

```bash
git add lib/v1/domain/board-day.js lib/v1/apps/schedule-board.js lib/v1/css/schedule-board.css _templates/standard-tournament-template/schedule.html _templates/dual-meet-template/schedule.html _templates/team-tournament-template/schedule.html _templates/CLAUDE.md _tests/unit/domain/board-day.test.mjs _tests/unit/apps/board-day-picker.test.mjs
git status --short
git commit -m "Schedule board: a Day picker for an event with more than one day"
```

`git status` before the commit: those nine files staged (`A ` for the three new ones, `M ` for
the rest), the three `??` lines of §4.1, and nothing else.

### 4.6 Check in a browser

The event's own board is Phase 2, so check the picker on a scratch copy of the template.
`*beta*` files are git-ignored in this repo. From the site repo root:

```bash
node -e "const fs=require('fs');let t=fs.readFileSync('_templates/standard-tournament-template/schedule.html','utf8');t=t.split('{{EVENT_KEY}}').join('generation-cup-2026').split('{{EVENT_TITLE}}').join('Generation Cup (scratch)').split('{{SCHEDULE_DAY_KEY}}').join('generation-cup-day1');fs.writeFileSync('events/generation-cup-2026/schedule-beta.html',t)"
node <path-to>/make-gencup-fixture.mjs
npx --yes http-server -p 8000 -c-1
```

The fixture script (Appendix A) prints the two lines quoted in §5.8. At **1440px**, open
`http://localhost:8000/events/generation-cup-2026/schedule-beta.html?fixture=day1-finished&courts=1-3`:

- [ ] The bar has a **DAY** label and a select reading "Nov 28 · Next Gen" between the meta line
      and the **COURTS** filter.
- [ ] Choosing "Nov 29 · Gen X" reloads the page at
      `…/schedule-beta.html?fixture=day1-finished&day=generation-cup-day2`: `courts=1-3` is gone
      and the select reads "Nov 29 · Gen X". (In fixture mode the data stays the `day1-finished`
      state, since the fixture's file doesn't follow the day.)
- [ ] The chevron at the right collapses the bar and the Day picker goes with it; the address
      gains `&compact=1` and keeps `day=`.
- [ ] At **375px** the picker sits on its own row under the meta line, and
      `document.documentElement.scrollWidth === innerWidth` is `true` in the console.

Then an existing board, with **real data**: `http://localhost:8000/events/upap-2026/schedule.html`.

- [ ] It reads "NOV 22 · 65 MATCHES · 17 SLOTS · ALL COURTS" with no Day picker, and in the
      console `performance.getEntriesByType('resource').filter(e => /events\.json/.test(e.name)).length`
      is `0`: a board without the container reads no `events.json`.

Stop the server and remove the scratch page and the fixture:

```bash
rm events/generation-cup-2026/schedule-beta.html
git checkout -- _fixtures/config.json
rm -rf _fixtures/generation-cup-2026
git status --short
```

`git status` prints the three `??` lines of §4.1 and nothing else.

### 4.7 Documentation

In `D:\Personal\SAGE\sage-docs` (`git checkout main`, `git pull`, `git status --short` prints
nothing; work on `main`, as that repo is worked), each page in the present tense:

1. `docs/features/schedule-board.md` (CRLF): replace the two lines from
   `The board shows exactly one day. The event's page is set to that day, and the` through
   `colour key for the categories is set by hand from the scoring workbook's own` with:

   ```markdown
   The board shows one day at a time. At an event with more than one day, a
   **Day** picker in the bar switches between them, and on each event day the
   board opens on that day by itself; on any other date it opens on the day its
   page is set to. The colour key for the categories is set by hand from the
   scoring workbook's own
   ```

2. `docs/technical/schedule-board.md` (CRLF): find the line ``## Venue filter — `?venue=` ``
   and insert before it:

   ```markdown
   ## Day picker — `?day=`

   A page whose markup has `<div class="dayfilter" id="dayfilter" hidden></div>`
   (every template's, before the venue filter) reads the event's days from
   `events.json` and, with two or more, fills it with a **Day** select. The day
   shown is `?day=` when it names one of the event's days, else the day whose
   `date` is today in the browser's time zone, else the page's `DAY_KEY`
   (`lib/v1/domain/board-day.js`, `boardDay`); with `events.json` unreadable it
   is `DAY_KEY`. Choosing a day reloads the page with `?day=` set and `?courts=`
   and `?venue=` dropped, so the snapshot, the live channel and the staff view
   all start on the new day. A page without the container reads no `events.json`
   (unless it is a team event's) and shows `DAY_KEY` only. The compact header
   hides the picker. In fixture mode the data is the fixture state's whatever
   the day.

   ```

3. `docs/technical/site-engine.md`: find the line ``event's `display.pairs` from `events.json`.``
   and replace it with:

   ```markdown
   event's `display.pairs` from `events.json`. A board whose page has the
   `#dayfilter` container reads the event's days from `events.json` too, for its
   **Day** picker; which day it shows is `domain/board-day.js`
   ([schedule board](schedule-board.md#day-picker-day)).
   ```

4. `docs/specs/event-spec-template.md`:
   - Replace the line (two leading spaces: it is in §1.1's list)

     ```markdown
       | With more than one day: how the board changes day (§1.3) | An engine phase, or a push every evening of the event |
     ```

     with:

     ```markdown
       | With more than one day: what each day is called (a division, a wave, a generation) | Each day's `label` in `events.json` |
     ```
   - Replace the three lines from
     `The engine handles more (the board's **Venue** row, the Hub's day picker), but no event has used`
     through `both came before the engine. An event with more of either changes these parts of the spec (the`
     with:

     ```markdown
     The engine handles more (the board's **Venue** row and **Day** picker, the Hub's day picker).
     Pickle for Sight (one day, two facilities) and BKL Cup 2026 (five days × three facilities) came
     before the engine; The Ultimate Generation Cup 2026 is the first engine-era event with two days.
     An event with more of either changes these parts of the spec (the
     ```
   - Replace the sixteen lines from
     `**Several days.** One `days` entry per day in §4.6, each day key unique across the whole file`
     through `Each day's go-live, dry run (§8.5) and fixture state are then per day.` with:

     ```markdown
     **Several days.** One `days` entry per day in §4.6, each day key unique across the whole file
     (`<event-key>-day1`, `-day2`, …), each with its own `isLive` and its own workbooks. The Hub's day
     picker comes from `events.json`, and so does the board's: a board made from a template shows a
     **Day** picker at an event with two days or more, opens on the day whose date is today, and
     otherwise on its `DAY_KEY`, which the spec sets to the first day. Nothing is pushed during the
     event. Each day's go-live, dry run (§8.5) and fixture state are per day; fixture mode reads
     `/_fixtures/<event-key>/<state>.json` whatever the day, so the fixture writes one state per day
     (`day1-finished`, `day2-finished`).
     ```

5. **This spec moves to `in-progress/`** by the procedure in `docs/specs/README.md` (its status
   line, the specs README, the two folder READMEs, `mkdocs.yml`); its status line names Phase 1's
   commit.
6. A MINOR docs version: a `docs/changelog.md` entry (**Changed**: the schedule board's Day
   picker, naming the four pages; this spec to in progress) and `mkdocs.yml`'s `copyright`.

The workspace `CLAUDE.md` needs no change. *Commit* in `sage-docs`:
`docs <version>: the schedule board's Day picker; Generation Cup spec in progress`.

### 4.8 Full check

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
cd _tests && npm run verify; cd ..
```

It must pass. A failure means something outside this phase's files changed: run `git status` and
stop and report.

### 4.9 Stop and ask: push Phase 1

Tell the owner Phase 1 is ready, with §4.3's test count, §4.6's results and §4.8's, and that
pushing changes the engine every board loads (existing boards have no container and are
unchanged). Each push is its own yes:

1. **The site:**

   ```bash
   git checkout main
   git pull
   git merge --ff-only gencup-engine
   git push
   ```

2. **The docs:** `cd /d/Personal/SAGE/sage-docs && git pull --rebase && git push`.

If `--ff-only` refuses or the rebase conflicts: stop and report. **Check, a minute or two
later:** `https://sage-match-control.github.io/events/upap-2026/schedule` still reads as before,
with no Day picker.

---

## 5. Phase 2 — the schedule board and `events.json`

### 5.1 Start

Phase 1 must be on `main`.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
```

This must print exactly the three `??` lines of §4.1. Anything else: stop and report.

```bash
git checkout -b gencup-board
cd _tests && npm test; cd ..
```

`npm test` must pass, **904 tests**, before you change anything.

### 5.2 Copy the board and fill in its tokens

```bash
cp _templates/standard-tournament-template/schedule.html events/generation-cup-2026/schedule.html
```

The copy is CRLF. Replace **every** occurrence of each token:

| Token | Replace with |
| --- | --- |
| `{{EVENT_KEY}}` | `generation-cup-2026` |
| `{{EVENT_TITLE}}` | `The Ultimate Generation Cup 2026` |
| `{{SCHEDULE_DAY_KEY}}` | `generation-cup-day1` |

**Check:** `grep -n '{{' events/generation-cup-2026/schedule.html` prints nothing, and
`grep -c 'id="dayfilter"' events/generation-cup-2026/schedule.html` prints `1` (the copy carries
Phase 1's container).

### 5.3 The board's theme

All edits in §5.3 are in `events/generation-cup-2026/schedule.html`.

**(a) The theme comment.** Replace the three lines from

```css
  /* Theme lifted from the SAGE tools (scoresheet-generator / tournament-calculator):
```

through

```css
     Barlow Condensed labels, Inter body. */
```

with:

```css
  /* The Ultimate Generation Cup 2026 palette, the same tokens as this event's
     index.html (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §3):
     deep violet structure, the pubmat's pink for the accent, a cool off-white
     paper. The event title in the bezel takes the pubmat's condensed capitals,
     Anton; the rest of the board keeps Archivo Black, Barlow Condensed and Inter. */
```

**(b) The colours.** Replace the eleven lines from

```css
    --navy:#14263C;
```

through

```css
    --radius:14px;
```

with:

```css
    --navy:#2A1052;
    --navy-deep:#170830;
    --green:#FF4FA3;
    --green-dark:#B0125F;
    --paper:#F7F5FA;
    --paper-dim:#EEEAF4;
    --line:#DCD5E6;
    --ink:#1E1030;
    --ink-soft:#5E5470;
    --white:#FFFFFF;
    --radius:14px;
```

**(c) The font link.** Replace the line

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

with:

```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

**(d) The title's font.** Replace the line (the file has exactly one)

```html
</style>
```

with these three:

```css
  /* Generation Cup: the event title in the pubmat's condensed capitals. */
  .bezel h1{ font-family:'Anton',sans-serif; font-weight:400; letter-spacing:.03em; }
</style>
```

Leave the block headed **lib/v1/css/schedule-board.css's house colours, through this page's
tokens** exactly as it is.

### 5.4 The board's category colours

Replace the **31** lines from

```js
// Category colours are the organiser's own, read off the colour-coded SCHEDULE
```

through the first line after it that is exactly

```js
};
```

(the template's comment block, its `EXAMPLE — replace` line and its seven example entries,
ending with the `};` that closes `const CAT_META = {`) with:

```js
// Category colours: the workbooks' own where they have one, read off the two SCHEDULE tabs through the Sheets API
// (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §3). Hue = event (Men's blue, Women's magenta,
// Mixed orange), lighter = lower level. The workbooks leave AMD, AXD and OWD grey, so those three take the
// generator's own Advanced shade, and OWD its darkest magenta. Keys are the category token of each team code
// (LIMD_3 -> LIMD). Cells apply the hue as a 40% tint over white.
const CAT_META = {
  NMD:  { short:'N MD',    color:'#C9DAF8' },
  NWD:  { short:'N WD',    color:'#EAD1DC' },
  NXD:  { short:'N XD',    color:'#FCE5CD' },
  LIMD: { short:'LI MD',   color:'#6D9EEB' },
  LIWD: { short:'LI WD',   color:'#C27BA0' },
  LIXD: { short:'LI XD',   color:'#F6B26B' },
  HIMD: { short:'HI MD',   color:'#1155CC' },
  HIXD: { short:'HI XD',   color:'#B45F06' },
  AMD:  { short:'A MD',    color:'#1C4587' },
  AXD:  { short:'A XD',    color:'#783F04' },
  OWD:  { short:'OPEN WD', color:'#4C1130' }
};
```

Leave the `mountScheduleBoard({ … })` call below it unchanged.

**Checks.** Each command's expected output follows it.

```bash
grep -n "EXAMPLE\|#14263C\|#7CB92C\|#F6F7F2\|#5C8F1F" events/generation-cup-2026/schedule.html
```

Prints nothing. (`LIWD`, `LIMD` and the other template example keys are this event's real
categories, with the same colours, so they are not checked for.)

```bash
grep -c "short:'" events/generation-cup-2026/schedule.html
grep -c "Anton" events/generation-cup-2026/schedule.html
wc -l < events/generation-cup-2026/schedule.html
```

Prints `11`, `3` (the font link, the comment, the rule) and `150`.

### 5.5 Add the board to the house-colours test

In `_tests/unit/house-colours.test.mjs`, find the line

```js
  'events/upap-2026/index.html',
```

and replace it with these two:

```js
  'events/upap-2026/index.html',
  'events/generation-cup-2026/schedule.html',
```

Run the unit tests (§0.4). They must pass, **907 tests**.

### 5.6 Register the event in `event-data`

```bash
cd /d/Personal/SAGE/event-data
git checkout main
git pull
git status --short
```

Prints nothing. Then the keys must be free:

```bash
grep -c '"generation-cup-day1"\|"generation-cup-day2"\|"generation-cup-2026"' config/events.json
```

Prints `0`. Save this outside the repo as `add-gencup-event.mjs`, and run it from the
`event-data` root:

```js
// Adds The Ultimate Generation Cup 2026 to event-data/config/events.json, after the last event.
// Run from the event-data repo root: node <path>/add-gencup-event.mjs
import fs from 'node:fs';

const path = 'config/events.json';
const raw = fs.readFileSync(path, 'utf8');
const config = JSON.parse(raw);
if (JSON.stringify(config, null, 2) !== raw) throw new Error('events.json is not in JSON.stringify(…, null, 2) form: stop and report');
if (config.events['generation-cup-2026']) throw new Error('generation-cup-2026 is already registered: stop and report');

config.events['generation-cup-2026'] = {
  type: 'standard',
  title: 'The Ultimate Generation Cup 2026',
  attendance: 'desks',
  scoreEntry: 'console',
  scoring: { to: 11, winBy: 1 },
  days: {
    'generation-cup-day1': {
      label: 'Nov 28 \u00b7 Next Gen',
      date: '2026-11-28',
      isLive: 'auto',
      facilities: [
        { name: 'Kingcourts', sheetId: '1WQ2sksqI-xoRulvkCoUHYIhCxAmBDSbGnJhRbzGqU0k' },
      ],
    },
    'generation-cup-day2': {
      label: 'Nov 29 \u00b7 Gen X',
      date: '2026-11-29',
      isLive: 'auto',
      facilities: [
        { name: 'Kingcourts', sheetId: '1fe5mzoPO2XoeBS54TZLjq3I8GZ0lIOS2Sss76O9RhY8' },
      ],
    },
  },
  display: {
    divisions: {
      N: 'Novice',
      LI: 'Low Intermediate',
      HI: 'High Intermediate',
      A: 'Advanced',
      O: 'Open',
    },
    events: { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' },
  },
};

fs.writeFileSync(path, JSON.stringify(config, null, 2));
console.log(Object.keys(config.events).slice(-2).join(' then '));
```

It must print `upap-2026 then generation-cup-2026`. What the entry means:

- `"attendance": "desks"`: staff check players in on a desk page (Phase 4), one link per day.
- `"scoreEntry": "console"`: scores are entered from Control Center only; there is no scorer page.
- `"isLive": "auto"`: each day's Hub shows live scores from 4 hours before that day's first match.
- `"name": "Kingcourts"`: must equal each workbook's facility name exactly; live-sync setup
  compares them.
- `\u00b7` is the `·` in each day's label; the file stores it as `·`.
- The order of the `divisions` keys sets the display order, lowest level first.
- `"scoring": { "to": 11, "winBy": 1 }`: every game, the Final and Bronze included, is to 11 and
  ends at 11 (§10). The score dialog warns only on a score that breaks it; it never stops a save.
  With no `podiumWinBy`, the podium matches take `winBy`.

**Check:**

```bash
git diff --stat
node -e "const c=JSON.parse(require('fs').readFileSync('config/events.json','utf8'));const e=c.events['generation-cup-2026'];console.log(e.type, Object.entries(e.days).map(([k,d])=>k+'='+d.label+'@'+d.facilities[0].name).join(' '), Object.values(e.display.divisions).join(' | '))"
```

`git diff --stat` shows only `config/events.json`, 48 lines added and none removed. The second
command prints:

```
standard generation-cup-day1=Nov 28 · Next Gen@Kingcourts generation-cup-day2=Nov 29 · Gen X@Kingcourts Novice | Low Intermediate | High Intermediate | Advanced | Open
```

*Commit* on `main`, **don't push yet**:

```bash
git add config/events.json
git commit -m "The Ultimate Generation Cup 2026: register generation-cup-2026 (days generation-cup-day1 and -day2, Kingcourts)"
cd /d/Personal/SAGE/sage-match-control.github.io
```

### 5.7 Commit the site's Phase 2

```bash
git add events/generation-cup-2026/schedule.html _tests/unit/house-colours.test.mjs
git status --short
git commit -m "The Ultimate Generation Cup 2026: schedule board, re-skinned from the pubmat"
```

`git status` before the commit: `A  events/generation-cup-2026/schedule.html`,
`M  _tests/unit/house-colours.test.mjs`, `?? events/generation-cup-2026/assets/`, the two
unrelated `??` lines of §4.1, and nothing else. **Never stage the pubmat.**

### 5.8 Check the board and Control Center in a browser

Save Appendix A's script outside the repo as `make-gencup-fixture.mjs`, run it from the site repo
root, and serve the site:

```bash
node <path-to>/make-gencup-fixture.mjs
npx --yes http-server -p 8000 -c-1
```

It must print exactly:

```
Day 1: 166 matches, 85 pairs; gold/bronze NMD NM 6A/NM 9A, NWD NW 8A/NW 2A, NXD NX 9A/NX 2A, LIMD LM 12A/LM 10A, LIWD LW 1A/LW 7A, LIXD LX 7A/LX 4A, HIMD HM 8A/HM 1A, HIXD HX 3A/HX 4A, AMD AM 1A/AM 3A, AXD AX 1A/AX 2A
Day 2: 49 matches, 25 pairs; gold/bronze NMD GNM 4A/GNM 2A, LIMD GLM 1A/GLM 11A, LIXD GLX 3A/GLX 4A, AMD GAM 1A/GAM 2A, OWD GOW 3A/GOW 1A
```

**Day 1** at **1440px**: `http://localhost:8000/events/generation-cup-2026/schedule.html?fixture=day1-finished`
(before 28 November it opens on Day 1, its `DAY_KEY`).

- [ ] The bar is deep violet; the title "The Ultimate Generation Cup 2026" is in tall condensed
      capitals (Anton); the meta line reads "NOV 28 · NEXT GEN · 166 MATCHES · 20 SLOTS · ALL
      COURTS"; the **DAY** select reads "Nov 28 · Next Gen" and offers "Nov 29 · Gen X".
- [ ] The court filter offers **ALL** and 1 to 10; the time column runs 9:00 AM to 4:55 PM.
- [ ] The legend shows ten chips, in the order the categories first appear: `N MD`, `N WD`,
      `LI MD`, `LI WD`, `HI MD`, `N XD`, `LI XD`, `A MD`, `HI XD`, `A XD`.
- [ ] 34 cells read **OPEN**; 10 cards carry a **FINAL** tag, 7 **BRONZE**, 14 **SF**, 9 **QF**
      and 1 **R16** (`NXD`'s, at 3:15 PM on Court 2).

**Day 2**: open `http://localhost:8000/events/generation-cup-2026/schedule.html?fixture=day2-finished&courts=1-3`
and choose "Nov 29 · Gen X" in the Day select. (In fixture mode the data follows `?fixture=`,
not the day, which is why this check starts from the `day2-finished` state.)

- [ ] The page reloads at `…?fixture=day2-finished&day=generation-cup-day2`: `courts=1-3` is gone.
- [ ] The meta line reads "NOV 29 · GEN X · 49 MATCHES · 14 SLOTS · ALL COURTS"; the court filter
      offers **ALL** and 1 to 6; the first slot is 10:00 AM.
- [ ] The legend shows `N MD`, `LI MD`, `A MD`, `OPEN WD`, `LI XD`; 35 cells read **OPEN**; 5
      **FINAL** tags and 1 **BRONZE** (`LIMD`'s, at 1:45 PM on Court 3).
- [ ] At **375px** the Day select has its own row under the meta line, and there is no
      horizontal scroll.

Control Center, Day 1:
`http://localhost:8000/tools/control-center.html?fixture=day1-finished&event=generation-cup-2026&day=generation-cup-day1`

- [ ] The header reads "THE ULTIMATE GENERATION CUP 2026"; the day picker offers "Nov 28 · Next
      Gen" and "Nov 29 · Gen X". Mission Control shows **Kingcourts**, "166 / 166 done · All
      matches done · Actual end 5:30 PM", and "Auto — goes live 5:00 AM, Nov 28 (4h before first
      serve, 9:00 AM)".
- [ ] **Standings** names the ten categories in full ("Novice Men's Doubles" … "Advanced Mixed
      Doubles"), no raw code, and no *Unmapped category* warning.
- [ ] **Awards** reads "10 of 10 podiums complete"; each podium's gold and bronze are the fixture
      line's (Novice Men's Doubles NM 6A, NM 3A, NM 9A; …; Advanced Men's Doubles AM 1A, AM 2A,
      AM 3A). The four-pair categories' bronzes went to #3 without a Bronze match: that is
      correct.

Control Center, Day 2:
`http://localhost:8000/tools/control-center.html?fixture=day2-finished&event=generation-cup-2026&day=generation-cup-day2`

- [ ] "49 / 49 done · All matches done · Actual end 4:00 PM" and "Auto — goes live 6:00 AM, Nov
      29 (4h before first serve, 10:00 AM)".
- [ ] **Awards** reads "5 of 5 podiums complete": Novice Men's Doubles GNM 4A, GNM 3A, GNM 2A;
      Low Intermediate Men's Doubles GLM 1A, GLM 5A, GLM 11A; Low Intermediate Mixed Doubles GLX
      3A, GLX 2A, GLX 4A; Advanced Men's Doubles GAM 1A, GAM 3A, GAM 2A; Open Women's Doubles GOW
      3A, GOW 2A, GOW 1A.
- [ ] **Expected, not bugs:** **Open Tournament Hub** leads to "Page not found" (the Hub is
      Phase 3).

Stop the server and remove the fixture:

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/generation-cup-2026
git status --short
```

`git status` shows nothing under `_fixtures/`.

### 5.9 Full check

`npm run verify` (§0.4) must pass. A failure means something outside this phase's files changed:
run `git status` and stop and report.

### 5.10 Stop and ask: push Phase 2

Tell the owner Phase 2 is ready, with the results of §5.8 and §5.9, and that pushing publishes
the board and registers the event (Control Center lists it within about a minute). On a yes:

```bash
git checkout main
git pull
git merge --ff-only gencup-board
git push
cd /d/Personal/SAGE/event-data
git pull --rebase
git push
cd /d/Personal/SAGE/sage-match-control.github.io
```

If `--ff-only` refuses or the rebase conflicts, stop and report. **Check, a minute or two
later:** `https://sage-match-control.github.io/events/generation-cup-2026/schedule` shows the
violet bar with "Schedule isn't published yet — check back soon." until a workbook syncs, and its
Day select offers both days; Control Center lists "The Ultimate Generation Cup 2026".

---

## 6. Phase 3 — the Tournament Hub

### 6.1 Start

Phase 2 must be on `main`.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
```

This must print exactly:

```
?? events/archives/ladies-dink-2026/assets/pubmat.webp
?? events/generation-cup-2026/assets/
?? events/upap-2026/assets/pubmat.jpg
```

Anything else: stop and report.

```bash
git checkout -b gencup-hub
cd _tests && npm test; cd ..
```

`npm test` must pass, **907 tests**.

### 6.2 Copy the Hub and fill in its tokens

```bash
cp _templates/standard-tournament-template/index.html events/generation-cup-2026/index.html
```

The copy is LF. Replace every token:

| Token | Replace with |
| --- | --- |
| `{{EVENT_KEY}}` | `generation-cup-2026` |
| `{{EVENT_TITLE}}` | `The Ultimate Generation Cup 2026` |
| `{{EVENT_TAGLINE}}` | `Pampanga Paddle Aces` |
| `{{EVENT_HEADLINE}}` | `2026` |
| `{{EVENT_DATE_RANGE}}` | `28–29 November 2026` (an en dash) |
| `{{VENUE}}` | `Kingcourts Pickleball, San Fernando, Pampanga` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr>SAGExGenCup` |
| `{{EVENT_LOGO}}` | `assets/logo.jpg` |

**Check:** `grep -n '{{' events/generation-cup-2026/index.html` prints nothing.

### 6.3 The Hub's content

All edits in §6.3–§6.5 are in `events/generation-cup-2026/index.html`, and each anchor is
quoted as it reads after §6.2.

**(a) The logo's alt text.** Replace the line

```html
    <img src="assets/logo.jpg" alt="The Ultimate Generation Cup 2026 logo" />
```

with:

```html
    <img src="assets/logo.jpg" alt="Pampanga Paddle Aces logo" />
```

**(b) The header.** Replace the four lines from

```html
  <div class="tagline-script">Pampanga Paddle Aces</div>
```

through

```html
  <div class="domination-line">2026</div>
```

(the tagline, the eyebrow, the title and the headline) with these five:

```html
  <div class="tagline-script">Pampanga Paddle Aces</div>
  <h1 class="title">The <span class="accent">Ultimate</span><br>Generation Cup</h1>
  <div class="domination-line">2026</div>
  <div class="gen-days"><span class="gen-chip next">Next Gen &middot; Sat 28 Nov</span><span class="gen-chip genx">Gen X &middot; Sun 29 Nov</span></div>
  <div class="eyebrow">28–29 November 2026<br>Kingcourts Pickleball &nbsp;·&nbsp; San Fernando, Pampanga</div>
```

**(c) The category search's placeholder.** Replace the line

```html
          <input id="catFilterInput" type="text" class="cat-filter-input" placeholder="e.g. Beginner 18+ Men's Doubles" autocomplete="off" inputmode="latin" />
```

with:

```html
          <input id="catFilterInput" type="text" class="cat-filter-input" placeholder="e.g. Low Intermediate Men's Doubles" autocomplete="off" inputmode="latin" />
```

**(d) The footer.** Replace the line

```html
  The Ultimate Generation Cup 2026 &middot; 28–29 November 2026 &middot; Kingcourts Pickleball, San Fernando, Pampanga<br>
```

with these three:

```html
  The Ultimate Generation Cup 2026 &middot; 28–29 November 2026<br>
  Kingcourts Pickleball, San Fernando, Pampanga<br>
  Organised by the Pampanga Paddle Aces<br>
```

### 6.4 The Hub's colours and font

**(a) The font link.** Replace the line

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

with:

```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

**(b) The theme comment.** Replace the five lines from

```css
     S.A.G.E. house palette — navy structure, green accent, off-white
```

through

```css
     as one product.
```

with:

```css
     The Ultimate Generation Cup 2026 pubmat's palette
     (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §3): deep
     violet structure from its purple, its hot pink for the accent, a cool
     off-white paper, and its pink, blue and violet brush strokes in the
     hero. Shared with this event's schedule.html.
```

**(c) The contrast note.** Replace the seven lines from

```css
     Contrast rules this palette has to respect — the green is a *fill*
```

through

```css
       - Small text on paper is --ink (~13.8:1) or --ink-soft (~5.9:1).
```

with these six:

```css
     Contrast rules this palette has to respect — the pink accent is a
     *fill* colour, not a text colour:
       - --green (pink) on white is ~3.0:1. Use it as a background (with
         --navy text on top, ~5.3:1) or on the violet hero (~5.3:1).
       - --green-dark on white is ~6.8:1 and on paper ~6.3:1.
       - Small text on paper is --ink (~16.6:1) or --ink-soft (~6.5:1).
```

**(d) The colours.** Replace the nine lines from

```css
    --navy:#14263C;
```

through

```css
    --ink-soft:#5B6B74;
```

with:

```css
    --navy:#2A1052;
    --navy-deep:#170830;
    --green:#FF4FA3;
    --green-dark:#B0125F;
    --paper:#F7F5FA;
    --paper-dim:#EEEAF4;
    --line:#DCD5E6;
    --ink:#1E1030;
    --ink-soft:#5E5470;
```

**(e) The hero's own colours.** Find the line (the file has exactly one)

```css
    --radius:14px;
```

and replace it with itself followed by:

```css
    /* The pubmat's own colours, for this page's hero only (spec §3). */
    --gc-pink:#E8197D;
    --gc-blue:#1E6FE0;
    --gc-violet:#B98CFF;
    --gc-pink-deep:#C2126A;
    --gc-blue-deep:#1A5FC8;
```

### 6.5 The Hub's header styling

Replace the line (the file has exactly one)

```html
</style>
```

with:

```css
/* ==== The Ultimate Generation Cup 2026: the pubmat's hero (sage-docs/docs/specs/.../ultimate-generation-cup-2026-spec.md §3) ==== */
/* The pubmat's brush strokes: a pink band across the top left, a blue band low on the right. */
header.hero{
  background-color:var(--navy);
  background-image:
    linear-gradient(160deg, color-mix(in srgb, var(--gc-pink) 55%, transparent) 0 14%, transparent 14%),
    linear-gradient(160deg, transparent 0 82%, color-mix(in srgb, var(--gc-blue) 45%, transparent) 82% 91%, transparent 91%);
}
/* "Pampanga Paddle Aces": the pubmat's condensed capitals, above the title. */
.tagline-script{ font-family:'Barlow Condensed',sans-serif; font-style:normal; font-weight:700; text-transform:uppercase; letter-spacing:.14em; font-size:clamp(14px,3.6vw,18px); margin:0 0 6px; }
/* "The Ultimate Generation Cup": the pubmat's condensed capitals, "Ultimate" in its pink. */
h1.title{ font-family:'Anton',sans-serif; font-weight:400; font-size:clamp(34px,9vw,64px); line-height:1.04; letter-spacing:.01em; text-transform:uppercase; margin:0; }
h1.title .accent{ color:var(--green); }
/* "2026": the pubmat's violet, slanted like its script. */
.domination-line{ font-family:'Anton',sans-serif; font-style:italic; font-weight:400; font-size:clamp(30px,8vw,52px); line-height:1.1; letter-spacing:.02em; text-transform:none; color:var(--gc-violet); margin:2px 0 14px; }
/* Which generation plays which day: the pubmat's blue for Next Gen, its pink for Gen X. */
.gen-days{ display:flex; flex-wrap:wrap; justify-content:center; gap:8px; margin:0 0 16px; }
.gen-chip{ font-family:'Barlow Condensed',sans-serif; font-weight:700; font-size:14px; letter-spacing:.08em; text-transform:uppercase; color:var(--white); padding:5px 12px; border-radius:6px; }
.gen-chip.next{ background:var(--gc-blue-deep); }
.gen-chip.genx{ background:var(--gc-pink-deep); }
/* Date, then venue, under the chips, in white. */
.eyebrow{ color:var(--white); line-height:1.9; margin-bottom:22px; }
</style>
```

Leave the block of rules restating the engine's house colours through the page's tokens (the
`color-mix(…)` lines above `</style>`) exactly as it is.

**Checks.**

```bash
grep -n "#14263C\|#7CB92C\|#F6F7F2\|#5C8F1F" events/generation-cup-2026/index.html
grep -c "Anton" events/generation-cup-2026/index.html
grep -c -- "--gc-" events/generation-cup-2026/index.html
wc -l < events/generation-cup-2026/index.html
```

The first prints nothing; then `3`, `10` (five defined, five used) and `292`.

### 6.6 Add the Hub to the house-colours test

In `_tests/unit/house-colours.test.mjs`, find the line

```js
  'events/generation-cup-2026/schedule.html',
```

and replace it with these two:

```js
  'events/generation-cup-2026/schedule.html',
  'events/generation-cup-2026/index.html',
```

Run the unit tests (§0.4). They must pass, **910 tests**.

### 6.7 Commit the site's Phase 3

```bash
git add events/generation-cup-2026/index.html events/generation-cup-2026/assets/logo.jpg _tests/unit/house-colours.test.mjs
git add events/generation-cup-2026/assets/qr.png 2>/dev/null; true
git status --short
git commit -m "The Ultimate Generation Cup 2026: Tournament Hub, re-skinned from the pubmat"
```

`git status` before the commit: `A ` for `index.html` and `assets/logo.jpg` (and
`assets/qr.png` if it has come), `M ` for the test, `?? events/generation-cup-2026/assets/pubmat.jpg`,
the two unrelated `??` lines, and nothing else. **Never stage the pubmat.**

### 6.8 Check the Hub in a browser

Build the fixture again and serve the site, from the repo root:

```bash
node <path-to>/make-gencup-fixture.mjs
npx --yes http-server -p 8000 -c-1
```

It prints §5.8's two lines again. Open
`http://localhost:8000/events/generation-cup-2026/?fixture=day1-finished`.

At **1280px**:

- [ ] The hero is deep violet with a pink band across its top left and a blue band low on its
      right. Top to bottom: the round Paddle Aces badge × the S.A.G.E. logo; "PAMPANGA PADDLE
      ACES"; "THE **ULTIMATE**" (Ultimate in pink) over "GENERATION CUP" in tall condensed
      capitals; "2026" in slanted violet; two chips, "NEXT GEN · SAT 28 NOV" (blue) and "GEN X ·
      SUN 29 NOV" (pink); "28–29 NOVEMBER 2026"; "KINGCOURTS PICKLEBALL · SAN FERNANDO,
      PAMPANGA"; the subtitle; **Select tournament day** offering "Nov 28 · Next Gen" and "Nov 29 ·
      Gen X".
- [ ] The QR panel at the top right shows `tinyurl.com/SAGExGenCup`.
- [ ] Choosing "Nov 28 · Next Gen" shows the tabs and "Nov 28 · Next Gen · Live · last synced
      5:30:00 PM". **Standings** lists the ten categories by full name in the workbook's order,
      no *Unmapped category*; Novice Men's Doubles' podium is NM 6A / NM 6B, NM 3A / NM 3B, NM 9A
      / NM 9B, and its four brackets' tables are headed Bracket 1 to Bracket 4.
- [ ] The footer reads "The Ultimate Generation Cup 2026 · 28–29 November 2026", "Kingcourts
      Pickleball, San Fernando, Pampanga", "Organised by the Pampanga Paddle Aces", "Powered by
      S.A.G.E. Match Control Experts".

At **900px, 1024px, 1120px and 1280px** (reload the page at each width), paste this in the
console. It must print `[]` each time — no header text under the QR panel:

```js
(() => { const q = document.querySelector('.qr-panel').getBoundingClientRect(); return ['.tagline-script','h1.title','.domination-line','.gen-days','.eyebrow','.hero-logos','.subtitle','.day-picker'].filter(s => { const el = document.querySelector(s), r = document.createRange(); r.selectNodeContents(el); return [...r.getClientRects(), ...[...el.querySelectorAll('img')].map(i => i.getBoundingClientRect())].some(b => Math.min(b.right, q.right) > Math.max(b.left, q.left) && Math.min(b.bottom, q.bottom) > Math.max(b.top, q.top)); }); })()
```

At **375px**:

- [ ] The hero doesn't show: the slim bar is at the top with the badge, "28–29 NOVEMBER 2026
      KINGCOU…" in pink over "THE ULTIMATE GE…", and **CHOOSE A DAY**, whose menu offers "Nov 28
      · Next Gen" and "Nov 29 · Gen X". (A phone always shows the bar, as on every Hub.)
- [ ] No horizontal scroll (`document.documentElement.scrollWidth === innerWidth`).
- [ ] Developer tools: no console errors except the failed request for `assets/qr.png`, if it
      hasn't come.

Stop the server and remove the fixture as at the end of §5.8.

### 6.9 Full check

```bash
cd _tests && npm run verify; cd ..
```

It must pass. A failure means something outside this phase's files changed: run `git status` and
stop and report.

### 6.10 Stop and ask: push Phase 3

Tell the owner Phase 3 is ready, with the results of §6.8 and §6.9, whether `qr.png` was
included, and that pushing publishes the Hub at the short link. On a yes:

```bash
git checkout main
git pull
git merge --ff-only gencup-hub
git push
```

If `--ff-only` refuses, stop and report. **Check, a minute or two later:**
`https://sage-match-control.github.io/events/generation-cup-2026/` shows the header as §6.8 has
it, and Control Center's **Open Tournament Hub** opens it.

---

## 7. Phase 4 — the desk page, the dry-run checklist and the QR image

### 7.1 Start

Phase 3 must be on `main`.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
```

This must print exactly the three `??` lines of §6.1, except that the event's line now reads
`?? events/generation-cup-2026/assets/pubmat.jpg` (and `?? events/generation-cup-2026/assets/qr.png`
if the QR image came after Phase 3). Anything else: stop and report.

```bash
git checkout -b gencup-rest
cd _tests && npm test; cd ..
```

`npm test` must pass, **910 tests**.

### 7.2 Copy the files and fill in their tokens

```bash
cp _templates/attendance/attendance.html events/generation-cup-2026/attendance.html
cp _templates/dry-run-checklist-template.md events/generation-cup-2026/dry-run-checklist.md
```

`attendance.html` is CRLF, `dry-run-checklist.md` LF. In both, replace every `{{EVENT_KEY}}`
with `generation-cup-2026` and every `{{EVENT_TITLE}}` with `The Ultimate Generation Cup 2026`
(`attendance.html` has one and two of them, the checklist none and three). The desk page keeps
the house colours (§10). **Check:** `grep -rnI '{{' events/generation-cup-2026/` prints nothing.

### 7.3 Check the desk page

Serve the site and open `http://localhost:8000/events/generation-cup-2026/attendance.html`:

- [ ] The tab's title is "The Ultimate Generation Cup 2026 — Attendance" and the page reads "Ask
      the operator for a desk link." in the house colours.

### 7.4 Commit

```bash
git add events/generation-cup-2026/attendance.html events/generation-cup-2026/dry-run-checklist.md
git add events/generation-cup-2026/assets/qr.png 2>/dev/null; true
git status --short
git commit -m "The Ultimate Generation Cup 2026: desk page and dry-run checklist"
```

`git status` before the commit: `A ` for the two files (and `qr.png` if it is new),
`?? events/generation-cup-2026/assets/pubmat.jpg`, the two unrelated `??` lines, and nothing
else. **Never stage the pubmat.**

### 7.5 Full check

```bash
cd _tests && npm run verify; cd ..
```

It must pass. A failure: run `git status` and stop and report.

### 7.6 Stop and ask: push Phase 4

Tell the owner Phase 4 is ready, with §7.3's result and whether `qr.png` was included. On a yes:

```bash
git checkout main
git pull
git merge --ff-only gencup-rest
git push
```

If `--ff-only` refuses, stop and report. **Check, a minute or two later:**
`https://sage-match-control.github.io/events/generation-cup-2026/attendance` reads "Ask the
operator for a desk link." If `qr.png` still hasn't come, say so: it is committed and pushed on
its own when it does, with the owner's yes.

---

## 8. Report back, at the end of each phase

- Which checks passed, and any that didn't, with what you saw.
- That the fixture (and in Phase 1 the scratch `schedule-beta.html`) was removed.
- Whether `qr.png` was present.
- What was pushed, and to which repo, only after the owner's yes.
- `git status --short` in every repo you touched.

---

## 9. Operator tasks — people, not the implementer

Both workbooks were read on 2026-11-23. These happen in Google Sheets and on the day.

### 9.1 Workbook checks before sync setup

- **Day 2 has no match numbers yet.** Its `SCHEDULE` is fully packed (49 matches, 10:00 AM to
  3:25 PM), but its `CSV` tab reads "Not found" for the first match and leaves the rest
  unnumbered. Run **SAGE → Fill match numbers** in the Day 2 workbook before setting up its live
  sync; `CSV` should then number 1–49 in schedule order, as Appendix A assumes. (Numbers may
  repeat across the two days: each day is its own snapshot.)
- **Day 1's `HIMD` Final is missing one bracket link.** `Variables!AD:AG` goes from `HIMD_F_1`
  straight to `HIXD_F_1`, so `STANDINGSCSV` gives `HIMD_F_2` no `match` or `from`, and the
  bracket chart can't draw that Final's second seat. Type `HIMD_F_2`, `F`, `W:SF-2` into an empty
  row of `Variables!AD:AF` (the lookup reads the whole of `AD:AG`, so any empty row works, e.g.
  row 89), then check that `STANDINGSCSV`'s `HIMD_F_2` row shows `F` and `W:SF-2`. A duplicate
  `AXD_B_2 | B` further down the same table is harmless; leave it.
- **Rosters.** Neither workbook has player names yet (every name cell holds its team code), and
  both `ATTENDANCE` tabs have their header and no roster. Names go into each category tab; then
  **Update roster** in Control Center's Attendance tab, per day.
- **Generator version**: **SAGE → Help** in each workbook should read Standard Tournament
  Generator 1.5.0; anything older means a workbook nobody may regenerate.
- **`Court Control`**: not checked in this read. Confirm each workbook's tab lists its courts
  (10 on Day 1, 6 on Day 2) before score entry.
- **The `AMD`, `AXD` and `OWD` cells are grey** in the workbooks; the board gives them their own
  colours (§3.4). Nothing to do.
- Players entered in two categories on the same day, and back-to-back matches, are the
  organiser's call.

### 9.2 Share each workbook, then set up live sync — once each, after Phase 2 is pushed

For **each** workbook:

1. Share it with `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**;
   set **General access** to **Restricted**.
2. **Extensions → Apps Script**: confirm `sheets-sync.gs` is there and current.
3. **SAGE → Set up live sync**: Day **`generation-cup-day1`** for the Day 1 workbook and
   **`generation-cup-day2`** for the Day 2 workbook (not the `day1`/`day2` the generator wrote in
   `Variables`, which sync doesn't read); Venue **`Kingcourts`** (exact); the default tabs. The
   secret only if asked, from whoever manages Cloud Run, never pasted into chat.
4. **Confirm:** a minute later
   `https://sage-match-control.github.io/event-data/generation-cup-2026/data/generation-cup-day1.json`
   (and `…-day2.json`) lists `Kingcourts` with 166 (and 49) matches.

### 9.3 Go-live

Each day's Hub goes live 4 hours before its first serve: **5:00 AM on 28 November** and **6:00
AM on 29 November**. Control Center's override changes either.

### 9.4 The board on the day

Control Center's **Open schedule** opens the board on the day whose date it is: Day 1 on 28
November, Day 2 on 29 November, nothing to change overnight. A wall screen left on from Day 1
switches at its next reload; reload it on the morning of Day 2, or pick the day in its **Day**
select.

### 9.5 The QR panel

After `qr.png` is pushed: `node _templates/hub-pubmat/render.mjs generation-cup-2026`, print
`_templates/hub-pubmat/out/generation-cup-2026/qr-panel.pdf`, and scan it before mounting it.

### 9.6 Dry run, by Thursday 26 November

`events/generation-cup-2026/dry-run-checklist.md` against the real workbooks, **once per day**:
each day's sync, desk link, score entry and go-live. This event's own checks: the board's **Day**
select switches between the two real days; before 28 November **Open schedule** opens Day 1.
There is no prize draw (§10): skip the checklist's prize-draw rehearsal. Enter one score of each
kind in the rehearsal, an 11–9 and an 11–10, to see that neither warns under the event's
win-by-1 rule, and type a 12–10 to see that it does (then cancel it).

### 9.7 After the event

Set `LIVE_BASE_URL = ''` in every page of `events/generation-cup-2026/`. Leave the event in
`events.json`. Move this spec to `implemented/`.

---

## 10. Questions, answered by the owner on 23 November 2026

| Question | Answer |
| --- | --- |
| How the board changes day | A day picker on the board, like the Hub's: Phase 1 |
| Attendance and score entry | Desk links (`attendance.html`), scores in Control Center only |
| The labels | The workbooks' names (Novice, Low Intermediate, High Intermediate, Advanced, Open); the generation goes in each day's label |
| Which generation which day | Day 1 (28 Nov) **Next Gen**, Day 2 (29 Nov) **Gen X** |
| The short link | `tinyurl.com/SAGExGenCup`; the owner makes both the short link and the QR image, and sends the image later. The phases go ahead without it |
| The raffle | **No prize draw.** The organiser runs the pubmat's raffle; nothing on the site or in the dry run |
| The logo | The Pampanga Paddle Aces badge, sent by the owner |
| The staff pages' colours | The house colours |
| The scoring rule (`scoring` in `events.json`) | Every match to 11, win by 1, the podium matches too: `{ "to": 11, "winBy": 1 }` |
| Build order | Not asked; the default, with the engine phase first |

---

## 11. Decisions, for reference

- **Event key `generation-cup-2026`, day keys `generation-cup-day1` and `-day2`**: short, and the
  day keys can't collide with any other event's.
- **A Day picker instead of an evening push**: the owner's choice. It works for every future
  multi-day event made from a template, and nothing is pushed inside the freeze.
- **The picker is markup-driven**, not a new setting: only a page with the container gets it, so
  every existing board, including archived ones, is unchanged and reads no more than before.
- **Today's date picks the day** when the address doesn't, so the wall display needs nobody on
  the morning of Day 2. Control Center's **Open schedule** is therefore not changed.
- **Choosing a day reloads the page** rather than swapping data in place: the live channel and
  the staff view follow the new day with no extra code, and `?courts=` can't point past Day 2's
  six courts.
- **`events.json` ships with the board**: registering early lets the operators set up live sync
  and rehearse on the real board while the Hub is built.
- **The generation is in the day label**, not the division: the workbooks' categories are the
  same levels on both days, and the label is what every picker shows.
- **The board takes the workbooks' colours** for the eight categories that have them, so the wall
  and the printed schedule agree; only the three grey ones are ours.

---

## Appendix A — the fixture script

Used in §4.6, §5.8 and §6.8, and never committed. Save it **outside the repo** and run it from
the site repo root. It writes `_fixtures/generation-cup-2026/day1-finished.json` and
`day2-finished.json` and adds the event to `_fixtures/config.json`; the end of §5.8 says how to
remove them.

```js
// Writes a temporary, local-only fixture for The Ultimate Generation Cup 2026, shaped like the two real workbooks:
// Day 1 (Next Gen, 10 courts, 166 matches, 9:00 AM to 4:55 PM) and Day 2 (Gen X, 6 courts, 49 matches, 10:00 AM
// to 3:25 PM, numbered in schedule order as SAGE -> Fill match numbers will number them). Every match is played,
// with fixed made-up scores and player names; each playoff seat is filled the way the workbook's STANDINGSCSV
// says (a bracket's rank, a best runner-up, or the winner or loser of an earlier match).
//   _fixtures/generation-cup-2026/day1-finished.json   Day 1's snapshot
//   _fixtures/generation-cup-2026/day2-finished.json   Day 2's snapshot
//   _fixtures/config.json                              gains a "generation-cup-2026" entry with both days
// Neither is ever committed. Undo with:
//   git checkout -- _fixtures/config.json && rm -rf _fixtures/generation-cup-2026
// usage, from the site repo root: node <path>/make-gencup-fixture.mjs
import fs from 'node:fs';
import path from 'node:path';

const EVENT = 'generation-cup-2026';
const FACILITY = 'Kingcourts';

// Each workbook's SCHEDULE, slot by slot: "<court>:<team code 1>:<team code 2>", courts in order, empty ones left out.
const DAY1_SLOTS = [
  ['9:00 AM',  '1:NMD_1:NMD_2 2:NMD_3:NMD_4 3:NMD_11:NMD_12 4:NWD_1:NWD_2 5:NWD_3:NWD_4 6:LIMD_1:LIMD_2 7:LIMD_3:LIMD_4 8:LIMD_9:LIMD_10 9:LIWD_1:LIWD_2 10:LIWD_3:LIWD_4'],
  ['9:25 AM',  '1:NMD_5:NMD_6 2:NMD_8:NMD_9 4:NWD_5:NWD_6 5:NWD_7:NWD_8 6:LIMD_5:LIMD_6 7:LIMD_7:LIMD_8 8:LIMD_12:LIMD_13 9:LIWD_1:LIWD_3 10:LIWD_4:LIWD_5'],
  ['9:50 AM',  '1:NMD_1:NMD_3 2:NMD_2:NMD_4 3:NMD_12:NMD_13 4:NWD_1:NWD_3 5:NWD_2:NWD_4 6:LIMD_1:LIMD_3 7:LIMD_2:LIMD_4 8:LIMD_10:LIMD_11 9:LIWD_6:LIWD_7 10:LIWD_8:LIWD_9'],
  ['10:15 AM', '1:NMD_6:NMD_7 2:NMD_9:NMD_10 4:NWD_5:NWD_7 5:NWD_6:NWD_8 6:LIMD_5:LIMD_7 7:LIMD_6:LIMD_8 8:LIMD_13:LIMD_14 9:LIWD_1:LIWD_4 10:LIWD_2:LIWD_5'],
  ['10:40 AM', '1:NMD_1:NMD_4 2:NMD_2:NMD_3 3:NMD_11:NMD_13 4:NWD_1:NWD_4 5:NWD_2:NWD_3 6:LIMD_1:LIMD_4 7:LIMD_2:LIMD_3 8:LIMD_9:LIMD_11 9:LIWD_6:LIWD_8 10:LIWD_7:LIWD_9'],
  ['11:05 AM', '1:NMD_5:NMD_7 2:NMD_8:NMD_10 4:NWD_5:NWD_8 5:NWD_6:NWD_7 6:LIMD_5:LIMD_8 7:LIMD_6:LIMD_7 8:LIMD_12:LIMD_14 9:LIWD_3:LIWD_5 10:LIWD_2:LIWD_4'],
  ['11:30 AM', '1:NMD_QF_1:NMD_QF_2 2:NMD_QF_3:NMD_QF_4 3:NMD_QF_5:NMD_QF_6 4:NMD_QF_7:NMD_QF_8 5:LIMD_QF_1:LIMD_QF_2 6:LIMD_QF_3:LIMD_QF_4 7:LIMD_QF_5:LIMD_QF_6 8:LIMD_QF_7:LIMD_QF_8 9:LIWD_6:LIWD_9 10:LIWD_7:LIWD_8'],
  ['11:55 AM', '1:HIMD_1:HIMD_2 2:HIMD_3:HIMD_4 3:NWD_SF_1:NWD_SF_2 4:NWD_SF_3:NWD_SF_4 5:NMD_SF_1:NMD_SF_2 6:NMD_SF_3:NMD_SF_4 7:LIMD_SF_1:LIMD_SF_2 8:LIMD_SF_3:LIMD_SF_4 9:LIWD_1:LIWD_5 10:LIWD_2:LIWD_3'],
  ['12:20 PM', '1:HIMD_5:HIMD_6 2:HIMD_7:HIMD_8 3:NWD_B_1:NWD_B_2 4:NWD_F_1:NWD_F_2 5:NMD_F_1:NMD_F_2 6:NMD_B_1:NMD_B_2 7:LIMD_F_1:LIMD_F_2 8:LIMD_B_1:LIMD_B_2 9:LIWD_SF_1:LIWD_SF_2 10:LIWD_SF_3:LIWD_SF_4'],
  ['12:45 PM', '1:HIMD_1:HIMD_3 2:HIMD_2:HIMD_4 3:NXD_1:NXD_2 4:NXD_3:NXD_4 5:NXD_9:NXD_10 6:NXD_11:NXD_12 7:LIWD_F_1:LIWD_F_2 8:LIWD_B_1:LIWD_B_2 9:LIXD_1:LIXD_2 10:LIXD_3:LIXD_4'],
  ['1:10 PM',  '1:HIMD_5:HIMD_7 2:HIMD_6:HIMD_8 4:NXD_5:NXD_6 5:NXD_7:NXD_8 6:AMD_1:AMD_2 7:AMD_3:AMD_4 9:LIXD_6:LIXD_7 10:LIXD_8:LIXD_9'],
  ['1:35 PM',  '1:HIMD_1:HIMD_4 2:HIMD_2:HIMD_3 4:NXD_1:NXD_3 5:NXD_2:NXD_4 6:NXD_9:NXD_11 7:NXD_10:NXD_12 9:LIXD_1:LIXD_3 10:LIXD_4:LIXD_5'],
  ['2:00 PM',  '1:HIMD_5:HIMD_8 2:HIMD_6:HIMD_7 4:NXD_5:NXD_7 5:NXD_6:NXD_8 6:AMD_1:AMD_3 7:AMD_2:AMD_4 9:LIXD_6:LIXD_8 10:LIXD_7:LIXD_9'],
  ['2:25 PM',  '1:HIMD_SF_1:HIMD_SF_2 2:HIMD_SF_3:HIMD_SF_4 4:NXD_1:NXD_4 5:NXD_2:NXD_3 6:NXD_9:NXD_12 7:NXD_10:NXD_11 9:LIXD_1:LIXD_4 10:LIXD_2:LIXD_5'],
  ['2:50 PM',  '1:HIMD_F_1:HIMD_F_2 2:HIMD_B_1:HIMD_B_2 4:NXD_5:NXD_8 5:NXD_6:NXD_7 6:AMD_1:AMD_4 7:AMD_2:AMD_3 9:LIXD_6:LIXD_9 10:LIXD_7:LIXD_8'],
  ['3:15 PM',  '2:NXD_R16_5:NXD_R16_6 4:HIXD_1:HIXD_2 5:HIXD_3:HIXD_4 6:AMD_F_1:AMD_F_2 9:LIXD_3:LIXD_5 10:LIXD_2:LIXD_4'],
  ['3:40 PM',  '2:NXD_QF_3:NXD_QF_4 4:HIXD_1:HIXD_3 5:HIXD_2:HIXD_4 6:AXD_1:AXD_2 7:AXD_3:AXD_4 9:LIXD_1:LIXD_5 10:LIXD_2:LIXD_3'],
  ['4:05 PM',  '1:NXD_SF_1:NXD_SF_2 3:NXD_SF_3:NXD_SF_4 4:HIXD_1:HIXD_4 5:HIXD_2:HIXD_3 6:AXD_1:AXD_3 7:AXD_2:AXD_4 9:LIXD_SF_1:LIXD_SF_2 10:LIXD_SF_3:LIXD_SF_4'],
  ['4:30 PM',  '1:NXD_B_1:NXD_B_2 3:NXD_F_1:NXD_F_2 5:HIXD_F_1:HIXD_F_2 6:AXD_1:AXD_4 7:AXD_2:AXD_3 9:LIXD_F_1:LIXD_F_2 10:LIXD_B_1:LIXD_B_2'],
  ['4:55 PM',  '6:AXD_F_1:AXD_F_2'],
];
const DAY2_SLOTS = [
  ['10:00 AM', '1:NMD_1:NMD_2 2:NMD_3:NMD_4 3:LIMD_1:LIMD_2 4:LIMD_3:LIMD_4 5:AMD_1:AMD_2 6:OWD_1:OWD_2'],
  ['10:25 AM', '2:LIMD_9:LIMD_10 3:LIMD_5:LIMD_6 4:LIMD_7:LIMD_8 5:AMD_2:AMD_3 6:OWD_2:OWD_3'],
  ['10:50 AM', '1:NMD_1:NMD_3 2:NMD_2:NMD_4 3:LIMD_1:LIMD_3 4:LIMD_2:LIMD_4 5:AMD_1:AMD_3 6:OWD_1:OWD_3'],
  ['11:15 AM', '2:LIMD_10:LIMD_11 3:LIMD_5:LIMD_7 4:LIMD_6:LIMD_8 5:AMD_2:AMD_1 6:OWD_2:OWD_1'],
  ['11:40 AM', '1:NMD_1:NMD_4 2:NMD_2:NMD_3 3:LIMD_1:LIMD_4 4:LIMD_2:LIMD_3 5:AMD_3:AMD_2 6:OWD_3:OWD_2'],
  ['12:05 PM', '1:NMD_F_1:NMD_F_2 2:LIMD_9:LIMD_11 3:LIMD_5:LIMD_8 4:LIMD_6:LIMD_7 5:AMD_3:AMD_1 6:OWD_3:OWD_1'],
  ['12:30 PM', '2:LIMD_R16_5:LIMD_R16_6 5:AMD_F_1:AMD_F_2 6:OWD_F_1:OWD_F_2'],
  ['12:55 PM', '2:LIMD_QF_3:LIMD_QF_4'],
  ['1:20 PM',  '1:LIMD_SF_1:LIMD_SF_2 3:LIMD_SF_3:LIMD_SF_4'],
  ['1:45 PM',  '1:LIMD_F_1:LIMD_F_2 3:LIMD_B_1:LIMD_B_2'],
  ['2:10 PM',  '4:LIXD_1:LIXD_2 5:LIXD_3:LIXD_4'],
  ['2:35 PM',  '4:LIXD_1:LIXD_3 5:LIXD_2:LIXD_4'],
  ['3:00 PM',  '4:LIXD_1:LIXD_4 5:LIXD_2:LIXD_3'],
  ['3:25 PM',  '4:LIXD_F_1:LIXD_F_2'],
];

// The playoff ladders the workbooks' STANDINGSCSV writes: [seat, match, from, seat label].
const B_AND_F = [['B_1', 'B', 'L:SF-1'], ['B_2', 'B', 'L:SF-2'], ['F_1', 'F', 'W:SF-1'], ['F_2', 'F', 'W:SF-2']];
const LADDERS = {
  qf8: [
    ['QF_1', 'QF-1', '', 'Bracket 1 · #1'], ['QF_2', 'QF-1', '', 'Bracket 2 · #2'], ['QF_3', 'QF-2', '', 'Bracket 3 · #1'], ['QF_4', 'QF-2', '', 'Bracket 4 · #2'],
    ['QF_5', 'QF-3', '', 'Bracket 2 · #1'], ['QF_6', 'QF-3', '', 'Bracket 1 · #2'], ['QF_7', 'QF-4', '', 'Bracket 4 · #1'], ['QF_8', 'QF-4', '', 'Bracket 3 · #2'],
    ['SF_1', 'SF-1', 'W:QF-1'], ['SF_2', 'SF-1', 'W:QF-2'], ['SF_3', 'SF-2', 'W:QF-3'], ['SF_4', 'SF-2', 'W:QF-4'], ...B_AND_F,
  ],
  sf4: [
    ['SF_1', 'SF-1', '', 'Bracket 1 · #1'], ['SF_2', 'SF-1', '', 'Bracket 2 · #2'], ['SF_3', 'SF-2', '', 'Bracket 2 · #1'], ['SF_4', 'SF-2', '', 'Bracket 1 · #2'], ...B_AND_F,
  ],
  r16: [
    ['R16_5', 'R16-3', '', '2nd-best runner-up'], ['R16_6', 'R16-3', '', '3rd-best runner-up'], ['QF_3', 'QF-2', 'W:R16-3'], ['QF_4', 'QF-2', '', 'Best runner-up'],
    ['SF_1', 'SF-1', '', 'Bracket 1 · #1'], ['SF_2', 'SF-1', '', 'Bracket 2 · #1'], ['SF_3', 'SF-2', '', 'Bracket 3 · #1'], ['SF_4', 'SF-2', 'W:QF-2'], ...B_AND_F,
  ],
  // One bracket, top two to the Final; the bronze goes to #3 with no match (B_2 is a BYE).
  final: [['F_1', 'F'], ['F_2', 'F'], ['B_1', 'B'], ['B_2', 'B']],
};

// Category -> [name stem, bracket sizes, ladder], in each workbook's order.
const DAY1_CATS = {
  NMD: ['NM', [4, 3, 3, 3], 'qf8'], NWD: ['NW', [4, 4], 'sf4'], NXD: ['NX', [4, 4, 4], 'r16'],
  LIMD: ['LM', [4, 4, 3, 3], 'qf8'], LIWD: ['LW', [5, 4], 'sf4'], LIXD: ['LX', [5, 4], 'sf4'],
  HIMD: ['HM', [4, 4], 'sf4'], HIXD: ['HX', [4], 'final'], AMD: ['AM', [4], 'final'], AXD: ['AX', [4], 'final'],
};
const DAY2_CATS = {
  NMD: ['GNM', [4], 'final'], LIMD: ['GLM', [4, 4, 3], 'r16'], LIXD: ['GLX', [4], 'final'],
  AMD: ['GAM', [3], 'final'], OWD: ['GOW', [3], 'final'],
};

const catOf = code => code.split('_')[0];
const isRoundRobin = code => /^[A-Z]+_\d+$/.test(code);
const BYE = ['BYE ', 'BYE '];

function buildDay({ day, label, slots, cats, seed }) {
  // Fixed pseudo-random scores: the same file every run.
  const rand = n => { seed = (seed * 1103515245 + 12345) % 2147483648; return Math.floor(seed / 65536) % n; };
  const score = () => { const loser = rand(10); return rand(2) ? [11, loser] : [loser, 11]; };

  const names = {}, bracketOf = {};
  for (const [cat, [stem, sizes]] of Object.entries(cats)) {
    let n = 0;
    sizes.forEach((size, b) => { for (let i = 0; i < size; i++) { n++; names[`${cat}_${n}`] = [`${stem} ${n}A `, `${stem} ${n}B `]; bracketOf[`${cat}_${n}`] = b + 1; } });
  }
  const rec = {};
  for (const code of Object.keys(names)) rec[code] = { w: 0, l: 0, pf: 0, pa: 0 };
  const tally = (code, pf, pa) => { const r = rec[code]; r.pf += pf; r.pa += pa; if (pf > pa) r.w++; else r.l++; };
  const better = (x, y) => rec[y].w - rec[x].w || rec[y].pf / rec[y].pa - rec[x].pf / rec[x].pa;
  const bracket = (cat, b) => Object.keys(names).filter(k => catOf(k) === cat && bracketOf[k] === b).sort(better);
  const runnersUp = cat => cats[cat][1].map((_, b) => bracket(cat, b + 1)[1]).sort(better);
  const ladder = cat => LADDERS[cats[cat][2]];

  const results = {};   // "<CAT> <match>" -> { winner, loser }
  const seatNames = code => {
    const cat = catOf(code), slot = code.slice(cat.length + 1);
    const row = ladder(cat).find(r => r[0] === slot);
    if (cats[cat][2] === 'final') {
      const order = bracket(cat, 1), i = { F_1: 0, F_2: 1, B_1: 2 }[slot];
      return i === undefined || !order[i] ? BYE : names[order[i]];
    }
    const [, , from, seat] = row;
    if (from) { const r = results[`${cat} ${from.slice(2)}`]; return from[0] === 'W' ? r.winner : r.loser; }
    const m = /^Bracket (\d+) · #(\d+)$/.exec(seat);
    if (m) return names[bracket(cat, +m[1])[+m[2] - 1]];
    return names[runnersUp(cat)[{ 'Best runner-up': 0, '2nd-best runner-up': 1, '3rd-best runner-up': 2 }[seat]]];
  };

  const rows = [], playoffRows = {};
  let num = 0;
  for (const [time, courts] of slots) {
    for (const entry of courts.split(' ')) {
      const [court, c1, c2] = entry.split(':');
      const playoff = !isRoundRobin(c1);
      const p1 = playoff ? seatNames(c1) : names[c1], p2 = playoff ? seatNames(c2) : names[c2];
      const [s1, s2] = score();
      rows.push([++num, '', time, `Court ${court}`, c1, p1[0], p1[1], s1, c2, p2[0], p2[1], s2]);
      if (!playoff) { tally(c1, s1, s2); tally(c2, s2, s1); continue; }
      const cat = catOf(c1), lr = ladder(cat).find(r => r[0] === c1.slice(cat.length + 1));
      results[`${cat} ${lr[1]}`] = s1 > s2 ? { winner: p1, loser: p2 } : { winner: p2, loser: p1 };
      for (const [code, p, a, b] of [[c1, p1, s1, s2], [c2, p2, s2, s1]]) {
        const r = ladder(cat).find(x => x[0] === code.slice(cat.length + 1));
        playoffRows[code] = [code, p[0], p[1], a > b ? 1 : 0, a > b ? 0 : 1, (a / Math.max(b, 1)).toFixed(4), '', r[1], r[2] || '', r[3] || ''];
      }
    }
  }

  // STANDINGSCSV, as the workbook writes it: each category's pairs, then its ladder's seats.
  const standings = [];
  for (const cat of Object.keys(cats)) {
    for (const code of Object.keys(names).filter(k => catOf(k) === cat)) {
      const r = rec[code];
      standings.push([code, names[code][0], names[code][1], r.w, r.l, (r.pf / r.pa).toFixed(4), bracketOf[code], '', '', '']);
    }
    for (const [slot, match, from = '', seat = ''] of ladder(cat)) {
      const code = `${cat}_${slot}`;
      if (playoffRows[code]) { standings.push(playoffRows[code]); continue; }
      const p = seatNames(code);
      standings.push([code, p[0], p[1], 0, 0, '0.0000', '', match, from, seat]);
    }
  }

  const csv = table => table.map(r => r.map(v => /[,"]/.test(String(v)) ? `"${v}"` : v).join(',')).join('\r\n');
  const matchesCsv = csv([['matchNumber', 'court', 'Schedule', 'CourtAssignment', 'teamCode1', 'team1Player1', 'team1Player2', 'team1Score', 'teamCode2', 'team2Player1', 'team2Player2', 'team2Score'], ...rows]);
  const standingsCsv = csv([['teamCode', 'player1', 'player2', 'wins', 'loss', 'quotient', 'bracket', 'match', 'from', 'seat'], ...standings]);
  const at = day.endsWith('1') ? '2026-11-28T09:30:00.000Z' : '2026-11-29T08:00:00.000Z';
  const snapshot = {
    day, label, isLive: true, generatedAt: at, publishedAt: at,
    facilities: [{ name: FACILITY, matchesCsv, standingsCsv, syncedAt: at, lastEditAt: at }],
    failedFacilities: [], staleFacilities: [],
  };
  const podium = cat => {
    const f = results[`${cat} F`], b = results[`${cat} B`];
    const bronze = b ? b.winner : seatNames(`${cat}_B_1`);
    return `${cat} ${f.winner[0].trim()}/${bronze[0].trim()}`;
  };
  const summary = `${rows.length} matches, ${Object.keys(names).length} pairs; gold/bronze ${Object.keys(cats).map(podium).join(', ')}`;
  return { snapshot, summary };
}

const day1 = buildDay({ day: 'generation-cup-day1', label: 'Nov 28 · Next Gen', slots: DAY1_SLOTS, cats: DAY1_CATS, seed: 20261128 });
const day2 = buildDay({ day: 'generation-cup-day2', label: 'Nov 29 · Gen X', slots: DAY2_SLOTS, cats: DAY2_CATS, seed: 20261129 });

fs.mkdirSync(path.join('_fixtures', EVENT), { recursive: true });
fs.writeFileSync(path.join('_fixtures', EVENT, 'day1-finished.json'), JSON.stringify(day1.snapshot, null, 2) + '\n');
fs.writeFileSync(path.join('_fixtures', EVENT, 'day2-finished.json'), JSON.stringify(day2.snapshot, null, 2) + '\n');

const configPath = path.join('_fixtures', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.events[EVENT] = {
  type: 'standard', title: 'The Ultimate Generation Cup 2026', attendance: 'desks', scoreEntry: 'console', scoring: { to: 11, winBy: 1 },
  days: {
    'generation-cup-day1': { label: 'Nov 28 · Next Gen', date: '2026-11-28', isLive: 'auto', facilities: [{ name: FACILITY, sheetId: 'FIXTUREGENCUPDAY1SHEETID000000000000' }] },
    'generation-cup-day2': { label: 'Nov 29 · Gen X', date: '2026-11-29', isLive: 'auto', facilities: [{ name: FACILITY, sheetId: 'FIXTUREGENCUPDAY2SHEETID000000000000' }] },
  },
  display: {
    divisions: { N: 'Novice', LI: 'Low Intermediate', HI: 'High Intermediate', A: 'Advanced', O: 'Open' },
    events: { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' },
  },
};
fs.writeFileSync(configPath, JSON.stringify(config, null, 2) + '\n');

console.log(`Day 1: ${day1.summary}`);
console.log(`Day 2: ${day2.summary}`);
```

It must print the two lines quoted in §5.8, the same on every run. Day 1's `HIMD_F_2` row carries
`F` and `W:SF-2`, as the workbook will once §9.1's fix is made.
