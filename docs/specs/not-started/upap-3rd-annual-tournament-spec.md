# Spec — UPAP 3rd Annual Tournament

> **Status: not started.** Written 2026-11-11 against `sage-match-control.github.io` at
> `628412c`, `event-data`'s `config/events.json` at `3de92502`, and the event workbook as read
> that day through the Sheets API. The owner answered every open question the same day; the
> answers are in §9.
>
> **Built in three phases** (§0.3): the schedule board with the event's `events.json` entry,
> then the Tournament Hub, then everything else. Each phase is its own branch, its own checks
> and its own push.
>
> **Dry-run on 2026-11-11.** Every edit of §4.3–§4.4 and §5.3–§5.5 was applied to a detached
> worktree of the site at `628412c`: every anchor matched exactly once. Appendix A's fixture
> was built and the board, the Hub, Control Center and the desk page were checked in a browser
> at 375px, 1280px and 1440px; the checks in §4.8, §5.8 and §6.3 are what they showed. The
> site's unit tests passed after each phase's test change (573, then 576). §4.6's
> `events.json` script was run on a copy of the file and printed what §4.6 expects. Not run in
> the dry run: `npm run verify`'s screenshot comparison, and any commit or push.
>
> **Deadline.** The event is on **Sunday 22 November 2026**. The site repo takes no merge within
> 3 days of an event, so **all three phases reach the site's `main` by Wednesday 18 November**.
> After that, push only with the owner's explicit go-ahead.

Build the event pages for the **UPAP 3rd Annual Tournament**, a one-day doubles tournament run
by the **UP Alumni Pickleball Community (UPAP)** on **Sunday 22 November 2026** at **The Foundry
Pickleball Court, Cainta, Rizal**, and register the event so the live system knows it.

The pages are copies of the site's standard event template, filled in for this event and
**re-skinned from the organiser's pubmat** (its poster): new colours, a rounded display font
in the Hub's header, and the UPAP logo's four colours. **No shared code changes.**

---

## 0. Briefing — read all of it before you change anything

### 0.1 What you are working in

| Thing | Where | What it is |
| --- | --- | --- |
| **The site repo** | `D:\Personal\SAGE\sage-match-control.github.io` | A GitHub Pages site: hand-written HTML and native JavaScript modules, no build step, no framework. Pushing its `main` branch publishes it within a minute or two |
| **The engine** | `lib/v1/` in the site repo | Shared JavaScript and CSS that every event page loads. **You never edit it** |
| **The template** | `_templates/standard-tournament-template/` (`index.html`, `schedule.html`) | The starting point for a standard event's pages. Each is a short HTML "shell": markup, a `<style>` block of colours, and a small settings script that calls the engine. **You copy it, never edit it** |
| **The `event-data` repo** | `D:\Personal\SAGE\event-data` | The shared data store. Its `config/events.json` is the registry of every event, its days and its workbooks. The live API reads it, so adding this event there is what "registers" it. Its `main` also receives automatic sync commits from the API, so always `git pull` right before you change it |
| **The workbook** | Google Sheet `1CS7oCb748BYnqzNSB5Dmeb_pYvAMBLNk7E7iZ8SavJs` | Where the operators keep the schedule and scores. Once set up (§8.2) it publishes to `event-data`, and the pages read that. You don't touch it |
| **Control Center** | `tools/control-center.html` in the site repo | The operators' console. It lists every event in `events.json` |

Words this spec uses:

- **Team code**: how the workbook names a pair, `<CATEGORY>_<n>`, e.g. `B45HMD_2`. Playoff
  seats are `<CATEGORY>_F_1` (Final, seat 1), `<CATEGORY>_B_2` (Bronze, seat 2) and so on.
- **Category token**: the part before the first `_` (`B45HMD`). It is a **division** (`B45H`)
  followed by an **event** (`MD`).
- **Fixture mode**: on `localhost` only, adding `?fixture=finished` to an engine page's URL
  makes it read `/_fixtures/config.json` instead of the real `events.json`, and
  `/_fixtures/<event-key>/finished.json` instead of the real published data. Appendix A
  writes a made-up finished day of this event there so you can see the pages full of results.
  **The fixture is never committed.**
- **House colours**: the site's default palette (navy `#14263C`, green `#7CB92C`, off-white
  `#F6F7F2`). A **re-skin** replaces them in a page's `:root { … }` block. The test
  `_tests/unit/house-colours.test.mjs` checks that a re-skinned page still restates certain
  engine rules through its own colours; it lists the pages it checks, and §4.5 and §5.6 add
  this event's two pages to that list.

### 0.2 Rules

- **Find edits by their anchor text, never by line number.** Every edit quotes the exact text
  to find. Each anchor occurs exactly once in its file. If an anchor isn't there, or is there
  more than once, **stop and report**. Don't guess and don't improvise a different edit.
