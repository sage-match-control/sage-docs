# Spec — Facility progress (matches done, matches left, estimated finish)

Show, for each facility (venue) on a tournament day, how many matches are
done, how many are left, and an estimated finish time. It appears in two
places in the Control Center: as a **card per facility** on the Live Matches
tab, and as **one line per facility** in Mission Control's *Facility sync
status*. Both places flag the estimate when that facility's data is stale.

**Status: not started.**

Everything here runs in the browser, using data the page already loads every
10 s. There is no server change, no data change, no spreadsheet change, and
no new network request.

## How to use this spec

- **§0–§3** explain what exists and what to build. Read them once.
- **§4** is the implementation: exact edits, each followed by a check. Follow
  it literally. The code in §4 is complete; don't redesign it.
- **§5** is verification. Every check has an exact expected result.
- **§6–§7** are the docs update and the commits. Both are part of the job.
- **§8** lists when to stop and ask instead of guessing.
- **§9** records why things are the way they are. Read it before changing
  any number or rule.

---

## 0. Background — what you need to know first

### 0.1 Where things are

`D:\Coding Projects\SAGE` is a plain folder holding separate git repos. This
spec touches two of them:

| Repo | What changes |
| --- | --- |
| `sage-match-control.github.io/` | `tools/control-center.html`, the only code file |
| `sage-docs/` | Two documentation pages, plus this spec's move to `implemented/` (§6) |

`tools/control-center.html` is one self-contained page: inline `<style>`, one
inline `<script>`, no build step, no framework, no package.json, no tests. It
is the operators' console for running a tournament. Its tabs are **Live
Matches**, **Match Finder**, **Standings**, **Awards** and **Mission Control**.

The file uses **CRLF** line endings. The anchors in §4 are printed with plain
newlines. Match them line by line, and make sure the file still has CRLF
endings when you finish (§4 step 6 checks this).

### 0.2 How match data reaches the page

For each tournament day, each facility has a Google Sheet. A sync pipeline
publishes a JSON snapshot of every facility's matches. Every 10 s the console
fetches that snapshot (`loadLiveData`) and re-renders its tabs. Each match row
has a scheduled time (`Schedule`, e.g. `"3:25 PM"`) and a scheduled court
(`CourtAssignment`, e.g. `"Court 3"`). A row gets scores once it's played,
and optionally a live-court marker while it's being played.

### 0.3 Existing identifiers this spec uses

All of these already exist in `tools/control-center.html`, at top level in
its single `<script>`. Don't rename, move or modify any of them.

| Identifier | What it is |
| --- | --- |
| `MATCHES_BY_FACILITY` | `[{ name, matches }]`, one entry per facility in the loaded day, rebuilt on every poll |
| A match (`matches[i]`) | `{ num, time, court, liveCourt, t1, t1p1, t1p2, t2, t2p1, t2p2, t1Score, t2Score, played }`. `time` is the raw `Schedule` text. `court` is the scheduled court text. `liveCourt` is non-empty while the match is being played. `played` is `true` when both scores are present |
| `LAST_SNAPSHOT` | The last fetched snapshot: `{ facilities: [{ name, syncedAt, … }], failedFacilities: [name, …], … }`, or `null` before the first load |
| `DAYS`, `currentDayIndex` | The selected event's days, `[{ key, label, date }]` (`date` is `"YYYY-MM-DD"` or `null`), and the index of the selected one (or `null`) |
| `matchByeSide(m)` | `null` for a real match. Non-null when one side is a bye, which is never played |
| `courtNumberFrom(text)` | `"Court 3"` → `"3"`. `null` if there's no number |
| `parseScheduleTimeToMinutes(text)` | `"3:25 PM"` → `925` (minutes since midnight). `null` if it doesn't parse |
| `formatMinutesAsClock(mins)` | `925` → `"3:25 PM"`. Only valid for 0–1439 |
| `escapeHtml(s)` | HTML-escapes a string |
| `relativeTimeFromNow(iso)` | `"just now"`, `"14 min ago"`, `"2h ago"`, … |
| `STALE_WARNING_MS` | `5 * 60 * 1000`. Mission Control turns a facility's sync row amber when its data is older than this |
| `renderLiveMatches()` | Redraws the Live Matches tab. Called on every poll |
| `renderOrganizerStatus()` | Redraws Mission Control's *Facility sync status* rows. Called on every poll |

Find each one with `grep -n "function <name>(" tools/control-center.html` (or
`grep -n "<name> ="` for the variables). If any of them is missing or has a
different shape from the table, **stop and ask** (§8).

---

## 1. Scope

| In | Out — do not touch |
| --- | --- |
| `sage-match-control.github.io/tools/control-center.html` only | Every event's public `index.html`, both `_templates/`, `schedule.html` boards, archived pages |
| The Live Matches tab (`#liveResults`) | Standings, Match Finder, Awards tabs |
| Mission Control's *Facility sync status* rows (`#orgStatusBox`) | Every other Mission Control control: go-live, sign-in, resync, public pages |
| | `sage-tools-api` (so no `package.json` bump), `event-data`, `scripts/*.gs` |
| | Any existing function's behaviour. Only two existing functions change: `renderLiveMatches` gets **one added line** (§4 step 4), and `renderOrganizerStatus`'s synced-row template gets **one added line and one appended class** (§4 step 5) |
| | `parseScheduleTimeToMinutes` and the go-live logic that uses it (§9) |

---

## 2. What the operator sees

### 2.1 Live Matches — one card per facility

A row of cards between the "courts in play" stat and the court tables, one
card per facility, in `MATCHES_BY_FACILITY` order. The card shows the facility
name even when the day has only one facility.

```
┌──────────────────────────────────────────────┐
│ PCPH MAIN                                    │
│ ████████████░░░░░░░░░░░░░░░░  42 / 116       │
│ Done 42 · Left 74 · In play 3                │
│ Est. finish 9:40 PM · 15 min behind          │
│ Scheduled end 9:25 PM                        │
└──────────────────────────────────────────────┘
```

Line 4 (the finish line) and the optional lines under it:

