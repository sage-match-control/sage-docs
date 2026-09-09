# Spec — Bracket Generator

Promote the per-event `bracket-generator.html` into a single evergreen tool at
`tools/bracket-generator.html`, branded as S.A.G.E. rather than as whichever
event it was copied for, and give it an **optional Event name field** that
supplies at run time the one thing the per-event copies got from build-time
token replacement.

| File | Repo | Change |
| --- | --- | --- |
| `tools/bracket-generator.html` | `sage-match-control.github.io` | new — the tool (§3–§6) |
| `tools/index.html` | `sage-match-control.github.io` | one row on the ops roster (§8) |
| `_templates/dual-meet-template/bracket-generator.html` | `sage-match-control.github.io` | deleted (§7) |
| `_templates/standard-tournament-template/bracket-generator.html` | `sage-match-control.github.io` | deleted (§7) |
| `events/pnf-x-bup-dual-meet/bracket-generator.html` | `sage-match-control.github.io` | replaced with a redirect stub (§7.2) |
| `_templates/CLAUDE.md` | `sage-match-control.github.io` | §8 removed, §2 and §3 amended (§7.3) |
| `docs/features/bracket-generator.md` | `sage-docs` | new — organizer page (§11.1) |
| `docs/technical/bracket-generator.md` | `sage-docs` | new — its technical counterpart (§11.2) |
| six further doc files | `sage-docs`, site, project root | §11.3–§11.9 |

**No running code changes.** No new endpoint, no auth, no CORS, no
`event-data` read, no `package.json` bump and no Cloud Run deploy. The tool
stays what it already is: one self-contained HTML file that fetches nothing.

## Build order

Work in this order. Every step leaves a page that loads and draws, so the
result can be checked in a browser after any of them.

1. **Fork the file.** Copy `_templates/dual-meet-template/bracket-generator.html`
   to `tools/bracket-generator.html`. That copy specifically — it is the
   current, S.A.G.E.-themed one. The `events/pnf-x-bup-dual-meet/` copy is a
   palette behind (§1) and the archive copy is older still; forking either is
   the one mistake that silently undoes the theming.
2. **§4 — the chrome.** Replace all six token sites, add the logo to the
   eyebrow row and the shared favicon block. `grep '{{' tools/bracket-generator.html`
   must come back empty before moving on.
3. **§4.1 + §6 — one palette source.** Add the `THEME` banner, promote the four
   badge pairs to tokens, convert the canvas to `cssVar`. Do this *before* §3:
   it touches the same export functions the event name work does, and doing it
   second means editing them twice.
4. **§3 — the Event name field.** Markup, state, the render sites in §3.3,
   canvas truncation, escaping, persistence, `?event=`.
5. **§7 — retire the copies.** Delete both template copies, write the redirect
   stub, amend `_templates/CLAUDE.md` §2, §3 and §8.
6. **§8 — the ops roster row.**
7. **§11 — the docs.** Nine files across three repos. Not a follow-up task.
8. **§10 — run the acceptance checklist.**

Steps 5, 6 and 7 are independent of 2–4 and can land separately; 2, 3 and 4 are
strictly ordered.

### Conventions to match

`sage-match-control.github.io` has no build step, no framework and no
dependencies. Keep it that way:

- One self-contained file — inline `<style>`, inline `<script>`, no bundler,
  no CDN, no new `<link>` beyond the two Google Fonts stylesheets already there.
- Match the file's existing idiom: `const`/arrow functions, template literals,
  `escapeHtml()` on anything reaching `innerHTML`, comments that explain *why*
  rather than restating the line below them.
- Do not reformat or "tidy" the ~1,100 lines being carried across. Every
  unrelated diff hunk is noise in a review of a file that was copied, not
  written.

---

## 1. Current state

Four copies of the same ~1,118-line page exist:

| Copy | State |
| --- | --- |
| `_templates/dual-meet-template/bracket-generator.html` | S.A.G.E.-themed, current |
| `_templates/standard-tournament-template/bracket-generator.html` | byte-identical to the above, by design |
| `events/pnf-x-bup-dual-meet/bracket-generator.html` | **stale** — still the pre-S.A.G.E. dark-neon palette, though its own header comment claims byte-identity with the template |
| `events/archives/bkl-cup-2026/bracket-generator.html` | frozen archive |

