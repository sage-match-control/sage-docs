# Spec — Pickle & Friends × 1Bataan United Picklers dual meet

Instantiate `_templates/dual-meet-template/` for a one-day dual meet on
**12 September 2026** at **Pampanga Pickleball Center**.

This is an *instantiation*, not a code change: follow
`sage-match-control.github.io/_templates/CLAUDE.md`, which is the authoritative
runbook. This spec supplies the filled-in values and records the decisions
specific to this event. Where the two disagree, the runbook wins.

**All parameters are settled (§1) and all artefacts are delivered — nothing
outstanding.** §1.1 below documents where they actually ended up, since it
differs from the original plan in one respect: event-specific images (QR +
both club logos) live in a **local `assets/` subfolder inside this event's own
folder**, not directly alongside `index.html` as first assumed, and not in the
shared site-root `/assets/`.

---

## 1. Settled parameters

| Decision | Value |
|---|---|
| Event key | `pnf-x-bup-dual-meet` |
| Club codes | `PNF` (Pickle & Friends) / `BUP` (1Bataan United Picklers) |
| Mixed doubles code | `XD` — consistent with bkl-cup-2026 and the PPA dual meet, **not** the pubmat's `MXD` |
| Go-live | **12:00 noon PH** (`LIVE_GO_LIVE_HOUR_PH = 12`), three hours before first serve |
| Courts | **9**, numbered 1–9 at the single venue |
| Facility sheet ID | `1IcnsOEA5iGljEmh6Z0ChhIYn580MZ9mzpVEZY55pX0U` |
| QR asset / link | `assets/qr.png` (event-local) / `tinyurl.com/SAGExPNFxBUP` |

### 1.1 Event-local assets — delivered, at a different path than first planned

All three live in `events/pnf-x-bup-dual-meet/assets/` — a subfolder created
specifically for this event, sitting next to `index.html`:

| File | Used for |
|---|---|
| `assets/qr.png` | `{{QR_IMAGE}}` — the QR panel on `index.html` |
| `assets/pnf-logo.jpg` | Club-logos row (§2.2) — Pickle & Friends Community |
| `assets/bup-logo.jpg` | Club-logos row (§2.2) — 1Bataan United Picklers |

**Why this path and not root-absolute `/assets/`:** the site's shared
`/assets/` folder (`logo.png`, `favicons/`) holds S.A.G.E.'s own operator
branding, used by every page on the site. These three files are specific to
*this one event* — putting them there would mix per-event content into a
folder meant for platform-wide assets, and would need every future event to
pick non-colliding filenames (`qr.png` alone would already collide across
events). A plain relative path (`assets/qr.png`, no leading `/` or `../`) is
also archive-safe: moving `events/pnf-x-bup-dual-meet/` to
`events/archives/pnf-x-bup-dual-meet/` as a whole folder — including this
subfolder — preserves the relationship with no re-prefixing, same as the
root-absolute convention achieves for the shared assets (§6 in the templates
runbook).

`tinyurl.com/SAGExPNFxBUP` points at
`https://sage-match-control.github.io/events/pnf-x-bup-dual-meet/`; the QR
image encodes that short link, not the long URL.

> **The pubmat prints `MXD`; the site uses `XD`.** That divergence is
> deliberate, but it is a live trap for whoever builds the spreadsheet — a
> mixed-doubles row keyed `PNF_LIMXD_1` will silently fail to parse and the
> pair will vanish from the finder and standings. Say so explicitly when
> handing over the sheet template, and check it during §8.

Everything else is derived from the pubmat and is not in question: date, venue,
both club names, and the category list.

---

## 2. Identity and tokens

