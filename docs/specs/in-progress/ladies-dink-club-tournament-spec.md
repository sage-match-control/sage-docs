# Spec — Ladies' Dink 1st Club Tournament (team tournament)

> **Status: in progress.** The three engine changes (§4–§6; `sage-tools-api` 3.5.0 and the
> site's `main`), the event's pages (§7) and its `events.json` entry (§8) are built and
> pushed. Left: the workbook's fixes and live sync setup (§14), the club logo and QR image
> (§2), the dry run, and the two open questions (§15 Q10, Q11).
>
> Written 2026-10-31 against `sage-match-control.github.io`
> at `d7ed8ed`, `sage-tools-api` 3.4.0 (`b61381a`), `event-data`'s current
> `config/events.json`, the event workbook as exported that day (§2), and the
> organiser's notes (§1.1). The owner answered the questions on 31 October 2026
> (§15). **Two questions are still open:** what settles three or more teams level on
> wins *and* on quotient (§15 Q10), and whether first serve is 3:00 or 3:30 PM (§15
> Q11). Neither blocks the build.
>
> **The tiebreaker code is `_TB`**, as the workbook writes it (`QF-1_TB`), changed from
> `_D` on 31 October at the owner's request; the dry run below was repeated with it.
>
> **Dry-run on 2026-10-31.** Every edit in §4–§7 was applied, exactly as written
> here, to clean copies of the site at `d7ed8ed` and the API at `b61381a`. Every
> anchor matched once. The site's `npm test` went from 466 to 488 tests, all
> passing, and the API's `facilityCompletion.test.mjs` from 33 to 38. Appendix A's
> fixture was built and the Hub, the board and Control Center were checked in a
> browser at 375px and desktop width. §9.3 lists what they showed. The comparison
> harness (`npm run compare`) and the full API suite were not run.
>
> **Deadline.** The event is on **Saturday 7 November 2026**. The site repo takes no
> merge within 3 days of an event, so the site freeze runs from Wednesday 4 to
> Friday 6 November. **Both site pushes (§11) reach `main` by Tuesday 3 November.**

Build the event site for the **Ladies' Dink 1st Club Tournament**, a one-day
women's team tournament on **Saturday 7 November 2026**, from 3:00 PM, at the
**Pampanga Pickleball Center**. The organiser is the **Ladies' Dink** club.
S.A.G.E. runs match control.

The site is an instance of `_templates/team-tournament-template/`. It is the third
`"team"` event, after PickleDrive and the Millennium Invitational. It uses
Millennium's `teamScoring: "games"` (teams are ranked by games won) and needs three
compatible engine changes first:

- **A. The playoff tiebreaker game** (§4). A playoff matchup is two games, so it
  can end 1–1. Then one nominated pair from each team plays one more game to 7.
  The engine has no "game played only if needed" for team events. Without the
  change, a matchup won 2–0 never reads Final, and the facility is never stamped
  complete: on Appendix A's finished fixture, today's engine reports 4 matches
  left. This change touches the site and the API, because the "played" rule
  exists in both.
- **B. Lettered brackets** (§5). The workbook's brackets are **A–D** and the team
  codes `A1`…`D4`. The engine reads a bracket only as a number, so "A" reads as no
  bracket.
- **C. The Tied pill's wording** (§6). It names Millennium's playoff, "one game
  to 5". It becomes wording that fits any event.

---

## 0. How to use this spec

You can carry this out with no other context: every value, every edit, every test
and every check is here. Read §0 to §3 before you change anything.

### 0.1 Rules

- **Find edits by their anchor text, not line numbers.** Every edit is a **Find**
  block and a **Replace with** block. Each Find block occurs **exactly once** in its
  file. If one doesn't, **stop and report**. Don't guess.
- **Copy code blocks exactly**, including comments and blank lines.
- **Use Git Bash.** Python is not installed. Node is (22.12 or later).
- **The engine (`lib/v1/`) is edited only compatibly** (`_templates/CLAUDE.md` §8).
  This spec's edits follow that rule. Two of them add **new modules**
  (`domain/deciders.js`, `domain/brackets.js`) that existing modules import in the
  same push. That is deliberate. The rule's danger is a browser holding a cached
  old copy of a module whose new export another file needs, but a file that didn't
  exist before can't be cached stale. **Do not** move those functions into an
  existing module.
- **Test first.** For each part, add its tests, run them and see them fail, then
  make the change and see them pass. **Never weaken or delete a test.** §6 edits one
  existing test, deliberately; that is the only one.
- **Never edit a finished event's pages:** `events/millennium-invitational-2026/`,
  `events/pickledrive-anniversary-2026/`, `events/piggleball-2026/`,
  `events/pickle-for-sight-2026/`, `events/pnf-x-bup-dual-meet/`,
  `events/archives/`.
- **Never push and never merge into `main` without the owner's yes in chat.** §11
  lists the points where you stop and ask.
- **Commit** after each step marked *commit*, with a plain message saying what
  changed. End each message with the attribution line your session's instructions
  give.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.2 The site doesn't wait for the workbook

The event workbook (§2) is nearly ready: its tiebreaker rows and `STANDINGSCSV` match
this spec, and what is left (§14.1) is mostly the organiser's re-draw. **None of it
blocks this build:**

- §4–§10 are built and checked against Appendix A's fixture, never the workbook.
- Registering the event in `events.json` (§8) publishes nothing. Nothing reaches
  the site until someone runs **SAGE → Set up live sync** in the workbook (§14.2),
  and the people fixing the workbook do that last.
- The workbook already uses this spec's tiebreaker code (`<side>_TB`) and brackets
  (`A`–`D` in `STANDINGSCSV`). The facility name `Pampanga Pickleball Center` and day key
  `ladies-dink-day1` are entered at sync setup (§14.2).

So build in this order, whatever state the workbook is in. Never edit the workbook
yourself.

### 0.3 Setup

The repos are under `D:\Personal\SAGE`. Paths in §4.1–§7 are relative to the site
repo, `sage-match-control.github.io`. In §4.8–§4.9 they are relative to
`sage-tools-api`.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git status --short
```

It must print only:

```
?? events/ladies-dink-2026/
```

That is the staged pubmat (§2), plus `assets/logo.png` and `assets/qr.png` if the owner
has saved them. Anything else in the output: stop and report.

`main` is one commit ahead of `origin/main`: `416807d`, Millennium's after-the-event
switch-off (`LIVE_BASE_URL = ''` on its four pages, `_templates/CLAUDE.md` §7),
committed on 31 October at the owner's request and not pushed. It isn't part of this
work, but this branch starts from it, so it reaches `origin/main` with Push 1 (§11).
Mention it when you ask for that push. Don't pull before branching: `git pull` would
only merge, and `main` must stay as it is.

```bash
git checkout main
git fetch
git status -sb
git checkout -b ladies-dink
cd _tests
npm install
npm test
cd ..
```

`git status -sb` must read `## main...origin/main [ahead 1]`; if `origin/main` has moved on,
stop and ask. `npm test` must pass (466 tests on 31 October) before you change anything.

```bash
cd /d/Personal/SAGE/sage-tools-api
git status --short
git checkout main
git pull
git checkout -b ladies-dink-tiebreaker
npm test
```

`git status` must print nothing, and `npm test` must pass.

### 0.4 Order of work

1. §4.1–§4.7: the site side of change A, test first. *Commit in the site repo.*
2. §4.8–§4.9: the API side of change A, test first, with the version bump. *Commit
   in `sage-tools-api`.*
3. §5: change B, test first. *Commit.*
4. §6: change C. *Commit.*
5. **Stop: Push 1** (§11). Continue on the branch whatever the answer.
6. §7: the event's pages. *Commit once.*
7. §9: the browser check, on Appendix A's fixture, which is never committed.
8. §10: the full test run.
9. **Stop: Push 2** (§11).
10. §8: `events.json`, in `event-data`. *Commit*; push only on the owner's yes.
11. §12: the documentation. *Commit in each repo.*
12. §13: report back.

---

## 1. Event facts

| Fact | Value |
|---|---|
| Organiser | Ladies' Dink (the club). S.A.G.E. Match Control Experts runs match control |
| Event name | Ladies' Dink 1st Club Tournament, a **Team Event**. Pubmat slogan: "Ladies got game." |
| Date | Saturday, 7 November 2026, one day. First serve **3:00 PM** |
| Venue | **Pampanga Pickleball Center** (PPC). One facility, **6 courts** |
| Players | **64 women**. The organiser's "64 pairs" means 64 players: 16 teams × 4 |
| Teams | **16 teams of 4**, each named, with a captain. Pairs are drawn blind and fixed for the day: players 1–2 are pair 1, players 3–4 pair 2. Codes `A1`–`A4`, `B1`–`B4`, `C1`–`C4`, `D1`–`D4`: bracket letter, then position |
| Brackets | **4 brackets of 4** (A–D). Each bracket plays a round robin of 6 matchups |
| Matchup | **2 games**: pair 1 against pair 1, pair 2 against pair 2. Each team plays 3 matchups in the group stage, so 6 games ("a maximum of 6 wins") |
| Playoffs | The **top 2 of each bracket** (8 teams): **quarterfinals**, **semifinals**, **Bronze** and **Final** |
| Playoff tiebreaker | A playoff matchup at **1–1** plays **one more game**: each team nominates one of its pairs, side-out scoring, **to 7**. The organiser's notes call it a "dreambreaker". The site calls it **Tiebreaker** (**TB**) |
| Matches | **72**: 48 group games, 8 QF, 4 SF, 2 Bronze and 2 Final games, and **8 tiebreaker games** (one per playoff matchup, played only when needed) |
| Times | 25-minute slots. Group stage 3:00–5:55 PM on courts 1–6. QF game 1 at 6:45, game 2 at 7:10, tiebreaker at 7:35, each QF on its own court (1–4). SF games at 8:00 (courts 1–2 and 3–4), tiebreakers at 8:25 (courts 1 and 3). Bronze and Final at 8:50 (courts 1–2 and 3–4), tiebreakers at 9:15 (courts 1 and 3) |
| Site title | **Ladies' Dink** (browser tab, Control Center, footer, schedule board). The hero reads "1st Club Tournament" |
| Check-in and scoring | Check-in desk links (`"attendance": "desks"`) and scorer links for the courts (`"scoreEntry": "links"`) |
| Short link | `tinyurl.com/SAGExLD2026` → `https://sage-match-control.github.io/events/ladies-dink-2026/`. The owner makes it and its QR image |

### 1.1 Ranking and ties

The organiser's notes (31 October 2026, also in the workbook's `MISC Org Details`
tab, partly in Filipino), with the owner's answers:

1. **Games won** in the bracket's played group games.
2. **Two teams level:**
   1. head-to-head wins, the games each won in their matchup (with 2-game
      matchups, a 1–1 split will be common);
   2. still level: the **head-to-head quotient**, points for ÷ points against in
      those two games;
   3. still level: a **tiebreaker game to 7** between them.
3. **Three or more level:**
   1. the **quotient**, points for ÷ points against over all their group games;
   2. two still level go to step 2;
   3. three or more with the same quotient: **undecided** (§15 Q10).

Steps 1, 2.1, 2.2, 3.1 and 3.2 are already the engine's `"games"` ranking
(`teamRankBracketByGames` in `lib/v1/domain/teams.js`), so the site computes them
with no change. Steps 2.3 and 3.3 happen off the site. A tie that reaches them keeps
the workbook's order. Once the bracket's round robin is over, its teams share the
higher rank and show the **Tied** pill (§6). The organiser settles it by typing the
qualifiers into the quarterfinal slots, as for every playoff. The group-stage
tiebreaker game has no row in `SCHEDULE`, so nothing records its result.

A group matchup at 1–1 has no winner in the organiser's rules; only games count. The
engine's `"games"` scoring then names the side with more total points as the
matchup's winner on its card. That is cosmetic: the ranking counts games only.
Leave it (§16 D3).

### 1.2 Playoff seeding

**Crossover**, so teams from the same bracket meet again only in the Final:

| QF matchup | Sides | Seats |
|---|---|---|
| 1 | `QF-1` v `QF-2` | Bracket A #1 v Bracket B #2 |
| 2 | `QF-3` v `QF-4` | Bracket C #1 v Bracket D #2 |
| 3 | `QF-5` v `QF-6` | Bracket B #1 v Bracket A #2 |
| 4 | `QF-7` v `QF-8` | Bracket D #1 v Bracket C #2 |

`SF-1` v `SF-2` are the winners of QF 1 and QF 2, and `SF-3` v `SF-4` the winners of
QF 3 and QF 4. Bronze is the two SF losers, and the Final the two SF winners. The
organiser types each qualifier's code over its slot (`QF-1` becomes `QF-A1`), as at
every team event. The `matchUp` value changes with it (`QF-A1 v QF-B2`), for all
three rows of the matchup.

