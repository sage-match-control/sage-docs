# Spec — Piggleball Chairman's Cup

> **Status: implemented.** The event ran on Saturday 3 October 2026. The
> site (`events/piggleball-2026/`: `index.html`, `schedule.html`,
> `dry-run-checklist.md` and the attendance desk page), its registry entry
> (§11) and the QR image are built, and live sync ran from the second
> workbook (§1) from 07:25 to 16:07, publishing 238 syncs with live push on.
> The rosters were pasted in and `isLive` was back on `"auto"` for the day.
> All 68 matches were published, the facility was stamped complete at 16:07,
> and the event's pages now have live push off (`LIVE_BASE_URL = ''`), as
> every finished event's do. Sync timings for the day are in
> [Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026).
>
> §13.1's IXD bronze (item 3) went as a walkover: the organiser scheduled it
> as match #100, `IXD_B_1` against `IXD_B_2`, with the absent side's names
> entered as `-` and a score of 11–0, not as a BYE. The Awards tab reads that
> as an ordinary win; see the [Awards tab](awards-podium-tab-spec.md)'s §8.

Build the event site for the **1st Piggleball Chairman's Cup**, a one-day
open-entry pickleball tournament on **Saturday, 3 October 2026** at
**Centro Atletico, Cubao, Quezon City**. The National Federation of Hog
Farmers, Inc. (NFHFI, "NATFED") presents it as part of its **Pig Sports
Festival**.

The festival also includes a badminton tournament, the 23rd Pigminton
Chairman's Cup. **That tournament is out of scope.** The site covers only
the pickleball tournament and never mentions the badminton one.

The site is an instance of `_templates/standard-tournament-template/`. The
runbook for instantiating a template is
`sage-match-control.github.io/_templates/CLAUDE.md`. This spec gives every
value and every edit for this event, so you should not need to make any
design decisions. If this spec and the runbook disagree, **stop and ask**.
Don't pick one.

---

## 0. How to use this spec

- **Do sections 3–11 in order.** Each section lists exact edits, then a
  check to run. Don't move on until the check passes.
- **Where you work:** `D:\Personal\SAGE\sage-match-control.github.io` (the
  site repo) for §3–§10. §11 edits `D:\Personal\SAGE\event-data`. All paths
  in §3–§10 are relative to the site repo.
- **Find edits by their anchor text, not line numbers.** Every edit quotes
  the exact text to find. The template changes over time, so line numbers
  would drift.
- **Use a shell with `grep`** (Git Bash) for the checks. Python is not
  installed on this machine. Node is.
- **Don't commit or push** unless the user asks.
- **Don't edit anything under `_templates/`.** If a template edit seems
  necessary, stop and report it instead.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

---

## 1. Event facts

| Fact | Value |
|---|---|
| Presenter | National Federation of Hog Farmers, Inc. (NFHFI), called "NATFED" on the pubmat |
| Event name | 1st Piggleball Chairman's Cup (pickleball tournament) |
| Festival | Pig Sports Festival. The site mentions it once, in the hero tagline |
| Date | Saturday, 3 October 2026 — one day |
| Venue | Centro Atletico, West Road, Cubao, Quezon City. One facility, 3 courts |
| Divisions | Novice, Intermediate |
| Categories | Novice Open Doubles (`ND`, 10 pairs), Intermediate Men's Doubles (`IMD`, 9 pairs), Intermediate Mixed Doubles (`IXD`, 5 pairs) |
| Matches | 56, numbered 1–56, first serve 9:00 AM, 25-minute slots, last slot 4:30 PM |
| Slogan | "Let's rally and dink for fun, camaraderie and a stronger pig industry!" |
| Closing line | "One Pig Industry. Stronger Together!" |
| Beneficiary | Proceeds go to scholarship programs |
| Short link | `tinyurl.com/SAGExPiggleball` → `https://sage-match-control.github.io/events/piggleball-2026/` |

**The site shows none of these:** the badminton tournament, the four
contact names and phone numbers, the pubmat's "8:00 AM onwards" (the
workbook's first match is 9:00 AM), and the four pubmat icons (friendships,
healthy living, industry, scholarships). The site is a live results hub for
players who have already entered.