The page takes a category name, a list of pairs (one per line) and a bracket
count; shuffles the pairs behind a ~3s deceleration animation; deals them
round-robin into lettered brackets; and exports the result as a hand-drawn
PNG or a plain-text file.

### 1.1 Nothing in it is per-event

There is no configuration block, no sheet access, no `event-data` fetch, no
query string. `_templates/CLAUDE.md` §2 step 4 already says so outright:
"`bracket-generator.html` needs no config at all."

The only event-specific content is three tokens, in six places:

| Location | Token use |
| --- | --- |
| `<title>` | `{{EVENT_TITLE}} — Bracket Draw` |
| `.eyebrow` | `{{EVENT_TITLE}} · {{EVENT_DATE_RANGE}} · {{VENUE}}` |
| `h1.title` | `{{EVENT_TITLE}} <span class="accent">Bracket Draw</span>` |
| `<footer>` | `{{EVENT_TITLE}} · Bracket Draw Tool` |
| `exportAsText()` | `'{{EVENT_TITLE}} \u00b7 Bracket Draw Tool\n'` |
| `exportAsImage()` | canvas header eyebrow and canvas footer |

All six are captions. None of them changes what the tool computes.

### 1.2 Nothing links to it

Neither the event `index.html` nor `tools/index.html` links to the page. It is
reachable only by knowing the URL. The `pnf-x-bup-dual-meet` copy drifting a
full palette behind the template — while still carrying a comment asserting it
had not — is the predictable result: a page nobody opens is a page nobody
notices is wrong.

### 1.3 What the duplication costs

`_templates/CLAUDE.md` §8 exists solely to police the two template copies
staying byte-identical, and both files carry a header comment repeating the
rule. Every future change to the tool is a three-file edit — two templates plus
each live event's copy — with no mechanism that catches a missed one.

---

## 2. Why it moves

Three reasons, in order of weight:

1. **It is already an evergreen tool wearing event clothes.** A tool whose
   entire input is typed by hand into the page belongs beside the other
   hand-input tools — `scoresheet-generator.html`,
   `tournament-calculator.html` — not inside a folder that gets frozen when
   the event ends.
2. **One copy cannot drift.** The §8 mirroring rule, the two header comments
   and the per-event copy all disappear. Instantiating a template stays a
   single `cp -r`; it just copies one file fewer.
3. **It becomes findable.** A row on the ops roster is the difference between
   a tool the team uses and a URL one person remembers.

The counter-argument — that a per-event copy is stamped with the event name
for free — is real, and §3 is the answer to it.

---

## 3. The Event name field

The one capability the per-event copies had that a shared tool loses: the
exported PNG and text file carried the event's name without anyone typing it.
An operator posting a draw to a group chat wants that stamp; a shared tool that
silently drops it is a downgrade.

So the tool asks for it — **optionally**, once per session, and remembers it.

### 3.1 Markup

A new first field in `.input-panel`, above **Category name**:

```html
<div class="field">
  <label class="field-label" for="eventNameInput">Event name <span class="field-optional">optional</span></label>
  <input id="eventNameInput" class="field-input" type="text"
         placeholder="e.g. BKL Pickleball Cup 2026" autocomplete="off" maxlength="80" />
  <div class="field-meta">
    <span class="field-hint">Stamped on the exported image and text file. Leave blank for plain S.A.G.E. branding.</span>
  </div>
</div>
```

`.field-optional` is a new inline chip on the label — uppercase, `--ink-soft`,
Barlow Condensed, matching the existing `.field-label` treatment.
`.field-hint` reuses the `.field-meta` row the pairs counter already sits in.

`maxlength="80"` is a guardrail, not the wrap strategy — §3.4 still truncates.

### 3.2 State

```js
const eventNameInput = document.getElementById('eventNameInput');
let currentEventName = '';
```

`currentEventName` is captured in `generateBrackets()` alongside
`currentCategory`, from the same `.trim()`:

```js
currentCategory  = categoryInput.value.trim();
currentEventName = eventNameInput.value.trim();
```

Capturing at draw time rather than reading the input at export time is
deliberate and matches `currentCategory`: what the operator saw settle is
exactly what exports, even if they keep typing afterwards.

### 3.3 Where it renders

