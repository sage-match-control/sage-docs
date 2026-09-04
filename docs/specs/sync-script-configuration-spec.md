# Spec — Live sync script configuration

Move `sheets-sync.gs`'s per-spreadsheet configuration out of source and into
Script Properties, collected through a menu-driven setup dialog that validates
what it is given before saving it. Ship the sync script inside the Dual Meet
Master so a generated workbook needs no code pasting at all.

Three files change:

| File | Change |
| --- | --- |
| `sage-tools-api/scripts/sheets-sync.gs` | config → Script Properties; setup dialog; validation; shared `onOpen`; `showScoresheetLink` reads config (§3–§8) |
| `sage-tools-api/scripts/sheet-generator.gs` | shared `onOpen` contract; post-generation marker (§7, §9) |
| `sage-tools-api/scripts/mock-apps-script.mjs` | a `PropertiesService` stub, so §9's marker write survives the verify harness |

Plus the documentation in §13, which is part of the work, not a follow-up.

**Neither `.gs` file is a deploy.** Both run inside a spreadsheet and ship by
being pasted into that workbook's script project. Do **not** bump
`package.json` and do **not** add a `README.md` changelog entry — nothing about
what Cloud Run runs is affected. The mock change is a test-harness file and is
likewise not a deploy.

Nothing in `src/` changes. No endpoint, no auth, no `events.json` schema change.

---

## 1. The failure mode this closes

`sheets-sync.gs` carries three per-spreadsheet values as source constants
(`sheets-sync.gs:52`, `:53`, `:64`):

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

## 2. Implementation order

Work in this order. Each step leaves the file in a syntactically valid state,
and §12's syntax check can be run after any of them.

1. **§3** — replace the CONFIG block with the property model, add
   `readSyncConfig_()` / `readSharedSecret_()`, and convert `triggerSync_()`
   and `onEditInstallable` to read from them.
2. **§8** — convert `showScoresheetLink` to read from `readSyncConfig_()`.
   Do this immediately after §3: it is the other existing reader of the two
   constants being deleted, and leaving it until later means the file
   references identifiers that no longer exist.
3. **§5** — `listSheetTabs` and `resolveWatchedGids_`.
4. **§4** + **§6** — the validation calls and the dialog that drives them.
   `saveSyncSetup` is the join between the two; write §4's helpers first.
5. **§7** — replace *both* files' existing `onOpen` with the shared body, and
   add the two builders.
6. **§9** — `sheet-generator.gs`'s marker and the mock's `PropertiesService`.
7. **§12** — run the syntax check and the verify harness.
8. **§13** — the documentation. Not optional and not a separate task.

### 2.1 Code style

Match each file's existing conventions:

- `sheets-sync.gs` uses `const`, arrow functions and template literals.
- `sheet-generator.gs` uses `var` and `function` declarations exclusively, and
  string concatenation rather than template literals.

**Where a declaration must exist in both files it must be `var`.** Apps Script
loads every `.gs` in a project into one global lexical scope: two top-level
`const` declarations of the same name are a project-wide
`Identifier has already been declared` error that breaks *both* features, while
duplicate `var` declarations and duplicate `function` declarations are both
legal. This applies to `PROP_TABS_GENERATED_FOR` (§7, §9) and to the `onOpen`
body itself.

There are no top-level name collisions between the two files today. Before
adding any new top-level name to `sheets-sync.gs`, check it against
`sheet-generator.gs`'s `var` declarations — in particular, that file already
owns `SCHEDULE_SHEET_NAME` and `COURT_CONTROL_SHEET_NAME`, which is why §3.1's
new constants are named `*_TAB_NAME`.

---

## 3. Property model

All configuration lives in `PropertiesService.getScriptProperties()`.

| Key | Holds | Copy-inheritable |
| --- | --- | --- |
| `SYNC_SHARED_SECRET` | the shared secret | **Yes** |
| `SYNC_DAY_KEY` | this workbook's day key | No |
| `SYNC_FACILITY_NAME` | this workbook's facility name | No |
| `SYNC_WATCHED_GIDS` | JSON array of GID strings | No |
| `SYNC_BOUND_SPREADSHEET_ID` | the spreadsheet these identity values were saved for | — |
| `TABS_GENERATED_FOR` | the spreadsheet `generateEventTabs` ran against | — |

`SYNC_SHARED_SECRET` is the key `sheets-sync.gs` already uses today, under the
constant name `SCRIPT_PROP_SECRET_KEY`. **The stored key name does not change**,
so an existing workbook's secret survives the paste — only the constant
identifier in source is renamed for consistency with the rest of the block.

`PROP_LAST_EDIT` (`lastEditTime`) is unrelated debounce bookkeeping and is left
exactly as it is.

### 3.1 The constant block

Replacing `sheets-sync.gs:51-72` in full — the `CONFIG` comment banner at
`:51` through the `SCORESHEET_GENERATOR_URL` declaration ending at `:72`:

```js
// ---- Property keys — where per-workbook configuration actually lives ----
// Nothing in this file is spreadsheet-specific: the file is byte-identical in
// every workbook, and everything that identifies THIS workbook is stored in
// Script Properties by SAGE -> Set up live sync. See §1 of
// sage-docs/docs/specs/sync-script-configuration-spec.md for why.
const PROP_SHARED_SECRET = 'SYNC_SHARED_SECRET';
const PROP_DAY_KEY = 'SYNC_DAY_KEY';
const PROP_FACILITY_NAME = 'SYNC_FACILITY_NAME';
const PROP_WATCHED_GIDS = 'SYNC_WATCHED_GIDS';
const PROP_BOUND_SPREADSHEET_ID = 'SYNC_BOUND_SPREADSHEET_ID';
const PROP_LAST_EDIT = 'lastEditTime';

// Declared with `var`, not `const`: sheet-generator.gs declares this same name,
// and a generated workbook carries both files in one Apps Script project, where
// duplicate top-level `const` is a project-breaking error and duplicate `var`
// is not. Keep the two declarations identical.
var PROP_TABS_GENERATED_FOR = 'TABS_GENERATED_FOR';

// ---- Platform constants — identical in every workbook, none of them secret ----
const CLOUD_RUN_BASE_URL = 'https://sage-tools-api-811926984834.us-central1.run.app'; // no trailing slash
const SCORESHEET_GENERATOR_URL =
  'https://sage-match-control.github.io/tools/scoresheet-generator.html';
const DEBOUNCE_MS = 10000; // wait for ~10s of quiet before syncing (best-effort)
const SETTLE_HANDLER = 'runIfSettled';

// Tabs pre-ticked in the setup dialog when they exist. Not a watch list — the
// operator's ticks are, and they are stored as GIDs (§5).
const SCHEDULE_TAB_NAME = 'SCHEDULE';
const COURT_CONTROL_TAB_NAME = 'Court Control';
```

> **`Court Control`, not `COURT CONTROL`.** `sheet-generator.gs:229` creates the
> tab as `Court Control`. Name-based lookup must use that exact string. The
> `// "SCHEDULE", "COURT CONTROL"` comment on today's `WATCHED_SHEET_GIDS` line
> is wrong about the second name and goes away with the line.

### 3.2 The secret is shared; identity is not

The secret is one Cloud Run environment variable, identical for every workbook
of every event, so a copy inheriting it is correct and saves re-entering an
opaque string per workbook.

The day key and facility name identify *this* workbook. A copy inheriting them
is precisely the §1 failure in a new form — so they are guarded.

### 3.3 The spreadsheet-ID guard

Identity values are only honoured when they were saved for *this* spreadsheet,
and only when they are complete:

```js
/**
 * This workbook's sync identity, or null if it has none. Null means one of:
 * never set up, or set up in a DIFFERENT spreadsheet that this one was copied
 * from. Both are "unconfigured" and both must behave identically — a copied
 * workbook inheriting a day key would publish its scores over another
 * facility's snapshot.
 */
function readSyncConfig_() {
  const props = PropertiesService.getScriptProperties();
  if (props.getProperty(PROP_BOUND_SPREADSHEET_ID) !== SpreadsheetApp.getActive().getId()) {
    return null;
  }
  const dayKey = props.getProperty(PROP_DAY_KEY);
  const facilityName = props.getProperty(PROP_FACILITY_NAME);
  if (!dayKey || !facilityName) return null;

  let watchedGids = [];
  try {
    watchedGids = JSON.parse(props.getProperty(PROP_WATCHED_GIDS) || '[]');
  } catch (e) {
    watchedGids = [];
  }
  return { dayKey, facilityName, watchedGids };
}
```

A non-null return therefore always carries a usable `dayKey` and
`facilityName`, which is what lets every caller in §3.4, §7 and §8 treat
`readSyncConfig_()` as a single truthiness check.

The secret is read separately and is **never** subject to this guard:

```js
function readSharedSecret_() {
  return PropertiesService.getScriptProperties().getProperty(PROP_SHARED_SECRET);
}
```

This is what makes §10 safe. Whether Google carries Script Properties across a
spreadsheet copy is a platform behaviour this spec deliberately does not depend
on: if properties copy, the secret is inherited and identity is rejected by the
guard; if they do not, everything is simply unset and the dialog asks for all of
it. Both paths are correct.

**Verify the behaviour anyway before implementing §10** — set a property on the
Master, copy it, read the property back in the copy. The result decides only
whether the dialog's secret field is usually pre-satisfied (§6.2), not whether
the design holds.

### 3.4 Reading config at sync time

`triggerSync_()` reads from `readSyncConfig_()` and `readSharedSecret_()`
instead of module constants, and returns without acting when either is missing:

```js
function triggerSync_() {
  const config = readSyncConfig_();
  if (!config) {
    console.error('Live sync is not configured for this spreadsheet. Run SAGE -> Set up live sync.');
    return;
  }
  const secret = readSharedSecret_();
  if (!secret) {
    console.error('No shared secret stored. Run SAGE -> Set up live sync.');
    return;
  }
  // ...unchanged from here: URL build, UrlFetchApp.fetch, status logging...
}
```

Everything below that point is untouched, with `config.dayKey` and
`config.facilityName` substituted for `DAY_KEY` and `FACILITY_NAME` — the
`fetchTimeoutSeconds: 30` and `muteHttpExceptions: true` options and their
comments stay exactly as they are.

`onEditInstallable` gains the same early exit, and takes its watch list from the
config rather than a constant. The order matters: an unconfigured workbook must
return **before** touching `PROP_LAST_EDIT` or scheduling a trigger, so it does
nothing at all rather than scheduling work that cannot succeed.

