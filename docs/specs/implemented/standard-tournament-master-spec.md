# Spec — Standard Tournament Master workbook

Generate a **standard-format** tournament workbook from a Tournament Time
Calculator CSV export, the way `dual-meet-sheet-generator-spec.md` already
does for dual meets — instead of duplicating last event's facility
spreadsheet and find-replacing every category key, court label and match
number by hand.

> **Status: implemented.** The bound Apps Script is
> `sage-tools-api/scripts/standard-generator.gs`, running in the **SAGE
> Standard Tournament Master** workbook, whose `_CATEGORY_TEMPLATE` is an
> unchanged copy of the Pickle for Sight Annex's `HIMD` tab. The generator
> passes `scripts/verify-standard-generator.mjs`, which checks §16 against a
> mocked master fed both Pickle for Sight plan CSVs. §18 records where the
> code departs from this document; read it before trusting a cell reference.
>
> This is the "standard-tournament equivalent" the root `CLAUDE.md` had been
> waiting on.

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
it means exactly that and is not restated. The three are, all in
`sage-docs/docs/specs/implemented/`:

| Called here | File | Covers |
| --- | --- | --- |
| Phase 1 spec | `dual-meet-sheet-generator-spec.md` | The master-copy + bound-script approach, the plan CSV, category tabs, validation |
| Phase 2 spec | `dual-meet-schedule-generator-spec.md` | `SCHEDULE`'s geometry, match placement, category colours, tiling mechanics |
| Phase 3 spec | `dual-meet-readouts-generator-spec.md` | `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`, the rebuilt-in-place guard |

---

## 0. Orientation — read this first if you have no context

**What this is.** SAGE runs pickleball tournaments. A **standard tournament**
is the ordinary open-entry shape: pairs enter a category, get drawn into
round-robin brackets, and the top finishers from each bracket advance into a
single-elimination playoff. It is not a dual meet — there are no clubs, no
cross-product group stage, and no fixed two-entrant playoff.

Each **facility, each day** runs off its own Google Sheets workbook. BKL Cup
2026 Day 2 had three: Main, Annex and Dreamcourts. Pickle for Sight 2026 has
two: PCPH Main and PCPH Annex. That is the single biggest structural
difference from a dual meet, which is one workbook for one venue, and §3 is
about it.

**Where things are.**

| Thing | Path |
| --- | --- |
| The dual-meet generator this one copies from | `sage-tools-api/scripts/sheet-generator.gs` |
| The live-sync script that shares the Apps Script project (and owns `SAGE → Fill match numbers`) | `sage-tools-api/scripts/sheets-sync.gs` |
| The Tournament Time Calculator, source of the plan CSV and of `playoffPlan()` | `sage-match-control.github.io/tools/tournament-calculator.html` |
| The named-function contract §11 adopts | `sage-docs/docs/technical/named-function-library.md` |
| The console's Awards tab, which produces the podium (§7.7) | `sage-docs/docs/specs/implemented/awards-podium-tab-spec.md` |
| The event these references come from | `sage-docs/docs/specs/in-progress/pickle-for-sight-spec.md` |
| The public site template that parses the team codes (§8) | `sage-match-control.github.io/_templates/standard-tournament-template/index.html` |

**Vocabulary.** Same as the dual-meet specs, minus clubs, plus:

| Term | Means |
| --- | --- |
| PFS, BKL | The two reference events (§1): Pickle for Sight 2026 and BKL Cup 2026 |
| category | A division, keyed `NMD`, `LIXD`, `LI40MD`, … — one tab each |
| pair | Two players who play together; the atomic competitor |
| bracket | A round-robin pool inside a category. Sizes may be **uneven** |
| slot | One start time across all courts — two `SCHEDULE` rows |
| stage | One playoff round: `R32`, `R16`, `QF`, `SF`, `B`, `F` |
| playoff slot | One numbered seat in a stage, e.g. `SF_3` — gets its own team code |
| team code | `<KEY>_<i>` for a group pair, `<KEY>_<STAGE>_<k>` for a playoff slot |
| fresh seat | A playoff slot filled from the group stage by the qualifier draw, or by a bye (§7.6) |
| fed seat | A playoff slot filled by the winner of an earlier playoff match (§7.5) |
| facility | One venue running one subset of the day's categories |
| `MATCHES` band | One category's column of court blocks on the `MATCHES` tab (§10.2.2) |
| wave | A group of categories scheduled together; a later wave starts after an earlier one ends (§10.2.4) |

**Section-reference convention.** A bare `§n` means this document. A
reference into another spec always names it — "Phase 1 spec §11.1", "Phase 3
spec §5.4".

---

## 1. Sources

None of the workbook structure is in source control. Everything below was
read directly out of four hand-built workbooks from two events, exported as
`.xlsx` and read cell by cell (Phase 3 spec §1.1 has the technique):

| Workbook | File ID | What it gave |
| --- | --- | --- |
| Pickle for Sight · **PCPH Main** | `1-MYwVDD8z0ENtEiy3HztRTDghcGxlAmCEwn6p6OZzRA` | 4 categories, 116 matches, courts 1-4, the full-draw and tiered-bye ladders, the wave schedule (§10.2.4) |
| Pickle for Sight · **PCPH Annex** | `1dC0_GOnumT3_0D9MOEO8lKu_Y69oE5wIFdkK-MY33XY` | 5 categories, 125 matches, courts 5-9, the round-robin-only shape (§5.2) |
| BKL Day 2 · **Main** (LI40) | `1resKkD3e8kjxJEcQkruzKW8N93NFjuqgtmD9_hwYHQs` | 3 categories, 41 matches, 12 court blocks, the twice-to-beat case |
| BKL Day 2 · **Dreamcourts** (LI18) | `1H5aHgogxauIgbb3xoBSuLQjVZCCdZcI_9iBXUMp_9SI` | 3 categories, 84 matches, 15 court blocks, a 4-stage playoff ladder, `Standings` and `Timeline (Individual)` |

The two Pickle for Sight workbooks are `[REF]` copies of the event's
workbooks. The operators built those by hand from a BKL copy, following
`pickle-for-sight-spec.md` §12.1, and corrected them against this spec
before the copies were taken. They are the **authoritative** reference:
where they and the BKL workbooks disagree, this spec follows Pickle for
Sight. BKL is used only for what Pickle for Sight does not have — the
twice-to-beat final (§5.2) and the 12- and 15-court widths. The BKL workbooks are registered in
`event-data/config/events.json` under `bkl-cup-2026 → day3`, alongside a
third (Annex) not read here.

Each Pickle for Sight workbook has its own calculator plan CSV, one per
venue, not one for the whole event (§3, §4):

| File | `courts` | Categories | Projected |
| --- | --- | --- | --- |
| `pickle-for-sight-tournament-2026-09-27-plan MAIN.csv` | 4 | `NWD NMD NXD LIWD` | 116 matches, 9:05 PM |
| `pickle-for-sight-tournament-2026-09-27-plan ANNEX.csv` | 5 | `LIMD LIXD HIWD HIMD HIXD` | 125 matches, 7:25 PM |

**Read coverage.** Every tab of all four workbooks was read as cell values
plus formulas, and every `<definedNames>` library in full. The `Poster` tab
holds only images and was not inspected further.

### 1.1 The reference events reconcile against the calculator exactly

Worth stating up front, because it is what licenses §5: the calculator's
`calcCategory()` reproduces all four reference workbooks' match counts to the
match, from nothing but pair count, bracket count, `advance`, `fill` and
`solo_format`.

| Workbook | Category | Pairs | Brackets | RR | Playoff | Total |
| --- | --- | --- | --- | --- | --- | --- |
| PFS Main | `NWD` | 8 | 2 (4-4) | 12 | 4 | 16 |
| PFS Main | `NMD` | 15 | 3 (5-5-5) | 30 | 6 (tiered) | 36 |
| PFS Main | `NXD` | 20 | 4 (5-5-5-5) | 40 | 8 | 48 |
| PFS Main | `LIWD` | 8 | 2 (4-4) | 12 | 4 | 16 |
| | | | | | **PFS Main total** | **116** |
| PFS Annex | `LIMD` | 19 | 4 (5-5-5-4) | 36 | 8 | 44 |
| PFS Annex | `LIXD` | 18 | 4 (5-5-4-4) | 32 | 8 | 40 |
| PFS Annex | `HIWD` | 3 | 1 | 3 | 0 (`solo_format: rr`) | 3 |
| PFS Annex | `HIMD` | 13 | 3 (5-4-4) | 22 | 6 (tiered) | 28 |
| PFS Annex | `HIXD` | 6 | 2 (3-3) | 6 | 4 | 10 |
| | | | | | **PFS Annex total** | **125** |

116 and 125 are each workbook's `CSV` row count and its last match number
less the base (`1116`, `2125`). The bracket sizes reconcile against each
category tab's group score grids (§7.4). There is one grid per bracket, sized
5-4-4 in `HIMD`, 5-5-4-4 in `LIXD` and 3-3 in `HIXD`.

`NMD` and `HIMD` are both 3 brackets × top 2 under `fill: bye`, the same
shape as BKL's `LI18MD` below. Both workbooks build it as exactly the ladder
the calculator plans: one `R16` match, one `QF`, two `SF`, a bronze and a
final (§7.5).

| Workbook | Category | Pairs | Brackets | RR | Playoff | Total |
| --- | --- | --- | --- | --- | --- | --- |
| BKL Main | `LI40MD` | 9 | 2 (5-4) | 16 | 4 | 20 |
| BKL Main | `LI40WD` | 4 | 1 | 6 | 2 (twice-to-beat) | 8 |
| BKL Main | `LI40XD` | 7 | 2 (4-3) | 9 | 4 | 13 |
| | | | | | **BKL Main total** | **41** |
| BKL Dreamcourts | `LI18MD` | 15 | 3 (5-5-5) | 30 | 6 | 36 |
| BKL Dreamcourts | `LI18WD` | 8 | 2 (4-4) | 12 | 4 | 16 |
| BKL Dreamcourts | `LI18XD` | 16 | 4 (4-4-4-4) | 24 | 8 | 32 |
| | | | | | **BKL Dreamcourts total** | **84** |

41 is `Timeline!C2` and the `CSV` row count in BKL Main; 84 is the same in
BKL Dreamcourts. The bracket sizes reconcile against the `bracket` column's own
`A`/`B`/`C`/`D` tallies (13/13/9/4 across the three Dreamcourts categories).

`LI18MD` is the first reference that proved the tiered-bye machinery: 3 brackets × top 2
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

The BKL workbooks carry a **different, older library** — 15 functions where
the Dual Meet Master has 23, and none of them built on `STACKBLOCKS`:

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

The Pickle for Sight workbooks already run §11's library. Both carry the same
24 functions: the dual meet's 23, with `MATCHCOURT` in §10.5's row-5 form,
plus `GETWINSCORE`. They run at 4 and at 5 court blocks with no column list
anywhere. That library is no longer a proposal; it is proven on a standard
workbook, and §14.1 starts from it, less the two functions nothing calls
(§11).

Two masters rather than one, for the same reason there are two site
templates: the category tab is a genuinely different organism (§7), the
playoff model is inverted (§8), and one workbook covers one facility rather
than one event (§3). A shared master would be a mode switch on every tab.

### 2.1 What the hand-built workbooks get wrong

Recorded because each one is a requirement in disguise — the generator's job
is to make these unrepresentable, not to reproduce them.

