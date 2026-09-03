# Spec — Runtime-fetched sync config

Move the event/day/facility registry out of `sage-tools-api/src/sync/SyncConfig.mjs`
and into a JSON file in the shared `event-data` repo, fetched at runtime via the
GitHub Contents API.

**Goal:** adding an event, adding a day, or fixing a wrong sheet ID becomes a
commit to a data repo, not a Cloud Build image rebuild. Today `SyncConfig.mjs`
is source code, so every one of those requires `gcloud run deploy --source .`,
which rebuilds a Dockerfile that `apt-get install`s Chromium.

**Deliverable is the mechanism, not an event.** No new event is registered by
this task. §9 covers the one-time migration.

Line references are to the files as they stand today (post-templating):
`SyncConfig.mjs` 176 lines, `SyncService.mjs` 137, `GitHubPublisher.mjs` 92,
`SheetsCsvFetcher.mjs` 132, `GvizCsvFetcher.mjs` 88.

---

## 1. Why this is safe here

The standard objection to runtime-fetched config is that it puts a network
dependency in a live-event hot path. That objection does not apply:
`api.github.com` is **already** a hard dependency of every single sync —
`SyncService.syncDay()` calls `publisher.fetchExisting(path)` (`SyncService.mjs:81`)
and then `publisher.publish(...)` (`:117`). If GitHub is unreachable, the sync
fails regardless of where config lives.

So this adds one more call to a service the sync already cannot work without —
not a new class of failure. And that call is cacheable far more aggressively
than the snapshot read, because config changes at human speed.

The real win is **mid-event agility**, not per-event setup convenience. Setup is
dominated by hand-installing Apps Script on each facility spreadsheet; one extra
deploy is noise next to that. The scenario that actually hurts is a facility
spreadsheet being recreated the morning of an event and needing a full image
rebuild to correct one ID while matches are starting.

---

## 2. Settled decisions

**D1 — Config lives in the `event-data` repo at `config/events.json`.**
Same repo the snapshots already go to. `GITHUB_TOKEN` already has
`Contents: Read and write` there, it is already the per-event home, and it means
one repo to look at when debugging. Not a separate `sage-config` repo — that
would be another repo and another token scope for no benefit.

**D2 — Read it through the Contents API, not GitHub Pages.**
Pages caches responses ~10 minutes with no purge API (the site templates already
fight this with a cache-busting query param, see `snapshotUrlFor`). Config
staleness there would be silent and confusing. `GitHubPublisher.fetchExisting()`
(`GitHubPublisher.mjs:52`) already does exactly the right request and returns
`{ json, sha }` — reuse it verbatim.

**D3 — In-memory cache with a TTL, plus last-known-good on failure.**
A warm instance does not re-fetch config per sync. A fetch failure falls back to
the last good config it holds rather than failing the sync.

**D4 — Validation rejects a bad config instead of crashing the process.**
Today a duplicate day key throws at module load (`SyncConfig.mjs:101-112`), which
is a good fail-fast for source code but wrong for remote data — it would turn a
typo in a data repo into a crash-looping service. Validation stays, but a config
that fails it is *refused* and the last-known-good is kept.

**D5 — Tab-name/GID defaults stay in code.**
`DEFAULT_MATCHES_SHEET_NAME` / `DEFAULT_STANDINGS_SHEET_NAME` / `DEFAULT_CSV_GID`
/ `DEFAULT_STANDINGS_GID` (`SyncConfig.mjs:15-18`) are platform defaults, not
per-event data. The JSON may override any of them per day, exactly as a day entry
can today.

> **Partially superseded.** The GID half of this decision was implemented as
> written, then removed once it caused a real incident: a workbook duplicated
> from another event's inherits that event's tab GIDs, so a GID-based override
> correct for one spreadsheet silently pointed at the wrong tab in another
> (see `event-data/config/README.md`, and `pnf-x-bup-dual-meet`'s day entry in
> `events.json` for the specific case). Both fetch paths now address tabs by
> name only — `GvizCsvFetcher.mjs` builds its gviz URL with `&sheet=<name>`
> instead of `&gid=<gid>`. `DEFAULT_CSV_GID`/`DEFAULT_STANDINGS_GID` and the
> per-day `csvGid`/`standingsGid` override fields described in §3 and §6
> below no longer exist. The tab-*name* half of D5 is unchanged.

**D6 — `/sync/:day` stays exactly as it is.**
Day keys remain globally unique, so the event is still derivable from the key.
The 15 Apps Script installs already on the bkl spreadsheets keep working
untouched.

---

## 3. The config file

