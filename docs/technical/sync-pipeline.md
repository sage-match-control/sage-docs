# Sync pipeline

How a score typed into a Google Sheet ends up on the public site, and how
`sage-tools-api` knows which spreadsheets belong to which event without a
redeploy.

## The sheet → GitHub path

```
Facility Google Sheet
   |  installable onEdit trigger, debounced  (scripts/sheets-sync.gs)
   v
POST /sync/:day?facility=<name>   (X-Sync-Secret header, or an operator's bearer token)
   |  SheetsCsvFetcher (default) or GvizCsvFetcher (?method=csv fallback)
   |  merge with the currently-published snapshot — never drop a facility on failure
   v
GitHub Contents API commit -> event-data/<event-key>/data/<day>.json
   |  GitHub Pages redeploys on push (no cache-purge step)
   v
Event page fetches https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
```

Key pieces, in `sage-tools-api/src/sync/`:

- **`SheetsCsvFetcher`** — the default path, reads via the Sheets API.
- **`GvizCsvFetcher`** — a fallback path (`?method=csv`) using Google's
  `gviz` CSV export, for when the Sheets API path has trouble.
- **`SyncService`** — orchestrates a sync: resolves the day's config,
  fetches each facility (in parallel, via `ConcurrencyPool`), merges results
  with whatever's already published (a facility that fails to fetch keeps
  its last-known-good data rather than vanishing from the snapshot), and
  hands the result to the publisher.
- **`GitHubPublisher`** — wraps the Contents API. Used both to publish a new
  snapshot and (shared with the config store below) to read
  `config/events.json`.

Both fetchers are pure functions of `(facility, sheets)` — neither imports
config directly; `SyncService` resolves the day's sheet names once and
passes them down.

**Apps Script side:** `scripts/sheets-sync.gs`, installed once per facility
spreadsheet, watches that workbook's configured tabs (SCHEDULE and Court
Control by default) via an *installable* `onEdit` trigger (a bare `onEdit(e)`
can't call `UrlFetchApp`, which is why it has to be installed rather than the
default simple trigger), debounces edits, and POSTs to Cloud Run.

The file itself is **identical in every workbook** and holds no
spreadsheet-specific values. Everything that identifies a workbook — day key,
facility name, watched tabs — lives in that project's Script Properties, set
through **SAGE → Set up live sync**, a menu-driven dialog rather than edited
constants. Setup validates what it's given before saving anything:

- A `GET /sync/config` call, gated on the entered secret, checks the secret
  itself (401 blocks) and that the entered day key is registered (blocks with
  the real list of registered day keys otherwise).
- A real test sync (`POST /sync/:day?facility=`) checks the facility name —
  an "unknown facility" rejection blocks and names the valid alternatives —
  and, on success, publishes a real snapshot as end-to-end proof the workbook
  is wired up.
- A timeout, 5xx, or unreachable host at either step is **not** blocking: the
  configuration may be correct and Cloud Run simply unavailable, so setup
  saves anyway and reports a warning instead.

Identity values (day key, facility name, watched tabs) are only honoured for
the spreadsheet they were saved for, keyed off a stored spreadsheet ID — a
workbook copied from an already-configured one reads as unconfigured rather
than inheriting the source's day key and publishing over its snapshot. The
shared secret is **not** guarded this way, since it's the same one Cloud Run
env var for every workbook of every event.

### How the secret reaches a workbook

The secret is the one value an operator should never have to type, and the one
that has to survive a workbook being *copied* — a generated dual-meet workbook
starts life as a copy of the Dual Meet Master.

Script Properties can't do that job. They belong to the Apps Script project,
and copying a spreadsheet creates a new, empty one, so a secret set in the
Master's Project Settings reaches no copy. The secret is therefore stored as
**developer metadata on the spreadsheet itself**, which Drive does copy, under
the key `SAGE_SYNC_SECRET`. It is entered once on the Master through **SAGE →
Set shared secret**, which validates it against Cloud Run before writing.

Every copy made afterwards carries it. Setting up a facility workbook is
therefore day, venue and tabs only — the dialog reads *Secret stored* and never
asks. At the copy's first successful setup the value is promoted into that
workbook's own Script Properties, so it stands alone from then on and rotating
the Master's secret can't unconfigure a venue mid-event.

Only the secret is carried. Day key and facility name are deliberately excluded:
metadata copying is the whole point of it, so putting identity there would hand
every copy an inherited venue and defeat the guard above.

