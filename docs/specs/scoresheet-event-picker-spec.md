# Spec — Scoresheet Generator event picker

Give `tools/scoresheet-generator.html` a second way to supply its matches:
pick a registered **event → day → facility** and pull that facility's matches
straight from the published snapshot in `event-data`, instead of downloading
the workbook's `CSV` tab and re-uploading the file.

Two files change, in two repos:

| File | Repo | Change |
| --- | --- | --- |
| `tools/scoresheet-generator.html` | `sage-match-control.github.io` | the picker itself (§4–§6) |
| `scripts/sheets-sync.gs` | `sage-tools-api` | a `SAGE → Generate Scoresheets` menu that deep-links into it (§8) |

**No running code changes.** No new endpoint, no auth, no CORS work, no
`event-data` schema change — and specifically **no `package.json` bump and no
Cloud Run deploy**: `sheets-sync.gs` runs inside a spreadsheet and ships by
being pasted into that workbook's script project, so nothing about what Cloud
Run is running is affected. Same arrangement as every other `scripts/*.gs`
change.

The two halves are independent and either can ship first — the menu is a link,
so the page must work standalone anyway.

**This adds a source; it removes nothing.** Drag-and-drop and click-to-browse
CSV upload stay exactly as they are, as a first-class peer of the picker rather
than a degraded fallback — see §7.5.

---

## 1. Current state

The page is four steps (`tools/scoresheet-generator.html`, form at ~line 412):

| Step | Card | Control |
| --- | --- | --- |
| 1 | Name the event | `#evtName` |
| 2 | Pick a scoresheet type | four `.type-card` radios, `name="type"` |
| 3 | Upload your matches CSV | `#dropzone` + `#csvInput` |
| 4 | Add blank scoresheets | `#blankCount` |

The submit handler (~line 616) reads those four, builds a `FormData`, and POSTs
to the NDJSON streaming endpoint:

```js
const fd = new FormData();
fd.append('evt', evt);
fd.append('type', type);
fd.append('csv', file);          // <- a File from #csvInput
fd.append('blanks', String(blanks));
```

`API_URL` is
`https://sage-tools-api-811926984834.us-central1.run.app/scoresheets/generate/stream`.

The page is **not** a PWA — it registers no service worker and has no
`.webmanifest` of its own (unlike `control-center.html` and
`tournament-calculator.html`). Nothing here changes that.

### What the operator does today

Open the workbook → click the `CSV` tab → File → Download → Comma Separated
Values → find the file in Downloads → open this page → type the event name →
pick a type → upload → generate. This spec removes the middle five steps for any
event registered in `event-data/config/events.json`.

---

## 2. Why the published snapshot is the source

The decisive fact, and the reason this is a frontend-only change:

**The snapshot's `matchesCsv` is already the exact CSV this tool parses.**

`event-data/pnf-x-bup-dual-meet/data/pnf-x-bup-day1.json` →
`facilities[].matchesCsv` is a CSV **string** whose header is:

```
matchNumber,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score
```

`Match` in `sage-tools-api/src/scoresheets/PageChunkBuilder.mjs` reads
`matchNumber`, `teamCode1`, `team1Player1`, `team1Player2`, `teamCode2`,
`team2Player1`, `team2Player2` — every one present. `CsvService` maps by header
*name*, not position, so the differing column order is irrelevant and the extra
score/court/schedule columns are ignored.

That string can be handed to the existing `FormData` path as a `Blob` with no
transformation whatsoever.

Three further properties make this the right source rather than a new backend
fetch:

- **Same origin.** `event-data` is a project repo under the same GitHub Pages
  org, so it serves from `https://sage-match-control.github.io/event-data/...` —
  the same origin as this page. No CORS, no preflight, no allow-list.
- **No secrets.** The snapshot is public static JSON. Fetching the *sheet*
  instead would need `GOOGLE_SHEETS_API_KEY`, which means a backend endpoint,
  which means the operator token, which means sign-in on a tool that has never
  had one.
- **Already the pattern.** `control-center.html` resolves every event this exact
  way (`configUrlFor` / `snapshotUrlFor`, ~line 2096). This spec mirrors it.

### The freshness caveat, and why it is already handled

A snapshot only exists once a sync has fired for that day. That is not a
practical constraint: pasting rosters into a category tab *is* an edit, which
fires `sheets-sync.gs`'s `onEdit` trigger, which publishes. And
`features/preparing-an-event.md` §5 puts scoresheet printing in the dry run —
which happens after rosters are in, and whose entire purpose is proving a sync
reaches the public page.

Where it can't be satisfied, the upload path is right there. The UI must say so
rather than dead-ending (§6.2).

---

## 3. Data sources

### 3.1 Registry — `config/events.json`

```
https://sage-match-control.github.io/event-data/config/events.json?t=<Date.now()>
```

The `?t=` cache-buster is required, not optional: GitHub Pages caches responses
for ~10 minutes with no way to force an early refresh. Carry over the comment
from `control-center.html` ~line 2087 explaining this.

Fields this spec reads (full schema in `technical/event-data-config.md`):