```js
function onEditInstallable(e) {
  const config = readSyncConfig_();
  if (!config) return; // unconfigured — do nothing at all, not even schedule

  const editedGid = e && e.range ? String(e.range.getSheet().getSheetId()) : null;
  if (!editedGid || !config.watchedGids.includes(editedGid)) {
    return; // edit was on an unrelated tab (or e is missing/unexpected) — ignore
  }
  // ...unchanged: PROP_LAST_EDIT write, trigger cleanup, newTrigger...
}
```

---

## 4. Validation

Setup validates before it saves. Two of the three checks are blocking.

### 4.1 Secret and day key — `GET /sync/config`

```
GET {CLOUD_RUN_BASE_URL}/sync/config
X-Sync-Secret: <entered secret>
```

Handled by `handleConfigDiagnostics` in `sage-tools-api/src/sync/routes.mjs`,
behind `requireSyncSecretOrAuthToken`. Returns
`{ sha, source, loadedAt, ageMs, events: [...], days: [...] }`, where `days` is
`SyncConfigSnapshot.knownDays()` — every day key across every registered event.

- **401** → the secret is wrong. Blocking. `The shared secret was rejected.
  Check it against Cloud Run's SYNC_SHARED_SECRET.`
- **200, and `days` excludes the entered day key** → blocking. Report it with
  the real list, which is exactly what the operator needs to pick from:
  `Day key "<entered>" is not registered. Registered day keys: <days joined>.`
- **200 with the day key present** → proceed.
- **Network failure / 5xx** → non-blocking (§4.3).

### 4.2 Facility name — a real test sync

```
POST {CLOUD_RUN_BASE_URL}/sync/<dayKey>?facility=<encoded facilityName>
X-Sync-Secret: <secret>
```

`SyncService.syncDay` (`src/sync/SyncService.mjs:55-60`) rejects an unknown
facility with a `ValidationError` — **HTTP 400** — whose message names the valid
ones:

```
<label>: unknown facility "X". Known facilities: Main, Annex, Dreamcourts
```

The response body is `{ "error": "<that message>" }`.

- **Non-2xx whose body contains `unknown facility`** → blocking. Surface the
  message verbatim; it already names the alternatives. Match on the substring
  rather than on the status code, so an unrelated 400 still falls through to
  §4.3 rather than being reported as a bad facility name.
- **2xx** → success. Report it, including the response body's counts.
- **Any other non-2xx, or a network failure** → non-blocking (§4.3).

Facility names are compared exactly and case-sensitively by the sync itself.
Do not trim or case-fold the entered value to be helpful — that would hide the
mismatch this check exists to catch. (Leading/trailing whitespace *is* trimmed
at §6.3 step 1, before the value is ever compared; that is a paste artefact, not
a name.)

An unregistered day key would be rejected here too, as an `UnknownSyncDayError`
— but §4.1 has already blocked that case, so this check never sees it.

This publishes a real snapshot. That is intended: it is the end-to-end proof the
workbook is wired up. Running it before rosters are pasted publishes a snapshot
with empty player names, which the next edit replaces — harmless, and worth a
line in the dialog's success message so it is not mistaken for a fault.

### 4.3 Non-blocking failures

A timeout, a 5xx, or an unreachable host means the configuration may be perfectly
correct and Cloud Run is simply unavailable. Save the configuration, install the
trigger, and report a warning rather than an error:

```
Saved, but the test sync could not reach Cloud Run (<reason>).
Use SAGE -> Sync now to retry once it is reachable.
```

Both calls use `muteHttpExceptions: true` so a non-2xx returns a response rather
than throwing, and both are wrapped in `try`/`catch` so a genuine transport
failure becomes a §4.3 warning rather than an unhandled exception in the dialog.

Only a **rejected secret**, an **unregistered day key**, and an **unknown
facility** block the save. Each of those is a value that is definitively wrong.

---

## 5. Resolving watched GIDs

The watched tabs are resolved by name at setup and their GIDs stored. Matching
on GID at edit time is retained, so renaming a tab afterwards does not break the
trigger.

```js
function resolveWatchedGids_(selectedNames) {
  const ss = SpreadsheetApp.getActive();
  const gids = [];
  selectedNames.forEach(name => {
    const sheet = ss.getSheetByName(name);
    if (sheet) gids.push(String(sheet.getSheetId()));
  });
  return gids;
}
```

The dialog presents every tab in the workbook as a checkbox list, pre-ticking
`SCHEDULE` and `Court Control` where they exist. A generated workbook needs no
interaction here; a hand-built one whose tabs are named differently is a matter
of ticking the right boxes rather than reading GIDs out of tab URLs.

At least one tab must resolve — blocking, with
`Pick at least one tab to watch for score edits.` Check this on the resolved
GIDs, not on the submitted names, so a name that no longer matches a tab is
caught rather than saved as an empty watch list.

Provide the tab list to the dialog from a server function:

```js
function listSheetTabs() {
  return SpreadsheetApp.getActive().getSheets().map(s => ({
    name: s.getName(),
    gid: String(s.getSheetId())
  }));
}
```

---

## 6. The setup dialog

### 6.1 Entry point

One server function backs both menu labels:

```js
function showSyncSetup() {
  const html = HtmlService.createHtmlOutput(SYNC_SETUP_HTML_)
    .setWidth(420).setHeight(560);
  SpreadsheetApp.getUi().showModalDialog(html, 'Live sync');
}
```