| Surface | Event name set | Blank |
| --- | --- | --- |
| `<title>` | *(static)* `Bracket Generator — S.A.G.E.` | same |
| Page `.eyebrow` | *(static)* `S.A.G.E. · Match Control Experts` | same |
| Page `h1.title` | *(static)* `Bracket <span class="accent">Generator</span>` | same |
| Page `<footer>` | *(static)* S.A.G.E. branding | same |
| Results header | new `#resultEvent` line above `#resultTitle` | element hidden |
| Text export line 1 | `<EVENT NAME>`, then category on line 2 | category on line 1, as today |
| Text export footer | `<event name> · Bracket Draw` | `S.A.G.E. Bracket Draw` |
| Canvas header eyebrow | `<EVENT NAME> · BRACKET DRAW` | `S.A.G.E. · BRACKET DRAW` |
| Canvas footer | `<event name> · Powered by S.A.G.E. Match Control Experts` | `Powered by S.A.G.E. Match Control Experts` |
| Export filename | `<event>-<category>-brackets.{png,txt}` | `<category>-brackets.{png,txt}` |

**The page's own chrome never takes the event name.** Title, eyebrow, `h1` and
footer are fixed S.A.G.E. branding — this is a tool, and it is the same tool
whichever event is being drawn. The event name is *output* branding: it belongs
on the artifact that leaves the page.

The `#resultEvent` line exists so the operator can see what the export will say
before they export it, without needing a separate preview.

### 3.4 The canvas needs truncation the tokens never did

`{{EVENT_TITLE}}` was replaced at instantiation with a known-short string. A
typed event name is arbitrary, and `ctx.fillText` neither wraps nor clips — a
long name paints straight off both edges of the PNG.

Both canvas surfaces therefore run through the existing helper:

```js
const eyebrowText = (currentEventName ? currentEventName.toUpperCase() + ' \u00b7 ' : 'S.A.G.E. \u00b7 ') + 'BRACKET DRAW';
ctx.fillText(truncateToWidth(ctx, eyebrowText, width - 120), width / 2, 42);
```

and likewise for the footer, measured at the footer's own font. `truncateToWidth`
already exists and is already used for the category title; this is the same
call, not new machinery.

### 3.5 Escaping

The event name reaches three sinks, and all three are safe by construction:

- `#resultEvent` — assigned with `textContent`, like `#resultTitle`.
- `ctx.fillText` — canvas takes a string, not markup.
- the text export and the download filename — `slugify()` strips everything
  outside `[a-z0-9-]`.

**It must never be interpolated into an `innerHTML` string.**
`renderBracketGrid()` builds markup by template literal and is the one place in
the file where that rule could be broken; it has no reason to reference the
event name, and must not start.

### 3.6 Persistence

An operator draws eight categories for one event in one sitting. Retyping the
event name eight times is exactly the friction that makes them stop bothering,
at which point the field has bought nothing.

So the event name — and **only** the event name — is remembered:

```js
const STORAGE_EVENT_KEY = 'sage-bracket-event-name';
function saveEventName(v){ try{ localStorage.setItem(STORAGE_EVENT_KEY, v); } catch(e){} }
function loadEventName(){ try{ return localStorage.getItem(STORAGE_EVENT_KEY) || ''; } catch(e){ return ''; } }
```

Written on `change` (not on every keystroke), read on page load to prefill the
input. Wrapped in `try/catch` because `localStorage` throws in some
private-browsing modes — the same treatment `control-center.html` and
`tournament-calculator.html` already give it.

Category and pairs are **not** persisted. They change every draw, and a
prefilled pair list is a live hazard: an operator who does not notice it draws
the wrong category's brackets.

### 3.7 `?event=` prefill

The input also accepts a query parameter:

```
/tools/bracket-generator.html?event=BKL%20Pickleball%20Cup%202026
```

`?event=` wins over the stored value and is itself stored, so a link handed to
an operator sets them up for the whole session. This matches the deep-link
convention `tools/scoresheet-generator.html` already established (see
[Scoresheet Generator event picker](scoresheet-event-picker-spec.md) §5.7) and
costs four lines; specifying it now avoids reopening the file when Control
Center or a workbook menu wants to link in.

Nothing else is deep-linkable. Category and pairs stay typed.

---

## 4. Branding

The page keeps its **own hero** — the paddle SVG, the oversized title, the
subtitle — *and* takes the family identity the other tools carry.

