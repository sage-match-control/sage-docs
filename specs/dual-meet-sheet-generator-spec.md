# Spec — Dual Meet Sheet Generator

Generate a dual-meet event workbook from a Tournament Time Calculator CSV
export, instead of duplicating the previous event's spreadsheet by hand and
find-replacing every team code.

Scoped to **dual meets only** — the club-A-versus-club-B format described in
`calculator-dual-meet-spec.md` and instantiated by
`_templates/dual-meet-template/` on the site side. Standard-format
tournaments are out of scope and must be rejected, not approximated.

**Status: Phase 1 built and in use.** Implemented at
`sage-tools-api/scripts/sheet-generator.gs`. Phase 2 (§10.1, `SCHEDULE`) and
Phase 3 (§10.2) remain unbuilt.

**Before trusting any cell reference in this document, read §13.** Building
Phase 1 against the live workbook turned up a dozen places where this spec
describes something the reference workbook does not actually do — the CODES
column's row count, where the roster sits, the shape of the feeder formulas,
and an entire undocumented score grid on the playoff blocks. §13 lists every
divergence and what the generator does instead. Where §13 and the sections
below disagree, §13 is what shipped.

Phase 1 (§4) generates every tab except the schedule. Phase 2 (§10.1)
generates `SCHEDULE` and is deliberately separated — it needs input the
calculator does not currently export.

**Read in this order:** §1 for why the design is shaped the way it is and
§1.1 for the functions every formula depends on; §4.0 for how a category tab
actually works; then §4.1 and §4.6 for the two variants; then §13 for what
changed in the building. §11 has the Apps Script mechanics and §12 a worked
example — note that §12's exact addresses are superseded by §13.

---

## 1. Why a bound script on a template copy

The category tabs do not run on native Sheets formulas. They run on
container-bound Apps Script custom functions that exist **only inside the
workbook** and are not in any repo:

```
GETTOTALWINS             GETTOTALLOSES         GETTOTALSCORE
GETTOTALOPPONENTSCORE    GETSCOREQUOTIENT      GETSCOREAGAINSTPAIR
GETPLAYERNAMESBYTEAMCODE GETTEAM1CODEBYMATCH   GETTEAM2CODEBYMATCH
SORTBYWINS
```

Verified — grepping both repos plus `event-data` for these names returns
nothing. A generator that creates a *blank* spreadsheet therefore produces
seven tabs of `#NAME?`.

**Consequence: generation must start from a copy of a master workbook.**
That single constraint rules out a Cloud Run endpoint driving the Sheets API,
which would also need a service account, Drive scopes, file-ownership and
sharing handling, and a bet on `files.copy` preserving a bound script project.

| | |
|---|---|
| Where it lives | `sage-tools-api/scripts/sheet-generator.gs` |
| How it ships | Bound Apps Script in the master workbook |
| Precedent | `scripts/sheets-sync.gs` — same repo location, same install story |
| New infra | none |
| New credentials | none |
| `sage-tools-api` change | none — no version bump, no deploy |

Operator flow:

1. `File -> Make a copy` of **SAGE Dual Meet Master**
2. Open the copy, `SAGE -> Generate event tabs`
3. Paste the calculator CSV, fill the two fields in §5, Generate

Keep parsing and building as pure functions — `parsePlanCsv(text)` and
`buildCategoryTab(ss, cat, settings)` — so a `POST /sheets/generate` endpoint
could later reuse them with only the transport changed.

### 1.1 The custom function contract

These are the ground truth for every formula in this spec, and they are
**not in any repo** — read them in the master workbook via
`Extensions -> Apps Script` before writing code. The signatures below are
what the existing workbooks' formulas demonstrably rely on:

| Function | Takes | Returns |
|---|---|---|
| `GETTOTALWINS(code)` | team code | number |
| `GETTOTALLOSES(code)` | team code | number (note the spelling — one `S`) |
| `GETTOTALSCORE(code)` | team code | points scored |
| `GETTOTALOPPONENTSCORE(code)` | team code | points conceded |
| `GETSCOREQUOTIENT(code)` | team code | ratio, used unrounded here and `ROUND(...,4)` in `STANDINGSCSV` |
| `GETSCOREAGAINSTPAIR(a, b)` | two team codes | `a`'s score in the `a` v `b` match |
| `GETPLAYERNAMESBYTEAMCODE(code)` | team code | **2-element array** — spills down two rows, or `INDEX(...,1)` / `INDEX(...,2)` |
| `SORTBYWINS(range)` | an `A:H` block range | team codes ranked by wins, then quotient |
| `GETTEAM1CODEBYMATCH(n)` / `GETTEAM2CODEBYMATCH(n)` | match number | team code — used by `CSV` and `Court Control`, not by category tabs |

`SORTBYWINS` is always wrapped in `INDEX(..., k)` to pull the k-th ranked
code, and must be entered as an **array formula**.

Scores are read from the `SCHEDULE` tab, which is why category tabs are
read-only summaries — nothing is typed into them except player names.

---

## 2. Input

The calculator's Export CSV. Comment lines start with `#`. Columns:

```
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format
```

Reference export (`pnf-x-bup-dual-meet-2026-09-12-plan.csv`):

```
setting,title,PNF x BUP Dual Meet
setting,date,2026-09-12
setting,start,15:00
setting,courts,9
setting,duration_min,25
setting,format,dual
setting,club_a,PNF
setting,club_b,BUP
category,LIWD,Low Intermediate Women's Doubles,,1,2,bye,,4,4,
category,LIMD,Low Intermediate Men's Doubles,,1,2,bye,,4,4,
... 7 category rows total
```

Note the empty `teams` and `po_format` cells — that is correct and current
(§2.1). The generator reads `club_a`, `club_b`, `teams_a`, `teams_b`,
`brackets`, `key` and `value`, and nothing else.

Second reference export, two brackets
(`pnf-x-bup-dual-meet-2026-09-12-plan 2 brackets.csv`):

```
setting,format,dual
setting,club_a,PNF
setting,club_b,BUP
category,LIWD,Low Intermediate Women's Doubles,,2,2,bye,,8,8,semis
```

36 matches = 32 group (2 brackets, `n = 4`, so `2 x n²`) + 2 club semis +
bronze + final.
`po_format` is populated here because `brackets > 1` makes it live — see
§4.6 and §9.1.