| Situation | Line 4 | Extra lines |
| --- | --- | --- |
| `left === 0` | `All matches done` | *(none)* |
| `plannedEnd === null` (no match has both a parseable time and a court) | `No scheduled times yet` | *(none)* |
| Event day active (§3.1), `diff > 0` | `Est. finish <eta> · <gap> behind` | `Scheduled end <plannedEnd>` |
| Event day active, `diff < 0` | `Est. finish <eta> · <gap> ahead` | `Scheduled end <plannedEnd>` |
| Event day active, `diff === 0` | `Est. finish <eta> · on schedule` | *(none)* |
| Event day not active, or the day has no `date` | `Scheduled end <plannedEnd>` | *(none)* |

`<eta>` and `<plannedEnd>` are clock labels and `<gap>` is a duration, both
formatted as in §2.4. `diff` is defined in §3.7.

Two more optional lines, in this order, after `Scheduled end`:

1. **Stale** (§2.3), in orange:
   `Data from <relativeTimeFromNow(syncedAt)> — estimate reads late until this sheet syncs again`.
2. **Unscheduled matches**, when `left > 0` and `unscheduledLeft > 0` (§3.3):
   `<n> match(es) without a court/time — counted as work left, not on the schedule`.

### 2.2 Mission Control — one line per facility

Inside *Facility sync status*, each facility row that has synced at least
once gets a second line under the existing one:

```
● PCPH Main    Synced 1 min ago                                    Google Sheet
               42 / 116 done · Est. finish 9:40 PM · 15 min behind
● PCPH Annex   Last attempt failed — showing data from 14 min ago  Google Sheet
               30 / 125 done · Est. finish 9:50 PM · 45 min behind · stale — may read late
```

The line is `<done> / <total> done · <line 4 from §2.1>`, plus
` · stale — may read late` when stale. It never shows the `Scheduled end` or
unscheduled lines: this is deliberately one line. The row's existing first
line (dot, name, sync detail, sheet link) is unchanged. A facility that has
never synced, or has no matches in the snapshot, gets no second line.

### 2.3 Stale

A facility's estimate is **stale** when all three of these hold:

1. The event day is active (§3.1). This includes the hours after midnight
   when play runs late.
2. The facility has matches left (`left > 0`).
3. The facility's data fails the **same test** Mission Control already uses
   to turn its sync row amber: its last sync attempt failed (its name is in
   `LAST_SNAPSHOT.failedFacilities`), **or** its `syncedAt` is older than
   `STALE_WARNING_MS`.

"Behind" and "stale" text is orange `#B3541A`, the colour Mission Control
already uses for a failing sync.

### 2.4 Labels

**Clock labels** (`eventClock`, §4 step 3) turn an event-day minute (§3.1)
into a time. When the minute falls on a later calendar day than the event's
date, they add a day offset:

| Event-day minute | Label |
| --- | --- |
| `1000` | `4:40 PM` |
| `1455` | `12:15 AM (+1 day)` |
| `1500` | `1:00 AM (+1 day)` |
| `2940` | `1:00 AM (+2 days)` |

The offset counts from the **event's date**, not from whenever the page is
read. "12:40 AM (+1 day)" means the same thing at 11 PM and at 12:30 AM.

**Durations** (`formatGap`, §4 step 3) for "behind" and "ahead":

| Minutes | Label |
| --- | --- |
| `20` | `20 min` |
| `60` | `1 h` |
| `75` | `1 h 15 min` |
| `1020` | `17 h` |

---

## 3. The estimate

### 3.1 Time unit: event-day minutes

Every time in this spec is **minutes since midnight Philippine time at the
start of the selected day's `date`**. It is not "minutes since the most
recent midnight". The two only differ after midnight: event-day minutes keep
counting past 1440 instead of wrapping to 0, so 12:30 AM the next morning is
1470, not 30.

`DAY_ROLLOVER_MIN = 360` (6:00 AM) is the cut-off that decides which
night-time values belong to the event day:

| Value | Rule |
| --- | --- |
| A `Schedule` time | `eventScheduleMinutes(raw)`: `parseScheduleTimeToMinutes(raw)`, plus 1440 if that's below 360. So `"12:15 AM"` → 1455, `"5:59 AM"` → 1799, and `"6:00 AM"` → 360 |
| `nowMin` | `eventDayNowMinutes()`: `floor((Date.now() − Date.parse(day.date + 'T00:00:00+08:00')) ÷ 60000)`, or `null` when there's no selected day or it has no `date` |
| **Event day active** | `isEventDayActive(nowMin)`: `nowMin !== null && 0 ≤ nowMin < 1440 + 360`. That is from midnight at the start of `day.date` until 6:00 AM the next morning |

### 3.2 The basis

This is the formula operators already use:

```
time left = (remaining matches + total blank slots − blank slots done) × est. duration ÷ number of courts
```

A facility's schedule is a grid: rows are time slots and columns are courts.
Every cell either holds a match or is **blank**, meaning that court sits idle
in that slot. Blanks come from waiting on a bracket, uneven categories, or a
court that finishes early. They cost time just like matches do, which is why
the formula counts them. §3.3–3.6 define each term precisely and make four
changes to the formula (§3.6 lists them). The est. duration is fixed at the
schedule's own slot length.

### 3.3 Counts

`real` = the facility's matches where `matchByeSide(m) === null`. A bye is
never a match, so its grid cell counts as blank.

| Name | Value |
| --- | --- |
| `total` | `real.length` |
| `done` | `real.filter(m => m.played).length` |
| `left` | `total − done` |
| `inPlay` | `real.filter(m => !m.played && m.liveCourt).length` |
| `unscheduledLeft` | Unplayed real matches where `eventScheduleMinutes(m.time) === null` **or** `courtNumberFrom(m.court) === null` |

A bracket match whose teams are still TBD counts as left.

### 3.4 The grid

Only **placed** real matches go on the grid: ones with both a parseable time
and a court number. Unscheduled matches stay off the grid but still count as
remaining work (§3.5).