Keeping the hero matters because this page is a *stage*, not a form. The
scoresheet generator is four numbered steps nobody watches; the ~3s shuffle
exists so a room can watch a draw land, and `runShuffleAnimation()` scrolls the
page to put the whole grid in view for exactly that reason. Collapsing it to a
utility bar would undercut the one thing it is shaped around.

But "its own hero" must not mean "looks like it came from somewhere else." So
the hero gains the S.A.G.E. mark:

- `<title>` → `Bracket Generator — S.A.G.E.`
- The eyebrow row carries `/assets/logo.png` beside
  `S.A.G.E. · Match Control Experts` — the same mark and the same words
  `scoresheet-generator.html` opens with, in the hero's own proportions rather
  than the bezel's.
- `h1.title` → `Bracket <span class="accent">Generator</span>`, keeping the
  display weight and the accent span
- `<footer>` → `S.A.G.E. Bracket Generator` over
  `Powered by S.A.G.E. Match Control Experts`
- the shared favicon block the other tools carry (`/assets/favicons/…` plus
  `site.webmanifest`), which the per-event copies never had

Someone landing on any of the three tools sees the same mark and the same
eyebrow in the first line; what differs below it is the shape of the work.

### 4.1 Re-skinning is a one-block edit

The tool has to survive being re-skinned for a specific event on request, so
**every colour it paints — screen and export alike — resolves from one `:root`
block.** Today that is only half true: the CSS resolves through tokens, and the
PNG export duplicates the palette in JS literals (§6).

Requirements:

1. One `THEME` banner at the top of the `<style>`, matching the banner the
   event templates and the other tools carry, holding the brand tokens and the
   role aliases (`--court`, `--cork`, `--amber`, `--muted`, …) that let the
   rules below read by intent rather than by hue.
2. **The four bracket badge colours become tokens**, not JS literals —
   `--badge-N-fill` / `--badge-N-text` for N in 0..3. The `.badge-*` CSS rules
   and the canvas export both read them, so they cannot drift apart. Two of the
   pairs ship deliberately darkened for contrast (§6); as tokens they keep those
   values while staying in the one block a re-skin edits.
3. **The canvas export reads the live values** rather than restating them —
   `getComputedStyle(document.documentElement).getPropertyValue('--navy')` and
   so on for the background, glows, header and footer fills. Re-skinning
   `:root` re-skins the exported PNG with no second edit.
4. A short comment in that banner naming what a re-skin changes and what it must
   not: the tokens, and nothing else.

This is a strict improvement on the status quo even without a re-skin — it
deletes the entire class of bug where the exported image stops matching the page
it came from.

The palette values themselves do not change. The default stays the S.A.G.E.
house palette, and the accessibility floor in §6 is a property of whatever
palette is loaded, not of these particular hex values.

---

## 5. What does not change

Everything below the input panel is carried across as-is:

- the shuffle animation, its deceleration curve, and the
  `prefers-reduced-motion` branch that skips it
- `dealBrackets()` — shuffle, then round-robin into buckets, so bracket sizes
  differ by at most one
- the bracket-count clamp to the pair count, and its inline message
- `bracketLabel()`'s A, B, … Z, AA, AB spreadsheet-column naming
- the responsive `cols-2` / `cols-3` grid
- the canvas export's layout, word-wrapping and colour handling

**No new dependency.** The page keeps its two Google Fonts links and nothing
else — no CDN, no bundler, no service worker.

---

## 6. The canvas reads the palette instead of duplicating it

This is what makes §4.1's one-block re-skin real, and it deletes a standing
hazard: `_templates/CLAUDE.md` §8 currently warns that the PNG export's palette
lives in JS literals, is hand-kept in step with the `.badge-*` CSS rules above
it, and that changing one without the other makes the exported image stop
matching the page it came from. Reading the tokens removes the possibility
rather than documenting it.

### 6.1 Promote the badge colours to tokens

Four of the eight badge values are already tokens; four are inline literals.
Replace `bracket-generator.html:353-356`:

```css
.badge-0{background:var(--navy);   color:var(--white);}
.badge-1{background:var(--green);  color:var(--navy);}
.badge-2{background:#4C7A19;       color:var(--white);}
.badge-3{background:#6C5CE0;       color:var(--white);}
```

with rules that read eight new `:root` tokens:

```css
.badge-0{background:var(--badge-0-fill); color:var(--badge-0-text);}
.badge-1{background:var(--badge-1-fill); color:var(--badge-1-text);}
.badge-2{background:var(--badge-2-fill); color:var(--badge-2-text);}
.badge-3{background:var(--badge-3-fill); color:var(--badge-3-text);}
```