`brackets` is per category row, so one event may mix single- and
double-bracket categories. The generator must handle that, not assume the
whole file is uniform.

### 2.1 Columns the generator must ignore

| Column | Status |
|---|---|
| `teams` | **Always ignore.** No dual UI field since "Pairs per club" replaced it; never read by `calcDualCategory`. Means *total pairs in the category*, so a 4v4 category is 8 — never read it as pairs-per-club. |
| `advance`, `fill` | **Always ignore.** Standard-format bracket fields, unused on the dual path. |
| `po_format` | **Read only when `brackets > 1`.** Inert below that — see §9.1. |

As of the export fix (§9.1), `teams` is written **empty** in dual format, and
`po_format` is written empty for single-bracket categories, so a current
export makes the point self-evident. **The generator must not depend on
that.** Exports predating the fix — including the ones this spec was first
written against, and any the operator still has on disk — carry `teams: 16`
and `po_format: semis` in those cells. Both shapes must produce identical
output (§8).

---

## 3. Validation

Run all checks **before** creating or modifying any tab, and report every
failure at once rather than stopping at the first.

| Rule | Reason |
|---|---|
| `setting format == dual` | Templates encode club-vs-club geometry |
| every `brackets` is `1` or `2` | A club playoff needs exactly two bracket winners to pair off (§4.6.1). Three brackets would leave three winners and no defined structure — this is a competition-rules limit, **not** a template limit (§4.1) |
| every `teams_a == teams_b` | Bracket blocks are square; unequal clubs would leave byes the templates cannot render |
| `teams_a % brackets == 0`, quotient in `2..12` | Block size `n = teams_a / brackets` must divide evenly (§4.1.1). The range is a sanity bound, not a structural limit — the layout is computed, so any `n` lays out. Known-good: 4 (PNF x BUP), 6 (PPA x Club 2600) |
| ~~`po_format != 'aggregate'` when `brackets > 1`~~ | **No longer rejected — implemented, see §9.4.** |
| `key` unique, `[A-Z0-9]{2,8}`, legal tab name | Tab name is the key |
| at least one category row | — |
| no existing tab collides | See §9.2 |

A failure must abort with the offending rows named. Emitting a tab whose
geometry does not match the config it was generated from is the failure mode
this section exists to prevent — it produces plausible-looking wrong
medalists rather than an obvious error.

---

## 4. Phase 1 — every tab except the schedule

### 4.0 Anatomy of a category tab

Read this before §4.1. Every generated tab is the same organism at a
different size; the columns never move, only the row counts.

#### 4.0.1 Column map

| Col | Holds |
|---|---|
| `A` | Team code — **repeated on both rows** of a pair |
| `B` | Player name — a lookup formula on the pair's first row that spills to the second |
| `D` `E` | W, L |
| `F` `G` | Points for, points against |
| `H` | Quotient |
| `I` | Notes — **playoff blocks only**; carries the feeder result (§4.1.4) |
| `J` | Matches played, `=D+E` — group blocks only |
| `K` | Bracket rank — group blocks only |
| `M`–… | Cross-club score grid (§4.0.3) |
| `AA` `AB` `AC` | Club A name-entry scaffold: index, names, code link (§4.0.2) |
| `AF` `AG` | Club A codes, randomized codes |
| `AP` `AQ` `AR` | Club B name-entry scaffold |
| `AU` `AV` | Club B codes, randomized codes |
| `BA`–`BD` | Awards block — **deleted**, see §4.2 |

Header chrome: row 1 has the title in `A1` and club names in `AB1`/`AQ1`;
row 2 has the `STEP 1/2/3` labels; rows 3–4 carry the column headers
(`W`, `L`, `Team Scr`, `Opp
Scr` — note the newline, `Q`, `Notes`,
`Br Rank`) and the scaffold's `NAMES`/`CODES`/`RANDOMIZED CODES` labels.
Each group block repeats the `W`/`L`/… header on its own header row.

#### 4.0.2 How names reach the standings

This is the least obvious mechanism on the tab. Three columns cooperate:

- `AF5:AF…` — one pair code per row, `<club>_<key>_1..teams_a`
- `AG5:AG…` — the same codes after the operator shuffles them (`STEP 3`)
- `AB5:AB…` — **two rows per pair**, the operator's typed player names
- `AC5:AC…` — links each name row to a code, alternating:

```
AC5  =AG5        first name of pair 1
AC6  =AC5        second name of pair 1
AC7  =AG6        first name of pair 2
AC8  =AC7
```

so `AC<5+2i> = AG<5+i>` and the even row repeats the row above.

Column `B` then resolves a name from the code:

```
B4  =IFNA(FILTER($AB$5:$AB1028,$AC$5:$AC1028=A4),A4)
```

It spills two rows, and falls back to showing the team code when no name has
been entered yet — which is why a freshly generated tab reads
`PNF_LIWD_1` in the name column rather than erroring.

Club B mirrors this exactly in `AP`/`AQ`/`AR` + `AU`/`AV`, with `B` pointing
at `$AQ$5:$AQ1028` / `$AR$5:$AR1028`.

#### 4.0.3 The cross-club score grid

Two mirrored `n x n` grids, one per club, each anchored on its own block's
first row. For a block whose first pair row is `first`, and the opposing
block's is `otherFirst`:

| Cell | Content |
|---|---|
| `M<first>` | `<club> SCORES` label |
| `N<first>` … `<n`-th col`><first>` | `=M<otherFirst+1+j>` — the opponents' row labels |
| `M<first+1+k>` | `=A<first + 2k>` — this club's pair codes, one row per pair |
| grid body | `=GETSCOREAGAINSTPAIR($M<row>, <col>$<first>)` |

Verified at both sizes: `n=4` spans `N:Q`, `n=6` spans `N:S`. The grid is
`n` rows tall (one per pair, **not** two) and `n` columns wide.

#### 4.0.4 Merges

Every pair occupies two rows, and the per-pair stat columns are merged
vertically across them — `D4:D5`, `E4:E5`, … for each pair in turn. Group
blocks merge `D`–`H` plus `J` and `K`; playoff blocks merge `D`–`H` only
(`I` stays unmerged, since its two rows hold different formulas).

