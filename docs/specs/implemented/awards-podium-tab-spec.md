# Spec — Awards tab (podium finishers + image export)

A fifth tab on `tools/match-control.html` that shows each category's podium —
gold, silver, bronze — and exports them as SAGE-branded PNGs: one card per
category, or the whole tournament as a single sheet.

**Status: built.** `tools/match-control.html` has the Awards tab, bye handling,
and both export paths implemented per this spec.

The podium is already decided by data the console has loaded. Every category
plays a Final (`_F`) and a Bronze match (`_B`), both carrying scores in the
day snapshot's `matchesCsv`. Nothing new needs to be synced, configured, or
entered by hand — this tab reads what is already there and draws it.

This spec is written to be implemented directly. Section 2 defines the data
rules, section 3 the tab, section 4 the image export, and section 5 the build
order. Sections 6–9 are acceptance, decisions, open questions, and scope.

---

## 1. Why this belongs in the console

| | |
|---|---|
| Path | `tools/match-control.html`, new `data-view="awards"` tab |
| Public URL | `/tools/match-control` (unchanged) |
| New config | none |
| New dependency | none |
| New sync surface | none |
| `sage-tools-api` change | none — no version bump, no deploy |

Three reasons it goes here rather than in the spreadsheet or a separate page:

1. **The data is already in the browser.** `MATCHES` (from `matchesCsv`) has
   every Final and Bronze match with both teams' player names and scores.
   Deriving the podium is a pure function over state the console already
   holds — no fetch, no poll, no new failure mode.
2. **The sheet's own `Awards` tab isn't synced.** Only `CSV` and
   `STANDINGSCSV` are pulled by `sage-tools-api`. Anything typed into the
   `Awards` tab today is invisible to every SAGE surface, and re-typing
   winners by hand at the end of a long day is exactly when transcription
   errors happen.
3. **The operator is already here at the moment of need.** The podium matters
   in the ten minutes between the last Final and the awarding ceremony —
   when someone is standing at the console watching Mission Control, not
   opening a spreadsheet.

> This is a **read-only, derived** tab. It writes nothing, publishes nothing,
> and needs no auth — same as Live Matches, Match Finder and Standings.
> Signed-out operators see it in full.

---

## 2. Where the podium comes from

### 2.1 Derived from matches, not configured

Everything below is computed from `MATCHES` — the parsed `matchesCsv` rows
already produced by `rowsToMatches()`. Each match object carries what the
derivation needs:

```js
{ num, time, court, liveCourt,
  t1, t1p1, t1p2, t2, t2p1, t2p2,
  t1Score, t2Score, played }
```

The existing helpers do all the classification, unchanged:

| Helper | Gives |
|---|---|
| `parseCode(code)` | `{ club, category, rest }` — handles both `type` values |
| `roundKeyword(rest)` | `'F'` for Finals, `'B'` for Bronze, `null` for pool |
| `matchInstanceOf(rest)` | `2` in `F(2)` — twice-to-beat instance, else `null` |
| `categoryLabel(category)` | `'LIWD'` → `"Low Intermediate Women's Doubles"` |
| `clubLabel(club)` | `'PNF'` → `"Pickle & Friends Community"` |
| `buildCategoryOrderIndex()` | the display order Standings already uses |

> **No new parsing.** If a code shape ever changes, it changes in `parseCode`
> and this tab follows automatically. A second, parallel notion of "which
> match is the final" is exactly the kind of drift the console spec's §2.2
> set out to remove.

### 2.2 The three places

For each distinct `category` token present in the day's matches:

| Place | Comes from |
|---|---|
| 🥇 Gold | **Winner** of the decisive Final (`roundKeyword === 'F'`) |
| 🥈 Silver | **Loser** of that same Final |
| 🥉 Bronze | **Winner** of the decisive Bronze match (`roundKeyword === 'B'`) |

