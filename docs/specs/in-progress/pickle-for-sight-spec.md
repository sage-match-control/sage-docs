# Spec — CLSO Pickle for Sight tournament

> **Status: in progress.** The site (§3–§9) is built and committed, both
> venue workbooks are built, and live sync is set up and publishing for both
> `PCPH Main` and `PCPH Annex` (§10, §12.1, §12.2). Still open: the dry run
> (§12.4), due by Friday 25 September, and setting `pickle-for-sight-day1`'s
> `isLive` back to `"auto"` in `event-data/config/events.json` — it is
> currently hardcoded `true` from go-live testing.

Build the event site for **Pickle for Sight**, a one-day open-entry
pickleball tournament on **Sunday, 27 September 2026**. It is played across
two venues, **PCPH Main** and **PCPH Annex**, in Angeles City. The Central
Luzon Society of Ophthalmology (CLSO) presents it in partnership with
S.A.G.E.

The site is an instance of `_templates/standard-tournament-template/`. The
runbook for instantiating a template is
`sage-match-control.github.io/_templates/CLAUDE.md`. This spec gives every
value and every edit for this event, so you should not need to make any
design decisions. If this spec and the runbook disagree, **stop and ask**.
Don't pick one.

---

## 0. How to use this spec

- **Do sections 3–10 in order.** Each section lists exact edits, then a
  check to run. Don't move on until the check passes.
- **Where you work:** `D:\Coding Projects\SAGE\sage-match-control.github.io`
  (the site repo) for §3–§9. §10 edits `D:\Coding Projects\SAGE\event-data`.
  All paths in §3–§9 are relative to the site repo.
- **Find edits by their anchor text, not line numbers.** Every edit quotes
  the exact text to find. The template changes over time, so line numbers
  would drift.
- **Don't commit or push** unless the user asks.
- **Don't edit anything under `_templates/`.** If a template edit seems
  necessary, stop and report it instead.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

---

## 1. Event facts

| Fact | Value |
|---|---|
| Presenter | Central Luzon Society of Ophthalmology (CLSO) |
| Event name | Pickle for Sight |
| Slogan | Play Hard. Win Big. Restore Sight. |
| Date | Sunday, 27 September 2026 — one day |
| Venues | PCPH Main (4 courts), PCPH Annex (5 courts), Angeles City |
| Divisions | Novice, Low Intermediate, High Intermediate |
| Events | Men's, Women's and Mixed Doubles in every division — 9 categories |
| Logo | The mascot (eyeball pickleball with goggles, holding a CLSO paddle) |
| Short link | `tinyurl.com/SAGExPickleForSight` → `https://sage-match-control.github.io/events/pickle-for-sight-2026/` |
| Beneficiary | JBL Ophtha patients |

**The site shows none of these:** playing hours, registration fee, prizes,
phone number, Facebook page, or the pubmat's QR (which points to
registration). The site is a live results hub for players who have already
entered.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| Mascot image | **Staged** at `events/pickle-for-sight-2026/assets/mascot.webp` (1254×1254, white background). Use it as is: no conversion, no background removal | §5.2 |
| QR image for the short link | **Supplied by the user** as `events/pickle-for-sight-2026/assets/qr.png`. If it isn't there, do everything else and report it missing. Don't generate one | Template QR panel |
| Sheet ID for the PCPH Main workbook | `1rNIlK2Zz3zTaLILlpq4rCGIQbtr0ECpG4A3OdP4v25A` | §10 |
| Sheet ID for the PCPH Annex workbook | `1ks84WK7vo5FAenpPbZocm9PLPpLoS6_SRPivVJqmS-0` | §10 |

---

## 3. Create the event folder

The folder already exists, because it holds the staged mascot. Copy the
template's **contents** into it. The trailing `/.` matters: without it,
`cp` would create a nested `standard-tournament-template/` folder inside
the existing one.

```bash
cp -r _templates/standard-tournament-template/. events/pickle-for-sight-2026/
```

