# Spec — Tournament Calculator: team tournament format

> **Status: not started.** Nothing here is built. Written 2026-10-05 after the first
> team event,
> [PickleDrive Club One Year Celebration](../implemented/pickledrive-club-anniversary-team-tournament-spec.md),
> and rewritten 2026-11-12 against
> `sage-match-control.github.io/tools/tournament-calculator.html` at `628412c`, the two
> team events since,
> [Millennium Invitational](../implemented/millennium-invitational-spec.md) (24 October)
> and [Ladies' Dink](../implemented/ladies-dink-club-tournament-spec.md) (7 November), and
> the draft plan of a two-day team event on 5–6 December. The owner answered every
> question on 2026-11-12 (§10).
>
> **Dry-run on 2026-11-12.** Every edit in §5 was applied, exactly as written, to a copy
> of the calculator at `628412c`; every anchor matched once. §6's test file passed
> (9 tests) against that copy, and §9's browser checks were made on it at desktop width
> and 375 px with no console errors.

Add a third option to the Tournament Calculator's **Format** menu, **Team
tournament**, so a team event can be planned the way a standard tournament or a dual
meet already is: how many matchups and matches, how many time slots, and what time
the last match finishes, before the draw is final.

---

## 0. How to use this spec

You can carry this out with no other context: every value, every edit, every test and
every check is here. Read §0 to §4 before you change anything.

### 0.1 Where things are

The workspace is `D:\Personal\SAGE`. It holds several independent git repos; two
change here:

| Repo | Path | What changes |
| --- | --- | --- |
| The site | `D:\Personal\SAGE\sage-match-control.github.io` | `tools/tournament-calculator.html` (§5) and one new test file, `_tests/unit/tools/tournament-calculator-team.test.mjs` (§6) |
| The docs | `D:\Personal\SAGE\sage-docs` | three docs pages, this spec's status, the changelog and `mkdocs.yml` (§8) |

The calculator is one self-contained static page: inline `<style>`, one classic inline
`<script>` (not a module), SheetJS from cdnjs for `.xlsx`, no build step, no framework.
The site's `_tests/` folder is the only Node package (dev-only, never published).
Nothing in `sage-tools-api` or `event-data` changes.

### 0.2 Rules

- **Find edits by their anchor text, not line numbers.** Every edit in §5 quotes an
  exact piece of text that occurs **once** in the file. If an anchor isn't there, or
  is there more than once, **stop and report**. Don't guess.
- **Apply §5's edits in order, exactly as written.** The code was dry-run (status
  block). Don't restyle, rename or "improve" it, and don't make design decisions: if
  this spec seems wrong or two parts disagree, **stop and ask** the owner.
- **Keep the file's style and line endings.** The page is written compactly: 2-space
  indent, single quotes, semicolons, short names. The working copy has CRLF line
  endings (git stores LF); keep them. Check with `git ls-files --eol tools/tournament-calculator.html`:
  it should still read `i/lf w/crlf` afterwards, and `git diff` should show only your
  lines.
- **Don't touch** the standard and dual-meet maths (`calcCategory`, `calcDualCategory`,
  `playoffPlan`, `seqParts`, `catTime`), their category cards, `PRESETS`, `buildChart`,
  the CSV header line, `GENERATOR_MASTERS`' two entries, or `tools/sw.js`. No new
  dependency anywhere.
- **Use Git Bash** for the commands (Windows; the paths above are the same in Git Bash
  as `/d/Personal/SAGE/...`).
- **Commit, but never push or merge without the owner's go-ahead.** The site deploys to
  GitHub Pages on push to `main`, and takes no merge within 3 days of an event (check
  the dates in `event-data/config/events.json`). End each commit message with the
  attribution line the session gives you.

### 0.3 Order of work

1. In the site repo: `git checkout main && git pull && git checkout -b calculator-team-format`.
2. Apply §5's edits E1–E29 to `tools/tournament-calculator.html`, in order.
3. Add §6's test file. Run the site's tests (§6): all pass, the new ones included.
4. Serve the site and make §9's checks in a browser.
5. Commit the site change: `Tournament Calculator: team tournament format`.
6. Report to the owner and wait for the go-ahead to merge and push.
7. Once the site change is live, make §8's docs changes in `sage-docs` on a branch,
   commit them as `docs 2.62.0: the calculator's team format` (or the next MINOR
   version if 2.62.0 is taken), and wait for the go-ahead to push.

### 0.4 Words used

| Word | Meaning |
| --- | --- |
| **matchup** | One team meeting another: *P* matches, one per pair of the two teams |
| ***P*** | Matches per matchup |
| ***S*** | Matches on court at once: how many of a matchup's matches share a time slot |
| **slot** | One match's time, `duration + buffer` minutes |
| **lane** | The *S* courts one matchup holds. `lanes = floor(courts / S)` matchups run at once |
| **block** | The slots one matchup holds its lane: `ceil(P / S)` |
| **groupBlock** | A group-stage matchup's block: one more when every matchup has a tiebreaker |
| ***R*** | The biggest group's number of rounds: *n − 1* for an even *n*, *n* for an odd one |
| **tiebreaker** | One more game, played only when a matchup's games are level (`TB`) |

## 1. Why the model differs from the other two formats

| | Standard | Dual meet | **Team** |
| --- | --- | --- | --- |
| Unit that plays | a pair, in a category | a pair vs a pair from the other club | a **team** meets a team: one **matchup** of several matches |
| Categories | many, each with its own count and brackets | many | **none**: one competition |
| Group stage | round robin inside a bracket, one match per pairing | cross-club, one match per pairing | round robin inside a group, one **matchup** per pairing, each *P* matches |
| Playoffs | per category, from the top *k* of each group | per category | one shared playoff, from a Round of 16 down to the Bronze and Final |
| A drawn result | not possible | not possible | a matchup of an even number of matches can end level; it may then play a **tiebreaker game** |
| What limits the clock | courts, and that a pair can't be on two courts | the same | courts, **and** that a team plays one matchup at a time, which takes several slots |

So the team format is **one competition with no category cards**, worked out in
matchups rather than matches, by its own function `calcTeam`. The category cards stay
in memory (and in the saved plan) while the team format is shown, so switching format
and back loses nothing.

### 1.1 The team events so far

The model is built from these, and each is a test (§4, §6).

| | PickleDrive (3 Oct) | Millennium (24 Oct) | Ladies' Dink (7 Nov) |
| --- | --- | --- | --- |
| Teams, groups | 15 in 3 groups of 5 | 8 in 1 group | 16 in 4 groups of 4 |
| Matches per matchup | 4 | 4 | **2** |
| On court at once per matchup | 2 (pairs 1–2, then 3–4) | 2 in the group stage; all 4 in the Bronze and Final | 2; the quarterfinals one at a time, each on its own court |
| Courts, slot | 10, 25 min | 8, 30 min | 6, 25 min |
| Playoff | QF, SF, Bronze + Final (8 teams), first published as SF, Bronze + Final (4 teams) | **Bronze + Final straight from the standings** (4 teams, no semifinals) | QF, SF, Bronze + Final (top 2 of each group) |
| Bronze and Final | at the same time | **one after the other** (8:00 and 8:30 PM) | at the same time |
| Tiebreaker game | no | no | **yes**: a playoff matchup at 1–1 plays one more game, in its own slot after each round |
| Before the playoffs | one 25-min slot free | none | one 25-min slot, kept for group-stage tie games |
| Matches | 136 (SF plan), 152 (QF plan) | 120 | 72, of which 8 are tiebreakers played only if needed |
| Finish | 4:05 PM (SF plan) | 9:00 PM | 9:40 PM |

The December draft adds a Round of 16 (16 teams: the top 2 of 8 groups) and a
tiebreaker after any level matchup, group stage included (§4).

Scoring (points or games won), ranking and tie rules, `_TB` row codes, lettered
brackets, playoff seeding, lineup rules and lineup reveal are the workbook's and the
event site's. None of them changes the number of matches or slots, so none of them is
a calculator input (§11).

## 2. The model

### 2.1 Inputs

The team panel's inputs (stored in the page's `team` object, cleaned by `cleanTeam`):