A "team" resolves to its two player names (`t1p1`/`t1p2` or `t2p1`/`t2p2`)
plus, for `dual-meet` events, its club label from `parseCode(code).club`.

Winner is decided by score, not by position: `t1Score > t2Score` → team 1.
A played match with equal scores is a data error, not a tie — see §2.7.

### 2.3 Byes — how a walkover bronze is encoded

**In a twice-to-beat bracket there is often no bronze match to play** — the
third-place finisher is already determined by the bracket's structure. Rather
than special-case that in code, it is encoded in the data: the operator adds a
Bronze match row whose *opponent* is a **bye**, making the bronze medalist the
winner of that match by walkover.

This keeps §2.2's rule intact — bronze is always "winner of the Bronze
match" — with no branch for bracket format.

#### Detecting a bye

A match **side** is a bye when any of its identifying cells is the literal
text `BYE`, trimmed and case-insensitive:

```js
const BYE_RE = /^bye$/i;

function sideIsBye(code, p1, p2){
  return BYE_RE.test((code || '').trim())
      || BYE_RE.test((p1   || '').trim())
      || BYE_RE.test((p2   || '').trim());
}

function matchByeSide(m){
  const t1Bye = sideIsBye(m.t1, m.t1p1, m.t1p2);
  const t2Bye = sideIsBye(m.t2, m.t2p1, m.t2p2);
  if(t1Bye && t2Bye) return 'both';   // data error — see §2.7
  if(t1Bye) return 't1';
  if(t2Bye) return 't2';
  return null;
}
```

Checking the team code **and** both player cells is deliberate: the sheet
fills team codes and player names by formula, so whichever cell the operator
overrides with `BYE` must work. Do not require a particular one.

> Note `rowsToMatches()` rewrites a side's players to `'TBD'` when
> `t1p1 === t1 || t1p1 === ''`. A side typed as `BYE` in the player cell but
> left with its formula-generated team code will **not** be caught by that
> rule (`'BYE' !== 'PNF_LIWD_B'`), so `BYE` survives into `t1p1` and the test
> above sees it. A side where `BYE` is in the *team code* is caught by the
> `code` check. Both paths work.

#### What a bye means for the result

> **A bye-decided match is decisive without scores.** `rowsToMatches()` sets
> `played = t1Score !== null && t2Score !== null`, so a bye match with empty
> score cells has `played === false`. The podium derivation must therefore
> **not gate on `played` when a bye side is present** — the non-bye side wins
> outright.

Concretely, for any Bronze or Final match:

| Condition | Result |
|---|---|
| `matchByeSide(m)` is `'t1'` | team 2 wins (walkover) |
| `matchByeSide(m)` is `'t2'` | team 1 wins (walkover) |
| `matchByeSide(m)` is `null` and `m.played` | higher score wins |
| `matchByeSide(m)` is `null` and `!m.played` | pending |
| `matchByeSide(m)` is `'both'` | data error — §2.7 |

The bye side is **never** rendered as a person. It contributes no name, no
club, and no card row — it exists only to make the other side a winner.

### 2.4 "Decisive" — twice-to-beat and replays

`matchInstanceOf` exists because some brackets run a Final as `F(1)`/`F(2)`,
where the second match is only played if the first goes the wrong way. So:

> Among all matches for a category with the same `roundKeyword`, the decisive
> one is **the resolved instance with the highest instance number** (treating
> a missing instance as `0`), where *resolved* means `played` **or**
> bye-decided per §2.3. If none are resolved, that placing is pending.

One `filter` + `reduce`. A bracket that never needed `F(2)` reports `F(1)`'s
result correctly; one that did play `F(2)` reports the right champion rather
than the superseded first result.

### 2.5 Categories with no bracket

A `standard`-type event can have a category that never plays a Final — pure
round robin, standings decide it. When **no `F` match exists at all** for a
category, fall back to the top three `RR` standings rows, using the identical
sort Standings already applies:

```
wins desc, then quotient desc
```

Rows failing `isEmptyStanding()` (placeholder rows where `player1` equals the
team code) are excluded before taking the top three.

> Mark these podiums visibly as **"by standings"** rather than presenting them
> as a bracket result. An operator reading the card should never have to
> wonder which mechanism produced it.

### 2.6 Byes must not surface anywhere a match is presented as playable

A bye match is **not a match**. It is never played, never assigned a court,
and has no opponent. Rendering it alongside real fixtures is noise at best and
misleading at worst — an operator seeing `PNF_LIWD_B vs BYE` sitting unplayed
on the Live board will reasonably think something is wrong.

Exclude bye matches — and bye standings rows — from every existing surface:

| Surface | Function | Change |
|---|---|---|
| **Standings** — bronze block | `renderSharedStages()` (dual-meet, line ~2551) and `renderStageTables()` (all types, line ~2348) | Drop the whole `BRONZE` stage block for any category whose bronze was bye-decided; also drop any individual row that is itself a `BYE` |
| **Live Matches** | `renderLiveMatches()` / the facility court grouping | Filter out `matchByeSide(m) !== null` before grouping |
| **Match Finder** | `ticketHTML()` callers, and `teamMatches.findIndex(m => !m.played && !m.liveCourt)` (line ~3542) | Filter bye matches out of a team's match list so no "Next Up" ticket ever points at one |
| **Team index / autocomplete** | `rebuildTeamIndex()` | Never index `BYE` as a searchable pair |

Two rules, applied in that order, cover all of it:

1. **Row-level:** a standings row whose `teamCode`/`player1` is `BYE` is
   dropped wherever standings rows are consumed.
2. **Category-level:** if a category's decisive Bronze match is bye-decided,
   its `BRONZE` block is omitted from Standings entirely — there was no
   contest to report. The result still appears on the Awards tab, and the
   medalist still appears in their own Round Robin standings row.

> Rule 2 is why this is a category-level decision, not just a row filter.
> Dropping only the `BYE` row would leave a lone one-sided "Bronze Battle"
> block showing a single team with no opponent and no score — worse than
> showing nothing.

### 2.7 Incomplete and contradictory states are shown, never guessed

| Situation | Card shows |
|---|---|
| Final not yet played | Gold/Silver slots read **"Pending"**; Bronze fills independently if resolved |
| Bronze not yet played and not bye-decided | Bronze slot reads **"Pending"** |
| Final played, scores equal | Podium suppressed for that category; visible **"Check the score for match #N"** warning naming the match number |
| Both sides of a match are `BYE` | Same warning treatment, naming the match number |
| Team slot still a placeholder (`isEmptyStanding` shape) | Names render as **"TBD"**, consistent with `pairCell()` elsewhere |
| No `F` match and no RR standings | Category **omitted entirely** from the tab and from exports |

> This mirrors §2.4 of the console spec: a wrong podium that looks finished is
> worse than one that says it is not. Nothing here silently invents a winner.

### 2.8 The derivation function

The whole of §2 reduces to one pure function, which the tab and both export
paths all call:

```js
// -> [{ category, source, gold, silver, bronze, warning }]
//    source : 'bracket' | 'standings'
//    gold/silver/bronze : { p1, p2, club, clubLabel } | null   (null = pending)
//    warning : string | null
function buildPodiums(){ /* ... */ }
```

Ordered by `CATEGORY_ORDER_INDEX`, so the Awards tab, the per-category
exports, and the combined sheet all agree with the Standings tab's order.

---

## 3. The tab

### 3.1 Placement

A fifth button in the existing `#viewTabsWrap`, after Standings:

```html
<button class="view-tab" data-view="awards" type="button">Awards</button>
```

Wired the same way as the other four in the `viewTabs.forEach` click handler
(line ~3039): toggle `#awardsResults` on `view === 'awards'` and call
`renderAwards()`. Nothing about the existing four changes.

