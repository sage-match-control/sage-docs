# Standard Tournament Generator

Builds one venue's scoring workbook for one day of a standard tournament —
every category tab, ready to score — from the plan you exported from the
Tournament Calculator. It replaces duplicating last event's venue
spreadsheet and find-replacing every category key, court number and team
code by hand.

It runs in the **SAGE Standard Tournament Master** workbook: make a copy of
it for each venue-day, then **SAGE → Generate event tabs** in the copy.

**Standard tournaments only.** A dual-meet plan is rejected; that belongs in
the [Dual Meet Sheet Generator](dual-meet-sheet-generator.md).

## One workbook per venue per day

A dual meet is one workbook. A standard tournament is one workbook for
**each venue on each day** — Pickle for Sight's single day at two venues is
two workbooks. You run the generator once in each, and tell it three things
the plan can't:

- **Which categories this venue runs.** Tick them in the sidebar. If you
  exported one plan per venue, **Select all** ticks the lot.
- **The venue's court labels**, in schedule order — `1,2,3,4` at one venue,
  `5,6,7,8,9` at the next. Their number is the venue's court count, and they
  are the court numbers the website shows.
- **Waves.** Each category gets a wave number; wave 2 plays after all of
  wave 1 is finished. Mixed doubles defaults to wave 2 so a player entered in
  both a same-gender and a mixed category is never due on two courts at once.

## What it builds

For each ticked category, a tab with:

- Every pair in one list, with a bracket number beside the first pair of each
  bracket, and the win/loss/score/quotient formulas already wired up.
- A score grid per bracket. Bracket sizes can be uneven — 13 pairs in three
  brackets is 5-4-4 — and each grid is sized to its own bracket.
- The whole playoff ladder the calculator planned, including the byes it
  gives group winners, down to the bronze and final. Each match works out
  where its winner goes, so the ladder fills itself in as results come in.
- Two paste-in areas: the roster (names, then codes shuffled by hand), and
  the **qualifier draw**, where group-stage qualifiers are drawn by lot into
  playoff slots. The draw lists which finisher each row is waiting for, and
  the slot table beside it lists every slot the draw can land on. A
  category with **two brackets** has no draw: its semifinals are a
  crossover (Br 1 #1 v Br 2 #2, Br 2 #1 v Br 1 #2), so the slots come
  filled in and you only type the names.

Plus a **`MATCHES`** tab, laying out every category's matches side by side,
one row per round, and an **empty `SCHEDULE`** grid sized for the day, with
court headings and start times but no matches in it.

It also fills in `Variables`, `Title` and `Reference for Players`, and builds
the readout tabs — `Court Control`, `Timeline`, and the `CSV` and
`STANDINGSCSV` tabs the live sync publishes.

## Packing the schedule

The generator doesn't place matches on courts; you do, by copying from
`MATCHES` into `SCHEDULE`. That keeps you in charge of which categories share
the courts when.

1. On `MATCHES`, select whole court blocks — both rows of the slot, starting
   at a block's left edge — and paste them onto a `SCHEDULE` slot with
   **Paste special → Formula only**. The label beside each row (`RR 1`, `SF`,
   `F + B`) is to the left of the block, so it doesn't come with it.
2. Keep to the rules nothing checks for you: no pair twice in one slot; a
   playoff round only after the round before it has finished, with a free
   slot after the round robin for the qualifier draw; a final and its bronze
   in the same slot; mixed doubles after the same-gender categories.
3. Leave unused courts blank or `-`.
4. **SAGE → Fill match numbers.** Each venue takes its own thousand: the
   first venue enters `1000` (matches `1001` onward), the second `2000`, and
   so on. Nothing checks the venues against each other.
5. Check `Timeline`: its total should equal the match count on `MATCHES`, and
   no cell should show more than 1.
6. Set up live sync.

The generator's execution log carries a suggested layout that follows those
rules, if you'd rather copy one than work one out.

## If it refuses to run

It checks everything before writing anything, lists every problem at once,
and leaves the workbook untouched. Common ones:

- **The plan is a dual meet.**
- **A court label is missing or repeated.**
- **No category ticked.**
- **A playoff too deep** — a ladder that would need a round before the Round
  of 32. The message names the category; switch it to wildcards, or use
  fewer brackets.
- **A bracket of more than 13 pairs.** Add brackets.
- **The workbook has already been generated.** Like the dual-meet generator,
  it builds several tabs in place, so it runs once per workbook. Start again
  from a fresh copy of the master.

---
**Technical:** [how the generator is built](../technical/standard-tournament-generator.md)
