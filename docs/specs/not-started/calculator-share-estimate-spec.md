# Spec — Tournament Calculator: share the estimate, and more category presets

> **Status: not started.** Nothing here is built. Written 2026-11-21 against
> `sage-match-control.github.io/tools/tournament-calculator.html` at `c68e309`, from the
> owner's request to send a plan's estimate to an organiser as an image or as text, and the
> two Paddle Aces The Ultimate Generation Cup day plans (28 and 29 November 2026) as the
> worked examples.
>
> **Dry-run on 2026-11-21.** Every edit in §5 was applied, exactly as written, to a copy of
> the site at `c68e309` (a detached worktree); every anchor matched once and the page's script
> parses. §6's test file passed (6 tests) against that copy, and so did the site's whole unit
> suite (`npm test`: 663 pass, 0 fail). §7's checks were made on that copy in Chromium through
> Playwright at 1280 px and 375 px: both Paddle Aces plans imported, Copy text put §4.1's
> message on the clipboard, Save image downloaded the PNG of §3, the empty plan disabled both
> buttons, and there was no console error and no sideways scroll.

Two changes to the Tournament Calculator, both in `tools/tournament-calculator.html`:

1. **Share the estimate.** A **Share the estimate** box under the per-category breakdown, with
   **Save image** (a PNG of the estimate) and **Copy text** (the same figures as a plain
   message), so an operator can send a plan's finish time and category breakdown to an
   organiser in Messenger, Viber or anywhere else.
2. **More category presets.** **Beginner** Women's, Men's and Mixed Doubles (`BWD`, `BMD`,
   `BXD`), and **Open** Women's and Mixed Doubles (`OWD`, `OXD`) beside the existing Open
   Men's Doubles (`OPEN`).

---

## 0. How to use this spec

You can carry this out with no other context: every value, every edit, every test and every
check is here. Read §0 to §4 before you change anything.

### 0.1 Where things are

The workspace is `D:\Personal\SAGE`. It holds several independent git repos; two change here:

| Repo | Path | What changes |
| --- | --- | --- |
| The site | `D:\Personal\SAGE\sage-match-control.github.io` | `tools/tournament-calculator.html` (§5) and one new test file, `_tests/unit/tools/tournament-calculator-share.test.mjs` (§6) |
| The docs | `D:\Personal\SAGE\sage-docs` | three docs pages, this spec's status, the changelog and `mkdocs.yml` (§8) |

The calculator is one self-contained static page: inline `<style>`, one classic inline
`<script>` (not a module), SheetJS from cdnjs for `.xlsx`, no build step, no framework. The
site's `_tests/` folder is the only Node package (dev-only, never published). Nothing in
`sage-tools-api` or `event-data` changes.

### 0.2 Rules

- **First, check the team format isn't built.** Run
  `grep -c "isTeamFormat" tools/tournament-calculator.html` in the site repo. It must print
  `0`. If it prints anything else, the
  [calculator team format](calculator-team-format-spec.md) has landed since this spec was
  written: **stop and ask the owner**, because the share box then needs a rule for the team
  format that this spec doesn't have.
- **Find edits by their anchor text, not line numbers.** Every edit in §5 quotes an exact piece
  of text that occurs **once** in the file. If an anchor isn't there, or is there more than
  once, **stop and report**. Don't guess.
- **Apply §5's edits in order, exactly as written.** The code was dry-run (status block).
  Don't restyle, rename or "improve" it, and don't make design decisions: if this spec seems
  wrong or two parts disagree, **stop and ask** the owner.
- **Keep the file's style and line endings.** The page is written compactly: 2-space indent,
  single quotes, semicolons, short names. The working copy has CRLF line endings (git stores
  LF); keep them. Check with `git ls-files --eol tools/tournament-calculator.html`: it should
  still read `i/lf w/crlf` afterwards, and `git diff` should show only your lines.
- **Don't touch** the maths (`calcCategory`, `calcDualCategory`, `playoffPlan`, `seqParts`,
  `catTime`, `calcAll`), `buildChart`, the CSV format (`buildPlanCsv`, `importCSVText`), the
  `.xlsx` export, `GENERATOR_MASTERS`, or `tools/sw.js`. No new dependency anywhere.
- **Use Git Bash** for the commands (Windows; the paths above are the same in Git Bash as
  `/d/Personal/SAGE/...`).
- **Commit, but never push or merge without the owner's go-ahead.** The site deploys to GitHub
  Pages on push to `main`, and takes no merge within 3 days of an event (check the dates in
  `event-data/config/events.json`). End each commit message with the attribution line the
  session gives you.

### 0.3 Order of work

1. In the site repo: `git checkout main && git pull && git checkout -b calculator-share-estimate`.
2. Make §0.2's team-format check.
3. Apply §5's edits E1–E11 to `tools/tournament-calculator.html`, in order.
4. Add §6's test file. Run the site's tests (§6): all pass, the new ones included.
5. Serve the site and make §7's checks in a browser.
6. Commit the site change: `Tournament Calculator: share the estimate as an image or text; Beginner and Open presets`.
7. Report to the owner and wait for the go-ahead to merge and push.
8. Once the site change is live, make §8's docs changes in `sage-docs` on a branch, commit them
   as `docs 2.76.0: the calculator shares its estimate; Beginner and Open presets` (or the next
   MINOR version if 2.76.0 is taken), and wait for the go-ahead to push.

## 1. What the page does

- **The Share the estimate box** sits in the results column, between the **Per-category
  breakdown** panel and the **Reset / Import CSV / Export CSV / Download .xlsx** buttons. It
  has a title, one line of help, and two buttons side by side: **Save image** (green, primary)
  and **Copy text**. The file buttons are for the operator's own work; this box is for sending
  the estimate to someone else, so it sits apart from them.
- **Both buttons are disabled while the plan has no matches** (no categories, or a dual-meet
  category with no pairs), and the help line then reads *Add a category first.* Otherwise it
  reads *Send the projected finish and the category breakdown to an organiser.*
- **Save image** draws the estimate (§3) and:
  - on a phone or tablet (a device whose main pointer is coarse, a finger), opens the
    system share sheet with the PNG attached, so it goes straight to a chat app or Photos. If
    the share sheet can't take a file, or fails, the PNG downloads instead; if the operator
    cancels the sheet, nothing happens;
  - anywhere else, downloads the PNG as `<file base>-estimate.png`, where the file base is the
    one the `.xlsx` already uses (`fileBase`): the title in lower case with every run of
    non-word characters turned into `-`, then `-` and the date. Paddle Aces Day 1's is
    `paddle-aces-the-ultimate-generation-cup-day-1-2026-11-28-estimate.png`.
  It confirms in the page's existing message strip (under the file buttons): *Saved
  &lt;name&gt;* or *Shared &lt;name&gt;*.
- **Copy text** copies the message of §4 and confirms *Estimate copied — paste it into any
  chat.* If the clipboard is refused it tries the page's existing `execCommand` fallback
  (`copyFallback`), and if that fails too it says *Could not reach the clipboard. Use Save image
  instead.*
- **Printing** hides the box, as it already hides the file buttons.
- **The finish alert** under the big clock reads its level (good, after 10 PM, past midnight)
  from the same `finishLevel` the summary uses, so the image's verdict and the page's alert
  can't disagree. Its wording doesn't change.
- **The category menu** gains five presets (§2). Nothing else on the page changes.

## 2. The presets

The menu lists, in this order, by level from lowest to highest, Women's, Men's then Mixed in
each:

| Level | Codes |
| --- | --- |
| **Beginner** (new) | `BWD` Beginner Women's Doubles · `BMD` Beginner Men's Doubles · `BXD` Beginner Mixed Doubles |
| Novice | `NWD` · `NMD` · `NXD` |
| Low Intermediate | `LIWD` · `LIMD` · `LIXD` |
| Intermediate | `IWD` · `IMD` · `IXD` |
| High Intermediate | `HIWD` · `HIMD` · `HIXD` |
| Advanced | `AWD` · `AMD` · `AXD` |
| **Open** | `OWD` Open Women's Doubles (new) · `OPEN` Open Men's Doubles · `OXD` Open Mixed Doubles (new) |
| | `CUSTOM` Custom… |