Placed before Mission Control deliberately — Awards is a spectator-facing
readout like the three tabs left of it; Mission Control is the operator's
control surface and stays rightmost.

### 3.2 Layout

One card per category, in `CATEGORY_ORDER_INDEX` order. Each card:

- Category label as the heading (`categoryLabel`)
- Three rows: gold, silver, bronze — medal-tinted rank badge, both player
  names, club name underneath for `dual-meet`
- A per-card **"Export image"** button
- A `by standings` chip when §2.5's fallback produced it
- A `walkover` chip on the bronze row when it was bye-decided

Above the cards, a summary bar:

- Count of categories with a complete podium vs total
- One **"Export whole tournament"** button
- When some categories are still pending, a plain line saying so, so an
  operator exporting mid-day knows the sheet will have gaps

Responsive: cards in a grid collapsing to one column under ~700px, same
`NARROW SCREENS` approach the rest of the page uses.

---

## 4. Image export

### 4.1 Canvas 2D, no library

`tools/match-control.html` today has **zero JS dependencies** — Google Fonts
is the only external resource. Keeping it that way rules out `html2canvas`,
and rules out a `.zip` bundler, which is the real reason bulk export is one
combined image (§7).

Drawing directly with the Canvas 2D API instead:

- No dependency, no CDN that can be down on venue wifi
- Deterministic output — the PNG does not depend on viewport size, zoom,
  scroll position, or which browser the operator happens to be on
- No `foreignObject`/HTML-rasterisation font quirks
- No canvas tainting, since nothing cross-origin is drawn

The cost is that the card design lives in drawing code rather than CSS —
acceptable for one fixed, deliberately-designed layout, and the same tradeoff
the scoresheet generator already makes server-side.

### 4.2 Geometry

Design at a logical **1080 × 1350** (social portrait), rendered at 2× into a
2160 × 2700 backing store and exported at that size:

```js
const EXPORT_W = 1080, EXPORT_H = 1350, EXPORT_SCALE = 2;
canvas.width  = EXPORT_W * EXPORT_SCALE;
canvas.height = EXPORT_H * EXPORT_SCALE;
ctx.scale(EXPORT_SCALE, EXPORT_SCALE);   // draw in logical units throughout
```

Every coordinate below is in logical units. Vertical rhythm:

| Band | Content |
|---|---|
| Top ~200px | SAGE mark, `S.A.G.E.` eyebrow, event title, day label/date |
| ~180px | Category name — Archivo Black, wrapped to two lines if long |
| 3 × ~250px | Gold, silver, bronze rows |
| Bottom ~120px | Facility/venue name, `sage-match-control.github.io` footer |

A bronze row that was bye-decided renders the medalist normally, with a small
`WALKOVER` tag where a score would otherwise sit. The bye itself is never
drawn.

### 4.3 The whole-tournament sheet

Same 1080 logical width, height computed from category count:

```
height = HEADER_H + (categories.length * ROW_H) + FOOTER_H
```

with a condensed per-category row (category name left, three placings right)
rather than the full card. One PNG, one download, readable as a results sheet.

> **Canvas size ceiling.** iOS Safari caps total canvas area near 16.7M
> pixels. At `EXPORT_SCALE = 2` and 2160px wide, that is roughly 7700px of
> backing height. Compute the area before allocating and **drop to
> `EXPORT_SCALE = 1` when it would exceed a safe budget** rather than
> producing a blank canvas — a silently-empty PNG is the failure mode to
> avoid. A ~20-category event stays well inside 2× regardless.

### 4.4 Fonts must be loaded before the first stroke

`ctx.font` silently falls back to a system font if the family is not yet
loaded, and there is no error to catch. The page renders Archivo Black in its
hero, but the specific weights this drawing needs may never have been
requested by any DOM node.

So, before drawing, explicitly load every face used and await them:

```js
await Promise.all([
  '400 96px "Archivo Black"',
  '700 44px "Barlow Condensed"',
  '600 34px "Barlow Condensed"',
  '600 30px "Inter"',
  '400 24px "Inter"',
].map(f => document.fonts.load(f)));
await document.fonts.ready;
```

> Skipping this is the single most likely way the exported image comes out
> wrong while the on-screen tab looks perfect — and it only shows up on a cold
> load, which is exactly the day-of condition.

### 4.5 The SAGE mark

The console's shield already exists as inline SVG in the hero
(`svg.crown-mark`, line ~1638). Reuse that artwork rather than transcribing it:

1. `cloneNode(true)` the element.
2. Set explicit `width`/`height` attributes — **Safari renders a blob SVG at
   zero size without them.**
3. Substitute the CSS-dependent colours, which do not resolve in a standalone
   document: `currentColor` → literal hex, `var(--court-dark)` → `#14263C`.
4. `XMLSerializer` → `Blob` (`image/svg+xml`) → object URL → `Image`.
5. `await img.decode()`, then `drawImage`.

Cache the decoded `Image` on first use so per-category exports in a row do not
re-decode it, and revoke the object URL once decoded. An inline SVG with no
external references does **not** taint the canvas, so `toBlob` stays available.

### 4.6 Branding

Drawn from the theme tokens at the top of the file, hard-coded as literals in
the drawing code (a canvas cannot read CSS custom properties):

| Element | Colour |
|---|---|
| Card background | `--navy-deep` `#0B1826` |
| Panel / rows | `--navy` `#14263C` |
| Accent, rank badges, rules | `--green` `#7CB92C` |
| Primary text | `#F6F7F2` (`--paper`) |
| Secondary text (club, venue) | `#9FB1AC` |
| Medal tints | gold `#D4AF37`, silver `#C0C6CC`, bronze `#B0763A` |

Typography matches the site: **Archivo Black** for the category headline and
eyebrow, **Barlow Condensed** for rank labels and metadata, **Inter** for
player names.

> Keep the medal tints for the rank badges only. The card's structure stays
> navy/green so it reads as a SAGE artefact first and a podium second.

### 4.7 Download mechanics

```js
canvas.toBlob(blob => {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = filename;
  document.body.appendChild(a); a.click(); a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 0);
}, 'image/png');
```

Filenames, lowercased and slugified:

```
<event-key>-<day-key>-<category>-podium.png
pnf-x-bup-dual-meet-pnf-x-bup-day1-liwd-podium.png

<event-key>-<day-key>-podium-all.png
```

The day key is included because the tab is day-scoped (§8) — two days of the
same event must not collide in an operator's Downloads folder.

---

## 5. Build steps

1. **Bye handling first, since it touches existing tabs.** Add `sideIsBye()`
   / `matchByeSide()` (§2.3) and apply the four exclusions in §2.6. Verify
   against the current PNF snapshot that nothing changes yet — no byes exist
   in it, so Live/Finder/Standings must render byte-identically. That is the
   regression guard before anything new is added.
2. **`buildPodiums()`** (§2.8), verified against the live `pnf-x-bup-day1`
   snapshot. All seven categories currently return pending — that is the
   correct answer before the event and is itself the first test.
3. **The tab.** Button, panel, `renderAwards()`, wired into the existing
   `viewTabs` handler. Screen layout and pending states complete before any
   export code exists.
4. **Per-category export.** Font preload, SAGE mark rasterisation, card
   drawing, download. Verify the PNG opens at 2160 × 2700 with correct fonts
   on a hard reload (cold font cache).
5. **Whole-tournament sheet**, including the canvas-area guard from §4.3.

No `sage-tools-api` change, so no `package.json` bump and no Cloud Run deploy.

---

## 6. Acceptance

**Bye handling (§2.3, §2.6)**