| Input | Field | Default | Range | Meaning |
| --- | --- | --- | --- | --- |
| Teams, *T* | `teams` | 15 | 2–100 | |
| Groups, *G* | `groups` | 3 | 1–30 | Split as evenly as possible by the existing `groupSizes(T, G)`, bigger groups first, every group at least 2 (15 into 3 is 5-5-5) |
| Matches per matchup, *P* | `perMatchup` | 4 | 1–8 | One match per pair |
| Matches on court at once, *S* | `perSlot` | 2 | 1–*P* | Matches that share a player can't share a slot: in a 3-match matchup whose third match is played by players from the first two, *S* is 2. The calculator doesn't know rosters, so this is a hint under the inputs, never a check |
| First playoff round | `playoff` | `qf` | §2.2 | Every playoff ends in a Bronze and a Final. The organiser chooses who advances; the calculator only counts |
| Tiebreaker game | `tiebreaker` | `off` | `off`, `playoffs`, `all` | **Playoffs**: every playoff matchup keeps a tiebreaker. **Every matchup** (`all`): group matchups too. Always `off` while *P* is odd |
| Break before the playoffs | `breakOn`, `breakMin` | ticked, 25 | 0–240 min | Idle time between the last group slot and the first playoff slot. Unticking it keeps the minutes for when it's ticked again |

The existing **Courts**, **Match duration**, **Buffer** and **Start time** inputs are
shared with the other formats.

### 2.2 Derived

- **Group matchups** = Σ over groups of *n(n−1)/2* (the existing `rrMatches`).
  **Group matches** = that × *P*.
- **First playoff round**, its qualifiers *Q* and its rounds (`TEAM_PLAYOFFS`):

  | First round | Value | *Q* | Rounds, in order (matchups) |
  | --- | --- | --- | --- |
  | Round of 16 | `r16` | 16 | Round of 16 8 · Quarterfinals 4 · Semifinals 2 · Bronze + Final 2 |
  | Quarterfinals | `qf` | 8 | Quarterfinals 4 · Semifinals 2 · Bronze + Final 2 |
  | Semifinals | `sf` | 4 | Semifinals 2 · Bronze + Final 2 |
  | Bronze + Final | `final_bronze` | 4 | Bronze + Final 2 (1st v 2nd, 3rd v 4th, from the standings) |

  **Bronze and Final are one round**, played at the same time. When *T* < *Q*, the plan
  starts at the largest round that fits; below 4 teams there is no playoff (and so no
  break). **Playoff matches** = playoff matchups × *P*.
- **Tiebreakers**: one per matchup that keeps one (every playoff matchup with
  `playoffs`, every matchup with `all`). **Matches** counts them, as the most that can
  be played, and says how many are tiebreakers.

### 2.3 Time

All in slots of `dur + buf` minutes:

- **Group stage** = `max( ceil(groupMatchups / lanes), R ) × groupBlock`. The courts
  limit it to `ceil(groupMatchups / lanes)` waves; a team plays one matchup at a time,
  so a group needs at least *R* waves. Groups run side by side.
- **Each playoff round** = `ceil(matchupsInRound / lanes) × block`, **plus one slot**
  when the tiebreaker is on: a tiebreaker starts only after its matchup's games, so the
  last wave's tiebreakers need a slot of their own. Rounds run one after another.
- **Play time** = `slots × (dur + buf) − buf + break`. **Finish** = start + play time,
  formatted by the existing `fmtTime` (with its `(+1d)`).

The model holds a matchup's whole lane for its whole block, even a slot in which it
uses fewer courts (the third match of a 3-match matchup played 2 at a time, or a
tiebreaker). That keeps the finish one the plan can be sure of; an alert (§7.4) says
when a tighter schedule could finish the group stage sooner. It has no rest between a
team's matchups: a plan that rests its teams lengthens the break instead (§10 Q10).

## 3. What the page does

- **Format** lists *Standard tournament*, *Dual meet (club vs. club)* and *Team
  tournament*. Team hides the **Categories** panel, the club names and the **Build the
  event workbook** box, and shows the **Team tournament** panel.
- **The Team tournament panel**: four number inputs (teams, groups, matches per
  matchup, matches on court at once) and the hint under them; **First playoff round**
  as four toggle buttons (one disabled, with a tooltip, when it needs more teams than
  there are); chips listing each playoff round's matchups (`+ TB` where it keeps
  tiebreakers); **Tiebreaker game** as three toggle buttons (*Playoffs* and *Every
  matchup* disabled while *P* is odd, with a note); and the break's checkbox and
  minutes (the minutes disabled while the box is unticked).
- **Results**: the big clock with `Team tournament` in the line under it; the stat grid
  (Matches, with *Matches · incl. up to N tiebreakers* when there are any; Time slots;
  Play time); the alerts (§7); and the table, titled **Breakdown**, with one row per
  stage: Stage | Matchups | Matches | Slots | Time. A **Break** row follows the group
  stage when there is a break, a stage with tiebreakers shows *incl. N TB* under its
  matches, and a **Total** row ends it. The table keeps the page's `table.tl` styles,
  so on a phone it scrolls sideways under a sticky first column. A footnote under it
  says how the time is worked out and that who advances is the organiser's choice.
- Everything recalculates on input and is saved (in `localStorage` under the existing
  `ttc-config` key) with the team settings; a saved plan from before this change loads
  with the team defaults. **Reset** sets them back to the defaults.
- **Export CSV**, **Import CSV** and **Download .xlsx** handle a team plan (§7.1, §7.2).

## 4. Worked examples (the verification targets)

Each is a test in §6 and a browser check in §9. All have no buffer.

**PickleDrive.** 15 teams, 3 groups, *P* 4, *S* 2, 10 courts, 25-minute matches, 9:00 AM,
tiebreaker off. Lanes 5, block 2, *R* 5, 30 group matchups, group stage
`max(6, 5) × 2` = **12 slots**.

| First round | Break | Matchups | Matches | Slots | Finish |
| --- | --- | --- | --- | --- | --- |
| `sf` (the published plan) | 25 min | 34 | 136 | 16 | **4:05 PM** |
| `qf` (the plan the workbook was changed to) | off | 38 | 152 | 18 | **4:30 PM** |
| `qf` | 25 min | 38 | 152 | 18 | **4:55 PM** |

The first row is the event's published schedule: group stage to 2:00 PM, semifinals
from 2:25 PM after a one-slot break, Final and Bronze from 3:15 PM.

**Millennium.** 8 teams, 1 group, *P* 4, *S* 2, 8 courts, 30-minute matches, 1:00 PM,
`final_bronze`, tiebreaker off, break off. Lanes 4, block 2, *R* 7, group stage
`max(7, 7) × 2` = 14 slots, Bronze + Final 2. **30 matchups, 120 matches, 16 slots,
9:00 PM**, the event's own count and end. (It played the Bronze and then the Final, each
with all four matches on court; the model plays them side by side and ends at the same
time.)

**Ladies' Dink.** 16 teams, 4 groups, *P* 2, *S* 2, 6 courts, 25-minute matches, 3:00 PM,
`qf`, tiebreaker **Playoffs**, break 25 min. Lanes 3, block 1, *R* 3.

| Stage | Matchups | Matches | Slots | Starts |
| --- | --- | --- | --- | --- |
| Group stage (4 groups: 4-4-4-4) | 24 | 48 | `max(8, 3) × 1` = 8 | 3:00 PM |
| Break | | | | 6:20 PM |
| Quarterfinals | 4 | 12 (incl. 4 TB) | `ceil(4/3) × 1 + 1` = 3 | 6:45 PM |
| Semifinals | 2 | 6 (incl. 2 TB) | `1 + 1` = 2 | 8:00 PM |
| Bronze + Final | 2 | 6 (incl. 2 TB) | `1 + 1` = 2 | 8:50 PM |
| **Total** | **32** | **72** (incl. 8 TB) | **15** | finish **9:40 PM** |

That is the event's timetable and its 72 matches. With the tiebreaker off: 64 matches,
12 slots, **8:25 PM**.

