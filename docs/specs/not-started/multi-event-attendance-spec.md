# Spec — Attendance for every event

Make staff attendance work for any registered event and show up in Control
Center, without per-workbook Apps Script deployments.

> **Status: not started — idea only.** This page records the direction and
> what is already known, so a full spec can be written from it later. It is
> **not** an implementation guide: nothing here has been checked beyond what
> §2 says was verified, and §5 lists what has to be answered first.

Builds on [Event attendance](../implemented/event-attendance-spec.md), which
did this for Pickle for Sight only.

---

## 1. The problem

Attendance today is built for one event with two workbooks. Every piece that
scales with the number of workbooks is manual:

| Per workbook, today | Why it doesn't scale |
| --- | --- |
| Paste `attendance.gs` into its Apps Script project | By hand, per workbook |
| Deploy it as a web app, authorize, copy the `/exec` URL | By hand, per workbook |
| Record that URL in the page's hardcoded `VENUES` | Hardcoded per event page |
| Run `attendanceResync` once to pre-fill `ATTENDANCE` | By hand, per workbook |
| Run `attendanceResync` again after any player swap | By hand, and someone has to notice the swap |

For Pickle for Sight that is 2 workbooks. BKL Cup 2026 had 15 (5 days × 3
facilities).

## 2. What we already know

Verified during the Pickle for Sight build (2026-09-23):

- **The read side already scales.** The page reads `ATTENDANCE` straight from
  the workbook's CSV export by tab name —
  `https://docs.google.com/spreadsheets/d/<sheetId>/gviz/tq?tqx=out:csv&sheet=ATTENDANCE`
  — with no Apps Script involved. The fetch works cross-origin from
  `sage-match-control.github.io` (and from `localhost`), and it reflected a
  write in 407 ms and 702 ms across two runs.
- **Every registered workbook is already link-viewable.** The sync's
  `SheetsCsvFetcher` reads sheets with `GOOGLE_SHEETS_API_KEY`, and an API
  key only reaches sheets shared "Anyone with the link". So any facility
  sheet in `events.json` can serve the CSV read with no extra setup.
- **Every facility's `sheetId` is already in `event-data/config/events.json`**,
  under `events.<event>.days.<day>.facilities[]`, with its `name`. Control
  Center is already driven by that file. The page's hardcoded `VENUES` list
  duplicates information the registry already holds.
- **The write side is what doesn't scale.** Marking goes through a web app
  bound to each workbook (`doPost`, **Execute as: Me**, access: Anyone). A
  bound script can only write its own workbook, so every workbook needs its
  own deployment and its own URL.
- **All Apps Script web-app work shares one account's quota.** They run as
  the SAGE account, so every phone's write counts against that account's
  simultaneous-execution limit. This is why reads were moved to the CSV
  export.
- **Swaps don't self-heal any more.** Once reads came from `ATTENDANCE`
  as-is, a code/slot reassigned in `STANDINGSCSV` stays stale on the page
  until `attendanceResync` is run by hand.
- **The existing API can't write to Sheets.** It reads with an API key,
  which is read-only. This was why spec §8 of Event attendance chose Apps
  Script over Cloud Run for a single event.

## 3. The direction

Move writes into `sage-tools-api`, keyed by the registry, and drop the
per-workbook Apps Script entirely.

- **Writes.** A new endpoint on Cloud Run marks one player, addressed by
  day key + facility name (the same way `/sync/:day?facility=` is), and
  looks up the `sheetId` in the already-cached `events.json`. It writes
  through the Sheets API as Cloud Run's runtime service account, so no new
  secret or key file is needed. Each workbook just has to be shared with
  that service account's email.
- **Pre-fill and swaps become automatic.** Every edit to a facility sheet
  already fires `sheets-sync.gs` → `POST /sync/:day`, and the sync already
  reads `STANDINGSCSV`. After a sync, the server can reconcile `ATTENDANCE`
  against that roster, the same way `attendanceResync` does now: add missing
  players and reset rows whose name changed. No manual resync step. (Writes
  through the API shouldn't fire the installable `onEdit`, so this shouldn't
  loop — verify.)