**The generator must create these merges itself** for every pair it lays
down. Copy whichever columns the prototype row-pair has merged rather than
hardcoding the list.

### 4.1 Category tabs — single bracket

**There is one template tab, `_CATEGORY_TEMPLATE`, for both variants.**

A category tab is built from exactly two repeating block types (§4.0.4), and
their formatting is identical across every workbook, every `n`, and both
bracket counts:

| Block type | Rows | Merged columns | Instances |
|---|---|---|---|
| Group pair | `2n` per block | `B, D, E, F, G, H, J, K` | 2 blocks at `brackets: 1`, 4 at `brackets: 2` |
| Playoff pair | 4 per block | `D, E, F, G, H` | Bronze + Final, plus 2 Semis when `brackets: 2` |

The semi block is structurally indistinguishable from bronze and final —
same shape, same merges, same `=IF(F<r>=11,1,0)` head-to-head idiom. It is
not a new kind of thing, just another instance of a block the generator
already lays down.

So the two variants differ only in **how many blocks get emitted**, which
the layout loop (§4.1.1) already handles. The template carries one prototype
of each block type plus the shared chrome; it does not carry a finished tab
shape for either variant.

Each block holds `n = teams_a / brackets` pairs (§4.1.1). This section covers
the single-bracket arrangement; §4.6 covers the two-bracket one — same
template, same algorithm, more iterations.

All seven single-bracket tabs in the reference workbook are structurally
identical *at that event's `n = 4`* — same 1028 rows, same 89 merges, same
~80 formulas, same headers. That uniformity is what makes one template
viable; it is not evidence the geometry is fixed. Verified by diffing `LIWD`
against `LIMD` cell by cell — they differ only in codes and labels.

**Build by `template.copyTo(ss)`, then lay the tab out from `n`.**

The copy supplies only what does not vary with pair count: column widths,
fills and merges, header chrome, the name-entry scaffold's labels, and —
the reason a copy is mandatory at all — the workbook's bound custom
functions (§1).

**It does not supply row geometry.** `n` is variable (§4.1.1), so the number
of rows in every block, the position of every block, and the ranges inside
every formula are all computed and written by the generator. A template tab
is a *style carrier and a formula prototype*, not a finished shape to stamp
a few cells into.

#### 4.1.1 Geometry — everything scales with `n`

`n = teams_a / brackets` is the pairs each club fields in one bracket. It is
**not fixed** — both real events differ, and neither is authoritative:

| Event | `teams_a` | `brackets` | `n` | Rows per block | Gate |
|---|---|---|---|---|---|
| PNF x BUP (`LIWD`) | 4 | 1 | 4 | 8 | `SUM(D4:E11)=16` |
| PPA x Club 2600 (`IWD`) | 12 | 2 | 6 | 12 | `SUM(D4:E15)=36` |

Every pair occupies **two rows** (one per player), so a block is `2n` rows.
Every pair plays only the `n` opposing pairs inside its own bracket, so a
block is complete when `W + L` across its `n` pairs equals **`n²`** — the
constant each `IF(SUM(...)=…)` gate checks before seeding a playoff slot.

The generator computes a layout table once per tab and derives every range
from it:

```
row = 1                                  # A1 title
for b in 1..brackets:
    for club in [A, B]:
        row += (b > 1 and club == A) ? 3 : 2   # blank(s), then header
        header = row
        first  = row + 1
        last   = first + 2n - 1
        emit block(bracket=b, club, header, first, last)
        row = last
```

Playoff blocks follow the same shape with a fixed two entrants (`2 x 2 = 4`
rows): a banner row, then per-club semi blocks if `brackets > 1` (§4.6),
then bronze and final. Both reference tabs match this to within one row of
cosmetic gap; pick one set of gap constants and apply it consistently rather
than reproducing the hand-built variance.

Worked out for the single-bracket reference tab (`LIWD`, `n = 4`) — compare
against §4.6.2, which is the same algorithm at `n = 6, brackets = 2`:

| Rows | Block |
|---|---|
| `1` | Title |
| `3` / `4:11` | Club A header, then `n` pairs (`2n = 8` rows) |
| `13` / `14:21` | Club B header, then `n` pairs |
| `24` | `FINALS MATCHES` banner |
| `26` / `27:30` | `BRONZE B` — club A `_B` v club B `_B` |
| `32` / `33:36` | `FINALS` — club A `_F` v club B `_F` |

At `brackets = 1` there is **no semis block** — the bronze and final feeders
read straight off the group blocks (§4.1.4).

#### 4.1.2 What the generator writes

Derived from the layout table, per block:

| What | Where | Value |
|---|---|---|
| Tab name | — | `category.key` |
| Display title | `A1` | `category.value`, uppercased |
| Club label | block `header` in `A`; `AB1` / `AQ1` | `club_a` / `club_b` |
| Score grid header | `M<first>` | `<club> SCORES` |
| Pair codes | `A<first>:A<last>` | `<club>_<key>_<i>`, each on two rows |
| Roster codes | `AF`/`AG` (A), `AU`/`AV` (B), from row 5 | `<club>_<key>_1..teams_a`, plus 2 rows slack |
| Playoff entrants | bronze and final blocks | `<club>_<key>_B`, `<club>_<key>_F` |

Tabs are ordered to match CSV row order.

#### 4.1.3 Formulas the generator fills

The template carries one prototype row-pair per block type. The generator
fills it down to `2n` rows and rewrites every range from the layout table.
Nothing here can be inherited unchanged, because all of it is `n`-dependent:

- Per-pair stats, filled down each block:
  `=GETTOTALWINS(A<r>)`, `=GETTOTALLOSES(A<r>)`, `=GETTOTALSCORE(A<r>)`,
  `=GETTOTALOPPONENTSCORE(A<r>)`, `=GETSCOREQUOTIENT(A<r>)`,
  `J` = `=D<r>+E<r>`
- The cross-club score grid, an **`n x n`** array of
  `=GETSCOREAGAINSTPAIR(...)` spanning `M:` through the `n`-th column
- The name-entry scaffold's `AC`/`AR` code-linking formulas, `2 x teams_a`
  rows deep
- The playoff feeders, with both the range and the gate constant computed
  (§4.1.4, §4.6.4)

#### 4.1.4 Playoff feeders — single bracket

