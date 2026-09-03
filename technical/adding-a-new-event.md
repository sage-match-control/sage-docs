# Adding a new event

Instantiating a new tournament site is a copy-and-fill job against one of
two reusable templates in `sage-match-control.github.io/_templates/`, not a
copy-an-old-event-and-hunt-for-hardcoded-strings job. This page is a
condensed overview; the full step-by-step (with the complete token table)
is `_templates/CLAUDE.md` in that repo.

## Choosing a template

- **Two clubs facing off** → `dual-meet-template/`
- **Everything else** (open-entry bracket tournament) → `standard-tournament-template/`

Day count and category count don't affect this choice — both templates
handle any number of tournament days and divisions/events.
`dual-meet-template/` additionally handles exactly two clubs; it's not a
general multi-club template.

> **Dual meets: generate the scoring workbook, don't copy last event's.**
> These steps instantiate the *site*; they assume the event's Google Sheets
> already exist (step 8 installs the sync trigger into them). For a dual
> meet, build that workbook with the
> [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) — it produces
> every tab the event needs from the Tournament Calculator plan, and the
> workbook is ready to score once player names are pasted in. The full
> sequence, from planning to dry run, is
> [preparing an event](../features/preparing-an-event.md).

## The steps, in order

1. **Copy the template folder** into `events/<event-key>/` in the site
   repo. `<event-key>` becomes this event's folder name in *both* that repo
   and `event-data` — pick it once, keep it identical everywhere.
2. **Add the event's images** — a QR PNG, and (dual-meet only) both clubs'
   logos.
3. **Replace every `{{TOKEN}}`** — event identity, dates, venue, club
   names/codes. `grep -r '{{' events/<event-key>/` must come back empty
   when done.
4. **Fill in the example config** in `index.html`'s `CONFIGURATION`
   block — `DAYS`, `FACILITIES`, `DIVISIONS`, `EVENTS`, and (dual-meet)
   `CLUBS`.
5. **Leave the theme alone** unless the event genuinely needs its own — every
   page ships the shared S.A.G.E. palette and re-skinning means changing
   one `:root` block, nothing else.
6. **Set up the schedule board** (`schedule.html`) — the day key it shows,
   and `CAT_META` (each category's wall-display color, read off the source
   spreadsheet's own color-coding — see [schedule board](schedule-board.md)
   for why these can't be read any other way).
7. **Register the event** in `event-data/config/events.json` — `type`,
   `title`, one entry per day, optional `display` labels. See [event
   registry schema](event-data-config.md). Commit; no `sage-tools-api`
   redeploy needed, live within `SYNC_CONFIG_TTL_MS`.
8. **Install the sync script** (`scripts/sheets-sync.gs`, from
   `sage-tools-api`) once per facility spreadsheet — set `DAY_KEY`,
   `FACILITY_NAME`, `CLOUD_RUN_BASE_URL` in its `CONFIG` block, run
   `setup()` once to authorize and install the trigger.
9. **Create the data folder** in `event-data` — `<event-key>/data/`, or let
   the first successful sync create it.
10. **Copy the dry-run checklist** template into the event's own folder —
    a combined pre-event rehearsal (fake a few rows, verify the console
    catches them correctly, revert) and day-of runbook. See [Running an
    event, day of](../features/running-an-event-day.md) for the plain-
    language version of the day-of half.

## Required spreadsheet columns

Check this first if a new event's page loads but renders empty — the most
common cause. Exact, case-sensitive:

- **Matches tab:** `matchNumber`, `teamCode1`, `team1Player1`,
  `team1Player2`, `teamCode2`, `team2Player1`, `team2Player2`, `Schedule`,
  `team1Score`, `team2Score`, `CourtAssignment`, `court`. `court` is the
  *live* court (distinct from the scheduled `CourtAssignment`) — without it
  every court sits on "No match playing" forever.
- **Standings tab:** `teamCode`, `player1`, `player2`, `wins`, `loss`,
  `quotient`, `bracket`.

## Team code format

- Standard: `<DIVISION><EVENT>_<REST>` (e.g. `B18MD_1`, `HI40XD_SF_2`,
  `B35XD_F_1_(2)`)
- Dual meet: `<CLUB>_<DIVISION><EVENT>_<REST>`

`_(N)` suffixes mark a twice-to-beat playoff instance. See [Control
Center, incl. Awards tab](control-center.md) for how that gets parsed and
resolved.

## Root-absolute asset paths — don't "fix" them to relative

Both templates load icons and images with root-absolute paths
(`/assets/logo.png`, one shared `assets/` folder at the repo root), not
relative ones — a relative path resolves differently depending on how deep
a page is nested, so root-absolute paths work regardless of nesting. Leave
them that way; archiving an event later is then a plain `mv` with nothing
to re-prefix.

## Things kept in sync by hand (no automatic check)

- `event-data/config/events.json`'s `days` ↔ the `DAYS` array in the event
  page ↔ each spreadsheet's `DAY_KEY`/`FACILITY_NAME` in `sheets-sync.gs`.
  Facility names are compared exactly, case-sensitive.
- `EVENT_KEY` — identical across the site-repo folder name, the
  `event-data` folder name, and the registry key.

---
**Full runbook:** `_templates/CLAUDE.md` in the site repo (complete token
table, required-column detail, archiving instructions).
**Related:** [event registry schema](event-data-config.md), [Running an event, day of](../features/running-an-event-day.md).