| Defect | Where | Consequence |
| --- | --- | --- |
| `SCHEDULE` is 15 court blocks wide but `Variables!C5` says `courts: 6` | BKL Dreamcourts | Every pace estimate on `Court Control` divides by the wrong number |
| Court labels repeat — `10 11 12 13 14 15 7 8 9 10 11 12 13 14 15` | BKL Dreamcourts | `CSV!D` reports the wrong court for any match past block 9 |
| `CSV!C2`/`D2` scan a hardcoded `SCHEDULE!$F$6:$BZ$78` — 10 blocks | both BKL | Silently wrong for courts 11+; only survived because 6 were used |
| Named-function column list (12) ≠ schedule width (15) | BKL Dreamcourts | Courts 13-15 invisible to every standings formula |
| `Court Control!C42 = SUM(LI40MD!D:D)+SUM(LI40XD!D:D)+SUM(LI40WD!D:D)` | BKL Main | Add a category, forget the sum, "matches done" goes quietly stale |
| `Reference for Players!A377 = FILTER(HIMD!…)` → `#REF!` | BKL Main | A leftover anchor for a category this facility never ran |
| `Reference for Players` carries an anchor for all nine of the event's categories; the four or five the other venue runs resolve to `#REF!` | PFS Main, PFS Annex | Same class as the row above. Harmless while nothing reads those rows, but every spill over the tab has to step over them (§10.7) |
| `Timeline` body is a 24-term `COUNTIFS` chain, one pair of terms per court | both BKL | Unmaintainable; wrong the moment a court is added |
| `Variables!I38` reads `Low Intermediate 18+ Women's Doubles` on the `HI18WD` row | BKL Main | Same class as the `LIMD`/`LIWD` typo Phase 1 spec §4.3 fixed structurally |
| A playoff block's `C` cell claims the winner's next code is `<KEY>_F_SF-1` | all four | Not a real code — see §7.5 |
| STEP 3 (`AI`) is prefilled `<KEY>_1 … <KEY>_<n>` in order on every category tab | PFS Main, PFS Annex | The unshuffled mapping §7.6 forbids: every pair maps to its own roster slot, and the step looks done |
| `LIXD`'s playoff slot table (`AT:AU`) ends in two stray rows, `F1x2`/`F1` and `F2x2`/`F2` | PFS Annex | Harmless — no drawn number can equal `F1x2` — but a leftover of how BKL's `LI40WD` rigged its twice-to-beat final inside the scaffold (§7.5.1). The generator writes exactly one row per fresh seat (§7.6) |

---

## 3. The unit of generation is a facility-day, not an event

A dual meet is one workbook. A standard tournament is **one workbook per
facility per day**. The plan CSV may describe the whole event (BKL) or one
facility (Pickle for Sight exported one CSV per venue, §1). Three
consequences, all of which the generator has to be told about because the
CSV cannot carry them:

**Categories are assigned to facilities by hand.** BKL Day 2 put the LI40
categories on Main and the LI18 categories on Dreamcourts. Pickle for Sight
put Novice and `LIWD` on Main and the rest of Low and High Intermediate on
the Annex. Nothing derives that. The operator picks, either by exporting one
CSV per venue from the calculator or by selecting categories in the sidebar
(§6.3).

**Court numbering is facility-local and non-contiguous.** BKL Main ran courts
1-12, Dreamcourts 7-15. Pickle for Sight runs one continuous range across its
two venues, Main 1-4 and Annex 5-9, because the site places a match at a
venue by its court number (`pickle-for-sight-spec.md` §12.1). The label is
free text in `SCHEDULE!` row 5 and is what `CSV!D` publishes, so it has to be
an input, not `"Court " & c`.

**Match numbers are facility-scoped and offset.** BKL Main starts at `1001`,
Dreamcourts at `3001`, Annex (by the same rule) at `2001`. Pickle for Sight
Main starts at `1001` and Annex at `2001`. All the facilities' `CSV` tabs
merge into one day snapshot in `event-data`, so a collision would silently
overwrite a match. The operator types the offset when numbering the packed
schedule with `SAGE → Fill match numbers` (§6.2, §10.2.3).

Every category row in the CSV goes into `Variables!H:P` (§6.3). The workbook
documents the whole plan it was given and generates tabs for its own slice.

---

## 4. Input — the plan CSV in standard format

Same file as Phase 1 spec §2, same columns:

```
type,key,value,teams,brackets,advance,fill,note,teams_a,teams_b,po_format,solo_format
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
| `solo_format` | **Live, one-bracket categories only.** `ttb` (twice-to-beat final) or `rr` (round robin only, no playoff) — §5.2. The calculator writes it only for a standard category that resolves to one bracket and is not a 2-pair best-of-3, and leaves it empty everywhere else. Empty, or a missing 12th column in an older 11-column export, means `ttb` — the same default the calculator's own Import CSV applies |

The file's data rows carry fewer cells than the header — `setting` rows stop
after `value` — so parse positionally and treat a missing trailing cell as
empty, as `sheet-generator.gs`'s `parsePlanCsv` already does.

`setting format` must read `standard`. A `dual` file is rejected, not
approximated — the exact converse of Phase 1 spec §3.

Settings used: `title`, `date`, `start`, `duration_min`, `buffer_min`.
`courts` is read but **overridden** by the per-facility court list (§6.1).
In a whole-event CSV it is the event's total across all venues. In a
per-venue CSV (Pickle for Sight's `4` and `5`) it is that venue's count, and
the generator warns, without failing, when it differs from the number of
court labels entered. `club_a`/`club_b` are dual-meet settings and are
ignored.

---

## 5. Every playoff shape the calculator can produce

This is the enumeration the generator's playoff builder has to cover. All of
it is `playoffPlan(groups, adv, fill, soloFormat)` and its helpers
(`groupSizes`, `reduceToTarget`, `solveForTarget`, `tieredRounds`,
`calcCategory`) in `sage-match-control.github.io/tools/tournament-calculator.html`;
the branch order below is the code's own. The generator reimplements them in
Apps Script and must give the same numbers — §1.1's tables are the check.

### 5.1 `teams == 2` — best-of-3, no round robin

`calcCategory` short-circuits before `groupSizes` is ever called.

```
plan = { rounds: [{label:'Final · best of 3', matches:3, byes:0}], total:3, min:2, bestOf3:true }
```

No group stage, no bracket, no bronze. Three match slots, of which the third
is played only if the series splits 1-1 — the plan counts the worst case, so
the workbook must carry three. No reference workbook has this shape; §7.5.1
specifies its blocks.

### 5.2 `brackets == 1` — two shapes, picked by `solo_format`

`playoffPlan(groups, adv, fill, soloFormat)` reads `soloFormat` only when
`groups == 1`.

**`solo_format: ttb` (the default) — round robin then a twice-to-beat final.**

```
plan = { rounds: [{label:'Final · twice-to-beat', matches:2}], single:true, rrOnly:false,
         qualifiers:2, bronzeText:'RR #3 automatic' }
```

RR #1 needs one win; RR #2 must win twice. Two match slots, and **bronze is
not a played match** — the third-placed pair takes it on record. The workbook
still carries it as a row, a walkover against `BYE`, because that is how the
console reads a bronze with no match (§7.5.1).

BKL Main's `LI40WD` (4 pairs, 6 RR + 2) is the only reference, and it was
rigged by hand: its final exists only on `SCHEDULE` and in the qualifier
scaffold, not as blocks on the tab. Its codes are the ones to copy:
`LI40WD_F_1_(1)`, `LI40WD_F_2_(1)` for game 1 and `LI40WD_F_1_(2)`,
`LI40WD_F_2_(2)` for game 2. The `(n)` suffix is already understood by the
site — `matchInstanceOf()` in the standard template parses it and
`roundLabel()` renders "Final · Match 2". The blocks are specified in §7.5.1.

**`solo_format: rr` — round robin only.**

```
plan = { rounds: [], total:0, qualifiers:0, single:true, rrOnly:true, bronzeText:null }
```

No playoff at all. The round-robin standings decide every medal: RR #1 gold,
#2 silver, #3 bronze. `rounds` is empty rather than a zero-match round, so
the generator builds nothing from it: no stage banner, no playoff block, no
qualifier scaffold rows and no playoff slot table.

Verified in PFS Annex's `HIWD` (3 pairs, `advance: 1`, 3 matches). The tab is
the group block and one 3×3 score grid, and nothing below them. `HIWD` never
appears in a playoff stage of the schedule.

With no playoff row to carry `GOLD`/`SILVER`/`BRONZE`, the podium comes from
the standings. The console's Awards tab already does exactly that when a
category has no final (§7.7).

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

**`fill`**: the two options answer the same question — the field doesn't fill
a bracket cleanly, so what do we do with the empty seats? — but they measure
those seats against different things.

`bye` keeps the tiered ladder above untouched: the empty seats are byes in
the earliest round, and group winners keep their skip ahead.

`wc` abandons the ladder and fills the draw out to a **complete** bracket:
enough wildcard entrants to reach the next power of two, so every qualifier
starts in the same round, nobody byes, and nobody has a longer road to gold
than anyone else. The count is measured against the bracket, not against the
one play-in round the ladder happened to leave short — 3 brackets × top 2 is
6 qualifiers in a draw of 8, so **2** wildcards, not the 1 bye the tiered
ladder's first round contained. `wc = 2^ceil(log2(direct)) - direct`, and the
rounds are the plain halving from that bracket size down to the Final.

This changes the entrant count, so it changes the number of playoff slots the
tab must carry. Note the two fills can differ in round *count* as well as
size: `LI18MD` under `bye` is the four-round ladder `R16 · QF · SF · Final`,
and under `wc` is the three-round `QF · SF · Final` of a full draw of 8.
For `advance == 1` the two rules agree — a flat bracket's first-round byes
already number `2^ceil(log2(groups)) - groups` — so only tiered categories
(`advance >= 2`) see any change.

**A tiered ladder can be deeper than its entrant count suggests.** Round
labels walk back from the Final, so a ladder with six rounds labels its first
one `Round of 64`, however few pairs are in it. Under `fill: bye`, 15 of the
calculator's shapes with at most 32 qualifiers go past `R32` — among them
7 brackets × top 2 (14 qualifiers, 6 rounds) and 3 brackets × top 4
(12 qualifiers, 6 rounds). §8's code grammar stops at `R32`, so §9 caps the
ladder's depth as well as its entrant count.

### 5.5 The shapes, as a table

| `teams` | `brackets` | `advance` | Shape | Playoff matches | Bronze |
| --- | --- | --- | --- | --- | --- |
| 2 | — | — | best of 3 | 3 (min 2) | none |
| any | 1 | — | RR + twice-to-beat (`solo_format: ttb`) | 2 (min 1) | RR #3, no match |
| any | 1 | — | RR only (`solo_format: rr`) | 0 | RR #3, no match |
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
| Facility label | empty | `Variables!C8`, and through it `Title!B7` — e.g. `PCPH Main` |
| Court labels | — | §6.1 |
| Categories in this workbook | none selected | Which tabs get built (§3) |
| Wave, per selected category | `2` for a key ending `XD`, else `1` | The suggested layout and the `SCHEDULE` height (§10.2.4) |
| Day label | empty | `Variables!P` on the selected rows |

### 6.1 Court labels

A comma-separated list, in schedule order: `1,2,3,4` or `7,8,9,10,11,12`.
Its **length is the court count** for this workbook — `Variables!C5`, the
`SCHEDULE` width and the `Court Control` block count all come from it, so the
Dreamcourts mismatch in §2.1 cannot recur.

Labels render as `Court <label>` in `SCHEDULE!` row 5 and are what `CSV!D`
publishes. Duplicates are rejected (§9).

### 6.2 Match numbers

The generator writes no match numbers. The operator packs `SCHEDULE` by hand
(§10.2.3) and then runs `SAGE → Fill match numbers`, which asks for a base
and numbers every placed match from base + 1, left to right across the
courts and down the slots. It writes the same numbers into `CSV!A`.

The convention the reference events use, facility index × 1000 (so Main
enters `1000` and gets `1001` onward, the Annex `2000` and gets `2001`
onward), is a convention, not a rule anything enforces. Nothing in
`events.json` orders facilities, and no workbook can see another's numbers,
so keeping the ranges apart is the operator's check. The generator's
completion message restates the convention.

### 6.3 Categories, and the plan block

`Variables!H:P` gets **every** category row from the CSV, plus a `day` column
(`P`) and a `facility` column (`Q`, new — §10.1) that the generator fills in
for the rows it selected and leaves blank for the rest. Tabs are built only
for the selected keys, in CSV order.

That is what the reference workbooks do (BKL Main's `Variables` lists all
40-odd categories of the whole five-day event while carrying three tabs; both
Pickle for Sight workbooks list all nine of the event's categories), and it is
worth keeping: `B1` on each category tab reads its display name back out of
that block by key, and the operator can see at a glance which slice of the
event this workbook is.

With a per-venue CSV every category in it belongs to this workbook, so the
sidebar has a **Select all** control. The default stays "none selected": the
generator cannot tell a per-venue CSV from a whole-event one, and a
whole-event CSV with everything ticked would build every venue's tabs into
one workbook.

---

## 7. The category tab

### 7.1 Column map

Verified cell by cell against BKL's `LI40MD` and `LI18MD` and all nine Pickle
for Sight category tabs, which have the same columns. Columns never move; only
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
| `K` | **Bracket** number on each bracket's first group pair (§7.2); **Next Round** and the medal label on playoff rows (§7.5) |
| `L` | Bracket rank — hand-typed |
| `N` | Score-grid row labels |
| `O`–… | Score-grid body (§7.4) |
| `AB` | Notes column header for the roster scaffold |
| `AC` `AD` `AE` | Roster scaffold: pair index, **STEP 1 · NAMES**, code link — two rows per pair |
| `AG` `AH` `AI` | Roster scaffold: number, **STEP 2 · CODES**, **STEP 3 · RANDOMIZED CODES** — **one row per pair** |
| `AM` `AN` `AO` `AP` | Qualifier scaffold — the playoff draw (§7.6) |
| `AT` `AU` `AV` | Playoff slot table (§7.6) |

Header chrome: `A1` is the raw key; `B1 = FILTER(Variables!J:J, Variables!I:I=A1)`
is the display name; `AD1 = A1` is what every scaffold formula concatenates
against. Rows 2-4 carry the `STEP 1/2/3` labels and the scaffold headers,
and `K3` reads `FOR NOTES ONLY`;
row 5 carries `W`, `L`, `Team Scr`, `Opp\nScr`, `Q`, `Matches Done`,
`Bracket`, `Br Rank`. Group pairs start at row 6.

### 7.2 Group blocks — one continuous list, brackets marked in `K`

Unlike a dual meet, **the brackets of a category are not separate blocks**.
All `teams` pairs run continuously from row 6, two rows each, and bracket
membership is carried by column `K`: the bracket number on the **first pair
of each bracket**, and nothing on the others. `HIMD` (5-4-4) has `K = 1` on
pair 1, `2` on pair 6 and `3` on pair 10, with no gap, no second header row
and no banner between them.

Keep that. It makes the block geometry independent of the bracket split, and
it means codes stay `<KEY>_1 … <KEY>_<teams>` in one unbroken run — which is
what the roster scaffold, the `Reference for Players` spill and the bracket
generator's own output all assume.

The generator knows the split, so it writes each number on the right pair.
The same `groupSizes` split drives `STANDINGSCSV!G` (§10.4) and the score
grids (§7.4), one grid per bracket anchored on the same first pair, so the
three always agree.

Per-pair formulas, filled down:

```
A6  = A$1&"_"&Variables!AB91          LI40MD_1     (the reference-number ladder, §7.3)
A7  = A6                              second row of the pair
B6  = IFNA(FILTER($AD$5:$AD178,$AE$5:$AE178=A6),A6)
D6  = GETTOTALWINS(A6)      E6 = GETTOTALLOSES(A6)
F6  = GETTOTALSCORE(A6)     G6 = GETTOTALOPPONENTSCORE(A6)
H6  = GETSCOREQUOTIENT(A6)  J6 = D6+E6
K6  = <bracket number>                literal, on each bracket's first pair only
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

