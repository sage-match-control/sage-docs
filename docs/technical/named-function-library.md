# The Named Function library

The 23 workbook-level `LAMBDA` definitions that every dual meet workbook runs
on. They live under **Data → Named functions** inside the workbook itself,
not in any repo and not in the bound Apps Script project — an easy thing to
confuse, since [the generator](dual-meet-sheet-generator.md) *is* bound Apps
Script and lives in the same workbook.

This page is the reference copy. It exists because the library is otherwise
only readable by opening a workbook and clicking through 23 dialogs, which
makes it the one part of the system with no greppable source.

**A generated workbook inherits these by being a copy of the master.** They
cannot be created through the Sheets API, which is the constraint that forces
the whole generate-from-a-copy design.

**The standard-tournament library is this one, with two changes.** The SAGE
Standard Tournament Master carries 22 functions: these 23 less `SORTBYWINS`
(a standard playoff's entrants are drawn, not sorted), with `MATCHCOURT`
reading the court label from `SCHEDULE` row 5 instead of the block's
position. `standard-tournament-master-spec.md` §10.5 and §11 give the
definition and the reasoning.

## Reading the library out of a workbook

Two routes, both awkward:

- **Data → Named functions**, then Edit on each entry. Authoritative, but one
  dialog per function.
- **Export as `.xlsx` and unzip it** — `xl/workbook.xml`, the `<definedNames>`
  element. Note that the Drive export URL redirects to a
  `googleusercontent.com` host with no CORS headers, so this has to be a real
  file download, not a `fetch` from the Sheets page.

## The three tiers

Everything bottoms out in `STACKBLOCKS`. Nothing reads `SCHEDULE` directly
except `STACKBLOCKS`, `COUNTPAIRAT`, `MATCHTIME` and `MATCHCOURT`.

```
SCHEDULE
  └── STACKBLOCKS(first_col, step, first_row)
        ├── GETPLAYERSCOLUMN1 / GETPLAYERSCOLUMN2      (team codes)
        ├── GETSCORESCOLUMN1  / GETSCORESCOLUMN2       (scores)
        └── GETMATCHNUMBERS                            (match numbers)
              ├── GETMATCHES  ── GETSCOREAGAINSTPAIR
              ├── GETMATCHRESULTSBYTEAMCODE ── GETTOTALWINS / GETTOTALLOSES
              └── GETTEAM1CODEBYMATCH / GETTEAM2CODEBYMATCH

'Reference for Players'
  └── GETTEAMCODES + GETPLAYERNAMES ── GETPLAYERNAMESBYTEAMCODE
```

### Base layer

`SCHEDULE` lays courts out as a repeating 8-column block from column `E`.
`STACKBLOCKS` walks that period and stacks one column from every court block
into a single array, deriving the court count from the sheet's own width so
nothing built on it needs regenerating when the court count changes.

```
=LET(
  n,   INT((COLUMNS(SCHEDULE!$1:$1) - first_col) / step) + 1,
  ref, LAMBDA(c, LET(L, SUBSTITUTE(ADDRESS(1, c, 4), "1", ""),
                     INDIRECT("SCHEDULE!" & L & first_row & ":" & L))),
  REDUCE(ref(first_col), SEQUENCE(n - 1, 1, first_col + step, step),
         LAMBDA(acc, c, VSTACK(acc, ref(c)))))
```

Each stacked array is **court-major**: all of court 1's rows, then all of
court 2's, every block the same height. `MATCHTIME` and `MATCHCOURT` both
depend on that ordering and on the block height being
`ROWS(SCHEDULE!$B$6:$B)`.

### Internal plumbing — never called from a cell

| Function | Definition |
| --- | --- |
| `GETPLAYERSCOLUMN1()` | `STACKBLOCKS(5, 8, 6)` — team 1 codes |
| `GETPLAYERSCOLUMN2()` | `STACKBLOCKS(7, 8, 6)` — team 2 codes |
| `GETSCORESCOLUMN1()` | `STACKBLOCKS(9, 8, 6)` — team 1 scores |
| `GETSCORESCOLUMN2()` | `STACKBLOCKS(10, 8, 6)` — team 2 scores |
| `GETMATCHNUMBERS()` | `STACKBLOCKS(6, 8, 6)` |
| `GETMATCHES()` | `ArrayFormula(GETPLAYERSCOLUMN1() & GETPLAYERSCOLUMN2())` |
| `GETTEAMCODES()` | `{'Reference for Players'!$A$3:$A}` |
| `GETPLAYERNAMES()` | `{'Reference for Players'!$B$3:$B & " " & 'Reference for Players'!$C$3:$C}` |
| `GETMATCHRESULTSBYTEAMCODE(teamcode)` | a boolean per completed match — see below |

`GETMATCHRESULTSBYTEAMCODE` is the hot path, since every standings row calls
it twice:

```
=LET(
  col_t1,  GETPLAYERSCOLUMN1(),
  col_t2,  GETPLAYERSCOLUMN2(),
  col_s1,  GETSCORESCOLUMN1(),
  col_s2,  GETSCORESCOLUMN2(),
  my_pts,  ARRAYFORMULA(IF(col_t1 = teamcode, col_s1, col_s2)),
  opp_pts, ARRAYFORMULA(IF(col_t1 = teamcode, col_s2, col_s1)),
  is_mine, ARRAYFORMULA((col_t1 = teamcode) + (col_t2 = teamcode)),
  played,  ARRAYFORMULA(ISNUMBER(col_s1) * ISNUMBER(col_s2)),
  IFERROR(FILTER(ARRAYFORMULA(my_pts > opp_pts), ARRAYFORMULA((is_mine > 0) * played)), ""))
```

It reads the four column primitives once each. An earlier shape walked the
pair's matches with `MAP` and looked each score up individually, which cost
`8 + 8M` `STACKBLOCKS` evaluations for `M` matches — roughly 144 per standings
row against 4 today.

### Called from generated formulas

| Function | Returns | Written by the generator into |
| --- | --- | --- |
| `GETTOTALWINS(paircode)` | `COUNTIF(GETMATCHRESULTSBYTEAMCODE(…), TRUE)` | category tab col `D` |
| `GETTOTALLOSES(paircode)` | same against `FALSE` (note the spelling — one `S`) | category tab col `E` |
| `GETTOTALSCORE(paircode)` | points scored | category tab col `F` |
| `GETTOTALOPPONENTSCORE(paircode)` | points conceded | category tab col `G` |
| `GETSCOREQUOTIENT(paircode)` | for / against, `0` on error | category tab col `H`, `STANDINGSCSV` (rounded to 4) |
| `GETSCOREAGAINSTPAIR(pair1, pair2)` | `pair1`'s score in that match, else `"No Match Found"` | the score grids |
| `SORTBYWINS(range)` | an `A:H` block's codes ranked by wins then quotient — no head-to-head, unlike the site's `rankStandings()` | playoff feeders, always inside `INDEX(…, k)` |
| `GETPLAYERNAMESBYTEAMCODE(teamcode)` | **2-element array**, spills two rows | `Court Control` |
| `GETTEAM1CODEBYMATCH(n)` / `GETTEAM2CODEBYMATCH(n)` | team code, else `"-"` | `CSV`, `Court Control` |
| `COUNTPAIRAT(time_value, pair_code)` | a pair's match count at one slot time | `Timeline` body |
| `MATCHTIME(n)` / `MATCHCOURT(n)` | slot time / `"Court N"` | `CSV!C`, `CSV!D` |

`GETSCOREQUOTIENT` inlines both totals rather than calling
`GETTOTALSCORE`/`GETTOTALOPPONENTSCORE`, so the four primitives are read once
instead of twice — the two total functions still exist because columns `F`
and `G` call them directly.

`COUNTPAIRAT` builds on the primitives rather than re-deriving the period-8
walk:

```
=LET(
  col_t1,   GETPLAYERSCOLUMN1(),
  col_t2,   GETPLAYERSCOLUMN2(),
  courts,   INT((COLUMNS(SCHEDULE!$1:$1) - 5) / 8) + 1,
  col_time, REDUCE(SCHEDULE!$B$6:$B, SEQUENCE(courts - 1),
              LAMBDA(acc, unused, VSTACK(acc, SCHEDULE!$B$6:$B))),
  SUMPRODUCT((col_time = time_value) * ((col_t1 = pair_code) + (col_t2 = pair_code))))
```

The `REDUCE`/`VSTACK` tiling repeats `SCHEDULE`'s single time column once per
court so it lines up with the court-major stacked arrays.

## Traps

These have all cost real debugging time.

**`LET` names cannot look like cell references.** Columns run to `ZZZ`, so any
1–3 letter word followed by digits is a cell: `p1`, `s2`, `row1`, `col3`, plus
the R1C1 forms `r1`, `c1`, `rc`. Sheets rejects them with *"argument N of LET
is not a valid name"*. Use letters-only names or include an underscore — which
is why the library's bindings are `col_t1`, `my_pts`, `pts_agst`, and why the
generator's own aggregate feeder uses `br1_win`/`br2_win`. Bare single letters
are fine (`n`, `c`, `L`, `acc`).

**`CHOOSEROWS` does not broadcast an array of row indices.** Given one it
takes the first and silently returns a single row. A `COUNTPAIRAT` built on
`CHOOSEROWS(SCHEDULE!$B$6:$B, MOD(SEQUENCE(…), h) + 1)` collapses `col_time`
to the value of `B6`, which dumps every one of a pair's matches into the first
time slot while leaving the row total correct — a failure that looks like a
scheduling bug rather than a formula bug. Use the `REDUCE`/`VSTACK` idiom
above.

**`IF` needs an explicit `ARRAYFORMULA` to go elementwise**, even inside `LET`
in a named function. `FILTER`'s condition argument and `SUMPRODUCT` are
array-aware on their own; `IF` is not.

**`STACKBLOCKS` uses `INDIRECT`, which is volatile.** Anything derived from it
recalculates on every edit anywhere in the workbook — including every edit the
sync trigger fires on. Removing it would mean baking `SCHEDULE`'s row and
column bounds into the formula, and those vary by court count across events,
so it stays.

**The two team-code primitives derive their court count from different
offsets** — `GETPLAYERSCOLUMN1` from `w - 5`, `GETPLAYERSCOLUMN2` from
`w - 7`. They agree only when `SCHEDULE`'s width is exactly `4 + 8 × courts`,
which is what the generator produces. A stray trailing column on `SCHEDULE`
would make them different lengths and break any formula that combines them
elementwise, `COUNTPAIRAT` included.

**`GETTEAMCODES` and `GETPLAYERNAMES` are anchored by sheet id, not name**, so
renaming `Reference for Players` is safe. `STACKBLOCKS` and the readout
functions reach `SCHEDULE` through `INDIRECT("SCHEDULE!"…)` and a literal
`SCHEDULE!` reference, so renaming *that* tab is not.

## Verifying a change

`scripts/verify-sheet-generator.mjs` does not help here — it asserts formula
*text*, never evaluates it. The only real check is a workbook.

The reliable method is a differential one: keep a reference copy of a
generated event workbook, apply the change to a second copy, then compare
every readout tab between them. `Timeline` and `STANDINGSCSV` between them
exercise most of the library, and a value-for-value match across `CSV`,
`STANDINGSCSV`, `Timeline` and the category tabs is strong evidence a refactor
is behaviour-preserving. `Court Control` always differs by its live "Current
Time" cell.

## The team workbook's library

A `"team"` event's workbook (PickleDrive's, the prototype) starts from this
library and departs from it in three ways. It defines 25 named functions, not
23: it has none of the dual-meet library's standings and helper functions
(`GETTOTALWINS`, `GETTOTALLOSES`, `GETTOTALSCORE`, `GETTOTALOPPONENTSCORE`,
`GETSCOREQUOTIENT`, `GETMATCHRESULTSBYTEAMCODE`, `SORTBYWINS`,
`GETSCOREAGAINSTPAIR`, `COUNTPAIRAT`, `GETMATCHES`, `GETTEAMCODES`,
`GETPLAYERNAMES`, `GETWINSCORE`), because a team workbook ranks teams and
matchups with its own family. The reasoning is in
`sage-docs/docs/specs/.../team-workbook-stack-cache-spec.md`.