**The site shows none of these:** the registration fee, the cash prizes, dinner,
the pubmat's net and ball artwork. The site is a live results hub for players who
have already entered.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| Pubmat | **Staged** at `events/ladies-dink-2026/assets/pubmat.webp` (1414×2000). Reference only: no page loads it. §7's colours were sampled from it | §7 |
| Club logo | **To be saved by the owner** as `events/ladies-dink-2026/assets/logo.png`. It was sent in chat on 31 October: 480×480, the blue "LADIES' DINK" wordmark with a pink star and a lime ball, on an off-white background. If the file isn't there, do everything else and report it missing. **Don't** crop one from the pubmat. Its background isn't transparent, so §7 gives its hero tile the same off-white | §7 `{{EVENT_LOGO}}` |
| QR image | **To follow from the owner** as `events/ladies-dink-2026/assets/qr.png`, encoding `tinyurl.com/SAGExLD2026`. If it's missing, report it. Don't generate one | §7 `{{QR_IMAGE}}` |
| Event workbook | `1CITP2HOs8Ix5m0qZbkM1GDbDaotWv17i046VeLip7RI`, "2026-11-07 Ladies Dink Tournament", a copy of the Millennium workbook being adapted by hand. Anyone with the link can read it. Not needed for the build (§0.2) | §8, §14 |

---

## 3. How this event differs from Millennium

| | Millennium | This event | What the engine does with it |
|---|---|---|---|
| Teams | 8, codes are colours | 16, codes `A1`…`D4` | `parseSideCode('A1_2')` gives side `A1`, pair 2; a side with no `-` is a group side. **Fits** |
| Brackets | 1, `bracket` `1` | 4, lettered `A`–`D` | `groupOf` does `parseInt(row.bracket)`, so `A` reads as no bracket. **Change B** |
| Matchup | 4 games | 2 games | Matchups are the rows sharing a `matchUp`. **Fits** |
| Ranking | `teamScoring: "games"` | the same (§1.1) | **Fits** |
| Playoffs | Bronze, Final | QF, SF, Bronze, Final | Standings draws each stage that has matchups. **Fits** |
| **Playoff at 1–1** | can't happen | a tiebreaker game | No notion of a game played only if needed. **Change A** |
| Unsettled tie | "Tied · playoff game / round robin", "to 5" | a game to 7; three-way undecided | **Change C** |
| Pairs | MD, XD, WD, GD | two pairs | `display.pairs` `{1: Pair 1/P1, 2: Pair 2/P2}` (§8). No `lineupRules`, no `lineupReveal` |
| Courts | 8 | 6 | The board counts courts from the data. **Fits** |
| Team colours | 8, with shapes | none | 16 teams is more than the marks keep apart (§16 D5) |

The public Hub never shows team codes (`hooks.teamLetters` is `false` in
`lib/v1/apps/hub.js`), so players see "Bracket A" and team names. Control Center
shows the codes too.

---

## 4. Change A: the playoff tiebreaker game

### 4.1 The rule

A **tiebreaker game** is a match whose `teamCode1` **and** `teamCode2` both end in
`_TB` (after trimming), e.g. `QF-1_TB` v `QF-2_TB`. It belongs to the matchup whose
rows share its `matchUp` value.

It is **not needed** once **every other match of its matchup is played** (both
scores in) **and one side has won more of them**. "Side" is the team code before
the first `_`. A game with equal scores counts for neither side. Until then it
counts as a match like any other, so it is part of "matches left". A tiebreaker row
with no `matchUp` value is always counted. Standard and dual-meet events never have
`_TB` codes, so nothing changes for them.

A tiebreaker that isn't needed is never played and never counted. The site returns
it from `unneededSeriesGames(matches)` (`lib/v1/domain/series.js`), the set that
already holds a series final's unneeded games. Every consumer of that set then
handles it with no edit of its own: facility progress, the board's greyed
*Not needed* card, Match Finder's "next match", the ticket view, and the scorer
page's and Control Center's refusal to score a game that isn't needed. The team
views use it too: a matchup is final without it (§4.5), and its card doesn't list
it (§4.6).

### 4.2 A new module, `lib/v1/domain/deciders.js`

**Create** `lib/v1/domain/deciders.js`:

```js
// A team playoff matchup's tiebreaker game (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §4).
//
// Both its team codes end in "_TB" (QF-1_TB v QF-2_TB). It is played only when the matchup's other games leave
// the two sides level on games won: once they are all played and one side has won more, it is never played,
// so it is not a match left. A game with equal scores counts for neither side. A tiebreaker row with no
// matchUp value is always counted.
//
// Mirrors sage-tools-api's facilityCompletion.mjs (isDeciderCode, unneededDeciders):
// _tests/unit/parity-server.test.mjs checks the two agree, because "matches left" here and the published
// completedAt would otherwise disagree about when the day ended.

export const DECIDER_RE = /_TB$/;

/** @param {string} code */
export function isDeciderCode(code){ return DECIDER_RE.test((code || '').trim()); }

/** @param {{ t1: string, t2: string }} m */
export function isDecider(m){ return isDeciderCode(m.t1) && isDeciderCode(m.t2); }

// The side of a team code: everything before the first "_" (QF-A1_2 -> QF-A1).
const sideOfCode = code => { const c = (code || '').trim(); const i = c.indexOf('_'); return i < 0 ? c : c.slice(0, i); };

/**
 * The tiebreaker games whose matchup the other games already settled.
 * @param {Array<{ matchUp?: string, t1: string, t2: string, t1Score: number|null, t2Score: number|null, played: boolean }>} matches
 * @returns {Set<object>}  the unneeded tiebreaker games, as the same objects
 */
export function unneededDeciders(matches){
  const byMatchUp = new Map();
  matches.forEach(m => {
    if(!m.matchUp) return;
    if(!byMatchUp.has(m.matchUp)) byMatchUp.set(m.matchUp, []);
    byMatchUp.get(m.matchUp).push(m);
  });
  const out = new Set();
  byMatchUp.forEach(ms => {
    const deciders = ms.filter(isDecider);
    const games = ms.filter(m => !isDecider(m));
    if(!deciders.length || !games.length || !games.every(m => m.played)) return;
    const wins = new Map();
    games.forEach(m => {
      if(m.t1Score === m.t2Score) return;
      const side = sideOfCode(m.t1Score > m.t2Score ? m.t1 : m.t2);
      wins.set(side, (wins.get(side) || 0) + 1);
    });
    const [a = 0, b = 0] = Array.from(wins.values());
    if(a !== b) deciders.forEach(d => out.add(d));
  });
  return out;
}
```

### 4.3 `lib/v1/domain/series.js`

**Find** in `lib/v1/domain/series.js`:

```js
export const SERIES_GAME_RE
```

**Replace with:**

```js
import { unneededDeciders } from './deciders.js';

export const SERIES_GAME_RE
```

**Find** in `lib/v1/domain/series.js`:

```js
/**
 * The series games after the decider: never played, so never counted.
```

**Replace with:**

```js
/**
 * The games never played, so never counted: a series final's games after the decider (walkSeries), and a team
 * playoff matchup's tiebreaker game once the matchup's other games settled it (deciders.js).
```

**Find** in `lib/v1/domain/series.js`:

```js
export function unneededSeriesGames(matches){
  const out = new Set();
  seriesGroups(matches).forEach(list => walkSeries(list).unneeded.forEach(m => out.add(m)));
  return out;
}
```

**Replace with:**

```js
export function unneededSeriesGames(matches){
  const out = new Set();
  seriesGroups(matches).forEach(list => walkSeries(list).unneeded.forEach(m => out.add(m)));
  unneededDeciders(matches).forEach(m => out.add(m));
  return out;
}
```

### 4.4 `lib/v1/domain/schedule-grid.js`

The board's rows carry `c1`/`c2`/`s1`/`s2`, and `markUnneededGames` copies them into
the shape the series rule reads. Give it the `matchUp` too. Board rows of a team
event already have one.

**Find** in `lib/v1/domain/schedule-grid.js`:

```js
  const views = rows.map(m => ({ src: m, t1: m.c1, t2: m.c2, t1Score: m.s1, t2Score: m.s2, played: m.s1 !== null && m.s2 !== null }));
```

**Replace with:**

```js
  const views = rows.map(m => ({ src: m, matchUp: m.matchUp, t1: m.c1, t2: m.c2, t1Score: m.s1, t2Score: m.s2, played: m.s1 !== null && m.s2 !== null }));
```

### 4.5 `lib/v1/domain/teams.js`

**Find** in `lib/v1/domain/teams.js`:

```js
import { parseCSV } from './csv.js';
```

**Replace with:**

```js
import { parseCSV } from './csv.js';
import { isDeciderCode, unneededDeciders } from './deciders.js';
```

In `teamMatchupResult`, leave out a tiebreaker that isn't needed. **Find** in
`lib/v1/domain/teams.js`:

```js
  let pts1 = 0, pts2 = 0, pairs1 = 0, pairs2 = 0, played = 0;
  ms.forEach(m => {
    if(!m.played) return;
```

**Replace with:**

```js
  let pts1 = 0, pts2 = 0, pairs1 = 0, pairs2 = 0, played = 0;
  // A tiebreaker game the other games made unnecessary (deciders.js) is never played, so never counted.
  const skip = unneededDeciders(ms);
  ms.forEach(m => {
    if(!m.played || skip.has(m)) return;
```

**Find** in `lib/v1/domain/teams.js`:

```js
  const total = ms.length;
```

**Replace with:**

```js
  const total = ms.length - skip.size;
```

A matchup won 2–0 is then `final` with `total` 2. At 1–1 its tiebreaker counts, so
the matchup stays `in-progress` until the tiebreaker is scored, and `"games"`
scoring then decides it 2–1. The returned fields are the same as before.

Give a tiebreaker its label. **Find** in `lib/v1/domain/teams.js`:

```js
 * A match code's pair label. `pairs` is the event's (EventConfig.pairs). A pair
 * number the event doesn't list reads "Pair <n>" rather than blank, so a
 * workbook mistake shows; a code with no pair number has no label.
```

**Replace with:**

```js
 * A match code's pair label. `pairs` is the event's (EventConfig.pairs). A pair
 * number the event doesn't list reads "Pair <n>" rather than blank, so a
 * workbook mistake shows; a code with no pair number has no label. A playoff
 * matchup's tiebreaker game (a code ending "_TB", deciders.js) reads "Tiebreaker" / "TB".
```

**Find** in `lib/v1/domain/teams.js`:

```js
export function pairLabel(code, short, pairs = PAIRS){
  const n = pairOf(code);
```

**Replace with:**

```js
export function pairLabel(code, short, pairs = PAIRS){
  if(isDeciderCode(code)) return short ? 'TB' : 'Tiebreaker';
  const n = pairOf(code);
```

`teamGameRecords` and the bracket ranking skip playoff sides already, and a
tiebreaker is only ever in a playoff, so they need no change. `lineup-rules.js` reads
a pair number, which a tiebreaker doesn't have, so it is skipped there too.

### 4.6 `lib/v1/views/teams.js`: the matchup card

A matchup card lists every match of its matchup. Leave out a tiebreaker that isn't
needed. One that may still be needed is listed, with the label TB and "Lineup not
set" until the operator types the nominated players in, which tells players a third
game may come.

**Find** in `lib/v1/views/teams.js`:

```js
  const rowMatches = onlyMatch === null ? mu.matches : mu.matches.filter(m => m.num === onlyMatch);
```

**Replace with:**

```js
  // A tiebreaker game the matchup's other games made unnecessary is not listed (model.unneededGames).
  const listed = model.unneededGames ? mu.matches.filter(m => !model.unneededGames.has(m)) : mu.matches;
  const rowMatches = onlyMatch === null ? listed : listed.filter(m => m.num === onlyMatch);
```

`model.unneededGames` is `unneededSeriesGames(matches)`, set by `buildDayModel`
(`lib/v1/domain/model.js`), so after §4.3 it holds the tiebreakers.

### 4.7 Site tests

Add each block **before** making its change, run `npm test` from `_tests/`, and
see the new tests fail (a missing module counts). Then make §4.2–§4.6 and see every
test pass.

**Append to the end of** `_tests/unit/domain/csv-byes-series.test.mjs`:

```js
// ---- a team playoff matchup's tiebreaker game (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §4) ----
import { DECIDER_RE, isDeciderCode, isDecider, unneededDeciders } from '../../../lib/v1/domain/deciders.js';
import * as DkTeams from '../../../lib/v1/domain/teams.js';
import { computeFacilityProgress as dkProgress } from '../../../lib/v1/domain/progress.js';
import { markUnneededGames as dkMarkUnneeded } from '../../../lib/v1/domain/schedule-grid.js';

// One match of matchup `mu`: [code1, score1, code2, score2], a null score unplayed.
const dkMatch = (mu, t1, s1, t2, s2) => ({ matchUp: mu, t1, t2, t1Score: s1, t2Score: s2, played: s1 !== null && s2 !== null });
// QF 1: game 1, game 2 and the tiebreaker, with the scores given ([s1, s2] or null each).
const dkQf = (g1, g2, tb = null, mu = 'QF-1 v QF-2') => [
  dkMatch(mu, 'QF-1_1', g1 ? g1[0] : null, 'QF-2_1', g1 ? g1[1] : null),
  dkMatch(mu, 'QF-1_2', g2 ? g2[0] : null, 'QF-2_2', g2 ? g2[1] : null),
  dkMatch(mu, 'QF-1_TB', tb ? tb[0] : null, 'QF-2_TB', tb ? tb[1] : null),
];
const dkUnneeded = ms => Array.from(unneededDeciders(ms)).map(m => m.t1);

test('isDeciderCode: a team code ending in _TB', () => {
  assert.equal(DECIDER_RE.source, '_TB$');
  for (const c of ['QF-1_TB', ' SF-2_TB ', 'Fi-A1_TB']) assert.equal(isDeciderCode(c), true, c);
  for (const c of ['QF-1_1', 'A1_2', 'IXD_F_1_(2)', 'QF-1_tb', '', undefined, null]) assert.equal(isDeciderCode(c), false, String(c));
  assert.equal(isDecider({ t1: 'QF-1_TB', t2: 'QF-2_TB' }), true);
  assert.equal(isDecider({ t1: 'QF-1_TB', t2: 'QF-2_2' }), false);
});

test('unneededDeciders: the tiebreaker is not needed once the other games settle the matchup', () => {
  assert.deepEqual(dkUnneeded(dkQf(null, null)), [], 'nothing played');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], null)), [], 'one game played');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], [11, 9])), ['QF-1_TB'], '2-0');
  assert.deepEqual(dkUnneeded(dkQf([5, 11], [9, 11])), ['QF-1_TB'], '0-2');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], [9, 11])), [], '1-1: needed');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], [9, 9])), ['QF-1_TB'], '1-0 and a level game: one side won more');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], [9, 11], [7, 4])), [], '1-1 and played: it was needed');
  assert.deepEqual(dkUnneeded(dkQf([11, 5], [11, 9], null, '')), [], 'no matchUp: always counted');
  // Two matchups in one list are judged apart.
  const two = [...dkQf([11, 5], [11, 9]), ...dkQf([11, 5], [9, 11], null, 'QF-3 v QF-4').map(m => ({ ...m, t1: m.t1.replace('QF-1', 'QF-3'), t2: m.t2.replace('QF-2', 'QF-4') }))];
  assert.deepEqual(dkUnneeded(two), ['QF-1_TB']);
  // A team typed over the seeds: the sides change, the rule doesn't.
  const typed = dkQf([11, 5], [11, 9], null, 'QF-A1 v QF-B2').map(m => ({ ...m, t1: m.t1.replace('QF-1', 'QF-A1'), t2: m.t2.replace('QF-2', 'QF-B2') }));
  assert.deepEqual(dkUnneeded(typed), ['QF-A1_TB']);
});

test('unneededSeriesGames also returns a tiebreaker that is not needed', () => {
  const ms = dkQf([11, 5], [11, 9]);
  assert.deepEqual(Array.from(unneededSeriesGames(ms)).map(m => m.t1), ['QF-1_TB']);
  assert.deepEqual(Array.from(unneededSeriesGames(dkQf([11, 5], [9, 11]))), []);
});

test('teamMatchupResult: a settled matchup is final without its tiebreaker; 1-1 waits for it', () => {
  const twoNil = DkTeams.teamMatchupResult(dkQf([11, 5], [11, 9]), 'games');
  assert.deepEqual([twoNil.state, twoNil.winner, twoNil.total, twoNil.played, twoNil.pairs1, twoNil.pairs2], ['final', 1, 2, 2, 2, 0]);
  const level = DkTeams.teamMatchupResult(dkQf([11, 5], [9, 11]), 'games');
  assert.deepEqual([level.state, level.total, level.played], ['in-progress', 3, 2]);
  const decided = DkTeams.teamMatchupResult(dkQf([11, 5], [9, 11], [4, 7]), 'games');
  assert.deepEqual([decided.state, decided.winner, decided.pairs1, decided.pairs2, decided.total], ['final', 2, 1, 2, 3]);
  // No tiebreaker rows: unchanged.
  const plain = DkTeams.teamMatchupResult(dkQf([11, 5], [11, 9]).slice(0, 2), 'games');
  assert.deepEqual([plain.state, plain.total], ['final', 2]);
});

test('pairLabel: a tiebreaker reads Tiebreaker / TB', () => {
  assert.equal(DkTeams.pairLabel('QF-1_TB', true), 'TB');
  assert.equal(DkTeams.pairLabel('QF-1_TB', false), 'Tiebreaker');
  const pairs = { 1: { full: 'Pair 1', short: 'P1' }, 2: { full: 'Pair 2', short: 'P2' } };
  assert.equal(DkTeams.pairLabel('QF-1_2', true, pairs), 'P2');
  assert.equal(DkTeams.pairLabel('A1_3', true, pairs), 'Pair 3');
});

test('facility progress leaves out a tiebreaker that is not needed', () => {
  const ms = dkQf([11, 5], [11, 9]).map((m, i) => ({ ...m, num: i + 1, time: '6:45 PM', court: 'Court 1', liveCourt: '', t1p1: 'a', t1p2: 'b', t2p1: 'c', t2p2: 'd' }));
  const p = dkProgress(ms, false, null);
  assert.deepEqual([p.total, p.done, p.left], [2, 2, 0]);
  const open = dkQf([11, 5], [9, 11]).map((m, i) => ({ ...m, num: i + 1, time: '6:45 PM', court: 'Court 1', liveCourt: '', t1p1: 'a', t1p2: 'b', t2p1: 'c', t2p2: 'd' }));
  assert.deepEqual([dkProgress(open, false, null).total, dkProgress(open, false, null).left], [3, 1]);
});

test('the schedule board marks a tiebreaker that is not needed', () => {
  const rows = dkQf([11, 5], [11, 9]).map(m => ({ matchUp: m.matchUp, c1: m.t1, c2: m.t2, s1: m.t1Score, s2: m.t2Score }));
  dkMarkUnneeded(rows);
  assert.deepEqual(rows.map(r => r.notNeeded), [false, false, true]);
});
```

These `import` lines sit at the end of the file. That is valid for an ES module
(imports are hoisted). If you prefer, move them up to the file's other imports;
nothing else changes. The file already imports `test`, `assert` and
`unneededSeriesGames`.

**Append to the end of** `_tests/unit/views/teams.test.mjs`:

```js
// ---- a team playoff matchup's tiebreaker game (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §4) ----
const DK_HEADER = 'matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score';
// A day with one quarterfinal between Alpha (A1) and Beta (B2): game 1, game 2, then the tiebreaker row.
const dkQfDay = (g2, tb = ['', '']) => buildDayModel(
  { key: 'e', type: 'team', display: {}, days: [], title: 'E', teamScoring: 'games' },
  { key: 'd', date: '2026-11-07' },
  { isLive: true, facilities: [{ name: 'Main',
    matchesCsv: [DK_HEADER,
      '1,QF-A1 v QF-B2,,6:45 PM,Court 1,QF-A1_1,a1,a2,11,QF-B2_1,b1,b2,5',
      `2,QF-A1 v QF-B2,,7:10 PM,Court 1,QF-A1_2,a3,a4,${g2[0]},QF-B2_2,b3,b4,${g2[1]}`,
      `3,QF-A1 v QF-B2,,7:35 PM,Court 1,QF-A1_TB,,,${tb[0]},QF-B2_TB,,,${tb[1]}`].join('\n'),
    standingsCsv: 'teamCode,teamName,bracket\nA1,Alpha,A\nB2,Beta,B\nQF-A1,Alpha,\nQF-B2,Beta,' }] },
  { now: 0, alwaysLive: true },
);

test('a matchup card leaves out a tiebreaker that is not needed, and lists one that may be', () => {
  const settled = dkQfDay([11, 9]);
  const mu = settled.team.matchups[0];
  const html = matchupCardHTML(settled, mu, {});
  assert.match(html, /tm-pill final">Final/);
  assert.doesNotMatch(html, /#3</);
  assert.doesNotMatch(html, />TB</);
  const level = dkQfDay([9, 11]);
  const levelHtml = matchupCardHTML(level, level.team.matchups[0], {});
  assert.match(levelHtml, /tm-pill progress">In progress/);
  assert.match(levelHtml, /<span>#3<\/span><span class="tm-plabel">TB<\/span>/);
  assert.match(levelHtml, /Lineup not set/);
  const decided = dkQfDay([9, 11], [7, 4]);
  assert.match(matchupCardHTML(decided, decided.team.matchups[0], {}), /tm-pill final">Final/);
  assert.equal(decided.team.matchups[0].result.winner, 1);
});
```

The file already imports `buildDayModel` and `matchupCardHTML`.

The parity test runs the site's rule and the API's on the same CSVs (§4.8 changes
the API's). **Find** in `_tests/unit/parity-server.test.mjs`:

```js
test('the played and BYE rules: the card and completedAt agree on every case', options, () => {
```

**Replace with:**

```js
// A team playoff matchup's tiebreaker game (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §4):
// game 1, game 2 and the tiebreaker of one quarterfinal, with the scores given ('' unplayed).
const TEAM_HEADER = 'matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score';
const qfCsv = ([a1, a2], [b1, b2], [c1, c2], mu = 'QF-1 v QF-2') => [TEAM_HEADER,
  `1,${mu},,6:45 PM,Court 1,QF-1_1,a,b,${a1},QF-2_1,c,d,${a2}`,
  `2,${mu},,7:10 PM,Court 1,QF-1_2,e,f,${b1},QF-2_2,g,h,${b2}`,
  `3,${mu},,7:35 PM,Court 1,QF-1_TB,,,${c1},QF-2_TB,,,${c2}`].join('\n');
const DECIDER_CASES = {
  'nothing played': [qfCsv(['', ''], ['', ''], ['', '']), false],
  'one game played': [qfCsv([11, 5], ['', ''], ['', '']), false],
  '2-0, tiebreaker blank': [qfCsv([11, 5], [11, 9], ['', '']), true],
  '0-2, tiebreaker blank': [qfCsv([5, 11], [9, 11], ['', '']), true],
  '1-1, tiebreaker blank': [qfCsv([11, 5], [9, 11], ['', '']), false],
  '1-1, tiebreaker played': [qfCsv([11, 5], [9, 11], [7, 4]), true],
  '1-0 and a level game, tiebreaker blank': [qfCsv([11, 5], [9, 9], ['', '']), true],
  '2-0, no matchUp: the tiebreaker still counts': [qfCsv([11, 5], [11, 9], ['', ''], ''), false],
  '2-0, decimal scores': [qfCsv(['11.5', 5], [11, '9.0'], ['', '']), true],
};

test('a playoff tiebreaker: the card and completedAt agree on every case', options, () => {
  for (const [label, [csv, complete]] of Object.entries(DECIDER_CASES)) {
    assert.equal(siteSaysComplete(csv), complete, `site: ${label}`);
    assert.equal(server.facilityIsComplete(csv), complete, `server: ${label}`);
  }
});

test('the played and BYE rules: the card and completedAt agree on every case', options, () => {
```

It reads `sage-tools-api` from the sibling folder. It fails until §4.8 is made
there too. That is expected: make §4.8 before you commit.

### 4.8 API: `src/sync/domain/facilityCompletion.mjs`

The API stamps each facility's `completedAt` from the same rule (workspace
`CLAUDE.md`, "Things that must be kept in sync by hand"). Mirror §4.2 there.

**Find** in `src/sync/domain/facilityCompletion.mjs`:

```js
// The rules here mirror the site's lib/v1/domain/ exactly: csv.js (parseCSV),
// matches.js (rowsToMatches), byes.js (sideIsBye), series.js
// (seriesGameOf/unneededSeriesGames) and progress.js (computeFacilityProgress's
// `left`).
```

**Replace with:**

```js
// The rules here mirror the site's lib/v1/domain/ exactly: csv.js (parseCSV),
// matches.js (rowsToMatches), byes.js (sideIsBye), series.js
// (seriesGameOf/unneededSeriesGames), deciders.js (isDeciderCode/
// unneededDeciders) and progress.js (computeFacilityProgress's `left`).
```

**Find** in `src/sync/domain/facilityCompletion.mjs`:

```js
// true when every non-bye match carries both scores, leaving out a series
// final's games that were never needed.
```

**Replace with:**

```js
// A team playoff matchup's tiebreaker game carries "_TB" on both team codes,
// e.g. QF-1_TB v QF-2_TB. It is not needed once the matchup's other games (the
// rows with the same matchUp) are all played and one side has won more of
// them. A game with equal scores counts for neither side; a tiebreaker row
// with no matchUp is always needed. Mirrors the site's lib/v1/domain/deciders.js.
const DECIDER_RE = /_TB$/;

export function isDeciderCode(code) {
    return DECIDER_RE.test((code || "").trim());
}

const sideOfCode = code => {
    const c = (code || "").trim();
    const i = c.indexOf("_");
    return i < 0 ? c : c.slice(0, i);
};

//   games: [{ key, matchUp, code1, score1, code2, score2 }]  ->  Set of keys
export function unneededDeciders(games) {
    const byMatchUp = new Map();
    for (const g of games) {
        if (!g.matchUp) continue;
        if (!byMatchUp.has(g.matchUp)) byMatchUp.set(g.matchUp, []);
        byMatchUp.get(g.matchUp).push(g);
    }
    const unneeded = new Set();
    for (const list of byMatchUp.values()) {
        const isDecider = g => isDeciderCode(g.code1) && isDeciderCode(g.code2);
        const deciders = list.filter(isDecider);
        const others = list.filter(g => !isDecider(g));
        if (!deciders.length || !others.length) continue;
        if (!others.every(g => hasScore(g.score1) && hasScore(g.score2))) continue;
        const wins = new Map();
        for (const g of others) {
            const s1 = parseInt(String(g.score1).trim(), 10), s2 = parseInt(String(g.score2).trim(), 10);
            if (s1 === s2) continue;
            const side = sideOfCode(s1 > s2 ? g.code1 : g.code2);
            wins.set(side, (wins.get(side) || 0) + 1);
        }
        const [a = 0, b = 0] = Array.from(wins.values());
        if (a !== b) for (const d of deciders) unneeded.add(d.key);
    }
    return unneeded;
}

// true when every non-bye match carries both scores, leaving out a series
// final's games that were never needed and a team playoff matchup's
// tiebreaker game that was never needed.
```