- **Open Men's Doubles keeps the code `OPEN`.** Saved plans, exported CSVs and generated
  workbooks already carry it; renaming it to `OMD` would turn every one of them into a custom
  category on import.
- **A new category still starts from Novice.** **+ Add category** picks the first preset not
  yet used, counting from `NWD` and wrapping round to `BWD` after `OXD`, so a fresh plan starts
  with `NWD` as it does today and Beginner is chosen from the menu.
- **Import CSV recognises the new codes.** A plan exported with `OWD` as a custom category
  (Paddle Aces Day 2's) now imports as the `OWD` preset, with the preset's name. A plan saved
  in the browser keeps its custom category until the operator picks the preset.
- **Outside the calculator nothing reads the menu.** The Standard Tournament Generator takes
  any code from the plan CSV. The Dual Meet Sheet Generator colours a category tab by its
  level and gives none to a level it doesn't know (`categoryColor_`, as it already does for
  `OPEN` and custom codes), so a Beginner or Open category in a dual meet gets no tab colour.

## 3. The image

A portrait PNG, **1080 px wide**, drawn at **2×** (2160 px), as tall as its categories need
(Paddle Aces Day 1, ten categories: 2160 × 4900 px, about 1.1 MB). It is drawn on a
`<canvas>` in the page with the page's own colours and fonts (Archivo Black, Barlow Condensed,
Inter), with no library. Top to bottom:

1. **Header band** (navy): the S.A.G.E. logo (`/assets/logo.png`, already precached by the
   service worker) on a white rounded tile, the kicker *S.A.G.E. · SCHEDULE ESTIMATE* in green,
   the title in upper case (wrapped), the clubs (*PNF vs BUP*, dual meet only) in green, then
   the date and settings line (§4.2's `date · meta`), with a dashed green rule at its foot.
2. **Projected finish** (deep navy box): the label, the finish time large, and the verdict in
   a pill coloured by its level: green *Comfortable one-day plan*, amber *Finishes after
   10 PM*, red *Runs past midnight*.
3. **Four figures** in white tiles: pairs, matches (labelled *MATCHES, MAX* when any category's
   count is a range), time slots, play time.
4. **Categories** (white card, navy title strip): a column head (*CATEGORY · FORMAT*, *MATCHES*,
   *TIME*), then one row per category with matches: the code in green and the name, the format
   line and the playoff line in grey (each wrapped); on the right the match count with its RR +
   PO split under it, and the category's time.
5. **The footnote**, then a rule and the signature line *S.A.G.E. MATCH CONTROL EXPERTS* ·
   *TOURNAMENT TIME CALCULATOR*.

The layout runs twice: once to measure the height, once to paint. The band, the table's card
and the page background are filled last, behind what is drawn, once their heights are known.

**Why the click draws synchronously.** Safari refuses `navigator.share` unless it starts inside
the click. So the fonts and the logo are loaded when the page boots (`preloadShareAssets`),
and the click draws the canvas and encodes it with `toDataURL` (not the asynchronous `toBlob`)
before calling `navigator.share`, with no `await` between. If a font hasn't loaded (offline on
a first visit), the canvas uses the fallback faces in each font stack; if the logo hasn't, the
tile is drawn empty.

## 4. The text, and the summary both are built from

### 4.1 The message

Plain text, one block per category, so it reads the same in Messenger, Viber, WhatsApp, SMS or
email: no table (chat apps use proportional fonts, so columns fall apart), no `*bold*`
(Messenger and SMS show the asterisks), no emoji. Paddle Aces Day 2, as the test pins it:

```text
PADDLE ACES THE ULTIMATE GENERATION CUP DAY 2
Schedule estimate

Sunday, November 29, 2026
Start 10:00 AM · 6 courts · 25-min matches

Projected finish: 1:45 PM
Comfortable one-day plan
25 pairs · up to 53 matches · 9 time slots · 3h 45m play time

CATEGORIES

NMD · Novice Men's Doubles
4 pairs · 1 group of 4 · round robin
Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze
7–8 matches (6 RR + 1–2 PO) · 2h 5m

LIMD · Low Intermediate Men's Doubles
11 pairs · 3 groups (4-4-3) · top 2 advance, top seed bye
Playoffs: R16, QF, SF, Bronze, Final
21 matches (15 RR + 6 PO) · 2h 55m

LIXD · Low Intermediate Mixed Doubles
4 pairs · 1 group of 4 · round robin
Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze
7–8 matches (6 RR + 1–2 PO) · 2h 5m

AMD · Advanced Men's Doubles
3 pairs · 1 group of 3 · double round robin
Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze
7–8 matches (6 RR + 1–2 PO) · 3h 20m

OWD · Open Women's Doubles
3 pairs · 1 group of 3 · double round robin
Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze
7–8 matches (6 RR + 1–2 PO) · 3h 20m

Estimate only: assumes all 6 courts are in use from 10:00 AM with 25-minute matches and no buffer. Twice-to-beat finals and best-of-3 matches are counted at full length, so the day ends earlier if they’re decided sooner.
S.A.G.E. Tournament Time Calculator
```

The date is written in the browser's own locale (the `.xlsx` header does the same), so a
phone set to English (Philippines) may write *Sunday, 29 November 2026*.

### 4.2 The summary

`buildEstimateSummary(cfg, r)` takes the page's `readForm()` and `calcAll()` results and
returns one plain object; `buildEstimateText` and `layoutEstimate` read only that object. It
reads the DOM never, and the page's `cats` variable never (it reads `cfg.cats`), which is what
lets §6 test it outside a browser.

