# Spec — Dual Meet Schedule Generator (Phase 2)

Phase 2 of the Dual Meet Sheet Generator: the `SCHEDULE` tab. Split out of
`dual-meet-sheet-generator-spec.md` §10.1, which now stubs to this file.

Phase 1 built every category tab plus `Variables`, `Title` and
`Reference for Players`, and is documented in that spec — read its §1.1
(the Named Function contract), §2 (the plan CSV) and §3 (validation)
first. This spec does not restate them; it inherits them and adds the
inputs and rules only the schedule needs.

Same delivery story as Phase 1: bound Apps Script in
`sage-tools-api/scripts/sheet-generator.gs`, no new infra, no Cloud Run
deploy, no `package.json` bump.

| | |
| --- | --- |
| Scope | The `SCHEDULE` tab only |
| Not in scope | `CSV`, `STANDINGSCSV`, `Court Control`, `Timeline` — see `dual-meet-readouts-generator-spec.md` |
| Depends on | Phase 1's pair codes — the seam is §5 |

---

## 0. Orientation — read this first if you have no context

**What this is.** SAGE runs pickleball tournaments. A "dual meet" is two
clubs playing each other across several categories (age/skill/gender
divisions). Each event runs off one Google Sheets workbook. The **Dual Meet
Sheet Generator** is bound Apps Script inside that workbook that builds the
whole event's tabs from a CSV the Tournament Time Calculator exports.

**Where everything lives.**

| Thing | Path |
| --- | --- |
| The generator | `sage-tools-api/scripts/sheet-generator.gs` |
| Phase 1 spec — read it first | `sage-docs/specs/dual-meet-sheet-generator-spec.md` |
| The colour palette's other consumer | `sage-docs/specs/schedule-screen-spec.md` §3.1 |
| The live sync that reads `SCHEDULE` | `sage-tools-api/scripts/sheets-sync.gs` |
| The calculator that writes the plan CSV | `sage-match-control.github.io/tools/tournament-calculator.html` |

`sage-tools-api/scripts/*.gs` is **not** part of the Cloud Run service. It
ships by pasting into a spreadsheet's own bound script project. Changing it
is not a deploy, and does **not** bump `package.json`.

**How it ships.** The operator copies the *SAGE Dual Meet Master* workbook,
opens the copy, and runs `SAGE -> Generate event tabs` from the menu,
pasting the calculator's CSV into a sidebar. Everything runs inside that
one spreadsheet.

**Why it must be bound script.** The category tabs depend on a library of
Named Functions (`GETTOTALWINS`, `SORTBYWINS`, `GETSCOREAGAINSTPAIR`, … —
workbook-level `LAMBDA` definitions, not container-bound Apps Script custom
functions) that exist only inside the master workbook and are in no repo. A
blank spreadsheet produces `#NAME?` everywhere. That rules out a Cloud Run /
Sheets API approach entirely. See Phase 1 spec §1 and §1.1.

**What Phase 1 already built** (all of it working): every category tab, plus
`Variables`, `Title` and `Reference for Players`.

**What this spec adds:** the `SCHEDULE` tab — the one the operator types
scores into, and the one every Named Function reads match results from.
Without it a generated workbook cannot run an event.

**Vocabulary.**

| Term | Means |
| --- | --- |
| category | A division, keyed `LIWD`, `AMD`, … — one tab each |
| pair | Two players who play together; the atomic competitor |
| `n` | Pairs per club per bracket — `teams_a / brackets` |
| bracket | A group within a category; 1 or 2 of them |
| slot | One start time across all courts — two sheet rows |
| team code | `<CLUB>_<KEY>_<i>`, e.g. `PNF_LIWD_3` — the join key for everything |

**There is no test framework.** No linter, no build step, no TypeScript.
`buildMatchList` is pure precisely so it can be checked by hand against the
reference data in §10 before any Sheets code is written.

**Section-reference convention.** A bare `§n` means this document. A
reference into another spec always names it — "Phase 1 spec §11.1",
"`schedule-screen-spec.md` §3.1". Both documents have a §11, and both carry
so the attribution matters.

---

## 1. Sources

Everything below was read directly out of two workbooks, because none of it
is in source control:

| Workbook | File ID | What it gave |
| --- | --- | --- |
| Reference (PNF x BUP, played) | `1DH7BNVdvSkyaywavLzUlfCP4NEhT0SjnzFr_e0LVfFg` | A complete 126-match `SCHEDULE`, 9 courts |
| Master template | `1cBvQ5r6Nm29BJ7Hpsk_gonX31hWJHOrCK1WPLgQaqtg` | The stripped `SCHEDULE` prototype the generator copies |

**Read coverage.** The reference `SCHEDULE` was read through row 32; the
Drive text export truncates there, so the last two slots (rows 33-35) were
recovered from that workbook's own `CSV` tab, which was read complete — all
126 match rows. The template `SCHEDULE` was read complete.

---

## 2. Layout

### 2.1 The grid

Rows 1-4 are blank. Row 5 is the header band. Data starts at row 6.

Columns are a **repeating 8-column court block** starting at column D:

```
 A   B      C   | D  E  F  G  H  I  J | K   | L ... R | S   | ...
     Time       |      Court 1        | sep |  Court 2| sep |
```

- `A` — blank
- `B` — time, merged vertically across the slot's two rows
- `C` — blank spacer
- `D:J` — one court's block, 7 columns
- `K` — blank separator, making the period 8