| Name | Value |
| --- | --- |
| `slot` | Median of every **positive** gap between consecutive distinct times **on the same court**, over placed matches, played or not. For an even count, take the lower middle value. If there are no gaps, use `25` |
| `start` | Earliest placed time |
| `row(m)` | `Math.round((t − start) / slot)`. Rounding snaps slightly staggered courts (3:25 vs 3:35) into the same row |
| `rows` | `max(row) + 1`. An entirely empty row in the middle, such as a lunch break, still counts |
| `C` (courts) | Number of distinct court numbers among placed matches |
| `occupied` | Distinct `(court, row)` pairs holding at least one placed match |
| `blanks` | `rows × C − occupied` |
| `F` (frontier) | Lowest `row` of any **unplayed** placed match, or `rows` if there are none |
| `blanksDone` | Blank cells whose row is `< F` |
| `plannedEnd` | `start + rows × slot` |

A blank counts as done once every match in its row and every earlier row has
been played.

### 3.5 Remaining work, in cells

```
unitsLeft = (left − 0.5 × inPlay) + (blanks − blanksDone)
```

An in-play match is on average half done, so it counts as half a cell.
`left` already includes `unscheduledLeft`.

### 3.6 Time left, and the changes to the basis formula

```
queue    = max over courts of (that court's unplayed placed matches, in-play counting 0.5)
timeLeft = max(unitsLeft × slot ÷ C,  queue × slot)
eta      = max(nowMin, start) + timeLeft        // only when the event day is active and left > 0; otherwise null
```

The four changes to the basis formula:

1. **Blanks counted by frontier.** A blank is used up only once its whole row
   is finished, not just because its time has passed.
2. **In-play matches count as half done.**
3. **Unscheduled matches still count** as remaining work.
4. **Busiest-court floor.** A match can't be split across courts, so the
   estimate is never shorter than the longest single court's queue. This
   keeps "one match left" from reading as a third of a match.

Before the first match, `eta` equals `plannedEnd` (on schedule). After that,
it's the current time plus the work left, so a late start pushes the whole
day back.

### 3.7 Rounding

```
round5(x) = Math.round(x / 5) × 5
etaShown  = eventClock(round5(eta))
diff      = round5(eta − plannedEnd)        // > 0 behind, < 0 ahead, 0 on schedule
gap       = formatGap(|diff|)
```

Each value is rounded from the unrounded number.

### 3.8 Worked example — the reference for §5 checks 3 and 15

A facility with 3 courts, `slot = 25`, `start` = 3:00 PM (900). Now = 3:55 PM (955).

| Row (time) | Court 1 | Court 2 | Court 3 |
| --- | --- | --- | --- |
| 0 (3:00 PM) | played | played | played |
| 1 (3:25 PM) | played | **in play** | blank |
| 2 (3:50 PM) | unplayed | blank | blank |
| 3 (4:15 PM) | unplayed | unplayed | blank |

- `total 8`, `done 4`, `left 4`, `inPlay 1`, `rows 4`, `C 3`,
  `blanks = 12 − 8 = 4`.
- `F = 1` (the in-play match), so `blanksDone = 0`.
- `unitsLeft = (4 − 0.5) + (4 − 0) = 7.5`.
- `queue`: court 1 has 2 unplayed, court 2 has 0.5 + 1 = 1.5, so `queue = 2`.
- `timeLeft = max(7.5 × 25 ÷ 3, 2 × 25) = max(62.5, 50) = 62.5`.
- `eta = 955 + 62.5 = 1017.5` → `round5` → 1020 → **5:00 PM**.
- `plannedEnd = 900 + 4 × 25 = 1000` (4:40 PM); `diff = round5(17.5) = 20`.

The card reads `Est. finish 5:00 PM · 20 min behind` / `Scheduled end 4:40 PM`.

Move the same grid 8 hours later (11:00 PM to 12:15 AM) with now = 1435
(11:55 PM), and every number is 480 higher: `eta 1497.5`, `plannedEnd 1480`.
The card then reads `Est. finish 1:00 AM (+1 day) · 20 min behind` /
`Scheduled end 12:40 AM (+1 day)`.

---

## 4. Implementation steps

Every edit is in `sage-match-control.github.io/tools/control-center.html`,
anchored on quoted text. Each anchor must match **exactly once**. If one
doesn't, stop and ask (§8).

### Step 0 — Run the page locally

From `D:\Coding Projects\SAGE`:

```bash
python -m http.server 8123 --directory sage-match-control.github.io
```

(`.claude/launch.json` in that folder has the same thing as the `static-site`
configuration.) Open
`http://localhost:8123/tools/control-center.html?event=pnf-x-bup-dual-meet&day=pnf-x-bup-day1`.
The page loads the real event data over the internet. Confirm it shows the
Live Matches tab with a court table before editing anything. Reload after
each step.

### Step 1 — CSS

Insert right after this existing block (its closing `}` included):

```css
  .facility-title .facility-count{
    font-family:'Barlow Condensed',sans-serif;
    font-size:10.5px;
    letter-spacing:.04em;
    text-transform:none;
    color:var(--ink-soft);
  }
```

the following:

```css

  .facility-progress{
    display:grid;
    grid-template-columns:repeat(auto-fit, minmax(240px, 1fr));
    gap:12px;
    margin:0 0 22px;
  }
  .facility-progress:empty{display:none;}
  .fp-card{
    background:var(--white);
    border:1px solid var(--line);
    border-radius:12px;
    padding:14px 18px;
    color:var(--court-line);
    min-width:0;
  }
  .fp-name{
    font-family:'Archivo Black',sans-serif;
    font-size:13px;
    letter-spacing:.03em;
    text-transform:uppercase;
    margin-bottom:8px;
  }
  .fp-bar-row{display:flex; align-items:center; gap:10px; margin-bottom:8px;}
  .fp-bar{flex:1; height:8px; border-radius:4px; background:var(--line); overflow:hidden;}
  .fp-bar > span{display:block; height:100%; background:var(--amber);}
  .fp-frac{font-size:12px; color:var(--ink-soft); white-space:nowrap;}
  .fp-line{font-size:13px; line-height:1.5;}
  .fp-line b{color:var(--cork);}
  .fp-sub{font-size:11.5px; color:var(--ink-soft);}
  .fp-behind{color:#B3541A; font-weight:600;}
  .org-status-row.has-progress{flex-wrap:wrap; row-gap:4px;}
  .org-status-progress{
    flex:1 1 100%;
    padding-left:17px;
    font-size:12px;
    color:var(--muted);
  }
  .org-status-progress b{color:var(--ink); font-weight:600;}
```