- [ ] A Bronze match with `BYE` in the opponent's team code yields that
      category's bronze medalist with no scores entered
- [ ] The same, with `BYE` in the opponent's *player name* cell instead
- [ ] The bye never appears on Live Matches, in Match Finder results, as a
      "Next Up" ticket, or in the search autocomplete
- [ ] Standings omits the whole `BRONZE` block for a bye-decided category
- [ ] The bronze medalist still appears in their own Round Robin standings row
- [ ] A match with `BYE` on **both** sides shows the warning naming the match
      number, and no podium for that category
- [ ] Against the current PNF snapshot (no byes present), Live Matches, Match
      Finder and Standings render exactly as they do today

**Podium derivation (§2.2, §2.4, §2.5, §2.7)**

- [ ] Current unplayed PNF snapshot: all seven categories listed, every
      placing "Pending", no fabricated winners, no console errors
- [ ] With a Final's scores filled in: gold/silver populate; bronze stays
      pending until its own match resolves
- [ ] `F(1)` and `F(2)` both played → reports `F(2)`'s winner
- [ ] `F(1)` played, `F(2)` not → reports `F(1)`'s winner
- [ ] Final played with equal scores → warning naming the match, podium
      suppressed
- [ ] A `standard`-type event with an RR-only category shows a top-three
      podium tagged **by standings**

**Export (§4)**

- [ ] Per-category PNG: 2160 × 2700, correct fonts on a cold load, SAGE shield
      present, club names shown for dual-meet
- [ ] A bye-decided bronze row shows the medalist plus a `WALKOVER` tag, and
      never the word `BYE`
- [ ] Whole-tournament PNG: every non-omitted category present, in the same
      order as the Standings tab
- [ ] A sheet that would exceed the canvas area budget falls back to 1× and
      still exports a readable image

**General**

- [ ] Tab is usable signed-out
- [ ] At 375px the cards stack to one column and the document does not scroll
      horizontally

---

## 7. Decisions taken

1. **Social portrait, 1080 × 1350**, rendered at 2×. Matches how these
   actually get shared after an event.
2. **Bulk export is one combined image**, not N files. Browsers throttle
   multi-file downloads, and bundling to `.zip` would put the first JS
   dependency on a page that has none.
3. **Top three only.** The bronze match's loser is not shown.
4. **Twice-to-beat bronze is encoded as a bye**, not special-cased in code, so
   §2.2's single rule ("bronze = winner of the Bronze match") holds for every
   bracket format.
5. **A bye-decided bronze removes the whole `BRONZE` block from Standings**
   for that category, not just the `BYE` row — a one-sided Bronze Battle block
   is worse than none.
6. Podium is **per selected day**, like every other tab.
7. Club shown as **text**, not a logo — same reasoning as console spec §2.1.
8. Tab is **always visible**, including mid-tournament, consistent with the
   console ignoring `isLive` for its own display.

---

## 8. Still open

- **Multi-day events.** Finals happen on the last day, so that day's Awards
  tab is the tournament's podium. Whether a multi-day event should offer an
  aggregate export spanning days is deferred until one needs it — the
  derivation is day-scoped and would need a cross-day snapshot fetch.
- **Special awards** (MVP, sportsmanship, most improved). The sheet's `Awards`
  tab may hold these; they are not derivable from match data and would need a
  new synced range.
- Whether the awarding ceremony wants a **landscape variant** for the venue
  displays in addition to the portrait social card.
- Whether byes should ever be used outside the Bronze match (e.g. a walkover
  in an earlier round). The detection in §2.3 is generic and would handle it,
  but only the Bronze case is specified and tested here.

---

## 9. Out of scope

- Any change to `sage-tools-api`, `events.json`, or the sync pipeline
- Publishing podium images anywhere automatically
- The public event `index.html` — this is an operator surface only
- Editing or overriding a derived podium by hand
- PDF output (the scoresheet pipeline is the place for print artefacts)