Top of a club's standings to the final, second to bronze. Shown at `n = 4`;
`16` is `n²` and every range comes from the layout table:

```
I34  =IF(SUM(D4:E11)=16, INDEX(SORTBYWINS(A4:H11),1), "-")    club A finalist
I28  =IF(SUM(D4:E11)=16, INDEX(SORTBYWINS(A4:H11),2), "-")    club A bronze
I36  =IF(SUM(D14:E21)=16, INDEX(SORTBYWINS(A14:H21),1), "-")  club B finalist
I30  =IF(SUM(D14:E21)=16, INDEX(SORTBYWINS(A14:H21),2), "-")  club B bronze
```

#### 4.1.5 Player names are out of scope

The calculator CSV carries no player names. Generated tabs come out with the
`AB` / `AQ` name-entry blocks empty (`2 x teams_a` rows each); pasting the
roster stays a
human step. The completion toast must say so.

### 4.2 Remove the awards block

Columns `BA:BD` on each category tab — the Gold/Silver/Bronze podium readout
at `BA3:BD9` and its merges — are **cut from the template entirely**.

Safe to delete: verified that no cell outside a category tab's own `BA:BD`
range references it. (`BA`–`BD` hits on `SCHEDULE` are that tab's own court
columns, unrelated.) The console's Awards tab
(`awards-podium-tab-spec.md`) derives the podium from the day snapshot
independently, so nothing downstream loses a source.

This also deletes a live bug rather than requiring it be fixed: the Silver
cell read `=IFNA(IFS(D35,A33,D33,A735),"-")`, where `A735` is a typo for
`A35`, so Silver fell back to `"-"` whenever club A won the final.

### 4.3 `Variables`

| Cells | Value |
|---|---|
| `C2:C8` | `category.key`, one row per category |
| `D2:D8` | `category.value` |
| `G2` | `setting courts` |
| `H2` | `setting start` |
| `I2` | `setting duration_min`, as a duration |

The `A2` reference-number ladder (`=A2+1` chained down column A) is constant
and comes with the template.

Generating `C:D` from `category.key` is the **structural fix for the
`LIMD`/`LIWD` typo** in the current sheet, where `C2` reads `LIMD` while `D2`
reads "Low Intermediate Women's Doubles" — leaving the map with a duplicate
`LIMD` and no `LIWD` at all. Derived from the key, it cannot drift again.

### 4.4 `Title`

| Cell | Value |
|---|---|
| `B6` | `setting title` |
| `B7` | `setting date` formatted, plus the venue label from §5 |
| `B8` | `https://sage-match-control.github.io/events/<event-key>` |

`B8` is the second fix: the current sheet still points at
`events/ppa-x-club-2600-dual-meet`, inherited from the workbook this one was
duplicated from — the same lineage that produced the stale-gid note already
recorded in `event-data/config/events.json`.

`<event-key>` defaults to the slugified title and is overridable (§5).
`"PNF x BUP Dual Meet"` slugifies to `pnf-x-bup-dual-meet`, which is exactly
the key already registered in `events.json`.

### 4.5 `Reference for Players`

One anchor per category, each spilling team code and both player names out
of its category tab:

```
=FILTER(<KEY>!$A$1:$C580, <KEY>!$A$1:A580>"")
```

Anchors are placed at **cumulative offsets computed from each tab's actual
height**, not a fixed stride — a two-bracket tab (§4.6) spills roughly twice
as many rows as a single-bracket one, so a uniform stride sized for the
larger wastes space and one sized for the smaller silently overlaps.

```
offset(0)   = 3
offset(i+1) = offset(i) + spillHeight(tab i) + 20   // 20 rows slack
```

`spillHeight` is the count of non-empty column-A cells the generator just
wrote to that tab. It knows this exactly, having just laid the tab out
(§4.1.1) — roughly `4 x teams_a` plus banners and playoff rows, so it grows
with both `n` and `brackets` rather than being a constant.

This replaces the current hand-placed irregular anchors at
`A3, A81, A169, A253, A335, A417, A499`, which were sized by hand for one
event's `n` and would overlap silently at a larger one. Check the last
anchor still lands inside the `A2:A1553` range `STANDINGSCSV` reads, and
widen that range if a large event ever overruns it.

Deliberately left as formulas rather than pasted values, so the tab
self-populates the moment names are entered on the category tabs. That is
what `STANDINGSCSV` consumes, through
`UNIQUE(FILTER('Reference for Players'!A2:A1553, REGEXMATCH(...,"_")))`.

### 4.6 Category tabs — two brackets

For `brackets: 2`, `teams_a: 8`, `teams_b: 8`, `po_format: semis`.

#### 4.6.1 The competition rules being encoded

- Every club-A pair plays every club-B pair **within its own bracket**. No
  pair ever faces a teammate during the group stage.
- Each club is ranked **separately within each bracket**, by wins then
  quotient.
- Each club's two bracket winners then play off **internally** to decide that
  club's #1 and #2.
- Final and Bronze are always one pair from each club.

Match count, matching the reference export's 36: 32 group
(2 brackets x 4 x 4) + 2 club semis (one per club) + 1 bronze + 1 final.

#### 4.6.2 Layout

Taken from a working example rather than invented: `IWD` in the archived
**PPA x Club 2600** workbook, which is `teams_a: 12, brackets: 2`, so `n = 6`
(12 rows per block, two per pair). Row numbers below are that tab's; the
generator computes its own from `n`.

Same template and same layout loop as §4.1 — the only difference is that
`brackets: 2` emits four group blocks instead of two and inserts a semis
stage before the finals stage.

| Rows | Block |
|---|---|
| `A1` | Title |
| `3` / `4:15` | Bracket 1 — club A header, then `n` pairs |
| `17` / `18:29` | Bracket 1 — club B header, then `n` pairs |
| `32` / `33:44` | Bracket 2 — club A header, then `n` pairs |
| `46` / `47:58` | Bracket 2 — club B header, then `n` pairs |
| `60` | `SEMIFINALS MATCHES` banner |
| `62` / `63:66` | Club A semi — `SF_1` v `SF_2`, **intra-club** |
| `68` / `69:72` | Club B semi — `SF_1` v `SF_2`, **intra-club** |
| `74` | `FINALS MATCHES` banner |
| `76` / `77:80` | `BRONZE B` — club A `_B` v club B `_B` |
| `82` / `83:86` | `FINALS` — club A `_F` v club B `_F` |