**Check:** `ls events/pickle-for-sight-2026` shows `index.html`,
`schedule.html` and `assets/`, and `assets/` still contains
`mascot.webp`.

---

## 4. Replace the `{{TOKENS}}`

Replace every occurrence in both `events/pickle-for-sight-2026/index.html`
and `events/pickle-for-sight-2026/schedule.html`:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `pickle-for-sight-2026` |
| `{{EVENT_TITLE}}` | `Pickle for Sight` |
| `{{EVENT_TAGLINE}}` | `CLSO presents` |
| `{{EVENT_HEADLINE}}` | `Play Hard. Win Big. Restore Sight.` |
| `{{EVENT_DATE_RANGE}}` | `27 September 2026` |
| `{{VENUE}}` | `PCPH, Angeles City` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/SAGExPickleForSight` |
| `{{SCHEDULE_DAY_KEY}}` | `pickle-for-sight-day1` |

None of these values contain `&`, so no HTML-entity escaping is needed.

**Check:** this must print nothing:

```bash
grep -rn '{{' events/pickle-for-sight-2026/
```

---

## 5. `index.html` — content edits

All edits are in `events/pickle-for-sight-2026/index.html`.

### 5.1 Configuration block

Inside the `CONFIGURATION` section of the `<script>`, replace each of these
four `// EXAMPLE — replace` blocks completely, including the
`// EXAMPLE — replace` comment line.

`DAYS`: find

```js
const DAYS = [
  // EXAMPLE — replace
  { key: 'pickle-for-sight-2026-day1', label: 'Day 1', date: '2026-01-01' }
];
```

(the token replacement in §4 has already filled in the key). Replace it
with:

```js
const DAYS = [
  { key: 'pickle-for-sight-day1', label: 'Sep 27', date: '2026-09-27' }
];
```

`FACILITIES`: replace the whole array with:

```js
const FACILITIES = [
  { name: 'PCPH Main',  courts: [1, 4] },
  { name: 'PCPH Annex', courts: [5, 9] }
];
```

`DIVISIONS`: replace the whole object with:

```js
const DIVISIONS = {
  N:  { name: 'N',  full: 'Novice' },
  LI: { name: 'LI', full: 'Low Intermediate' },
  HI: { name: 'HI', full: 'High Intermediate' }
};
```

`EVENTS`: replace the whole object with:

```js
const EVENTS = {
  MD: "Men's Doubles",
  WD: "Women's Doubles",
  XD: "Mixed Doubles"
};
```

Leave `DIVISION_ORDER`, `EVENT_ORDER`, `GO_LIVE_LEAD_HOURS`, `STAGE_META`
and everything else in the block unchanged.

**Check:** `grep -n "EXAMPLE" events/pickle-for-sight-2026/index.html`
prints nothing.

### 5.2 Hero — title accent and mascot

Find:

```html
<h1 class="title">Pickle for Sight</h1>
```

Replace with these two lines. The mascot goes directly above the title:

```html
<div class="event-mascot"><img src="assets/mascot.webp" alt="Pickle for Sight mascot" width="1254" height="1254" /></div>
<h1 class="title"><span class="accent">Pickle</span> for Sight</h1>
```

In the `<style>` block, find this line:

```css
  h1.title .accent{ color:var(--green); }
```

Directly below it, add:

```css
  /* Pickle for Sight — event mascot in a white circle badge. The source image
     is square on a white background, so it is inset (not cropped) at 70% of
     the circle: a square inside a circle is at most 1/√2 ≈ 70.7% of the
     diameter, so its corners stay inside and nothing is clipped. The circle is
     a wrapper, because a percentage padding on the <img> would resolve against
     the hero's width, not the image's. */
  .event-mascot{
    width:clamp(150px,38vw,230px); aspect-ratio:1/1; margin:0 auto 18px;
    display:grid; place-items:center; border-radius:50%; background:var(--white);
    border:2px solid rgba(94,145,6,.55); box-shadow:var(--card-shadow);
  }
  .event-mascot img{ width:70%; height:auto; display:block; }
```

