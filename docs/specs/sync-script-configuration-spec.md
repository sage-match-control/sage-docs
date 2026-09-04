# Spec — Live sync script configuration

Move `sheets-sync.gs`'s per-spreadsheet configuration out of source and into
Script Properties, collected through a menu-driven setup dialog that validates
what it is given before saving it. Ship the sync script inside the Dual Meet
Master so a generated workbook needs no code pasting at all.

Two files change, both in `sage-tools-api/scripts/`:

| File | Change |
| --- | --- |
| `sheets-sync.gs` | config → Script Properties; setup dialog; `SAGE` menu; validation (§2–§7) |
| `sheet-generator.gs` | shared `onOpen` contract; post-generation marker (§6, §8) |

**Neither is a deploy.** Both run inside a spreadsheet and ship by being pasted
into that workbook's script project. Do **not** bump `package.json` and do not
add a `README.md` changelog entry — nothing about what Cloud Run runs is
affected. `node scripts/verify-sheet-generator.mjs` must still pass.

---

## 1. The failure mode this closes

`sheets-sync.gs` carries three per-spreadsheet values as source constants:

```js
const DAY_KEY = 'day2';
const FACILITY_NAME = 'Main';
const WATCHED_SHEET_GIDS = ['331216315', '359580789'];
```

Its install procedure is "paste this whole file in", which is also how an update
is applied — and a paste resets those lines to the shipped defaults.

`day2` + `Main` is a **registered, valid sync target**: BKL Cup 2026's first
day, Main facility. `archived` is a Control Center display flag only —
it appears nowhere in `sage-tools-api/src/sync/`, so `POST /sync/day2` resolves
and publishes exactly as a live event would.

A forgotten re-edit after a paste therefore does not fail. It publishes that
workbook's `CSV` tab into `bkl-cup-2026/data/day2.json` — wrong event, no error
raised anywhere, and the only symptom is a page that stops updating. Day keys
being globally unique is what guarantees the mistake lands on a valid target
instead of a 404.

The fix is that configuration must not live in the bytes being overwritten.

---

## 2. Property model

All configuration lives in `PropertiesService.getScriptProperties()`.

| Key | Holds | Copy-inheritable |
| --- | --- | --- |
| `SYNC_SHARED_SECRET` | the shared secret | **Yes** |
| `SYNC_DAY_KEY` | this workbook's day key | No |
| `SYNC_FACILITY_NAME` | this workbook's facility name | No |
| `SYNC_WATCHED_GIDS` | JSON array of GID strings | No |
| `SYNC_BOUND_SPREADSHEET_ID` | the spreadsheet these identity values were saved for | — |
| `TABS_GENERATED_FOR` | the spreadsheet `generateEventTabs` ran against | — |

`PROP_LAST_EDIT` (`lastEditTime`) is unrelated debounce bookkeeping and is left
exactly as it is.

### 2.1 The secret is shared; identity is not

The secret is one Cloud Run environment variable, identical for every workbook
of every event, so a copy inheriting it is correct and saves re-entering an
opaque string per workbook.

The day key and facility name identify *this* workbook. A copy inheriting them
is precisely the §1 failure in a new form — so they are guarded.

### 2.2 The spreadsheet-ID guard

Identity values are only honoured when they were saved for *this* spreadsheet:

```js
function readSyncConfig_() {
  var props = PropertiesService.getScriptProperties();
  var boundTo = props.getProperty(PROP_BOUND_SPREADSHEET_ID);
  if (boundTo !== SpreadsheetApp.getActive().getId()) {
    return null;   // unconfigured: copied from another workbook, or never set
  }
  return {
    dayKey: props.getProperty(PROP_DAY_KEY),
    facilityName: props.getProperty(PROP_FACILITY_NAME),
    watchedGids: JSON.parse(props.getProperty(PROP_WATCHED_GIDS) || '[]')
  };
}
```

The secret is read separately and is **never** subject to this guard:

```js
function readSharedSecret_() {
  return PropertiesService.getScriptProperties().getProperty(PROP_SHARED_SECRET);
}
```