Notes, so nothing here gets "fixed":
- `.fp-behind` hard-codes `#B3541A`, the colour `.org-status-warn .org-status-detail`
  already uses. **Don't swap it for `var(--amber)`**: in this file `--amber` is
  `#5C8F1F`, a green, and a warning must not look like good news. The bar
  fill does use `var(--amber)`, because progress is good news.
- `padding-left:17px` is the status dot's 8px plus the row's 9px gap, so the
  second line sits under the facility name.

Check: `grep -c "fp-card{" tools/control-center.html` → `1`, and
`grep -c "org-status-progress{" tools/control-center.html` → `1`.

### Step 2 — Markup

Replace

```html
    <div class="live-board-meta" id="liveBoardMeta"></div>
    <div id="liveBoard"></div>
```

with

```html
    <div class="live-board-meta" id="liveBoardMeta"></div>
    <div class="facility-progress" id="facilityProgress"></div>
    <div id="liveBoard"></div>
```

Check: `grep -c 'id="facilityProgress"' tools/control-center.html` → `1`.

### Step 3 — Functions

Insert the following immediately **before** the line
`function renderLiveMatches(){`, exactly as written:

```js
// ---- Facility progress (sage-docs/docs/specs/.../facility-progress-spec.md) ----
// Done / left / estimated finish per facility, computed from the matches
// already loaded — no fetch. Shown twice: a card per facility on Live Matches
// (renderFacilityProgress, called by renderLiveMatches) and one line per
// facility in Mission Control's sync status (facilityStatusProgressHTML,
// called by renderOrganizerStatus). Both run on every poll. The estimate is
// the operators' own formula — (matches left + blank slots left) × slot
// length ÷ courts — floored at the busiest court's own queue.
const facilityProgressEl = document.getElementById('facilityProgress');

// Event-day minutes (spec §3.1): minutes since midnight PH at the start of the
// selected day's date, so play that runs past midnight keeps counting up
// (12:30 AM next morning = 1470) instead of wrapping to 0. Anything before
// DAY_ROLLOVER_MIN belongs to the night after the event day, not its morning.
const DAY_ROLLOVER_MIN = 6 * 60;

function eventScheduleMinutes(raw){
  const t = parseScheduleTimeToMinutes(raw);
  if(t === null) return null;
  return t < DAY_ROLLOVER_MIN ? t + 1440 : t;
}

function eventDayNowMinutes(){
  const day = currentDayIndex !== null ? DAYS[currentDayIndex] : null;
  if(!day || !day.date) return null;
  return Math.floor((Date.now() - Date.parse(`${day.date}T00:00:00+08:00`)) / 60000);
}

// From midnight at the start of day.date until DAY_ROLLOVER_MIN the next
// morning — so an event running late keeps its estimate past midnight.
function isEventDayActive(nowMin){
  return nowMin !== null && nowMin >= 0 && nowMin < 1440 + DAY_ROLLOVER_MIN;
}

// Event-day minutes → "4:40 PM", or "12:15 AM (+1 day)" once past the
// event date's midnight. The offset is from the event's date, not from now.
function eventClock(mins){
  const r = Math.round(mins);
  const dayOffset = Math.floor(r / 1440);
  const label = formatMinutesAsClock(((r % 1440) + 1440) % 1440);
  return dayOffset > 0 ? `${label} (+${dayOffset} day${dayOffset === 1 ? '' : 's'})` : label;
}

// Minutes → "20 min", "1 h", "1 h 15 min".
function formatGap(mins){
  const h = Math.floor(mins / 60), m = mins % 60;
  if(h === 0) return `${m} min`;
  return m === 0 ? `${h} h` : `${h} h ${m} min`;
}

function computeFacilityProgress(matches, isEventDay, nowMin){
  const real = matches.filter(m => matchByeSide(m) === null);
  const total = real.length;
  const done = real.filter(m => m.played).length;
  const left = total - done;
  const inPlay = real.filter(m => !m.played && m.liveCourt).length;

  const placed = [];
  let unscheduledLeft = 0;
  real.forEach(m => {
    const c = courtNumberFrom(m.court);
    const t = eventScheduleMinutes(m.time);
    if(c === null || t === null){ if(!m.played) unscheduledLeft++; return; }
    placed.push({ c, t, played: m.played });
  });
  const base = { total, done, left, inPlay, unscheduledLeft };
  if(placed.length === 0){
    return { ...base, slot: null, courts: 0, rows: 0, blanks: 0, blanksDone: 0,
             plannedEnd: null, eta: null };
  }

  // Slot length: median positive gap between consecutive times on one court.
  const timesByCourt = new Map();
  placed.forEach(p => {
    if(!timesByCourt.has(p.c)) timesByCourt.set(p.c, new Set());
    timesByCourt.get(p.c).add(p.t);
  });
  const gaps = [];
  timesByCourt.forEach(set => {
    const sorted = Array.from(set).sort((a, b) => a - b);
    for(let i = 1; i < sorted.length; i++) gaps.push(sorted[i] - sorted[i - 1]);
  });
  gaps.sort((a, b) => a - b);
  const slot = gaps.length ? gaps[Math.floor((gaps.length - 1) / 2)] : 25;

  // The grid: rows are slots from the day's first match, columns are courts.
  const start = Math.min(...placed.map(p => p.t));
  placed.forEach(p => { p.row = Math.round((p.t - start) / slot); });
  const rows = Math.max(...placed.map(p => p.row)) + 1;
  const courts = timesByCourt.size;
  const occupied = new Set(placed.map(p => p.c + '|' + p.row));
  const unplayedRows = placed.filter(p => !p.played).map(p => p.row);
  const frontier = unplayedRows.length ? Math.min(...unplayedRows) : rows;
  let blanks = 0, blanksDone = 0;
  timesByCourt.forEach((_, c) => {
    for(let r = 0; r < rows; r++){
      if(occupied.has(c + '|' + r)) continue;
      blanks++;
      if(r < frontier) blanksDone++;
    }
  });
  const plannedEnd = start + rows * slot;

  const unitsLeft = (left - 0.5 * inPlay) + (blanks - blanksDone);
  const queueByCourt = new Map();
  real.forEach(m => {
    const c = courtNumberFrom(m.court);
    if(m.played || c === null || eventScheduleMinutes(m.time) === null) return;
    queueByCourt.set(c, (queueByCourt.get(c) || 0) + (m.liveCourt ? 0.5 : 1));
  });
  const queue = queueByCourt.size ? Math.max(...queueByCourt.values()) : 0;

  let eta = null;
  if(isEventDay && left > 0){
    eta = Math.max(nowMin, start) + Math.max(unitsLeft * slot / courts, queue * slot);
  }

  return { ...base, slot, courts, rows, blanks, blanksDone, plannedEnd, eta };
}

// Same rule as renderOrganizerStatus's isWarn — keep the two in step.
// STALE_WARNING_MS is declared further down the script; this only ever runs
// from a poll or render, after the whole script has loaded.
function facilityDataIsStale(name){
  const entry = LAST_SNAPSHOT ? (LAST_SNAPSHOT.facilities || []).find(f => f.name === name) : null;
  if(!entry) return false;
  const failed = (LAST_SNAPSHOT.failedFacilities || []).includes(name);
  const ageMs = Date.now() - new Date(entry.syncedAt || 0).getTime();
  return failed || ageMs > STALE_WARNING_MS;
}

// Everything both displays need for one facility, or null if it has no
// matches loaded. `stale` already applies spec §2.3's event-day/left conditions.
function facilityProgressFor(name){
  const f = MATCHES_BY_FACILITY.find(x => x.name === name);
  if(!f) return null;
  const nowMin = eventDayNowMinutes();
  const isEventDay = isEventDayActive(nowMin);
  const p = computeFacilityProgress(f.matches, isEventDay, nowMin);
  const entry = LAST_SNAPSHOT ? (LAST_SNAPSHOT.facilities || []).find(x => x.name === name) : null;
  return {
    p, isEventDay,
    stale: isEventDay && p.left > 0 && facilityDataIsStale(name),
    syncedAt: entry ? entry.syncedAt || null : null
  };
}

// The finish line (spec §2.1 line 4), shared by the card and Mission Control.
// scheduledEnd is the clock label for the card's extra line, or null.
function facilityFinishParts(p, isEventDay){
  const round5 = x => Math.round(x / 5) * 5;
  if(p.left === 0) return { finish: 'All matches done', scheduledEnd: null };
  if(p.plannedEnd === null) return { finish: 'No scheduled times yet', scheduledEnd: null };
  if(isEventDay && p.eta !== null){
    const diff = round5(p.eta - p.plannedEnd);
    const etaText = `Est. finish <b>${eventClock(round5(p.eta))}</b>`;
    const finish = diff > 0 ? `${etaText} &middot; <span class="fp-behind">${formatGap(diff)} behind</span>`
      : diff < 0 ? `${etaText} &middot; ${formatGap(-diff)} ahead`
      : `${etaText} &middot; on schedule`;
    return { finish, scheduledEnd: diff !== 0 ? eventClock(p.plannedEnd) : null };
  }
  return { finish: `Scheduled end <b>${eventClock(p.plannedEnd)}</b>`, scheduledEnd: null };
}

function facilityProgressCardHTML(name, info){
  const { p, isEventDay, stale, syncedAt } = info;
  const pct = p.total ? Math.round((p.done / p.total) * 100) : 0;
  const { finish, scheduledEnd } = facilityFinishParts(p, isEventDay);
  const subs = [];
  if(scheduledEnd) subs.push(`Scheduled end ${scheduledEnd}`);
  if(stale) subs.push(`<span class="fp-behind">Data from ${relativeTimeFromNow(syncedAt)} &mdash; estimate reads late until this sheet syncs again</span>`);
  if(p.left > 0 && p.unscheduledLeft > 0){
    subs.push(`${p.unscheduledLeft} match${p.unscheduledLeft === 1 ? '' : 'es'} without a court/time &mdash; counted as work left, not on the schedule`);
  }
  return `<div class="fp-card">
    <div class="fp-name">${escapeHtml(name)}</div>
    <div class="fp-bar-row">
      <div class="fp-bar"><span style="width:${pct}%"></span></div>
      <span class="fp-frac">${p.done} / ${p.total}</span>
    </div>
    <div class="fp-line">Done <b>${p.done}</b> &middot; Left <b>${p.left}</b> &middot; In play <b>${p.inPlay}</b></div>
    <div class="fp-line">${finish}</div>
    ${subs.map(s => `<div class="fp-sub">${s}</div>`).join('')}
  </div>`;
}

function renderFacilityProgress(){
  if(!facilityProgressEl) return;
  facilityProgressEl.innerHTML = MATCHES_BY_FACILITY
    .map(({ name }) => facilityProgressCardHTML(name, facilityProgressFor(name)))
    .join('');
}

// Mission Control: the second line inside a facility's sync-status row, or ''
// when there's nothing to show (spec §2.2).
function facilityStatusProgressHTML(name){
  const info = facilityProgressFor(name);
  if(!info || info.p.total === 0) return '';
  const { finish } = facilityFinishParts(info.p, info.isEventDay);
  const staleText = info.stale ? ' &middot; <span class="fp-behind">stale &mdash; may read late</span>' : '';
  return `<span class="org-status-progress">${info.p.done} / ${info.p.total} done &middot; ${finish}${staleText}</span>`;
}

```