The hero now stacks: S.A.G.E. crown mark → "CLSO presents" → date · venue →
mascot → title → slogan → subtitle. Leave the crown-mark SVG alone. It is
S.A.G.E.'s mark and stays on every event page.

### 5.3 Footer cause line

Find:

```html
<footer>
  Pickle for Sight &middot; 27 September 2026 &middot; PCPH, Angeles City<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

Replace with:

```html
<footer>
  Pickle for Sight &middot; 27 September 2026 &middot; PCPH, Angeles City<br>
  Every registration helps restore sight for JBL Ophtha patients.<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

### 5.4 Example text left over from the template

Find:

```html
placeholder="e.g. Beginner 18+ Men's Doubles"
```

Replace with:

```html
placeholder="e.g. Novice Men's Doubles"
```

Leave the code comments that mention `B18MD_1` / `B35XD_F_1_(1)` alone.
They document the code format, not this event.

---

## 6. Theme — colours from the pubmat

The pubmat is navy and grass green on white. These values were **sampled
from the pubmat image's pixels** (dominant colour per region), not judged by
eye:

| Pubmat element | Sampled |
|---|---|
| "SIGHT", the banners, the footer bar | navy `#05133B` |
| Darkest navy edges | `#020B29` |
| "PICKLE" | green `#588A05` |
| Bottom strip, "₱1,500" | dark green `#3C6B02` |

### 6.1 The palette

| Token | Old (house) | New | Why |
|---|---|---|---|
| `--navy` | `#14263C` | `#05133B` | Pubmat navy |
| `--navy-deep` | `#0B1826` | `#020B29` | Pubmat's darkest navy |
| `--green` | `#7CB92C` | `#5E9106` | Pubmat green `#588A05`, lightened slightly. Navy text on the exact pubmat green is 4.35:1, which fails AA (4.5:1). On `#5E9106` it is 4.75:1 and passes. The two look the same |
| `--green-dark` | `#5C8F1F` | `#3C6B02` | Pubmat dark green |
| `--paper` | `#F6F7F2` | `#F5F6F8` | Pubmat's cool white, not the house palette's warm off-white |
| `--paper-dim` | `#ECEEE6` | `#E9ECF1` | Cool, to match |
| `--line` | `#D9DED2` | `#D3D9E3` | Cool, to match |
| `--ink` | `#14263C` | `#05133B` | Same as `--navy` |
| `--ink-soft` | `#5B6B74` | `#4A5572` | Navy-tinted grey |
| `--white`, `--radius` | — | unchanged | |

Measured contrast (WCAG): `--green` on white 3.80:1, navy on `--green`
4.75:1, `--green-dark` on white 6.37:1, `--ink` on paper 16.7:1,
`--ink-soft` on paper 6.85:1, white on navy 18.1:1.

**Not added: the pubmat's royal blue (`#0336C6`, the prize bars and the
eye) or its yellow (`#FAE304`, the CLSO logo).** The template has no role
that a second accent colour would fill, so adding one means choosing which
elements to recolour. That is a design change, not a token swap. The mascot
already carries both colours onto the page.

### 6.2 Edit the `:root` blocks

In **both** `index.html` and `schedule.html`, change the brand tokens in
the `:root{ … }` block under the `THEME` banner to the "New" column above.
Leave the role aliases in `index.html` (`--court:var(--green);` and the
rest) unchanged. They resolve through the tokens automatically.

### 6.3 Replace hard-coded copies of the old colours

Some rules in the template use the house colours as raw `rgba()`/hex instead
of `var()`. Replace every occurrence below, in the file(s) listed. Keep each
alpha value (the last number) as it is. Only the three colour channels
change.