| Token | Value |
|---|---|
| `{{EVENT_KEY}}` | `pnf-x-bup-dual-meet` |
| `{{EVENT_TITLE}}` | `Dual Meet` |
| `{{EVENT_DATE_RANGE}}` | `12 September 2026` |
| `{{VENUE}}` | `Pampanga Pickleball Center` |
| `{{CLUB_A_NAME}}` | `Pickle & Friends Community` |
| `{{CLUB_B_NAME}}` | `1Bataan United Picklers` |
| `{{CLUB_A_CODE}}` | `PNF` |
| `{{CLUB_B_CODE}}` | `BUP` |
| `{{CLOUD_RUN_BASE_URL}}` | `https://sage-tools-api-811926984834.us-central1.run.app` |
| `{{EVENT_TAGLINE}}` | *(blank)* |
| `{{EVENT_HEADLINE}}` | *(blank)* |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/SAGExPNFxBUP` |

`{{EVENT_KEY}}` must satisfy `^[a-z0-9][a-z0-9-]*$` (enforced by
`SyncConfigStore`) and must be identical in three places: the folder under
`events/`, the `EVENT_KEY` constant, and the folder in the `event-data` repo.

The clubs do **not** go in `{{EVENT_TITLE}}` — settled, after a brief detour.
An earlier revision tried putting the full club names in the title, on top of
the eyebrow already rendering `Club A × Club B · date · venue` directly above
the `<h1>`; confirmed in the browser that this printed both club names twice
in a row. Reverted back to `Dual Meet`. The clubs' visual identity is now
carried by the logo row instead (§2.2), which is arguably a better fit for a
poster-style hero than a second line of text would have been anyway.

### 2.1 Hero copy

Four stacked slots, top to bottom:

| Slot | bkl-cup-2026 used | This event |
|---|---|---|
| `{{EVENT_TAGLINE}}` — small italic | "The Battle for the Court Begins!" | **blank** |
| `{{EVENT_TITLE}}` — big gradient headline | "BKL Pickleball Cup" | `Dual Meet` |
| `{{EVENT_HEADLINE}}` — large uppercase block | "6 Days. 1 Goal. Total Domination." | **blank** |
| Fixed subtitle | "Find your pair's full match schedule…" | unchanged — not a token |

Both the tagline and headline slots ended up blank. That's deliberate, not a
placeholder-replace that got forgotten: a one-night dual meet doesn't need a
slogan the way bkl's six-day tournament did, and the visual weight that would
have gone there instead goes to the club-logos row (§2.2), which sits between
the eyebrow and the title. The eyebrow still renders `Pickle & Friends
Community × 1Bataan United Picklers · 12 September 2026 · Pampanga Pickleball
Center`.

### 2.2 Club logos — custom addition, not a template token

Not part of the dual-meet template's stock feature set — `{{CLUB_A_NAME}}` /
`{{CLUB_B_NAME}}` only ever fill text (the eyebrow, `CLUBS` config). This adds
a circular-logo row, `pnf-logo.jpg` × `bup-logo.jpg` separated by `×`.

**Placement, revised once:** first sitting above the eyebrow (between it and
the blank tagline), then moved to **below the eyebrow, above the title** —
between the club names (as text) and the big "DUAL MEET" gradient headline.
Stacking order in the hero is now: crown mark → tagline (blank) → eyebrow →
**club logos** → `<h1>` title → subtitle.

**Size, also revised once:** the circles started at `clamp(52px,12vw,76px)`
and were bumped to `clamp(84px,20vw,140px)` — roughly 60% larger at both the
mobile floor and the desktop ceiling. The `×` separator and the row's `gap`
were scaled up to match (14–18px → 22–34px; 16px gap → 22px), so the
separator and spacing still read as proportionate to the now-larger circles
rather than looking left behind.

Added to **both** `index.html` and `match-control.html` (identical CSS class
+ markup in both, matching how the rest of the hero is kept in sync between
them), not to `bracket-generator.html` — that page's header is a plain
eyebrow + `<h1>` with no equivalent slot, and adding one would be a real
layout addition, not a copy-paste.

```css
.club-logos{ display:flex; align-items:center; justify-content:center; gap:22px; margin:0 0 20px; }
.club-logos img{ width:clamp(84px,20vw,140px); aspect-ratio:1/1; border-radius:50%;
  object-fit:cover; border:2px solid rgba(226,236,250,.35); box-shadow:var(--card-shadow); background:var(--paper); }
.club-logos .club-logos-sep{ font-family:'Anton',sans-serif; font-size:clamp(22px,5vw,34px); color:var(--court); opacity:.85; }
```

```html
<div class="eyebrow">…</div>
<div class="club-logos">
  <img src="assets/pnf-logo.jpg" alt="Pickle &amp; Friends Community logo" />
  <span class="club-logos-sep">&times;</span>
  <img src="assets/bup-logo.jpg" alt="1Bataan United Picklers logo" />