`event-data/config/events.json`:

```json
{
  "version": 1,
  "_comment": "Source of truth for sage-tools-api's sync registry. See config/README.md.",
  "defaults": {
    "matchesSheetName": "CSV",
    "standingsSheetName": "STANDINGSCSV",
    "csvGid": "2136121736",
    "standingsGid": "1507029786"
  },
  "events": {
    "bkl-cup-2026": {
      "_comment": "Ended Aug 2026. Archived pages read from the pre-migration location; a resync now publishes to event-data/bkl-cup-2026/data/ instead. Harmless — nothing triggers one.",
      "days": {
        "day2": {
          "label": "Day 1 · Aug 15",
          "facilities": [
            { "name": "Main", "sheetId": "1hbjqbH3H1Z-..." }
          ]
        }
      }
    }
  }
}
```

- `defaults` is optional; anything it omits falls back to the in-code default (D5).
- A day may override any subset of the four `defaults` keys.
- `version` is a schema version, checked on load so a future shape change fails
  loudly rather than silently mis-parsing.
- JSON has no comments, and the comments currently in `SyncConfig.mjs` are
  load-bearing documentation (the day-key offset history, the `event-data`
  migration note). `_comment` string keys carry the short ones; the rest move to
  `config/README.md`, written as part of this task.

### 3.1 Validation rules

Run on every load, before a config is accepted:

| Rule | Why |
| --- | --- |
| `version` equals the supported version | Fail loudly on a future shape change |
| Event keys match `^[a-z0-9][a-z0-9-]*$` | **Path safety** — the key becomes a path segment in `repoPathFor()`; `../` must be impossible |
| Day keys match `^[a-z0-9][a-z0-9-]*$` | Same — the key becomes the filename, and arrives from the URL |
| Day keys globally unique across events | Preserves today's guard (`SyncConfig.mjs:101-112`); stops one event publishing into another's folder |
| Each day has a non-empty `label` and an array `facilities` | `SyncService` uses `label` in every error message |
| Facility names unique within a day | `SyncService` filters by name (`:52`); duplicates are ambiguous |
| At least one event, each with at least one day | Catches a truncated/empty file |

Path safety is the one genuinely new concern. Today event keys are hardcoded in
source, so a traversal is impossible by construction; once they arrive from a
fetched file, `repoPathFor()` (`SyncConfig.mjs:161`) would happily build
`../../foo/data/day2.json`. The slug regex is what closes that.

### 3.2 Sheet IDs become publicly listed

`event-data` is public (GitHub Pages requires it on a free plan), so this file
publishes the list of spreadsheet IDs. Sheet IDs are not secrets — the sheets are
already "anyone with the link can view" by design, and the published snapshots
already contain all their match data. But it does make the set of spreadsheets
enumerable, including tabs that are *not* published. Note in `config/README.md`:
only put IDs of spreadsheets whose sharing is already link-viewable, and don't
keep private working notes in a tab of a synced spreadsheet.

---

## 4. New: `SyncConfigStore`

`src/sync/SyncConfigStore.mjs` — owns fetching, caching, validating.

```js
export class SyncConfigStore {
  constructor(publisher, logger, { path = "config/events.json", ttlMs = 60000, fallback = null }) {}

  // Resolves to a SyncConfigSnapshot (§5). Cached for ttlMs.
  async get() {}
}
```

Behavior:

- **Cache hit within TTL** → return the held snapshot, no network.
- **Cache miss / expired** → `publisher.fetchExisting(path)`, validate, swap in.
- **In-flight de-duplication** → concurrent callers on a cold instance share one
  promise, and a rejection is not cached. Same pattern as
  `getScoresheetService()` in `index.mjs:49-65` — follow it.
- **Fetch fails, snapshot held** → log a warning, keep serving the held one.
- **Fetch succeeds but validation fails** → log an error naming the failed rule,
  keep serving the held one. Never swap in an invalid config.
- **Nothing held and no usable fetch** → fall back to the bundled seed (§4.1);
  if that is also unusable, throw `SyncConfigUnavailableError` (503).

TTL default 60s, overridable via `SYNC_CONFIG_TTL_MS`. At 60s a live event costs
at most one small config read per instance per minute, and a config fix is live
within a minute without a redeploy.

### 4.1 Bundled fallback seed

`src/sync/events.seed.json`, shipped in the image, is a copy of the config as of
the last deploy. It is used **only** when an instance has nothing cached and the
remote fetch or validation failed — i.e. a cold start during a GitHub outage, or
a cold start after someone commits a broken config mid-event. That second case is
exactly the scenario this whole change exists to make safer, so the seed earns
its keep.