**Same day as PickleDrive.** `pickledrive-anniversary-2026` runs on 3 October
2026 too, at Kingcourts. The two events have separate keys, pages and
workbooks, and never share data. Nothing in this spec touches PickleDrive's
files.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| NFHFI logo | **Staged** at `events/piggleball-2026/assets/logo.webp` (960×958, white background). Use it as is: no conversion, no cropping | §4 `{{EVENT_LOGO}}` |
| Pubmat | **Staged** at `events/piggleball-2026/assets/pubmat.webp` (1736×2000). Reference for §6–§7 only. No page loads it | — |
| QR image for the short link | **To follow from the user** as `events/piggleball-2026/assets/qr.png`. The short link itself, `tinyurl.com/SAGExPiggleball`, is confirmed. If the image isn't there yet, do everything else and report it missing. Don't generate one | Template QR panel |
| Event workbook | `1QAa2FlH2uiY0JWJNSpe5JGuwlWYE544_UPfYYHnDJmU` (replaced `1PLbtKQWYCEejy0a9EWFbdx51BDss6G_v1k-kdCKA4zo` and, before that, `1zdmfKXpz9jrNLSU--3h-GhEueuSRH6IoSmtgfaliveM`, on 1 October 2026; that one had itself replaced the first workbook, `1QX38GVquZjsha24BLJkx7tTWxc09sc6oWW99ALUsegE`, on 30 September 2026) | §11 |

---

## 3. Create the event folder

The folder already exists, because it holds the staged images. Copy the
template's **contents** into it. The trailing `/.` matters: without it,
`cp` would create a nested `standard-tournament-template/` folder inside
the existing one.

```bash
cp -r _templates/standard-tournament-template/. events/piggleball-2026/
```

**Check:** `ls events/piggleball-2026` shows `index.html`, `schedule.html`
and `assets/`, and `assets/` still contains `logo.webp` and `pubmat.webp`.

---

## 4. Replace the `{{TOKENS}}`

Replace every occurrence in both `events/piggleball-2026/index.html` and
`events/piggleball-2026/schedule.html`:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `piggleball-2026` |
| `{{EVENT_TITLE}}` | `Piggleball Chairman's Cup` |
| `{{EVENT_TAGLINE}}` | `NATFED presents · Pig Sports Festival` |
| `{{EVENT_HEADLINE}}` | `1st Chairman's Cup` |
| `{{EVENT_DATE_RANGE}}` | `3 October 2026` |
| `{{VENUE}}` | `Centro Atletico, Cubao` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/SAGExPiggleball` |
| `{{EVENT_LOGO}}` | `assets/logo.webp` |
| `{{SCHEDULE_DAY_KEY}}` | `piggleball-day1` |

- The apostrophe in `Chairman's` is a plain ASCII `'`. It is safe
  everywhere these tokens land: HTML text, double-quoted attributes, and
  one JavaScript **backtick** template literal in `schedule.html`
  (`print-title`). No token lands in a single-quoted JS string.
- The `·` in the tagline is the literal middle-dot character (U+00B7). The
  files are UTF-8, so type it as is, not as `&middot;`.
- None of these values contain `&`, so no HTML-entity escaping is needed.

**Check:** this must print nothing:

```bash
grep -rn '{{' events/piggleball-2026/
```

---

## 5. `index.html` — content edits

All edits are in `events/piggleball-2026/index.html`.

### 5.1 Configuration block

Inside the `CONFIGURATION` section of the `<script>`, replace each of these
four `// EXAMPLE — replace` blocks completely, including the
`// EXAMPLE — replace` comment line.

`DAYS`: find

```js
const DAYS = [
  // EXAMPLE — replace
  { key: 'piggleball-2026-day1', label: 'Day 1', date: '2026-01-01' }
];
```

(the token replacement in §4 has already filled in the key). Replace it
with:

```js
const DAYS = [
  { key: 'piggleball-day1', label: 'Oct 3', date: '2026-10-03' }
];
```

`FACILITIES`: replace the whole array with:

```js
const FACILITIES = [
  { name: 'Centro Atletico', courts: [1, 3] }
];
```

`DIVISIONS`: replace the whole object with:

```js
const DIVISIONS = {
  N: { name: 'N', full: 'Novice' },
  I: { name: 'I', full: 'Intermediate' }
};
```

`EVENTS`: replace the whole object with:

```js
const EVENTS = {
  D:  'Open Doubles',
  MD: "Men's Doubles",
  XD: 'Mixed Doubles'
};
```

Leave `DIVISION_ORDER`, `EVENT_ORDER`, `CODE_REGEX`, `GO_LIVE_LEAD_HOURS`,
`STAGE_META` and everything else in the block unchanged.

**Why a one-letter event code works.** `CODE_REGEX` is built from these keys
as `^(N|I)(D|MD|XD)_(.+)$`. `ND_1` splits to `N` + `D`. `IMD_1` can't split
as `I` + `D`, because the character after `I` is `M`, so it splits to
`I` + `MD`. No division key is a prefix of another, and no code is
ambiguous.