Court `c` (1-based) starts at column `4 + 8(c - 1)`. The **last court has no
trailing separator**, so a `courts`-wide sheet is exactly:

```
grid width = 8 * courts + 2
```

9 courts → 74 columns (`BV`); 12 courts → 98 columns (`CT`). That is the
number the `CSV` tab's column-stacking formulas have to track, and the
reason they cannot be hand-listed.

### 2.2 Inside one court block

| Col | Offset | Content | Written by | Merge |
| --- | --- | --- | --- | --- |
| D | +0 | team 1 player names — one per row | formula | none |
| E | +1 | `teamCode1` | generator | merged, both rows |
| F | +2 | match number | generator | first row only |
| G | +3 | `teamCode2` | generator | merged, both rows |
| H | +4 | team 2 player names — one per row | formula | none |
| I | +5 | `team1Score` | operator, live | merged, both rows |
| J | +6 | `team2Score` | operator, live | merged, both rows |
| K | +7 | separator | — | — |

Row 5 carries `Court <c>` at the `F` offset, and a single `SCORE` header
merged horizontally across `I5:J5`. `B5` reads `Time`.

Three merges per court block, and they are different shapes — worth keeping
straight:

| Merge | Shape |
| --- | --- |
| `I5:J5` | horizontal, once in the header band — the `SCORE` label |
| `E`, `G`, `I`, `J` | vertical, per slot — `E6:E7`, `I6:I7`, `J6:J7`, … |
| `B` | vertical, per slot — the time |

So `I` and `J` are merged *horizontally* on row 5 and *vertically* from row
6 down. The header band is tiled once per court; the slot merges are tiled
per court per slot.

**The generator writes three columns per match: `E`, `F`, `G`.** Nothing
else. `D` and `H` are name lookups off `E` and `G` — put a team code in, the
names appear — and they come across for free when the prototype block is
tiled, since their references are relative to their own block. `I` and `J`
stay empty for the operator to fill during play.

That is the whole write surface, and it is what keeps `buildScheduleTab`
small: tile the prototype, then batch-write three literal columns per court
plus the time in `B`.

### 2.3 One time slot is two rows

Rows 6-7 are slot 1, rows 8-9 slot 2, and so on. The row pair exists because
each side fields **two players**, one name per row in `D` and `H`.
Everything else in the block is merged across both rows.

```
slot s occupies rows 6 + 2(s - 1) and 7 + 2(s - 1)
```

### 2.4 Empty cells

An unused court in a slot carries `-` in `E`, `F` and `G`, with `I`/`J`
blank. That is the same `-` sentinel the schedule board and event pages
already skip on (`schedule-screen-spec.md` §5.1: a row where `teamCode1` is
`-` is a spacer row), so unused courts must be **written**, not left blank.

### 2.5 The template prototype

The master's `SCHEDULE` is stripped to exactly **one court block, one time
slot** — header rows 1-5, a single `Court 1` block in `D:J`, and one slot at
rows 6-7 carrying `-` in `E`/`F`/`G` and no time in `B`.

This is the same contract `_CATEGORY_TEMPLATE` has in Phase 1: the prototype
carries live formatting and formulas, and the generator tiles it. Phase 2
grows it in both directions — right by court, down by slot — rather than
rebuilding formatting by hand.

---

## 3. Inputs

Phase 1 reads `club_a`, `club_b`, `teams_a`, `teams_b`, `brackets`, `key`
and `value`. Phase 2 additionally reads the timing settings, which
`parsePlanCsv` already stores wholesale and simply never used:

| Setting | Used for |
| --- | --- |
| `courts` | Court block count — the sheet's width |
| `start` | Slot 1's time |
| `duration_min` | Slot pitch |
| `buffer_min` | Slot pitch |

```
slot time = start + (slot - 1) * (duration_min + buffer_min)
```

The reference confirms this: `duration_min: 25`, `buffer_min: 0`, start
`3:00 PM`, giving 3:00, 3:25, 3:50 … 8:50 PM across 15 slots.

No parser change is needed.

---

## 4. Match list

> Derived from the reference workbook's 126 matches. The rotation below
> reproduces all 112 group matches exactly. Confirm before building.

### 4.1 Group stage — cross product as `n` rounds

A category with `n` pairs per club per bracket plays the full `n x n` cross
product, decomposed into `n` **rounds** of `n` matches. Within a round every
pair plays exactly once, so a round can never self-conflict.

Round `r` (1-based) pairs club A's pair `i` against club B's pair:

```
j = ((i - 1) - (r - 1)) mod n + 1
```

For `n = 4` that is:

| Round | Pairings |
| --- | --- |
| 1 | 1v1 2v2 3v3 4v4 |
| 2 | 1v4 2v1 3v2 4v3 |
| 3 | 1v3 2v4 3v1 4v2 |
| 4 | 1v2 2v3 3v4 4v1 |

### 4.2 Emission order

Rounds are the outer loop, categories the inner:

```
for r in 1..n:
    for cat in CSV order:
        emit that category's round r
then all bronze matches
then all finals
```

With 7 categories at `n = 4`: matches 1-28 are round 1, 29-56 round 2,
57-84 round 3, 85-112 round 4, 113-119 bronze, 120-126 finals. Verified
against every match number in the reference.

