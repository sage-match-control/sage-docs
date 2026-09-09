# Spec — Standard Tournament Master workbook

Generate a **standard-format** tournament workbook from a Tournament Time
Calculator CSV export, the way `dual-meet-sheet-generator-spec.md` already
does for dual meets — instead of duplicating last event's facility
spreadsheet and find-replacing every category key, court label and match
number by hand.

> **Status: not built.** Nothing in this document exists yet. It specifies
> both the **SAGE Standard Tournament Master** workbook (the thing a copy is
> made of) and the bound Apps Script that fills a copy in.
>
> This is the "standard-tournament equivalent" the root `CLAUDE.md` and
> `bracket-generator-workbook-handoff-spec.md` §2 have both been waiting on.

| | |
| --- | --- |
| Where the code goes | `sage-tools-api/scripts/standard-generator.gs` (new file, §12.1) |
| How it ships | Bound Apps Script in the master workbook — paste, not deploy |
| New infra | none |
| New credentials | none |
| `sage-tools-api` change | none — no version bump, no Cloud Run deploy |
| Reuses | `sheet-generator.gs`'s pure helpers verbatim where the shape matches (§12.2) |

**Read the three dual-meet specs first.** This one inherits their delivery
story, their Apps Script mechanics and most of their vocabulary, and only
writes down what is *different*. Where a section here says "same as Phase N",
it means exactly that and is not restated.

---

## 0. Orientation — read this first if you have no context

**What this is.** SAGE runs pickleball tournaments. A **standard tournament**
is the ordinary open-entry shape: pairs enter a category, get drawn into
round-robin brackets, and the top finishers from each bracket advance into a
single-elimination playoff. It is not a dual meet — there are no clubs, no
cross-product group stage, and no fixed two-entrant playoff.

Each **facility, each day** runs off its own Google Sheets workbook. BKL Cup
2026 Day 2 had three: Main, Annex and Dreamcourts. That is the single biggest
structural difference from a dual meet, which is one workbook for one venue,
and §3 is about it.

**Vocabulary.** Same as the dual-meet specs, minus clubs, plus:

| Term | Means |
| --- | --- |
| category | A division, keyed `LI40MD`, `LI18XD`, … — one tab each |
| pair | Two players who play together; the atomic competitor |
| bracket | A round-robin pool inside a category. Sizes may be **uneven** |
| slot | One start time across all courts — two `SCHEDULE` rows |
| stage | One playoff round: `R32`, `R16`, `QF`, `SF`, `B`, `F` |
| playoff slot | One numbered seat in a stage, e.g. `SF_3` — gets its own team code |
| team code | `<KEY>_<i>` for a group pair, `<KEY>_<STAGE>_<k>` for a playoff slot |
| facility | One venue running one subset of the day's categories |

**Section-reference convention.** A bare `§n` means this document. A
reference into another spec always names it — "Phase 1 spec §11.1", "Phase 3
spec §5.4".

---

## 1. Sources

None of the workbook structure is in source control. Everything below was
read directly out of two played BKL Cup 2026 workbooks, exported as `.xlsx`
and unzipped (Phase 3 spec §1.1 has the technique):

| Workbook | File ID | What it gave |
| --- | --- | --- |
| BKL Day 2 · **Main** (LI40) | `1resKkD3e8kjxJEcQkruzKW8N93NFjuqgtmD9_hwYHQs` | 3 categories, 41 matches, 12 court blocks, the twice-to-beat case |
| BKL Day 2 · **Dreamcourts** (LI18) | `1H5aHgogxauIgbb3xoBSuLQjVZCCdZcI_9iBXUMp_9SI` | 3 categories, 84 matches, 15 court blocks, a 4-stage playoff ladder and two extra tabs |

Both are registered in `event-data/config/events.json` under
`bkl-cup-2026 → day3`, alongside a third (Annex) not read here.

**Read coverage.** Every tab of both workbooks was read as cell values plus
formulas, and both `<definedNames>` libraries in full. The `Poster` tab holds
only images and was not inspected further.

### 1.1 The reference events reconcile against the calculator exactly

Worth stating up front, because it is what licenses §5: the calculator's
`calcCategory()` reproduces both reference workbooks' match counts to the
match, from nothing but pair count, bracket count and `advance: 2`.

| Workbook | Category | Pairs | Brackets | RR | Playoff | Total |
| --- | --- | --- | --- | --- | --- | --- |
| Main | `LI40MD` | 9 | 2 (5-4) | 16 | 4 | 20 |
| Main | `LI40WD` | 4 | 1 | 6 | 2 (twice-to-beat) | 8 |
| Main | `LI40XD` | 7 | 2 (4-3) | 9 | 4 | 13 |
| | | | | | **Main total** | **41** |
| Dreamcourts | `LI18MD` | 15 | 3 (5-5-5) | 30 | 6 | 36 |
| Dreamcourts | `LI18WD` | 8 | 2 (4-4) | 12 | 4 | 16 |
| Dreamcourts | `LI18XD` | 16 | 4 (4-4-4-4) | 24 | 8 | 32 |
| | | | | | **Dreamcourts total** | **84** |

41 is `Timeline!C2` and the `CSV` row count in Main; 84 is the same in
Dreamcourts. The bracket sizes reconcile against the `bracket` column's own
`A`/`B`/`C`/`D` tallies (13/13/9/4 across the three Dreamcourts categories).

`LI18MD` is the one that proves the tiered-bye machinery: 3 brackets × top 2
= 6 qualifiers, which `tieredRounds([3,3])` reduces as **1 + 1 + 2 + 1**
matches plus a bronze — and the workbook has exactly two `R16` slots, two
`QF` slots, four `SF` slots, two `B` and two `F`. Even the *labels* match:
the calculator names rounds by walking back from the Final and doubling, so
a four-round ladder is `R16 · QF · SF · Final` regardless of how few pairs
are actually in the early rounds.

---

## 2. Why this is a second master, not a mode of the first

The category tabs run on **Named Functions** — workbook-level `LAMBDA`
definitions under `Data → Named functions`, in no repo, uncreatable through
the Sheets API. Phase 1 spec §1 works through why that forces
generate-from-a-copy plus bound script. All of it applies here unchanged.

What is new is that the standard workbooks carry a **different, older
library** — 15 functions where the Dual Meet Master has 23, and none of them
built on `STACKBLOCKS`:

```
main.xlsx        GETPLAYERSCOLUMN1 = LAMBDA({SCHEDULE!$E$6:$E ; …$M$6:$M ; … ; $CO$6:$CO})   12 blocks
dream.xlsx       GETPLAYERSCOLUMN1 = LAMBDA({SCHEDULE!$E$6:$E ; …$M$6:$M ; … ; $DM$6:$DM})   15 blocks
```

**The column list is hand-edited per workbook to match its court count.**
That is not a stylistic difference, it is the defect that makes a standard
generator worth building at all — and it has already bitten (§2.1). §11
replaces the whole library with the dual meet's, so a generated standard
workbook derives its court count from the sheet's own width like every dual
meet workbook already does.

Two masters rather than one, for the same reason there are two site
templates: the category tab is a genuinely different organism (§7), the
playoff model is inverted (§8), and one workbook covers one facility rather
than one event (§3). A shared master would be a mode switch on every tab.

### 2.1 What the hand-built workbooks get wrong

Recorded because each one is a requirement in disguise — the generator's job
is to make these unrepresentable, not to reproduce them.

