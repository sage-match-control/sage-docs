# Spec — Angeles City Employees Sportsfest Pickleball Tournament

> **Status: in progress.** Written 2026-10-21 against `sage-match-control.github.io`
> at `9f6717d`, `event-data`'s current `config/events.json`, and the event workbook
> as exported that day. The owner answered every open question the same day; the
> answers are in §17.
>
> **Built and pushed, 2026-10-21:** §4–§10 (the event's pages, with the logo) are on
> the site's `main` (`75c3e42`), and §14's `events.json` entry is in `event-data`'s
> `main`. The workbook syncs (§16.2): its snapshot lists `Pickle Place` with all 96
> matches and the players' names, so §16.1's items 1 and 3 are done. **Still to do:**
> `assets/qr.png` (§2) and the QR panel (§16.4), the rest of §16.1 (the `Court Control`
> check, the start time and the players in two categories), the playoff draw on the
> day, and the dry run (§16.5).
>
> **Dry-run on 2026-10-21.** §4–§9 were applied to a detached worktree of the site
> at `9f6717d`: every anchor matched once and every check passed. §11's fixture
> was built and the Hub, the board, Control Center and the desk page were checked
> in a browser at 375px, 1280px and 1440px; §11.3 is what they showed. §12's
> `npm run verify` and §14 were not run.
>
> **Deadline.** The event is on **Friday 13 November 2026**, and the site repo takes
> no merge within 3 days of an event. The pages reach the site's `main` by
> **Monday 9 November**. After that, push only with the owner's go-ahead.

Build the event site for the **Angeles City Employees Sportsfest Pickleball
Tournament**, a one-day tournament for Angeles City government employees on
**Friday 13 November 2026**, 8:00 AM to 5:00 PM, at **Pickle Place**. The
organiser is the **Angeles City Sports Office (ACSO)**.

The Sportsfest also has other sports (the ACSO logo shows volleyball, basketball,
boxing and swimming). **They are out of scope.** The site covers only the
pickleball tournament.

The site is an instance of `_templates/standard-tournament-template/`, the third
`"standard"` event after Pickle for Sight and Piggleball, and the first made from
the template since the site engine replaced the pages' configuration blocks. It
needs **no engine change**: the one unusual thing about this event, team codes
without a division (§3), is handled by the event's `events.json` entry alone.

There is **no pubmat**, so the site keeps the S.A.G.E. house theme: no colour or
font changes anywhere. The only colours this spec sets are the schedule board's
category colours, which are the organiser's own (§7).

The runbook for instantiating a template is
`sage-match-control.github.io/_templates/CLAUDE.md`. This spec gives every value
and every edit, so you should not need to make any design decisions. If this
spec and the runbook disagree, **stop and ask**. Don't pick one.

---

## 0. How to use this spec

You can carry this out with no other context: every value, every edit and every
check is here. Read §0 to §3 before you change anything.

### 0.1 Rules

- **Find edits by their anchor text, not line numbers.** Every edit quotes the
  exact text to find, and each anchor occurs once in its file. If an anchor
  isn't there, or is there twice, **stop and report**. Don't guess.
- **Every anchor is a single line.** The site checkout has Windows line endings
  (CRLF, from `core.autocrlf`), so a multi-line search can miss text that looks
  identical. Keep the files' line endings as they are.
- **Use Git Bash** for the commands and checks. Python is not installed on this
  machine; Node is (22.23.3 or later).
- **Edit only `events/acso-sportsfest-2026/`** in the site repo, and only
  `config/events.json` in `event-data`. Don't edit `lib/`, `tools/`,
  `_templates/`, `_tests/` or any other event's folder. If something there seems
  to need a change, stop and report it.
- **Don't change the theme.** Leave every `:root` block, font link and colour in
  the copied pages exactly as the template has them (§8).
- **Commit** where §0.3 says *commit*, with a plain message saying what changed,
  ending with the attribution line your session's instructions give.
- **Never push, and never merge into `main`, without the owner's yes in chat.**
  Pushing the site's `main` publishes it; pushing `event-data`'s `main` registers
  the event with the live API. §0.3 marks where you stop and ask.
- Don't rename, "fix" or tidy anything this spec doesn't mention.

### 0.2 Setup

The site repo is `D:\Personal\SAGE\sage-match-control.github.io`. Paths in
§4–§13 are relative to it.

```bash
cd /d/Personal/SAGE/sage-match-control.github.io
git status --short
```

This must print only `?? events/acso-sportsfest-2026/` (the staged logo, §2).
Anything else: stop and report.

```bash
git checkout main
git pull
git checkout -b acso-sportsfest
cd _tests
npm install
npm test
cd ..
```

`npm test` must pass before you change anything. If it doesn't, stop and report.

### 0.3 Order of work

1. §4–§9, the event's pages. *Commit once* (§10).
2. §11, the browser check, on a temporary fixture that is never committed.
3. §12, the full test run.
4. **Stop: push** (§13). Ask the owner.
5. §14, in `D:\Personal\SAGE\event-data`. *Commit*; push only on the owner's yes.
6. §15, report back.

---

## 1. Event facts

| Fact | Value |
|---|---|
| Organiser | Angeles City Sports Office (ACSO) |
| Event name | Angeles City Employees Sportsfest Pickleball Tournament |
| Date | Friday, 13 November 2026 — one day |
| Hours | 8:00 AM to 5:00 PM (the organiser's). The workbook's first match is 9:00 AM and its last slot 2:00 PM (§16.1 item 5) |
| Venue | Pickle Place. One facility, **8 courts** (`Court 1`–`Court 8`) |
| Categories | Men's Doubles (`MD`, 10 pairs), Women's Doubles (`WD`, 10 pairs), Mixed Doubles (`XD`, 20 pairs). **No divisions** (§3) |
| Format | `MD` and `WD`: two round-robin brackets of 5, top 2 of each to semifinals, then Bronze and Final. `XD`: four brackets of 5, top 2 of each to quarterfinals, then semifinals, Bronze and Final. Every final is a single match |
| Matches | 96, numbered 1–96, 25-minute slots from 9:00 AM. Round robin 9:00 AM–12:45 PM (matches 1–80), then playoffs 1:10 PM–2:00 PM (81–96) |
| Event key | `acso-sportsfest-2026` |
| Day key | `acso-sportsfest-day1` |
| Short link | `tinyurl.com/SAGExACSO2026` → `https://sage-match-control.github.io/events/acso-sportsfest-2026/` |

The schedule, from the workbook's `SCHEDULE` tab (courts left to right):

| Slot | Courts 1–2 | Courts 3–4 | Courts 5–8 |
|---|---|---|---|
| 9:00 AM – 12:45 PM | `MD` round robin | `WD` round robin | `XD` round robin |
| 1:10 PM | `MD` semifinals | `WD` semifinals | `XD` quarterfinals |
| 1:35 PM | `MD` Final, Bronze | `WD` Final, Bronze | `XD` semifinals on courts 5 and 7 |
| 2:00 PM | — | — | `XD` Final (court 5), Bronze (court 7) |

**The site shows none of these:** the other Sportsfest sports and the organiser's
8:00 AM–5:00 PM hours. The site is a live results hub for players who have
already entered, and its times come from the workbook.

---

## 2. Inputs

| Input | Status | Used in |
|---|---|---|
| ACSO logo | **Staged** at `events/acso-sportsfest-2026/assets/logo.webp` (1254×1254, 247 KB, white background, a round badge). Use it as is: no conversion, no cropping | §5 `{{EVENT_LOGO}}` |
| Pubmat | **None.** The organiser has none; the site keeps the house theme (§8) | — |
| QR image for the short link | **To follow from the owner** as `events/acso-sportsfest-2026/assets/qr.png`. If it isn't there, do everything else and report it missing. Don't generate one | Template QR panel |
| Short link | **Confirmed:** `tinyurl.com/SAGExACSO2026` (§17). The owner creates it | §5 `{{QR_URL}}` |
| Event workbook | `18UH-CHx4_qIdI5y89qrSKDQR0Cd0dzKWQRQfKunNFjg`, generated from the Standard Tournament Master | §14 |

---

## 3. How this event differs from Piggleball

[Piggleball](../implemented/piggleball-chairmans-cup-spec.md) was the last
standard event. Four differences matter:

1. **The pages are shells now.** Piggleball's spec filled in `DAYS`,
   `FACILITIES`, `DIVISIONS` and `EVENTS` arrays inside `index.html`. Those are
   gone: the Hub reads its days, facility and labels from `events.json` (§14).
   There is no configuration block to fill in.
2. **No re-skin.** No pubmat, so no palette, font or hero-style changes (§8).
3. **No divisions.** Every team code is `<EVENT>_<REST>` (`MD_1`, `WD_SF_2`,
   `XD_QF_7`), where earlier events had `<DIVISION><EVENT>_<REST>` (`IMD_1`).
   The engine splits a category token by matching `display.divisions`' keys as
   prefixes, then looks the remainder up in `display.events`. An **empty-string
   division key**, `"divisions": { "": "" }`, matches every token as a prefix and
   leaves the whole token (`MD`) as the event code, so `MD` reads "Men's
   Doubles": the division label is empty and the engine trims the space.
   Checked against `lib/v1/domain/codes.js` and `lib/v1/domain/attendance.js`
   (`categoryLabel`, `buildCategoryOrderIndex`, `defaultCategoryLabel`):
   `MD_1`, `WD_SF_2`, `XD_QF_7` label as "Men's Doubles", "Women's Doubles",
   "Mixed Doubles", order MD, WD, XD, with no *Unmapped category* warning.
   **Without** that key the pages still work but show the raw codes `MD`, `WD`,
   `XD`. Don't drop it, and don't invent a division name.
4. **Eight courts**, so the schedule board has eight columns.

---

## 4. Create the event folder

The folder already exists, because it holds the staged logo. Copy the template's
**contents** into it. The trailing `/.` matters: without it, `cp` would create a
nested `standard-tournament-template/` folder inside the existing one.

```bash
cp -r _templates/standard-tournament-template/. events/acso-sportsfest-2026/
cp _templates/attendance/attendance.html events/acso-sportsfest-2026/attendance.html
```

The event takes attendance through desk links (`"attendance": "desks"`, §14), so
it gets the desk page. It enters scores from Control Center only
(`"scoreEntry": "console"`), so it gets **no** `scorer.html`.

**Check:** `ls events/acso-sportsfest-2026` shows `index.html`, `schedule.html`,
`attendance.html` and `assets/`, and `assets/` still contains `logo.webp`.

---

## 5. Replace the `{{TOKENS}}`

Replace every occurrence in `index.html`, `schedule.html` and `attendance.html`
of `events/acso-sportsfest-2026/`:

| Token | Replace with |
|---|---|
| `{{EVENT_KEY}}` | `acso-sportsfest-2026` |
| `{{EVENT_TITLE}}` | `Angeles City Employees Sportsfest Pickleball` |
| `{{EVENT_TAGLINE}}` | `Angeles City Sports Office presents` |
| `{{EVENT_HEADLINE}}` | `Pickleball Tournament` |
| `{{EVENT_DATE_RANGE}}` | `13 November 2026` |
| `{{VENUE}}` | `Pickle Place` |
| `{{QR_IMAGE}}` | `assets/qr.png` |
| `{{QR_URL}}` | `tinyurl.com/<wbr>SAGExACSO2026` |
| `{{EVENT_LOGO}}` | `assets/logo.webp` |
| `{{SCHEDULE_DAY_KEY}}` | `acso-sportsfest-day1` |

- `attendance.html` has only `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`.
- None of these values contains `&` or a quote. `{{QR_URL}}`'s `<wbr>` is HTML,
  typed as written: it lets the link wrap after the `/` on desktop.

**Check:** this must print nothing:

```bash
grep -rnI '{{' events/acso-sportsfest-2026/
```

---

## 6. `index.html` — content edits

All edits are in `events/acso-sportsfest-2026/index.html`.

### 6.1 Hero — the logo's alt text

The token filled the organiser's logo's alt text with the event title. The logo is
the Sports Office's, not the tournament's. Find:

```html
    <img src="assets/logo.webp" alt="Angeles City Employees Sportsfest Pickleball logo" />
```

Replace with:

```html
    <img src="assets/logo.webp" alt="Angeles City Sports Office logo" />
```

### 6.2 Hero — title

The full title is too long for the hero's display type: it would take four lines
on a phone. The hero splits it instead: "Angeles City" is already in the tagline
above, "Pickleball Tournament" is the headline below, so the title itself is
"Employees Sportsfest", with "Sportsfest" in the accent colour. Find (as it reads
after §5):

```html
  <h1 class="title">Angeles City Employees Sportsfest Pickleball</h1>
```

Replace with:

```html
  <h1 class="title">Employees <span class="accent">Sportsfest</span></h1>
```

`h1.title .accent` is already styled by the engine's `hub.css` (the house green on
the navy hero). Add no CSS.

The hero now stacks: ACSO logo × S.A.G.E. logo → "Angeles City Sports Office
presents" → "13 NOVEMBER 2026 · PICKLE PLACE" → "EMPLOYEES **SPORTSFEST**" →
"PICKLEBALL TOURNAMENT" → the subtitle.

### 6.3 Footer

Find the footer's first line (as it reads after §5):

```html
  Angeles City Employees Sportsfest Pickleball &middot; 13 November 2026 &middot; Pickle Place<br>
```

Replace it with these two lines:

```html
  Angeles City Employees Sportsfest Pickleball Tournament &middot; 13 November 2026 &middot; Pickle Place<br>
  Organised by the Angeles City Sports Office<br>
```

The footer then reads, in full:

```html
<footer>
  Angeles City Employees Sportsfest Pickleball Tournament &middot; 13 November 2026 &middot; Pickle Place<br>
  Organised by the Angeles City Sports Office<br>
  Powered by S.A.G.E. Match Control Experts
</footer>
```

### 6.4 Example text left over from the template

Find:

```html
placeholder="e.g. Beginner 18+ Men's Doubles"
```

Replace with:

```html
placeholder="e.g. Mixed Doubles"
```

Leave the pair-search placeholder `e.g. John Smith / Jane Doe` as it is.

**Check:** each must print exactly one line:

```bash
grep -n 'alt="Angeles City Sports Office logo"' events/acso-sportsfest-2026/index.html
grep -n '<span class="accent">Sportsfest</span>' events/acso-sportsfest-2026/index.html
grep -n 'Organised by the Angeles City Sports Office' events/acso-sportsfest-2026/index.html
grep -n 'e.g. Mixed Doubles' events/acso-sportsfest-2026/index.html
```

---

## 7. `schedule.html` — category colours

These are the **organiser's** colours: the category fills on the workbook's own
`SCHEDULE` tab, read from the sheet's HTML view for this spec. All three are in
Google Sheets' standard palette:

| Category | Fill | Sheets' name |
|---|---|---|
| `MD` | `#4A86E8` | cornflower blue |
| `WD` | `#CC4125` | red berry |
| `XD` | `#FF9900` | orange |

In `events/acso-sportsfest-2026/schedule.html`, find the comment line

```js
// Category colours are the organiser's own, read off the colour-coded SCHEDULE
```

and replace **everything from that line down to and including the `};` that
closes `const CAT_META = {`** (the template's comment block, its `EXAMPLE —
replace` line and its seven-category example, `LIWD` to `HIXD`) with:

```js
// Category colours are the organiser's own, read off the colour-coded SCHEDULE
// tab of the event workbook (Google Sheets' standard palette): Men's cornflower
// blue, Women's red berry, Mixed orange. Cell fills are formatting, not data, so
// nothing detects a change: if the organiser recolours the sheet, re-read them by
// hand. Keys are the category token of each team code (MD_1 -> MD); this event
// has no divisions. Cells apply the hue as a 40% tint over white.
const CAT_META = {
  MD: { short:'MD', color:'#4A86E8' },
  WD: { short:'WD', color:'#CC4125' },
  XD: { short:'XD', color:'#FF9900' }
};
```

Leave the `mountScheduleBoard({ … })` call below it unchanged.

**Check:** the first must print nothing, the second three lines:

```bash
grep -n "EXAMPLE\|LIWD\|HIXD\|AMD" events/acso-sportsfest-2026/schedule.html
grep -n "color:'#4A86E8'\|color:'#CC4125'\|color:'#FF9900'" events/acso-sportsfest-2026/schedule.html
```

---

## 8. Theme — no change

Leave the `:root` blocks of `index.html` and `schedule.html`, their Google Fonts
links and every other colour exactly as copied. The house palette (navy
`#14263C`, green `#7CB92C`, off-white paper `#F6F7F2`) and type (Archivo Black,
Barlow Condensed, Inter) are the theme. `attendance.html` keeps the house theme
too, like every desk page.

**Check:** this must print nothing (the copied pages still match the template
apart from §5–§7):

```bash
diff <(sed -n '/:root{/,/}/p' _templates/standard-tournament-template/index.html) <(sed -n '/:root{/,/}/p' events/acso-sportsfest-2026/index.html)
diff <(sed -n '/:root{/,/}/p' _templates/standard-tournament-template/schedule.html) <(sed -n '/:root{/,/}/p' events/acso-sportsfest-2026/schedule.html)
```

---

## 9. Dry-run checklist

```bash
cp _templates/dry-run-checklist-template.md events/acso-sportsfest-2026/dry-run-checklist.md
```

Replace `{{EVENT_TITLE}}` with `Angeles City Employees Sportsfest Pickleball` in
the copy.

**Check:** `grep -rnI '{{' events/acso-sportsfest-2026/` still prints nothing.

---

## 10. Commit

```bash
git add events/acso-sportsfest-2026
git status --short
```

`git status` must list only files under `events/acso-sportsfest-2026/`:
`index.html`, `schedule.html`, `attendance.html`, `dry-run-checklist.md`,
`assets/logo.webp`, and `assets/qr.png` if the owner supplied it. Commit them as
one commit, e.g. *"Angeles City Employees Sportsfest: event site from the
standard template (house theme, organiser's category colours)"*.

---

## 11. Verify in a browser

The real workbook won't publish until the operator has fixed and wired it up
(§16), so check the pages on a **temporary fixture**: a made-up finished day of
this event, built on the workbook's real schedule. On `localhost` only,
`?fixture=finished` makes every engine page read `/_fixtures/config.json` as the
registry and `/_fixtures/<event>/finished.json` as the day's data.

### 11.1 Make the fixture

Save Appendix A's script **outside the repo** (for example in your scratchpad) as
`make-acso-fixture.mjs`, then from the site repo root:

```bash
node <path-to>/make-acso-fixture.mjs
```

It must print:

```
96 matches, 40 pairs; champions MD Men 7A, WD Women 10A, XD Mixed 4A
```

### 11.2 Serve and open

Serve the **repo root**, not the event folder: the pages load root-absolute
`/assets/…` and `/lib/v1/…` paths. From the site repo root:

```bash
npx --yes http-server -p 8000 -c-1
```

(If your environment runs servers another way, serve the same folder on any port;
the host must be `localhost` for fixture mode.) Open:

- the Hub: `http://localhost:8000/events/acso-sportsfest-2026/?fixture=finished`
- the board: `http://localhost:8000/events/acso-sportsfest-2026/schedule.html?fixture=finished`
- Control Center: `http://localhost:8000/tools/control-center.html?fixture=finished`
- the desk page: `http://localhost:8000/events/acso-sportsfest-2026/attendance.html?fixture=finished`

### 11.3 Checks

The Hub, at **375px** and **desktop** width:

- [ ] The hero reads, top to bottom: the ACSO badge in a white tile (whole, not
      cropped) × the S.A.G.E. logo, "Angeles City Sports Office presents",
      "13 NOVEMBER 2026 · PICKLE PLACE" in green, "EMPLOYEES SPORTSFEST" in white
      with "SPORTSFEST" in green, "PICKLEBALL TOURNAMENT" in green, then the
      subtitle.
- [ ] At 375px the title takes at most two lines, breaking between the words.
- [ ] The day picker is hidden (one day) and the day loads.
- [ ] The theme is the house one: navy hero, off-white paper, green accents.
- [ ] Desktop only: the QR panel shows `tinyurl.com/SAGExACSO2026`, and the image if
      `assets/qr.png` was supplied (a broken image otherwise, §2).
- [ ] The footer shows three lines, with "Organised by the Angeles City Sports
      Office" in the middle.
- [ ] **Match Finder:** the stats row reads 3 divisions. Searching "Men 7A" lists
      that pair's matches, labelled "Men's Doubles", ending with the Final.
- [ ] **Standings** (desktop): three category columns, in the order **Men's
      Doubles, Women's Doubles, Mixed Doubles**, never the raw `MD`/`WD`/`XD`. Men's
      and Women's each have Bracket 1 and Bracket 2 then Semifinals, Bronze Battle
      and Finals; Mixed has Brackets 1–4 then Quarterfinals, Semifinals, Bronze
      Battle and Finals. Men's Finals shows Men 3A / Men 3B **0 VS 11** Men 7A /
      Men 7B. (The fixture seats its playoffs by a fixed draw, so a bracket's
      second place in the table isn't always the pair in the semifinal: Men's
      Bracket 2 lists Men 8A second on head-to-head, and Men 9A plays the
      semifinal. That is the fixture, not a bug.)
- [ ] **Standings** (375px): the tab first asks for a category. The category
      search's placeholder reads "e.g. Mixed Doubles", typing "mixed" offers
      "Mixed Doubles", and the category buttons read Men's Doubles, Women's
      Doubles, Mixed Doubles.
- [ ] DevTools: the console shows no errors, and the only 404 is `assets/qr.png`
      when it hasn't been supplied.

The board, at **1280px**:

- [ ] The bezel reads "Angeles City Employees Sportsfest Pickleball", the court
      filter offers **ALL** and 1–8, and the time column runs from 9:00 AM to
      2:00 PM in 13 slots. At this width courts 7–8 are off to the right (the grid
      scrolls sideways); a wall display at 1920px shows all eight.
- [ ] Courts 1–2 are blue (`MD`), 3–4 red (`WD`), 5–8 orange (`XD`), and the
      legend shows `MD`, `WD` and `XD` chips in those colours.
- [ ] Open the board with `&courts=5-8` added to the URL: at 1:35 PM courts 6
      and 8 read **OPEN**; at 2:00 PM court 5 holds the `FINAL` and court 7 the
      `BRONZE`, and courts 6 and 8 read **OPEN**.

Control Center (it opens on this event, the newest date; pick it if it doesn't):

- [ ] The masthead reads "ANGELES CITY EMPLOYEES SPORTSFEST…", and Mission
      Control shows **Pickle Place**, "96 / 96 done", and "Auto — goes live
      5:00 AM, Nov 13 (4h before first serve, 9:00 AM)".
- [ ] **Standings** offers the categories "Men's Doubles", "Women's Doubles" and
      "Mixed Doubles", and there is **no** *Unmapped category* warning anywhere.
- [ ] **Awards** reads "3 of 3 podiums complete": Men's Doubles Men 7A / Men 7B
      then Men 3A / Men 3B; Women's Doubles Women 10A / Women 10B then Women 6A /
      Women 6B; Mixed Doubles Mixed 4A / Mixed 4B then Mixed 15A / Mixed 15B.
- [ ] **Attendance** reads "Pickle Place: No roster yet". Expected in fixture
      mode: the console's 404s for `scorer.html`, `scorer-beta.html` and
      `_fixtures/acso-sportsfest-2026/attendance-pickle-place-finished.csv`.
- [ ] Another standard event still labels as before. Fixture mode lists only the
      fixtures' events, so open
      `http://localhost:8000/tools/control-center.html?fixture=pre&event=attendance-demo-2026&day=attendance-demo-2026-day1`:
      its Standings categories read "Novice Men's Doubles", "Novice Women's
      Doubles" and "Low Intermediate Men's Doubles".

The desk page:

- [ ] Opened without a link, it reads "Ask the operator for a desk link." and
      requests nothing that 404s. (A real link needs the live API; the dry run
      checks one, §16.5.)

Stop the server.

### 11.4 Remove the fixture

```bash
git checkout -- _fixtures/config.json
rm -rf _fixtures/acso-sportsfest-2026
git status --short
```

`git status` must show nothing under `_fixtures/`. The fixture is never committed:
the comparison harness reads `_fixtures/config.json`, and an extra event there
would change Control Center's event list in its screenshots.

---

## 12. The full test run

```bash
cd _tests
npm run verify
cd ..
```

It must pass. This change adds an event folder and touches nothing the harness
compares, so a failure means something outside `events/acso-sportsfest-2026/`
changed: run `git status` and stop and report.

---

## 13. Push — stop and ask

Tell the owner the branch is ready, with §11's and §12's results, and ask whether
to merge it into `main` and push. On a yes:

```bash
git checkout main
git pull
git merge --ff-only acso-sportsfest
git push
```

If `--ff-only` refuses, stop and report. Pushing publishes the pages at
`https://sage-match-control.github.io/events/acso-sportsfest-2026/` within a minute
or two. Until §14 is pushed too, the Hub shows no days: that is expected.

---

## 14. Register the event in `event-data`

In `D:\Personal\SAGE\event-data\config\events.json`, add this entry inside
`"events"`, **after** `"millennium-invitational-2026"` (currently the last entry),
and add a comma after the closing brace of that entry:

```json
"acso-sportsfest-2026": {
  "type": "standard",
  "title": "Angeles City Employees Sportsfest Pickleball",
  "attendance": "desks",
  "scoreEntry": "console",
  "days": {
    "acso-sportsfest-day1": {
      "label": "Nov 13",
      "date": "2026-11-13",
      "isLive": "auto",
      "facilities": [
        { "name": "Pickle Place", "sheetId": "18UH-CHx4_qIdI5y89qrSKDQR0Cd0dzKWQRQfKunNFjg" }
      ]
    }
  },
  "display": {
    "divisions": { "": "" },
    "events": { "MD": "Men's Doubles", "WD": "Women's Doubles", "XD": "Mixed Doubles" }
  }
}
```

- `"divisions": { "": "" }` is deliberate and required: §3 item 3.
- Key order in `events` sets display order: Men's, Women's, Mixed, as on the
  workbook.
- The facility name `Pickle Place` is the workbook's own (`Variables`' facility
  column, `Title`). Sync setup compares it exactly, so §16.2 must type the same.
- `attendance: "desks"` and `scoreEntry: "console"` are what Piggleball, the last
  standard event, used: desk links for check-in, scores entered from Control
  Center. The owner confirmed both (§17).

**Before saving, confirm no other event already uses this day key:**

```bash
cd /d/Personal/SAGE/event-data
grep -n '"acso-sportsfest-day1"' config/events.json
```

This must print nothing. Day keys are validated as **globally unique across every
event** in this file, because the day key alone is the sync route. A duplicate
makes the whole file fail validation, and Cloud Run then keeps serving the last
good config, so this event never registers. If the key is taken, **stop and ask**.

**Check:** run from `event-data/`:

```bash
node -e "const c=JSON.parse(require('fs').readFileSync('config/events.json','utf8'));const e=c.events['acso-sportsfest-2026'];console.log(e.type, Object.keys(e.days), JSON.stringify(e.display.divisions))"
```

It must print `standard [ 'acso-sportsfest-day1' ] {"":""}`. Then commit it in
`event-data` (on `main`, which is how that repo is worked) and ask the owner before
`git push`: pushing registers the event with the live API and adds it to Control
Center within about a minute.

---

## 15. Done — report back

Report:
- which of §4–§12 passed, and that the fixture was removed (§11.4);
- whether `qr.png` was present, and the short link used;
- whether the site branch was pushed (only with the owner's go-ahead);
- whether §14 passed, and whether it was pushed;
- the uncommitted files, in every repo touched.

---

## 16. Operator tasks — people, not the implementer

These happen in Google Sheets and on the day. They are listed so the constraints
they put on the data are written down. The workbook was checked on 21 October
2026 from an export.

### 16.1 Workbook fixes before sync setup

The workbook was generated from the
[Standard Tournament Master](../implemented/standard-tournament-master-spec.md), so
its tabs, codes and column names match what the site reads. `SCHEDULE` is packed
and numbered 1–96 with no gaps or repeats, its times are `h:mm AM/PM`, its courts
`Court 1`–`Court 8`, and `STANDINGSCSV` lists every pair and playoff slot with its
bracket. The `ATTENDANCE` tab has its header. What is left:

1. **`CSV` lists only match 1.** Rows 3–97 hold the match numbers 2–96 but every
   other column is blank, so the site would show one match. **SAGE → Fill match
   numbers** writes the numbers into `CSV!A` *and* copies `CSV` row 2's formulas
   down a row per match; this time the copy didn't happen. Re-run **SAGE → Fill
   match numbers**, starting after `0`. If the rows are still blank, copy `CSV!B2:L2`
   and paste it into `B3:L97`. **Check:** `CSV` has 96 rows, and row 97 reads match
   96, 2:00 PM, Court 7, `XD_B_1` against `XD_B_2`.
2. **`Court Control` shows `#REF!`** on all 8 courts. As at Piggleball, that may be
   only because no match is on court yet, or a broken formula. The Live board reads
   `CSV`'s `court` column, which Court Control feeds, so test it: put a match on
   Court 1 the way operators will on the day, confirm the `#REF!` clears and that
   match's `court` cell in `CSV` fills in. If it doesn't, compare the formulas with
   the master's Court Control and repair them. Undo the test afterwards.
3. **Rosters aren't in yet.** Every pair still shows its code (`MD_1`, …) as its
   player name. Paste the names into each category tab (`MD`, `WD`, `XD`), then run
   **Update roster** for the `ATTENDANCE` tab from Control Center.
4. **The playoff draw.** Each category tab has a "place 1st playoffs bunutan here"
   block that seats the bracket qualifiers in the `SF` (`MD`, `WD`) or `QF` (`XD`)
   slots. It is done on the day, after the round robin.
5. **Start time.** The organiser's hours are 8:00 AM to 5:00 PM; the workbook's
   first match is 9:00 AM and its last at 2:00 PM. Confirm 9:00 AM with the
   organiser (8:00 AM is presumably arrival and the opening). The site's go-live
   reads the workbook, not the hours.
6. **Tight playoffs — the organiser's call.** The `MD` and `WD` semifinals
   (1:10 PM) follow the last round-robin slot (12:45 PM) directly, with no slot
   free for the draw, and each Final follows its semifinal directly (1:35 PM);
   `XD` plays quarterfinal, semifinal and Final in three consecutive slots. The
   site shows whatever the workbook says, so nothing here needs changing unless
   the organiser wants rest.
7. **Players in two categories.** `MD`, `WD` and `XD` run side by side all morning,
   so an employee entered in `MD` or `WD` and also in `XD` can be due on two courts
   in the same slot. `Timeline` only catches the *same pair* twice in a slot. Once
   names are in (item 3), look for any name in both `XD` and one of the others, and
   check their slots.
8. **`Variables`' `title`, `courts` and `facility` settings are blank.** Harmless:
   `Title` and the category rows carry what the workbook needs, and the site reads
   neither. Leave them.
9. **Colours.** Keep the `SCHEDULE` tab's category fills as they are. §7's
   `CAT_META` copies them. If the organiser recolours them, §7 has to be redone by
   hand.

### 16.2 Share it, then set up live sync — once, last

Do this **after** §16.1. Until setup runs, nothing publishes, so you can edit
freely before it.

**Before you start:** §14's `events.json` entry must be pushed to `event-data`.
Setup checks the day key and venue name against the server. Allow about a minute
after the push.

1. **Share** the workbook with
   `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**,
   and set **General access** to **Restricted** (`_templates/CLAUDE.md` §2 step 13).
2. **Extensions → Apps Script.** Confirm `sheets-sync.gs` is there; a workbook made
   from the master carries it. If the **SAGE** menu lacks **Set up live sync**,
   replace the whole file with the current `sage-tools-api/apps-script/sheets-sync.gs`
   and save. Reload the spreadsheet.
3. **SAGE → Set up live sync.** Enter:

   | Field | Value |
   |---|---|
   | Day | `acso-sportsfest-day1` |
   | Venue | `Pickle Place` — exact, including capitals |
   | Tabs to watch | `SCHEDULE` and `Court Control` (pre-ticked) |
   | Secret | Only if asked. It is `SYNC_SHARED_SECRET` from Cloud Run's env vars. Get it from whoever manages Cloud Run. Never paste it into chat, a doc or this spec |

4. **Save.** Setup checks the secret, day and venue, then sends one real test
   update before saving. If anything is wrong it says what and saves nothing.

**Confirm it's live:** open
`https://sage-match-control.github.io/event-data/acso-sportsfest-2026/data/acso-sportsfest-day1.json`
about a minute after setup. It should list one facility, `Pickle Place`, with 96
matches in its `matchesCsv`. **SAGE → Sync now** re-sends immediately.

### 16.3 Go-live

The site shows scores and the Live/Standings tabs from **4 hours before the
earliest scheduled match**: 5:00 AM on the day, for a 9:00 AM start. It needs no
configuration. To show them earlier or later, use Control Center's go-live
override.

### 16.4 The QR panel

Once `assets/qr.png` is in the pushed site, print the venue board's QR panel:

```bash
node _templates/hub-pubmat/render.mjs acso-sportsfest-2026
```

It writes `qr-panel.pdf` and `board.pdf` to `_templates/hub-pubmat/out/acso-sportsfest-2026/`
(git-ignored). Scan the printed panel before mounting it.

### 16.5 Dry run

Run the dry run against the real workbook by **Wednesday 11 November**, using
`events/acso-sportsfest-2026/dry-run-checklist.md`. Also check:

- Control Center renders this event with the **standard** layout, labels the
  categories "Men's Doubles", "Women's Doubles" and "Mixed Doubles", and shows no
  *Unmapped category* warning.
- The Live board shows 8 courts and no venue heading row (one facility).
- The schedule board draws 8 court columns, `MD` blue, `WD` red, `XD` orange.
- **Issue desk link** produces a link that opens the desk page with the roster.

### 16.6 After the event

Turn live push off for the pages (`LIVE_BASE_URL = ''` in `index.html`,
`schedule.html` and `attendance.html`), as `_templates/CLAUDE.md` §7 describes.
Leave the event in `events.json`.

---

## 17. Questions, answered by the owner on 21 October 2026

| Question | Answer |
|---|---|
| The short link | `tinyurl.com/SAGExACSO2026` (§2, §5). The owner creates it and supplies `assets/qr.png` |
| Attendance and score entry | `"attendance": "desks"`, `"scoreEntry": "console"`, as at Piggleball (§14): desk links for check-in, scores from Control Center, no `scorer.html` |
| The venue line | "Pickle Place" alone (§5) |
| First serve | 9:00 AM, as the workbook has it. The organiser's 8:00 AM is arrival and the opening (§16.1 item 5) |

---

## 18. Decisions, for reference

- **Event key `acso-sportsfest-2026`, day key `acso-sportsfest-day1`.** Short,
  carry the organiser's own acronym (the big word on its logo), and the day key has
  an event prefix because day keys must be globally unique (§14).
- **Title "Angeles City Employees Sportsfest Pickleball".** The tournament's own
  name, not the Sportsfest's, because the site covers only the pickleball (as
  Piggleball's title was the tournament's, not the Pig Sports Festival's). It drops
  the trailing "Tournament", which would read twice in the browser tab's
  "… Tournament — Tournament Hub". The footer gives the full name.
- **Hero split across tagline, title and headline** (§6.2): the full title in the
  hero's display type would take four lines on a phone.
- **Empty division key** (§3 item 3), not a made-up division like "Open": the
  organiser has no divisions, and an invented label would show on every card.
- **House theme.** No pubmat, so no re-skin. The ACSO logo is shown as supplied; its
  red, yellow, green and blue are not lifted into the palette, because that would be
  a re-skin from a logo alone, which no event has done.
- **The board's colours are the organiser's** (§7), so the wall display matches the
  printed schedule, as at every standard event.
- **No hours on the page.** The organiser's 8:00 AM–5:00 PM would sit next to match
  times starting at 9:00 AM and confuse players; the Hub's times come from the
  workbook.

---

## Appendix A — the fixture script

Used by §11 only, and never committed. Save it outside the repo and run it from the
site repo root. It writes `_fixtures/acso-sportsfest-2026/finished.json` and adds
the event to `_fixtures/config.json`; §11.4 removes both.

```js
// Writes a temporary, local-only fixture for the Angeles City Employees Sportsfest, shaped like the real event:
// the workbook's own SCHEDULE (96 matches on 8 courts, 9:00 AM to 2:00 PM), MD and WD in two round-robin brackets
// of 5 then semifinals, Bronze and Final, XD in four brackets of 5 then quarterfinals, semifinals, Bronze and Final.
// Every match is played, with fixed made-up scores and player names.
//   _fixtures/acso-sportsfest-2026/finished.json   the snapshot
//   _fixtures/config.json                          gains an "acso-sportsfest-2026" entry
// Neither is ever committed. Undo with:
//   git checkout -- _fixtures/config.json && rm -rf _fixtures/acso-sportsfest-2026
// usage, from the site repo root: node <path>/make-acso-fixture.mjs
import fs from 'node:fs';
import path from 'node:path';

const EVENT = 'acso-sportsfest-2026', DAY = 'acso-sportsfest-day1';

// The workbook's SCHEDULE, slot by slot: each slot lists Court 1, Court 2, … in order, '-' for an empty court.
const SLOTS = [
  ['9:00 AM',  'MD_1:MD_2 MD_3:MD_4 WD_1:WD_2 WD_3:WD_4 XD_1:XD_2 XD_3:XD_4 XD_6:XD_7 XD_8:XD_9'],
  ['9:25 AM',  'MD_6:MD_7 MD_8:MD_9 WD_6:WD_7 WD_8:WD_9 XD_11:XD_12 XD_13:XD_14 XD_16:XD_17 XD_18:XD_19'],
  ['9:50 AM',  'MD_1:MD_3 MD_4:MD_5 WD_1:WD_3 WD_4:WD_5 XD_1:XD_3 XD_4:XD_5 XD_6:XD_8 XD_9:XD_10'],
  ['10:15 AM', 'MD_6:MD_8 MD_9:MD_10 WD_6:WD_8 WD_9:WD_10 XD_11:XD_13 XD_14:XD_15 XD_16:XD_18 XD_19:XD_20'],
  ['10:40 AM', 'MD_1:MD_4 MD_2:MD_5 WD_1:WD_4 WD_2:WD_5 XD_1:XD_4 XD_2:XD_5 XD_6:XD_9 XD_7:XD_10'],
  ['11:05 AM', 'MD_6:MD_9 MD_7:MD_10 WD_6:WD_9 WD_7:WD_10 XD_11:XD_14 XD_12:XD_15 XD_16:XD_19 XD_17:XD_20'],
  ['11:30 AM', 'MD_3:MD_5 MD_2:MD_4 WD_3:WD_5 WD_2:WD_4 XD_3:XD_5 XD_2:XD_4 XD_8:XD_10 XD_7:XD_9'],
  ['11:55 AM', 'MD_8:MD_10 MD_7:MD_9 WD_8:WD_10 WD_7:WD_9 XD_13:XD_15 XD_12:XD_14 XD_18:XD_20 XD_17:XD_19'],
  ['12:20 PM', 'MD_1:MD_5 MD_2:MD_3 WD_1:WD_5 WD_2:WD_3 XD_1:XD_5 XD_2:XD_3 XD_6:XD_10 XD_7:XD_8'],
  ['12:45 PM', 'MD_6:MD_10 MD_7:MD_8 WD_6:WD_10 WD_7:WD_8 XD_11:XD_15 XD_12:XD_13 XD_16:XD_20 XD_17:XD_18'],
  ['1:10 PM',  'MD_SF_1:MD_SF_2 MD_SF_3:MD_SF_4 WD_SF_1:WD_SF_2 WD_SF_3:WD_SF_4 XD_QF_1:XD_QF_2 XD_QF_3:XD_QF_4 XD_QF_5:XD_QF_6 XD_QF_7:XD_QF_8'],
  ['1:35 PM',  'MD_F_1:MD_F_2 MD_B_1:MD_B_2 WD_F_1:WD_F_2 WD_B_1:WD_B_2 XD_SF_1:XD_SF_2 - XD_SF_3:XD_SF_4 -'],
  ['2:00 PM',  '- - - - XD_F_1:XD_F_2 - XD_B_1:XD_B_2 -'],
];
const CATS = { MD: { name: 'Men', pairs: 10, brackets: 2 }, WD: { name: 'Women', pairs: 10, brackets: 2 }, XD: { name: 'Mixed', pairs: 20, brackets: 4 } };

// Fixed pseudo-random scores: the same file every run.
let seed = 20261113;
const rand = n => { seed = (seed * 1103515245 + 12345) % 2147483648; return Math.floor(seed / 65536) % n; };
const score = () => { const loser = rand(10); return rand(2) ? [11, loser] : [loser, 11]; };

// A pair's players, like the workbook's (each name ends in a space). Playoff slots take their qualifier's names.
const names = {};
for (const [cat, c] of Object.entries(CATS)) {
  for (let n = 1; n <= c.pairs; n++) names[`${cat}_${n}`] = [`${c.name} ${n}A `, `${c.name} ${n}B `];
}
const bracketOf = code => Math.ceil(Number(code.split('_')[1]) / 5);

const rec = {};   // code -> { w, l, pf, pa }
const tally = (code, pf, pa) => {
  const r = rec[code] || (rec[code] = { w: 0, l: 0, pf: 0, pa: 0 });
  r.pf += pf; r.pa += pa; if (pf > pa) r.w++; else r.l++;
};
const winnerOf = {}, loserOf = {};   // "MD_SF_1:MD_SF_2" -> the winning / losing pair's names
const rows = [];
let num = 0;

// Rank a category's round-robin bracket: wins, then points quotient.
const ranked = (cat, b) => Object.keys(rec).filter(k => k.startsWith(cat + '_') && /^[A-Z]+_\d+$/.test(k) && bracketOf(k) === b)
  .sort((x, y) => rec[y].w - rec[x].w || rec[y].pf / rec[y].pa - rec[x].pf / rec[x].pa);
// Who fills each playoff slot once the stage before it is over (the organiser's draw, fixed here).
const seat = (cat, code) => {
  const rest = code.slice(cat.length + 1);
  const top = (b, i) => names[ranked(cat, b)[i]];
  if (CATS[cat].brackets === 2) {
    if (rest.startsWith('SF_')) return { SF_1: top(1, 0), SF_2: top(2, 1), SF_3: top(2, 0), SF_4: top(1, 1) }[rest];
  } else {
    if (rest.startsWith('QF_')) return { QF_1: top(1, 0), QF_2: top(2, 1), QF_3: top(2, 0), QF_4: top(1, 1),
      QF_5: top(3, 0), QF_6: top(4, 1), QF_7: top(4, 0), QF_8: top(3, 1) }[rest];
    if (rest.startsWith('SF_')) { const k = Number(rest.slice(3)); return winnerOf[`${cat}_QF_${2 * k - 1}:${cat}_QF_${2 * k}`]; }
  }
  const sf = k => `${cat}_SF_${2 * k - 1}:${cat}_SF_${2 * k}`;
  return { F_1: winnerOf[sf(1)], F_2: winnerOf[sf(2)], B_1: loserOf[sf(1)], B_2: loserOf[sf(2)] }[rest];
};

const playoffRows = [];
SLOTS.forEach(([time, courts]) => {
  courts.split(' ').forEach((pair, i) => {
    if (pair === '-') return;
    const [c1, c2] = pair.split(':');
    const cat = c1.split('_')[0];
    const isPlayoff = !/^[A-Z]+_\d+$/.test(c1);
    const p1 = isPlayoff ? seat(cat, c1) : names[c1], p2 = isPlayoff ? seat(cat, c2) : names[c2];
    const [s1, s2] = score();
    rows.push([++num, '', time, `Court ${i + 1}`, c1, p1[0], p1[1], s1, c2, p2[0], p2[1], s2]);
    if (isPlayoff) {
      winnerOf[pair] = s1 > s2 ? p1 : p2; loserOf[pair] = s1 > s2 ? p2 : p1;
      playoffRows.push([c1, p1[0], p1[1], s1 > s2 ? 1 : 0, s1 > s2 ? 0 : 1, (s1 / Math.max(s2, 1)).toFixed(4), '']);
      playoffRows.push([c2, p2[0], p2[1], s2 > s1 ? 1 : 0, s2 > s1 ? 0 : 1, (s2 / Math.max(s1, 1)).toFixed(4), '']);
    } else { tally(c1, s1, s2); tally(c2, s2, s1); }
  });
});

const csv = table => table.map(r => r.join(',')).join('\r\n');
const matchesCsv = csv([['matchNumber', 'court', 'Schedule', 'CourtAssignment', 'teamCode1', 'team1Player1', 'team1Player2', 'team1Score', 'teamCode2', 'team2Player1', 'team2Player2', 'team2Score'], ...rows]);
const rrRows = Object.keys(names).map(code => { const r = rec[code];
  return [code, names[code][0], names[code][1], r.w, r.l, (r.pf / r.pa).toFixed(4), bracketOf(code)]; });
const standingsCsv = csv([['teamCode', 'player1', 'player2', 'wins', 'loss', 'quotient', 'bracket'], ...rrRows, ...playoffRows]);

const at = '2026-11-13T06:40:00.000Z';
const snapshot = {
  day: DAY, label: 'Nov 13', isLive: true, generatedAt: at, publishedAt: at,
  facilities: [{ name: 'Pickle Place', matchesCsv, standingsCsv, syncedAt: at, lastEditAt: at }],
  failedFacilities: [], staleFacilities: [],
};
fs.mkdirSync(path.join('_fixtures', EVENT), { recursive: true });
fs.writeFileSync(path.join('_fixtures', EVENT, 'finished.json'), JSON.stringify(snapshot, null, 2) + '\n');

const configPath = path.join('_fixtures', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.events[EVENT] = {
  type: 'standard', title: 'Angeles City Employees Sportsfest Pickleball', attendance: 'desks', scoreEntry: 'console',
  days: { [DAY]: { label: 'Nov 13', date: '2026-11-13', isLive: 'auto', facilities: [{ name: 'Pickle Place', sheetId: 'FIXTUREACSOSHEETID0000000000000000' }] } },
  display: { divisions: { '': '' }, events: { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' } },
};
fs.writeFileSync(configPath, JSON.stringify(config, null, 2) + '\n');

const champ = cat => `${cat} ${winnerOf[`${cat}_F_1:${cat}_F_2`][0].trim()}`;
console.log(`${rows.length} matches, ${rrRows.length} pairs; champions ${['MD', 'WD', 'XD'].map(champ).join(', ')}`);
```