Checks:
- `grep -c "function computeFacilityProgress" tools/control-center.html` → `1`.
- `grep -c "parseScheduleTimeToMinutes" tools/control-center.html` is exactly
  **one more** than before this step: `eventScheduleMinutes` is the only new
  caller.

### Step 4 — Render the cards on every poll

In `renderLiveMatches`, replace

```js
  restoreScrollPositions(boardEl, scrollPositions);
}
```

with

```js
  restoreScrollPositions(boardEl, scrollPositions);
  renderFacilityProgress();
}
```

That two-line anchor occurs exactly once in the file, directly after the
`boardEl.innerHTML = …` line.

No reset is needed on event or day switch: `selectEvent` already hides
`#liveResults`, and the next `loadLiveData` call re-renders everything.

### Step 5 — Mission Control line

In `renderOrganizerStatus`, replace

```js
    return `<div class="org-status-row ${isWarn ? 'org-status-warn' : 'org-status-ok'}">
      <span class="org-status-dot"></span>
      <span class="org-status-name">${escapeHtml(name)}</span>
      <span class="org-status-detail">${detail}</span>
      ${sheetLinkHTML(name)}
    </div>`;
```

with

```js
    const progressHTML = facilityStatusProgressHTML(name);
    return `<div class="org-status-row ${isWarn ? 'org-status-warn' : 'org-status-ok'}${progressHTML ? ' has-progress' : ''}">
      <span class="org-status-dot"></span>
      <span class="org-status-name">${escapeHtml(name)}</span>
      <span class="org-status-detail">${detail}</span>
      ${sheetLinkHTML(name)}
      ${progressHTML}
    </div>`;
```