**December draft (5–6 December 2026).** Three divisions, each its own plan (one day's
two divisions play on separate courts, so they are two plans). 9:00 AM, 25-minute
matches, break 25 min (the draft's *standings check*), 3 pairs per team.

| Division | Plan | Stage slots | Matchups | Matches | Slots | Finish |
| --- | --- | --- | --- | --- | --- | --- |
| Choice (5 Dec) | 48 teams, 8 groups, *P* 3, *S* 3, 18 courts, `r16`, tiebreaker off | 20 · R16 2 · QF 1 · SF 1 · Bronze + Final 1 | 136 | 408 | 25 | **7:50 PM** |
| Select (6 Dec) | 20 teams, 4 groups, *P* 3, *S* 3, 12 courts, `qf`, tiebreaker off | 10 · QF 1 · SF 1 · Bronze + Final 1 | 48 | 144 | 13 | **2:50 PM** |
| Prime (6 Dec) | 8 teams, 2 groups, *P* 2, *S* 2, 6 courts, `sf`, tiebreaker **Every matchup** | `max(4, 3) × 2` = 8 · SF 2 · Bronze + Final 2 | 16 | 48 (incl. 16 TB) | 12 | **2:25 PM** |

Choice and Select match the draft slot for slot. Prime's draft finishes at 2:50 PM: it
rests every team one slot after each round-robin matchup, so its round robin takes 9
slots; with a **50-minute** break the calculator gives the same 2:50 PM. Prime also shows
the idle-courts alert with *about 7 slots instead of 8*.

**3 matches, 2 at a time.** PickleDrive's plan with *P* 3: block 2, the group stage stays
12 slots, and the idle-courts alert reads *about 10 slots instead of 12*
(`max(ceil(90 / 10) + 1, 5 × 2)`). With *P* 4 there is no such alert.

## 5. The code: `tools/tournament-calculator.html`

Each edit names an anchor and what to do at it:

- **Insert after**: add the code as new lines after the whole line that contains the anchor.
- **Insert before**: add the code as new lines before the whole line that contains the anchor.
- **Replace**: replace exactly the anchor text (which may be part of a line) with the code.

The new functions (`teamDefaults`, `cleanTeam`, `calcTeam`, `teamStageRows`,
`writeTeamForm`, `renderTeamPanel`, `renderTeamResults`, `isTeamFormat`) start at column 0,
as every function in the page does: §6's test extracts them by name. `calcTeam` reads
only its argument and the page's pure helpers (`groupSizes`, `rrMatches`,
`bracketShorthand`), never the DOM or the `team` variable.

### E1. CSS for the team panel

**Insert after** the line containing:

```css
.fillbtn[aria-pressed="true"]:hover{color:#fff}
```

Add these lines:

```css
/* team format panel (calculator-team-format-spec §3) */
div.f{margin-bottom:12px}
div.f>span.lab{display:block;font-family:'Barlow Condensed';font-weight:600;font-size:12.5px;
  letter-spacing:.12em;text-transform:uppercase;color:var(--muted);margin-bottom:6px}
.chk{display:flex;align-items:center;gap:8px;font-size:14px;cursor:pointer;margin:6px 0 12px}
.chk input{width:18px;height:18px;accent-color:var(--green-dark)}
#teamPanel .po:disabled{opacity:.45;cursor:not-allowed}
```

### E2. The third Format option

**Insert after** the line containing:

```html
<option value="dual">Dual meet (club vs. club)</option>
```

Add these lines:

```html
            <option value="team">Team tournament</option>
```

### E3. An id on the Categories panel

**Replace** this text:

```html
<div class="panel" style="margin-top:18px">
```

with:

```html
<div class="panel" id="catPanel" style="margin-top:18px">
```

### E4. The Team tournament panel

**Insert before** the line containing:

```html
  </form>
```

Add these lines:

```html
    <div class="panel" id="teamPanel" style="margin-top:18px" hidden>
      <h2>Team tournament</h2>
      <div class="body">
        <div class="grid2">
          <label class="f"><span class="lab">Teams</span><input type="number" id="t-teams" min="2" max="100" value="15"></label>
          <label class="f"><span class="lab">Groups</span><input type="number" id="t-groups" min="1" max="30" value="3"></label>
          <label class="f"><span class="lab">Matches per matchup</span><input type="number" id="t-per" min="1" max="8" value="4"></label>
          <label class="f"><span class="lab">Matches on court at once</span><input type="number" id="t-slot" min="1" max="8" value="2"></label>
        </div>
        <div class="hint">One match per pair. A match whose players also play another match of the same matchup can’t share its slot.</div>
        <div class="f"><span class="lab">First playoff round</span><div class="playoffs" id="t-playoff"></div></div>
        <div class="playoffs" id="t-chips" style="margin:-4px 0 12px"></div>
        <div class="f"><span class="lab">Tiebreaker game</span><div class="playoffs" id="t-tb"></div>
          <div class="hint" id="t-tbnote" style="margin:6px 0 0"></div></div>
        <div class="grid2">
          <label class="chk"><input type="checkbox" id="t-break" checked> Break before the playoffs</label>
          <label class="f"><span class="lab">Break (min)</span><input type="number" id="t-breakmin" min="0" max="240" value="25"></label>
        </div>
      </div>
    </div>
```

### E5. An id on the Matches label

**Replace** this text:

```html
<div class="k">Matches</div>
```

with:

```html
<div class="k" id="r-matcheslab">Matches</div>
```

### E6. An id on the table title

**Replace** this text:

```html
<h2>Per-category breakdown</h2>
```

with:

```html
<h2 id="r-tabletitle">Per-category breakdown</h2>
```

### E7. An id on the table head

**Replace** this text:

```html
<thead><tr><th>Category</th>
```

with:

```html
<thead id="r-thead"><tr><th>Category</th>
```

### E8. The team model

**Insert before** the line containing:

```js
// Per-category RR/PO wall-clock time, given courts run in parallel but
```

Add these lines:

```js
/* ---------------- team format ----------------
   One competition and no categories: teams meet in matchups of P matches,
   S of them on court at once. sage-docs/docs/specs/.../calculator-team-format-spec.md §2. */
const TEAM_PLAYOFFS={
  r16:{q:16,label:'Round of 16',rounds:[['Round of 16',8],['Quarterfinals',4],['Semifinals',2],['Bronze + Final',2]]},
  qf:{q:8,label:'Quarterfinals',rounds:[['Quarterfinals',4],['Semifinals',2],['Bronze + Final',2]]},
  sf:{q:4,label:'Semifinals',rounds:[['Semifinals',2],['Bronze + Final',2]]},
  final_bronze:{q:4,label:'Bronze + Final',rounds:[['Bronze + Final',2]]}
};
const TEAM_PLAYOFF_ORDER=['r16','qf','sf','final_bronze'];
const TEAM_TIEBREAKERS=['off','playoffs','all'];
function teamDefaults(){
  return {teams:15, groups:3, perMatchup:4, perSlot:2, playoff:'qf', tiebreaker:'off', breakOn:true, breakMin:25};
}
// The team settings, clamped as the form's min/max are. Used by every reader of
// them, so the form, a saved plan and an imported CSV are all read alike. An odd
// number of matches can't end level, so it has no tiebreaker.
function cleanTeam(t){
  const d=teamDefaults(); t=t||{};
  const int=(v,lo,hi,def)=>{const n=parseInt(v,10);return Number.isFinite(n)?Math.min(hi,Math.max(lo,n)):def;};
  const perMatchup=int(t.perMatchup,1,8,d.perMatchup);
  return {
    teams:int(t.teams,2,100,d.teams), groups:int(t.groups,1,30,d.groups),
    perMatchup, perSlot:Math.min(perMatchup,int(t.perSlot,1,8,d.perSlot)),
    playoff:TEAM_PLAYOFF_ORDER.includes(t.playoff)?t.playoff:d.playoff,
    tiebreaker:perMatchup%2?'off':TEAM_TIEBREAKERS.includes(t.tiebreaker)?t.tiebreaker:d.tiebreaker,
    breakOn:t.breakOn===undefined?d.breakOn:!!t.breakOn,
    breakMin:int(t.breakMin,0,240,d.breakMin)
  };
}
function calcTeam(cfg){
  const t=cleanTeam(cfg.team), notes=[];
  const P=t.perMatchup, S=t.perSlot, tb=t.tiebreaker, courts=cfg.courts;
  if(courts<S){
    notes.push(['bad',`Needs at least ${S} courts: a matchup puts ${S} of its matches on court at once. Results show 0 until fixed.`]);
    return {team:true, t, per:[], stages:[], notes, matches:0, tiebreakers:0, matchups:0, slots:0, playMin:0, breakMin:0,
      endMin:null, lanes:0, block:0, groupBlock:0, playoff:null};
  }
  const listJoin=a=>a.length<2?a.join(''):a.slice(0,-1).join(', ')+' and '+a[a.length-1];
  const sizes=groupSizes(t.teams,t.groups);
  if(sizes.length!==t.groups) notes.push(['warn',`Each group needs at least 2 teams: using ${sizes.length} group${sizes.length>1?'s':''}.`]);
  // A lane is the S courts one matchup holds; a block, the slots it holds them.
  // With a tiebreaker in every matchup, a group matchup holds its lane one slot
  // more, and the team's next matchup waits for it.
  const lanes=Math.floor(courts/S), block=Math.ceil(P/S), groupBlock=block+(tb==='all'?1:0);
  const groupMatchups=rrMatches(sizes);
  const R=sizes.reduce((m,n)=>Math.max(m,n%2?n:n-1),0);
  const groupSlots=Math.max(Math.ceil(groupMatchups/lanes),R)*groupBlock;
  const stages=[{label:`Group stage (${sizes.length} group${sizes.length>1?'s':''}: ${bracketShorthand({sizes})})`, kind:'group',
    matchups:groupMatchups, matches:groupMatchups*P, tiebreakers:tb==='all'?groupMatchups:0, slots:groupSlots}];
  if(sizes.includes(2)) notes.push(['warn','A 2-team group plays one matchup.']);
  if(R>Math.ceil(groupMatchups/lanes)) notes.push(['warn',`The biggest group’s ${R} rounds, not the courts, set the length of the group stage.`]);
  // A first round needing more teams than there are falls back to the largest that fits.
  let key=t.playoff;
  if(TEAM_PLAYOFFS[key].q>t.teams){
    key=TEAM_PLAYOFF_ORDER.find(k=>TEAM_PLAYOFFS[k].q<=t.teams)||null;
    const want=TEAM_PLAYOFFS[t.playoff];
    notes.push(['warn',key
      ?`Only ${t.teams} teams: the ${want.label} needs ${want.q}. Starting at the ${TEAM_PLAYOFFS[key].label} instead.`
      :'A Bronze and a Final need 4 teams: no playoff, the group stage alone.']);
  }
  const po=key?TEAM_PLAYOFFS[key]:null;
  if(po){
    if(po.q<sizes.length) notes.push(['warn',`Fewer playoff places (${po.q}) than groups (${sizes.length}): not every group can send its winner.`]);
    // Rounds run one after another. A round's tiebreakers wait for its last
    // wave's games, so they take one slot more.
    po.rounds.forEach(([label,m])=>stages.push({label, kind:'playoff', matchups:m, matches:m*P,
      tiebreakers:tb==='off'?0:m, slots:Math.ceil(m/lanes)*block+(tb==='off'?0:1)}));
    const idle=po.rounds.filter(([,m])=>m<lanes).map(([l])=>l);
    if(idle.length) notes.push(['warn',`Some courts sit idle during the ${listJoin(idle)}: fewer matchups than the ${lanes} that fit on court at once.`]);
    if(tb==='playoffs') notes.push(['warn','Each playoff round keeps one slot for tiebreakers; the finish assumes every one is played.']);
  }
  if(tb==='all') notes.push(['warn','Every matchup keeps one slot for its tiebreaker, group stage included; the finish assumes every one is played.']);
  // When a matchup's last slot leaves some of its courts empty, a tighter
  // schedule than this model's could finish the group stage sooner (spec §7.4).
  if(P%S!==0||tb==='all'){
    const courtSlots=groupMatchups*P+(tb==='all'?groupMatchups:0);
    const x=Math.max(Math.ceil(courtSlots/courts)+1,R*groupBlock);
    if(x<groupSlots) notes.push(['warn',`Each matchup holds ${S} courts for ${groupBlock} slots, but its last slot uses ${tb==='all'?1:P%S}. A schedule that fills the idle courts with another matchup’s games could finish the group stage in about ${x} slots instead of ${groupSlots}. The finish time assumes ${groupSlots}.`]);
  }
  const breakMin=po&&t.breakOn?t.breakMin:0;
  const slots=stages.reduce((s,x)=>s+x.slots,0);
  const tiebreakers=stages.reduce((s,x)=>s+x.tiebreakers,0);
  const matches=stages.reduce((s,x)=>s+x.matches,0)+tiebreakers;
  const playMin=slots?slots*(cfg.dur+cfg.buf)-cfg.buf+breakMin:0;
  return {team:true, t, per:[], stages, notes, matches, tiebreakers,
    matchups:stages.reduce((s,x)=>s+x.matchups,0), slots, playMin, breakMin,
    endMin:matches?cfg.startMin+playMin:null, lanes, block, groupBlock, playoff:key};
}
// The stage table's rows, shared by the page and the .xlsx. Each stage's time
// is its slots, the last stage's without its trailing buffer, so the column
// adds up to the play time, the Break row included.
function teamStageRows(cfg,r){
  const slotLen=cfg.dur+cfg.buf, rows=[];
  r.stages.forEach((s,i)=>{
    rows.push({label:s.label, matchups:s.matchups, matches:s.matches+s.tiebreakers, tiebreakers:s.tiebreakers,
      slots:s.slots, min:s.slots*slotLen-(i===r.stages.length-1?cfg.buf:0)});
    if(i===0&&r.breakMin) rows.push({label:'Break', matchups:null, matches:null, tiebreakers:0, slots:null, min:r.breakMin});
  });
  return rows;
}
```

### E9. calcAll hands the team format to calcTeam

**Insert before** the line containing:

```js
  const per = cfg.cats.map(c=>calcCategory(c,cfg));
```

Add these lines:

```js
  if(cfg.format==='team') return calcTeam(cfg);
```

### E10. The team settings' state

**Insert after** the line containing:

```js
let nextId=1;
```

Add these lines:

```js
let team=teamDefaults(); // the team format's settings (spec §2.1), kept while another format is shown
```

### E11. isTeamFormat, and the standard table head to restore

**Insert after** the line containing:

```js
function isDualFormat(){ const el=$('f-format'); return !!el && el.value==='dual'; }
```

Add these lines:

```js
function isTeamFormat(){ const el=$('f-format'); return !!el && el.value==='team'; }
const STD_THEAD=$('r-thead').innerHTML; // restored when the format leaves 'team'
```

### E12. Format switch shows the right panel; no handoff for team

**Replace** this text:

```js
  const m=GENERATOR_MASTERS[isDualFormat()?'dual':'standard'];
```

with:

```js
  // The team format shows its own panel instead of the category cards, and
  // has no master yet, so no handoff (calculator-team-format-spec §7.3).
  const teamMode=isTeamFormat();
  if($('teamPanel')) $('teamPanel').hidden=!teamMode;
  if($('catPanel')) $('catPanel').hidden=teamMode;
  if(teamMode) writeTeamForm();
  const m=GENERATOR_MASTERS[teamMode?'team':isDualFormat()?'dual':'standard'];
```

### E13. Hide the handoff box when there is no master

**Replace** this text:

```js
  const g=$('genbox'); if(g) g.hidden=false;
```

with:

```js
  const g=$('genbox'); if(g) g.hidden=!m;
  if(!m) return;
```

### E14. readForm carries the format and the team settings

**Replace** this text:

```js
    format: isDualFormat()?'dual':'standard', clubA, clubB,
```

with:

```js
    format: isTeamFormat()?'team':isDualFormat()?'dual':'standard', clubA, clubB, team:{...team},
```

### E15. recalc: the team format renders its own results

**Insert after** the line containing:

```js
  const dual=cfg.format==='dual';
```

Add these lines:

```js
  if(cfg.format==='team'){ renderTeamPanel(); renderTeamResults(cfg,r); save(); return; }
  $('r-tabletitle').textContent='Per-category breakdown';
  $('r-thead').innerHTML=STD_THEAD;
  $('r-matcheslab').textContent='Matches';
```

### E16. The team panel's and results' rendering

**Insert before** the line containing:

```js
function groupShorthand(p){
```

Add these lines:

```js
/* ---------------- team format: panel and results (spec §3) ---------------- */
// Writes the (cleaned) team settings into the panel's inputs.
function writeTeamForm(){
  team=cleanTeam(team);
  $('t-teams').value=team.teams; $('t-groups').value=team.groups;
  $('t-per').value=team.perMatchup; $('t-slot').value=team.perSlot;
  $('t-break').checked=team.breakOn; $('t-breakmin').value=team.breakMin;
  renderTeamPanel();
}
// The two button rows, from the settings as calcTeam reads them.
function renderTeamPanel(){
  const t=cleanTeam(team), odd=t.perMatchup%2===1;
  $('t-playoff').innerHTML=TEAM_PLAYOFF_ORDER.map(k=>{
    const po=TEAM_PLAYOFFS[k], off=po.q>t.teams;
    return `<button type="button" class="po fillbtn" data-tplayoff="${k}" aria-pressed="${t.playoff===k}"${off?` disabled title="Needs ${po.q} teams"`:''}>${po.label}</button>`;
  }).join('');
  $('t-tb').innerHTML=[['off','Off'],['playoffs','Playoffs'],['all','Every matchup']].map(([k,l])=>
    `<button type="button" class="po fillbtn" data-ttb="${k}" aria-pressed="${t.tiebreaker===k}"${odd&&k!=='off'?' disabled':''}>${l}</button>`).join('');
  $('t-tbnote').textContent=odd?'A matchup of an odd number of matches can’t end level.':'';
  $('t-breakmin').disabled=!t.breakOn;
}
function renderTeamResults(cfg,r){
  const cell=v=>v==null?'—':v;
  const tbTxt=n=>n?`<span class="win">incl. ${n} TB</span>`:'';
  $('r-matches').textContent=r.matches;
  $('r-matcheslab').textContent=r.tiebreakers?`Matches · incl. up to ${r.tiebreakers} tiebreakers`:'Matches';
  $('r-slots').textContent=r.slots;
  $('r-hours').textContent=r.matches?fmtDur(r.playMin):'0m';
  $('r-end').textContent=r.endMin!=null?fmtTime(r.endMin):'—';
  const dateStr=cfg.date?new Date(cfg.date+'T00:00').toLocaleDateString([], {weekday:'short',month:'long',day:'numeric',year:'numeric'}):'';
  $('r-head').textContent=[cfg.title||'Untitled tournament',dateStr,`from ${fmtTime(cfg.startMin)} on ${cfg.courts} court${cfg.courts>1?'s':''}`,'Team tournament']
    .filter(Boolean).join(' · ');
  document.title=(cfg.title?cfg.title+' · ':'')+'Time Calculator';
  const alerts=[];
  if(r.matches){
    const endClock=r.endMin%1440;
    if(r.endMin-cfg.startMin>15*60||r.endMin>=24*60) alerts.push(['bad','This plan runs past midnight. Add courts, cut teams, or shorten matches.']);
    else if(endClock>=22*60){
      // Courts help only a whole lane at a time, so the suggestion is one more lane (spec §7.4).
      const more=(r.lanes+1)*r.t.perSlot, alt=calcTeam({...cfg,courts:more});
      alerts.push(['warn','Finishing after 10 PM — tight for a one-day event.'
        +(alt.endMin!=null&&alt.endMin<r.endMin?` With ${more} courts: finish at ${fmtTime(alt.endMin)}.`:'')]);
    }
    else alerts.push(['good',`Comfortable one-day plan. Last match steps off court around ${fmtTime(r.endMin)}.`]);
  }
  r.notes.forEach(n=>alerts.push(n));
  $('r-alerts').innerHTML=alerts.map(([cls,t])=>`<div class="alert ${cls}">${t}</div>`).join('');
  $('r-tabletitle').textContent='Breakdown';
  $('r-thead').innerHTML='<tr><th>Stage</th><th>Matchups</th><th>Matches</th><th>Slots</th><th>Time</th></tr>';
  $('r-rows').innerHTML=r.matches
    ? teamStageRows(cfg,r).map(x=>`<tr><td>${x.label}</td><td class="num">${cell(x.matchups)}</td><td class="num">${cell(x.matches)}${tbTxt(x.tiebreakers)}</td><td class="num">${cell(x.slots)}</td><td class="num">${fmtDur(x.min)}</td></tr>`).join('')
      +`<tr><td><b>Total</b></td><td class="num"><b>${r.matchups}</b></td><td class="num"><b>${r.matches}</b>${tbTxt(r.tiebreakers)}</td><td class="num"><b>${r.slots}</b></td><td class="num"><b>${fmtDur(r.playMin)}</b></td></tr>`
    : '<tr><td colspan="5" style="text-align:center;color:var(--muted);padding:16px">No matches: see the alert above</td></tr>';
  $('t-chips').innerHTML=r.stages.filter(s=>s.kind==='playoff').map(s=>
    `<span class="tagchip">${s.label}: ${s.matchups}${s.tiebreakers?' <small>+ TB</small>':''}</span>`).join('');
  const S=r.t.perSlot;
  $('r-windows').innerHTML=r.matches
    ? `Time = slots of ${cfg.dur}-min matches${cfg.buf?` + ${cfg.buf}-min buffer`:''}. A matchup holds ${S} court${S>1?'s':''} for ${r.block} slot${r.block>1?'s':''}`
      +(r.t.tiebreaker!=='off'?', and its tiebreaker one slot more':'')
      +`; ${r.lanes} matchup${r.lanes>1?'s':''} fit on ${cfg.courts} courts at once, and no team plays two matchups at once.`
      +' Who advances is the organiser’s choice: the calculator only counts the places.'
    : '';
  $('r-windows').style.display=r.matches?'':'none';
}
```

### E17. Export CSV: no club names for team

**Replace** this text:

```js
   ['format',cfg.format],['club_a',cfg.clubA],['club_b',cfg.clubB]]
```

with:

```js
   ['format',cfg.format],['club_a',cfg.format==='team'?'':cfg.clubA],['club_b',cfg.format==='team'?'':cfg.clubB]]
```

### E18. Export CSV: the team settings, and no category rows

**Insert before** the line containing:

```js
  // `teams` (dual: no live UI field once "Pairs per club" replaced it — see
```

Add these lines:

```js
  // The team format: seven settings and no category rows (spec §7.1). `playoff`
  // is the first round actually planned, after any fallback for too few teams.
  if(cfg.format==='team'){
    const t=cleanTeam(cfg.team);
    [['teams',t.teams],['groups',t.groups],['matches_per_matchup',t.perMatchup],['matches_per_slot',t.perSlot],
     ['playoff',r.playoff||t.playoff],['tiebreaker',t.tiebreaker],['break_min',t.breakOn?t.breakMin:0]]
      .forEach(([k,v])=>lines.push(['setting',k,csvCell(v),'','','',''].join(',')));
    return lines.join('\n');
  }
```

### E19. Download .xlsx: the team format's empty-plan message

**Replace** this text:

```js
  if(!r.matches){ioMsg('Add at least one category before downloading.','bad');return;}
```

with:

```js
  if(!r.matches){ioMsg(cfg.format==='team'?'This plan has no matches yet: see the alerts.':'Add at least one category before downloading.','bad');return;}
```

### E20. Download .xlsx: a header line for team

**Insert after** the line containing:

```js
  if(cfg.format==='dual') pushHeaderLine(`Dual meet: ${cfg.clubA} vs ${cfg.clubB}`);
```

Add these lines:

```js
  if(cfg.format==='team') pushHeaderLine('Team tournament');
```

### E21. Download .xlsx: the team stage table

**Insert before** the line containing:

```js
  const head=['Code','Category','Pairs','Brackets','Bracket sizes','Format','RR matches','PO matches','Total','RR time','PO time','Total time','Note'];
```

Add these lines:

```js
  // The team format: the stage table of the page, with its totals (spec §7.2).
  if(cfg.format==='team'){
    aoa.push(['Stage','Matchups','Matches','Of which tiebreakers','Slots','Time']);
    teamStageRows(cfg,r).forEach(x=>aoa.push([x.label,x.matchups==null?'':x.matchups,x.matches==null?'':x.matches,
      x.tiebreakers||'',x.slots==null?'':x.slots,fmtDur(x.min)]));
    aoa.push(['TOTAL',r.matchups,r.matches,r.tiebreakers||'',r.slots,fmtDur(r.playMin)]);
    const ws=XLSX.utils.aoa_to_sheet(aoa);
    ws['!cols']=[{wch:40},{wch:10},{wch:10},{wch:20},{wch:7},{wch:10}];
    ws['!merges']=merges.map(m=>({s:m.s,e:{r:m.e.r,c:5}}));
    const wb=XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb,ws,'Breakdown');
    try{
      XLSX.writeFile(wb, fileBase(cfg)+'-breakdown.xlsx');
      ioMsg('Downloaded '+fileBase(cfg)+'-breakdown.xlsx','good');
    }catch(err){ioMsg('Download failed in this viewer — open the file in a regular browser.','bad');}
    return;
  }
```