Interleaving by round rather than by category is what keeps a category's own
matches ~28 apart, which is what makes §4.4's guarantee hold.

**Category order is CSV order**, unchanged from how the operator arranged
them in the calculator. Round 1 of the reference runs `LIWD LIMD LIXD HIWD
HIMD HIXD AMD` — which is both the plan CSV's own order and the organiser's
stated intent: ascending level (LI, HI, A), and `WD`, `MD`, `XD` within each
level.

Those two coincide here, and the generator follows the CSV rather than
re-deriving the level ordering from the keys. §6.4 does parse a key into
level and type, but ordering cannot lean on that the way colour can: colour
degrades per category (an unparseable key simply gets no fill), whereas a
sort needs a total order over *every* category at once, and the calculator
permits `OPEN` and operator-defined `CUSTOM` codes that yield no level.
Following the CSV also keeps the order operator-controllable — reorder the
categories in the calculator and the schedule follows.

Bronze and finals are emitted in the same CSV order. The reference orders
them differently (`LIXD LIWD LIMD AMD HIMD HIWD HIXD`), by hand and to no
rule the plan carries — see §6.

### 4.3 Placement and numbering

Matches fill `(slot, court)` in reading order — slot first, then court — and
match numbers are assigned in that same order, densely, from 1.

Two rules on top of that:

- **Bronze and finals each start a fresh slot.** Group matches pack densely,
  but the first bronze match begins a new slot even when the previous slot
  has free courts, and likewise for the first final.
- **A partial slot is centred, not left-packed.** A slot holding `m` matches
  on `courts` courts starts at court `floor((courts - m) / 2) + 1`.

Both rules fall out of one shape: split the match list into three phases —
group, bronze, finals — and chunk each into runs of `courts`. A phase's last
chunk is the only partial one, and centring is one term on the court index:

```
court = i + floor((courts - chunkSize) / 2) + 1
```

Match numbers are then just the position in the flattened placement order,
so centring does not complicate numbering.

Both hold across all three partial slots in the reference: 4 matches on 9
courts → courts 3-6; 7 matches → courts 2-8, twice.

### 4.4 Why no pair is ever double-booked

A slot holds `courts` consecutive match numbers. Two matches involving the
same pair are either in the same round (impossible — §4.1) or in different
rounds, which §4.2 puts `categories * n` apart. So the guarantee is:

```
courts <= categories * n
```

At 9 courts, 7 categories, `n = 4` there are 28 matches per round — huge
headroom. A very small event on many courts could violate it, so this is a
**validation rule**, not an assumption.

---

## 5. The seam with Phase 1

`SCHEDULE` writes team codes; the category tabs read scores back by those
codes through `GETSCOREAGAINSTPAIR` and friends. The codes must match Phase
1's exactly, and Phase 1 already emits all of them:

| Match | Code | Emitted by |
| --- | --- | --- |
| Group | `<CLUB>_<KEY>_<i>` | `pairCode_` |
| Bronze | `<CLUB>_<KEY>_B` | `writePlayoffBlockCrossClub_` |
| Final | `<CLUB>_<KEY>_F` | `writePlayoffBlockCrossClub_` |
| Semi | `<CLUB>_<KEY>_SF_1` / `_SF_2` | `writePlayoffBlockIntraClub_` |

Confirmed against the reference: bronze rows carry `PNF_LIWD_B` /
`BUP_LIWD_B`, finals `PNF_LIWD_F` / `BUP_LIWD_F`.

Playoff rows therefore carry **literal synthetic codes**, not formulas
pointing into the category tab. Who those codes resolve to is the category
tab's business — its feeder formulas. The schedule only needs a stable
handle to record a score against.

**Two brackets adds a semi round.** The reference is single-bracket
throughout, so a `semis` category's two intra-club semifinals have no
precedent in it. They slot between the last group round and bronze, and
`dual-meet-sheet-generator-spec.md` §13.6 already flags the two-bracket
`semis` path as never yet generated — Phase 2 is where it gets exercised.

---

## 6. Category colours

The seven hexes in `schedule-screen-spec.md` §3.1 are not arbitrary: they
are three columns of **Google Sheets' own standard palette**, picked by
event type, at a shade picked by level. That makes them derivable rather
than a lookup table that has to grow by hand for every new category.

### 6.1 Hue by event type

| Suffix | Hue |
| --- | --- |
| `WD` | magenta |
| `MD` | cornflower blue |
| `XD` | orange |

### 6.2 Shade by level

Each hue is a column of seven shades, light to dark. Levels occupy fixed
rungs on that column — **the rung is a property of the level, not of how
many levels the event happens to have**:

| Level | Rung |
| --- | --- |
| `N` novice | lighter 2 |
| `LI` low intermediate | lighter 1 |
| `I` intermediate | base |
| `HI` high intermediate | darker 1 |
| `A` advanced | darker 2 |

Fixing the ladder globally, rather than spreading whatever levels appear
across the column, is deliberate: adding an advanced category to an event
must not recolour the categories already in it.

The reference bears this out. `WD` and `XD` have only two levels there but
still use *lighter 1* and *darker 1* — the same rungs `MD`'s first two
levels use — rather than stretching to fill the column.

### 6.3 The table

