# Spec — Double round robin and one-bracket finals

> **Status: not started.** Nothing here is built. Written 2026-11-10 against
> `sage-match-control.github.io/tools/tournament-calculator.html`,
> `sage-tools-api/apps-script/standard-tournament-generator.gs` 1.4.0 and the site engine
> `lib/v1/` as of that date. Every decision is taken (§11) except the first event's name and
> date.

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

You add two things. Both apply to **one-bracket categories only**.

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

### 0.2 Where things are

The workspace root is `D:\Personal\SAGE`. It holds four independent git repos, and you change
three of them:

| Repo | What you change | Section |
| --- | --- | --- |
| `sage-match-control.github.io/` (GitHub Pages site, no build step) | `tools/tournament-calculator.html` (the calculator, one self-contained HTML file); tests in `_tests/unit/`; `_templates/dry-run-checklist-template.md` | §3, §6, §8 |
| `sage-tools-api/` | **Only** `apps-script/standard-tournament-generator.gs` and `apps-script/verify-standard-tournament-generator.mjs`. Nothing under `src/`, `test/` or `package.json` | §4 |
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
  do not merge to `main`: the owner merges. Commit only if the owner asks you to.
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
  existing checks prove this, and you must not edit them (§4.11).
- **Generator version.** Bump `STANDARD_TOURNAMENT_GENERATOR_VERSION` from `'1.4.0'` to
  `'1.5.0'`, and add a history line above it (§4.1). `apps-script/` is not part of the
  Cloud Run service: do **not** bump `sage-tools-api/package.json`, and do not add a
  `README.md` changelog entry there.
- **Docs.** Write in the present tense: describe what the system does, not what changed.
  Every `sage-docs` change carries a version bump and a changelog entry (§8).

### 0.4 Commands

```bash
cd sage-tools-api && node apps-script/verify-standard-tournament-generator.mjs
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

Every box in §9 is ticked, both verify commands pass, and §11 is unchanged. The one check
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
| Site (Hub, Control Center, scorer, board) | identifies a match by its match number; podium, bronze walkover and the "played" rule already handle these codes | nothing to change; tests only (§6) |
| Sync API (`sage-tools-api/src/`) | counts each match on its own | nothing to change (§5) |

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

---

## 3. Calculator — `sage-match-control.github.io/tools/tournament-calculator.html`

The functions named here exist today. Search for them by name.

### 3.1 State

- `defaultCat()` returns a category object. Add `rrLegs: 1` to it.
- Configs saved before this change have no `rrLegs` (they persist under the `ttc-config`
  key). Read a missing or invalid value as 1 everywhere: `c.rrLegs === 2 ? 2 : 1`.

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
const legs = (c.rrLegs===2 && groups===1) ? 2 : 1;
if(c.rrLegs===2 && groups>1) notes.push('A double round robin is for one bracket only, so this category plays a single round robin.');
if(legs===2 && sizes[0]>6) notes.push(`A double round robin of ${sizes[0]} pairs means ${2*(sizes[0]-1)} matches per pair. Consider a single round robin, or more brackets.`);
```

- `notes` is declared later in the function today. Move its declaration up so these lines
  can use it.
- Multiply the round-robin count by `legs`: `const rr = rrMatches(sizes) * legs;`.
- Pass `solo` (not `c.soloFormat`) to `playoffPlan`.
- Add `legs` and `solo` to the returned object.
- The `teams === 2` best-of-3 early return is unchanged. It has no round robin, so `legs` is
  ignored there.

### 3.4 Timing: `rrRoundFloor(p)` and `seqParts(p, courts)`

- `rrRoundFloor`: multiply the result by `(p.legs || 1)`.
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
2. **New `legsUI`**, shown only when `p.plan.single` is true (one bracket, not best-of-3, not
   a dual meet). Two buttons: `data-legs="1"` *Single round robin* and `data-legs="2"`
   *Double round robin*, with `aria-pressed` from `p.legs`. Put it before `soloUI` in
   `planEl.innerHTML`.
3. **Click handler** (`$('cats').addEventListener('click', …)`): add
   `if(fb.dataset.legs)c.rrLegs=Number(fb.dataset.legs);` beside the existing `dataset.solo`
   line.
4. **Summary line** (`lineEl.innerHTML`): the `p.plan.single` branch today always describes
   twice-to-beat. Branch on `p.plan.solo`:
   - `ttb`: unchanged.
   - `final_bronze`: `<b>${p.total} matches</b> — ${grpTxt} → ${p.rr} round-robin + final and bronze. RR #1 v #2 for gold, #3 v #4 for bronze.`
   - `final`: `<b>${p.total} matches</b> — ${grpTxt} → ${p.rr} round-robin + final. RR #1 v #2 for gold; #3 takes bronze with no match.`
   - `rr` keeps its own existing branch, which comes before `single`.
   - When `p.legs === 2`, every one of these says `double round robin` instead of
     `round-robin`, for example `→ 12 double round robin + final.`