### E22. Import CSV: read a team plan

**Replace** this text:

```js
  const importedDual=st.format==='dual';
```

with:

```js
  const importedDual=st.format==='dual', importedTeam=st.format==='team';
  // A team plan's settings (spec §7.1). break_min 0 is an unticked break, which
  // keeps the default minutes for when it is ticked again; a missing one is the default.
  if(importedTeam){
    const bm=parseInt(st.break_min,10);
    team=cleanTeam({teams:st.teams, groups:st.groups, perMatchup:st.matches_per_matchup, perSlot:st.matches_per_slot,
      playoff:st.playoff, tiebreaker:st.tiebreaker, breakOn:bm!==0, breakMin:bm>0?bm:25});
  }
```

### E23. Import CSV: switch the format menu to team

**Replace** this text:

```js
  if($('f-format'))$('f-format').value=importedDual?'dual':'standard';
```

with:

```js
  if($('f-format'))$('f-format').value=importedTeam?'team':importedDual?'dual':'standard';
```

### E24. Import CSV: the team format's message

**Insert before** the line containing:

```js
  const fallbackNote=(importedDual&&usedFallbackSplit)?' · older export — pairs split evenly per club, please review':'';
```

Add these lines:

```js
  if(importedTeam){ioMsg('Imported: team tournament'+(st.title?' · '+st.title:''),'good');return;}
```

### E25. Reset: the team settings back to their defaults

**Insert after** the line containing:

```js
  if($('f-clubB'))$('f-clubB').value='Club B';
```

Add these lines:

```js
  team=teamDefaults();
```

### E26. The team panel's events

**Insert after** the line containing:

```js
if($('f-format'))$('f-format').addEventListener('change',()=>{toggleDualClubsVisibility();renderCats();recalc();});
```

Add these lines:

```js
// The team panel (spec §3). Inputs keep what was typed; cleanTeam clamps it
// wherever it is read, as the category cards' inputs do.
const TEAM_FIELDS={'t-teams':'teams','t-groups':'groups','t-per':'perMatchup','t-slot':'perSlot','t-breakmin':'breakMin'};
Object.entries(TEAM_FIELDS).forEach(([id,k])=>$(id).addEventListener('input',()=>{team[k]=$(id).value;recalc();}));
$('t-break').addEventListener('change',()=>{team.breakOn=$('t-break').checked;recalc();});
$('teamPanel').addEventListener('click',e=>{
  const b=e.target.closest('.fillbtn'); if(!b||b.disabled) return;
  if(b.dataset.tplayoff) team.playoff=b.dataset.tplayoff;
  if(b.dataset.ttb) team.tiebreaker=b.dataset.ttb;
  recalc();
});
```

### E27. Boot: a saved team plan loads even with no categories

**Replace** this text:

```js
  if(saved&&saved.cats&&saved.cats.length){
```

with:

```js
  if(saved&&((saved.cats&&saved.cats.length)||saved.format==='team')){
```

### E28. Boot: restore the team format and its settings

**Replace** this text:

```js
    if($('f-format'))$('f-format').value=saved.format==='dual'?'dual':'standard';
```

with:

```js
    if($('f-format'))$('f-format').value=['dual','team'].includes(saved.format)?saved.format:'standard';
    team=cleanTeam(saved.team);
```

### E29. Boot: a saved plan with no categories gets the default one

**Replace** this text:

```js
    cats=saved.cats.map(c=>({...c,id:nextId++}));
```

with:

```js
    cats=saved.cats&&saved.cats.length?saved.cats.map(c=>({...c,id:nextId++})):[defaultCat()];
```

## 6. The test

Create `_tests/unit/tools/tournament-calculator-team.test.mjs` (a new folder) with exactly
this content. It pulls `calcTeam` and its helpers out of the page with the harness's own
extractor and runs §4's examples. The suite's `npm test` picks it up by its glob
(`unit/**/*.test.mjs`); nothing else needs registering.