| Rung | Level | Magenta `WD` | Cornflower `MD` | Orange `XD` |
| --- | --- | --- | --- | --- |
| lighter 2 | `N` | `#D5A6BD` | `#A4C2F4` | `#F9CB9C` |
| lighter 1 | `LI` | **`#C27BA0`** | **`#6D9EEB`** | **`#F6B26B`** |
| base | `I` | `#A64D79` | `#3C78D8` | `#E69138` |
| darker 1 | `HI` | **`#741B47`** | **`#1155CC`** | **`#B45F06`** |
| darker 2 | `A` | `#4C1130` | **`#1C4587`** | `#783F04` |

**Bold** = one of the seven confirmed against `schedule-screen-spec.md`
§3.1, which was itself recovered by sampling the rendered sheet. The other
eight are read off the same palette columns and **should be eyeballed once
in the Sheets colour picker** before they ship — nothing in the reference
exercises them.

### 6.4 Deriving it from a key

```
type  = last two characters of the key, if in {WD, MD, XD}
level = everything before that
```

`LIWD` → level `LI`, type `WD` → magenta lighter 1 → `#C27BA0`.

A key that does not end in `WD`/`MD`/`XD`, or whose level is not in §6.2,
gets **no fill** — and a trace line saying so. That is the honest outcome:
the calculator's own presets include `OPEN` and an operator-defined
`CUSTOM` code, and validation only requires `[A-Z0-9]{2,8}`, so unmatched
keys are expected rather than exceptional. Inventing a colour for one would
put a wrong category colour on the wall display, which is worse than a
plain row.

Note the calculator's preset levels are `N`/`I`/`A` while the reference
event used `LI`/`HI`/`A`. Both are covered by §6.2, and an event mixing
them still orders correctly.

### 6.5 Applied as conditional formatting, not as fills

The colour lands on **columns `D` and `H`** of each court block — the two
player-name columns (§2.2) — framing each match and leaving the codes and
scores in between on white. The web schedule board mirrors this as "the
solid hue as a 5px left rule" (`schedule-screen-spec.md` §3).

> `J` would be the obvious other edge of the `D:J` span, and is wrong: `J`
> is `team2Score` (§2.2), so colouring it contradicts this section's own
> "leaving the codes and scores in between on white".

It is applied as **conditional format rules, one per category** — not by
setting cell backgrounds. Three reasons, in order of weight:

1. **The colour follows the data.** Move a match to another court or slot
   by editing its team codes and the colour goes with it. Baked-in fills
   would strand the old cell coloured and leave the new one white, and
   rescheduling mid-event is normal.
2. One `setConditionalFormatRules` call per tab instead of thousands of
   per-cell `setBackground` calls — see the Phase 1 spec §11.1 on batching
   and the 6-minute execution limit.
3. The rules survive the operator inserting a slot by hand.

### 6.6 The rule

One rule per category, applied to the `D` and `H` columns of every court
block (`2 * courts` ranges per rule, via `setRanges`) over rows 6 to the
last slot row. The separator column `K` is excluded by leaving it out of
the ranges rather than by guarding in the formula.

```
=LET(code, INDEX($A$6:$<LC>$<LR>,
                 2 * INT((ROW() - 6) / 2) + 1,
                 8 * INT((COLUMN() - 4) / 8) + 5),
     IFERROR(INDEX(SPLIT(code, "_"), 2) = "<KEY>", FALSE))
```

`<LC>`/`<LR>` are the generated sheet's last column and row; `<KEY>` is the
category key. Two pieces of arithmetic carry the whole rule:

- `2 * INT((ROW() - 6) / 2) + 1` — **normalises to the slot's top row.**
  `E` is merged across the slot's two rows, and a merged range holds its
  value only in the top-left cell, so a rule evaluated on row 7 that read
  `E7` directly would see a blank. Both rows must look at row 6.
- `8 * INT((COLUMN() - 4) / 8) + 5` — **finds the block's own code column.**
  Column `D` (4) and column `H` (8) both resolve to `E` (5); court 2's `D`
  (12) and `H` (16) both resolve to `M` (13). The arithmetic maps every
  column of a block — offsets 0 to 7, so `D` through the separator `K` —
  onto that block's own `E`, which is why changing *which* columns the rule
  covers never requires touching the formula.

The test is `INDEX(SPLIT(code, "_"), 2)` — the second underscore-delimited
token of `<CLUB>_<KEY>_<rest>` — compared exactly. That is deliberately not
a substring match: `AMD` and a hypothetical `MD` would collide under one.
`IFERROR` catches the `-` placeholder in unused courts, which splits to a
single token.

`INDEX` over a bounded range is used rather than `OFFSET` because `OFFSET`
is volatile and this evaluates on every cell of every rule.

### 6.7 Where the hexes live

Phase 1 already writes `Variables`: `C` category key, `D` category name,
`G2` courts, `H2` start, `I2` duration. Column `E` is free.

The generator writes each category's resolved hex to `E` on that category's
row, **and sets that cell's own background to it** so the operator sees a
swatch beside the key rather than a string.

`Variables` is a *record*, not an input — the generator computes the colour,
builds the rules, and writes `E` so the value is inspectable and documented.
Making `E` an override that a "recolour" menu item reads back is a sensible
follow-up, but it is not Phase 2.

---

## 7. Decisions

Every question this spec opened is now closed. Recorded here so a later
reader knows they were considered, not overlooked.