5. **`hasTTB`** (the "counted at full length" note) is true today for every `single`
   non-`rrOnly` plan. Make it
   `r.per.some(p=>p.plan&&(p.plan.solo==='ttb'||p.plan.bestOf3))`, keeping the `p.plan&&`
   guard.
6. **`advlab`** stays `Playoffs · single group format` for all four formats.
7. **The per-category table** (`struct`, using `SHORT`) already renders `F 1 · Bz 1` and
   `F 1 · Bz auto` from the rounds. Append ` · 2 legs` when `p.legs === 2`.

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
  under the same condition as today (standard, `p.plan.single`, not best-of-3).
- New 13th cell: `'2'` when `!isDual && p && p.legs===2`, else `''`. A single round robin
  writes it empty, so an unchanged plan differs from today's export only in the header line.
- `buildPlanCsv` also feeds the generator handoff (the clipboard copy), so nothing else
  changes for that.

### 3.8 CSV import: `importCSVText(text)`

- `soloFormat`: accept `'ttb'`, `'rr'`, `'final_bronze'` and `'final'`. Anything else is
  `'ttb'`.
- `rrLegs`: `(row[12]||'').trim()==='2' ? 2 : 1`. Add it to the pushed category object.
- An old 12-column file imports exactly as before.

### 3.9 `.xlsx` export: `exportXLSX()`

The Format cell (`fmt`) for a standard category with rounds: append ` (#3 bronze auto)` when
`p.plan.bronzeText` is set, and ` · 2 legs` when `p.legs === 2`. The existing `SHORT` map
already renders `Final 1, Bronze 1`. No new columns.

### 3.10 Do not touch

The dual-meet code and its CSV columns, `GENERATOR_MASTERS`, `tools/sw.js`, and
`tools/tournament-calculator-beta.html` (an untracked scratch copy, ignored by `**/*beta*`).

---

## 4. Generator — `sage-tools-api/apps-script/standard-tournament-generator.gs`

### 4.1 Version

```
//   1.5  double round robin (rr_legs) and the one-bracket final and final_bronze formats
var STANDARD_TOURNAMENT_GENERATOR_VERSION = '1.5.0';
```

### 4.2 Input: `parsePlanCsv(text)`

Add `rrLegs: (row[12] || '').trim(),` beside `soloFormat: (row[11] || '').trim(),`.

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

1. Add `rrLegs: cat.rrLegs === '2' ? 2 : 1` to the initial `p` object.
2. After the `if (teams === 2) … else …` block, add these errors:
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

4. **`MATCHES` row labels.** The line
   `p.matchesRows = p.rrRows.map(function (row, idx) { return { label: 'RR ' + (idx + 1), … } })`
   labels leg 2's rows `RR 1 · Leg 2` … `RR n · Leg 2` instead, where `n` is the number of
   leg-1 rounds (`p.rrRows.length / 2` when `p.rrLegs === 2`). Leg 1 stays `RR 1` … `RR n`.
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

The rows that result, which §4.11 checks:

| Category | Group pairs | Gap | `FINALS` banner | First block label row (seats) | Second block label row (seats) |
| --- | --- | --- | --- | --- | --- |
| Format A, 5 pairs (`finalBronze`) | 6–15 | 16 | 17 | `B` at 19 (20, 22) | `F` at 25 (26, 28) |
| Format B, 3 pairs (`final`) | 6–11 | 12 | 13 | `F` at 15 (16, 18) | `B` at 21 (22, 24) |

### 4.6 Qualifier scaffold

`writeQualifierScaffold_` needs no change. Rows with `ap` set are written as literal codes
in `AP`, and an empty `p.freshSeats` means no slot table, exactly as for `ttb`. `AM` stays
`-`. The operator types each pair's names into `AO` once the round robin is over.

**The operator seats from the Hub's or Control Center's standings, not the category tab's.**
The tab ranks by wins and quotient only. The site breaks a tie on wins by head-to-head first
(`rankStandings` in `lib/v1/domain/standings.js`), and in a double round robin that counts
both meetings. §8 puts this in the dry-run checklist.

### 4.7 The score grid in a double round robin: `writeGroupGrid_`