</div>
<h1 class="title">Dual Meet</h1>
```

`clamp()` scales the circles between mobile and desktop without a media query
— at the current values, the 140px ceiling is reached once the viewport hits
~700px wide (`20vw × 700px = 140px`). `aspect-ratio:1/1` + `object-fit:cover`
keeps them circular regardless of the source images' actual dimensions
(1024×1024 and 1254×1254 here, but the CSS doesn't depend on that). Because
this lives only in this event's two files — not in `_templates/` — it does
not carry forward to the next dual-meet event automatically. If a future
event wants the same treatment, it is a copy of this CSS+markup block, not
something the runbook currently hands you for free.

## 3. Configuration block

Single day, single venue — both of which the template already handles as
first-class cases (day picker hides and auto-loads; facility heading rows and
the per-facility resync row are suppressed).

```js
const DAYS = [
  { key: 'pnf-x-bup-day1', label: 'Sep 12', date: '2026-09-12', isLive: 'auto' }
];

const LIVE_GO_LIVE_HOUR_PH = 12;   // noon PH — 3h before first serve (§1)

const FACILITIES = [
  { name: 'Pampanga Pickleball Center', courts: [1, 9] }
];

const DIVISIONS = {
  LI: { name: 'LI', full: 'Low Intermediate' },
  HI: { name: 'HI', full: 'High Intermediate' },
  A:  { name: 'A',  full: 'Advanced' }
};

const EVENTS = {
  WD: "Women's Doubles",
  MD: "Men's Doubles",
  XD: "Mixed Doubles"
};

const DIVISION_ORDER = null;   // key order above is already the pubmat's order
const EVENT_ORDER    = null;
```

### 3.1 Advanced is men's-doubles only — no config needed

The pubmat lists Advanced with only MD. **Do not try to express that in
`DIVISIONS`/`EVENTS`.** Those two objects declare the vocabulary the code
parser understands, not the categories that exist. Displayed categories are
derived from the rows actually present in the standings data, so an Advanced
Women's Doubles column simply never appears if no such pair is entered.
Declaring the full cross-product is correct and creates nothing empty.

### 3.2 Division key order matters for parsing

`CODE_REGEX` is built by joining `Object.keys(DIVISIONS)` into an alternation,
and JavaScript preserves insertion order. `LI`, `HI`, `A` are mutually
non-prefixing, so the order above is safe. If a division is ever added whose
code is a prefix of another (e.g. adding `AB` alongside `A`), **list the longer
code first** — regex backtracking usually rescues it, but relying on that is
avoidable. The "don't hand-edit this regex" comment in the template stands;
change the order in `DIVISIONS` instead.

### 3.3 Resulting team code format

```
<CLUB>_<DIVISION><EVENT>_<REST>
```

Concretely: `PNF_LIMD_1`, `BUP_HIXD_3`, `PNF_AMD_SF_1`,
`BUP_LIWD_F_1_(2)`.

Both spreadsheets must use these exactly — the codes are the join key between
the schedule and standings tabs and everything the site renders.

## 4. Theme

The pubmat's palette is closer to the shipped default than it looks. The
template already ships bkl's lime-on-near-black scheme; the substantive change
is **orange → hot pink** for the accent colour.

```js
:root{
  --court:      #B8E830;   /* lime — pubmat court green (was #C6FF00) */
  --court-dark: #080808;   /* true black, not navy-black (was #060B16) */
  --court-line: #F2F2F2;
  --ink:        #0B1B3C;   /* unchanged — dark text on white cards */
  --paper:      #FFFFFF;
  --amber:      #FFE24D;   /* pubmat's yellow accent bars (was #FFD400) */
  --cork:       #FF3D9A;   /* HOT PINK — the one real change (was #FF7A29) */
  --muted:      #6B7690;
  --navy-mid:   #141414;   /* black-on-black gradient, no navy (was #122548) */
}
```

> **Colour values above are eyeballed from the pubmat, not sampled from source
> art.** If the club has brand hexes — particularly the pink and the lime —
> use those instead.

### 4.1 One change falls outside the `:root` block

The template's `body` background hard-codes two radial gradients in `rgba()`,
including a **blue** one (`rgba(45,110,220,.24)`) that has no counterpart in the
pubmat and will read as wrong against a true-black ground. That is the single
documented exception to "the theme block is the only place you restyle an
event" — retint that gradient to the pubmat's gold (roughly
`rgba(200,160,48,.20)`) or drop it.

The pubmat's gold halftone dot texture is **not** attempted. It could be
approximated with a repeating `radial-gradient`, but that is a bigger change to
`body` than a theme swap and risks the busy-background problem the template's
existing gradients were tuned to avoid. Out of scope unless asked for.

### 4.2 `bracket-generator.html` was missed on the first pass — fixed

The theming steps above were originally applied to `index.html` and
`match-control.html` only; `bracket-generator.html` was left on the template's
stock lime/navy/orange palette, including the blue body gradient §4.1 already
flags. Caught this by chance — a routine file-diff notification surfaced the
untouched `:root` block. Since fixed: the same six variables, the same
gradient retint, and three more spots specific to this file that the other
two pages don't have, because canvas can't read CSS custom properties —
`bracket-generator.html`'s exported-PNG drawing code duplicates the palette in
raw JS:

- `bgGrad` (the exported image's own background gradient) — `#060B16` /
  `#122548` / `#060B16` → `#080808` / `#141414` / `#080808`