| Find | Replace with | Files | Is |
|---|---|---|---|
| `rgba(124,185,44,` | `rgba(94,145,6,` | both | old `--green` |
| `rgba(92,143,31,` | `rgba(60,107,2,` | `index.html` | old `--green-dark` |
| `rgba(11,24,38,` | `rgba(2,11,41,` | both | old `--navy-deep` |
| `rgba(20,38,60,` | `rgba(5,19,59,` | `index.html` | old `--navy` |
| `rgba(246,247,242,` | `rgba(245,246,248,` | `index.html` | old `--paper` |
| `.badge-2{background:#4C7A19;}` | `.badge-2{background:var(--green-dark);}` | `index.html` | Its comment says it was darkened because the old `--green-dark` failed with white text. The new one passes (6.37:1) |
| `'#14263C'` (in `readableOn()`) | `'#05133B'` | `schedule.html` | navy chip text |
| `color:'#5B6B74'` (in `FALLBACK`) | `color:'#4A5572'` | `schedule.html` | old `--ink-soft` |

**Leave these alone:** `rgba(20,27,44,…)` (neutral shadow), the reds
`#B3261E`/`#FFB4A2` and their `rgba(179,38,30,…)`/`rgba(255,180,162,…)`/
`rgba(255,90,110,…)` (error states), `rgba(52,199,120,…)`, `#6C5CE0`
(`.badge-3`), every `rgba(255,255,255,…)` and `rgba(0,0,0,…)`, and the
`CAT_META` hues (§7).

### 6.4 Update the contrast note

In `index.html`'s `THEME` banner comment, replace these lines:

```
       - --green on white is ~2.3:1 and fails AA at any size. Use it as a
         background (with --navy text on top, ~5.4:1) or on a navy panel.
       - --green-dark on white is ~4.0:1 — still under the 4.5:1 body
```

and the line after them, up to and including

```
       - Small text on paper is --ink (~13.8:1) or --ink-soft (~5.9:1).
```

with:

```
       - --green on white is ~3.8:1 — large/display text only. As a fill,
         put --navy text on it (~4.75:1) or use it on a navy panel.
       - --green-dark on white is ~6.4:1 and is safe for body text.
       - Small text on paper is --ink (~16.7:1) or --ink-soft (~6.9:1).
     Pickle for Sight: palette sampled from the event pubmat (navy #05133B,
     green #588A05 lightened to #5E9106 for AA, dark green #3C6B02).
```

**Check:** each of these must print nothing:

```bash
grep -n "#14263C\|#0B1826\|#7CB92C\|#5C8F1F\|#F6F7F2\|#ECEEE6\|#D9DED2\|#5B6B74\|#4C7A19" events/pickle-for-sight-2026/*.html
grep -n "rgba(124,185,44\|rgba(92,143,31\|rgba(11,24,38\|rgba(20,38,60\|rgba(246,247,242" events/pickle-for-sight-2026/*.html
```

---

## 7. `schedule.html` — category colours

In `events/pickle-for-sight-2026/schedule.html`, replace the **entire**
`const CAT_META = { … };` object (it currently holds another event's seven
categories, including `AMD`) with:

```js
const CAT_META = {
  NWD:  { short:'N WD',  color:'#D5A6BD' },
  LIWD: { short:'LI WD', color:'#C27BA0' },
  HIWD: { short:'HI WD', color:'#741B47' },
  NMD:  { short:'N MD',  color:'#A4C2F4' },
  LIMD: { short:'LI MD', color:'#6D9EEB' },
  HIMD: { short:'HI MD', color:'#1155CC' },
  NXD:  { short:'N XD',  color:'#F9CB9C' },
  LIXD: { short:'LI XD', color:'#F6B26B' },
  HIXD: { short:'HI XD', color:'#B45F06' }
};
```

These are the same colours PNF × BUP used, from the S.A.G.E. category
ladder
([`dual-meet-schedule-generator-spec.md`](../implemented/dual-meet-schedule-generator-spec.md)
§6). The hue follows the event (WD magenta, MD blue, XD orange), and the
shade follows the level: Novice *lighter 2*, LI *lighter 1*, HI *darker 1*.
They are **not** changed by the theme in §6. Leave the comment above
`readableOn()` that mentions specific hexes as it is.