- **`D` and `H` are formulas.** Confirmed by the organiser: the only things
  filled in to place a match are the team codes in `E` and `G`; the names
  follow. Match numbers in `F` are filled left to right, top to bottom.
  See §2.2 — this is why the generator's write surface is three columns.
- **Playoff ordering follows CSV order** (§4.2). The reference's
  `LIXD LIWD LIMD AMD HIMD HIWD HIXD` was arranged by hand; the organiser
  has confirmed the generator should not try to reproduce it.
- **`I5:J5` is one merged `SCORE` header** (§2.2), not a vertical merge into
  row 4. `I` and `J` merge vertically per slot from row 6 down.
- **Colour goes on `D` and `H`, via conditional formatting** (§6.5-§6.6) —
  literally those two columns, not the `D:J` block. (Recorded as `D` and
  `J`, which is a score column, not an edge.)
- **Partial slots are centred** (§4.3). Confirmed as a deliberate choice: the
  phase chunking §4.3 already requires makes centring a single extra term, so
  it costs nothing over left-packing and matches the organiser's sheet.

---

## 8. Implementation notes

Phase 1's own §11.1 covers the general Apps Script mechanics — batching,
`copyTo`, the 6-minute execution limit, `flush()`. Read it. This section
covers only what is new or different in Phase 2.

### 8.1 Where the code goes

All of it in `sage-tools-api/scripts/sheet-generator.gs`, the same bound
script Phase 1 lives in. Two new functions plus a hook:

```js
// PURE — no Sheets API, sits with parsePlanCsv / computeLayout / colLetter_
function buildMatchList(parsed)          // -> [{ matchNumber, slot, court, timeMinutes,
                                         //       codeA, codeB, catKey, phase }]
function categoryColor_(key)             // -> '#RRGGBB' or null

// SHEETS GLUE
function buildScheduleTab_(ss, matches, settings, categories)
```

`buildMatchList` takes the output of the existing `parsePlanCsv(text)` and
returns plain data. It must not touch `SpreadsheetApp` — that is what makes
it verifiable against the reference's 126 rows without a workbook, and it
follows the same rule Phase 1's header comment sets out under "PURE VS.
SHEETS-API CODE".

Hook it into `generateEventTabs` **last** — after the category tabs and
after `Variables`, `Title` and `Reference for Players`.

Nothing orders it earlier. `categoryColor_` is a pure function of the
category key (§6.4), so the colour column §6.7 adds to `Variables` resolves
whether or not `SCHEDULE` has been built. What orders it *later* is failure
blast radius: `SCHEDULE` is the most expensive and failure-prone step in the
run (hundreds of merge-carrying `copyTo` calls, §8.5), and in the
before-`Variables` position a single timeout takes three cheap,
deterministic tabs down with it. Last means a `SCHEDULE` failure costs only
`SCHEDULE` — which is what `dual-meet-sheet-generator-spec.md` §11.3 wants
out of a partial failure.

### 8.2 Existing helpers to reuse — do not rewrite these

| Helper | Does |
| --- | --- |
| `parsePlanCsv(text)` | Already returns `{ settings, categories }` with every setting |
| `validatePlan(parsed)` | Returns an array of all failures; extend it, don't wrap it |
| `colLetter_(idx)` | 1-based column index → `A1` letters |
| `ensureRowCount_(sheet, n)` / `ensureColumnCount_(sheet, n)` | Grow the grid before writing past it |
| `clubCode_(settings, 'A'\|'B')` | `settings.club_a` / `club_b` with a fallback |
| `pairCode_(settings, cat, club, i)` | `<CLUB>_<KEY>_<i>` — the group code, §5 |
| `trace_(msg)` | `[sage]`-prefixed execution log |
| `logStep_(msg)` | Progress line the sidebar polls |

`COL` (near the top of the file) maps `A: 1, B: 2, D: 4 … K: 11`. Phase 2's
court arithmetic is relative to `COL.D`.

### 8.3 SCHEDULE is rebuilt in place — and that needs a guard

Every other tab in Phase 1 is *created* by copying `_CATEGORY_TEMPLATE`.
`SCHEDULE` is not: the master's own `SCHEDULE` **is** the prototype (§2.5),
and the generator grows it where it sits. There is no `_SCHEDULE_TEMPLATE`
and none should be added — the tab's GID is referenced by
`scripts/sheets-sync.gs` (`WATCHED_SHEET_GIDS`), so replacing the tab
would break the live sync trigger for that workbook.

That makes the operation **non-idempotent**: running it twice would tile an
already-tiled sheet.

So, matching Phase 1's collision policy (`dual-meet-sheet-generator-spec.md`
§9.2 — abort, never half-overwrite), validation must **abort if `SCHEDULE`
is not in its pristine one-court, one-slot state**:

```js
if (schedule.getLastRow() > 7 || schedule.getLastColumn() > 11) abort
```

Message the operator toward the fix: start again from a fresh copy of the
master. Do not attempt to reset the tab by deleting rows and columns — a
half-reset schedule mid-event is exactly the outcome §9.2 exists to prevent.

### 8.4 Growing the sheet

Slot count and width both come from the match list:

```
slots = max(m.slot for m in matches)
width = 8 * courts + 2
rows  = 5 + 2 * slots
```

Call `ensureColumnCount_(schedule, width)` and
`ensureRowCount_(schedule, rows)` **before** any tiling. The prototype may
have been trimmed rather than cleared, exactly as Phase 1 found for
`_CATEGORY_TEMPLATE`'s `AA:AV`.