`writeGroupGrid_(sheet, templateSheet, first, nb)` is called from `buildCategoryTab`, where
`p` is in scope. Add a fifth parameter `legs` and pass `p.rrLegs`. Each off-diagonal body
cell is today
`'=GETSCOREAGAINSTPAIR($N' + r + ',' + colLetter_(COL.O + c) + '$' + first + ')'`. When
`legs === 2`, write this instead, with the same `$N<r>` and `<col>$<first>` references:

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
  by ` · ` (for example `11 · 7`), and blank until one is played.
- **Why it is written inline:** the alternative is a new Named Function. Apps Script cannot
  create one, so it would have to be added to the master workbook by hand, and older
  workbooks would not have it.
- **The building blocks:** `GETPLAYERSCOLUMN1/2`, `GETSCORESCOLUMN1/2` and `GETMATCHNUMBERS`
  are existing Named Functions (`sage-docs/docs/technical/named-function-library.md`).
  `LET` names may not look like cell references, which is why they are `t_one` and not `t1`.
- **What reads it:** nothing. The grid is display only.
- **A single round robin** keeps `GETSCOREAGAINSTPAIR`.

### 4.8 The `CSV` tab: scores by match number — `buildCsvTab_`

The site reads every match from the `CSV` tab. Its score columns are
`'=GETSCOREAGAINSTPAIR($E2,$I2)'` (`H`, team 1) and `'=GETSCOREAGAINSTPAIR($I2,$E2)'` (`L`,
team 2), which identify a match by its two codes. In a double round robin both meetings
would publish the same score, and the second meeting would look played as soon as the first
was.

`buildCsvTab_(ss)` is called from `generateEventTabs`, where `plans` is in scope. Change the
call to `buildCsvTab_(ss, plans.some(function (p) { return p.rrLegs === 2; }))`. When that
second argument is true, write `H2` and `L2` as:

```
H2  =IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN1(),GETMATCHNUMBERS()=$A2),1),""))
L2  =IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN2(),GETMATCHNUMBERS()=$A2),1),""))
```

- Why it is right: `E2` is `GETTEAM1CODEBYMATCH($A2)`, the team-1 code at the match's
  position in the same stacked columns, so team 1's score is `GETSCORESCOLUMN1` at that
  position.
- Otherwise, write today's formulas unchanged.
- `Fill match numbers` (an existing menu item) copies row 2 down, so nothing else changes.

### 4.9 Codes carry no leg

A double round robin adds no code. Each pair keeps its one code, `<KEY>_<i>`, in both legs.
A round-robin match is still its two pairs' codes, and only the match number tells two
meetings apart. Do not add a leg suffix: `GETTOTALWINS` and the other totals add up every
match under one code, and several readers depend on one code per pair:

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
so its result is unaffected. `rr_legs` does not change any link row, so it is not inferred.

`playoffLinkRows_` needs no change. The new seats come out with `match` `F` or `B`, `from`
empty and `seat` empty, like `ttb`'s.

### 4.11 Verify script: `apps-script/verify-standard-tournament-generator.mjs`

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

Use the existing `run(name, csv, options)` helper (`options` = `{ facility, courtLabels,
selected }`), its `cell(sheet, row, col)` reader (columns are 1-based: A = 1, K = 11,
P = 16, AP = 42), and `plansOf(sandbox, csv, keys)` for plan-level checks. Then check:

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
are `F (1)`, `F (2)` and `B`, as 1.4.0 builds them.

**Whole workbook** (a run with all four selected): `MATCHES!B2` is `12 + 8 + 14 + 9 = 43`.
`CSV!H2` is
`=IF($A2="","",IFERROR(INDEX(FILTER(GETSCORESCOLUMN1(),GETMATCHNUMBERS()=$A2),1),""))`.

**Single-round-robin workbook** (a second run selecting only `FAMD`): `CSV!H2` is
`=GETSCOREAGAINSTPAIR($E2,$I2)`.

**Add bracket links**: on the all-four workbook, `addBracketLinks_(ss)` returns `ok: true`
and reports nothing missing. Copy how the existing `links` block calls and reads it.

**Rejections** (in the new block, using `sandbox.generateEventTabs(csv, opts)` and its
`errors`):
- `final_bronze` with 3 pairs → an error containing `"final_bronze" needs 4 pairs`.
- `rr_legs` of `3` → `rr_legs must be empty, 1 or 2`.
- `rr_legs` `2` on a 2-pair category → `a 2-pair category has none`.
- `rr_legs` `2` on 8 pairs in 2 brackets → `is for one bracket only`.
- an unknown `solo_format` such as `semis` → `solo_format must be empty, "ttb", "rr", "final_bronze" or "final"`.