Built with `HtmlService.createHtmlOutput` on a string constant and
`google.script.run` for the round trip — the same construction
`sheet-generator.gs`'s `GENERATE_SIDEBAR_HTML_` uses (`sheet-generator.gs:740`).
Follow that constant for the inline `<style>` and the string-concatenation
style, and `showScoresheetLink`'s smaller dialog (`sheets-sync.gs:245`) for the
colour palette.

`getSyncSetupState`, `listSheetTabs` and `saveSyncSetup` are called from the
dialog through `google.script.run` and therefore must **not** be given
trailing-underscore names. Everything else new in this file is private and takes
the underscore.

### 6.2 Fields

| Field | Prefill | Notes |
| --- | --- | --- |
| Day key | current value | text |
| Facility name | current value | text; a note reads *matched exactly, including capitalisation* |
| Tabs to watch | §5's checkbox list | pre-ticked per §5 |
| Shared secret | **never prefilled** | hidden entirely when one is already stored, replaced by a *Secret stored* line and a **Replace secret** link that reveals the field |

The dialog loads current state through one server call:

```js
function getSyncSetupState() {
  const config = readSyncConfig_();
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

### 6.3 Save

```js
function saveSyncSetup(dayKey, facilityName, watchedTabNames, secretOrNull)
```

In order:

1. Trim `dayKey` and `facilityName`. Both required, and §5's resolved-GID check
   must pass — otherwise return a blocking result and write nothing.
2. Resolve the secret: `secretOrNull` when supplied, else the stored one. None
   available → blocking, `A shared secret is required.`
3. §4.1's `GET /sync/config`. Blocking failures return here, having written
   nothing.
4. §4.2's test sync. An `unknown facility` rejection returns here, having
   written nothing.
5. Write all properties, including `SYNC_BOUND_SPREADSHEET_ID` =
   `SpreadsheetApp.getActive().getId()`, and the secret when newly supplied.
6. Install the `onEditInstallable` trigger if not already present — the same
   `ScriptApp.getProjectTriggers()` check `setup()` performs today.
7. Return the result object below.

**Nothing is written before step 5.** A rejected configuration must leave the
workbook exactly as it was, so a failed attempt cannot half-configure a
spreadsheet and leave a trigger firing against a wrong day key.

The return shape, in every case:

```js
{ ok: true,  message: '<what happened>', warning: '<§4.3 text>' | null }
{ ok: false, error: '<the one blocking reason>' }
```

Blocking failures are **returned, not thrown**. `google.script.run`'s failure
handler is reserved for genuine exceptions; a wrong secret is an expected
outcome and belongs in the success path so the dialog can render it inline
without the operator losing what they typed.

### 6.4 Client behaviour

The dialog's own script:

- On load, calls `getSyncSetupState()` and renders the fields per §6.2. Until it
  returns, show a *Loading…* line and keep **Save** disabled.
- **Save** disables itself and both text inputs, shows a *Checking…* line, and
  calls `saveSyncSetup(...)` with the ticked tabs' **names**, and the secret
  field's value or `null` when the field is hidden.
- `withSuccessHandler`: on `ok: false`, render `error` in the error style,
  re-enable the form, and leave every field as the operator left it. On
  `ok: true`, render `message` (plus `warning` in the warning style, when
  present) and leave the dialog open — the operator closes it. Do not auto-close
  on success; the test-sync result in `message` is the point of the exercise.
- `withFailureHandler`: render the exception message in the error style and
  re-enable the form.

### 6.5 `setup()` stays

Keep the `setup()` function callable from the editor's function dropdown as a
fallback for a workbook whose menu has not loaded. It prompts for day key,
facility name and secret via `ui.prompt`, resolves the default tabs by name
without asking, and calls the same `saveSyncSetup` path so validation is
identical. It reports the returned `error` or `message` through `ui.alert`.

Its existing "already set — leaving it as-is" branch for the secret is
subsumed by §6.3 step 2: prompt for the secret only when
`readSharedSecret_()` returns nothing.

### 6.6 `Sync now`

`testSyncNow()` is kept and exposed as a menu item, enabled only when the
workbook is configured. It calls `triggerSync_()` and reports the outcome
through `ui.alert` rather than requiring the operator to read Executions. To do
that, `triggerSync_()` returns a `{ ok, status, body }` object (or `null` when
it exited early as unconfigured) in addition to its existing logging; the
`runIfSettled` caller ignores the return value.

---

## 7. The `SAGE` menu and the `onOpen` contract

### 7.1 Current state

**Both files already declare `onOpen`**, at `sheets-sync.gs:228` and
`sheet-generator.gs:724`, added when the scoresheet event picker shipped. They
are not identical: each builds the whole menu while feature-detecting the
other's handler function by name (`showGenerateSidebar` / `showScoresheetLink`).
That works, but it makes every menu change a two-file edit with no structural
guarantee the two agree, and it cannot express "this item is not available right
now" without both files knowing why.

This section **replaces both**.

### 7.2 The contract

A generated workbook carries both scripts in one Apps Script project. Two
`onOpen` declarations there is not an error — the declarations hoist, the last
file loaded wins silently, and load order is not something the operator
controls.

**Both files therefore declare a byte-identical `onOpen` that delegates to
feature-detected builders.** Each file owns only its own items, and which
declaration wins cannot matter because both bodies are the same text:

```js
// ============================================================================
// MENU
// ============================================================================
// This block is duplicated VERBATIM in sheets-sync.gs and sheet-generator.gs.
// A generated workbook carries both scripts; duplicate declarations don't
// error, the last file loaded wins, and load order isn't controllable. Keeping
// the two bodies identical makes that race irrelevant. Each file contributes
// only its own items, via the builder it defines, and each builder adds its own
// leading separator only if something is already above it — so a workbook where
// one builder contributes nothing gets no stray divider.
// Change one, change both.
function onOpen() {
  var menu = SpreadsheetApp.getUi().createMenu('SAGE');
  var state = { count: 0 };
  if (typeof addSyncMenuItems_ === 'function') addSyncMenuItems_(menu, state);
  if (typeof addGeneratorMenuItems_ === 'function') addGeneratorMenuItems_(menu, state);
  menu.addToUi();
}
```

`state.count` is the number of items added so far. A builder that is about to
add its first item checks `state.count > 0` and calls `menu.addSeparator()`
first, then increments `state.count` by however many items it added. A builder
that adds nothing must leave `state` untouched.

This is the fix for the case the old two-body arrangement could not express: in
a generated, configured workbook the generator contributes **no** items (§9's
marker), and a fixed `menu.addSeparator()` between the two halves would leave a
divider dangling at the bottom of the menu.

In `sheets-sync.gs`:

```js
function addSyncMenuItems_(menu, state) {
  if (state.count > 0) menu.addSeparator();
  if (readSyncConfig_()) {
    menu.addItem('Generate Scoresheets', 'showScoresheetLink');
    menu.addItem('Sync now', 'testSyncNow');
    menu.addItem('Live sync settings', 'showSyncSetup');
    state.count += 3;
  } else {
    menu.addItem('Set up live sync', 'showSyncSetup');
    state.count += 1;
  }
}
```

`Generate Scoresheets` is inside the configured branch because its deep link is
built from the day key and facility name (§8). An unconfigured workbook has
neither, so the item would only ever produce a broken link; the one thing that
workbook offers is `Set up live sync`.

In `sheet-generator.gs`:

```js
function addGeneratorMenuItems_(menu, state) {
  if (PropertiesService.getScriptProperties().getProperty(PROP_TABS_GENERATED_FOR)
      === SpreadsheetApp.getActive().getId()) {
    return; // already generated — the item can only error from here
  }
  if (state.count > 0) menu.addSeparator();
  menu.addItem('Generate event tabs', 'showGenerateSidebar');
  state.count += 1;
}
```

`onOpen` runs as a simple trigger without authorization, so both builders must
stay free of anything needing a scope. `PropertiesService` and
`SpreadsheetApp.getActive()` are both fine.

### 7.3 Resulting menus

| Workbook | `SAGE` menu |
| --- | --- |
| Dual Meet Master | `Generate event tabs` |
| Fresh copy of the Master | `Set up live sync` · ─── · `Generate event tabs` |
| Generated, sync configured | `Generate Scoresheets` · `Sync now` · `Live sync settings` |
| Generated, sync not yet configured | `Set up live sync` |
| Hand-built, sync configured | `Generate Scoresheets` · `Sync now` · `Live sync settings` |
| Hand-built, not yet configured | `Set up live sync` |

A configured workbook offers no destructive action. `Generate event tabs`
appears only where it can still legitimately run.

---

## 8. `showScoresheetLink` reads its config

The scoresheet event picker shipped after this spec was first written, and
`showScoresheetLink` (`sheets-sync.gs:245`) reads `DAY_KEY` and `FACILITY_NAME`
— the two constants §3 deletes. `scoresheet-event-picker-spec.md` §8.5
anticipated this and scoped it to exactly one change: the values come from
`readSyncConfig_()` instead.

```js
function showScoresheetLink() {
  const config = readSyncConfig_();
  if (!config) {
    SpreadsheetApp.getUi().alert(
      'This workbook has no day or venue configured yet. Run SAGE -> Set up live sync first.'
    );
    return;
  }

  const url = SCORESHEET_GENERATOR_URL +
    '?day=' + encodeURIComponent(config.dayKey) +
    '&facility=' + encodeURIComponent(config.facilityName);
  // ...unchanged from here: the same dialog HTML, with config.dayKey and
  // config.facilityName substituted into the <dl> readout...
}
```

The guard is unreachable from the menu — §7.2 only offers the item when
`readSyncConfig_()` is non-null — but the function is still callable from the
editor's function dropdown, and the alert is cheaper than an opaque failure.

**The `<dl>` readout stays.** `scoresheet-event-picker-spec.md` §8.4 is explicit
that showing the day and venue before the click is the feature, not decoration:
it is where a misconfigured workbook gets caught. Moving the values into Script
Properties does not weaken that — it is now the only place the operator sees
them outside the setup dialog.

Both values are interpolated into HTML. As source constants they were not a live
injection path; as operator-entered property values they are closer to one. They
already reach the `href` through `encodeURIComponent`; escape them in the `<dl>`
readout too rather than relying on a day key never containing a `<`.

---

## 9. `sheet-generator.gs` — the generated marker

At the end of a successful `generateEventTabs`, record which spreadsheet was
built. Place the write immediately before the `return { message: ... }` at
`sheet-generator.gs:1135`, after `logStep_('Done.')` at `:1133`:

```js
PropertiesService.getScriptProperties()
  .setProperty(PROP_TABS_GENERATED_FOR, SpreadsheetApp.getActive().getId());