The same workbook carries single-bracket categories (`HIXD`, `HIMD`) in the
identical file, confirming both variants coexist and §2's mixed-event
requirement is real, not hypothetical.

#### 4.6.3 Team codes

Group pairs run `1..teams_a` per club, the first `n` in bracket 1 and the
next `n` in bracket 2. Bracket membership is **not** encoded in the code —
it is carried by the `bracket` column (§4.6.5).

Club-semis entrants add four codes per category, matching the existing
workbook's convention exactly — **note the underscore before the digit**:

```
<club_a>_<key>_SF_1   <club_a>_<key>_SF_2
<club_b>_<key>_SF_1   <club_b>_<key>_SF_2
```

`SF` is already understood downstream — `roundKeyword()` in the dual-meet
template matches `/(^|_)SF(_|\d|$)/`, and `standingsStageKey()` maps it to
the `SF` stage, which is in `STAGE_ORDER` but deliberately **not** in
`CROSS_CLUB_STAGE_KEYS`. That is exactly right for an intra-club round: the
site renders SF once per club column rather than spanning both.

`_B` and `_F` are unchanged, so the console's Awards tab
(`awards-podium-tab-spec.md`), which keys off those two suffixes,
keeps working with no change.

#### 4.6.4 Feeders

The feeder pattern is **identical to §4.1.4** — `INDEX(SORTBYWINS(block),1)`
for the winner, `,2)` for the runner-up, gated on the block being complete.
Only the block it points at changes. Verbatim from the reference tab:

```
I64  =IF(SUM(D4:E15)=36,  INDEX(SORTBYWINS(A4:H15),1),  "-")   A SF_1 <- A bracket 1 winner
I66  =IF(SUM(D33:E44)=36, INDEX(SORTBYWINS(A33:H44),1), "-")   A SF_2 <- A bracket 2 winner
I70  =IF(SUM(D18:E29)=36, INDEX(SORTBYWINS(A18:H29),1), "-")   B SF_1 <- B bracket 1 winner
I72  =IF(SUM(D47:E58)=36, INDEX(SORTBYWINS(A47:H58),1), "-")   B SF_2 <- B bracket 2 winner

I84  =IF(SUM(D63:E66)=2,  INDEX(SORTBYWINS(A63:H66),1), "-")   A _F <- A semi winner
I78  =IF(SUM(D63:E66)=2,  INDEX(SORTBYWINS(A63:H66),2), "-")   A _B <- A semi loser
I86  =IF(SUM(D69:E72)=2,  INDEX(SORTBYWINS(A69:H72),1), "-")   B _F <- B semi winner
I80  =IF(SUM(D69:E72)=2,  INDEX(SORTBYWINS(A69:H72),2), "-")   B _B <- B semi loser
```

Two gate values, both derived (§4.1.1): `n²` for a group block (36 here, 16
at `n = 4`), and `2` for a semi block — two entrants, one match, so `W + L`
totals 2 once played.

Intra-club W/L uses the template's existing head-to-head idiom, unchanged:

```
D63  =IF(F63=11,1,0)      E63  =IF(F65=11,1,0)
```

#### 4.6.5 The `bracket` column becomes load-bearing

`STANDINGSCSV`'s `bracket` column (G) is blank today and unused. With two
brackets it **must** be populated — `1` and `2`, or `A` and `B` — because
that is how the site separates the two group tables.

`renderStageTables()` in the dual-meet template groups RR rows by
`s.bracket` and sorts each group by wins desc then quotient desc, which is
precisely the ranking rule in §4.6.1. Its comment is explicit that a blank
bracket means "one single group, no Br column shown" — correct for
single-bracket categories, and wrong for these.

Populating that column is Phase 3 work (§10.2), but it is recorded here
because it is a *correctness* requirement of two-bracket support, not a
cosmetic one: leave it blank and both brackets merge into one table ranked
against each other.

---

## 5. Inputs the CSV does not carry

Two sidebar fields:

| Field | Default | Used by |
|---|---|---|
| Venue label | empty | `Title!B7` — the CSV has the date but no venue (`PPC`) |
| Event key | slugified `setting title` | `Title!B8` |

---

## 6. The master workbook

A one-time manual job, and the larger half of the work:

1. Copy the current PNF x BUP workbook.
2. Reduce the seven category tabs to a single hidden `_CATEGORY_TEMPLATE`
   with names cleared and the `BA:BD` awards block deleted (§4.2).
3. Confirm that template contains **one prototype of each block type**
   (§4.1): one group pair (two rows, `B/D/E/F/G/H/J/K` merged) and one
   playoff pair (two rows, `D`–`H` merged). The stripped `LIWD` already has
   both. Nothing two-bracket-specific is needed — see §4.1.
4. Clear `Variables!C2:D8`, `Title!B6:B8`, and the `Reference for Players`
   anchors — the generator writes all of them.
5. Keep the bound script project, which carries the custom functions in §1.
6. Name it **SAGE Dual Meet Master**.

One template covers both variants, so there is no second template to build
and nothing here gates the two-bracket path.

---

## 7. Build order

1. `parsePlanCsv` + §3 validation, reporting all failures together
2. `_CATEGORY_TEMPLATE` in the master workbook (§6) — one template, both variants
3. `buildCategoryTab` — copy, rename, compute the layout table (§4.1.1),
   size every block, then write §4.1.2 values and §4.1.3 formulas
4. `Variables` (§4.3) and `Title` (§4.4)
5. `Reference for Players` (§4.5)
6. Menu + sidebar with the two fields from §5

---

## 8. Acceptance

Run against the reference CSV in §2 and confirm:

- Seven tabs named `LIWD LIMD LIXD HIWD HIMD HIXD AMD`, in CSV order
- The two-bracket reference CSV yields one `LIWD` tab with four group blocks,
  two intra-club semi blocks, and bronze/final — 36 matches' worth of slots
- A mixed CSV (some categories `brackets: 1`, some `2`) generates each from
  its own template, as the PPA x Club 2600 workbook does