This is the synced-row branch, after `const detail = …`. Leave the
never-synced branch above it (the one with `org-status-none`) alone.
`loadLiveData` already calls `renderOrganizerStatus()` on every poll, after
`MATCHES_BY_FACILITY` and `LAST_SNAPSHOT` are set, so no new call is needed.

Check: `grep -c "facilityStatusProgressHTML(name)" tools/control-center.html`
→ `2` (the definition and this call).

### Step 6 — Syntax and line-ending check

From the `sage-match-control.github.io` folder:

```bash
node -e "const s=require('fs').readFileSync('tools/control-center.html','utf8');const a=s.indexOf('<script>')+8,b=s.lastIndexOf('</script>');require('fs').writeFileSync(require('os').tmpdir()+'/cc.js',s.slice(a,b))" && node --check "$(node -p "require('os').tmpdir()")/cc.js"
```

→ no output.

```bash
file tools/control-center.html
```

→ `… with CRLF line terminators`. And `git diff --stat` shows only
`tools/control-center.html`, with roughly 200 added lines and 1 or 2 removed.
If it shows the whole file changed, the line endings were converted: undo
that before going on.

---

## 5. Verification

Serve the page as in §4 step 0 and run each check. Checks 1, 2 and 10 are
read off the page. The rest run in the browser's developer console on the
loaded page, because all the new functions are globals. Checks 3–7, 11 and 15
use this setup, which reproduces the §3.8 grid. Paste it into the console
first:

```js
const mk = (num, time, court, state) => ({ num, time, court,
  liveCourt: state === 'live' ? court : '', played: state === 'played',
  t1: 'A', t1p1: 'a', t1p2: 'b', t2: 'B', t2p1: 'c', t2p2: 'd' });
const base = [
  mk(1,'3:00 PM','Court 1','played'), mk(2,'3:00 PM','Court 2','played'), mk(3,'3:00 PM','Court 3','played'),
  mk(4,'3:25 PM','Court 1','played'), mk(5,'3:25 PM','Court 2','live'),
  mk(6,'3:50 PM','Court 1',''),
  mk(7,'4:15 PM','Court 1',''),       mk(8,'4:15 PM','Court 2',''),
];
const info = (p, stale = false, syncedAt = null) => ({ p, isEventDay: true, stale, syncedAt });
```

**Real data.** These event URLs load the two events used below:
- `?event=pnf-x-bup-dual-meet&day=pnf-x-bup-day1` (PNF × BUP, Sep 12 2026, finished)
- `?event=pickle-for-sight-2026&day=pickle-for-sight-day1` (Pickle for Sight, Sep 27 2026)

1. **Finished day.** PNF × BUP, Live Matches: one card,
   `Pampanga Pickleball Center`, full bar, `126 / 126`,
   `Done 126 · Left 0 · In play 0`, `All matches done`.
2. **Day not yet active, two facilities.** Pickle for Sight, Live Matches:

   | Card | Counts | Finish line |
   | --- | --- | --- |
   | PCPH Main | `0 / 116`, `Done 0 · Left 116 · In play 0` | `Scheduled end 9:55 PM` |
   | PCPH Annex | `0 / 125`, `Done 0 · Left 125 · In play 0` | `Scheduled end 9:05 PM` |

   If you run this on 27 Sep 2026, or before 6:00 AM on 28 Sep (PH), the
   day is active and the finish line reads `Est. finish …` instead. After
   that, the counts will have changed. In either case, see the note after
   check 17.
3. **§3.8 worked example.** `computeFacilityProgress(base, true, 955)` returns
   `total 8, done 4, left 4, inPlay 1, unscheduledLeft 0, slot 25, courts 3,
   rows 4, blanks 4, blanksDone 0, plannedEnd 1000, eta 1017.5`. And
   `facilityProgressCardHTML('Test', info(computeFacilityProgress(base, true, 955)))`
   contains `5:00 PM`, `20 min behind` and `Scheduled end 4:40 PM`.
4. **Before first serve.**
   `computeFacilityProgress(base.map(m => ({ ...m, played: false, liveCourt: '' })), true, 840)`
   → `eta 1000`, equal to `plannedEnd`. Its card (wrapped in `info(…)`)
   contains `on schedule` and does not contain `Scheduled end`.
5. **The last match of the day counts as a whole match.**
   `computeFacilityProgress(base.map(m => ({ ...m, played: m.num !== 8, liveCourt: '' })), true, 975)`
   → `blanksDone 3`, `eta 1000`. That's one full `slot` from the busiest-court
   floor, not `2 × 25 ÷ 3 ≈ 16.7`. Its card contains `on schedule`.
6. **Byes are ignored.**
   `JSON.stringify(computeFacilityProgress([...base, { ...mk(9,'4:40 PM','Court 1',''), t2: 'BYE' }], true, 955)) === JSON.stringify(computeFacilityProgress(base, true, 955))`
   → `true`.
7. **Unscheduled matches count.**
   `computeFacilityProgress([...base, mk(9,'','Court 1','')], true, 955)` →
   `total 9, left 5, unscheduledLeft 1, rows 4`, `eta ≈ 1025.83`. That's check
   3 plus `25 ÷ 3`.
8. **Polling.** Leave Live Matches open for 20 s. The cards stay in place
   (no flicker to empty), and the console shows no errors.
9. **Phone width.** In the browser's device toolbar at 375 px wide, the
   cards stack one per row and the page doesn't scroll sideways.