It looks like an indirection worth removing, but keep it. The qualifier
scaffold's ordinals (`AN`, §7.6) read the same ladder cells. Both are
relative references that step two rows per pair, so the prototype block
(§14.1) can be copied down with `copyTo` and every copy numbers itself. A
literal `_1`, `_2`, … would have to be written pair by pair.

### 7.4 The score grid

One grid per bracket, anchored on that bracket's first pair row. For a
bracket of `n_b` pairs whose first pair row is `first`:

| Cell | Content |
| --- | --- |
| `O<first>` … `<n_b`-th col`><first>` | `=N<first+1>` … `=N<first+n_b>` — the column headers |
| `N<first+1+k>` | `=A<first + 2k>` — one row per pair, **not** two |
| body `(r, c)` | `=GETSCOREAGAINSTPAIR($N<r>, <c>$<first>)`, diagonal left blank |

Verified at `n_b = 5` (`O:S`), `n_b = 4` (`O:R`) and `n_b = 3` (`O:Q`), and
on the uneven splits: `HIMD`'s grids sit at rows 6, 16 and 24 for its 5-4-4
brackets, which are pairs 1, 6 and 10. Note the offset from the
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

Every example below is a real cell in PFS Main's `NMD` (15 pairs, 3
brackets × top 2, `fill: bye`), whose ladder is `R16` → `QF` → `SF` → `B` + `F`.
Its last group pair sits on rows 34-35.

**Row geometry.** Everything below the group block is built from three
pieces, stacked with no other rows between them:

| Piece | Rows | `NMD` |
| --- | --- | --- |
| gap after the group block | 1 blank row | row 36 |
| **stage banner** | 1 banner row, then 1 blank row | `R16` at 37; `QF` at 45; `SF` at 53; `FINALS` at 67 |
| **match block** | 1 label row, 4 entrant rows (2 per entrant), 1 blank row | `R16-3` at 39-44; `QF-2` at 47-52; `SF-1` at 55-60; `SF-2` at 61-66; `B` at 69-74; `F` at 75-80 |

A stage is its banner followed by its match blocks, in block-label order.
The next stage's banner follows the last block's blank row.

**Stage banner** — the stage keyword in `A`, the stage before it in `I`
(`-` for the first stage) and the stage after it in `K`:

```
A53 : SF        H53 : "before:"   I53 : QF     J53 : "next:"   K53 : F
```

The finals banner is just `A67 : FINALS`.

**Match block** — a label row, then two entrants of two rows each:

```
A55 = A53&"-1"                             SF-1              block label (§7.6.1 numbers it)
B55 : "winner's next code ->"
C55 = <the cell of the seat the winner moves to>   see the note below
D55..H55 : W / L / Team Scr / Opp Scr / Q headers
I55 : "Notes"                              (on early-round blocks)
K55 : "Next Round"

A56 = A$1&"_"&A53&"_1"                     NMD_SF_1          seat number from §7.6.1
B56 = <name lookup, see below>
D56 = GETTOTALWINS(A56)   E56 = GETTOTALLOSES(A56)   F56 = GETTOTALSCORE(A56)
G56 = GETTOTALOPPONENTSCORE(A56)          H56 = GETSCOREQUOTIENT(A56)
K56 = <advance formula, see below>
A57 = A56     B57 : second name (spilled)  K57 = K56
A58 = A$1&"_"&A53&"_2"                     NMD_SF_2
… same shape on rows 58-59
```

The `B` and `F` blocks under `FINALS` have the same shape, but their label
row carries only `A69 : B` (or `A75 : F`) and the `D`–`H` headers: no
"winner's next code", no `C`, no `Next Round`. Their seat codes are
`A70 = A$1&"_"&A69&"_1"` → `NMD_B_1`, and so on. Each block also has a
two-entrant score grid in `N:P` on the rows after its label, per §7.4.

**Name lookup** in `B` — two forms, and which one a seat gets is the whole
point:

| Seat is | `B` formula |
| --- | --- |
| **fresh** — drawn in from the group stage, or byeing into this round | `=IFNA(FILTER(AO:AO, AP:AP=A<r>), A<r>)` — the qualifier scaffold (§7.6) |
| **fed** — the winner of a previous-round match | `=IFNA(FILTER(B<lo>:B<hi>, K<lo>:K<hi>=A<r>), A<r>)`, where `<lo>:<hi>` spans the previous stage's rows |

A single stage mixes both:

| Block | Seat | Name lookup | Winner goes to |
| --- | --- | --- | --- |
| `R16-3` | `R16_5`, `R16_6` | both fresh | `QF_3` (`C39 = A48`) |
| `QF-2` | `QF_3` | fed — `B48 = IFNA(FILTER(B38:B44, K38:K44=A48), A48)` | `SF_4` (`C47 = A64`) |
| | `QF_4` | fresh | |
| `SF-1` | `SF_1`, `SF_2` | both fresh | `F_1`, loser to `B_1` |
| `SF-2` | `SF_3` | fresh | `F_2`, loser to `B_2` |
| | `SF_4` | fed from `QF-2` | |
| `B`, `F` | `_1`, `_2` | fed — `B70 = IFNA(FILTER(B54:B66, K54:K66=A70), A70)` | `BRONZE` / `GOLD`, `SILVER` |

`QF_4` fresh beside `QF_3` fed is precisely §5.4's tiered bye, rendered in
formulas; HIMD builds the same ladder, and BKL's `LI18MD` did too. In a
full draw with no byes (`NXD`, `LIMD`, `LIXD`: 4 brackets × top 2), every
`QF` seat is fresh and every `SF` seat is fed, `QF-k`'s winner to `SF_k`.
§7.6.1 gives the one numbering rule that produces both.

**A final can have a fresh seat.** A flat ladder of 3 (3 brackets × top 1)
is one `SF` and a bye straight into the final, so `F_1` is drawn in from the
qualifier scaffold and only `F_2` is fed from the `SF` block. No reference
workbook has this shape. It follows from §7.6.1's numbering, which treats the
final like any other stage.

**Advance formula** in `K`, on each entrant's first row; the second row
mirrors it (`K57 = K56`):

```
early rounds   K40 = IF(D40>0, C39, "-")                               winner moves to C's seat
semifinals     K56 = IFNA(IFS(D56>0, A76, E56>0, A70), "-")            winner -> F_1, loser -> B_1
               K62 = IFNA(IFS(D62>0, A78, E62>0, A72), "-")            SF-2: F_2 and B_2
bronze         K70 = IF(D70>0, "BRONZE", "-")
final          K76 = IFNA(IFS(D76>0, "GOLD", E76>0, "SILVER"), "-")
```

`K` is not merged on playoff rows, because the entrant's two rows hold
different things (§7.2).

> **The `C` cell is inconsistent in every reference and the generator must
> pick one form.** Early-round blocks point it at the real destination cell
> (`C39 = A48`, resolving to `NMD_QF_3`), which is what `K` then uses.
> Semifinal blocks instead synthesise a label — `C55 = $A$1&"_"&K53&"_"&A55`
> → `NMD_F_SF-1` — which is **not a team code that exists anywhere**. It is
> harmless only because the SF's own `K` formula ignores `C` and names the
> destination cells directly. Generate the early-round form everywhere: on
> a semifinal block, `C` points at the `F` seat its winner takes (`C55 = A76`).

#### 7.5.1 Finals without a ladder — twice-to-beat and best-of-3

Two shapes have a final but no ladder to feed it (§5.1, §5.2). No reference
workbook builds either as blocks — BKL's `LI40WD` put its twice-to-beat codes
on `SCHEDULE` by hand, and seeded them from four hand-typed rows in its
qualifier scaffold (`AM71:AP78`, with `F1x2`/`F2x2` rows in the slot table)
plus a `LI40WD_B` bronze seat. What follows is designed, using the same
block shape and geometry as §7.5 and the codes BKL used.

**Twice-to-beat** (one bracket, `solo_format: ttb`). After the group block,
the gap row and a `FINALS` banner, three blocks in this order:

| Block label (`A`) | Seats | Name lookup (`B`) | `K` |
| --- | --- | --- | --- |
| `F (1)` | `<KEY>_F_1_(1)`, `<KEY>_F_2_(1)` | `F_1`: `=IFNA(FILTER(AO:AO, AP:AP=$A$1&"_F_1"), A<r>)`; `F_2`: the same against `"_F_2"` | the final's formula |
| `F (2)` | `<KEY>_F_1_(2)`, `<KEY>_F_2_(2)` | the same two lookups as game 1 | the final's formula |
| `B` | `<KEY>_B_1`, `<KEY>_B_2` | `B_1`: `=IFNA(FILTER(AO:AO, AP:AP=A<r>), A<r>)`; `B_2`: the literal `BYE` on both rows | `B_1`: `BRONZE`; `B_2`: `-` |

The qualifier scaffold (§7.6) has three rows and **no draw** — the round robin
decides who goes where, so `AM` stays `-`, the slot table is empty, and `AP`
is a literal written by the generator:

| `AN` tier label | `AP` | `AO` |
| --- | --- | --- |
| `RR #1 - twice to beat` | `<KEY>_F_1` | the RR #1 pair's names, operator-written |
| `RR #2 - challenger` | `<KEY>_F_2` | the RR #2 pair's names |
| `RR #3 - bronze` | `<KEY>_B_1` | the RR #3 pair's names |