| Defect | Where | Consequence |
| --- | --- | --- |
| `SCHEDULE` is 15 court blocks wide but `Variables!C5` says `courts: 6` | Dreamcourts | Every pace estimate on `Court Control` divides by the wrong number |
| Court labels repeat — `10 11 12 13 14 15 7 8 9 10 11 12 13 14 15` | Dreamcourts | `CSV!D` reports the wrong court for any match past block 9 |
| `CSV!C2`/`D2` scan a hardcoded `SCHEDULE!$F$6:$BZ$78` — 10 blocks | both | Silently wrong for courts 11+; only survived because 6 were used |
| Named-function column list (12) ≠ schedule width (15) | Dreamcourts | Courts 13-15 invisible to every standings formula |
| `Court Control!C42 = SUM(LI40MD!D:D)+SUM(LI40XD!D:D)+SUM(LI40WD!D:D)` | Main | Add a category, forget the sum, "matches done" goes quietly stale |
| `Reference for Players!A377 = FILTER(HIMD!…)` → `#REF!` | Main | A leftover anchor for a category this facility never ran |
| `Timeline` body is a 24-term `COUNTIFS` chain, one pair of terms per court | both | Unmaintainable; wrong the moment a court is added |
| `Variables!I38` reads `Low Intermediate 18+ Women's Doubles` on the `HI18WD` row | Main | Same class as the `LIMD`/`LIWD` typo Phase 1 spec §4.3 fixed structurally |
| A playoff block's `C` cell claims the winner's next code is `<KEY>_F_SF-1` | both | Not a real code — see §8.5 |

---

## 3. The unit of generation is a facility-day, not an event

A dual meet is one workbook. A standard tournament is **one workbook per
facility per day**, and the plan CSV describes the whole event. Three
consequences, all of which the generator has to be told about because the
CSV cannot carry them:

**Categories are assigned to facilities by hand.** BKL Day 2 put the LI40
categories on Main and the LI18 categories on Dreamcourts. Nothing derives
that. The operator picks.

**Court numbering is facility-local and non-contiguous.** Main ran courts
1-12, Dreamcourts 7-15. The label is free text in `SCHEDULE!` row 5 and is
what `CSV!D` publishes, so it has to be an input, not `"Court " & c`.

**Match numbers are facility-scoped and offset.** Main starts at `1001`,
Dreamcourts at `3001`, Annex (by the same rule) at `2001`. Three facilities'
`CSV` tabs merge into one day snapshot in `event-data`, so a collision would
silently overwrite a match. The offset is a required input.

The full plan for every category of every day still goes into
`Variables!H:P` (§6.3) — the workbook documents the whole event and generates
tabs for its own slice.

---

## 4. Input — the plan CSV in standard format

Same file as Phase 1 spec §2, same columns:

```
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format
```

with the standard-format columns live and the dual ones inert. The mirror
image of Phase 1 spec §2.1:

| Column | Standard format |
| --- | --- |
| `teams` | **Live.** Total pairs in the category |
| `brackets` | **Live.** Number of round-robin pools (`c.group`, 1-30) |
| `advance` | **Live.** Top `k` per bracket qualify, 1-4 |
| `fill` | **Live.** `bye` or `wc` — how a short playoff field is filled (§5.4) |
| `note` | Free text, straight to `Variables!O` |
| `teams_a`, `teams_b` | **Always ignore.** The calculator still writes them (`c.teamsA`/`c.teamsB` persist across a format switch), and in standard format they are meaningless leftovers |
| `po_format` | **Always ignore.** Written empty in standard format by `exportCSV`; older exports may carry `semis` |

`setting format` must read `standard`. A `dual` file is rejected, not
approximated — the exact converse of Phase 1 spec §3.

Settings used: `title`, `date`, `start`, `duration_min`, `buffer_min`.
`courts` is read but **overridden** by the per-facility court list (§6.1),
since the calculator's number is the event's total across all venues.

---

## 5. Every playoff shape the calculator can produce

This is the enumeration the generator's playoff builder has to cover. All of
it is `playoffPlan(groups, adv, fill)` and its helpers in
`tournament-calculator.html`; the branch order below is the code's own.

### 5.1 `teams == 2` — best-of-3, no round robin

`calcCategory` short-circuits before `groupSizes` is ever called.

```
plan = { rounds: [{label:'Final · best of 3', matches:3, byes:0}], total:3, min:2, bestOf3:true }
```

No group stage, no bracket, no bronze. Three match slots, of which the third
is played only if the series splits 1-1 — the plan counts the worst case, so
the workbook must carry three.

### 5.2 `brackets == 1` — round robin then a twice-to-beat final

```
plan = { rounds: [{label:'Final · twice-to-beat', matches:2}], single:true,
         qualifiers:2, bronzeText:'RR #3 automatic' }
```

RR #1 needs one win; RR #2 must win twice. Two match slots, and **bronze is
not a match** — the third-placed pair takes it on record.

Verified in Main's `LI40WD` (4 pairs, 6 RR + 2). Its codes are the ones to
copy: `LI40WD_F_1_(1)`, `LI40WD_F_2_(1)` for game 1 and
`LI40WD_F_1_(2)`, `LI40WD_F_2_(2)` for game 2. The `(n)` suffix is already
understood by the site — `matchInstanceOf()` in the standard template parses
it and `roundLabel()` renders "Final · Match 2".

### 5.3 `brackets >= 2 && brackets * advance == 2` — a bare final

Two brackets, top 1 each. `playoffPlan` returns early:

```
plan = { rounds:[{label:'Final', matches:1}], qualifiers:2, bronzeText:null }
```

**No bronze at all**, not even an automatic one. Worth calling out because
every other multi-bracket shape has one.

### 5.4 `brackets >= 2` — the tiered single-elimination ladder

The general case, and the one with the machinery. `tieredRounds(Array(adv).fill(groups))`:

- **`advance == 1`** — one homogeneous tier of `groups` entrants, reduced by
  `reduceToTarget` to 2 and then a final. Byes by lot, and they occur only in
  the single earliest round.
- **`advance >= 2`** — tiered. All group **winners** outrank all runners-up,
  which outrank 3rd places. Winners bye all the way to round
  `R = smallest power of 2 strictly greater than groups`, skipping every
  earlier round entirely. The remaining `R - groups` seats at that round are
  played for by the lower tiers, flattened into one pool, through their own
  preliminary round(s).

Round labels are assigned **backwards from the Final**, doubling: the last
round is `Final`, then `Semifinals`, `Quarterfinals`, `Round of 16`, `Round
of 32`. The label describes the round's structural position, not how many
pairs are in it — which is why `LI18MD`'s ladder of 1, 1, 2, 1 matches is
labelled `R16 · QF · SF · Final`.

**Bronze**: a `Bronze match` round is spliced in before the Final when
`qualifiers >= 4`; below that `bronzeText` is `'losing semifinalist
automatic'` and there is no match.

**`fill`**: `bye` leaves the first round's byes as byes. `wc` converts every
bye into a wildcard entrant — `matches += byes; byes = 0` — so the field
grows and the bracket has no byes at all. This changes the entrant count, so
it changes the number of playoff slots the tab must carry.

### 5.5 The shapes, as a table

| `teams` | `brackets` | `advance` | Shape | Playoff matches | Bronze |
| --- | --- | --- | --- | --- | --- |
| 2 | — | — | best of 3 | 3 (min 2) | none |
| any | 1 | — | RR + twice-to-beat | 2 (min 1) | RR #3, no match |
| any | 2 | 1 | RR + Final | 1 | **none** |
| any | ≥2 | 1 | RR + flat single-elim, byes by lot | `reduceToTarget(groups,2)` + 1 | match iff `groups >= 4` |
| any | ≥2 | ≥2 | RR + tiered single-elim | `tieredRounds` | match iff `groups*adv + wc >= 4` |

### 5.6 Bracket sizes are uneven, and that is load-bearing

`groupSizes(teams, brackets)` splits as evenly as possible and **enlarges the
first brackets** with the remainder: 26 pairs over 6 brackets is `5-5-4-4-4-4`.
Real: `LI40MD` is `5-4`, `LI40XD` is `4-3`.

