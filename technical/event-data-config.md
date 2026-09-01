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
  "defaults": {
    "matchesSheetName": "CSV",
    "standingsSheetName": "STANDINGSCSV"
  },
  "events": {
    "<event-key>": {
      "type": "dual-meet",              // or "standard" — required, picks the console's layout
      "archived": false,                // optional, console-only: hides from the event picker
      "title": "PNF × BUP Dual Meet",   // optional, console-only: masthead label
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
- `type` is read **only** by the Match Control console — `sage-tools-api`
  never looks at it. Not inferred: an unmatched code would fail *silently*
  the moment the code shape ever changes, so it's required and explicit.
- `display` maps are all `code → label`; **order comes from key order**, so
  there's no separate ordering config to keep in step. No logo field —
  `display.clubs` maps to a plain name string only; the console shows the
  3-letter code, not a logo, on every row.
- A facility with an empty/missing `sheetId` is treated as "not set up yet"
  and skipped rather than fetched — lets you add a day's entry before its
  spreadsheet exists.
- `matchesSheetName`/`standingsSheetName` can be overridden per day, if that
  spreadsheet's tabs are literally named something else. **Deliberately no
  GID equivalent** — see the incident note below.
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
  also the `/sync/:day` route, so two events sharing one would race to
  publish into each other's data.
- Each day needs a non-empty `label` and a `facilities` array (can be
  empty).
- `isLive`, if present, must be `true`, `false`, or the literal string
  `"auto"`.
- Facility names must be unique within a day.
- At least one event, each with at least one day.

If unsure whether an edit is valid before committing, check `GET
/sync/config` (`X-Sync-Secret` header) after committing — it reports which
config revision is actually live, and whether the service fell back to its
bundled seed because the commit failed validation.

## Why there's no per-tab GID override

An earlier version of this schema let a day override its matches/standings
tab by numeric GID instead of name. It was removed after a real incident: a
workbook duplicated from another event inherited that event's tab GIDs, so
a GID-based override correct for one spreadsheet silently pointed at a
leftover tab from a completely different event — with no error, just wrong
data being published. A tab GID is assigned per-workbook and doesn't carry
over when a spreadsheet is duplicated; a tab **name** doesn't carry that
risk. Both `sage-tools-api` fetch paths now address tabs by name only.

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
