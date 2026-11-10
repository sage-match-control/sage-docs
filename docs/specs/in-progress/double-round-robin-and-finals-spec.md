# Spec — Double round robin, best-of-3 round robin and one-bracket finals

> **Status: in progress.** Built and live: the calculator, Standard Tournament Generator 1.5.0
> (in `sage-tools-api`'s `apps-script/`), `sage-tools-api` 3.7.0 (deployed) and the site
> engine's best-of-3 rule (both pushes), with their tests and the docs, all on `main` in each
> repo. Left: the paste of the generator into the Standard Tournament Master (§7), which still
> runs 1.4.1, the owner's checks of §9 steps 2 and 5 (the generated Format B and `GBMD`
> workbooks, and the dry run), and moving this spec to `implemented/`.
> Written 2026-11-10 against
> `sage-match-control.github.io/tools/tournament-calculator.html`,
> `sage-tools-api/apps-script/standard-tournament-generator.gs` 1.4.1, `sage-tools-api` 3.6.2
> (`src/sync/domain/facilityCompletion.mjs`) and the site engine `lib/v1/` as of that date.
> Every decision is taken (§11) except the first event's name and date.

This spec is written to be built by someone with no other context. §0 is the briefing:
read it first, then build in the order of §9.

---

## 0. Briefing

### 0.1 What you are building

Pickleball tournaments are planned in the **Tournament Calculator** (a web page). Its plan is
then turned into a Google Sheets workbook by the **Standard Tournament Generator** (an Apps
Script). A category (for example "Novice Mixed Doubles") is split into **brackets**. Each
bracket plays a **round robin** (every pair plays every other pair). The best pairs then
play a **playoff**.

When a category has exactly **one bracket**, the tools offer two playoff formats today:

- `ttb`: twice-to-beat. RR #1 v RR #2 for gold, where #1 needs one win and #2 must win twice.
  RR #3 takes bronze without playing.
- `rr`: no playoff. The round-robin standings decide gold, silver and bronze.

You add three things. The first two apply to **one-bracket categories only**.

1. **A double round robin** (`rr_legs: 2`): every pairing plays twice, as a "leg 1" and a
   "leg 2".
2. **Two more one-bracket playoff formats** (`solo_format`):
   - `final_bronze`: RR #1 v RR #2, one game, for gold. RR #3 v RR #4, one game, for bronze.
     Needs 4 or more pairs.
   - `final`: RR #1 v RR #2, one game, for gold. RR #3 takes bronze without playing.

Any leg count goes with any of the four formats, which makes eight combinations:

| `rr_legs` | `final_bronze` | `final` | `ttb` | `rr` |
| --- | --- | --- | --- | --- |
| 1 (single, default) | **new** | **new** | exists | exists |
| 2 (double) | **new** | **new** | **new** | **new** |

The organiser who asked for this wants two of them: single + `final_bronze` ("Format A") and
double + `final` ("Format B"). They also want to be able to choose a bronze match or
twice-to-beat after a double round robin, so all eight are built.

3. **Best-of-3 round-robin meetings** (`rr_best_of: 3`), in a category with **any number of
   brackets** ("Format C"). Every pairing meets once, and each meeting is a best-of-3 series:
   the first pair to win two games takes it, so game 3 is played only at 1–1. The playoff is
   the one that category would have anyway, and every playoff match keeps its usual length:
   the multi-bracket ladder (QF, SF, Final, Bronze) is one game a match, and a one-bracket
   category still chooses `ttb`, `final_bronze`, `final` or `rr`. In the standings, **W and L
   count meetings won and lost**, not games. The quotient counts the points of every game
   played.

   Best of 3 does **not** combine with a double round robin: `rr_best_of: 3` and `rr_legs: 2`
   together are rejected. A 2-pair category already plays a best-of-3 Final and has no round
   robin, so best of 3 does not apply to it.

Items 1 and 2 need no change on the site and none in the sync API. Item 3 does: today, only a
series **Final** (`<KEY>_F_<seat>_(<game>)`) knows that a game is not needed. To count a
best-of-3 meeting, the site's `lib/v1/domain/` and the API's `facilityCompletion.mjs` learn a
second kind of series (§5, §6).

### 0.2 Where things are

The workspace root is `D:\Personal\SAGE`. It holds four independent git repos, and you change
three of them:

| Repo | What you change | Section |
| --- | --- | --- |
| `sage-match-control.github.io/` (GitHub Pages site, no build step) | `tools/tournament-calculator.html` (the calculator, one self-contained HTML file); for best of 3, `lib/v1/domain/matches.js`, `series.js`, `schedule-grid.js` and `standings.js`; tests in `_tests/unit/`; `_templates/dry-run-checklist-template.md` | §3, §6, §8 |
| `sage-tools-api/` | `apps-script/standard-tournament-generator.gs` and `apps-script/verify-standard-tournament-generator.mjs`. For best of 3 only: `src/sync/domain/facilityCompletion.mjs`, its test, `package.json` and the `README.md` changelog | §4, §5 |
| `sage-docs/` (mkdocs) | feature, technical and usage pages, this spec's status | §8 |

The workspace root itself is a fourth repo (`sage-workspace`). It holds `CLAUDE.md`, and you
update one bullet there (§8).

Read these before starting:

- The root `CLAUDE.md` (repo rules for all of the above).
- `sage-docs/docs/specs/implemented/standard-tournament-master-spec.md`: §5 (playoff shapes),
  §7 (the category tab), §8 (team codes) and §10.2.2 (`MATCHES`). Every workbook term used
  below is defined there.

### 0.3 Rules you must follow

- **Branches.** Work on a new branch in each repo (`double-rr-and-finals`). Do not push and
  do not merge to `main`: the owner merges. Commit only if the owner asks you to. When you
  commit in the site repo, put the `standings.js` change and its test (§6.2, second push) in a
  commit of their own, **last** on the branch, so the owner can merge everything before it
  first. If you do not commit, tell the owner which files belong to that second push.
- **Timing.** No merge within 3 days of an event. The Angeles City Employees Sportsfest is
  on 13 November 2026, so nothing merges before 14 November.
- **Line endings.** `tournament-calculator.html`, `standard-tournament-generator.gs` and
  `verify-standard-tournament-generator.mjs` use **CRLF**. Keep it.
- **Style.** The calculator is a classic script with 2-space indent and single quotes. The
  generator is Apps Script (ES5-style `var` and `function`, no arrow functions, no
  `let`/`const`) with 2-space indent and single quotes. The verify script is ESM with 4-space
  indent and double quotes. Copy the style of the code around you.
- **Existing behaviour must not change.** A plan CSV that uses neither new feature must give
  the same calculator numbers and the same generated workbook as today. The verify script's
  existing checks prove this, and you must not edit them (§4.12).
- **Generator version.** Bump `STANDARD_TOURNAMENT_GENERATOR_VERSION` from `'1.4.1'` to
  `'1.5.0'`, and add a history line above it (§4.1). `apps-script/` is not part of the
  Cloud Run service, so the generator change alone bumps neither `sage-tools-api/package.json`
  nor its `README.md` changelog.
- **API version.** The `facilityCompletion.mjs` change (§5) is a Cloud Run change. Write its
  test first and watch it fail. Bump `package.json` from `3.6.2` to `3.7.0`, and add a
  `README.md` Changelog entry that ends with a **Tests:** line. Pushing to `main` deploys it,
  so it follows the 3-day rule too.
- **`lib/v1` changes are compatible only** (root `CLAUDE.md`, "sage-match-control.github.io"):
  add exports and fields, never remove or rename them. A module may only use another module's
  **new export** in a later push than the one that adds it, because a browser can cache a file
  for 10 minutes. §6.2 splits the site change into two pushes for this reason.
- **Docs.** Write in the present tense: describe what the system does, not what changed.
  Every `sage-docs` change carries a version bump and a changelog entry (§8).

### 0.4 Commands

```bash
cd sage-tools-api && node apps-script/verify-standard-tournament-generator.mjs
```

```bash
cd sage-tools-api && npm test
```

```bash
cd sage-tools-api && npm run verify
```

```bash
cd sage-match-control.github.io/_tests && npm install && npm test
```

```bash
cd sage-match-control.github.io/_tests && npm run verify
```

- The first command runs the generator against a mocked Google Sheets. `--only=<scenario>`
  runs one scenario and `--trace` echoes the generator's log.
- `npm run verify` in `sage-tools-api` runs the service tests and all three Apps Script
  verify scripts.
- `npm test` in `_tests` runs the site's unit tests. `npm run verify` there adds the
  Playwright comparison harness, which needs browsers installed (`npx playwright install`).
- The calculator has no automated tests. Serve the site folder with any static server (for
  example `npx --yes http-server sage-match-control.github.io -p 8123`) and open
  `/tools/tournament-calculator.html`.

### 0.5 Done means

Every box in §9 is ticked, every command in §0.4 passes, and §11 is unchanged. The one check
you cannot do yourself is §9 step 2's real-workbook check, which needs Google Sheets. Hand
that to the owner as a list.

---

## 1. Why the work is where it is

| Piece | Today | What the new formats need |
| --- | --- | --- |
| Round robin | single only: a bracket of `n` plays `n(n−1)/2` matches | double: every pairing twice |
| One-bracket playoff | `ttb` or `rr` | `final_bronze` and `final` too |
| Codes `<KEY>_F_1`, `_F_2`, `_B_1`, `_B_2` | already the Final and Bronze codes of multi-bracket playoffs | the same codes: every reader already knows them |
| W, L, points, quotient (`GETTOTALWINS` …) | add up every completed match under a pair's code | already right for a double round robin |
| `CSV` tab scores | `GETSCOREAGAINSTPAIR(team1, team2)`: looked up **by the two codes** | **wrong** for a double round robin, because two meetings share both codes (§4.8) |
| Category tab score grid | one cell per pairing, same lookup | **wrong** for the same reason (§4.7) |
| Site (Hub, Control Center, scorer, board) | identifies a match by its match number; podium, bronze walkover and the "played" rule already handle these codes | nothing to change for items 1–2; tests only (§6) |
| Sync API (`sage-tools-api/src/`) | counts each match on its own | nothing to change for items 1–2 (§5) |
| **Best of 3:** W and L | one per game | **wrong**: a meeting won 2–1 would score 2 W and 1 L. A series count on the category tab and `STANDINGSCSV` (§4.11) |
| **Best of 3:** "matches left", "All matches done", `completedAt` | a game is "not needed" only in a series Final (`<KEY>_F_<seat>_(<game>)`) | **wrong**: game 3 after a 2–0 would stay a match left for ever. A flag column on `CSV` (§4.8) and a second series rule on both sides (§5, §6) |
| **Best of 3:** head-to-head on the site (`rankStandings`) | one per game | **wrong** for the same reason as W and L (§6.1) |
| **Best of 3:** playoff seating | the sheet never sorts a standard playoff's seats (the master has no `SORTBYWINS`): the operator types or draws them from the site's standings | right once `STANDINGSCSV` and head-to-head count meetings (§4.11, §6.1) |

## 2. The model

### 2.1 Round-robin legs

`rr_legs` is `1` (default) or `2`, per category. It is honoured only when the category
resolves to **one bracket of 3 or more pairs**. With `L` legs and a bracket of `n` pairs:

```
round-robin matches  = L × n(n − 1) / 2
round-robin rounds   = L × (n odd ? n : n − 1)      (the minimum number of time slots)
```

The playoff is seeded from the final standings exactly as with a single round robin.

### 2.2 The four one-bracket formats

| `solo_format` | Playoff matches (max / min) | Pairs seated from the round robin | Bronze | Needs |
| --- | --- | --- | --- | --- |
| `ttb` (default) | 2 / 1 | 2 | RR #3, no match | 3+ pairs |
| `rr` | 0 / 0 | 0 | from standings | 3+ pairs |
| **`final_bronze`** | 2 / 2 | 4 | **played**: RR #3 v RR #4 | **4+ pairs** |
| **`final`** | 1 / 1 | 2 | RR #3, no match | 3+ pairs |

### 2.3 Time slots for the playoff

| `solo_format` | Playoff slots |
| --- | --- |
| `ttb` | 2 (the games run one after the other) |
| `rr` | 0 |
| `final_bronze` | `ceil(2 / courts)`. Final and Bronze have four different pairs, so they share a slot on 2+ courts |
| `final` | 1 |

### 2.4 Worked examples (calculator verification targets)

One category. Settings: 2 courts, 20-minute matches, 0-minute buffer, start 08:00.

| Category | Pairs | Legs | Format | RR matches | Playoff | Total | Slots | Finish |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A1 | 4 | 1 | `final_bronze` | 6 | 2 | **8** | 4 | **9:20 AM** |
| A2 | 5 | 1 | `final_bronze` | 10 | 2 | **12** | 6 | **10:00 AM** |
| B1 | 3 | 2 | `final` | 6 | 1 | **7** | 7 | **10:20 AM** |
| B2 | 4 | 2 | `final` | 12 | 1 | **13** | 7 | **10:20 AM** |
| B3 | 4 | 2 | `final_bronze` | 12 | 2 | **14** | 7 | **10:20 AM** |
| B4 | 3 | 2 | `ttb` | 6 | 2 (min 1) | **8** (7–8) | 8 | **10:40 AM** |

The working, using the calculator's own rule (`calcAll`: slots = max(ceil(total / courts),
RR slots + playoff slots), where RR slots = max(ceil(RR matches / courts), round-robin
rounds)):

- A1: RR slots max(3, 3) = 3, playoff 1, slots max(4, 4) = 4, 4 × 20 min = 80 min.
- B1: RR slots max(3, 6) = 6, playoff 1, slots max(4, 7) = 7, 140 min. In a 3-pair bracket
  one pair sits out every round, so the second court is idle.

### 2.5 Best-of-3 meetings

`rr_best_of` is `1` (default) or `3`, per category. It is honoured in any standard category
with a round robin (3 or more pairs, any number of brackets), and only with `rr_legs` 1. With
a bracket of `n` pairs:

```
meetings             = n(n − 1) / 2
round-robin games    = 3 × meetings          (the plan, at full length)
least games          = 2 × meetings          (every meeting won 2–0)
round-robin rounds   = 3 × (n odd ? n : n − 1)
```

- **A meeting's three games run back to back on one court.** A round of the round robin
  therefore takes three time slots, and the plan counts every game 3. Twice-to-beat is
  already planned at its full length in the same way. When a meeting ends 2–0, its game 3
  is not played, the court is free a slot early, and the day ends earlier than planned.
- **Standings.** W is the number of meetings a pair won (2–0 or 2–1) and L the number it lost.
  A tied game counts for neither pair, and a meeting that never reaches two wins counts for
  neither. Points scored, points conceded and the quotient add up every game played, as they
  do today. The site ranks by W, then head-to-head **meetings** won among the tied pairs,
  then quotient. The category tab ranks nothing itself.
- **The playoff** is seeded from those standings exactly as today, and plays exactly as it
  would without best of 3. Each playoff match is one game, except the twice-to-beat Final's
  two games.
- **Codes carry no game.** Each pair keeps its one code `<KEY>_<i>`. The three games of a
  meeting are three rows with the same two codes, and match number puts them in order (§4.9).

### 2.6 Best-of-3 worked examples (calculator verification targets)

The same settings as §2.4: 2 courts, 20-minute matches, 0-minute buffer, start 08:00. Each
row is one category with `rr_best_of` 3.

| Category | Pairs | Brackets | Playoff | Meetings | RR games (least) | Playoff matches | Total (least) | Slots | Finish |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C1 | 4 | 1 | `final_bronze` | 6 | 18 (12) | 2 | **20** (14) | 10 | **11:20 AM** |
| C2 | 4 | 1 | `ttb` | 6 | 18 (12) | 2 (min 1) | **20** (13) | 11 | **11:40 AM** |
| C3 | 8 | 2, top 2 advance | SF, Bronze, Final | 12 | 36 (24) | 4 | **40** (28) | 20 | **2:40 PM** |
| C4 | 3 | 1 | `rr` | 3 | 9 (6) | 0 | **9** (6) | 9 | **11:00 AM** |

The working, by `calcAll`'s rule as in §2.4:

- C1: RR slots max(ceil(18 / 2), 3 × 3) = 9, playoff 1, slots max(10, 10) = 10, 200 min.
- C3: RR slots max(ceil(36 / 2), 3 × 3) = 18. The playoff is the semifinals (1 slot), then
  the Final and Bronze together (1 slot), so 2 slots. Slots max(ceil(40 / 2), 20) = 20, 400 min.
  With single games the same category is 16 matches in 8 slots, finishing at 10:40 AM, as the
  calculator shows today.
- C4: RR slots max(ceil(9 / 2), 3 × 3) = 9. One pair sits out each round, so the second court
  is idle throughout.

---

## 3. Calculator — `sage-match-control.github.io/tools/tournament-calculator.html`

The functions named here exist today. Search for them by name.

### 3.1 State

- `defaultCat()` returns a category object. Add `rrLegs: 1` and `rrBestOf: 1` to it.
- Configs saved before this change have neither (they persist under the `ttc-config` key).
  Read a missing or invalid value as 1 everywhere: `c.rrLegs === 2 ? 2 : 1` and
  `c.rrBestOf === 3 ? 3 : 1`.

### 3.2 `playoffPlan(groups, adv, fill, soloFormat)`

Its `groups === 1` branch handles `'rr'` and otherwise returns the `ttb` plan. Make it four
branches, each returning the plan object with the same fields the existing branches return
(`rounds, total, min, qualifiers, direct, ...nofill, single, rrOnly, bronzeText`). Add one
new field, `solo`, set to the format the plan was built for:

```js
// final_bronze
{ rounds: [{label:'Bronze match',matches:1,byes:0},{label:'Final',matches:1,byes:0}],
  total:2, min:2, qualifiers:4, direct:4, ...nofill, single:true, rrOnly:false, bronzeText:null, solo:'final_bronze' }
// final
{ rounds: [{label:'Final',matches:1,byes:0}],
  total:1, min:1, qualifiers:2, direct:2, ...nofill, single:true, rrOnly:false, bronzeText:'RR #3 automatic', solo:'final' }
```

Add `solo:'rr'` to the existing `rr` return and `solo:'ttb'` to the existing `ttb` return.
Leave every multi-bracket return untouched (no `solo` field).

### 3.3 `calcCategory(c, cfg)`

After `sizes` and `groups` are computed, and before `playoffPlan` is called:

```js
let solo = c.soloFormat || 'ttb';
if(groups===1 && solo==='final_bronze' && sizes[0]<4){
  solo='final';
  notes.push('Final + bronze needs 4 pairs, so RR #3 takes bronze without a match.');
}
const games = (c.rrBestOf===3 && groups>0) ? 3 : 1;
const legs = (c.rrLegs===2 && groups===1 && games===1) ? 2 : 1;
if(c.rrLegs===2 && games===3) notes.push('Best-of-3 meetings are played once per pairing, so this category is not a double round robin.');
else if(c.rrLegs===2 && groups>1) notes.push('A double round robin is for one bracket only, so this category plays a single round robin.');
if(legs===2 && sizes[0]>6) notes.push(`A double round robin of ${sizes[0]} pairs means ${2*(sizes[0]-1)} matches per pair. Consider a single round robin, or more brackets.`);
const biggest = groups>0 ? Math.max(...sizes) : 0;
if(games===3 && biggest>5) notes.push(`Best of 3 in a bracket of ${biggest} means up to ${3*(biggest-1)} games per pair. Consider more brackets.`);
```

- `notes` is declared later in the function today. Move its declaration up so these lines
  can use it.
- Count meetings, then games:
  `const meetings = rrMatches(sizes) * legs;` and `const rr = meetings * games;`. `rr` stays
  the number of round-robin **matches** (rows in `SCHEDULE`), which is what every timing
  function reads.
- `totalMin` becomes `meetings * (games===3 ? 2 : 1) + plan.min`. `total` stays `rr + plan.total`.
- Pass `solo` (not `c.soloFormat`) to `playoffPlan`.
- Add `legs`, `games`, `meetings` and `solo` to the returned object.
- The `teams === 2` best-of-3 early return is unchanged. It has no round robin, so `legs` and
  `games` are ignored there. Give its returned object `legs:1, games:1, meetings:0` so no
  reader meets an `undefined`.
- Only an imported or hand-edited config can carry both `rrLegs: 2` and `rrBestOf: 3`. The UI
  (§3.5) never sets both. Best of 3 then wins, with the first note above.

### 3.4 Timing: `rrRoundFloor(p)` and `seqParts(p, courts)`

- `rrRoundFloor`: multiply the result by `(p.legs || 1) * (p.games || 1)`. In a best-of-3
  round robin, every round's meetings take three slots, because each meeting's games run
  back to back (§2.5). The dual-meet branch returns before this point and is unchanged.
- `seqParts`: replace `if(p.plan.single)return {rr:rrS,po:p.plan.rrOnly?0:2};` with a branch
  on `p.plan.solo`: `'rr'` → 0, `'ttb'` → 2, `'final'` → 1, `'final_bronze'` →
  `Math.ceil(2/courts)`.

`catTime` and `calcAll` need no change.

### 3.5 UI (inside `recalc`)

`recalc` builds each category card's playoff controls as HTML strings: `fillUI`, `soloUI` and
the chips. All buttons use the class `po fillbtn` and a `data-*` attribute. The `#cats` click
handler copies `fb.dataset.solo` into `c.soloFormat`.