- **Reads stay on the CSV export.** They already scale and stay off every
  quota discussed here.
- **One page, config-driven.** An attendance page (a tool, or a Control
  Center view) takes the event and day from `events.json` the way Control
  Center already does, instead of a per-event copy with a hardcoded
  `VENUES` list.
- **Control Center** uses its existing operator bearer token for writes.
  The desk-staff page keeps working without a sign-in, or gets a per-event
  PIN (open question, §5).

### Options considered

| | Per-workbook web app (now) | One standalone Apps Script web app | Writes in `sage-tools-api` (preferred) |
| --- | --- | --- | --- |
| Deploys per event | one per workbook | none after the first | none |
| Where write targets live | `/exec` URLs added somewhere | `events.json` (checked as an allowlist) | `events.json` (already cached) |
| Swaps | manual resync per workbook | manual resync | reconciled on every sync |
| Control Center sign-in | none | none | existing operator token |
| Cost | toil | broader scope: write access to any sheet the SAGE account owns | a service account shared on each workbook, a Google API dependency, an API deploy |

The standalone Apps Script option (`SpreadsheetApp.openById` with the
`sheetId` passed in, validated against `events.json`) is the fallback if the
service-account route runs into trouble.

## 4. What changes, roughly

- `sage-tools-api`: a write endpoint, a Sheets API client authenticated as
  the runtime service account, and an attendance reconcile step after each
  successful sync. Version bump and deploy.
- `event-data/config/events.json`: possibly nothing. The `sheetId`s are
  already there, and a per-event on/off flag for attendance may be all
  that's needed.
- `sage-match-control.github.io`: one config-driven attendance page and/or
  a Control Center attendance view, replacing
  `events/pickle-for-sight-2026/attendance.html`'s hardcoded pattern.
- `attendance.gs`: retired for new events. Pickle for Sight's two
  deployments get archived after the event anyway (its spec §6.4).
- Event setup (`_templates/CLAUDE.md`): one new step, sharing the workbook
  with the service account. It disappears entirely if the Drive-folder
  inheritance in §5 works.

## 5. Open questions — answer before writing the full spec

- **Sharing at scale.** Does a workbook copied into a Drive folder shared
  with the service account inherit that access? "Make a copy" drops the
  copy in My Drive by default, so this depends on where the generator
  handoff and the operator put copies.
- **Service-account auth on Cloud Run.** Confirm Sheets API calls work
  through the runtime service account's default credentials with no key
  file. Confirm which account that is (default compute, or a dedicated one).
- **Sheets API write quota.** The limit is per user per project, and a
  service account counts as one user — reportedly around 60 writes per
  minute (unverified). Is that enough for several desks checking in at once,
  plus the reconcile writes after every sync? Batching reconcile into one
  call per sync helps.
- **Concurrency without the script lock.** Cloud Run runs several instances
  with concurrency 4, so there's no `LockService` equivalent. If every
  player is pre-filled with a fixed row, a mark becomes an update to a known
  row rather than an append, which may remove the race altogether. Confirm.
- **Does an API write fire `onEdit`?** Expected no. If it does, reconcile
  could re-trigger the sync.
- **Staff access.** No sign-in (as now: anyone with the link can mark) or a
  per-event PIN? The page source is public, so no secret can live in it.
- **Where the reads come from in Control Center.** The CSV export straight
  from the browser, or through the API? Attendance must never land in
  `event-data`, which is public.
- **Multi-day events.** One `ATTENDANCE` tab per facility workbook already
  means one set of marks per day, since workbooks are per day. Check that
  holds for dual meets too.
- **Withdrawals.** Today a withdrawn pair's row is left in `ATTENDANCE`. Should
  the reconcile hide or remove it?
- **Cold starts.** `min-instances 0`, so a staff mark can land on a cold
  instance. Sync-only cold starts skip puppeteer, but check how long a mark
  waits in practice.

## 6. Out of scope for now

- Changing anything for Pickle for Sight (27 September 2026). Its per-workbook
  setup stays as built.
- Walk-ins added from the page, and showing attendance publicly. Both are
  unchanged from the original spec.