- The header/title `ctx.fillStyle` calls — `#FFD400` → `#FFE24D`,
  `#C6FF00` → `#B8E830`
- `BADGE_FILL` / `BADGE_TEXT` (the bracket-slot colour badges baked into the
  exported image) — cork/court/amber entries updated to match; the fourth
  badge colour (`#8B7CF6`, purple) isn't part of this theme and is unchanged
- The canvas `glow()` call's raw `rgba(198,255,0,...)` → `rgba(184,232,48,...)`

**If this event's templates are ever re-themed again, re-check all three
files, not just the two with visible on-page styles** — the export canvas is
easy to forget precisely because you don't see it until you click "Export as
image."

### 4.3 Found, not fixed: a wider hardcoded-glow pattern across all three files

While chasing §4.2's gap, found that `rgba(198,255,0,…)` — raw lime, matching
the *stock* `--court` value — appears throughout the CSS in all three files
(not just bracket-generator) in glow/shadow/hover effects that were never
wired to the `--court` variable in the original bkl template: the crown mark's
drop-shadow, the `.regal-rule` divider gradient, the `h1.title` gradient-text
fill (`#8FE94B`/`#C6FF00`/`#F5FF8C`, also unthemed), border hovers, pulse
animations. A dozen-plus occurrences across `index.html`, `match-control.html`,
and `bracket-generator.html`.

**Deliberately not fixed as part of this change.** It's a template-level gap
(present in `_templates/dual-meet-template/` itself, inherited from bkl, not
introduced by this instantiation), it's low-opacity decorative glow rather
than a solid brand colour, and fixing it properly means either a wider sweep
across all three files here or — better — teaching `_templates/` to reference
`var(--court)` in these spots so every future event gets it for free instead
of every instantiation re-discovering the same gap. Flagging for a decision,
not fixing silently.

## 5. Backend

Add to `event-data`'s `config/events.json` (a commit — **no Cloud Run
redeploy**, per the runtime sync config):

```json
"pnf-x-bup-dual-meet": {
  "days": {
    "pnf-x-bup-day1": {
      "label": "Sep 12",
      "facilities": [
        { "name": "Pampanga Pickleball Center", "sheetId": "1IcnsOEA5iGljEmh6Z0ChhIYn580MZ9mzpVEZY55pX0U" }
      ]
    }
  }
}
```

- The day key must be **globally unique across every event** in that file.
  `bkl-cup-2026` already holds `day2`–`day6`, so the prefixed key above is
  required, not stylistic.
- The facility `name` must match the `FACILITIES` entry in §3 **exactly** —
  it is compared case-sensitively.
- Only add `matchesSheetName` / `standingsSheetName` overrides if this
  event's sheet tabs are literally named something other than `CSV` /
  `STANDINGSCSV`. This event's are correctly named, so no override is
  needed — **this day's actual investigation is why GID-based overrides
  (`csvGid` / `standingsGid`) don't exist any more.** This sheet's
  `STANDINGSCSV` tab sits at GID `1801965166` because the workbook was
  duplicated from the archived `ppa-x-club-2600-dual-meet` sheet and
  inherited its old tabs' GIDs — the shared default GID silently resolved
  to a leftover tab from that old event instead. Both fetch paths in
  `sage-tools-api` now address tabs by name only; see
  `event-data/config/README.md`.

Then install `scripts/sheets-sync.gs` on the facility spreadsheet with
`DAY_KEY = 'pnf-x-bup-day1'`, `FACILITY_NAME = 'Pampanga Pickleball
Center'`, and the Cloud Run URL from §2 — and **check `WATCHED_SHEET_GIDS`
against this spreadsheet's own tab GIDs**, which will not match the shipped
defaults.

## 6. Steps