1. **`soloUI`** gets four buttons in this order: `data-solo="ttb"` *Twice-to-beat final*,
   `data-solo="final_bronze"` *Final + bronze match*, `data-solo="final"` *Final, #3 bronze*,
   `data-solo="rr"` *Round robin only*.
   - `aria-pressed` is true on the button whose value equals `p.plan.solo` (the format
     actually planned, after §3.3's fallback).
   - When `p.sizes[0] < 4`, the *Final + bronze match* button gets `disabled` and
     `title="Needs 4 pairs: RR #3 v RR #4"`.
2. **New `rrUI`**: one row that chooses how the round robin is played. It is shown for every
   standard category with a round robin (`!p.plan.bestOf3 && p.groups > 0`, and the dual-meet
   branch has already returned), with one bracket or more. Three buttons:
   - `data-rr="single"`: *Single round robin*
   - `data-rr="double"`: *Double round robin*. It gets `disabled` and
     `title="One bracket only"` when `p.groups > 1`.
   - `data-rr="bo3"`: *Best-of-3 meetings*

   `aria-pressed` is true on `bo3` when `p.games === 3`, on `double` when `p.legs === 2`, and
   on `single` otherwise. The flags come from the plan, after §3.3's fallbacks. Build it like
   `soloUI`: a `<div class="playoffs" style="margin-bottom:6px">` holding three
   `<button type="button" class="po fillbtn" data-rr="…" aria-pressed="…">` elements. Put
   `rrUI` first in `planEl.innerHTML`, before `fillUI` and `soloUI`.
3. **Click handler** (`$('cats').addEventListener('click', …)`): add, beside the existing
   `dataset.solo` line,
   `if(fb.dataset.rr){c.rrLegs=fb.dataset.rr==='double'?2:1;c.rrBestOf=fb.dataset.rr==='bo3'?3:1;}`.
   Each button sets both fields, so the two are never on together.
4. **Summary line** (`lineEl.innerHTML`): the `p.plan.single` branch today always describes
   twice-to-beat. Branch on `p.plan.solo`:
   - `ttb`: unchanged, apart from the two substitutions below.
   - `final_bronze`: `<b>${p.total} matches</b> — ${grpTxt} → ${p.rr} round-robin + final and bronze. RR #1 v #2 for gold, #3 v #4 for bronze.`
   - `final`: `<b>${p.total} matches</b> — ${grpTxt} → ${p.rr} round-robin + final. RR #1 v #2 for gold; #3 takes bronze with no match.`
   - `rr` keeps its own existing branch, which comes before `single`.
   - When `p.legs === 2`, every one of these says `double round robin` instead of
     `round-robin`, for example `→ 12 double round robin + final.`
   - When `p.games === 3`, in every branch here **and** in the multi-bracket branch (the last
     one, `… round-robin + ${p.po} playoff …`):
     - the bold count reads `<b>${p.totalMin}–${p.total} matches</b>`
     - `${p.rr} round-robin` reads `${p.meetings} best-of-3 round-robin meetings`
     - a one-bracket line, which ends with a full stop, gets
       ` Each meeting is first to 2 wins; game 3 is played only at 1–1.` added at the end
     - the multi-bracket line gets ` · each meeting first to 2 wins, game 3 only at 1–1`
       added at the end

     For C3 of §2.6 the whole line is `28–40 matches — 2 groups (4-4) → 12 best-of-3
     round-robin meetings + 4 playoff · 4 direct qualifiers (top 2 per group) · each meeting
     first to 2 wins, game 3 only at 1–1` (`2 groups (4-4)` is what `groupShorthand` prints).
5. **`hasTTB`** (the "counted at full length" note) is true today for every `single`
   non-`rrOnly` plan. Make it
   `r.per.some(p=>p.plan&&(p.plan.solo==='ttb'||p.plan.bestOf3||p.games===3))`, keeping the
   `p.plan&&` guard. The note already says "twice-to-beat finals and best-of-3 series", which
   covers a best-of-3 meeting.
6. **`advlab`** stays `Playoffs · single group format` for all four formats.
7. **The per-category table** (`struct`, using `SHORT`) already renders `F 1 · Bz 1` and
   `F 1 · Bz auto` from the rounds. Append ` · 2 legs` when `p.legs === 2`, and
   ` · RR best of 3` when `p.games === 3`. The `rrOnly` text (`RR only · standings decide`)
   takes the same suffix.

### 3.6 Chart: `buildChart(p)`

Its `if(plan.single){…}` block draws twice-to-beat. Branch on `plan.solo`, reusing the
helpers in scope (`col`, `box`, `slot`):

- `ttb`: unchanged.
- `final_bronze`: a `Final` column with
  `box(slot('<span class="sd">RR #1</span>'),slot('<span class="sd">RR #2</span>'),'gold')`,
  and a `Bronze match` column with
  `box(slot('<span class="sd">RR #3</span>'),slot('<span class="sd">RR #4</span>'),'bronzebox')`.
  Legend: `One game each. The winner of the Final takes gold, the loser silver; the winner of
  the Bronze match takes bronze.`
- `final`: the same `Final` column, plus the existing twice-to-beat bronze column
  (`RR #3 bronze — automatic, no match`). Legend: `One game. The winner takes gold, the loser
  silver.`

### 3.7 CSV export: `buildPlanCsv(cfg, r)`

- Header: append `,rr_legs`. The header becomes
  `type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs`.
- `soloFormatCell`: write `p.plan.solo` (the effective format) instead of `c.soloFormat`,
  under the same condition as today: standard format, `p.plan.single`, and not
  `p.plan.bestOf3` (the 2-pair best-of-3 Final, not best-of-3 meetings).
- New 13th cell: `'2'` when `!isDual && p && p.legs===2`, else `''`. A single round robin
  writes it empty, so an unchanged plan differs from today's export only in the header line.
- Header: append `,rr_best_of` after `rr_legs`. The full header is
  `type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs,rr_best_of`.
- New 14th cell: `'3'` when `!isDual && p && p.games===3`, else `''`.
- `buildPlanCsv` also feeds the generator handoff (the clipboard copy), so nothing else
  changes for that.

### 3.8 CSV import: `importCSVText(text)`

- `soloFormat`: accept `'ttb'`, `'rr'`, `'final_bronze'` and `'final'`. Anything else is
  `'ttb'`.
- `rrLegs`: `(row[12]||'').trim()==='2' ? 2 : 1`. Add it to the pushed category object.
- `rrBestOf`: `(row[13]||'').trim()==='3' ? 3 : 1`. Add it too.
- An old 12-column file imports exactly as before, and so does a 13-column one.

### 3.9 `.xlsx` export: `exportXLSX()`

The Format cell (`fmt`) for a standard category with rounds: append ` (#3 bronze auto)` when
`p.plan.bronzeText` is set, ` · 2 legs` when `p.legs === 2`, and ` · RR best of 3` when
`p.games === 3` (for an `rr` category too). The existing `SHORT` map already renders
`Final 1, Bronze 1`. No new columns. The RR matches column holds `p.rr`, the games at full
length, which agrees with the timing columns beside it.

### 3.10 Do not touch

The dual-meet code and its CSV columns, `GENERATOR_MASTERS`, `tools/sw.js`, and
`tools/tournament-calculator-beta.html` (an untracked scratch copy, ignored by `**/*beta*`).

---

## 4. Generator — `sage-tools-api/apps-script/standard-tournament-generator.gs`

### 4.1 Version

```
//   1.5  double round robin (rr_legs), best-of-3 round-robin meetings (rr_best_of) and the
//        one-bracket final and final_bronze formats
var STANDARD_TOURNAMENT_GENERATOR_VERSION = '1.5.0';
```

### 4.2 Input: `parsePlanCsv(text)`

Add `rrLegs: (row[12] || '').trim(),` and `rrBestOf: (row[13] || '').trim(),` beside
`soloFormat: (row[11] || '').trim(),`.

A note on names: `p.shape === 'bo3'` already exists, and it means a **2-pair category's
best-of-3 Final**. The new setting is `rrBestOf` everywhere in code, and nothing below touches
the `'bo3'` shape.

### 4.3 Validation: `validatePlan(parsed, options)`

Beside the existing `solo_format` check:

```js
var SOLO_FORMATS_ = ['', 'ttb', 'rr', 'final_bronze', 'final'];   // top level, with the other constants
...
if (SOLO_FORMATS_.indexOf(cat.soloFormat) < 0) {
  errors.push(label + ': solo_format must be empty, "ttb", "rr", "final_bronze" or "final" (found "' + cat.soloFormat + '").');
}
if (cat.rrLegs !== '' && cat.rrLegs !== '1' && cat.rrLegs !== '2') {
  errors.push(label + ': rr_legs must be empty, 1 or 2 (found "' + cat.rrLegs + '").');
}
if (cat.rrBestOf !== '' && cat.rrBestOf !== '1' && cat.rrBestOf !== '3') {
  errors.push(label + ': rr_best_of must be empty, 1 or 3 (found "' + cat.rrBestOf + '").');
}
```

This replaces the existing `solo_format` check. The checks that need the bracket split go in
`planCategory` (§4.4), whose `p.errors` `validatePlan` already collects.

### 4.4 Plan: `playoffPlan_` and `planCategory(cat)`

**`playoffPlan_(groups, adv, fill, soloFormat)`**, in its `groups === 1` branch:

```js
if (soloFormat === 'rr') return { shape: 'rr', rounds: [], qualifiers: 0, wc: 0, bronze: false, tiered: false };
if (soloFormat === 'final_bronze') return { shape: 'finalBronze', rounds: [{ matches: 1 }], qualifiers: 4, wc: 0, bronze: true, tiered: false };
if (soloFormat === 'final') return { shape: 'final', rounds: [{ matches: 1 }], qualifiers: 2, wc: 0, bronze: false, tiered: false };
return { shape: 'ttb', rounds: [{ matches: 2 }], qualifiers: 2, wc: 0, bronze: false, tiered: false };
```

**`planCategory(cat)`:**

1. Add `rrLegs: cat.rrLegs === '2' ? 2 : 1` and `rrBestOf: cat.rrBestOf === '3' ? 3 : 1` to
   the initial `p` object.
2. After the `if (teams === 2) … else …` block, add these errors:
   - `p.rrBestOf === 3 && p.shape === 'bo3'` → `'category "' + key + '": rr_best_of 3 needs a round robin, and a 2-pair category has none (it already plays a best of 3).'`
   - `p.rrBestOf === 3 && p.rrLegs === 2` → `'category "' + key + '": rr_best_of 3 and rr_legs 2 cannot be combined; choose a double round robin or best-of-3 meetings.'`
   - `p.rrLegs === 2 && p.shape === 'bo3'` → `'category "' + key + '": rr_legs 2 needs a round robin, and a 2-pair category has none (it plays a best of 3).'`
   - `p.rrLegs === 2 && p.groups > 1` → `'category "' + key + '": a double round robin (rr_legs 2) is for one bracket only, and this category has ' + p.groups + ' brackets.'`
   - `p.shape === 'finalBronze' && p.sizes[0] < 4` → `'category "' + key + '": solo_format "final_bronze" needs 4 pairs for its Bronze match (RR #3 v RR #4); this bracket has ' + p.sizes[0] + '.'`
3. **Leg 2.** Where `p.rrByBracket` is built, when `p.rrLegs === 2 && p.groups === 1`, append
   leg 2 to the bracket's rounds: the same rounds, in the same order, with each match's
   `a` and `b` swapped and `leg: 2` added. Give leg-1 matches `leg: 1`.

   ```js
   var leg1 = roundRobinRounds_(…).map(…);            // as today, with leg: 1 on each match
   return p.rrLegs === 2 && p.groups === 1
     ? leg1.concat(leg1.map(function (round) {
         return round.map(function (m) { return { a: m.b, b: m.a, kind: 'rr', bracket: m.bracket, leg: 2 }; });
       }))
     : leg1;
   ```

3a. **Best-of-3 games.** Right after the loop that builds `p.rrRows` (it lines up each round
   across the brackets) and **before** `p.rrCount` is computed, when `p.rrBestOf === 3`,
   replace each round-robin row with three copies of it, one per game, with `game` 1, 2, 3 on
   each match:

   ```js
   if (p.rrBestOf === 3) {
     p.rrRows = p.rrRows.reduce(function (rows, row) {
       return rows.concat([1, 2, 3].map(function (g) {
         return row.map(function (m) { return { a: m.a, b: m.b, kind: 'rr', bracket: m.bracket, game: g }; });
       }));
     }, []);
   }
   ```

   - Each meeting's three games sit in three consecutive `MATCHES` rows, in the same block
     column. Packed that way, they run back to back on one court, as §2.5 plans.
   - `p.rrCount` is then the number of games (3 × meetings), and `p.totalMatches` and
     `MATCHES!B2` count every game 3, just as they count the `ttb` walkover.
   - Every row keeps its bracket's matches under its bracket's headers, so `p.blockLabels`
     needs no change.
   - The sides are **not** swapped between games.
   - `packSchedule_` (which sizes `SCHEDULE` and logs a suggested packing) needs no change.
     It never puts a pair in two matches of one slot, and it never reorders a queue, so each
     meeting's three games land in three consecutive slots, and `SCHEDULE` is sized for every
     game 3.

4. **`MATCHES` row labels.** The line
   `p.matchesRows = p.rrRows.map(function (row, idx) { return { label: 'RR ' + (idx + 1), … } })`
   labels leg 2's rows `RR 1 · Leg 2` … `RR n · Leg 2` instead, where `n` is the number of
   leg-1 rounds (`p.rrRows.length / 2` when `p.rrLegs === 2`). Leg 1 stays `RR 1` … `RR n`.
   When `p.rrBestOf === 3`, row `idx` is labelled
   `'RR ' + (Math.floor(idx / 3) + 1) + ' · G' + (idx % 3 + 1)`: `RR 1 · G1`, `RR 1 · G2`,
   `RR 1 · G3`, `RR 2 · G1`, and so on.
5. **The two new shapes.** Add branches beside `else if (p.shape === 'ttb') {…}`. Both reuse
   existing block **kinds**, so `writePlayoffBlock_`, `seatCodeFormula_`, `nameLookupFormula_`
   and `advanceFormula_` need no change:

   ```js
   } else if (p.shape === 'finalBronze') {
     // Bronze before Final, as the multi-bracket ladder lays them out (master spec §7.5).
     p.finals = [
       { label: 'B', kind: 'bronze', seats: [1, 2].map(function (s) { return { kw: 'B', number: s, code: key + '_B_' + s }; }) },
       { label: 'F', kind: 'final',  seats: [1, 2].map(function (s) { return { kw: 'F', number: s, code: key + '_F_' + s }; }) },
     ];
     p.qualifierRows = [
       { tier: 'RR #1 - final', ap: key + '_F_1' }, { tier: 'RR #2 - final', ap: key + '_F_2' },
       { tier: 'RR #3 - bronze', ap: key + '_B_1' }, { tier: 'RR #4 - bronze', ap: key + '_B_2' },
     ];
   } else if (p.shape === 'final') {
     // Final, then the bronze walkover, as twice-to-beat lays them out (master spec §7.5.1).
     p.finals = [
       { label: 'F', kind: 'final', seats: [1, 2].map(function (s) { return { kw: 'F', number: s, code: key + '_F_' + s }; }) },
       { label: 'B', kind: 'ttbBronze', seats: [1, 2].map(function (s) { return { kw: 'B', number: s, code: key + '_B_' + s, bye: s === 2 }; }) },
     ];
     p.qualifierRows = [
       { tier: 'RR #1 - final', ap: key + '_F_1' }, { tier: 'RR #2 - final', ap: key + '_F_2' },
       { tier: 'RR #3 - bronze', ap: key + '_B_1' },
     ];
   }
   ```

   What the reused kinds produce:
   - `final` gives seat code `=A$1&"_"&A<label row>&"_1"`, which is `<KEY>_F_1`. Its name
     lookup is the fresh-seat form `=IFNA(FILTER(AO:AO,AP:AP=A<r>),A<r>)`. Column `K` gets
     `GOLD`/`SILVER`.
   - `bronze` gives `<KEY>_B_n`, the same fresh lookup, and `K` `=IF(D<r>>0,"BRONZE","-")`.
   - `ttbBronze` writes `BYE` on the bye seat, and `K` `BRONZE` / `-`.
6. **`MATCHES` playoff rows.** Beside `if (p.shape === 'ttb') { p.matchesRows.push(…) }`:
   - `finalBronze`: one row
     `{ label: 'F + B', matches: [matchOf_(p.finals[1], 'po'), matchOf_(p.finals[0], 'po')], stageOrder: 1, unit: true }`,
     with the Final first.
   - `final`: one row
     `{ label: 'F + B', matches: [matchOf_(p.finals[0], 'po'), matchOf_(p.finals[1], 'po')], stageOrder: 1, unit: true }`.

`p.totalMatches`, `p.playoffCount` and `p.band` are derived from `p.matchesRows` and need no
change. `MATCHES!B2` (the total) therefore counts the `final` walkover as one match, as it
already does for `ttb`.

### 4.5 Category tab layout: `computeTabLayout_(p)`

Its condition `if (p.shape === 'ladder' || p.shape === 'ttb' || p.shape === 'bo3')` decides
whether a finals section is laid out. Add `'finalBronze'` and `'final'`. The existing
`p.finals.forEach(function (f) { placeBlock(f, f.kind); })` then places the blocks.

The rows that result, which §4.12 checks:

| Category | Group pairs | Gap | `FINALS` banner | First block label row (seats) | Second block label row (seats) |
| --- | --- | --- | --- | --- | --- |
| Format A, 5 pairs (`finalBronze`) | 6–15 | 16 | 17 | `B` at 19 (20, 22) | `F` at 25 (26, 28) |
| Format B, 3 pairs (`final`) | 6–11 | 12 | 13 | `F` at 15 (16, 18) | `B` at 21 (22, 24) |

### 4.6 Qualifier scaffold

`writeQualifierScaffold_` needs no change. Rows with `ap` set are written as literal codes
in `AP`, and an empty `p.freshSeats` means no slot table, exactly as for `ttb`. `AM` stays
`-`. The operator types each pair's names into `AO` once the round robin is over.

**The operator seats from the Hub's or Control Center's standings, not the category tab's.**
The tab ranks nothing itself: the Standard Tournament Master has no `SORTBYWINS`, because a
standard playoff's seats are typed or drawn. The site breaks a tie on wins by head-to-head first
(`rankStandings` in `lib/v1/domain/standings.js`), and in a double round robin that counts
both meetings. §8 puts this in the dry-run checklist.

### 4.7 The score grid in a double or best-of-3 round robin: `writeGroupGrid_`

`writeGroupGrid_(sheet, templateSheet, first, nb)` is called from `buildCategoryTab`, where
`p` is in scope. Add a fifth parameter `byMatch` and pass
`p.rrLegs === 2 || p.rrBestOf === 3`. Each off-diagonal body cell is today
`'=GETSCOREAGAINSTPAIR($N' + r + ',' + colLetter_(COL.O + c) + '$' + first + ')'`. When
`byMatch` is true, write this instead, with the same `$N<r>` and `<col>$<first>` references:

```js
function legGridFormula_(r, col, first) {
  var me = '$N' + r, them = col + '$' + first;
  return '=LET(t_one,GETPLAYERSCOLUMN1(),t_two,GETPLAYERSCOLUMN2(),' +
    's_one,GETSCORESCOLUMN1(),s_two,GETSCORESCOLUMN2(),nums,GETMATCHNUMBERS(),' +
    'mine,ARRAYFORMULA(IF(t_one=' + me + ',s_one,s_two)),' +
    'hit,ARRAYFORMULA((t_one=' + me + ')*(t_two=' + them + ')+(t_two=' + me + ')*(t_one=' + them + ')),' +
    'IFERROR(TEXTJOIN(" · ",TRUE,SORT(FILTER(mine,hit>0),FILTER(nums,hit>0),TRUE)),""))';
}
```

For the cell at row 7, column P of a grid anchored at row 6, that is:

```
=LET(t_one,GETPLAYERSCOLUMN1(),t_two,GETPLAYERSCOLUMN2(),s_one,GETSCORESCOLUMN1(),s_two,GETSCORESCOLUMN2(),nums,GETMATCHNUMBERS(),mine,ARRAYFORMULA(IF(t_one=$N7,s_one,s_two)),hit,ARRAYFORMULA((t_one=$N7)*(t_two=P$6)+(t_two=$N7)*(t_one=P$6)),IFERROR(TEXTJOIN(" · ",TRUE,SORT(FILTER(mine,hit>0),FILTER(nums,hit>0),TRUE)),""))
```

- **What it shows:** the row pair's score in each meeting, in match-number order, joined
  by ` · ` (for example `11 · 7`), and blank until one is played. In a best-of-3 round robin
  it shows each game of the meeting the same way, for example `11 · 6 · 11`.
- **Why it is written inline:** the alternative is a new Named Function. Apps Script cannot
  create one, so it would have to be added to the master workbook by hand, and older
  workbooks would not have it.
- **The building blocks:** `GETPLAYERSCOLUMN1/2`, `GETSCORESCOLUMN1/2` and `GETMATCHNUMBERS`
  are existing Named Functions (`sage-docs/docs/technical/named-function-library.md`).
  `LET` names may not look like cell references, which is why they are `t_one` and not `t1`.
- **What reads it:** nothing. The grid is display only.
- **A single round robin of one-game meetings** keeps `GETSCOREAGAINSTPAIR`.

### 4.8 The `CSV` tab: scores by match number — `buildCsvTab_`

The site reads every match from the `CSV` tab. Its score columns are
`'=GETSCOREAGAINSTPAIR($E2,$I2)'` (`H`, team 1) and `'=GETSCOREAGAINSTPAIR($I2,$E2)'` (`L`,
team 2), which identify a match by its two codes. In a double round robin both meetings
would publish the same score, and the second meeting would look played as soon as the first
was. A best-of-3 meeting's three games have the same problem.

`buildCsvTab_(ss)` is called from `generateEventTabs`, where `plans` is in scope. Change the
call to

```js
var byMatch = plans.some(function (p) { return p.rrLegs === 2 || p.rrBestOf === 3; });
var bestOfKeys = plans.filter(function (p) { return p.rrBestOf === 3; }).map(function (p) { return p.key; });
buildCsvTab_(ss, byMatch, bestOfKeys);
```

`plans` there holds the selected categories only, so `bestOfKeys` names only the categories
this workbook builds.

When the second argument (`byMatch`) is true, write `H2` and `L2` as:

```
H2  =IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN1(),GETMATCHNUMBERS()=$A2),1),""))
L2  =IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN2(),GETMATCHNUMBERS()=$A2),1),""))
```

- Why it is right: `E2` is `GETTEAM1CODEBYMATCH($A2)`, the team-1 code at the match's
  position in the same stacked columns, so team 1's score is `GETSCORESCOLUMN1` at that
  position.
- Otherwise, write today's formulas unchanged.
- `Fill match numbers` (an existing menu item) copies row 2 down, so nothing else changes.

**The `bestOf` column (best of 3 only).** When the third argument (`bestOfKeys`, the keys of
the categories with `rr_best_of` 3, in plan order) is not empty, the tab gets a 13th column:

- `ensureColumnCount_(sheet, 13)` first. The master's `CSV` tab has 12 columns, and the mock
  refuses a write past the sheet's edge, as Sheets does.
- `M1` is the header `bestOf`. `CSV_HEADERS_` itself stays 12 long, so a workbook without
  best of 3 is unchanged.
- `M2` is `=IF(REGEXMATCH($E2,"^(<keys>)_\d+$"),3,"")`, where `<keys>` is `bestOfKeys` joined
  by `|`. For `GAMD` and `GBMD` that is
  `=IF(REGEXMATCH($E2,"^(GAMD|GBMD)_\d+$"),3,"")`. A key is `[A-Z0-9]{2,8}` (`validatePlan`),
  so it needs no regex escaping. In the Apps Script string the backslash is written `\\d`.
- The cell reads `3` on a round-robin game of a best-of-3 category and is blank everywhere
  else, playoff matches included: a playoff seat code such as `GAMD_F_1` does not match
  `_\d+$` after the key.
- `Fill match numbers` copies `M2` down with the rest of row 2, with no change to
  `sheets-sync.gs`: `fillCsvMatchNumbers_` copies row 2 across `sheet.getLastColumn()`, so it
  takes every column the tab has.

The site and the sync API read `bestOf` by its header name (§5, §6), so its position does not
matter to them. A `CSV` tab without the column reads as no best of 3 anywhere.

### 4.9 Codes carry no leg and no game

A double round robin and a best-of-3 meeting add no code. Each pair keeps its one code,
`<KEY>_<i>`, in both legs and in all three games. A round-robin match is still its two pairs'
codes, and only the match number tells two meetings, or two games, apart. Do not add a leg or
game suffix: `GETTOTALWINS` and the other totals add up every match under one code, and
several readers depend on one code per pair:

- `STANDINGSCSV!G`, which matches only `<KEY>_<number>`
- the roster scaffold and `Reference for Players`
- the bracket draw import
- the site's pair index

### 4.10 `SAGE → Add bracket links`: `addBracketLinks_(ss)`

This re-plans each generated category from the `Variables` plan block, which does not
record `solo_format`. Today it passes
`soloFormat: known[c.key + '_F_1_(1)'] ? 'ttb' : 'rr'`. Replace that with:

| Condition (in order) | `soloFormat` |
| --- | --- |
| `known[c.key + '_F_1_(1)']` | `'ttb'` |
| `known[c.key + '_F_1']`, and the category tab's `AP5:AP12` holds the literal `c.key + '_B_2'` | `'final_bronze'` |
| `known[c.key + '_F_1']` | `'final'` |
| otherwise | `'rr'` |

Read `AP` from the tab named `c.key` with `getRange('AP5:AP12').getValues()`. A multi-bracket
category also has `_F_1`, but `playoffPlan_` ignores `soloFormat` when there are 2+ brackets,
so its result is unaffected. `rr_legs` and `rr_best_of` do not change any link row, so
neither is inferred.

`playoffLinkRows_` needs no change. The new seats come out with `match` `F` or `B`, `from`
empty and `seat` empty, like `ttb`'s.

### 4.11 Series standings in a best-of-3 round robin

W and L count **meetings** (§2.5). Today both the category tab (`D`, `E`, written by
`writeGroupPairs_`) and `STANDINGSCSV` (`D`, `E`, written by `buildStandingsCsvTab_`) call
`GETTOTALWINS` / `GETTOTALLOSES`, which count games. Add one helper beside `legGridFormula_`:

```js
/** §4.11 — the meetings `me` won ('won') or lost ('lost'): the rivals it took, or gave up, 2 games to. No leading "=". */
function seriesCountFormula_(me, side) {
  var op = side === 'won' ? '>' : '<';
  return 'LET(t_one,GETPLAYERSCOLUMN1(),t_two,GETPLAYERSCOLUMN2(),' +
    's_one,GETSCORESCOLUMN1(),s_two,GETSCORESCOLUMN2(),' +
    'played,ARRAYFORMULA(ISNUMBER(s_one)*ISNUMBER(s_two)),' +
    'rival,ARRAYFORMULA(IF(t_one=' + me + ',t_two,IF(t_two=' + me + ',t_one,""))),' +
    'took,ARRAYFORMULA(played*((t_one=' + me + ')*(s_one' + op + 's_two)+(t_two=' + me + ')*(s_two' + op + 's_one))),' +
    'IFERROR(SUM(MAP(UNIQUE(FILTER(rival,rival<>"")),LAMBDA(o,IF(SUMPRODUCT((rival=o)*took)>=2,1,0)))),0))';
}
```

- **How it counts.** `rival` is the other pair in every row where `me` plays. `took` is 1 on
  each played game `me` won (for `'won'`) or lost (for `'lost'`). For each distinct rival,
  2 or more such games is one meeting. `SUMPRODUCT` is used, not `SUMIF`, because it takes
  the arrays `LET` builds as well as ranges. A tied game is neither `>` nor `<`, so it counts for
  neither pair, the same rule as the site's series walk (`walkSeries`). `played` is the same
  "both scores are numbers" rule as `GETMATCHRESULTSBYTEAMCODE`.
- **Category tab.** In `writeGroupPairs_`, when `p.rrBestOf === 3`, write
  `'=' + seriesCountFormula_('A' + r, 'won')` in `D` and `'=' + seriesCountFormula_('A' + r, 'lost')`
  in `E`. `F`, `G` and `H` (points scored, points conceded, quotient) keep their formulas and
  add up every game. `J` (`=D+E`, "Matches Done") then counts meetings decided. For `A6` the
  `D6` formula is:

  ```
  =LET(t_one,GETPLAYERSCOLUMN1(),t_two,GETPLAYERSCOLUMN2(),s_one,GETSCORESCOLUMN1(),s_two,GETSCORESCOLUMN2(),played,ARRAYFORMULA(ISNUMBER(s_one)*ISNUMBER(s_two)),rival,ARRAYFORMULA(IF(t_one=A6,t_two,IF(t_two=A6,t_one,""))),took,ARRAYFORMULA(played*((t_one=A6)*(s_one>s_two)+(t_two=A6)*(s_two>s_one))),IFERROR(SUM(MAP(UNIQUE(FILTER(rival,rival<>"")),LAMBDA(o,IF(SUMPRODUCT((rival=o)*took)>=2,1,0)))),0))
  ```

- **Playoff seating.** The sheet sorts no seats: the Standard Tournament Master has no
  `SORTBYWINS`, and every standard playoff seat is typed in or drawn by the operator, from
  the site's standings (§4.6). Those standings read W and L from `STANDINGSCSV`, below.
- **`STANDINGSCSV`.** Give `buildStandingsCsvTab_` a third parameter, `bestOfKeys`, and change
  its call in `generateEventTabs` to `buildStandingsCsvTab_(ss, codeCount, bestOfKeys)` (§4.8's
  variable). When it is not empty, write `D` and `E` for row `r` as
  `'=IF(REGEXMATCH($A' + r + ',"^(' + keys + ')_\\d+$"),' + seriesCountFormula_('$A' + r, 'won') + ',GETTOTALWINS($A' + r + '))'`
  and the same with `'lost'` and `GETTOTALLOSES`. A code from any other category, and every
  playoff seat code, keeps today's count. When `bestOfKeys` is empty, write today's formulas.
- **Cost.** Each formula reads the four stacked columns once, like `GETMATCHRESULTSBYTEAMCODE`,
  and only the pairs of a best-of-3 category carry it.
- **Why inline.** The same reason as §4.7: Apps Script cannot add a Named Function.
- **Three rows per pairing.** A pairing is expected to have exactly three rows in `SCHEDULE`.
  A fourth row (a packing mistake) would count towards the formula's two wins, while the site
  reads only the first three (§6.1). The dry-run checklist covers this (§8).

### 4.12 Verify script: `apps-script/verify-standard-tournament-generator.mjs`

**Do not edit any existing check.** Add a scenario block `if (!only || only === "finals") { … }`
before the `reject` block. Build the CSV inline, like the `bkl` block does, with the 13-column
header:

```
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs
setting,title,Finals test
setting,date,2026-12-01
setting,start,08:00
setting,courts,4
setting,duration_min,20
setting,buffer_min,0
setting,format,standard
category,FAMD,Format A,5,1,2,bye,,,,,final_bronze,
category,FBMD,Format B,3,1,2,bye,,,,,final,2
category,FCMD,Bronze after double,4,1,2,bye,,,,,final_bronze,2
category,FDMD,Twice-to-beat after double,3,1,2,bye,,,,,ttb,2
```

Use the script's existing helpers:

- `run(name, csv, options)` generates a workbook in a fresh copy of the mocked master and
  returns `{ ss, sandbox, S, cell, result }`. On a throw or a rejection it counts a failure
  and returns `null`, so guard with `if (out) { … }`. Pass
  `{ facility: "Main", courtLabels: "1,2,3,4", selected: [ …keys ] }`.
- `cell(sheet, row, col)` returns a cell's formula text (with its `=`) or its value, and the
  string `"(empty)"` for an empty cell. "Is empty" below means `"(empty)"`. Columns are
  1-based: A = 1, K = 11, M = 13, P = 16, AD = 30, AN = 40, AP = 42.
- `plansOf(sandbox, csv, keys)` returns `planCategory`'s result for each listed key.
- `check(label, actual, expected)` records one check. Compare arrays by joining them into a
  string, as the existing checks do.
- For a rejection, do as the `reject` block does: `const sb = load(buildMasterFixture());`, then
  `sb.generateEventTabs(csv, opts)`, and look for the message in the result's `errors`. A
  rejected run returns `errors`; it does not throw.

Then check:

**Format A (`FAMD`, 5 pairs, single round robin, `final_bronze`)**
- plan: `rrCount + playoffCount` is `10 + 2`, `shape` is `finalBronze`.
- tab: `A17` is `FINALS`, `A19` is `B`, `A25` is `F`, `A20` is `=A$1&"_"&A19&"_1"`,
  `A26` is `=A$1&"_"&A25&"_1"`, `B20` is `=IFNA(FILTER(AO:AO,AP:AP=A20),A20)`, `K20` is
  `=IF(D20>0,"BRONZE","-")`, `K26` is `=IFNA(IFS(D26>0,"GOLD",E26>0,"SILVER"),"-")`.
- scaffold: `AP5`, `AP7`, `AP9`, `AP11` are `FAMD_F_1`, `FAMD_F_2`, `FAMD_B_1`, `FAMD_B_2`.
  `AN6` is `RR #1 - final`. `AM5` is `-`.
- grid: `P7` is still `=GETSCOREAGAINSTPAIR($N7,P$6)`.
- link rows (`playoffLinkRows_`): `FAMD_B_1|B||`, `FAMD_B_2|B||`, `FAMD_F_1|F||`, `FAMD_F_2|F||`.

**Format B (`FBMD`, 3 pairs, double, `final`)**
- plan: `rrCount + playoffCount` is `6 + 2` (the Final plus the walkover), and
  `matchesRows` labels are `RR 1, RR 2, RR 3, RR 1 · Leg 2, RR 2 · Leg 2, RR 3 · Leg 2, F + B`.
- leg 2 swaps sides: each leg-2 row's match has `a` and `b` of the same-numbered leg-1 row,
  reversed.
- tab: `A13` is `FINALS`, `A15` is `F`, `A21` is `B`, `B24` is `BYE`, `K22` is `BRONZE`,
  `K24` is `-`.
- scaffold: `AP5`, `AP7`, `AP9` are `FBMD_F_1`, `FBMD_F_2`, `FBMD_B_1`. `AP11` is empty.
- grid: `P7` is exactly the formula printed in §4.7.

**`FCMD`** (4 pairs, double, `final_bronze`): `rrCount + playoffCount` is `12 + 2`.
**`FDMD`** (3 pairs, double, `ttb`): `rrCount + playoffCount` is `6 + 3`, and its blocks
are `F (1)`, `F (2)` and `B`, as 1.4.1 builds them.

**Whole workbook** (a run with all four selected): `MATCHES!B2` is `12 + 8 + 14 + 9 = 43`.
`CSV!H2` is
`=IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN1(),GETMATCHNUMBERS()=$A2),1),""))`.
`CSV!M1` is empty (no best of 3 here), and `STANDINGSCSV!D2` is still `=GETTOTALWINS($A2)`.

**Single-round-robin workbook** (a second run selecting only `FAMD`): `CSV!H2` is
`=GETSCOREAGAINSTPAIR($E2,$I2)`.

**Add bracket links**, on the all-four workbook:

- First write every code of the four plans into `STANDINGSCSV` column A as plain values, from
  row 2, as the `links` block's `as12` does: for each code `c` at index `i` of
  `plansOf(sandbox, csv).flatMap((p) => p.codes)`, set cell `(i + 2, 1)` of `S("STANDINGSCSV")`
  to `{ value: c }`. The mock evaluates no formula, so `A2`'s `UNIQUE` spill is empty there,
  and `addBracketLinks_` would otherwise find no codes.
- Then `sandbox.addBracketLinks_(ss).ok` is `true`. With today's re-planning (`'rr'` for any
  category without `_F_1_(1)`), `FAMD` and `FBMD` would report extra codes and `ok` would be
  `false`, so this check fails before §4.10 is built.
