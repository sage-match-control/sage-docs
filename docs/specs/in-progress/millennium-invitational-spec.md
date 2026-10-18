# Spec — Millennium Invitational (team tournament)

> **Status: in progress.** Written 2026-10-18 against `sage-match-control.github.io`
> at `7f47c18`, `event-data`'s current `config/events.json`, the event workbook
> as exported that day, and the organiser's format and scoring sheet (§1.1). The
> owner answered every open question on 18 October 2026; the answers are in §17.
>
> **Built, 2026-10-18,** on the site repo's `millennium-invitational` branch: §4
> (the engine's `teamScoring: "games"`, with its tests) and §5–§12 (the event's
> pages), checked in a browser on Appendix A's fixture (§13, since removed), and
> §14's `events.json` entry, committed in `event-data`. **Not yet pushed:** neither
> site push (§4.8) nor the `event-data` push has been made, and each waits for the
> owner's yes. **Still to do:** `assets/qr.png` and the short link (§2), the workbook
> fixes (§16.1), sync setup (§16.2) and the dry run (§16.5).
>
> **Dry-run on 2026-10-18.** §4's code and §4.6's tests were applied to a copy of
> the site at `7f47c18`: the new tests fail before the change and pass after it,
> and the rest of the unit suite still passes. §5–§12's edits were applied to a
> copy of the template, every anchor matched once and every check passed, and the
> pages rendered with Appendix A's fixture. The comparison harness (§4.7) was not
> run.
>
> **Deadline.** The event is on **Saturday 24 October 2026**, and the site repo
> takes no merge within 3 days of an event. Both pushes of §4.8 (the engine change,
> then the pages) reach `main` by **Tuesday 20 October**. After that, push only with
> the owner's go-ahead.

Build the event site for the **Millennium Pickleball Club 2nd Invitational
Tournament**, "Paying It Forward!", a one-day charity team tournament on
**Saturday 24 October 2026**, 1:00 PM to 9:00 PM, at **Plus 63 Pickleball
Courts, Magalang, Pampanga**. All proceeds go to the **Children's Home of Mary
the Eucharist**. Plus 63 Pickleball Club is the official partner. S.A.G.E. is the
official match controller.

The site is an instance of `_templates/team-tournament-template/`, the second
`"team"` event after PickleDrive and the first one made from the template. It
needs one engine change first: this event's teams score **games won**, where
PickleDrive's scored **total points** (§3). The engine change is opt-in per event
and changes nothing for any other page (§4).

The runbook for instantiating a template is
`sage-match-control.github.io/_templates/CLAUDE.md`. This spec gives every value
and every edit, so you should not need to make any design decisions. If this
spec and the runbook disagree, **stop and ask**. Don't pick one.

---

## 0. How to use this spec

You can carry this out with no other context: every value, every edit, every test
and every check is here. Read §0 to §4 before you change anything.

### 0.1 Rules

- **Find edits by their anchor text, not line numbers.** Every edit quotes the
  exact text to find, and each anchor occurs once in its file. If an anchor
  isn't there, or is there twice, **stop and report**. Don't guess.
- **Use Git Bash** for the commands and checks. Python is not installed on this
  machine; Node is (22.23.3 or later).
- **The engine (`lib/v1/`) is edited only compatibly** (`_templates/CLAUDE.md` §8):
  add an export, an optional parameter or a field; never remove, rename, or
  change what an existing call returns when called as before. §4 is written that
  way. If a change seems to need more, stop and report it.
- **Test first.** For each part of §4, add its tests from §4.6, run `npm test` and
  see them fail, then make the change and see them pass.
- **Never weaken or delete a test to make a change pass.** If an existing test
  fails, your change is wrong.
- **Don't edit `_templates/`** except the documentation paragraph in §4.9, and
  **never edit a finished event's pages**: `events/pickledrive-anniversary-2026/`,
  `events/piggleball-2026/`, `events/pickle-for-sight-2026/`,
  `events/pnf-x-bup-dual-meet/`, `events/archives/`.
- **Commit** after each step of §0.3 marked *commit*, with a plain message saying
  what changed, ending with the attribution line your session's instructions give.
- **Never push, and never merge into `main`, without the owner's yes in chat.**
  Pushing `main` publishes the site. §0.3 marks the two points where you stop and
  ask.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.2 Setup

The site repo is `D:\Personal\SAGE\sage-match-control.github.io`. Paths in
§4–§13 are relative to it.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git status --short
```

This must print only `?? events/millennium-invitational-2026/` (the staged logo
and pubmat, §2). Anything else: stop and report.

```bash
git checkout main
git pull
git checkout -b millennium-invitational
cd _tests
npm install
npm test
cd ..
```

`npm test` must pass before you change anything. If it doesn't, stop and report.

### 0.3 Order of work

1. §4.1, with its tests. *Commit.*
2. §4.2, with its tests. *Commit.*
3. **Stop: Push 1** (§4.8). Ask the owner; continue on the branch either way.
4. §4.3, §4.4, §4.5, each with its tests. *Commit each.*
5. §5–§12, the event's pages. *Commit once.*
6. §13, the browser check, on a temporary fixture that is never committed.
7. §4.7, the full test run.
8. **Stop: Push 2** (§4.8).
9. §14, in `D:\Personal\SAGE\event-data`. *Commit*; push only on the owner's yes.
10. §4.9, the documentation, in the repos it names. *Commit each repo.*
11. §15, report back.

---

## 1. Event facts

| Fact | Value |
|---|---|
| Organiser | Millennium Pickleball Club |
| Event name | 2nd Invitational Tournament, "Paying It Forward!" — a charity event |
| Date | Saturday, 24 October 2026, one day. Pubmat: 1:00 PM to 9:00 PM |
| Venue | Plus 63 Pickleball Courts, Magalang, Pampanga. One facility, 8 courts |
| Partner | Plus 63 Pickleball Club (official partner). S.A.G.E. is the official match controller |
| Beneficiary | Children's Home of Mary the Eucharist. All proceeds go to it |
| Pubmat slogan | "Good people, great games, brighter tomorrows" |
| Teams | 8 of 8 players: `Red`, `Orange`, `Yellow`, `Green`, `Blue`, `Purple`, `Sky`, `Pink`. The colour is the team code |
| Pairs in a matchup | 4 games: pair 1 Men's Doubles, pair 2 Men's Doubles, pair 3 Women's Doubles, pair 4 Mixed Doubles |
| Matches | 120, numbered 1–120: 112 in the round robin (28 matchups), 4 in the Bronze matchup, 4 in the Final |
| Times | 30-minute slots. First serve 1:00 PM, last round-robin slot 7:30 PM, Bronze 8:00 PM, Final 8:30 PM |
| Site title | **Millennium Pickleball Club** (browser tab, Control Center, footer, schedule board). The hero reads "2nd Invitational Tournament" |
| Check-in and scoring | Check-in desk links (`"attendance": "desks"`) and scorer links for the courts (`"scoreEntry": "links"`) |
| Short link | `tinyurl.com/SAGExMPC2026` → `https://sage-match-control.github.io/events/millennium-invitational-2026/`. The owner creates it; its QR image is to follow (§2) |

### 1.1 Format and scoring — the organiser's

From the organiser's sheet, sent 18 October 2026:

- **Format.** A single round robin, every team against every other. Each team's
  captain chooses its doubles pairs per category for each round, submitted to
  match control round by round, and a pair may not be repeated after a round.
  The **four teams with the most wins** go on: **3rd and 4th** play for 3rd place,
  **1st and 2nd** play for 2nd place and overall champion.
- **Ranking.** Teams are ranked by **number of wins**: the games they have won
  in the round robin.
- **Ties**, among two or more teams level on wins:
  1. **total points** won over all their round-robin games. The higher ranks
     higher;
  2. still level: **head-to-head wins**, the games each won against the other
     tied team(s);
  3. still level: **head-to-head points**, the points each won in those games.

The organiser's first sheet gave point differential, then head-to-head wins. The
owner revised it on 18 October 2026 to the order above. Their last key reads
"then points won"; total points are already the first tiebreaker, so it is read
as the points won in the head-to-head games; the owner confirmed it (§17 Q2).

**The site shows none of these:** the two sponsorship phone numbers, the Plus 63
club logo, the pubmat's ball and court artwork. The site is a live results hub
for players who have already entered.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| Club logo | **Staged** at `events/millennium-invitational-2026/assets/logo.png`: 512×512 RGBA. It is the user's 2000×2000 PNG cropped to the badge (bounding box 593–1406 × 592–1407 of the original) with everything outside the badge's circle made transparent, because the badge filled only 41% of the original's width and would have shown at a third of the hero's 56px tile. Use it as is | §6 `{{EVENT_LOGO}}` |
| Pubmat | **Staged** at `events/millennium-invitational-2026/assets/pubmat.webp` (614×753, a phone screenshot). Reference for §8–§9 only. No page loads it. The user may send a sharper copy. If it arrives, replace this file; the sampled colours in §8 don't need redoing unless they visibly differ | — |
| Format and scoring sheet | Two images from the organiser, transcribed in §1.1 | §1.1, §4 |
| QR image for the short link | **To follow from the owner** as `events/millennium-invitational-2026/assets/qr.png`, encoding `tinyurl.com/SAGExMPC2026`. If it isn't there yet, do everything else and report it missing. Don't generate a QR image | Template QR panel |
| Event workbook | `1ulbHdH5iU5lNm6fqpvbbvxfdht-cMvjGrtoZaN8ru6k`, "2026-10-24 Millennium Pickleball Club", owned by `sagematchcontrol@gmail.com`. A copy of PickleDrive's workbook being adapted by hand. §16.1 lists what is still PickleDrive's | §14, §16 |

---

## 3. How this event differs from PickleDrive

The team template was built from PickleDrive's pages, and the runbook says only
an event of PickleDrive's shape can have its workbook copied from PickleDrive's.
This event is **not** that shape. The owner copied the workbook anyway and is
adapting it by hand. Most of the site side still fits, because the engine reads
every team-event fact from the published data rather than from a fixed format.
The scoring rule does not.

| | PickleDrive | This event | What the engine does with it |
|---|---|---|---|
| Teams | 15, codes `A`–`O`, each with a name | 8, codes are colours (`Red` …) | `parseSideCode` takes everything before `_` as the side, so `Red_3` is side `Red`, pair 3. A side with no `-` is a group side. **Fits, as long as every team has a name** in the workbook: a qualifier typed into a playoff slot reads `Seed Red · TBD` until its team has a name. §16.1 item 2 names each team, with the colour itself when it has no other name |
| Brackets | 3 of 5 | 1 of 8 | Standings draws one "Bracket 1" table; the Teams tab one "Bracket 1" heading. **Fits** |
| Group stage | Round robin per bracket | One full round robin: 7 rounds of 4 matchups | A matchup is the rows sharing a `matchUp` value. **Fits** |
| Playoffs | QF, SF, Bronze, Final (8 seeds) | Bronze and Final only (4 seeds) | Standings draws a section only for a playoff stage that has matchups, so only **Bronze** and **Final** appear. **Fits** |
| Pairs | MD, WD, XD, XD | MD, MD, WD, XD | `display.pairs` in `events.json` (§14); the repeated type is numbered "MD 1", "MD 2". **Fits** |
| Courts | 10 | 8 | The board takes the court count from the highest `CourtAssignment`. **Fits** |
| **Matchup winner** | More **total points** over the 4 games; pair wins never decide it | More **games won** (§1.1) | `teamMatchupResult` adds points. **Wrong for this event**: a team winning 3 games 11–9 and losing one 2–11 would be shown losing 35–38 |
| **Ranking** | Points scored, quotient, head-to-head points, pair wins | Games won, total points, head-to-head wins, head-to-head points | `teamRankBracket` sorts by points scored from `STANDINGSCSV` first. **Wrong for this event** |