| Path | Use |
| --- | --- |
| `events.<key>.archived` | if truthy, **exclude** from the picker |
| `events.<key>.title` | event dropdown label; fall back to the key |
| `events.<key>.days.<dayKey>.label` | day dropdown label |
| `events.<key>.days.<dayKey>.date` | day sort key, and the "today" default |

Nothing else. `type`, `display`, `facilities[].sheetId` and `isLive` are all
irrelevant here — facility names come from the snapshot, not the registry, so a
facility registered but never synced can't be offered (§7.3).

### 3.2 Snapshot — `<event-key>/data/<day-key>.json`

```
https://sage-match-control.github.io/event-data/<eventKey>/data/<dayKey>.json?t=<Date.now()>
```

Top-level keys: `day`, `label`, `isLive`, `generatedAt`, `facilities`,
`failedFacilities`, `staleFacilities`.

Each `facilities[]` entry: `{ name, matchesCsv, standingsCsv, syncedAt }`.

Read `name`, `matchesCsv`, `syncedAt`. Ignore `standingsCsv` entirely.
`failedFacilities` and `staleFacilities` are string arrays used only for the
warning in §6.3.

---

## 4. UI

### 4.1 Step order changes

The cards get reordered so the flow reads downward. **Move the existing "Name
the event" card below the new source card** — the event name is auto-filled from
the picker, and a field that fills itself from a control *below* it reads wrong.

| Step | Card | Change |
| --- | --- | --- |
| 1 | **Choose your matches** | **new** — mode toggle, plus either the pickers or the existing dropzone |
| 2 | Name the event | unchanged markup, moved down one, auto-filled in event mode |
| 3 | Pick a scoresheet type | unchanged |
| 4 | Add blank scoresheets | unchanged |

Renumber the `.step-num` spans accordingly. The existing dropzone markup
(`#dropzone`, `#csvInput`, `#dzText`) **moves into step 1's upload panel
verbatim** — same ids, same classes, so the existing `dropzone`/`csvInput`
wiring and the `.dz-file` filename display keep working untouched.

### 4.2 Step 1 markup

Reuse `.type-card` for the mode toggle — it is already a styled radio and needs
no new CSS beyond a two-column grid:

```html
<!-- Step 1: Where the matches come from -->
<div class="card">
  <div class="step-label"><span class="step-num">1</span><span class="step-title">Choose your matches</span></div>
  <p class="step-help">Pull them from a registered event, or upload a CSV yourself.</p>

  <div class="type-grid two">
    <label class="type-card">
      <input type="radio" name="source" value="event" checked>
      <span class="dot"></span>
      <span class="t-name">Registered event</span>
      <span class="t-desc">Straight from live event data</span>
    </label>
    <label class="type-card">
      <input type="radio" name="source" value="upload">
      <span class="dot"></span>
      <span class="t-name">Upload a CSV</span>
      <span class="t-desc">Any matches file</span>
    </label>
  </div>

  <!-- Event mode -->
  <div id="eventPanel" class="source-panel">
    <div class="picker-grid">
      <label class="f"><span class="lab">Event</span>
        <select id="eventSelect"></select></label>
      <label class="f"><span class="lab">Day</span>
        <select id="daySelect"></select></label>
      <label class="f"><span class="lab">Venue</span>
        <select id="facilitySelect"></select></label>
    </div>
    <p class="picker-meta" id="pickerMeta"></p>
  </div>

  <!-- Upload mode: the existing dropzone, moved here verbatim -->
  <div id="uploadPanel" class="source-panel" hidden>
    <div class="dropzone" id="dropzone"> ... unchanged ... </div>
    <input type="file" id="csvInput" accept=".csv">
  </div>
</div>
```

Two markup notes:

- **Drop `required` from `#csvInput`.** It is now conditionally required, and a
  `required` control inside a `hidden` panel blocks native form submission with
  a focus error the user cannot see. §5.4 validates in JS instead.
- `.type-grid.two` is a one-line CSS addition — the existing `.type-grid` is
  sized for four cards.

### 4.3 New CSS

Add beside the existing `.type-grid` block (~line 167). Use the existing custom
properties; introduce no new colours beyond the two state colours below.

```css
.type-grid.two{grid-template-columns:repeat(2,1fr);}
.source-panel{margin-top:18px;}
.picker-grid{display:grid; gap:12px; grid-template-columns:1fr;}
@media (min-width:620px){ .picker-grid{grid-template-columns:1.4fr 1fr 1.2fr;} }
.picker-grid .f{display:flex; flex-direction:column; gap:6px;}
.picker-grid .lab{
  font-family:'Barlow Condensed',sans-serif;
  font-weight:600; font-size:13px; letter-spacing:.06em;
  text-transform:uppercase; color:var(--ink-soft);
}
.picker-grid select{
  width:100%; padding:11px 12px;
  border:1px solid var(--line); border-radius:10px;
  background:var(--white); color:var(--ink);
  font-family:'Inter',sans-serif; font-size:15px;
}
.picker-grid select:disabled{background:var(--paper-dim); color:var(--ink-soft);}
.picker-meta{
  margin:12px 0 0; font-size:13px; color:var(--ink-soft); min-height:18px;
}
.picker-meta.warn{color:#9A6B00;}
.picker-meta.err{color:#B3261E;}
```