- No `#NAME?` anywhere — the custom functions resolve
- No `BA:BD` content on any category tab
- `Variables!C2:D8` has seven distinct keys including `LIWD`
- `Title!B8` reads `…/events/pnf-x-bup-dual-meet`
- Pasting a roster into `LIWD!AB5:AB12` (the `n = 4` case) populates `LIWD!B4:B11`,
  `Reference for Players`, and `STANDINGSCSV` without further action
- A CSV with `teams_a: 5, brackets: 2` (not divisible), or
  `po_format: aggregate` with `brackets: 2`, is rejected with a clear message
  and leaves the workbook untouched
- A pre-fix export (`teams: 16`, `po_format: semis`) and a post-fix export
  (both cells empty) of the same event produce byte-identical output — the
  regression test for §2.1

---

## 9. Decisions and open questions

### 9.1 `po_format` and `teams` are ignored, deliberately

**Fixed in the calculator — but the rule stands.** `exportCSV` now blanks
both cells when the UI gives the operator no way to set them (see the end of
this section). Current exports therefore show them empty. The generator still
must not read them, because older exports do carry values and both must
behave the same (§2.1, §8).

Six of seven rows in the original export read `po_format: semis`, which
looks like a per-category choice. It is residue from the `newCat()` default
at `tournament-calculator.html:692`, and had no effect anywhere:

- The toggle is not rendered at all below two brackets —
  `const poUI = p.groups>=2 ? … : ''` (line ~796)
- The value is discarded before use — `gs.brackets>1 ? (…) : 'aggregate'`
  (line ~422)
- `dualPlayoffPlan`'s `if(k<=1)` branch returns bronze+final before
  `poFormat` is read (line ~376)

The arithmetic confirms it: the export header's 126 matches is
7 × 16 group + 7 × 2 playoff. Live `semis` rows would add two matches each,
giving 138.

**Below two brackets the generator must not read this column** — the value
there is noise, and the single-bracket template implements the only playoff
shape the calculator will actually plan. At two brackets it *is* live and
must be read, which is what §4.6 and §9.4 handle.

The same root cause hit the `teams` column: dual cards render "Pairs per
club" (`data-k="pairsPerClub"`, writing only `c.teamsA`/`c.teamsB`) and never
expose `c.teams`, so every dual category exported whatever `teams` it was
born with — `16` from `newCat()`, or a stale leftover. It was
self-perpetuating, since import read column 4 straight back into `c.teams`.

**Both fixed** in `exportCSV`, which now blanks each cell exactly when its
UI control is absent:

```js
const isDual=cfg.format==='dual';
const teamsCell=isDual?'':c.teams;
const poFormatCell=(isDual&&p&&p.groups>=2)?(c.poFormat||''):'';
```

`p.groups>=2` mirrors the same gate the playoff-format toggle uses in
`recalc()`. No import change was needed — the existing fallbacks
(`teams||16`, `po_format` → `'semis'`) already treat a blank cell as the
untouched default, so round-trips are byte-identical. Standard format is
untouched: `teams` is still a live field there.

### 9.2 Decided — collision policy: abort

Settled as the section's own default suggested: if any target tab name
already exists, nothing is written and the sidebar lists every collision.
Renaming to `<key>_old` was rejected — silently keeping a half-replaced set
of tabs around mid-event is exactly the outcome worth avoiding, and the
operator can rename or delete deliberately before re-running.

Implemented in `generateEventTabs`, alongside the §3 validation errors, so a
collision is reported in the same "nothing was changed" pass as a bad CSV
rather than as a separate failure mode.

### 9.3 Open — the `HIgh` typo

The HIWD row's `value` reads `HIgh Intermediate Women's Doubles`. Invisible in
`A1` because the generator uppercases, but `Variables!D5` is written verbatim
and would regress a cell that is currently correct.

Fix the CSV at source rather than title-casing `value` in code — normalising
casing would also quietly rewrite legitimately-styled names later.

---

### 9.4 Implemented — multi-bracket `aggregate`

Someone wanted it: PNF x BUP 2026's `AMD` category is 8 pairs a side over 2
brackets with `po_format: aggregate`. No longer rejected.

The rule, as the operator stated it: each club takes the **top 1 of each of
its brackets**, then those two are compared on record — most wins, then best
quotient — with the better one going to the final and the other to bronze.
No extra match is played. The calculator's own arithmetic agrees: its
`dualPlayoffPlan` emits bronze + final only under `aggregate`, and the
reference CSV's 215-match header reconciles exactly when `AMD` contributes
34 (32 group + bronze + final) rather than the 36 a semis category costs.

This section's warning about `SORTBYWINS` was the real obstacle and it held:
it ranks one contiguous `A:H` block, and there is no single range spanning a
club's two brackets. So the feeder is an explicit two-way comparison rather
than one wider sort, with `LET` binding each bracket's winner once:

```
=LET(br1_win,IF(SUM($J$4:$J$11)=16,INDEX(SORTBYWINS($A$4:$H$11),1),"-"),
     br2_win,IF(SUM($J$14:$J$21)=16,INDEX(SORTBYWINS($A$14:$H$21),1),"-"),
     IF(OR(br1_win="-",br2_win="-"),"-",
        IF(OR(GETTOTALWINS(br1_win)>GETTOTALWINS(br2_win),
              AND(GETTOTALWINS(br1_win)=GETTOTALWINS(br2_win),
                  GETSCOREQUOTIENT(br1_win)>=GETSCOREQUOTIENT(br2_win))),
           br1_win,br2_win)))