**Check:** `grep -n "EXAMPLE" events/piggleball-2026/index.html` prints
nothing. Then run this from the site repo root. It must print
`N D | I MD | I XD | N D | I XD`:

```bash
node -e "const re=/^(N|I)(D|MD|XD)_(.+)$/;console.log(['ND_1','IMD_SF_2','IXD_F_1_(1)','ND_B_1','IXD_5'].map(c=>{const m=c.match(re);return m[1]+' '+m[2]}).join(' | '))"
```

### 5.2 Hero — title

Find (as it reads after §4):

```html
<h1 class="title">Piggleball Chairman's Cup</h1>
```

Replace with:

```html
<h1 class="title"><span class="accent">Pig</span>gleball</h1>
```

The headline line directly below it already reads "1st Chairman's Cup"
from §4, so the two lines together say "PIGGLEBALL / 1ST CHAIRMAN'S CUP",
as on the pubmat.

### 5.3 Hero — slogan

Find:

```html
<p class="subtitle">Find your pair's full match schedule — every round, every opponent, in order.</p>
```

Directly **above** it, add:

```html
<div class="slogan-script">Let's rally and dink for fun, camaraderie and a stronger pig industry!</div>
```

The hero now stacks: NFHFI logo × S.A.G.E. logo → "NATFED PRESENTS · PIG
SPORTS FESTIVAL" → "3 OCTOBER 2026 · CENTRO ATLETICO, CUBAO" → "**PIG**GLEBALL"
→ "★ 1ST CHAIRMAN'S CUP ★" → slogan in script → subtitle. The CSS for the
new line and the stars is in §7.

### 5.4 Footer cause line

Find (as it reads after §4):

```html
<footer>
  Piggleball Chairman's Cup &middot; 3 October 2026 &middot; Centro Atletico, Cubao<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

Replace with:

```html
<footer>
  Piggleball Chairman's Cup &middot; 3 October 2026 &middot; Centro Atletico, Cubao<br>
  One Pig Industry. Stronger Together! Proceeds support scholarship programs.<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

### 5.5 Example text left over from the template

Find:

```html
placeholder="e.g. Beginner 18+ Men's Doubles"
```

Replace with:

```html
placeholder="e.g. Intermediate Men's Doubles"
```

Leave the code comments that mention `B18MD_1` / `B35XD_F_1_(1)` alone.
They document the code format, not this event.

---

## 6. Theme — colours from the pubmat