The **Set shared secret** menu item appears only where it belongs — on the
Master (labelled *Replace shared secret*, since that's the rotation path) and
on a hand-built workbook that carries no secret at all. Copies don't show it.
The secret now travels inside the spreadsheet file, so sharing a copy of the
Master shares the secret with it.

`sheets-sync.gs` also declares `onOpen`, contributing **Generate
Scoresheets** (deep-links into the Scoresheet Generator with this workbook's
day and venue preselected — a link only, no HTTP request and no new OAuth
scope), **Sync now**, **Pause live sync** / **Resume live sync**, **Live
sync settings** once configured (or **Set up live sync** otherwise),
**Set shared secret** where it applies (see above), and
**Help** — always present regardless of configuration state, since this is
the one file guaranteed to be in every workbook. Help shows a workflow
refresher plus this workbook's live status (configured/not, paused/not) in
one dialog, written for organizers rather than developers.

Pausing is for editing a watched tab (rosters, a mid-event schedule fix) without
publishing every intermediate state — it leaves the saved configuration and
the installed trigger alone, it just makes `onEditInstallable` and the
debounce trigger no-op until resumed. It's a backstop, not the primary
safeguard: the trigger doesn't exist at all until `Set up live sync` has run
once, so the normal workflow — finish rosters and schedule fixes, wire up
sync last — never needs it. `Sync now` still works while paused, since
that's an explicit manual action rather than the automatic edit-triggered
path pausing is scoped to. A generated dual-meet workbook carries both this file
and `sheet-generator.gs`, and Apps Script silently lets the last-loaded
file's `onOpen` win when two are declared — so both files declare a
**byte-identical** `onOpen` body that delegates to feature-detected builders
(`addSyncMenuItems_` in this file, `addGeneratorMenuItems_` in
`sheet-generator.gs`), each contributing only its own items and its own
leading separator. Which declaration wins can't matter, since both bodies are
the same text. Change one file's `onOpen`, change the other's to match — see
`sage-docs/docs/specs/sync-script-configuration-spec.md` §7 for the full
contract, and the divergences/design notes in
`sage-docs/docs/specs/scoresheet-event-picker-spec.md` §8.3 for why a
shared-contract shape was chosen over the alternatives.

## The runtime-fetched event registry

The event/day/facility registry — which spreadsheets exist, their
labels, their `isLive` state — lives at `event-data/config/events.json`, not
in `sage-tools-api` source. This is deliberate: source-code config means
adding an event or fixing a wrong sheet ID requires a full Cloud Build image
rebuild (`gcloud run deploy --source .`, which reinstalls Chromium). A
JSON file in a data repo means the same change is a commit, live within
about a minute, with no redeploy.

**`SyncConfigStore`** (`src/sync/SyncConfigStore.mjs`) owns fetching,
caching, and validating it:

- **Cache hit within the TTL** (`SYNC_CONFIG_TTL_MS`, default 60s) — no
  network call.
- **Cache miss** — fetches via `GitHubPublisher.fetchExisting()` (the same
  Contents API path used for snapshots), validates, swaps in.
- **Concurrent callers on a cold instance** share one in-flight request; a
  rejection isn't cached.
- **Fetch fails, something already held** — logs a warning, keeps serving
  the held config. A sync during a GitHub outage still fails, but at the
  same point it always would have (fetching/publishing the snapshot itself)
  — config availability doesn't add a new failure mode.
- **Fetch succeeds but validation fails** — the bad config is *refused
  wholesale*, not partially applied; the previous good config keeps serving,
  and an error names the failed rule. A typo in a commit can't crash-loop
  the service or half-apply.
- **Nothing held, no usable fetch** — falls back to a bundled seed
  (`events.seed.json`, a snapshot of the config as of the last deploy),
  logged at error level. This only fires on a cold start during a GitHub
  outage, or a cold start right after someone commits a broken config —
  exactly the scenario the whole mechanism exists to make safer.

Validation (rejects the whole file if any rule fails): a schema `version`
check, event/day keys restricted to `^[a-z0-9][a-z0-9-]*$` (they become
path segments and filenames — this is what makes a `../` traversal
impossible), day keys globally unique *across every event* (two events can
never race to publish into each other's folder, since a day key is also the
`/sync/:day` route), each day needs a `label` and a `facilities` array, and
facility names unique within a day. Full shape and rules:
[event registry schema](event-data-config.md).

**Observability:**

- `GET /ping`'s `X-Sync-Config` header reports the cached config's short
  SHA + source (`a1b2c3d/remote` or `seed/fallback`) — but never *triggers*
  a load itself, since Cloud Run's health checks hit this endpoint and it
  must stay fast even when GitHub is unreachable.
- `GET /sync/config` (secret- or token-gated) returns the full resolved
  view for debugging: SHA, source, age, every registered event/day.

## `/sync/:day/live` — the go-live override

Separate from a data sync: this endpoint (operator-token-only, no shared
secret) is Mission Control's public-site kill switch. It writes the day's
`isLive` value (`true` / `false` / `"auto"`) into `config/events.json`, then
immediately republishes that day's snapshot with the new value stamped in
— so a later real sync can never silently revert an operator's override,
since every sync re-reads `isLive` from the registry each time. See
[Auth](auth.md) for the token vs. shared-secret distinction.

## Sheet tabs are addressed by name, not GID

Both fetch paths address a facility's matches/standings tabs by name
(`matchesSheetName`/`standingsSheetName`, both overridable per day, default
`CSV`/`STANDINGSCSV`) — never by numeric tab GID, since a GID is assigned
per-workbook and doesn't carry over if a spreadsheet is ever duplicated
from another event's.

## Live delivery today, and a planned upgrade

Today, the client polls the published GitHub Pages snapshot directly on a
10-second interval (`POLL_INTERVAL_MS`), with a cache-busting query param
(GitHub Pages caches responses for ~10 minutes with no purge API, so this is
required, not optional). End-to-end, a score typed into a sheet takes
roughly **40–60 seconds** to reach a viewer — almost all of it GitHub Pages
rebuilding the site after the data commits.

**Not yet built:** a spec (`fast-data-delivery-spec.md`) proposes cutting
that to **~5–7 seconds** by adding Cloudflare R2 as a "hot" delivery path
in front of a tiny 3-second-polled pointer file (a hash + timestamp) that
only triggers a payload fetch when something actually changed, with GitHub
kept as a durability backup and automatic fallback. It's a well-scoped
addition to this pipeline, not a rewrite — the sheet→sync→GitHub half above
is unchanged — but it does add its own failure modes (R2 outages, CDN cache
misconfiguration, concurrent-write races) that the spec's acceptance
checklist is built around. Not reflected anywhere in this doc site's
architecture pages until it ships.

---
**Features:** [Control Center's Mission Control tab](../features/control-center.md#mission-control) uses this pipeline's resync/go-live actions.
