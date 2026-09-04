# Spec — Schedule screen (venue wall display)

Add a **standalone schedule board** to an event site: courts as columns, time
slots as rows, one face-off card per match, colour-coded by category, with live
matches highlighted and results filling in as they are played.

Built and reviewed as a mockup against real synced data from
`pnf-x-bup-dual-meet` (126 matches / 15 slots / 9 courts). This spec records the
decisions that mockup settled and what still has to be built to ship it.

**Scope note.** This is the *first* instantiation. Once it is working, the
intent is to fold the reusable parts back into
`_templates/dual-meet-template/`. That backport is explicitly **out of scope
here** (§10) and will be a separate pass.

---

## 1. Why a standalone page, not a tab

| | |
|---|---|
| Path | `events/<event-key>/schedule.html` |
| Public URL | `/events/<event-key>/schedule` |
| Discoverability | **Unlisted publicly** — linked only from Mission Control (§4.1) |

GitHub Pages resolves extensionless HTML (verified: `/tools/scoresheet-generator`
returns `200` on the live site), so naming the file `schedule.html` yields the
clean `/schedule` URL with no directory and no config. This matches how
`tools/` already reads.

It is a separate page rather than a fourth tab in `index.html` because:

1. **Different audience, opposite needs.** `index.html` is a phone lookup tool
   for a player finding their own match — it wants hero, search, day picker,
   tab bar, footer. The schedule is a passive wall display where every one of
   those costs a match row. The masthead here is deliberately 48px.
2. **The themes collide.** This page uses the SAGE tools theme (§3); the event
   pages use the dark event theme. Both define `--paper`, `--ink`, `.cell`,
   `.legend`, `.dot`. Separate documents avoid a whole class of cross-contamination
   bugs.
3. **Weight.** `index.html` is already 114 KB; the schedule is ~18 KB
   standalone. Folding it in taxes every phone visitor for a view they will
   never open.

### 1.1 Duplication cost — smaller here than usual

The repo's known weak spot is hand-synced duplication (`index.html` and
`match-control.html` share a ~106-line configuration block). This page does
**not** meaningfully add to it: it derives club from the team-code prefix,
carries its own `CAT_META`, and needs none of `DIVISIONS`, `EVENTS`, `CLUBS`,
`STAGE_META`, search, standings, or bracket logic.

It needs exactly four things, ~5 lines total:

```js
const EVENT_KEY     = 'pnf-x-bup-dual-meet';
const GHPAGES_OWNER = 'sage-match-control';
const GHPAGES_REPO  = 'event-data';
const DAY_KEY       = 'pnf-x-bup-day1';
const COURTS        = 9;   // or derive from CourtAssignment — see §5.2
```

Add these to CLAUDE.md's "things that must be kept in sync by hand" list.

---

## 2. Layout

```
┌──────────────────────────────────────────────────────────────┐
│ [logo] S.A.G.E. · MATCH CONTROL EXPERTS    COURTS [All][1..9] │  48px masthead
│        PNF × BUP Dual Meet   meta          legend    ● LIVE   │
├──────┬───────────┬───────────┬───────────┬───────────────────┤
│      │  COURT 1  │  COURT 2  │  COURT 3  │       ...         │  sticky header row
├──────┼───────────┼───────────┼───────────┼───────────────────┤
│ 3:00 │ [match]   │ [match]   │ [match]   │                   │  sticky time column
│ 3:25 │ [match]   │ [match]   │ [match]   │                   │
└──────┴───────────┴───────────┴───────────┴───────────────────┘
```

- **Columns = courts**, from `CourtAssignment`. **Rows = time slots**, from
  `Schedule`, in first-appearance order.
- Column headers and the time column are both `position:sticky`, so they stay
  visible when the board scrolls.
- A court with no match in a slot renders a dimmed dashed **"Open"** placeholder,
  never a collapsed/missing cell — the grid must not reflow as matches come and go.

### 2.1 Match cell — face-off

```
┌───────────────────────────────────────┐
│ #47  HI MD  RR              ● COURT 2 │   number · category · stage · live
│ [PNF] Ericson G      11 : 8   Allan [BUP] │
│       Mike T                  Jun     │
└───────────────────────────────────────┘
```

- Symmetric: team 1 left, score centred, team 2 right, mirrored.
- Player names **stacked**, two lines, truncating with ellipsis.
- The club tag sits **inline beside** the names, not on its own line — this is
  what keeps the cell at 60px and lets 15 slots fit 1080px.