```

The bronze feeder is identical but returns the other side. Both stay `"-"`
until both brackets are complete.

Two traps worth recording, both of which cost a debugging round:

- **`LET` names cannot look like cell references.** The obvious `w1`/`w2`
  are read as cells W1 and W2 and Sheets rejects them outright with
  *"argument 1 of LET is not a valid name"* — hence `br1_win`/`br2_win`,
  since an underscore cannot appear in A1 notation.
- The gate is `n²` computed from the pair count, never a literal. A 3-pair
  category emits 9, a 6-pair one 36.

Aggregate emits **no semi blocks and no semis banner** (`usesSemis_` gates
both the layout and the writers), so an aggregate tab is shorter than the
equivalent `semis` tab by the banner plus two blocks.

## 10. Out of scope

### 10.1 Phase 2 — `SCHEDULE`

Nothing in the CSV contains a schedule. The calculator computes match
*counts* only (`dualPlayoffPlan`, line ~376). Either port the cross-product
pairing plus court and slot assignment into the generator, or teach the
calculator to export the match list. That fork should be decided before
Phase 2 starts.

### 10.2 Phase 3 — `CSV`, `STANDINGSCSV`, `Court Control`

Row-count-driven: fixed formulas sized to the match count and roster.
Mechanical once Phase 2 fixes the numbers.

### 10.3 Not planned

- Standard-format (non-dual) tournaments
- Block sizes outside `2..12` pairs per club per bracket (§4.1.1)
- More than two brackets in one category
- Multi-bracket `aggregate` (§9.4)
- Player name import (§4.1.5)

---

## 11. Implementation notes

### 11.1 Apps Script mechanics

- `templateSheet.copyTo(ss)` returns the new sheet named `Copy of <name>`;
  `setName(key)` then `showSheet()` it, and `setActiveSheet`/`moveActiveSheet`
  to order it.
- Size a block with `insertRowsAfter(row, count)` **before** writing into it,
  and do all inserts for a tab before any writes — inserting shifts every
  row below and invalidates offsets you already computed.
- **Formatting propagates on insert.** Inserting rows inside an existing
  prototype block carries its fills, borders and column formats down, which
  is what lets one template serve every `n` and both bracket counts (§4.1).
  Merges do **not** propagate — create those explicitly per pair (§4.0.4).
- To place a second, third or fourth block, `copyTo` the prototype block's
  range onto the target rows and then resize it, rather than rebuilding
  formatting by hand.
- Write in **batches**, never cell by cell.
  `getRange(r, c, numRows, numCols).setValues(...)` for literals and
  `.setFormulas(...)` for formulas. A per-cell loop on a 1000-row tab will
  hit the 6-minute execution limit.
- `SORTBYWINS` feeders must be set with `setFormula` on a range entered as an
  array formula — check how the prototype cell is stored in the template and
  reproduce it, rather than assuming a plain `setFormula` suffices.
- `SpreadsheetApp.flush()` after the tab is complete, before moving to the
  next, keeps partial state from confusing a later `getRange` read.
- Custom functions recalculate lazily. Immediately after generation the tab
  may show `Loading...`; that is expected and not a failure to report.

### 11.2 Ordering

Generate all category tabs first, then `Variables`, `Title`, and
`Reference for Players` — the last one needs each category tab's final
spill height (§4.5), which is only known once its tab exists.

### 11.3 Failure handling

Validation (§3) runs to completion and reports every problem at once. After
validation passes, a mid-run failure should leave the workbook in whatever
state it reached and say plainly which tabs were written — do not attempt a
rollback, and do not continue past an error hoping later tabs succeed.

### 11.4 What "done" looks like to the operator

A completion toast naming the tabs created, and stating explicitly that
**player names still need pasting** (§4.1.5) — the single most likely
misreading is that a generated workbook is ready to run.

---

## 12. Worked example

Input: the seven-category reference CSV in §2 (`brackets: 1`,
`teams_a: teams_b: 4`, so `n = 4`), clubs `PNF` and `BUP`.

Expected `LIWD` tab after generation:

```
A1                    LOW INTERMEDIATE WOMEN'S DOUBLES
A3                    PNF                     (+ W/L/Team Scr/Opp Scr/Q/Notes/Br Rank headers)
A4:A11                PNF_LIWD_1 .. PNF_LIWD_4, each on two rows
D4  E4  F4  G4  H4    =GETTOTALWINS(A4) =GETTOTALLOSES(A4) =GETTOTALSCORE(A4)
                      =GETTOTALOPPONENTSCORE(A4) =GETSCOREQUOTIENT(A4)   (merged D4:D5 etc.)
J4                    =D4+E4
B4                    =IFNA(FILTER($AB$5:$AB1028,$AC$5:$AC1028=A4),A4)

M4                    PNF SCORES
N4:Q4                 =M15  =M16  =M17  =M18
M5:M8                 =A4   =A6   =A8   =A10
N5                    =GETSCOREAGAINSTPAIR($M5,N$4)          ... through Q8

A13                   BUP                     (mirror of the A3 block)
A14:A21               BUP_LIWD_1 .. BUP_LIWD_4
M14                   BUP SCORES
N14:Q14               =M5  =M6  =M7  =M8
M15:M18               =A14 =A16 =A18 =A20

A24                   FINALS MATCHES
A27:A28  A29:A30      PNF_LIWD_B / BUP_LIWD_B
I28                   =IF(SUM(D4:E11)=16,INDEX(SORTBYWINS(A4:H11),2),"-")
I30                   =IF(SUM(D14:E21)=16,INDEX(SORTBYWINS(A14:H21),2),"-")
A33:A34  A35:A36      PNF_LIWD_F / BUP_LIWD_F
I34                   =IF(SUM(D4:E11)=16,INDEX(SORTBYWINS(A4:H11),1),"-")
I36                   =IF(SUM(D14:E21)=16,INDEX(SORTBYWINS(A14:H21),1),"-")

AB1  AQ1              PNF   BUP
AF5:AF10              PNF_LIWD_1 .. PNF_LIWD_6      (teams_a + 2 slack)
AU5:AU10              BUP_LIWD_1 .. BUP_LIWD_6
AB5:AB12  AQ5:AQ12    empty — operator pastes names here
AC5 =AG5, AC6 =AC5, AC7 =AG6, AC8 =AC7, ...
```

Plus `Variables!C2:D8` carrying all seven key/name pairs, `Title!B6:B8`, and
seven `Reference for Players` anchors.

Change that CSV's `LIWD` row to `brackets: 2, teams_a: 8, teams_b: 8,
po_format: semis` and the same category becomes the §4.6 shape: four group
blocks of `n = 4`, gates of `16`, two intra-club semi blocks feeding bronze
and final, and `PNF_LIWD_1..8` / `BUP_LIWD_1..8` pair codes plus four
`_SF_1`/`_SF_2` codes.

---

## 13. As built — where this spec and the workbook disagreed

Phase 1 was implemented against the live PNF x BUP workbook rather than
against this document alone, and the two turned out to differ in a dozen
places. Everything below is what `sheet-generator.gs` actually does, verified
cell by cell in the reference workbook. **Where this section contradicts §4
or §12, this section is correct** — the earlier sections are kept as the
original design record.

The recurring lesson: this spec's worked example (§12) was written from one
category at one size (`n = 4`, one bracket). Several details that look like
general rules there are artefacts of that one case.

### 13.1 The roster scaffold

| Spec says | Actually |
|---|---|
| CODES column is `teams_a + 2` rows, "2 slack" (§4.0.2, §12) | Exactly `teams_a` rows. The reference has no slack rows at all. |
| Scaffold sits beside each club's own block | **Both** clubs' scaffolds sit beside **club A's** bracket-1 block — same rows, club A in `AA:AG`, club B in `AP:AV`. Club B's is nowhere near club B's block. |
| (not mentioned) | A pair-index number `1..teams_a` sits in `AA`/`AP`, one per pair, on that pair's first name row only. |
| (not mentioned) | Row `header-1` carries `STEP 1/2/3`; row `header` carries "Put names here" / "Copy these numbers" / "Put randomized codes". |

Colors are fixed values read off the reference, not derived: header row
`#70ad47`, index column `#cccccc`, NAMES and RANDOMIZED `#fff2cc`, the
`AC`/`AR` link column `#b7b7b7`, CODES `#999999`.