- `Variables` column AD (30), read from row 2 down, lists `FAMD_B_1, FAMD_B_2, FAMD_F_1, FAMD_F_2`
  in that order, and `FBMD_F_1, FBMD_F_2, FBMD_B_1, FBMD_B_2` in that order. The order shows
  that `FAMD` was re-planned as `final_bronze` and `FBMD` as `final`: the two give the same
  codes, in a different order.

**Rejections.** Each CSV is the scenario's header and `setting` rows plus one category row,
with that key selected. The error that comes back contains the text given.

| Category row | Error contains |
| --- | --- |
| `category,SMALLMD,Small,3,1,2,bye,,,,,final_bronze,` | `"final_bronze" needs 4 pairs` |
| `category,LEGSMD,Legs,5,1,2,bye,,,,,,3` | `rr_legs must be empty, 1 or 2` |
| `category,TWOMD,Two,2,1,2,bye,,,,,,2` | `rr_legs 2 needs a round robin` |
| `category,WIDEMD,Wide,8,2,2,bye,,,,,,2` | `is for one bracket only` |
| `category,SEMIMD,Semis,5,1,2,bye,,,,,semis,` | `solo_format must be empty, "ttb", "rr", "final_bronze" or "final"` |

**A second new scenario, `bo3`**, in its own block `if (!only || only === "bo3") { … }` right
after `finals`, with the 14-column header:

```
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs,rr_best_of
setting,title,Best of 3 test
setting,date,2026-12-01
setting,start,08:00
setting,courts,4
setting,duration_min,20
setting,buffer_min,0
setting,format,standard
category,GAMD,Bo3 final and bronze,4,1,2,bye,,,,,final_bronze,,3
category,GBMD,Bo3 two brackets,8,2,2,bye,,,,,,,3
category,GCMD,One-game meetings,4,1,2,bye,,,,,ttb,,
```

Check:

**`GAMD`** (4 pairs, one bracket, best of 3, `final_bronze`)
- plan: `rrCount + playoffCount` is `18 + 2`, `shape` is `finalBronze`, `rrBestOf` is 3.
- `matchesRows` labels start `RR 1 · G1, RR 1 · G2, RR 1 · G3, RR 2 · G1` and the round-robin
  rows end at `RR 3 · G3`, followed by `F + B`.
- The three rows of each meeting hold the same matches (`a` and `b` equal, in the same
  order), with `game` 1, 2 and 3.
- tab: `D6` is exactly the formula printed in §4.11. `E6` is the same with `<` in place of
  `>` in both comparisons. `F6` is still `=GETTOTALSCORE(A6)`, `H6` still
  `=GETSCOREQUOTIENT(A6)`.