declared in the `THEME` block with today's values:

```css
--badge-0-fill:var(--navy);  --badge-0-text:var(--white);
--badge-1-fill:var(--green); --badge-1-text:var(--navy);
--badge-2-fill:#4C7A19;      --badge-2-text:var(--white);
--badge-3-fill:#6C5CE0;      --badge-3-text:var(--white);
```

> **Keep the accessibility note with these four lines.** Each fill is paired
> with a text colour clearing 4.5:1 against it. Two do not clear it at their
> natural values — `--green-dark` under white is 4.3:1, and the stock purple is
> 3.3:1 — so `--badge-2-fill` and `--badge-3-fill` ship **darkened**. They are
> not `var(--green-dark)` and they are not the stock purple, on purpose. A
> re-skin may change them; it must re-check the contrast when it does.

### 6.2 The canvas resolves tokens at export time

Add one helper and use it for every colour `exportAsImage` and
`drawBracketCard` paint:

```js
// The exported PNG is hand-drawn to a canvas, which takes colour strings, not
// CSS variables — so it resolves the same :root tokens the page renders from.
// This is what keeps a re-skin a one-block edit (§4.1): change the tokens and
// the export follows, with no second list of colours to keep in step.
// `fallback` matters: a token that resolves empty (a malformed re-skin) would
// make `ctx.fillStyle = ''` a silent no-op, leaving whatever colour was set
// last. Falling back to the shipped value degrades to today's palette instead.
const cssVar = (name, fallback) =>
  getComputedStyle(document.documentElement).getPropertyValue(name).trim() || fallback;
```

Every literal below `bracket-generator.html:600` maps to an existing token:

| Literal | Token | Where |
| --- | --- | --- |
| `#14263C` | `--navy` | `BADGE_FILL[0]`, card title, header title |
| `#7CB92C` | `--green` | `BADGE_FILL[1]` |
| `#4C7A19`, `#6C5CE0` | `--badge-2-fill`, `--badge-3-fill` | §6.1 |
| `#FFFFFF` | `--white` | `BADGE_TEXT`, card body, gradient midpoint |
| `#F6F7F2` | `--paper` | background gradient stops |
| `#ECEEE6` | `--paper-dim` | pair-number chip |
| `#D9DED2` | `--line` | card border |
| `#5B6B74` | `--ink-soft` | card count, header meta, footer |
| `#5C8F1F` | `--green-dark` | header eyebrow |

`BADGE_FILL` / `BADGE_TEXT` become functions of the tokens rather than literal
arrays, built once per export rather than at module load — a token read at load
time would miss a re-skin applied afterwards:

```js
const badgeFills = [0,1,2,3].map(i => cssVar(`--badge-${i}-fill`));
const badgeTexts = [0,1,2,3].map(i => cssVar(`--badge-${i}-text`));
```

The two glows are the one case needing more than a lookup — they are a token at
low alpha (`rgba(124,185,44,.10)` is `--green` at 10%, `rgba(20,38,60,.05)` is
`--navy` at 5%). Add a hex-to-rgba shim beside `cssVar` and pass the token
through it, so a re-skinned green tints the export's glow too:

```js
const cssVarAlpha = (name, a) => {
  const hex = cssVar(name);
  const m = /^#?([0-9a-f]{6})$/i.exec(hex);
  if(!m) return `rgba(0,0,0,${a})`;   // token isn't a plain hex — no glow rather than a wrong one
  const n = parseInt(m[1], 16);
  return `rgba(${n >> 16 & 255},${n >> 8 & 255},${n & 255},${a})`;
};
```

Every call site passes the literal it replaces as the fallback —
`cssVar('--navy', '#14263C')` — so the mapping table above doubles as the
fallback list and a broken token degrades to exactly today's export.

---

## 7. Retiring the copies

### 7.1 Templates

Delete both `_templates/*/bracket-generator.html`. Neither template's
`index.html` links to it, so nothing else in either folder changes.

### 7.2 The live event copy

`events/pnf-x-bup-dual-meet/bracket-generator.html` becomes a redirect stub to
`/tools/bracket-generator.html`, following the precedent set by
`tools/match-control.html` when the console was renamed — any bookmark or
pasted link keeps working.