10. **Mission Control on days that aren't active.** Open Mission Control on
    each event. Each facility row keeps its existing first line and gains a
    second line:

    | Event | Second line |
    | --- | --- |
    | PNF × BUP | `126 / 126 done · All matches done` |
    | Pickle for Sight | `0 / 116 done · Scheduled end 9:55 PM` (Main), `0 / 125 done · Scheduled end 9:05 PM` (Annex) |

    Neither shows `stale`, even though the PNF × BUP sync is days old: the day
    isn't active and nothing is left.
11. **Stale line.**
    `facilityProgressCardHTML('Test', info(computeFacilityProgress(base, true, 955), true, new Date(Date.now() - 14 * 60000).toISOString()))`
    contains `Data from 14 min ago`, after `Scheduled end 4:40 PM`. The same
    call with `stale` `false` does not contain `Data from`.
12. **Mission Control at phone width.** At 375 px, the second line sits under
    the facility name and the page doesn't scroll sideways.
13. **Nothing else moved.** Match Finder, Standings and Awards look as they
    did before, and so does every other Mission Control control.
14. **Night-time schedule times.** `eventScheduleMinutes('12:15 AM')` → `1455`,
    `eventScheduleMinutes('5:59 AM')` → `1799`, `eventScheduleMinutes('6:00 AM')`
    → `360`, `eventScheduleMinutes('11:50 PM')` → `1430`,
    `eventScheduleMinutes('')` → `null`.
15. **Schedule crossing midnight.** The §3.8 grid moved 8 hours later:

    ```js
    const late = { '3:00 PM': '11:00 PM', '3:25 PM': '11:25 PM', '3:50 PM': '11:50 PM', '4:15 PM': '12:15 AM' };
    const r15 = computeFacilityProgress(base.map(m => ({ ...m, time: late[m.time] })), true, 1435);
    ```

    → `rows 4, blanks 4, plannedEnd 1480, eta 1497.5`. And
    `facilityProgressCardHTML('Test', info(r15))` contains
    `1:00 AM (+1 day)`, `20 min behind` and `Scheduled end 12:40 AM (+1 day)`.
    If `rows` is more than 4, the 12:15 AM row was read as the start of the
    day, which means `eventScheduleMinutes` isn't being used somewhere.
16. **Labels and event-day window.**
    - `eventClock(1000)` → `'4:40 PM'`, `eventClock(1500)` →
      `'1:00 AM (+1 day)'`, `eventClock(2940)` → `'1:00 AM (+2 days)'`.
    - `formatGap(20)` → `'20 min'`, `formatGap(60)` → `'1 h'`, `formatGap(75)`
      → `'1 h 15 min'`, `formatGap(1020)` → `'17 h'`.
    - `isEventDayActive(0)`, `isEventDayActive(1470)` and
      `isEventDayActive(1799)` → `true`.
    - `isEventDayActive(-1)`, `isEventDayActive(1800)` and
      `isEventDayActive(null)` → `false`.
17. **Still active after midnight.** With Pickle for Sight loaded, pretend
    its day was yesterday:
    `DAYS[currentDayIndex].date = new Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Manila' }).format(new Date(Date.now() - 864e5)); renderLiveMatches(); renderOrganizerStatus();`
    - `eventDayNowMinutes()` is 1440 plus the current PH time in minutes.
    - Before 6:00 AM PH, `isEventDayActive(eventDayNowMinutes())` → `true`,
      and the cards and Mission Control lines show `Est. finish … (+1 day)`.
      The `behind` gap is shown in hours, e.g. `17 h behind`, because nothing
      "yesterday" was played.
    - From 6:00 AM PH, it's `false` and they show `Scheduled end`.

    Reload the page afterwards to undo the change.

If checks 1, 2 or 10 show different counts, the event's published data has
changed since this spec was written. Recompute the expected values from the
snapshot and report it. **Don't change the §3 formulas to make a check pass.**
Checks 3–7 and 11–17 use fixed inputs, so they must match exactly.

---

## 6. Documentation (same change)

Write in the present tense: describe what the console does, not what changed.

**6.1** `sage-docs/docs/features/control-center.md`. Under `## Live Matches`,
after the paragraph that ends `…wall-mounted screen at the venue would
show.`, add:

```markdown
Above the courts, a card per venue shows how many matches are done, how many
are left, and — on the day itself — an estimated finish time and how far
ahead of or behind schedule that venue is. The estimate is the operators'
own formula: matches left plus idle court slots left, times the scheduled
match length, divided by the number of courts. It never counts less than one
match length per match still queued on the busiest court, and it keeps
working past midnight until 6:00 AM. If a venue's sheet stops syncing, the
card says so, because no new scores means the estimate drifts later on its own.
```

Under `## Mission Control`, at the end of the **Facility sync status**
bullet (after `…not on Control Center.`), add:

```markdown
  Under each venue, a second line repeats its matches done and estimated
  finish from Live Matches, and adds "stale — may read late" when that
  venue's data is old enough to make the estimate unreliable.
```

**6.2** `sage-docs/docs/technical/control-center.md`. Insert a new section
immediately before the line `## Installability`:

```markdown
## Facility progress

`computeFacilityProgress(matches, isEventDay, nowMin)` turns one facility's
matches into done/left counts, a schedule grid (slot length, rows, courts,
blank cells) and an estimated finish. `renderFacilityProgress` draws a card
per facility on Live Matches (called from `renderLiveMatches`), and
`facilityStatusProgressHTML` adds a line to each Mission Control sync row
(called from `renderOrganizerStatus`). Both go through `facilityProgressFor`
and `facilityFinishParts`, so the two can't disagree.

Times are *event-day minutes*: minutes since midnight at the start of the
day's `date`, which keep counting past 1440 after midnight. Schedule times
before `DAY_ROLLOVER_MIN` (6:00 AM) are read as the night after the event
day. The stale test in `facilityDataIsStale` is the same rule as the sync
row's `isWarn`; change both or neither. Design and worked examples:
[facility progress spec](../specs/implemented/facility-progress-spec.md).
```

The link above points at `implemented/` because §7 moves the spec there in
the same change.

---

## 7. Finish: move the spec, commit

**7.1 Move this spec to `implemented/`**, following `sage-docs/docs/specs/README.md`:

1. `git mv docs/specs/not-started/facility-progress-spec.md docs/specs/implemented/facility-progress-spec.md`
2. In the spec, change `**Status: not started.**` to `**Status: built.**`.
3. `docs/specs/not-started/README.md`: delete the table row that starts
   `| [Facility progress](facility-progress-spec.md)`.