Everything downstream has to stop assuming a square block:

- Round-robin matches are `Σ n_b(n_b - 1)/2`, not one formula in one `n`.
- A group block is `2 · n_b` rows, and the four blocks of a category are
  different heights.
- The completion gate for a bracket is `n_b(n_b - 1)/2` matches, i.e.
  `SUM(J) = n_b(n_b - 1)` across the block.
- The `bracket` column cannot be `ceil(i / n)` the way Phase 3 spec §6.3
  computes it. §10.4 replaces it.

`groupSizes` also floors the bracket count at `floor(teams/2)` — every
bracket needs at least 2 pairs — and the calculator emits a note when it has
to reduce. The generator must apply the same reduction rather than trusting
`brackets` blindly, or a 5-pair, 4-bracket category generates four blocks and
the calculator planned three.

Likewise `advance` is capped at the **smallest** bracket's size
(`adv = min(requestedAdv, min(sizes))`), so a category with a bracket of 2
cannot advance 3.

---

## 6. Inputs the CSV does not carry

A sidebar, same pattern as the dual-meet generator's:

| Field | Default | Used by |
| --- | --- | --- |
| Facility label | empty | `Variables!C8`, `Title!B16` — e.g. `PCPH Main` |
| Court labels | — | §6.1 |
| First match number | `1001` | §6.2 |
| Categories in this workbook | none selected | Which tabs get built (§3) |
| Day label | empty | `Variables!P` on the selected rows |
| Event key | slugified `setting title` | `Title!B17` |

### 6.1 Court labels

A comma-separated list, in schedule order: `1,2,3,4` or `7,8,9,10,11,12`.
Its **length is the court count** for this workbook — `Variables!C5`, the
`SCHEDULE` width, the `Court Control` block count and the pace divisor all
come from it, so the Dreamcourts mismatch in §2.1 cannot recur.

Labels render as `Court <label>` in `SCHEDULE!` row 5 and are what `CSV!D`
publishes. Duplicates are rejected (§9).

### 6.2 First match number

The base for this workbook's match numbering. Numbers run densely from it.
The convention the reference event used — facility index × 1000, plus 1 — is
a **suggestion in the field's help text, not a rule the generator enforces**;
nothing in `events.json` orders facilities, so the generator cannot derive
it. It *can* and must check that the range does not collide with any other
facility's, which it cannot see — so instead it states the range it used in
the completion toast, loudly, for the operator to check against the other
workbooks.

### 6.3 Categories, and the plan block

`Variables!H:P` gets **every** category row from the CSV, plus a `day` column
(`P`) and a `facility` column (`Q`, new — §10.1) that the generator fills in
for the rows it selected and leaves blank for the rest. Tabs are built only
for the selected keys, in CSV order.

That is what the reference workbooks do (Main's `Variables` lists all 40-odd
categories of the whole five-day event while carrying three tabs), and it is
worth keeping: `B1` on each category tab reads its display name back out of
that block by key, and the operator can see at a glance which slice of the
event this workbook is.

---

## 7. The category tab

### 7.1 Column map

Verified cell by cell against `LI40MD` and `LI18MD`. Columns never move; only
row counts vary.

| Col | Holds |
| --- | --- |
| `A` | Team code — **repeated on both rows** of a pair |
| `B` | Player name — a lookup on the pair's first row, mirrored on the second |
| `C` | Seed/marker text (`!`, `!!`, `!!!`) — hand-typed, generator leaves blank |
| `D` `E` | W, L |
| `F` `G` | Points for, points against |
| `H` | Quotient |
| `I` | Free-text note (`Replaced Guy Godoy`) — hand-typed |
| `J` | Matches Done, `=D+E` — group rows only |
| `K` | **Bracket** on group rows; **Next Round** and the podium label on playoff rows (§7.5) |
| `L` | Bracket rank — hand-typed |
| `N` | Score-grid row labels |
| `O`–… | Score-grid body (§7.4) |
| `AB` | Notes column header for the roster scaffold |
| `AC` `AD` `AE` | Roster scaffold: pair index, **STEP 1 · NAMES**, code link |
| `AG` `AH` `AI` | Roster scaffold: number, **STEP 2 · CODES**, **STEP 3 · RANDOMIZED CODES** |
| `AM` `AN` `AO` `AP` | Qualifier scaffold — the playoff draw (§7.6) |
| `AT` `AU` `AV` | Playoff slot table (§7.6) |
| `BA` `BB` `BC` | Awards block — Gold / Silver / Bronze (§7.7). **Kept**, unlike the dual meet |

Header chrome: `A1` is the raw key; `B1 = FILTER(Variables!J:J, Variables!I:I=A1)`
is the display name; `AD1 = A1` is what every scaffold formula concatenates
against. Rows 2-4 carry the `STEP 1/2/3` labels and the scaffold headers;
row 5 carries `W`, `L`, `Team Scr`, `Opp\nScr`, `Q`, `Matches Done`,
`Bracket`, `Br Rank`. Group pairs start at row 6.

### 7.2 Group blocks — one continuous list, brackets marked in `K`

Unlike a dual meet, **the brackets of a category are not separate blocks**.
All `teams` pairs run continuously from row 6, two rows each, and bracket
membership is carried only by column `K`: `LI40MD` has `K = 1` on pairs 1-5
and `K = 2` on pairs 6-9, with no gap, no second header row and no banner
between them.

Keep that. It makes the block geometry independent of the bracket split, and
it means codes stay `<KEY>_1 … <KEY>_<teams>` in one unbroken run — which is
what the roster scaffold, the `Reference for Players` spill and the bracket
generator's own output all assume.

Per-pair formulas, filled down:

```
A6  = A$1&"_"&Variables!AB91          LI40MD_1     (the reference-number ladder, §7.3)
A7  = A6                              second row of the pair
B6  = IFNA(FILTER($AD$5:$AD178,$AE$5:$AE178=A6),A6)
D6  = GETTOTALWINS(A6)      E6 = GETTOTALLOSES(A6)
F6  = GETTOTALSCORE(A6)     G6 = GETTOTALOPPONENTSCORE(A6)
H6  = GETSCOREQUOTIENT(A6)  J6 = D6+E6
K6  = <bracket number>                literal, written by the generator
```

`D`–`H` and `J` are merged vertically across the pair's two rows. `K` is
merged on group rows; on playoff rows it is **not**, because its two rows
hold different formulas (§7.5) — the same rule Phase 1 spec §4.0.4 states for
column `I` there.

### 7.3 `Variables!AB` is a counter, not a draw

`A6 = A$1&"_"&Variables!AB91`, `A8 = A$1&"_"&Variables!AB93`, and so on. The
`AB` column is a ladder — `AB91 = 1`, `AB93 = AB91+1`, every second row — so
the effect is simply `_1`, `_2`, `_3`. It is the standard workbook's
counterpart to the dual meet's `Variables!A2` ladder, spaced to the two-row
pair stride so a fill-down works.

It looks like an indirection worth removing and it is not: the `Standings`
tab (§10.6) concatenates against the *same* ladder cells, so ten categories'
codes stay row-aligned across that board for free.

### 7.4 The score grid

One grid per bracket, anchored on that bracket's first pair row. For a
bracket of `n_b` pairs whose first pair row is `first`:

| Cell | Content |
| --- | --- |
| `O<first>` … `<n_b`-th col`><first>` | `=N<first+1>` … `=N<first+n_b>` — the column headers |
| `N<first+1+k>` | `=A<first + 2k>` — one row per pair, **not** two |
| body `(r, c)` | `=GETSCOREAGAINSTPAIR($N<r>, <c>$<first>)`, diagonal left blank |