It is not the source of truth and must say so in a `_comment` at its top. When
the seed is used, log at error level and reflect it in the diagnostics (§7) so a
silently-stale fallback can't masquerade as healthy.

---

## 5. `SyncConfig` becomes a resolved snapshot

Today's `SyncConfig` is a class of statics over a module-level `EVENTS` const.
Turn it into an instance over injected data, constructed by the store after
validation. **The method surface stays synchronous and identical**, so only the
construction site changes:

```js
export class SyncConfigSnapshot {
  constructor({ events, defaults, sha, loadedAt, source }) {}

  getDay(day)                 // unchanged shape: { day, event, label, facilities }
  knownDays()
  repoPathFor(event, day)
  sheetsFor(day)              // NEW — see §6
}
```

`getDay()` keeps filtering out facilities with an empty `sheetId`
(`SyncConfig.mjs:146`) and keeps `event` additive on the result. `knownDays()`
still spans every event so `UnknownSyncDayError` keeps listing something useful.
`repoPathFor()` is unchanged.

`SyncService` gains the store as a constructor dependency and resolves once per
sync:

```js
async syncDay(day, { facilityName = null, method = "sheets" } = {}) {
  const config = await this.configStore.get();
  const { event, label, facilities: allFacilities } = config.getDay(day);
  ...
}
```

Everything after that line in `syncDay` is unchanged except `SyncConfig.repoPathFor`
→ `config.repoPathFor` (`:80`).

---

## 6. Fetchers stop importing config

Both fetchers currently reach back into config themselves:
`SheetsCsvFetcher.mjs:73-74` calls `SyncConfig.matchesSheetName(day)` /
`standingsSheetName(day)`, and `GvizCsvFetcher.mjs:33-34` calls
`SyncConfig.csvUrl(day, sheetId)` / `standingsUrl(day, sheetId)`.

Replace with a resolved spec passed down from `SyncService`:

```js
const sheets = config.sheetsFor(day);
// { matchesSheetName, standingsSheetName, csvGid, standingsGid } — defaults applied
const results = await Promise.allSettled(
  targetFacilities.map(f => csvFetcher.fetchFacility(f, sheets))
);
```

- `fetchFacility(facility, day)` → `fetchFacility(facility, sheets)` in both
  fetchers. Same `{ name, matchesCsv, standingsCsv }` output, so nothing
  downstream changes.
- `csvUrl()` / `standingsUrl()` are **deleted** from config; `GvizCsvFetcher`
  builds its own gviz URLs from `csvGid` / `standingsGid`. Those URLs are a
  gviz implementation detail and belong in the gviz fetcher.
- Both fetchers drop `import { SyncConfig }` entirely and become pure functions
  of their arguments.

Worth doing regardless of where config lives — it is the right shape either way.

Also update the error text at `SheetsCsvFetcher.mjs:93-95`, which currently tells
the operator to "check the tab names in SyncConfig.mjs" — that file will no
longer be where tab names live.

---

## 7. Observability

Knowing *which* config is live is essential once it can change without a deploy.

- **`GET /ping`** gains an `X-Sync-Config` response header: short SHA plus source,
  e.g. `a1b2c3d/remote` or `seed/fallback`. It must report only what is
  **already cached** and must never trigger a load — Cloud Run health checks hit
  this endpoint, and it must not fail or slow down when GitHub is down. Omit the
  header when nothing has been loaded yet.
- **`GET /sync/config`** (new, behind the same `requireSyncSecret` middleware as
  `POST /sync/:day` in `routes.mjs:16`) returns the resolved view for debugging:
  `{ sha, source, loadedAt, ageMs, events: [...], days: [...] }`. Never returns
  the token or the shared secret.
- **Mission Control**: the connection check in both templates'
  `match-control.html` (`:3018-3022`) already reads `X-App-Version` off `/ping`
  and prints it. Append the config SHA to that same line, so
  "Connected to Cloud Run — 210ms · v1.4.0 · cfg a1b2c3d". Mirror the change into
  both templates (they are maintained as a pair).

---

## 8. Failure modes

| Situation | Behavior |
| --- | --- |
| Warm instance, config unchanged | No network call at all (TTL cache) |
| Config edited in `event-data` | Live within `SYNC_CONFIG_TTL_MS` on every instance, no deploy |
| GitHub unreachable, instance warm | Sync still fails — but at `fetchExisting`/`publish`, as it does today. Config is not the new cause |
| GitHub unreachable, cold start | Bundled seed (§4.1) is used; logged at error level. Sync still fails at publish, for the same pre-existing reason |
| Broken config committed, instance warm | Rejected by validation; last-known-good kept; error logged naming the rule |
| Broken config committed, cold start | Bundled seed used; logged at error level; `/sync/config` shows `source: "fallback"` |
| Duplicate day key across two events | Config refused wholesale (not partially applied) — same protection as today's module-load throw |
| Event key containing `../` | Config refused by the slug rule (§3.1) |