The mock does not evaluate formulas, so these checks compare formula **text**. Whether the
formulas compute correctly is checked in a real workbook (§9 step 2).

---

## 5. Sync API (`sage-tools-api/src/`): no change

`src/sync/domain/facilityCompletion.mjs` counts each published match on its own, and score
entry writes one match's `SCHEDULE` cells by match number. Neither cares that two matches
share a pair of codes. The site's parity test (§6) runs this server code against the same
inputs, so no `sage-tools-api/test/` change is needed.

## 6. Site (`sage-match-control.github.io`): tests only

Nothing in `lib/v1/`, the event templates or Control Center needs to change:

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

## 7. What the masters need

The **Standard Tournament Master** workbook carries its own copy of the generator, pasted
into its Apps Script project. After the generator merges, the owner pastes the new
`standard-tournament-generator.gs` over the old one and checks that **SAGE → Help** shows
1.5.0. Nothing else in the master changes: no new Named Function and no template change.

Until that paste, the master runs 1.4.0. 1.4.0 rejects `final_bronze` and `final` (unknown
`solo_format`), but silently ignores `rr_legs`. A plan with a double round robin and `ttb`
or `rr` would therefore build a single round robin without a word. Use no plan with
`rr_legs` before the paste.

## 8. Docs and runbooks

**`sage-docs`**, on its own branch. Use the present tense, and describe what the tools do.

- `docs/features/tournament-calculator.md` and `docs/technical/tournament-calculator.md`: the
  round-robin toggle, the four one-bracket formats, the fallbacks and notes, and the
  `rr_legs` column.
- `docs/features/standard-tournament-generator.md` and
  `docs/technical/standard-tournament-generator.md`: the `final_bronze` and `final` shapes,
  the double round robin's `MATCHES` rows, its grid, its `CSV` score formulas, and seating
  from the site's standings.
- `docs/technical/named-function-library.md`: where the generator calls the column
  primitives from cells (§4.7 and §4.8).
- `docs/specs/implemented/standard-tournament-master-spec.md`: add the two shapes to §5.5's
  table and §8's code grammar, with a link to this spec.
- `docs/usage/plan-the-event.md` and `docs/usage/glossary.md`: "double round robin",
  "final and bronze".
- Move this spec to `implemented/` by the procedure in `docs/specs/README.md` ("Moving a
  spec between folders"), and add a MINOR changelog entry (`docs/changelog.md`, with the same
  version in `mkdocs.yml`'s `copyright` line).

**Site repo:** `_templates/dry-run-checklist-template.md`. For a one-bracket Final, seat
RR #1–#4 from the Hub's or Control Center's standings (§4.6). For a double round robin,
pack all of leg 1 into `SCHEDULE` before any of leg 2 (§10).

**Workspace root:** `CLAUDE.md`, the `standard-tournament-generator.gs` bullet. Add one
sentence: from 1.5.0 it builds a one-bracket double round robin (`rr_legs`) and the `final`
and `final_bronze` playoffs, and a workbook with a double round robin reads `CSV` scores by
match number. Cite this spec as `sage-docs/docs/specs/.../double-round-robin-and-finals-spec.md`.

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
   - [ ] The leg toggle is hidden at 2+ brackets. A category saved on 2 legs then given 2
     brackets shows the fallback note and exports `rr_legs` empty.
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
3. **Site tests** (§6). `npm test` in `_tests/` passes, then `npm run verify`.
4. **Dry run** (the owner), on generated Format A and Format B workbooks: Hub standings, Final
   and Bronze cards, Control Center's Awards tab, and "All matches done" on the facility card.
5. **Docs** (§8).

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

## 11. Decisions

Taken by the owner on 2026-11-10.

| # | Question | Decision |
| --- | --- | --- |
| D1 | Bronze after a double round robin | **The organiser chooses** RR #3 automatically (`final`) or RR #3 v RR #4 (`final_bronze`, 4+ pairs). Never the last-placed pair |
| D2 | Gold after a double round robin | **Either** a one-game Final or twice-to-beat. All four one-bracket formats go with either leg count |
| D3 | Double round robin with 2+ brackets? | **No, one bracket only.** The calculator hides it; the generator rejects it |
| D4 | `CSV` scores by match number in every new workbook? | **Only in a workbook with a double round robin, for now.** Revisit once one has run at an event |
| D5 | "Leg 2" where players see it? | **Not now** |
| D6 | First event to use these formats | **Within a month** of 2026-11-10. Its name and date are **still open**. The build is scheduled back from that date: nothing merges before 14 November, and all of §9 is done at least 3 days before the event |
| D7 | Say outright that the templates and Control Center need no change? | **Yes** (§6) |