### 8.5 Tiling — use `copyTo`, not `insertRows`

Phase 1's §11.1 warns that formatting propagates on insert but **merges do
not**. Phase 2 is merge-heavy (§2.2), so inserting rows would produce a
schedule with no per-slot merges at all.

`Range.copyTo(destination)` copies values, formats, formulas *and merges*,
and adjusts relative references — which is what carries the `D`/`H` name
lookups into every tiled block without rewriting them. Use it for both axes.

Order matters: **grow down first, then across.** Each horizontal copy then
carries every slot with it.

```
1. vertical:   for s in 2..gridSlots
                 copy B6:J7  ->  B<6+2(s-1)>:J<7+2(s-1)>
               one copy per slot, spacer column C included, so the time
               column's own merge comes along

2. horizontal: for c in 2..courts
                 src  = D5:J<lastRow>            (header band + all slots)
                 dest = <blockStart(c)>5

3. relabel:    row 5, F-offset of each block -> "Court <c>"
```

**Every court copies `D:J` — 7 columns, no exceptions, and no separator
column is ever written.** Blocks sit 8 apart while only 7 columns are
written, so the one-column gap falls out for free as untouched space, and
"the last court has no trailing separator" (§2.1) stops being a case to
handle at all. The master's prototype has no `K` to copy from in any case —
it is one court wide, and one court is always the last court (§2.5). Only
the *width* of those gap columns needs setting, per §8.10.

`gridSlots` is `slots + 1`: the grid runs one row-pair past the last real
slot to carry the event's end time (readouts spec §3.2).

Loop the vertical copy per slot rather than relying on `copyTo` tiling a
larger destination. At 30 slots and 12 courts that is ~42 calls, nowhere
near the execution limit, and it removes a behaviour that would have to be
verified.

### 8.6 Writing values into merged cells

`E` and `G` are merged across each slot's two rows; `F` is not. A write of
the whole `(2 * slots) x 3` rectangle per court is one `setValues` call:

```
row 6 + 2(s-1):  [ codeA, matchNumber, codeB ]
row 7 + 2(s-1):  [ '',    '',          ''    ]
```

Writing into a range that contains merges lands on each merge's anchor.
**Verify this against a real workbook early** — it decides whether the write
loop is `courts` calls or `courts * slots` calls. If the runtime rejects it,
fall back to one `setValues` per slot per court; at 12 x 30 that is 360
calls, still comfortable.

Unused courts take `-` in `E`, `F` and `G` (§2.4) — written, not skipped.

### 8.7 Times are numbers, not strings

`settings.start` arrives as `"15:00"` (24-hour, from the calculator's own
export). The sheet renders `3:00 PM` because the prototype's `B` column
carries a time number format, which `copyTo` propagates.

Writing the string `"3:00 PM"` would work visually and break any formula
that treats the column as a time. The value has to be a **fraction of a
day**, the same convention `writeVariablesTab_` already uses for `I2`.

Write it as the reference workbook's own **chained formulas**, not as a
literal this script computes:

```
B6  =Variables!H2
B8  =B6+Variables!$I$2
B10 =B8+Variables!$I$2   … one per slot, through the end-time row
```

The obvious alternative — computing each slot in JavaScript as
`(startMin + (slot - 1) * pitch) / 1440` and writing literals — renders
identically and is wrong. Phase 3's `Timeline` builds its own time headers
by exactly this chain (`=B$2+Variables!$I$2`, readouts spec §5.2) and
`COUNTPAIRAT` matches a slot's time **exactly**. The same nominal time
reached two different ways lands on floats that differ in the last few bits,
so the two silently stop comparing equal. Chaining off the same two cells
makes both sides the same computation, so they cannot drift.

Full account of how that surfaced: readouts spec §14.4.

### 8.8 Conditional formatting

```js
var rules = schedule.getConditionalFormatRules();   // keep what's there
categories.forEach(function (cat) {
  var color = categoryColor_(cat.key);
  if (!color) return;                                // §6.4 — no fill, trace it
  var ranges = [];
  for (var c = 1; c <= courts; c++) {
    var start = COL.D + 8 * (c - 1);
    ranges.push(schedule.getRange(6, start,     2 * slots, 1));  // D — team 1 names
    ranges.push(schedule.getRange(6, start + 4, 2 * slots, 1));  // H — team 2 names
  }
  rules.push(SpreadsheetApp.newConditionalFormatRule()
    .whenFormulaSatisfied(colorFormulaFor_(cat.key, lastCol, lastRow))
    .setBackground(color)
    .setRanges(ranges)
    .build());
});
schedule.setConditionalFormatRules(rules);
```

**`setConditionalFormatRules` replaces every rule on the sheet.** Read the
existing ones first and append, as above — the prototype may carry its own
(Phase 1 has `applyScoreConditionalFormat_` doing the equivalent for
category tabs). Overwriting them silently is the easy mistake here.

The formula is §6.6. Build it per category with the generated sheet's real
last column and row substituted in.

### 8.9 Tracing and failure

Follow Phase 1 exactly: `trace_` every computed value a wrong schedule would
otherwise hide — slot count, grid width, each court's block start column,
and the full match list as `#<n> <slot>/<court> <codeA> v <codeB>`. A
misplaced match is invisible in a finished tab unless you already know which
cell was expected.