---

## 9. Migration (one-time)

1. Create `config/events.json` in `event-data` from the current `EVENTS` object
   in `SyncConfig.mjs:32-93`, verbatim — same day keys, same sheet IDs, same
   labels. Carry the bkl `_comment` notes across.
2. Write `config/README.md` in that repo: the shape, the validation rules, the
   sheet-ID caution (§3.2), and a pointer to `_templates/CLAUDE.md`.
3. Implement `SyncConfigStore` + `SyncConfigSnapshot`; delete the `EVENTS` const
   and the module-load `DAY_INDEX` loop from `SyncConfig.mjs`.
4. Rewire `SyncService` (store injection, `config.repoPathFor`, `sheetsFor`) and
   both fetchers (§6). Wire the store in `index.mjs` next to the publisher —
   it needs the same `GitHubPublisher` instance.
5. Commit the seed (§4.1) as a copy of step 1's file.
6. **Deploy once.** This is the last deploy required to add or change an event.
7. Verify against the acceptance checklist (§10) before touching the runbook.
8. Update `_templates/CLAUDE.md`: §2 step 6 becomes "edit `config/events.json` in
   `event-data`", and **step 7 (commit and deploy `sage-tools-api`) is deleted**
   from the per-event path. Add a line to §0 noting the config file's location
   and that a change to it is live within ~a minute. Also fix the note in §2
   step 6 that points at "the defaults at the top of `SyncConfig.mjs`".
9. Update the root `CLAUDE.md`: the `SyncConfig.mjs (day/facility registry)`
   description in the Layout section, and the "Things that must be kept in sync
   by hand" bullet, which currently names `SyncConfig.mjs EVENTS[<event>].days`.

**Rollback:** revert the code change and redeploy. The config file can stay in
`event-data` — an old build simply ignores it.

---

## 10. Acceptance checklist

**Behavior**

- [ ] `POST /sync/day2` still resolves to the `bkl-cup-2026` event and publishes
      to `bkl-cup-2026/data/day2.json` — the move must not change what any
      existing day key means.
- [ ] Editing `config/events.json` (e.g. changing a `label`) is reflected by a
      running instance within `SYNC_CONFIG_TTL_MS`, with no redeploy.
- [ ] A second sync within the TTL makes no additional config request.
- [ ] Committing a config with a duplicate day key across two events leaves a
      warm instance serving the previous config, with an error logged.
- [ ] Committing a config with an event key of `../evil` is refused.
- [ ] With the remote config unreachable and nothing cached, the bundled seed is
      used and the fact is logged at error level.
- [ ] Two concurrent syncs on a cold instance trigger exactly one config fetch.
- [ ] `?method=csv` still works, using per-day GID overrides where present.
- [ ] A day whose facilities all have empty `sheetId` still raises the existing
      `ValidationError` ("no facilities configured yet").

**Shape**

- [ ] Neither fetcher imports `SyncConfig` any more.
- [ ] `csvUrl` / `standingsUrl` no longer exist in the config module.
- [ ] No sheet ID, event key, or day key remains hardcoded in `sage-tools-api`
      source, except inside `events.seed.json`.
- [ ] `GET /ping` responds normally (and fast) with GitHub unreachable.

**Docs**

- [ ] `event-data/config/README.md` exists and documents the shape + validation.
- [ ] `_templates/CLAUDE.md` no longer tells the reader to deploy when adding an
      event, and its §0/§2 references to `SyncConfig.mjs` are corrected.
- [ ] Root `CLAUDE.md` describes the registry's new home.

---

## 11. Out of scope

- Registering any new event.
- Migrating the published `bkl-cup-2026` snapshots into `event-data`.
- An admin UI for editing the config. Editing JSON in GitHub's web editor is the
  interface.
- CI validation inside the `event-data` repo. The service-side validation in §3.1
  is the guard; a pre-commit check there can come later if it proves needed.
- A `npm run validate-config` CLI. Useful, but not required for correctness given
  last-known-good + seed.
- Any change to the scoresheet half of `sage-tools-api`, to auth, or to the
  `/sync/:day` route shape.
- Per-event GitHub repos or a second Cloud Run service.