This is what makes §8 safe. Whether Google carries Script Properties across a
spreadsheet copy is a platform behaviour this spec deliberately does not depend
on: if properties copy, the secret is inherited and identity is rejected by the
guard; if they do not, everything is simply unset and the dialog asks for all of
it. Both paths are correct.

**Verify the behaviour anyway before implementing §8** — set a property on the
Master, copy it, read the property back in the copy. The result decides only
whether the dialog's secret field is usually pre-satisfied (§5.2), not whether
the design holds.

### 2.3 Constants that stay in source

```js
const CLOUD_RUN_BASE_URL = 'https://sage-tools-api-811926984834.us-central1.run.app';
const DEBOUNCE_MS = 10000;
const SETTLE_HANDLER = 'runIfSettled';
const SCHEDULE_TAB_NAME = 'SCHEDULE';
const COURT_CONTROL_TAB_NAME = 'Court Control';
```

Identical in every install, none of them secret. `SCHEDULE_TAB_NAME` and
`COURT_CONTROL_TAB_NAME` are new and feed §4.

> **`Court Control`, not `COURT CONTROL`.** `sheet-generator.gs:229` creates the
> tab as `Court Control`. Name-based lookup must use that exact string.

### 2.4 Reading config at sync time

`triggerSync_()` reads from `readSyncConfig_()` and `readSharedSecret_()`
instead of module constants, and returns without acting when either is missing:

```js
function triggerSync_() {
  var config = readSyncConfig_();
  var secret = readSharedSecret_();
  if (!config || !config.dayKey || !config.facilityName) {
    console.error('Live sync is not configured for this spreadsheet. Run SAGE -> Set up live sync.');
    return;
  }
  if (!secret) {
    console.error('No shared secret stored. Run SAGE -> Set up live sync.');
    return;
  }
  // ...unchanged from here: URL build, UrlFetchApp.fetch, status logging...
}
```

`onEditInstallable` gains the same early exit — read config first, and return
before touching `PROP_LAST_EDIT` or scheduling a trigger when it is null. An
unconfigured workbook must do nothing at all rather than schedule work that
cannot succeed.

---

## 3. Validation

Setup validates before it saves. Two of the three checks are blocking.

### 3.1 Secret and day key — `GET /sync/config`

```
GET {CLOUD_RUN_BASE_URL}/sync/config
X-Sync-Secret: <entered secret>
```

Returns `{ sha, source, loadedAt, ageMs, events: [...], days: [...] }`.

- **401** → the secret is wrong. Blocking. `The shared secret was rejected.
  Check it against Cloud Run's SYNC_SHARED_SECRET.`
- **200, and `days` excludes the entered day key** → blocking. Report it with
  the real list, which is exactly what the operator needs to pick from:
  `Day key "<entered>" is not registered. Registered day keys: <days joined>.`
- **200 with the day key present** → proceed.
- **Network failure / 5xx** → non-blocking (§3.3).

### 3.2 Facility name — a real test sync

```
POST {CLOUD_RUN_BASE_URL}/sync/<dayKey>?facility=<encoded facilityName>
X-Sync-Secret: <secret>
```

`SyncService` rejects an unknown facility with a message naming the valid ones:

```
<label>: unknown facility "X". Known facilities: Main, Annex, Dreamcourts
```

- **Non-2xx whose body contains `unknown facility`** → blocking. Surface the
  response body verbatim; it already names the alternatives.
- **2xx** → success. Report it, including the response body's counts.
- **Any other non-2xx, or a network failure** → non-blocking (§3.3).

Facility names are compared exactly and case-sensitively by the sync itself.
Do not trim or case-fold the entered value to be helpful — that would hide the
mismatch this check exists to catch.

This publishes a real snapshot. That is intended: it is the end-to-end proof the
workbook is wired up. Running it before rosters are pasted publishes a snapshot
with empty player names, which the next edit replaces — harmless, and worth a
line in the dialog's success message so it is not mistaken for a fault.

### 3.3 Non-blocking failures

A timeout, a 5xx, or an unreachable host means the configuration may be perfectly
correct and Cloud Run is simply unavailable. Save the configuration, install the
trigger, and report a warning rather than an error:

```
Saved, but the test sync could not reach Cloud Run (<reason>).
Use SAGE -> Sync now to retry once it is reachable.
```

Only a **rejected secret**, an **unregistered day key**, and an **unknown
facility** block the save. Each of those is a value that is definitively wrong.

---

## 4. Resolving watched GIDs

The watched tabs are resolved by name at setup and their GIDs stored. Matching
on GID at edit time is retained, so renaming a tab afterwards does not break the
trigger.

```js
function resolveWatchedGids_(selectedNames) {
  var ss = SpreadsheetApp.getActive();
  var gids = [];
  selectedNames.forEach(function (name) {
    var sheet = ss.getSheetByName(name);
    if (sheet) gids.push(String(sheet.getSheetId()));
  });
  return gids;
}
```

The dialog presents every tab in the workbook as a checkbox list, pre-ticking
`SCHEDULE` and `Court Control` where they exist. A generated workbook needs no
interaction here; a hand-built one whose tabs are named differently is a matter
of ticking the right boxes rather than reading GIDs out of tab URLs.

At least one tab must be ticked — blocking, with
`Pick at least one tab to watch for score edits.`

Provide the tab list to the dialog from a server function:

```js
function listSheetTabs() {
  return SpreadsheetApp.getActive().getSheets().map(function (s) {
    return { name: s.getName(), gid: String(s.getSheetId()) };
  });
}
```

---

## 5. The setup dialog

### 5.1 Entry points

One server function backs both menu labels:

```js
function showSyncSetup() {
  var html = HtmlService.createHtmlOutput(SYNC_SETUP_HTML_)
    .setWidth(420).setHeight(560);
  SpreadsheetApp.getUi().showModalDialog(html, 'Live sync');
}
```

Built with `HtmlService.createHtmlOutput` on a string constant and
`google.script.run` for the round trip — the same construction
`sheet-generator.gs`'s `GENERATE_SIDEBAR_HTML_` uses. Follow that file's
conventions for the inline `<style>` and the string concatenation style.

### 5.2 Fields

| Field | Prefill | Notes |
| --- | --- | --- |
| Day key | current value | text |
| Facility name | current value | text; a note reads *matched exactly, including capitalisation* |
| Tabs to watch | §4's checkbox list | pre-ticked per §4 |
| Shared secret | **never prefilled** | hidden entirely when one is already stored, replaced by a *Secret stored* line and a **Replace secret** link that reveals the field |

The dialog loads current state through one server call:

```js
function getSyncSetupState() {
  var config = readSyncConfig_();
  return {
    dayKey: config ? config.dayKey : '',
    facilityName: config ? config.facilityName : '',
    hasSecret: !!readSharedSecret_(),
    tabs: listSheetTabs(),
    watchedGids: config ? config.watchedGids : [],
    defaultTabNames: [SCHEDULE_TAB_NAME, COURT_CONTROL_TAB_NAME]
  };
}
```

When `watchedGids` is empty, tick the tabs named in `defaultTabNames`;
otherwise tick whatever matches the stored GIDs.

A stored secret is never returned to the client, displayed, or logged — only
the `hasSecret` boolean crosses that boundary.

### 5.3 Save

```js
function saveSyncSetup(dayKey, facilityName, watchedTabNames, secretOrNull)
```

In order:

1. Trim `dayKey` and `facilityName`. Both required, and at least one tab —
   otherwise return blocking errors and write nothing.
2. Resolve the secret: `secretOrNull` when supplied, else the stored one. None
   available → blocking, `A shared secret is required.`
3. §3.1's `GET /sync/config`. Blocking failures return here, having written
   nothing.
4. §3.2's test sync. An `unknown facility` rejection returns here, having
   written nothing.
5. Write all properties, including `SYNC_BOUND_SPREADSHEET_ID` =
   `SpreadsheetApp.getActive().getId()`, and the secret when newly supplied.
6. Install the `onEditInstallable` trigger if not already present — the same
   check `setup()` performs today.
7. Return `{ ok, message, warning }` for the dialog to render.

**Nothing is written before step 5.** A rejected configuration must leave the
workbook exactly as it was, so a failed attempt cannot half-configure a
spreadsheet and leave a trigger firing against a wrong day key.