Verified at `n_b = 5` (`O:S`) and `n_b = 4` (`O:R`). Note the offset from the
dual meet: labels are in `N` and the body starts at `O`, where the dual meet
uses `M` and `N`.

Both triangles are filled. Unlike a dual meet's cross-club grid, a
round-robin grid is genuinely two-sided — `(i,j)` is i's score against j and
`(j,i)` is j's — so a `GETSCOREAGAINSTPAIR` goes in every off-diagonal cell.

> Several cells in the reference grids are hardcoded numbers rather than
> formulas, from operators typing over them mid-event. Do not read that as
> structure.

Playoff blocks carry a grid too, on the same mechanism, condensing each
entrant's two rows into one grid row exactly as Phase 1 spec §13.2 describes:

```
N102 = A101       the block label, e.g. "SF-1"
O102 = N103       P102 = N104          entrant headers
N103 = A103       N104 = A105          one grid row per entrant, pointing at its SECOND row
P103 = GETSCOREAGAINSTPAIR($N103,P102)
O104 = GETSCOREAGAINSTPAIR($N104,O102)
```

The grid's header row is `block.header + 1`, one below the block's `A`-column
label.

### 7.5 The playoff ladder — forward propagation

**This is the inversion.** A dual meet's playoff entrants are pulled
backwards by feeder formulas that sort a group block (`INDEX(SORTBYWINS(…),1)`).
A standard tournament's are pushed **forwards**: each playoff block computes
where its winner goes, writes that code into column `K`, and the next round's
block finds its own occupants by searching the previous block's `K` for its
own code.

It has to work that way. A dual meet's bronze and final entrants are a
deterministic function of two group blocks. A standard ladder's are a
function of a draw plus a chain of results with byes in it — there is no
range for `SORTBYWINS` to sort.

**Stage banner.** One row per stage, carrying the stage keyword in `A` and
the *next* stage in `K`:

```
A99 : SF        H99 : "before:"   I99 : QF     J99 : "next:"   K99 : F
```

**Per-match sub-block**, two entrants, four rows, one per match in the stage:

```
A101 = A99&"-1"                             SF-1              block label
B101 : "winner's next code ->"
C101 = <the next round's slot cell>         see the note below
K101 : "Next Round"
D101..H101 : W / L / Team Scr / Opp Scr / Q headers

A102 = A$1&"_"&A99&"_1"                     LI40MD_SF_1
B102 = <name lookup, see below>
D102..H102 = GETTOTAL*/GETSCOREQUOTIENT(A102)
K102 = <advance formula, see below>
A103 = A102   B103 = mirror   K103 = K102
A104 = A$1&"_"&A99&"_2"                     LI40MD_SF_2
… same shape
```

**Name lookup** — two forms, and which one a slot gets is the whole point:

| Slot is | `B` formula |
| --- | --- |
| **fresh** — drawn in from the group stage, or byeing into this round | `=IFNA(FILTER(AO:AO, AP:AP=A<r>), A<r>)` — the qualifier scaffold (§7.6) |
| **fed** — the winner of a previous-round match | `=IFNA(FILTER(B<prevLo>:B<prevHi>, K<prevLo>:K<prevHi>=A<r>), A<r>)` |

A single stage mixes both. `LI18MD`'s `QF-2` has `QF_3` fed from `R16` and
`QF_4` fresh — which is precisely §5.4's tiered bye, rendered in formulas.

**Advance formula** in `K`, three variants:

```
early rounds   K69  = IF(D69>0, C68, "-")                              winner only
semifinals     K111 = IFNA(IFS(D111>0, A131, E111>0, A125), "-")       winner -> F, loser -> B
bronze         K116 = IF(D116>0, "BRONZE", "-")
final          K122 = IFNA(IFS(D122>0, "GOLD", E122>0, "SILVER"), "-")
```

The second row of each entrant mirrors: `K103 = K102`.

> **The `C` cell is inconsistent in the reference and the generator must pick
> one form.** Early-round blocks point it at the real destination cell
> (`C68 = A91`, resolving to `LI18MD_QF_3`), which is what `K` then uses.
> Semifinal blocks instead synthesise a label — `C110 = $A$1&"_"&K108&"_"&A110`
> → `LI40MD_F_SF-1` — which is **not a team code that exists anywhere**. It is
> harmless only because the SF's own `K` formula ignores `C` and names the
> destination cells directly. Generate the early-round form everywhere.

**Finals stage.** A `FINALS` banner row, then a `B` block and an `F` block,
each two entrants, whose `B` lookups read the semifinal blocks' `K` range.
Codes are `<KEY>_B_1` / `<KEY>_B_2` and `<KEY>_F_1` / `<KEY>_F_2`.

### 7.6 Two scaffolds, two draws

The roster scaffold (`AC`–`AI`) is the same three-step shape as the dual
meet's, one column set instead of two:

| Step | Col | State on a fresh workbook |
| --- | --- | --- |
| index | `AC` | `1 … teams`, one per pair, generator-written |
| **STEP 1 · NAMES** | `AD` | blank — operator pastes, 2 rows per pair |
| link | `AE` | `=FILTER($AI$5:$AI<end>, $AG$5:$AG<end>=AC<r>)`, mirrored on the second row |
| number | `AG` | `1 … teams`, generator-written |
| **STEP 2 · CODES** | `AH` | `=$AD$1&"_"&AG<r>` |
| **STEP 3 · RANDOMIZED** | `AI` | **blank** — operator pastes the codes back shuffled |

STEP 3 stays blank for the reason `sheet-generator.gs:1869` gives and
`bracket-generator-workbook-handoff-spec.md` §1 quotes: pre-seeding it makes
an undone step look done, and an unshuffled STEP 3 maps every pair to its own
roster slot, defeating the blinding.

The **qualifier scaffold** (`AM`–`AP`) is the second draw, and has no
dual-meet equivalent — a dual meet's playoff entrants are decided by record,
so there is nothing to draw. Here the group-stage qualifiers are drawn by lot
into the playoff slots ("PLACE 1ST PLAYOFFS BUNUTAN HERE" in the reference):

| Col | Holds |
| --- | --- |
| `AM` | The **drawn slot number** for this qualifier, or `-` if this ordinal has no slot — operator-written |
| `AN` | Qualifier ordinal, `=Variables!AB<row>` (§7.3) |
| `AO` | Qualifier player names, 2 rows per pair — operator-written |
| `AP` | `=IFNA(FILTER(AV:AV, AT:AT=AM<r>), "-")`, mirrored on the second row |

and the **playoff slot table** (`AT`–`AV`) is what `AP` resolves against:

| Col | Holds |
| --- | --- |
| `AT` | Slot index, `1 … slots` |
| `AU` | The stage the slot belongs to — `SF`, `QF`, `R16`, `R32` |
| `AV` | `=$AD$1&"_"&AU<r>&"_"&AT<r>` |

**The generator writes `AU`.** It is the one column that encodes the whole
tiered structure: the reference `LI40MD` has `SF SF SF SF QF QF QF QF R32 …`
running down it, and that ordering is exactly what §5.4 computes. Leaving it
to the operator is what makes today's build error-prone.

Note that `AV` numbers slots **globally across stages** — `LI40MD_QF_5`
follows `LI40MD_SF_4` — rather than restarting at 1 per stage. Preserve that:
the site's `roundKeyword()` matches on the keyword, not the number, and
`Reference for Players` sorts naturally.

### 7.7 The Awards block stays

`BA:BC`, three rows, reading the tab's own podium out of column `K`:

```
BA4 : "Gold"      BB4 = IFNA(FILTER(A:A, K:K=BA4), "-")    BC4 = GETPLAYERNAMESBYTEAMCODE(BB4)
BA6 : "Silver"    BB6 = IFNA(FILTER(A:A, K:K=BA6), "-")    BC6 = …
BA8 : "Bronze"    BB8 = IFNA(FILTER(A:A, K:K=BA8), "-")    BC8 = …
```