**Check:** `grep -n "AMD" events/pickle-for-sight-2026/schedule.html`
prints nothing.

---

## 8. Dry-run checklist

```bash
cp _templates/dry-run-checklist-template.md events/pickle-for-sight-2026/dry-run-checklist.md
```

Replace `{{EVENT_TITLE}}` with `Pickle for Sight` in the copy.

**Check:** `grep -rn '{{' events/pickle-for-sight-2026/` still prints
nothing.

---

## 9. Verify in a browser

Serve the **repo root**, not the event folder. The pages load
root-absolute `/assets/…` paths, which fail from a sub-folder or from
`file://`.

```bash
python -m http.server 8000
```

Run it from `sage-match-control.github.io/`. Open
`http://localhost:8000/events/pickle-for-sight-2026/` and
`http://localhost:8000/events/pickle-for-sight-2026/schedule.html`.

**Expected before the first sync:** the fetch of
`…/event-data/pickle-for-sight-2026/data/pickle-for-sight-day1.json` fails (404), so the
page shows no match data. That is correct, not a bug.

Check each of these at **375px** and **desktop** width:

- [ ] The hero reads, top to bottom: crown mark, "CLSO presents",
      "27 SEPTEMBER 2026 · PCPH, ANGELES CITY", the mascot in a white circle,
      "**PICKLE** FOR SIGHT" (with "PICKLE" green), "PLAY HARD. WIN BIG.
      RESTORE SIGHT.", then the subtitle.
- [ ] The whole mascot is visible: the paddle top and both shoes are not
      clipped, and no square edge shows inside the circle.
- [ ] The slogan wraps to at most three lines at 375px.
- [ ] The day picker is hidden. There is one day, so it auto-loads.
- [ ] The QR panel shows the image (if `qr.png` was supplied) and the text
      `tinyurl.com/SAGExPickleForSight`.
- [ ] The footer shows three lines, with the JBL line in the middle.
- [ ] The navy is visibly deeper and bluer than the house navy. The paper is
      cool white, not cream.
- [ ] The console shows no errors except the expected `pickle-for-sight-day1.json` fetch
      failure. No 404s for `/assets/…` or `assets/mascot.webp`.

Stop the server when you're done.

---

## 10. Register the event in `event-data`

In
`D:\Coding Projects\SAGE\event-data\config\events.json`, add this entry
inside `"events"`, after `"pnf-x-bup-dual-meet"`, and add a comma after the
closing brace of the entry before it:

```json
"pickle-for-sight-2026": {
  "type": "standard",
  "title": "Pickle for Sight",
  "days": {
    "pickle-for-sight-day1": {
      "label": "Sep 27",
      "date": "2026-09-27",
      "isLive": "auto",
      "facilities": [
        { "name": "PCPH Main",  "sheetId": "1rNIlK2Zz3zTaLILlpq4rCGIQbtr0ECpG4A3OdP4v25A" },
        { "name": "PCPH Annex", "sheetId": "1ks84WK7vo5FAenpPbZocm9PLPpLoS6_SRPivVJqmS-0" }
      ]
    }
  },
  "display": {
    "divisions": { "N": "Novice", "LI": "Low Intermediate", "HI": "High Intermediate" },
    "events":    { "MD": "Men's Doubles", "WD": "Women's Doubles", "XD": "Mixed Doubles" }
  }
}
```

**Before saving, confirm no other event already uses this day key:**

```bash
grep -n '"pickle-for-sight-day1"' config/events.json
```

This must print nothing. Day keys are validated as **globally unique
across every event** in this file (`SyncConfigStore.mjs`), because the day
key alone is the sync route (`POST /sync/:day`). A duplicate makes the
whole file fail validation. Cloud Run then keeps serving the last good
config, so this event never registers. If the key is taken, **stop and
ask**. Don't pick a different key yourself.