### 5.4 `setup()` stays

Keep the `setup()` function callable from the editor's function dropdown as a
fallback for a workbook whose menu has not loaded. It prompts for day key,
facility name and secret via `ui.prompt`, resolves the default tabs by name
without asking, and calls the same `saveSyncSetup` path so validation is
identical.

### 5.5 `Sync now`

`testSyncNow()` is kept and exposed as a menu item, enabled only when the
workbook is configured. It calls `triggerSync_()` and reports the outcome
through `ui.alert` rather than requiring the operator to read Executions.

---

## 6. The `onOpen` contract

`sheet-generator.gs` declares `onOpen`. A workbook generated from the Master
carries **both** scripts once §8 lands, so `sheets-sync.gs` declaring one too
puts two `onOpen` declarations in a single Apps Script project.

That is not an error. The declarations hoist, the last file loaded wins
silently, and load order is not something the operator controls.

**Both files therefore declare a byte-identical `onOpen` that delegates to
feature-detected builders.** Each file owns only its own items, and which
declaration wins cannot matter because both bodies are the same text:

```js
// This body is duplicated verbatim in sheets-sync.gs and sheet-generator.gs.
// A generated workbook carries both scripts; duplicate declarations don't
// error, the last file loaded wins, and load order isn't controllable. Keeping
// the two bodies identical makes that race irrelevant. Each file contributes
// only its own items, via the builder it defines. Change one, change both.
function onOpen() {
  var menu = SpreadsheetApp.getUi().createMenu('SAGE');
  var hasSync = (typeof addSyncMenuItems_ === 'function');
  var hasGenerator = (typeof addGeneratorMenuItems_ === 'function');
  if (hasSync) addSyncMenuItems_(menu);
  if (hasSync && hasGenerator) menu.addSeparator();
  if (hasGenerator) addGeneratorMenuItems_(menu);
  menu.addToUi();
}
```

In `sheets-sync.gs`:

```js
function addSyncMenuItems_(menu) {
  if (readSyncConfig_()) {
    menu.addItem('Sync now', 'testSyncNow');
    menu.addItem('Live sync settings', 'showSyncSetup');
  } else {
    menu.addItem('Set up live sync', 'showSyncSetup');
  }
}
```

In `sheet-generator.gs`:

```js
function addGeneratorMenuItems_(menu) {
  if (PropertiesService.getScriptProperties().getProperty(PROP_TABS_GENERATED_FOR)
      !== SpreadsheetApp.getActive().getId()) {
    menu.addItem('Generate event tabs', 'showGenerateSidebar');
  }
}
```

`onOpen` runs as a simple trigger without authorization, so both builders must
stay free of anything needing a scope. `PropertiesService` and
`SpreadsheetApp.getActive()` are both fine.

Resulting menus:

| Workbook | `SAGE` menu |
| --- | --- |
| Dual Meet Master | `Generate event tabs` |
| Fresh copy of the Master | `Set up live sync` · ─── · `Generate event tabs` |
| Generated, sync configured | `Sync now` · `Live sync settings` |
| Hand-built, sync configured | `Sync now` · `Live sync settings` |
| Hand-built, not yet configured | `Set up live sync` |

A configured workbook offers no destructive action. `Generate event tabs`
appears only where it can still legitimately run.

---

## 7. `sheet-generator.gs` — the generated marker

At the end of a successful `generateEventTabs`, record which spreadsheet was
built:

```js
PropertiesService.getScriptProperties()
  .setProperty(PROP_TABS_GENERATED_FOR, SpreadsheetApp.getActive().getId());
```

Written last, after every tab is built, so a run that fails partway leaves the
menu item in place.

This is menu hygiene, not a safety mechanism — `REBUILT_IN_PLACE_TABS`'
prototype-shape check already refuses a second run. The value is that a live
event workbook stops offering an action that can only produce an error.

Storing the spreadsheet ID rather than a boolean means an inherited property
from a copy does not suppress the item in a workbook that genuinely needs it —
the same mechanism as §2.2.