Phase 1 spec §4.2 **deletes** this block from the dual-meet template. Do not
carry that decision across: here it has a live consumer, the `Awards` tab
(§10.7), which reads `BB4`/`BC4`/`BB6`/… through `INDIRECT` on the category
key. The dual meet could delete it because the console's Awards tab derives
the podium from the day snapshot instead.

Fix the same typo Phase 1 spec §4.2 found, though — the reference's Silver
cell reads `=IFNA(IFS(D35,A33,D33,A735),"-")` in some tabs, where `A735` is a
typo for `A35`. The `FILTER` form above is the one to generate.

**The bronze case that has no bronze match.** Under §5.2 nothing ever writes
`"BRONZE"` into a playoff row's `K`, because there is no bronze match — the
third-placed group pair takes it on record. The reference handles this by the
operator typing `Bronze` into that pair's **group-row `K`**, the same column
that otherwise holds the bracket number. That overload works but is invisible
and undocumented; §14.2 records the decision to keep it, with the generator
writing an explicit `Bronze:` prompt into `I` on the finals banner row.

---

## 8. Team code grammar

One category key, one underscore-delimited suffix. No club segment.

| Slot | Code | Count |
| --- | --- | --- |
| Group pair | `<KEY>_<i>`, `i = 1 … teams` | `teams` |
| Playoff slot | `<KEY>_<STAGE>_<k>` where `STAGE ∈ {R32, R16, QF, SF}` and `k` is the global slot index | 2 per playoff match |
| Bronze | `<KEY>_B_1`, `<KEY>_B_2` | 2 |
| Final | `<KEY>_F_1`, `<KEY>_F_2` | 2 |
| Twice-to-beat final | `<KEY>_F_1_(g)`, `<KEY>_F_2_(g)` for `g = 1, 2` | 4 |
| Best-of-3 final | `<KEY>_F_1_(g)`, `<KEY>_F_2_(g)` for `g = 1, 2, 3` | 6 |

**Total codes per category = `teams` + 2 × playoff matches.** Verified across
all six reference categories, including the twice-to-beat one.

Every form above is already parsed by the standard-tournament site template —
`roundKeyword()` matches `R16`/`QF`/`SF`/`F`/`B`, `matchInstanceOf()` reads
the `(g)` suffix, and `STAGE_ORDER` is `['R16','QF','SF','BRONZE','FINAL']`.
**`R32` is in the code grammar but not in `STAGE_ORDER` or
`ROUND_KEYWORD_LABELS`**; a category deep enough to need it renders its R32
rows under an unsorted trailing stage. That is a site fix, tracked in §15,
not a reason to rename the codes.

---

## 9. Validation

All checks run **before** anything is written, and every failure is reported
at once — Phase 1 spec §3's rule, and Phase 1 spec §9.2's abort-on-collision
policy, both unchanged.

| Rule | Reason |
| --- | --- |
| `setting format == standard` | A dual file describes a different geometry |
| `key` unique, `[A-Z0-9]{2,8}`, legal tab name | Tab name is the key |
| at least one category selected | — |
| `teams` in `2..100` | The calculator's own bound |
| `brackets` reduced to `floor(teams/2)` if larger, with a warning | Matches `groupSizes` (§5.6) |
| `advance` capped at the smallest bracket, with a warning | Matches `calcCategory` (§5.6) |
| total playoff entrants ≤ 32 | The `Charts` reference tree tops out at 32 (§10.8), and no real category has come close |
| court labels non-empty, unique, ≤ 24 | §6.1, and the Dreamcourts defect in §2.1 |
| `firstMatchNumber` a positive integer | §6.2 |
| `courts <= Σ over categories of (matches in one RR round)` | Phase 2 spec §4.4's no-double-booking guarantee, restated for round robins (§10.2) |
| no existing tab collides | Phase 1 spec §9.2 |
| every rebuilt-in-place tab is pristine | Phase 3 spec §8.2 |

A failure aborts with the offending rows named and the workbook untouched.
Emitting a tab whose geometry does not match the plan it came from is the
failure this section exists to prevent: it produces plausible-looking wrong
medalists rather than an obvious error.

---

## 10. The other tabs

### 10.1 `Variables`

| Cells | Value |
| --- | --- |
| `A2:C8` | `setting` rows: `title`, `date`, `start`, `courts`, `duration_min`, `buffer_min`, `location` |
| `C5` | **court count from the label list** (§6.1), not the CSV's `courts` |
| `C6` | **slot pitch** — `(duration_min + buffer_min)/1440`, per Phase 3 spec §4.1, labelled "Slot Pitch" |
| `C8` | Facility label |
| `H:P` | The full plan block — `type, key, value, teams, brackets, advance, fill, note, day` |
| `Q` | **New — `facility`.** Filled for the rows this workbook generated tabs for, blank otherwise |
| `R` | **New — court labels**, one per row from `R2`, so `SCHEDULE`'s row 5 can be a formula rather than 12 literals |
| `S` | **New — the bracket map** (§10.4): every group code, one per row |
| `T` | **New —** the bracket letter for the code in `S` |
| `AB90…` | The reference-number ladder (§7.3), every second row |

`H:P` is written from the CSV verbatim, which structurally fixes the
`HI18WD`-labelled-as-Low-Intermediate defect in §2.1 the same way Phase 1
spec §4.3 fixed its counterpart.

### 10.2 `SCHEDULE`

Geometry is **identical to the dual meet's** — Phase 2 spec §2 in full: rows
1-4 blank, row 5 the header band, data from row 6, a repeating 8-column court
block from column `D`, a slot is two rows, width is `8 · courts + 2`, unused
courts carry `-` in `E`/`F`/`G`, and the grid runs one row-pair past the last
slot to carry the end time (Phase 3 spec §3.2).

Four differences:

**Rows 1-3 carry a title band** — `=Title!$B$15`, `=Title!$B$16`, and the
literal `Powered by SAGE Match Control Experts` — repeated at a second
column part-way across so it stays visible when scrolled. Cosmetic, but it is
what the venue screenshots.

**Row 7 of each slot carries `vs`** at the match-number column's offset,
between the two name rows.

**Match numbers chain rather than being written.** `F6` is the literal first
match number and every subsequent one is `= <previous block's F cell> + 1`,
reading left to right and wrapping down. That is worth keeping: renumbering
after an insert is then a single edit.

**Court labels come from `Variables!R`**, so row 5 reads
`="Court "&Variables!R2` rather than a literal.

**The match list** is the piece with no dual-meet counterpart. Phase 2 spec
§4's cross-product rotation does not apply; a standard group stage is a
round robin *within* each bracket.

- Generate each bracket's rounds by the **circle method**: fix pair 1, rotate
  the rest; a bracket of odd `n_b` gets a phantom entrant, and whoever draws
  it sits that round out. `n_b` (or `n_b - 1` if even) rounds, each with at
  most `floor(n_b/2)` matches, and within a round no pair appears twice.
- **Interleave by round across brackets and categories**, exactly as Phase 2
  spec §4.2 does — rounds outer, categories inner, brackets inner to that —
  so a pair's own matches stay far apart.
- Playoff stages each **start a fresh slot** and are emitted stage by stage
  in `STAGE_ORDER`, categories inner. A stage cannot overlap the one before
  it: its entrants are the previous stage's winners.
- Partial slots are **centred**, per Phase 2 spec §4.3.
- Match numbers are the position in placement order, offset by
  `firstMatchNumber - 1`.