- Club is read **per side** from the team-code prefix, never assumed by
  position. Semifinals are intra-club (`PNF_LIWD_SF_1` vs `PNF_LIWD_SF_2`), so
  a cell can legitimately be PNF-vs-PNF. Hardcoding PNF-left/BUP-right is wrong
  and was a real bug caught in review.

### 2.1.1 Stage pill

A second pill beside the category chip says where in the ladder the match sits.

| Suffix in team code | Pill | Style |
|---|---|---|
| `1`, `2`, `3`… (pool number) | `RR` | outline, muted |
| `R16` | `R16` | solid navy |
| `QF` | `QF` | solid navy |
| `SF` | `SF` | solid navy |
| `B` | `BRONZE` | solid navy |
| `F` | `FINAL` | solid navy |

Pool play is ~89% of cells (112 of 126 here), so `RR` is a quiet outline —
shouting it 112 times would be noise. Playoff rounds invert to solid navy so a
Bronze or Final is findable at a glance across a wall.

- Classification regexes are **lifted verbatim from `roundKeyword()` in
  `index.html`**, so this page classifies a code exactly as the rest of the site
  does. Order matters — `SF` is tested before `F`.
- An unrecognised suffix falls back to `RR`, matching `standingsStageKey()`.
- Read from team 1 only; both sides of a match are always the same stage.
- `QF`/`SF`/`R16` are supported even though the current event emits only
  `1-4`/`B`/`F` — the bracket generator produces them, and this should not need
  editing the first time a larger event runs.

### 2.2 Collapsible header

A chevron at the right of the masthead collapses it. State is carried in the
URL as `?compact=1`, composing with `?courts=` (§4) so a screen can be
bookmarked into both at once.

| Element | Expanded | Collapsed |
|---|---|---|
| Logo + `S.A.G.E. · MATCH CONTROL EXPERTS` | shown | **shown** |
| Event name | shown | **shown** |
| Category legend + live key | shown | **shown** |
| Sub line | `Sep 12 · 126 matches · 15 slots · courts 1-5` | `Sep 12 · 126 matches` |
| Court filter pills | shown | hidden |
| `PDF` export button | shown | hidden |
| Masthead height | 48px | 36px |

The legend stays because a viewer needs the colour key to read the board at
all. What hides is operator chrome — the court filter (its state is already in
the URL) and the export button (Ctrl+P still prints, so nothing is lost). The
sub keeps date and match count and drops the slot count and court scope.

> The expand chevron stays visible in both states, otherwise a collapsed screen
> can never be reopened.

> **Match count is event-wide, identical on every screen.** It is the *scope*
> ("courts 1-5") that says what a given screen shows. A filtered count would
> make the same label mean different things on the two halves of a split-screen
> setup.

> **The page must be a flex column, not `height:calc(100vh - 48px)`.** The
> masthead changes height when collapsed, and any hardcoded offset silently
> desyncs and leaves a dead strip. With `body{display:flex;flex-direction:column}`
> and `.grid-wrap{flex:1 1 auto;min-height:0}` the board always takes exactly
> the remaining space — verified summing to the viewport in both states.

### 2.3 States

| State | Condition | Treatment |
|---|---|---|
| Upcoming | no scores, not live | full opacity, `VS` in centre |
| Live | `court` column non-empty | green ring + glow, `● COURT n` pill |
| Completed | both scores present | dimmed to `opacity:.84`, shows `11 : 8` |

`.done` is `.84` and not lower: at `.72` the darkest category (HI WD) fell to
**4.48:1** for body text, under the 4.5:1 AA floor. `.84` restores it to 5.63:1.

---

## 3. Theme — SAGE tools

Taken verbatim from `tools/scoresheet-generator.html` and
`tools/tournament-calculator.html`, which share one `:root`:

```css
--navy:#14263C;  --navy-deep:#0B1826;
--green:#7CB92C; --green-dark:#5C8F1F;
--paper:#F6F7F2; --paper-dim:#ECEEE6;  --line:#D9DED2;
--ink:#14263C;   --ink-soft:#5B6B74;   --white:#FFFFFF;
--radius:14px;
```

Type: **Archivo Black** (display/numbers), **Barlow Condensed** (uppercase
tracked labels), **Inter** (body). Logo at `/assets/logo.png`.

The tools render a tall navy `.bezel` panel on a `.stand`. That is a good
desktop flourish but costs ~120px of a wall display, so the schedule keeps the
navy bezel, green eyebrow and Archivo Black title but compresses them into one
sticky 48px bar. Same identity, a twelfth of the height.