Match the `select` padding/border/radius to the page's existing `input` rule so
the three dropdowns and `#evtName` look like one family.

### 4.4 Footer copy

The footer currently reads:

> Powered by **S.A.G.E.** — sends your CSV straight to the scoresheet API, nothing is stored.

Change to:

> Powered by **S.A.G.E.** — matches go straight to the scoresheet API, nothing is stored.

---

## 5. Behaviour

### 5.1 Constants and state

Add near `API_URL` at the top of the `<script>`:

```js
// event-data is a project repo under the same GitHub Pages org, so these are
// same-origin — no CORS. Pages caches every response for ~10 minutes with no
// way to force an early refresh; the ?t= param is what keeps a fetch fresh
// instead of stuck on a stale copy. Same arrangement as control-center.html.
const GHPAGES_OWNER = 'sage-match-control';
const GHPAGES_REPO  = 'event-data';
const configUrlFor   = () => `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/config/events.json?t=${Date.now()}`;
const snapshotUrlFor = (eventKey, dayKey) => `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${eventKey}/data/${dayKey}.json?t=${Date.now()}`;
const FETCH_TIMEOUT_MS = 8000;   // venue wifi hangs rather than fails outright

let REGISTRY = new Map();  // eventKey -> registry entry, archived excluded
let SNAPSHOT = null;       // the currently selected day's snapshot, or null
```

Read the mode with `form.querySelector('input[name="source"]:checked').value` at
the point of use — no separate mode variable to keep in step.

### 5.2 Load the registry on page load

```js
async function fetchJson(url){
  const ctrl = new AbortController();
  const timer = setTimeout(() => ctrl.abort(), FETCH_TIMEOUT_MS);
  try {
    const res = await fetch(url, { signal: ctrl.signal });
    if (!res.ok) throw new Error(`responded ${res.status}`);
    return await res.json();
  } finally { clearTimeout(timer); }
}

async function loadRegistry(){
  const raw = await fetchJson(configUrlFor());
  REGISTRY = new Map(Object.entries(raw.events || {}).filter(([, e]) => !e.archived));
}
```

On `DOMContentLoaded`: call `loadRegistry()`, then populate the event select and
cascade. On throw, or on an empty registry, fall back to upload mode (§6.1).

### 5.3 The cascade

**Events** — insertion order of `raw.events`, labelled `title || key`.

**Days** — mirror `control-center.html` ~line 5279: map the day entries to
`{key, label, date}` and sort by `date`, undated days last:

```js
const days = Object.entries(rawEvent.days || {})
  .map(([key, d]) => ({ key, label: d.label, date: d.date || null }))
  .sort((a, b) => (a.date || '9999-99-99').localeCompare(b.date || '9999-99-99'));
```

Default selection: the day whose `date` is today, else the first day whose
`date` is in the future, else the first day. (`control-center.html` ~line 3723
does exactly this — copy the logic, not the surrounding console state.)

**Facilities** — these require a snapshot, so selecting a day fetches
`snapshotUrlFor(eventKey, dayKey)` into `SNAPSHOT` and fills the venue select
from `SNAPSHOT.facilities[].name`. While that fetch is in flight, disable the
venue select and set `#pickerMeta` to `Loading day data…`.

A facility whose `matchesCsv` has no data rows is listed but **disabled**, its
option text suffixed `— no matches yet`. Listing it is deliberate: a venue
silently missing from the dropdown looks like a config bug, and the operator
needs to be able to tell the two apart.

Every selection change re-runs `updatePickerMeta()` (§6.3) and, in event mode,
re-fills `#evtName` (§5.5).

### 5.4 Submit

Replace the `const file = csvInput.files[0]` block. Everything downstream — the
`FormData`, the fetch, `readNdjsonStream`, the progress bar, `base64ToBlob`,
`triggerDownload` — is untouched.

```js
const mode = form.querySelector('input[name="source"]:checked').value;

let csvPart, csvPartName;
if (mode === 'event') {
  const fac = selectedFacility();          // SNAPSHOT.facilities.find(...)
  if (!fac || !countMatches(fac.matchesCsv)) {
    setStatus('Pick an event, day and venue that has matches in it.', 'err');
    return;
  }
  csvPart = new Blob([fac.matchesCsv], { type: 'text/csv' });
  csvPartName = 'matches.csv';
} else {
  csvPart = csvInput.files[0];
  if (!csvPart) { setStatus('Add a CSV file to continue.', 'err'); return; }
  csvPartName = csvPart.name;
}

if (!evt) { setStatus('Add an event name to continue.', 'err'); return; }
// ...existing blanks validation, unchanged...

const fd = new FormData();
fd.append('evt', evt);
fd.append('type', type);
fd.append('csv', csvPart, csvPartName);   // third arg matters — see below
fd.append('blanks', String(blanks));
```