Hard constraint: no pair twice in one slot. Soft preference, not enforced:
at least one idle slot between a pair's matches — the reference violates it
freely (`LI40MD_1` plays slots 1 and 2) and no operator has asked for it.

**Category colours** carry over from Phase 2 spec §6 unchanged, including the
conditional-format mechanism and the `INDEX(SPLIT(code,"_"),2)` test — which
happens to be *simpler* here, since a standard code's second token is already
the key with no club segment to skip. The level ladder in Phase 2 spec §6.2
needs two rungs added for the BKL level codes (`B` beginner, `AB` advanced
beginner) and the age-band keys (`LI40MD` → level `LI40`, type `MD`) mean
§6.4's parse has to strip a trailing digit run from the level before looking
it up. A key that still does not resolve gets no fill, per that section.

### 10.3 `Court Control`

Court blocks are the dual meet's: 3 rows each from row 5, `B` the court
number, `C` the operator-typed match number, `D`/`E` the codes via
`GETTEAM1CODEBYMATCH`/`GETTEAM2CODEBYMATCH`, and the names spilling below.

Everything else differs:

- **The stats block sits below the court blocks**, not beside them, so both
  regions grow downward and the stats block's row is
  `3 · courts + <gap>`. (Phase 3 spec §5.1's fixed `G:M` rows 5-10 is a
  dual-meet-only arrangement.)
- `Court Control!C41` reads total matches from `Timeline!$C$2`.
- **Matches done must become `=SUM(STANDINGSCSV!$D:$D)`**, replacing the
  hand-listed per-category sum in §2.1.
- The pace maths is otherwise Phase 3 spec §5.1's, reading `Variables!$C$6`
  (pitch) and `Variables!$C$5` (courts), with `Blank Slots` / `Blank Slots
  Done` beside it.
- Below the court blocks the reference carries a **standby strip** — `Next
  Stand By`, `TBA Stand By`, and six `Match to Record` rows, each the same
  three-row `GETTEAM1CODEBYMATCH` block with no court number. Keep it; it is
  how the desk queues the next matches, and it is fixed-size.

### 10.4 `STANDINGSCSV`

Phase 3 spec §5.4's seven columns, unchanged, with one column that needs a
new derivation.

```
A2 = UNIQUE(FILTER('Reference for Players'!$A$3:$A,
                   REGEXMATCH('Reference for Players'!$A$3:$A, "_")))
B2 = INDEX(GETPLAYERNAMESBYTEAMCODE($A2),1)      C2 = …,2)
D2 = GETTOTALWINS($A2)   E2 = GETTOTALLOSES($A2)   F2 = ROUND(GETSCOREQUOTIENT($A2),4)
```

**`G` — the bracket.** Phase 3 spec §6.3 computes `ceil(i / n)` from a single
`n`. Uneven bracket sizes (§5.6) make that wrong, so it becomes a lookup into
the map the generator writes at `Variables!S:T`:

```
G2 = IF($A2="", "", IFERROR(VLOOKUP($A2, Variables!$S:$T, 2, FALSE), ""))
```

A playoff code is not in the map, `VLOOKUP` fails, `IFERROR` blanks it —
correct, since a bronze entrant belongs to no bracket. The letters are
`A`, `B`, `C`, … matching the reference and the bracket generator's own
lettered cards; the dual meet's numeric `1`/`2` is a different convention and
both are fine by the site, which treats the column as an opaque grouping
label.

Row count is `Σ (teams + 2 × playoff matches)` (§8) — 75 for Dreamcourts, 41
pairs' worth plus playoff slots for Main.

### 10.5 `CSV`

Phase 3 spec §5.3's tab plus one column: the reference has a literal `v`
between the two teams at column `I`, pushing `teamCode2` to `J` and the
second team's cells to `K`/`L`/`M`. **13 columns, not 12.** Keep it — the
`GvizCsvFetcher` reads by header name, and the column is what makes the tab
readable to a human scanning it mid-event.

| Col | Header | Row 2 |
| --- | --- | --- |
| `A` | `matchNumber` | literal, `firstMatchNumber …` |
| `B` | `court` | `=IFNA(FILTER('Court Control'!$B$5:$B$<lastCourtRow>,'Court Control'!$C$5:$C$<lastCourtRow>=$A2),"")` |
| `C` | `Schedule` | `=MATCHTIME($A2)` |
| `D` | `CourtAssignment` | `=MATCHCOURT($A2)` |
| `E` | `teamCode1` | `=GETTEAM1CODEBYMATCH($A2)` |
| `F` `G` | `team1Player1/2` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($E2),1)` / `,2)` |
| `H` | `team1Score` | `=GETSCOREAGAINSTPAIR($E2,$J2)` |
| `I` | `v` | literal `v` |
| `J` | `teamCode2` | `=GETTEAM2CODEBYMATCH($A2)` |
| `K` `L` | `team2Player1/2` | as `F`/`G` off `$J2` |
| `M` | `team2Score` | `=GETSCOREAGAINSTPAIR($J2,$E2)` |

`C` and `D` **must** become the `MATCHTIME`/`MATCHCOURT` named functions
(§11). The reference's `BYROW`/`BYCOL` scan over a hardcoded
`SCHEDULE!$F$6:$BZ$78` is the third defect in §2.1 and cannot survive a court
count it was not typed for.

`MATCHCOURT` returns `"Court " & <block index>`, which is the block's
*position*, not its label. With non-contiguous court labels (§6.1) that is
wrong: Dreamcourts' block 1 is Court 10. Redefine it against `Variables!R`:

```
MATCHCOURT = LAMBDA(match_number, IFERROR(
  LET(h,   ROWS(SCHEDULE!$B$6:$B),
      pos, MATCH(match_number, GETMATCHNUMBERS(), 0),
      "Court " & INDEX(Variables!$R:$R, INT((pos - 1) / h) + 2)),
  "Not found"))
```

### 10.6 `Timeline`, `Timeline (Individual)` and `Standings`

**`Timeline`** is Phase 3 spec §5.2's pair × slot conflict grid, with the
columns shuffled: `B` is `TOTAL`, `C` is the spilled label column, and the
time headers run from `D`. Its body must become `=COUNTPAIRAT(D$2,$C3)`,
replacing the 24-term `COUNTIFS` chain in §2.1. `C2 = SUM($D$3:<last>)/2` is
the match count, and the `/2` is because every match appears twice.

**`Timeline (Individual)`** has no dual-meet counterpart and is required
here. It is the same idea **transposed and keyed by player**: row 3 is a
transposed `UNIQUE` spill of `'Reference for Players'!B&" "&C`, column `A` is
the slot times, and the body is

```
=COUNTIF(SCHEDULE!<slotRow1>:<slotRow1>, B$3) + COUNTIF(SCHEDULE!<slotRow2>:<slotRow2>, B$3)
```

with a per-player `COUNTIF(<column>, ">1")` conflict tally on row 2.

It exists because `Timeline` **cannot** catch the conflict that actually
happens. A pair's playoff entry gets a *new* team code (§8), so a human
playing `LI18MD_4`'s last group match and `LI18MD_SF_2` in the same slot
shows as two different rows, each reading 1. Only the name matches. A dual
meet has the same code change but a much thinner playoff, and its schedule is
generated in one pass from a cross product; a standard tournament's playoff
entrants are drawn, so this is the check that catches a bad draw.

Whole-row `COUNTIF` makes it court-count-independent, which is worth
preserving over anything cleverer.

**`Standings`** is a wall-display board: up to ten categories side by side in
an 8-column period from `E`, each `Br / code / names / RR / W / L / Q`, with
`E4` etc. holding the category key and the codes built by concatenating
against the same `Variables!AB` ladder (§7.3). Present in Dreamcourts, absent
from Main. Generate it for every workbook — the reference's ten fixed slots
become `categories` slots — and read the display name with the same
`FILTER(Variables!$J:$J, Variables!$I:$I = <key>)` the category tabs use.

### 10.7 `Awards`, `Title`, `Reference for Players`, `QA Checklist`, `Poster`

**`Awards`** — one block per category, three podium rows each, reading the
category tab's own `BA:BC` block (§7.7) through `INDIRECT`:

```
F10 : LI40MD                                     the key, generator-written
F9  = FILTER(Variables!$J:$J, Variables!$I:$I=F10)
G11 = indirect("'"&F10&"'!BB4")     H11 = indirect("'"&F10&"'!BC4")     Gold
G13 = indirect("'"&F10&"'!BB6")     H13 = …!BC6")                       Silver
G15 = indirect("'"&F10&"'!BB8")     H15 = …!BC8")                       Bronze
B4  = Title!B15&" - "&Variables!C8
```

Blocks stack at a fixed 9-row stride. This is the printable podium sheet, and
is unrelated to the console's Awards tab (`awards-podium-tab-spec.md`), which
derives its own podium from the published snapshot.

**`Title`** — `B15 = Variables!C2`, `B16 = TEXT(Variables!C3,"MMM DD")&" - "&Variables!C8`,
`B17` the public URL. The generator writes only `Variables`; all three are
formulas and need no touching.

**`Reference for Players`** — Phase 1 spec §4.5's tab, one `FILTER` anchor per
category tab, at **cumulative offsets computed from each tab's actual
height** rather than the reference's hand-placed `A3, A106, A262, A377`
(the last of which is a `#REF!` to a category this facility never ran).