```js
// The Tournament Calculator's team format (sage-docs/docs/specs/.../calculator-team-format-spec.md).
// calcTeam is pure: it is pulled out of the page and run here against the spec's
// worked examples (§3): the three team events so far and a draft plan.
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import path from 'node:path';
import { SITE_ROOT } from '../../helpers/paths.mjs';
import { extractFunction, extractConst } from '../../helpers/extract.mjs';

const page = fs.readFileSync(path.join(SITE_ROOT, 'tools/tournament-calculator.html'), 'utf8');
const consts = ['rrMatches', 'TEAM_PLAYOFFS', 'TEAM_PLAYOFF_ORDER', 'TEAM_TIEBREAKERS'].map(n => extractConst(page, n).text);
const fns = ['groupSizes', 'bracketShorthand', 'teamDefaults', 'cleanTeam', 'calcTeam', 'teamStageRows', 'fmtTime']
  .map(n => extractFunction(page, n));
const { calcTeam, teamStageRows, fmtTime } =
  new Function([...consts, ...fns].join('\n') + '\nreturn { calcTeam, teamStageRows, fmtTime };')();

// A plan: the shared settings, then the team ones over teamDefaults().
const plan = (shared, team) => ({ dur: 25, buf: 0, startMin: 9 * 60, courts: 10, ...shared, team });
const finish = r => fmtTime(r.endMin);
const notes = r => r.notes.map(([, text]) => text).join('\n');

test('PickleDrive (3 Oct): the published SF plan, and the QF plan with and without the break', () => {
  const t = { teams: 15, groups: 3, perMatchup: 4, perSlot: 2, tiebreaker: 'off' };
  const sf = calcTeam(plan({}, { ...t, playoff: 'sf' }));
  assert.deepEqual([sf.matchups, sf.matches, sf.slots, finish(sf)], [34, 136, 16, '4:05 PM']);
  assert.equal(sf.stages[0].slots, 12);
  const qf = calcTeam(plan({}, { ...t, playoff: 'qf', breakOn: false }));
  assert.deepEqual([qf.matchups, qf.matches, qf.slots, finish(qf)], [38, 152, 18, '4:30 PM']);
  assert.equal(finish(calcTeam(plan({}, { ...t, playoff: 'qf' }))), '4:55 PM');
});

test('Millennium (24 Oct): Bronze + Final straight from the standings', () => {
  const r = calcTeam(plan({ courts: 8, dur: 30, startMin: 13 * 60 },
    { teams: 8, groups: 1, perMatchup: 4, perSlot: 2, playoff: 'final_bronze', tiebreaker: 'off', breakOn: false }));
  assert.deepEqual([r.matchups, r.matches, r.slots, finish(r)], [30, 120, 16, '9:00 PM']);
  assert.equal(r.stages[0].slots, 14);
});

test("Ladies' Dink (7 Nov): a tiebreaker in every playoff round", () => {
  const t = { teams: 16, groups: 4, perMatchup: 2, perSlot: 2, playoff: 'qf', tiebreaker: 'playoffs' };
  const r = calcTeam(plan({ courts: 6, startMin: 15 * 60 }, t));
  assert.deepEqual([r.matchups, r.matches, r.tiebreakers, r.slots, finish(r)], [32, 72, 8, 15, '9:40 PM']);
  assert.deepEqual(r.stages.map(s => s.slots), [8, 3, 2, 2]);
  const off = calcTeam(plan({ courts: 6, startMin: 15 * 60 }, { ...t, tiebreaker: 'off' }));
  assert.deepEqual([off.matches, off.slots, finish(off)], [64, 12, '8:25 PM']);
});

test('December draft: Choice from a Round of 16, Select from the quarterfinals', () => {
  const choice = calcTeam(plan({ courts: 18 },
    { teams: 48, groups: 8, perMatchup: 3, perSlot: 3, playoff: 'r16', tiebreaker: 'off' }));
  assert.deepEqual([choice.matchups, choice.matches, choice.slots, finish(choice)], [136, 408, 25, '7:50 PM']);
  assert.deepEqual(choice.stages.map(s => s.slots), [20, 2, 1, 1, 1]);
  const select = calcTeam(plan({ courts: 12 },
    { teams: 20, groups: 4, perMatchup: 3, perSlot: 3, playoff: 'qf', tiebreaker: 'off' }));
  assert.deepEqual([select.matchups, select.matches, select.slots, finish(select)], [48, 144, 13, '2:50 PM']);
});

test('December draft: Prime, a tiebreaker in every matchup, and its rest as a longer break', () => {
  const t = { teams: 8, groups: 2, perMatchup: 2, perSlot: 2, playoff: 'sf', tiebreaker: 'all' };
  const r = calcTeam(plan({ courts: 6 }, t));
  assert.deepEqual([r.matchups, r.matches, r.tiebreakers, r.slots, finish(r)], [16, 48, 16, 12, '2:25 PM']);
  assert.equal(r.stages[0].slots, 8);
  assert.match(notes(r), /about 7 slots instead of 8/);
  assert.equal(finish(calcTeam(plan({ courts: 6 }, { ...t, breakMin: 50 }))), '2:50 PM');
});

test('3 matches, 2 at a time: the model keeps its slots and the alert names a tighter schedule', () => {
  const t = { teams: 15, groups: 3, perSlot: 2, playoff: 'qf', tiebreaker: 'off' };
  const three = calcTeam(plan({}, { ...t, perMatchup: 3 }));
  assert.equal(three.stages[0].slots, 12);
  assert.match(notes(three), /about 10 slots instead of 12/);
  assert.doesNotMatch(notes(calcTeam(plan({}, { ...t, perMatchup: 4 }))), /idle courts/);
});

test('the stage rows add up to the totals, the Break row and a buffer included', () => {
  const cfg = plan({ buf: 5, courts: 6, startMin: 15 * 60 },
    { teams: 16, groups: 4, perMatchup: 2, perSlot: 2, playoff: 'qf', tiebreaker: 'playoffs' });
  const r = calcTeam(cfg);
  const rows = teamStageRows(cfg, r);
  assert.deepEqual(rows.map(x => x.label.split(' (')[0]),
    ['Group stage', 'Break', 'Quarterfinals', 'Semifinals', 'Bronze + Final']);
  assert.equal(rows.reduce((s, x) => s + x.min, 0), r.playMin);
  assert.equal(rows.reduce((s, x) => s + (x.matches || 0), 0), r.matches);
  assert.equal(rows.reduce((s, x) => s + (x.slots || 0), 0), r.slots);
});

test('too few teams: the first round falls back, and below 4 there is no playoff and no break', () => {
  const six = calcTeam(plan({}, { teams: 6, groups: 2, playoff: 'r16' }));
  assert.equal(six.playoff, 'sf');
  assert.match(notes(six), /Starting at the Semifinals/);
  const three = calcTeam(plan({}, { teams: 3, groups: 1, playoff: 'qf' }));
  assert.equal(three.playoff, null);
  assert.deepEqual([three.stages.length, three.breakMin], [1, 0]);
  assert.match(notes(three), /need 4 teams/);
});

test("settings outside the form's range are clamped, and an odd matchup has no tiebreaker", () => {
  const few = calcTeam(plan({ courts: 1 }, { perSlot: 2 }));
  assert.deepEqual([few.matches, few.endMin], [0, null]);
  assert.equal(few.notes[0][0], 'bad');
  const groups = calcTeam(plan({}, { teams: 5, groups: 3 }));
  assert.equal(groups.stages[0].label, 'Group stage (2 groups: 3-2)');
  assert.match(notes(groups), /using 2 groups/);
  const odd = calcTeam(plan({}, { perMatchup: 3, tiebreaker: 'all' }));
  assert.deepEqual([odd.t.tiebreaker, odd.tiebreakers], ['off', 0]);
  const strings = calcTeam(plan({}, { teams: '15', groups: '3', perMatchup: '4', perSlot: '2', breakMin: '25' }));
  assert.equal(finish(strings), finish(calcTeam(plan({}, {}))));
});
```

Run the site's whole unit suite (first time: `npm install` in `_tests`):

```bash
cd /d/Personal/SAGE/sage-match-control.github.io/_tests && npm test
```

Every test passes, the nine new ones included. Run it once **before** §5's edits too: the
new file then fails (the functions don't exist yet), which shows it tests the change.
`npm run compare` (the Playwright harness) covers the event pages, not the calculator,
and need not be run for this change.

## 7. Behaviour reference

What §5's code does, for checking it and for the docs (§8).

### 7.1 Export and Import CSV

The file format is unchanged: the same comment lines and header line
(`type,key,value,teams,…,rr_legs,rr_best_of`), then `setting` rows. A team plan writes
the nine shared settings (`title`, `date`, `start`, `courts`, `duration_min`, `buffer_min`,
`format` = `team`, `club_a` and `club_b` both empty) and seven of its own, and **no
`category` rows**:

| Setting | Value |
| --- | --- |
| `teams`, `groups`, `matches_per_matchup`, `matches_per_slot` | numbers |
| `playoff` | `r16`, `qf`, `sf` or `final_bronze`: the first round actually planned, after any fallback for too few teams |
| `tiebreaker` | `off`, `playoffs` or `all` |
| `break_min` | the minutes used: `0` when the break is unticked |

Import reads a `team` file through `cleanTeam`, so an out-of-range or unknown value reads
as the default; `break_min` `0` imports with the box unticked and 25 in the field, and a
missing one as the default. It switches the menu to Team and reports *Imported: team
tournament*. A standard or dual file imports exactly as before. A team file opened in a
copy of the page from before this change imports as a standard plan with the categories
already on screen; only an offline copy can be that old (the service worker fetches the
page network-first), which is acceptable.

### 7.2 Download .xlsx

The same **Breakdown** sheet and file name (`fileBase`), with a *Team tournament* header
line, then the stage table: Stage | Matchups | Matches | Of which tiebreakers | Slots | Time,
and a TOTAL row.

### 7.3 Generator handoff

There is no Team Tournament Master yet
([its spec](team-tournament-master-spec.md) is not started), so the **Build the event
workbook** box is hidden for the team format. When the Master exists, adding a `team`
entry (file ID and help text) to `GENERATOR_MASTERS` brings the box back with nothing else
changed. Both existing generators already refuse a plan whose `format` is not theirs.

### 7.4 Alerts

The first alert is the finish-time one, as in the other formats: past midnight (*bad*),
after 10 PM (*warn*) or comfortable (*good*). For a team plan, the after-10 PM alert
suggests the courts of **one more lane**, `(lanes + 1) × S`, worked out with `calcTeam`
(*With 15 courts: finish at 8:40 PM.*), and says nothing more when that finish is no
earlier: a court that adds no lane changes nothing. Then `calcTeam`'s notes:

| When | Class | Says |
| --- | --- | --- |
| courts < *S* | bad | needs at least *S* courts; results show 0 |
| fewer groups than asked (each needs 2 teams) | warn | using *G′* groups |
| a group of 2 | warn | a 2-team group plays one matchup |
| *R* > `ceil(groupMatchups / lanes)` | warn | the biggest group's rounds, not the courts, set the group stage |
| *T* < *Q* | warn | starting at a smaller first round, or no playoff below 4 teams |
| *Q* < groups | warn | not every group can send its winner |
| a playoff round with fewer matchups than lanes | warn | some courts sit idle during those rounds (one alert naming them) |
| tiebreaker `playoffs` / `all` | warn | the slots kept for tiebreakers; the finish assumes all are played |
| *P* not a multiple of *S*, or tiebreaker `all`, and *X* < the group slots | warn | a tighter schedule could finish the group stage in about *X* slots; *X* = `max( ceil(groupCourtSlots / courts) + 1, R × groupBlock )`, *groupCourtSlots* being the group matches plus their tiebreakers |

## 8. Documentation (`sage-docs`)

Make these after the site change is live (§0.3 step 7). Write in the present tense:
what the calculator does, not what changed.

1. **`docs/features/tournament-calculator.md`.**
   - Rename `## Two formats` to `## Three formats` and add a third bullet after the dual
     meet one: *"**Team tournament** — named teams meet in matchups of several matches,
     one per pair, in groups and then a playoff. One competition, no categories."*
   - Add a section `## Team tournaments` before `## Exporting`, saying in plain words: the
     inputs (§2.1, with the hint about players in two matches); that the playoff starts at
     the Round of 16, quarterfinals, semifinals or the Bronze and Final, and always ends in
     a Bronze and a Final played together; the tiebreaker choices; that the match count
     includes every possible tiebreaker and the finish assumes they are all played; that
     the model keeps a matchup's courts for its whole time, so a tighter schedule can
     finish sooner and the alert says by how much; and that a rest between matchups is
     planned as a longer break.
   - In `## Handing the plan to a sheet generator`, add: *"The team format has no master
     workbook yet, so the box is hidden for it."*
2. **`docs/technical/tournament-calculator.md`.** Add a section `## Team mode` before
   `## PWA (installable, offline)`: `calcTeam` and its model (§2.2–2.3, the lane and block
   terms, why the lane is held for the whole block), `cleanTeam` as the one reader of the
   settings, the `team` object in `readForm` and the saved plan, the CSV settings (§7.1),
   that `recalc` hands the team format to `renderTeamResults`, and the test file. In
   `## Sheet generator handoff`, add that the team format has no entry in
   `GENERATOR_MASTERS`, so `toggleDualClubsVisibility()` hides the box. Add this spec to
   the page's closing **Specs:** line.
3. **`docs/usage/plan-the-event.md`.** Replace the `- **Team:**` bullet under
   `## Where the formats differ` with one that says to choose **Team tournament** in the
   calculator, and that there is no master to hand the plan to yet, so step 2 starts from
   the last team event's workbook: keep its link to
   `build-the-workbooks.md#team-the-pickledrive-copy`.
4. **This spec moves to `implemented/`**, by the procedure in `docs/specs/README.md`,
   *Moving a spec between folders*: its status line (built, with the site commit), the
   specs index and the two folder READMEs, `mkdocs.yml`'s nav, and every real link to it
   (`grep -rn "calculator-team-format-spec.md" docs/`). Add the spec under **Tournament
   Calculator** in the index's **Implemented** section.
5. **Changelog and version**: a MINOR entry at the top of `docs/changelog.md` (`## [x.y.z] —
   YYYY-MM-DD`, Manila date) under **Changed**, and the same version in `mkdocs.yml`'s
   `copyright` line.

## 9. Browser checks

Serve the site from the workspace and open the calculator:

```bash
cd /d/Personal/SAGE && npx --yes http-server sage-match-control.github.io -p 8000 -c-1
```

Open `http://localhost:8000/tools/tournament-calculator.html`. Clear the saved plan first
(DevTools console: `localStorage.removeItem('ttc-config')`, then reload).

- [ ] **Standard is unchanged.** Before switching, note the default plan's finish (one
  16-pair category, 4 courts: **12:45 PM**) and the table head (*Category, Teams, Brackets,
  RR, PO, Total, RR time, PO time, Total time*). Dual meet behaves as before.