### 13.2 The score grids

§4.0.3 describes the group-block grid only. **Playoff blocks have one too** —
undocumented here, found by reading the reference's bronze block. It is the
same `GETSCOREAGAINSTPAIR` mechanism, self-referential, with both entrants
serving as both row and column headers:

```
M14 "BRONZE B"   N14 =M15   O14 =M16      (label + entrant headers)
M15 =A15         M16 =A17                 (one grid row per entrant)
O15 =GETSCOREAGAINSTPAIR($M15,O14)        (A's score vs B)
N16 =GETSCOREAGAINSTPAIR($M16,N14)        (B's score vs A)
N15, O16 blank                            (an entrant against itself)
```

Two things about that geometry are easy to get wrong, and both were, twice:

- The grid's header row is `block.first`, **one row below** the block's
  A-column label at `block.header`.
- Like the group grid, it condenses each entrant's two rows into **one** grid
  row, so the two body rows are adjacent (`first+1`, `first+2`) even though
  the entrants themselves are two rows apart. The `M` formula still points at
  the entrant's own row (`A<first+3>`, not `A<first+2>`).

Grid formatting is **copied from named prototype cells in the template**
rather than set property by property — that is what carries the cell borders,
which an earlier `setBackground`-only version silently dropped. Sources:
`M4`/`N4`/`M5`/`N5` for club A's grid, `M8`/`N8`/`M9`/`N9` for club B's (so
each club's own colour comes along), and `M14`/`N14`/`O14`/`M15`/`M16`/`O15`
for playoff grids.

The reference also applies a **conditional format** to the grid columns —
`value >= 11` fills `#93C47D` over a flat `#D9EAD3` base. Conditional rules
are a sheet-level property that no `PASTE_FORMAT` copy carries, so the
generator adds the rule itself.

### 13.3 The feeder formulas

§4.1.4's shape is right in outline and wrong in three details:

| Spec (§4.1.4) | Actually |
|---|---|
| `SUM(D4:E11)` | `SUM($J$4:$J$11)` — column J is matches-played (`=D+E`); same total, but J is the column that means it |
| relative ranges | absolute (`$A$4:$H$11`) |
| feeder on the entrant's row | feeder on the entrant's **second** row, with the first row mirroring it (`I26` = `=I27`) |

That mirroring is why §4.0.4 notes column `I` stays unmerged while `D:H` are
merged: its two rows genuinely hold different formulas.

The gate is `n²` computed from the pair count. §12's `16` is that one
example's value, not a constant.

### 13.4 Block order at two brackets

§4.6.2 interleaves the clubs bracket by bracket (b1 club A, b1 club B, b2
club A, b2 club B). The generator groups **by club** instead — club A's
bracket 1 and 2 back to back, then club B's.

The interleaved order forced a run of blank padding rows between the two
clubs: club A's roster lists all `teams_a` pairs, which needs more vertical
room than its bracket-1 block alone has. Grouping a club's brackets together
means the roster spans that club's own blocks naturally, so no block is ever
padded — every block is exactly `2n` rows, and a club's pairs read top to
bottom. Single-bracket geometry is unchanged.

### 13.5 Smaller corrections

- **Column K (`Br Rank`) is left blank.** The spec names the column but never
  defines a formula, and the reference workbook has none either — only the
  header label. The operator fills it in by hand.
- **Playoff W/L** uses §4.6.4's `=IF(F<r>=11,1,0)` head-to-head idiom on
  every playoff block, not just semis — a bronze/final entrant's code is
  fresh for that single match, and the `F=11` form is the one the spec shows
  working end to end.
- **Gap constants** are this generator's own consistent set, as §4.1.1
  explicitly permits ("pick one set of gap constants and apply it
  consistently"). Single-bracket row numbers happen to match the reference;
  two-bracket ones do not, because of §13.4.
- **`_CATEGORY_TEMPLATE` must match `computeLayout(1, 1, 1, false)`.** The
  template is the n=1 instance of the same layout function, so the file's own
  header comment and the code cannot drift apart. Change the gap constants and
  the template has to be rebuilt to match.
- **Trace logging.** Every run writes its full computed geometry, every grid
  anchor and every generated feeder formula to the Apps Script execution log
  via `Logger`, prefixed `[sage]`. Layout bugs here are invisible in a
  finished tab unless you already know which row was expected.

### 13.6 Still not built

Phase 2 (§10.1 `SCHEDULE`) and Phase 3 (§10.2 `CSV`, `STANDINGSCSV`,
`Court Control`) are unchanged and unbuilt. A freshly generated workbook has
correct category tabs, `Variables`, `Title` and `Reference for Players`, but
its `SCHEDULE` and the tabs derived from it still carry whatever event the
master was copied from — so it is **not** ready to run an event as-is. The
completion toast says player names still need pasting, which understates
this.

Also untested against a real generation: the two-bracket **`semis`** path.
Every category in the reference CSV is single-bracket except `HIXD`
(`semis`) and `AMD` (`aggregate`), and the layout and formulas for `semis`
have been verified by computation but never by generating the tab.