The two wrong rows reach the Hub's matchup cards and Bracket 1 table, the Live
Matches rows, Control Center's team views and its **Awards** podium
(`buildTeamPodium` reads each matchup's winner). §4 adds the event's rule as an
option, chosen by a new `teamScoring` field in `events.json`.

**Two consequences of the format to expect, both correct:**

- The four teams typed into the Bronze and Final slots each get an
  **Advances** pill in the Bracket 1 table, the 3rd- and 4th-placed teams
  included. "Advances" here means "plays a playoff matchup".
- Before the organiser types the qualifiers in, a playoff side reads
  `Seed 1 · TBD`. §16.4 fixes the seed numbers so that "Seed 1" is always the
  better-placed team.

---

## 4. Engine change — `teamScoring: "games"`

A team event may set `"teamScoring": "games"` in `events.json`. Absent (or
`"points"`), everything behaves exactly as today. With `"games"`:

- a matchup is won by the side that won **more of its games**. Level on games, the
  side with **more total points** wins, the organiser's first tiebreaker. Still
  level, it is a **tie**, shown as today (§17 Q3);
- a bracket is ranked by **games won**, then **total points**, then, among the
  teams still level, **head-to-head wins** and then **head-to-head points**, then
  the sheet's order (§1.1). Only played group-stage games count, all computed
  from the matches, so every key comes from the same scores;
- the bracket table shows **W**, **L** and **Pts** (total points won) instead of
  PF, PA and Quotient, and a matchup's headline score is its **games** (`3 – 1`),
  with the points underneath.

A game counts as won by the side with the higher score once both scores are in
(the `played` rule of `lib/v1/domain/matches.js`). Equal scores count for neither
side. The awards podium needs no edit: it reads each matchup's `result.winner`.

### 4.1 `lib/v1/data/registry.js` — read the field

In `eventConfigFrom`, find:

```js
    scoreEntry: raw.scoreEntry,
    attendance: raw.attendance,
  };
```

Replace with:

```js
    scoreEntry: raw.scoreEntry,
    attendance: raw.attendance,
    teamScoring: teamScoringFrom(raw.teamScoring, eventKey),
  };
```

and add, directly above the `/**` comment of `eventConfigFrom`:

```js
// A team event's scoring (sage-docs/docs/specs/.../millennium-invitational-spec.md §4): "points" (PickleDrive's
// rule, the default) or "games". Anything else warns and reads as "points".
function teamScoringFrom(raw, eventKey){
  if(raw === undefined || raw === 'points') return 'points';
  if(raw === 'games') return 'games';
  console.warn(`events.json: teamScoring for ${eventKey} must be "points" or "games"; using "points"`);
  return 'points';
}
```

Add to that function's doc comment, after the sentence about `pairs`:
"`teamScoring` is a team event's scoring rule, `"points"` unless events.json
says `"games"`."

In `lib/v1/domain/model.js`'s `EventConfig` typedef, after the `attendance`
line, add:

```js
 * @property {"points"|"games"} teamScoring     a team event's scoring rule (events.json teamScoring; "points" without it)
```

The tests keep their own copy of this shape: `eventConfig` in
`_tests/helpers/fixtures.mjs`, which the existing test `eventConfigFrom agrees with
the fixture events` compares against. Find, in that function:

```js
    scoreEntry: e.scoreEntry,
    attendance: e.attendance,
  };
```

Replace with:

```js
    scoreEntry: e.scoreEntry,
    attendance: e.attendance,
    teamScoring: e.teamScoring === 'games' ? 'games' : 'points',
  };
```

`sage-tools-api` reads only the event fields it needs, so the new field is
ignored there and needs no API change.

### 4.2 `lib/v1/domain/teams.js` — the rule

Every change below adds an optional parameter whose default is today's
behaviour.

**`teamMatchupResult`.** Change the signature to
`teamMatchupResult(ms, scoring = 'points')`. Replace its decision line:

```js
  else if(pts1 === pts2) state = 'tie';
  else { state = 'final'; winner = pts1 > pts2 ? 1 : 2; }
  return { side1, side2, pts1, pts2, pairs1, pairs2, played, total, state, winner };
```

with:

```js
  else if(scoring === 'games' && pairs1 !== pairs2){ state = 'final'; winner = pairs1 > pairs2 ? 1 : 2; }
  else if(pts1 === pts2) state = 'tie';
  else { state = 'final'; winner = pts1 > pts2 ? 1 : 2; }
  return { side1, side2, pts1, pts2, pairs1, pairs2, played, total, state, winner, scoring };
```

and rewrite its doc comment's first line to: "ms = the matchup's matches (same
matchUp). `scoring` "points" (the default): won on total points across its
matches; pair wins never decide it. "games": won on matches won (`pairs1`/`pairs2`),
then total points; equal on both is a tie."

**`buildTeamData`.** Change the signature to
`buildTeamData(matches, standings, { scoring = 'points' } = {})`, pass `scoring`
as the second argument of its `teamMatchupResult(ms)` call, and return
`{ rowByCode, matchups, advancing, scoring }`. Add to its doc comment:
"`scoring` the event's teamScoring, echoed on the result for the views".

**New export `teamGameRecords`**, directly above `teamRankBracket`:

```js
/**
 * Each side's record in the day's played group-stage matches, for "games"
 * scoring: matches won and lost, and points for and against. A match with equal
 * scores counts for neither side's W or L.
 * @param {object[]} matches  the day's matches
 * @returns {Map<string, { won:number, lost:number, pf:number, pa:number }>}  keyed by side
 */
export function teamGameRecords(matches){
  const out = new Map();
  const rec = side => {
    if(!out.has(side)) out.set(side, { won: 0, lost: 0, pf: 0, pa: 0 });
    return out.get(side);
  };
  matches.forEach(m => {
    if(!m.played || stageOf(sideOf(m.t1)) !== null) return;
    const a = rec(sideOf(m.t1)), b = rec(sideOf(m.t2));
    a.pf += m.t1Score; a.pa += m.t2Score;
    b.pf += m.t2Score; b.pa += m.t1Score;
    if(m.t1Score > m.t2Score){ a.won++; b.lost++; }
    else if(m.t2Score > m.t1Score){ b.won++; a.lost++; }
  });
  return out;
}
```

**`teamRankBracket`.** Change the signature to
`teamRankBracket(rows, matches, scoring = 'points')`. At its top, before the
existing `const groupMatches = …` line, add:

```js
  if(scoring === 'games') return teamRankBracketByGames(rows, matches);
```

and directly below `teamRankBracket`, add:

```js
/**
 * Bracket ranking for "games" scoring (the Millennium Invitational's rules,
 * sage-docs/docs/specs/.../millennium-invitational-spec.md §1.1):
 *   1. matches won in the bracket's played group matches
 *   2. total points won in those matches
 *   3. head-to-head wins: among the teams still level, matches won against each other
 *   4. head-to-head points: the points each won in those matches
 * Rows level on all four keep their STANDINGSCSV order (the sorts are stable).
 */
function teamRankBracketByGames(rows, matches){
  const recs = teamGameRecords(matches);
  const recOf = code => recs.get(code) || { won: 0, lost: 0, pf: 0, pa: 0 };
  const byWonPf = rows.slice().sort((a, b) =>
    (recOf(b.teamCode).won - recOf(a.teamCode).won) || (recOf(b.teamCode).pf - recOf(a.teamCode).pf));
  const out = [];
  for(let i = 0; i < byWonPf.length;){
    const ri = recOf(byWonPf[i].teamCode);
    let j = i + 1;
    while(j < byWonPf.length && recOf(byWonPf[j].teamCode).won === ri.won && recOf(byWonPf[j].teamCode).pf === ri.pf) j++;
    const tied = byWonPf.slice(i, j);
    if(tied.length > 1){
      const codes = new Set(tied.map(s => s.teamCode));
      const h2hWins = new Map(tied.map(s => [s.teamCode, 0]));
      const h2hPts = new Map(tied.map(s => [s.teamCode, 0]));
      matches.forEach(m => {
        if(!m.played || stageOf(sideOf(m.t1)) !== null) return;
        const s1 = sideOf(m.t1), s2 = sideOf(m.t2);
        if(!codes.has(s1) || !codes.has(s2)) return;
        h2hPts.set(s1, h2hPts.get(s1) + m.t1Score);
        h2hPts.set(s2, h2hPts.get(s2) + m.t2Score);
        if(m.t1Score > m.t2Score) h2hWins.set(s1, h2hWins.get(s1) + 1);
        else if(m.t2Score > m.t1Score) h2hWins.set(s2, h2hWins.get(s2) + 1);
      });
      tied.sort((a, b) =>
        (h2hWins.get(b.teamCode) - h2hWins.get(a.teamCode)) ||
        (h2hPts.get(b.teamCode) - h2hPts.get(a.teamCode)));
    }
    out.push(...tied);
    i = j;
  }
  return out;
}
```

Update the file's header comment: after "a matchup is won on total points", add
"(or, for an event with `teamScoring: "games"`, on matches won; see
`teamMatchupResult`)".

### 4.3 `lib/v1/domain/model.js` — pass the event's rule

In `buildDayModel`, find:

```js
    model.team = { ...buildTeamData(matches, standings), roster, rosterByCode: byCode };
```

Replace with:

```js
    model.team = { ...buildTeamData(matches, standings, { scoring: event.teamScoring }), roster, rosterByCode: byCode };
```

### 4.4 `lib/v1/views/teams.js` — show it

Add `teamGameRecords` to the import from `../domain/teams.js`. Every branch
below reads `model.team.scoring === 'games'`; the `else` is today's markup,
unchanged.

**Matchup card** (`matchupCardHTML`). Find:

```js
  const middle = started
    ? `<div class="tm-mid"><div class="tm-score">${ptsOf(L)} – ${ptsOf(R)}</div><div class="tm-pairs">pairs ${L === 1 ? r.pairs1 : r.pairs2}–${R === 1 ? r.pairs1 : r.pairs2}</div></div>`
    : `<div class="tm-mid"><div class="tm-vs">VS</div></div>`;
```

Replace with:

```js
  const gamesOf = n => n === 1 ? r.pairs1 : r.pairs2;
  // teamScoring "games": the headline is matches won, points underneath; "points": the other way round.
  const middle = !started
    ? `<div class="tm-mid"><div class="tm-vs">VS</div></div>`
    : model.team.scoring === 'games'
      ? `<div class="tm-mid"><div class="tm-score">${gamesOf(L)} – ${gamesOf(R)}</div><div class="tm-pairs">points ${ptsOf(L)}–${ptsOf(R)}</div></div>`
      : `<div class="tm-mid"><div class="tm-score">${ptsOf(L)} – ${ptsOf(R)}</div><div class="tm-pairs">pairs ${gamesOf(L)}–${gamesOf(R)}</div></div>`;
```

**Live Matches row.** Find:

```js
  const matchupHTML = mu ? `Matchup ${mu.result.pts1} – ${mu.result.pts2}` : '&nbsp;';
```

Replace with:

```js
  const matchupHTML = !mu ? '&nbsp;'
    : model.team.scoring === 'games' ? `Matchup ${mu.result.pairs1} – ${mu.result.pairs2}`
    : `Matchup ${mu.result.pts1} – ${mu.result.pts2}`;
```

**Bracket table** (`groupTableHTML`). At the top of the function, before
`// Rank numbers appear once any team in the bracket has a result.`, add:

```js
  if(model.team.scoring === 'games') return groupTableByGamesHTML(model, group, rows, hooks);
```

and directly below `groupTableHTML`, add:

```js
// teamScoring "games": W, L and total points won from the played group matches; rank numbers once any of the
// bracket's teams has played.
function groupTableByGamesHTML(model, group, rows, hooks){
  const recs = teamGameRecords(model.matches);
  const recOf = code => recs.get(code) || { won: 0, lost: 0, pf: 0, pa: 0 };
  const anyPlayed = rows.some(s => { const r = recOf(s.teamCode); return r.pf + r.pa > 0; });
  const ranked = anyPlayed ? teamRankBracket(rows, model.matches, 'games') : rows;
  const label = group === '' ? 'No bracket' : `Bracket ${group}`;
  return `<div class="group-card">
    <h3 class="group-title">${escapeHtml(label)}</h3>
    <div class="gt-wrap" data-group-key="${escapeHtml(group)}">
      <table class="gt">
        <thead><tr><th class="rk">${anyPlayed ? '#' : ''}</th><th>Team</th><th class="num">W</th><th class="num">L</th><th class="num">Pts</th></tr></thead>
        <tbody>${ranked.map((s, i) => { const r = recOf(s.teamCode); return `<tr>
          <td class="rk">${anyPlayed ? i + 1 : ''}</td>
          <td class="tm">${escapeHtml(s.teamName || s.teamCode)}${hooks.teamLetters === false ? '' : `<span class="chip">${escapeHtml(s.teamCode)}</span>`}${model.team.advancing.has(s.teamCode) ? '<br><span class="adv-pill">Advances</span>' : ''}</td>
          <td class="num">${r.won}</td>
          <td class="num">${r.lost}</td>
          <td class="num">${r.pf}</td>
        </tr>`; }).join('')}</tbody>
      </table>
    </div>
  </div>`;
}
```

`model.team.scoring` is undefined in any model built before this change (or by a
cached older `teams.js`), which reads as "points".

### 4.5 `tools/control-center.html` — pass the rule when a score is saved in fixture mode

Control Center builds its day model through `buildDayModel`, so §4.3 covers it,
but one fixture-mode save rebuilds the team data itself. Find:

```js
    if(CURRENT_TYPE === 'team') MODEL.team = { ...MODEL.team, ...Teams.buildTeamData(MODEL.matches, MODEL.standings) };
```

Replace with:

```js
    if(CURRENT_TYPE === 'team') MODEL.team = { ...MODEL.team, ...Teams.buildTeamData(MODEL.matches, MODEL.standings, { scoring: CURRENT_EVENT_CONFIG && CURRENT_EVENT_CONFIG.teamScoring }) };
```

`CURRENT_EVENT_CONFIG` is the selected event's `eventConfigFrom` result: it is
declared near the top of the page's script and set when an event is chosen
(`CURRENT_EVENT_CONFIG = eventConfig;`).

### 4.6 Tests

Append each block below to the **end** of its file, with a blank line before it,
when you do the part of §4 it belongs to. Every name they use is already imported
in that file: `test`, `assert`, `mock` and `eventConfigFrom` in the data tests;
`T` (`domain/teams.js`), `A` (`domain/awards.js`), `buildDayModel` and the
`tm(t1, s1, t2, s2)` match helper in the domain tests; `buildDayModel`,
`groupTableHTML`, `matchupCardHTML` and `liveRowHTML` in the view tests. Before
the change, `npm test` (from `_tests/`) must show each new test failing; after it,
every test passing.

**With §4.1**, together with the helper change in §4.1, append to
`_tests/unit/data/data-modules.test.mjs`:

```js
// ---- teamScoring (sage-docs/docs/specs/.../millennium-invitational-spec.md §4) -------------------------------------

test('eventConfigFrom reads teamScoring: "points" unless the event says "games"', t => {
  const warn = mock.method(console, 'warn', () => {});
  t.after(() => warn.mock.restore());
  const registry = { events: { g: { type: 'team', teamScoring: 'games' }, p: { type: 'team', teamScoring: 'points' }, none: { type: 'team' }, odd: { type: 'team', teamScoring: 'wins' } } };
  assert.equal(eventConfigFrom(registry, 'g').teamScoring, 'games');
  assert.equal(eventConfigFrom(registry, 'p').teamScoring, 'points');
  assert.equal(eventConfigFrom(registry, 'none').teamScoring, 'points');
  assert.equal(warn.mock.calls.length, 0);
  assert.equal(eventConfigFrom(registry, 'odd').teamScoring, 'points');
  assert.equal(warn.mock.calls.length, 1);
  assert.equal(warn.mock.calls[0].arguments[0], 'events.json: teamScoring for odd must be "points" or "games"; using "points"');
});
```

The existing test `eventConfigFrom agrees with the fixture events` fails after
the `registry.js` change until the helper change is made too; that is expected,
and the two go in one commit.

**With §4.2**, append to `_tests/unit/domain/teams-awards-attendance-grid-model.test.mjs`:

```js
// ---- teamScoring "games" (sage-docs/docs/specs/.../millennium-invitational-spec.md §4) ------------------------------

test('teamScoring "games": a matchup is won on matches won, then total points', () => {
  // Side 1 wins three matches but scores fewer points: it wins on games, and would lose on points.
  const ms = [tm('A_1', 11, 'B_1', 9), tm('A_2', 11, 'B_2', 9), tm('A_3', 11, 'B_3', 9), tm('A_4', 2, 'B_4', 11)];
  const g = T.teamMatchupResult(ms, 'games');
  assert.deepEqual([g.pts1, g.pts2, g.pairs1, g.pairs2, g.state, g.winner, g.scoring], [35, 38, 3, 1, 'final', 1, 'games']);
  const p = T.teamMatchupResult(ms);
  assert.deepEqual([p.winner, p.scoring], [2, 'points'], 'the default still decides on points');
  // 2–2 on games: more total points wins.
  const split = T.teamMatchupResult([tm('A_1', 11, 'B_1', 0), tm('A_2', 11, 'B_2', 0), tm('A_3', 9, 'B_3', 11), tm('A_4', 9, 'B_4', 11)], 'games');
  assert.deepEqual([split.pairs1, split.pairs2, split.pts1, split.pts2, split.state, split.winner], [2, 2, 40, 22, 'final', 1]);
  // Level on games and on points: a tie.
  assert.equal(T.teamMatchupResult([tm('A_1', 11, 'B_1', 5), tm('A_2', 5, 'B_2', 11)], 'games').state, 'tie');
  assert.equal(T.teamMatchupResult([tm('A_1', 11, 'B_1', 3), tm('A_2', null, 'B_2', null)], 'games').state, 'in-progress');
  assert.equal(T.teamMatchupResult([tm('A_1', null, 'B_1', null)], 'games').state, 'not-started');
});

test('teamGameRecords: W, L, PF and PA from played group matches only', () => {
  const recs = T.teamGameRecords([
    tm('A_1', 11, 'B_1', 7), tm('A_2', 4, 'B_2', 11),
    tm('A_3', 9, 'B_3', 9),                         // equal scores: points count, no W or L
    tm('A_4', null, 'B_4', null),                   // not played
    tm('Fi-A_1', 11, 'Fi-B_1', 0),                  // a playoff match
  ]);
  assert.deepEqual(recs.get('A'), { won: 1, lost: 1, pf: 24, pa: 27 });
  assert.deepEqual(recs.get('B'), { won: 1, lost: 1, pf: 27, pa: 24 });
  assert.equal(recs.has('Fi-A'), false);
});

test('teamScoring "games": bracket ranking by wins, total points, head-to-head wins, head-to-head points', () => {
  const rows = (...codes) => codes.map(teamCode => ({ teamCode, pf: 0, quotient: null }));
  const rank = (codes, matches) => T.teamRankBracket(rows(...codes), matches, 'games').map(r => r.teamCode);
  // 1–2. C wins two; A and B one each, and B has more total points (16 to 14).
  assert.deepEqual(rank(['A', 'B', 'C'], [
    tm('C_1', 11, 'A_1', 0), tm('C_2', 11, 'B_2', 0), tm('A_3', 11, 'B_3', 5), tm('B_4', 11, 'A_4', 3),
  ]), ['C', 'B', 'A']);
  // 3. A, B and C each win two and score 32; among themselves A won two, B one, C none.
  assert.deepEqual(rank(['C', 'B', 'A', 'D'], [
    tm('A_1', 11, 'B_1', 9), tm('A_2', 11, 'C_2', 1), tm('B_3', 11, 'C_3', 0),
    tm('D_1', 12, 'A_3', 10), tm('B_4', 12, 'D_2', 10), tm('C_4', 11, 'D_3', 9), tm('C_1', 20, 'D_4', 18),
  ]), ['A', 'B', 'C', 'D']);
  // 4. A and B each win one, score 23 and beat each other once: A won more points against B (20 to 13).
  assert.deepEqual(rank(['B', 'A', 'C'], [
    tm('A_1', 11, 'B_1', 2), tm('B_2', 11, 'A_2', 9), tm('C_3', 11, 'A_3', 3), tm('C_4', 12, 'B_4', 10),
  ]), ['C', 'A', 'B']);
  // Level on all four, or nothing played: the sheet's order. A playoff match counts for nothing.
  assert.deepEqual(rank(['B', 'A'], []), ['B', 'A']);
  assert.deepEqual(rank(['A', 'B'], [tm('Fi-A_1', 0, 'Fi-B_1', 11)]), ['A', 'B']);
  // The default ranking is unchanged by the new parameter.
  assert.deepEqual(T.teamRankBracket([{ teamCode: 'A', pf: 1, quotient: 1 }, { teamCode: 'B', pf: 2, quotient: 1 }], []).map(r => r.teamCode), ['B', 'A']);
});
```

**With §4.3**, append to the same file:

```js
// A team day with one Final: Alpha wins three games on fewer points (35 to 38).
const TEAM_MATCH_HEADER = 'matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score';
const finalDay = teamScoring => buildDayModel(
  { key: 'e', type: 'team', display: {}, days: [], title: 'E', teamScoring },
  { key: 'd', date: '2026-10-24' },
  { isLive: true, facilities: [{ name: 'Main',
    matchesCsv: [TEAM_MATCH_HEADER,
      '1,Fi-A v Fi-B,,8:30 PM,Court 1,Fi-A_1,a1,a2,11,Fi-B_1,b1,b2,9',
      '2,Fi-A v Fi-B,,8:30 PM,Court 2,Fi-A_2,a3,a4,11,Fi-B_2,b3,b4,9',
      '3,Fi-A v Fi-B,,8:30 PM,Court 3,Fi-A_3,a5,a6,11,Fi-B_3,b5,b6,9',
      '4,Fi-A v Fi-B,,8:30 PM,Court 4,Fi-A_4,a7,a8,2,Fi-B_4,b7,b8,11'].join('\n'),
    standingsCsv: 'teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket\nA,Alpha,0,0,-,1\nB,Beta,0,0,-,1\nFi-1,,0,0,-,\nFi-2,,0,0,-,' }] },
  { now: 0, alwaysLive: true },
);

test('teamScoring reaches the day model and decides the podium', () => {
  const games = finalDay('games');
  assert.equal(games.team.scoring, 'games');
  assert.deepEqual([A.buildTeamPodium(games).gold.teamName, A.buildTeamPodium(games).silver.teamName], ['Alpha', 'Beta']);
  const points = finalDay('points');
  assert.equal(points.team.scoring, 'points');
  assert.equal(A.buildTeamPodium(points).gold.teamName, 'Beta');
});
```

**With §4.4**, append to `_tests/unit/views/teams.test.mjs`:

```js
// ---- teamScoring "games" (sage-docs/docs/specs/.../millennium-invitational-spec.md §4) ------------------------------

// Alpha beats Beta three games to one on fewer points (35 to 38); Gamma hasn't played.
const GAMES_HEADER = 'matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score';
const gamesModel = teamScoring => buildDayModel(
  { key: 'e', type: 'team', display: {}, days: [], title: 'E', teamScoring },
  { key: 'd', date: '2026-10-24' },
  { isLive: true, facilities: [{ name: 'Main',
    matchesCsv: [GAMES_HEADER,
      '1,A v B,Court 1,1:00 PM,Court 1,A_1,a1,a2,11,B_1,b1,b2,9',
      '2,A v B,,1:00 PM,Court 2,A_2,a3,a4,11,B_2,b3,b4,9',
      '3,A v B,,1:00 PM,Court 3,A_3,a5,a6,11,B_3,b5,b6,9',
      '4,A v B,,1:00 PM,Court 4,A_4,a7,a8,2,B_4,b7,b8,11'].join('\n'),
    standingsCsv: 'teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket\nG,Gamma,0,0,-,1\nB,Beta,38,35,1.0857,1\nA,Alpha,35,38,0.9211,1' }] },
  { now: 0, alwaysLive: true },
);

test('teamScoring "games": the bracket table shows W, L and Pts, ranked on games', () => {
  const m = gamesModel('games');
  const html = groupTableHTML(m, '1', m.standings);
  assert.match(html, /<th class="num">W<\/th><th class="num">L<\/th><th class="num">Pts<\/th>/);
  assert.doesNotMatch(html, /Quotient/);
  const names = [...html.matchAll(/<td class="tm">([A-Za-z]+)/g)].map(x => x[1]);
  assert.deepEqual(names, ['Alpha', 'Beta', 'Gamma']);
  assert.match(html, /<td class="rk">1<\/td>\s*<td class="tm">Alpha[\s\S]*?<td class="num">3<\/td>\s*<td class="num">1<\/td>\s*<td class="num">35<\/td>/);
  // Before any game, no rank numbers and the sheet's order.
  const fresh = buildDayModel(m.event, m.day, { isLive: true, facilities: [{ name: 'Main', matchesCsv: GAMES_HEADER, standingsCsv: 'teamCode,teamName,bracket\nG,Gamma,1\nA,Alpha,1' }] }, { now: 0, alwaysLive: true });
  const freshHtml = groupTableHTML(fresh, '1', fresh.standings);
  assert.doesNotMatch(freshHtml, /<td class="rk">1<\/td>/);
  assert.deepEqual([...freshHtml.matchAll(/<td class="tm">([A-Za-z]+)/g)].map(x => x[1]), ['Gamma', 'Alpha']);
});

test('teamScoring "games": the matchup card and the Live row lead with games', () => {
  const m = gamesModel('games');
  const card = matchupCardHTML(m, m.team.matchups[0]);
  assert.match(card, /<div class="tm-score">3 – 1<\/div><div class="tm-pairs">points 35–38<\/div>/);
  assert.match(liveRowHTML(m, 'Court 1', m.matches[0]), /Matchup 3 – 1/);
  // The default leads with points, as before.
  const p = gamesModel(undefined);
  assert.match(matchupCardHTML(p, p.team.matchups[0]), /<div class="tm-score">35 – 38<\/div><div class="tm-pairs">pairs 3–1<\/div>/);
  assert.match(liveRowHTML(p, 'Court 1', p.matches[0]), /Matchup 35 – 38/);
});
```

§4.5 has no unit test: Control Center is an HTML page. §13 checks it in the browser.

The existing team tests (`a matchup is won on total points; pair wins never
decide it`, `bracket ranking: points, quotient, then head-to-head points, then pair
wins`, and every view test) must pass **unchanged**: they are the proof that the
default is untouched.

### 4.7 The full test run

Before each push, from `_tests/`, with Appendix A's fixture removed
(`git status --short` shows nothing under `_fixtures/`):

```bash
npm run verify
```

It runs the unit tests, then the comparison harness, which renders every case on
the baseline (`main`) and on your branch in Playwright's Chromium and compares
text and screenshots. It takes several minutes; if your tool has a time limit, run
it in the background and wait for it to finish. If Chromium is missing, run
`npx playwright install chromium` first.

It must pass with **no accepted differences**. No existing page or fixture sets
`teamScoring`, so every case renders as before. A failing case writes
`baseline.png`, `branch.png`, `diff.png` and `text.diff` to `_tests/out/<case>/`;
a difference is a bug in §4, not something to accept. Stop and report it.

### 4.8 Two pushes

A module may start using another module's new export or option only in a later
push to `main` (`_templates/CLAUDE.md` §8), because browsers keep a file up to
10 minutes:

1. **Push 1:** the branch holds §4.1 and §4.2 and nothing else. They add a field
   and optional parameters that nothing passes yet.
2. **Push 2, at least 10 minutes after Push 1:** §4.3, §4.4, §4.5 and the event's
   pages (§5–§12).

At each, run §4.7, then ask the owner in chat, saying what the push contains and
that `npm run verify` passed. Only on a yes:

```bash
git checkout main
git pull
git merge --no-ff millennium-invitational -m "Millennium Invitational: <what this push contains>"
git push origin main
git checkout millennium-invitational
```

Both pushes by **Tuesday 20 October**. If the owner says no, or doesn't answer,
carry on with the branch and report where it stands. `events.json` (§14) can be
committed to `event-data` at any time: until Push 2, `teamScoring` is read but
not used.

### 4.9 Documentation, after Push 2

Write in the present tense: what the system does, never "used to" or "now does".
Every page below describes a team matchup as won on total points; add that an
event with `teamScoring: "games"` is won on games won, then total points, and
ranked by games won, total points, head-to-head wins, head-to-head points. Find
the sentences with `grep -rn "total points" docs/` in `sage-docs`.