| Field | What it holds |
| --- | --- |
| `title` | The tournament's title, or *Untitled tournament* |
| `clubs` | *&lt;Club A&gt; vs &lt;Club B&gt;* for a dual meet, else empty |
| `date` | The date in full (*Saturday, November 28, 2026*), or empty when the plan has none |
| `meta` | *Start 10:00 AM · 10 courts · 25-min matches*, with *+ 5-min buffer* when there is one, and *1 court* for one |
| `finish`, `level`, `verdict` | The finish time; `finishLevel`'s `good`, `warn` or `bad`; and its words (§3) |
| `pairs` | Every category's pairs, both clubs' for a dual meet (the `.xlsx` TOTAL row's count) |
| `matches`, `ranged` | The plan's match count, which counts every twice-to-beat final and best-of-3 at full length; `ranged` when any category's count is a range, so the text says *up to* and the image *MATCHES, MAX* |
| `slots`, `play` | Time slots, and play time as the page writes it (*7h 5m*) |
| `cats` | One entry per category that has matches (below) |
| `foot` | *Estimate only: assumes all N courts are in use from &lt;start&gt; with N-minute matches and no buffer.* (or *and a N-minute buffer.*, and *the court is* for one court), then, when `ranged`, the full-length sentence |

Each category entry:

| Field | Standard category | Dual-meet category |
| --- | --- | --- |
| `code`, `name` | The preset's, or the custom code and name (`catCode`, `catName`) | The same |
| `format` | *N pairs · &lt;groups&gt; · …*: *top K advance* (plus *+ N wildcards* and *, top seed bye* / *, top N seeds bye* when the plan has them, and *· best-of-3 round robin* for best-of-3 meetings) for a playoff from several groups; *round robin* (or *double round robin*, *best-of-3 round robin*) for one group with a final; *… only, standings decide* for round robin only; *2 pairs · head-to-head, no round robin* for two pairs. Groups read *1 group of 4* or *4 groups (4-3-3-3)* (`groupShorthand`) | *5 v 5 pairs · 1 bracket · cross-club group stage*, or *2 brackets (3v3, 3v3)*; *1 v 1 pairs · one match decides it* |
| `playoff` | *Playoffs: QF, SF, Bronze, Final*, the page's round names shortened (Round of 16 → R16, Quarterfinals → QF, Semifinals → SF, Bronze match → Bronze); for one group: *Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze*, *Final: #1 v #2 · #3 takes bronze*, *Final: #1 v #2 · Bronze: #3 v #4*, *Final: best of 3 (first to 2 wins)*; empty for round robin only | *Playoffs:* and the rounds (*Club playoff*, *Bronze*, *Final*); empty for 1 v 1 |
| `count` | The match count, or a range when the least and most differ (*7–8*, with an en dash) | The count |
| `split` | *15 RR + 8 PO*, each side a range when it has one (*6 RR + 1–2 PO*, *24–36 RR + 4 PO*); empty with no playoff and for two pairs | *25 group + 2 PO*; empty for 1 v 1 |
| `time` | The category's time as the table's **Total time** column computes it | The same |

The text's category block is the code and name, the format, the playoff (when there is one),
then *&lt;count&gt; matches (&lt;split&gt;) · &lt;time&gt;* (*1 match* for one).

## 5. The code: `tools/tournament-calculator.html`

Each edit names an anchor and what to do at it:

- **Insert after**: add the code as new lines after the whole line that contains the anchor.
- **Insert before**: add the code as new lines before the whole line that contains the anchor.
- **Replace**: replace exactly the anchor text (which may span lines) with the code.

The new functions start at column 0, as every function in the page does: §6's test extracts
`finishLevel`, `buildEstimateSummary`, `buildEstimateText` and `defaultCat` by name, and the
consts `PRESETS`, `SHARE_ROUND` and `SHARE_VERDICT`.

### E1. CSS for the share box and disabled buttons

**Insert after** the line containing:

```css
.genbox .btn{display:block;text-decoration:none}
```

these lines:

```css
/* Share the estimate: an image and a plain-text message for an organiser,
   both built from one summary (buildEstimateSummary). */
.sharebox{margin-top:14px;border:1.5px solid #C4DFA0;border-radius:12px;padding:13px 15px;background:#F1F8E7}
.sharebox h3{margin:0 0 4px;font-family:'Barlow Condensed';font-weight:700;letter-spacing:.06em;
  text-transform:uppercase;font-size:14px;color:var(--navy)}
.sharebox p{margin:0 0 11px;font-size:13px;color:var(--muted);line-height:1.5}
.sharerow{display:grid;grid-template-columns:1fr 1fr;gap:8px}
.btn:disabled,.btn:disabled:hover{opacity:.45;cursor:not-allowed;background:#fff;color:var(--navy)}
.btn.primary:disabled,.btn.primary:disabled:hover{background:var(--green-dark);color:#fff}
```

### E2. Hide the share box when printing

**Replace** this text:

```css
header.hero,.toolrow,.addcat,.del,form .panel{display:none}
```

with:

```css
header.hero,.toolrow,.sharebox,.addcat,.del,form .panel{display:none}
```

### E3. The share box, above the file buttons

**Insert before** the line containing:

```html
<div class="toolrow">
```

these lines:

```html
    <div class="sharebox" id="sharebox">
      <h3>Share the estimate</h3>
      <p id="sharehelp">Send the projected finish and the category breakdown to an organiser.</p>
      <div class="sharerow">
        <button type="button" class="btn primary" id="shareimg">Save image</button>
        <button type="button" class="btn" id="sharetext">Copy text</button>
      </div>
    </div>
```

### E4. The presets: Beginner first, Open Women's and Open Mixed beside Open Men's

**Replace** this text:

```js
const PRESETS=[
  ['NWD',"Novice Women's Doubles"],['NMD',"Novice Men's Doubles"],['NXD','Novice Mixed Doubles'],
```

with:

```js
const PRESETS=[
  ['BWD',"Beginner Women's Doubles"],['BMD',"Beginner Men's Doubles"],['BXD','Beginner Mixed Doubles'],
  ['NWD',"Novice Women's Doubles"],['NMD',"Novice Men's Doubles"],['NXD','Novice Mixed Doubles'],
```

### E5. The Open presets

**Replace** this text:

```js
  ['OPEN',"Open Men's Doubles"],['CUSTOM','Custom…']];
```

with:

```js
  ['OWD',"Open Women's Doubles"],['OPEN',"Open Men's Doubles"],['OXD','Open Mixed Doubles'],['CUSTOM','Custom…']];
```

### E6. A new category still starts from Novice

**Replace** this text:

```js
  const free=PRESETS.find(p=>p[0]!=='CUSTOM'&&!used.includes(p[0]))||PRESETS[0];
```

with:

```js
  // Beginner heads the menu, but a new category starts from Novice and
  // reaches Beginner only after Open, as it did before Beginner existed.
  const from=PRESETS.findIndex(p=>p[0]==='NWD');
  const free=[...PRESETS.slice(from),...PRESETS.slice(0,from)].find(p=>p[0]!=='CUSTOM'&&!used.includes(p[0]))||PRESETS[0];
```

### E7. recalc: the finish alert reads finishLevel (1 of 2)

**Replace** this text:

```js
    if(r.endMin-cfg.startMin>15*60||r.endMin>=24*60) alerts.push(
```

with:

```js
    const level=finishLevel(cfg,r);
    if(level==='bad') alerts.push(
```

### E8. recalc: the finish alert reads finishLevel (2 of 2)

**Replace** this text:

```js
    else if(endClock>=22*60) alerts.push(['warn',
```

with:

```js
    else if(level==='warn') alerts.push(['warn',
```

### E9. recalc: the share buttons follow the plan

**Insert after** the line containing:

```js
  $('r-windows').style.display=(r.matches||hasTTB)?'':'none';
```

these lines:

```js
  const canShare=r.matches>0;
  $('shareimg').disabled=!canShare;$('sharetext').disabled=!canShare;
  $('sharehelp').textContent=canShare?'Send the projected finish and the category breakdown to an organiser.':'Add a category first.';
```

### E10. The share code: summary, text, image

**Insert before** the line containing:

```js
/* ---------------- export / import CSV ---------------- */
```

these lines:

```js
/* ---------------- share the estimate ---------------- */
// One summary of the plan (buildEstimateSummary), rendered two ways: a
// plain-text message (buildEstimateText) and a PNG (drawEstimate). Both read
// only the summary, so the image and the text can't disagree. See
// sage-docs/docs/specs/.../calculator-share-estimate-spec.md.
const SHARE_ROUND={'Quarterfinals':'QF','Semifinals':'SF','Bronze match':'Bronze','Final':'Final'};
const SHARE_VERDICT={good:'Comfortable one-day plan',warn:'Finishes after 10 PM',bad:'Runs past midnight'};
// The finish alert's three levels; recalc's alert and the summary both read it.
function finishLevel(cfg,r){
  if(!r.matches) return null;
  if(r.endMin-cfg.startMin>15*60||r.endMin>=24*60) return 'bad';
  return r.endMin%1440>=22*60?'warn':'good';
}
function buildEstimateSummary(cfg,r){
  const dual=cfg.format==='dual';
  const rng=(a,b)=>a===b?String(b):a+'–'+b;
  const cats=r.per.map((p,i)=>{
    if(!p.total) return null;
    const c=cfg.cats[i];
    const ct=catTime(p,cfg);
    const mins=(ct.rr||0)+(ct.po||0)+(ct.rr&&ct.po?cfg.buf:0);
    const k={code:catCode(c),name:catName(c),format:'',playoff:'',count:rng(p.totalMin,p.total),split:'',time:mins?fmtDur(mins):'—'};
    const rounds=()=>p.plan.rounds.map(rd=>SHARE_ROUND[rd.label]||rd.label.replace(/^Round of (\d+)$/,'R$1')).join(', ');
    if(p.dual){
      const br=p.groups<=1?'1 bracket':p.groups+' brackets ('+p.aSizes.map((n,j)=>n+'v'+p.bSizes[j]).join(', ')+')';
      k.format=p.po?`${p.a} v ${p.b} pairs · ${br} · cross-club group stage`:`${p.a} v ${p.b} pairs · one match decides it`;
      if(p.po){k.playoff='Playoffs: '+rounds();k.split=`${p.rr} group + ${p.po} PO`;}
      return k;
    }
    if(p.plan.bestOf3){
      k.format=`${c.teams} pairs · head-to-head, no round robin`;
      k.playoff='Final: best of 3 (first to 2 wins)';
      return k;
    }
    const grp=p.groups===1?'1 group of '+p.sizes[0]:groupShorthand(p);
    const rrWord=p.games===3?'best-of-3 round robin':p.legs===2?'double round robin':'round robin';
    if(p.plan.rrOnly) k.format=`${c.teams} pairs · ${grp} · ${rrWord} only, standings decide`;
    else if(p.plan.single){
      k.format=`${c.teams} pairs · ${grp} · ${rrWord}`;
      k.playoff={ttb:'Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze',
        final:'Final: #1 v #2 · #3 takes bronze',final_bronze:'Final: #1 v #2 · Bronze: #3 v #4'}[p.plan.solo]||'';
    }else{
      const extra=(p.plan.wc?` + ${p.plan.wc} wildcard${p.plan.wc>1?'s':''}`:'')
        +(p.plan.byes?(p.plan.byes===1?', top seed bye':`, top ${p.plan.byes} seeds bye`):'');
      k.format=`${c.teams} pairs · ${grp} · top ${p.adv} advance${extra}`+(rrWord==='round robin'?'':' · '+rrWord);
      k.playoff='Playoffs: '+rounds();
    }
    if(p.po) k.split=rng(p.games===3?p.meetings*2:p.rr,p.rr)+' RR + '+rng(p.plan.min,p.po)+' PO';
    return k;
  }).filter(Boolean);
  const ranged=r.per.some(p=>p.total!==p.totalMin);
  const courts=cfg.courts===1?'the court is':`all ${cfg.courts} courts are`;
  return {
    title:cfg.title||'Untitled tournament',
    clubs:dual?`${cfg.clubA} vs ${cfg.clubB}`:'',
    date:cfg.date?new Date(cfg.date+'T00:00').toLocaleDateString([], {weekday:'long',month:'long',day:'numeric',year:'numeric'}):'',
    meta:[`Start ${fmtTime(cfg.startMin)}`,`${cfg.courts} court${cfg.courts>1?'s':''}`,`${cfg.dur}-min matches`+(cfg.buf?` + ${cfg.buf}-min buffer`:'')].join(' · '),
    finish:r.matches?fmtTime(r.endMin):'—', level:finishLevel(cfg,r)||'good',
    verdict:SHARE_VERDICT[finishLevel(cfg,r)||'good'],
    pairs:r.per.reduce((s,p,i)=>s+(p.dual?(p.a||0)+(p.b||0):(cfg.cats[i].teams||0)),0),
    matches:r.matches, ranged, slots:r.slots, play:fmtDur(r.playMin), cats,
    foot:`Estimate only: assumes ${courts} in use from ${fmtTime(cfg.startMin)} with ${cfg.dur}-minute matches`
      +(cfg.buf?` and a ${cfg.buf}-minute buffer.`:' and no buffer.')
      +(ranged?' Twice-to-beat finals and best-of-3 matches are counted at full length, so the day ends earlier if they’re decided sooner.':'')
  };
}
function buildEstimateText(s){
  const L=[s.title.toUpperCase()];
  if(s.clubs) L.push(s.clubs);
  L.push('Schedule estimate','');
  if(s.date) L.push(s.date);
  L.push(s.meta,'','Projected finish: '+s.finish,s.verdict,
    `${s.pairs} pairs · ${s.ranged?'up to ':''}${s.matches} matches · ${s.slots} time slot${s.slots===1?'':'s'} · ${s.play} play time`,
    '','CATEGORIES');
  s.cats.forEach(k=>{
    L.push('',k.code+' · '+k.name,k.format);
    if(k.playoff) L.push(k.playoff);
    L.push(`${k.count} match${k.count==='1'?'':'es'}${k.split?` (${k.split})`:''} · ${k.time}`);
  });
  L.push('',s.foot,'S.A.G.E. Tournament Time Calculator');
  return L.join('\n');
}
// The image: 1080 px wide, drawn at 2x, as tall as its categories need.
const SHARE_W=1080, SHARE_SCALE=2;
const SHARE_C={navy:'#14263C',deep:'#0B1826',paper:'#F6F7F2',card:'#FFFFFF',line:'#D9DED2',row:'#EEF1ED',
  green:'#7CB92C',greenDark:'#5C8F1F',muted:'#5B6B74',
  good:['#F1F8E7','#3F6317'],warn:['#FFF6DC','#6B4E0C'],bad:['#FBEAE6','#7C2D1E']};
const SHARE_F={
  kicker:'600 22px "Barlow Condensed", "Arial Narrow", sans-serif',
  label:'600 19px "Barlow Condensed", "Arial Narrow", sans-serif',
  title:'52px "Archivo Black", Impact, sans-serif',
  clock:'92px "Archivo Black", Impact, sans-serif',
  stat:'700 40px "Barlow Condensed", "Arial Narrow", sans-serif',
  code:'700 28px "Barlow Condensed", "Arial Narrow", sans-serif',
  big:'700 32px "Barlow Condensed", "Arial Narrow", sans-serif',
  clubs:'600 26px Inter, Arial, sans-serif', meta:'400 24px Inter, Arial, sans-serif',
  pill:'600 22px Inter, Arial, sans-serif', name:'600 24px Inter, Arial, sans-serif',
  fmt:'400 21px Inter, Arial, sans-serif', small:'400 18px Inter, Arial, sans-serif',
  foot:'400 20px Inter, Arial, sans-serif'};
const SHARE_LOGO=new Image();
// Loaded when the page boots, so the click that saves the image can draw
// without waiting (see saveEstimateImage).
function preloadShareAssets(){
  SHARE_LOGO.src='/assets/logo.png';
  try{
    if(document.fonts&&document.fonts.load)
      [SHARE_F.title,SHARE_F.kicker,SHARE_F.stat,SHARE_F.name,SHARE_F.meta].forEach(f=>document.fonts.load(f).catch(()=>{}));
  }catch(e){}
}
function shareWrap(ctx,text,maxW){
  const lines=[];let cur='';
  String(text).split(' ').forEach(w=>{
    const t=cur?cur+' '+w:w;
    if(cur&&ctx.measureText(t).width>maxW){lines.push(cur);cur=w;}else cur=t;
  });
  if(cur) lines.push(cur);
  return lines;
}
// Lays the image out twice: once to measure its height (draw=false), once to
// paint it. The header band, the table's card and the page behind them are
// filled last, under what is already drawn, once their heights are known.
function layoutEstimate(ctx,s,logo,draw){
  const W=SHARE_W,P=48,IW=W-2*P,C=SHARE_C,F=SHARE_F;
  const put=(t,x,y,f,color,align)=>{if(!draw)return;ctx.font=f;ctx.fillStyle=color;ctx.textAlign=align||'left';ctx.fillText(t,x,y);};
  const box=(x,y,w,h,rad,fill,stroke)=>{if(!draw)return;ctx.beginPath();
    if(ctx.roundRect)ctx.roundRect(x,y,w,h,rad);else ctx.rect(x,y,w,h);
    if(fill){ctx.fillStyle=fill;ctx.fill();}if(stroke){ctx.strokeStyle=stroke;ctx.lineWidth=2;ctx.stroke();}};
  const wrap=(t,f,w)=>{ctx.font=f;return shareWrap(ctx,t,w);};
  const spaced=on=>{if('letterSpacing' in ctx)ctx.letterSpacing=on?'3px':'0px';};
  ctx.textBaseline='alphabetic';
  let y=P;
  // Header: the logo on a white tile, the kicker, title, clubs, date and settings.
  box(P,y,64,64,14,'#FFFFFF');
  if(draw&&logo&&logo.complete&&logo.naturalWidth) ctx.drawImage(logo,P+3,y+3,58,58);
  spaced(true);put('S.A.G.E. · SCHEDULE ESTIMATE',P+84,y+41,F.kicker,C.green);spaced(false);
  y+=64+30;
  wrap(s.title.toUpperCase(),F.title,IW).forEach(l=>{put(l,P,y+50,F.title,C.paper);y+=62;});
  if(s.clubs){put(s.clubs,P,y+30,F.clubs,C.green);y+=44;}
  y+=6;
  wrap([s.date,s.meta].filter(Boolean).join(' · '),F.meta,IW).forEach(l=>{put(l,P,y+24,F.meta,'rgba(246,247,242,0.85)');y+=34;});
  y+=34;
  const bandH=y;
  if(draw){ctx.save();ctx.setLineDash([12,9]);ctx.strokeStyle='rgba(124,185,44,0.5)';ctx.lineWidth=3;
    ctx.beginPath();ctx.moveTo(0,bandH-12);ctx.lineTo(W,bandH-12);ctx.stroke();ctx.restore();}
  y+=32;
  // Projected finish, with the verdict in a pill coloured by its level.
  box(P,y,IW,236,24,C.deep);
  spaced(true);put('PROJECTED FINISH',W/2,y+50,F.kicker,C.green,'center');spaced(false);
  put(s.finish,W/2,y+146,F.clock,C.paper,'center');
  ctx.font=F.pill;const pw=ctx.measureText(s.verdict).width+44,pc=C[s.level]||C.good;
  box(W/2-pw/2,y+170,pw,44,22,pc[0]);put(s.verdict,W/2,y+200,F.pill,pc[1],'center');
  y+=236+20;
  // Four figures.
  const stats=[[String(s.pairs),'PAIRS'],[String(s.matches),s.ranged?'MATCHES, MAX':'MATCHES'],[String(s.slots),'TIME SLOTS'],[s.play,'PLAY TIME']];
  const sw=(IW-36)/4;
  stats.forEach(([v,k],i)=>{const x=P+i*(sw+12);box(x,y,sw,108,16,C.card,C.line);
    put(v,x+sw/2,y+56,F.stat,C.deep,'center');spaced(true);put(k,x+sw/2,y+88,F.label,C.muted,'center');spaced(false);});
  y+=108+28;
  // Categories: code and name, format, playoff | matches (RR + PO) | time.
  const top=y,colT=140,colM=180,left=P+24,textW=IW-48-colT-colM-12,xM=P+IW-24-colT,xT=P+IW-24;
  box(P,y,IW,58,[20,20,0,0],C.deep);
  spaced(true);put('CATEGORIES',left,y+38,F.kicker,C.paper);spaced(false);
  y+=58;
  spaced(true);put('CATEGORY · FORMAT',left,y+34,F.label,C.muted);put('MATCHES',xM,y+34,F.label,C.muted,'right');put('TIME',xT,y+34,F.label,C.muted,'right');spaced(false);
  y+=48;
  s.cats.forEach(k=>{
    if(draw){ctx.fillStyle=C.row;ctx.fillRect(P+2,y,IW-4,2);}
    let ry=y+18;
    ctx.font=F.code;const cw=ctx.measureText(k.code).width+12;
    put(k.code,left,ry+28,F.code,C.greenDark);
    wrap(k.name,F.name,textW-cw).forEach(l=>{put(l,left+cw,ry+27,F.name,C.navy);ry+=34;});
    ry+=2;
    [k.format,k.playoff].filter(Boolean).forEach(t=>wrap(t,F.fmt,textW).forEach(l=>{put(l,left,ry+21,F.fmt,C.muted);ry+=29;}));
    put(k.count,xM,y+48,F.big,C.navy,'right');
    if(k.split) put(k.split,xM,y+76,F.small,C.muted,'right');
    put(k.time,xT,y+48,F.big,C.navy,'right');
    y=Math.max(ry,y+90)+14;
  });
  const tableH=y-top+8;
  y+=8+26;
  // The footnote and the signature line.
  wrap(s.foot,F.foot,IW).forEach(l=>{put(l,P,y+20,F.foot,C.muted);y+=29;});
  y+=22;
  if(draw){ctx.fillStyle=C.line;ctx.fillRect(P,y,IW,2);}
  y+=22;
  spaced(true);put('S.A.G.E. MATCH CONTROL EXPERTS',P,y+20,F.label,C.muted);put('TOURNAMENT TIME CALCULATOR',P+IW,y+20,F.label,C.muted,'right');spaced(false);
  y+=20+P;
  if(draw){
    ctx.save();ctx.globalCompositeOperation='destination-over';
    box(P,top,IW,tableH,20,C.card,C.line);
    ctx.fillStyle=C.navy;ctx.fillRect(0,0,W,bandH);
    ctx.fillStyle=C.paper;ctx.fillRect(0,0,W,y);
    ctx.restore();
  }
  return y;
}
function drawEstimate(s,logo){
  const h=Math.ceil(layoutEstimate(document.createElement('canvas').getContext('2d'),s,logo,false));
  const cv=document.createElement('canvas');cv.width=SHARE_W*SHARE_SCALE;cv.height=h*SHARE_SCALE;
  const ctx=cv.getContext('2d');ctx.scale(SHARE_SCALE,SHARE_SCALE);
  layoutEstimate(ctx,s,logo,true);
  return cv;
}
// Synchronous on purpose (toDataURL, not toBlob): navigator.share must start
// inside the click, and Safari refuses one that starts after an await.
function canvasPngFile(cv,name){
  const d=cv.toDataURL('image/png'),bin=atob(d.slice(d.indexOf(',')+1)),u=new Uint8Array(bin.length);
  for(let i=0;i<bin.length;i++)u[i]=bin.charCodeAt(i);
  return new File([u],name,{type:'image/png'});
}
function downloadFile(blob,name){
  try{
    const a=document.createElement('a');a.href=URL.createObjectURL(blob);a.download=name;
    document.body.appendChild(a);a.click();a.remove();
    setTimeout(()=>URL.revokeObjectURL(a.href),2000);
    ioMsg('Saved '+name,'good');
  }catch(err){ioMsg('Download failed in this viewer — open the page in a regular browser.','bad');}
}
// A phone (a coarse pointer) gets the share sheet, so the image goes straight
// to a chat app; anything else downloads the PNG.
function saveEstimateImage(){
  const cfg=readForm(),r=calcAll(cfg);
  if(!r.matches){ioMsg('Add at least one category first.','bad');return;}
  const name=fileBase(cfg)+'-estimate.png';
  let file;
  try{file=canvasPngFile(drawEstimate(buildEstimateSummary(cfg,r),SHARE_LOGO),name);}
  catch(err){ioMsg('Could not draw the image in this browser. Use Copy text instead.','bad');return;}
  const phone=!!(window.matchMedia&&window.matchMedia('(pointer:coarse)').matches);
  if(phone&&navigator.canShare&&navigator.canShare({files:[file]})){
    navigator.share({files:[file],title:cfg.title||'Schedule estimate'})
      .then(()=>ioMsg('Shared '+name,'good'))
      .catch(err=>{if(err&&err.name==='AbortError')return;downloadFile(file,name);});
    return;
  }
  downloadFile(file,name);
}
function copyEstimateText(){
  const cfg=readForm(),r=calcAll(cfg);
  if(!r.matches){ioMsg('Add at least one category first.','bad');return;}
  const text=buildEstimateText(buildEstimateSummary(cfg,r));
  const done=ok=>ioMsg(ok?'Estimate copied — paste it into any chat.':'Could not reach the clipboard. Use Save image instead.',ok?'good':'bad');
  if(navigator.clipboard&&navigator.clipboard.writeText){
    navigator.clipboard.writeText(text).then(()=>done(true),()=>done(copyFallback(text)));
  }else{
    done(copyFallback(text));
  }
}
```

### E11. The share buttons' events, and the assets loaded at boot

**Insert after** the line containing:

```js
$('xlsxbtn').addEventListener('click',exportXLSX);
```

these lines:

```js
$('shareimg').addEventListener('click',saveEstimateImage);
$('sharetext').addEventListener('click',copyEstimateText);
preloadShareAssets();
```

## 6. The test

Create `_tests/unit/tools/tournament-calculator-share.test.mjs` (a new folder) with exactly this
content. It pulls the summary, the text, `finishLevel`, `defaultCat` and the calculator's maths
out of the page with the harness's own extractor and runs them on the two Paddle Aces plans, a
dual meet and the other playoff shapes. The suite's `npm test` picks it up by its glob
(`unit/**/*.test.mjs`); nothing else needs registering.

```js
// The Tournament Calculator's shared estimate and its category presets
// (sage-docs/docs/specs/.../calculator-share-estimate-spec.md). The summary and
// its text are pure: they are pulled out of the page and run here against the
// two Paddle Aces day plans, a dual meet and the edge cases.
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import path from 'node:path';
import { SITE_ROOT } from '../../helpers/paths.mjs';
import { extractFunction, extractConst } from '../../helpers/extract.mjs';

const page = fs.readFileSync(path.join(SITE_ROOT, 'tools/tournament-calculator.html'), 'utf8');
const consts = ['rrMatches', 'roundLabel', 'PRESETS', 'catCode', 'catName', 'SHARE_ROUND', 'SHARE_VERDICT']
  .map(n => extractConst(page, n).text);
const fns = ['groupSizes', 'reduceToTarget', 'solveForTarget', 'tieredRounds', 'playoffPlan', 'splitEven',
  'dualGroupStage', 'dualPlayoffPlan', 'calcDualCategory', 'calcCategory', 'rrRoundFloor', 'seqParts', 'calcAll',
  'catTime', 'fmtTime', 'fmtDur', 'groupShorthand', 'defaultCat', 'finishLevel', 'buildEstimateSummary',
  'buildEstimateText'].map(n => extractFunction(page, n));
const lib = new Function('let cats=[],nextId=1;\n' + [...consts, ...fns].join('\n')
  + '\nreturn { PRESETS, catName, defaultCat, calcAll, finishLevel, buildEstimateSummary, buildEstimateText,'
  + ' setCats: c => { cats = c; } };')();

// A category as Import CSV makes it, and a plan around it.
const cat = (code, teams, group, o = {}) => ({ code, name: '', teams, group, adv: 2, fill: 'bye', soloFormat: 'ttb',
  rrLegs: 1, rrBestOf: 1, groupDual: 1, teamsA: 5, teamsB: 5, poFormat: 'semis', ...o });
const plan = (o, cats) => ({ title: '', date: '', start: '10:00', startMin: 600, courts: 4, dur: 25, buf: 0,
  format: 'standard', clubA: 'Club A', clubB: 'Club B', ...o, cats });
const estimate = cfg => lib.buildEstimateSummary(cfg, lib.calcAll(cfg));
const longDate = iso => new Date(iso + 'T00:00').toLocaleDateString([], { weekday: 'long', month: 'long', day: 'numeric', year: 'numeric' });
const TTB = 'Final: twice-to-beat (#1 needs one win, #2 must win twice) · #3 takes bronze';
const FOOT_TTB = ' Twice-to-beat finals and best-of-3 matches are counted at full length, so the day ends earlier if they’re decided sooner.';

test('Paddle Aces Day 1 (28 Nov): the whole message', () => {
  const cfg = plan({ title: 'Paddle Aces The Ultimate Generation Cup DAY 1', date: '2026-11-28', courts: 10 }, [
    cat('NMD', 13, 4), cat('NWD', 8, 2), cat('NXD', 12, 3), cat('LIMD', 14, 4), cat('LIWD', 9, 2),
    cat('LIXD', 10, 2), cat('HIMD', 7, 2), cat('HIXD', 4, 1), cat('AMD', 4, 1), cat('AXD', 4, 1)]);
  const s = estimate(cfg);
  assert.deepEqual([s.finish, s.level, s.pairs, s.matches, s.ranged, s.slots, s.play],
    ['5:05 PM', 'good', 85, 170, true, 17, '7h 5m']);
  assert.equal(lib.buildEstimateText(s), [
    'PADDLE ACES THE ULTIMATE GENERATION CUP DAY 1', 'Schedule estimate', '',
    longDate('2026-11-28'), 'Start 10:00 AM · 10 courts · 25-min matches', '',
    'Projected finish: 5:05 PM', 'Comfortable one-day plan',
    '85 pairs · up to 170 matches · 17 time slots · 7h 5m play time', '', 'CATEGORIES', '',
    "NMD · Novice Men's Doubles", '13 pairs · 4 groups (4-3-3-3) · top 2 advance', 'Playoffs: QF, SF, Bronze, Final',
    '23 matches (15 RR + 8 PO) · 2h 30m', '',
    "NWD · Novice Women's Doubles", '8 pairs · 2 groups (4-4) · top 2 advance', 'Playoffs: SF, Bronze, Final',
    '16 matches (12 RR + 4 PO) · 2h 5m', '',
    'NXD · Novice Mixed Doubles', '12 pairs · 3 groups (4-4-4) · top 2 advance, top seed bye',
    'Playoffs: R16, QF, SF, Bronze, Final', '24 matches (18 RR + 6 PO) · 2h 55m', '',
    "LIMD · Low Intermediate Men's Doubles", '14 pairs · 4 groups (4-4-3-3) · top 2 advance',
    'Playoffs: QF, SF, Bronze, Final', '26 matches (18 RR + 8 PO) · 2h 30m', '',
    "LIWD · Low Intermediate Women's Doubles", '9 pairs · 2 groups (5-4) · top 2 advance',
    'Playoffs: SF, Bronze, Final', '20 matches (16 RR + 4 PO) · 2h 55m', '',
    'LIXD · Low Intermediate Mixed Doubles', '10 pairs · 2 groups (5-5) · top 2 advance',
    'Playoffs: SF, Bronze, Final', '24 matches (20 RR + 4 PO) · 2h 55m', '',
    "HIMD · High Intermediate Men's Doubles", '7 pairs · 2 groups (4-3) · top 2 advance',
    'Playoffs: SF, Bronze, Final', '13 matches (9 RR + 4 PO) · 2h 5m', '',
    'HIXD · High Intermediate Mixed Doubles', '4 pairs · 1 group of 4 · round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 2h 5m', '',
    "AMD · Advanced Men's Doubles", '4 pairs · 1 group of 4 · round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 2h 5m', '',
    'AXD · Advanced Mixed Doubles', '4 pairs · 1 group of 4 · round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 2h 5m', '',
    'Estimate only: assumes all 10 courts are in use from 10:00 AM with 25-minute matches and no buffer.' + FOOT_TTB,
    'S.A.G.E. Tournament Time Calculator'].join('\n'));
});

test("Paddle Aces Day 2 (29 Nov): Open Women's Doubles is a preset, double round robins", () => {
  const cfg = plan({ title: 'Paddle Aces The Ultimate Generation Cup DAY 2', date: '2026-11-29', courts: 6 }, [
    cat('NMD', 4, 1), cat('LIMD', 11, 3), cat('LIXD', 4, 1),
    cat('AMD', 3, 1, { rrLegs: 2 }), cat('OWD', 3, 1, { rrLegs: 2 })]);
  const s = estimate(cfg);
  assert.deepEqual([s.finish, s.pairs, s.matches, s.slots, s.play], ['1:45 PM', 25, 53, 9, '3h 45m']);
  assert.equal(lib.buildEstimateText(s), [
    'PADDLE ACES THE ULTIMATE GENERATION CUP DAY 2', 'Schedule estimate', '',
    longDate('2026-11-29'), 'Start 10:00 AM · 6 courts · 25-min matches', '',
    'Projected finish: 1:45 PM', 'Comfortable one-day plan',
    '25 pairs · up to 53 matches · 9 time slots · 3h 45m play time', '', 'CATEGORIES', '',
    "NMD · Novice Men's Doubles", '4 pairs · 1 group of 4 · round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 2h 5m', '',
    "LIMD · Low Intermediate Men's Doubles", '11 pairs · 3 groups (4-4-3) · top 2 advance, top seed bye',
    'Playoffs: R16, QF, SF, Bronze, Final', '21 matches (15 RR + 6 PO) · 2h 55m', '',
    'LIXD · Low Intermediate Mixed Doubles', '4 pairs · 1 group of 4 · round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 2h 5m', '',
    "AMD · Advanced Men's Doubles", '3 pairs · 1 group of 3 · double round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 3h 20m', '',
    "OWD · Open Women's Doubles", '3 pairs · 1 group of 3 · double round robin', TTB,
    '7–8 matches (6 RR + 1–2 PO) · 3h 20m', '',
    'Estimate only: assumes all 6 courts are in use from 10:00 AM with 25-minute matches and no buffer.' + FOOT_TTB,
    'S.A.G.E. Tournament Time Calculator'].join('\n'));
});

test('the other playoff shapes, a buffer, and one court', () => {
  const s = estimate(plan({ courts: 1, buf: 5, startMin: 540 }, [
    cat('BWD', 2, 1),
    cat('OXD', 5, 1, { soloFormat: 'final_bronze' }),
    cat('BMD', 4, 1, { soloFormat: 'final' }),
    cat('BXD', 3, 1, { soloFormat: 'rr' }),
    cat('OPEN', 8, 2, { rrBestOf: 3 })]));
  const [h2h, fb, fin, rr, bo3] = s.cats;
  assert.deepEqual([h2h.name, h2h.format, h2h.playoff, h2h.count, h2h.split],
    ["Beginner Women's Doubles", '2 pairs · head-to-head, no round robin', 'Final: best of 3 (first to 2 wins)', '2–3', '']);
  assert.deepEqual([fb.name, fb.playoff, fb.count, fb.split],
    ['Open Mixed Doubles', 'Final: #1 v #2 · Bronze: #3 v #4', '12', '10 RR + 2 PO']);
  assert.deepEqual([fin.playoff, fin.count], ['Final: #1 v #2 · #3 takes bronze', '7']);
  assert.deepEqual([rr.format, rr.playoff, rr.count, rr.split],
    ['3 pairs · 1 group of 3 · round robin only, standings decide', '', '3', '']);
  assert.deepEqual([bo3.format, bo3.count, bo3.split],
    ['8 pairs · 2 groups (4-4) · top 2 advance · best-of-3 round robin', '28–40', '24–36 RR + 4 PO']);
  assert.equal(s.meta, 'Start 9:00 AM · 1 court · 25-min matches + 5-min buffer');
  assert.match(s.foot, /^Estimate only: assumes the court is in use from 9:00 AM with 25-minute matches and a 5-minute buffer\. Twice/);
  assert.equal(s.title, 'Untitled tournament');
  assert.equal(s.date, '');
  assert.ok(!lib.buildEstimateText(s).includes('\n\n\n'), 'no date line, and no doubled blank line');
});

test('a dual meet: the clubs under the title, pairs per club, cross-club group stage', () => {
  const s = estimate(plan({ format: 'dual', clubA: 'PNF', clubB: 'BUP' }, [
    cat('NMD', 10, 4, { teamsA: 5, teamsB: 5, groupDual: 1 }),
    cat('LIMD', 10, 4, { teamsA: 6, teamsB: 6, groupDual: 2 }),
    cat('HIMD', 2, 4, { teamsA: 1, teamsB: 1, groupDual: 1 })]));
  assert.equal(s.clubs, 'PNF vs BUP');
  assert.equal(s.pairs, 24);
  assert.deepEqual(s.cats.map(k => [k.format, k.playoff, k.count, k.split]), [
    ['5 v 5 pairs · 1 bracket · cross-club group stage', 'Playoffs: Bronze, Final', '27', '25 group + 2 PO'],
    ['6 v 6 pairs · 2 brackets (3v3, 3v3) · cross-club group stage', 'Playoffs: Club playoff, Bronze, Final', '22', '18 group + 4 PO'],
    ['1 v 1 pairs · one match decides it', '', '1', '']]);
  assert.deepEqual(lib.buildEstimateText(s).split('\n').slice(0, 3), ['UNTITLED TOURNAMENT', 'PNF vs BUP', 'Schedule estimate']);
  assert.ok(lib.buildEstimateText(s).includes('\n1 match · '));
});

test('finishLevel: the three alert levels', () => {
  const at = (startMin, endMin) => lib.finishLevel({ startMin }, { matches: 1, endMin });
  assert.equal(at(540, 21 * 60 + 59), 'good');
  assert.equal(at(540, 22 * 60), 'warn');
  assert.equal(at(540, 24 * 60), 'bad');
  assert.equal(at(300, 300 + 15 * 60 + 5), 'bad');
  assert.equal(lib.finishLevel({ startMin: 540 }, { matches: 0, endMin: null }), null);
});

test('presets: Beginner first, the three Opens, and a new category still starts from Novice', () => {
  const codes = lib.PRESETS.map(p => p[0]);
  assert.deepEqual(codes, ['BWD', 'BMD', 'BXD', 'NWD', 'NMD', 'NXD', 'LIWD', 'LIMD', 'LIXD', 'IWD', 'IMD', 'IXD',
    'HIWD', 'HIMD', 'HIXD', 'AWD', 'AMD', 'AXD', 'OWD', 'OPEN', 'OXD', 'CUSTOM']);
  assert.deepEqual(['BWD', 'BMD', 'BXD', 'OWD', 'OPEN', 'OXD'].map(code => lib.catName({ code })), [
    "Beginner Women's Doubles", "Beginner Men's Doubles", 'Beginner Mixed Doubles',
    "Open Women's Doubles", "Open Men's Doubles", 'Open Mixed Doubles']);
  lib.setCats([]);
  assert.equal(lib.defaultCat().code, 'NWD');
  lib.setCats(codes.slice(3, -1).map(code => ({ code })));
  assert.equal(lib.defaultCat().code, 'BWD');
  lib.setCats(codes.slice(0, -1).map(code => ({ code })));
  assert.equal(lib.defaultCat().code, 'BWD', 'every preset used: the first one, as before');
});
```

Run it, then the whole suite:

```bash
cd /d/Personal/SAGE/sage-match-control.github.io/_tests
npm install
node --test unit/tools/tournament-calculator-share.test.mjs
npm test
```

The first prints `# pass 6` and `# fail 0`; the second ends with `# fail 0`. If a test fails,
the page's code differs from §5: compare it with §5 rather than editing the test. Never weaken
a test to make it pass.

## 7. Browser checks

Serve the site from the workspace and open the calculator:

```bash
cd /d/Personal/SAGE && npx --yes http-server sage-match-control.github.io -p 8000 -c-1
```

Open `http://localhost:8000/tools/tournament-calculator.html`. Clear the saved plan first
(DevTools console: `localStorage.removeItem('ttc-config')`, then reload).

Save these two plans as files to import (they are the owner's Paddle Aces exports):

`paddle-aces-day-1.csv`:

```text
# Tournament Time Calculator export — use Import CSV to restore this setup
# 170 matches · projected finish 5:05 PM
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs,rr_best_of
setting,title,Paddle Aces The Ultimate Generation Cup DAY 1,,,,
setting,date,2026-11-28,,,,
setting,start,10:00,,,,
setting,courts,10,,,,
setting,duration_min,25,,,,
setting,buffer_min,0,,,,
setting,format,standard,,,,
setting,club_a,Club A,,,,
setting,club_b,Club B,,,,
category,NMD,Novice Men's Doubles,13,4,2,bye,,2,1,,,,
category,NWD,Novice Women's Doubles,8,2,2,bye,,3,2,,,,
category,NXD,Novice Mixed Doubles,12,3,2,bye,,2,2,,,,
category,LIMD,Low Intermediate Men's Doubles,14,4,2,bye,,2,2,,,,
category,LIWD,Low Intermediate Women's Doubles,9,2,2,bye,,2,2,,,,
category,LIXD,Low Intermediate Mixed Doubles,10,2,2,bye,,2,2,,,,
category,HIMD,High Intermediate Men's Doubles,7,2,2,bye,,2,1,,,,
category,HIXD,High Intermediate Mixed Doubles,4,1,2,bye,,2,1,,ttb,,
category,AMD,Advanced Men's Doubles,4,1,2,bye,,5,5,,ttb,,
category,AXD,Advanced Mixed Doubles,4,1,2,bye,,5,5,,ttb,,
```

`paddle-aces-day-2.csv`:

```text
# Tournament Time Calculator export — use Import CSV to restore this setup
# 53 matches · projected finish 1:45 PM
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format,rr_legs,rr_best_of
setting,title,Paddle Aces The Ultimate Generation Cup DAY 2,,,,
setting,date,2026-11-29,,,,
setting,start,10:00,,,,
setting,courts,6,,,,
setting,duration_min,25,,,,
setting,buffer_min,0,,,,
setting,format,standard,,,,
setting,club_a,Club A,,,,
setting,club_b,Club B,,,,
category,NMD,Novice Men's Doubles,4,1,2,bye,,2,1,,ttb,,
category,LIMD,Low Intermediate Men's Doubles,11,3,2,bye,,2,2,,,,
category,LIXD,Low Intermediate Mixed Doubles,4,1,2,bye,,2,2,,ttb,,
category,AMD,Advanced Men's Doubles,3,1,2,bye,,5,5,,ttb,2,
category,OWD,Open Women's Doubles,3,1,2,bye,,5,5,,ttb,2,
```

- [ ] **Presets.** The fresh plan's one category is `NWD`. Its menu lists `BWD`, `BMD`, `BXD`
  first and `OWD — Open Women's Doubles`, `OPEN — Open Men's Doubles`,
  `OXD — Open Mixed Doubles` just above *Custom…*. **+ Add category** adds `NMD`, then `NXD`.
- [ ] **The share box** sits between the breakdown panel and the Reset / Import CSV buttons,
  with both buttons enabled. Remove every category: both buttons grey out and the help line
  reads *Add a category first.* Add one back: they enable again.
- [ ] **Day 2 import.** Import `paddle-aces-day-2.csv`: the last category's menu shows
  `OWD — Open Women's Doubles` (a preset, no custom code fields), and the page reads **1:45 PM**,
  53 matches, 9 time slots, 3h 45m.
- [ ] **Copy text.** Click it: the message strip says *Estimate copied — paste it into any
  chat.* Paste into a text editor: it is §4.1's message, character for character (the date
  line in your browser's locale).
- [ ] **Save image.** Click it: `paddle-aces-the-ultimate-generation-cup-day-2-2026-11-29-estimate.png`
  downloads and the strip says *Saved …*. Open it: 2160 px wide, the layout of §3, the five
  categories, the green *Comfortable one-day plan* pill, *MATCHES, MAX* under 53, and real
  fonts (blocky Archivo Black title, narrow Barlow Condensed figures), not a fallback serif.
- [ ] **Day 1.** Import `paddle-aces-day-1.csv` and Save image: **5:05 PM**, ten category rows,
  85 · 170 · 17 · 7h 5m, the three twice-to-beat rows reading *7–8* over *6 RR + 1–2 PO*, and
  the long title wrapped onto two lines inside the band.
- [ ] **The verdict follows the alert.** With Day 1 loaded, set the start to 20:00: the finish
  reads 3:05 AM (+1d), the page's alert turns red (*runs past midnight*), and a new image's pill
  reads *Runs past midnight* in red. Set it to 15:00: 10:05 PM, the amber alert, and the pill
  *Finishes after 10 PM*.
- [ ] **Dual meet.** Click **Reset**, choose *Dual meet*, and set the clubs to `PNF` and `BUP`.
  Copy text's first two lines are `UNTITLED TOURNAMENT` and `PNF vs BUP`, and the category reads
  *5 v 5 pairs · 1 bracket · cross-club group stage*, *Playoffs: Bronze, Final*, *27 matches
  (25 group + 2 PO)*; the image shows *PNF vs BUP* in green under the title.
- [ ] **Phone.** In DevTools' device toolbar at 375 px: the two buttons sit side by side, and
  the page doesn't scroll sideways. (The share sheet itself needs a real phone: on one, Save
  image opens it with the PNG attached. Check this once on an Android or iPhone after the
  change is live, if the owner wants it checked before an event.)
- [ ] **No console errors**, at desktop width and at 375 px, through every step above.
- [ ] **Offline still works**: in DevTools → Application the service worker is registered;
  with *Offline* ticked, a reload still shows the calculator, and Save image still draws.

## 8. Documentation (`sage-docs`)

Make these after the site change is live (§0.3 step 8). Write in the present tense: what the
calculator does, not what changed.

1. **`docs/features/tournament-calculator.md`.**
   - Add a section `## Categories` before `## How the round robin is played`, saying in plain
     words: each category is picked from a menu of presets, Beginner, Novice, Low Intermediate,
     Intermediate, High Intermediate, Advanced and Open, each as Women's, Men's and Mixed
     Doubles (with their codes, as §2's table, noting Open Men's Doubles is `OPEN`); anything
     else is *Custom…*, with its own code and name; a new category starts from Novice.
   - Add a section `## Sharing the estimate` after `## Exporting`: the box and its two buttons;
     what the image and the message hold (§3, §4: the finish, the verdict, the four figures,
     each category's format, playoff, matches and time, the footnote) and what they leave out
     (warnings, notes, playoff charts); a phone gets the share sheet and a computer a download;
     that *up to* and *MATCHES, MAX* mean twice-to-beat finals and best-of-3 matches are
     counted at full length.
2. **`docs/technical/tournament-calculator.md`.** Add a section `## Sharing the estimate` before
   `## PWA (installable, offline)`: `buildEstimateSummary` as the one source for
   `buildEstimateText` and `layoutEstimate` (§4.2); `finishLevel`, shared with the page's
   finish alert; the two-pass canvas layout and the deferred backgrounds (§3); why the click
   draws and encodes synchronously, and `preloadShareAssets` (§3); the coarse-pointer test for
   the share sheet; and the test file. Add a paragraph on the presets to it as well: `PRESETS`'
   order, `defaultCat`'s rotation from `NWD`, Import CSV turning a known code into the preset,
   and the Dual Meet Sheet Generator's `categoryColor_` leaving `B…` and `O…` levels without a
   tab colour. Add this spec to the page's closing **Specs:** line.
3. **`docs/usage/plan-the-event.md`.** After step 6 (*Check the finish time…*), add a step:
   *To send the estimate to the organiser, use **Save image** or **Copy text** under the
   breakdown. On a phone, Save image opens the share sheet.* Renumber the step after it.
4. **This spec moves to `implemented/`**, by the procedure in `docs/specs/README.md`, *Moving a
   spec between folders*: its status line (built, with the site commit), the specs index and
   the two folder READMEs, `mkdocs.yml`'s nav, and every real link to it
   (`grep -rn "calculator-share-estimate-spec.md" docs/`). Add the spec under **Tournament
   Calculator** in the index's **Implemented** section.
5. **Changelog and version**: a MINOR entry at the top of `docs/changelog.md` (`## [x.y.z] —
   YYYY-MM-DD`, Manila date) under **Changed**, and the same version in `mkdocs.yml`'s
   `copyright` line.

## 9. Decisions

Taken by the owner on 2026-11-21, after a mockup and samples of both outputs:

| # | Question | Answer |
| --- | --- | --- |
| D1 | Where do the buttons go? | **A box of their own between the breakdown and the file buttons**, since they are for someone else, not the operator's files |
| D2 | What does the image hold? | **The finish, its verdict, pairs / matches / time slots / play time, and per category its format, playoff, matches with the RR + PO split, and time.** No warnings (they are the operator's), no category notes, no playoff charts, no RR time / PO time split |
| D3 | How does the text read? | **Plain lines, one block per category**: no table, no `*bold*`, no emoji, so it reads the same in every chat app |
| D4 | Range counts | **The total says *up to* (image: *MATCHES, MAX*)**, since the calculator's total counts twice-to-beat finals and best-of-3 at full length while a category reads *7–8* |
| D5 | Round names | **The calculator's own, shortened** (*R16, QF, SF, Bronze, Final*), even where a six-qualifier ladder calls its first round *Round of 16*, so the share matches the page and the workbook |
| D6 | Phone and computer | **A phone gets the share sheet, a computer downloads.** No *Copy image* button in this version |
| D7 | Presets | **Beginner WD/MD/XD, and Open WD and XD beside Open Men's `OPEN`**, which keeps its code. A new category still starts from Novice |

## 10. Out of scope

- A *Copy image* button, a PDF, or sending the estimate anywhere from the page itself.
- The team tournament format (the [calculator team format](calculator-team-format-spec.md),
  not built): whichever of the two is built second gives the share box a rule for it (§0.2).
- Category notes, warnings or playoff charts in the image or the text.
- Any change to the maths, the CSV or `.xlsx` formats, the generator handoff, `tools/sw.js`,
  the generators (including tab colours for Beginner and Open), the event sites,
  `sage-tools-api` or `event-data`.

---
**Related:** [Tournament Calculator usage](../../features/tournament-calculator.md) ·
[technical](../../technical/tournament-calculator.md) ·
[Calculator team format](calculator-team-format-spec.md) ·
[Double round robin and finals](../in-progress/double-round-robin-and-finals-spec.md)