**Pass the third `append` argument.** `ScoresheetService` writes the upload to
`path.join(workspace.tempDir, csvFileName || "input.csv")` using multer's
`originalname`. A `Blob` appended without a filename arrives as `"blob"`. The
fallback would cope, but an explicit `matches.csv` keeps server-side logs and
temp paths readable, and costs one argument.

`countMatches` counts non-empty lines after the header, tolerating CRLF — the
snapshot's `matchesCsv` uses `\r\n`:

```js
const countMatches = (csv) =>
  csv.split(/\r?\n/).slice(1).filter(l => l.trim() !== '').length;
```

### 5.5 Event name auto-fill

In event mode, set `#evtName` to the registry `title` (falling back to the event
key) whenever the event selection changes — **but only if the field is empty or
still holds the previously auto-filled value**. Track the last auto-filled
string in a variable and compare: a name the operator typed themselves must
survive changing the day or venue.

The field stays editable in both modes and remains required.

### 5.6 Output filename

Currently:

```js
const safeName = evt.replace(/[^a-z0-9]+/gi, '_').replace(/^_+|_+$/g, '') || 'scoresheet';
triggerDownload(blob, `${safeName.toUpperCase()}_SCORESHEET`);
```

In event mode, append the day key and venue so a multi-venue day's stacks are
distinguishable in the Downloads folder:

```
PNF_X_BUP_DUAL_MEET_PNF_X_BUP_DAY1_PAMPANGA_PICKLEBALL_CENTER_SCORESHEET.pdf
```

Build it by running each part through the same `[^a-z0-9]+ -> _` slug helper and
joining with `_`. Upload mode keeps today's `<EVT>_SCORESHEET` exactly.

---

### 5.7 Deep-link query parameters

The page accepts three optional query params so the workbook menu (§8) — or a
bookmark — can land on a preselected venue.

| Param | Value | Effect |
| --- | --- | --- |
| `day` | a day key, e.g. `pnf-x-bup-day1` | selects the owning event **and** that day |
| `facility` | a facility name, URL-encoded | selects that venue |
| `type` | one of the four registered type ids | preselects the scoresheet type radio |

`?day=` alone is enough to identify the event. **Day keys are globally unique
across every event** — an enforced validation rule in `SyncConfigStore`, because
a day key is also the `/sync/:day` route — so the page resolves the event by
scanning the registry for the day, and no `event` param is needed. Accept an
`event` param as an optional, ignored-if-inconsistent hint only if it turns out
to be wanted; do not require it.

Applied **after** `loadRegistry()` resolves and before the default day selection
in §5.3, so a deep link wins over the today/next-upcoming default. Once applied,
params are inert: a subsequent manual change to any dropdown is never
overridden, and the params are not re-read or written back to the URL.

Matching rules:

- `day` — exact match against day keys.
- `facility` — **exact and case-sensitive**, matching how the sync itself
  compares facility names everywhere else in the system. Do not
  case-fold or trim as a convenience; that would make this page the one place
  in S.A.G.E. where a facility-name mismatch silently works, and mask the exact
  bug the operator needs to find in `sheets-sync.gs`.
- `type` — must be one of the four ids in `ScoresheetConfig.mjs`
  (`scoresheet`, `scoresheet-best-of-3`, `scoresheet-no-ref`,
  `scoresheet-no-ref-wide`). An unrecognised value is ignored silently and the
  default `scoresheet` radio stays checked.

**Archived events resolve by deep link only.** `REGISTRY` excludes them (§7.2),
but a link naming an archived event's day key still works: resolve against the
unfiltered `raw.events`, and if it hits, add that one event to the dropdown for
this session with an `(archived)` suffix on its label. Reprinting a finished
event's scoresheets is rare but legitimate, and the alternative is a dead link
with a confusing "unknown day" error. Archived events never appear in the
dropdown otherwise.

## 6. Error and edge states

All of these write to `#pickerMeta` (or `#statusMsg` for submit-time failures).
None may leave the page with no way forward — the upload path is always one
radio click away, and the copy should say so.

### 6.1 Registry unavailable

`events.json` 404s, times out, or has zero non-archived events.

- Force upload mode: check the `upload` radio, show `#uploadPanel`.
- **Disable** the `Registered event` radio and add a `title` explaining why.
- `#pickerMeta` (`.err`): `Couldn't load the event list — upload a CSV instead.`

The page must remain fully usable. This is the pre-change behaviour, so a
registry outage is a graceful degrade, not an outage of the tool.

### 6.2 No snapshot for the selected day

`snapshotUrlFor(...)` 404s — the common real case, a day registered but never
synced.

- Clear and disable the venue select.
- `#pickerMeta` (`.warn`): `No published data for this day yet — it appears
  after the first sync. Upload a CSV instead.`
- Submit in event mode is blocked by §5.4's guard.

### 6.3 Freshness and partial syncs — `updatePickerMeta()`

When a facility is selected and has matches, `#pickerMeta` shows:

```
128 matches · synced 4 minutes ago
```

from `countMatches(fac.matchesCsv)` and a relative time off `fac.syncedAt`.

This is the single most useful line on the new card: it is how the operator
confirms the roster paste actually landed before printing 128 sheets.

Escalate to `.warn` and append when either applies:

- `SNAPSHOT.staleFacilities` names this facility →
  `· venue last synced <relative>, data may be behind the sheet`
- `SNAPSHOT.failedFacilities` has an entry starting `"<name>: "` →
  `· last sync for this venue failed`

Both arrays hold strings (`failedFacilities` entries are `"<name>: <error>"`), so
match on the name prefix, not equality.

### 6.4 Placeholder playoff rows

Bronze/Final rows carry `-` or the bare team code until the group stage
completes — `Match.#formatTeam` already renders those as `<code> - ` with blank
name lines. **This is correct and wanted**: it produces a pre-printed final with
blank name lines to write into. Do not filter these rows out, and do not warn
about them.

### 6.5 Trailing whitespace in names

Snapshot names carry trailing spaces (`"Jazmin Solis "`). Harmless in the
rendered PDF, identical to the manual-download path, and not this page's to fix.
Leave it.

---

### 6.6 Deep link that doesn't resolve

A link from a workbook whose `DAY_KEY` or `FACILITY_NAME` is wrong is the whole
reason these messages matter — they are the fastest diagnosis of a
`sheets-sync.gs` misconfiguration the system currently offers, so name the
offending value rather than failing generically.

**Unknown `day`** — no event in the registry (archived included) owns it. Fall
back to the normal default selection, and `#pickerMeta` (`.err`):

> `Day "<day>" isn't in the event registry — check DAY_KEY in this workbook's
> sync script. Showing <event title> instead.`

**Unknown `facility`** — the day resolved, the venue name didn't. Select the
event and day normally, leave the venue select unselected, and `#pickerMeta`
(`.err`), listing what does exist:

> `No venue named "<facility>" on this day — facility names are case-sensitive.
> This day has: Main, Annex, Dreamcourts.`

Naming the case sensitivity in the message is deliberate: a case mismatch
between `FACILITY_NAME` and `events.json` is a known silent failure mode in this
system (`features/preparing-an-event.md` §4 calls it out), and this is the one
screen where it becomes visible.

**Registry unreachable with params present** — §6.1 wins. The page falls back to
upload mode; the params are dropped and not reported. A deep link is a
convenience, never the only way in.

## 7. Decisions

### 7.1 One venue per PDF; no "All venues" option

Each facility workbook numbers its own matches `1..N`, so concatenating two
venues produces a PDF with duplicate match numbers — actively harmful on a paper
sheet, which is identified by that number. One venue at a time, generated twice,
is correct. Generating N PDFs in sequence from one click is a reasonable future
convenience and is out of scope (§10).

### 7.2 Archived events are excluded

Mirrors `control-center.html` ~line 5331. Nobody prints scoresheets for a
finished event, and BKL Cup 2026's snapshots no longer feed its archived pages
(see the `_comment` on that event in `events.json`).

### 7.3 Facilities come from the snapshot, not the registry

The registry lists facilities that may have no spreadsheet yet — a blank
`sheetId` is explicitly allowed, so a day's entry can exist before its workbook
does. Offering one would produce a picker entry that can never generate
anything. The snapshot lists what actually synced.

### 7.4 The workbook menu opens a link; it does not call the API

A `SAGE → Generate Scoresheets` menu item is in scope (§8), but **only as a
deep link**. The version first considered — Apps Script POSTing the `CSV` tab to
`/scoresheets/generate` via `UrlFetchApp` and handing back a PDF — is rejected.
Three of its four problems simply do not exist for a link, and the fourth is
solved by putting the item in a different file:

- **Timeouts.** `UrlFetchApp` cannot consume NDJSON progressively, so it would
  have to call the blocking `/scoresheets/generate` and sit there against Apps
  Script's execution limit. The streaming endpoint exists precisely because
  Puppeteer takes real time on a large day; that trades a working progress bar
  for a hard cap. **A link makes no request at all**, and generation runs in
  the browser against the streaming endpoint exactly as it does today.
- **PDF delivery.** Apps Script cannot hand a file to the browser. It would need
  either a Drive scope — re-authorization on every copy, files accumulating in
  Drive — or the whole PDF pushed through an HTML dialog as base64. **A link
  needs no new scope**: the PDF downloads from the page, in the browser, the way
  it already does.
- **Distribution.** `sheet-generator.gs` is bound to the SAGE Dual Meet Master,
  so an item there reaches dual-meet workbooks *copied after the change* — not
  existing copies, and not standard tournaments at all. **Resolved by putting
  the item in `sheets-sync.gs` instead** (§8.1), which is installed on every
  facility workbook of every event, and which already holds the two values the
  link needs.
- **Menu hazard.** The `SAGE` menu's only item today is `Generate event tabs`:
  run-once, non-idempotent, and a failed run costs a fresh copy of the master.
  Parking a repeatable action beside it invites the wrong click. **Still true,
  and still worth guarding** — §8.3 keeps the two items apart with a separator
  and orders them so the destructive one is not the default landing spot.

### 7.5 Upload stays a first-class peer, not a fallback

The two sources sit as equal radio cards, event mode selected on load. Upload is
not a hidden escape hatch, an "advanced" disclosure, or something reachable only
when the registry fails — it is one click away at all times and its behaviour is
byte-identical to the pre-change page.