**Check:** `node -e "JSON.parse(require('fs').readFileSync('config/events.json','utf8'))"`
runs without error.

---

## 11. Done — report back

Report:
- which of §3–§9 passed,
- whether `qr.png` was present,
- whether §10 passed,
- the uncommitted files, in both repos.

---

## 12. Operator tasks — people, not the implementer

These happen in Google Sheets and on the day. They are listed so the
constraints they put on the data are written down.

### 12.1 Two workbooks, built by hand

There is no generator for standard tournaments yet
([Standard Tournament Master](../not-started/standard-tournament-master-spec.md) is not
started). Build one workbook per venue by duplicating a bkl-cup-2026
facility workbook, then:

1. Rename every category key to the nine in §7. **Use `XD`, not the
   pubmat's `MXD`.** A code like `LIMXD_1` doesn't parse, and that pair
   silently disappears from the site.
2. **Court numbers run continuously across the venues.** Main uses
   `Court 1`…`Court 4`, and the Annex uses `Court 5`…`Court 9` in both
   `CourtAssignment` and `court`. The site works out a match's venue from
   its court number, so an Annex "Court 1" would show up under Main. If the
   Annex's own signs say Court 1–5, keep that mapping for scoresheets and
   announcements only.
3. **Match numbers must not overlap between the two workbooks.** The
   schedule screen merges both and sorts by `matchNumber`. Once each
   workbook's SCHEDULE is final, run **SAGE → Fill match numbers** in it
   (re-paste the current `sheets-sync.gs` first if the menu item is
   missing). Enter **1000** in the Main workbook (1001, 1002, …) and
   **2000** in the Annex workbook (2001, 2002, …). It fills the `CSV`
   tab's `matchNumber` column with the same numbers. If it still warns
   about the `CSV` tab, set that column to the range it gives. Run it again
   after any schedule change.

   Keep both workbooks on the **same time slots** (same first time, same
   slot length) if the schedule screen will be shown with **All** venues.
   That view orders its rows by match number: Main's times come first, and
   any time found only in the Annex is added at the bottom. Each venue's
   own view (§12.4) isn't affected.
4. Keep each category's round robin and playoff in **one** workbook.
5. Name the readout tabs `CSV` and `STANDINGSCSV`, with the columns in
   runbook §4. `Schedule` values must be `h:mm AM/PM`, or the day never
   goes live automatically.
6. Colour the SCHEDULE tab with the nine `CAT_META` hexes (§7). They are
   all in Google Sheets' standard palette.
7. Clear all previous-event data. Then set up live sync (§12.2).

### 12.2 Live sync setup — once per workbook, last

Do this **after** the workbook is finished (rosters pasted, schedule
final, match numbers filled). Until setup runs, nothing publishes, so you
can edit freely before it.

**Before you start:** the `events.json` entry (§10) must be committed to
`event-data`. Setup checks the day key and venue name against the server,
so it rejects them if the event isn't registered yet. After committing,
allow about a minute for the server to pick it up.

In **each** workbook:

1. **Extensions → Apps Script.** Replace the whole script with the current
   `sage-tools-api/scripts/sheets-sync.gs`, and save. Do this even if the
   copied bkl workbook already has a script: an older copy has no **Fill
   match numbers**.
2. Reload the spreadsheet. The **SAGE** menu appears.
3. **SAGE → Set up live sync** (or **Live sync settings**, if the menu
   shows that instead). Enter:

   | Field | PCPH Main workbook | PCPH Annex workbook |
   |---|---|---|
   | Day | `pickle-for-sight-day1` | `pickle-for-sight-day1` |
   | Venue | `PCPH Main` | `PCPH Annex` |
   | Tabs to watch | `SCHEDULE` and `Court Control` (pre-ticked) | same |
   | Secret | Only if asked — see below | same |

   - **Venue is exact, including capitals.** `PCPH Main`, not `PCPH main`
     or `Main`. It must match `events.json` (§10).
   - **Both workbooks use the same day.** One day, two venues: the venue
     is what tells them apart.
   - **Secret:** a copy of a bkl workbook doesn't carry the stored secret
     over, so the dialog will probably ask for it. It is the value of
     `SYNC_SHARED_SECRET` in the Cloud Run service's environment
     variables. Get it from whoever manages Cloud Run. Never paste it into
     chat, a doc or this spec. If the dialog shows *Secret stored*
     instead, skip it.