### 3.1 Category colours — organiser-owned, do not invent

Read off the colour-coded `SCHEDULE` tab of the source spreadsheet so the wall
display and the printed schedule agree.

| Category | Colour | |
|---|---|---|
| `LIWD` | `#C27BA0` | light berry |
| `HIWD` | `#741B47` | dark berry |
| `LIMD` | `#6D9EEB` | light blue |
| `HIMD` | `#1155CC` | mid blue |
| `AMD`  | `#1C4587` | dark navy |
| `LIXD` | `#F6B26B` | light orange |
| `HIXD` | `#B45F06` | dark orange |

The scheme is **hue = event** (Women's berry, Men's blue, Mixed orange),
**lightness = division** (LI light → HI dark → A darkest).

> **These are not exportable.** Cell fills are formatting, so they appear in
> neither the CSV nor the gviz export — both tabs export byte-identical text.
> Google Sheets also paints the grid to a single `<canvas>`, so the DOM exposes
> only Google's own toolbar chrome. They were recovered by sampling canvas
> pixels and matching colour run-lengths against the known category-by-court
> grid, cross-checked across five rows. **If the organiser recolours the sheet,
> these must be re-read by hand — nothing detects drift.**

Applied as a **40% tint** of the hue over white, with the solid hue as a 5px
left rule. 40% is the measured ceiling: it holds ~6.9:1 for navy body text on
the darkest category, where 48% drops to 5.7 and 55% to 4.8.

Category chip text colour is **derived per category** from luminance
(`>140` → navy, else white), not hardcoded — the palette spans `#F6B26B` to
`#741B47` and a fixed colour is unreadable at one end.

---

## 4. Court filter

| | |
|---|---|
| UI | `All` + one pill per court, in the masthead |
| State | **the URL** — `?courts=1-5`, `?courts=1,2,7-9` |
| Default | no param → all courts |

The URL is the point. A two-screen venue setup bookmarks one URL per screen:

```
/events/pnf-x-bup-dual-meet/schedule?courts=1-5     ← screen A
/events/pnf-x-bup-dual-meet/schedule?courts=6-9     ← screen B
```

Each screen boots straight into its range after a refresh or power cycle with
nobody touching the machine. Clicking pills rewrites the URL live
(`history.replaceState`) so an operator can dial in a layout and bookmark
whatever they land on. Selections collapse to the shortest form (`1,2,3,4` →
`1-4`).

Rules:
- Deselecting the last remaining court is refused — never render a blank board.
- The **legend** follows the filter (it keys what is on screen); the **match
  count** does not (§2.2).
- Verified lossless: `?courts=1-5` (71 cells) + `?courts=6-9` (55) = 126.

### 4.1 Launching it — Mission Control

The board is unlisted from the public pages, so Mission Control in
`match-control.html` is its only entry point: operators get a launcher,
spectators never see a link.

A `Schedule board` heading at the foot of the organizer panel with a single
`Open schedule` button → `schedule.html`.

- `target="_blank" rel="noopener"` — the operator is mid-session, and a wall
  display is a second screen, not a navigation away.
- **One button, no preset court ranges.** Splitting is done on the board itself
  (§4), which writes the range into the URL for bookmarking. Duplicating those
  presets here would be a second place to keep the court maths correct for no
  gain.

---

### 4.2 Export — print / PDF

A `PDF` button in the masthead builds print-only pages and calls
`window.print()`. Browsers' "Save as PDF" covers the PDF ask; the same output
prints on paper.

**Why print and not a canvas render.** `bracket-generator.html` exports by
hand-drawing to a `<canvas>` and calling `toBlob()`. That works there because a
bracket card is simple and fits one image. Here it would mean re-implementing
every face-off cell in Canvas 2D *and* chunking pages by hand — a second layout
implementation that drifts from the CSS the moment either changes. Printing
gets page breaking, paper size, margins and DPI from the browser for free, and
`gridHTML(courts, rows)` is shared with the live board, so **there is exactly
one implementation of a cell**.

### 4.2.1 Two-dimensional pagination

Both axes can overflow a sheet independently, so both are chunked and the pages
are the cross product. Courts are the outer loop, so one court group's whole day
lands on consecutive sheets.

| Budget | Value | Derivation |
|---|---|---|
| `PRINT_COURTS_PER_PAGE` | 4 | A4 landscape ≈ 1054px usable width |
| `PRINT_SLOTS_PER_PAGE` | 15 | ≈726px usable height, less ~70px of headers, ÷ ~42px per row |