`events/archives/bkl-cup-2026/bracket-generator.html` is **left exactly as it
is.** Archived events are frozen; a redirect there would send someone looking
at a 2026 archive to a tool that no longer says 2026 on it.

### 7.3 `_templates/CLAUDE.md`

- **§8** — deleted whole. Its canvas-palette warning is not relocated but
  *superseded*: §6 removes the duplication the warning was about. Only the
  contrast note survives, and it lives beside the tokens it constrains (§6.1).
- **§2 step 4** — drop "`bracket-generator.html` needs no config at all."
- **§2 step 5** — drop `bracket-generator.html` from the
  two-places-that-don't-resolve-through-`:root` warning, leaving only
  `schedule.html`'s `CAT_META`.
- **§3** — the `{{EVENT_TITLE}}` row's meaning loses ", bracket generator", and
  the intro line loses "and — for a few of these — `bracket-generator.html`".

After this, `grep -r 'bracket-generator' _templates/` returns only the
incidental `.badge-*` cross-reference in `index.html` and the canvas note in
`schedule.html`, both of which stay accurate.

---

## 8. The ops roster row

Add to `tools/index.html` under **Pre-Tournament Prep**, after Tournament Time
Calculator:

```html
<div class="cap-row">
  <div>
    <p class="cap-name"><a href="/tools/bracket-generator">Bracket Generator</a></p>
    <p class="cap-desc">Paste in a category's pairs, set how many brackets, and let the draw run on screen. Exports a printable image or a plain-text list, stamped with the event name.</p>
  </div>
  <span class="status"><span class="dot"></span>Live</span>
</div>
```

**Pre-Tournament Prep, not day-of.** A draw is run once per category before
play starts; it reads nothing live and touches no event data.

---

## 9. Decisions

### 9.1 The event name is optional, and blank is a first-class state

Making it required would be one line shorter and wrong. Two real cases have no
event name to give: a practice or dry-run draw, and a club drawing an internal
ladder that is not a tournament. Both should produce a clean S.A.G.E.-branded
export, not a validation error or a PNG reading `UNTITLED EVENT`.

### 9.2 Remembered, not asked for again

See §3.6. The alternative — a modal on first load — is worse: it interrupts the
one case the tool must serve fastest, which is an operator who opened it to
settle a draw with people waiting.

### 9.3 Rejected — keep a per-event copy *and* add the tool

Suggested by the fact that an event's own folder is where an operator already
has a tab open. Rejected: it keeps every cost in §1.3 while adding a second
place the tool can be found in two different states. The redirect stub (§7.2)
gets the same convenience with one copy.

### 9.4 Rejected — a PWA, like the calculator

`tournament-calculator.html` has a service worker and a manifest because venue
wifi is unreliable and the calculator is used standing in a gym. The bracket
generator has the same profile and would benefit — but `tools/sw.js` is
deliberately scoped to the calculator's own page path, and its header comment
is explicit that everything else in `/tools/` must pass straight to the
network. Widening that scope is its own change with its own cache-invalidation
questions. Out of scope here; noted as a candidate.

### 9.5 Rejected — pulling pairs from `event-data`

The snapshot holds matches and standings, not registration lists, so the pairs
a draw needs are not in it. Even if they were, a draw happens before the event
has published anything. Hand-pasting is correct here, not a limitation.

---

## 10. Acceptance checklist

**The move**

- [ ] `/tools/bracket-generator.html` loads with no console errors and no
      network request beyond the two Google Fonts stylesheets.
- [ ] `grep -r '{{' tools/bracket-generator.html` returns nothing.
- [ ] `/events/pnf-x-bup-dual-meet/bracket-generator.html` redirects to the
      tool; the `bkl-cup-2026` archive copy still opens unchanged.
- [ ] Neither `_templates/` folder contains `bracket-generator.html`, and
      instantiating either template still needs only `cp -r`.
- [ ] `_templates/CLAUDE.md` has no §8, and its §2/§3 no longer mention the
      bracket generator.
- [ ] The Bracket Generator row appears on `/tools/` and its link resolves.

**The event name field**

- [ ] Blank event name: draw, then export both formats. The PNG eyebrow reads
      `S.A.G.E. · BRACKET DRAW`, the PNG footer omits any event name, the text
      file starts with the category, and filenames are `<category>-brackets.*`.
- [ ] Event name set: both exports carry it, the `#resultEvent` line shows it,
      and filenames are `<event>-<category>-brackets.*`.
