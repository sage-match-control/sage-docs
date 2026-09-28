# Spec — Team tournament (PickleDrive Club One Year Celebration)

> **Status: in progress.** Every rule is settled (§14). The public site
> (§5–§6), the schedule board (§7), Control Center's `team` type (§9), the
> registry entry (§10) and the operator runbook (§11) are built and were
> checked against the three test fixtures. Still open: the QR image, which the
> organiser supplies (it needs no code change when it arrives — §5.2); the
> short link, which the organiser owns and did not yet redirect to the event
> page when last checked; and installing the sync script in the event
> workbook and running the dry run. The event is **Saturday, 3 October
> 2026**. The template (§15) follows the event.

Build the website for a **team tournament**. Named teams of eight players
meet in **matchups**. A matchup is four doubles matches between the same two
teams.

> **Terminology — use these words everywhere, on the pages and in code.**
> A **matchup** is one team against another: four **matches**, e.g.
> *DinkTadors vs LOB is in the Air*. A **match** is one game between two
> pairs (Match #1–136). A **tie** means **equal points** — nothing else.
> When a matchup finishes with both teams on the same total, it is a
> **tie**. Never use "tie" for the team-vs-team meeting itself, in page text
> or in new identifiers. That is always a **matchup**, the same word the
> workbook uses (its `matchUp` column and `MatchUps` tab).
No template exists for this format yet. This event is the **prototype**:
build it as one event folder now, and extract a template from it after the
event (§15).

The headline requirement: **team names appear everywhere a team appears**.
That means Standings, Match Finder, Live Matches, the schedule board, and
every Control Center view.

This spec is written for an implementer with **no other context**. It gives
every value and every rule. You should not need to make product decisions.
If this spec disagrees with the code, or with
`sage-match-control.github.io/_templates/CLAUDE.md` (the template runbook),
**stop and ask**. Don't pick one.

---

## 0. How to use this spec

- **Repos.** All under `D:\Personal\SAGE\`:
  - `sage-match-control.github.io\` — the static site. §5–§9 and §11 happen
    here. Paths in those sections are relative to this repo.
  - `event-data\` — the data repo. §10 edits `config\events.json` there.
  - `sage-docs\` — documentation. §12 edits it.
  - `sage-tools-api\` — the backend. **Nothing in it changes.**
- **Order.** Do §5 → §12 in order. Each section ends with a check. Don't
  move on until it passes.
- **Find edits by anchor text, not line numbers.** Quoted anchors are exact.
  Line numbers drift.
- **Read before you edit.** The files are large (`index.html` ~3,000
  lines, `tools/control-center.html` ~5,800). Read each function you
  change in full first.
- **Don't edit anything under `_templates/`.** The template comes later
  (§15).
- **Don't commit or push** unless the user asks.
- **No build step, no framework, no new dependencies.** Each page is one
  self-contained HTML file with an inline `<style>` and `<script>`. The only
  external resources allowed are Google Fonts and what the page already
  loads.
- **Python is not installed on this machine.** Serve pages with Node (§8.1).
- Don't rename, "fix" or tidy anything this spec doesn't mention.

---

## 1. Event facts

| Fact | Value |
| --- | --- |
| Name | PickleDrive Club One Year Celebration |
| Organiser | PickleDrive Club (`@pickledriveclub`) |
| Format | Team tournament |
| Pubmat slogan | *Not just a tournament, it's a celebration!* |
| Date | Saturday, 3 October 2026 — one day. First serve 9:00 AM |
| Venue | Kingcourts — one facility, **10 courts** (Court 1–10) |
| Teams | 15, codes `A`–`O`, each with a name (e.g. `A` = DinkTadors) |
| Groups | 3 groups of 5. Group 1: A–E. Group 2: F–J. Group 3: K–O |
| Roster | 8 players per team (4 men, 4 women) |
| Matchup | 4 matches, one game each. Pair 1 Men's Doubles, Pair 2 Women's Doubles, Pair 3 Mixed Doubles, Pair 4 Mixed Doubles |
| Group stage | Full round robin in each group: 10 matchups per group, 30 matchups, 120 matches (#1–120). 25-minute slots from 9:00 AM. Pairs 1–2 of a matchup play in one slot, pairs 3–4 in the next |
| Semifinals | 2 matchups, 8 matches (#121–128), 2:25 PM and 2:50 PM, Courts 4–7 |
| Final and Bronze | 1 matchup each, played at the same time, 8 matches (#129–136), 3:15 PM and 3:40 PM, Courts 4–7 |
| Total | 136 matches, 34 matchups |

**The site shows none of these pubmat details:** registration fee,
inclusions (shirt, meals, drinks, DJ, raffle, photos), prizes, or social
handles. The site is a live results hub for people already entered.

---

## 2. Inputs — already staged

| Input | Where | Notes |
| --- | --- | --- |
| Event logo | `sage-match-control.github.io/events/pickledrive-anniversary-2026/assets/logo.png` | 500×500 PNG. White script "PickleDrive Club" on a photographed sage-green fabric background. A full square image, **not** a transparent mark. Use as is |
| Pubmat | `…/events/pickledrive-anniversary-2026/assets/pubmat.webp` | 1023×1537. Reference for §6 only. The page does not load it |
| Test snapshots | `sage-match-control.github.io/_fixtures/pickledrive-anniversary-2026/{pre,finished,edge}.json` | Exact copies of what the sync publishes, built from the organiser's workbooks. See §8 |
| Test registry | `sage-match-control.github.io/_fixtures/config.json` | Registers only this event, for testing Control Center locally (§9.9) |
| Event workbook | Spreadsheet `1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA` | The live workbook. Its sheet ID goes in §10 |
| Finished-state workbook | Spreadsheet `1YmJYvpGMYOTZA9EkF7M2WaxldHo6huWuBKU6nWss9oo` | A copy with every score filled in, source of `finished.json` |
| Short link | `tinyurl.com/SAGExPickleDrive2026` | Should redirect to `https://sage-match-control.github.io/events/pickledrive-anniversary-2026/`. Checked in §5.2 |
| QR image | **To follow.** It will be dropped in at `events/pickledrive-anniversary-2026/assets/qr.png` | See §5.2. Don't generate one |

`_fixtures/` starts with an underscore, so GitHub Pages (Jekyll) never
publishes it. It is served only by your local server. Leave it in place, and
don't add a `.nojekyll` file to this repo.

---

## 3. The data

### 3.1 How it arrives

A Google Apps Script in the workbook posts edits to `sage-tools-api`. That
service copies two tabs, `CSV` and `STANDINGSCSV`, as raw CSV text into one
JSON file per day. It publishes the file to:

```
https://sage-match-control.github.io/event-data/pickledrive-anniversary-2026/data/pickledrive-anniversary-2026-day1.json
```

Every page polls that URL every 10 seconds. The file's shape:

```json
{
  "day": "pickledrive-anniversary-2026-day1",
  "label": "Oct 3",
  "isLive": true,
  "generatedAt": "2026-10-03T07:50:00.000Z",
  "facilities": [
    { "name": "Kingcourts", "matchesCsv": "<CSV tab>", "standingsCsv": "<STANDINGSCSV tab>", "syncedAt": "…" }
  ],
  "failedFacilities": [],
  "staleFacilities": []
}
```

`isLive` is `true`, `false` or `"auto"`. The existing
`computeDayIsLive()` already handles it, so don't change that function.

### 3.2 `matchesCsv` — one row per match

```
matchNumber,matchUp,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score
1,A v B,,9:00 AM,Court 1,A_1,Tristen Pamintuan ,Errol Cunanan ,11,B_1,Jr Pineda ,Patrick Garcia ,1
```

| Column | Meaning |
| --- | --- |
| `matchNumber` | 1–136. Rows without one are spacers — skip them (the parser already does) |
| `matchUp` | **The matchup.** All four matches of a matchup share it: `A v B`, `SF-A v SF-F`. **Group matches into matchups by this column only.** Never infer matchups from codes or time slots |
| `court` | The court a match is being played on **right now**, as a bare number (`6`), or blank. Drives Live Matches |
| `Schedule` | Slot time, `9:00 AM` |
| `CourtAssignment` | Scheduled court, `Court 1` |
| `teamCode1` / `teamCode2` | `<SIDE>_<PAIR>` (§3.4). `teamCode1` is always the first team named in `matchUp` |
| `team1Player1` … `team2Player2` | Player names. **They carry trailing spaces.** The existing parser trims them |
| `team1Score` / `team2Score` | Blank until played. A match is **played** when both are present |

**Lineups are set per matchup.** Before a captain enters a matchup's lineup, each
player cell holds the pair's own code (`A_1`). The existing `rowsToMatches`
already turns a name equal to its code into `'TBD'`. In this spec,
**"lineup not set"** means `t1p1 === 'TBD'` (or `t2p1`). The same player can
play a different pair in each matchup.

### 3.3 `standingsCsv` — one row per team

```
teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket
A,DinkTadors,176,16,11.00000,1
…
O,Tito Dink & Joey,16,176,0.09091,3
SF-A,DinkTadors,44,4,11.00000,
SF-F,Pampanga Wildcards,4,44,0.09091,
…
Fi-K,KaDINKo,4,44,0.09091,
```

| Column | Meaning |
| --- | --- |
| `teamCode` | `A`–`O` for the 15 teams. Then 8 playoff rows: `SF-*` ×4, `Br-*` ×2, `Fi-*` ×2 |
| `teamName` | **The only source of team names.** Blank on a playoff row whose slot isn't decided yet |
| `totalPoints` / `totalOpponentPoints` | Points scored / conceded. Group rows cover that team's group matches; a playoff row covers that one matchup |
| `quotient` | `totalPoints / totalOpponentPoints`, as text. **`-` before any match is played** — parse it as `null` |
| `bracket` | Group number `1`–`3` on rows `A`–`O`. Blank on playoff rows |

This shape differs from both existing templates. It has no `player1`,
`player2`, `wins` or `loss`. The workbook has other tabs, including one
named `Standings`. **The site never reads any of them.** It uses only these
two CSVs.

### 3.4 Team codes

| Code | Side | Stage | Slot | Pair |
| --- | --- | --- | --- | --- |
| `A_3` | `A` | group | — | 3 |
| `SF-1_2` | `SF-1` | Semifinal | `1` (undecided) | 2 |
| `SF-A_2` | `SF-A` | Semifinal | `A` (decided: team A) | 2 |
| `Br-F_4` | `Br-F` | Bronze | `F` | 4 |
| `Fi-A_1` | `Fi-A` | Final | `A` | 1 |

- **Side** = everything before the first `_`. **Pair** = the number after it.
- **Stage** = the part of the side before its first `-`: `SF`, `Br` or
  `Fi`. No `-` means a group match.
- **Slot** = the part after the `-`. Until the organiser fills it in, it is
  the seed number: `SF-1`…`SF-4`, `Br-1`/`Br-2`, `Fi-1`/`Fi-2`. The
  organiser **types the qualifying team's letter** into the workbook's
  `MatchUps` tab (`D604`, `D614`, `D624`, `D634` for the four semifinal
  seeds, then `D644`–`D674` for Bronze and Final). The slot then becomes
  that letter (`SF-A`). No formula picks the teams — **the workbook's entry
  is the decision**, and the site only displays it. **A code changes
  mid-event.** Never key saved state on a playoff code.
- **Base team** — the group-stage team a side belongs to. For a group side,
  the side itself. For a playoff side, the slot, when the slot is a team
  code that exists in `STANDINGSCSV` (`SF-A` → `A`). A numeric slot has no
  base team yet.

---

## 4. Rules

These rules are settled. Implement them exactly, the same way on every page.

### 4.1 Team names

```
teamNameOf(side):
  1. the teamName on the STANDINGSCSV row whose teamCode === side, if non-blank
  2. else the teamName of baseTeamOf(side), if it has one
  3. else null
sideLabel(side):
  teamNameOf(side) ?? (stage !== group ? `Seed ${slot} · TBD` : side)
```

Wherever a page shows a team, show `sideLabel`. Beside it, show the **base
team letter** as a small chip (`A`), never the raw code (`SF-A_2`). Omit the
chip when there is no base team.

### 4.2 Group standings — from `STANDINGSCSV` only

- One table per group (`bracket` `1`, `2`, `3`), built from rows `A`–`O`
  only.
- **Rank by `quotient`, highest first. Nothing else.** No wins, no
  head-to-head, no points difference. Rows tied on quotient keep their
  order in `STANDINGSCSV`. Use a stable sort.
- Before the first result every quotient is `null`. Show teams in
  `STANDINGSCSV` order, with **no rank numbers** and quotient shown as `—`.
  Rank numbers appear once any row in that group has a quotient.
- Columns, in order: **rank**, **team** (name plus letter chip), **PF**
  (`totalPoints`), **PA** (`totalOpponentPoints`), **Quotient**, to 4
  decimals. The existing `fmtQ()` formats it. Quotient is the ranking
  column, so bold it.
- Matchups won and lost are **not** in this table. They show on the matchup cards
  (§5.6).

### 4.3 Who advances — the workbook decides

Four teams reach the semifinals: the three group winners and the best
runner-up. **The site does not work this out.** The organiser enters the
four qualifiers in the workbook (§3.4), and the site reflects that entry:

- A team **advances** when its letter is the slot of any `SF-*` side in the
  matches (`SF-A` → team A advances). Mark it **"Advances"** on its group
  table.
- Before any `SF-*` slot is a team letter, show no labels.
- Slots fill one at a time as the organiser types, so labels can appear one
  by one. That is expected.
- The site never predicts or checks the choice — not even on equal
  quotients. If the workbook and the table seem to disagree, the workbook
  is right.

### 4.4 Who wins a matchup — total points

- **Matchup score** = each side's points added up over the matchup's four matches.
  Unplayed matches count 0.
- **A matchup is final when all four of its matches are played.** Then:
  - the side with more total points **wins the matchup**, and the other loses;
  - **equal total points** = the matchup is a **tie**. Show it as "Tie" and
    name no winner. Control Center also shows a warning
    (§9.6).
- **Pair wins never decide a matchup.** A team can win three pairs 11–9, lose
  the fourth 0–11, and lose the matchup 33–38. The **pair count** (matches won
  per side, e.g. `3–1`) is shown as secondary information only.
- **Before a matchup is final:**
  - no match played → **not started**. Show the time and courts, no score;
  - some played → **in progress**. Show the running points and an "In
    progress" marker, never a winner.

Put this in one function and use it everywhere:

```js
// ms = the matchup's matches (same matchUp). Returns
// { side1, side2, pts1, pts2, pairs1, pairs2, played, total, state, winner }
// state: 'not-started' | 'in-progress' | 'final' | 'tie'
// winner: 1 | 2 | null
function teamMatchupResult(ms){ … }
```

`side1` and `side2` are the sides of `teamCode1` and `teamCode2` of the
matchup's first match. Every match in a matchup has the same two sides in the same
order.

### 4.5 Stage labels

| Stage | Label | Order |
| --- | --- | --- |
| group | `Group 1` / `Group 2` / `Group 3` (from the base team's `bracket`) | 0 |
| `SF` | `Semifinal` | 1 |
| `Br` | `Bronze` | 2 |
| `Fi` | `Final` | 3 |

Pair labels: `1` Men's Doubles, `2` Women's Doubles, `3` Mixed Doubles,
`4` Mixed Doubles. Short forms for tight spaces: `MD`, `WD`, `XD`, `XD`.

---

## 5. Public site — `events/pickledrive-anniversary-2026/index.html`

Start from `_templates/standard-tournament-template/`. Its day handling,
polling, Live Matches board and Match Finder shell carry over. Everything
built around categories and pairs is replaced.

### 5.1 Create the event folder

The folder already exists and holds `assets/`. Copy the template's
**contents** into it:

```bash
cp -r _templates/standard-tournament-template/. events/pickledrive-anniversary-2026/
```

**Check:** `ls events/pickledrive-anniversary-2026` lists `assets`,
`index.html` and `schedule.html`, with no nested template folder.

### 5.2 Replace the `{{TOKENS}}`

In both `index.html` and `schedule.html`:

| Token | Value |
| --- | --- |
| `{{EVENT_KEY}}` | `pickledrive-anniversary-2026` |
| `{{EVENT_TITLE}}` | `PickleDrive Club One Year Celebration` |
| `{{EVENT_TAGLINE}}` | `Not just a tournament, it's a celebration!` |
| `{{EVENT_HEADLINE}}` | `Team Tournament` |
| `{{EVENT_DATE_RANGE}}` | `October 3, 2026` |
| `{{VENUE}}` | `Kingcourts` |
| `{{EVENT_LOGO}}` | `assets/logo.png` |
| `{{SCHEDULE_DAY_KEY}}` | `pickledrive-anniversary-2026-day1` |
| `{{QR_IMAGE}}` | `assets/qr.png` — set it even if the file isn't there yet. The QR arrives later and is dropped in at that path, so no code change is needed then |
| `{{QR_URL}}` | `tinyurl.com/SAGExPickleDrive2026` |

**Check:** `grep -rn '{{' events/pickledrive-anniversary-2026/` prints
nothing.

**Check the short link resolves to this page:**

```bash
curl -sI https://tinyurl.com/SAGExPickleDrive2026 | grep -i '^location'
```

It should print `https://sage-match-control.github.io/events/pickledrive-anniversary-2026/`
(a trailing slash or none are both fine). If it points anywhere else, or
nowhere, **don't change anything** — report it. The link belongs to the
organiser.

Until `qr.png` exists, the QR panel shows a broken image. That is expected;
report whether the file was present.

### 5.3 Hero

The template's hero reads `{{EVENT_TITLE}}` in the `<h1 class="title">`.
The logo already says "PickleDrive Club", so replace the `<h1>` line with:

```html
<h1 class="title">One Year <span class="accent">Celebration</span></h1>
```

Change the subtitle paragraph (`<p class="subtitle">`) to:

```html
<p class="subtitle">Follow every team, every matchup and every court — live.</p>
```

Change the search label and placeholder:

| Anchor | New text |
| --- | --- |
| `Enter your pair's names` | `Search a team or a player` |
| `placeholder="e.g. John Smith / Jane Doe"` | `placeholder="e.g. DinkTadors or Tristen Pamintuan"` |

The event logo is square with a photographed background. In
`.hero-logos img`, remove `background:var(--white);` and set
`object-fit:cover;`. Keep the 14px radius, so it reads as a rounded tile.

### 5.4 Configuration block

Between `// CONFIGURATION` and `// END CONFIGURATION`:

- `DAYS` → `[{ key: 'pickledrive-anniversary-2026-day1', label: 'Oct 3', date: '2026-10-03' }]`
- `FACILITIES` → `[{ name: 'Kingcourts', courts: [1, 10] }]`
- **Delete** `DIVISIONS`, `EVENTS`, `DIVISION_ORDER`, `EVENT_ORDER`, their
  `_RESOLVED` constants, `CODE_REGEX`, `STAGE_META`, `STAGE_ORDER` and
  `BADGE_CLASSES`. Then delete or rewrite every function that used them.
  §5.5–§5.9 say what replaces each. **Check:** a grep for each deleted name
  finds nothing.
- **Add:**

  ```js
  // ---- Team format --------------------------------------------------------
  // Pair number (the digit after "_" in a team code) -> label.
  const PAIRS = {
    1: { full: "Men's Doubles",   short: 'MD' },
    2: { full: "Women's Doubles", short: 'WD' },
    3: { full: 'Mixed Doubles',   short: 'XD' },
    4: { full: 'Mixed Doubles',   short: 'XD' }
  };
  // Playoff stage prefix (before "-" in a side) -> label and display order.
  // A side with no "-" is a group-stage side.
  const STAGES = {
    SF: { label: 'Semifinal', order: 1 },
    Br: { label: 'Bronze',    order: 2 },
    Fi: { label: 'Final',     order: 3 }
  };
  ```

- **Local test fixtures.** Replace `snapshotUrlFor` with:

  ```js
  // Local testing only: on localhost, ?fixture=<name> loads
  // /_fixtures/<EVENT_KEY>/<name>.json instead of the published snapshot.
  // _fixtures/ is never published (Jekyll skips "_" folders), and the
  // hostname check keeps this inert on the live site.
  const FIXTURE = ['localhost', '127.0.0.1'].includes(location.hostname)
    ? new URLSearchParams(location.search).get('fixture')
    : null;
  const snapshotUrlFor = dayKey => FIXTURE
    ? `/_fixtures/${EVENT_KEY}/${encodeURIComponent(FIXTURE)}.json?t=${Date.now()}`
    : `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${EVENT_KEY}/data/${dayKey}.json?t=${Date.now()}`;
  ```

### 5.5 Parsing and shared helpers

- **`rowsToMatches`** — also read `matchUp` (trimmed) into `m.matchUp`.
  Keep everything else.
- **`rowsToStandings`** — rewrite for §3.3. Return
  `{ teamCode, teamName, pf, pa, quotient, bracket }`. `quotient` is `null`
  unless the cell parses as a number. Skip rows with a blank `teamCode`.
- **Replace `parseCode`** with helpers built on §3.4. Keep a `Map` cache, as
  the template does, because these run on every 10-second render.

  ```js
  sideOf(code)        // "SF-A_2" -> "SF-A"
  pairOf(code)        // "SF-A_2" -> 2   (number, or null)
  stageOf(side)       // "SF-A" -> "SF";  "A" -> null (group)
  slotOf(side)        // "SF-A" -> "A";   "SF-1" -> "1";  "A" -> "A"
  baseTeamOf(side)    // "A" -> "A"; "SF-A" -> "A"; "SF-1" -> null
  teamNameOf(side)    // §4.1
  sideLabel(side)     // §4.1
  groupOf(side)       // bracket of baseTeamOf(side), as a number, or null
  stageLabel(side)    // "Group 1" / "Semifinal" / "Bronze" / "Final"  (§4.5)
  ```

- **Matchups.** After each load, build the matchups:

  ```js
  let TEAM_MATCHUPS = [];  // [{ matchUp, matches:[…], side1, side2, stage, group, times:[…], courts:[…], result }]
  ```

  Group `MATCHES` by `matchUp`. Ignore matches with a blank `matchUp`.
  `times` = the distinct `Schedule` values, in match order. `courts` = the
  distinct `CourtAssignment` values. `result` = `teamMatchupResult(matches)` (§4.4).
  Order matchups by their lowest match number.
- Delete what only served categories or pairs: `divisionMeta`,
  `divisionLabel`, `divisionEventLabel`, `categorySortKey`,
  `matchInstanceOf`, `roundKeyword`, `roundLabel`, `standingsStageKey`,
  `isEmptyStanding`, `standingRowClass`, `pairCell`, `rankStandings`,
  `headToHeadWins`, `renderStageTables`, `pairUpMatchups`,
  `scoreForTeamCode`, `matchupSideHtml`, `matchupScoreHTML`,
  `matchupInstanceLabel`, `renderMatchup`, `renderCategoryCard`,
  `renderCategoryToggleBar`, `renderCatAutocomplete`, `selectCategory`,
  `updateClearCatBtn`, `pairKey`, and the category toggle and filter state.
  If one of them turns out to be used by something this spec keeps, keep it
  and say so in your report.

  **Name clash to avoid.** The template's `pairUpMatchups`, `renderMatchup`
  and `matchup*` functions are unrelated. They pair up the two sides of a
  bracket match in a standard tournament, and Control Center still uses them
  for standard and dual-meet events. Every new name in this spec therefore
  starts with `teamMatchup` or `TEAM_MATCHUP` (`teamMatchupResult`,
  `teamMatchupCardHTML`, `TEAM_MATCHUPS`). Keep that prefix on anything new
  you add for this format.

### 5.6 The matchup card

One component, used by Standings, Match Finder and Control Center. Write it
as `teamMatchupCardHTML(matchup, { focusBase = null, compact = false })`.

```
┌──────────────────────────────────────────────────────────────┐
│ GROUP 1 · 9:00 & 9:25 AM · Courts 1–2          [In progress] │
│                                                              │
│  DinkTadors   (A)        33 – 20        LOB is in the Air (B)│
│  WINNER                  pairs 2–1                           │
│ ──────────────────────────────────────────────────────────── │
│  #1   MD  Tristen Pamintuan / Errol Cunanan  11–9  Jr Pineda / Patrick Garcia │
│  #2   WD  Luisse Rutao / Roxanne Pabalan     11–9  DJ Mallari / Jacq David    │
│  #11  XD  Jello Miranda / Weng Sagum         11–2  Jr Cancio / Nikki Calalang │
│  #12  XD  Lineup not set                       –   Lineup not set             │
└──────────────────────────────────────────────────────────────┘
```

- **Header:** stage label, the matchup's times joined with ` & `, and its
  courts. Show consecutive courts as a range (`Courts 1–2`), otherwise a
  list. Then a state pill: *In progress*, *Final* or *Tie*.
  Show no pill when not started.
- **Headline:** `sideLabel` and letter chip for each side. The **matchup score**
  (`pts1 – pts2`) sits large in the middle, once any match is played.
  **"pairs X–Y"** goes small under it. On a final matchup, mark the winner
  (the word *Winner* under their name and a winning style) and mute the
  loser. A tied matchup marks neither.
- **Match rows:** one per match, in match-number order. Each shows the
  match number, pair short label, both sides' players and the score. A
  played match bolds the higher score. A side with no lineup reads
  *Lineup not set*. A match live on a court right now shows a live dot and
  `Court N`.
- `focusBase` (a base team letter): put that team on the **left**, swapping
  sides and scores on display only, and give it the template's "You" badge.
- `compact`: headline only, match rows hidden behind a *Show matches*
  toggle. Standings uses this for group matchups (§5.7).

### 5.7 Standings view

Replace `renderStandings` and `renderStandingsBody` with one
`renderStandings()`. Keep `captureScrollPositions` and
`restoreScrollPositions` around the rebuild, as the template does. Remove
the category toggle bar, the `#categoryFilterWrap` markup, and their event
listeners.

Top to bottom:

1. **Groups** — the three tables from §4.2 and §4.3, headed *Group 1*,
   *Group 2*, *Group 3*. Side by side on desktop, stacked at 375px.
2. **Playoffs** — the Semifinal matchups, then Final, then Bronze, as full matchup
   cards. Before the playoffs exist in the data, the cards still show
   (`Seed 1 · TBD vs Seed 2 · TBD`), because the matches are always in the
   CSV.
3. **Group matchups** — a collapsible section per group, closed by default,
   holding that group's 10 matchups as `compact` matchup cards.

Empty state (no `STANDINGSCSV` rows at all): *No standings yet.*

### 5.8 Match Finder

Replace the pair index (`rebuildTeamIndex`, `allTeams`, `resolveTeam`,
`runSearch`, `renderAutocomplete`, `renderIntro`, `ticketHTML`) with a
team-and-player index.

- **Index**, rebuilt on every load:
  - one **team** entry per `STANDINGSCSV` row `A`–`O`: `{ kind:'team', base:'A', label:'DinkTadors' }`;
  - one **player** entry per distinct trimmed player name in `MATCHES`,
    excluding `'TBD'`: `{ kind:'player', name }`. Match names
    case-insensitively.
- **Autocomplete** (same keyboard behaviour as now): up to 8 suggestions,
  **teams first**, then players. A team shows its letter chip. A player
  shows the team name(s) they've played for, from the matches they appear
  in.
- **Team result:** a heading with the team name, letter chip and group.
  Under it, every matchup involving that base team, in schedule order, as full
  matchup cards with `focusBase` set. That covers playoff matchups too, once the
  slot resolves to that team. Mark the first not-started matchup *Next up*, as
  the template's `isNext` does.
- **Player result:** the player's name, then every **match** they play,
  each shown as its matchup's card **with only that match's row expanded** and
  the other rows hidden. The heading under the name reads e.g. *DinkTadors
  · 4 matches*.
- If any matchup involving the searched team — or, for a player search, any
  matchup at all — still has *Lineup not set*, add one line under the results:
  *Some lineups aren't in yet. Matches appear here once the team captain
  enters them.*
- **Empty and ambiguous states:** keep the template's wording pattern
  ("No matches found for …", "Several … match — pick one").
- **Intro** (nothing searched): three stats — *136 Matches*, *34 Matchups*,
  *15 Teams* (counted from the data) — then the full team list as tappable
  chips grouped by group. Tapping a chip runs that team's search.
- **Saved search** (`STORAGE_SEARCH_KEY`): store `team:A` or
  `player:<name>`, never a playoff code. Restore it on reload.

### 5.9 Live Matches

Keep the court-by-court table: one row per court 1–10, every court always
shown, driven by the `court` column. Change what a row shows:

- The existing category line (`live-cat-cell`) becomes **`<team 1> vs <team 2>`**
  (`sideLabel`s), then ` · ` and the stage label and pair label, e.g.
  *DinkTadors vs LOB is in the Air · Group 1 · Women's Doubles*.
- The team cells show the players, as now, with the base-team letter chip
  in place of the raw code.
- Under the match score, add the **running matchup score**:
  *Matchup 33 – 20* (both sides' points over that matchup's played matches, §4.4).
- The *No match playing* row stays as it is.

### 5.10 Footer

Leave the footer as the template has it, with the tokens from §5.2.

**Check for §5:** §8.2 (the `pre` fixture) passes.

---

## 6. Theme and fonts — from the pubmat

The pubmat is deep forest green and antique gold on cream, with a serif
display face, a wide-tracked geometric sans and a brush script. Replace the
S.A.G.E. house navy and green with it. The colours below were **sampled
from the pubmat's pixels**, not judged by eye.

| Pubmat element | Sampled |
| --- | --- |
| "ONE YEAR", "TEAM TOURNAMENT", icon rings | forest `#092114` |
| "CELEBRATION", "It's a Celebration!" | gold `#83602A` |
| Inclusions panel | cream `#E8DED3` |
| Trees | olive `#46511F` |
| Sky | `#88BEE5` |

### 6.1 Palette

Apply to the `:root{ … }` block under the `THEME` banner in **both**
`index.html` and `schedule.html`. Leave the role aliases (`--court:var(--green);`
and the rest) unchanged. The token names stay the same (`--navy`,
`--green`) so every rule keeps resolving; only their values change.

| Token | Old | New | Role |
| --- | --- | --- | --- |
| `--navy` | `#14263C` | `#102A1E` | Forest — structure, hero, headings |
| `--navy-deep` | `#0B1826` | `#092114` | Pubmat's darkest forest |
| `--green` | `#7CB92C` | `#C9A45C` | Light gold — the accent **fill** |
| `--green-dark` | `#5C8F1F` | `#7A5824` | Dark gold — accent **text** on light backgrounds |
| `--paper` | `#F6F7F2` | `#F4EEE6` | Warm cream page |
| `--paper-dim` | `#ECEEE6` | `#E8DED3` | Pubmat cream |
| `--line` | `#D9DED2` | `#D8CCBD` | Warm rule |
| `--ink` | `#14263C` | `#102A1E` | Same as `--navy` |
| `--ink-soft` | `#5B6B74` | `#56605A` | Muted green-grey |
| `--white`, `--radius` | — | unchanged | |

Measured contrast (WCAG 2.x):

| Pair | Ratio | Use |
| --- | --- | --- |
| `--ink` on `--paper` | 13.28 | body text |
| `--ink-soft` on `--paper` / `--paper-dim` | 5.66 / 4.92 | secondary text |
| `--green-dark` on white / `--paper` / `--paper-dim` | 6.46 / 5.61 / 4.87 | accent text |
| `--navy` text on `--green` fill | 6.52 | chips, pills, active tabs |
| `--green` on `--navy` / `--navy-deep` | 6.52 / 7.22 | gold text in the hero |
| white on `--navy` | 15.31 | hero text |
| `--green` on white | **2.35 — fails** | never use `--green` as text on a light background |

### 6.2 Hard-coded copies of the old colours

Some rules use the old colours as raw values. Replace every occurrence and
keep each alpha (the last number):

| Find | Replace | Files | Was |
| --- | --- | --- | --- |
| `rgba(124,185,44,` | `rgba(201,164,92,` | both | `--green` |
| `rgba(92,143,31,` | `rgba(122,88,36,` | `index.html` | `--green-dark` |
| `rgba(11,24,38,` | `rgba(9,33,20,` | both | `--navy-deep` |
| `rgba(20,38,60,` | `rgba(16,42,30,` | `index.html` | `--navy` |
| `rgba(246,247,242,` | `rgba(244,238,230,` | `index.html` | `--paper` |
| `.badge-2{background:#4C7A19;}` | delete the rule (`BADGE_CLASSES` is gone) | `index.html` | |
| `'#14263C'` in `readableOn()` | `'#102A1E'` | `schedule.html` | chip text |
| `color:'#5B6B74'` in `FALLBACK` | `color:'#56605A'` | `schedule.html` | |

**Leave alone:** `rgba(20,27,44,…)` (neutral shadow), the error reds
(`#B3261E`, `#FFB4A2`, `rgba(179,38,30,…)`, `rgba(255,180,162,…)`,
`rgba(255,90,110,…)`), `rgba(52,199,120,…)`, `#6C5CE0`, and every
`rgba(255,255,255,…)` and `rgba(0,0,0,…)`.

**Check:** both print nothing:

```bash
grep -n "#14263C\|#0B1826\|#7CB92C\|#5C8F1F\|#F6F7F2\|#ECEEE6\|#D9DED2\|#5B6B74\|#4C7A19" events/pickledrive-anniversary-2026/*.html
grep -n "rgba(124,185,44\|rgba(92,143,31\|rgba(11,24,38\|rgba(20,38,60\|rgba(246,247,242" events/pickledrive-anniversary-2026/*.html
```

### 6.3 Fonts

Pubmat faces and their nearest Google Fonts:

| Pubmat | Google Font | Role on the site |
| --- | --- | --- |
| "ONE YEAR" — heavy high-contrast serif | **Playfair Display** 700/800 | Hero title and section headings only |
| "CELEBRATION", "TEAM TOURNAMENT", "INCLUSIONS" — wide-tracked geometric sans caps | **Montserrat** 500/600/700/800 | Labels, tabs, table headers, body, numbers and scores |
| "Not just a tournament, It's a Celebration!" — brush script | **Satisfy** | Hero tagline only |

Steps, in both files:

1. Replace the Google Fonts `<link>` with:

   ```html
   <link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;500;600;700;800&family=Playfair+Display:wght@700;800&family=Satisfy&display=swap" rel="stylesheet">
   ```

2. `'Archivo Black'` → `'Montserrat'`, and add `font-weight:800;` to that
   rule (Archivo Black has a single heavy weight, so those rules often set
   none or `400`). Scores and numbers also get
   `font-variant-numeric:tabular-nums;`.
3. `'Barlow Condensed'` → `'Montserrat'`. Keep the rule's weight. Montserrat
   is much wider than a condensed face, so for every rule you change, halve
   its `letter-spacing` and check it at 375px. If text wraps or overflows
   where it didn't before, reduce that rule's `font-size` by 1–2px. Don't
   restructure the layout.
4. `'Inter'` → `'Montserrat'`.
5. Then override only these rules:
   - `h1.title` — `font-family:'Playfair Display',serif; font-weight:800;`.
     `h1.title .accent` stays `color:var(--green)`, so "Celebration" is gold
     on forest, as on the pubmat.
   - `.domination-line` (the "Team Tournament" line) — Montserrat 700,
     `letter-spacing:.3em`, `color:var(--white)`.
   - `.tagline-script` — `font-family:'Satisfy',cursive; font-style:normal;
     font-weight:400; color:var(--green); font-size:clamp(20px,5vw,28px);`
   - Section headings — the Standings group and playoff titles and the
     Match Finder result `h2` — `font-family:'Playfair Display',serif;
     font-weight:700;`.
6. `.hero-logos-sep` — Montserrat 800.

**Check:** `grep -n "Archivo Black\|Barlow Condensed\|'Inter'" events/pickledrive-anniversary-2026/*.html`
prints nothing.

### 6.4 Contrast note

In `index.html`'s `THEME` banner comment, replace the lines about
`--green` and `--green-dark` contrast with:

```
       - --green is a light gold FILL (~2.35:1 on white — never text on a
         light background). Put --navy text on it (~6.5:1), or use it as
         text on a --navy/--navy-deep panel (~6.5 / 7.2:1).
       - --green-dark (dark gold) is safe for text on white, --paper and
         --paper-dim (6.5 / 5.6 / 4.9:1).
       - Small text on paper is --ink (~13.3:1) or --ink-soft (~5.7:1).
     PickleDrive Club: palette sampled from the event pubmat (forest
     #092114, gold #83602A, cream #E8DED3). Fonts: Playfair Display,
     Montserrat, Satisfy.
```

---

## 7. Schedule board — `events/pickledrive-anniversary-2026/schedule.html`

The venue wall display: courts as columns, time slots as rows, one card
per match. It fetches the same snapshot. Nothing links to it except
Control Center's *Open schedule* button.

1. **Fixture override** — in `fetchDaySnapshot()`, build the URL with the
   same localhost `?fixture=` rule as §5.4.
2. **Team names** — `buildScheduleData()` also parses
   `snapshot.facilities[].standingsCsv`. From the `teamCode`, `teamName` and
   `bracket` columns build the maps §4.1 needs. `parseFacilityCsv()` also
   reads `matchUp`.
3. **Colour key** — the organiser's SCHEDULE tab is uncoloured, so choose
   the colours from the pubmat. Replace `CAT_META` and set `m.cat` from the
   match's group, or `PO` for any playoff match:

   ```js
   const CAT_META = {
     G1: { short:'GROUP 1',  color:'#4F5B24' },  // olive (trees)
     G2: { short:'GROUP 2',  color:'#88BEE5' },  // sky
     G3: { short:'GROUP 3',  color:'#C9A45C' },  // gold
     PO: { short:'PLAYOFFS', color:'#102A1E' }   // forest
   };
   ```

   `readableOn()` already picks the text colour per hue: forest on the two
   light hues, white on the two dark ones.
4. **`stageOf(code)`** — rewrite for §3.4: side prefix `SF` → `'SF'`,
   `Br` → `'B'`, `Fi` → `'F'`, anything else → `'RR'`. Keep `STAGE_META`'s
   labels as they are.
5. **Card (`cellHTML`)** — in each `fo-side`, above the player names, add
   the team name as `<span class="fo-team">…</span>`: one line, ellipsis on
   overflow, Montserrat 700. Next to the stage chip in `cell-top`, add the
   pair short label (`MD`/`WD`/`XD`). A side whose lineup isn't set (player
   name equal to its code) shows *Lineup TBD* in place of the names.
6. Theme and fonts: §6.1–§6.3 already cover this file.

**Check:** §8.2 and §8.4 schedule-board items pass.

---

## 8. Verify the site

### 8.1 Serve it

From `sage-match-control.github.io/` (the repo root, so root-absolute
`/assets/…` paths resolve):

```bash
npx --yes http-server -p 8000 -c-1
```

Open `http://localhost:8000/events/pickledrive-anniversary-2026/?fixture=<name>`
and `…/schedule.html?fixture=<name>`. All three fixtures have `isLive: true`,
so Live Matches and Standings are visible. Without `?fixture=`, the page
fetches the real snapshot, which 404s until the first sync. That is
expected and shows *schedule isn't loaded yet*.

Check every item at **375px** and at desktop width. The browser console
must show no errors.

### 8.2 `?fixture=pre` — before the event

- [ ] Hero, top to bottom: the logo tile and the S.A.G.E. logo, the gold
      script tagline, "OCTOBER 3, 2026 · KINGCOURTS", "ONE YEAR
      **CELEBRATION**" (serif, "Celebration" gold), "TEAM TOURNAMENT"
      (wide-tracked).
- [ ] Standings: three groups of five, in sheet order (A–E, F–J, K–O), all
      names shown, no rank numbers, quotient `—`, no *Advances* labels.
- [ ] Playoffs: 2 semifinal cards reading `Seed 1 · TBD vs Seed 2 · TBD` and
      `Seed 3 · TBD vs Seed 4 · TBD`, then Final and Bronze cards with seeds.
- [ ] Match Finder: typing `dink` suggests exactly 8 teams and no players:
      DinkTadors, Dink ang Bato, Royal Dinkers, Dink me Baby one More Time,
      Can i buy you a DINK?, KaDINKo, Lord of the Dinks, Tito Dink & Joey. Picking DinkTadors lists its 4 group matchups with every row
      *Lineup not set*, then the lineup note. No playoff matchups, because no
      slot is `A` yet.
- [ ] Searching `Tristen` finds no player (no lineups yet). The empty state
      shows.
- [ ] Live Matches: courts 1–10, all *No match playing*.
- [ ] Schedule board: 10 court columns, each card showing both team names
      and *Lineup TBD*. Group colours olive, sky and gold, playoffs forest.
- [ ] Colours are forest, gold and cream, with no navy or house green
      anywhere. No text is set in a condensed font.

### 8.3 `?fixture=finished` — after the event

Every match is 11–1, so every matchup is 44–4.

- [ ] Group 1: A DinkTadors 11.0000, B 2.4286, C 1.0000, D 0.4118, E 0.0909.
      Groups 2 and 3 have the same shape (F, K first).
- [ ] *Advances* on A, F, K and B — the four letters in the `SF-*` slots.
      Nothing marks G or L, even though they are tied with B at 2.4286:
      the workbook chose B (§4.3).
- [ ] Semifinals: DinkTadors vs Pampanga Wildcards, KaDINKo vs LOB is in the
      Air. Final: DinkTadors **Winner** vs KaDINKo, 44–4, pairs 4–0.
      Bronze: Pampanga Wildcards beat LOB is in the Air. Every playoff card
      lists 4 matches.
- [ ] Searching DinkTadors shows 4 group matchups, a semifinal and the Final:
      6 cards, DinkTadors always on the left.
- [ ] Searching `Tristen Pamintuan` shows 6 matches, each inside its matchup
      card, with only that match's row showing.

### 8.4 `?fixture=edge` — hand-built cases

| Case | Where | Expect |
| --- | --- | --- |
| Pairs don't decide a matchup | A v B (#1, 2, 11, 12): A wins three 11–9, loses 0–11 | **LOB is in the Air wins.** The card reads `33 – 38`, "pairs 3–1", with B marked *Winner* |
| Tie | C v D (#3, 4, 13, 14) | 38–38, *Tie*, no winner marked |
| Final on points | Final (#129, 130, 133, 134): 11–5, 11–5, 3–11, 4–11 | **KaDINKo wins 32–29**, pairs 2–2 |
| In progress | Bronze (#131 11–8, #132 live, #135–136 unplayed) | *In progress*, `11 – 8`, no winner. Live Matches court 6 shows Pampanga Wildcards vs LOB is in the Air · Bronze · Women's Doubles · *Matchup 11 – 8* |
| Lineup not set | K v O (#107, 108, 117, 118) | Every row *Lineup not set*, no score. On the schedule board, *Lineup TBD* |
| Ranking | Group 1 quotients: E 3.0, C 1.5, B 1.5, A 0.8, D 0.2. `STANDINGSCSV` lists C before B | Group 1 ranks **E, C, B, A, D** |

(The `edge` fixture's points columns were not recomputed after its
quotients were changed, so PF and PA won't match the quotients. Only the
ranking is under test.)

---

## 9. Control Center — `tools/control-center.html`

The operator console. It is **one file shared by every event**, and it
branches on the event's `type` from `event-data/config/events.json`, held
in `CURRENT_TYPE`. Add a third type, **`"team"`**.

**The two existing types must behave exactly as before.** Every change
below is either inside a `CURRENT_TYPE === 'team'` branch or additive
(e.g. an extra parsed field nothing else reads).

Port the logic from §5 rather than re-deriving it. Copy `teamMatchupResult`, the
§5.5 helpers, `teamMatchupCardHTML` and their CSS into the console. Rename any that
collide with an existing console function (`team`-prefix them), and keep
them together in one clearly bannered block:
`// ---- team type (sage-docs/docs/specs/.../team-tournament-spec.md) ----`.

### 9.1 Config and type

- In `selectEvent`, the type check becomes
  `rawEvent.type === 'dual-meet' || rawEvent.type === 'standard' || rawEvent.type === 'team'`.
- Update the error message after it to read
  `(must be "dual-meet", "standard" or "team")`.
- Update the `CURRENT_TYPE` declaration comment to list `'team'`.

### 9.2 Local fixtures

Give `configUrlFor` and `snapshotUrlFor` the same localhost-only override
as §5.4:

- `?fixture=<name>` loads `/_fixtures/config.json` as the registry;
- the snapshot becomes `/_fixtures/<eventKey>/<name>.json`.

Nothing changes on the live site.

### 9.3 Parsing

- `rowsToMatches` — also read `matchUp`. This is additive.
- `rowsToStandings` — also read `teamName`, `totalPoints` and
  `totalOpponentPoints` when those columns exist. This is additive: existing
  events have no such columns, so the fields stay empty.
- `parseCode` — add a `team` branch returning
  `{ club:null, category:null, rest }`, so no shared code breaks. Team
  views use the §5.5 helpers, not `parseCode`.

### 9.4 Standings

In `renderStandings`, branch to a new `renderTeamStandings()` **before**
the category code runs (`categoryTokensPresent`, `buildCategoryOrderIndex`).
Hide the category toggle bar and filter for `team`, as dual meets do.
`renderTeamStandings()` renders exactly §5.7.

### 9.5 Live Matches, Match Finder, intro

Branch `liveTableRowsHTML`, `rebuildTeamIndex`, `resolveTeam`,
`runSearch`, `renderAutocomplete` and `renderIntro` on `team` to the §5.8
and §5.9 behaviour. The console builds its courts from the data
(`buildFacilityCourtGroups`), not from config, so leave that alone.

### 9.6 Awards

For `team`, `renderAwards` uses a new `buildTeamPodium()` instead of
`buildPodiums()`. It returns one podium in the existing shape:

```js
{ category:'__team__', source:'bracket',
  gold, silver, bronze,          // each null (Pending) or a medalist
  bronzeWalkover:false, warning, orderIdx:0 }
```

- **Gold** = the Final matchup's winner. **Silver** = its loser. **Bronze** = the
  Bronze matchup's winner. All by §4.4. Each is `null` (*Pending*) until that matchup
  is final.
- **Medalist** = `{ teamName, base, roster, p1:teamName, p2:'', club:null, clubLabel:null }`.
  `roster` = every distinct player who played a match for that base team in
  any matchup, sorted A–Z.
- **A tied Final or Bronze** sets
  `warning: 'Final is a tie on points — check match #…'` (or *Bronze*),
  naming the lowest match number in that matchup. Gold, silver and bronze stay
  `null`.
- `medalRowHTML` and the bronze row in `awardsCardHTML` — when
  `medalist.teamName` is set, show the team name (bold) with the roster
  under it, comma-separated, in `--ink-soft`, instead of `p1 / p2`.
- `categoryLabel('__team__')` → `'Team Championship'`.
- Image export (`drawPodiumCard`, `renderWholeTournamentCard`): when a
  medalist has `teamName`, draw the team name where the pair names go.
  Leave the roster off the image.
- `computeOverallChampion()` already returns `null` for non-dual-meet
  events. Leave it.

### 9.7 Mission Control and facility progress

No change. Progress counts matches, which is still right. The "played" and
BYE rules exist twice: `computeFacilityProgress` and `sideIsBye` here, and
`src/sync/facilityCompletion.mjs` in `sage-tools-api`. Leave both as they
are — this format has no byes.

### 9.8 Theme

Control Center is shared, so it keeps the S.A.G.E. house palette and fonts.
Don't apply §6 here.

### 9.9 Verify

Serve as in §8.1 and open
`http://localhost:8000/tools/control-center.html?fixture=<name>`. Pick the
event if it isn't already selected.

- [ ] `pre`, `finished` and `edge` show the same Standings, Match Finder
      and Live results as §8.2–§8.4.
- [ ] Awards, `finished`: Gold DinkTadors, Silver KaDINKo, Bronze Pampanga
      Wildcards, each with an 8-name roster. *Export image* downloads a
      PNG showing team names.
- [ ] Awards, `edge`: Gold **KaDINKo**, Silver DinkTadors, Bronze *Pending*
      (the matchup is in progress).
- [ ] Awards, `pre`: all three *Pending*.
- [ ] Mission Control's *Open schedule* links to
      `/events/pickledrive-anniversary-2026/schedule`.
- [ ] **Regression**, without `?fixture=` (the live registry): Pickle for
      Sight (`standard`) and the PNF × BUP dual meet (`dual-meet`) render
      Standings, Match Finder, Live Matches and Awards as before, with no
      console errors.

---

## 10. Register the event — `event-data/config/events.json`

Add inside `"events"`, after the last event entry (mind the comma before
it):

```json
"pickledrive-anniversary-2026": {
  "type": "team",
  "title": "PickleDrive Club One Year Celebration",
  "days": {
    "pickledrive-anniversary-2026-day1": {
      "label": "Oct 3",
      "date": "2026-10-03",
      "isLive": "auto",
      "facilities": [
        { "name": "Kingcourts", "sheetId": "1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA" }
      ]
    }
  }
}
```

No `display` block. This type takes its names from the data.

`sage-tools-api` never reads `type`, so `"team"` needs no backend change.
It validates the rest of the file, and **a day key used twice anywhere in
the file rejects the whole file.** Cloud Run then keeps serving the last
good config and this event never registers.

**Check:**

```bash
grep -c '"pickledrive-anniversary-2026-day1"' config/events.json
node -e "JSON.parse(require('fs').readFileSync('config/events.json','utf8'))"
```

The first must print `1`, and the second must run without error. If the
day key already exists, **stop and ask**.

Also update `config/README.md` in the same repo: where it lists
`"dual-meet"` or `"standard"` for `type`, add `"team"`, with one line saying
team events take names from `STANDINGSCSV` and need no `display` block.

---

## 11. Operator runbook copy

Copy `_templates/dry-run-checklist-template.md` to
`events/pickledrive-anniversary-2026/dry-run-checklist.md` and replace
`{{EVENT_TITLE}}`. Then add one step to its rehearsal section:

> Enter one matchup's lineup on the `MatchUps` tab. Within about 30 seconds,
> the site should show those players' names in place of *Lineup not set*.
> If it doesn't, check that `MatchUps` is ticked in **SAGE → Set up live
> sync**.
>
> Then type a team letter into `MatchUps!D604` (Semifinal seed 1). The
> site's first semifinal card should switch from *Seed 1 · TBD* to that
> team's name, and the team should get an *Advances* label. Put the seed
> back to `1` afterwards.

And one step to its day-of section:

> When the group stage ends, enter the four semifinalists' letters in
> `MatchUps` (`D604`, `D614`, `D624`, `D634`). After the semis, enter
> the Bronze and Final teams (`D644`–`D674`). The site takes the
> semifinalists from these cells — it does not pick them itself.

---

## 12. Documentation — `sage-docs`

Write in the present tense: describe what the system does, not what
changed.

| File | Add |
| --- | --- |
| `docs/features/tournament-hub.md` | A *Team events* section: reading a matchup card, why a matchup is won on total points, searching by team or player, *Lineup not set* |
| `docs/features/control-center.md` | The team Standings and the team Awards podium |
| `docs/technical/control-center.md` | `"team"` as a third `CURRENT_TYPE`, and which functions branch on it |
| `docs/technical/event-data-config.md` | `type: "team"` |
| `docs/technical/adding-a-new-event.md` | A team event has no template yet: point to this spec |
| `docs/technical/schedule-board.md` | Team events colour by group, not category |

Root `D:\Personal\SAGE\CLAUDE.md`: where it describes Control Center's
`type`, add `"team"`. Where it lists the `events/` folders, add this event.

Then, per `docs/specs/README.md` ("Moving a spec between folders"), move
this spec from `not-started/` to `in-progress/`: update its status line,
the three READMEs and `mkdocs.yml`.

---

## 13. Report back

- Which checks in §5–§12 passed, and any that didn't, with what you saw.
- Whether `assets/qr.png` was present, and where the short link redirects.
- Any function from §5.5's delete list that you kept, and why.
- Every changed and new file, per repo, uncommitted.

---

## 14. Decisions (settled — don't revisit)

| # | Decision |
| --- | --- |
| D1 | Group ranking is `quotient` from `STANDINGSCSV` only. Rows tied on quotient keep sheet order |
| D2 | A matchup is won on total points across its 4 matches. Pair wins never decide it. Equal points is a **tie**, shown as "Tie", with no winner. The team-vs-team meeting is a **matchup**, never a "tie" |
| D3 | Standings come from `STANDINGSCSV`. Matchup results come from the matches, grouped by `matchUp`. The workbook's `Standings` tab is ignored |
| D4 | Semifinalists are the 3 group winners plus the best runner-up. The organiser enters them in the workbook's `MatchUps` tab. The site shows that entry and never computes or second-guesses it, including on equal quotients |
| D5 | The award roster is every player who played for the team |
| D6 | Event key `pickledrive-anniversary-2026`, day key `pickledrive-anniversary-2026-day1` |
| D7 | 10 courts |
| D8 | Theme and fonts follow the pubmat on the event pages. Control Center keeps the house theme |
| D9 | Public site, schedule board and Control Center all ship before 3 October. The template comes after |

---

## 15. After the event — the template

Not part of this build. Once the event has run:

1. Copy the event folder to `_templates/team-tournament-template/`.
   Tokenise event values and mark `PAIRS`, `STAGES` and `FACILITIES` as
   `// EXAMPLE — replace`.
2. Generalise what this event hard-codes: any number of groups, any number
   of pairs per matchup, and other playoff stages (QF).
3. Move the fixture override (§5.4) into both existing templates as well.
4. Document the template in `_templates/CLAUDE.md` §1 and §3–§5.
5. Consider a *Team Tournament Master* sheet generator. This event's
   workbook was built by hand.

---

## 16. Out of scope

- Any change to `sage-tools-api` or to any Apps Script.
- Team logos.
- Scoresheets. `tools/scoresheet-generator.html` is untested with these
  team codes, so if an operator plans to print from it, flag it rather than
  fixing it.
- Attendance (`attendance.gs`, `attendance.html`).
- Re-theming Control Center.