**Find** in `src/sync/domain/facilityCompletion.mjs`:

```js
    const iS1 = at("team1Score"), iS2 = at("team2Score");
```

**Replace with:**

```js
    const iS1 = at("team1Score"), iS2 = at("team2Score");
    const iMU = at("matchUp");
```

**Find** in `src/sync/domain/facilityCompletion.mjs`:

```js
        real.push({ key: r, code1: cell(row, iT1), score1: cell(row, iS1), code2: cell(row, iT2), score2: cell(row, iS2) });
```

**Replace with:**

```js
        real.push({ key: r, matchUp: cell(row, iMU).trim(), code1: cell(row, iT1), score1: cell(row, iS1), code2: cell(row, iT2), score2: cell(row, iS2) });
```

**Find** in `src/sync/domain/facilityCompletion.mjs`:

```js
    const unneeded = unneededSeriesGames(real);
```

**Replace with:**

```js
    const unneeded = new Set([...unneededSeriesGames(real), ...unneededDeciders(real)]);
```

### 4.9 API tests, version and changelog

Tests first. **Find** in `test/unit/sync/domain/facilityCompletion.test.mjs`:

```js
import { facilityIsComplete, resolveCompletedAt, seriesGameOf, unneededSeriesGames } from "../../../../src/sync/domain/facilityCompletion.mjs";
```

**Replace with:**

```js
import { facilityIsComplete, resolveCompletedAt, seriesGameOf, unneededSeriesGames, isDeciderCode, unneededDeciders } from "../../../../src/sync/domain/facilityCompletion.mjs";
```

**Append to the end of** `test/unit/sync/domain/facilityCompletion.test.mjs`:

```js
describe("a team playoff matchup's tiebreaker game (_TB)", () => {
    const HEADER = "matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score";
    // Game 1, game 2 and the tiebreaker of one quarterfinal; "" is unplayed.
    const qf = ([a1, a2], [b1, b2], [c1, c2], mu = "QF-1 v QF-2") => [HEADER,
        `1,${mu},,6:45 PM,Court 1,QF-1_1,a,b,${a1},QF-2_1,c,d,${a2}`,
        `2,${mu},,7:10 PM,Court 1,QF-1_2,e,f,${b1},QF-2_2,g,h,${b2}`,
        `3,${mu},,7:35 PM,Court 1,QF-1_TB,,,${c1},QF-2_TB,,,${c2}`].join("\n");

    it("isDeciderCode: a code ending in _TB", () => {
        assert.equal(isDeciderCode("QF-1_TB"), true);
        assert.equal(isDeciderCode(" SF-A1_TB "), true);
        assert.equal(isDeciderCode("QF-1_1"), false);
        assert.equal(isDeciderCode("IXD_F_1_(2)"), false);
        assert.equal(isDeciderCode(""), false);
        assert.equal(isDeciderCode(undefined), false);
    });

    it("unneededDeciders: keys of tiebreakers whose matchup is settled", () => {
        const g = (key, matchUp, code1, score1, code2, score2) => ({ key, matchUp, code1, score1, code2, score2 });
        const settled = [g(1, "M", "QF-1_1", "11", "QF-2_1", "5"), g(2, "M", "QF-1_2", "11", "QF-2_2", "9"), g(3, "M", "QF-1_TB", "", "QF-2_TB", "")];
        assert.deepEqual([...unneededDeciders(settled)], [3]);
        const level = [g(1, "M", "QF-1_1", "11", "QF-2_1", "5"), g(2, "M", "QF-1_2", "9", "QF-2_2", "11"), g(3, "M", "QF-1_TB", "", "QF-2_TB", "")];
        assert.deepEqual([...unneededDeciders(level)], []);
        assert.deepEqual([...unneededDeciders(settled.map(x => ({ ...x, matchUp: "" })))], []);
    });

    it("facilityIsComplete: a settled matchup's blank tiebreaker is not left", () => {
        assert.equal(facilityIsComplete(qf([11, 5], [11, 9], ["", ""])), true);
        assert.equal(facilityIsComplete(qf([5, 11], [9, 11], ["", ""])), true);
    });

    it("facilityIsComplete: at 1-1 the tiebreaker must be scored", () => {
        assert.equal(facilityIsComplete(qf([11, 5], [9, 11], ["", ""])), false);
        assert.equal(facilityIsComplete(qf([11, 5], [9, 11], [7, 4])), true);
    });

    it("facilityIsComplete: a tiebreaker row with no matchUp always counts (fails closed)", () => {
        assert.equal(facilityIsComplete(qf([11, 5], [11, 9], ["", ""], "")), false);
    });
});
```

Then make §4.8 and run `npm test`, then `npm run verify`.

**Version.** In `package.json`, `"version": "3.4.0"` → `"version": "3.5.0"`: a
feature, so a minor bump (workspace `CLAUDE.md`). `apps-script/` is untouched.

**Changelog.** In `README.md`, directly under the `## Changelog` section's
introductory paragraph and above `### 3.4.0`, add:

```markdown
### 3.5.0

- **A team playoff matchup's tiebreaker game is left out once it isn't needed.** A
  match whose two team codes both end in `_TB` (`QF-1_TB` v `QF-2_TB`) is the
  tiebreaker of the matchup sharing its `matchUp`. Once that matchup's other games
  are all played and one side has won more of them, the tiebreaker is never played,
  so `facilityIsComplete` doesn't wait for it, and `completedAt` is stamped when the
  last needed game ends. A tiebreaker row with no `matchUp` is always counted. It
  mirrors the site's `lib/v1/domain/deciders.js`; the site's
  `_tests/unit/parity-server.test.mjs` checks the two agree. No route, setting or
  response changes.
- **Tests:** `test/unit/sync/domain/facilityCompletion.test.mjs`.
```

*Commit in `sage-tools-api`* on `ladies-dink-tiebreaker`, then *commit the site's
§4.2–§4.7* on `ladies-dink`.

---

## 5. Change B: lettered brackets

### 5.1 The rule

A team's `bracket` (`STANDINGSCSV`) may be a number (`1`, as today) or **one
letter** (`A`). A lettered bracket reads **"Bracket A"** wherever a numbered one
reads "Bracket 1". Brackets sort with no bracket first, then numbers by value, then
letters A–Z. Anything else (blank, `#N/A`, `AB`) is no bracket, as today. Every
bracket that exists today keys and sorts exactly as before.

### 5.2 A new module, `lib/v1/domain/brackets.js`

**Create** `lib/v1/domain/brackets.js`:

```js
// A team's bracket (STANDINGSCSV's `bracket`), as the team views key and sort it
// (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §5): a number ("1", as PickleDrive and
// Millennium publish) or a single letter ("A", as the Ladies' Dink workbook does). Anything else is no bracket.

/**
 * @param {string|null|undefined} raw
 * @returns {number|string|null}  1 for "1", "A" for "a", null for "", "#N/A", "AB"
 */
export function bracketKeyOf(raw){
  const s = (raw === null || raw === undefined ? '' : String(raw)).trim();
  if(/^\d+$/.test(s)) return parseInt(s, 10);
  if(/^[A-Za-z]$/.test(s)) return s.toUpperCase();
  return null;
}

/**
 * Sort order for bracket keys: no bracket (null) first, then numbers by value, then letters A-Z.
 * @param {number|string|null} a
 * @param {number|string|null} b
 */
export function compareBrackets(a, b){
  const rank = k => k === null ? 0 : typeof k === 'number' ? 1 : 2;
  return (rank(a) - rank(b)) || (typeof a === 'number' ? a - b : String(a).localeCompare(String(b)));
}
```

### 5.3 `lib/v1/domain/teams.js`

**Find** in `lib/v1/domain/teams.js`:

```js
import { isDeciderCode, unneededDeciders } from './deciders.js';
```

**Replace with:**

```js
import { isDeciderCode, unneededDeciders } from './deciders.js';
import { bracketKeyOf } from './brackets.js';
```

**Find** in `lib/v1/domain/teams.js`:

```js
  const row = base ? rowByCode.get(base) : null;
  const n = row ? parseInt(row.bracket, 10) : NaN;
  return isNaN(n) ? null : n;
}
```

**Replace with:**

```js
  const row = base ? rowByCode.get(base) : null;
  return row ? bracketKeyOf(row.bracket) : null;
}
```

`stageLabel` reads `groupOf`, so a lettered team reads "Bracket A" there too.
`schedule-grid.js` builds the board's category `G${group}` from it, so the board's
categories become `GA`…`GD`, which §7.5 names. That file needs no edit.

### 5.4 `lib/v1/views/teams.js`

**Find** in `lib/v1/views/teams.js`:

```js
import { fmtQ } from '../domain/standings.js';
```

**Replace with:**

```js
import { fmtQ } from '../domain/standings.js';
import { bracketKeyOf, compareBrackets } from '../domain/brackets.js';
```

The Standings tab's "Bracket matchups" sections. **Find** in `lib/v1/views/teams.js`:

```js
  const groupNumbers = Array.from(new Set(matchups.filter(mu => mu.stage === null && mu.group !== null).map(mu => mu.group)))
    .sort((a, b) => a - b);
```

**Replace with:**

```js
  const groupNumbers = Array.from(new Set(matchups.filter(mu => mu.stage === null && mu.group !== null).map(mu => mu.group)))
    .sort(compareBrackets);
```

Match Finder's team index. **Find** in `lib/v1/views/teams.js`:

```js
    group: parseInt(s.bracket, 10) || null,
```

**Replace with:**

```js
    group: bracketKeyOf(s.bracket) || null,
```

The Match Finder's first screen (`introHTML`) groups teams by bracket; "no bracket"
was the key `0` and is now `null`, which `compareBrackets` sorts first, as `0` was.
**Find** in `lib/v1/views/teams.js`:

```js
    const g = t.group === null ? 0 : t.group;
```

**Replace with:**

```js
    const g = t.group;
```

**Find** in `lib/v1/views/teams.js`:

```js
  const groupKeys = Array.from(groups.keys()).sort((a, b) => a - b);
  return `
```

**Replace with:**

```js
  const groupKeys = Array.from(groups.keys()).sort(compareBrackets);
  return `
```

**Find** in `lib/v1/views/teams.js`:

```js
      <h3>${g === 0 ? 'Teams' : `Bracket ${g}`}</h3>
```

**Replace with:**

```js
      <h3>${g === null ? 'Teams' : `Bracket ${g}`}</h3>
```

The "Bracket matchups" toggle reads its bracket from a `data-tm-group` attribute.
**Find** in `lib/v1/views/teams.js`:

```js
      const g = parseInt(groupToggle.dataset.tmGroup, 10);
```

**Replace with:**

```js
      const g = bracketKeyOf(groupToggle.dataset.tmGroup);
```

Then check Control Center for the same assumption:

```bash
grep -n "parseInt([^)]*bracket\|\.group - \|a - b" tools/control-center.html
```

On 31 October this printed nothing about brackets (Control Center draws team
brackets through `lib/v1/views/teams.js`). If it prints a line that parses or
subtracts a bracket, fix it the same way and name it in the report (§13).

### 5.5 Tests

Tests first, as in §4.7.

**Append to the end of** `_tests/unit/domain/teams-awards-attendance-grid-model.test.mjs`:

```js
// ---- lettered brackets (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §5) ---------------------
import { bracketKeyOf, compareBrackets } from '../../../lib/v1/domain/brackets.js';

test('bracketKeyOf: a number or one letter; anything else is no bracket', () => {
  assert.equal(bracketKeyOf('1'), 1);
  assert.equal(bracketKeyOf(' 2 '), 2);
  assert.equal(bracketKeyOf('A'), 'A');
  assert.equal(bracketKeyOf('d'), 'D');
  for (const raw of ['', '#N/A', 'AB', '1A', '-', undefined, null]) assert.equal(bracketKeyOf(raw), null, String(raw));
});

test('compareBrackets: no bracket first, then numbers, then letters', () => {
  assert.deepEqual(['B', 2, null, 'A', 10, 1].sort(compareBrackets), [null, 1, 2, 10, 'A', 'B']);
});

test('groupOf and stageLabel read a lettered bracket', () => {
  const rowByCode = new Map([['A1', { teamCode: 'A1', bracket: 'A' }], ['B', { teamCode: 'B', bracket: '1' }], ['X', { teamCode: 'X', bracket: '' }]]);
  assert.equal(T.groupOf('A1', rowByCode), 'A');
  assert.equal(T.groupOf('QF-A1', rowByCode), 'A');
  assert.equal(T.groupOf('B', rowByCode), 1);
  assert.equal(T.groupOf('X', rowByCode), null);
  assert.equal(T.stageLabel('A1', rowByCode), 'Bracket A');
  assert.equal(T.stageLabel('B', rowByCode), 'Bracket 1');
});
```