Chunking is **balanced**, not fill-then-orphan: 9 courts at 4/page gives 3+3+3,
never 4+4+1.

Verified across shapes — every one fits the sheet height and is lossless (no
match dropped or duplicated):

| Shape | Sheets | Notes |
|---|---|---|
| 9 courts, 15 slots | 3 | this event |
| 4 courts, 15 slots | 1 | small venue |
| 3 courts, 15 slots | 1 | small venue |
| 9 courts, 29 slots | 6 | 12-hour day — splits both ways |
| 3 courts, 29 slots | 2 | |
| 1 court, 40 slots | 3 | degenerate case still correct |

Page header carries `Courts 1–3`, the time window (**only when the day is
actually split**, else it is noise), and `page N of M`. Single-court pages read
`Court 1`, not `Courts 1–1`.

### 4.2.2 Print-specific rules

- **`print-color-adjust:exact`** is required, or browsers drop the category
  tints and the whole colour-coding — the point of the board — prints blank.
- **`.done` is forced back to `opacity:1`.** On screen dimming separates
  finished matches; on paper it just prints muddy. Ink is cheap, contrast is not.
- **Name font is 6.5pt, not 7pt.** At 7pt a 4-court sheet (~244px columns)
  clipped a handful of names, and paper has no hover tooltip to recover them.
  6.5pt clears every shape above with no extra sheets; 6pt buys nothing more.
- `beforeprint` rebuilds the pages, so Ctrl+P (not just the button) always
  reflects the current court filter.
- The export honours the court filter: filtering to `?courts=1-5` prints only
  those courts.

> **Image export was considered and not built.** A PNG of a 3250px board is
> unwieldy, and producing one would need either the canvas-duplication above or
> a CDN dependency (`html2canvas`), which this repo's self-contained pages avoid.
> Print-to-PDF covers the same need and paginates properly.

---

## 5. Data

Source: the synced snapshot the rest of the site already reads —
`https://<owner>.github.io/<repo>/<event-key>/data/<day>.json`, field
`facilities[].matchesCsv`.

Columns used: `matchNumber`, `Schedule`, `CourtAssignment`, `court`,
`teamCode1/2`, `team1Player1/2`, `team2Player1/2`, `team1Score`, `team2Score`.

### 5.1 Parsing rules

| Input | Handling |
|---|---|
| Row where `teamCode1` is `-` | Spacer row — skip |
| Player cell `#REF!` or empty | Treat as no name; fall back to the team code, muted italic |
| Score cell empty / non-numeric | `null` — match is not complete |
| Both scores present | Completed; higher score wins |
| `court` non-empty | Live on that court |

### 5.2 Court assignment

`CourtAssignment` holds `"Court 1"…"Court 9"`. Parse the trailing integer.

Defensive placement, in order:
1. Valid court, lane free → place there.
2. Blank / unparseable (`"SCORE"` has been observed) / duplicate → first free lane.
3. Slot already full → **surface, never drop.** Flag `+n` on the time cell with
   the match numbers in a `title`.

This matters: an earlier snapshot had a slot with 10 matches for 9 courts,
`Court 1` assigned twice, and 5 cells containing `SCORE`. Current data is clean
(126/126 placed directly), but the sheet is hand-edited and will regress.

> Do **not** fall back to row-order-implies-court. An earlier build did, and it
> happened to be correct, which is exactly what makes it dangerous — it is an
> undocumented contract that breaks silently on a row reorder.

### 5.3 Live polling

Poll on the same `POLL_INTERVAL_MS = 10000` the event pages use, with the same
cache-busting `?t=${Date.now()}`.

- **Cache-busting is required, not optional.** The snapshot is regenerated
  behind a fixed filename; without it a stale copy renders silently and looks
  correct. This was hit during review.
- Re-render must **preserve the current court filter** and not reset scroll.
- A failed poll must leave the last good board on screen — a wall display
  showing stale data beats one showing an error.

---

## 6. Responsive behaviour

Court columns are `1fr` — they share whatever width the window gives, and get
**wider** when the filter narrows the set.

| | 1280px | 1600px | 1920px | 3250px |
|---|---|---|---|---|
| 9 courts | scrolls | scrolls | 199px ✓ | 347px ✓ |
| 5 courts | 234px ✓ | 298px ✓ | 362px ✓ | — |
| 4 courts | 294px ✓ | 374px ✓ | 454px ✓ | — |

