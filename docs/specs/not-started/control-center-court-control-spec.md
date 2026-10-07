# Spec — Court Control from Control Center

> **Status: not started — outline only.** Written 2026-10-05 against
> `sage-tools-api` 2.8.0, `tools/control-center.html` and the three workbook
> generators as of that date. It records what is known and the shape agreed so
> far, not an implementation guide: §5 lists the owner's open decisions, and
> the detail is written once those are settled.
>
> Builds on [Score entry from Control Center](../implemented/control-center-score-entry-spec.md),
> and should wait for its real-Google checks (§11.2, C1–C8), since it reuses
> the same write path.

Let a signed-in operator put a match on a court, change it or clear it from
Control Center, without opening the facility workbook.

---

## 1. Why

Since score entry, **Court Control** is the one job that still keeps the
operator in the Google Sheet all day. It is also the field that decides what
counts as live: Control Center's Live Matches, the schedule board and the
Tournament Hub all read it. With it in the console, the console can run the
whole day, not just show it.

## 2. What we know

- **One input cell per court, the same in every workbook type.** Court Control
  has one 3-row block per court from row 5 (`5 + 3·(k − 1)`): column B is the
  court label (merged over the 3 rows), column C the match number, typed by
  hand. D and E are formulas that read the match. This holds for the dual-meet
  generator (`dual-meet-generator.gs`, B is the court number), the standard
  generator (`standard-tournament-generator.gs`, B is `=Variables!R<k+1>`) and the
  hand-built PickleDrive team workbook.
- **The snapshot's `court` value is column B.** `CSV`'s `court` column is
  `FILTER('Court Control'!B:B, 'Court Control'!C:C = matchNumber)`, so the
  label the console shows is exactly the value to look the block up by.
- **An API write fires no `onEdit`.** As with score entry (its D4 and §10), the
  API has to publish through `syncService.syncDay` in the same request, or
  nothing reaches the pages until the next edit by hand.
- **C is not always a plain number.** After the 3 October event, the team
  workbook's typed match numbers in `Court Control!C` read `#REF!`
  ([Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md) §2).
  The write has to refuse a cell holding a formula or an error, as score entry
  refuses a formula in a score cell.
- **Typing in the sheet stays.** `sheets-sync.gs` watches Court Control by its
  pinned GID and is unchanged: hand edits and console edits write the same
  cell and can be mixed.
- **The pieces to reuse exist:** the service account and `SheetsClient`
  (writes guarded by a per-feature allowlist), operator tokens, `scheduleGrid`
  to check that a match number exists in `SCHEDULE`, and the optimistic
  `expected` check with a 409 carrying the sheet's current value.

## 3. Proposed shape

**API** (`sage-tools-api`, a minor version):

```
PUT /v1/days/{day}/facilities/{facility}/courts/{court}/match
{ "matchNumber": 57 | null, "expected": { "matchNumber": 42 | null } }
```

1. Read `Court Control!B5:C` and find the block whose B equals `{court}`.
2. Refuse when C holds a formula or an error, when the match is not in
   `SCHEDULE`, or when it is already on another court.
3. 409 with the current value when C no longer equals `expected`.
4. Write C (`values:batchUpdate`, RAW) or clear it (`values:batchClear`) for
   `null`. `SheetsClient` gains a third allowlist: that one cell.
5. Publish via `syncService.syncDay` for that facility; log one audit line.

**Control Center.** In **Live Matches**, which already shows one row per
court, a signed-in operator gets **Set match** on each court. A dialog suggests
the next match (the earliest scheduled one that is neither played nor on a
court, preferring matches assigned to that court) and offers **Clear**.
Warnings, never refusals: the match is already scored; one of its players is
live on another court; it is scheduled for a different court. A court action,
not a match action, so score entry's "matches are clickable only in Match
Finder" rule (its D8) still holds.

**Phase 2, the payoff.** After saving the score of a match that is live on a
court, the score dialog offers *"Court 3 is free: put #57 on it?"*. That is the
event-day rhythm ("score it, call the next one") in one step. It changes the
shared `SCORE CLIENT` block and so the scorer page too.

## 4. Not changing

`sheets-sync.gs` and every `.gs` file, the generators, the snapshot format,
the live Worker, score entry's routes and attendance.

## 5. Decisions for the owner

| # | Question | Leaning |
| --- | --- | --- |
| Q1 | May scorers set courts from the scorer page, or only operators? | Operators only, at first |
| Q2 | Gate it on the event's existing `scoreEntry` setting, or a new `courtControl` one? | Reuse `scoreEntry`: one fewer switch |
| Q3 | Build the "save score, then next match" prompt with it, or after? | After, as phase 2 |

## 6. Blocked on

Score entry's real-Google checks (C1–C8). Not before or during an event.

---
**Related:** [Score entry from Control Center](../implemented/control-center-score-entry-spec.md) ·
[Run the day](../../usage/run-the-day.md) ·
[Control Center technical](../../technical/control-center.md)