### `StackCache`: the stacks, built once

The five primitives do not call `STACKBLOCKS`. They return the columns of a
hidden `StackCache` tab, one `VSTACK` of the ten court blocks' columns in
each of `A1:E1`, written as direct, open-ended ranges
(`=VSTACK(SCHEDULE!F6:F, SCHEDULE!N6:N, …, SCHEDULE!BZ6:BZ)`):

| Cell | Holds | Block columns (court 1 → court 10) | Primitive |
| --- | --- | --- | --- |
| `A1` | match numbers | `F N V AD AL AT BB BJ BR BZ` | `GETMATCHNUMBERS` = `StackCache!$A$1:$A` |
| `B1` | team 1 codes | `E M U AC AK AS BA BI BQ BY` | `GETPLAYERSCOLUMN1` = `StackCache!$B$1:$B` |
| `C1` | team 2 codes | `G O W AE AM AU BC BK BS CA` | `GETPLAYERSCOLUMN2` = `StackCache!$C$1:$C` |
| `D1` | team 1 scores | `I Q Y AG AO AW BE BM BU CC` | `GETSCORESCOLUMN1` = `StackCache!$D$1:$D` |
| `E1` | team 2 scores | `J R Z AH AP AX BF BN BV CD` | `GETSCORESCOLUMN2` = `StackCache!$E$1:$E` |