4. `docs/specs/implemented/README.md`: add this row at the end of the table:
   `| [Facility progress](facility-progress-spec.md) | Live Matches' per-facility progress cards and Mission Control's progress line |`
5. `docs/specs/README.md`: delete the `**[Facility progress](not-started/facility-progress-spec.md)**`
   bullet under `## Not started`. Under `## Implemented` → `### Control Center`,
   add after the Schedule screen bullet:

   ```markdown
   - **[Facility progress](implemented/facility-progress-spec.md)** — matches
     done, matches left and estimated finish per facility, on Live Matches and
     in Mission Control.
   ```
6. `mkdocs.yml`: delete the line
   `          - Facility progress: specs/not-started/facility-progress-spec.md`
   and add
   `          - Facility progress: specs/implemented/facility-progress-spec.md`
   after the `- PNF x BUP dual meet: …` line in the `Implemented:` list.
7. Check: `grep -rn "not-started/facility-progress-spec.md" docs mkdocs.yml` → no output.

**7.2 Commit**, one commit per repo:

- `sage-match-control.github.io`: `tools/control-center.html` only. Message:
  `Control Center: per-facility progress and estimated finish`.
- `sage-docs`: the two doc pages plus the spec move. Message:
  `Document facility progress; mark its spec implemented`.

**Do not push.** Pushing `sage-match-control.github.io` deploys the live
site. The user pushes both repos when ready.

---

## 8. Stop and ask — don't guess

Stop and report back instead of improvising if:

- Any anchor in §4 matches zero times or more than once.
- Any identifier in §0.3 is missing, renamed, or shaped differently.
- `node --check` fails and the cause isn't a typo you introduced.
- `git diff` shows changes outside the lines §4 adds or edits.
- Any check in 3–7 or 11–17 gives a different value. Those inputs are fixed,
  so a different result means the code differs from §4.
- The page can't load event data locally (network or CORS errors in the
  console).
- You find yourself wanting to change a number (6:00 AM, 25, 0.5, the 5-minute
  rounding, the stale rule) or a colour. §9 explains each one.

---

## 9. Decisions

- **Built on the operators' formula.** Operators already estimate this way,
  so the console's number and a hand calculation agree. It measures work
  left rather than clock slippage, so a court reassignment or an
  out-of-order match changes which cells are done, not the total.
- **Blanks are counted over the full grid, trailing ones included.** With
  every cell counted, `(matches + blanks) ÷ courts` equals the number of
  rows, so on an on-time day the formula reproduces the schedule exactly
  (check 4). Dropping a court's trailing blanks would spread the remaining
  matches across courts that have nothing left to play.
- **Blanks are done by frontier, not by clock.** Tying a blank to the clock
  would mark it done whenever the day ran late, making the estimate more
  optimistic exactly when things were slipping.
- **Fixed match length, not a measured pace.** Measuring pace since first
  serve mixes up a late start with slow play. A 10-minute late start would be
  read as every match running slow and projected onto the rest of the day:
  about 90 minutes too late after three rounds. A fixed `slot` counted from
  **now** treats a late start as a shift of the whole day. A rolling-window
  pace would avoid that, but it needs a per-browser log that starts over
  whenever the console is opened fresh. It was considered and not chosen.
- **Busiest-court floor, not a row floor.** A floor of "rows left × slot" is
  just the schedule again, and it would override the formula whenever blanks
  exist. The busiest single queue only takes over at the end of the day,
  which is where the pooled term is wrong.
- **Rounded to 5 minutes.** The input is hand-typed scores arriving through a
  debounced sync. Minute-level precision would be false precision.
- **Can show "ahead".** When courts get ahead of their rows, the finished
  work shows up as fewer cells left.
- **In-play depends on the sheet.** In-play is counted from the sheet's
  live-court column. If operators don't mark live courts, `In play` reads 0
  and those matches count as fully left. That makes the estimate at most half
  a cell per court late.
- **Both tabs, at different sizes.** Live Matches gets the full card: it's the
  tab operators leave open. Mission Control gets one line inside the existing
  sync rows. Next to the sync age, it separates "behind schedule" from "not
  syncing". A full card there would push the go-live, sign-in and resync
  controls down a narrow column.
- **Stale reuses Mission Control's own rule**, so the progress line and the
  amber sync row can't disagree. `facilityDataIsStale` repeats `isWarn`'s
  condition instead of editing `renderOrganizerStatus` to call it, keeping the
  edits to existing code to §4 steps 4–5. The comment on it marks the pair.
- **Stale only while the event day is active and matches are left.** Old data
  is normal for a finished facility or a day that isn't being played.
- **Warning colour is the sync row's orange.** "Behind" and "stale" use
  `#B3541A` from `.org-status-warn`. `--amber` was checked in the browser: it
  is the theme's green (`#5C8F1F`), despite its name.
- **Past midnight: count from the event's date.** Plain minutes-since-midnight
  breaks two ways after 12:00 AM. When play runs late, the date rolls over and
  the estimate and stale flag disappear at the moment they're most wanted.
  And a scheduled 12:15 AM match parses as minute 15, becomes the day's start,
  and throws every other match 20–50 rows out. Event-day minutes fix both.
- **The 6:00 AM cut-off.** A night-time time like "12:15 AM" is ambiguous. No
  event has ever started play before 6:00 AM, and several run into the
  evening, so anything earlier is read as the night after. It's one constant,
  `DAY_ROLLOVER_MIN`, if that ever changes.
- **Day offset in labels, counted from the event's date.** "12:40 AM (+1 day)"
  reads the same at 11 PM and at 12:30 AM. "Tomorrow" would flip meaning at
  midnight.
- **Hours for long gaps.** `17 h behind` is readable; `1020 min behind` is not.
  Gaps under an hour stay in minutes, which is the usual case.
- **Go-live left alone.** The public site's auto go-live
  (`earliestScheduleMinutes`) has the same 12:15 AM problem, because it reads
  `parseScheduleTimeToMinutes` directly. It's out of scope. This spec doesn't
  change `parseScheduleTimeToMinutes`, so go-live behaves exactly as before.