Per `_templates/CLAUDE.md` §2, with this event's specifics. Steps 1–6 are
pure file work and are done; 7–8 touch live systems.

1. `cp -r _templates/dual-meet-template events/pnf-x-bup-dual-meet/`
2. Replace every token (§2). `grep -r '{{' events/pnf-x-bup-dual-meet/` comes
   back empty
3. Fill the config block (§3) in **both** `index.html` and `match-control.html`
4. Apply the theme (§4) to **all three files**, including
   `bracket-generator.html` (§4.2 — easy to miss, since it has no on-page
   visual until you export)
5. Create `events/pnf-x-bup-dual-meet/assets/` and drop in `qr.png`,
   `pnf-logo.jpg`, `bup-logo.jpg` (§1.1); add the club-logos block (§2.2) to
   `index.html` and `match-control.html`
6. Verify the spreadsheet's columns (§7) **before** wiring anything up
7. Commit `config/events.json` in `event-data` (§5); confirm
   `pnf-x-bup-day1` appears in `GET /sync/config` before continuing
8. Install the Apps Script on the facility spreadsheet (§5), then trigger a
   test sync and confirm it writes
   `pnf-x-bup-dual-meet/data/pnf-x-bup-day1.json`

`bracket-generator.html` needs no configuration — copy as-is.

The `data/` folder in `event-data` does not need creating by hand; the first
successful sync creates it.

## 7. Spreadsheet columns — verify before anything else

The most common cause of a new event rendering blank. Exact, case-sensitive:

- **Matches tab**: `matchNumber`, `teamCode1`, `team1Player1`, `team1Player2`,
  `teamCode2`, `team2Player1`, `team2Player2`, `Schedule`, `team1Score`,
  `team2Score`, `CourtAssignment`, `court`
- **Standings tab**: `teamCode`, `player1`, `player2`, `wins`, `loss`,
  `quotient`, `bracket`

`court` is the **live** court and is distinct from the scheduled
`CourtAssignment`. Without it the Live Matches tab reads "No matches are
currently on court." indefinitely, no matter how much other data is correct.

## 8. Acceptance

- [ ] `grep -r '{{' events/pnf-x-bup-dual-meet/` returns nothing.
- [ ] No `bkl`, `Pampanga Paddle Aces`, `Club 2600`, or other prior-event
      string survives anywhere in the folder.
- [ ] Day picker is **hidden** and the single day auto-loads (§3).
- [ ] Facility heading row on the Live board and the per-facility resync row in
      Mission Control are both **suppressed** (single facility).
- [ ] Standings show the club-vs-club win summary with the leader highlighted;
      desktop shows category columns with both clubs stacked inside, mobile
      shows one column per club.
- [ ] A pair with the same two names entered on both clubs is not merged in the
      autocomplete.
- [ ] Before the go-live threshold, `index.html` hides scores and the
      Live/Standings tabs; `match-control.html` shows them regardless.
- [ ] Mission Control's connection check reaches `/ping` and reports a config
      SHA; a resync writes `pnf-x-bup-dual-meet/data/pnf-x-bup-day1.json`.
- [x] Shared-site asset paths (favicons, `/assets/logo.png`) are root-absolute
      and icons resolve. Event-local assets (`assets/qr.png`, both club logos)
      are relative on purpose (§1.1) — don't "fix" them to root-absolute.
      Verified in the browser, real HTTP server: no 404s except the expected
      external-fetch ones (unpublished snapshot data), across all three pages.
- [x] QR panel renders the real image (not a broken-image icon) and the
      printed text reads `tinyurl.com/SAGExPNFxBUP`. Verified.
- [x] Club-logos row renders as two circles separated by `×`, on both
      `index.html` and `match-control.html`, at both mobile and desktop
      widths (`clamp()`-based sizing, no media query). Verified.
- [x] `bracket-generator.html`'s theme matches the other two pages — background,
      title colour, eyebrow colour. Verified after §4.2's fix; was failing
      before it.
- [ ] Mobile (≤980px) and desktop standings layouts both render.

## 9. Out of scope

- Generating the QR PNG or creating the tinyurl (§1.1 — values are fixed, the artefacts are a separate errand).
- The gold halftone texture from the pubmat (§4.1).
- Any change to `_templates/` itself. If something here needs a template edit
  rather than a config value, stop and treat that as a separate change — the
  templates serve future events too.
- Archiving. That happens after the event; because asset paths are
  root-absolute, the folder moves into `events/archives/` with no re-prefixing.