- [ ] An 80-character event name is truncated with an ellipsis inside the PNG
      canvas — nothing paints past either edge, at both the eyebrow and the
      footer font size.
- [ ] Editing the event name *after* a draw does not change that draw's header
      or its exports until the next draw.
- [ ] Reloading the page prefills the last event name; the category and pairs
      fields come back **empty**.
- [ ] `?event=Foo` prefills and overrides the stored value, and persists to the
      next visit without the parameter.
- [ ] An event name containing `<script>`, `&` and `"` renders literally in the
      results header and the PNG, and produces a slug-safe filename.
- [ ] With `localStorage` unavailable (private window, site data blocked) the
      page still loads and draws; only the prefill is lost.

**One palette source (§4.1, §6)**

- [ ] `grep -nE '#[0-9A-Fa-f]{6}' tools/bracket-generator.html` returns hits
      only inside the `THEME` block and as `cssVar` fallbacks — no colour
      literal survives anywhere else in the JS.
- [ ] The exported PNG's bracket-header colours match the on-page `.badge-*`
      colours exactly, before and after a re-skin.
- [ ] Changing `--navy`, `--green` and `--paper` in `:root` alone re-skins the
      page **and** the exported PNG — including both background glows — with no
      second edit anywhere.
- [ ] Deleting a token's declaration leaves the export rendering in the shipped
      palette rather than blank or mis-filled.
- [ ] `--badge-2-fill` and `--badge-3-fill` are still the darkened values, not
      `var(--green-dark)` and not the stock purple.

**Regression**

- [ ] The shuffle animation, its scroll-into-view, and the
      `prefers-reduced-motion` skip all behave as they do today.
- [ ] Bracket sizes still differ by at most one; the count still clamps to the
      pair count with its inline message.

---

## 11. Docs to write

Part of the work, not a follow-up. Nine files, in three repos.

**Present tense throughout.** Describe what the system does, not what it used
to do or when it changed — git history and this spec's §13 already carry that.
No page in `features/` mentions a file path, a token, a CSS variable or a repo;
that split is the whole point of the two sections.

### 11.1 New — `sage-docs/docs/features/bracket-generator.md`

The organizer-facing page. Cover, in this order:

- **What it's for** — settling who plays in which bracket, in front of the room
  if you want, without wrestling a spreadsheet into a diagram.
- **How you use it** — type the category, paste the pairs one per line, say how
  many brackets, press the button. The draw runs on screen and lands on a
  result; *Randomize again* re-draws from the same pairs.
- **The event name** — optional, remembered between draws, and what it changes:
  it appears on the image and the text file you export, not on the tool itself.
  Say plainly that leaving it blank is fine.
- **What comes out** — an image to print or post, and a plain-text list. Name
  the filenames, since that is how an operator finds them again after exporting
  eight categories.
- **What it doesn't do** — it doesn't know your event. Pairs are typed in, no
  registration list is pulled, nothing is published anywhere, and the draw is
  uniformly random with no seeding or protected pairings. This paragraph is
  load-bearing: it is what stops someone expecting the draw to avoid pairing
  club-mates.

Same evergreen framing as `scoresheet-generator.md` and
`tournament-calculator.md` — one tool, every event, not a per-event page. End
with the house footer: `**Technical:** [bracket generator](../technical/bracket-generator.md)`.

### 11.2 New — `sage-docs/docs/technical/bracket-generator.md`

Every `features/` page has a `technical/` counterpart; this is it. Short — the
tool has no pipeline. Cover:

- One self-contained static file, no build step, no dependency beyond two
  Google Fonts stylesheets, no network call at run time.
- The deal: shuffle, then round-robin into buckets, so sizes differ by at most
  one; the count clamps to the pair count.
- **The one-palette-source arrangement (§4.1, §6)** — that the PNG is
  hand-drawn to a canvas and resolves the same `:root` tokens the page renders
  from, so re-skinning is a single block edit and the export cannot drift from
  the page. Carry §6.1's contrast note here too: `--badge-2-fill` and
  `--badge-3-fill` ship darkened and a re-skin must re-check them.
- Where the event name is stored (`localStorage`, per browser, best-effort) and
  that `?event=` prefills it.
- That it replaced a per-event copy, and the redirect stub that keeps old links
  working.

Footer: `**Features:** [bracket generator](../features/bracket-generator.md)`,
matching the direction the other technical pages point.