`generateEventTabs`'s own behaviour, output, and the assertions in
`scripts/verify-sheet-generator.mjs` are untouched. The mock supplies no
`PropertiesService`, so guard the write for the verify harness the way the file
guards other host-only calls, or extend `scripts/mock-apps-script.mjs` with a
`PropertiesService` stub — either is acceptable, the choice is whichever fits
the mock's existing shape.

---

## 8. Shipping the sync script in the Dual Meet Master

Once §2–§6 land, add `sheets-sync.gs` to the Dual Meet Master's script project
as a second file, and set `SYNC_SHARED_SECRET` once in the Master's
**Project Settings → Script properties**.

A copy of the Master then carries both scripts. The workflow is: copy the
Master, `SAGE → Generate event tabs`, `SAGE → Set up live sync`, and enter the
day key and facility name — the secret is already there if properties survive
the copy, and is one more field in the same dialog if not. No Apps Script editor
at any point.

Safe only because of §2.2: a copy inherits the secret and nothing that
identifies a workbook.

### 8.1 Exposure this creates

The secret sits in every facility workbook's Script Properties, so the Master is
one more location rather than a new kind of location. What it adds is that the
Master is the artifact that gets copied, so anyone able to copy it holds a
secret authorizing `POST /sync/:day` for **every registered day of every
event** — not only their own.

That is an acceptable trade for a team that controls access to the Master. It
stops being one if the Master is ever shared outside that team, at which point
the secret comes out of the Master and the dialog asks for it per workbook —
a change to §8 alone, with §2–§7 unaffected.

---

## 9. Rolling this out to existing workbooks

Per workbook: paste the new `sheets-sync.gs` over the old one, reload the
spreadsheet, then `SAGE → Set up live sync` and enter day key, facility name and
secret.

Until that setup runs, the workbook is unconfigured and syncs nothing — the
early exits in §2.4 make that a logged no-op rather than a misdirected publish.
Sync resumes at the end of the dialog, and §3.2's test sync confirms it before
the operator leaves.

Archived events need no migration.

---

## 10. Acceptance checklist

### Configuration and the guard

- [ ] A freshly pasted `sheets-sync.gs` in an unconfigured workbook shows
      `SAGE → Set up live sync`, and editing a watched tab logs the
      "not configured" message and schedules no trigger.
- [ ] After a successful setup, `SYNC_DAY_KEY`, `SYNC_FACILITY_NAME`,
      `SYNC_WATCHED_GIDS` and `SYNC_BOUND_SPREADSHEET_ID` are all present, and
      editing a watched tab syncs after the debounce.
- [ ] Re-pasting the whole file over a configured workbook and reloading leaves
      it configured and syncing, with nothing re-entered.
- [ ] Copying a configured workbook produces one whose menu reads
      `Set up live sync` — identity rejected by the guard — while a stored
      secret, if properties carry across the copy, is still recognised
      (`hasSecret` true, no secret field shown).
- [ ] The stored secret is never returned to the dialog, rendered, or logged.

### Validation

- [ ] A wrong secret blocks the save with the §3.1 message and writes nothing.
- [ ] An unregistered day key blocks the save and the error lists the real
      registered day keys.
- [ ] A facility name with wrong capitalisation blocks the save and the error
      names the valid facilities for that day.
- [ ] A valid configuration completes, reports the test sync's result, and the
      published snapshot for that day contains this facility.
- [ ] With Cloud Run unreachable, a save with plausible values still writes the
      config, installs the trigger, and reports the §3.3 warning.
- [ ] A save rejected at any blocking check leaves properties and triggers
      exactly as they were.

### Watched tabs

- [ ] In a generated workbook the dialog pre-ticks `SCHEDULE` and
      `Court Control` and needs no interaction.
- [ ] Renaming a watched tab after setup does not break syncing.
- [ ] A workbook with neither default tab ticks nothing, and saving with no tab
      ticked is blocked.

### Menu

- [ ] Each row of §6's menu table renders as described.
- [ ] In a workbook holding both scripts, the menu is correct **with the two
      files in either order** — verify both, since load order decides which
      `onOpen` wins.
- [ ] `Generate event tabs` disappears after a successful generation and
      reappears in a fresh copy of the Master.