- [ ] **Team format.** Choose *Team tournament*: the Categories panel, club names and the
  generator box hide; the Team tournament panel shows. The defaults (15 teams, 3 groups,
  *P* 4, *S* 2, Quarterfinals, tiebreaker Off, break 25) on 10 courts from 9:00 AM read
  **4:55 PM**, 152 matches, 18 slots.
- [ ] **Every plan of §4 reproduces** its matchups, matches, slots and finish (PickleDrive's
  three, Millennium, Ladies' Dink with the tiebreaker Playoffs and Off, Choice, Select,
  Prime with the break at 25 and at 50). Ladies' Dink reads *Matches · incl. up to 8
  tiebreakers*, and its table: Group stage 24 · 48 · 8 · 3h 20m; Break 25m; Quarterfinals
  4 · 12 incl. 4 TB · 3 · 1h 15m; Semifinals and Bronze + Final 2 · 6 incl. 2 TB · 2 · 50m;
  Total 32 · 72 incl. 8 TB · 15 · 6h 40m.
- [ ] **Buttons.** With 8 teams, *Round of 16* and *Quarterfinals* are disabled with a
  tooltip; with *P* 3, *Playoffs* and *Every matchup* are disabled and the note shows.
  Unticking the break takes 25 minutes off, removes the Break row and disables the minutes;
  ticking it brings the same minutes back.
- [ ] **Alerts.** Prime shows *about 7 slots instead of 8*; PickleDrive with *P* 3 shows
  *about 10 slots instead of 12*. 48 teams, 8 groups, *P* 3, *S* 3, 12 courts, 9:00 AM,
  `qf`: **11:10 PM** and *With 15 courts: finish at 8:40 PM*; typing 15 courts gives
  8:40 PM, and 13 courts still 11:10 PM. 1 court with *S* 2: results 0 and the *bad*
  alert.
- [ ] **CSV round trip.** Export CSV for Ladies' Dink; open the file: no `category` rows,
  `club_a`/`club_b` empty, the seven team settings (`tiebreaker,playoffs`,
  `break_min,25`). Reset, then Import CSV: Team format, 9:40 PM, *Imported: team
  tournament*. Untick the break, export and import again: the box is unticked, the field
  reads 25, 9:15 PM.
- [ ] **.xlsx.** Download .xlsx opens with the *Team tournament* line and the stage table
  with its TOTAL row.
- [ ] **Saved plan.** A team plan survives a reload, in the Team format. Switching to
  Standard and back keeps both the categories and the team settings.
- [ ] **No console errors**, at desktop width and at 375 px (DevTools device toolbar), and
  no sideways scroll of the page at 375 px (the table scrolls inside its box).
- [ ] **Offline still works**: in DevTools → Application the service worker is registered;
  with *Offline* ticked, a reload still shows the calculator.

## 10. Decisions

Answered by the owner on 2026-11-12:

| # | Question | Answer |
| --- | --- | --- |
| Q1 | Default break before the playoffs? | **25 minutes, and it can be removed**: a checkbox, ticked, beside the minutes |
| Q2 | Offer the Final after the Bronze, as Millennium played them? | **No.** Same finish either way |
| Q3 | *S* derived from the roster, or one per stage? | **One input, default 2, for every stage** |
| Q4 | Does **Matches** count the tiebreakers? | **Yes**, with *incl. up to N tiebreakers*: the workbook numbers them and the finish assumes them. On the day, Control Center's *matches left* drops the ones not needed |
| Q5 | A playoff bracket chart? | **Not in this version** |
| Q6 | Team names in the calculator? | **No**: names and rosters are the generator's inputs |
| Q7 | Default matches per matchup? | **4** |
| Q8 | Which playoffs? | **A first round from the Round of 16 down to Bronze + Final, always ending in a Bronze and a Final** |
| Q9 | A tiebreaker in the group stage? | **Yes**, as *Every matchup* (the December draft's Prime) |
| Q10 | A rest between a team's matchups? | **No: the calculator is an estimate.** A plan that rests its teams lengthens the break. An input would only approximate it: a simple scheduler needed 10 slots for Prime where the draft manages 9 |

## 11. Out of scope

- Building the schedule (which matchup is on which court in which slot), and any scoring,
  ranking, tie or seeding rule: those are the workbook's and the site's.
- Games to settle teams level in the standings (Ladies' Dink's): the break keeps room for them.
- Rest between matchups, and gaps inside the playoffs: planned as a longer break (Q10).
- Several divisions in one plan: a day with two divisions on separate courts is two plans.
- Team names, rosters, lineups, row codes, bracket letters, a bracket chart, categories
  inside a team event, a best-of-3 final, a Final without a Bronze.
- Any change to the Team Tournament Master, the event-site template, `sage-tools-api`,
  `event-data` or `tools/sw.js`.

---
**Related:** [Tournament Calculator usage](../../features/tournament-calculator.md) ·
[technical](../../technical/tournament-calculator.md) ·
[Team Tournament Master](team-tournament-master-spec.md) ·
[Team tournament event-site template](../implemented/team-tournament-template-spec.md) ·
[Millennium Invitational](../implemented/millennium-invitational-spec.md) ·
[Ladies' Dink](../implemented/ladies-dink-club-tournament-spec.md)