Floor is `minmax(190px, 1fr)`: below that the face-off stops being readable, so
the wrapper scrolls rather than crushing cells further. The court filter is the
intended answer on smaller screens — fewer courts means wider cells.

> Do not reintroduce `min-width:max-content` on the grid. It pins the grid to
> its content width and forces a horizontal scrollbar even when there is room
> to fit; it was what prevented the board from ever shrinking to the window.

---

## 7. Build steps

1. Copy the reviewed mockup to `events/<event-key>/schedule.html`.
2. Replace the static `fetch('schedule-data-beta.json')` with the live snapshot
   URL + the CSV parsing of §5.
3. Add the `POLL_INTERVAL_MS` refresh loop (§5.3), preserving filter and scroll.
4. Wire the four config constants (§1.1), and read the date from the snapshot's
   `label` field rather than hardcoding it (the mockup hardcodes `Sep 12`).
5. Add the Mission Control launchers (§4.1) to `match-control.html`. Leave
   `index.html` untouched — no public link.
6. Delete `schedule-beta.html`, `schedule-darkmode-beta.html` and
   `schedule-data-beta.json` once shipped (all `*beta*`, already gitignored).

---

## 8. Acceptance

- [ ] `/events/pnf-x-bup-dual-meet/schedule` loads; `.html` form also works.
- [ ] 9 court columns, 15 time slots, 126 matches, none dropped.
- [ ] Semifinal cells show PNF-vs-PNF / BUP-vs-BUP, not PNF-vs-BUP.
- [ ] Every cell carries a stage pill; pool matches read `RR` as an outline and
      Bronze/Final invert to solid navy. For this event: 112 `RR`, 7 `BRONZE`,
      7 `FINAL`.
- [ ] The extra pill does not overflow the cell header at any supported width,
      and does not push the live `COURT n` pill out.
- [ ] `?courts=1-5` and `?courts=6-9` split 71/55, sum to 126.
- [ ] Clicking pills updates the URL; reloading restores the same courts.
- [ ] Chevron collapses the header to logo + eyebrow + event name + legend +
      `Sep 12 · 126 matches`; the court filter and `PDF` button hide, the
      chevron itself stays.
- [ ] `?courts=6-9&compact=1` boots collapsed **and** on courts 6-9; expanding
      drops `compact=1` and leaves `courts=6-9` intact.
- [ ] Masthead + board sum to the viewport height in both states (no dead strip).
- [ ] Match count reads the same event-wide total on every screen regardless of
      filter; only the scope text changes.
- [ ] Mission Control shows an `Open schedule` button that opens in a new tab.
- [ ] `PDF` button paginates: 9 courts / 15 slots → 3 sheets, none taller than
      the sheet, no match dropped or duplicated.
- [ ] A long day (≈29 slots) splits down as well as across; a 3-court venue
      does not split across at all.
- [ ] Printed sheets keep the category tints (`print-color-adjust`), and no
      player name is clipped at any supported court count.
- [ ] Ctrl+P (without using the button) produces the same paginated output.
- [ ] Live matches show the green ring and `COURT n`; completed show scores.
- [ ] A new score in the sheet appears within ~10s without a manual refresh,
      and does not reset the court filter or scroll position.
- [ ] Board fills the window at 1920 and 3250 wide with no horizontal scroll.
- [ ] Not linked from any other page.

---

## 9. Known data issues (not this page's job to fix)

- **`#REF!` player names.** 8 cells in the current snapshot. The page degrades
  to the team code; the fix belongs in the spreadsheet.
- **Division-code mismatch.** Earlier snapshots used `IWD`/`IMD`/`IXD` while
  the site's `DIVISIONS` config is `LI`/`HI`/`A`. Current data uses the correct
  codes. Codes that do not match a configured division silently fall back to
  "Other" in the event pages — worth watching.
- **Playoff slots unseeded.** Bronze/Final rows carry codes but no players yet;
  they render as muted codes. Expected, not a bug.

---

## 10. Out of scope

- Backporting to `_templates/dual-meet-template/` — deliberately deferred; will
  be a separate pass once this page is settled.
- Any change to `index.html` or `bracket-generator.html`. The **only** edit
  outside the new page is the Mission Control launcher block in
  `match-control.html` (§4.1).
- Standings, brackets, search, or day switching — this page shows one day's
  schedule and nothing else.
- Writing anything back to the spreadsheet. The mock-score generator used
  during review (`--mock-scores=N`) is a review aid only and ships with nothing.