- [ ] `Sync now` reports its outcome in a dialog.
- [ ] Opening a spreadsheet triggers no authorization prompt; clicking a menu
      item prompts on first use and succeeds after approval.

### Regression

- [ ] `node scripts/verify-sheet-generator.mjs` passes.
- [ ] `generateEventTabs` output is unchanged — same tabs, same formulas.
- [ ] The debounce still collapses a burst of edits into one sync.

---

## 11. Out of scope

- Publishing the sync script as an Apps Script **Library**. It is the way to
  stop updating N workbooks by hand, and is worth revisiting once this lands,
  but it puts a cross-project dependency in the live sync path, needs library
  access granted per editor, and still needs a per-workbook version bump when
  pinned.
- `clasp` or Apps Script API pushes.
- `CLOUD_RUN_BASE_URL` moving to properties — identical everywhere, not secret.
- The scoresheet event picker and its `SAGE → Generate Scoresheets` item
  (`scoresheet-event-picker-spec.md`). That item is added through
  `addSyncMenuItems_` when it ships; nothing else in it interacts with this.
- Any change to the sync protocol, endpoints, debounce behaviour, or
  `events.json`.
- A standard-tournament sheet generator.

---

## 12. Documentation changes

Present tense throughout: describe what the system does, not what changed.

### `sheets-sync.gs` header comment

Rewrite the install steps as:

1. Open the spreadsheet → Extensions → Apps Script.
2. Paste this file in.
3. Reload the spreadsheet and choose **SAGE → Set up live sync**.
4. Enter the day key and facility name, confirm the tabs to watch, and supply
   the shared secret if prompted.

State that the file is identical in every workbook and holds no
spreadsheet-specific values, that configuration lives in Script Properties, and
that setup validates the secret, the day key and the facility name against
Cloud Run before saving.

Remove the `WATCHED_SHEET_GIDS` step — GIDs are resolved by tab name.

### `sage-docs/docs/technical/sync-pipeline.md`

- Configuration lives in Script Properties, collected by `SAGE → Set up live
  sync`; the script file itself is identical across every workbook.
- Setup validates against `GET /sync/config` and a test sync, so a bad secret,
  an unregistered day key, or a mis-capitalised facility name is caught at
  setup.
- The spreadsheet-ID guard: identity values apply only to the spreadsheet they
  were saved for; the shared secret is not guarded, because it is the same
  everywhere.
- The shared `onOpen` contract (§6), and that both files must carry identical
  bodies.

### `sage-docs/docs/technical/dual-meet-sheet-generator.md`

- The Master carries `sheets-sync.gs` alongside the generator, so a copy is
  ready for both.
- The shared `onOpen` contract, from this file's side, with a pointer to
  `sync-pipeline.md`.
- `Generate event tabs` appears only in a workbook that has not been generated.

### `sage-docs/docs/technical/adding-a-new-event.md`

Step 8 becomes: reload the spreadsheet and run **SAGE → Set up live sync**,
entering the day key and facility name. Note that setup verifies both against
Cloud Run and performs a test sync, and that a dual meet's workbook carries the
script already.

### `sage-docs/docs/features/preparing-an-event.md`

- §2: a generated workbook is ready for sync setup without any code step.
- §4: wiring the sync is a menu item in each workbook; facility names are
  matched exactly and setup confirms the name against the registry.

### `sage-docs/docs/features/dual-meet-sheet-generator.md`

The `SAGE` menu carries both the generator and live-sync items. `Set up live
sync` is safe and repeatable; `Generate event tabs` runs once per workbook.

### `sage-match-control.github.io/_templates/CLAUDE.md`

§2 step 8: paste the file, reload, run `SAGE → Set up live sync`, enter day key
and facility name. Drop the GID-hunting and constant-editing instructions.

### `sage-docs/docs/specs/README.md`

List this spec under **Sync & live data**.

### `D:\Coding Projects\SAGE\CLAUDE.md`

Under "Things that must be kept in sync by hand", state that each workbook's day
key and facility name are set through `SAGE → Set up live sync` and stored in
Script Properties, and that setup validates them against the registry.

---

## 13. Divergences

*(Empty at time of writing — record here where the built feature departs from
this document.)*