The `B` block is a **walkover**: `<KEY>_B_1` against a side whose name cell
reads `BYE`. `GETPLAYERNAMESBYTEAMCODE` returns `BYE` for `<KEY>_B_2`, so it
reaches `CSV` as that side's player, and the console's Awards tab makes the
other side bronze without scores (`awards-podium-tab-spec.md` §2.3). It
takes a match number and a `SCHEDULE` slot like any match. The decisive
final is whichever of game 1 or game 2 was played last (that spec's §2.4);
game 2 is scheduled even though it is played only if the challenger wins
game 1, because the calculator's count is the worst case.

**Best-of-3** (`teams == 2`, §5.1). The two pairs are still listed as group
pairs `<KEY>_1` and `<KEY>_2`, so the roster scaffold works unchanged, but
they have no score grid, no `K` and no qualifier scaffold. After the gap row
and a `FINALS` banner come three blocks, `F (1)`, `F (2)` and `F (3)`, with
seats `<KEY>_F_1_(g)` and `<KEY>_F_2_(g)`. `F_1`'s name lookup reads group
pair 1 directly, `=IFNA(FILTER(B$6:B$9, A$6:A$9=$A$1&"_1"), A<r>)`, and `F_2`
the same against `"_2"`. `K` carries the final's formula. No bronze.

### 7.6 Two scaffolds, two draws

The roster scaffold (`AC`–`AI`) is the same three-step shape as the dual
meet's, one column set instead of two:

| Step | Col | State on a fresh workbook |
| --- | --- | --- |
| index | `AC` | `1 … teams`, on each pair's first row (from `AC5`), generator-written |
| **STEP 1 · NAMES** | `AD` | blank — operator pastes, 2 rows per pair |
| link | `AE` | `=FILTER($AI$5:$AI<end>, $AG$5:$AG<end>=AC<r>)`, mirrored on the second row |
| number | `AG` | `1 … teams`, one row per pair from `AG5`, generator-written |
| **STEP 2 · CODES** | `AH` | `=$AD$1&"_"&AG<r>`, one row per pair |
| **STEP 3 · RANDOMIZED** | `AI` | **blank** — operator pastes the codes back shuffled, one row per pair |

STEP 3 stays blank for the reason `sheet-generator.gs`'s `writeRosterScaffold_` gives:
pre-seeding it makes an undone step look done, and an unshuffled STEP 3 maps
every pair to its own roster slot, defeating the blinding.
`bracket-draw-name-import-spec.md` is the one thing allowed to fill it, since
there the order comes from a seeded, published draw. Every Pickle for Sight category tab
shows the cost of getting this wrong: STEP 3 prefilled in order (§2.1).

The **qualifier scaffold** (`AM`–`AP`) is the second draw, and has no
dual-meet equivalent — a dual meet's playoff entrants are decided by record,
so there is nothing to draw. Here the group-stage qualifiers are drawn by lot
into the playoff slots. The reference's headers are `1st PLAYOFFS` (`AM3`),
`LETTERS HERE` (`AM4`), `PLACE NAMES HERE` (`AO4`) and `PLACE 1ST PLAYOFFS
BUNUTAN HERE` (`AU3`, with the typo fixed). Pairs run two rows each from
row 5, one per qualifier:

| Col | Holds |
| --- | --- |
| `AM` | The **drawn slot number** for this qualifier — operator-written; generated as `-` |
| `AN` | First row: qualifier ordinal, `=Variables!AB<row>` (§7.3). Second row: the qualifier's **tier label**, generator-written (below) |
| `AO` | Qualifier player names, 2 rows per pair — operator-written |
| `AP` | `=IFNA(FILTER(AV:AV, AT:AT=AM<r>), "-")`, mirrored on the second row |

The generator writes exactly `qualifiers` rows (`plan.qualifiers`, wildcards
included), none for a round-robin-only or best-of-3 category, and the three
fixed rows of §7.5.1 for a twice-to-beat one. The tier label says which
finisher each row is waiting for, best tier first, in the order the ladder
seats them:

- tiered (`advance >= 2`, `fill: bye`): `Br 1 - Rank 1` … `Br <g> - Rank 1`,
  then `Rank 2 - 1st` … `Rank 2 - <g>th`, then `Rank 3 - …`. The runners-up
  are ranked against each other across brackets. This is the form `NMD` and
  `HIMD` carry, where the operator typed the winners as `Br 1`, `Br 3`,
  `Br 2`; the generator writes brackets in order.
- flat or full draw (`advance == 1`, or `fill: wc`): `Br <b> - Rank <r>` for
  every bracket and rank, then `Wildcard 1` … `Wildcard <wc>`.

The **playoff slot table** (`AT`–`AV`) is what `AP` resolves against:

| Col | Holds |
| --- | --- |
| `AT` | Slot number — a **fresh** seat's number, see below |
| `AU` | The stage the slot belongs to — `F`, `SF`, `QF`, `R16`, `R32` |
| `AV` | `=$AD$1&"_"&AU<r>&"_"&AT<r>` |

**The table lists fresh seats only, one row per qualifier.** Fed seats are
filled by formula and are never drawn, so they have no row. `HIMD`'s table is
the whole ladder of §7.5 in six rows: `1 SF`, `2 SF`, `3 SF`, `4 QF`,
`5 R16`, `6 R16`. The `Charts` tab's `PLAYOFFS OF 3 BRACKETS WITH TOP 2` tree
numbers the same seats `1` to `6`.

**The generator writes `AU`.** It is the one column that encodes the whole
tiered structure. Leaving it to the operator is what left `NWD`'s table
describing a quarterfinal it does not have (§2.1).

#### 7.6.1 Numbering the playoff seats

One rule gives every playoff code its number. It reproduces all four
reference ladders — the tiered bye (`NMD`, `HIMD`, BKL's `LI18MD`), the full
8-draw (`NXD`, `LIMD`, `LIXD`) and the bare semifinal (`NWD`, `LIWD`, `HIXD`)
— and, run against the calculator's own `playoffPlan`, it gives unique codes
and unique block labels to all 105 multi-bracket combinations (2-30
brackets × advance 1-4 × `bye`/`wc`, at most 32 qualifiers) that pass §9's
depth cap. The only label clashes it produces are in ladders past `R32`.

Inputs, from `playoffPlan`: the rounds earliest to latest, bronze excluded,
each with its match count `m`. Round `i`'s stage keyword counts back from the
end: `F`, `SF`, `QF`, `R16`, `R32`. Its **fed** seats number `m` of round
`i - 1` (winners coming up), 0 for the earliest round, and its **fresh** seats
are the rest, `2m - fed`: group qualifiers entering here, and byes.

1. Walk the stages **latest first** — `F`, then `SF`, `QF`, `R16`, `R32` —
   with one counter `c`, starting at 0, shared by all of them.
2. In each stage, number its fresh seats `c + 1`, `c + 2`, … and advance `c`.
   Order the fresh seats best tier first.
3. Lay the stage's seats out fresh seats first, then fed seats, and pair them
   in that order into matches: seats 1 and 2 are the first match, 3 and 4 the
   second, and so on.
4. A fed seat whose match partner is fresh number `n` takes `n + 1` if `n` is
   odd and `n - 1` if `n` is even — the other half of `n`'s pair — unless the
   stage already uses that number.
5. Every other fed seat takes the lowest number not yet used in its stage.
6. A match's block label is `<STAGE>-<j>` with `j = ceil(lowest number in
   the match / 2)`.
7. Wire each stage to the one before it (the next earlier round): the
   stage's fed seats, in ascending number, take the winners of the earlier
   stage's matches, in ascending `j`. `B` is fed by the `SF` losers as §7.5
   describes.

Worked for 3 brackets × top 2 (`HIMD`): `F` has no fresh seats. `SF` numbers
`1 2 3` fresh; the fed seat partners `3`, so `4`; blocks `SF-1 {1,2}`,
`SF-2 {3,4}`. `QF` numbers `4` fresh; its fed partner is `3`; block `QF-2`.
`R16` numbers `5 6`; block `R16-3`. `SF_4` takes `QF-2`'s winner and `QF_3`
takes `R16-3`'s. For a full draw of 8, `SF` has no fresh seats, so its fed
seats take `1 2 3 4` and `QF` numbers `1 … 8`, which wires `QF-k` to `SF_k`.

Fresh numbers are unique across the whole ladder, so `AT` alone identifies a
seat and one drawn number in `AM` resolves to exactly one code. Fed numbers
repeat across stages (`SF_3` and `QF_3` are different seats), which is fine,
because the site's `roundKeyword()` matches on the keyword, not the number.

### 7.7 No awards block

The category tab has **no** awards block. BKL's tabs carried one in `BA:BC`
(`Gold`/`Silver`/`Bronze`, each `FILTER(A:A, K:K=<medal>)`) for a printable
`Awards` tab to read. Neither Pickle for Sight workbook has either, and a
generated workbook follows them — the same decision Phase 1 spec §4.2 took
for the dual meet.

The podium lives in the console instead. Control Center's Awards tab
(`awards-podium-tab-spec.md`) derives it from the published snapshot:
gold and silver from the decisive `F` match, bronze from the `B` match. It
already handles both shapes that have no match to read:

- **Round robin only** (§5.2, `solo_format: rr`) — with no `F` match at all,
  it takes gold, silver and bronze from the standings (that spec's §2.5).
- **Twice-to-beat** (§5.2) — the third-placed pair's bronze is a `B` row
  whose opponent is a bye (that spec's §2.3).

Column `K` on the finals still carries `GOLD`, `SILVER` and `BRONZE` from the
§7.5 advance formulas. Nothing in the workbook reads them; they are for the
desk to see at a glance.

---

## 8. Team code grammar

One category key, one underscore-delimited suffix. No club segment.

| Slot | Code | Count |
| --- | --- | --- |
| Group pair | `<KEY>_<i>`, `i = 1 … teams` | `teams` |
| Playoff slot | `<KEY>_<STAGE>_<k>` where `STAGE ∈ {R32, R16, QF, SF}` and `k` is the seat number from §7.6.1 | 2 per playoff match |
| Bronze | `<KEY>_B_1`, `<KEY>_B_2` | 2 |
| Final | `<KEY>_F_1`, `<KEY>_F_2` | 2 |
| Twice-to-beat final | `<KEY>_F_1_(g)`, `<KEY>_F_2_(g)` for `g = 1, 2` | 4 |
| Twice-to-beat bronze walkover | `<KEY>_B_1`, and `<KEY>_B_2` named `BYE` (§7.5.1) | 2 |
| Best-of-3 final | `<KEY>_F_1_(g)`, `<KEY>_F_2_(g)` for `g = 1, 2, 3` | 6 |
| Round robin only | group pairs only | `teams` |

**Total codes per category = `teams` + 2 × the matches its playoff blocks
hold** — the calculator's playoff count, plus one for a twice-to-beat
category's bronze walkover. Verified against all fifteen reference
categories; BKL's `LI40WD`, which had no walkover row, carried `teams + 4`.

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
| `solo_format` empty, `ttb` or `rr` | §4; anything else is a hand-edited file |
| total playoff entrants ≤ 32 | The `Charts` reference tree tops out at 32 (§10.8), and no real category has come close |
| playoff ladder at most 5 rounds, bronze excluded — no stage before `R32` | §5.4: 15 `bye` shapes under the 32-entrant cap still reach `Round of 64` or deeper, which §8 has no code for. The message names the category and suggests `fill: wc` or fewer brackets |
| court labels non-empty, unique, ≤ 24 | §6.1, and the Dreamcourts defect in §2.1 |
| wave a positive integer per selected category | §10.2.4 |
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
| `C6` | **slot pitch as a time** — `duration_min + buffer_min` minutes. The Pickle for Sight workbooks label it `duration_min` and hold `0:25`, with `buffer_min` (`0`) in `C7`; every slot-time formula adds `Variables!$C$6` (§10.2, §10.6). Writing the sum here keeps those formulas as they are and still honours a non-zero buffer |
| `C8` | Facility label |
| `H1:P1` | Headers `type, key, value, teams, brackets, advance, fill, note, day` |
| `H11:P…` | The full plan block, one row per CSV category from row 11, in CSV order, and **nothing below it** — as in both Pickle for Sight workbooks. `STANDINGSCSV!G` reads `I` (key), `K` (teams) and `L` (brackets) to work out each pair's bracket (§10.4), so these columns must stay in place |
| `Q` | **New — `facility`.** Filled for the rows this workbook generated tabs for, blank otherwise |
| `R` | **New — court labels**, one per row from `R2`, so `SCHEDULE`'s row 5 can be a formula rather than 12 literals |
| `AB90…` | The reference-number ladder (§7.3), every second row |

`H:P` is written from the CSV verbatim, which structurally fixes the
`HI18WD`-labelled-as-Low-Intermediate defect in §2.1 the same way Phase 1
spec §4.3 fixed its counterpart. `solo_format` is
not a column here: it changes only the playoff, which the category tab
already shows.

### 10.2 `SCHEDULE` and `MATCHES`

The generator lays every category's matches out on a new **`MATCHES`** tab,
one category beside the next, and builds `SCHEDULE` as an **empty grid**.
The operator packs the day by copying court blocks from `MATCHES` into
`SCHEDULE` (§10.2.3). Packing by hand is how both Pickle for Sight schedules
were made, and it keeps the operator in control of which categories share
the courts when. Generating the packed schedule too is written down in
§10.2.4 as a possible future implementation.

#### 10.2.1 `SCHEDULE` — an empty grid

Geometry is the dual meet's — Phase 2 spec §2: rows 1-4 blank but for the
title band, row 5 the header band, data from row 6, a repeating 8-column
court block from column `D` (`D` names, `E` team 1, `F` match number under
`Court <n>`, `G` team 2, `H` names, `I`/`J` scores), a slot is two rows,
and the grid runs one row-pair past the last slot to carry the end time
(Phase 3 spec §3.2). One difference from Phase 2, taken from the Pickle for
Sight `SCHEDULE`s: the last court keeps its separator column, so the width
is **`8 · courts + 3`** (`AI` at 4 courts, `AQ` at 5). `STACKBLOCKS` gives the
same court count at either width.

**What the generator writes:**

- **The title band** in rows 1-3 — `D1 = Title!B6`, `D2 = Title!B7`, and
  `Powered by` / `SAGE Match Control Experts` in `D3`/`E3`. Cosmetic, but it
  is what the venue screenshots.
- **Court headings** in row 5, `="Court "&Variables!R<n>` from the court
  labels (§6.1), with `SCORE` over each block's `I`.
- **Slot times** in `B`: `B6 = Variables!C4`, then `B<r> = B<r-2>+Variables!$C$6`
  down every slot and the end-time row.
- **Every court block on every slot:** the name formulas
  `D<r> = GETPLAYERNAMESBYTEAMCODE(E<r>)` and
  `H<r> = GETPLAYERNAMESBYTEAMCODE(G<r>)`, which spill the pair's two names
  down both rows of the slot (there is no `vs` row), plus the merges, column
  widths, row heights and the category-colour rules. A blank code shows blank
  names.

**What it leaves blank:** the team codes in `E`/`G`, the match numbers in
`F`, and the scores.

**Height.** `N` slots, where `N` is the slot count the §10.2.4 packer
reaches for this workbook's categories, **plus 4 spare**. The packer needs
33 slots for PFS Main and 29 for PFS Annex, so `N` is 37 and 33. (The
operators' own hand layouts used 31 and 30.) A hand layout can be looser or
tighter than the packer's, and trailing empty slots cost nothing:
`Fill match numbers` skips them. Running short costs more, because
`SCHEDULE` and `Timeline` (§10.6) both have to be extended by hand. The
generator runs the packer in memory only to get `N` — nothing it computes is
written to `SCHEDULE`.

#### 10.2.2 `MATCHES` — every category's matches, side by side

A new tab with **the same geometry as `SCHEDULE`**: 8-column court blocks
from `D`, two-row slots from row 6, the same name formulas in `D`/`H` and the
same merges. The shared geometry is what makes packing a copy and paste
(§10.2.3). Nothing reads `MATCHES`: every named function, `Fill match
numbers`, the sync and every readout read `SCHEDULE` by name, so the codes
here count for nothing until they are pasted there.

**A band per category.** Categories sit side by side in CSV order, each in a
band of consecutive court blocks, no gap between bands. A category's band is
`c` blocks wide:

```
c = max( Σ over brackets of floor(n_b / 2),       one full round-robin round
         the most matches in any playoff row,     a row never needs to wrap
         1 )
```

**A row per round.** Down its band, a category's rows are:

1. **Round-robin rounds**, one row-pair per round, `max over brackets of
   (n_b odd ? n_b : n_b - 1)` rows. Row `r` holds round `r` of every bracket,
   bracket 1's matches in the leftmost blocks. Rounds come from the circle
   method: fix pair 1, rotate the rest; a bracket of odd `n_b` gets a phantom
   entrant, and whoever draws it sits that round out. A bracket with fewer
   rounds than the category's largest leaves its blocks blank in the later
   rows.
2. **One row per playoff stage**, earliest first (`R32` → `SF`), holding
   the stage's matches in block-label order (`QF-1`, `QF-2`, …, §7.6.1).
3. **The finals row:** `F`, then `B` beside it when there is a bronze
   match — every Pickle for Sight category with a bronze match plays it in
   the same slot as the final. A twice-to-beat category takes two rows:
   `F (1)` with the bronze walkover beside game 1, then `F (2)` (§7.5.1). A
   best-of-3 takes three, `F (1)` to `F (3)`.

No pair appears twice in any `MATCHES` row, so any row can be pasted into a
single `SCHEDULE` slot as it is.

| Cell | Content |
| --- | --- |
| row 3, band's first block | The category's display name, `=FILTER(Variables!$J:$J, Variables!$I:$I="<KEY>")`, merged across the band |
| row 5, each block's `F` | `<KEY> <k>` for the band's `k`-th block, e.g. `NMD 1` … `NMD 6` — never `Court …` |
| `B` on each row | The row's label: `RR 1` … `RR <n>`, then the stage keyword (`R16`, `QF`, `SF`), then `F + B` (or `F (1)`, `F (2)`, …) |
| rows 1-2 | `A1 : MATCHES`; `A2 : Total matches` |
| `E`/`G` | The match's two codes |
| `F` | Blank — match numbers exist only on `SCHEDULE` |
| an unused block | `-` in `E`/`F`/`G`, as on `SCHEDULE` |
| `B2` | The number of matches on the tab, e.g. `116`, as a plain number — what `Timeline!B3` must read once packing is done (§10.2.3). It is the calculator's total plus one per twice-to-beat category, for its bronze walkover |

Column `B` holds labels, not times: `MATCHES` has no clock. The category
colour rules apply here as on `SCHEDULE`, so each band reads in its
category's colour.

For the Pickle for Sight categories:

| Venue | Category | Brackets | Band | RR rows | Playoff rows | Rows |
| --- | --- | --- | --- | --- | --- | --- |
| Main | `NWD` | 4-4 | 4 | 3 | `SF`, `F + B` | 5 |
| Main | `NMD` | 5-5-5 | 6 | 5 | `R16`, `QF`, `SF`, `F + B` | 9 |
| Main | `NXD` | 5-5-5-5 | 8 | 5 | `QF`, `SF`, `F + B` | 8 |
| Main | `LIWD` | 4-4 | 4 | 3 | `SF`, `F + B` | 5 |
| Annex | `LIMD` | 5-5-5-4 | 8 | 5 | `QF`, `SF`, `F + B` | 8 |
| Annex | `LIXD` | 5-5-4-4 | 8 | 5 | `QF`, `SF`, `F + B` | 8 |
| Annex | `HIWD` | 3 | 1 | 3 | — | 3 |
| Annex | `HIMD` | 5-4-4 | 6 | 5 | `R16`, `QF`, `SF`, `F + B` | 9 |
| Annex | `HIXD` | 3-3 | 2 | 3 | `SF`, `F + B` | 5 |

Main's `MATCHES` is 22 court blocks wide and 9 slot rows tall; the Annex's is
25 wide and 9 tall.

#### 10.2.3 Packing `SCHEDULE` by hand

The operator's procedure, written into the generator's completion message
and the dry-run checklist:

1. **Copy court blocks from `MATCHES` into `SCHEDULE`.** Select one or more
   whole court blocks on a `MATCHES` row — both rows of the slot, starting at
   a block's `D` column — and paste them with **Paste special → Formula
   only** at the start of a court block on a `SCHEDULE` slot's first row.
   That brings the codes and the name formulas, which still point at their
   own `E`/`G`, and nothing else: no formats, merges or colour rules, which
   `SCHEDULE` already has and which a plain paste would pile up. `SCHEDULE`'s
   own colour rules colour the codes. A partial row is fine: pasting a
   category's round across two slots, or splitting a row between categories,
   only has to respect the rules below.
2. **Keep the rules the generator used to guarantee.** None of these is
   checked while pasting:
   - **no pair twice in one slot** — `Timeline` (§10.6) shows a `2`;
   - **a playoff stage only after the stage before it has finished,** and
     the first stage only after the category's whole round robin, ideally
     with one slot free for the qualifier draw (§7.6);
   - **a category's final and bronze in the same slot;**
   - **`XD` after `MD`/`WD`** at the same venue, for players entered in both.
     Nothing checks this one at all (§15).
3. **Clear the rest.** Leave unused courts blank or `-`.
4. **Run `SAGE → Fill match numbers`** with the venue's base (§6.2). It
   numbers every slot with a code on both sides, puts `-` in empty ones, and
   writes the same numbers into `CSV!A`, copying `CSV` row 2's formulas down
   to add a row per match (§10.5). Re-run it after any later move.
5. **Check `Timeline`:** its grand total (`B3`) equals `MATCHES!B2`, and no
   body cell exceeds 1. A match pasted twice makes the total too high and
   shows as a code with too many matches in `Timeline`'s `A` column; one
   never pasted makes the total short.
6. Set up live sync (`pickle-for-sight-spec.md` §12.2).

#### 10.2.4 Possible future: generating the packed schedule

Two ways the generator could fill `SCHEDULE` itself, recorded so the design
does not have to be rediscovered. Neither is built. The packer below already
runs, in memory, to size `SCHEDULE` (§10.2.1); the generation log records its
result as a suggested layout the operator can follow.

- **Option 1 — generated schedule, `MATCHES` as documentation.** The
  generator packs `SCHEDULE` with the algorithm below and numbers it. `MATCHES`
  stays as the readable plan behind it.
- **Option 3 — generated from `MATCHES`, operator adjusts.** The packer
  reads its queues straight from `MATCHES` instead of rebuilding them. An
  operator who reorders rows there re-runs a `SAGE → Pack schedule` command
  and `SCHEDULE` follows. That only works while `SCHEDULE` can still be
  rebuilt — before the day goes live and scores exist.

**Why waves.** A player can enter one same-gender doubles category and one
mixed. Pickle for Sight schedules every `XD` category after every `MD` and
`WD` category at the same venue is finished: on Main, `NXD` starts in the
slot after the `NMD` and `NWD` finals; on the Annex, `LIXD` starts in the
slot after `HIWD`'s last match and `HIXD` one slot later. Nobody is due on
two courts at once, and nobody finishes a final and walks straight into a
group match. The wave is the sidebar's per-category input (§6), defaulting
to `2` for an `XD` key and `1` for everything else.

**The packer.**

1. **One queue per category** — its `MATCHES` rows, top to bottom, each row
   left to right (§10.2.2). The finals row stays one unit: `F` and `B` are
   always placed in the same slot. A twice-to-beat category's second game and
   a best-of-3's later games each go one slot after the game before.
2. **Waves in order.** Wave `w + 1` starts in the slot after wave `w`'s last
   occupied slot. Within a wave, fill slots one at a time, courts left to
   right. For each slot, visit the wave's categories that still have queued
   matches, in CSV order, starting from category `t mod k` for slot index `t`
   and `k` categories left — so categories take turns going first. From each
   visited category take matches off the front of its queue while they are
   eligible and courts remain, and stop at that category's first ineligible
   match: a queue is never reordered.
3. **Eligibility.** A match is eligible for slot `t` when:
   - neither of its pairs is already in slot `t` — the hard constraint;
   - it is a playoff match, and every match of the same category's previous
     stage is in a slot before `t` — a stage cannot overlap the one before
     it, because its entrants are that stage's winners;
   - it is the category's **first** playoff stage, and its last round-robin
     match is at least two slots before `t` — one clear slot for the
     qualifier draw (§7.6). This is waived when no other category in the wave
     has an eligible match for slot `t`, so a category alone in its wave does
     not leave the courts idle. Pickle for Sight leaves that slot in six of
     its eight playoff categories.
4. **Match numbers** (options 1 and 3 only) are the position in placement
   order, offset by the base — the same order `Fill match numbers` uses.

Partial slots are **centred**, per Phase 2 spec §4.3. Soft preference, not
enforced: at least one idle slot between a pair's matches. Pickle for Sight
mostly gives one — taking turns in step 2 is what produces that — but
breaks it for 17 pairs across the two workbooks (`NWD` pairs 1-4 play slots
13 and 14). If either option is built, the §9 validation gains a per-wave
warning when `courts` exceeds the wave's matches in one round-robin round,
since that wave would leave courts idle in every round-robin slot (Phase 2
spec §4.4's guarantee, restated).

**Category colours** carry over from Phase 2 spec §6, including the
conditional-format mechanism, on both `SCHEDULE` and `MATCHES`, with one
change to the test. A dual-meet code
is `<CLUB>_<KEY>_<n>`, so `colorFormulaFor_` compares
`INDEX(SPLIT(code,"_"),2)`. A standard code is `<KEY>_<n>` with no club
segment, so the key is token **1**: the standard generator's copy compares
`INDEX(SPLIT(code,"_"),1)`. Copied unchanged, the test compares the pair
number against the key and nothing is ever coloured. The level ladder in Phase
2 spec §6.2 is `{N, LI, I, HI, A}`, which already covers every Pickle for
Sight key (`N`, `LI`, `HI` × `MD`/`WD`/`XD`). It needs two rungs added for the
BKL level codes (`B` beginner, `AB` advanced beginner), and the age-band keys
(`LI40MD` → level `LI40`, type `MD`) mean Phase 2 spec §6.4's parse has to
strip a trailing digit run from the level before looking it up. A key that
still does not resolve gets no fill, per that section.

### 10.3 `Court Control`

Court blocks only, as in both Pickle for Sight workbooks: `B4`/`C4` read
`Court`/`Match`, then one 3-row block per court from row 5. In each block
`B` is the court's **label** from `Variables!R` (§6.1) — `1`-`4` on Main,
`5`-`9` on the Annex — not its position, since `CSV!B` publishes it as the
live court (§10.5). `C` is the operator-typed match number, `D`/`E` the codes
via `GETTEAM1CODEBYMATCH`/`GETTEAM2CODEBYMATCH`, and the names spill below
through `GETPLAYERNAMESBYTEAMCODE`. `A1` reads `COURT CONTROL` and `B2`
`Matches Currently Running`.

Nothing else. The dual meet's stats block (Phase 3 spec §5.1) and BKL's
standby strip (`Next Stand By`, `TBA Stand By`, `Match to Record`) are gone
from both Pickle for Sight workbooks and are not generated. The tab's height
is `3 · courts + 4` rows.

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
`n`. Uneven bracket sizes (§5.6) make that wrong. Instead it recomputes
`groupSizes` from the category's plan row: `teams` in `Variables!K` and
`brackets` in `Variables!L`, keyed by `Variables!I` (§10.1):

```
G2 = IF($A2 = "", "",
  IFERROR(
    LET(
      key,   INDEX(SPLIT($A2, "_"), 1),
      idx,   VALUE(REGEXEXTRACT($A2, "^[^_]+_(\d+)$")),
      pairs, VLOOKUP(key, Variables!$I:$K, 3, FALSE),
      want,  VLOOKUP(key, Variables!$I:$L, 4, FALSE),
      nb,    MAX(1, MIN(IF(want = "", 1, want), INT(pairs / 2))),
      size,  INT(pairs / nb),
      extra, MOD(pairs, nb),
      big,   extra * (size + 1),
      IF(idx <= big,
        ROUNDUP(idx / (size + 1), 0),
        extra + ROUNDUP((idx - big) / size, 0))
    ),
  "")
)
```

This is the formula both Pickle for Sight workbooks already run, character
for character, on every `STANDINGSCSV` row.

It rests on two facts from earlier sections:

- **Codes run in bracket order.** Group codes are `<KEY>_1 … <KEY>_<teams>`
  in one unbroken run, with bracket 1's pairs first (§7.2). So a pair's
  bracket follows from its index `i` alone.
- **The split is `groupSizes`'s.** `nb` applies the same `floor(teams/2)`
  cap (§5.6), with a blank `brackets` meaning one bracket. The first `extra`
  brackets take `size + 1` pairs and the rest take `size`. 26 pairs over 6
  brackets is `5-5-4-4-4-4`, `LI40MD`'s 9 over 2 is `5-4`, and `LI40XD`'s 7
  over 2 is `4-3`, and Pickle for Sight's `HIMD` 13 over 3 is `5-4-4`. So `G`
  agrees with the category tab's `K` column and its score grids.

Worked through for `LIMD_11` at 26 pairs over 6 brackets: `size = 4`,
`extra = 2`, `big = 10`. Pairs 1–10 fill the two brackets of 5, and pair 11
is the first of the brackets of 4: `2 + ceil(1/4) = 3`.

**Only group codes get a bracket.** The pattern `^[^_]+_(\d+)$` matches a
code that is exactly `<KEY>_<number>`. It is anchored at both ends because
playoff codes such as `<KEY>_SF_1`, `<KEY>_QF_2` or `<KEY>_R16_3` also end
in a number, and an unanchored `_(\d+)$` would give them a bracket. They
fail to match, as do `<KEY>_B_1`, `<KEY>_F_1` and `<KEY>_F_1_(2)`, and
`IFERROR` blanks them. That is correct, since a playoff entrant belongs to no bracket.

`G` is the bracket **number**, `1`, `2`, `3`, the same convention as the dual
meet's `STANDINGSCSV` and the numbers the generator writes into the category
tab's `K`. The site treats the column as an opaque grouping label. The `LET`
names are letters only, since a name like `b1` parses as a cell reference and
`LET` rejects it.

The generator writes nothing extra for this. The formula reads the plan row
the generator already writes, so there is no separate code → bracket map to
keep in step with the split.

Row count is the sum of every category's codes (§8): 75 for BKL
Dreamcourts, 95 for PFS Main and 111 for PFS Annex.

The `A2` spill must read `'Reference for Players'!$A$3:$A` open-ended, per
Phase 3 spec §7. Both Pickle for Sight workbooks read `$A3:A1553`, which
works today and silently truncates the day a longer category is added.

### 10.5 `CSV`

Phase 3 spec §5.3's tab: **12 columns, `A:L`**, the same header names as the
dual meet's. The BKL workbooks had a thirteenth, a literal `v` between the two
teams; the Pickle for Sight workbooks dropped it, and the generator follows
them. `GvizCsvFetcher` reads by header name, so neither shape breaks the sync.

| Col | Header | Row 2 |
| --- | --- | --- |
| `A` | `matchNumber` | blank — `Fill match numbers` writes it (§10.2.3) |
| `B` | `court` | `=IFNA(FILTER('Court Control'!$B:$B,'Court Control'!$C:$C=$A2),"")` |
| `C` | `Schedule` | `=MATCHTIME($A2)` |
| `D` | `CourtAssignment` | `=MATCHCOURT($A2)` |
| `E` | `teamCode1` | `=GETTEAM1CODEBYMATCH($A2)` |
| `F` `G` | `team1Player1/2` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($E2),1)` / `,2)` |
| `H` | `team1Score` | `=GETSCOREAGAINSTPAIR($E2,$I2)` |
| `I` | `teamCode2` | `=GETTEAM2CODEBYMATCH($A2)` |
| `J` `K` | `team2Player1/2` | as `F`/`G` off `$I2` |
| `L` | `team2Score` | `=GETSCOREAGAINSTPAIR($I2,$E2)` |

`B` reads `Court Control`'s whole columns, as both Pickle for Sight
workbooks do, so it covers every court block at any court count. Nothing
else on that tab has a number in `C` (§10.3).

The generator writes the header and **one** formula row, with `A2` blank.
`Fill match numbers` copies row 2 down to one row per placed match and
numbers them (§10.2.3); until it runs, the tab publishes nothing.

`C` and `D` **must** be the `MATCHTIME`/`MATCHCOURT` named functions (§11),
as they already are in both Pickle for Sight workbooks. The BKL reference's
`BYROW`/`BYCOL` scan over a hardcoded `SCHEDULE!$F$6:$BZ$78` is the third
defect in §2.1 and cannot survive a court count it was not typed for.

The dual meet's `MATCHCOURT` returns `"Court " & <block index>`, which is the
block's *position*, not its label. With non-contiguous court labels (§6.1)
that is wrong: Dreamcourts' block 1 is Court 10, and a PCPH Annex workbook
whose courts are 5–9 reports its block 1 as Court 1. The standard library's
`MATCHCOURT` reads the label the block itself carries in `SCHEDULE` row 5
instead. This is the definition both Pickle for Sight workbooks run, and it
reads `Court 5` for the Annex's first match:

```
MATCHCOURT = LAMBDA(match_number, IFERROR(
  LET(
    h,     ROWS(SCHEDULE!$B$6:$B),
    pos,   MATCH(match_number, GETMATCHNUMBERS(), 0),
    blk,   INT((pos - 1) / h),
    label, INDEX(SCHEDULE!$5:$5, 1, 6 + 8 * blk),
    "Court " & REGEXEXTRACT(TO_TEXT(label), "\d+")
  ),
  "Not found"))
```

- `blk` is the 0-based block index, as before. `GETMATCHNUMBERS()` is
  `STACKBLOCKS(6, 8, 6)`, so block `blk`'s match-number column is
  `6 + 8 * blk`. That is the `F` offset, where row 5 carries the court
  heading (Phase 2 spec §2.2).
- It reads row 5 rather than `Variables!R` directly, so the result is
  always the label the operator sees on `SCHEDULE`. Row 5 is itself
  `="Court "&Variables!R<n>` in a generated workbook, so the two agree. A
  hand-edited heading, or a hand-built workbook with no `Variables!R`, still
  resolves correctly.
- `REGEXEXTRACT(..., "\d+")` normalises the heading to `Court <n>` whether
  row 5 reads `Court 5` or a bare `5`. `Court <n>` is the form the site
  parses: it takes the trailing integer of `CourtAssignment`.
- The `LET` names are letters only. A name like `b1` parses as a cell
  reference and `LET` rejects it.

`Court Control`'s column `B` (§10.3) must carry the same labels, not
`1..courts`: it is what `CSV!B` publishes as the live `court`. A 5–9 venue
numbered 1–5 there shows its live matches on another venue's courts.

### 10.6 `Timeline`

**`Timeline`** is Phase 3 spec §5.2's pair × slot conflict grid, in the
layout both Pickle for Sight workbooks use:

| Cell | Content |
| --- | --- |
| `A1` | `TIMELINE` |
| `B2`, `B3` | `TOTAL`, and the match count `=SUM($C$4:<lastCol><lastRow>)/2` — the `/2` because every match appears twice |
| `C3`, `D3`, … | slot times: `C3 = Variables!C4`, then `D3 = C$3+Variables!$C$6`, one column per slot |
| `B4` | `=UNIQUE(FILTER('Reference for Players'!$A$3:$A, 'Reference for Players'!$A$3:$A <> ""))` — every code, plus each anchor's key row |
| `A<r>` | `=SUM(C<r>:<lastCol><r>)` — that code's match count |
| body | `=COUNTPAIRAT(C$3,$B4)` |

The body replaces the BKL workbooks' 24-term `COUNTIFS` chain (§2.1), and the
Pickle for Sight workbooks already run it, with the grand total over the
whole body (`SUM(C4:AH115)/2` on Main, `SUM(C4:AF132)/2` on the Annex).
`<lastCol>` and `<lastRow>` are the last slot's column and the last code's
row: one time column per `SCHEDULE` slot, all `N` of them including the
spares (§10.2.1), and one row per code in `Reference for Players`. One thing the generator does differently: the `B4` spill reads
`$A$3:$A` open-ended, where the Pickle for Sight workbooks read the bounded
`A2:A115` (Annex `A2:A132`).

`Timeline` checks **codes**, not people. A pair's playoff entry gets a new
code (§8), and a player entered in two categories has two unrelated codes,
so a human booked on two courts in one slot reads as two rows of 1 each.
BKL Dreamcourts had a `Timeline (Individual)` tab keyed by player name to
catch exactly that. Neither Pickle for Sight workbook has it, and it is not
generated; §15 records the gap.

BKL Dreamcourts also carried a **`Standings`** wall-display board. Neither
Pickle for Sight workbook has it, nothing reads it, and the public event page
shows live standings. It is not generated and not in the master.

Nor is an **`Awards`** tab. BKL had one; neither Pickle for Sight workbook
does, and the podium comes from the console (§7.7).

### 10.7 `Title`, `Reference for Players`, `Poster`

**`Title`** — `B6 = Variables!C2` and `B7 = TEXT(Variables!C3,"MMM DD")&" - "&Variables!C8`,
both formulas, and nothing else. The generator writes only `Variables`.

**`Reference for Players`** — Phase 1 spec §4.5's tab, one anchor per category
tab, in the form both Pickle for Sight workbooks use: the key in `F`, and a
formula in `A` that reads the tab through `INDIRECT`, so every anchor is the
same text.

```
F3 : NMD
A3 = FILTER(INDIRECT("'"&F3&"'!$A$1:$C"), INDIRECT("'"&F3&"'!$A$1:A") > "")
```

The generator writes one anchor per **generated** category, at **cumulative
offsets computed from each tab's actual height**. The reference hand-places
anchors for all nine of the event's categories in both workbooks, at `A3`,
`A106`, `A262`, `A377`, …, `A1009`, so the anchors for the other venue's
categories resolve to `#REF!` (§2.1). Each generated workbook has anchors
for its own categories only.

Both downstream spills (`STANDINGSCSV!A2`, `Timeline!B4`) must read
`$A$3:$A` open-ended, per Phase 3 spec §7.

The BKL workbooks carried a static **`QA Checklist`** tab. Neither Pickle
for Sight workbook has it, and it is not in the master.

**`Poster`** — images only. Ships blank in the master.

### 10.8 `Charts`

A static parent → child bracket tree, `PARENT`/`CHILD` pairs down `B`/`C`,
for a 32-pair ladder and for the irregular ladders the tiered byes produce
(`PLAYOFFS OF 23 PAIRS`, with slot names like `QF_R16-3` where a bye spans
two rounds). The Pickle for Sight copies, identical in both workbooks, add
`PLAYOFFS OF 3 BRACKETS WITH TOP 2` at `A243`: seats `1 2 3` into the
semifinals and `4` into the quarterfinal that feeds `SF-4`, with `5` and `6`
playing in below it. That is §7.6.1's numbering for the `NMD`/`HIMD` shape.

**Ships in the master unchanged and is not generated.** It is a reference the
operator reads while filling the playoff slot table by hand. Once the
generator writes `AU` itself (§7.6) the tab becomes documentation rather than
a working surface, which is the right direction — but deleting it is out of
scope here, and the irregular trees on it are the clearest existing statement
of what §5.4 produces.

---

## 11. The Named Function library

Replace the BKL workbooks' 15-function library with the Dual Meet Master's
23, and take the whole `technical/named-function-library.md` contract
as-is, with two changes: `MATCHCOURT` is the row-5 label form in §10.5, and
`SORTBYWINS` is left out. That is **22 functions**. Both Pickle for Sight
workbooks run the 22 plus two dead ones (below), read out of their
`<definedNames>` and identical in both. The three that matter most:

| Function | Why |
| --- | --- |
| `STACKBLOCKS(first_col, step, first_row)` | Derives the court count from the sheet's own width. Removes the hand-edited column lists in §2.1 and the whole class of bug behind them |
| `MATCHTIME(m)` / `MATCHCOURT(m)` | Replace `CSV!C`/`D`'s hardcoded `BYROW`/`BYCOL` window. `MATCHCOURT` needs the row-5 label variant in §10.5 |
| `COUNTPAIRAT(time, code)` | Replaces `Timeline`'s 24-term `COUNTIFS` chain |

Every call site in both Pickle for Sight workbooks was counted, in cells and
inside the other functions' definitions. Twenty-two functions are reached.
Two are not, and neither is carried:

| Function | Why it is dead |
| --- | --- |
| `GETWINSCORE()` → `11` | Inherited from the BKL library. No cell and no function calls it |
| `SORTBYWINS(range)` | The dual meet's playoff feeders call it to rank a group block. A standard tournament's playoff entrants are drawn (§7.5), so nothing sorts |

Of the 22, ten are called only by other functions: `STACKBLOCKS`, the four
column primitives, `GETMATCHNUMBERS`, `GETMATCHES`,
`GETMATCHRESULTSBYTEAMCODE`, `GETTEAMCODES` and `GETPLAYERNAMES`. They look
unused from the tabs and are not. The BKL library's
`GETTEAM1SCOREBYMATCH`/`GETTEAM2SCOREBYMATCH` and `GETPLAYERBYRANK` are gone
from the Pickle for Sight library too.

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

`parseCsvRows_`, `colLetter_`,
`parseTimeToMinutes_`, `minutesToClock_`, `trace_`, `logStep_`, `resetLog_`,
`getGenerationLog`, `ensureRowCount_`, `ensureColumnCount_`, `mergeVertical_`,
`stampFormat_`, `assertTabsPristine_` and the `REBUILT_IN_PLACE_TABS` shape,
`categoryColor_` (with §10.2's level-ladder extension), and the whole
`buildScheduleTab_` tiling strategy — `copyTo` on both axes, grow down then
across, flush between stages, copy column widths and row heights explicitly
(Phase 2 spec §8.5, §8.10, §8.11).

Copied, then changed:

- `parsePlanCsv` reads only the dual-meet fields (`key`, `value`,
  `brackets`, `po_format`, `teams_a`, `teams_b`). The standard copy reads
  `teams` (index 3), `brackets` (4), `advance` (5), `fill` (6), `note` (7)
  and `solo_format` (11) instead, with the same positional, missing-cell-is-
  empty parsing.
- `colorFormulaFor_` tests `INDEX(SPLIT(code, "_"), 1)`, not `…, 2)` —
  a standard code has no club segment (§10.2).

Rewritten for this shape: `computeLayout`, `buildCategoryTab`,
every `write*Block*_`, every feeder builder, and `readoutGeometry_`. New:
`buildMatchesTab` (§10.2.2), and `packSchedule_`, the §10.2.4 packer, which
is pure, returns a slot list, and today only sizes `SCHEDULE` and feeds the
suggested layout in the log. `buildScheduleTab_`'s tiling writes the empty
grid of §10.2.1.

### 12.3 Rebuilt in place

Same registry and same pristine guard as Phase 3 spec §8.2. `SCHEDULE` and
`Court Control` keep the hard constraint that makes it necessary: their tab
GIDs are what the live sync trigger watches, stored in that workbook's
`SYNC_WATCHED_GIDS` Script Property (`sync-script-configuration-spec.md` §5),
so replacing either kills the sync while the sheet still looks correct.

| Tab | Prototype |
| --- | --- |
| `SCHEDULE` | 1 court block, 1 slot |
| `Court Control` | 2 court blocks |
| `Timeline` | header rows 1-3 + 1 body row, stripped to `A:D` |
| `CSV` | header row + 1 match row, `A:L` |
| `STANDINGSCSV` | header row + 1 pair row, `A:G` |
| `Reference for Players` | 1 anchor |

`MATCHES` is not in this table. Like a category tab it is a **new** tab,
inserted by the generator; the master has none, and a workbook that already
has one fails §9's tab-collision check.

Phase 3 spec §8.2's warning about `Timeline`'s width applies unchanged: its
built column count scales with the event, so a prototype left wide lets an
already-built tab read as pristine and get tiled twice.

### 12.4 Ordering

Category tabs → `Variables` → `Title` → `Reference for Players` →
`MATCHES` → `SCHEDULE` → the readouts. `Reference for Players` needs each
category tab's final spill height; everything else needs `SCHEDULE`'s
returned geometry.
`SCHEDULE` runs late for the blast-radius reason in Phase 2 spec §8.1.

### 12.5 Tracing and failure

Phase 2 spec §8.9 and Phase 1 spec §11.3, unchanged. Every computed extent,
every block anchor, every `MATCHES` band, the packer's suggested layout
slot by slot, and — new here — the resolved
playoff ladder per category as `<KEY> <STAGE> slot <k> <- fresh | winner of <STAGE>-<j>`.
A wrong ladder is invisible in a finished tab unless you already know which
slot was expected.

On failure mid-build: no rollback, no continuing. Report which tabs were
written.

---

## 13. Build order

1. `parsePlanCsv` + §9 validation, reporting every failure together
2. **`planCategory(cat)`** — pure. Bracket sizes, RR match list per bracket,
   the full playoff ladder from §5 and its seat numbering from §7.6.1.
   Verified by hand against §1.1's fifteen reference categories before any
   Sheets code exists
3. The master workbook (§14.1) — the larger half of the work
4. `buildCategoryTab` — group blocks, score grids, the playoff ladder,
   both scaffolds
5. `Variables` (§10.1), `Title`, `Reference for Players`
6. `buildMatchesTab`, `packSchedule_` (for the height) and the empty
   `SCHEDULE` grid (§10.2)
7. `CSV` and `STANDINGSCSV` — the two the live sync needs
8. `Court Control`, `Timeline`

Steps 1-2 are worth doing and checking on their own: `planCategory` is where
every §5 branch lives, it is the only part with real logic, and it can be
verified against two played events without a workbook. Pickle for Sight's
two plan CSVs are the fixtures: feeding each through steps 1-2 must give
§1.1's match counts and §7.6's slot tables exactly.

---

## 14. What the master workbook must carry

A one-time manual job, and the larger half of the work — Phase 1 spec §6's
counterpart.

### 14.1 Building it

1. Copy the Pickle for Sight **PCPH Annex** reference workbook (§1). It
   already runs §11's library and holds four of the five playoff shapes:
   tiered (`HIMD`), full draw (`LIMD`, `LIXD`), bare semifinal (`HIXD`) and
   round robin only (`HIWD`).
2. Delete the two dead named functions, `GETWINSCORE` and `SORTBYWINS`
   (§11), leaving 22 with `MATCHCOURT` in its row-5 form.
3. Replace the five category tabs with a single hidden
   `_CATEGORY_TEMPLATE`: an **unchanged copy of the `HIMD` tab**. It holds
   one of every block type the generator formats from (§18), so nothing is
   trimmed, moved or added.
4. Point the `Timeline!B4` and `STANDINGSCSV!A2` spills at
   `'Reference for Players'!$A$3:$A`, open-ended.
5. `Variables`: add column `Q` (facility) and `R` (court labels) with
   headers, and relabel `B6` to say it holds the slot pitch (§10.1).
6. Strip every rebuilt-in-place tab to §12.3's prototype shape, and
   `Reference for Players` to one anchor — **last**, then generate once
   immediately, per Phase 3 spec §12.
7. Name it **SAGE Standard Tournament Master**.

### 14.2 Decisions taken in the master, not in code

- **No awards block and no `Awards` tab** (§7.7). The console produces the
  podium from the snapshot, round-robin-only and twice-to-beat included.
- **Column `K` keeps its double duty** — bracket number on each bracket's
  first group pair, next-round code and medal label on playoff rows. It is
  genuinely two columns' worth of meaning in one, but the whole
  forward-propagation chain (§7.5) keys off it, and splitting it would touch
  every playoff formula to fix nothing that is broken.
- **`Charts` ships as-is** (§10.8).
- **`Standings`, `Timeline (Individual)` and `QA Checklist` are not
  carried** (§10.6, §10.7). Neither Pickle for Sight workbook has them.

---

## 15. Open questions

**`R32` is not in the site's `STAGE_ORDER`.** The standard template's
`STAGE_ORDER` is `['R16','QF','SF','BRONZE','FINAL']` and
`ROUND_KEYWORD_LABELS` has no `R32` entry, so a category deep enough to reach
one renders those rows in an unsorted trailing stage with no label. Adding
both is a two-line site change, but it needs an event that actually reaches
R32 to test against, and none has. Validation caps both entrants (32) and
ladder depth (no stage before `R32`) in §9, so nothing deeper can be
generated.

**Nothing checks a player booked twice.** `Timeline` works on codes
(§10.6), and a human has a different code in each category and in each
playoff stage. Packing `XD` after `MD`/`WD` (§10.2.3) keeps same-gender and
mixed doubles apart, which is the common case, but that is the operator's
rule to keep, and a bad qualifier draw or a hand-moved match can still put
one player on two courts at once. Nothing in the workbook flags it.
BKL's `Timeline (Individual)` did, by counting player names per slot row.
Whether to bring it back is undecided; Pickle for Sight ran without it.

**Drawing within tiers.** In a tiered ladder the qualifier draw is not one
lot: group winners must land on the winners' seats (`1 … g`, §7.6.1) and
runners-up on the rest. The scaffold's tier labels (§7.6) tell the operator
which is which, but nothing stops a runner-up being given seat `1`. A check
column beside `AM` that flags a seat outside its row's tier would catch it;
whether that is worth the extra formulas has not been decided.

**Whether the qualifier scaffold should be drawn in a tool.**
`bracket-draw-name-import-spec.md` fills STEP 3 of the roster scaffold from a
Bracket Generator draw. §7.6's `AM` column is a *second* draw with the same
shape and the same hand-shuffling problem, but no artifact to import: it
happens after the round robin, on the day, from the standings. Whether it
should go through the tool at all, or get its own in-sheet draw, is open —
that spec's §8.3 gives the reasoning and its §13 keeps it out of scope.

**Per-facility court label ordering.** Only matters if §10.2.4's packer is
ever built. §6.1 takes the labels in schedule order, and the packer would
fill courts in that order. Whether a partial slot should centre on the
*label* order or on the physical court order is undecided and has never
mattered.

---

## 16. Acceptance

Run against the two Pickle for Sight plan CSVs (§1) and confirm:

- The **MAIN** CSV, all four categories selected, facility `PCPH Main`,
  courts `1,2,3,4`: four category tabs; a `MATCHES` tab 22 court blocks wide
  and 9 slot rows tall with bands of 4, 6, 8 and 4 blocks for `NWD`, `NMD`,
  `NXD`, `LIWD`, and `MATCHES!B2` reading 116; an empty `SCHEDULE` of 37
  slots; and a `CSV` of one formula row
- The **ANNEX** CSV, all five selected, facility `PCPH Annex`, courts
  `5,6,7,8,9`: a `MATCHES` tab 25 blocks wide, `MATCHES!B2` reading 125, and
  an empty `SCHEDULE` of 33 slots
- No pair appears twice in any `MATCHES` row, and each band's rows are its
  round-robin rounds, then its playoff stages, then `F + B` (§10.2.2)
- Pasting every `MATCHES` row into its own `SCHEDULE` slot and running
  `Fill match numbers` with base `1000` gives a `CSV` of **116** rows,
  `1001`-`1116`; on the Annex with base `2000`, **125** rows, `2001`-`2125`,
  and `CSV!D` reads `Court 5` for match `2001`, not `Court 1`
- The generation log's suggested layout puts no `NXD` match in any slot before
  the last `NMD`, `NWD` or `LIWD` match, and no `LIXD` or `HIXD` match before
  the last `LIMD`, `HIMD` or `HIWD` match (§10.2.4)
- `HIMD`'s group block is 13 pairs in one run with `K = 1` on pair 1, `2` on
  pair 6 and `3` on pair 10, and three score grids, `5×5`, `4×4`, `4×4`
- `HIMD` and `NMD` generate §7.5's tiered ladder: `R16-3` (`R16_5`, `R16_6`),
  `QF-2` (`QF_3` fed from `R16-3`, `QF_4` fresh), `SF-1` (`SF_1`, `SF_2`),
  `SF-2` (`SF_3`, `SF_4` fed from `QF-2`), `B` and `F`; the slot table reads
  `1 SF`, `2 SF`, `3 SF`, `4 QF`, `5 R16`, `6 R16`, and the qualifier
  scaffold's six tier labels read `Br 1 - Rank 1` … `Rank 2 - 3rd`
- `NXD`, `LIMD` and `LIXD` generate a full draw: `QF_1` … `QF_8` fresh,
  `QF-k`'s winner to `SF_k`, and an eight-row slot table
- `NWD`, `LIWD` and `HIXD` have a four-row, all-`SF` slot table
- `HIWD` generates a group block and one `3×3` grid, **no** playoff rows, no
  slot table and no qualifier rows; its `MATCHES` band is 1 block wide and 3
  rows tall
- STEP 3 (`AI`) is blank on every category tab
- `STANDINGSCSV` has one row per pair **and per playoff slot** — 95 rows for
  Main, 111 for the Annex — with `G` reading `1`, `2`, `3`, `4` on group rows
  and blank on playoff rows
- Once packed, `Timeline`'s grand total equals `MATCHES!B2` (116, 125); no
  body cell exceeds 1
- `Court Control` is court blocks only, labelled `1`-`4` on Main and `5`-`9`
  on the Annex, and `CSV!B` reports the live court for a match typed into
  any of them
- The workbook defines exactly §11's 22 named functions
- No `#NAME?` and no `#REF!` anywhere

And against BKL Cup 2026 Day 2, for the shapes Pickle for Sight lacks:

- A CSV reconstructed from BKL Day 2, selecting `LI40MD LI40WD LI40XD`,
  courts `1..12`, gives `MATCHES!B2` of **42** — the calculator's 41 plus
  `LI40WD`'s bronze walkover; selecting `LI18MD LI18WD LI18XD`, courts
  `7..15`, gives **84**, and once packed and numbered from base `3000`,
  `CSV!D` reads `Court 10` for match `3001`
- `LI40WD` generates the twice-to-beat blocks of §7.5.1 — `F (1)`, `F (2)` and
  a `B` walkover whose `B_2` side reads `BYE` — a three-row qualifier scaffold
  with no draw, and a `MATCHES` band of two rows, `F (1)` holding game 1 and
  the walkover, then `F (2)`
- A 2-pair category generates the best-of-3 blocks of §7.5.1, no score grid,
  no qualifier scaffold, and a `MATCHES` band of three rows
- Every named function resolves at 4, 5, 12 and 15 court blocks with no edit
  to any column list

And in general:

- A category whose ladder would need a stage before `R32` (7 brackets ×
  top 2, `fill: bye`) is rejected by name, with the workbook untouched
- Running the generator twice is **refused**, not half-applied
- A `format: dual` CSV is rejected with a clear message and the workbook
  untouched
- The sync publishes a generated workbook once it is packed and numbered,
  with no other hand-editing

---

## 17. Out of scope

- Any change to `sheet-generator.gs` or the Dual Meet Master
- Player name import — rosters stay a paste, both scaffolds (§7.6)
- Filling STEP 3 or the qualifier draw automatically (§15)
- Seeded or non-random draws
- Generating the packed `SCHEDULE` — options 1 and 3 (§10.2.4)
- Checking a hand-packed `SCHEDULE` against the packing rules beyond what
  `Timeline` already shows (§10.2.3)
- Cross-facility match-number allocation (§6.2)
- Categories split across two facilities in one day
- `R32` site support (§15)

---

## 18. Divergences

Where `standard-generator.gs` departs from the sections above, or fills in
something they leave open.

**Departures**

- **`MATCHES` block headers name the bracket** (§10.2.2): row 5 reads
  `<KEY> Br <b>` over the blocks that hold bracket `b`'s round-robin
  matches — `HIMD Br 1`, `HIMD Br 1`, `HIMD Br 2`, … for a 5-4-4 split —
  rather than `<KEY> <k>`. A block past the round robin's width, which only a
  wide playoff row needs, keeps `<KEY> <k>`. Brackets sit left to right,
  `floor(n_b / 2)` blocks each; every round of a bracket fills exactly that
  many, and brackets run largest first, so a bracket's matches stay under
  its own headers in every round-robin row.
- **Two brackets play a crossover, not a draw** (§7.6). With two brackets
  the playoff seating is fixed, so the qualifier scaffold's slot column
  (`AM`) is written in rather than left `-` for a draw. Top 2: `Br 1 -
  Rank 1` takes slot 1, `Br 2 - Rank 2` slot 2, `Br 2 - Rank 1` slot 3 and
  `Br 1 - Rank 2` slot 4, so `SF-1` is Br 1 #1 v Br 2 #2 and `SF-2` is
  Br 2 #1 v Br 1 #2. Top 1: Br 1 #1 and Br 2 #1 take the final's slots 1 and
  2. The operator types only the names. Three or more brackets are still
  drawn by lot, as are two brackets with top 3 or 4 or with wildcards.
- **Round-robin rounds follow SAGE's bracket guide, not the circle
  method** (§10.2.2). The RR tab of the *BRACKET GUIDE* workbook lists, for
  each bracket size, which pairs meet in each round and in what order; the
  generator carries it as `RR_GUIDE_` and renumbers each bracket from its own
  first pair. The guide covers brackets of 3, 4, 5, 6, 7, 9 and 10 (2 is a
  single match). Any other size falls back to the circle method, and the
  generator warns which category did.
- **`MATCHES` row labels are not in column `B`** (§10.2.2). Categories sit
  side by side, so one row means different things for different bands — row
  4 is `NWD`'s `SF` and `NMD`'s `RR 4`. Each band carries its own labels
  instead, on each slot's first row in the separator column just left of the
  band's first block (column `C` for the first band). A paste that starts at
  a block's `D` column, as §10.2.3 step 1 says, never picks them up. `B` is
  blank below row 2.
- **No bracket may hold more than 13 pairs.** A group score grid runs from
  `O`, and the roster scaffold's notes column is `AB`, so a bracket of 14 or
  more would write its grid into the scaffold. §9 gains the rule; the
  calculator already warns above 8.
- **`fill` is validated** — empty, `bye` or `wc` — alongside §9's
  `solo_format` check, for the same reason.
- **The twice-to-beat and best-of-3 blocks take the `F` block's format.**
  They are the same shape (§7.5.1), so the master needs no prototype of its
  own for them.
- **`CSV!B2` is guarded on a blank `A`**:
  `=IF($A2="","",IFNA(FILTER('Court Control'!$B:$B,…)))`. With `A2` blank,
  §10.5's bare form matches every blank `C` cell on `Court Control`, spills,
  and shows `#REF!` on the one-row tab the generator leaves.
- **`SCHEDULE`'s and `MATCHES`' name formulas are guarded on a blank
  code**: `=IF(E6="","",GETPLAYERNAMESBYTEAMCODE(E6))`, not §10.2.1's bare
  call, which shows `#REF!` in every slot of an empty `SCHEDULE`.
- **Conditional formats on `SCHEDULE` and `MATCHES` are lifted off before
  tiling and put back afterwards** with one range per block column. Tiled
  with `copyTo`, each rule ends up listing every cell separately, and past a
  few hundred cells `setConditionalFormatRules` fails outright.
- **The generator writes `Title!B6`/`B7`** (§10.7 says it writes only
  `Variables`). They are the same two formulas the master carries, written
  so a master missing them still titles `SCHEDULE`.
- **`B` and `AB` level colours** (§10.2.4). `B` is Sheets' "light 3" row;
  `AB` is halfway between "light 3" and `N`'s "light 2", because the
  standard palette has no swatch between the two.

**Where the spec was silent**

- **The master's prototypes.** `_CATEGORY_TEMPLATE` is an unchanged copy
  of the reference's `HIMD` tab, not a reduced prototype tab (§14.1 step 3).
  It already holds one of every block, and the generator copies format only,
  never content, from fixed cells of it: group pair rows 6-7, the grid cells
  at `N6:P7`, the `R16` banner and `R16-3` block (rows 33-40), `FINALS`,
  `B` and `F` (rows 63-76), the roster at `AB5:AE6` and `AG5:AI5`, the draw at
  `AM5:AP6`, the slot table at `AT4:AV4`, and the all-black row 34 for every
  empty cell. So a generated tab has the reference's fills, fonts, borders
  and alignment, and no colour or font lives in code. The tab's own
  conditional formats come along, stretched to each tab's height; the two
  pinned to `HIMD`'s cells (`SUM($D$6:$E$13)=312`) are dropped. The
  generator checks for the copy's banner labels before writing anything.
  `Court Control`'s prototype is `B:E`; `Reference for Players`' one anchor
  is §10.7's formula wrapped in `IFERROR(…,"")` with `F3` blank, so it
  spills nothing.
- **The slot table** starts at row 4, one row above the other scaffolds, as
  in the reference, with no header row of its own.
- **Flat and full-draw tier labels** run rank-major: every bracket's winner,
  then every bracket's runner-up, then the wildcards.
- **Court labels with no digit** are accepted with a warning: `MATCHCOURT`
  publishes the number in the heading, so such a court publishes as
  `Not found`.
- **The packer splits a finals row wider than the court count** into single
  matches, so a one-court workbook can still place a final and its bronze.
