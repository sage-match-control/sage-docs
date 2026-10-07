# Event registry schema

`event-data/config/events.json` — the live event/day/facility registry for
`sage-tools-api`'s sync feature, fetched and cached at runtime. See [sync
pipeline](sync-pipeline.md) for how it's fetched/cached/validated; this page
is the shape.

Editing and committing this file (on `main`) is how you add an event, add a
day, or fix a wrong sheet ID — **no `sage-tools-api` redeploy needed**;
every running instance re-checks it within `SYNC_CONFIG_TTL_MS` (~60s by
default).

## Shape

```jsonc
{
  "version": 1,
  "livePush": true,                   // optional: false turns live push off (written by Control Center's Sync method switch)
  "defaults": {
    "matchesSheetName": "CSV",
    "standingsSheetName": "STANDINGSCSV"
  },
  "events": {
    "<event-key>": {
      "type": "dual-meet",              // or "standard" or "team" — required, picks the console's layout
      "archived": false,                // optional, console-only: hides from the event picker
      "title": "PNF × BUP Dual Meet",   // optional, console-only: masthead label
      "attendance": "desks",            // optional: "console" | "desks" — staff check-in; absent means none
      "scoreEntry": "links",            // optional: "console" | "links" — entering scores; absent means off
      "days": {
        "<day-key>": {
          "label": "Day 1 · Aug 15",
          "date": "2026-08-15",         // optional, console-only: chronological ordering + "today" default
          "isLive": "auto",             // "auto" | true | false — the go-live override target
          "facilities": [
            { "name": "Main", "sheetId": "1hbjqbH3H1..." }
          ]
        }
      },
      "display": {                      // optional — console shows raw codes without it
        "divisions": { "LI": "Low Intermediate" },
        "events":    { "MD": "Men's Doubles" },
        "clubs":     { "PNF": "Pickle & Friends Community" }
      }
    }
  }
}
```

- `<event-key>` must match this event's folder name under `events/` in
  `sage-match-control.github.io` **and** its folder name in `event-data`.
- `type` is read **only** by Control Center — `sage-tools-api`
  never looks at it. Not inferred: an unmatched code would fail *silently*
  the moment the code shape ever changes, so it's required and explicit.
- A `"team"` event takes its team names from `STANDINGSCSV` and needs no
  `display` block, except `display.pairs` when its pair labels aren't the
  defaults. `sage-tools-api` never reads `type` or `display`, so it needs no
  backend change.
- `display.pairs` (team events only) labels each pair of a matchup, keyed by
  the pair number in a team code (the `3` of `A_3`):
  `"3": { "full": "Mixed Doubles", "short": "XD" }`. Both labels are non-empty
  strings. A type that repeats is numbered ("XD 1", "XD 2"). Without it the
  labels are MD, WD, XD 1 and XD 2; a malformed map falls back to those with
  a console warning on the page, and a pair number the map lacks reads
  "Pair <n>". The Hub, the board, the scorer page and Control Center read it.
- `display` maps are all `code → label`; **order comes from key order**, so
  there's no separate ordering config to keep in step. No logo field —
  `display.clubs` maps to a plain name string only; the console shows the
  3-letter code, not a logo, on every row.
- `attendance` turns on [event attendance](event-attendance.md):
  `"console"` lets operators mark people in Control Center, `"desks"` also
  allows desk links. Anything else is rejected on load. A day used with desk
  links needs a `date`.
- `scoreEntry` turns on entering a match's score from Control Center and the API
  (see [Entering a score](../features/score-entry.md)):
  `"console"` for signed-in operators, `"links"` for operators **and** scorer
  links (see [Scorer page](scorer-page.md)). Absent means off. Anything else is
  rejected on load. Mission Control's **Scorer links** switch moves an event between
  `"links"` and `"console"` by writing this value (`PUT
  /v3/events/{event}/score-entry`); it never turns score entry on or off, so adding
  the setting is a commit here. Every facility workbook of the event must be shared
  with the API's service account as **Editor**, as for attendance, or a save fails
  with a message naming the account.
- A facility with an empty/missing `sheetId` is treated as "not set up yet"
  and skipped rather than fetched — lets you add a day's entry before its
  spreadsheet exists.
- `matchesSheetName`/`standingsSheetName` can be overridden per day, if that
  spreadsheet's tabs are literally named something else. So can
  `rosterSheetName`, the team roster tab a `"team"` event publishes (default
  `Teams`; other types publish no roster). **Deliberately no
  GID equivalent** — a GID is assigned per-workbook and doesn't carry over
  if a spreadsheet is ever duplicated from another event's; a tab name
  does.
- A `"_comment"` string key is allowed anywhere in the tree for notes that
  have no other home in JSON; ignored by validation.

## Validation

Enforced on every load; a file failing any rule is **rejected wholesale** —
the service keeps serving whatever it last loaded successfully (or its
bundled fallback seed) rather than partially applying a broken commit:

- `version` must match the supported version.
- Every event key and day key must match `^[a-z0-9][a-z0-9-]*$` — both
  become path segments/filenames, so an invalid key is refused rather than
  sanitized.
- Day keys must be **globally unique across every event** — a day key is
  also a path segment of every sync (`/v3/days/{day}/…`), so two events sharing one would race to
  publish into each other's data.
- Each day needs a non-empty `label` and a `facilities` array (can be
  empty).
- `isLive`, if present, must be `true`, `false`, or the literal string
  `"auto"`.
- `attendance`, if present, must be `"console"` or `"desks"`, and `scoreEntry`,
  if present, must be `"console"` or `"links"`.
- `livePush`, if present, must be `true` or `false`. `false` makes every
  sync and Live/Hide publish to GitHub alone; absent or `true` leaves live
  push to Cloud Run's environment. Normally written by Control Center's
  **Sync method** switch, not by hand.
- Facility names must be unique within a day.
- At least one event, each with at least one day.

If unsure whether an edit is valid before committing, check `GET
/v3/diagnostics/sync` (`X-Sync-Secret` header) after committing — it reports which
config revision is actually live, and whether the service fell back to its
bundled seed because the commit failed validation.

## Sheet IDs are effectively public

`event-data` is public (GitHub Pages requires it on the free plan), so this
file publishes every registered facility spreadsheet's ID. That's
expected — sheet IDs aren't secrets, the sheets are already
link-shareable by design, and the published snapshots already contain
everything on their synced tabs. But it does make the *set* of
spreadsheets enumerable: only register spreadsheets that are already
meant to be public, and never keep private organizer notes in an extra tab
of a spreadsheet that's registered here.

---
**Full reference:** `event-data/config/README.md` (this page is a condensed version).
**Related:** [Adding a new event](adding-a-new-event.md), [sync pipeline](sync-pipeline.md).