The pubmat is navy and red on white, with yellow highlights (the Philippine
flag's colours). The values below were **sampled from the pubmat's pixels**
(the dominant colour of each region), not judged by eye:

| Pubmat element | Sampled |
|---|---|
| "SPORTS", the date/time/venue banner | navy `#022058` |
| "PIG", "Festival", the "PICKLEBALL TOURNAMENT" bar | red `#D40202` |
| The pickleball, "STRONGER TOGETHER!" | yellow `#FBAB04` |
| Page background | cool white `#F2F9FE` |

### 6.1 The palette

The token **names** stay the same (`--navy`, `--green`, …), so every rule
in the template keeps resolving. Only their values change. The template's
`--green` is its accent *fill*, used on navy panels and under navy text, and
`--green-dark` is its accent for *text and fills on light backgrounds* and
for the "live" state. Yellow fills the first role and red the second, so
"green" here no longer means green.

| Token | Old (house) | New | Why |
|---|---|---|---|
| `--navy` | `#14263C` | `#022058` | Pubmat navy |
| `--navy-deep` | `#0B1826` | `#01153D` | Navy darkened for shadows. The pubmat has no darker navy |
| `--green` | `#7CB92C` | `#FBAB04` | Pubmat yellow — accent **fill** |
| `--green-dark` | `#5C8F1F` | `#D40202` | Pubmat red — accent **text**, live dot, live pill |
| `--paper` | `#F6F7F2` | `#F4F8FC` | Pubmat's cool white |
| `--paper-dim` | `#ECEEE6` | `#E8EEF6` | Cool, to match |
| `--line` | `#D9DED2` | `#D2DBE8` | Cool, to match |
| `--ink` | `#14263C` | `#022058` | Same as `--navy` |
| `--ink-soft` | `#5B6B74` | `#4A5572` | Navy-tinted grey |
| `--white`, `--radius` | — | unchanged | |

Measured contrast (WCAG 2.x):

| Pair | Ratio | Use |
|---|---|---|
| `--ink` on `--paper` | 14.58 | body text |
| `--ink-soft` on `--paper` / `--paper-dim` | 6.95 / 6.35 | secondary text |
| `--green-dark` (red) on white / `--paper` / `--paper-dim` | 5.51 / 5.17 / 4.72 | accent text, stat numbers |
| white on `--green-dark` (red) | 5.51 | live pill, `.badge-2` |
| `--navy` text on `--green` (yellow) | 8.09 | active tabs, chips |
| `--green` (yellow) on `--navy` / `--navy-deep` | 8.09 / 9.29 | eyebrow, headline in the hero |
| white on `--navy` | 15.55 | hero text |
| `--green` (yellow) on white | **1.92 — fails** | never use `--green` as text on a light background |
| red on `--navy` | **2.82 — fails** | never put red text directly on navy. §7.3 gives the title's red "PIG" a white outline for this reason |

Every rule in the template that uses `--green`/`--court` as a **text**
colour sits on a navy panel (checked for this spec), so the yellow is safe
everywhere it lands.

### 6.2 Edit the `:root` blocks

In **both** `index.html` and `schedule.html`, change the brand tokens in
the `:root{ … }` block under the `THEME` banner to the "New" column above.
Leave the role aliases in `index.html` (`--court:var(--green);` and the
rest) unchanged. They resolve through the tokens automatically.

### 6.3 Replace hard-coded copies of the old colours

Some rules use the house colours as raw `rgba()`/hex instead of `var()`.
Replace every occurrence below, in every file that contains it. Keep each
alpha value (the last number) as it is. Only the three colour channels
change.

| Find | Replace with | Files | Was |
|---|---|---|---|
| `rgba(124,185,44,` | `rgba(251,171,4,` | both | old `--green` |
| `rgba(92,143,31,` | `rgba(212,2,2,` | `index.html` | old `--green-dark` (the live dot's pulse) |
| `rgba(11,24,38,` | `rgba(1,21,61,` | both | old `--navy-deep` |
| `rgba(20,38,60,` | `rgba(2,32,88,` | `index.html` | old `--navy` |
| `rgba(246,247,242,` | `rgba(244,248,252,` | `index.html` | old `--paper` |
| `.badge-2{background:#4C7A19;}` | `.badge-2{background:var(--green-dark);}` | `index.html` | Its comment says it was darkened because the old `--green-dark` failed with white text. The red passes (5.51:1) |
| `'#14263C'` (in `readableOn()`) | `'#022058'` | `schedule.html` | navy chip text |
| `color:'#5B6B74'` (in `FALLBACK`) | `color:'#4A5572'` | `schedule.html` | old `--ink-soft` |

**Leave these alone:** `rgba(20,27,44,…)` (neutral shadow), the error reds
`#B3261E`/`#FFB4A2` and their `rgba(179,38,30,…)`/`rgba(255,180,162,…)`/
`rgba(255,90,110,…)`, `rgba(52,199,120,…)`, `#6C5CE0` (`.badge-3`), every
`rgba(255,255,255,…)` and `rgba(0,0,0,…)`, and the `CAT_META` hues (§8).

### 6.4 Update the contrast note

In `index.html`'s `THEME` banner comment, replace these lines:

```
       - --green on white is ~2.3:1 and fails AA at any size. Use it as a
         background (with --navy text on top, ~5.4:1) or on a navy panel.
       - --green-dark on white is ~4.0:1 — still under the 4.5:1 body
         floor, so keep it for large/display text only.
       - Small text on paper is --ink (~13.8:1) or --ink-soft (~5.9:1).
```

with:

```
       - --green is a YELLOW fill (~1.9:1 on white — never text on a light
         background). Put --navy text on it (~8.1:1), or use it as text on
         a --navy/--navy-deep panel (~8.1 / 9.3:1).
       - --green-dark is RED. Safe for text on white/--paper (~5.5 / 5.2:1)
         and under white text (~5.5:1). Never as text on navy (~2.8:1).
       - Small text on paper is --ink (~14.6:1) or --ink-soft (~7.0:1).
     Piggleball Chairman's Cup: palette sampled from the event pubmat
     (navy #022058, red #D40202, yellow #FBAB04). Fonts: Archivo italic
     for display headings, Kaushan Script for the hero slogan.
```

**Check:** each of these must print nothing:

```bash
grep -n "#14263C\|#0B1826\|#7CB92C\|#5C8F1F\|#F6F7F2\|#ECEEE6\|#D9DED2\|#5B6B74\|#4C7A19" events/piggleball-2026/*.html
grep -n "rgba(124,185,44\|rgba(92,143,31\|rgba(11,24,38\|rgba(20,38,60\|rgba(246,247,242" events/piggleball-2026/*.html
```

---

## 7. Fonts — from the pubmat

The pubmat's faces and their nearest Google Fonts:

| Pubmat | Google Font | Role on the site |
|---|---|---|
| "PIG SPORTS", "PIGGLEBALL" — heavy italic grotesque | **Archivo** italic 800/900 | Hero title, headline and section headings only |
| "CHAIRMAN'S CUP", "PICKLEBALL TOURNAMENT", the banners — bold condensed italic caps | **Barlow Condensed** italic 700 (already loaded upright) | Hero tagline and eyebrow |
| "Festival", "Let's rally and dink for fun…" — brush script | **Kaushan Script** | Hero slogan only |

Everything else keeps the template's fonts: Archivo Black (upright) for
scores and numbers, Barlow Condensed (upright) for labels and tabs, Inter
for body text. They are close to the pubmat already, and upright figures
are easier to read in score tables. **Archivo** (the family with italics)
and **Archivo Black** (the template's single-weight upright face) are two
different Google families. Both are loaded, and both names appear in the
CSS.

### 7.1 `index.html` — Google Fonts link

Find:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

Replace with:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo:ital,wght@1,800;1,900&family=Archivo+Black&family=Barlow+Condensed:ital,wght@0,400;0,500;0,600;0,700;1,700&family=Inter:wght@400;500;600;700&family=Kaushan+Script&display=swap" rel="stylesheet">
```

### 7.2 `index.html` — heading rules

In each rule below, replace only the lines shown. Leave its other
declarations (size, colour, spacing, margins) as they are.

| Rule | Replace | With |
|---|---|---|
| `h1.title{` | `font-family:'Archivo Black',sans-serif;` and `font-weight:400;` | `font-family:'Archivo',sans-serif;` `font-style:italic;` `font-weight:900;` |
| `.domination-line{` | `font-family:'Archivo Black',sans-serif;` | `font-family:'Archivo',sans-serif;` `font-style:italic;` `font-weight:900;` |
| `.results-head h2{` | `font-family:'Archivo Black',sans-serif;` and `font-weight:400;` | `font-family:'Archivo',sans-serif;` `font-style:italic;` `font-weight:800;` |
| `.standings-col-head .cat-title{` | same as above | same as above |
| `.stage-banner .stage-title{` | same as above | same as above |
| `.eyebrow{` | `font-weight:600;` | `font-weight:700;` `font-style:italic;` |
| `.tagline-script{` | `font-family:'Inter',sans-serif;` | `font-family:'Barlow Condensed',sans-serif;` `text-transform:uppercase;` |

In `.tagline-script`, also change `letter-spacing:.01em;` to
`letter-spacing:.14em;`. It keeps `font-style:italic;` and
`font-weight:700;`.

### 7.3 `index.html` — title accent, stars and slogan

Find this line:

```css
  h1.title .accent{ color:var(--green); }
```

Replace it with:

```css
  /* Piggleball: "PIG" is red with a white outline, as on the pubmat. Red on
     navy alone is ~2.8:1, so the outline is what makes it readable. It is
     drawn with eight text-shadows instead of -webkit-text-stroke, because a
     stroke eats into the glyphs and thins the italic. */
  h1.title .accent{
    color:var(--green-dark);
    text-shadow:
      -2px -2px 0 var(--white),  2px -2px 0 var(--white),
      -2px  2px 0 var(--white),  2px  2px 0 var(--white),
       0   -2px 0 var(--white),  0    2px 0 var(--white),
      -2px  0   0 var(--white),  2px  0   0 var(--white);
  }
  /* Stars either side of the headline, as around "CHAIRMAN'S CUP" on the
     pubmat. The second `content` gives them empty alt text, so screen
     readers skip them. A browser that doesn't know that syntax keeps the
     first line. */
  .domination-line::before{ content:"★ "; content:"★ " / ""; color:var(--white); }
  .domination-line::after { content:" ★"; content:" ★" / ""; color:var(--white); }
  /* The pubmat's brush-script slogan. White, not red: red on navy fails. */
  .slogan-script{
    font-family:'Kaushan Script',cursive;
    font-size:clamp(17px,4.4vw,22px);
    line-height:1.35;
    color:var(--white);
    max-width:520px;
    margin:4px auto 16px;
  }
```

### 7.4 `schedule.html` — fonts

Find:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

Replace with:

```html
<link href="https://fonts.googleapis.com/css2?family=Archivo:ital,wght@1,900&family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

In the `.bezel h1{` rule, replace `font-family:'Archivo Black',sans-serif;`
with `font-family:'Archivo',sans-serif; font-style:italic;`, and on its next
line replace `font-weight:400;` with `font-weight:900;`. Change nothing else
on the board: it is a wall display, and upright type reads better across a
room.

**Check:** both must print a match:

```bash
grep -n "Kaushan+Script" events/piggleball-2026/index.html
grep -n "family=Archivo:ital" events/piggleball-2026/schedule.html
```

---

## 8. `schedule.html` — category colours

These are the **organiser's** colours: the category fills on the
workbook's own SCHEDULE tab, read from the sheet's HTML export for this
spec. They are all in Google Sheets' standard palette.

In `events/piggleball-2026/schedule.html`, replace the **entire**
`const CAT_META = { … };` object (it currently holds another event's seven
categories, including `AMD`) with:

```js
const CAT_META = {
  ND:  { short:'NOVICE', color:'#666666' },
  IMD: { short:'INT MD', color:'#3C78D8' },
  IXD: { short:'INT XD', color:'#E69138' }
};
```

The keys are exactly the category prefix of each team code (`ND_1` →
`ND`), which is what the board looks them up by. `readableOn()` picks the
chip text: white on `ND` and `IMD`, navy on `IXD`.

Then find the comment line:

```js
// The palette spans very light (LI XD #F6B26B) to very dark (HI WD #741B47),
```

and replace it with:

```js
// The palette spans light (IXD #E69138) to dark (ND #666666),
```

**Check:** `grep -n "AMD\|LIWD\|HIXD" events/piggleball-2026/schedule.html`
prints nothing.

---

## 9. Dry-run checklist

```bash
cp _templates/dry-run-checklist-template.md events/piggleball-2026/dry-run-checklist.md
```

Replace `{{EVENT_TITLE}}` with `Piggleball Chairman's Cup` in the copy.

**Check:** `grep -rn '{{' events/piggleball-2026/` still prints nothing.

---

## 10. Verify in a browser

Serve the **repo root**, not the event folder. The pages load
root-absolute `/assets/…` paths, which fail from a sub-folder or from
`file://`. From `sage-match-control.github.io/`:

```bash
npx --yes http-server -p 8000 -c-1
```

Open `http://localhost:8000/events/piggleball-2026/` and
`http://localhost:8000/events/piggleball-2026/schedule.html`.

**Expected before the first sync:** the fetch of
`…/event-data/piggleball-2026/data/piggleball-day1.json` fails (404), so
the page shows no match data. That is correct, not a bug.

Check each of these at **375px** and **desktop** width:

- [ ] The hero reads, top to bottom: the NFHFI logo × the S.A.G.E. logo,
      "NATFED PRESENTS · PIG SPORTS FESTIVAL" (italic condensed caps),
      "3 OCTOBER 2026 · CENTRO ATLETICO, CUBAO" (yellow), "**PIG**GLEBALL"
      (italic, "PIG" red with a white outline, the rest white),
      "★ 1ST CHAIRMAN'S CUP ★" (yellow, white stars), the slogan in brush
      script, then the subtitle.
- [ ] "PIGGLEBALL" fits on one line at 375px.
- [ ] The slogan wraps to at most three lines at 375px.
- [ ] The NFHFI logo shows whole inside its white tile. Nothing is cropped.
- [ ] The day picker is hidden. There is one day, so it auto-loads.
- [ ] Desktop only: the QR panel shows the image (if `qr.png` was supplied)
      and the text `tinyurl.com/SAGExPiggleball`.
- [ ] The footer shows three lines, with the "One Pig Industry" line in the
      middle.
- [ ] The navy is deeper and bluer than the house navy. The paper is cool
      white, not cream. Nothing on the page is green.
- [ ] DevTools → Network: `Archivo`, `Kaushan Script` and the other fonts
      load without errors. The hero title is not a synthesised (slanted
      Archivo Black) italic: its letterforms are narrower than the upright
      score numbers.
- [ ] The schedule board's bezel title reads "Piggleball Chairman's Cup" in
      italic, on the new navy.
- [ ] The console shows no errors except the expected
      `piggleball-day1.json` fetch failure. No 404s for `/assets/…` or
      `assets/logo.webp`.

Stop the server when you're done.

---

## 11. Register the event in `event-data`

In `D:\Personal\SAGE\event-data\config\events.json`, add this entry inside
`"events"`, **after** `"pickledrive-anniversary-2026"` (currently the last
entry), and add a comma after the closing brace of that entry:

```json
"piggleball-2026": {
  "type": "standard",
  "title": "Piggleball Chairman's Cup",
  "days": {
    "piggleball-day1": {
      "label": "Oct 3",
      "date": "2026-10-03",
      "isLive": "auto",
      "facilities": [
        { "name": "Centro Atletico", "sheetId": "1QAa2FlH2uiY0JWJNSpe5JGuwlWYE544_UPfYYHnDJmU" }
      ]
    }
  },
  "display": {
    "divisions": { "N": "Novice", "I": "Intermediate" },
    "events":    { "D": "Open Doubles", "MD": "Men's Doubles", "XD": "Mixed Doubles" }
  }
}
```

Control Center splits a category like `IMD` by matching the longest
division key first, then looking the remainder up in `events`. `N` and `I`
are both one letter and neither prefixes the other, so `ND` → Novice +
Open Doubles and `IMD` → Intermediate + Men's Doubles. Key order sets
display order: Novice first.

**Before saving, confirm no other event already uses this day key:**

```bash
grep -n '"piggleball-day1"' config/events.json
```

This must print nothing. Day keys are validated as **globally unique
across every event** in this file (`SyncConfigStore.mjs`), because the day
key alone is the sync route (`POST /sync/:day`). A duplicate makes the
whole file fail validation. Cloud Run then keeps serving the last good
config, so this event never registers. If the key is taken, **stop and
ask**. Don't pick a different key yourself.

**Check:** run from `event-data/`:

```bash
node -e "JSON.parse(require('fs').readFileSync('config/events.json','utf8'))"
```

It must run without error.

---

## 12. Done — report back

Report:
- which of §3–§10 passed,
- whether `qr.png` was present,
- whether §11 passed,
- the uncommitted files, in both repos.

---

## 13. Operator tasks — people, not the implementer

These happen in Google Sheets and on the day. They are listed so the
constraints they put on the data are written down. The workbook was
checked on 30 September 2026. Items 1–3 were true then.

### 13.1 Workbook fixes before sync setup

The workbook was generated from the
[Standard Tournament Master](../implemented/standard-tournament-master-spec.md),
so its tabs, codes and columns already match what the site reads. `CSV`
and `STANDINGSCSV` have the runbook §4 columns, `Schedule` values are
`h:mm AM/PM`, and `CourtAssignment` is `Court 1`–`Court 3`.

1. **`Court Control` shows `#REF!`.** Its match columns read `#REF!` on all
   three courts. That may only be because no match is on court yet, or it
   may be a broken formula. The Live board depends on the `CSV` tab's
   `court` column, which Court Control feeds, so test it: put a match on
   Court 1 the way operators will on the day, and confirm the `#REF!`
   clears and `CSV`'s `court` column picks the match up. If it doesn't,
   compare the formulas with the master's Court Control and repair them.
   Undo the test afterwards.
2. **Rosters aren't in yet.** Every pair still shows its code (`ND_1`, …) as
   its player names. Paste the names into each category tab (`ND`, `IMD`,
   `IXD`).
3. **`IXD` has a bronze placeholder with no match.** `STANDINGSCSV` lists
   `IXD_B_1` and `IXD_B_2` (a BYE), but no IXD bronze match is scheduled:
   IXD is one 5-pair bracket with a twice-to-beat final (`IXD_F_*_(1)` and
   `_(2)`). The site builds standings from `STANDINGSCSV`, so expect an IXD
   "Bronze Battle" block showing TBD against BYE. Ask the organiser whether
   IXD awards a bronze. If it does, schedule the match. If it doesn't, the
   block is harmless but confusing: remove the IXD bronze rows where the
   IXD tab produces them, so they drop out of `STANDINGSCSV`.
4. **Match numbers are fine as they are.** One facility, numbered 1–56, so
   there is no range to reserve. Re-run **SAGE → Fill match numbers**
   (starting after 0) only if the schedule changes.
5. **Start time.** The pubmat says "8:00 AM onwards". The workbook's first
   match is 9:00 AM. Confirm 9:00 with the organiser. The site's auto
   go-live reads the workbook, not the pubmat.
6. **Colours.** Keep the SCHEDULE tab's category fills as they are. §8's
   `CAT_META` copies them. If the organiser recolours them, §8 has to be
   redone by hand.

### 13.2 Live sync setup — once, last

Do this **after** §13.1 is done. Until setup runs, nothing publishes, so
you can edit freely before it.

**Before you start:** the `events.json` entry (§11) must be committed to
`event-data`. Setup checks the day key and venue name against the server,
so it rejects them if the event isn't registered yet. After committing,
allow about a minute for the server to pick it up.

1. **Extensions → Apps Script.** Confirm `sheets-sync.gs` is there. A
   workbook copied from the master already carries it. If the **SAGE** menu
   lacks **Set up live sync**, replace the whole file with the current
   `sage-tools-api/apps-script/sheets-sync.gs` and save.
2. Reload the spreadsheet.
3. **SAGE → Set up live sync** (or **Live sync settings**). Enter:

   | Field | Value |
   |---|---|
   | Day | `piggleball-day1` |
   | Venue | `Centro Atletico` — exact, including capitals. **Not** "Quezon City", which is what the workbook's `Variables` tab has in its facility column |
   | Tabs to watch | `SCHEDULE` and `Court Control` (pre-ticked) |
   | Secret | Only if asked. It is `SYNC_SHARED_SECRET` from Cloud Run's env vars. Get it from whoever manages Cloud Run. Never paste it into chat, a doc or this spec |

4. **Save.** Setup checks the secret, day and venue, then sends one real
   test update before saving. If anything is wrong it says what and saves
   nothing.

**Confirm it's live:** open
`https://sage-match-control.github.io/event-data/piggleball-2026/data/piggleball-day1.json`
about a minute after setup. It should list one facility, `Centro Atletico`.
**SAGE → Sync now** re-sends immediately and reports the result.

### 13.3 Go-live

The site shows scores and the Live/Standings tabs from **4 hours before the
earliest scheduled match**: 5:00 AM on the day, for a 9:00 AM start. It
needs no configuration. To show them earlier or later, use Control
Center's go-live override.

### 13.4 Dry run

Run the dry run against the real workbook by **Friday 2 October**, using
`dry-run-checklist.md`. Also check:

- Control Center renders this event with the **standard** layout, and
  labels the categories "Novice Open Doubles", "Intermediate Men's
  Doubles" and "Intermediate Mixed Doubles", with no *Unmapped category*
  warning.
- The Live board shows three courts and no venue heading row (one
  facility).
- The schedule screen draws 3 court columns, with `ND` cells grey, `IMD`
  blue and `IXD` orange.
- An `IXD` final result shows both `_(1)` and `_(2)` games under one
  twice-to-beat final, not as two separate finals.

---

## 14. Decisions, for reference

- **Event key `piggleball-2026`, day key `piggleball-day1`.** The key is the
  tournament's own name, not the festival's, because the site covers only
  the pickleball. The day key carries an event prefix because day keys must
  be globally unique (§11), following `pickle-for-sight-day1`.
- **Title "Piggleball Chairman's Cup", "1st" in the headline.** The pubmat
  reads "1st PIGGLEBALL ★ CHAIRMAN'S CUP ★". An ordinal is wrong in a
  `<title>` or the Control Center masthead, and would date next year's
  page, so it sits only in the hero headline.
- **Divisions `N`/`I`, events `D`/`MD`/`XD`.** These are the workbook's
  own codes (`ND`, `IMD`, `IXD`), so nothing in the sheet changes. "Open
  Doubles" is the event's name for `ND`. The workbook's `Variables` tab
  labels it "Genderless Doubles", but neither the site nor Control Center
  reads that tab, so the two can differ.
- **Yellow in the fill role, red in the text role.** The template's accent
  fill sits under navy text and on navy panels. Red fails there (2.82:1),
  and yellow passes (8.09:1). Red passes as text on white and under white
  text (5.51:1), which is the other role. Both pubmat accents therefore
  appear without any rule changing its structure.
- **No pubmat pink.** The pig's skin (`#FD9E83`) and the logo's pink are
  illustration colours, not brand colours. The NFHFI logo already carries
  them onto the page.
- **The hero stays navy.** The pubmat's ground is white, but the template's
  hero controls (tabs, day picker, sync line) are styled for a navy panel. A
  white hero is a template redesign, not a re-skin.
- **Upright numbers.** Archivo Black stays upright for scores and stats,
  and only headings go italic. Italic figures in a dense score table are
  harder to read at a glance.
- **The pubmat is staged but not loaded.** It is 477 KB and duplicates what
  the hero already says. It stays in `assets/` as the colour and font
  reference.
- **No contact numbers.** The pubmat lists four private individuals' phone
  numbers for registration enquiries. Registration is closed by the time
  anyone uses this site.