It is not redundant with the picker. It is the only path for:

- an event that was never registered in `events.json` — a scrimmage, a club
  night, a one-off, anything run before the site work is done;
- a hand-edited or trimmed CSV — reprinting one court's afternoon, or a bracket
  someone fixed up by hand;
- a standard tournament whose workbook predates any of this;
- printing before the day's first sync has published anything (§6.2);
- generating from a machine that can't reach `event-data`.

**Implications for implementation:** the dropzone markup is moved, never
rewritten (§4.2); the upload branch of the submit handler and its
`<EVT>_SCORESHEET.pdf` filename are preserved verbatim (§5.4, §5.6); and both
modes must be exercised by the acceptance checklist, not just the new one (§9).

A toggle is used rather than showing both controls at once so it is never
ambiguous which source will generate — a half-filled hidden panel cannot
silently win over the visible one. That ambiguity is worth a click to avoid on a
tool that commits hundreds of printed pages.

### 7.6 Rejected — a backend endpoint that reads the sheet live

A `GET /scoresheets/matches?event=&day=&facility=` reusing `SheetsCsvFetcher`
would remove the snapshot dependency entirely. It also needs
`GOOGLE_SHEETS_API_KEY` (server-side only), which means the operator bearer
token, which means adding sign-in to a tool that has deliberately never had one.
The snapshot is fresh enough for the one moment this is used (§2), so this is
the escape hatch if §6.2 ever proves to bite in practice — not day-one work.

---

## 8. The workbook menu — `sheets-sync.gs`

`SAGE → Generate Scoresheets` opens the generator in a new tab with this
workbook's day and venue already selected. It makes no HTTP request, needs no
new OAuth scope, and touches nothing in the spreadsheet.

### 8.1 Why `sheets-sync.gs` and not `sheet-generator.gs`

`sheets-sync.gs` is installed **once per facility spreadsheet, for every
event** — dual meet and standard tournament alike, including BKL Cup 2026's
fifteen hand-built workbooks. `sheet-generator.gs` is bound to the Dual Meet
Master and reaches only dual-meet copies made after the change.

More decisively, `sheets-sync.gs` already holds exactly the two values the link
needs, and they are already required to be correct for that workbook to sync at
all:

```js
const DAY_KEY = 'day2';
const FACILITY_NAME = 'Main';
```

Nothing new to configure, and no new way for the workbook to be wrong: a link
that lands on the wrong venue means the sync was already pointing at the wrong
venue, and §6.6's error message says so in as many words.

### 8.2 New config constant

Add to the **"usually leave as-is"** block, not the per-spreadsheet block — it
is the same for every install:

```js
const SCORESHEET_GENERATOR_URL =
  'https://sage-match-control.github.io/tools/scoresheet-generator.html';
```

Leave the header comment's "edit these three per spreadsheet" wording alone —
the three stay three.

### 8.3 The `onOpen` collision — read this before writing the menu

> **`sync-script-configuration-spec.md` §6 is authoritative for the `onOpen`
> contract.** It resolves the same collision with identical `onOpen` bodies in
> both files delegating to feature-detected `addSyncMenuItems_` /
> `addGeneratorMenuItems_` builders, which is stricter than the two
> feature-detecting bodies below. If that spec is built, add
> `Generate Scoresheets` inside `addSyncMenuItems_` and skip the rest of this
> section. The explanation of *why* the collision exists still applies.

`sheets-sync.gs` defines **no `onOpen` today**; `sheet-generator.gs` defines the
only one (~line 723). A generated dual-meet workbook ends up carrying **both**
scripts — the master copy brings the generator, and step 8 of
`technical/adding-a-new-event.md` pastes the sync script in afterwards.

Two files in one Apps Script project both declaring `function onOpen()` is not
an error. The declarations hoist, the last one loaded wins silently, and load
order is not something the operator controls. Adding a naive `onOpen` to
`sheets-sync.gs` would therefore make the `SAGE` menu lose either
`Generate event tabs` or `Generate Scoresheets` depending on file order — a bug
that appears only in workbooks that have both, only after a reload, and looks
exactly like a paste that didn't take.

**Fix: make both definitions produce the same menu**, so whichever one wins is
irrelevant. Each file feature-detects the other's entry point rather than
assuming it exists.

In `sheets-sync.gs`:

```js
// NOTE: sheet-generator.gs declares its own onOpen, and a generated dual-meet
// workbook carries BOTH scripts. Duplicate declarations don't error — the last
// file loaded wins, and load order isn't controllable. So this body and
// sheet-generator.gs's must build the SAME menu, each feature-detecting the
// other's entry point. Change one, change the other.
function onOpen() {
  const menu = SpreadsheetApp.getUi().createMenu('SAGE');
  menu.addItem('Generate Scoresheets', 'showScoresheetLink');
  if (typeof showGenerateSidebar === 'function') {
    menu.addSeparator().addItem('Generate event tabs', 'showGenerateSidebar');
  }
  menu.addToUi();
}
```

In `sheet-generator.gs`, replacing the existing `onOpen`:

```js
// See the matching note in sheets-sync.gs — these two bodies must agree.
function onOpen() {
  const menu = SpreadsheetApp.getUi().createMenu('SAGE');
  if (typeof showScoresheetLink === 'function') {
    menu.addItem('Generate Scoresheets', 'showScoresheetLink');
    menu.addSeparator();
  }
  menu.addItem('Generate event tabs', 'showGenerateSidebar');
  menu.addToUi();
}
```

Both yield `Generate Scoresheets · ─── · Generate event tabs` when both scripts
are present, and the single available item otherwise. The separator and the
ordering are §7.4's menu-hazard guard: the safe, repeatable item is first, and
the run-once destructive one is not what the cursor lands on.

Editing `sheet-generator.gs` here does **not** bump `package.json` and is not a
deploy, same as any other `scripts/*.gs` change. It also does not require
`node scripts/verify-sheet-generator.mjs` to change — that script exercises
`generateEventTabs`, not the menu — but running it is still the cheap check
that the file still parses.

### 8.4 The dialog

Apps Script cannot open a URL from server-side code, and a `window.open()` fired
on load inside the dialog's sandboxed iframe is blocked as a popup. **The
navigation must come from a real user click on an anchor.** This is the one
thing that will silently not work if implemented the obvious way.

```js
function showScoresheetLink() {
  const url = SCORESHEET_GENERATOR_URL +
    '?day=' + encodeURIComponent(DAY_KEY) +
    '&facility=' + encodeURIComponent(FACILITY_NAME);

  const html = HtmlService.createHtmlOutput(
    '<style>body{font-family:Arial,sans-serif;font-size:13px;padding:8px 12px;}' +
    'a.btn{display:inline-block;margin-top:14px;padding:10px 18px;background:#7CB92C;' +
    'color:#fff;border-radius:8px;text-decoration:none;font-weight:bold;}' +
    'dl{margin:10px 0;} dt{color:#5B6B74;font-size:11px;text-transform:uppercase;}' +
    'dd{margin:0 0 8px;font-weight:bold;}</style>' +
    '<p>Opens the Scoresheet Generator with this workbook preselected.</p>' +
    '<dl><dt>Day</dt><dd>' + DAY_KEY + '</dd>' +
    '<dt>Venue</dt><dd>' + FACILITY_NAME + '</dd></dl>' +
    '<a class="btn" href="' + url + '" target="_blank" rel="noopener" ' +
    'onclick="google.script.host.close()">Open Scoresheet Generator</a>'
  ).setWidth(380).setHeight(230);

  SpreadsheetApp.getUi().showModalDialog(html, 'Generate Scoresheets');
}
```

The dialog **shows `DAY_KEY` and `FACILITY_NAME` before the click** on purpose:
it lets the operator catch a misconfigured workbook here, rather than after
landing on §6.6's error. That readout is the feature, not decoration — do not
reduce this to a bare button.

`rel="noopener"` accompanies `target="_blank"` as standard practice. Both values
are interpolated into HTML, so if either is ever made operator-editable at
runtime they need escaping; as module-level constants edited in source, they are
not a live injection path.

### 8.5 If `DAY_KEY` / `FACILITY_NAME` move to Script Properties

There is a separate, larger proposal to stop `sheets-sync.gs` carrying
per-spreadsheet constants in source at all and read them from Script Properties
instead (the pattern that file already uses for the shared secret). If that
lands, this section changes in exactly one place — `showScoresheetLink` reads
the two values from `PropertiesService.getScriptProperties()` rather than
module constants:

```js
const props = PropertiesService.getScriptProperties();
const dayKey   = props.getProperty(PROP_DAY_KEY);
const facility = props.getProperty(PROP_FACILITY_NAME);
if (!dayKey || !facility) {
  SpreadsheetApp.getUi().alert('Run setup() first — this workbook has no day/venue configured yet.');
  return;
}
```

Nothing else in §8 depends on where the values come from. The two changes are
independent and can ship in either order.

### 8.6 Install instructions

`sheets-sync.gs`'s header comment is its install runbook (steps 1–6). Add a
sentence to step 6 noting that reloading the spreadsheet also adds
`SAGE → Generate Scoresheets`, and that it needs no authorization of its own —
it opens a link and calls nothing.

The same note belongs in `_templates/CLAUDE.md` §2 step 8 in the site repo,
which walks through this install per facility spreadsheet.

## 9. Acceptance checklist

- [ ] Page loads with **Registered event** selected, the event dropdown
      populated from `events.json`, and `bkl-cup-2026` **absent** (archived).
- [ ] Selecting an event fills the day dropdown sorted by `date`, defaulting to
      today / next upcoming / first.
- [ ] Selecting a day fetches the snapshot and fills the venue dropdown from
      `facilities[].name`; the venue select is disabled while that is in flight.
- [ ] `#pickerMeta` shows a match count and a relative `syncedAt` for the
      selected venue.
- [ ] `#evtName` auto-fills from the registry `title`; typing a custom name and
      then changing day or venue does **not** overwrite it.