```
A3 = FILTER(LI40MD!$A$1:$C<end>, LI40MD!$A$1:A<end> > "")
```

Both downstream spills (`STANDINGSCSV!A2`, `Timeline!C3`) must read
`$A$3:$A` open-ended, per Phase 3 spec §7. The reference reads `A2:A1553`,
which is bounded and starts a row early.

**`QA Checklist`** — a static 27-row pre/post-schedule checklist with tick
boxes. Ships in the master; the generator does not touch it, except to fix
the row that reads `Does it follow the correct tournament type?` with `Dual
Meet` prefilled in `C5`.

**`Poster`** — images only. Ships blank in the master.

### 10.8 `Charts`

A static parent → child bracket tree, `PARENT`/`CHILD` pairs down `B`/`C`,
for a 32-pair ladder and for the irregular ladders the tiered byes produce
(`PLAYOFFS OF 23 PAIRS`, with slot names like `QF_R16-3` where a bye spans
two rounds).

**Ships in the master unchanged and is not generated.** It is a reference the
operator reads while filling the playoff slot table by hand. Once the
generator writes `AU` itself (§7.6) the tab becomes documentation rather than
a working surface, which is the right direction — but deleting it is out of
scope here, and the irregular trees on it are the clearest existing statement
of what §5.4 produces.

---

## 11. The Named Function library

Replace the standard workbooks' 15-function library with the Dual Meet
Master's 23, plus `COUNTPAIRAT`, and take the whole
`technical/named-function-library.md` contract as-is. The three additions
that matter:

| Function | Why |
| --- | --- |
| `STACKBLOCKS(first_col, step, first_row)` | Derives the court count from the sheet's own width. Removes the hand-edited column lists in §2.1 and the whole class of bug behind them |
| `MATCHTIME(m)` / `MATCHCOURT(m)` | Replace `CSV!C`/`D`'s hardcoded `BYROW`/`BYCOL` window. `MATCHCOURT` needs the `Variables!R` variant in §10.5 |
| `COUNTPAIRAT(time, code)` | Replaces `Timeline`'s 24-term `COUNTIFS` chain |

Two the standard library has that the dual meet's does not — keep both, they
cost nothing and the reference's formulas use them:

- `GETTEAM1SCOREBYMATCH(m)` / `GETTEAM2SCOREBYMATCH(m)`
- `GETWINSCORE()` → `11`, and `GETPLAYERBYRANK(range)` → `SORTBYWINS(range)`

`SORTBYWINS` is `INDEX(SORT({range}, 4, FALSE, 8, FALSE), 0, 1)` — sort by
column 4 (W) descending, then column 8 (Q) descending. Note this is relative
to the range's own first column, so a category tab's `A:H` block sorts by
`D` then `H`. Identical in both libraries.

`Timeline`'s time headers, `SCHEDULE`'s slot times and `COUNTPAIRAT` all have
to be the **same chained computation** off `Variables!C4` and `Variables!C6`,
for the float-equality reason in Phase 2 spec §8.7. Do not compute slot times
in JavaScript and write literals.

---

## 12. Implementation

### 12.1 Where the code goes

A **new file**, `sage-tools-api/scripts/standard-generator.gs`, bound to the
SAGE Standard Tournament Master. Not a mode inside `sheet-generator.gs`:

- The two masters are different workbooks, so neither script ever needs the
  other's code at runtime; a shared file would ship dual-meet code into the
  standard master and vice versa.
- `sheet-generator.gs` is 2,800 lines and its layout engine is written around
  square, per-club blocks. Every one of §7's differences would be a branch.
- The `onOpen` block is already duplicated verbatim across `sheets-sync.gs`
  and `sheet-generator.gs` with a "Change one, change both" note. This adds a
  third copy of that block and nothing else.

Menu: `SAGE → Generate event tabs`, same sidebar pattern, same
`logStep_`-polled progress, same self-removing menu item once
`PROP_TABS_GENERATED_FOR` is set.

### 12.2 What to copy from `sheet-generator.gs`

Verbatim, as pure functions — these have no dual-meet assumptions in them:

`parseCsvRows_`, `parsePlanCsv`, `slugifyTitle_`, `colLetter_`,
`parseTimeToMinutes_`, `minutesToClock_`, `trace_`, `logStep_`, `resetLog_`,
`getGenerationLog`, `ensureRowCount_`, `ensureColumnCount_`, `mergeVertical_`,
`stampFormat_`, `assertTabsPristine_` and the `REBUILT_IN_PLACE_TABS` shape,
`categoryColor_` and `colorFormulaFor_` (with §10.2's level-ladder
extension), and the whole `buildScheduleTab_` tiling strategy — `copyTo` on
both axes, grow down then across, flush between stages, copy column widths
and row heights explicitly (Phase 2 spec §8.5, §8.10, §8.11).

Rewritten for this shape: `computeLayout`, `buildCategoryTab`,
`buildMatchList`, every `write*Block*_`, every feeder builder, and
`readoutGeometry_`.

### 12.3 Rebuilt in place

Same registry and same pristine guard as Phase 3 spec §8.2. `SCHEDULE` and
`Court Control` keep the hard constraint that makes it necessary: their tab
GIDs are what the live sync trigger watches, stored in that workbook's
`SYNC_WATCHED_GIDS` Script Property (`sync-script-configuration-spec.md` §5),
so replacing either kills the sync while the sheet still looks correct.

| Tab | Prototype |
| --- | --- |
| `SCHEDULE` | 1 court block, 1 slot |
| `Court Control` | 2 court blocks + the stats block + the standby strip |
| `Timeline` | header row + 1 body row, stripped to `A:E` |
| `Timeline (Individual)` | header rows + 1 body row, stripped to `A:C` |
| `Standings` | 1 category column |
| `CSV` | header row + 1 match row, `A:M` |
| `STANDINGSCSV` | header row + 1 pair row, `A:G` |
| `Awards` | 1 category block |
| `Reference for Players` | 1 anchor |

Phase 3 spec §8.2's warning about `Timeline`'s width applies to both Timeline
tabs and to `Standings`: their built column count scales with the event, so a
prototype left wide lets an already-built tab read as pristine and get tiled
twice.

### 12.4 Ordering