4. **Save.** Setup checks the secret, the day and the venue, then sends
   one real test update before saving. If anything is wrong it says what
   and saves nothing. The usual cause is a venue typo, or §10 not being
   live yet.
5. Check the dialog's saved values show `pickle-for-sight-day1` and this
   workbook's own venue, **not** the bkl workbook's old day or venue.

**Confirm both are live:** after setting up both workbooks, open
`https://sage-match-control.github.io/event-data/pickle-for-sight-2026/data/pickle-for-sight-day1.json`.
Give it about a minute after the second setup. It should list **two**
facilities, `PCPH Main` and `PCPH Annex`. **SAGE → Sync now** in either
workbook re-sends immediately and reports the result.

**After setup:** edits to SCHEDULE and Court Control publish on their own,
about 10 seconds after typing stops. For a late roster or schedule fix you
don't want shown mid-edit, use **SAGE → Pause live sync**, then **Resume
live sync** and **Sync now** when done. Re-running **Fill match numbers**
doesn't publish by itself. Follow it with **Sync now**.

**SAGE → Help** in the workbook has this procedure and a troubleshooting
list, written for organizers.

### 12.3 Go-live

The site shows scores and the Live/Standings tabs from **4 hours before the
earliest scheduled match** in either workbook. It needs no configuration.
To show them earlier or later, use the Control Center's go-live override.

### 12.4 Dry run

Run the dry run against both real workbooks by **Friday 25 September**,
using `dry-run-checklist.md`, with at least one match on an Annex court.
Also check:

- The Live board shows **PCPH Main** and **PCPH Annex** heading rows, and
  the Annex match sits under Annex.
- Mission Control shows a resync row for each venue.
- The Control Center renders this event with the **standard** layout. It is
  the first registered `type: "standard"` event.
- The schedule screen draws 9 court columns, with Main on courts 1–4.
- Its **Venue** row shows All / PCPH Main / PCPH Annex. Choosing PCPH Annex
  shows courts 5–9 and only Annex matches. Bookmark
  `/events/pickle-for-sight-2026/schedule?venue=PCPH%20Main` on the Main
  screen and `…?venue=PCPH%20Annex` on the Annex screen.

---

## 13. Decisions, for reference

- **Event key `pickle-for-sight-2026`.** It is the name players see, and
  the year leaves room for a second edition.
- **Day key `pickle-for-sight-day1`.** Day keys must be globally unique
  across every event (§10), so the key carries an event prefix, per runbook
  §2 step 7. A bare `day1` would work only for the first event to claim it.
  This follows the same pattern as PNF × BUP's `pnf-x-bup-day1`.
- **Division code `N` for Novice.** The Tournament Time Calculator already
  emits `NMD`/`NWD`/`NXD`. `N`, `LI` and `HI` don't prefix each other, so
  the generated `CODE_REGEX` is unambiguous.
- **`{{VENUE}}` is `PCPH, Angeles City`.** Both venues are PCPH. The
  Main/Annex split shows where the page separates venues.
- **The mascot is inset in a circle, not cropped.** A circle crop would
  clip the paddle, which carries the CLSO mark.
- **The mascot stays `.webp`.** Every browser the site supports renders it,
  and converting it would need tooling the repo doesn't have.
- **The mascot appears on `index.html` only.** The schedule screen is a
  wall display, and the whole screen goes to the court grid.

### Template gap found (not fixed here)

`_templates/standard-tournament-template/schedule.html` ships PNF × BUP's
seven `CAT_META` entries, not a neutral example. It could ship the full
category ladder instead. That would be a separate change to `_templates/`.