- [ ] Generating in event mode produces the same PDF as downloading that
      workbook's `CSV` tab and uploading it — verify against
      `pnf-x-bup-dual-meet` / `pnf-x-bup-day1`: page count and first/last match
      identical.
- [ ] The downloaded filename includes event, day and venue.
- [ ] Switching to **Upload a CSV** shows the dropzone; drag-drop,
      click-to-browse, and the `.dz-file` filename display all still work, and
      generating produces today's `<EVT>_SCORESHEET.pdf` name.
- [ ] With `events.json` unreachable (block it in devtools), the page falls back
      to upload mode, disables the event radio, explains why, and still
      generates a PDF from a file.
- [ ] Picking a day with no published snapshot shows the §6.2 message and blocks
      submit with a readable error rather than posting an empty CSV.
- [ ] A facility with a header-only `matchesCsv` appears in the dropdown,
      disabled, labelled `— no matches yet`.
- [ ] Blank-scoresheet count, the progress bar, and midstream
      `{"phase":"error"}` handling are unchanged in both modes.
- [ ] Layout holds at 375px wide — the three pickers stack.

### Deep links (§5.7, §6.6)

- [ ] `?day=pnf-x-bup-day1&facility=Pampanga%20Pickleball%20Center` lands with
      the event, day and venue all selected and the match count shown — without
      an `event` param.
- [ ] A deep link beats the today/next-upcoming day default.
- [ ] Changing any dropdown after arriving via a deep link sticks; the params
      do not reassert themselves.
- [ ] `?type=scoresheet-no-ref` preselects that radio; `?type=nonsense` is
      ignored and leaves **Standard** checked.
- [ ] `?facility=pampanga%20pickleball%20center` (wrong case) does **not**
      match, and the error names the case sensitivity and lists the real venues.
- [ ] `?day=nope` shows the §6.6 message naming `nope` and still leaves a
      usable page.
- [ ] A deep link to an archived event's day key (`day2`, BKL Cup 2026)
      resolves and shows that event in the dropdown suffixed `(archived)`;
      reloading without params drops it again.

### Workbook menu (§8)

- [ ] In a workbook with **only** `sheets-sync.gs`, the `SAGE` menu appears
      after reload with `Generate Scoresheets` alone.
- [ ] In a workbook with **only** `sheet-generator.gs` (the Dual Meet Master),
      the `SAGE` menu still shows `Generate event tabs` alone — unchanged
      behaviour.
- [ ] In a workbook with **both**, the menu shows both items with a separator
      between them, **and this holds whichever order the two files sit in** —
      test it both ways, since load order is what decides which `onOpen` wins.
- [ ] The dialog displays this workbook's `DAY_KEY` and `FACILITY_NAME` before
      the click.
- [ ] Clicking the button opens the generator in a **new tab** with both params
      set, and closes the dialog. It is not popup-blocked.
- [ ] The menu item triggers no authorization prompt on a workbook whose sync
      script is already authorized, and issues no HTTP request from Apps Script.
- [ ] A workbook with a deliberately wrong `FACILITY_NAME` lands on §6.6's
      case-sensitivity error rather than a blank or misleading selection.
- [ ] `node scripts/verify-sheet-generator.mjs` still passes after the
      `sheet-generator.gs` `onOpen` edit.

## 10. Out of scope

- Any `sage-tools-api` change, including a version bump. Nothing about what
  Cloud Run runs changes.
- A "generate for every venue on this day" batch button (§7.1).
- A live-sheet fetch endpoint (§7.6).
- Sign-in on this page. It stays a no-auth public tool.
- Standings — `standingsCsv` is read by nothing here.
- Making the page a PWA, or wiring it to `tools/sw.js`.
- Any change to the four scoresheet types, their templates, or
  `ScoresheetConfig.mjs`.
- Filtering, reordering or de-duplicating matches before generation. What the
  venue's `CSV` tab holds is what prints.

## 11. Docs to update when this ships

- `features/scoresheet-generator.md` — step 3 becomes "pick an event or upload a
  CSV"; note the match count and sync freshness line.
- `features/preparing-an-event.md` §5 — printing scoresheets no longer needs a
  CSV download.
- `technical/scoresheet-pipeline.md` — one line noting the page can source its
  CSV from a published snapshot, and the deep-link params it accepts.
- `technical/sync-pipeline.md` — `sheets-sync.gs` now also contributes a `SAGE`
  menu; note the shared-`onOpen` contract with `sheet-generator.gs` (§8.3), as
  that is the non-obvious part someone will otherwise break.
- `technical/dual-meet-sheet-generator.md` — same contract, from the other file's
  side.
- `technical/adding-a-new-event.md` step 8 and `_templates/CLAUDE.md` §2 step 8
  — installing the sync script now also gets you the menu.
- `features/dual-meet-sheet-generator.md` — the `SAGE` menu has a second item;
  say plainly that it is safe and repeatable, unlike its neighbour.

`specs/README.md` already lists this spec — it was indexed when the spec was
written, not when the feature ships.

## 12. Divergences

*(Empty at time of writing — record here where the built feature departs from
this document, as `dual-meet-sheet-generator-spec.md` §13 does.)*