| Repo | File | What to add |
| --- | --- | --- |
| `sage-docs` | `docs/features/tournament-hub.md` | Beside "Why a matchup is won on total points": the games option, and that the Bracket table then shows W, L and Pts |
| `sage-docs` | `docs/features/control-center.md` | Both places a team matchup is decided on total points, including the Awards podium |
| `sage-docs` | `docs/features/team-workbook.md` | Where it says the team with more total points wins: the games option |
| `sage-docs` | `docs/technical/control-center.md` | Where `teamMatchupResult` is described: its `scoring` parameter |
| `sage-docs` | `docs/technical/site-engine.md` | `EventConfig.teamScoring`, `teamGameRecords`, and `teamRankBracket`'s `scoring` parameter |
| `sage-docs` | `docs/technical/event-data-config.md` | Beside `display.pairs`: the optional event field `teamScoring` |
| `sage-docs` | `docs/usage/glossary.md`, `docs/usage/README.md`, `docs/usage/run-the-day.md` | Their one-line team summaries: "won on total points, or on games won where the event says so" |
| `sage-docs` | this spec | Move it from `not-started/` to `in-progress/` (the site is built; the event hasn't run) by the procedure in `docs/specs/README.md`, and rewrite its status line to say what is built and pushed |
| `sage-docs` | `docs/changelog.md` and `mkdocs.yml` | A MINOR entry above the latest one, and the same version in `mkdocs.yml`'s `copyright` line (`docs/changelog.md` explains the format) |
| `event-data` | `config/README.md` | Beside `pairs`: `teamScoring`, optional, team events only, `"points"` (the default) or `"games"`, and what each decides |
| `sage-match-control.github.io` | `_templates/CLAUDE.md` | §2 step 7, after the `display.pairs` bullet: set `teamScoring: "games"` for an event scored on games won |
| workspace | `D:\Personal\SAGE\CLAUDE.md` | The bullet on the team-event rules (`lib/v1/domain/teams.js`): who wins a matchup and the bracket ranking follow the event's `teamScoring` |

---

## 5. Create the event folder

The folder already exists, because it holds the staged images. Copy the
template's **contents** into it. The trailing `/.` matters: without it, `cp`
would create a nested `team-tournament-template/` folder inside the existing
one.

```bash
cp -r _templates/team-tournament-template/. events/millennium-invitational-2026/
cp _templates/attendance/attendance.html events/millennium-invitational-2026/attendance.html
cp _templates/scorer/scorer.html events/millennium-invitational-2026/scorer.html
```

**Check:** `ls events/millennium-invitational-2026` shows `index.html`,
`schedule.html`, `attendance.html`, `scorer.html` and `assets/`, and `assets/` still contains
`logo.png` and `pubmat.webp`.

---

## 6. Replace the `{{TOKENS}}`

Replace every occurrence in `index.html`, `schedule.html`, `attendance.html` and
`scorer.html` of `events/millennium-invitational-2026/`:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `millennium-invitational-2026` |
| `{{EVENT_TITLE}}` | `Millennium Pickleball Club` |
| `{{EVENT_TAGLINE}}` | `Millennium Pickleball Club` |
| `{{EVENT_HEADLINE}}` | `Paying It Forward!` |
| `{{EVENT_DATE_RANGE}}` | `24 October 2026` |
| `{{VENUE}}` | `Plus 63 Pickleball Courts, Magalang` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr>SAGExMPC2026` |
| `{{EVENT_LOGO}}` | `assets/logo.png` |
| `{{SCHEDULE_DAY_KEY}}` | `millennium-invitational-2026-day1` |

- `attendance.html` and `scorer.html` have only `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`.
- None of these values contains `&` or a quote. `{{QR_URL}}`'s `<wbr>` is HTML,
  typed as written: it lets the link wrap after the `/` on desktop.

**Check:** this must print nothing:

```bash
grep -rn '{{' events/millennium-invitational-2026/
```

---

## 7. `index.html` — content edits

All edits are in `events/millennium-invitational-2026/index.html`. The Hub's
days, facility, pair labels and scoring come from `events.json` (§14); there is
no configuration block to fill in.

### 7.1 Hero — title

Find (as it reads after §6):

```html
  <h1 class="title">Millennium Pickleball Club</h1>
```

Replace with:

```html
  <h1 class="title">2nd Invitational<br><span class="accent">Tournament</span></h1>
```

The pubmat stacks "2ND INVITATIONAL" in white over "TOURNAMENT" in gold. The
engine's `h1.title .accent` already colours the accent with `--green`, which §8
makes the pubmat's gold. The headline directly below reads "Paying It Forward!"
from §6. The ordinal stays in the hero only (§18).

### 7.2 Hero — slogan

Find:

```html
  <p class="subtitle">Follow every team, every matchup and every court — live.</p>
```

Directly **above** it, add:

```html
  <div class="slogan-script">Good people, great games, brighter tomorrows</div>
```

The hero now stacks: Millennium logo × S.A.G.E. logo → "Millennium Pickleball
Club" → "24 OCTOBER 2026 · PLUS 63 PICKLEBALL COURTS,
MAGALANG" → "2ND INVITATIONAL / **TOURNAMENT**" → "Paying It Forward!" →
slogan → subtitle. The CSS is in §9.

### 7.3 Footer cause line

Find (as it reads after §6):

```html
<footer>
  Millennium Pickleball Club &middot; 24 October 2026 &middot; Plus 63 Pickleball Courts, Magalang<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

Replace with:

```html
<footer>
  Millennium Pickleball Club &middot; 24 October 2026 &middot; Plus 63 Pickleball Courts, Magalang<br>
  A charity event. All proceeds go to the Children's Home of Mary the Eucharist.<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

---

## 8. Theme — colours from the pubmat and the logo

The pubmat is gold and lime on a very dark green, and the club's badge is a
muted green on cream. The values below were **sampled from the images' pixels**
(the dominant colour of each region, with headless Chrome), not judged by eye:

| Source | Element | Sampled |
|---|---|---|
| Pubmat | Date and venue panels | dark green `#032519`, darker `#01140C` |
| Pubmat | "TOURNAMENT" | gold `#FCDC5B` |
| Pubmat | "PAYING IT FORWARD!" banner | lime `#BDE25B` |
| Logo | Badge ring | green `#455C4C` |
| Logo | Background | cream `#EEE8DC` |

### 8.1 The palette

The token **names** stay the same (`--navy`, `--green`, …), so every rule in the
engine's CSS keeps resolving; only their values change. The template's `--green`
is its accent *fill*, used on navy panels and under navy text. `--green-dark` is
its accent for *text and fills on light backgrounds* and for the "live" state.
Gold fills the first role and the logo's green the second.

| Token | Old (house) | New | Why |
|---|---|---|---|
| `--navy` | `#14263C` | `#032519` | Pubmat panel green: the hero, structure |
| `--navy-deep` | `#0B1826` | `#01140C` | Pubmat's darker panel green |
| `--green` | `#7CB92C` | `#FCDC5B` | Pubmat gold — accent **fill** |
| `--green-dark` | `#5C8F1F` | `#455C4C` | Logo green — accent **text**, live dot, live pill |
| `--paper` | `#F6F7F2` | `#EEE8DC` | Logo cream |
| `--paper-dim` | `#ECEEE6` | `#E3DCCD` | Cream, one step darker |
| `--line` | `#D9DED2` | `#D3CAB8` | Cream, to match |
| `--ink` | `#14263C` | `#032519` | Same as `--navy` |
| `--ink-soft` | `#5B6B74` | `#4E5D54` | Green-tinted grey |
| `--white`, `--radius` | — | unchanged | |

`index.html` also gets one new token, for the headline (§9.2):

```css
    --lime:#BDE25B;
```

Measured contrast (WCAG 2.x):

| Pair | Ratio | Use |
|---|---|---|
| `--ink` on `--paper` / `--paper-dim` | 13.42 / 12.00 | body text |
| `--ink-soft` on `--paper` / `--paper-dim` | 5.70 / 5.10 | secondary text |
| `--green-dark` on white / `--paper` / `--paper-dim` | 7.27 / 5.96 / 5.33 | accent text, stat numbers |
| white on `--green-dark` | 7.27 | live pill, `.badge-2` |
| `--navy` text on `--green` (gold) | 12.11 | active tabs, chips |
| `--green` (gold) on `--navy` / `--navy-deep` | 12.11 / 14.03 | eyebrow, title accent, sync line |
| `--lime` on `--navy` | 11.07 | headline |
| white on `--navy` | 16.37 | hero text |
| `--green` (gold) on white | **1.35 — fails** | never use `--green` as text on a light background |
| `--green-dark` on `--navy` | **2.25 — fails** | never put the logo green on the hero. §9.3 moves the three engine rules that do onto gold |

### 8.2 Edit the `:root` blocks

In **both** `index.html` and `schedule.html`, change the brand tokens in the
`:root{ … }` block to the "New" column above (in `index.html` it is under the
`THEME` banner; in `schedule.html` it is the first rule inside `<style>`, and has
the same nine tokens), and add
`--lime` to `index.html`'s, after `--radius`. Leave the role aliases in
`index.html` (`--court:var(--green);` and the rest) unchanged. They resolve
through the tokens automatically.

In `index.html`, `--card-shadow` holds a raw copy of the old `--navy-deep`.
Replace `rgba(11,24,38,.28)` in it with `rgba(1,20,12,.28)`.

### 8.3 Update the contrast note

In `index.html`'s `THEME` banner comment, replace these lines:

```
       - --green on white is ~2.3:1 and fails AA at any size. Use it as a
         background (with --navy text on top, ~5.4:1) or on a navy panel.
       - --green-dark on white is ~4.0:1 — still under the 4.5:1 body
         floor, so keep it for large/display text only.
       - Small text on paper is --ink (~13.8:1) or --ink-soft (~5.9:1).
```

with:

```
       - --green is GOLD (~1.4:1 on white — never text on a light
         background). Put --navy text on it (~12.1:1), or use it as text on
         a --navy/--navy-deep panel (~12.1 / 14.0:1).
       - --green-dark is the logo's GREEN. Safe for text on white/--paper
         (~7.3 / 6.0:1) and under white text (~7.3:1). Never on the hero
         (~2.3:1 on --navy).
       - Small text on paper is --ink (~13.4:1) or --ink-soft (~5.7:1).
     Millennium Invitational: palette sampled from the event pubmat (panel
     green #032519, gold #FCDC5B, lime #BDE25B) and the club logo (green
     #455C4C, cream #EEE8DC). Fonts: Anton for the hero title, Permanent
     Marker for the headline and slogan.
```

**Check:** each must print nothing:

```bash
grep -n "#14263C\|#0B1826\|#7CB92C\|#5C8F1F\|#F6F7F2\|#ECEEE6\|#D9DED2\|#5B6B74" events/millennium-invitational-2026/index.html
grep -n "rgba(11,24,38" events/millennium-invitational-2026/index.html
```

`schedule.html`'s `CAT_META` still holds `#14263C` until §10.1 replaces it; §10
checks that file.

---

## 9. `index.html` — fonts and the engine's fixed colours

The pubmat's faces and their nearest Google Fonts:

| Pubmat | Google Font | Role on the site |
|---|---|---|
| "2ND INVITATIONAL", "TOURNAMENT" — heavy condensed caps | **Anton** | Hero title only |
| "PAYING IT FORWARD!", "GOOD PEOPLE GREAT GAMES BRIGHTER TOMORROWS" — marker brush | **Permanent Marker** | Hero headline and slogan only |
| Date, venue and partner text — bold condensed caps | Barlow Condensed (the template's) | unchanged |

Everything else keeps the template's fonts: Archivo Black for scores and
numbers, Barlow Condensed for labels and tabs, Inter for body text.

### 9.1 Google Fonts link

Find:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

Replace with:

```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&family=Permanent+Marker&display=swap" rel="stylesheet">
```

### 9.2 The hero's type

The hero's rules live in the engine (`lib/v1/css/hub.css`), which every event
shares, so this event overrides them in its own `<style>`, which loads after the
engine's stylesheets. Find the last rule of the template's own touches:

```css
@media (max-width:480px){ .view-tabs{gap:6px;} .view-tab{font-size:11px; padding:9px 12px;} }
```

Directly **below** it, still inside `<style>`, add:

```css
/* ==== Millennium Invitational: the pubmat's hero type ==== */
h1.title{ font-family:'Anton',sans-serif; font-weight:400; letter-spacing:.02em; line-height:1.02; }
/* The pubmat's lime "Paying It Forward!" banner, as marker brush text. */
.domination-line{ font-family:'Permanent Marker',cursive; text-transform:none; letter-spacing:0; color:var(--lime); }
.slogan-script{
  font-family:'Permanent Marker',cursive;
  font-size:clamp(15px,3.8vw,19px);
  line-height:1.35;
  color:var(--white);
  opacity:.92;
  max-width:520px;
  margin:2px auto 14px;
}
/* On a phone the slogan wraps; balance it so "tomorrows" isn't left alone. */
.slogan-script{ text-wrap:balance; }
```

### 9.3 The engine's fixed colours

`lib/v1/css/` writes some house colours as raw `rgba()`/hex instead of `var()`,
so a re-skin's tokens never reach them. This is the first re-skinned event on
the engine, so it carries the overrides itself (§18). Directly below §9.2's
block, add:

```css
/* ==== Millennium Invitational: lib/v1/css's raw house colours, in this palette ====
   Each rule repeats an engine selector with gold rgba(252,220,91), panel green
   rgba(3,37,25)/rgba(1,20,12), logo green rgba(69,92,76) or cream
   rgba(238,232,220) in place of the house values. The last three move the
   logo green off the hero, where it fails (~2.3:1). */
header.hero{ box-shadow:0 24px 60px -28px rgba(1,20,12,.45); }
.regal-rule::before, .regal-rule::after{ background:linear-gradient(90deg, transparent, rgba(252,220,91,.7)); }
.regal-rule::after{ background:linear-gradient(90deg, rgba(252,220,91,.7), transparent); }
.view-tab:hover{ border-color:rgba(252,220,91,.6); }
.day-picker select:hover, .day-picker select:focus{ border-color:rgba(252,220,91,.6); }
.search-panel::before{ border-color:rgba(252,220,91,.35); }
.ac-item:hover small, .ac-item.active small{ color:rgba(3,37,25,.75); }
td.live-court-cell{ background:rgba(252,220,91,.16); }
.cat-toggle-chip:hover{ border-color:rgba(252,220,91,.6); }
.standings-col-head{ background:rgba(238,232,220,.94); }
.badge-2{ background:var(--green-dark); }
table.gt td{ border-bottom-color:rgba(3,37,25,.07); }
.tm-side.win{ background:rgba(252,220,91,.22); }
.tm-row{ border-bottom-color:rgba(3,37,25,.07); }
.tm-row.live{ background:rgba(252,220,91,.22); }
.tm-row-meta .tm-plabel{ background:rgba(3,37,25,.08); }
.tag.pool{ background:rgba(252,220,91,.18); }
.live-dot{ box-shadow:0 0 0 0 rgba(69,92,76,.5); }
@keyframes live-pulse{
  0%{ box-shadow:0 0 0 0 rgba(69,92,76,.45); }
  70%{ box-shadow:0 0 0 8px rgba(69,92,76,0); }
  100%{ box-shadow:0 0 0 0 rgba(69,92,76,0); }
}
.regal-rule .diamond{ background:var(--green); }
.sync-status.ok{ color:var(--green); }
.hero :focus-visible{ outline-color:var(--green); }
```

The engine's `.club-summary`, `.club-summary-item.leader` and `.club-subhead`
also carry house colours. They are dual-meet views a team Hub never draws, so
they are left out.

**Check:** both must print a match:

```bash
grep -n "Permanent+Marker" events/millennium-invitational-2026/index.html
grep -n "@keyframes live-pulse" events/millennium-invitational-2026/index.html
```

---

## 10. `schedule.html` — colour key and fixed colours

### 10.1 Colour key

A team workbook's `SCHEDULE` tab is uncoloured, so these colours are ours. One
bracket, one hue for every round-robin cell (the logo green), and the pubmat's
gold for the Bronze and Final, so the two playoff slots stand out at the end of
the night.

In `events/millennium-invitational-2026/schedule.html`, replace the whole
`const CAT_META = { … };` object, and the `// EXAMPLE — replace with this event's
own brackets.` line above it, with:

```js
const CAT_META = {
  G1: { short:'ROUND ROBIN', color:'#455C4C' },
  PO: { short:'PLAYOFFS',    color:'#FCDC5B' }
};
```

Navy text on the cells' 40% tints reads at 8.59:1 (round robin) and better on
the gold. `readableOn()` picks the chip text: white on the green, dark on the
gold.

If `ROUND ROBIN` wraps or is cut off in the bezel's legend at 1280px (§13),
change it to `RR`. Change nothing else.

### 10.2 Fixed colours

Find the narrow-screen comment's closing line inside `<style>`:

```css
     is the identity of the board, and it is what the print path shares. */
```

Directly **below** it, add:

```css
  /* Millennium Invitational: lib/v1/css/schedule-board.css's raw house colours, in this palette. */
  .bezel{ box-shadow:0 10px 30px -14px rgba(1,20,12,.55); }
  .cell.live{ box-shadow:0 0 0 2px var(--green), 0 6px 18px -6px rgba(252,220,91,.6); }
```

Leave the bezel's fonts alone: it is a wall display, and the template's upright
type reads best across a room.

**Check:** both must print nothing:

```bash
grep -n "G2\|G3\|#1155CC\|EXAMPLE" events/millennium-invitational-2026/schedule.html
grep -n "#14263C\|#0B1826\|#7CB92C\|#5C8F1F\|#F6F7F2\|#ECEEE6\|#D9DED2\|#5B6B74" events/millennium-invitational-2026/schedule.html
```

---

## 11. Attendance desk and scorer pages

`attendance.html` was copied and filled in §5–§6. It keeps the house theme, like
every desk page. Nothing else to edit. It is linked from nowhere public; Control
Center's **Issue desk link** produces the link.

`scorer.html` was copied and filled the same way. The event is
`"scoreEntry": "links"` (§14): court scorers enter scores on their phones from a
link Mission Control's **Issue scorer link** produces, and the page is linked from
nowhere public. It keeps the house theme. Nothing else to edit.

---

## 12. Dry-run checklist

```bash
cp _templates/dry-run-checklist-template.md events/millennium-invitational-2026/dry-run-checklist.md
```

Replace `{{EVENT_TITLE}}` with `Millennium Pickleball Club` in the copy.

**Check:** `grep -rn '{{' events/millennium-invitational-2026/` still prints nothing.

---

## 13. Verify in a browser

The real workbook won't publish until the operator has fixed and wired it up
(§16), so check the pages on a **temporary fixture**: a made-up finished day of
this event, shaped like the real one. On `localhost` only, `?fixture=finished`
makes every engine page read `/_fixtures/config.json` as the registry and
`/_fixtures/<event>/finished.json` as the day's data.

### 13.1 Make the fixture

Save Appendix A's script **outside the repo** (for example in your scratchpad) as
`make-millennium-fixture.mjs`, then from the site repo root:

```bash
node <path-to>/make-millennium-fixture.mjs
```

It must print:

```
120 matches; ranking Green 19W 243pts, Orange 17W 227pts, Blue 14W 223pts, Red 14W 209pts, Sky 14W 208pts, Yellow 13W 212pts, Pink 11W 180pts, Purple 10W 199pts
Final: Fi-Green v Fi-Orange; Bronze: Br-Blue v Br-Red
```

### 13.2 Serve and open

Serve the **repo root**, not the event folder: the pages load root-absolute
`/assets/…` and `/lib/v1/…` paths. From the site repo root:

```bash
npx --yes http-server -p 8000 -c-1
```

(If your environment runs servers another way, serve the same folder on any port;
the host must be `localhost` for fixture mode.) Open:

- the Hub: `http://localhost:8000/events/millennium-invitational-2026/?fixture=finished`
- the board: `http://localhost:8000/events/millennium-invitational-2026/schedule.html?fixture=finished`
- Control Center: `http://localhost:8000/tools/control-center.html?fixture=finished`,
  then pick **Millennium Pickleball Club**
- the scorer page: `http://localhost:8000/events/millennium-invitational-2026/scorer.html?fixture=finished`

### 13.3 Checks

The Hub, at **375px** and **desktop** width:

- [ ] The hero reads, top to bottom: the Millennium badge (a whole circle, no
      cream corners) × the S.A.G.E. logo, "Millennium Pickleball Club" (one line,
      even at 375px), "24 OCTOBER 2026 · PLUS 63 PICKLEBALL COURTS, MAGALANG"
      in gold, "2ND INVITATIONAL" in white over "TOURNAMENT" in gold (Anton),
      "Paying It Forward!" in lime marker, the slogan in white marker, then the
      subtitle.
- [ ] At 375px the eyebrow wraps to at most two lines and "2ND INVITATIONAL"
      fits on one.
- [ ] The rule under the tabs is gold with a gold diamond, and the "Live ·
      last synced" line under it is gold.
- [ ] Desktop only: the QR panel shows the short link, and the image if
      `assets/qr.png` was supplied (a broken image otherwise, §2).
- [ ] The footer shows three lines, with the charity line in the middle.
- [ ] The hero is dark green, not navy; the paper is cream; nothing is the
      house's bright green.
- [ ] The day picker is hidden (one day) and the day loads.
- [ ] **Standings:** one table, "Bracket 1", with columns **W**, **L** and **Pts**,
      ranked Green, Orange, Blue, Red, Sky, Yellow, Pink, Purple (Blue, Red and Sky
      all have 14 wins and are separated by points), with an **Advances** pill on
      Green, Orange, Blue and Red. Then **Playoffs** with **Bronze** and **Final**
      sections, and no Quarterfinals or Semifinals.
- [ ] The **Final** card shows Green beating Orange **3 – 1** with "points 35–38"
      under it: the winner on games, with fewer points. The winning side is tinted
      gold, not green.
- [ ] A matchup's rows are labelled "MD 1", "MD 2", "WD", "XD".
- [ ] The **Teams** tab shows eight roster cards under "Bracket 1".
- [ ] DevTools: `Anton` and `Permanent Marker` load; the console shows no errors;
      no 404 except `assets/qr.png` when it hasn't been supplied.

The board:

- [ ] The bezel reads "Millennium Pickleball Club" on the dark green, with 8 court
      columns and slots from 1:00 PM to 8:30 PM.
- [ ] Round-robin cells are green-tinted, the 8:00 PM and 8:30 PM cells (Bronze,
      Final) gold-tinted, and the legend shows `ROUND ROBIN` and `PLAYOFFS` on one
      line each at 1280px (§10.1 has the fallback).

Control Center:

- [ ] Millennium Pickleball Club opens with the team layout, the same Bracket 1
      order and the Final as **3 – 1** to Green.
- [ ] Its Awards podium (if shown without signing in) is Green, Orange, Red.
- [ ] Switch to **PickleDrive Club One Year Celebration**: its Bracket tables
      still show PF, PA and Quotient, and its matchup cards lead with total
      points. (The default is untouched.)

The scorer page:

- [ ] Opened without a link, it reads "Ask the operator for a scorer link." and
      the console shows no errors. (A real link needs the live API; the dry run
      checks one, §16.5.)

Stop the server.

### 13.4 Remove the fixture

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/millennium-invitational-2026
git status --short
```

`git status` must show nothing under `_fixtures/`. The fixture is never committed:
the comparison harness reads `_fixtures/config.json`, and an extra event there
would change Control Center's event list in its screenshots.

---

## 14. Register the event in `event-data`

In `D:\Personal\SAGE\event-data\config\events.json`, add this entry inside
`"events"`, **after** `"piggleball-2026"` (currently the last entry), and add a
comma after the closing brace of that entry:

```json
"millennium-invitational-2026": {
  "type": "team",
  "title": "Millennium Pickleball Club",
  "teamScoring": "games",
  "attendance": "desks",
  "scoreEntry": "links",
  "days": {
    "millennium-invitational-2026-day1": {
      "label": "Oct 24",
      "date": "2026-10-24",
      "isLive": "auto",
      "facilities": [
        { "name": "Plus 63", "sheetId": "1ulbHdH5iU5lNm6fqpvbbvxfdht-cMvjGrtoZaN8ru6k" }
      ]
    }
  },
  "display": {
    "pairs": {
      "1": { "full": "Men's Doubles",   "short": "MD" },
      "2": { "full": "Men's Doubles",   "short": "MD" },
      "3": { "full": "Women's Doubles", "short": "WD" },
      "4": { "full": "Mixed Doubles",   "short": "XD" }
    }
  }
}
```

- `teamScoring: "games"` is §1.1's rule, read by §4.
- The pairs are the workbook's own: `MatchUps` gives pairs 1 and 2 two men,
  pair 3 two women and pair 4 a man and a woman, and its `Pairings Guide` reads
  MD, MD, WD, XD. `numberRepeatedPairs` turns the two MDs into "MD 1" and
  "MD 2". Without this block the labels would be PickleDrive's defaults (MD, WD,
  XD 1, XD 2), which are wrong for this event.
- The facility name `Plus 63` is the workbook's own (`Variables` location,
  `Title`). Sync setup compares it exactly, so §16.2 must type the same.
- `attendance: "desks"` and `scoreEntry: "links"` are the owner's choice: desk
  links for check-in, and scorer links for the courts (§11).

**Before saving, confirm no other event already uses this day key:**

```bash
grep -n '"millennium-invitational-2026-day1"' config/events.json
```

This must print nothing. Day keys are validated as **globally unique across
every event** in this file, because the day key alone is the sync route. A
duplicate makes the whole file fail validation, and Cloud Run then keeps
serving the last good config, so this event never registers. If the key is
taken, **stop and ask**.

**Check:** run from `event-data/`:

```bash
node -e "JSON.parse(require('fs').readFileSync('config/events.json','utf8'))"
```

It must run without error. Then commit it in `event-data` (on `main`, which is
how that repo is worked) and ask the owner before `git push`: pushing registers
the event with the live API and adds it to Control Center within about a minute.

---

## 15. Done — report back

Report:
- §4: the commits, the new tests, `npm run verify`'s result, and whether both
  pushes were made (only with the owner's go-ahead);
- which of §5–§13 passed, and that the fixture was removed (§13.4);
- whether `qr.png` and the short link were present;
- whether §14 passed;
- the uncommitted files, in every repo touched.

---

## 16. Operator tasks — people, not the implementer

These happen in Google Sheets and on the day. They are listed so the
constraints they put on the data are written down. The workbook was checked twice
on 18 October 2026 from an export; §16.1 is the state at the second check.

### 16.1 Workbook fixes before sync setup

The schedule itself is right: `CSV` rows 2–121 hold matches 1–120, every team
meets every other team exactly once, no team plays twice in one slot, pairs
always meet their own number (`Red_3` against `Orange_3`), times run 1:00 PM to
8:30 PM, and `CourtAssignment` is `Court 1`–`Court 8`. What is still
PickleDrive's, or test data:

1. **Done.** `STANDINGSCSV` `A2:A9` hold the eight colours, all in bracket `1`
   (`F2:F9`), and PickleDrive's old rows are gone, the unused quarterfinal and
   semifinal rows included. Rows 10–13 are `Br-1`, `Br-2`, `Fi-1` and `Fi-2`,
   formulas reading `MatchUps` (`E732`, `E742`, `E752`, `E762`). Under
   `teamScoring: "games"` the site ranks from the matches, so `totalPoints` and
   `quotient` don't need to change.
2. **`Variables` lists PickleDrive's teams too.** Its "Team Event Details"
   columns `R:U` (Bracket, Team Code, Team Name, Team Captain) still hold `A`–`O`
   in rows 2–16, and `STANDINGSCSV`'s `teamName` (`B2:B9`) is a formula that looks
   each team's code up in column `S` and returns column `T`:
   `FILTER(Variables!T$2:T1546, Variables!S$2:S1546=A2)`. The colours are already
   listed in `S35:S42`, inside that range, but `T35:T42` beside them are empty,
   so every `teamName` is blank. **Fix:** type each team's name into `T35:T42`,
   beside its colour, and clear `R2:U16` (PickleDrive's `A`–`O`). Put each colour
   in column `S` **once**: a second match makes `FILTER` return two names and the
   cell errors. **Every team needs a name**: its own name if it has one (§17 Q1),
   otherwise the colour itself (`Red`). Without a name, a team typed into the
   Bronze or the Final reads `Seed Red · TBD` on every page. The names can wait
   (the owner's call, 18 October), but they must be in before the dry run's
   playoff check (§16.5), and in any case before the seeds are typed on the day
   (§16.4). Until then the Hub, the board and the round robin all work, with each
   team shown by its colour.
3. **The `Teams` tab's name column isn't recognised.** The sync publishes the
   roster (the Hub's Teams tab, and attendance's **Update roster**) from the
   column headed `Player` or `FINAL LEVEL ORDER`, and this tab's header row
   (row 3) has `NAMES` and `NAMES 2` instead. Rename the header of the column
   that holds each player's name to `Player`. Without it the Hub's Teams tab
   stays hidden and **Update roster** fails. Then type the 64 players in.
4. **Done.** `CSV`'s 31 leftover rows (match numbers 121–151 with `#N/A` beside
   them) are deleted, and it lists exactly the 120 matches.
5. **Every match has a test score.** `SCHEDULE` holds 11–1 or 1–11 for all 120
   matches, Bronze and Final included. Clear every score so the cell is truly
   empty (**Delete**, not a space).
6. **Done.** `ATTENDANCE` is cleared below its header (**Update roster** refills
   it from `Teams`), and the `Raffle` tab with PickleDrive's players is deleted.
7. **`Court Control` shows `#REF!`** on every court. Test it as at Piggleball:
   put a match number on Court 1 the way operators will on the day, confirm the
   `#REF!` clears and `CSV`'s `court` column shows the match, then undo it. If
   it doesn't clear, compare its formulas with PickleDrive's workbook.
8. **`FINAL RANK` and `Awards` are still PickleDrive's.** `FINAL RANK` ranks for
   eight quarterfinal seeds across three brackets with wildcards, and `Awards`
   shows `#REF!`/`#N/A` beside each placing. Neither is published. Before the
   evening, make `FINAL RANK` rank the eight teams by §1.1's rules (the
   `Standings` tab already counts each team's wins (`Pair Wins`) and points (`Team Score`)), or ignore it and
   seed from the Hub (§16.4); repair `Awards` before the ceremony
   ([Awards tab](../implemented/awards-podium-tab-spec.md)). The workbook's
   `Variables` lists `tie_breaker_tier1` as "points quotient"; the event's
   first tiebreaker is total points (§1.1).
9. **Name the workbook** `2026-10-24 Millennium Pickleball Club - Plus 63`, the
   runbook's `<date> <title> - <FACILITY>`. Optionally retitle the `Title` tab
   (`Millennium Pickleball Club Team Event`), which heads `SCHEDULE` and
   `Awards`; no page reads it.

**Check, after 1–6:** `STANDINGSCSV` lists the eight colours in bracket `1`
(rows 2–9), each with a `teamName` once item 2 is done, then `Br-1`, `Br-2`,
`Fi-1`, `Fi-2`; `CSV` lists 120 matches with every score empty; every Bronze and
Final row reads its seed (`Br-1_1` …, `Fi-2_4`).

At the last check (18 October), the workbook's data ran through the engine with
§4 applied: 120 matches, 28 round-robin matchups in one bracket, a Bronze and a
Final, 8 courts, slots from 1:00 PM. Items 2, 3, 5 and 7–9 were still open.

### 16.2 Share it, then set up live sync — once, last

Do this **after** §16.1. Until setup runs, nothing publishes, so edit freely
before it.

**Before you start:** the `events.json` entry (§14) must be committed to
`event-data`. Setup checks the day key and venue against the server, so it
rejects them if the event isn't registered yet. Allow about a minute after the
commit.

1. **Share** the workbook with
   `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**,
   and set **General access** to **Restricted**. It was shared by link when this
   spec was written. The sync, the people list and attendance read it as that
   account, and `ATTENDANCE` will hold names.
2. **Extensions → Apps Script.** Confirm `sheets-sync.gs` is there; the copy
   carries it, but not its trigger.
3. Reload the spreadsheet, then **SAGE → Set up live sync** (or **Live sync
   settings**). Enter:

   | Field | Value |
   |---|---|
   | Day | `millennium-invitational-2026-day1` |
   | Venue | `Plus 63` — exact, including the space and capital P |
   | Tabs to watch | `SCHEDULE` and `Court Control` (pre-ticked) |
   | Secret | Only if asked. It is `SYNC_SHARED_SECRET` from Cloud Run's env vars. Never paste it into chat, a doc or this spec |

4. **Save.** Setup checks the secret, day and venue, then sends one real test
   update. Setup replaces the day key the copy carried over from PickleDrive.

**Confirm it's live:** open
`https://sage-match-control.github.io/event-data/millennium-invitational-2026/data/millennium-invitational-2026-day1.json`
about a minute after setup. It should list one facility, `Plus 63`, with a
non-empty `rosterCsv` once `Teams` has names (§16.1 item 3).

### 16.3 Go-live

The Hub shows scores and the Live and Standings tabs from **4 hours before the
earliest scheduled match**: 9:00 AM on the day, for a 1:00 PM start. To change
that, use Control Center's go-live override.

### 16.4 On the day

**Lineups, round by round.** Each captain submits the round's four pairs to match
control before the round (§1.1). The operator types them into `MatchUps` for that
round's matchups. Until then the Hub shows the round's matches as **Lineup not
set**. No page checks the organiser's "no repeat doubles" rule; that stays with
the organiser and match control.

**Scorer links.** On Mission Control, **Issue scorer link** gives a 24-hour link for
the day; send it to each court scorer. A score entered on it is written into that
match's two `SCHEDULE` score cells and published at once, which is why the
workbook is shared with the service account as **Editor** (§16.2). The event's
**Accepting / Stopped** switch on Mission Control stops and restarts every link.

**The Bronze and the Final.** After the last round-robin slot (7:30 PM), rank the
eight teams by §1.1's rules. The Hub's and Control Center's Bracket 1 table shows
that order (§4), and the workbook's `Standings` tab has the same counts. Then type
the qualifiers' colours over the seed numbers in `MatchUps`:

| Cell | Seed | Team |
|---|---|---|
| `D744` | `Fi-1` | 1st |
| `D754` | `Fi-2` | 2nd |
| `D724` | `Br-1` | 3rd |
| `D734` | `Br-2` | 4th |

The organiser's schedule lists the Bronze as "4th vs 3rd". The site numbers it
the other way round so that "Seed 1" is always the better-placed team (§18);
which side is listed first changes nothing else. The cells are typed values, not
formulas. A typed colour turns `Fi-1` into `Fi-Red`, which every page reads as
team Red.

### 16.5 Dry run

Run the dry run against the real workbook by **Thursday 22 October**, using
`dry-run-checklist.md`. Also check:

- Control Center renders the event with the **team** layout, one bracket, the
  pair labels "MD 1", "MD 2", "WD", "XD", and no unmapped-team warning.
- With temporary scores in one matchup where one side wins three games on fewer
  points (for example 11–9, 11–9, 11–9, 2–11): the Hub and Control Center show
  that side as the winner, `3 – 1`, and Bracket 1 ranks it above its opponent.
  Clear the scores afterwards.
- The Live board shows 8 courts and no venue heading row (one facility).
- Typing a colour into `D744` (then undoing it) makes the Final's side read that
  team's name on the Hub, Control Center and the board.
- A desk link opens the desk page with the 64 players after **Update roster**.
- A scorer link, issued from Mission Control, opens the scorer page on a phone;
  a score entered there appears on the Hub within seconds. Clear it afterwards.

---

## 17. Questions, answered by the owner on 18 October 2026

| # | Question | Answer |
|---|---|---|
| Q1 | Do the teams have names, or only their colours? | Names come later. Until then each team shows by its colour; names (or the colour itself) go in `Variables` `T35:T42` before the dry run (§16.1 item 2) |
| Q2 | The last tiebreaker, "then points won": which points? | The points won in the head-to-head games between the tied teams (§1.1) |
| Q3 | A Bronze or Final matchup that ends 2–2 on games? | Total points across its four games decide it; level on those too, it shows as a **Tie** (§4) |
| Q4 | The short link | `tinyurl.com/SAGExMPC2026`; the owner creates it and sends `qr.png` |
| Q5 | Check-in and scoring on the day? | Check-in desk links and scorer links: `"attendance": "desks"`, `"scoreEntry": "links"` (§11, §14) |
| Q6 | The site title? | **Millennium Pickleball Club** (§6) |
| Q7 | May the implementer push `main` itself? | No: stop and ask the owner at each of the two pushes (§4.8) |

A sharper pubmat may still arrive (§2). It changes nothing unless its colours
visibly differ from §8's.

---

## 18. Decisions, for reference

- **Scoring is an engine option, not a fork.** The rule lives in
  `lib/v1/domain/teams.js`, which the Hub, the board, Control Center and its Awards
  podium all share, so one option corrects all of them. It is opt-in per event
  (`teamScoring`), defaulting to PickleDrive's rule, so no other page changes, and
  it is a compatible `v1` change (§4.8), not a `v2`.
- **Tiebreakers: total points, head-to-head wins, head-to-head points.** The
  owner's revision of 18 October 2026 replaces the organiser's first sheet (point
  differential, then head-to-head wins). Its closing "then points won" is read as
  head-to-head points, because overall points won already come second (§17 Q2,
  confirmed).
- **Ranking is computed from the matches.** `STANDINGSCSV` publishes points, not
  games, and changing its columns would touch every team page's reader. The
  matches already carry every score the rule needs.
- **A 2–2 playoff matchup goes to total points**, the organiser's first
  tiebreaker (§17 Q3, confirmed).
- **Event key `millennium-invitational-2026`, day key
  `millennium-invitational-2026-day1`.** The club's name plus the event's own,
  following `pickledrive-anniversary-2026`. The day key carries the event key
  because day keys must be globally unique (§14).
- **Title "Millennium Pickleball Club", the event in the hero.** The owner's
  choice. The hero says "2nd Invitational Tournament / Paying It Forward!", and no
  ordinal goes in the `<title>` or the Control Center masthead, where it would
  date next year's page, as decided for Piggleball.
- **Every team has a name, the colour by default.** The engine shows a playoff
  qualifier by its team's name and reads a nameless one as a seed still to be
  decided, so a name is required; the colour is the name when the organiser has no
  other. This needs no engine change.
- **The colours are the team codes.** The workbook already uses `Red_1` …, and the
  engine takes any text before `_` as the side, so nothing in the sheet is
  re-coded to letters.
- **Gold in the fill role, the logo's green in the text role.** The template's
  accent fill sits under navy text and on navy panels; gold passes there
  (12.11:1). The logo green passes as text on cream and under white text, which
  is the other role. The pubmat's lime is used once, for the headline, as on the
  pubmat.
- **Cream paper, from the logo.** The pubmat's own ground is the court artwork,
  not a flat colour. The logo's cream keeps the page warm against the dark
  green hero.
- **The hero stays dark.** The pubmat's panels are dark green, and the template's
  hero controls are styled for a dark panel.
- **Engine colour overrides live in the shell.** `lib/v1/css/` hard-codes some
  house colours. Fixing that in the engine (tokens for every tint) is another
  engine change days before an event, for looks only. A shell override is local,
  freezes with the event, and repeats only selectors the versioning rule already
  promises not to rename. The engine fix is worth doing after the event, so the
  next re-skin doesn't repeat §9.3.
- **"Seed 1" is the better-placed team.** Before the qualifiers are typed, the Hub
  shows `Seed 1 · TBD`. Making `Br-1` the 3rd-placed team keeps that label true;
  the organiser's schedule listing the Bronze as "4th vs 3rd" is order only.
- **Logo cropped, pubmat not loaded.** The badge filled 41% of the supplied
  image, so it is cropped to the circle. The pubmat stays in `assets/` as the
  colour and font reference, as at Piggleball.
- **No sponsor phone numbers or partner logo.** The phone numbers are for
  sponsorship enquiries, not for players. The hero pairs only the organiser's
  logo with S.A.G.E.'s, as on every event.

---

## Appendix A — the fixture script

Used by §13 only, and never committed. Save it outside the repo and run it from
the site repo root. It writes `_fixtures/millennium-invitational-2026/finished.json`
and adds the event to `_fixtures/config.json`; §13.4 removes both.

```js
// Writes a temporary, local-only fixture for the Millennium Invitational, shaped like the real event: 8 colour
// teams in one bracket, a full round robin of 4-game matchups on 8 courts from 1:00 PM, then a Bronze and a Final
// seeded by the "games" ranking. Each team is named after its colour, as the spec's §16.1 asks of a team with no
// other name. Every match is played, with fixed made-up scores and player names.
//   _fixtures/millennium-invitational-2026/finished.json   the snapshot
//   _fixtures/config.json                                  gains a "millennium-invitational-2026" entry
// Neither is ever committed. Undo with:
//   git checkout -- _fixtures/config.json && rm -rf _fixtures/millennium-invitational-2026
// usage, from the site repo root: node <path>/make-millennium-fixture.mjs
import fs from 'node:fs';
import path from 'node:path';

const EVENT = 'millennium-invitational-2026', DAY = 'millennium-invitational-2026-day1';
const TEAMS = ['Red', 'Orange', 'Yellow', 'Green', 'Blue', 'Purple', 'Sky', 'Pink'];

// Fixed pseudo-random scores: the same file every run.
let seed = 20261024;
const rand = n => { seed = (seed * 1103515245 + 12345) % 2147483648; return Math.floor(seed / 65536) % n; };
const game = () => { const loser = rand(10); return rand(2) ? [11, loser] : [loser, 11]; };
const player = (team, n) => `${team} Player ${n}`;
const time = slot => { const min = 13 * 60 + 30 * slot; const h = Math.floor(min / 60); return `${h > 12 ? h - 12 : h}:${String(min % 60).padStart(2, '0')} PM`; };

// Round robin by the circle method: 7 rounds of 4 matchups. Each round is two slots: pairs 1-2, then pairs 3-4.
const rounds = [];
const circle = TEAMS.slice();
for (let r = 0; r < 7; r++) {
  rounds.push([0, 1, 2, 3].map(i => [circle[i], circle[7 - i]]));
  circle.splice(1, 0, circle.pop());
}
const rows = [];
let num = 0;
const addMatchup = (side1, side2, slot, pairs, courtOf, scores) => pairs.forEach((p, k) => {
  const [s1, s2] = scores ? scores[k] : game();
  const base = s => s.replace(/^(Br|Fi)-/, '');
  rows.push([++num, `${side1} v ${side2}`, '', time(slot), `Court ${courtOf(k)}`,
    `${side1}_${p}`, player(base(side1), 2 * p - 1), player(base(side1), 2 * p), s1,
    `${side2}_${p}`, player(base(side2), 2 * p - 1), player(base(side2), 2 * p), s2]);
});
rounds.forEach((round, r) => {
  [[1, 2], [3, 4]].forEach((pairs, half) =>
    round.forEach(([a, b], i) => addMatchup(a, b, 2 * r + half, pairs, k => 2 * i + k + 1)));
});

// Rank by the event's rules (games won, total points; the fixed scores leave no deeper ties to break).
const rec = Object.fromEntries(TEAMS.map(t => [t, { won: 0, pf: 0, pa: 0 }]));
for (const row of rows) {
  const [a, b] = [row[5].split('_')[0], row[9].split('_')[0]];
  rec[a].pf += row[8]; rec[a].pa += row[12]; rec[b].pf += row[12]; rec[b].pa += row[8];
  if (row[8] > row[12]) rec[a].won++; else rec[b].won++;
}
const ranked = TEAMS.slice().sort((x, y) => rec[y].won - rec[x].won || rec[y].pf - rec[x].pf);
// The Bronze, then the Final, with the qualifiers typed over the seeds (§16.4). The Final's side 1 wins three
// games on fewer points, so the games rule visibly decides it.
addMatchup(`Br-${ranked[2]}`, `Br-${ranked[3]}`, 14, [1, 2, 3, 4], k => k + 1);
addMatchup(`Fi-${ranked[0]}`, `Fi-${ranked[1]}`, 15, [1, 2, 3, 4], k => k + 1, [[11, 9], [11, 9], [11, 9], [2, 11]]);

const csv = table => table.map(r => r.join(',')).join('\r\n');
const matchesCsv = csv([['matchNumber', 'matchUp', 'court', 'Schedule', 'CourtAssignment', 'teamCode1', 'team1Player1', 'team1Player2', 'team1Score', 'teamCode2', 'team2Player1', 'team2Player2', 'team2Score'], ...rows]);
const standingsCsv = csv([['teamCode', 'teamName', 'totalPoints', 'totalOpponentPoints', 'quotient', 'bracket'],
  ...TEAMS.map(t => [t, t, rec[t].pf, rec[t].pa, (rec[t].pf / rec[t].pa).toFixed(5), 1]),
  ['Br-1', '', 0, 0, '-', ''], ['Br-2', '', 0, 0, '-', ''], ['Fi-1', '', 0, 0, '-', ''], ['Fi-2', '', 0, 0, '-', '']]);
const rosterCsv = csv([['teamCode', 'player', 'level', 'gender'],
  ...TEAMS.flatMap(t => [1, 2, 3, 4, 5, 6, 7, 8].map(n => [t, player(t, n), '', n <= 4 || n === 7 ? 'M' : 'F']))]);

const snapshot = {
  day: DAY, label: 'Oct 24', isLive: true,
  generatedAt: '2026-10-24T13:00:00.000Z', publishedAt: '2026-10-24T13:00:00.000Z',
  facilities: [{ name: 'Plus 63', matchesCsv, standingsCsv, rosterCsv, syncedAt: '2026-10-24T13:00:00.000Z', lastEditAt: '2026-10-24T13:00:00.000Z' }],
  failedFacilities: [], staleFacilities: [],
};
fs.mkdirSync(path.join('_fixtures', EVENT), { recursive: true });
fs.writeFileSync(path.join('_fixtures', EVENT, 'finished.json'), JSON.stringify(snapshot, null, 2) + '\n');

const configPath = path.join('_fixtures', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.events[EVENT] = {
  type: 'team', title: 'Millennium Pickleball Club', teamScoring: 'games', attendance: 'desks', scoreEntry: 'links',
  days: { [DAY]: { label: 'Oct 24', date: '2026-10-24', isLive: 'auto', facilities: [{ name: 'Plus 63', sheetId: 'FIXTUREMILLENNIUMSHEETID000000000' }] } },
  display: { pairs: {
    1: { full: "Men's Doubles", short: 'MD' }, 2: { full: "Men's Doubles", short: 'MD' },
    3: { full: "Women's Doubles", short: 'WD' }, 4: { full: 'Mixed Doubles', short: 'XD' },
  } },
};
fs.writeFileSync(configPath, JSON.stringify(config, null, 2) + '\n');
console.log(`${rows.length} matches; ranking ${ranked.map(t => `${t} ${rec[t].won}W ${rec[t].pf}pts`).join(', ')}`);
console.log(`Final: Fi-${ranked[0]} v Fi-${ranked[1]}; Bronze: Br-${ranked[2]} v Br-${ranked[3]}`);
```