On failure mid-build, do not roll back and do not continue (Phase 1 §11.3).
`generateEventTabs`'s existing `try/catch` reports which tabs were written;
`SCHEDULE` needs to appear in that list.

The completion toast currently says only that player names need pasting.
Once Phase 2 lands it should stop implying the workbook is otherwise ready —
`CSV`, `STANDINGSCSV` and `Court Control` are still Phase 3 and still carry
the previous event's shape.

### 8.10 `copyTo` does not carry column widths or row heights

Found by generating one. `Range.copyTo` brings values, formats, formulas and
merges (§8.5) — but column width and row height are **sheet** properties,
not cell formats, so they do not come along. Left alone, every tiled court
and every tiled slot sits at Sheets' defaults (100px / 21px) while court 1
and slot 1 keep the master's sizing. It reads as "court 1 looks right,
courts 2-9 look wrong".

So after both tiling axes, copy the prototype's own dimensions:

- court 1's seven column widths (`D:J`) onto every other court
- slot 1's two row heights (rows 6, 7) onto every other slot's row pair

**Not `autoResizeColumns`.** At generation time `D` and `H` are empty —
their name lookups resolve only once the operator pastes rosters (§2.2) — so
auto-fitting would collapse exactly the two columns that need the most room.

The inter-court separator columns have no prototype to copy: the master is
one court wide, and one court is always the *last* court, which carries no
trailing separator (§2.5). They take **column `C`'s** width — the template's
own blank spacer, and the only stated-intent gap width in the sheet.

### 8.11 Flush between stages, or the whole tab times out

A full-size event is merge-heavy in a way the call count hides:
`courts * slots * 4` vertical merges (`E`, `G`, `I`, `J` per slot per
court) — 540 for the reference event, up to 1440 at 12 courts and 30 slots.
Every one of them is created by a `copyTo` in §8.5, and the value writes in
§8.6 then land *on those merges*.

Queued into a single un-flushed batch, that fails with
**`Service timed out: Spreadsheets`** — not from too many calls, but from
one batch too large to resolve. `SpreadsheetApp.flush()` between stages
(after growing, after the vertical axis, per court on the horizontal axis,
after the value writes) keeps any single batch small enough to settle.

This is the concrete form of `dual-meet-sheet-generator-spec.md` §11.1's
`flush()` note, and it is what makes §8.6's batched-write question moot:
flushing the merges before writing into them is what lets the write stay
`courts` calls rather than `courts * slots`.

---

## 9. Build order

1. `buildMatchList(parsed)` — pure, verified against the reference's 126 rows
   before any Sheets code exists
2. `categoryColor_(key)` — pure, §6
3. Extend `validatePlan` with `courts <= categories * n` (§4.4) and the
   pristine-`SCHEDULE` guard (§8.3)
4. `buildScheduleTab_` — grow, tile, write, then conditional formatting
5. Add the colour column to `writeVariablesTab_` (§6.7)
6. Hook into `generateEventTabs`; correct the completion toast

---

## 10. Worked example

Input: the reference plan CSV — `format: dual`, `club_a: PNF`, `club_b: BUP`,
`courts: 9`, `start: 15:00`, `duration_min: 25`, `buffer_min: 0`, and seven
categories `LIWD LIMD LIXD HIWD HIMD HIXD AMD`, each `brackets: 1`,
`teams_a: teams_b: 4` — so `n = 4`.

Derived: 7 x 16 group + 7 bronze + 7 finals = **126 matches**, 9 courts,
**15 slots**. Grid is `8 * 9 + 2 = 74` columns (`A:BV`) and `5 + 2 * 15 = 35`
rows.

Court block start columns: D(4), L(12), T(20), AB(28), AJ(36), AR(44),
AZ(52), BH(60), BP(68).

Slot 1 — rows 6-7, `B6:B7` = `15:00` (0.625 as a day fraction):

| Match | Court | Code cell | Value | Match-no cell | Code-2 cell | Value |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | `E6` | `PNF_LIWD_1` | `F6` | `G6` | `BUP_LIWD_1` |
| 2 | 2 | `M6` | `PNF_LIWD_2` | `N6` | `O6` | `BUP_LIWD_2` |
| 3 | 3 | `U6` | `PNF_LIWD_3` | `V6` | `W6` | `BUP_LIWD_3` |
| 4 | 4 | `AC6` | `PNF_LIWD_4` | `AD6` | `AE6` | `BUP_LIWD_4` |
| 5 | 5 | `AK6` | `PNF_LIMD_1` | `AL6` | `AM6` | `BUP_LIMD_1` |
| 6 | 6 | `AS6` | `PNF_LIMD_2` | `AT6` | `AU6` | `BUP_LIMD_2` |
| 7 | 7 | `BA6` | `PNF_LIMD_3` | `BB6` | `BC6` | `BUP_LIMD_3` |
| 8 | 8 | `BI6` | `PNF_LIMD_4` | `BJ6` | `BK6` | `BUP_LIMD_4` |
| 9 | 9 | `BQ6` | `PNF_LIXD_1` | `BR6` | `BS6` | `BUP_LIXD_1` |

Round 1 continues into slot 2 (rows 8-9, `15:25`) with match 10 =
`PNF_LIXD_2` v `BUP_LIXD_2` on court 1 — categories are not aligned to slot
boundaries, and must not be forced to be.

