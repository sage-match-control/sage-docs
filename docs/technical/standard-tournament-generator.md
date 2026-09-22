# Standard Tournament Generator

`sage-tools-api/scripts/standard-generator.gs` — bound Apps Script that builds
one facility-day's standard-tournament workbook from a Tournament Calculator
plan CSV. Implements
[`standard-tournament-master-spec.md`](../specs/implemented/standard-tournament-master-spec.md);
its §18 records where the code departs from it.

It is bound to the **SAGE Standard Tournament Master** workbook (spec §14),
whose hidden `_CATEGORY_TEMPLATE` is an unchanged copy of the Pickle for
Sight Annex's `HIMD` tab: every category tab takes its formats from fixed
cells of it (`TEMPLATE_PROTO`).

Same arrangement as the [Dual Meet Sheet Generator](dual-meet-sheet-generator.md):
it lives in the API repo for versioning, runs inside a workbook, and ships by
being pasted into the master's script project beside `sheets-sync.gs`.
Changing it is not a deploy. It needs a master for the same reason — the tabs
run on the workbook-level [Named Function library](named-function-library.md),
here in its 22-function standard form (spec §11), with the row-5-label
`MATCHCOURT`.

## A separate file, not a mode

The two generators share no code at runtime: each master carries only its
own. `standard-generator.gs` copies `sheet-generator.gs`'s pure helpers
verbatim (CSV parsing, `colLetter_`, the progress log, `assertTabsPristine_`)
and rewrites everything shaped around clubs. The `onOpen` block is the one
piece duplicated verbatim across all three `.gs` files, because a workbook
carries `sheets-sync.gs` plus one generator and the last-loaded `onOpen`
wins.

## Pipeline

```
parsePlanCsv ─► validatePlan ─► planCategory (per ticked category)
                                     │
                     computeTabLayout_ ─► packSchedule_ (in memory: sizes SCHEDULE)
                                     │
category tabs → Variables → Title → Reference for Players → MATCHES → SCHEDULE
             → CSV → STANDINGSCSV → Court Control → Timeline
```

Everything left of the Sheets writes is pure. `planCategory` is where every
playoff shape lives, and the only part with real logic.

### `planCategory`

Ports the calculator's `groupSizes`, `reduceToTarget`, `solveForTarget`,
`tieredRounds` and `playoffPlan` (keep them in step with
`tools/tournament-calculator.html`), then:

- **Round robin** from `RR_GUIDE_`, SAGE's bracket guide (the RR tab of the
  *BRACKET GUIDE* workbook): per bracket size, which pairs meet in each round
  and in what order. A size the guide lacks (8, 11-13) falls back to the
  circle method, with a warning. Brackets are uneven, so each carries its own
  first-pair index, and the guide's pair numbers are offset from it. Codes
  run `<KEY>_1 … <KEY>_<teams>` unbroken, bracket 1 first.
- **Seat numbering** (spec §7.6.1). Stages are walked from the final back,
  with one counter shared by every stage's fresh seats, so a drawn slot
  number identifies exactly one seat. Fed seats take the other half of a
  fresh partner's pair, or the lowest free number. Each stage's fed seats
  are wired to the previous stage's winners in block-label order.
- **Shapes without a ladder**: best-of-3 (2 pairs), twice-to-beat (one
  bracket, the default) with a bronze walkover against `BYE`, and
  round-robin only.

It also reports the `MATCHES` rows and band width, every team code the tab
holds (which sizes `STANDINGSCSV`), and any validation error: a ladder
deeper than `R32`, more than 32 entrants, or a code or label that would
repeat.

### Forward propagation

A standard ladder can't pull its entrants from a sorted group block the way
a dual meet's playoff does — there is a draw and a chain of results in the
way. So each block pushes its winner forward: column `K` evaluates to the
code of the seat the winner moves to, and a fed seat finds its names with
`FILTER` over the previous stage's `B`/`K` rows. Fresh seats read the
qualifier scaffold (`AO`/`AP`), which resolves a drawn slot number through
the slot table (`AT:AV`). A two-bracket category's slot numbers are written
in rather than drawn (`fixedSeating_`), because its semifinals are a fixed
crossover.

### `MATCHES` and the packer

`MATCHES` is a copy of the pristine `SCHEDULE` prototype, tiled to one
band per category, so pasting from it into `SCHEDULE` is a plain copy.
Nothing reads it. Row labels sit in the separator column left of each band,
out of the way of a paste.

`packSchedule_` places every match in waves, taking turns between categories,
never reordering a category's queue, holding each playoff stage until the one
before it has finished, and keeping one slot clear after a round robin for
the qualifier draw. Its only outputs are `SCHEDULE`'s height (slots + 4
spare) and the suggested layout in the execution log. It reproduces the
Pickle for Sight days at 33 and 29 slots.

### In place, once

`SCHEDULE`, `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV` and
`Reference for Players` are grown from prototypes already in the master
(`REBUILT_IN_PLACE_TABS`), because the sync watches `SCHEDULE`'s and
`Court Control`'s GIDs. A second run is refused before anything is written —
by the pristine check, and by the category tabs and `MATCHES` colliding by
name.

`CSV` gets its header and one formula row with `A2` blank. `SAGE → Fill match
numbers` (in `sheets-sync.gs`) copies that row down to one row per match.
Slot times on `SCHEDULE` and `Timeline`'s headers both chain off
`Variables!C4` + `Variables!$C$6`, never literals, so `COUNTPAIRAT`'s exact
match holds.

## Verifying a change

```bash
node scripts/verify-standard-generator.mjs
```

Runs the real `generateEventTabs` against `scripts/mock-apps-script.mjs`,
fed the two Pickle for Sight plan CSVs in `scripts/fixtures/`, plus a
reconstructed BKL Day 2 plan. It checks the spec's §16 figures, the ladder
cells of spec §7.5 in the tab, every one of the 105 multi-bracket
combinations §9 allows, the packer against the waves, and the packed
schedule through `sheets-sync.gs`'s real `planMatchNumbers_`.
`sheets-sync.gs` is loaded into the same context first, so a name clash
between the two files fails there. The mock does not evaluate formulas.
Change a prototype shape in `REBUILT_IN_PLACE_TABS` and the fixture in the
script has to change with it.

---
**Features:** [Standard Tournament Generator usage](../features/standard-tournament-generator.md)
**Spec:** [`standard-tournament-master-spec.md`](../specs/implemented/standard-tournament-master-spec.md)