The file already imports `test`, `assert` and `T` (`domain/teams.js`).

**Append to the end of** `_tests/unit/views/teams.test.mjs`:

```js
// ---- lettered brackets (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §5) ---------------------
// Two lettered brackets, listed B before A in the sheet, each with one played matchup of two games.
const lettered = buildDayModel(
  { key: 'e', type: 'team', display: {}, days: [], title: 'E', teamScoring: 'games' },
  { key: 'd', date: '2026-11-07' },
  { isLive: true, facilities: [{ name: 'Main',
    matchesCsv: ['matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score',
      '1,B1 v B2,,3:00 PM,Court 1,B1_1,p,q,11,B2_1,r,s,4',
      '2,B1 v B2,,3:00 PM,Court 2,B1_2,p,q,11,B2_2,r,s,6',
      '3,A1 v A2,,3:00 PM,Court 3,A1_1,p,q,3,A2_1,r,s,11',
      '4,A1 v A2,,3:00 PM,Court 4,A1_2,p,q,11,A2_2,r,s,8'].join('\n'),
    standingsCsv: 'teamCode,teamName,bracket\nB1,Bee One,B\nB2,Bee Two,B\nA1,Ay One,A\nA2,Ay Two,A' }] },
  { now: 0, alwaysLive: true },
);

test('lettered brackets: Standings titles and orders them A, B', () => {
  const html = teamStandingsHTML(lettered, {});
  assert.deepEqual([...html.matchAll(/<h3 class="group-title">([^<]+)<\/h3>/g)].map(x => x[1]), ['Bracket A', 'Bracket B']);
  assert.deepEqual([...html.matchAll(/data-tm-group="([^"]+)"/g)].map(x => x[1]), ['A', 'B']);
  const open = teamStandingsHTML(lettered, { openGroups: new Set(['B']) });
  assert.match(open, /data-tm-group="B" aria-expanded="true"/);
  assert.match(open, /data-tm-group="A" aria-expanded="false"/);
});

test('lettered brackets: Match Finder groups the teams under Bracket A and Bracket B', () => {
  const index = buildTeamIndex(lettered);
  assert.deepEqual(index.filter(e => e.kind === 'team').map(e => [e.base, e.group]), [['B1', 'B'], ['B2', 'B'], ['A1', 'A'], ['A2', 'A']]);
  const html = introHTML(lettered, index, '', new Set());
  assert.deepEqual([...html.matchAll(/<h3>([^<]+)<\/h3>/g)].map(x => x[1]), ['Bracket A', 'Bracket B']);
});

test('numbered brackets still read as before', () => {
  const html = teamStandingsHTML(model, {});
  assert.match(html, /<h3 class="group-title">Bracket 1<\/h3>/);
  assert.match(html, /data-tm-group="1"/);
  const intro = introHTML(model, buildTeamIndex(model), '', new Set());
  assert.match(intro, /<h3>Bracket 1<\/h3>/);
  assert.doesNotMatch(intro, /<h3>Teams<\/h3>/);
});
```