- grid: `P7` is the §4.7 formula, the same as Format B's.

**`GBMD`** (8 pairs, 2 brackets of 4, top 2, best of 3)
- plan: `shape` is `ladder`, `rrCount + playoffCount` is `36 + 4`, and `matchesRows` has
  9 round-robin rows (3 rounds × 3 games), each holding 4 matches, 2 from each bracket.
- tab: `D6` and `D14` (bracket 2's first pair, pair 5) carry the series formula for `A6` and
  `A14`.

**`GCMD`** (4 pairs, one-game meetings, `ttb`): `rrCount + playoffCount` is `6 + 3`, and
`D6` is still `=GETTOTALWINS(A6)`.

**Whole workbook** (all three selected):
- `MATCHES!B2` is `20 + 40 + 9 = 69`.
- `CSV!H2` reads scores by match number (the §4.8 formula).
- `CSV!M1` is `bestOf`, and `CSV!M2` is `=IF(REGEXMATCH($E2,"^(GAMD|GBMD)_\d+$"),3,"")`.
- `STANDINGSCSV!D2` is exactly the text below. Write it out in full in the test (in a JS
  string, `\d` is written `\\d`) rather than building it with the generator's helper, so
  that a change to the helper fails the check:

  ```
  =IF(REGEXMATCH($A2,"^(GAMD|GBMD)_\d+$"),LET(t_one,GETPLAYERSCOLUMN1(),t_two,GETPLAYERSCOLUMN2(),s_one,GETSCORESCOLUMN1(),s_two,GETSCORESCOLUMN2(),played,ARRAYFORMULA(ISNUMBER(s_one)*ISNUMBER(s_two)),rival,ARRAYFORMULA(IF(t_one=$A2,t_two,IF(t_two=$A2,t_one,""))),took,ARRAYFORMULA(played*((t_one=$A2)*(s_one>s_two)+(t_two=$A2)*(s_two>s_one))),IFERROR(SUM(MAP(UNIQUE(FILTER(rival,rival<>"")),LAMBDA(o,IF(SUMPRODUCT((rival=o)*took)>=2,1,0)))),0)),GETTOTALWINS($A2))
  ```

- `STANDINGSCSV!E2` is the same with `<` in place of `>` in both comparisons, and
  `GETTOTALLOSES($A2)` at the end.

**Rejections**, built the same way as the `finals` ones, with the 14-column header:

| Category row | Error contains |
| --- | --- |
| `category,BADMD,Bad,5,1,2,bye,,,,,,,2` | `rr_best_of must be empty, 1 or 3` |
| `category,TWOMD,Two,2,1,2,bye,,,,,,,3` | `rr_best_of 3 needs a round robin` |
| `category,BOTHMD,Both,4,1,2,bye,,,,,,2,3` | `cannot be combined` |

The mock does not evaluate formulas, so these checks compare formula **text**. Whether the
formulas compute correctly is checked in a real workbook (§9 step 2).

---

## 5. Sync API (`sage-tools-api/src/`)

### 5.1 Double round robin and the new finals: no change

`src/sync/domain/facilityCompletion.mjs` counts each published match on its own, and score
entry writes one match's `SCHEDULE` cells by match number. Neither cares that two matches
share a pair of codes. The site's parity test (§6) runs this server code against the same
inputs, so no `sage-tools-api/test/` change is needed for items 1 and 2.

### 5.2 Best of 3: game 3 after a 2–0 is not a match left

`facilityIsComplete` decides when a facility's day ended (`completedAt`). Today it leaves out
a series Final's games after the decider (`unneededSeriesGames`) and a team playoff's unneeded
tiebreaker (`unneededDeciders`). A best-of-3 meeting needs the same treatment, or a day whose
every meeting ended 2–0 is never complete.

**Test first.** In `test/unit/sync/domain/facilityCompletion.test.mjs`, copy the style of the
existing `describe("series finals: games that are never needed")` cases and add a
`describe` of these. The file's `row(n, t1, s1, t2, s2)` and `csv(...rows)` helpers have no
`bestOf` column: add `const BO3_HEADER = HEADER + ",bestOf"` and append `,3` to each row (or
`,` for the blank case):

| Case | Rows (`bestOf` 3 on each) | `facilityIsComplete` |
| --- | --- | --- |
| a meeting won 2–0 | `A_1` v `A_2` as matches 1, 2, 3: 11–5, 11–7, blank | `true` |
| a meeting at 1–1 | 11–5, 5–11, blank | `false` |
| a meeting won 2–1 | 11–5, 5–11, 11–9 | `true` |
| rows out of order | listed as match 3 (blank), then match 1 (11–7), then match 2 (11–5) | `true` |
| no `bestOf` column | 11–5, 11–7, blank, header without `bestOf` | `false`, as today |
| `bestOf` blank | the same rows with the column present but empty | `false` |
| two meetings | `A_1` v `A_2` won 2–0 and `A_3` v `A_4` at 1–1, all `bestOf` 3 | `false` |

The out-of-order case catches code that takes games in row order instead of match-number
order. By match number, matches 1 and 2 are games 1 and 2, both won by `A_1`, so match 3 is
not needed and the day is complete. In row order, the blank match 3 would be game 1 and
would stay a match left.

Run `npm test` and watch them fail, then add to `facilityCompletion.mjs`:

```js
// A best-of-3 round-robin meeting: the rows flagged bestOf 3 (the CSV tab's bestOf column)
// with the same two codes, in match-number order, are its games 1-3. The first side to win
// two takes it, and a game after that is not needed. A tied game counts for neither side;
// a fourth row is an ordinary match. Mirrors the site's lib/v1/domain/series.js
// (rrSeriesGroups, walkSeries).
//   games: [{ key, num, bestOf, code1, score1, code2, score2 }]  ->  Set of keys
export function unneededMeetingGames(games) {
    const byPair = new Map();
    for (const g of games) {
        if (g.bestOf !== 3) continue;
        const pair = [(g.code1 || "").trim(), (g.code2 || "").trim()].sort().join("|");
        if (!byPair.has(pair)) byPair.set(pair, []);
        byPair.get(pair).push(g);
    }
    const unneeded = new Set();
    for (const list of byPair.values()) {
        list.sort((x, y) => x.num - y.num);
        const wins = new Map();
        let decided = false;
        for (const g of list.slice(0, 3)) {
            if (decided) { unneeded.add(g.key); continue; }
            if (!hasScore(g.score1) || !hasScore(g.score2)) continue;
            const s1 = parseInt(String(g.score1).trim(), 10), s2 = parseInt(String(g.score2).trim(), 10);
            if (s1 === s2) continue;
            const side = (s1 > s2 ? g.code1 : g.code2).trim();
            wins.set(side, (wins.get(side) || 0) + 1);
            if (wins.get(side) >= 2) decided = true;
        }
    }
    return unneeded;
}
```

In `facilityIsComplete`:

- `const iBest = at("bestOf");`
- give each `real` row `num: parseInt(cell(row, iNum), 10)` and
  `bestOf: parseInt(cell(row, iBest), 10) === 3 ? 3 : null`. `cell` returns `""` for a missing
  column, so a CSV tab without `bestOf` gives `null`.
- add `...unneededMeetingGames(real)` to the `unneeded` set.
- add `unneededMeetingGames` and the site's `rrSeriesGroups` to the file's header comment,
  which lists what it mirrors.

Then `package.json` 3.7.0 and a `README.md` Changelog entry, for example: "A best-of-3
round-robin meeting's game 3 after a 2–0 is not a match left, so `completedAt` is stamped
when every meeting is decided. **Tests:** `test/unit/sync/domain/facilityCompletion.test.mjs`."
The root `CLAUDE.md`'s "Things that must be kept in sync by hand" bullet on the "played"
rules gets the same sentence (§8).

Score entry needs no change: it writes one match's cells by match number.

## 6. Site (`sage-match-control.github.io`)

### 6.0 Double round robin and the new finals: tests only

For items 1 and 2 of §0.1, nothing in `lib/v1/`, the event templates or Control Center needs
to change:

- The event pages are shells over `lib/v1/`. Control Center uses the same modules.
- A search of `lib/v1`, `tools/control-center.html` and `_templates/` found nothing that finds
  a match by its two pairs.
- The podium (`lib/v1/domain/awards.js`) already reads a one-game Final, a played Bronze and
  a bronze walkover.
- `headToHeadWins` counts every played meeting.
- No new `events.json` setting is needed.

Two differences are intended, not bugs:

- Both meetings of a double round robin read "Round robin" everywhere players look (§10).
- A one-bracket Final + Bronze shows as Final and Bronze match cards, without the playoff
  bracket chart. The chart (`lib/v1/domain/bracket.js`, `buildBracket`) draws only for two or
  more playoff rounds, which is how a twice-to-beat category already looks.

**Tests to add** (`node:test` with `assert/strict`; copy the style of the file named):

| File | Test |
| --- | --- |
| `_tests/unit/parity-server.test.mjs` | Add three entries to `CASES`, built with the file's `csvOf` and `row` helpers. The existing loop then checks that site and server agree on each. (1) `'a double round robin, one meeting played'`: `row(1,'A_1',…,11,'A_2',…,5)` and `row(2,'A_2',…,'','A_1',…,'')`. (2) the same, both played. (3) `'a one-game final and bronze'`: `A_F_1` v `A_F_2` and `A_B_1` v `A_B_2`, both played |
| `_tests/unit/domain/matches-standings-codes.test.mjs` | `'rankStandings: a 1–1 split in a double round robin falls through to quotient'`: two rows level on wins, two matches between them won one each, and the higher quotient ranks first. Copy `'rankStandings: wins first, then head-to-head among the tied, then quotient'` |
| `_tests/unit/domain/golive-progress.test.mjs` | `'two matches with the same two codes are two matches'`: `computeFacilityProgress(rowsToMatches(parseCSV(csv)), false, null)` with one of them played gives `total` 2 and `left` 1 |
| `_tests/unit/domain/teams-awards-attendance-grid-model.test.mjs` | `'a one-bracket Final and Bronze beside a round robin decide the podium from the bracket'`: the existing `'a Final and a Bronze decide the podium…'` test, with round-robin matches `ND_1`…`ND_4` added. `buildPodiums` still gives `source: 'bracket'` and the same medallists |

If one of these fails, stop and report it to the owner. The fix would be a compatible change
in `lib/v1` plus the matching change in `facilityCompletion.mjs`, and that is outside this
spec.

### 6.1 Best of 3: a compatible change in `lib/v1/domain/`

A best-of-3 meeting changes two things the site works out from the matches: which games are
**not needed** (and so not "matches left", and greyed on the cards, the board, the scorer and
Control Center, all of which already call `unneededSeriesGames`), and **head-to-head** in
`rankStandings`. W, L and quotient come from `STANDINGSCSV` (§4.11) and need nothing here.
Control Center, the templates and the event shells do not change.

1. **`matches.js`, `rowsToMatches`.** Read the column by name, `const iBest = idx('bestOf')`,
   and give every Match `bestOf: iBest > -1 && parseInt(row[iBest], 10) === 3 ? 3 : null`. Add
   `bestOf` to the Match shape in the file's header comment: "3 on a best-of-3 round-robin
   game (the workbook's `bestOf` column), else null".
2. **`schedule-grid.js`.** The board parses rows itself. Give its row object the same
   `bestOf`, read the same way. In `markUnneededGames`, add `num: m.num, bestOf: m.bestOf` to
   each view, so the series rule can see them.
3. **`series.js`.**
   - `walkSeries(list)` becomes `walkSeries(list, games = list[list.length - 1].game)`, and
     its first line, which works `games` out, goes. Every existing call passes one argument
     and behaves as before.
   - Add:

     ```js
     /**
      * A best-of-3 round-robin meeting's games: the matches flagged bestOf 3 with the same
      * facility and the same two codes, in match-number order, the first three of them. Seat 1
      * is the code that sorts first. Shaped like seriesGroups' lists, so walkSeries(list, 3)
      * reads them. Control Center and the scorer pass every facility's matches at once, and
      * match numbers repeat across facilities, so the facility is part of the key.
      * Mirrors sage-tools-api's facilityCompletion.mjs (unneededMeetingGames).
      * @param {object[]} matches
      * @returns {Map<string, Array<{ m: object, game: number, seat1: number, seat2: number }>>}
      *   "<facility>|<code>|<code>" -> its games
      */
     export function rrSeriesGroups(matches){
       const byPair = new Map();
       matches.forEach(m => {
         if(m.bestOf !== 3) return;
         const key = [m.facility || '', ...[m.t1, m.t2].sort()].join('|');
         if(!byPair.has(key)) byPair.set(key, []);
         byPair.get(key).push(m);
       });
       const out = new Map();
       byPair.forEach((list, key) => {
         const first = key.split('|')[1];
         out.set(key, list.sort((x, y) => x.num - y.num).slice(0, 3).map((m, i) => ({
           m, game: i + 1, seat1: m.t1 === first ? 1 : 2, seat2: m.t1 === first ? 2 : 1 })));
       });
       return out;
     }

     /** A best-of-3 meeting's games after the one that gave a side its second win. */
     export function unneededMeetingGames(matches){
       const out = new Set();
       rrSeriesGroups(matches).forEach(list => walkSeries(list, 3).unneeded.forEach(m => out.add(m)));
       return out;
     }
     ```

   - In `unneededSeriesGames`, add `unneededMeetingGames(matches).forEach(m => out.add(m));`
     beside the deciders line, and add "a best-of-3 round-robin meeting's games after its
     decider" to its comment. That call is within one module, so it can ship in the same
     push.
   - `seriesGroups` is unchanged. The podium (`awards.js`) reads Finals only, and a
     round-robin code never matches `SERIES_GAME_RE`.
4. **`standings.js`, `headToHeadWins`** (the **second push**, §0.3: it uses `series.js`'s new
   export and its new `walkSeries` argument). A best-of-3 meeting counts once, for the pair
   that took it:

   ```js
   import { rrSeriesGroups, walkSeries } from './series.js';
   …
   matches.forEach(m => {
     if(m.bestOf === 3) return; // counted once per meeting, below
     … // the existing body
   });
   rrSeriesGroups(matches).forEach(list => {
     const { m } = list[0];
     if(!wins.has(m.t1) || !wins.has(m.t2)) return;
     const { decidedBy } = walkSeries(list, 3);
     if(!decidedBy) return;
     const winner = decidedBy.t1Score > decidedBy.t2Score ? decidedBy.t1 : decidedBy.t2;
     wins.set(winner, wins.get(winner) + 1);
   });
   ```

   Update the comments on `headToHeadWins` and `rankStandings` to say a best-of-3 meeting is
   one result. `series.js` imports only `deciders.js`, so this adds no import cycle, and both
   files are `domain/`, which `_tests/unit/guards.test.mjs` allows.

### 6.2 Best of 3: tests and the two pushes

**First push** (`matches.js`, `schedule-grid.js`, `series.js`, with `sage-tools-api` 3.7.0 on
its own schedule):

| File | Test |
| --- | --- |
| `_tests/unit/domain/csv-byes-series.test.mjs` | `'a best-of-3 meeting won 2–0: game 3 is not needed'`: three rows `A_1` v `A_2` with a `bestOf` column of `3`, 11–5, 11–7, blank. `unneededSeriesGames` holds match 3 and only match 3. `'a best-of-3 meeting at 1–1 needs game 3'`. `'sides swapped between games are one meeting'`: game 2 listed `A_2` v `A_1`. `'without a bestOf column, three rows are three matches'`: the same 2–0 rows with no column, and `unneededSeriesGames` is empty |
| `_tests/unit/domain/golive-progress.test.mjs` | `'a best-of-3 meeting won 2–0 leaves no match'`: `computeFacilityProgress` gives `total` 2, `done` 2, `left` 0 |
| `_tests/unit/domain/matches-standings-codes.test.mjs` | `'rowsToMatches reads bestOf'`: `3` gives 3, blank, `1` and a missing column give `null` |
| `_tests/unit/parity-server.test.mjs` | Add `const BO3_HEADER = HEADER + ',bestOf'` and a `bo3` helper that appends `,3` to a `row(…)`. Add `CASES` for each row of §5.2's table, so that site and server agree on every one, the out-of-order case included |
| `_tests/unit/domain/csv-byes-series.test.mjs` (where `markUnneededGames` is tested) | `'the board marks a best-of-3 meeting's game 3 not needed after a 2–0'`: rows with `num` and `bestOf`, and only game 3's `notNeeded` is true |

**Second push** (`standings.js`), at least 10 minutes after the first is live:

| File | Test |
| --- | --- |
| `_tests/unit/domain/matches-standings-codes.test.mjs` | `'rankStandings: a best-of-3 meeting is one head-to-head result'`. Three rows level on wins: `A`, `B`, `C`. `A` beat `B` 2–1, `B` beat `C` 2–1, `C` beat `A` 2–0, and `A` has the best quotient. Counting meetings, the three are 1–1–1 and the quotient puts `A` first. Counting games, `A` (2 games) would rank last behind `B` and `C` (3 each). Copy `'rankStandings: wins first, then head-to-head among the tied, then quotient'` |

Match objects built by hand in these tests need `num` and `bestOf: 3`. The existing helpers
build neither: `M` in `matches-standings-codes.test.mjs` and `dkMatch` in
`csv-byes-series.test.mjs`. Write, for example, `{ ...M('A', 11, 'B', 5), num: 1, bestOf: 3 }`,
or build the matches from CSV text with `rowsToMatches(parseCSV(csv))`, which reads both.

Then `npm test` and `npm run verify` in `_tests/` for each push. The comparison harness must
show no difference on every existing page: no fixture has a `bestOf` column.

## 7. What the masters need

The **Standard Tournament Master** workbook carries its own copy of the generator, pasted
into its Apps Script project. After the generator merges, the owner pastes the new
`standard-tournament-generator.gs` over the old one and checks that **SAGE → Help** shows
1.5.0. Nothing else in the master changes: no new Named Function and no template change.

Until that paste, the master runs 1.4.1. 1.4.1 rejects `final_bronze` and `final` (unknown
`solo_format`), but silently ignores `rr_legs` and `rr_best_of`. A plan with a double round
robin, or with best-of-3 meetings, would therefore build one-game single round-robin meetings
without a word. Use no plan with `rr_legs` or `rr_best_of` before the paste.

A best-of-3 event also needs, **live before its dry run**:

- `sage-tools-api` 3.7.0 deployed (`GET /ping`'s `X-App-Version`), or `completedAt` is never
  stamped for a facility whose meetings end 2–0
- both site pushes of §6.2, or game 3 after a 2–0 stays a match left and head-to-head counts
  games.

## 8. Docs and runbooks

**`sage-docs`**, on its own branch. Use the present tense, and describe what the tools do.

- `docs/features/tournament-calculator.md` and `docs/technical/tournament-calculator.md`: the
  round-robin choice (single, double, best-of-3 meetings), the four one-bracket formats, the
  fallbacks and notes, and the `rr_legs` and `rr_best_of` columns.
- `docs/features/standard-tournament-generator.md` and
  `docs/technical/standard-tournament-generator.md`: the `final_bronze` and `final` shapes,
  the double and best-of-3 round robins' `MATCHES` rows, the grid, the `CSV` score formulas
  and `bestOf` column, the series W and L, and seating from the site's standings.
- `docs/technical/named-function-library.md`: where the generator calls the column
  primitives from cells (§4.7, §4.8 and §4.11).
- `docs/specs/implemented/standard-tournament-master-spec.md`: add the two shapes to §5.5's
  table and §8's code grammar, and the `bestOf` column to §10.5's `CSV` tab, with a link to
  this spec.
- `docs/technical/sync-pipeline.md` (where `completedAt` is described): a best-of-3
  meeting's unneeded game 3. `docs/technical/site-engine.md` gets the same rule beside the
  series-Final one.
- `docs/features/tournament-hub.md` and `docs/features/schedule-board.md` (where a series
  Final's unplayed games are described): a greyed game 3 in a best-of-3 meeting.
- `docs/usage/plan-the-event.md` and `docs/usage/glossary.md`: "double round robin",
  "best-of-3 meeting", "final and bronze".
- Move this spec to `implemented/` by the procedure in `docs/specs/README.md` ("Moving a
  spec between folders"), and add a MINOR changelog entry (`docs/changelog.md`, with the same
  version in `mkdocs.yml`'s `copyright` line).

**Site repo:** `_templates/dry-run-checklist-template.md`.

- For a one-bracket Final, seat RR #1–#4 from the Hub's or Control Center's standings (§4.6).
- For a double round robin, pack all of leg 1 into `SCHEDULE` before any of leg 2 (§10).
- For best-of-3 meetings:
  - Pack each meeting's three games (`RR n · G1`–`G3` on `MATCHES`) back to back on one
    court. Every pairing has exactly three rows.
  - After a 2–0, leave game 3's scores blank. Never type `0`–`0` into it: a typed score is a
    played game.
  - The organiser can move the next meeting onto that court early.
  - Check that a game 3 after a 2–0 is greyed on the Hub and that the facility card reads
    "All matches done" once every meeting is decided.

**Workspace root:** `CLAUDE.md`, two places:

- The `standard-tournament-generator.gs` bullet. Add one sentence: from 1.5.0 it builds a
  one-bracket double round robin (`rr_legs`), best-of-3 round-robin meetings in any bracket
  count (`rr_best_of`, W and L counting meetings) and the `final` and `final_bronze` playoffs.
  A workbook with either kind of round robin reads `CSV` scores by match number, and one
  with best of 3 adds a `bestOf` column. Cite this spec as
  `sage-docs/docs/specs/.../double-round-robin-and-finals-spec.md`.
- "Things that must be kept in sync by hand", the "played" and "BYE" rules. After the series
  Final sentence, add: "A best-of-3 round-robin meeting's games (rows with the `CSV` tab's
  `bestOf` at 3 and the same two codes, in match-number order) stop counting once a side has
  won two."

## 9. Build order and acceptance

1. **Calculator** (site repo, §3). Usable for planning on its own.
   - [ ] §2.4's six rows reproduced exactly: total matches, slots and finish time.
   - [ ] Unchanged numbers for existing shapes. Import
     `sage-tools-api/apps-script/fixtures/pfs-main-plan.csv` and check the totals against
     `standard-tournament-master-spec.md` §1.1 (NWD 16, NMD 36, NXD 48, LIWD 16).
   - [ ] Export then Import round-trips all eight combinations of §0.1. An old 12-column file
     imports unchanged.
   - [ ] *Final + bronze match* is disabled below 4 pairs. A category saved on
     `final_bronze` that drops to 3 pairs shows the fallback note and exports `final`.
   - [ ] *Double round robin* is disabled at 2+ brackets. A category saved on 2 legs then
     given 2 brackets shows the fallback note and exports `rr_legs` empty.
   - [ ] §2.6's four rows reproduced exactly: total (least), slots and finish time, and the
     summary line reads `… best-of-3 round-robin meetings …`.
   - [ ] *Best-of-3 meetings* is offered at 1 bracket and at 2+, and not for a 2-pair
     category or a dual meet. Pressing it releases *Double round robin*, and the other way
     round.
   - [ ] Export then Import round-trips best of 3 with each of the four one-bracket formats
     and with a multi-bracket category. A 13-column file imports with one-game meetings.
2. **Generator 1.5.0** (`sage-tools-api`, §4).
   - [ ] `node apps-script/verify-standard-tournament-generator.mjs` ends `ALL CHECKS PASSED`,
     with every pre-existing check untouched.
   - [ ] `npm run verify` in `sage-tools-api` passes.
   - [ ] **The owner, in a copy of the master with 1.5.0 pasted**, generates Format B from the
     calculator's handoff, packs `SCHEDULE`, runs Fill match numbers and enters scores:
     - each meeting of a pairing publishes its own score on `CSV`
     - an unplayed match publishes blank scores
     - the grid shows `11 · 7`
     - W, L and quotient on the tab and in `STANDINGSCSV` count both legs.
   - [ ] **The owner, likewise, generates `GBMD` of §4.12** (8 pairs, 2 brackets, best of 3),
     packs and fills it, and enters one meeting 2–0, one 2–1, one at 1–1 and one with a tied
     game:
     - `CSV!M` reads `3` on every round-robin game and is blank on every playoff match
     - the grid shows each game, for example `11 · 6 · 11`
     - W and L on the tab and in `STANDINGSCSV` count meetings: 2–0 and 2–1 are one W for the
       winner and one L for the loser; 1–1 and the tied game give neither
     - the quotient counts every game's points
     - the Hub's standings for each bracket rank by the meeting W, which is what the
       operator seats the semifinals from.
3. **API 3.7.0** (`sage-tools-api`, §5.2). Test first. `npm test`, then `npm run verify`.
   Changelog entry with its **Tests:** line.
4. **Site** (§6). `npm test` in `_tests/` passes, then `npm run verify`, for §6.0's tests and
   §6.2's first push, then again for the second push.
5. **Dry run** (the owner), on generated Format A, B and C workbooks: Hub standings, Final
   and Bronze cards, Control Center's Awards tab, and "All matches done" on the facility card.
   For Format C, also: a game 3 after a 2–0 is greyed on the Hub, the board and the scorer
   page; head-to-head breaks a tie by meetings won; and `completedAt` is stamped when the last
   meeting is decided.
6. **Docs** (§8).

## 10. Not in scope

- Labelling the second meeting "Leg 2" where players see it. Only the `MATCHES` tab labels
  it.
- A double round robin for 2+ brackets, in a dual meet or in a team event.
- Seating RR #1–#4 automatically. The operator types the names, as for twice-to-beat today.
- Any change to the existing `ttb`, `rr`, best-of-3 or multi-bracket playoffs.
- A best-of-3 final after a round robin. This was discussed and set aside: the site's series
  rule already handles three games, so it would be a fifth one-bracket format in the
  calculator and generator only.
- **Interleaving the legs.** All of leg 1 is played before any of leg 2:
  - halfway through, the standings mean something
  - a day running late can drop leg 2 and still have a complete round robin
  - no pairing plays twice in a row.

  The generator's suggested packing already keeps that order, because the leg-2 matches
  queue after leg 1. The operator keeps it when packing `SCHEDULE` by hand.
- Labelling a best-of-3 game "Game 2 of 3" where players see it (D12). Only the `MATCHES` tab
  labels it, and a game 3 that is not needed is greyed, as in a series Final today.
- Best-of-3 meetings in a dual meet or a team event, or together with a double round robin
  (D10).
- Longer playoff matches after a best-of-3 round robin. The playoff plays as it would
  without best of 3 (D8).
- Spreading a meeting's games across the day. They run back to back on one court (§2.5).

## 11. Decisions

Taken by the owner on 2026-11-10.

| # | Question | Decision |
| --- | --- | --- |
| D1 | Bronze after a double round robin | **The organiser chooses** RR #3 automatically (`final`) or RR #3 v RR #4 (`final_bronze`, 4+ pairs). Never the last-placed pair |
| D2 | Gold after a double round robin | **Either** a one-game Final or twice-to-beat. All four one-bracket formats go with either leg count |
| D3 | Double round robin with 2+ brackets? | **No, one bracket only.** The calculator disables it; the generator rejects it |
| D4 | `CSV` scores by match number in every new workbook? | **Only in a workbook with a double or best-of-3 round robin, for now.** Revisit once one has run at an event |
| D5 | "Leg 2" where players see it? | **Not now** |
| D6 | First event to use these formats | **Within a month** of 2026-11-10. Its name and date are **still open**. The build is scheduled back from that date: nothing merges before 14 November, and all of §9 is done at least 3 days before the event |
| D7 | Say outright that the templates and Control Center need no change? | **Yes** (§6). Best of 3 changes `lib/v1/domain/` only, and compatibly |
| D8 | Best-of-3 round robin: which categories, and which playoff? | **Any standard category with a round robin, at any bracket count.** The playoff is the one the category would have anyway, each match its usual length |
| D9 | Best of 3: what do W and L count? | **Meetings won and lost.** The quotient counts every game's points, and head-to-head on the site counts meetings |
| D10 | Best of 3 together with a double round robin? | **No.** The calculator offers one or the other; the generator rejects both together |
| D11 | Best of 3: how does the calculator plan game 3? | **At full length**, every meeting 3 games, with the least (every meeting 2–0) shown beside it |
| D12 | "Game 2 of 3" where players see it? | **Not now**, the same as D5. A game 3 that is not needed is greyed |