```

Written last, after every tab is built, so a run that fails partway leaves the
menu item in place.

`PROP_TABS_GENERATED_FOR` is declared in this file as well, with `var` and the
identical text §3.1 uses — see §2.1 for why it cannot be `const`. Put it beside
the other tab-name `var`s near `sheet-generator.gs:229`.

This is menu hygiene, not a safety mechanism — `REBUILT_IN_PLACE_TABS`'
prototype-shape check already refuses a second run. The value is that a live
event workbook stops offering an action that can only produce an error.

Storing the spreadsheet ID rather than a boolean means an inherited property
from a copy does not suppress the item in a workbook that genuinely needs it —
the same mechanism as §3.3.

### 9.1 The verify harness

`scripts/mock-apps-script.mjs`'s `buildSandbox` supplies no `PropertiesService`,
so the write above throws `PropertiesService is not defined` under
`scripts/verify-sheet-generator.mjs` and every scenario fails.

Add a stub to the object `buildSandbox` returns, alongside the existing
`CacheService` entry and backed by a `Map` the same way `cacheStore` is:

```js
PropertiesService: { getScriptProperties: () => scriptProperties },
```

with `getProperty` / `setProperty` / `deleteProperty` over that map. One store
per sandbox, so the rerun-refusal scenario — which calls `generateEventTabs`
twice on one sandbox — sees the marker persist across the two runs, exactly as a
real workbook would.

`generateEventTabs`'s own behaviour, output, and the assertions in
`verify-sheet-generator.mjs` are untouched; no fixture changes and no new
assertion is required. Guarding the write with a `typeof PropertiesService`
check instead of extending the mock is **not** the approach here — it would put
a test-shaped conditional in production code and leave the mock unable to
express the marker at all.

The mock's `createMenu` returns a menu object with only `addItem` and
`addToUi`, no `addSeparator`. That is fine: the harness never calls `onOpen`.
Do not add builder calls to the harness.

---

## 10. Shipping the sync script in the Dual Meet Master

Once §3–§9 land, add `sheets-sync.gs` to the Dual Meet Master's script project
as a second file, and set `SYNC_SHARED_SECRET` once in the Master's
**Project Settings → Script properties**.

A copy of the Master then carries both scripts. The workflow is: copy the
Master, `SAGE → Generate event tabs`, `SAGE → Set up live sync`, and enter the
day key and facility name — the secret is already there if properties survive
the copy, and is one more field in the same dialog if not. No Apps Script editor
at any point.

Safe only because of §3.3: a copy inherits the secret and nothing that
identifies a workbook.

This step is done by hand in Google, not by editing anything in this repo. It is
the one part of this spec that cannot be completed in code.

### 10.1 Exposure this creates

The secret sits in every facility workbook's Script Properties, so the Master is
one more location rather than a new kind of location. What it adds is that the
Master is the artifact that gets copied, so anyone able to copy it holds a
secret authorizing `POST /sync/:day` for **every registered day of every
event** — not only their own.

That is an acceptable trade for a team that controls access to the Master. It
stops being one if the Master is ever shared outside that team, at which point
the secret comes out of the Master and the dialog asks for it per workbook —
a change to §10 alone, with §3–§9 unaffected.

---

## 11. Rolling this out to existing workbooks

Per workbook: paste the new `sheets-sync.gs` over the old one, reload the
spreadsheet, then `SAGE → Set up live sync` and enter day key and facility name.
The secret is already stored under the same property key (§3) and is not
re-entered.

Until that setup runs, the workbook is unconfigured and syncs nothing — the
early exits in §3.4 make that a logged no-op rather than a misdirected publish.
Sync resumes at the end of the dialog, and §4.2's test sync confirms it before
the operator leaves.

Archived events need no migration.

---

## 12. Verification

### 12.1 What can be checked from this repo

Two commands, both from `sage-tools-api/`. Run both before considering the code
half done.

`node --check` cannot read a `.gs` file — it rejects the extension — so parse it
through `vm` instead:

```bash
node -e "const vm=require('vm'),fs=require('fs');for(const f of ['scripts/sheets-sync.gs','scripts/sheet-generator.gs'])new vm.Script(fs.readFileSync(f,'utf8'),{filename:f});console.log('syntax OK')"
```

```bash
node scripts/verify-sheet-generator.mjs
```

The verify harness must print `ALL CHECKS PASSED`, unchanged from before this
work. It exercises `sheet-generator.gs` only; nothing in `sheets-sync.gs` is
covered by any automated check.

Also confirm by inspection, since no tool checks them:

- The `onOpen` block (§7.2) is **byte-identical** in both files — diff the two
  regions, do not eyeball them.
- No top-level `const`/`let` name in `sheets-sync.gs` matches a top-level `var`
  name in `sheet-generator.gs` (§2.1).
- `PROP_TABS_GENERATED_FOR` is declared `var` in both files, with the same
  value.
- No identifier named `DAY_KEY`, `FACILITY_NAME` or `WATCHED_SHEET_GIDS` remains
  anywhere in `sheets-sync.gs`.

### 12.2 What cannot

Everything below runs inside Google and needs a real spreadsheet, a real
secret, and Cloud Run reachable. It is the operator's checklist, to be handed
over with the change — not something to claim as verified from here.

**Configuration and the guard**

- [ ] A freshly pasted `sheets-sync.gs` in an unconfigured workbook shows
      `SAGE → Set up live sync`, and editing a watched tab logs the
      "not configured" message and schedules no trigger.
- [ ] After a successful setup, `SYNC_DAY_KEY`, `SYNC_FACILITY_NAME`,
      `SYNC_WATCHED_GIDS` and `SYNC_BOUND_SPREADSHEET_ID` are all present, and
      editing a watched tab syncs after the debounce.
- [ ] Re-pasting the whole file over a configured workbook and reloading leaves
      it configured and syncing, with nothing re-entered.
- [ ] An existing pre-change workbook, after the paste, still has its secret
      recognised (`hasSecret` true) and asks only for day key and facility name.
- [ ] Copying a configured workbook produces one whose menu reads
      `Set up live sync` — identity rejected by the guard — while a stored
      secret, if properties carry across the copy, is still recognised.
- [ ] The stored secret is never returned to the dialog, rendered, or logged.

**Validation**

- [ ] A wrong secret blocks the save with the §4.1 message and writes nothing.
- [ ] An unregistered day key blocks the save and the error lists the real
      registered day keys.
- [ ] A facility name with wrong capitalisation blocks the save and the error
      names the valid facilities for that day.
- [ ] A valid configuration completes, reports the test sync's result, and the
      published snapshot for that day contains this facility.
- [ ] With Cloud Run unreachable, a save with plausible values still writes the
      config, installs the trigger, and reports the §4.3 warning.
- [ ] A save rejected at any blocking check leaves properties and triggers
      exactly as they were, and the dialog keeps what was typed.

**Watched tabs**

- [ ] In a generated workbook the dialog pre-ticks `SCHEDULE` and
      `Court Control` and needs no interaction.
- [ ] Renaming a watched tab after setup does not break syncing.
- [ ] A workbook with neither default tab ticks nothing, and saving with no tab
      ticked is blocked.

**Menu**

- [ ] Each row of §7.3's menu table renders as described, with no stray
      separator in the generated-and-configured case.
- [ ] In a workbook holding both scripts, the menu is correct **with the two
      files in either order** — verify both, since load order decides which
      `onOpen` wins.
- [ ] `Generate event tabs` disappears after a successful generation and
      reappears in a fresh copy of the Master.
- [ ] `Generate Scoresheets` opens the generator with this workbook's day and
      venue preselected, and still shows both in the dialog before the click.
- [ ] `Sync now` reports its outcome in a dialog.
- [ ] Opening a spreadsheet triggers no authorization prompt; clicking a menu
      item prompts on first use and succeeds after approval.

**Regression**

- [ ] `generateEventTabs` output is unchanged — same tabs, same formulas.
- [ ] The debounce still collapses a burst of edits into one sync.

---

## 13. Documentation changes

Part of this work, not a follow-up: the paste-and-edit-constants procedure is
written down in seven places, and every one of them is wrong the moment this
ships. Present tense throughout — describe what the system does, not what
changed.

### 13.1 `sheets-sync.gs`'s own header comment

The header (`sheets-sync.gs:1-49`) is the install runbook. Rewrite steps 1–6 as:

1. Open the spreadsheet → Extensions → Apps Script.
2. Paste this file in.
3. Reload the spreadsheet and choose **SAGE → Set up live sync**.
4. Enter the day key and facility name, confirm the tabs to watch, and supply
   the shared secret if prompted.

State that the file is identical in every workbook and holds no
spreadsheet-specific values, that configuration lives in Script Properties, and
that setup validates the secret, the day key and the facility name against
Cloud Run before saving.

Remove the `WATCHED_SHEET_GIDS` step — GIDs are resolved by tab name. Keep the
`IMPORTANT:` paragraph about installable vs simple triggers as is; it is still
true, with the setup dialog rather than `setup()` as the thing that installs it.

### 13.2 `sage-docs/docs/technical/sync-pipeline.md`

The Apps Script section (from `sync-pipeline.md:41`) and the `onOpen` paragraph
at `:48-56`:

- Configuration lives in Script Properties, collected by `SAGE → Set up live
  sync`; the script file itself is identical across every workbook.
- Setup validates against `GET /sync/config` and a test sync, so a bad secret,
  an unregistered day key, or a mis-capitalised facility name is caught at
  setup.
- The spreadsheet-ID guard: identity values apply only to the spreadsheet they
  were saved for; the shared secret is not guarded, because it is the same
  everywhere.
- The `onOpen` contract (§7): identical bodies delegating to feature-detected
  `addSyncMenuItems_` / `addGeneratorMenuItems_` builders, and that both files
  must carry the same body. This **replaces** the current description of two
  bodies that merely build the same menu.

### 13.3 `sage-docs/docs/technical/dual-meet-sheet-generator.md`

Rewrite "The `onOpen` it shares with `sheets-sync.gs`" (`:167-177`) for §7's
builder contract, and add:

- The Master carries `sheets-sync.gs` alongside the generator, so a copy is
  ready for both.
- `Generate event tabs` appears only in a workbook that has not been generated,
  via the `TABS_GENERATED_FOR` marker.
- A pointer to `sync-pipeline.md` for the sync half.

### 13.4 `sage-docs/docs/technical/adding-a-new-event.md`

Step 8 (`:53-54`) becomes: reload the spreadsheet and run **SAGE → Set up live
sync**, entering the day key and facility name. Note that setup verifies both
against Cloud Run and performs a test sync, and that a dual meet's workbook
carries the script already.

The "Things kept in sync by hand" entry at `:103` keeps the same three-way
relationship but names Script Properties rather than constants in the file.

### 13.5 `sage-docs/docs/features/preparing-an-event.md`

- §2 (`:56`): a generated workbook is ready for sync setup without any code
  step.
- §4 (`:96-109`): wiring the sync is a menu item in each workbook; facility
  names are matched exactly and setup confirms the name against the registry
  before saving.

### 13.6 `sage-docs/docs/features/dual-meet-sheet-generator.md`

At `:62`: the `SAGE` menu carries both the generator and the live-sync items.
`Set up live sync` is safe and repeatable; `Generate event tabs` runs once per
workbook and then disappears.

### 13.7 `sage-match-control.github.io/_templates/CLAUDE.md`

§2 step 8 (`:253-294`) is the longest version of the old procedure — sub-steps
1–6 plus the `testSyncNow` and `Generate Scoresheets` notes. Replace sub-steps
2–5 with: paste the file, reload, run `SAGE → Set up live sync`, enter day key
and facility name, confirm the watched tabs. Drop the GID-hunting and
constant-editing entirely. Keep the note that this repeats per facility
spreadsheet, and that `Sync now` is available from the menu as the manual test.

### 13.8 The two generator specs' GID notes

`dual-meet-schedule-generator-spec.md:597-599` and
`dual-meet-readouts-generator-spec.md:616-618` both say the tab GID is pinned in
`sheets-sync.gs` (`WATCHED_SHEET_GIDS`). The constraint survives — the trigger
still matches on GID, so replacing `SCHEDULE` or `Court Control` still kills the
sync — but the pin now lives in `SYNC_WATCHED_GIDS` in that workbook's Script
Properties, resolved from the tab names once at setup. One-line correction in
each; the surrounding reasoning stands.

### 13.9 `scoresheet-event-picker-spec.md`

- §8.3's note already defers to this spec's `onOpen` contract. Update it to say
  the contract has landed, and that `Generate Scoresheets` is contributed by
  `addSyncMenuItems_` and appears only in a configured workbook.
- §8.5 ("If `DAY_KEY` / `FACILITY_NAME` move to Script Properties") describes
  this as a possible future. It has happened — fold it into §8.4 as the way the
  values are read, and record the change in that spec's §12 divergences.

### 13.10 `D:\Coding Projects\SAGE\CLAUDE.md`

The "Things that must be kept in sync by hand" bullet at `:204` names
`DAY_KEY`/`FACILITY_NAME` in `sheets-sync.gs`. State instead that each
workbook's day key and facility name are set through `SAGE → Set up live sync`
and stored in that workbook's Script Properties, and that setup validates them
against the registry before saving.

### 13.11 Already done

`sage-docs/docs/specs/README.md` already lists this spec under **Sync & live
data**. No change needed.

---

## 14. Out of scope

- Publishing the sync script as an Apps Script **Library**. It is the way to
  stop updating N workbooks by hand, and is worth revisiting once this lands,
  but it puts a cross-project dependency in the live sync path, needs library
  access granted per editor, and still needs a per-workbook version bump when
  pinned.
- `clasp` or Apps Script API pushes.
- `CLOUD_RUN_BASE_URL` and `SCORESHEET_GENERATOR_URL` moving to properties —
  identical everywhere, neither secret.
- Any change to the sync protocol, endpoints, debounce behaviour, or
  `events.json`.
- Any change to `tools/scoresheet-generator.html` or the event picker's page
  half. §8 changes where the workbook menu reads two values; the page's contract
  with the query string is untouched.
- A standard-tournament sheet generator.

---

## 15. Divergences

- **Pause/resume, added post-spec.** Not in the original design: a
  `SYNC_PAUSED` Script Property (unguarded by spreadsheet ID — an inherited
  pause on a copy fails safe rather than misdirecting anything), an
  `isSyncPaused_()` check added to `onEditInstallable` and `runIfSettled`,
  and a `Pause live sync`/`Resume live sync` menu item in the configured
  branch of `addSyncMenuItems_` (§7.2), swapped by label/handler based on
  state at the last `onOpen`. Added because the primary safeguard — the
  `onEditInstallable` trigger not existing until `Set up live sync` runs —
  only covers roster-pasting and schedule fixes done *before* that first
  setup call. A workbook already configured (an early setup, a dry run, a
  reopened `Live sync settings`) had no way to stop automatic publishing
  during further edits without manually deleting the trigger in the Apps
  Script UI. `Sync now`/`testSyncNow` deliberately bypasses the pause check —
  it's an explicit manual action, not the automatic edit-triggered path
  pausing is scoped to. Surfaced in the setup dialog too
  (`getSyncSetupState`'s `paused` field) so reopening `Live sync settings` on
  a paused workbook doesn't silently hide that saving there won't resume it.
- **§4.3 ordering when §4.1 itself is unreachable.** §6.3 lists §4.1 then §4.2
  as steps 3 and 4 but doesn't say what happens to step 4 when step 3 comes
  back `unreachable` rather than `ok`/`blocking`. The built `saveSyncSetup`
  skips the test sync entirely in that case and saves with §4.1's warning: if
  `GET /sync/config` couldn't be reached, `POST /sync/:day` to the same host
  is extremely unlikely to fare differently, and attempting it anyway would
  just double the wait before the operator sees a result. §4.2 still runs
  normally whenever §4.1 came back `ok`.