Everything built on the primitives keeps its name, arguments and callers.
`STACKBLOCKS` stays defined and nothing calls it.

**A direct reference is not volatile.** `STACKBLOCKS` goes through
`INDIRECT`, so everything downstream of it recalculates on every edit
anywhere in the workbook. A cache built from direct references recalculates a
column only when a cell it covers changes, and a formula downstream only when a
column it reads changed. A score edit reaches the two score columns and what
depends on them, not the codes, names, times or courts.

**Why direct references are safe in a team workbook.** `STACKBLOCKS` derives
the court count from `SCHEDULE`'s width and uses `INDIRECT` so that one
library serves every event. A workbook's cache belongs to one event, whose
court count is fixed (10 here), and a generator writes the cache for its
event's court count. A direct reference also makes Sheets track the
dependency statically, which would be a circular dependency if any
`SCHEDULE` cell depended on the stacks. None does: `SCHEDULE`'s formulas are
player names, matchup titles, times and the playoff codes in rows 32 and
below, all computed from other tabs or from the same row's codes.

The stacked ranges are open-ended, so `SCHEDULE` is trimmed to its content
(row 46 in PickleDrive's workbook); each stacked block is
`ROWS(SCHEDULE!$B$6:$B)` tall, which `MATCHTIME` and `MATCHCOURT` rely on.
All five cache columns are the same length, so the elementwise `FILTER`s that
combine them line up.

### `MatchLookup` and its base-code columns

`MatchLookup` is a hidden tab holding one row per match for matches 1–140:
`matchNumber`, `matchUp`, `teamCode1`, `team1Player1`, `team1Player2`,
`team1Score`, `teamCode2`, `team2Player1`, `team2Player2`, `team2Score` in
`A:J` (`F` is `=GETTEAM1SCOREBYMATCH(A2)`, `J` is `=GETTEAM2SCOREBYMATCH(A2)`).
Two helper columns hold each side's **base team code**, the part of its code
before the first `_` (`A_1` → `A`, `SF-J_2` → `SF-J`):

| Cell | Header | Formula |
| --- | --- | --- |
| `K2` | `teamBase1` | `=ARRAYFORMULA(IFERROR(REGEXEXTRACT(TO_TEXT(C2:C), "^_*([^_]+)"), ""))` |
| `L2` | `teamBase2` | `=ARRAYFORMULA(IFERROR(REGEXEXTRACT(TO_TEXT(G2:G), "^_*([^_]+)"), ""))` |

A blank or unparseable code gives `""`, which never equals a team code. `K`
and `L` are the same length as the `$A` and `$B` ranges they are filtered
beside, as `FILTER` requires.

### The matchup family

`Standings` and `STANDINGSCSV` ask, for a matchup and a team, for that
team's match results, scores and opponents' scores. Six functions answer it
from `MatchLookup`'s rows instead of looking each match's scores up again;
each takes `(matchup, teamcode)` and is wrapped in `IFNA(…, "")`, which
returns `""` when no row matches:

| Function | Side | Rows kept | Returns |
| --- | --- | --- | --- |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `""` if either score is empty, else `F > J` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `""` if either score is empty, else `J > F` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `F` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `J` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `J` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `F` |

Every one also keeps only the rows whose `B` contains the matchup
(`ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B))`). For example:

```
=IFNA(FILTER(MatchLookup!$F$2:$F, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$K$2:$K = teamcode), "")
```

The wrappers (`GETMATCHRESULTSBYMATCHUPANDTEAMCODE`,
`GETMATCHSCORESBYMATCHUPANDTEAMCODE`, `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE`,
which stack both sides with `{…;…}`) and `GETPAIRWINSBYMATCHUPANDTEAMCODE` /
`GETPAIRLOSESBYMATCHUPANDTEAMCODE` (which count `TRUE` / `FALSE` in the
results) sit on top of these and take only values, so the arrays' shape does
not matter.

**Unplayed is `= ""`, not `ISBLANK`.** `F` and `J` are formula cells, and
`ISBLANK` on a formula cell is false whenever the formula returns an empty
string. `= ""` is true for an empty value and for an empty string, and false
for a score of `0`, so played and unplayed are told apart. An unplayed match
counts as neither a win nor a loss and adds no points.

### Called per match

`GETTEAM1CODEBYMATCH` / `GETTEAM2CODEBYMATCH` and `GETTEAM1SCOREBYMATCH` /
`GETTEAM2SCOREBYMATCH` are each `IFNA(FILTER(<primitive>(), GETMATCHNUMBERS() =
matchnumber), "-")`, so they read the cache. `GETMATCHUPTITLE(team1, team2)`
splits each code at `_` and joins the two with `CHAR(10) & " v " & CHAR(10)`.
`MATCHTIME` and `MATCHCOURT` are as in the dual-meet library.
`GETPLAYERNAMESBYTEAMCODE(matchup, teamcode)` filters `Reference for
Players` on both arguments.

---
**See also:** [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) — the
bound script that writes the formulas calling these.
[Team workbook recalculation](../specs/implemented/team-workbook-stack-cache-spec.md)
— how the team workbook's cache and matchup family were built and checked.