Phase boundaries for this event:

| Matches | Phase | Slots |
| --- | --- | --- |
| 1-28 | group round 1 | 1-4 (court 1 of slot 4) |
| 29-56 | group round 2 | 4-7 |
| 57-84 | group round 3 | 7-10 |
| 85-112 | group round 4 | 10-13 |
| 113-119 | bronze | 14 — fresh slot, 7 matches centred on courts 2-8 |
| 120-126 | finals | 15 — fresh slot, courts 2-8 |

Slot 13 holds only matches 109-112, centred on **courts 3-6**; courts 1, 2,
7, 8 and 9 carry `-`.

Every number above is verified against the reference workbook.

---

## 11. Acceptance

- The reference plan CSV reproduces **group matches 1-112** with identical
  numbers, times, courts and team codes
- It produces the same 14 playoff matches (7 bronze, 7 finals) in the same
  two slots, but in CSV order rather than the reference's hand order — a
  deliberate divergence from the reference workbook (§4.2).
- Grid width is exactly `8 * courts + 2`; no stray separator past the last court
- Every unused court carries `-` in `E`/`F`/`G`
- No pair appears twice in one slot, in any category
- A two-bracket `semis` category generates its intra-club semifinals
- The `CSV` tab's stacking formulas resolve against the generated width
  without editing (Appendix A)
- Running the generator twice on the same workbook is **refused**, not
  half-applied (§8.3)
- `Variables!E` carries one hex per category, with the cell painted to match
- Every tiled court and slot matches the prototype's own column widths and
  row heights — court 9 looks like court 1, slot 15 like slot 1 (§8.10)
- A full-size event completes without `Service timed out: Spreadsheets`
  (§8.11)

---

## Appendix A — reading `SCHEDULE` back by column, dynamically

Not Phase 2, but derived alongside it and needed by Phase 3.

The `CSV` and `STANDINGSCSV` tabs flatten `SCHEDULE` by stacking one column
from every court block. Written out by hand that is a fixed list, and it
breaks the moment the court count changes:

```
={SCHEDULE!$E$6:$E ; SCHEDULE!$M$6:$M ; SCHEDULE!$U$6:$U ; … ; SCHEDULE!$CO$6:$CO}
```

Since blocks are period 8 from column D (§2.1), the list is derivable from
the sheet's own width:

```
=LET(
  first, 5,
  step,  8,
  n,     INT((COLUMNS(SCHEDULE!$1:$1) - first) / step) + 1,
  ref,   LAMBDA(c, LET(L, SUBSTITUTE(ADDRESS(1, c, 4), "1", ""),
                       INDIRECT("SCHEDULE!" & L & "6:" & L))),
  REDUCE(ref(first), SEQUENCE(n - 1, 1, first + step, step),
         LAMBDA(acc, c, VSTACK(acc, ref(c))))
)
```

`first` is the column offset being stacked: `5` `teamCode1`, `6` match
number, `7` `teamCode2`, `9` `team1Score`, `10` `team2Score`.

Better as a Named Function (**Data → Named functions**) defined once in the
master, so every copy inherits it and each call site is one line —
`STACKBLOCKS(5, 8, 6)`. This is not merely a suggested improvement: it
exists in the master today, and `dual-meet-readouts-generator-spec.md` §2.1
builds the entire Phase 3 Named Function library on top of it —
`GETPLAYERSCOLUMN1`, `GETMATCHNUMBERS`, `GETPLAYERSCOLUMN2`,
`GETSCORESCOLUMN1` and `GETSCORESCOLUMN2` are each just one `STACKBLOCKS`
call with a different `first_col`. Read that spec's §2.1 for the base layer
this appendix's own formula grew into, rather than re-deriving it:

```
STACKBLOCKS(first_col, step, first_row)

=LET(
  n,   INT((COLUMNS(SCHEDULE!$1:$1) - first_col) / step) + 1,
  ref, LAMBDA(c, LET(L, SUBSTITUTE(ADDRESS(1, c, 4), "1", ""),
                     INDIRECT("SCHEDULE!" & L & first_row & ":" & L))),
  REDUCE(ref(first_col), SEQUENCE(n - 1, 1, first_col + step, step),
         LAMBDA(acc, c, VSTACK(acc, ref(c))))
)
```

Notes that cost time to rediscover:

- **`DROP` does not exist in Google Sheets** — it is Excel-only, as are
  `TAKE` and `EXPAND`. `VSTACK`, `CHOOSECOLS`, `TOCOL`, `REDUCE`, `LET` and
  `LAMBDA` all do. Hence `SEQUENCE(n - 1, …)` to build the tail rather than
  dropping the head.
- `INDIRECT` keeps the open-ended `E6:E` form, so the stacked height matches
  the hand-written original exactly. A `CHOOSECOLS` over a fixed rectangle
  is non-volatile but changes the height.
- `ADDRESS(1, c, 4)` gives `"E1"`; stripping the `1` yields the column
  letter, safe because column letters never contain digits.
- Convert **all** of these formulas together. If two of them are consumed
  row-for-row (codes against scores, say), one converted and one not will
  drift apart.
- `SEQUENCE(0, …)` errors, so a single-court sheet needs
  `IFERROR(…, ref(first))`.
- The block count tolerates up to 7 spare trailing columns before it invents
  a phantom court.
