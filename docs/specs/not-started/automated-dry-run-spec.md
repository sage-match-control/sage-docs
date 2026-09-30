# Spec — Automated dry run

Run Part 1 of an event's Control Center runbook (the rehearsal) with one
command, instead of someone working through it by hand.

> **Status: not started — idea only.** This page records what is known and
> what has to be proven before a full spec is written. It is **not** an
> implementation guide: §2 separates what was checked against the code from
> what is only believed, and §4 lists the spikes that settle the rest.

The runbook is `_templates/dry-run-checklist-template.md` in
`sage-match-control.github.io`, copied per event into
`events/<event-key>/dry-run-checklist.md`.

---

## 1. The problem

The rehearsal is manual, takes an operator's full attention, and is the step
most likely to be skipped when an event is close. It exists to prove one
chain end to end:

```
human edit in the facility sheet
  -> installable onEdit trigger (sheets-sync.gs)
  -> debounced POST /sync/:day
  -> event-data/<event-key>/data/<day>.json
  -> Control Center + public pages render it correctly
```

Part 2 of the runbook (the day itself) is out of scope: venue wifi, HDMI,
a phone check and judgement calls during play are not automatable.

## 2. What we already know

### Checked against the code

- **Only a UI edit tests the trigger.** `onEditInstallable` fires on user
  edits only; script and API edits don't fire it (`sheets-sync.gs` says so
  itself). So anything that writes through the Sheets API or Apps Script has
  to call `/sync` directly and skips the one link the rehearsal exists to
  prove.
- **Control Center's *Resync this day now* does not exercise Apps Script.**
  It calls `POST /sync/:day` on Cloud Run straight from the browser, and
  Cloud Run reads the sheet through the Sheets API. It proves the sheet ID is
  registered and readable, not that the trigger works. The runbook's §2.1
  says so.
- **Sync timing.** `DEBOUNCE_MS` is 3000, and the delayed trigger is
  best-effort: it can fire up to about a minute late. A checker has to poll
  the published snapshot for a change, not sleep a fixed time.
- **A paused workbook silently does nothing.** With **SAGE → Pause live
  sync** on, `onEditInstallable` returns before scheduling anything. A
  missing sync after an edit therefore means "trigger broken, *or* paused, or
  unconfigured" — a checker must say so rather than report a bare timeout.
- **Rendering can be tested without the sheet.** The console already loads
  `_fixtures/` on localhost via `?fixture=<name>`. The rehearsal's three sheet
  edits (one finished match, one live court, one unmapped category code) only
  exist to produce a snapshot in a known state; a script can build that
  snapshot directly from the real published one. Only PickleDrive has
  fixtures today.
- **Team codes and player names are often formulas.** Overwriting one in the
  UI deletes the formula. The published snapshot only carries the computed
  value, so restoring from it would leave a plain value behind.
- **Live/Hide is a real write.** `POST /sync/:day/live` commits to
  `event-data/config/events.json`. A test of it is a real config change and
  must always end on `auto`.
- **Per-type branches.** The team-event runbook has extra steps (the
  `MatchUps` lineup and seed cells, e.g. `D604`), so checks vary by the
  event's `type`.
- **Puppeteer is already a `sage-tools-api` dependency.**

### Believed, not yet verified

- Puppeteer keystrokes (`page.keyboard`) arrive as trusted input, so Sheets
  treats them as a user edit and the installable trigger fires.
- Sheets draws its grid on a canvas, so cells can't be targeted by selector.
  They can be reached by opening `…/edit#gid=<gid>&range=<A1>`, or through
  the Name Box (currently `#t-name-box`).
- Google refuses sign-in from an automated browser. A dedicated Chrome
  profile directory, signed into by hand once and then reused by Puppeteer,
  avoids that — and keeps the password out of the script.
- **Ctrl+Z** in the same browser session restores an overwritten formula
  exactly, and fires `onEdit` like any other edit.
- Sheets autocomplete can change typed text: Enter accepts a suggested
  completion when the typed value is a prefix of one already in the column.

## 3. Proposed shape

One Node script, `sage-tools-api/scripts/verify-event-dry-run.mjs
<event-key> <day>`, beside the existing `verify-*` scripts, in three layers
that each run on their own:

| Layer | Does | Touches the live workbook? |
| --- | --- | --- |
| **Preflight** | `GET /ping` (version, config sha), `GET /sync/config` (event and day registered), `POST /sync/:day`, every facility in the snapshot recently synced, go-live state | No |
| **Rendering** | Builds a fixture from the real snapshot with matches A/B/C applied, opens the console and public pages on localhost with `?fixture=`, asserts the runbook's §1.2–§1.3 checks against the real pages | No |
| **Rehearsal** | Drives the real facility sheet in Puppeteer: records originals, makes the three edits, waits for the snapshot to change, runs the rendering checks against live data, then restores | **Yes** |

Rendering assertions run against the real console code, never a Node
reimplementation of its rules: the played/BYE and team-event rules already
exist twice, and a checker must not become a third copy.

With the rehearsal layer in place, the runbook's Part 1 shrinks to whatever
the script can't cover.

## 4. Before the full implementation

Spikes, in order. Each has a pass condition; a failure changes the design.

1. **Sign-in and editing spike.** Puppeteer with a dedicated profile opens a
   *copy* of a facility workbook, types into one cell, and reads it back.
   Passes if the edit persists and the Apps Script execution log shows an
   `onEditInstallable` run. If Google blocks the profile, the rehearsal layer
   becomes "Claude in Chrome with an operator watching" instead of a script.
2. **Cell targeting.** Confirm `#gid=…&range=…` selects the cell on load, and
   whether it still works when the tab is already open. Fall back to the Name
   Box if not.
3. **Restore.** Overwrite a formula cell, then restore it with Ctrl+Z; confirm
   the formula (not its value) is back and that the undo also syncs.
4. **Autocomplete.** Confirm whether it fires on team-code columns and the
   reliable way to suppress it (Delete before Enter, or reading back).
5. **Court Control input.** Check whether its cells carry data validation
   that would reject or flag the test value.
6. **Choosing matches A/B/C automatically** from the snapshot, per event
   type — including what the "unmapped code" edit is for a team event.

Then, before writing the full spec:

- Decide where operator credentials for the console come from (local `.env`,
  never committed) and whether the rehearsal layer tests Live/Hide at all.
- Decide how a crashed run is recovered: the script writes originals to a
  local restore file first, and needs a `--restore` mode that works without
  the browser session Ctrl+Z depends on.

## 5. Open questions

- One facility per run, or every facility workbook for the day?
- Should the rehearsal layer refuse to run once the day is live (`auto` past
  its go-live time, or `true`)?
- Does the rendering layer need its own fixtures committed per event, or are
  they always generated fresh and discarded?