The file already imports `teamStandingsHTML`, `buildTeamIndex` and `introHTML`, and
defines `model` (PickleDrive's fixture, numbered brackets).

*Commit.*

---

## 6. Change C: the Tied pill's wording

A bracket tie that every tiebreaker leaves level shows a pill. Today it reads
*Tied · playoff game* (two teams) with the tooltip "one game to 5 decides it", or
*Tied · playoff round robin* (three or more) with "a round robin of games to 5
decides it". Those are Millennium's playoffs. Here the two-team game is to 7 and
the three-team rule is undecided (§1.1). The two-team pill keeps its text and loses
"to 5" from its tooltip. The three-or-more pill reads *Tied · playoff*.

**Find** in `lib/v1/views/teams.js`:

```js
  const tiePill = t => `<span class="tie-pill" title="Level on every tiebreaker: ${t.playoff === 'game' ? 'one game to 5 decides it' : 'a round robin of games to 5 decides it'}">Tied · ${t.playoff === 'game' ? 'playoff game' : 'playoff round robin'}</span>`;
```

**Replace with:**

```js
  const tiePill = t => `<span class="tie-pill" title="Level on every tiebreaker: ${t.playoff === 'game' ? 'a playoff game decides it' : 'a playoff decides it'}">Tied · ${t.playoff === 'game' ? 'playoff game' : 'playoff'}</span>`;
```

The existing test `teamScoring "games": a tie no tiebreaker settles shares its rank
and reads Tied` pins the old three-team text. This is a deliberate, owner-approved
change to what players see, so edit that one assertion and say so in the commit
message. **Find** in `_tests/unit/views/teams.test.mjs`:

```js
  assert.equal((html.match(/<span class="tie-pill"[^>]*>Tied · playoff round robin<\/span>/g) || []).length, 3);
```

**Replace with:**

```js
  assert.equal((html.match(/<span class="tie-pill" title="Level on every tiebreaker: a playoff decides it">Tied · playoff<\/span>/g) || []).length, 3);
```

**Append to the end of** `_tests/unit/views/teams.test.mjs`:

```js
// ---- the Tied pill fits any event (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §6) -------------
test('a two-team tie reads Tied · playoff game, with no point target', () => {
  // A and B split 1-1 with the same points (11-5, 5-11), and each beats C 2-0: level on wins, head-to-head
  // wins and head-to-head quotient.
  const m = buildDayModel(
    { key: 'e', type: 'team', display: {}, days: [], title: 'E', teamScoring: 'games' },
    { key: 'd', date: '2026-11-07' },
    { isLive: true, facilities: [{ name: 'Main',
      matchesCsv: ['matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score',
        '1,A v B,,3:00 PM,Court 1,A_1,p,q,11,B_1,r,s,5', '2,A v B,,3:00 PM,Court 2,A_2,p,q,5,B_2,r,s,11',
        '3,A v C,,3:25 PM,Court 1,A_1,p,q,11,C_1,r,s,0', '4,A v C,,3:25 PM,Court 2,A_2,p,q,11,C_2,r,s,0',
        '5,B v C,,3:50 PM,Court 1,B_1,p,q,11,C_1,r,s,0', '6,B v C,,3:50 PM,Court 2,B_2,p,q,11,C_2,r,s,0'].join('\n'),
      standingsCsv: 'teamCode,teamName,bracket\nA,Alpha,1\nB,Beta,1\nC,Gamma,1' }] },
    { now: 0, alwaysLive: true },
  );
  const html = groupTableHTML(m, '1', m.standings);
  assert.equal((html.match(/<span class="tie-pill" title="Level on every tiebreaker: a playoff game decides it">Tied · playoff game<\/span>/g) || []).length, 2);
  assert.doesNotMatch(html, /to 5/);
});
```

Millennium has finished, and its published standings show no tie, so its archived
pages read the same. *Commit*, with a message such as "Team standings: the Tied
pill names no point target (test edited: it pinned Millennium's wording)".

---

## 7. The event's pages

### 7.1 Copy the templates

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
cp _templates/team-tournament-template/index.html    events/ladies-dink-2026/index.html
cp _templates/team-tournament-template/schedule.html events/ladies-dink-2026/schedule.html
cp _templates/scorer/scorer.html                     events/ladies-dink-2026/scorer.html
cp _templates/attendance/attendance.html             events/ladies-dink-2026/attendance.html
cp _templates/dry-run-checklist-template.md          events/ladies-dink-2026/dry-run-checklist.md
```

### 7.2 Replace the tokens

In all five files, replace every occurrence:

| Token | Value |
|---|---|
| `{{EVENT_KEY}}` | `ladies-dink-2026` |
| `{{EVENT_TITLE}}` | `Ladies' Dink` |
| `{{EVENT_TAGLINE}}` | `Team Event` |
| `{{EVENT_DATE_RANGE}}` | `Saturday, 7 November 2026` |
| `{{VENUE}}` | `Pampanga Pickleball Center` |
| `{{EVENT_HEADLINE}}` | `Ladies got game.` |
| `{{QR_IMAGE}}` | `/events/ladies-dink-2026/assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr>SAGExLD2026` (the `<wbr>` lets it wrap after the `/`, `_templates/CLAUDE.md` §3) |
| `{{EVENT_LOGO}}` | `/events/ladies-dink-2026/assets/logo.png` |
| `{{SCHEDULE_DAY_KEY}}` | `ladies-dink-day1` |

The apostrophe in `Ladies' Dink` needs no escaping: every place it lands is HTML
text or a `"`-quoted attribute. Then:

```bash
grep -n "{{" events/ladies-dink-2026/*.html events/ladies-dink-2026/*.md
```

must print nothing.

### 7.3 The palette, in all four pages

The colours come from the pubmat: charcoal from its title, hot pink from its
blocks, cream from its stripes. **In each of** `index.html`, `schedule.html`,
`scorer.html` and `attendance.html` (four edits, the same Find each time):

**Find** in `events/ladies-dink-2026/index.html`:

```css
    --navy:#14263C;
    --navy-deep:#0B1826;
    --green:#7CB92C;
    --green-dark:#5C8F1F;
    --paper:#F6F7F2;
    --paper-dim:#ECEEE6;
    --line:#D9DED2;
    --ink:#14263C;
    --ink-soft:#5B6B74;
    --white:#FFFFFF;
```

**Replace with:**

```css
    --navy:#26262B;
    --navy-deep:#18181C;
    --green:#E36797;
    --green-dark:#B23A69;
    --paper:#F7F4E6;
    --paper-dim:#EEEAD3;
    --line:#DCD6BC;
    --ink:#232326;
    --ink-soft:#5A5F5E;
    --white:#FFFFFF;
```

Make the same edit in `events/ladies-dink-2026/schedule.html`,
`events/ladies-dink-2026/scorer.html` and `events/ladies-dink-2026/attendance.html`.

Measured contrast (WCAG): `--navy` text on `--green` 4.8:1 (the active tab, the
Advances pill); `--green` on `--navy` 4.8:1; white on `--navy` 15.1:1; `--ink` on
`--paper` 14.2:1; `--ink-soft` on `--paper` 5.9:1 and on `--paper-dim` 5.4:1;
`--green-dark` on white 5.7:1. `--green` on white is 3.2:1, so it is a fill colour
only, as the template's green was. The theme contract
(`_tests/unit/theme-contract.test.mjs`) is satisfied: no property is added to or
removed from the contract list.

### 7.4 `index.html`: the hero

The template's hero is dark (`--navy`). Keep it dark, charcoal now, with the
pubmat's stripes faint across it, its teal and rose for the date and the script, and
its pink for the accents. The order becomes the pubmat's: date and venue, "1st Club
Tournament", the "Team Event" script, then "Ladies got game." on an underline.

**The banner comment.** **Find** in `events/ladies-dink-2026/index.html`:

```css
  /* ============================ THEME ============================
     S.A.G.E. house palette — navy structure, green accent, off-white
     paper — taken from /assets/logo.png and shared verbatim with
     /tools/scoresheet-generator.html, /tools/tournament-calculator.html
     and this event's own schedule.html, so every S.A.G.E. surface reads
     as one product.

     To re-skin an event, change the brand tokens below and nothing else:
     every rule in this file resolves through them or through the role
     aliases underneath.

     Contrast rules this palette has to respect — the green is a *fill*
     colour, not a text colour:
       - --green on white is ~2.3:1 and fails AA at any size. Use it as a
         background (with --navy text on top, ~5.4:1) or on a navy panel.
       - --green-dark on white is ~4.0:1 — still under the 4.5:1 body
         floor, so keep it for large/display text only.
       - Small text on paper is --ink (~13.8:1) or --ink-soft (~5.9:1).
     ================================================================ */
```

**Replace with:**

```css
  /* ============================ THEME ============================
     The Ladies' Dink pubmat's palette (sage-docs/docs/specs/.../ladies-dink-club-tournament-spec.md §7):
     charcoal structure from its title, hot pink accent from its blocks, the
     cream of its stripes for paper, and its teal and rose in the hero. Shared
     with this event's schedule.html, scorer.html and attendance.html.

     To re-skin an event, change the brand tokens below and nothing else:
     every rule in this file resolves through them or through the role
     aliases underneath.

     Contrast rules this palette has to respect — the pink is a *fill*
     colour, not a text colour:
       - --green (the pink) on white is ~3.2:1 and fails AA for body text. Use
         it as a background (with --navy text on top, ~4.8:1) or on a navy panel.
       - --green-dark on white is ~5.7:1 and on paper ~5.1:1.
       - Small text on paper is --ink (~14.2:1) or --ink-soft (~5.9:1).
     ================================================================ */
```

**Two colours of the pubmat's own.** **Find** in `events/ladies-dink-2026/index.html`:

```css
    --radius:14px;

    /* Role aliases
```

**Replace with:**

```css
    --radius:14px;
    --ld-teal:#93C8D3;   /* the pubmat's stripes: the hero's date line */
    --ld-rose:#EBC0BD;   /* the pubmat's "Team Event" script */

    /* Role aliases
```

**Find** in `events/ladies-dink-2026/index.html`:

```css
    --card-shadow: 0 18px 40px -20px rgba(11,24,38,.28);
```

**Replace with:**

```css
    --card-shadow: 0 18px 40px -20px rgba(24,24,28,.28);
```

**The fonts.** Anton (the title), Montserrat (the date and the slogan) and
Sacramento (the script) join the template's three, which the engine's CSS still
uses. **Find** in `events/ladies-dink-2026/index.html`:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

**Replace with:**

```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&family=Montserrat:wght@500;800&family=Sacramento&display=swap" rel="stylesheet">
```

**The hero's order and title.** **Find** in `events/ladies-dink-2026/index.html`:

```html
  <div class="tagline-script">Team Event</div>
  <div class="eyebrow">Saturday, 7 November 2026 &nbsp;·&nbsp; Pampanga Pickleball Center</div>
  <h1 class="title">Ladies' Dink</h1>
  <div class="domination-line">Ladies got game.</div>
```

**Replace with:**

```html
  <div class="eyebrow">Saturday, 7 November 2026 &nbsp;·&nbsp; Pampanga Pickleball Center</div>
  <h1 class="title">1st Club Tournament</h1>
  <div class="tagline-script">Team Event</div>
  <div class="domination-line">Ladies got game.</div>
```

**The hero's CSS**, at the end of the page's `<style>`. **Find** in
`events/ladies-dink-2026/index.html`:

```css
@media (max-width:480px){ .view-tabs{gap:6px;} .view-tab{font-size:11px; padding:9px 12px;} }
</style>
```

**Replace with:**

```css
@media (max-width:480px){ .view-tabs{gap:6px;} .view-tab{font-size:11px; padding:9px 12px;} }
/* ==== Ladies' Dink: the pubmat's hero ==== */
/* The pubmat's horizontal stripes, faint teal on the charcoal hero. */
header.hero{
  background-color:var(--navy);
  background-image:repeating-linear-gradient(180deg, rgba(147,200,211,.07) 0 3px, transparent 3px 12px);
}
/* The logo's own off-white ground, so the tile's edge doesn't show. */
.hero-logos img{ background:#F7F7F0; }
.regal-rule::before{ background:linear-gradient(90deg, transparent, rgba(227,103,151,.7)); }
.regal-rule::after{ background:linear-gradient(90deg, rgba(227,103,151,.7), transparent); }
/* Date and venue, in the pubmat's heavy sans. */
.eyebrow{
  font-family:'Montserrat',sans-serif;
  font-weight:800;
  letter-spacing:.12em;
  color:var(--ld-teal);
  margin-bottom:10px;
}
/* "1st Club Tournament": the pubmat's condensed capitals. */
h1.title{
  font-family:'Anton',sans-serif;
  font-weight:400;
  font-size:clamp(34px,9vw,64px);
  letter-spacing:.01em;
  line-height:1.05;
  margin:0 0 2px;
}
/* "Team Event": the pubmat's rose script, under the title. */
.tagline-script{
  font-family:'Sacramento',cursive;
  font-style:normal;
  font-weight:400;
  font-size:clamp(34px,8vw,48px);
  line-height:1.1;
  color:var(--ld-rose);
  opacity:1;
  margin:-4px 0 8px;
}
/* "Ladies got game.": letter-spaced, on the pubmat's underline (in the accent pink). */
.domination-line{
  display:inline-block;
  font-family:'Montserrat',sans-serif;
  font-weight:500;
  font-size:clamp(13px,3.4vw,16px);
  letter-spacing:.18em;
  color:var(--white);
  border-bottom:3px solid var(--green);
  padding-bottom:4px;
  margin:4px 0 18px;
}
</style>
```

The page stays a shell under 300 lines (233 in the dry run).

### 7.5 `schedule.html`: the brackets' colours

The board tints each card with its category's colour and picks the text colour
itself (`readableOn`). Each colour below carries `--navy` text on its 40% tint at
6:1 or better.

**Find** in `events/ladies-dink-2026/schedule.html`:

```js
const CAT_META = {
  G1: { short:'BR 1',     color:'#1155CC' },
  G2: { short:'BR 2',     color:'#B45F06' },
  G3: { short:'BR 3',     color:'#741B47' },
  G4: { short:'BR 4',     color:'#38761D' },
  PO: { short:'PLAYOFFS', color:'#14263C' }
};
```

**Replace with:**

```js
const CAT_META = {
  GA: { short:'BRACKET A', color:'#2F50D3' },
  GB: { short:'BRACKET B', color:'#B23A69' },
  GC: { short:'BRACKET C', color:'#2B7A88' },
  GD: { short:'BRACKET D', color:'#A0524F' },
  PO: { short:'PLAYOFFS',  color:'#26262B' }
};
```

Blue from the logo's outline, pink from the pubmat, a darkened teal, a darkened
salmon, and the charcoal.

### 7.6 `scorer.html` and `attendance.html`

Nothing beyond §7.2 and §7.3. They are staff pages and stay light.

### 7.7 The dry-run checklist

**Find** in `events/ladies-dink-2026/dry-run-checklist.md`:

```markdown
- [ ] If Mission Control reads *Live updates: polling GitHub (push not connected)*
```

**Replace with:**

```markdown
- [ ] **Playoff tiebreakers.** A playoff matchup at 1–1 plays its **TB** row: type the two nominated players into that row's player cells, then enter its score like any other. A matchup won 2–0 needs nothing: its TB row reads *Not needed* on the board and drops off the Hub's card
- [ ] If Mission Control reads *Live updates: polling GitHub (push not connected)*
```

Commit the five files, but not `assets/pubmat.webp`, which is reference only. Add
`assets/logo.png` and `assets/qr.png` if the owner has saved them:

```bash
git add events/ladies-dink-2026/index.html events/ladies-dink-2026/schedule.html events/ladies-dink-2026/scorer.html events/ladies-dink-2026/attendance.html events/ladies-dink-2026/dry-run-checklist.md
git add events/ladies-dink-2026/assets/logo.png events/ladies-dink-2026/assets/qr.png 2>/dev/null; true
git status --short
```

*Commit.*

---

## 8. Register the event in `event-data`

In `D:\Personal\SAGE\event-data`, open `config/events.json` and add this entry to
`events`, after `"acso-sportsfest-2026"` (mind the comma after that entry's closing
brace):

```json
"ladies-dink-2026": {
  "type": "team",
  "title": "Ladies' Dink 1st Club Tournament",
  "teamScoring": "games",
  "attendance": "desks",
  "scoreEntry": "links",
  "days": {
    "ladies-dink-day1": {
      "label": "Nov 7",
      "date": "2026-11-07",
      "isLive": "auto",
      "facilities": [
        {
          "name": "Pampanga Pickleball Center",
          "sheetId": "1CITP2HOs8Ix5m0qZbkM1GDbDaotWv17i046VeLip7RI"
        }
      ]
    }
  },
  "display": {
    "pairs": {
      "1": {
        "full": "Pair 1",
        "short": "P1"
      },
      "2": {
        "full": "Pair 2",
        "short": "P2"
      }
    },
    "seedLabels": {
      "QF-1": "Bracket A #1",
      "QF-2": "Bracket B #2",
      "QF-3": "Bracket C #1",
      "QF-4": "Bracket D #2",
      "QF-5": "Bracket B #1",
      "QF-6": "Bracket A #2",
      "QF-7": "Bracket D #1",
      "QF-8": "Bracket C #2",
      "SF-1": "QF 1 winner",
      "SF-2": "QF 2 winner",
      "SF-3": "QF 3 winner",
      "SF-4": "QF 4 winner",
      "Br-1": "SF 1 loser",
      "Br-2": "SF 2 loser",
      "Fi-1": "SF 1 winner",
      "Fi-2": "SF 2 winner"
    }
  }
}
```

`display.seedLabels` is what a playoff slot reads before the organiser types a team
into it ("Bracket A #1 · TBD"), following §1.2. Validate, then commit:

```bash
cd /d/Personal/SAGE/event-data
node -e "JSON.parse(require('fs').readFileSync('config/events.json','utf8')); console.log('ok')"
git add config/events.json
git commit -m "Ladies' Dink 1st Club Tournament: register the event (7 Nov 2026, Pampanga Pickleball Center)"
```

Push only on the owner's yes (§11). The API picks it up within `SYNC_CONFIG_TTL_MS`
and the site on its next read. No deploy. It publishes nothing by itself (§0.2).

---

## 9. Verify in a browser

### 9.1 Make the fixture

Save Appendix A's script **outside the repo** (for example in your scratchpad) as
`make-ladies-dink-fixture.mjs`, then from the site repo root:

```bash
node <path-to>/make-ladies-dink-fixture.mjs
```

It must print:

```
72 matches per state; champion A2 (Kitchen Clutchers)
```

It writes `_fixtures/ladies-dink-2026/{pre,mid,finished}.json` and adds the event
to `_fixtures/config.json`. §9.4 removes both.

### 9.2 Serve and open

Serve the **repo root** on `localhost` (fixture mode works only there):

```bash
npx --yes http-server -p 8000 -c-1
```

(Or any static server on the repo root.) Open, with `?fixture=pre`, `mid` and
`finished`:

- the Hub: `http://localhost:8000/events/ladies-dink-2026/?fixture=mid`
- the board: `http://localhost:8000/events/ladies-dink-2026/schedule.html?fixture=mid`
- Control Center: `http://localhost:8000/tools/control-center.html?fixture=finished&event=ladies-dink-2026`
- the scorer and desk pages: `scorer.html` and `attendance.html` beside them

### 9.3 Checks

At **375px** and at desktop width. The dry run of 31 October showed every one of
these.

- [ ] **Hero** (375px): the logo × the S.A.G.E. logo on off-white tiles; then the
      date and venue in teal capitals; "1ST CLUB TOURNAMENT" in Anton, white; "Team
      Event" in rose script; "LADIES GOT GAME." on a pink underline. Faint
      horizontal stripes on charcoal. No horizontal scroll
      (`document.documentElement.scrollWidth === innerWidth`). The active tab is pink
      with charcoal text.
- [ ] **Standings, `mid`:** four tables titled **Bracket A**–**Bracket D**, columns
      W, L, PF, PA, Quotient. Bracket A reads Dink Me Baby, Kitchen Clutchers,
      Adobeshies, Peach Makers: the first two level on 4 wins with a 1–1 head-to-head,
      ordered by head-to-head quotient; the last two level on 2 wins, Adobeshies
      ahead on head-to-head although its quotient is lower. Bracket C reads GRL Power,
      Nacho Average Dinkers, Wok Stars, Pickleball-Silog: three teams on 4 wins,
      ordered by quotient. No Tied pill. The eight qualifiers carry **Advances**.
- [ ] **Playoffs, `mid`:** QF 1 (Dink Me Baby v Spice Girls) reads **Final**, 2 – 0,
      and lists two games, no TB. QF 2 reads **In progress**, 1 – 1, and lists a
      third row, **#58 TB**, "Lineup not set". QF 3 reads **Final**, 1 – 2, with its
      TB row scored 5–7. QF 4 reads VS. The semifinals read "Dink Me Baby v QF 2
      winner · TBD" and "Kitchen Clutchers v QF 4 winner · TBD", and Bronze and Final
      "SF 1 loser · TBD" and so on. A card's time line includes its TB slot ("6:45 &
      7:10 & 7:35 PM"); that is expected.
- [ ] **"Bracket matchups"**: four toggles, Bracket A–D, each "6 matchups", each
      opening its own matchups.
- [ ] **Match Finder** (first screen): team chips grouped under Bracket A–D.
- [ ] **Facility card / Mission Control:** `mid` has 16 matches left (QF 1's TB is
      not counted); `finished` reads **All matches done**.
- [ ] **Board, `finished`:** bracket chips BRACKET A–D and PLAYOFFS in §7.5's
      colours. Four TB cards greyed **Not needed**: #57 (QF 1), #60 (QF 4), #66
      (SF 2), #71 (Bronze). Tiebreaker cards read **TB**.
- [ ] **Control Center, `finished`, Awards:** gold Kitchen Clutchers (won the Final
      2–1 on its tiebreaker), silver Dink Me Baby, bronze GRL Power.
- [ ] **Control Center, `mid`:** opening the score dialog on #57 (QF 1's TB) is
      refused as not needed, as for a series game. #58 opens normally.
- [ ] The scorer page lists TB rows and refuses #57 the same way.
- [ ] No console errors on any page, except a 404 for `assets/qr.png` or
      `assets/logo.png` if the owner hasn't saved them yet.

If a check fails, the cause is in §4–§7. Fix it there, never in the fixture.

### 9.4 Remove the fixture

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/ladies-dink-2026
git status --short
```

`git status` must show nothing under `_fixtures/`. The fixture is never committed:
the harness reads `_fixtures/config.json`, and an extra event there changes Control
Center's event list in its screenshots.

---

## 10. The full test run

With the fixture removed (§9.4):

```bash
cd /d/Personal/SAGE/sage-match-control.github.io/_tests
npm run verify
```

`npm test` must pass (488 tests in the dry run) and the comparison harness must
show **no difference** on any existing page. Changes A, B and C change nothing an
existing fixture shows: no fixture has `_TB` codes, lettered brackets or a tie.
Treat any difference as a bug in §4–§6 until shown otherwise. If you're sure one is
intended, stop and ask the owner before adding a row to
`_tests/engine/accepted.mjs`.

Then:

```bash
cd /d/Personal/SAGE/sage-tools-api
npm run verify
```

---

## 11. Pushes: stop and ask

Ask the owner in chat before each. Each needs its own yes.

- **Push 1, the engine.** First the API: merge `ladies-dink-tiebreaker` into
  `sage-tools-api`'s `main` and push. That deploys through the Cloud Run trigger.
  Then check the version:

  ```bash
  curl -sI https://sage-tools-api-<…>.run.app/ping | grep -i x-app-version
  ```

  (The base URL is `CLOUD_RUN_BASE_URL` in the site's `lib/v1/platform.js`.) It
  must read `3.5.0`. Then the site's §4–§6 commits: push them to `main` (merge the
  branch up to and including §6's commit). The API goes first so the completion stamp
  knows the tiebreaker rule before any page does. This push also carries `416807d`, Millennium's
  live-push switch-off (§0.3); say so when you ask.
- **Push 2, the pages.** §7, merged to the site's `main`.
- **`event-data`**: §8.

Both site pushes land by **Tuesday 3 November**. After that the site is frozen until
the event. Never push to `main` from 4 to 6 November.

---

## 12. Documentation

In `D:\Personal\SAGE\sage-docs` (each page describes the system as it is, in the
present tense):

- `docs/technical/site-engine.md`, "The model every view reads" and the team rules
  around its line on `teamScoring`: add `domain/deciders.js` (the tiebreaker rule,
  §4.1, and that `unneededSeriesGames` returns it) and `domain/brackets.js`
  (`bracketKeyOf`, `compareBrackets`, §5.1). Replace "*Tied · playoff game* or *Tied
  · playoff round robin*" with "*Tied · playoff game* (two teams) or *Tied ·
  playoff* (three or more)".
- `docs/technical/sync-pipeline.md`, where it lists the rules `completedAt` mirrors
  (`seriesGameOf`/`unneededSeriesGames`): add the tiebreaker rule and
  `deciders.js`.
- `docs/features/tournament-hub.md`, under "Events scored on games won": a playoff
  matchup's tiebreaker (listed as TB while it may be needed, gone once the matchup
  is settled 2–0), lettered brackets, and the new Tied wording.
- `docs/features/schedule-board.md`: a tiebreaker card reads TB, greyed *Not
  needed* once the matchup is settled.
- `docs/features/team-workbook.md`: a playoff matchup may carry a tiebreaker row,
  coded `<side>_TB`, sharing the matchup's `matchUp`, with its player cells typed on
  the day; and `STANDINGSCSV`'s `bracket` may be a letter.
- This spec: move it to `in-progress/` once Push 1 is done, by the procedure in
  `docs/specs/README.md`, and update its status line.
- A MINOR version (`docs/changelog.md` and `mkdocs.yml`'s `copyright`) for each
  commit that changes the docs.

In `sage-match-control.github.io`, `_templates/CLAUDE.md` §5 ("Team code format"):
add that a playoff matchup's tiebreaker game is coded `<SIDE>_TB` and isn't counted
once the matchup's other games settle it, and that `STANDINGSCSV`'s `bracket` may
be a number or one letter. That is part of Push 2.

In the workspace `D:\Personal\SAGE\CLAUDE.md`, "Things that must be kept in sync by
hand", the paragraph on the "played" rule: add "A team playoff matchup's tiebreaker
game (both codes ending `_TB`) doesn't count once the matchup's other games are all
played and one side has won more of them." Commit it in the workspace repo; push on
the owner's yes.

---

## 13. Report back

Say, per repo: the commits, and what was pushed with the owner's yes. Also report:
any Find block that didn't match once; any Control Center line §5.4's grep turned
up; any harness difference; whether `assets/logo.png` and `assets/qr.png` were
present; and anything in §9.3 that didn't look as described.

---

## 14. Operator tasks: the workbook (people, not the implementer)

This list was sent to the people fixing the workbook on 31 October. It is here so the
spec is complete. **The implementer doesn't edit the workbook.**

### 14.1 Fixes before sync setup

As of the last export of 31 October. The workbook was edited throughout that day, so
re-check each item before relying on it.

**Done:**

- The test scores are cleared, `Awards` reads PENDING, and "PICKLEBALL-SILOG" is
  spelled right. `FINAL RANK` is gone, so §1.2's crossover stands.
- **All 8 tiebreaker rows** are in `MatchUps`, `SCHEDULE` and `CSV`, coded
  `<side>_TB`, each sharing its matchup's `matchUp` value, with its own match number in
  time order: groups 1–48, QF games 49–56, QF tiebreakers 57–60, SF games 61–64, SF
  tiebreakers 65–66, Bronze and Final games 67–70, their tiebreakers 71–72. `CSV` lists
  all 72 with no `#REF!` rows and no junk rows below. Their player cells repeat the
  code, which the site reads as "Lineup not set".
- **`STANDINGSCSV` is rebuilt**, with the owner's go-ahead, 33 rows:
  - rows 2–17: `A1`…`D4` typed in column A; `teamName`
    `=IFNA(FILTER(Variables!T$2:T,Variables!S$2:S=A2),"")`; `totalPoints`,
    `totalOpponentPoints` and `quotient` the workbook's own formulas
    (`GETMATCHSCORESBYMATCHUPANDTEAMCODE`, `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE`,
    `IFERROR(C/D,"-")`); `bracket` `=LEFT(A2,1)`, so `A`…`D`;
  - rows 18–33: the 16 playoff slots, each `=MatchUps!E<row>` (QF-1 `E484`, QF-2
    `E494`, … Fi-2 `E634`, ten rows apart), so a slot follows when the organiser types
    a qualifier over it; `teamName` looks up the code after the `-`; the same points
    formulas; `bracket` blank.

  Read through the site engine with §5's change, the workbook gives 6 matchups in
  each of Bracket A–D, 4 QF, 2 SF, Bronze and Final, and recognises all 8 tiebreakers.

**Still to do:**

1. **Team codes.** The `Teams` tab's Team Code column (`A1`…`D4`) is empty while the
   organiser redoes the draw, so `MatchUps` shows "-" for every player, and
   `Standings` and `STANDINGSCSV` have no team names. They fill in by themselves once
   the codes are back; nothing else needs touching.
2. **Player names in `CSV`.** Its player columns carry pair codes (`A1_1 `), not names.
   Check again once the codes are back, and if they still show codes, point `SCHEDULE`'s
   player cells at `MatchUps`' names.
3. **Captains.** Move "– Capt." out of the 16 captains' names into the `Notes` column
   (§15 Q6) once the draw is done; the owner has asked S.A.G.E. to do it. C1 (Wok Stars)
   still has no player 3.
4. **Small leftovers:** `Court Control` has match 1 on Court 1 (test data) and lists 12
   courts for a 6-court venue (harmless: the site doesn't read it); `Variables` lists the
   tiebreakers quotient-first, so make it match §1.1.
5. **`ATTENDANCE`** still holds Millennium's people. Leave it: once §8 is pushed, run
   Control Center's **Update roster** for this day.
6. **The timetable starts at 3:30 PM**, not the pubmat's 3:00 (§15 Q11): groups
   3:30–6:25, QF games 7:15 and 7:40, QF tiebreakers 8:05, SF 8:30, SF tiebreakers 8:55,
   Bronze and Final 9:20, their tiebreakers 9:45. The SF, Bronze and Final tiebreakers
   are on courts 2 and 4. The site reads times and courts from the workbook, so nothing
   in §4–§8 depends on this; only §1's table and Appendix A's made-up schedule differ.

### 14.2 Share it, then set up live sync, once, last

Share the workbook with
`sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**, and set
General access to **Restricted** (`_templates/CLAUDE.md` §2 step 13). Then
**SAGE → Set up live sync** with day key `ladies-dink-day1` and facility
`Pampanga Pickleball Center` (exact, case-sensitive), then **SAGE → Sync now**, and
check `https://sage-match-control.github.io/event-data/ladies-dink-2026/data/ladies-dink-day1.json`:
72 matches, 64 players. Requires §8 pushed first.

### 14.3 On the day

After the group stage the organiser types the 8 qualifiers into the QF slots
(§1.2). If a playoff matchup goes 1–1, the operator types the nominated pair's names
into its tiebreaker row, and its score goes in like any other.

### 14.4 QR, hub board and dry run

The owner makes the short link and `qr.png` (§2), then the hub board's QR panel
(`node _templates/hub-pubmat/render.mjs ladies-dink-2026`). The dry run
(`events/ladies-dink-2026/dry-run-checklist.md`) happens by **Thursday 5 November**.

---

## 15. Questions, answered by the owner on 31 October 2026

| # | Question | Answer | Where |
|---|---|---|---|
| Q1 | The notes say the top 2 "proceed to the semis", but 8 qualifiers need quarterfinals. The playoff path? | QF, SF, Bronze, Final, as the workbook has it | §1, §1.2 |
| Q2 | Two teams level on wins whose head-to-head is 1–1: what next? | The head-to-head quotient. Still level: a tiebreaker game to 7 | §1.1 |
| Q3 | How are the 8 qualifiers seeded? | Bracket crossover | §1.2, §8 `seedLabels` |
| Q4 | Lettered brackets on the site, or publish `1`–`4`? | Letters A–D | §5, §7.5 |
| Q5 | The tiebreaker's label? | "Tiebreaker" / "TB" | §4.5 |
| Q6 | Captains' names carry "– Capt.". Keep it in the name? | No: move it to `Notes` | §14.1 item 8 |
| Q7 | Check-in and score entry? | Check-in desk links and scorer links | §8 |
| Q8 | The short link? | `tinyurl.com/SAGExLD2026` | §1, §7.2 |
| Q9 | The facility name? | "Pampanga Pickleball Center" | §7.2, §8, §14.2 |
| Q10 | Three or more teams level on wins **and** quotient: what settles it? | **Open.** The site shows them as Tied either way (§6); the organiser needs the answer before the event | §1.1 step 3.3 |
| Q11 | The workbook now starts at 3:30 PM; the pubmat says 3:00. Which is first serve? | **Open.** The site reads times from the workbook, so the build doesn't wait for it | §1, §14.1 item 9 |
| — | The Tied pill named Millennium's "to 5". What should it say? | Generic wording | §6 |

---

## 16. Decisions, for reference

- **D1. Title.** `Ladies' Dink` is the short title (tab, footer, board, Control
  Center). `events.json`'s `title` carries the full name.
- **D2. A fixed tiebreaker label.** A configurable one would need a new `display`
  field and its validation for one event. "Tiebreaker" fits any team event.
- **D3. A group matchup at 1–1 shows a points winner.** The organiser's ranking
  ignores matchup wins, and changing it would change Millennium's archived pages.
- **D4. The Tied wording is generic, not per event.** A per-event text would need a
  new `events.json` field in a tight week. "A playoff game decides it" is true for
  Millennium and for this event.
- **D5. No team colours.** Sixteen teams exceed the shapes that keep marks apart.
- **D6. The tiebreaker is a code suffix (`_TB`), not a pair number or an
  `events.json` setting.** The API's completion stamp reads only the CSV, so the
  rule must be visible in the CSV alone. No other event uses the suffix, so nothing
  else changes.
- **D7. A tiebreaker isn't a series.** `walkSeries` (awards, series finals) is left
  alone. Only the "not needed" set is shared, because every consumer already
  respects it.
- **D8. New modules for new helpers.** `deciders.js` and `brackets.js` are new files
  so existing modules can use them in the same push (§0.1).
- **D9. The hero stays dark.** The pubmat is light, but the engine's hero CSS (tabs,
  day picker, sync line) is built for a dark hero. Charcoal with the pubmat's stripes
  keeps every engine rule working with no overrides beyond §7.4.

---

## Appendix A — the fixture script

Used by §9 only, and never committed. Save it outside the repo and run it from the
site repo root.

```js
// Writes a temporary, local-only fixture for the Ladies' Dink 1st Club Tournament, shaped like the real event:
// 16 teams (A1-D4) in four lettered brackets, 2-game matchups, the workbook's own schedule (25-minute slots,
// 6 courts), crossover quarterfinals, then semifinals, Bronze and Final, each playoff matchup with a tiebreaker
// row (codes ending "_TB"). Three states, each a snapshot:
//   pre        no scores; the playoff slots still read their seeds
//   mid        group stage played; Bracket A has two teams level on wins with a 1-1 head-to-head, Bracket C three
//              teams level on wins; QF 1 won 2-0 (tiebreaker blank), QF 2 at 1-1 (tiebreaker blank), QF 3 at 1-1
//              with its tiebreaker 5-7, QF 4 not started
//   finished   every match played; QF 2, SF 1 and the Final settled by their tiebreakers
// Files: _fixtures/ladies-dink-2026/{pre,mid,finished}.json, and _fixtures/config.json gains the event.
// Neither is ever committed. Undo with:
//   git checkout -- _fixtures/config.json && rm -rf _fixtures/ladies-dink-2026
// usage, from the site repo root: node <path>/make-ladies-dink-fixture.mjs
import fs from 'node:fs';
import path from 'node:path';

const EVENT = 'ladies-dink-2026', DAY = 'ladies-dink-day1', FACILITY = 'Pampanga Pickleball Center';
const TEAMS = {
  A1: 'Dink Me Baby', A2: 'Kitchen Clutchers', A3: 'Adobeshies', A4: 'Peach Makers',
  B1: 'Eggcellent Dinkers', B2: 'Spice Girls', B3: 'Pressure Cookers', B4: 'Kitchen Inasal',
  C1: 'Wok Stars', C2: 'GRL Power', C3: 'Nacho Average Dinkers', C4: 'Pickleball-Silog',
  D1: 'Whisk Takers', D2: 'Chicken Ala Dink', D3: 'Donut Get Smashed', D4: 'Cereal Killers',
};
// Made-up players: pair 1 is players 1-2, pair 2 players 3-4.
const player = (team, n) => `${TEAMS[team].split(' ')[0]} ${team}-${n}`;
const pairNames = (team, pair) => pair === 1 ? [player(team, 1), player(team, 2)] : [player(team, 3), player(team, 4)];

// The workbook's group stage, in match order: [matchUp sides, time, first court]. Each matchup is two rows,
// pair 1 on the first court and pair 2 on the next.
const GROUP = [
  ['A1 A2', '3:00 PM', 1], ['A3 A4', '3:00 PM', 3], ['B1 B2', '3:00 PM', 5],
  ['B3 B4', '3:25 PM', 1], ['C1 C2', '3:25 PM', 3], ['C3 C4', '3:25 PM', 5],
  ['D1 D2', '3:50 PM', 1], ['D3 D4', '3:50 PM', 3], ['A1 A3', '3:50 PM', 5],
  ['A2 A4', '4:15 PM', 1], ['B1 B3', '4:15 PM', 3], ['B2 B4', '4:15 PM', 5],
  ['C1 C3', '4:40 PM', 1], ['C2 C4', '4:40 PM', 3], ['D1 D3', '4:40 PM', 5],
  ['D2 D4', '5:05 PM', 1], ['A1 A4', '5:05 PM', 3], ['A2 A3', '5:05 PM', 5],
  ['B1 B4', '5:30 PM', 1], ['B2 B3', '5:30 PM', 3], ['C1 C4', '5:30 PM', 5],
  ['C2 C3', '5:55 PM', 1], ['D1 D4', '5:55 PM', 3], ['D2 D3', '5:55 PM', 5],
];
// Each group game's score [left, right], pair 1 then pair 2, keyed by the matchup.
//   A: A1 and A2 win 4 each and split 1-1 (A1 20-14 on points); A3 beats A4 2-0, so both have 2.
//   B and D: 1 sweeps, then 2, then 3.
//   C: C1, C2, C3 win 4 each, split 1-1 among themselves, and sweep C4; overall quotient C2 > C3 > C1.
const GROUP_SCORES = {
  'A1 A2': [[11, 3], [9, 11]], 'A3 A4': [[11, 6], [11, 8]], 'A1 A3': [[11, 7], [11, 5]],
  'A2 A4': [[11, 9], [6, 11]], 'A1 A4': [[11, 4], [8, 11]], 'A2 A3': [[11, 2], [11, 6]],
  'B1 B2': [[11, 5], [11, 7]], 'B3 B4': [[11, 8], [11, 9]], 'B1 B3': [[11, 4], [11, 3]],
  'B2 B4': [[11, 6], [11, 2]], 'B1 B4': [[11, 1], [11, 5]], 'B2 B3': [[11, 9], [11, 7]],
  'C1 C2': [[11, 9], [4, 11]], 'C3 C4': [[11, 6], [11, 7]], 'C1 C3': [[11, 10], [3, 11]],
  'C2 C4': [[11, 1], [11, 0]], 'C1 C4': [[11, 9], [11, 8]], 'C2 C3': [[11, 8], [6, 11]],
  'D1 D2': [[11, 6], [11, 8]], 'D3 D4': [[11, 7], [11, 9]], 'D1 D3': [[11, 2], [11, 5]],
  'D2 D4': [[11, 3], [11, 4]], 'D1 D4': [[11, 0], [11, 6]], 'D2 D3': [[11, 9], [11, 10]],
};
// The qualifiers, by the organiser's rules (§1.1): A1, A2 (A1 on head-to-head points); B1, B2;
// C2, C3 (quotient); D1, D2. Crossover seats (§1.2): QF-1..QF-8.
const QF_SEATS = ['A1', 'B2', 'C2', 'D2', 'B1', 'A2', 'D1', 'C3'];

// The playoff schedule, with the tiebreaker rows (§14.1 item 4). [stage, matchup number, time, court, pair]
const PLAYOFF = [
  ...[1, 2, 3, 4].map(k => ['QF', k, '6:45 PM', k, 1]),
  ...[1, 2, 3, 4].map(k => ['QF', k, '7:10 PM', k, 2]),
  ...[1, 2, 3, 4].map(k => ['QF', k, '7:35 PM', k, 'TB']),
  ['SF', 1, '8:00 PM', 1, 1], ['SF', 1, '8:00 PM', 2, 2], ['SF', 2, '8:00 PM', 3, 1], ['SF', 2, '8:00 PM', 4, 2],
  ['SF', 1, '8:25 PM', 1, 'TB'], ['SF', 2, '8:25 PM', 3, 'TB'],
  ['Br', 1, '8:50 PM', 1, 1], ['Br', 1, '8:50 PM', 2, 2], ['Fi', 1, '8:50 PM', 3, 1], ['Fi', 1, '8:50 PM', 4, 2],
  ['Br', 1, '9:15 PM', 1, 'TB'], ['Fi', 1, '9:15 PM', 3, 'TB'],
];
// Playoff scores per state, keyed "<stage><k>" -> [game 1, game 2, tiebreaker], each [left, right] or null.
const PLAYOFF_SCORES = {
  mid: {
    QF1: [[11, 4], [11, 6], null], QF2: [[11, 8], [7, 11], null], QF3: [[11, 9], [5, 11], [5, 7]], QF4: [null, null, null],
  },
  finished: {
    QF1: [[11, 4], [11, 6], null], QF2: [[11, 8], [7, 11], [7, 3]], QF3: [[11, 9], [5, 11], [5, 7]], QF4: [[11, 2], [11, 5], null],
    SF1: [[9, 11], [11, 6], [7, 6]], SF2: [[11, 7], [11, 9], null],
    Br1: [[11, 5], [11, 8], null], Fi1: [[11, 9], [8, 11], [4, 7]],
  },
};

const csvCell = v => v === null || v === undefined ? '' : String(v);
const HEADER = 'matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score,lineupShown';

function build(state){
  const rows = [];
  let num = 0;
  // Match numbers follow SCHEDULE's order (time, then court): the group stage 1-48, then the playoff rows in PLAYOFF's order.
  const push = (matchUp, time, court, c1, n1, s1, c2, n2, s2, at = ++num) =>
    rows.push([at, matchUp, '', time, `Court ${court}`, c1, n1[0], n1[1], csvCell(s1), c2, n2[0], n2[1], csvCell(s2), 'FALSE'].join(','));

  GROUP.forEach(([sides, time, court]) => {
    const [x, y] = sides.split(' ');
    const sc = state === 'pre' ? [[null, null], [null, null]] : GROUP_SCORES[sides];
    [1, 2].forEach(p => push(`${x} v ${y}`, time, court + p - 1, `${x}_${p}`, pairNames(x, p), sc[p - 1][0], `${y}_${p}`, pairNames(y, p), sc[p - 1][1]));
  });

  // Who sits in each playoff slot in this state: a team code once known, else the seed number.
  const scores = PLAYOFF_SCORES[state] || {};
  const winnerOf = {}, loserOf = {};
  const slot = (stage, k, side) => {
    if (state === 'pre') return null;
    if (stage === 'QF') return QF_SEATS[(k - 1) * 2 + side - 1];
    const prev = stage === 'SF' ? ['QF', 2 * k - 1 + side - 1] : ['SF', side];
    const key = `${prev[0]}${prev[1]}`;
    return stage === 'Br' ? loserOf[key] || null : winnerOf[key] || null;
  };
  const seedNo = (stage, k, side) => stage === 'QF' || stage === 'SF' ? 2 * (k - 1) + side : side;
  const sideCode = (stage, k, side) => `${stage}-${slot(stage, k, side) || seedNo(stage, k, side)}`;

  const decide = (key, a, b) => {
    const g = scores[key];
    if (!g) return;
    let wa = 0, wb = 0;
    g.forEach(s => { if (s) { if (s[0] > s[1]) wa++; else wb++; } });
    const done = g[0] && g[1] && (wa !== wb);
    if (done) { winnerOf[key] = wa > wb ? a : b; loserOf[key] = wa > wb ? b : a; }
  };
  // Settle stages in order so later slots can be filled.
  const playoffStanding = [];
  ['QF', 'SF', 'Br', 'Fi'].forEach(stage => {
    PLAYOFF.map((r, i) => [r, GROUP.length * 2 + 1 + i]).filter(([r]) => r[0] === stage).forEach(([[, k, time, court, pair], at]) => {
      const a = sideCode(stage, k, 1), b = sideCode(stage, k, 2);
      const ta = slot(stage, k, 1), tb = slot(stage, k, 2);
      const g = (scores[`${stage}${k}`] || [])[pair === 'TB' ? 2 : pair - 1] || [null, null];
      const names = (team, code) => pair === 'TB' || !team ? [code, code] : pairNames(team, pair);
      push(`${a} v ${b}`, time, court, `${a}_${pair}`, names(ta, `${a}_${pair}`), g[0], `${b}_${pair}`, names(tb, `${b}_${pair}`), g[1], at);
    });
    const ks = [...new Set(PLAYOFF.filter(r => r[0] === stage).map(r => r[1]))];
    ks.forEach(k => {
      [1, 2].forEach(side => playoffStanding.push([sideCode(stage, k, side), TEAMS[slot(stage, k, side)] || '']));
      decide(`${stage}${k}`, slot(stage, k, 1), slot(stage, k, 2));
    });
  });

  rows.sort((x, y) => parseInt(x, 10) - parseInt(y, 10));

  // STANDINGSCSV: the 16 teams (their points from the group games), then every playoff side.
  const pts = {};
  rows.forEach(r => {
    const c = r.split(',');
    if (c[8] === '' || c[12] === '' || c[5].includes('-')) return;
    const a = c[5].split('_')[0], b = c[9].split('_')[0];
    pts[a] = pts[a] || [0, 0]; pts[b] = pts[b] || [0, 0];
    pts[a][0] += +c[8]; pts[a][1] += +c[12]; pts[b][0] += +c[12]; pts[b][1] += +c[8];
  });
  const q = (f, a) => a ? (f / a).toFixed(5) : '-';
  const standings = ['teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket',
    ...Object.keys(TEAMS).map(t => { const p = pts[t] || [0, 0]; return `${t},${TEAMS[t]},${p[0]},${p[1]},${q(p[0], p[1])},${t[0]}`; }),
    ...playoffStanding.map(([code, name]) => `${code},${name},0,0,-,`)];
  const roster = ['teamCode,player,level,gender', ...Object.keys(TEAMS).flatMap(t => [1, 2, 3, 4].map(n => `${t},${player(t, n)},,W`))];

  const at = state === 'pre' ? '2026-11-07T05:00:00.000Z' : state === 'mid' ? '2026-11-07T11:20:00.000Z' : '2026-11-07T13:40:00.000Z';
  return {
    rows: rows.length, champion: winnerOf.Fi1,
    snapshot: {
      day: DAY, label: 'Nov 7', isLive: true, generatedAt: at, publishedAt: at,
      facilities: [{ name: FACILITY, matchesCsv: [HEADER, ...rows].join('\r\n'), standingsCsv: standings.join('\r\n'), rosterCsv: roster.join('\r\n'), syncedAt: at, lastEditAt: at }],
      failedFacilities: [], staleFacilities: [],
    },
  };
}

fs.mkdirSync(path.join('_fixtures', EVENT), { recursive: true });
const out = {};
for (const state of ['pre', 'mid', 'finished']) {
  out[state] = build(state);
  fs.writeFileSync(path.join('_fixtures', EVENT, `${state}.json`), JSON.stringify(out[state].snapshot, null, 2) + '\n');
}

const configPath = path.join('_fixtures', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.events[EVENT] = {
  type: 'team', title: "Ladies' Dink 1st Club Tournament", teamScoring: 'games', attendance: 'desks', scoreEntry: 'links',
  days: { [DAY]: { label: 'Nov 7', date: '2026-11-07', isLive: 'auto', facilities: [{ name: FACILITY, sheetId: 'FIXTURELADIESDINKSHEET000000000000' }] } },
  display: {
    pairs: { 1: { full: 'Pair 1', short: 'P1' }, 2: { full: 'Pair 2', short: 'P2' } },
    seedLabels: {
      'QF-1': 'Bracket A #1', 'QF-2': 'Bracket B #2', 'QF-3': 'Bracket C #1', 'QF-4': 'Bracket D #2',
      'QF-5': 'Bracket B #1', 'QF-6': 'Bracket A #2', 'QF-7': 'Bracket D #1', 'QF-8': 'Bracket C #2',
      'SF-1': 'QF 1 winner', 'SF-2': 'QF 2 winner', 'SF-3': 'QF 3 winner', 'SF-4': 'QF 4 winner',
      'Br-1': 'SF 1 loser', 'Br-2': 'SF 2 loser', 'Fi-1': 'SF 1 winner', 'Fi-2': 'SF 2 winner',
    },
  },
};
fs.writeFileSync(configPath, JSON.stringify(config, null, 2) + '\n');

console.log(`${out.pre.rows} matches per state; champion ${out.finished.champion} (${TEAMS[out.finished.champion]})`);
```