Category tabs → `Variables` → `Title` → `Reference for Players` →
`SCHEDULE` → the readouts. `Reference for Players` needs each category tab's
final spill height; everything else needs `SCHEDULE`'s returned geometry.
`SCHEDULE` runs late for the blast-radius reason in Phase 2 spec §8.1.

### 12.5 Tracing and failure

Phase 2 spec §8.9 and Phase 1 spec §11.3, unchanged. Every computed extent,
every block anchor, the full match list, and — new here — the resolved
playoff ladder per category as `<KEY> <STAGE> slot <k> <- fresh | winner of <STAGE>-<j>`.
A wrong ladder is invisible in a finished tab unless you already know which
slot was expected.

On failure mid-build: no rollback, no continuing. Report which tabs were
written.

---

## 13. Build order

1. `parsePlanCsv` + §9 validation, reporting every failure together
2. **`planCategory(cat)`** — pure. Bracket sizes, RR match list per bracket,
   and the full playoff ladder from §5. Verified by hand against §1.1's six
   reference categories before any Sheets code exists
3. The master workbook (§14.1) — the larger half of the work
4. `buildCategoryTab` — group blocks, score grids, the playoff ladder,
   both scaffolds, the awards block
5. `Variables` (§10.1), `Title`, `Reference for Players`
6. `buildMatchList` + `buildScheduleTab_` (§10.2)
7. `CSV` and `STANDINGSCSV` — the two the live sync needs
8. `Court Control`, both `Timeline`s, `Standings`, `Awards`

Steps 1-2 are worth doing and checking on their own: `planCategory` is where
every §5 branch lives, it is the only part with real logic, and it can be
verified against two played events without a workbook.

---

## 14. What the master workbook must carry

A one-time manual job, and the larger half of the work — Phase 1 spec §6's
counterpart.

### 14.1 Building it

1. Copy the BKL Day 2 **Main** workbook — 3 categories, the twice-to-beat
   case, and the smaller of the two schedules.
2. Replace the Named Function library with §11's.
3. Reduce the three category tabs to a single hidden `_CATEGORY_TEMPLATE`
   carrying **one prototype of each block type**: one group pair, one playoff
   sub-block (two entrants), one group score grid cell set, one playoff score
   grid cell set, and the scaffold and awards chrome with values cleared.
4. Rewrite `CSV!C2`/`D2` as `=MATCHTIME($A2)` / `=MATCHCOURT($A2)`, and
   `Timeline`'s body as `=COUNTPAIRAT(D$2,$C3)`.
5. Point both spills at `'Reference for Players'!$A$3:$A`.
6. Add `Variables` columns `Q` (facility), `R` (court labels), `S`/`T` (the
   bracket map), with headers.
7. Relabel `Variables!C6` "Slot Pitch"; fix the `QA Checklist` `Dual Meet`
   prefill.
8. Add `Standings` and `Timeline (Individual)` (Main has neither; copy from
   Dreamcourts).
9. Delete the `#REF!` anchor from `Reference for Players`.
10. Strip every rebuilt-in-place tab to §12.3's prototype shape — **last**,
    then generate once immediately, per Phase 3 spec §12.
11. Name it **SAGE Standard Tournament Master**.

### 14.2 Decisions taken in the master, not in code

- **The awards block stays** on the category tab (§7.7), because the `Awards`
  tab reads it. This deliberately diverges from Phase 1 spec §4.2.
- **Column `K` keeps its double duty** — bracket number on group rows,
  next-round code and podium label on playoff rows. It is genuinely two
  columns' worth of meaning in one, but the whole forward-propagation chain
  (§7.5) and the awards block both key off it, and splitting it would touch
  every playoff formula to fix nothing that is broken. The generator writes a
  `Bronze:` prompt into `I` on the finals banner row of a §5.2 category, so
  the one case where a human has to type into a group row's `K` says so.
- **`Charts` ships as-is** (§10.8).

---

## 15. Open questions

**`R32` is not in the site's `STAGE_ORDER`.** The standard template's
`STAGE_ORDER` is `['R16','QF','SF','BRONZE','FINAL']` and
`ROUND_KEYWORD_LABELS` has no `R32` entry, so a category deep enough to reach
one renders those rows in an unsorted trailing stage with no label. Adding
both is a two-line site change, but it needs an event that actually reaches
R32 to test against, and none has. Validation caps entrants at 32 (§9)
partly for this reason.

**Bracket letters versus numbers.** This spec generates `A`/`B`/`C` to match
the reference workbooks and the bracket generator's lettered cards; the dual
meet generates `1`/`2`. The site treats the column as opaque, so both work,
but one convention would be better. Not settled here because changing the
dual meet's would rewrite published snapshots' meaning mid-archive.

**Whether the qualifier scaffold should be filled by the bracket generator.**
`bracket-generator-workbook-handoff-spec.md` §5 is about STEP 3 of the roster
scaffold. §7.6's `AM` column is a *second* draw with the same shape and the
same hand-shuffling problem, and that spec explicitly deferred its design
until a standard tournament existed. It now does. Deciding it is that spec's
job, not this one's — but it should be decided against both scaffolds.

**Per-facility court label ordering.** §6.1 takes the labels in schedule
order and assumes the operator lists them in the order they want courts
filled. Whether a partial slot should centre on the *label* order or on the
physical court order is undecided and has never mattered.

---

## 16. Acceptance

Run against a plan CSV reconstructed from BKL Cup 2026 Day 2 and confirm:

- Selecting `LI40MD LI40WD LI40XD`, facility `PCPH Main`, courts `1..12`,
  first match `1001` produces three category tabs and a `CSV` with **41**
  rows numbered `1001`-`1041`
- Selecting `LI18MD LI18WD LI18XD`, facility `PCPH Dreamcourts`, courts
  `7..15`, first match `3001` produces **84** rows numbered `3001`-`3084`
- `LI40MD`'s group block is 9 pairs in one run with `K = 1` on pairs 1-5 and
  `K = 2` on pairs 6-9, and two score grids, `5×5` and `4×4`
- `LI40WD` generates a twice-to-beat final — four codes,
  `LI40WD_F_1_(1) … LI40WD_F_2_(2)` — and **no bronze match**
- `LI18MD` generates the tiered ladder from §1.1: two `R16` slots, two `QF`,
  four `SF`, two `B`, two `F`, with `QF_3` fed from `R16` and `QF_4` fresh
- `AU` on every category tab carries the stage keywords the ladder implies,
  in slot order, with no operator input
- `STANDINGSCSV` has one row per pair **and per playoff slot** — 75 rows for
  Dreamcourts — with `G` reading `A`/`B`/`C`/`D` on group rows and blank on
  playoff rows
- `CSV!D` reads `Court 10` for Dreamcourts' first match, not `Court 1`
- No `#NAME?` and no `#REF!` anywhere
- Every named function resolves at both 12 and 15 court blocks with no edit
  to any column list
- `Timeline`'s grand total equals the match count; no body cell exceeds 1
- `Timeline (Individual)` flags a deliberately-broken draw that puts one
  human in two slots at once, where `Timeline` does not
- Running the generator twice is **refused**, not half-applied
- A `format: dual` CSV is rejected with a clear message and the workbook
  untouched
- The sync publishes a generated workbook with no hand-editing

---

## 17. Out of scope

- Any change to `sheet-generator.gs` or the Dual Meet Master
- Player name import — rosters stay a paste, both scaffolds (§7.6)
- Filling STEP 3 or the qualifier draw automatically (§15)
- Seeded or non-random draws
- Cross-facility match-number allocation (§6.2 reports, it does not coordinate)
- Categories split across two facilities in one day
- `R32` site support (§15)

---

## 18. Divergences

*(None — nothing here is built. Record departures when it is.)*