### 11.3 `sage-docs/docs/features/README.md`

Index the new page under **For tournament organizers & day-of operators**,
after Tournament Calculator, in the existing `- **[Name](file.md)** — one-line
description` form.

### 11.4 `sage-docs/docs/technical/README.md`

Index the new technical page alongside `scoresheet-pipeline.md` and
`tournament-calculator.md`, matching whatever grouping that file already uses.

### 11.5 `sage-docs/docs/features/tournament-hub.md`

Delete the `## Bracket Generator` section (currently at :51). The tool is not
part of the public event page and never was — Tournament Hub is what players
and spectators see, and this is an organizer tool. Do not leave a stub heading;
if the surrounding prose reads as though something is missing, a single
sentence pointing at the new page is enough.

### 11.6 `sage-docs/docs/features/preparing-an-event.md`

Add the draw to **§1 Plan it** or **§2 Build the scoring workbook**, wherever
it sits in the real order of operations — brackets are drawn before the
workbook's category tabs can be filled in, so it belongs early. One or two
sentences and a link; this page is a sequence, not a tool description.

### 11.7 `sage-docs/docs/specs/implemented/event-templates-spec.md`

Two different kinds of edit in one file, and they are not treated the same:

- **Runbook content is corrected.** §1's placement tree (:34, :38) drops
  `bracket-generator.html` from both template folders, and §9's instantiation
  steps (:514) drop the line about renaming its sibling QR image placeholder.
  People follow these; they have to be true.
- **Decision records are annotated, not rewritten.** §3's settled-up-front line
  (:22) and **D3** (:114) recorded a decision that was correct when it was
  made. Mark D3 superseded by this spec with a one-line note saying the tool
  moved to `tools/` and why — the same treatment a divergences section gives.
  Deleting it would erase the reasoning rather than update it.
- §1's "byte-identical in both templates… add a comment at the top of both
  copies" paragraph (:55) goes with the copies.

### 11.8 `D:\Coding Projects\SAGE\CLAUDE.md`

Add `bracket-generator.html` to the `tools/` list in the
`sage-match-control.github.io` section, in the same shape as its neighbours.
Nothing in the `sage-tools-api` or `event-data` sections changes.

### 11.9 `sage-match-control.github.io/_templates/CLAUDE.md`

Already specified in §7.3 — §8 deleted, §2 steps 4 and 5 and §3's token table
amended. Listed here so the doc pass has one complete inventory.

---

## 12. Out of scope

- Offline/PWA support (§9.4).
- Seeded or otherwise non-random draws — snake seeding, keeping club-mates
  apart in round one, byes. The current deal is uniformly random by design;
  anything else is a different feature with its own spec.
- Reading pairs from `event-data` (§9.5).
- Writing a draw back anywhere. The tool exports files; it publishes nothing.
- A standard-tournament variant. One tool serves both event shapes because
  nothing in it knows about clubs.

The last two — writing a draw back, and the event-shape split — are picked up
in [workbook handoff](../not-started/bracket-generator-workbook-handoff-spec.md), which
specifies a SAGE menu route into this tool and what a dual-meet workbook's
roster scaffold actually wants from a draw. Not built: it waits on a
standard-tournament sheet generator.

---

## 13. Divergences

- **§4's hero mark placement is inverted.** The spec called for keeping the
  hero's own paddle-and-shield crest and adding `/assets/logo.png` beside the
  eyebrow text. Built the other way instead: the crest is replaced outright by
  `/assets/logo.png` in the hero-mark position, and the eyebrow stays plain
  text (`S.A.G.E. · Match Control Experts`) with no image in it. One S.A.G.E.
  mark appears once, at the top of the hero, rather than twice at two sizes.
- **§10's 80-character truncation check doesn't exercise the mechanism at
  that length.** `truncateToWidth()` is wired exactly as specified and does
  truncate with an ellipsis once text actually overflows the canvas width
  (verified with an artificially wide string) — but at the eyebrow/footer's
  actual font size (12px/11px Barlow Condensed), even an 80-character,
  all-caps, all-wide-letter name measures under half the available width, so
  no real 80-character event name reaches the truncation branch. Not a code
  gap: the guard is in place and fires correctly when a name (plus the
  ` · BRACKET DRAW` / ` · Powered by…` suffix) is actually wide enough to
  need it.
