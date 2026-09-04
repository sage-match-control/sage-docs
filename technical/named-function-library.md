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
| `SORTBYWINS(range)` | an `A:H` block's codes ranked by wins then quotient | playoff feeders, always inside `INDEX(…, k)` |
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

---
**See also:** [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) — the
bound script that writes the formulas calling these.
