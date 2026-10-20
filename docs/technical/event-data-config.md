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
  defaults and `display.teamColors` when its teams have colours. `sage-tools-api` never reads `type` or `display`, so it needs no
  backend change.
- `display.pairs` (team events only) labels each pair of a matchup, keyed by
  the pair number in a team code (the `3` of `A_3`):
  `"3": { "full": "Mixed Doubles", "short": "XD" }`. Both labels are non-empty
  strings. A type that repeats is numbered ("XD 1", "XD 2"). Without it the
  labels are MD, WD, XD 1 and XD 2; a malformed map falls back to those with
  a console warning on the page, and a pair number the map lacks reads
  "Pair <n>". The Hub, the board, the scorer page and Control Center read it.
- `display.teamColors` (team events only, optional) gives each team a colour,
  keyed by its group-stage team code: `"Red": "#D55E00"`. Values are hex
  colours (`#RGB` or `#RRGGBB`); any other value is dropped with a console
  warning on the page. Each coloured team gets a **mark**, its colour on a shape
  of its own, before its name wherever the name appears, and a matchup card or
  board card is tinted from one team's colour to the other's. The shapes follow
  the map's order: the first team is a circle, then square, triangle, diamond,
  star, hexagon, plus and an upside-down triangle, and a ninth team starts them
  again. A playoff side gets its team's mark once the organiser has typed the
  team into its slot; a team the map lacks gets none. The Millennium
  Invitational's colours are from the Okabe–Ito palette, which stays apart for
  most colour-blind readers, and the shape tells teams apart where colour
  can't. The Hub, the board and Control Center read it.
- `display.seedLabels` (team events only, optional) is each playoff side's
  placeholder until the organiser types its team in: `"Br-1": "Rank 3"` reads
  "Rank 3 · TBD" (`sideLabel`). Non-empty strings only; a side the map lacks
  reads "Seed <n> · TBD". The Hub, the board and the scorer page read it.
- `teamScoring` (team events only, optional) is the event's scoring rule.
  `"points"`, the default, decides a matchup on total points and ranks a
  bracket on points scored. `"games"` decides a matchup on matches won and then
  total points, and ranks a bracket on matches won, total points, head-to-head
  wins and head-to-head points. Anything else reads as `"points"`, with a
  console warning on the page. The Hub, the board, Control Center and its Awards
  podium read it; `sage-tools-api` does not.
- `lineupReveal` (team events only, optional): `true` makes the Hub and the
  schedule board show a match's lineup only once the published `CSV` tab's
  `lineupShown` column is `TRUE` for it (at the Millennium Invitational a formula
  reads it from the `MatchUps` tab's **Show lineup** tick box, one per matchup);
  until then they read it as a lineup not set
  (`buildDayModel`'s `hideUnshownLineups`, `buildScheduleData`'s option of the
  same name). Control Center always shows every lineup and marks a matchup whose
  lineup isn't shown yet. The names are still in the published snapshot, so this
  controls what the pages show, not who can read the data.
- `lineupRules` (team events only, optional) is what Control Center checks each
  lineup against, as warnings for operators (the public pages show none, and a
  warning blocks nothing). `pairs` gives each pair number a kind: `"mens-high"`
  (two men, at least one with a level in `advancedLevels`), `"mixed"` (a man and
  a woman), `"womens"` (two women) or `"open"` (any two). `advancedLevels` lists
  the roster levels that count as advanced, ignoring case; empty, the level isn't
  checked. `captainNotMixed: true` keeps a team's captain, the first player in its
  roster, out of the mixed pair. `allowed` names players the organiser lets into
  a kind whatever its gender rule says (`{ "Gian": ["womens"] }`: a man who may
  play Women's; he still counts as a man in Mixed). Every event with `lineupRules` also gets: a
  player plays once per round (matchup), and a round whose pairs are all in leaves
  no rostered player out. A player is matched to the roster by name, ignoring case
  and spaces; a check that needs a gender or level the roster doesn't give is
  skipped. Example: `"lineupRules": { "pairs": { "1": "mens-high", "2": "mixed", "3": "womens", "4": "open" }, "advancedLevels": ["HA", "HI"], "captainNotMixed": true }`.
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
meant to be public, and never keep private notes in an extra tab
of a spreadsheet that's registered here.

---
**Full reference:** `event-data/config/README.md` (this page is a condensed version).
**Related:** [Adding a new event](adding-a-new-event.md), [sync pipeline](sync-pipeline.md).