- **Every anchor is a single line.** The site checkout has Windows line endings (CRLF), so a
  search across several lines can miss text that looks identical. Where an edit replaces
  several lines, it names the **first** and the **last** line of the range: replace from the
  start of the first line through the end of the last line. Keep each file's line endings as
  they are (don't convert CRLF to LF or the reverse). One exception to "CRLF": the template's
  `index.html` is LF on disk, so §5.2's copy starts as LF. Converting that copy to CRLF to
  match `schedule.html` is fine. Git stores LF either way, so the commit is the same.
- **Replacement text is given in full** in each fenced block. Copy it exactly, including
  leading spaces. Never retype a value from memory.
- **Use Git Bash** for every command. Python is not installed on this machine; Node is
  (22.23.3 or later).
- **Edit only what the phase names:** files in `events/upap-2026/`, the one list in
  `_tests/unit/house-colours.test.mjs`, and `config/events.json` in `event-data`. Don't edit
  `lib/`, `tools/`, `_templates/`, `_fixtures/` (except by running Appendix A, which you then
  undo) or any other event's folder. If something there seems to need a change, stop and report.
- **One phase at a time.** A phase starts only when the one before it is merged into `main` and
  pushed. If you are asked to do a phase whose predecessor isn't on `main` yet, stop and say so.
- **Commit** where a phase says *Commit*, with the message it gives, followed by a blank line
  and the attribution line your session's instructions give.
- **Never push, and never merge into `main`, without the owner's yes in chat.** Each phase ends
  at a **Stop and ask** step.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.3 The phases

| Phase | Builds | Branch | Ends with |
| --- | --- | --- | --- |
| **1** (§4) | `events/upap-2026/schedule.html` (the schedule board, themed) and the event's entry in `event-data/config/events.json` | `upap-board` | Push of the site **and** `event-data` |
| **2** (§5) | `events/upap-2026/index.html` (the Tournament Hub, themed) and its logo | `upap-hub` | Push of the site |
| **3** (§6) | `events/upap-2026/attendance.html` (the check-in desk page), the dry-run checklist, and the QR image if not already in | `upap-rest` | Push of the site |

Why this order, and what is live in between:

- **After Phase 1** the event is registered: Control Center lists it and the operators can set
  up the workbook's live sync (§8.2), so the board shows real data from then on. Control
  Center's **Open Tournament Hub** button leads to the site's "Page not found" page until
  Phase 2, and **Issue desk link** must not be used until Phase 3 (the link would point at a
  page that doesn't exist yet). Don't hand out the short link before Phase 2.
- **After Phase 2** the public Hub is live at the short link.
- **After Phase 3** everything the event needs from the site is in.

### 0.4 Commands you will use in every phase

Run the site's unit tests (from the site repo root):

```bash
cd _tests && npm test; cd ..
```

Run the full check (unit tests, then a browser comparison of `main` against your branch):

```bash
cd _tests && npm run verify; cd ..
```

If `npm run verify` stops with a message that a Playwright browser is missing, run
`cd _tests && npx playwright install chromium; cd ..` once, then rerun it.

Serve the site for a browser check (from the site repo root; leave it running and stop it
with Ctrl+C when the phase's checks are done):

```bash
npx --yes http-server -p 8000 -c-1
```

It must serve the **repo root**, because the pages load root-absolute paths (`/lib/v1/…`,
`/assets/…`). The host must be `localhost` for fixture mode.

---

## 1. Event facts

From the pubmat and the event workbook (`Title`, `Variables`, `SCHEDULE`, `CSV` tabs).

| Fact | Value |
|---|---|
| Organiser | UP Alumni Pickleball Community (UPAP) |
| Event name | UPAP 3rd Annual Tournament |
| Date | Sunday, 22 November 2026 — one day |
| Venue | The Foundry Pickleball Court, Cainta, Rizal. One facility, **4 courts** (`Court 1`–`Court 4`). The workbook's facility name is exactly **`The Foundry`** |
| First serve | 9:00 AM |
| Matches | 65, numbered 1–65, 25-minute slots from 9:00 AM to 3:40 PM (17 slots) |
| Event key | `upap-2026` (the folder name under `events/`, the key in `events.json`, the folder in `event-data`) |
| Day key | `upap-day1` |
| Short link | `tinyurl.com/SAGExUPAP2026` → `https://sage-match-control.github.io/events/upap-2026/` |
| Not on the site | The entry fee and anything about registration |

**Categories.** Eight, each a **single bracket**. The division part of the code is age group
and DUPR level: `B45` below 45, `A45` 45 and above, `L` DUPR 3.25 and below, `H` DUPR 3.26
and above, `O` open.

| Code | The workbook's name | Pairs | Round robin | Playoff |
|---|---|---|---|---|
| `B45HMD` | Below 45 Men's Doubles (DUPR 3.26 and above) | 3 | **double**: every pairing twice (6 matches) | Final; RR #3 takes bronze without playing |
| `OWD` | Open Women's Doubles | 5 | single (10) | Final and Bronze |
| `B45LXD` | Below 45 Mixed Doubles (DUPR 3.25 and below) | 4 | single (6) | Final and Bronze |
| `B45HXD` | Below 45 Mixed Doubles (DUPR 3.26 and above) | 4 | single (6) | Final and Bronze |
| `A45LMD` | 45 and Above Men's Doubles (DUPR 3.25 and below) | 4 | single (6) | Final and Bronze |
| `A45HMD` | 45 and Above Men's Doubles (DUPR 3.26 and above) | 4 | single (6) | Final and Bronze |
| `A45LXD` | 45 and Above Mixed Doubles (DUPR 3.25 and below) | 3 | **double** (6) | Final; RR #3 takes bronze without playing |
| `A45HXD` | 45 and Above Mixed Doubles (DUPR 3.26 and above) | 3 | **double** (6) | Final; RR #3 takes bronze without playing |

30 pairs. Every Final and Bronze is one game: Final RR #1 v RR #2, Bronze RR #3 v RR #4. In a
3-pair category the workbook lists the Bronze seats as RR #3 against `BYE`, with no match
scheduled; the engine already reads that as RR #3's bronze. The shared engine already handles
double round robins and these playoffs; nothing in this spec changes how they work.

**The day runs in two waves** (the workbook's `SCHEDULE` tab):

| Slots | Court 1 | Court 2 | Court 3 | Court 4 |
|---|---|---|---|---|
| 9:00 AM – 12:20 PM (matches 1–35) | `B45HMD`, then `OWD` | `OWD` | `B45LXD` / `B45HXD` | `B45LXD` / `B45HXD` |
| 12:45 PM – 3:40 PM (matches 36–65) | `A45LXD` | `A45HXD` | `A45LMD` / `A45HMD` | `A45LMD` / `A45HMD` |

Courts left empty: Court 4 at 12:20 PM, Courts 1 and 2 at 2:00 PM.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| UPAP logo | **Already in the site repo, untracked**, at `events/upap-2026/assets/logo.jpg`: the four-block "UP AP" mark, 500×500, white background. Use it as is: no conversion, no cropping. Committed in Phase 2 | §5 |
| Pubmat | **Already in the site repo, untracked**, at `events/upap-2026/assets/pubmat.jpg`. **Reference only: never commit it.** The colours below were sampled from it and the logo | — |
| QR image for the short link | **To follow from the owner** as `events/upap-2026/assets/qr.png`. If it isn't there when a phase runs, carry on and report it missing. Don't generate one | §5 (if present), else §6 |
| Event workbook | `1CS7oCb748BYnqzNSB5Dmeb_pYvAMBLNk7E7iZ8SavJs` | §4.6 |

If `logo.jpg` or `pubmat.jpg` is missing when you start, stop and report.

---

## 3. Design notes (no action — the reasons behind the edits)

1. **The palette**, sampled from the pubmat and the logo, used by both pages:

   | Token | Value | Role, from |
   |---|---|---|
   | `--navy` | `#6B2443` | structure (headers, buttons): the pubmat's plum bands |
   | `--navy-deep` | `#45162B` | shadows, the deepest plum |
   | `--green` | `#A6E01A` | the accent fill (active tab, live dots): the pickleball's green |
   | `--green-dark` | `#4D7A0E` | the accent as text |
   | `--paper` | `#FBF8F4` | page background: the pubmat's cream |
   | `--paper-dim` | `#F1EBE5` | panels on the background |
   | `--line` | `#E0D6CF` | rules and borders |
   | `--ink` | `#3D1A2A` | body text, a near-black plum |
   | `--ink-soft` | `#6E5A63` | secondary text |
   | `--white` | `#FFFFFF` | unchanged |

   The token **names** stay `--navy` and `--green` even though the colours are plum and lime:
   the engine's CSS reads these names, so they can't be renamed. Measured contrast (WCAG):
   white on `--navy` 10.7:1; `--navy` on `--green` 6.8:1; `--ink` on `--paper` 14.4:1;
   `--ink-soft` on `--paper` 6.0:1; `--green-dark` on white 5.1:1. `--green` on white is
   1.6:1, so it is only ever a fill, as the template's green is.

2. **The logo's four colours**, used only in the Hub's header: maroon `#7D1013`, gold `#FEB71D`,
   green `#01563F`, black `#262022`; plus two from the pubmat for the header's diagonal bands,
   plum `#99335B` and olive `#80844F`.

3. **The font.** Comfortaa (Google Fonts) is the closest match to the pubmat's rounded title
   lettering. It is used only for the Hub's "0-0-3" and title, and the board's title.

4. **"0-0-3"** is the pubmat's centrepiece: pickleball's opening score call, with the 3 for
   the third edition. The Hub shows it large, above the title.

5. **The board's colours are chosen here, not copied.** Normally a standard event's board
   copies the category colours from the workbook's `SCHEDULE` tab. This workbook's
   `SCHEDULE` shades every round-robin match grey and every playoff gold or brown, so there is
   nothing to copy. Hue follows the event (Men's reds to the logo's maroon, Women's the logo's
   gold, Mixed greens to the logo's green), and the lower DUPR level gets the lighter shade.
   The four categories of each wave are four clearly different colours.

6. **The category labels** (owner's choice): "Below 45 (DUPR ≤3.25)", "Below 45 (DUPR 3.26+)",
   "Open", "45+ (DUPR ≤3.25)", "45+ (DUPR 3.26+)". The engine shows `<division> <event>`, e.g.
   "Below 45 (DUPR 3.26+) Men's Doubles". It finds the division by the **longest** matching
   prefix, so `O` only ever matches `OWD`.

---

## 4. Phase 1 — the schedule board and `events.json`

### 4.1 Start

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git status --short
```

This must print exactly:

```
?? events/upap-2026/
```

and, if it is still there, also `?? events/archives/ladies-dink-2026/assets/pubmat.webp` (an
unrelated untracked file; leave it alone). Anything else: stop and report.

```bash
git checkout main
git pull
git checkout -b upap-board
cd _tests && npm install && npm test; cd ..
```

`npm test` must pass before you change anything. If it doesn't, stop and report.

### 4.2 Copy the board and fill in its tokens

```bash
cp _templates/standard-tournament-template/schedule.html events/upap-2026/schedule.html
```

In `events/upap-2026/schedule.html`, replace **every** occurrence of each token:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `upap-2026` |
| `{{EVENT_TITLE}}` | `UPAP 3rd Annual Tournament` |
| `{{SCHEDULE_DAY_KEY}}` | `upap-day1` |

**Check:** this must print nothing:

```bash
grep -n '{{' events/upap-2026/schedule.html
```

### 4.3 The board's theme

All edits in §4.3 are in `events/upap-2026/schedule.html`.

**(a) The theme comment.** Replace from the line

```css
  /* Theme lifted from the SAGE tools (scoresheet-generator / tournament-calculator):
```

through the line

```css
     Barlow Condensed labels, Inter body. */
```

(three lines in all) with:

```css
  /* The UPAP 3rd Annual Tournament palette, the same tokens as this event's
     index.html (sage-docs/docs/specs/.../upap-3rd-annual-tournament-spec.md §3):
     plum structure, the ball's green accent, cream paper. The event title in the
     bezel takes the pubmat's rounded face, Comfortaa; the rest of the board keeps
     Archivo Black, Barlow Condensed and Inter. */
```

**(b) The colours.** Replace from the line

```css
    --navy:#14263C;
```

through the line

```css
    --radius:14px;
```

(eleven lines in all) with:

```css
    --navy:#6B2443;
    --navy-deep:#45162B;
    --green:#A6E01A;
    --green-dark:#4D7A0E;
    --paper:#FBF8F4;
    --paper-dim:#F1EBE5;
    --line:#E0D6CF;
    --ink:#3D1A2A;
    --ink-soft:#6E5A63;
    --white:#FFFFFF;
    --radius:14px;
```

**(c) The font link.** Find:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

Replace with:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Comfortaa:wght@700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

**(d) The title's font.** Find (the file has exactly one):

```html
</style>
```

Replace with these three lines:

```css
  /* UPAP: the event title in the pubmat's rounded face. */
  .bezel h1{ font-family:'Comfortaa',sans-serif; font-weight:700; }
</style>
```

Leave the block headed **lib/v1/css/schedule-board.css's house colours, through this page's
tokens** exactly as it is.

### 4.4 The board's category colours

In `events/upap-2026/schedule.html`, replace from the line

```js
// Category colours are the organiser's own, read off the colour-coded SCHEDULE
```

through the first line after it that is exactly

```js
};
```

(the template's comment block, its `EXAMPLE — replace` line and its example entries `LIWD` to
`HIXD`, ending with the `};` that closes `const CAT_META = {`) with:

```js
// Category colours are ours, not the organiser's: this workbook's SCHEDULE tab shades every
// round-robin match grey and every playoff gold or brown, so there is nothing to copy
// (sage-docs/docs/specs/.../upap-3rd-annual-tournament-spec.md §3). Hue = event, anchored on the
// UPAP mark: Men's reds to its maroon, Women's its gold, Mixed greens to its green. The lower
// DUPR level is the lighter shade. Keys are the category token of each team code
// (B45HMD_1 -> B45HMD). Cells apply the hue as a 40% tint over white.
const CAT_META = {
  B45LXD: { short:'B45 LO XD', color:'#93C47D' },
  B45HMD: { short:'B45 HI MD', color:'#CC0000' },
  B45HXD: { short:'B45 HI XD', color:'#38761D' },
  OWD:    { short:'OPEN WD',   color:'#FEB71D' },
  A45LMD: { short:'45+ LO MD', color:'#E06666' },
  A45LXD: { short:'45+ LO XD', color:'#6AA84F' },
  A45HMD: { short:'45+ HI MD', color:'#7D1013' },
  A45HXD: { short:'45+ HI XD', color:'#01563F' }
};
```

Leave the `mountScheduleBoard({ … })` call below it unchanged.

**Checks.** Each command's expected output follows it.

```bash
grep -n "EXAMPLE\|LIWD\|HIXD\|#14263C\|#7CB92C\|#F6F7F2" events/upap-2026/schedule.html
```

Prints nothing.

```bash
grep -c "short:'B45\|short:'OPEN\|short:'45+" events/upap-2026/schedule.html
```

Prints `8`.

```bash
grep -c "Comfortaa" events/upap-2026/schedule.html
```

Prints `3` (the font link, the comment, the rule).

### 4.5 Add the board to the house-colours test

In `_tests/unit/house-colours.test.mjs`, find:

```js
  'events/archives/ladies-dink-2026/schedule.html',
```

Replace with these two lines:

```js
  'events/archives/ladies-dink-2026/schedule.html',
  'events/upap-2026/schedule.html',
```

Then run the unit tests (§0.4). They must pass; the change adds three tests.

### 4.6 Register the event in `event-data`

```bash
cd /d/Personal/SAGE/event-data
git checkout main
git pull
git status --short
```

`git status` must print nothing. Anything else: stop and report.

**Check that the day key is free.** Day keys must be unique across every event in the file; a
duplicate makes the whole file fail validation and the live API keeps the last good copy, so
the event would never register.

```bash
grep -c '"upap-day1"\|"upap-2026"' config/events.json
```

Must print `0`. Anything else: stop and ask.

**Add the entry.** The file is exactly `JSON.stringify(…, null, 2)` with no newline at the end,
so this script adds the entry and leaves every other line as it was. Save it **outside the
repo** (for example in your scratchpad) as `add-upap-event.mjs`:

```js
// Adds the UPAP 3rd Annual Tournament to event-data/config/events.json, after the last event.
// Run from the event-data repo root: node <path>/add-upap-event.mjs
import fs from 'node:fs';

const path = 'config/events.json';
const raw = fs.readFileSync(path, 'utf8');
const config = JSON.parse(raw);
if (JSON.stringify(config, null, 2) !== raw) throw new Error('events.json is not in JSON.stringify(…, null, 2) form: stop and report');
if (config.events['upap-2026']) throw new Error('upap-2026 is already registered: stop and report');

config.events['upap-2026'] = {
  type: 'standard',
  title: 'UPAP 3rd Annual Tournament',
  attendance: 'desks',
  scoreEntry: 'console',
  days: {
    'upap-day1': {
      label: 'Nov 22',
      date: '2026-11-22',
      isLive: 'auto',
      facilities: [
        { name: 'The Foundry', sheetId: '1CS7oCb748BYnqzNSB5Dmeb_pYvAMBLNk7E7iZ8SavJs' },
      ],
    },
  },
  display: {
    divisions: {
      B45L: 'Below 45 (DUPR \u22643.25)',
      B45H: 'Below 45 (DUPR 3.26+)',
      O: 'Open',
      A45L: '45+ (DUPR \u22643.25)',
      A45H: '45+ (DUPR 3.26+)',
    },
    events: { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' },
  },
};

fs.writeFileSync(path, JSON.stringify(config, null, 2));
console.log(Object.keys(config.events).slice(-2).join(' then '));
```

Run it:

```bash
node <path-to>/add-upap-event.mjs
```

It must print `ladies-dink-2026 then upap-2026`.

What the entry means:

- `"attendance": "desks"`: staff check players in on a desk page (built in Phase 3).
- `"scoreEntry": "console"`: scores are entered from Control Center only; there is no scorer page.
- `"isLive": "auto"`: the Hub shows live scores from 4 hours before the first match.
- `"name": "The Foundry"`: must equal the workbook's facility name exactly; live-sync setup
  compares them.
- `\u2264` is the `≤` character; the file stores it as `≤`.
- The order of the `divisions` keys sets the display order: Below 45 (the morning wave, lower
  level first), then Open, then 45+.

**Check:**

```bash
git diff --stat
node -e "const c=JSON.parse(require('fs').readFileSync('config/events.json','utf8'));const e=c.events['upap-2026'];console.log(e.type, Object.keys(e.days).join(), e.days['upap-day1'].facilities[0].name, Object.values(e.display.divisions).join(' | '))"
```

`git diff --stat` must show only `config/events.json`, with 33 lines added and none removed.
The second command must print:

```
standard upap-day1 The Foundry Below 45 (DUPR ≤3.25) | Below 45 (DUPR 3.26+) | Open | 45+ (DUPR ≤3.25) | 45+ (DUPR 3.26+)
```

*Commit* in `event-data` (on `main`, which is how that repo is worked), **don't push yet**:

```bash
git add config/events.json
git commit -m "UPAP 3rd Annual Tournament: register upap-2026 (day upap-day1, The Foundry)"
cd /d/Personal/SAGE/sage-match-control.github.io
```

### 4.7 Commit the site's Phase 1

```bash
git add events/upap-2026/schedule.html _tests/unit/house-colours.test.mjs
git status --short
```

`git status` must show those two files staged (`A ` and `M `), `?? events/upap-2026/assets/`
untracked, and nothing else (apart from the Ladies' Dink file of §4.1). *Commit*:

```bash
git commit -m "UPAP 3rd Annual Tournament: schedule board, re-skinned from the pubmat"
```

### 4.8 Check the board and Control Center in a browser

From the site repo root, save Appendix A's script **outside the repo** as
`make-upap-fixture.mjs` and run it:

```bash
node <path-to>/make-upap-fixture.mjs
```

It must print exactly:

```
65 matches, 30 pairs; gold/bronze B45HMD BHM 3A/BHM 1A, OWD OW 3A/OW 2A, B45LXD BLX 4A/BLX 1A, B45HXD BHX 2A/BHX 1A, A45LMD ALM 3A/ALM 4A, A45HMD AHM 1A/AHM 2A, A45LXD ALX 1A/ALX 3A, A45HXD AHX 1A/AHX 2A
```

Serve the site (§0.4) and open the board at **1440px** wide:
`http://localhost:8000/events/upap-2026/schedule.html?fixture=finished`

- [ ] The top bar is plum; the title "UPAP 3rd Annual Tournament" is in a rounded font; the
      meta text reads "NOV 22 · 65 MATCHES · 17 SLOTS · ALL COURTS".
- [ ] The court filter offers **ALL**, 1, 2, 3, 4, and the time column runs 9:00 AM to 3:40 PM.
- [ ] The legend shows eight chips: `B45 HI MD`, `OPEN WD`, `B45 LO XD`, `B45 HI XD`,
      `45+ LO XD`, `45+ HI XD`, `45+ LO MD`, `45+ HI MD` (in the order the categories first
      appear in the day, not §4.4's order).
- [ ] Morning: court 1 red (`B45 HI MD`) with gold `OPEN WD` matches, court 2 gold, courts 3–4
      alternating light and dark green. From 12:45 PM: court 1 mid green, court 2 dark green
      (`45+ HI XD`), courts 3–4 light red and maroon.
- [ ] Exactly three cells read **OPEN**: court 4 at 12:20 PM, courts 1 and 2 at 2:00 PM.
- [ ] Eight cells carry a **FINAL** tag and five a **BRONZE** tag.

Open Control Center:
`http://localhost:8000/tools/control-center.html?fixture=finished&event=upap-2026&day=upap-day1`

- [ ] The header reads "UPAP 3RD ANNUAL TOURNAMENT". Mission Control shows **The Foundry**,
      "65 / 65 done · All matches done", and "Auto — goes live 5:00 AM, Nov 22 (4h before
      first serve, 9:00 AM)".
- [ ] **Standings** lists the categories as "Below 45 (DUPR ≤3.25) Mixed Doubles", "Below 45
      (DUPR 3.26+) Men's Doubles", "Below 45 (DUPR 3.26+) Mixed Doubles", "Open Women's
      Doubles", "45+ (DUPR ≤3.25) Men's Doubles", "45+ (DUPR ≤3.25) Mixed Doubles", "45+ (DUPR
      3.26+) Men's Doubles", "45+ (DUPR 3.26+) Mixed Doubles", in that order. No raw code like
      `B45HMD` appears, and there is **no** *Unmapped category* warning anywhere.
- [ ] **Awards** reads "8 of 8 podiums complete", with gold, silver, bronze: Below 45 (DUPR
      ≤3.25) Mixed Doubles BLX 4A, BLX 3A, BLX 1A; Below 45 (DUPR 3.26+) Men's Doubles BHM 3A,
      BHM 2A, BHM 1A; Below 45 (DUPR 3.26+) Mixed Doubles BHX 2A, BHX 3A, BHX 1A; Open Women's
      Doubles OW 3A, OW 5A, OW 2A; 45+ (DUPR ≤3.25) Men's Doubles ALM 3A, ALM 2A, ALM 4A; 45+
      (DUPR ≤3.25) Mixed Doubles ALX 1A, ALX 2A, ALX 3A; 45+ (DUPR 3.26+) Men's Doubles AHM 1A,
      AHM 4A, AHM 2A; 45+ (DUPR 3.26+) Mixed Doubles AHX 1A, AHX 3A, AHX 2A. (The three 3-pair
      categories' bronzes went to RR #3 without a Bronze match: that is correct.)
- [ ] **Attendance** reads "The Foundry: No roster yet".
- [ ] **Expected, not bugs:** 404s in the browser's network log for `scorer.html`,
      `scorer-beta.html` and a fixture attendance CSV; and **Open Tournament Hub** leading to
      "Page not found" (the Hub is Phase 2).

Stop the server. Then remove the fixture:

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/upap-2026
git status --short
```

`git status` must show nothing under `_fixtures/`.

### 4.9 Full check

Run `npm run verify` (§0.4). It must pass. A failure means something outside this phase's two
files changed: run `git status` and stop and report.

### 4.10 Stop and ask: push Phase 1

Tell the owner Phase 1 is ready, with the results of §4.8 and §4.9, and that pushing it will
(1) publish the board and (2) register the event, so it appears in Control Center within about
a minute. Ask whether to push both. On a yes:

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git merge --ff-only upap-board
git push
cd /d/Personal/SAGE/event-data
git pull --rebase
git push
cd /d/Personal/SAGE/sage-match-control.github.io
```

If `git merge --ff-only` refuses, or `git pull --rebase` reports a conflict, stop and report.
(`event-data` gets automatic sync commits, so the rebase is normal and usually clean.)

**Check, a minute or two after the push:**
`https://sage-match-control.github.io/events/upap-2026/schedule.html` loads with the plum top
bar (it reads "No schedule" or similar until the workbook's sync is set up, §8.2), and Control
Center at `https://sage-match-control.github.io/tools/control-center.html` lists "UPAP 3rd
Annual Tournament".

---

## 5. Phase 2 — the Tournament Hub

### 5.1 Start

Phase 1 must be on `main` (§0.2).

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
git checkout -b upap-hub
```

`git status` must print only `?? events/upap-2026/assets/` (and the Ladies' Dink file of
§4.1). Anything else: stop and report.

### 5.2 Copy the Hub and fill in its tokens

```bash
cp _templates/standard-tournament-template/index.html events/upap-2026/index.html
```

In `events/upap-2026/index.html`, replace **every** occurrence of each token:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `upap-2026` |
| `{{EVENT_TITLE}}` | `UPAP 3rd Annual Tournament` |
| `{{EVENT_TAGLINE}}` | `UP Alumni Pickleball Community` |
| `{{EVENT_HEADLINE}}` | `0-0-3` |
| `{{EVENT_DATE_RANGE}}` | `Sunday, 22 November 2026` |
| `{{VENUE}}` | `The Foundry, Cainta` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr>SAGExUPAP2026` |
| `{{EVENT_LOGO}}` | `assets/logo.jpg` |

`<wbr>` is HTML, typed exactly as shown: it lets the link wrap after the `/`.

**Check:** this must print nothing:

```bash
grep -n '{{' events/upap-2026/index.html
```

### 5.3 The Hub's content

All edits in §5.3–§5.5 are in `events/upap-2026/index.html`. The lines quoted below are as
they read **after** §5.2.

**(a) The logo's alt text.** Find:

```html
    <img src="assets/logo.jpg" alt="UPAP 3rd Annual Tournament logo" />
```

Replace with:

```html
    <img src="assets/logo.jpg" alt="UP Alumni Pickleball Community logo" />
```

**(b) The header's order.** The pubmat reads, top to bottom: the community's name, "0-0-3",
"3rd Annual Tournament", then the date and the venue. Three edits, **in this order**:

1. Delete this whole line (including its line ending):

   ```html
     <div class="domination-line">0-0-3</div>
   ```

2. Delete this whole line:

   ```html
     <div class="eyebrow">Sunday, 22 November 2026 &nbsp;·&nbsp; The Foundry, Cainta</div>
   ```

3. Find:

   ```html
     <h1 class="title">UPAP 3rd Annual Tournament</h1>
   ```

   Replace that one line with these four lines:

   ```html
     <div class="domination-line">0-0-3</div>
     <h1 class="title"><span class="accent">3rd</span> Annual Tournament</h1>
     <div class="upap-bar" aria-hidden="true"><span></span><span></span><span></span><span></span></div>
     <div class="eyebrow">Sunday, 22 November 2026<br>The Foundry &nbsp;·&nbsp; Cainta, Rizal</div>
   ```

   (Each of these lines starts with two spaces, like the lines around it.)

**(c) The footer.** Find:

```html
  UPAP 3rd Annual Tournament &middot; Sunday, 22 November 2026 &middot; The Foundry, Cainta<br>
```

Replace with these three lines:

```html
  UPAP 3rd Annual Tournament &middot; Sunday, 22 November 2026<br>
  The Foundry Pickleball Court, Cainta, Rizal<br>
  Organised by the UP Alumni Pickleball Community<br>
```

**(d) The category search's example.** Find:

```html
placeholder="e.g. Beginner 18+ Men's Doubles"
```

Replace with:

```html
placeholder="e.g. Open Women's Doubles"
```

Leave the pair search's placeholder `e.g. John Smith / Jane Doe` as it is.

**Checks.** Each of these must print exactly one line:

```bash
grep -n 'alt="UP Alumni Pickleball Community logo"' events/upap-2026/index.html
grep -n 'class="domination-line"' events/upap-2026/index.html
grep -n '<span class="accent">3rd</span> Annual Tournament' events/upap-2026/index.html
grep -n '<div class="eyebrow">' events/upap-2026/index.html
grep -n 'Organised by the UP Alumni Pickleball Community' events/upap-2026/index.html
grep -n "e.g. Open Women's Doubles" events/upap-2026/index.html
```

And in the file, the `domination-line` line must come directly before the `<h1 class="title">`
line.

### 5.4 The Hub's colours and font

**(a) The theme comment.** Replace from the line

```css
  /* ============================ THEME ============================
```

through the **first** line after it that is exactly

```css
     ================================================================ */
```

(19 lines in all) with:

```css
  /* ============================ THEME ============================
     The UPAP 3rd Annual Tournament pubmat's palette
     (sage-docs/docs/specs/.../upap-3rd-annual-tournament-spec.md §3): plum
     structure from its diagonal bands, the ball's green for the accent,
     its cream centre for paper, and the UPAP mark's maroon, gold, green
     and black in the hero. Shared with this event's schedule.html.

     To re-skin an event, change the brand tokens below and nothing else:
     every rule in this file resolves through them or through the role
     aliases underneath.

     Contrast rules this palette has to respect — the ball green is a
     *fill* colour, not a text colour:
       - --green on white is ~1.6:1. Use it as a background (with --navy
         text on top, ~6.8:1) or on the plum hero (~6.8:1).
       - --green-dark on white is ~5.1:1 and on paper ~4.8:1.
       - Small text on paper is --ink (~14.4:1) or --ink-soft (~6.0:1).
     ================================================================ */
```

**(b) The colours.** Replace from the line

```css
    --navy:#14263C;
```

through the line

```css
    --radius:14px;
```

(eleven lines in all) with:

```css
    --navy:#6B2443;
    --navy-deep:#45162B;
    --green:#A6E01A;
    --green-dark:#4D7A0E;
    --paper:#FBF8F4;
    --paper-dim:#F1EBE5;
    --line:#E0D6CF;
    --ink:#3D1A2A;
    --ink-soft:#6E5A63;
    --white:#FFFFFF;
    --radius:14px;
    /* The pubmat's and the UPAP mark's own colours, for this page's hero only (spec §3). */
    --upap-plum:#99335B;
    --upap-olive:#80844F;
    --upap-maroon:#7D1013;
    --upap-gold:#FEB71D;
    --upap-green:#01563F;
    --upap-black:#262022;
```

Leave everything below it in the `:root` block (`--court`, `--cork`, …, `--card-shadow`) and
the whole block headed **lib/v1/css's house colours, through this page's tokens** exactly as
they are.

**(c) The font link.** Find:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

Replace with:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Comfortaa:wght@700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

### 5.5 The Hub's header styling

Find (the file has exactly one):

```html
</style>
```

Replace it with:

```css
/* ==== UPAP 3rd Annual Tournament: the pubmat's hero (sage-docs/docs/specs/.../upap-3rd-annual-tournament-spec.md §3) ==== */
/* The pubmat's diagonal bands: a lighter plum across the top left, an olive band low on the right. */
header.hero{
  background-color:var(--navy);
  background-image:
    linear-gradient(152deg, color-mix(in srgb, var(--upap-plum) 75%, transparent) 0 28%, transparent 28%),
    linear-gradient(152deg, transparent 0 74%, color-mix(in srgb, var(--upap-olive) 40%, transparent) 74% 88%, transparent 88%);
}
/* "UP Alumni Pickleball Community": the pubmat's condensed capitals, above the score call. */
.tagline-script{ font-family:'Barlow Condensed',sans-serif; font-style:normal; font-weight:700; text-transform:uppercase; letter-spacing:.14em; font-size:clamp(14px,3.6vw,18px); margin:0 0 4px; }
/* "0-0-3": the pubmat's score call, in its rounded face and the ball's green. */
.domination-line{ font-family:'Comfortaa',sans-serif; font-weight:700; font-size:clamp(46px,14vw,88px); line-height:1; letter-spacing:.02em; text-transform:none; color:var(--green); margin:8px 0 6px; }
/* "3rd Annual Tournament": the pubmat's rounded face in mixed case, "3rd" in the UPAP gold. */
h1.title{ font-family:'Comfortaa',sans-serif; font-weight:700; font-size:clamp(26px,7vw,50px); line-height:1.12; letter-spacing:0; text-transform:none; margin:0 0 14px; }
h1.title .accent{ color:var(--upap-gold); }
/* The UPAP mark's four colours on a white chip, as the mark sits on its white ground. */
.upap-bar{ display:inline-flex; gap:3px; padding:4px; border-radius:6px; background:var(--white); margin:0 0 16px; }
.upap-bar span{ width:30px; height:8px; border-radius:2px; }
.upap-bar span:nth-child(1){ background:var(--upap-maroon); }
.upap-bar span:nth-child(2){ background:var(--upap-gold); }
.upap-bar span:nth-child(3){ background:var(--upap-green); }
.upap-bar span:nth-child(4){ background:var(--upap-black); }
/* Date, then venue, under the title, in white like the pubmat's. */
.eyebrow{ color:var(--white); line-height:1.9; margin-bottom:22px; }
</style>
```

Every colour in it goes through a `var(--…)` token; don't write a colour value directly into
these rules (the house-colours test rejects a page that writes the house colours itself).

**Checks.** Each command's expected output follows it.

```bash
grep -n "#14263C\|#7CB92C\|#F6F7F2" events/upap-2026/index.html
```

Prints nothing.

```bash
grep -c -- "--upap-" events/upap-2026/index.html
```

Prints `13`.

```bash
wc -l < events/upap-2026/index.html
```

Prints `295`.

### 5.6 Add the Hub to the house-colours test

In `_tests/unit/house-colours.test.mjs`, find:

```js
  'events/upap-2026/schedule.html',
```

Replace with these two lines:

```js
  'events/upap-2026/schedule.html',
  'events/upap-2026/index.html',
```

Run the unit tests (§0.4). They must pass.

### 5.7 Commit the site's Phase 2

If the owner has supplied `events/upap-2026/assets/qr.png`, include it:

```bash
git add events/upap-2026/index.html events/upap-2026/assets/logo.jpg _tests/unit/house-colours.test.mjs
git add events/upap-2026/assets/qr.png 2>/dev/null; true
git status --short
```

`git status` must show those files staged and `?? events/upap-2026/assets/pubmat.jpg`
untracked. **Never stage `pubmat.jpg`.** *Commit*:

```bash
git commit -m "UPAP 3rd Annual Tournament: Tournament Hub, re-skinned from the pubmat"
```

### 5.8 Check the Hub in a browser

Build the fixture again (Appendix A, same command and same output as §4.8), serve the site
(§0.4), and open `http://localhost:8000/events/upap-2026/?fixture=finished`.

At **375px** wide (your browser's phone emulation) and at **1280px**:

- [ ] The header is plum, with a lighter plum band across the top left and an olive band low on
      the right. Top to bottom it reads: the UPAP logo in a white tile, "×", the S.A.G.E. logo;
      "UP ALUMNI PICKLEBALL COMMUNITY" in white capitals; a large **0-0-3** in lime green;
      "**3rd** Annual Tournament" in a rounded font, white with "3rd" in gold; a white chip
      holding four short bars (maroon, gold, green, black); "SUNDAY, 22 NOVEMBER 2026" on one
      line and "THE FOUNDRY · CAINTA, RIZAL" on the next, in white; then the subtitle.
- [ ] At 375px, "3rd Annual Tournament" fits on one line.
- [ ] The tabs: the active one is lime with plum text; the others are outlined in white.
- [ ] No day picker is shown (one day), and the day loads.
- [ ] Below the header the page is cream, the "Find matches" button is plum, and the stat
      numbers read 65, 30 and 8 in dark green.
- [ ] At 1280px only: a QR panel at the top right shows `tinyurl.com/SAGExUPAP2026`, with the QR
      image if `qr.png` was supplied (a broken image otherwise; expected).
- [ ] The footer has four lines: "UPAP 3rd Annual Tournament · Sunday, 22 November 2026", "The
      Foundry Pickleball Court, Cainta, Rizal", "Organised by the UP Alumni Pickleball
      Community", "Powered by S.A.G.E. Match Control Experts".
- [ ] **Match Finder:** type `BHM 3A` in the search box and press **Find matches**. It reads "5
      matches found": matches #5, #9, #21, #25 (Round Robin) and #33 (Final), labelled "Below 45
      (DUPR 3.26+) Men's Doubles".
- [ ] **Standings** at 1280px: eight category columns in the order listed in §4.8's Standings
      check. Each has **Round robin · Bracket 1** then **Playoffs**. The three 3-pair categories
      (Below 45 (DUPR 3.26+) Men's Doubles and both 45+ Mixed) show **Finals** only, no Bronze
      Battle, and each of their pairs has W + L = 4. Below 45 (DUPR 3.26+) Men's Doubles' Finals
      reads BHM 3A / BHM 3B **11 VS 0** BHM 2A / BHM 2B. The other five show **Bronze Battle**
      then **Finals**. No bracket chart is drawn (expected for a single-bracket category).
- [ ] **Expected, not a bug:** a table's order can differ from who played the Final, because
      the fixture seats its playoffs by wins then points while the Hub also uses head-to-head.
      Open Women's Doubles lists OW 5A first and OW 3A second, both 3–1, though OW 3A played
      seat 1 of the Final.
- [ ] **Standings** at 375px: it first asks for a category; the search box's placeholder reads
      "e.g. Open Women's Doubles", and typing `open` offers "Open Women's Doubles".
- [ ] Browser developer tools: the console shows no errors, and the only failed request is
      `assets/qr.png` when it hasn't been supplied.

Stop the server and remove the fixture exactly as at the end of §4.8.

### 5.9 Full check, then stop and ask: push Phase 2

Run `npm run verify` (§0.4); it must pass. Then tell the owner Phase 2 is ready, with the
results of §5.8 and §5.9, and whether `qr.png` was included, and ask whether to push. Pushing
publishes the Hub at the short link's address. On a yes:

```bash
git checkout main
git pull
git merge --ff-only upap-hub
git push
```

If `--ff-only` refuses, stop and report. **Check:** a minute or two later,
`https://sage-match-control.github.io/events/upap-2026/` shows the plum header.

---

## 6. Phase 3 — the desk page, the dry-run checklist and the QR image

### 6.1 Start

Phase 2 must be on `main`.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git checkout main
git pull
git status --short
git checkout -b upap-rest
```

`git status` must print only `?? events/upap-2026/assets/pubmat.jpg`, plus
`?? events/upap-2026/assets/qr.png` if the owner has supplied it since Phase 2 (and the Ladies'
Dink file of §4.1). Anything else: stop and report.

### 6.2 Copy the two files and fill in their tokens

```bash
cp _templates/attendance/attendance.html events/upap-2026/attendance.html
cp _templates/dry-run-checklist-template.md events/upap-2026/dry-run-checklist.md
```

In **both** files, replace every occurrence of each token:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `upap-2026` (only `attendance.html` has it) |
| `{{EVENT_TITLE}}` | `UPAP 3rd Annual Tournament` |

Make **no other change** to either file. The desk page keeps the standard S.A.G.E. colours, as
every event's desk page does.

**Check:** this must print nothing:

```bash
grep -rnI '{{' events/upap-2026/
```

### 6.3 Check the desk page

Serve the site (§0.4) and open
`http://localhost:8000/events/upap-2026/attendance.html` (no fixture needed).

- [ ] The browser tab reads "UPAP 3rd Annual Tournament — Attendance", and the page reads "Ask
      the operator for a desk link." in the standard S.A.G.E. colours (navy and green). A real
      link needs the live system; the operators' dry run checks one (§8.5).

Stop the server.

### 6.4 Commit, full check, then stop and ask: push Phase 3

```bash
git add events/upap-2026/attendance.html events/upap-2026/dry-run-checklist.md
git add events/upap-2026/assets/qr.png 2>/dev/null; true
git status --short
git commit -m "UPAP 3rd Annual Tournament: attendance desk page and dry-run checklist"
```

`git status` before the commit must show only those files staged and
`?? events/upap-2026/assets/pubmat.jpg` untracked. Then run `npm run verify` (§0.4); it must
pass. Tell the owner Phase 3 is ready and ask whether to push. On a yes:

```bash
git checkout main
git pull
git merge --ff-only upap-rest
git push
```

If `qr.png` still hasn't been supplied, say so: when it arrives, it is committed and pushed on
its own (`git add events/upap-2026/assets/qr.png`, commit "UPAP 3rd Annual Tournament: QR
image", push with the owner's yes).

---

## 7. Report back, at the end of each phase

- Which checks of the phase passed, and any that didn't, with what you saw.
- That the fixture was removed (Phases 1 and 2).
- Whether `qr.png` was present.
- What was pushed, and to which repo, only after the owner's yes.
- `git status --short` in every repo you touched (`pubmat.jpg` untracked is expected).

---

## 8. Operator tasks — people, not the implementer

These happen in Google Sheets and on the day. They are written down so the constraints they
put on the data are known. The workbook was read on 11 November 2026.

### 8.1 Workbook checks before sync setup

The workbook is generated and packed: `SCHEDULE` and `CSV` hold all 65 matches, numbered 1–65,
with times `h:mm AM/PM` and courts `Court 1`–`Court 4`; `STANDINGSCSV` lists every pair and
playoff seat; the `ATTENDANCE` tab has its header. What is left:

1. **Generator version.** Confirm **SAGE → Help** in this workbook lists Standard Tournament
   Generator 1.5.0. Until the Standard Tournament Master itself carries 1.5.0, a fresh copy
   from it would build a 3-pair category as a single round robin without warning: don't
   regenerate this event.
2. **Rosters.** Every pair still shows its code (`B45HMD_1`, …) as its player name. Paste the
   names into each category tab, then run **Update roster** from Control Center's Attendance
   tab (after Phase 1).
3. **`Court Control` shows `#REF!`** on all 4 courts. Test it: put a match on Court 1 the way
   operators will on the day, and confirm the `#REF!` clears and that match's `court` cell in
   `CSV` fills in. If it doesn't, compare the formulas with the master's `Court Control` and
   repair them. Undo the test afterwards.
4. **Back-to-back matches: the organiser's call.** In each 3-pair category one pair sits out
   every round, so pairs 2 and 3 each play two slots in a row, twice: `B45HMD_2` at 9:00 and
   9:25 AM and at 10:40 and 11:05 AM, `B45HMD_3` at 9:25 and 9:50 AM and at 11:05 and 11:30 AM;
   `A45LXD` and `A45HXD` the same way from 12:45 PM. Three playoffs also follow their
   category's last round-robin slot directly: `OWD` (11:55 AM → 12:20 PM), `A45LXD` and
   `A45HXD` (3:15 → 3:40 PM). The site shows whatever the workbook says.
5. **Players in two categories.** Within a wave the categories run side by side, so a player in
   `B45HMD` and a Below 45 Mixed category, or in `OWD` and a Below 45 Mixed category (and the
   same for the 45+ categories in the afternoon), can be due on two courts in the same slot.
   Once names are in, look for any name in two categories of one wave and check their slots.
6. **Colours.** The board's colours are §4.4's, not the sheet's. Recolouring `SCHEDULE` changes
   nothing on the site.

### 8.2 Share the workbook, then set up live sync — once, after Phase 1 is pushed

1. **Share** the workbook with
   `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**, and set
   **General access** to **Restricted**.
2. **Extensions → Apps Script.** Confirm `sheets-sync.gs` is there. If the **SAGE** menu lacks
   **Set up live sync**, replace the whole file with the current
   `sage-tools-api/apps-script/sheets-sync.gs`, save, and reload the spreadsheet.
3. **SAGE → Set up live sync**, and enter:

   | Field | Value |
   |---|---|
   | Day | `upap-day1` |
   | Venue | `The Foundry` — exact, including capitals |
   | Tabs to watch | `SCHEDULE` and `Court Control` (pre-ticked) |
   | Secret | Only if asked: `SYNC_SHARED_SECRET` from Cloud Run's settings, from whoever manages Cloud Run. Never paste it into chat or a document |

4. **Save.** Setup checks the secret, day and venue, then sends one real test update.

**Confirm it's live:** about a minute later,
`https://sage-match-control.github.io/event-data/upap-2026/data/upap-day1.json` lists one
facility, `The Foundry`, with 65 matches, and the board shows the real schedule.

### 8.3 Go-live

The Hub shows scores and its Live and Standings tabs from **5:00 AM on the day** (4 hours before
the 9:00 AM first match). To change that, use Control Center's go-live override.

### 8.4 The QR panel

After `qr.png` is pushed, print the venue board's QR panel from the site repo:

```bash
node _templates/hub-pubmat/render.mjs upap-2026
```

It writes `qr-panel.pdf` and `board.pdf` to `_templates/hub-pubmat/out/upap-2026/`. Scan the
printed panel before mounting it.

### 8.5 Dry run, by Friday 20 November

Use `events/upap-2026/dry-run-checklist.md` (Phase 3) against the real workbook. Also check:

- Enter both meetings of one `B45HMD` pairing with different winners: each publishes its own
  score, and the Hub's W and L count both.
- Once a 3-pair category's round robin is entered, Control Center's Awards gives its bronze to
  RR #3, and the facility card reads "All matches done" after match 65.
- Control Center shows no *Unmapped category* warning; the Live board shows 4 courts.
- **Issue desk link** produces a link that opens the desk page with the roster.

This also serves as the double round robin spec's §9 step 5 for its Formats A (`OWD`) and B
(`B45HMD`).

### 8.6 After the event

Turn live push off for the pages: set `LIVE_BASE_URL = ''` in `index.html`, `schedule.html`
and `attendance.html`. Leave the event in `events.json`.

---

## 9. Questions, answered by the owner on 11 November 2026

| Question | Answer |
|---|---|
| The short link | `tinyurl.com/SAGExUPAP2026`. The owner creates it and supplies `assets/qr.png` |
| Attendance and score entry | Desk links for check-in (`"attendance": "desks"`), scores from Control Center only (`"scoreEntry": "console"`), no scorer page |
| The category labels | "Below 45 (DUPR ≤3.25)", "Below 45 (DUPR 3.26+)", "Open", "45+ (DUPR ≤3.25)", "45+ (DUPR 3.26+)" |
| The Hub's logo | The four-block UPAP mark (`assets/logo.jpg`), not the pubmat's fist-and-players crest |
| Build order (asked for the same day) | The schedule board with `events.json` first, then the Tournament Hub, then everything else |

First serve is the workbook's 9:00 AM; the pubmat gives no time.

---

## 10. Decisions, for reference

- **Event key `upap-2026`, day key `upap-day1`**: the community's own acronym; the day key
  carries the event's name because day keys must be unique across every event.
- **`events.json` ships with the board**, not the Hub: the board is useless without the event
  registered, and registering early lets the operators set up live sync and rehearse on the
  real board while the Hub is built.
- **Title "UPAP 3rd Annual Tournament"**, the workbook's. "UPAP 3rd Annual" alone isn't the
  event's name, though the browser tab then reads "… Tournament — Tournament Hub".
- **Plum, not the pubmat's red**: red text on plum can't be read. The logo's four colours appear
  only in the header's bar and "3rd".
- **Comfortaa only for the display text**; everything else keeps the template's fonts, which
  the shared CSS sets.
- **The desk page keeps the standard colours**, like every event's: it is a staff page.
- **No entry fee or hours on the page**: the Hub's times come from the workbook.

---

## Appendix A — the fixture script

Used in §4.8 and §5.8, and never committed. Save it **outside the repo** and run it from the
site repo root. It writes `_fixtures/upap-2026/finished.json` and adds the event to
`_fixtures/config.json`; the end of §4.8 says how to remove both.

```js
// Writes a temporary, local-only fixture for the UPAP 3rd Annual Tournament, shaped like the real event:
// the workbook's own SCHEDULE (65 matches on 4 courts, 9:00 AM to 3:40 PM), eight one-bracket categories.
// The 3-pair categories play a double round robin then a Final (RR #3 takes bronze against a BYE, no match);
// the 4- and 5-pair ones play a single round robin then a Final and a Bronze match.
// Every match is played, with fixed made-up scores and player names.
//   _fixtures/upap-2026/finished.json   the snapshot
//   _fixtures/config.json               gains an "upap-2026" entry
// Neither is ever committed. Undo with:
//   git checkout -- _fixtures/config.json && rm -rf _fixtures/upap-2026
// usage, from the site repo root: node <path>/make-upap-fixture.mjs
import fs from 'node:fs';
import path from 'node:path';

const EVENT = 'upap-2026', DAY = 'upap-day1';

// The workbook's SCHEDULE, slot by slot: each slot lists Court 1, Court 2, ... in order, '-' for an empty court.
const SLOTS = [
  ['9:00 AM',  'B45HMD_1:B45HMD_2 OWD_2:OWD_3 B45LXD_1:B45LXD_2 B45LXD_3:B45LXD_4'],
  ['9:25 AM',  'B45HMD_2:B45HMD_3 OWD_4:OWD_5 B45HXD_1:B45HXD_2 B45HXD_3:B45HXD_4'],
  ['9:50 AM',  'B45HMD_1:B45HMD_3 OWD_1:OWD_2 B45LXD_1:B45LXD_3 B45LXD_2:B45LXD_4'],
  ['10:15 AM', 'OWD_1:OWD_4 OWD_3:OWD_5 B45HXD_1:B45HXD_3 B45HXD_2:B45HXD_4'],
  ['10:40 AM', 'B45HMD_2:B45HMD_1 OWD_2:OWD_5 B45LXD_1:B45LXD_4 B45LXD_2:B45LXD_3'],
  ['11:05 AM', 'B45HMD_3:B45HMD_2 OWD_3:OWD_4 B45HXD_1:B45HXD_4 B45HXD_2:B45HXD_3'],
  ['11:30 AM', 'B45HMD_3:B45HMD_1 OWD_1:OWD_5 B45LXD_F_1:B45LXD_F_2 B45LXD_B_1:B45LXD_B_2'],
  ['11:55 AM', 'OWD_2:OWD_4 OWD_1:OWD_3 B45HXD_F_1:B45HXD_F_2 B45HXD_B_1:B45HXD_B_2'],
  ['12:20 PM', 'B45HMD_F_1:B45HMD_F_2 OWD_B_1:OWD_B_2 OWD_F_1:OWD_F_2 -'],
  ['12:45 PM', 'A45LXD_1:A45LXD_2 A45HXD_1:A45HXD_2 A45LMD_1:A45LMD_2 A45LMD_3:A45LMD_4'],
  ['1:10 PM',  'A45LXD_2:A45LXD_3 A45HXD_2:A45HXD_3 A45HMD_1:A45HMD_2 A45HMD_3:A45HMD_4'],
  ['1:35 PM',  'A45LXD_1:A45LXD_3 A45HXD_1:A45HXD_3 A45LMD_1:A45LMD_3 A45LMD_2:A45LMD_4'],
  ['2:00 PM',  '- - A45HMD_1:A45HMD_3 A45HMD_2:A45HMD_4'],
  ['2:25 PM',  'A45LXD_2:A45LXD_1 A45HXD_2:A45HXD_1 A45LMD_1:A45LMD_4 A45LMD_2:A45LMD_3'],
  ['2:50 PM',  'A45LXD_3:A45LXD_2 A45HXD_3:A45HXD_2 A45HMD_1:A45HMD_4 A45HMD_2:A45HMD_3'],
  ['3:15 PM',  'A45LXD_3:A45LXD_1 A45HXD_3:A45HXD_1 A45LMD_F_1:A45LMD_F_2 A45LMD_B_1:A45LMD_B_2'],
  ['3:40 PM',  'A45LXD_F_1:A45LXD_F_2 A45HXD_F_1:A45HXD_F_2 A45HMD_F_1:A45HMD_F_2 A45HMD_B_1:A45HMD_B_2'],
];
// Category -> [name stem, pairs]. In the workbook's order.
const CATS = {
  B45HMD: ['BHM', 3], OWD: ['OW', 5], B45LXD: ['BLX', 4], B45HXD: ['BHX', 4],
  A45LMD: ['ALM', 4], A45HMD: ['AHM', 4], A45LXD: ['ALX', 3], A45HXD: ['AHX', 3],
};
const catOf = code => code.split('_')[0];
const isRoundRobin = code => /^[A-Z0-9]+_\d+$/.test(code);

// Fixed pseudo-random scores: the same file every run.
let seed = 20261122;
const rand = n => { seed = (seed * 1103515245 + 12345) % 2147483648; return Math.floor(seed / 65536) % n; };
const score = () => { const loser = rand(10); return rand(2) ? [11, loser] : [loser, 11]; };

// A pair's players, like the workbook's (each name ends in a space). Playoff seats take their pair's names.
const names = {};
for (const [cat, [stem, pairs]] of Object.entries(CATS)) {
  for (let n = 1; n <= pairs; n++) names[`${cat}_${n}`] = [`${stem} ${n}A `, `${stem} ${n}B `];
}

const rec = {};   // code -> { w, l, pf, pa }, both legs of a double round robin counted
const tally = (code, pf, pa) => {
  const r = rec[code] || (rec[code] = { w: 0, l: 0, pf: 0, pa: 0 });
  r.pf += pf; r.pa += pa; if (pf > pa) r.w++; else r.l++;
};
// A category's round-robin order: wins, then points quotient. The playoff seats follow it.
const ranked = cat => Object.keys(rec).filter(k => catOf(k) === cat && isRoundRobin(k))
  .sort((x, y) => rec[y].w - rec[x].w || rec[y].pf / rec[y].pa - rec[x].pf / rec[x].pa);
const BYE = ['BYE ', 'BYE '];
const seat = code => {
  const cat = catOf(code), order = ranked(cat);
  const i = { F_1: 0, F_2: 1, B_1: 2, B_2: 3 }[code.slice(cat.length + 1)];
  return order[i] ? names[order[i]] : BYE;
};

const winnerOf = {}, loserOf = {};   // "<CAT>_F" / "<CAT>_B" -> names
const rows = [];
const playoffRows = {};              // code -> STANDINGSCSV row
let num = 0;
SLOTS.forEach(([time, courts]) => {
  courts.split(' ').forEach((pair, i) => {
    if (pair === '-') return;
    const [c1, c2] = pair.split(':');
    const playoff = !isRoundRobin(c1);
    const p1 = playoff ? seat(c1) : names[c1], p2 = playoff ? seat(c2) : names[c2];
    const [s1, s2] = score();
    rows.push([++num, '', time, `Court ${i + 1}`, c1, p1[0], p1[1], s1, c2, p2[0], p2[1], s2]);
    if (playoff) {
      const stage = c1.slice(0, c1.lastIndexOf('_'));   // B45HMD_F
      const kw = stage.slice(stage.lastIndexOf('_') + 1);
      winnerOf[stage] = s1 > s2 ? p1 : p2; loserOf[stage] = s1 > s2 ? p2 : p1;
      playoffRows[c1] = [c1, p1[0], p1[1], s1 > s2 ? 1 : 0, s1 > s2 ? 0 : 1, (s1 / Math.max(s2, 1)).toFixed(4), '', kw, '', ''];
      playoffRows[c2] = [c2, p2[0], p2[1], s2 > s1 ? 1 : 0, s2 > s1 ? 0 : 1, (s2 / Math.max(s1, 1)).toFixed(4), '', kw, '', ''];
    } else { tally(c1, s1, s2); tally(c2, s2, s1); }
  });
});

// STANDINGSCSV, as the workbook writes it: each category's pairs, then its F and B seats (a 3-pair category's
// B_1 is RR #3 against a BYE, with no match).
const standings = [];
for (const cat of Object.keys(CATS)) {
  for (const code of Object.keys(names).filter(k => catOf(k) === cat)) {
    const r = rec[code];
    standings.push([code, names[code][0], names[code][1], r.w, r.l, (r.pf / r.pa).toFixed(4), 1, '', '', '']);
  }
  for (const s of ['F_1', 'F_2', 'B_1', 'B_2']) {
    const code = `${cat}_${s}`;
    if (playoffRows[code]) { standings.push(playoffRows[code]); continue; }
    const p = seat(code);
    standings.push([code, p[0], p[1], 0, 0, '0.0000', '', s[0], '', '']);
  }
}

const csv = table => table.map(r => r.join(',')).join('\r\n');
const matchesCsv = csv([['matchNumber', 'court', 'Schedule', 'CourtAssignment', 'teamCode1', 'team1Player1', 'team1Player2', 'team1Score', 'teamCode2', 'team2Player1', 'team2Player2', 'team2Score'], ...rows]);
const standingsCsv = csv([['teamCode', 'player1', 'player2', 'wins', 'loss', 'quotient', 'bracket', 'match', 'from', 'seat'], ...standings]);

const at = '2026-11-22T08:10:00.000Z';
const snapshot = {
  day: DAY, label: 'Nov 22', isLive: true, generatedAt: at, publishedAt: at,
  facilities: [{ name: 'The Foundry', matchesCsv, standingsCsv, syncedAt: at, lastEditAt: at }],
  failedFacilities: [], staleFacilities: [],
};
fs.mkdirSync(path.join('_fixtures', EVENT), { recursive: true });
fs.writeFileSync(path.join('_fixtures', EVENT, 'finished.json'), JSON.stringify(snapshot, null, 2) + '\n');

const configPath = path.join('_fixtures', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.events[EVENT] = {
  type: 'standard', title: 'UPAP 3rd Annual Tournament', attendance: 'desks', scoreEntry: 'console',
  days: { [DAY]: { label: 'Nov 22', date: '2026-11-22', isLive: 'auto', facilities: [{ name: 'The Foundry', sheetId: 'FIXTUREUPAPSHEETID00000000000000000' }] } },
  display: {
    divisions: { B45L: 'Below 45 (DUPR ≤3.25)', B45H: 'Below 45 (DUPR 3.26+)', O: 'Open', A45L: '45+ (DUPR ≤3.25)', A45H: '45+ (DUPR 3.26+)' },
    events: { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' },
  },
};
fs.writeFileSync(configPath, JSON.stringify(config, null, 2) + '\n');

const podium = cat => {
  const gold = winnerOf[`${cat}_F`][0].trim(), bronze = (winnerOf[`${cat}_B`] || seat(`${cat}_B_1`))[0].trim();
  return `${cat} ${gold}/${bronze}`;
};
console.log(`${rows.length} matches, ${Object.keys(names).length} pairs; gold/bronze ${Object.keys(CATS).map(podium).join(', ')}`);
```
