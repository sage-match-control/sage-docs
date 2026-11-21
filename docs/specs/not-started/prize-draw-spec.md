# Spec — Prize draw (Lucky Dink)

> **Status: not started.** Nothing here is built. Written 2026-11-22 from the owner's request for a
> raffle and door-prize draw that SAGE runs on a big screen at its events, with the winners kept in
> the event's workbook so a draw can be looked up and checked afterwards. Revised the same day with
> the owner's answers: prize quantities, prizes reordered at any time, no seed box, **Draw again**
> instead of "absent", **Unlock** and **Edit list**, the Pickle Jar up to 300 names, and the display in
> its own window.
>
> **Dry-run on 2026-11-22**, in detached worktrees, of the code this spec gives in full:
>
> - **Site, at `937d92b`:** §4.3's `lib/v1/domain/prize-draw.js` and §4.4's test file. The 17 new tests
>   pass, and so does the site's whole unit suite (`npm test`: 694 tests, 684 pass, 0 fail, 10 skipped).
>   The engine's guards passed with the new module: it imports only `domain/`, uses no browser global,
>   and names no operator feature.
> - **API, at `60f43a8` (3.8.0):** all of §7's code, tests and edit scripts, applied as written to a clean
>   checkout in §7's order. `npm run verify` passed: 1997 tests, 0 fail, the coverage runs and every Apps
>   Script check included. The refreshed OpenAPI snapshot only added lines (451 added, 0 removed).
>
> The page itself (`tools/prize-draw.html`, §5) and the Mission Control and tools-roster edits (§6) were
> not dry-run. §5 specifies the page in full, and its reel and Pickle Jar code comes from working mockups
> the owner chose from.

A new tool, **Prize draw**, at `/tools/prize-draw`. On screen it is headed **S.A.G.E. Lucky Dink**.

- **What it does:** it draws raffle and door-prize winners in front of a crowd.
- **Names come from** a pasted list, the day's players, the people who have checked in, or plain ticket
  numbers.
- **Prizes** each have a name and a quantity. Any prize can be drawn all at once, and the order can change
  at any time.
- **Two ways to show a draw:** the **Reel** (the default) and the **Pickle Jar**. Either one plays on the
  operator's own screen or in a separate **display window** on the TV.
- **Every draw has a random draw code** (its seed), so anyone can re-run it later and get the same winners.

It is built in two phases, each shipped on its own:

| Phase | What | Repos |
| --- | --- | --- |
| **1** | The page and its display window, the draw's rules in the engine, a Mission Control tile and a tools-roster entry. The winners are kept in this browser and exported as text or an image | site, docs |
| **2** | Each winner is recorded as one row of a `DRAWS` tab in the event's facility workbook, through two new API routes. The page gains **Past draws**, which reopens, checks or continues a recorded draw | API, site, docs, workspace `CLAUDE.md` |

---

## 0. How to use this spec

You can carry this out with no other context. Every value, edit, test and check you need is here.
Read §0 to §4 before you change anything, and read §7 before Phase 2.

### 0.1 Where things are

The workspace is `D:\Personal\SAGE`. It holds several independent git repos, four of which change here:

| Repo | Path | What changes |
| --- | --- | --- |
| The site | `D:\Personal\SAGE\sage-match-control.github.io` | new: `lib/v1/domain/prize-draw.js`, `tools/prize-draw.html` and two test files under `_tests/unit/`; edited: `tools/control-center.html` (one tile) and `tools/index.html` (one roster entry) |
| The API | `D:\Personal\SAGE\sage-tools-api` | Phase 2 only: a new `src/draws/` feature, two methods on `SheetsClient`, two errors, the wiring, and their tests |
| The docs | `D:\Personal\SAGE\sage-docs` | new features and technical pages, index and nav entries, the API page, this spec's status, the changelog (§9) |
| The workspace | `D:\Personal\SAGE` | Phase 2 only: `CLAUDE.md` (§7.8) |

**The site**
- It is static GitHub Pages, with no build step and no framework.
- Event pages and Control Center are built on the **engine** in `lib/v1/`:
  - `domain/` holds pure rules;
  - `data/` holds fetches;
  - `views/` builds HTML;
  - `apps/` mounts pages.
- A page imports engine modules by root-absolute paths (`/lib/v1/...`).
- The dev-only tests live in `_tests/` (`cd _tests && npm install && npm test`).

**The API** (`sage-tools-api`)
- It is Node/Express on Cloud Run, ESM, with no build step.
- `npm test` runs its `node:test` suite.
- It deploys automatically on push to `main`.

**`event-data`**
- It holds `config/events.json` (the registry of events, days and facilities) and each day's published
  snapshot at `<event>/data/<day>.json`.
- Nothing in it changes here.

### 0.2 Rules

- **Find edits by their anchor text, not by line numbers.** Every edit to an existing file quotes text that
  occurs exactly **once** in that file. If an anchor isn't there, or appears more than once, **stop and
  report it**. Don't guess. §7.6's scripts check this themselves.
- **Use the code in this spec exactly as written.** §4 and §7 were dry-run (see the status block). Don't
  restyle, rename or "improve" it, and don't make design decisions. If the spec seems wrong, or two parts
  disagree, **stop and ask** the owner.
- **Keep each file's line endings.** The working copies on this machine have CRLF line endings, and git
  stores LF. A new file follows its neighbours, and `git diff` should show only your lines.
- **The engine rule** (`_templates/CLAUDE.md` §8):
  - `lib/v1/` changes only by adding.
  - You add one new module, `domain/prize-draw.js`.
  - Change no existing export, CSS class or theme property.
- **The API's rules** (in `sage-tools-api`'s README and the workspace `CLAUDE.md`):
  - Write each test, watch it fail, then make it pass.
  - Construct objects only in `src/app.mjs`.
  - New routes go under `/v3`.
  - Bump the version in `package.json`, and add a Changelog entry ending with a **Tests:** line.
  - Run `npm test` before every commit.
  - Never weaken or delete a test.
- **Use Git Bash** for the commands. The paths above are `/d/Personal/SAGE/...` there.
- **Commit, but never push or merge without the owner's go-ahead.**
  - The site and the API both deploy on push to `main`.
  - Merge nothing within 3 days of an event: check the dates in `event-data/config/events.json`.
  - End each commit message with the attribution line the session gives you.

### 0.3 Order of work

**Phase 1**

1. In the site repo, make a branch:

   ```bash
   git checkout main && git pull && git checkout -b prize-draw
   ```
2. Add §4.3's module and §4.4's test. Run `cd _tests && npm test`: everything passes, the 17 new tests
   included.
3. Build `tools/prize-draw.html` (§5).
4. Add §5.16's page test. Run `npm test` again: everything passes.
5. Make §6's two edits.
6. Serve the site and make §8.1's checks in a browser.
7. Commit as `Prize draw: Lucky Dink, a raffle and door-prize draw (Reel and Pickle Jar, display window); Mission Control tile`.
8. Report to the owner and wait for the go-ahead to merge and push.
9. Once it is live, make §9.1's docs changes in `sage-docs` on a branch, and commit them as
   `docs 2.78.0: the prize draw (Lucky Dink)` (or the next MINOR version, if that one is taken). Wait for
   the go-ahead to push.

**Phase 2.** Start only after the owner says Phase 1 is live and good.

1. In the API, make a branch:

   ```bash
   git checkout main && git pull && git checkout -b prize-draw-record
   ```
2. Apply §7.4–§7.7 in order, writing each test before its code. Run `npm test` and `npm run verify`.
3. Commit as `3.9.0: prize draws, GET …/prize-draws and PUT …/prize-draws/{drawId}/spins/{spin}`. Wait for
   the go-ahead, then push.
4. After the deploy, check that `GET /ping` answers `X-App-Version: 3.9.0` (§8.2).
5. In the site, on a new branch `prize-draw-record`, make §7.9's page changes, then §8.2's checks.
6. Commit as `Prize draw: record each winner to the event's workbook; Past draws`, and wait for the
   go-ahead.
7. Make the workspace `CLAUDE.md` edits (§7.8) and §9.2's docs changes (`docs 2.79.0`, or the next MINOR).
   Wait for the go-ahead to push each.

## 1. What it is

**Who and where**
- An operator runs it at the venue, usually on a laptop connected to a TV or projector.
- It opens from:
  - Mission Control's **Pages and tools** row, through a new **Open prize draw** tile that opens it on the
    selected event and day;
  - the tools roster at `/tools/`;
  - or straight at `/tools/prize-draw`.
- No public page links to it.

**A draw** is:
- one locked list of entries;
- one random draw code (the seed, made by the page; nobody types it);
- an optional list of prizes, each with a quantity;
- and a rule: **Each winner leaves the draw** (the default), or winners stay in.

Each press of **Spin** draws one prize unit, or a whole prize at once when that prize is set to **Draw all at
once**.

**Names on screen**
- The header reads **S.A.G.E. Lucky Dink**, and the tab title is *Prize draw · S.A.G.E.*
- The two styles are **Reel** and **Pickle Jar**.
- The "How it works" dialog is titled **No Rigging Zone**.
- A winner who isn't there is handled with **Draw again**.
- The words "absent" and "Big Dill" never appear.

**Fairness**
- Each spin's winners are fixed by the draw code, the locked list and the spin number (§4.1), **before**
  the animation starts. The animation only plays that result out.
- The prize a spin is for never enters that calculation, which is why prizes can be reordered freely.
- Anyone can check a draw: **Check this draw** in Phase 1, **Past draws** in Phase 2.

**What it never does**
- It never weights entries.
- It never redraws silently.
- It never lets a recorded winner change, except that a winner of the latest spin can be marked **drawn
  again**.

## 2. Decisions

| # | Decision | Source |
| --- | --- | --- |
| D1 | Both reveal styles, **Reel** the default, **Pickle Jar** the second | owner |
| D2 | Header **S.A.G.E. Lucky Dink**; tool name **Prize draw** in the roster, URL and tile; dialog **No Rigging Zone**; no "Big Dill" | owner |
| D3 | Winners are recorded in the event's facility workbook, in a `DRAWS` tab (Phase 2) | owner |
| D4 | A **Mission Control** tile opens it on the selected event and day | owner |
| D5 | Every prize has a quantity. By default each unit gets its own spin, labelled *Ball pack (1 of 3)*. A prize ticked **Draw all at once** draws its remaining units in one spin | owner |
| D6 | No seed box: the page always makes the draw code at random. It is shown small as **Draw code**, and kept in the exports and the workbook, so a draw stays checkable | owner |
| D7 | **Draw again** replaces "absent" everywhere: the button reads *Draw again*, the winners list *(drawn again)*, and the workbook's result `redrawn` | owner |
| D8 | **Unlock** reopens the list until the first spin. After that, **Edit list** closes the draw and starts a linked continuation: earlier winners stay out, the prizes not yet won carry over, and each part checks on its own | owner |
| D9 | The Pickle Jar holds one ball per entrant, up to **300**. The balls shrink as the list grows, and carry labels only up to 40. Above 300 the Jar is unavailable, with its reason | owner |
| D10 | **Open display** opens the reel or jar in a second window (or a tab, if popups are blocked) in the same browser; the controls stay in the first tab. No other device | owner |
| D11 | Prizes can be reordered at any time, before or during the draw. The next spin is for the first prize not yet won | owner |
| D12 | The draw rule: for spin n, the k smallest SHA-256(code\|n\|entry) win, smallest first. The same hash and code alphabet as the Bracket Generator's verifiable draw | spec |
| D13 | A staff tool: listed on the internal tools roster and in Mission Control, and on no public page. Anyone with the link can open it. **Load checked in** and recording (Phase 2) need an operator sign-in from an open Control Center tab in the same browser, exactly as the schedule board's staff view gets its sign-in | default; the owner may change it before the build |
| D14 | SAGE house colours always, not the event's | default |
| D15 | No sound | default |
| D16 | Ticket numbers are the place in the locked list (001, 002 …). A **Numbers** list (001 to N) is for physical stubs | spec |
| D17 | Phase 2 records to one facility workbook per draw: the day's only facility, or the one picked when a day has more than one | spec |

## 3. How a draw runs

1. **Set up.**
   - Optionally give the draw a title, such as *Door prizes*, and pick the event and day (Mission
     Control's tile already has).
   - Fill the **Names** box in one of four ways:
     - paste it;
     - **Load players**: everyone in the day's standings, or a team event's rosters;
     - **Load checked in** (needs a sign-in);
     - **Numbers**: 001 to N.
   - Optionally add prizes, each with a name, a quantity and **Draw all at once**, in the order they're
     drawn.
2. **Lock the list and start.** The page:
   - checks the list (§4.1);
   - makes the draw code and the draw id;
   - takes the list's fingerprint;
   - and locks the names and the rule.

   Until the first spin, **Unlock** reopens them.
3. **Spin** (the button, or Space, or Enter). The page:
   - works out the winners;
   - plays the animation to them, on its own stage or in the display window;
   - reveals the winners and the prize;
   - adds them to **Winners**, and in Phase 2 records them to the workbook.
4. **Draw again.** If a winner isn't there, **Draw again** beside their name, for the latest spin only,
   takes that person out of the draw. The same prize unit is drawn on the next spin.
5. **Reorder prizes** at any time, with ↑, ↓ and **Next** on each prize row. The next spin is for the top
   prize not yet won.
6. **Edit list** (after spins). It asks to confirm, then:
   - closes this draw;
   - opens the names for editing;
   - and, on **Lock**, starts a continuation. That has a new draw code, the list without earlier winners,
     and the prizes not yet won.
7. **The draw ends** when every prize has a winner (with prizes), or when nobody is left. Spin then reads
   *Draw complete*.
8. **Afterwards:**
   - **Copy results**, **Download .txt** and **Save image** (a winners board);
   - **Copy numbered list** (the locked list with ticket numbers);
   - **Check this draw**, which replays every part of the draw and ticks each winner.

## 4. The rules

### 4.1 Stated once

**Entries**
- One per line. Every run of white space becomes one space, and the ends are trimmed. Blank lines are
  dropped.
- A line that is the same person as an earlier one, by `attendanceKey` (accents, case and spacing ignored),
  is dropped and reported as merged.
- A locked list has 2 to 1000 entries, each at most 40 characters, with no `|`.

**The draw code (seed)**
- 8 random characters from `23456789ABCDEFGHJKMNPQRSTVWXYZ`, made by the page at Lock.
- There is no way to type one.
- The draw id is made the same way.

**The fingerprint**
- The SHA-256, in lowercase hex, of the entries joined by `\n` (no trailing line break).
- The page shows its first 8 characters.

**Prizes**
- Rows, each with a name (at most 60 characters), a quantity (1 to 50) and **Draw all at once**. At most 50
  rows and 200 units in all.
- Each unit is a **slot**, labelled *Name* (a quantity of 1) or *Name (k of n)*. A slot keeps its id
  (`<row id>:<k>`) however the rows are reordered.

**Spin n's winners**
- Each entrant still in the draw gets the key SHA-256(`code|n|entry`), in lowercase hex.
- A spin that draws k winners takes the k smallest keys, smallest first: place 1, 2 ….
- List order never matters, and nor does the prize.

**What a spin draws**
- The first slot not yet won, in the rows' current order.
- When that slot's row is **Draw all at once**, the spin draws every slot of that row not yet won.
- With no prizes, each spin draws one winner, and there is no end.

**Who is still in**
- A winner drawn again is always out.
- A winner who took a prize is out under **Each winner leaves the draw** (`remove-winners`), and stays in
  under `keep-winners`.
- A slot is won by a pick with result `won`. A pick with result `redrawn` leaves its slot to be drawn again.

**The check**
- It replays every recorded spin in order.
- A spin passes when its recorded winners are, place by place, the k smallest keys among who was still in,
  and the spin numbers run 1, 2, 3 ….

**A continued draw (Edit list)**
- It is a new draw, with its own code, id and fingerprint.
- Its list is the edited list less everyone the earlier parts made out (the same rule, applied across all
  parts).
- Its prizes are the earlier draw's slots not yet won, in their current order.
- It records the draw it continues (`continues`).
- Each part checks on its own.

### 4.2 Why this rule

- **Fair.** The key is a fixed hash, computed once per entrant before comparing, so the comparison is
  consistent. With distinct keys, every entrant is equally likely to rank first.
  - This is the decorate–sort argument of the
    [Bracket Generator's verifiable draw](../implemented/bracket-generator-verifiable-draw-spec.md) (§4.2).
  - Taking the k smallest is that same shuffle's first k.
- **Checkable.** It needs no state between spins beyond who is out, so anyone holding the list, the code
  and the rule can recompute any spin.
- **No hidden random source.** `Math.random()` is never used for a result. It only fills the reel's
  cosmetic rows and moves the jar's balls.
- **Prizes stay free.** The prize is never hashed, so reordering prizes, or drawing one all at once, can't
  favour anyone.
- **What it can't prevent:** an operator locking and unlocking until they like the code. Unlock is
  possible only before the first spin, and the "How it works" dialog says plainly that the code is made by
  the page.

### 4.3 `lib/v1/domain/prize-draw.js` (new)

Create this file exactly:

```js
// The prize draw's rules (sage-docs/docs/specs/.../prize-draw-spec.md §4): the entry list, the prizes, who wins
// each spin, and the check that replays a recorded draw. Pure: the SHA-256 function is passed in (the page passes
// one over crypto.subtle; the tests pass Node's), so nothing here touches the browser.
//
// The rule, stated once: for spin n, every entrant still in the draw gets the key
// SHA-256(seed + "|" + n + "|" + entry), as lowercase hex. A spin that draws k winners takes the k smallest keys,
// smallest first. The prize a spin is for never enters the hash, so prizes can be reordered at any time.
import { attendanceKey } from './attendance-marks.js';
import { BYE_RE } from './byes.js';
import { parseCSV } from './csv.js';
import { buildTeamRoster } from './teams.js';

/** The seed's characters: no I, L, O, U, 0 or 1, so a draw code reads aloud safely (the Bracket Generator's). */
export const SEED_ALPHABET = '23456789ABCDEFGHJKMNPQRSTVWXYZ';
export const SEED_LENGTH = 8;
/** The most entries a draw takes, and the longest entry: together they fit the workbook's one-cell list (§7). */
export const MAX_ENTRIES = 1000;
export const MAX_ENTRY_LENGTH = 40;
/** Prize rows: at most 50, each name at most 60 characters, each quantity 1 to 50, 200 prizes in all. */
export const MAX_PRIZE_ROWS = 50;
export const MAX_PRIZE_LENGTH = 60;
export const MAX_QUANTITY = 50;
export const MAX_PRIZES = 200;
/** The Pickle Jar holds one ball per entrant: at most this many, and labelled ones only up to JAR_LABEL_MAX. */
export const JAR_MAX_ENTRIES = 300;
export const JAR_LABEL_MAX = 40;

/** A seed from random bytes (the page passes crypto.getRandomValues(new Uint8Array(SEED_LENGTH))). Also a draw id. */
export function seedFromBytes(bytes){
  return [...bytes].map(b => SEED_ALPHABET[b % SEED_ALPHABET.length]).join('');
}

/** One entry, or a prize name, as the draw uses it: every run of white space one space, trimmed. */
export function cleanEntry(s){ return String(s ?? '').replace(/\s+/g, ' ').trim(); }

/**
 * Names in order. A name already in the list (the same person by attendanceKey: accents, case and spacing ignored)
 * is dropped and reported in `merged`, so a person is never in the draw twice.
 * @param {string[]} names
 * @returns {{ entries: string[], merged: string[] }}
 */
export function uniqueEntries(names){
  const seen = new Set(), entries = [], merged = [];
  for(const raw of names){
    const e = cleanEntry(raw);
    if(!e) continue;
    const k = attendanceKey(e);
    if(seen.has(k)){ merged.push(e); continue; }
    seen.add(k);
    entries.push(e);
  }
  return { entries, merged };
}

/** The Names box's text as entries: one per line, blank lines ignored. */
export function parseEntries(text){ return uniqueEntries(String(text ?? '').split(/\r?\n/)); }

/** Why this list can't be locked, as sentences, or [] when it can. */
export function entryProblems(entries){
  const out = [];
  if(entries.length < 2) out.push('A draw needs at least 2 entries.');
  if(entries.length > MAX_ENTRIES) out.push(`A draw takes at most ${MAX_ENTRIES} entries; this list has ${entries.length}.`);
  const long = entries.filter(e => e.length > MAX_ENTRY_LENGTH);
  if(long.length) out.push(`Shorten ${long.length === 1 ? 'this entry' : 'these entries'} to ${MAX_ENTRY_LENGTH} characters or fewer: ${long.join(', ')}.`);
  const bars = entries.filter(e => e.includes('|'));
  if(bars.length) out.push(`An entry can't contain "|": ${bars.join(', ')}.`);
  return out;
}

/** "001" to "<count>", for a raffle of numbered stubs. At least three digits. */
export function numberEntries(count){
  const n = Math.max(0, Math.floor(Number(count) || 0));
  const width = Math.max(3, String(n).length);
  return Array.from({ length: n }, (_, i) => String(i + 1).padStart(width, '0'));
}

/** An entry's ticket number: its place in the locked list, at least three digits ("007"). */
export function ticketOf(entries, entry){
  const i = entries.indexOf(entry);
  if(i < 0) return '';
  return String(i + 1).padStart(Math.max(3, String(entries.length).length), '0');
}

/** True when every entry is a ticket number already (a Numbers list), so nothing shows "#047" twice. */
export function isNumberList(entries){ return entries.length > 0 && entries.every(e => /^\d+$/.test(e)); }

/**
 * Every player on a day, from its published snapshot: a team event's rosters (rosterCsv), otherwise each pair's two
 * players from standingsCsv (player1, player2), BYE rows left out. Facility order, then sheet order; one entry per
 * person.
 * @param {{ facilities?: object[] }|null} snapshot
 * @param {'standard'|'dual-meet'|'team'|null} type
 * @returns {string[]}
 */
export function playersFromSnapshot(snapshot, type){
  if(type === 'team') return uniqueEntries(buildTeamRoster(snapshot).roster.map(r => r.player)).entries;
  const names = [];
  for(const f of (snapshot && snapshot.facilities) || []){
    const rows = parseCSV(f.standingsCsv || '');
    if(rows.length < 2) continue;
    const head = rows[0].map(h => h.trim());
    const iCode = head.indexOf('teamCode'), i1 = head.indexOf('player1'), i2 = head.indexOf('player2');
    if(i1 < 0) continue;
    for(const r of rows.slice(1)){
      const code = (r[iCode] || '').trim(), p1 = (r[i1] || '').trim(), p2 = i2 < 0 ? '' : (r[i2] || '').trim();
      if(BYE_RE.test(code) || BYE_RE.test(p1) || BYE_RE.test(p2)) continue;
      names.push(p1, p2);
    }
  }
  return uniqueEntries(names).entries;
}

/**
 * Everyone checked in, from each facility's people (GET .../people): present and not withdrawn, in roster order.
 * @param {object[][]} lists  one array of people per facility
 * @returns {string[]}
 */
export function checkedInFromPeople(lists){
  return uniqueEntries(lists.flat().filter(p => p && p.present === true && p.withdrawn !== true)
    .map(p => p.player || p.key)).entries;
}

/** The text the list's fingerprint is taken over: the entries in order, one per line, no line break at the end. */
export function listText(entries){ return entries.join('\n'); }

/**
 * Why these prize rows can't be used, as sentences, or [] when they can. A row is { id, name, quantity, together }.
 * @param {{ name: string, quantity: number }[]} rows
 */
export function prizeProblems(rows){
  const out = [];
  if(rows.length > MAX_PRIZE_ROWS) out.push(`List at most ${MAX_PRIZE_ROWS} prizes; this list has ${rows.length}.`);
  if(rows.some(r => !cleanEntry(r.name))) out.push('Give every prize a name, or remove its row.');
  const long = rows.filter(r => cleanEntry(r.name).length > MAX_PRIZE_LENGTH);
  if(long.length) out.push(`Shorten ${long.length === 1 ? 'this prize' : 'these prizes'} to ${MAX_PRIZE_LENGTH} characters or fewer: ${long.map(r => cleanEntry(r.name)).join(', ')}.`);
  if(rows.some(r => !Number.isInteger(r.quantity) || r.quantity < 1 || r.quantity > MAX_QUANTITY)) out.push(`Each quantity is a whole number from 1 to ${MAX_QUANTITY}.`);
  const total = rows.reduce((n, r) => n + (Number.isInteger(r.quantity) ? r.quantity : 0), 0);
  if(total > MAX_PRIZES) out.push(`At most ${MAX_PRIZES} prizes in all; these rows add up to ${total}.`);
  return out;
}

/**
 * The prizes to draw, one slot per unit, in row order: "Ball pack (1 of 3)", "Ball pack (2 of 3)" … and a plain name
 * for a quantity of 1. A slot's id is "<row id>:<unit>", so a slot keeps its identity when the rows are reordered.
 * @param {{ id: string, name: string, quantity: number, together: boolean }[]} rows
 * @returns {{ id: string, rowId: string, label: string, together: boolean }[]}
 */
export function prizeSlots(rows){
  return rows.flatMap(r => Array.from({ length: r.quantity }, (_, i) => ({
    id: `${r.id}:${i + 1}`,
    rowId: r.id,
    label: r.quantity > 1 ? `${cleanEntry(r.name)} (${i + 1} of ${r.quantity})` : cleanEntry(r.name),
    together: r.together === true,
  })));
}

/**
 * Who is still in the draw. `earlier` is every pick before the spin, as { winner, result: 'won' | 'redrawn' }. A
 * redrawn winner is always out; a winner who took a prize is out under 'remove-winners' and stays in under
 * 'keep-winners'. Also the list a continued draw starts from (§5.9).
 * @param {string[]} entries @param {{ winner: string, result: string }[]} earlier
 * @param {'remove-winners'|'keep-winners'} rule
 */
export function entrantsFor(entries, earlier, rule){
  const out = new Set(earlier.filter(s => s.result === 'redrawn' || rule === 'remove-winners').map(s => s.winner));
  return entries.filter(e => !out.has(e));
}

/**
 * What the next spin draws for, from the prize slots in their current order and every pick so far (a continued
 * draw's included). A slot is used once a pick with its slotId won it. The next slot is the first unused one; a
 * spin draws it alone, or, when its row is drawn all at once, every unused slot of that row.
 *   no prize list           -> { count: 1, slots: [] }   (spin as often as you like)
 *   every prize won         -> { count: 0, slots: [] }
 * @param {{ id: string, rowId: string, together: boolean }[]} slots
 * @param {{ slotId?: string, result: string }[]} picks
 */
export function nextSpin(slots, picks){
  if(!slots.length) return { count: 1, slots: [] };
  const used = new Set(picks.filter(p => p.result === 'won').map(p => p.slotId));
  const open = slots.filter(s => !used.has(s.id));
  if(!open.length) return { count: 0, slots: [] };
  const take = open[0].together ? open.filter(s => s.rowId === open[0].rowId) : [open[0]];
  return { count: take.length, slots: take };
}

/**
 * Spin n's winners: the `count` entrants with the smallest SHA-256(seed|n|entry), smallest first. Fewer when fewer
 * are left; [] when nobody is.
 * @param {{ seed: string, spin: number, entrants: string[], count: number, sha256Hex: (s: string) => Promise<string> }} a
 * @returns {Promise<{ winner: string, key: string }[]>}
 */
export async function drawWinners({ seed, spin, entrants, count, sha256Hex }){
  const keys = await Promise.all(entrants.map(e => sha256Hex(`${seed}|${spin}|${e}`)));
  return entrants.map((winner, i) => ({ winner, key: keys[i] }))
    .sort((a, b) => (a.key < b.key ? -1 : a.key > b.key ? 1 : 0))
    .slice(0, Math.max(0, count));
}

/** True when a prize list is set and every prize is won, or nobody is left to draw. */
export function drawFinished({ entries, slots, picks, allPicks = picks, rule }){
  if(nextSpin(slots, allPicks).count === 0) return true;
  return entrantsFor(entries, picks, rule).length === 0;
}

/**
 * Replays a recorded draw. Picks are grouped by spin; each spin's winners must be the k smallest keys among who was
 * still in, where k is how many winners it recorded, in place order (1, 2 …). Spins must run 1, 2, 3 ….
 * @param {{ entries: string[], seed: string, rule: string, picks: { spin: number, place: number, winner: string, result: string }[],
 *           sha256Hex: (s: string) => Promise<string> }} a
 * @returns {Promise<{ spin: number, place: number, expected: string|null, recorded: string, ok: boolean }[]>}
 */
export async function verifyDraw({ entries, seed, rule, picks, sha256Hex }){
  const spins = [...new Set(picks.map(p => p.spin))].sort((a, b) => a - b);
  const out = [];
  const earlier = [];
  for(let i = 0; i < spins.length; i++){
    const these = picks.filter(p => p.spin === spins[i]).sort((a, b) => a.place - b.place);
    const got = await drawWinners({ seed, spin: spins[i], entrants: entrantsFor(entries, earlier, rule), count: these.length, sha256Hex });
    these.forEach((p, j) => {
      const expected = got[j] ? got[j].winner : null;
      out.push({ spin: p.spin, place: p.place, expected, recorded: p.winner, ok: spins[i] === i + 1 && p.place === j + 1 && expected === p.winner });
    });
    earlier.push(...these);
  }
  return out;
}

/**
 * The draw as plain text, for Copy results and the text download (§5.13).
 * @param {{ title: string, eventLine: string, seed: string, fingerprint: string, rule: string, entries: string[],
 *           picks: { spin: number, place: number, prize: string, winner: string, result: string }[] }} d
 */
export function drawText(d){
  const numbers = isNumberList(d.entries);
  const who = w => numbers ? `#${w}` : w;
  const lines = ['S.A.G.E. LUCKY DINK'];
  lines.push([d.title, d.eventLine].filter(Boolean).join(' · ') || 'Prize draw');
  lines.push(`Draw code: ${d.seed}`);
  lines.push(`List: ${d.entries.length} entries · fingerprint ${d.fingerprint.slice(0, 8)}`);
  lines.push(d.rule === 'remove-winners' ? 'Each winner leaves the draw.' : 'Winners stay in the draw.');
  lines.push('');
  const picks = [...d.picks].sort((a, b) => a.spin - b.spin || a.place - b.place);
  for(const p of picks){
    const prize = p.prize ? `${p.prize}: ` : '';
    lines.push(`Spin ${p.spin} · ${prize}${who(p.winner)}${p.result === 'redrawn' ? ' (drawn again)' : ''}`);
  }
  if(!picks.length) lines.push('No spins yet.');
  lines.push('', 'Check it at sage-match-control.github.io/tools/prize-draw');
  return lines.join('\n');
}
```

### 4.4 `_tests/unit/domain/prize-draw.test.mjs` (new)

Create this file exactly.
- The pinned winners are the rule's own output, so a change to one is a change to the rule, never a test
  to update.
- The test seed `HXTE7Q4M` uses only the code alphabet, as every real draw code does.

```js
// The prize draw's rules (sage-docs/docs/specs/.../prize-draw-spec.md §4): entries, prizes, the winners of each
// spin, and the replay check. The pinned winners below are the rule's own output: if one changes, every recorded
// draw stops checking out, so a change here is a change to the rule, never a test to update.
import test from 'node:test';
import assert from 'node:assert/strict';
import { createHash } from 'node:crypto';
import {
  SEED_ALPHABET, MAX_ENTRIES, MAX_ENTRY_LENGTH, MAX_PRIZES, seedFromBytes, cleanEntry, uniqueEntries, parseEntries,
  entryProblems, numberEntries, ticketOf, isNumberList, playersFromSnapshot, checkedInFromPeople, listText,
  prizeProblems, prizeSlots, entrantsFor, nextSpin, drawWinners, drawFinished, verifyDraw, drawText,
} from '../../../lib/v1/domain/prize-draw.js';

const sha256Hex = async s => createHash('sha256').update(s, 'utf8').digest('hex');
const NAMES = ['Ana Reyes', 'Ben Cruz', 'Carla Santos', 'Dante Lim', 'Ella Tan', 'Francis Go'];
const row = (id, name, quantity = 1, together = false) => ({ id, name, quantity, together });

test('a seed uses only the read-aloud alphabet', () => {
  assert.equal(seedFromBytes([0, 1, 29, 30, 255]), '23Z2H');
  assert.ok(!/[ILOU01]/.test(SEED_ALPHABET));
});

test('entries: spaces collapsed, blank lines dropped, the same person once (accents, case and spacing ignored)', () => {
  assert.equal(cleanEntry('  Ana   Reyes \t'), 'Ana Reyes');
  assert.deepEqual(parseEntries('Ana Reyes\r\n\n  ana  reyes \nJosé Cruz\nJose cruz\nBen'), {
    entries: ['Ana Reyes', 'José Cruz', 'Ben'], merged: ['ana reyes', 'Jose cruz'],
  });
  assert.deepEqual(uniqueEntries(['', ' ', 'X']), { entries: ['X'], merged: [] });
});

test('a list that cannot be locked says why', () => {
  assert.deepEqual(entryProblems(['Ana']), ['A draw needs at least 2 entries.']);
  assert.deepEqual(entryProblems(Array.from({ length: MAX_ENTRIES + 1 }, (_, i) => `P${i}`)),
    [`A draw takes at most ${MAX_ENTRIES} entries; this list has ${MAX_ENTRIES + 1}.`]);
  const long = 'x'.repeat(MAX_ENTRY_LENGTH + 1);
  assert.deepEqual(entryProblems(['Ana', long]), [`Shorten this entry to 40 characters or fewer: ${long}.`]);
  assert.deepEqual(entryProblems(['Ana', 'B|C']), ['An entry can\'t contain "|": B|C.']);
  assert.deepEqual(entryProblems(NAMES), []);
});

test('numbers and tickets: at least three digits, wider when the list is longer', () => {
  assert.deepEqual(numberEntries(3), ['001', '002', '003']);
  assert.equal(numberEntries(1000)[999], '1000');
  assert.deepEqual(numberEntries('x'), []);
  assert.equal(ticketOf(NAMES, 'Carla Santos'), '003');
  assert.equal(ticketOf(NAMES, 'Nobody'), '');
  assert.equal(ticketOf(numberEntries(1200), '0047'), '0047');
  assert.equal(isNumberList(numberEntries(5)), true);
  assert.equal(isNumberList(NAMES), false);
  assert.equal(isNumberList([]), false);
});

test('players on a day: standings pairs without BYEs; a team event\'s rosters', () => {
  const standard = { facilities: [
    { name: 'A', standingsCsv: 'teamCode,player1,player2,wins\r\nMD_1,Ana Reyes ,Ben Cruz ,0\r\nMD_2,bye,,0\r\nBYE,X,Y,0\r\nWD_1,Carla Santos,ana reyes,0\r\n' },
    { name: 'B', standingsCsv: '' },
    { name: 'C', standingsCsv: 'teamCode,player1,player2\nXD_1,Dante Lim,Ella Tan\n' },
  ] };
  assert.deepEqual(playersFromSnapshot(standard, 'standard'), ['Ana Reyes', 'Ben Cruz', 'Carla Santos', 'Dante Lim', 'Ella Tan']);
  const team = { facilities: [{ name: 'A', standingsCsv: 'teamCode,teamName\nT1,Aces\n',
    rosterCsv: 'teamCode,player,level,gender\nT1,Ana Reyes,3.5,F\nT2,Ben Cruz,3.0,M\nT2,,3.0,M\n' }] };
  assert.deepEqual(playersFromSnapshot(team, 'team'), ['Ana Reyes', 'Ben Cruz']);
  assert.deepEqual(playersFromSnapshot(null, 'standard'), []);
});

test('checked in: present and not withdrawn, once each, in roster order', () => {
  const lists = [
    [{ key: 'ana reyes', player: 'Ana Reyes', present: true, withdrawn: false },
     { key: 'ben cruz', player: 'Ben Cruz', present: false, withdrawn: false },
     { key: 'carla santos', player: 'Carla Santos', present: true, withdrawn: true }],
    [{ key: 'dante lim', player: '', present: true, withdrawn: false },
     { key: 'ana reyes', player: 'Ana Reyes', present: true, withdrawn: false }],
  ];
  assert.deepEqual(checkedInFromPeople(lists), ['Ana Reyes', 'dante lim']);
});

test('the fingerprint text is the entries one per line, no trailing line break', async () => {
  assert.equal(listText(['A', 'B']), 'A\nB');
  assert.equal(await sha256Hex(listText(NAMES)), '3de62d1c8a16989807f68efa2a5e74faa684df68d92339114f81480aa6867c74');
});

test('prize rows: named, 1 to 50 each, 200 in all', () => {
  assert.deepEqual(prizeProblems([row('a', 'Paddle', 1), row('b', 'Ball pack', 3)]), []);
  assert.deepEqual(prizeProblems([row('a', ' ', 1)]), ['Give every prize a name, or remove its row.']);
  assert.deepEqual(prizeProblems([row('a', 'x'.repeat(61), 1)]), [`Shorten this prize to 60 characters or fewer: ${'x'.repeat(61)}.`]);
  assert.deepEqual(prizeProblems([row('a', 'Cap', 0)]), ['Each quantity is a whole number from 1 to 50.']);
  assert.deepEqual(prizeProblems([row('a', 'Cap', 2.5)]), ['Each quantity is a whole number from 1 to 50.']);
  const many = Array.from({ length: 5 }, (_, i) => row(`r${i}`, `P${i}`, 50));
  assert.deepEqual(prizeProblems(many), [`At most ${MAX_PRIZES} prizes in all; these rows add up to 250.`]);
  assert.deepEqual(prizeProblems(Array.from({ length: 51 }, (_, i) => row(`r${i}`, `P${i}`))), ['List at most 50 prizes; this list has 51.']);
});

test('prize slots: one per unit, numbered within a quantity, keeping their row id', () => {
  assert.deepEqual(prizeSlots([row('a', ' Paddle '), row('b', 'Ball pack', 3, true)]), [
    { id: 'a:1', rowId: 'a', label: 'Paddle', together: false },
    { id: 'b:1', rowId: 'b', label: 'Ball pack (1 of 3)', together: true },
    { id: 'b:2', rowId: 'b', label: 'Ball pack (2 of 3)', together: true },
    { id: 'b:3', rowId: 'b', label: 'Ball pack (3 of 3)', together: true },
  ]);
});

test('the next spin: one slot at a time, a whole row drawn all at once, a reorder obeyed, a redrawn prize again', () => {
  const rows = [row('a', 'Paddle'), row('b', 'Ball pack', 3, true), row('c', 'Cap', 2)];
  const slots = prizeSlots(rows);
  assert.deepEqual(nextSpin([], []), { count: 1, slots: [] });
  assert.deepEqual(nextSpin(slots, []).slots.map(s => s.id), ['a:1']);
  const afterPaddle = [{ slotId: 'a:1', result: 'won' }];
  assert.deepEqual(nextSpin(slots, afterPaddle).slots.map(s => s.id), ['b:1', 'b:2', 'b:3']);
  // Caps first: the rows reordered, so the slots are made again in the new order.
  const reordered = prizeSlots([rows[2], rows[0], rows[1]]);
  assert.deepEqual(nextSpin(reordered, afterPaddle).slots.map(s => s.id), ['c:1']);
  // One of the three ball-pack winners drawn again: only that one unit is drawn next.
  const packs = [...afterPaddle, { slotId: 'b:1', result: 'won' }, { slotId: 'b:2', result: 'redrawn' }, { slotId: 'b:3', result: 'won' }];
  assert.deepEqual(nextSpin(slots, packs).slots.map(s => s.id), ['b:2']);
  const all = ['a:1', 'b:1', 'b:2', 'b:3', 'c:1', 'c:2'].map(slotId => ({ slotId, result: 'won' }));
  assert.deepEqual(nextSpin(slots, all), { count: 0, slots: [] });
});

test('who is still in: drawn-again winners always out; prize winners out only under remove-winners', () => {
  const earlier = [{ winner: 'Ana Reyes', result: 'won' }, { winner: 'Ben Cruz', result: 'redrawn' }];
  assert.deepEqual(entrantsFor(NAMES, earlier, 'remove-winners'), ['Carla Santos', 'Dante Lim', 'Ella Tan', 'Francis Go']);
  assert.deepEqual(entrantsFor(NAMES, earlier, 'keep-winners'), ['Ana Reyes', 'Carla Santos', 'Dante Lim', 'Ella Tan', 'Francis Go']);
});

test('a spin\'s winners are the smallest SHA-256(seed|spin|entry), smallest first, whatever the list order', async () => {
  const [w] = await drawWinners({ seed: 'HXTE7Q4M', spin: 1, entrants: NAMES, count: 1, sha256Hex });
  assert.deepEqual(w, { winner: 'Ben Cruz', key: await sha256Hex('HXTE7Q4M|1|Ben Cruz') });
  const three = await drawWinners({ seed: 'HXTE7Q4M', spin: 1, entrants: NAMES, count: 3, sha256Hex });
  assert.deepEqual(three.map(x => x.winner), PINNED_THREE);
  const reversed = await drawWinners({ seed: 'HXTE7Q4M', spin: 1, entrants: [...NAMES].reverse(), count: 3, sha256Hex });
  assert.deepEqual(reversed.map(x => x.winner), PINNED_THREE);
  for(const e of NAMES.filter(n => !PINNED_THREE.includes(n))) assert.ok(await sha256Hex(`HXTE7Q4M|1|${e}`) > three[2].key);
  assert.equal((await drawWinners({ seed: 'HXTE7Q4M', spin: 1, entrants: ['A', 'B'], count: 5, sha256Hex })).length, 2);
  assert.deepEqual(await drawWinners({ seed: 'X', spin: 1, entrants: [], count: 1, sha256Hex }), []);
});

test('a whole draw, pinned: HXTE7Q4M over six names, remove-winners, spin 2 drawn again', async () => {
  const picks = [];
  for(let n = 1; n <= 4; n++){
    const [{ winner }] = await drawWinners({ seed: 'HXTE7Q4M', spin: n, entrants: entrantsFor(NAMES, picks, 'remove-winners'), count: 1, sha256Hex });
    picks.push({ spin: n, place: 1, winner, result: n === 2 ? 'redrawn' : 'won' });
  }
  assert.deepEqual(picks.map(s => s.winner), PINNED_DRAW);
});

test('a draw is finished when every prize is won, or nobody is left', () => {
  const slots = prizeSlots([row('p', 'Paddle')]);
  const won = [{ slotId: 'p:1', winner: 'A', result: 'won' }];
  assert.equal(drawFinished({ entries: ['A', 'B'], slots, picks: won, rule: 'keep-winners' }), true);
  assert.equal(drawFinished({ entries: ['A', 'B'], slots: [], picks: won, rule: 'remove-winners' }), false);
  assert.equal(drawFinished({ entries: ['A', 'B'], slots: [], picks: [...won, { winner: 'B', result: 'redrawn' }], rule: 'remove-winners' }), true);
  // A continued draw: its own picks decide who is left, every pick so far decides which prizes are won.
  assert.equal(drawFinished({ entries: ['C', 'D'], slots, picks: [], allPicks: won, rule: 'remove-winners' }), true);
});

test('the replay check: a true record checks out; a changed winner, seed, place or spin number does not', async () => {
  const picks = PINNED_DRAW.map((winner, i) => ({ spin: i + 1, place: 1, winner, result: i === 1 ? 'redrawn' : 'won' }));
  const ok = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4M', rule: 'remove-winners', picks, sha256Hex });
  assert.deepEqual(ok.map(r => r.ok), [true, true, true, true]);
  const swapped = picks.map((p, i) => i === 2 ? { ...p, winner: 'Ben Cruz' } : p);
  const bad = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4M', rule: 'remove-winners', picks: swapped, sha256Hex });
  assert.deepEqual(bad.slice(0, 3).map(r => r.ok), [true, true, false]);
  assert.equal(bad[2].expected, PINNED_DRAW[2]);
  const otherSeed = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4N', rule: 'remove-winners', picks, sha256Hex });
  assert.ok(otherSeed.some(r => !r.ok));
  const gap = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4M', rule: 'remove-winners', picks: [{ ...picks[0], spin: 2 }], sha256Hex });
  assert.equal(gap[0].ok, false);
});

test('the replay check on a spin that drew three at once', async () => {
  const picks = PINNED_THREE.map((winner, i) => ({ spin: 1, place: i + 1, winner, result: 'won' }));
  const ok = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4M', rule: 'remove-winners', picks, sha256Hex });
  assert.deepEqual(ok.map(r => [r.place, r.ok]), [[1, true], [2, true], [3, true]]);
  const outOfOrder = [picks[1], picks[0], picks[2]].map((p, i) => ({ ...p, place: i + 1 }));
  const bad = await verifyDraw({ entries: NAMES, seed: 'HXTE7Q4M', rule: 'remove-winners', picks: outOfOrder, sha256Hex });
  assert.deepEqual(bad.map(r => r.ok), [false, false, true]);
});

test('the plain-text record', () => {
  const text = drawText({ title: 'Door prizes', eventLine: 'UPAP 3rd Annual · Nov 22', seed: 'HXTE7Q4M',
    fingerprint: 'a41f09c2ffff', rule: 'remove-winners', entries: NAMES,
    picks: [{ spin: 2, place: 1, prize: 'Paddle', winner: 'Ben Cruz', result: 'won' },
            { spin: 1, place: 1, prize: 'Paddle', winner: 'Ana Reyes', result: 'redrawn' }] });
  assert.equal(text, [
    'S.A.G.E. LUCKY DINK', 'Door prizes · UPAP 3rd Annual · Nov 22', 'Draw code: HXTE7Q4M',
    'List: 6 entries · fingerprint a41f09c2', 'Each winner leaves the draw.', '',
    'Spin 1 · Paddle: Ana Reyes (drawn again)', 'Spin 2 · Paddle: Ben Cruz', '',
    'Check it at sage-match-control.github.io/tools/prize-draw',
  ].join('\n'));
  const numbers = drawText({ title: '', eventLine: '', seed: 'AB', fingerprint: '0123456789', rule: 'keep-winners',
    entries: numberEntries(3), picks: [{ spin: 1, place: 1, prize: '', winner: '002', result: 'won' }] });
  assert.equal(numbers.split('\n').slice(0, 7).join('\n'), [
    'S.A.G.E. LUCKY DINK', 'Prize draw', 'Draw code: AB', 'List: 3 entries · fingerprint 01234567',
    'Winners stay in the draw.', '', 'Spin 1 · #002'].join('\n'));
  const empty = drawText({ title: 'T', eventLine: '', seed: 'S', fingerprint: 'f', rule: 'keep-winners', entries: ['A', 'B'], picks: [] });
  assert.ok(empty.includes('\nNo spins yet.\n'));
});

const PINNED_DRAW = ['Ben Cruz', 'Ella Tan', 'Francis Go', 'Carla Santos'];
const PINNED_THREE = ['Ben Cruz', 'Carla Santos', 'Francis Go'];
```

Run it alone from `_tests/` with `node --test unit/domain/prize-draw.test.mjs`: 17 tests pass. Then run
`npm test`, which runs the engine's guards (`unit/guards.test.mjs`) over the new module too.

## 5. The page: `tools/prize-draw.html` (new)

### 5.1 File and head

One self-contained file: an inline `<style>`, **one** `<script type="module">`, and no other script. The same
file is both the controls and the display window, which it becomes with `?display=1` (§5.15). Copy these
head lines exactly:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="robots" content="noindex">
<title>Prize draw · S.A.G.E.</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<link rel="apple-touch-icon" sizes="180x180" href="/assets/favicons/apple-touch-icon.png">
<link rel="icon" type="image/png" sizes="32x32" href="/assets/favicons/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/assets/favicons/favicon-16x16.png">
<link rel="manifest" href="/assets/favicons/site.webmanifest">
```

### 5.2 Theme

House palette only, light only, as on the other tools:

```css
:root{
  --navy:#14263C; --navy-deep:#0B1826; --green:#7CB92C; --green-dark:#5C8F1F;
  --paper:#F6F7F2; --paper-dim:#ECEEE6; --line:#D9DED2; --ink:#14263C; --ink-soft:#5B6B74;
  --white:#FFFFFF; --radius:14px;
  --ball-a:#9ED34A; --ball-b:#C9E265; --ball-line:#5C8F1F;
  --dim:rgba(11,24,38,.6); --band:rgba(124,185,44,.12);
}
```

- **Green is a fill colour, not a text colour on white** (about 2.3:1). Use it on navy, or as a background
  under navy text.
- **Small text on paper** is `--ink` or `--ink-soft`.
- **Fonts:**
  - `body` is Inter.
  - Headings, the reel's rows and the winners' names are Barlow Condensed.
  - The header wordmark is Barlow Condensed 700 with `letter-spacing:1px`.
- **`body`** has `background:var(--paper)` and `overflow-x:hidden`.

### 5.3 Layout (the controls)

```
┌ header.bar (navy) ──────────────────────────────────────────────────────────────┐
│ S.A.G.E. LUCKY DINK      #eventLine      [How it works] [Open display] [Big screen] │
└─────────────────────────────────────────────────────────────────────────────────┘
┌ section.stage ─────────────────────────────┐ ┌ aside.setup ───────────────────┐
│ [Reel] [Pickle Jar]                         │ │ 1 Draw        (card)            │
│ #reel  or  #jar  (or #displayStatus)        │ │ 2 Names       (card)            │
│ #reveal                                     │ │ 3 Prizes      (card)            │
│ [Spin]                                      │ │ 4 Rule        (card)            │
│ #seedLine                                   │ │ [Lock] [Unlock] [Edit list] [New draw] │
└─────────────────────────────────────────────┘ │ 5 Winners     (card)            │
                                                └─────────────────────────────────┘
```

- **From 900px:** a grid, `grid-template-columns:minmax(0,1.4fr) minmax(0,1fr)`, with a 20px gap, and
  `max-width:1280px` centred.
- **Below 900px:** one column, the stage first. The side gutter is 16px, and nothing scrolls sideways at
  375px.
- **Cards:** white, `1px solid var(--line)`, `border-radius:var(--radius)`, 16px padding. Each card's title
  is Barlow Condensed 600 at 18px.
- **The header** (`header.bar`): navy, 12px 16px padding.
  - The wordmark: `S.A.G.E. <span class="dink">LUCKY DINK</span>`, with `.dink` in green.
  - `#eventLine`: paper colour, 13px, e.g. *UPAP 3rd Annual Tournament · Nov 22*.
  - The three buttons are outlined in paper colour.
  - On a phone, the event line wraps under the wordmark.

### 5.4 Element ids

Use exactly these ids. §5.16's test and §7.9 rely on them.

| Id | What |
| --- | --- |
| `eventLine`, `howBtn`, `displayBtn`, `bigBtn` | header |
| `styleReel`, `styleJar` | the style switch: two buttons in a `role="radiogroup"`, the chosen one `aria-checked="true"` |
| `reel`, `reelStrip` | the reel window and the strip inside it |
| `jar` | the Pickle Jar's `<canvas>` |
| `displayStatus` | shown in place of the stage while the display window is open (§5.15) |
| `reveal` | the reveal card |
| `spinBtn` | Spin |
| `seedLine` | the line under Spin |
| `titleInput`, `eventSelect`, `daySelect`, `facilitySelect` | card 1 |
| `srcPaste`, `srcPlayers`, `srcCheckedIn`, `srcNumbers` | card 2's four source buttons |
| `loadPlayersBtn`, `loadCheckedInBtn`, `numberCount`, `fillNumbersBtn` | card 2's source controls |
| `namesInput`, `namesCount`, `namesProblems` | card 2's list, its count line and its problems |
| `prizeRows`, `addPrizeBtn`, `prizesCount`, `prizeProblems` | card 3 |
| `removeWinners`, `showTickets` | card 4 |
| `lockBtn`, `unlockBtn`, `editListBtn`, `newDrawBtn` | under card 4 |
| `winnersList`, `copyBtn`, `txtBtn`, `pngBtn`, `listBtn`, `checkBtn`, `checkResult` | card 5 |
| `howDialog` | the No Rigging Zone `<dialog>` |
| `toast` | one status line for short messages (`role="status"`, `aria-live="polite"`) |

### 5.5 The script's imports and state

```js
import { CLOUD_RUN_BASE_URL } from '/lib/v1/platform.js';
import { loadRegistry, eventConfigFrom } from '/lib/v1/data/registry.js';
import { fetchDaySnapshotFromPages } from '/lib/v1/data/snapshot.js';
import { fetchJson } from '/lib/v1/data/api.js';
import { createOperatorSession } from '/lib/v1/data/operator-session.js';
import {
  SEED_LENGTH, JAR_MAX_ENTRIES, JAR_LABEL_MAX, MAX_QUANTITY, seedFromBytes, cleanEntry, parseEntries, entryProblems,
  numberEntries, ticketOf, isNumberList, playersFromSnapshot, checkedInFromPeople, listText, prizeProblems, prizeSlots,
  entrantsFor, nextSpin, drawWinners, drawFinished, verifyDraw, drawText,
} from '/lib/v1/domain/prize-draw.js';

const STORE_KEY = 'sage.prizeDraw';
const DISPLAY_CHANNEL = 'sage.prize-draw-display';
const sha256Hex = async s => [...new Uint8Array(await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s)))]
  .map(b => b.toString(16).padStart(2, '0')).join('');
const randomCode = () => seedFromBytes(crypto.getRandomValues(new Uint8Array(SEED_LENGTH)));
```

**Don't declare anything with the same name as a `prize-draw.js` export.** §5.16's test checks this.

The whole state is one object, saved to `localStorage[STORE_KEY]` as JSON after every change, inside
`try/catch` (storage may be unavailable):

```js
// { v: 2, title, event, day, facility, names, rule, showTickets, style,
//   prizeRows: [{ id, name, quantity, together }],          // id: 'p' + a counter, never reused
//   locked: false,
//   // when locked:
//   drawId, entries, seed, fingerprint, lockedAt (ISO),
//   picks: [{ spin, place, slotId, prize, ticket, winner, result: 'won'|'redrawn', drawnAt (ISO) }],
//   // earlier parts of the same draw, after Edit list (oldest first), for the winners list, exports and Check:
//   parts: [{ drawId, entries, seed, fingerprint, lockedAt, picks }] }
```

- **On load,** restore a saved state with `v === 2`. When it is locked, show the toast *Restored the draw
  from this browser.*
- **Anything else starts empty:** a failed read, or any other `v`.
- **Display only:** `showTickets`, `style` and the order of `prizeRows` may change at any time, locked or
  not.
- **The slots** are always `prizeSlots(state.prizeRows)`, in the rows' current order.
- **`allPicks`** is `[...state.parts.flatMap(p => p.picks), ...state.picks]`.

### 5.6 Card 1, Draw

- **Title** `#titleInput`: placeholder *Door prizes*, `maxlength="80"`.
- **Event** `#eventSelect`
  - The first option is *No event* (value `''`).
  - Then come the events in `events.json` whose `archived` is not `true`, newest first by their first
    day's `date`, each labelled with its `title`.
  - The registry comes from `await loadRegistry()`.
- **Day** `#daySelect`
  - It lists `eventConfigFrom(registry, key).days`, labelled `<label> · <date as "Nov 22">`.
  - It is hidden while no event is picked.
- **Facility** `#facilitySelect`: shown only when the day has more than one facility, the first one picked.
  Phase 2 records to it.
- **URL parameters:**
  - `?event=<key>&day=<dayKey>` preselects both, when they are in the registry; an unknown key is ignored.
  - When the operator changes either, the page writes them back with `history.replaceState`.
- **`#eventLine`** shows `<event title> · <day label>`, or nothing for *No event*.
- **If the registry can't load,** the event menu holds only *No event* and the toast says *Couldn't load
  the events. You can still paste names.*

### 5.7 Card 2, Names

There are four source buttons above `#namesInput`, a `<textarea>` of 10 rows, one name per line, with the
placeholder *One name per line*. A source button only chooses which helper shows, and every helper writes
into `#namesInput`, which the operator can still edit before locking.

- **Paste** (the default) shows nothing extra.
- **Players on a day** shows `#loadPlayersBtn` (**Load players**).
  - It needs an event and a day. Without them it is disabled, with the hint *Pick an event and day first.*
  - It calls `fetchDaySnapshotFromPages(event, day)`, then `playersFromSnapshot(snapshot, eventConfig.type)`,
    and fills the box, one per line.
  - An error with `status === 404` shows the toast *This day has no published data yet.* Any other error
    shows *Couldn't load the players. Try again.*
- **Checked in** shows `#loadCheckedInBtn` (**Load checked in**).
  - It needs an event, a day and an operator sign-in.
  - **The sign-in:**
    - On load, the page runs `session = createOperatorSession({ onSession, onSignedOut, answer: false })`,
      then `session.ask()`.
    - `onSession(s)` keeps `s` in memory only, never stored; `onSignedOut` clears it.
    - A sign-in counts while `s.expiresAt > Date.now()`.
  - Without a sign-in the button is disabled, with the hint *Open Control Center in this browser and sign
    in.* It enables by itself when a sign-in arrives.
  - **Loading** reads each facility of the day:

    ```js
    fetchJson(`${CLOUD_RUN_BASE_URL}/v3/days/${encodeURIComponent(day)}/facilities/${encodeURIComponent(f)}/people`, { token })
    ```

    Then it calls `checkedInFromPeople(lists)` and fills the box.
  - **Toasts:**
    - nobody checked in: *Nobody is checked in yet.*
    - a `401`: *Your sign-in has expired. Sign in again in Control Center.*
    - anything else: *Couldn't load who is checked in. Try again.*
- **Numbers** shows `#numberCount` (a number input, 2 to 1000, default 100) and `#fillNumbersBtn` (**Fill 1
  to N**), which fills the box with `numberEntries(n).join('\n')`.
- **`#namesCount`** updates on every input, from `parseEntries(namesInput.value)`.
  - It reads *120 names*.
  - When names were merged, it adds *· 2 merged (the same person twice): Ana Reyes, Ben Cruz*, listing at
    most 5 names, then *and 3 more*.
- **`#namesProblems`** lists `entryProblems(entries)`, only after **Lock** is pressed once.

### 5.8 Card 3, Prizes

The prize list is an editor, `#prizeRows`: one row per prize, and **Add prize** (`#addPrizeBtn`) below it.
A new row has an empty name, a quantity of 1, and *Draw all at once* unticked.

Each row is a `<div class="prize-row" data-id="p3">` holding, left to right:

| Control | Class | What |
| --- | --- | --- |
| text input | `.prize-name` | the name, `maxlength="60"`, placeholder *Paddle (Joola)* |
| number input | `.prize-qty` | the quantity, `min="1" max="50" step="1"`, 5em wide |
| checkbox and label | `.prize-together` | *Draw all at once*, shown only when the quantity is more than 1 |
| ↑ button | `.prize-up` | `aria-label="Move up"` |
| ↓ button | `.prize-down` | `aria-label="Move down"` |
| **Next** button | `.prize-next` | moves the row to the top of the rows not yet won |
| × button | `.prize-remove` | `aria-label="Remove prize"` |
| status | `.prize-left` | *3 left* / *1 of 3 left* / *Won* |

**Before Lock** everything is editable.

**Once locked:**
- **Can change:**
  - **↑, ↓ and Next** stay enabled, and act only among rows with a slot not yet won.
  - **Draw all at once** stays enabled on rows with two or more slots left.
- **Can't change:**
  - The name, the quantity and ×. They are disabled on rows with any slot won, so the labels already given
    to winners stay true.
  - On untouched rows the name and quantity stay editable, and **Add prize** stays enabled.
- **Won rows** (every slot won) sink to the bottom, greyed, and read *Won*.

**Re-rendering.** After every change to `prizeRows`, re-render the rows. `#prizesCount` then reads *8
prizes: next up Paddle (Joola)*, or *No prizes: spin as many times as you like*.

**Validation.** `prizeProblems(state.prizeRows)` lists the problems in `#prizeProblems`, and blocks Lock and
Spin while there are any.

**Phone layout.** On a phone each row wraps: the name on its own line, the rest under it.

### 5.9 Card 4, Lock, Unlock, Edit list, New draw

**Card 4** has two controls:
- `#removeWinners`: a checkbox, ticked by default, *Each winner leaves the draw*.
- `#showTickets`: a checkbox, *Show ticket numbers*, which is display only.
  - When it is on, the reel's rows and the reveal show `#047` before the name.
  - For a Numbers list (`isNumberList`) it is ticked, disabled and labelled *Ticket numbers (a Numbers
    list)*, and rows show the number alone.

**`#lockBtn`** (**Lock the list and start**, primary green):

```js
async function lockDraw(){
  const { entries: typed } = parseEntries(namesInput.value);
  const earlier = state.parts.flatMap(p => p.picks);
  const entries = earlier.length ? entrantsFor(typed, earlier, state.rule) : typed;   // a continuation drops earlier winners
  const problems = [...entryProblems(entries), ...prizeProblems(state.prizeRows)];
  if(problems.length){ showProblems(problems); return; }
  Object.assign(state, {
    locked: true, drawId: randomCode(), entries, seed: randomCode(), fingerprint: await sha256Hex(listText(entries)),
    rule: removeWinners.checked ? 'remove-winners' : 'keep-winners', lockedAt: new Date().toISOString(), picks: [],
  });
  save(); render();
}
```

**The secure-context guard.** If `crypto.subtle` is missing (a `file://` page), `#lockBtn` and `#spinBtn` are
disabled, with *The draw needs the page served over HTTPS.* The page never falls back to `Math.random()`.

**Once locked:**
- The names, the event, day and facility, and the rule are disabled.
- `#lockBtn` is hidden.
- `#seedLine` reads *Draw code HXTE7Q4M · 120 entries · list 3de62d1c · spin 3*, where the spin number is
  the next one. On a continuation it adds *· part 2*.

**`#unlockBtn`** (**Unlock**, outlined) shows only while locked with no picks yet. It sets
`locked: false`, clears `drawId`, `seed`, `fingerprint` and `lockedAt`, and re-renders. The list is
editable again, and the next Lock makes a new code.

**`#editListBtn`** (**Edit list**, outlined) shows while locked with at least one pick and no spin running.

```js
if(confirm('Edit the list? This part of the draw closes as it is. Earlier winners stay out, and the prizes not yet won carry over.')){
  state.parts.push({ drawId: state.drawId, entries: state.entries, seed: state.seed, fingerprint: state.fingerprint, lockedAt: state.lockedAt, picks: state.picks });
  Object.assign(state, { locked: false, picks: [], drawId: '', seed: '', fingerprint: '', lockedAt: '' });
  namesInput.value = state.entries.join('\n');
  save(); render();
}
```

- The rule stays as it was, and its checkbox stays disabled while `state.parts` isn't empty.
- On the next Lock, `lockDraw` drops everyone the earlier parts made out. The toast says *Part 2 has 112
  entries: 8 earlier winners left out.*

**`#newDrawBtn`** (**New draw**, outlined) shows whenever locked or `state.parts` isn't empty.
- It asks `confirm('Start a new draw? This one stays in its exports and, once recorded, in the workbook.')`.
- On yes, it keeps the title, event, day, facility and the prize rows, with their won counts reset. It
  clears everything else, `parts` included.

### 5.10 Spinning

```js
async function spin(){
  const slots = prizeSlots(state.prizeRows), allPicks = [...state.parts.flatMap(p => p.picks), ...state.picks];
  if(busy || !state.locked || drawFinished({ entries: state.entries, slots, picks: state.picks, allPicks, rule: state.rule })) return;
  busy = true; render();
  const n = (state.picks.length ? Math.max(...state.picks.map(p => p.spin)) : 0) + 1;
  const plan = nextSpin(slots, allPicks);                       // { count, slots }
  const entrants = entrantsFor(state.entries, state.picks, state.rule);
  const won = await drawWinners({ seed: state.seed, spin: n, entrants, count: plan.count, sha256Hex });
  const at = new Date().toISOString();
  const picks = won.map((w, i) => ({ spin: n, place: i + 1, slotId: plan.slots[i] ? plan.slots[i].id : '',
    prize: plan.slots[i] ? plan.slots[i].label : '', ticket: ticketOf(state.entries, w.winner), winner: w.winner, result: 'won', drawnAt: at }));
  await play(entrants, picks);                                   // §5.11, §5.12, or the display window (§5.15)
  state.picks.push(...picks);
  save(); busy = false; reveal(picks); render();
}
```

**`#spinBtn`**
- It reads *Spin* for the Reel, *Draw* for the Pickle Jar, *Drawing…* while busy, and *Draw complete* once
  `drawFinished`.
- It is disabled while busy, while finished, and while there are prize problems.
- Space and Enter call `spin()` when the focus isn't in an input, a textarea, a select or a button.

**Draw again**
- Each winner of the **latest** spin with result `won` gets a **Draw again** button (`.redraw-btn`). It
  appears in the reveal, and in that winner's row of `#winnersList`, while no spin is running.
- It sets that pick's `result` to `'redrawn'` and saves.
- It shows the toast *Drawing again for <prize>: <name> is out of the draw.* (*Drawing again: <name> is out
  of the draw.* with no prize), and re-renders.
- The slot is free again, so the next spin draws it.
- In Phase 2 it also records the spin again (§7.9).

**The reveal** (`#reveal`) is a white card, its text centred:
- **Before the first spin:** *Next up*, and the prize (or *Spin when ready*).
- **After a spin of one winner:**
  - *Winner · <prize>*, or just *Winner*.
  - The name in Barlow Condensed 700, at 44px (32px on a phone), navy.
  - With tickets shown, *#047* first, alone, at 56px in `--green-dark`, and 1.1 s later the name fades in
    under it.
  - A Numbers list shows the number only.
- **After a spin of several winners:** *Winners · Ball pack*, then one line per winner, *1 · Ana Reyes*,
  in Barlow Condensed 600, at 32px (24px on a phone). With 10 or more, use two columns.
- **Reduced motion.** With `matchMedia('(prefers-reduced-motion: reduce)').matches`, there is no reel
  transition and no jar mixing: the result shows after 400 ms.

### 5.11 The Reel

```html
<div id="reel" class="reel"><div id="reelStrip" class="strip"></div>
  <div class="dim top"></div><div class="dim bottom"></div><div class="band"></div><div class="pointer"></div></div>
```

**Sizes**
- `--row` is 64px: 52px below 600px wide, and 96px in big screen and in the display window.
- The window is `5 × --row` tall, `overflow:hidden`, with a `--navy-deep` background and radius 10px.
- Each row is Barlow Condensed 500, at `calc(var(--row) * .5)`, in `--paper`, with 22px side padding, on one
  line with an ellipsis.

**The overlays**
- `.dim.top` covers rows 1–2 and `.dim.bottom` covers rows 4–5, both `var(--dim)`.
- `.band` covers row 3: 2px green borders top and bottom, filled `var(--band)`.
- `.pointer` is a 14px green triangle at the band's left edge, pointing right.

**The strip**
- Before the first spin it shows the first five entries.
- A spin lands on place 1's winner. With several winners, the reveal lists them all.

`play` for the Reel:

```js
function playReel(entrants, winner){
  return new Promise(resolve => {
    const row = parseFloat(getComputedStyle(reel).getPropertyValue('--row')) || 64;
    const pick = () => entrants[Math.floor(Math.random() * entrants.length)];   // cosmetic rows only
    const seq = visibleRows.slice();
    for(let i = 0; i < 46; i++) seq.push(pick());
    const idx = seq.length; seq.push(winner, pick(), pick());
    reelStrip.innerHTML = seq.map(rowHtml).join('');                 // rowHtml escapes every name
    reelStrip.style.transition = 'none'; reelStrip.style.transform = 'translateY(0)'; void reelStrip.offsetHeight;
    const done = () => { reelStrip.children[idx].classList.add('hit'); visibleRows = seq.slice(idx - 2, idx + 3); resolve(); };
    if(reducedMotion()){ reelStrip.style.transform = `translateY(${-(idx - 2) * row}px)`; setTimeout(done, 400); return; }
    reelStrip.style.transition = 'transform 4.8s cubic-bezier(.08,.72,.12,1)';
    reelStrip.style.transform = `translateY(${-(idx - 2) * row}px)`;
    setTimeout(done, 4900);
  });
}
```

- `.hit` turns the winner's row green, at 700.
- `visibleRows` starts as `state.entries.slice(0, 5)`, padded with `''` when there are fewer than five.
- `rowHtml(e)` gives `<div class="r">…</div>`. With tickets shown, it is
  `<span class="n">#047</span><span>Ana Reyes</span>`, where `.n` is green, 700, at 0.85em, with a 64px
  minimum width.

### 5.12 The Pickle Jar

**The canvas**
- `<canvas id="jar" width="640" height="400">`, at `width:100%`, on a white card. The drum's centre is at
  (190, 190), with a radius of 150; the tray is at y = 372.
- **Availability.** `#styleJar` is disabled while the entrants, or the names in the box before Lock, number
  more than `JAR_MAX_ENTRIES` (300), with
  `title="The Pickle Jar holds up to 300 names; this draw has 412."`. With the Jar chosen and the list growing
  past 300, the style switches back to the Reel.

**The balls**
- One ball per current entrant (`entrantsFor`). The radius follows the count:

  ```js
  const ballRadius = n => Math.max(6, Math.min(19, 19 * Math.sqrt(40 / Math.max(1, n))));
  ```

  That is 19 up to 40 balls, about 11 at 120, and about 7 at 300.
- They alternate `--ball-a` and `--ball-b`, with a 1.5px `--ball-line` outline. Six holes are drawn while the
  radius is 10 or more.
- **Labels,** only while there are `JAR_LABEL_MAX` (40) balls or fewer: the initials (the first letter of
  the first two words, upper case), or for a Numbers list the number, at a smaller size. Above 40, no ball
  carries a label: the winning ball shows its name only in the reveal.
- **The drum** is a 10px navy ring around a paper disc, with 8 spokes that turn with the drum.

**`play` for the Jar**
- It runs the mockup's physics on `requestAnimationFrame`. Per frame:

  | Constant | Value |
  | --- | --- |
  | gravity | 0.38 |
  | random kick while mixing | ±0.3 |
  | damping | 0.996 |
  | wall restitution | 0.75 |
  | drum spin | grows by 0.002 to 0.07 while mixing, then decays ×0.97 |
  | wall's tangential kick | `ω·R·0.22` |
  | ball to ball | pushed apart, normal velocity exchanged ×0.9 |

- It mixes for 2.8 s. Then each winner's ball, in place order and 0.6 s apart:
  1. leaves the drum;
  2. drops to the chute at the drum's bottom over 0.7 s;
  3. rolls along the tray over 1.1 s, easing out cubic, to x = `560 − 34 × (balls in the tray)`, its radius
     growing to 33.

  It resolves when the last ball stops. The reveal (§5.10) shows the names.
- **The tray** keeps up to 12 drawn balls at radius 15, newest on the right. A pick drawn again takes its
  ball out of the tray.
- **Physics with many balls.** With more than 120 balls, check each ball only against balls in its own and
  neighbouring cells, on a grid of `4 × radius` squares, to keep the frame rate. Otherwise check every pair,
  as below.
- **The mockup's working code** is the reference. Port its step with the constants above (`r` is
  `ballRadius(balls.length)`):

```js
// reference: physics step (one frame)
ang += om; if(mix) om = Math.min(om + .002, .07); else om *= .97;
for(const b of balls){
  b.vy += .38; if(mix){ b.vx += (Math.random() - .5) * .6; b.vy += (Math.random() - .5) * .6; }
  b.x += b.vx; b.y += b.vy; b.vx *= .996; b.vy *= .996; b.a += b.vx * .04;
  const dx = b.x - CX, dy = b.y - CY, d = Math.hypot(dx, dy) || 1;
  if(d > R - r){ const nx = dx / d, ny = dy / d; b.x = CX + nx * (R - r); b.y = CY + ny * (R - r);
    const vn = b.vx * nx + b.vy * ny; if(vn > 0){ b.vx -= 1.75 * vn * nx; b.vy -= 1.75 * vn * ny; }
    b.vx += -ny * om * R * .22; b.vy += nx * om * R * .22; }
}
for(let i = 0; i < balls.length; i++) for(let j = i + 1; j < balls.length; j++){
  const p = balls[i], q = balls[j], dx = q.x - p.x, dy = q.y - p.y, d = Math.hypot(dx, dy);
  if(d > 0 && d < 2 * r){ const o = (2 * r - d) / 2, nx = dx / d, ny = dy / d;
    p.x -= nx * o; p.y -= ny * o; q.x += nx * o; q.y += ny * o;
    const rv = (q.vx - p.vx) * nx + (q.vy - p.vy) * ny;
    if(rv < 0){ p.vx += rv * nx * .9; p.vy += rv * ny * .9; q.vx -= rv * nx * .9; q.vy -= rv * ny * .9; } }
}
```

- **Idle.** While idle, the loop runs on so the balls settle, then stops after 3 s with no spin to save
  power. The next spin restarts it.

### 5.13 Winners and exports

**`#winnersList`**
- An `<ol>`, one row per pick, in spin then place order: the prize (or *Spin n*), a ticket chip when tickets
  are shown, the winner, and a **Draw again** button where §5.10 allows one.
- A pick drawn again is struck through, with *(drawn again)* after it.
- On a continued draw, each part has a heading: *Part 1 · Draw code H7XK2M9P*, *Part 2 · Draw code …*.
- Empty, it reads *No spins yet.*

**The buttons.** All four are enabled only once a draw is locked, or `parts` isn't empty.
- **`#copyBtn`** (**Copy results**) copies each part's `drawText({...})`, joined by a blank line.
  - `eventLine` is the header's event line, and `picks` is that part's picks.
  - It uses `navigator.clipboard.writeText`. When that is refused, it falls back to a hidden textarea and
    `document.execCommand('copy')`.
  - Toast: *Results copied.*, or *Couldn't reach the clipboard. Use Download .txt.*
- **`#txtBtn`** (**Download .txt**) downloads the same text as `<base>-draw.txt`.
  - `<base>` is the title (or *prize-draw*) in lower case, with every run of non-word characters turned into
    `-`.
  - Then come `-` and the local date of the first part's `lockedAt` (`YYYY-MM-DD`).
- **`#pngBtn`** (**Save image**) draws the winners board on a canvas and downloads `<base>-winners.png`.
  - It is 1200px wide, and `220 + 64 × picks` tall (picks drawn again included).
  - A navy band 120px tall, with *S.A.G.E.* in paper and *LUCKY DINK* in green (Barlow Condensed 700,
    56px).
  - The title and event line under it (Inter 600, 28px, navy).
  - Then one row per pick, as in `#winnersList`, without the buttons (Barlow Condensed 600, 34px). Picks
    drawn again are in `--ink-soft`, struck through.
  - A footer, *Draw code HXTE7Q4M · list 3de62d1c · check it at sage-match-control.github.io/tools/prize-draw*,
    with one line per part (Inter, 20px, `--ink-soft`).
  - Await `document.fonts.ready` before drawing.
- **`#listBtn`** (**Copy numbered list**) copies one line per entry, `001 Ana Reyes` (for a Numbers list,
  just the numbers), so the list can be posted before the draw.

**`#checkBtn`** (**Check this draw**)
- For each part, and then the current one, it runs:

  ```js
  verifyDraw({ entries, seed, rule, picks, sha256Hex })
  ```
- `#checkResult` lists every pick:
  - a match: *✓ Spin 2 · 1: Francis Go*
  - no match: *✗ Spin 3 · 1: recorded Ben Cruz, the rule gives Dante Lim*
- Then it gives a summary: *Every winner checks out.*, or *2 winners don't match this list and code.*

### 5.14 How it works, Big screen

**`#howBtn`** opens `#howDialog`, a `<dialog>`. It closes on Escape, on its **Close** button, and on a click
on the backdrop. Its title is **No Rigging Zone**, and its copy, exactly:

> **The list is locked before anything is drawn.** Once the first spin happens, nobody can quietly add or
> remove a name. The page shows a fingerprint of the locked list: change one letter and it changes. If the
> list has to change after that, the draw closes and a new part starts, and both parts stay on record.
>
> **A random code decides every spin.** When the list is locked, the page makes a random draw code. For each
> spin, every name still in the draw gets a code made from the draw code, the spin number and the name (a
> SHA-256 hash), and the smallest wins. Nobody can steer it, not even the person running the draw. The
> spinning on screen only shows a result that was already fixed. The prizes never enter it, so changing
> their order can't favour anyone.
>
> **Anyone can check it.** With the list, the draw code and the rule, *Check this draw* works every spin out
> again and shows whether each recorded winner is the one the code gives.

**`#bigBtn`** (**Big screen**) toggles `body.big`. Its label reads *Exit big screen* while it's on, and
Escape also leaves it. In big screen:
- `.setup` and `#eventLine` are hidden, and the stage fills the width.
- The reel's `--row` is 96px, the reveal's name 72px, and `#spinBtn` has 22px text and 18px padding.
- It uses no Fullscreen API.

### 5.15 The display window

**What it is.** **Open display** (`#displayBtn`) opens the same page in display mode, for the TV:

```js
const w = window.open('/tools/prize-draw?display=1', 'sage-prize-display', 'popup,width=1280,height=720');
if(!w) window.open('/tools/prize-draw?display=1', 'sage-prize-display');   // popups blocked: a tab
```

The operator drags it to the TV and presses F11. Both windows are the same browser, on one computer. They
talk over `BroadcastChannel(DISPLAY_CHANNEL)`, and nothing else.

**Display mode** (`new URLSearchParams(location.search).get('display') === '1'`):
- The page shows only the header's wordmark and `#eventLine`, the reel or jar at big-screen size, and the
  reveal.
- It shows no controls and no setup, never reads or writes `localStorage[STORE_KEY]`, and loads no registry.
- Until the controls answer, it reads *Waiting for the controls… Open Lucky Dink in another tab and press
  Open display.*

**The messages.** Every one is a plain object with a `type`:

| From | `type` | Carries | When |
| --- | --- | --- | --- |
| display | `hello` | — | on load |
| display | `here` | — | every 2 s |
| display | `played` | `spin` | when an animation finishes |
| display | `bye` | — | on `pagehide` |
| controls | `scene` | `title`, `eventLine`, `style`, `showTickets`, `numberList`, `visibleRows`, `entrants`, `reveal` (the last reveal's picks, or null), `nextPrize` | in answer to `hello`, and after any change the display shows |
| controls | `play` | `spin`, `style`, `entrants`, `picks` (spin, place, prize, ticket, winner) | at each spin |
| controls | `close` | — | when the controls tab closes (`pagehide`) |

**The controls side**
- A display counts as connected from its first `hello` or `here` until 5 s pass with neither.
- **While connected:**
  - The controls' stage is replaced by `#displayStatus`: *Showing on the display window. [Bring it back]*.
    **Bring it back** sends `close` and returns to showing locally.
  - `#displayBtn` reads *Display open*.
- **`play(entrants, picks)`:**
  - When connected, it posts `play` and resolves on the matching `played`, or after 15 s, whichever comes
    first.
  - Otherwise it runs `playReel(entrants, picks[0].winner)` or `playJar(entrants, picks)` locally.
- **After each spin**, the controls send `scene` with the new reveal.
- **The results stay where they are:** the winners list, Draw again and the exports all stay in the
  controls. The display never decides anything.

**The display side**
- On `scene`, it redraws.
- On `play`, it runs the same `playReel` or `playJar` with the given entrants and winners, shows the
  reveal, then posts `played`.
- On `close`, it shows *The controls have closed.*

### 5.16 `_tests/unit/tools/prize-draw-page.test.mjs` (new)

```js
// The prize draw page (sage-docs/docs/specs/.../prize-draw-spec.md §5): one module script that takes its rules from
// the engine, keeps no copy of them, and has the elements the spec names.
import test from 'node:test';
import assert from 'node:assert/strict';
import fs from 'node:fs';
import path from 'node:path';
import { SITE_ROOT } from '../../helpers/paths.mjs';
import { stripComments } from '../../helpers/extract.mjs';
import * as draw from '../../../lib/v1/domain/prize-draw.js';

const page = fs.readFileSync(path.join(SITE_ROOT, 'tools/prize-draw.html'), 'utf8');
const script = page.slice(page.indexOf('<script type="module">'), page.indexOf('</script>', page.indexOf('<script type="module">')));
// Every name the script declares, indented or not: function declarations and const/let/var bindings.
const declared = code => new Set([...code.matchAll(/\b(?:function\s*\*?\s*([A-Za-z_$][\w$]*)|(?:const|let|var)\s+([A-Za-z_$][\w$]*)\s*=)/g)]
  .map(m => m[1] || m[2]));

test('one inline script, a module, importing only root-absolute engine paths', () => {
  const inline = [...page.matchAll(/<script\b([^>]*)>/g)].map(m => m[1].trim()).filter(a => !/\bsrc=/.test(a));
  assert.deepEqual(inline, ['type="module"']);
  const froms = [...script.matchAll(/from\s+'([^']+)'/g)].map(m => m[1]);
  assert.ok(froms.includes('/lib/v1/domain/prize-draw.js'));
  for (const f of froms) assert.ok(f.startsWith('/lib/v1/'), f);
});

test('declares nothing the draw module exports', () => {
  const mine = declared(stripComments(script));
  for (const name of Object.keys(draw)) assert.ok(!mine.has(name), `the page declares ${name}`);
});

test('draws winners with drawWinners over SHA-256, and uses Math.random only for show', () => {
  const code = stripComments(script);
  assert.match(code, /drawWinners\(\{[^}]*sha256Hex/);
  const uses = (code.match(/Math\.random\(\)/g) || []).length;
  assert.ok(uses <= 6, `Math.random() appears ${uses} times: only the reel's filler rows and the jar's physics may use it`);
});

test('has every element the spec names, and no seed box', () => {
  const ids = ['eventLine', 'howBtn', 'displayBtn', 'bigBtn', 'styleReel', 'styleJar', 'reel', 'reelStrip', 'jar',
    'displayStatus', 'reveal', 'spinBtn', 'seedLine', 'titleInput', 'eventSelect', 'daySelect', 'facilitySelect',
    'srcPaste', 'srcPlayers', 'srcCheckedIn', 'srcNumbers', 'loadPlayersBtn', 'loadCheckedInBtn', 'numberCount',
    'fillNumbersBtn', 'namesInput', 'namesCount', 'namesProblems', 'prizeRows', 'addPrizeBtn', 'prizesCount',
    'prizeProblems', 'removeWinners', 'showTickets', 'lockBtn', 'unlockBtn', 'editListBtn', 'newDrawBtn',
    'winnersList', 'copyBtn', 'txtBtn', 'pngBtn', 'listBtn', 'checkBtn', 'checkResult', 'howDialog', 'toast'];
  for (const id of ids) assert.match(page, new RegExp(`id="${id}"`), id);
  assert.doesNotMatch(page, /id="seedInput"/);
  assert.match(page, /<title>Prize draw · S\.A\.G\.E\.<\/title>/);
  assert.match(page, /LUCKY DINK/);
  assert.match(page, /No Rigging Zone/);
  assert.match(page, /Draw again/);
  assert.doesNotMatch(page, /Big Dill/i);
  assert.doesNotMatch(page, /\babsent\b/i);
});
```

`stripComments` is an existing export of `_tests/helpers/extract.mjs`. The test keeps its own `declared()`,
because the helper's `functionNames` finds only declarations at the start of a line, and a page's script is
indented.

## 6. Mission Control and the tools roster

### 6.1 `tools/control-center.html`: the Open prize draw tile

Both edits are in `renderScheduleLauncher()`.

**Edit M1.** Find (once):

```js
  const scoresheetHref = '/tools/scoresheet-generator' + (day ? `?day=${encodeURIComponent(day.key)}` : '');
```

Add directly after it:

```js
  // The prize draw opens on this event and day (prize-draw-spec §6).
  const prizeDrawHref = '/tools/prize-draw?event=' + encodeURIComponent(CURRENT_EVENT_KEY) + (day ? `&day=${encodeURIComponent(day.key)}` : '');
```

**Edit M2.** Find (once):

```js
    tile(scoresheetHref, 'Open scoresheet generator', day ? `Printable scoresheets for ${escapeHtml(day.label)}` : 'Printable scoresheets');
```

Replace it with:

```js
    tile(scoresheetHref, 'Open scoresheet generator', day ? `Printable scoresheets for ${escapeHtml(day.label)}` : 'Printable scoresheets') +
    tile(prizeDrawHref, 'Open prize draw', 'Lucky Dink: raffles and door prizes');
```

- The tile opens in a new tab, like the others.
- `.mc-tiles` already fits any number of tiles in one row on a desktop and stacks them on a phone, so no CSS
  changes.
- The tile shows for every selected event, signed in or not.
- The site's `tools/control-center.html` opens Control Center in the same browser the prize draw runs in,
  which is what lets **Load checked in** and Phase 2's recording use its sign-in.

### 6.2 `tools/index.html`: the roster entry

Find (once):

```html
    <div class="cap-row">
      <div>
        <p class="cap-name">Operator Sign-In</p>
```

Insert directly before it:

```html
    <div class="cap-row">
      <div>
        <p class="cap-name"><a href="/tools/prize-draw">Prize Draw</a></p>
        <p class="cap-desc">Lucky Dink: draw raffle and door-prize winners on a big screen, from a pasted list, the day's players, who has checked in, or ticket numbers. Prizes can have a quantity and be drawn in any order, and every draw has a code, so anyone can check it afterwards.</p>
      </div>
      <span class="status"><span class="dot"></span>Live</span>
    </div>

```

## 7. Phase 2: recording winners in the workbook

### 7.1 What it does

- **Every winner** is written as one row of a `DRAWS` tab in the facility workbook of the draw's event day
  (`#facilitySelect`, or the day's only facility). A spin of three winners writes three rows.
- **Spin 1, place 1's row** also carries the whole locked list in one cell, and the draw it continues, so
  the record alone is enough to re-run the draw.
- **Retries are safe.** Sending a spin again writes nothing.
- **The one change a recorded winner takes** is being marked drawn again, while their spin is the draw's
  latest.
- **Nothing is published.** No public page reads `DRAWS`, and a Sheets API write fires no `onEdit` trigger.
- **Past draws.** The page can list the draws recorded for its day and facility, check them, and continue
  one.

### 7.2 The `DRAWS` tab

| Col | Header | Value |
| --- | --- | --- |
| A | `drawId` | 8 characters, made at Lock |
| B | `title` | the draw's title, or empty |
| C | `spin` | 1, 2, 3 … (a number) |
| D | `place` | 1, 2 … within the spin (a number) |
| E | `prize` | the prize slot's label, e.g. `Ball pack (2 of 3)`, or empty |
| F | `ticket` | `001` … (text), or empty |
| G | `winner` | the entry, exactly as in the list |
| H | `result` | `won` or `redrawn` |
| I | `drawnAt` | ISO 8601 UTC: the API's clock when the spin was first written |
| J | `seed` | the draw code |
| K | `fingerprint` | 64 lowercase hex |
| L | `rule` | `remove-winners` or `keep-winners` |
| M | `entries` | the list's length (a number) |
| N | `list` | spin 1, place 1 only: every entry, one per line in the cell |
| O | `continues` | spin 1, place 1 only: the draw id this draw continues, or empty |

- **Writes.**
  - Row 1 is the header, and it is frozen.
  - Every write is whole rows, A to O, at row 2 or below, written RAW, so a name starting with `=` stays
    text.
  - A new spin's rows go after the last non-blank row, all in one request.
- **The API creates the tab** on the first spin when it is missing or empty. A tab whose header row differs
  is refused (`409 draws_layout`).
- **Size.** The 1000-entry, 40-character limits keep the list under 41,000 characters, inside Sheets' limit
  of 50,000 per cell.

### 7.3 The two routes

| Method and path | What it does | Auth | Success |
| --- | --- | --- | --- |
| `GET /v3/days/{day}/facilities/{facility}/prize-draws` | list the draws recorded in that workbook | operator | `200 { day, facility, draws: [{ drawId, title, seed, fingerprint, rule, entries, list, continues, picks: [{ spin, place, prize, ticket, winner, result, drawnAt }] }] }`, `Cache-Control: no-store` |
| `PUT /v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}` | record one spin's winners | operator | `200 { day, facility, drawId, spin, rows, created, drawnAt }` |

**The PUT body:** `{ title?, seed, fingerprint, rule, entries, list?, continues?, winners: [{ place, prize?, ticket?, winner, result }] }`.
- `list` and `continues` are read on spin 1 only.
- `winners` are in place order, 1 to 50 of them.

**Errors:**

| Status | Code | When |
| --- | --- | --- |
| `400` | `validation_error` | a bad id, spin or body; a list that doesn't hash to `fingerprint`; a winner not in the list |
| `401` | `unauthorized` | no operator token; desk and scorer tokens are refused |
| `403` | `forbidden` | the event is archived (the PUT only) |
| `404` | — | an unknown day or facility |
| `405` | `method_not_allowed` | an unsupported method |
| `409` | `draws_layout` | the `DRAWS` header row is wrong |
| `409` | `draw_conflict` | a spin out of order, other settings than spin 1, or other winners than a spin already recorded |
| `415` | `unsupported_media_type` | a body that is not JSON |
| `502` / `503` | — | Sheets failed, or is busy |

The PUT has no GET twin, because spins are read with the list. That is a REST-conventions exception (M2),
recorded in `rest-conventions.test.mjs` and on the API page.

### 7.4 New files

Create each exactly. All of them were dry-run.

`src/draws/domain/drawsTab.mjs`:

```js
import { DrawsLayoutError } from "../../shared/errors.mjs";

// The DRAWS tab: one row per winner of a prize draw (a spin that draws three winners writes three rows, places 1–3),
// in the facility workbook the draw was recorded to. The draw's entry list, and the draw it continues, sit on its
// spin 1, place 1 row only. See sage-docs/docs/specs/.../prize-draw-spec.md §7.2.

export const DRAWS_TAB = "DRAWS";
export const HEADERS = [
    "drawId", "title", "spin", "place", "prize", "ticket", "winner", "result",
    "drawnAt", "seed", "fingerprint", "rule", "entries", "list", "continues",
];
export const READ_RANGE = "DRAWS!A:O";
export const HEADER_RANGE = "DRAWS!A1:O1";

/** "DRAWS!A7:O7": the one row a winner is written to. */
export function rowRange(row) {
    if (!Number.isInteger(row) || row < 2) throw new Error(`drawsTab: not a data row: ${row}`);
    return `DRAWS!A${row}:O${row}`;
}

/** True for exactly one whole data row, A to O, at row 2 or below: SheetsClient's DRAWS allowlist. */
export function isDrawRowRange(range) {
    const m = /^DRAWS!A([1-9]\d*):O([1-9]\d*)$/.exec(String(range));
    return !!m && m[1] === m[2] && Number(m[1]) >= 2;
}

const text = v => (v === null || v === undefined ? "" : String(v));
const blank = r => !r || r.every(v => text(v).trim() === "");

/**
 * The tab as read with readValues(sheetId, READ_RANGE) (UNFORMATTED_VALUE).
 *   null (no tab)            -> { exists: false, rows: [], nextRow: 2 }
 *   [] (a tab with nothing)  -> { exists: true, needsHeader: true, rows: [], nextRow: 2 }
 * Otherwise row 1 must be HEADERS exactly, or DrawsLayoutError (409). nextRow is the row after the last
 * non-blank one, so a blank row left by hand is never filled in.
 * @returns {{ exists: boolean, needsHeader?: boolean, rows: object[], nextRow: number }}
 */
export function readDrawsTab(values) {
    if (values === null) return { exists: false, rows: [], nextRow: 2 };
    if (values.length === 0 || blank(values[0])) {
        if (values.slice(1).some(r => !blank(r))) throw new DrawsLayoutError();
        return { exists: true, needsHeader: true, rows: [], nextRow: 2 };
    }
    const head = HEADERS.map((_, i) => text(values[0][i]).trim());
    if (head.some((h, i) => h !== HEADERS[i])) throw new DrawsLayoutError();
    const rows = [];
    let last = 0;
    values.forEach((r, i) => {
        if (i === 0 || blank(r)) return;
        last = i;
        const c = k => r[HEADERS.indexOf(k)];
        rows.push({
            row: i + 1,
            drawId: text(c("drawId")).trim(),
            title: text(c("title")),
            spin: Number(c("spin")),
            place: Number(c("place")),
            prize: text(c("prize")),
            ticket: text(c("ticket")),
            winner: text(c("winner")),
            result: text(c("result")),
            drawnAt: text(c("drawnAt")),
            seed: text(c("seed")),
            fingerprint: text(c("fingerprint")),
            rule: text(c("rule")),
            entries: Number(c("entries")),
            list: text(c("list")),
            continues: text(c("continues")),
        });
    });
    return { exists: true, rows, nextRow: last + 2 };
}

/** The draw's rows, in spin then place order. */
export function drawRows(tab, drawId) {
    return tab.rows.filter(r => r.drawId === drawId).sort((a, b) => a.spin - b.spin || a.place - b.place);
}

/** A winner's row as written: the HEADERS order; list and continues only on spin 1, place 1. */
export function pickRowValues(p) {
    const first = p.spin === 1 && p.place === 1;
    return [
        p.drawId, p.title, p.spin, p.place, p.prize, p.ticket, p.winner, p.result,
        p.drawnAt, p.seed, p.fingerprint, p.rule, p.entries, first ? p.list.join("\n") : "", first ? p.continues : "",
    ];
}

/**
 * Every draw on the tab, in the order its first row appears, for GET …/prize-draws. A draw's settings, list and
 * `continues` come from its spin 1, place 1 row; a draw whose first row was cut by hand has list [].
 */
export function groupDraws(tab) {
    const order = [];
    const by = new Map();
    for (const r of tab.rows) {
        if (!r.drawId) continue;
        if (!by.has(r.drawId)) { by.set(r.drawId, []); order.push(r.drawId); }
        by.get(r.drawId).push(r);
    }
    return order.map(drawId => {
        const rows = by.get(drawId).sort((a, b) => a.spin - b.spin || a.place - b.place);
        const head = rows.find(r => r.spin === 1 && r.place === 1);
        const first = head ?? rows[0];
        return {
            drawId,
            title: first.title,
            seed: first.seed,
            fingerprint: first.fingerprint,
            rule: first.rule,
            entries: first.entries,
            list: head && head.list !== "" ? head.list.split("\n") : [],
            continues: head ? head.continues : "",
            picks: rows.map(r => ({ spin: r.spin, place: r.place, prize: r.prize, ticket: r.ticket, winner: r.winner, result: r.result, drawnAt: r.drawnAt })),
        };
    });
}
```

`src/draws/domain/drawRequest.mjs`:

```js
import { ValidationError } from "../../shared/errors.mjs";

// Validates the prize-draw routes' input. The limits are the site's lib/v1/domain/prize-draw.js
// (MAX_ENTRIES, MAX_ENTRY_LENGTH, MAX_QUANTITY), so a draw the page locks is always one this route takes.
// See sage-docs/docs/specs/.../prize-draw-spec.md §7.3.

export const MAX_ENTRIES = 1000;
export const MAX_ENTRY_LENGTH = 40;
export const MAX_WINNERS_PER_SPIN = 50;
export const MAX_LABEL_LENGTH = 80;
const DRAW_ID = /^[2-9A-HJKMNP-TV-Z]{8}$/;   // eight characters of the seed alphabet (no I, L, O, U, 0, 1)
const ID_MESSAGE = "must be 8 characters from 2-9 and A-Z without I, L, O or U";

/** "K7Q4M2XA" -> "K7Q4M2XA". */
export function parseDrawIdParam(param) {
    if (typeof param !== "string" || !DRAW_ID.test(param)) throw new ValidationError(`drawId ${ID_MESSAGE}`);
    return param;
}

/** "3" -> 3. A spin number from the path: 1–9999. */
export function parseSpinParam(param) {
    if (typeof param !== "string" || !/^[1-9]\d{0,3}$/.test(param)) {
        throw new ValidationError("spin must be a spin number, such as 3");
    }
    return Number(param);
}

const str = (v, name, { max, required = false }) => {
    if (typeof v !== "string") throw new ValidationError(`${name} must be a string`);
    const s = v.trim();
    if (required && !s) throw new ValidationError(`${name} is required`);
    if (s.length > max) throw new ValidationError(`${name} is at most ${max} characters`);
    return s;
};
const oneOf = (v, name, values) => {
    if (!values.includes(v)) throw new ValidationError(`${name} must be ${values.map(x => `"${x}"`).join(" or ")}`);
    return v;
};

/**
 * The body of PUT …/prize-draws/{drawId}/spins/{spin} -> the spin, validated. `winners` are the spin's winners in
 * place order (1, 2 …). `list` is required on spin 1 and ignored on any other, and so is `continues` (the draw id
 * this draw continues, or empty).
 */
export function parseSpinRequest(body, spin) {
    if (body === null || typeof body !== "object" || Array.isArray(body)) throw new ValidationError("Body must be a JSON object");
    if (typeof body.seed !== "string" || !DRAW_ID.test(body.seed)) throw new ValidationError(`seed ${ID_MESSAGE}`);
    if (typeof body.fingerprint !== "string" || !/^[0-9a-f]{64}$/.test(body.fingerprint)) {
        throw new ValidationError("fingerprint must be 64 lowercase hex characters");
    }
    if (!Number.isInteger(body.entries) || body.entries < 2 || body.entries > MAX_ENTRIES) {
        throw new ValidationError(`entries must be a whole number from 2 to ${MAX_ENTRIES}`);
    }
    const w = body.winners;
    if (!Array.isArray(w) || w.length < 1 || w.length > MAX_WINNERS_PER_SPIN) {
        throw new ValidationError(`winners must list the spin's 1 to ${MAX_WINNERS_PER_SPIN} winners`);
    }
    const winners = w.map((x, i) => {
        if (x === null || typeof x !== "object") throw new ValidationError("each winner must be an object");
        if (x.place !== i + 1) throw new ValidationError("winners must be in place order: 1, 2, 3 …");
        const ticket = x.ticket === undefined ? "" : x.ticket;
        if (typeof ticket !== "string" || !/^\d{0,4}$/.test(ticket)) throw new ValidationError("ticket must be up to 4 digits, or empty");
        return {
            place: x.place,
            prize: x.prize === undefined ? "" : str(x.prize, "prize", { max: MAX_LABEL_LENGTH }),
            ticket,
            winner: str(x.winner, "winner", { max: MAX_ENTRY_LENGTH, required: true }),
            result: oneOf(x.result, "result", ["won", "redrawn"]),
        };
    });
    if (new Set(winners.map(x => x.winner)).size !== winners.length) throw new ValidationError("a spin can't draw the same entry twice");
    const out = {
        title: body.title === undefined ? "" : str(body.title, "title", { max: MAX_LABEL_LENGTH }),
        seed: body.seed,
        fingerprint: body.fingerprint,
        rule: oneOf(body.rule, "rule", ["remove-winners", "keep-winners"]),
        entries: body.entries,
        winners,
        list: [],
        continues: "",
    };
    if (spin === 1) {
        const list = body.list;
        if (!Array.isArray(list) || list.length !== body.entries) {
            throw new ValidationError("list must be the draw's entries, one string each, as many as entries");
        }
        for (const e of list) {
            if (typeof e !== "string" || e.trim() !== e || !e || e.length > MAX_ENTRY_LENGTH || e.includes("\n") || e.includes("|")) {
                throw new ValidationError(`each list entry is 1 to ${MAX_ENTRY_LENGTH} characters, trimmed, with no line break or "|"`);
            }
        }
        if (new Set(list).size !== list.length) throw new ValidationError("list has an entry twice");
        if (winners.some(x => !list.includes(x.winner))) throw new ValidationError("winner is not in the list");
        if (body.continues !== undefined && body.continues !== "" && (typeof body.continues !== "string" || !DRAW_ID.test(body.continues))) {
            throw new ValidationError(`continues ${ID_MESSAGE}, or empty`);
        }
        out.list = list;
        out.continues = body.continues ?? "";
    }
    return out;
}
```

`src/draws/DrawService.mjs`:

```js
import { createHash } from "node:crypto";
import { DrawConflictError, ForbiddenError, UnknownFacilityError, ValidationError } from "../shared/errors.mjs";
import { READ_RANGE, drawRows, groupDraws, pickRowValues, readDrawsTab } from "./domain/drawsTab.mjs";

// Records each spin of a prize draw as rows of a facility workbook's DRAWS tab (one row per winner), and lists the
// draws recorded there, so a draw can be reopened and checked later. Writes nothing else: no publish follows,
// because nothing on the public pages reads DRAWS. See sage-docs/docs/specs/.../prize-draw-spec.md §7.
const SETTINGS = ["seed", "fingerprint", "rule", "entries"];

export class DrawService {
    /**
     * @param {{ configStore: import("../shared/ports.mjs").RegistryReader,
     *           sheets: import("../shared/ports.mjs").DrawSheets,
     *           logger: import("../shared/Logger.mjs").Logger, now?: () => Date }} deps
     */
    constructor({ configStore, sheets, logger, now = () => new Date() }) {
        this.configStore = configStore;
        this.sheets = sheets;
        this.logger = logger;
        this.now = now;
    }

    async #facility(day, facilityName) {
        const config = await this.configStore.get();
        const { event, facilities } = config.getDay(day);          // UnknownSyncDayError -> 404 on /v3
        const facility = facilities.find(f => f.name === facilityName);
        if (!facility) throw new UnknownFacilityError(facilityName, facilities);
        return { config, event, sheetId: facility.sheetId };
    }

    // -> the 200 body of GET …/prize-draws.
    async listDraws({ day, facilityName }) {
        const { sheetId } = await this.#facility(day, facilityName);
        const tab = readDrawsTab(await this.sheets.readValues(sheetId, READ_RANGE));
        return { day, facility: facilityName, draws: groupDraws(tab) };
    }

    // input: parseSpinRequest's result. -> the 200 body of PUT …/spins/{spin}.
    async recordSpin({ day, facilityName, drawId, spin, input, actor }) {
        const { config, event, sheetId } = await this.#facility(day, facilityName);
        if (config.getEvent(event).archived) throw new ForbiddenError("This event is archived: no more draws are recorded for it");
        if (spin === 1) {
            const fingerprint = createHash("sha256").update(input.list.join("\n"), "utf8").digest("hex");
            if (fingerprint !== input.fingerprint) throw new ValidationError("fingerprint is not the SHA-256 of the list");
        }

        let tab = readDrawsTab(await this.sheets.readValues(sheetId, READ_RANGE));   // DrawsLayoutError -> 409
        if (!tab.exists || tab.needsHeader) {
            await this.sheets.createDrawsTab(sheetId);
            tab = { exists: true, rows: [], nextRow: 2 };
        }

        const rows = drawRows(tab, drawId);
        const head = rows.find(r => r.spin === 1 && r.place === 1);
        if (spin > 1) {
            if (!head) throw new DrawConflictError(`Draw ${drawId} has no spin 1 in this workbook. Record its spins in order.`);
            if (!rows.some(r => r.spin === spin - 1)) throw new DrawConflictError(`Draw ${drawId} has no spin ${spin - 1} yet. Record its spins in order.`);
            const list = head.list.split("\n");
            if (input.winners.some(w => !list.includes(w.winner))) throw new ValidationError("winner is not in the list");
        }
        if (head) {
            for (const k of SETTINGS) {
                if (head[k] !== input[k]) throw new DrawConflictError(`Draw ${drawId} is recorded with another ${k}. Nothing was written.`);
            }
        }

        const same = rows.filter(r => r.spin === spin);
        const done = (written, created, drawnAt) => {
            if (written.length) this.logger.info(`draw ${day}/${facilityName} ${drawId} spin ${spin} rows ${written.join(",")} by=${actor?.kind}`);
            return { day, facility: facilityName, drawId, spin, rows: written.length ? written : same.map(r => r.row), created, drawnAt };
        };
        if (same.length) {
            const conflict = () => new DrawConflictError(`Spin ${spin} of draw ${drawId} is already recorded with other winners. Nothing was written.`);
            if (same.length !== input.winners.length) throw conflict();
            const changes = [];
            for (const w of input.winners) {
                const at = same.find(r => r.place === w.place);
                if (!at || at.winner !== w.winner) throw conflict();
                if (at.result === w.result) continue;
                // The one change a recorded winner takes: drawn again, while this is still the draw's latest spin.
                const isLast = !rows.some(r => r.spin > spin);
                if (!(at.result === "won" && w.result === "redrawn" && isLast)) throw conflict();
                changes.push({ row: at.row, values: pickRowValues({ ...at, drawId, result: "redrawn", list: at.list === "" ? [] : at.list.split("\n") }) });
            }
            if (changes.length) await this.sheets.writeDrawRows(sheetId, changes);
            return done(changes.map(c => c.row), false, same[0].drawnAt);
        }

        const drawnAt = this.now().toISOString();
        const writes = input.winners.map((w, i) => ({
            row: tab.nextRow + i,
            values: pickRowValues({ drawId, spin, drawnAt, ...input, ...w }),
        }));
        await this.sheets.writeDrawRows(sheetId, writes);
        return done(writes.map(x => x.row), true, drawnAt);
    }
}
```

`src/draws/routes.mjs`:

```js
import express from "express";
import { parseDrawIdParam, parseSpinParam, parseSpinRequest } from "./domain/drawRequest.mjs";
import { asyncHandler } from "../server/asyncHandler.mjs";
import { requireJson } from "../server/requireJson.mjs";
import { methodNotAllowed } from "../server/methodNotAllowed.mjs";

// Prize-draw routes. See sage-docs/docs/specs/.../prize-draw-spec.md §7.
//
// Auth is a signed-in operator's bearer token (src/auth/middleware.mjs): a scorer or
// desk token, and the sync secret, are refused. No /v1 or legacy twin: it is new in /v3.

// Mounted at /v3 by src/app.mjs — paths here are relative to that.
export function drawV3Routes({ drawService, auth }) {
    const router = express.Router();

    /**
     * @openapi
     * /v3/days/{day}/facilities/{facility}/prize-draws:
     *   get:
     *     tags: [draws]
     *     operationId: listPrizeDraws
     *     summary: List the prize draws recorded in one facility workbook
     *     description: >
     *       Reads the workbook's DRAWS tab and answers each draw once, in the order its first row appears, with its
     *       settings, its entry list and the draw it continues (from its spin 1, place 1 row), and its winners in spin
     *       and place order. A workbook with no DRAWS tab answers an empty list. Archived events are read too.
     *       Operator bearer token only.
     *     security:
     *       - bearerAuth: []
     *     parameters:
     *       - name: day
     *         in: path
     *         required: true
     *         schema: { type: string }
     *         description: "A registered day key from event-data/config/events.json, e.g. upap-day1."
     *       - name: facility
     *         in: path
     *         required: true
     *         schema: { type: string }
     *         description: "A facility name on that day, URL-encoded. Compared exactly, and case-sensitive."
     *     responses:
     *       "200":
     *         description: "Every draw on the tab. Cache-Control: no-store."
     *         content:
     *           application/json:
     *             schema:
     *               type: object
     *               properties:
     *                 day: { type: string }
     *                 facility: { type: string }
     *                 draws:
     *                   type: array
     *                   items:
     *                     type: object
     *                     properties:
     *                       drawId: { type: string }
     *                       title: { type: string }
     *                       seed: { type: string }
     *                       fingerprint: { type: string }
     *                       rule: { type: string, enum: [remove-winners, keep-winners] }
     *                       entries: { type: integer }
     *                       list: { type: array, items: { type: string } }
     *                       continues: { type: string, description: "The draw this one continues, or empty." }
     *                       picks:
     *                         type: array
     *                         items:
     *                           type: object
     *                           properties:
     *                             spin: { type: integer }
     *                             place: { type: integer }
     *                             prize: { type: string }
     *                             ticket: { type: string }
     *                             winner: { type: string }
     *                             result: { type: string, enum: [won, redrawn] }
     *                             drawnAt: { type: string, format: date-time }
     *       "401":
     *         description: "No credentials, invalid or expired ones, or a scorer or desk token."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "404":
     *         description: "Unknown day or facility."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "409":
     *         description: "The DRAWS tab's header row is not the expected one (code draws_layout)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "502":
     *         description: "Google Sheets failed (code upstream_failure)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     */
    router.route("/days/:day/facilities/:facility/prize-draws")
        .get(auth.requireOperator, asyncHandler(async (req, res) => {
            const result = await drawService.listDraws({ day: req.params.day, facilityName: req.params.facility });
            res.set("Cache-Control", "no-store").status(200).json(result);
        }))
        .all(methodNotAllowed);

    /**
     * @openapi
     * /v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}:
     *   put:
     *     tags: [draws]
     *     operationId: recordPrizeDrawSpin
     *     summary: Record one spin of a prize draw
     *     description: >
     *       Writes the spin's winners, one row each, to the facility workbook's DRAWS tab (creating the tab, with its
     *       header row, the first time), as the API's service account. Spin 1 carries the draw's whole entry list,
     *       which must hash to `fingerprint`; later spins must follow the one before and carry spin 1's seed,
     *       fingerprint, rule and entry count. Sending a recorded spin again writes nothing (a retry); the one change
     *       it takes is a winner of the draw's latest spin marked drawn again (`redrawn`). Nothing on the public pages
     *       reads DRAWS, so nothing is published. Operator bearer token only. This resource has no GET: spins are read
     *       with GET …/prize-draws, the list the page reads.
     *     security:
     *       - bearerAuth: []
     *     parameters:
     *       - name: day
     *         in: path
     *         required: true
     *         schema: { type: string }
     *         description: "A registered day key from event-data/config/events.json, e.g. upap-day1."
     *       - name: facility
     *         in: path
     *         required: true
     *         schema: { type: string }
     *         description: "A facility name on that day, URL-encoded. Compared exactly, and case-sensitive."
     *       - name: drawId
     *         in: path
     *         required: true
     *         schema: { type: string, pattern: "^[2-9A-HJKMNP-TV-Z]{8}$" }
     *         description: "The draw's id, made by the page when the list is locked."
     *       - name: spin
     *         in: path
     *         required: true
     *         schema: { type: string, pattern: "^[1-9][0-9]{0,3}$" }
     *         description: "The spin's number, from 1."
     *     requestBody:
     *       required: true
     *       content:
     *         application/json:
     *           schema:
     *             type: object
     *             required: [seed, fingerprint, rule, entries, winners]
     *             properties:
     *               title: { type: string, maxLength: 80 }
     *               seed: { type: string, pattern: "^[2-9A-HJKMNP-TV-Z]{8}$" }
     *               fingerprint: { type: string, pattern: "^[0-9a-f]{64}$" }
     *               rule: { type: string, enum: [remove-winners, keep-winners] }
     *               entries: { type: integer, minimum: 2, maximum: 1000 }
     *               list: { type: array, items: { type: string, maxLength: 40 }, description: "Spin 1 only: every entry, in the locked order." }
     *               continues: { type: string, description: "Spin 1 only: the draw this one continues, or empty." }
     *               winners:
     *                 type: array
     *                 minItems: 1
     *                 maxItems: 50
     *                 items:
     *                   type: object
     *                   required: [place, winner, result]
     *                   properties:
     *                     place: { type: integer, minimum: 1, description: "1, 2 … in the order the rule ranks them." }
     *                     prize: { type: string, maxLength: 80 }
     *                     ticket: { type: string, pattern: "^[0-9]{0,4}$" }
     *                     winner: { type: string, maxLength: 40 }
     *                     result: { type: string, enum: [won, redrawn] }
     *     responses:
     *       "200":
     *         description: "The spin as stored."
     *         content:
     *           application/json:
     *             schema:
     *               type: object
     *               properties:
     *                 day: { type: string }
     *                 facility: { type: string }
     *                 drawId: { type: string }
     *                 spin: { type: integer }
     *                 rows: { type: array, items: { type: integer }, description: "The rows written, or the spin's rows when nothing was." }
     *                 created: { type: boolean }
     *                 drawnAt: { type: string, format: date-time }
     *       "400":
     *         description: "A bad draw id, spin or body, a list that doesn't hash to the fingerprint, or a winner not in the list (code validation_error)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "401":
     *         description: "No credentials, invalid or expired ones, or a scorer or desk token."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "403":
     *         description: "The event is archived (code forbidden)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "404":
     *         description: "Unknown day or facility."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "405":
     *         description: "Any method but PUT (code method_not_allowed). Carries Allow: PUT."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "409":
     *         description: "The DRAWS tab's header row is not the expected one (code draws_layout), or the spin doesn't fit the draw already recorded: out of order, other settings than spin 1, or other winners than a spin already recorded (code draw_conflict). Nothing was written."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "415":
     *         description: "The body is not application/json (code unsupported_media_type)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "502":
     *         description: "Google Sheets failed (code upstream_failure)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     *       "503":
     *         description: "Google Sheets is busy (code service_busy)."
     *         content:
     *           application/problem+json:
     *             schema: { $ref: '#/components/schemas/Problem' }
     */
    router.route("/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin")
        .put(auth.requireOperator, requireJson(), asyncHandler(async (req, res) => {
            const drawId = parseDrawIdParam(req.params.drawId);
            const spin = parseSpinParam(req.params.spin);
            const input = parseSpinRequest(req.body, spin);
            const result = await drawService.recordSpin({
                day: req.params.day, facilityName: req.params.facility, drawId, spin, input, actor: req.actor,
            });
            res.status(200).json(result);
        }))
        .all(methodNotAllowed);

    return router;
}
```

### 7.5 The parts the scripts append

**Set-up.** Save these two files, and both scripts of §7.6, into one folder **outside** the repo, such as
`/c/temp/prize-draw-edits/`. The second script reads both files from its own folder.

`errors-tail.part.mjs`, the two new errors, appended to `src/shared/errors.mjs`:

```js
// The DRAWS tab's header row isn't the one the API writes, or rows sit under an empty
// header. 409, like attendance_layout: the workbook is in a state the API won't touch.
export class DrawsLayoutError extends AppError {
    constructor() {
        super(
            "The DRAWS tab's header row is not drawId, title, spin, place, prize, ticket, winner, result, drawnAt, seed, fingerprint, rule, entries, list, continues — fix or rename the tab",
            409
        );
        this.code = "draws_layout";
    }
}

// A spin that doesn't fit the draw already recorded: out of order, with another seed,
// fingerprint, rule or entry count than its spin 1, or other winners than a spin already
// recorded. Nothing was written.
export class DrawConflictError extends AppError {
    constructor(message) {
        super(message, 409);
        this.code = "draw_conflict";
    }
}
```

`sheetsclient-draws.test.part.mjs`, appended to `test/unit/clients/SheetsClient.test.mjs`:

```js
// ---- Prize draws: the DRAWS tab (prize-draw-spec §7.4) ----------------------------------------
const DRAW_ROW = ["K7Q4M2XA", "Door prizes", 1, 1, "Paddle", "", "=Ana", "won", "2026-11-22T07:00:00.000Z",
    "HXTE7Q4M", "f".repeat(64), "remove-winners", 2, "=Ana\nBen", ""];

describe("SheetsClient.createDrawsTab", () => {
    it("adds the tab with a frozen row, then writes the header as RAW", async () => {
        reply(json({ replies: [{}] }), json({}));
        await client().createDrawsTab("S");
        assert.equal(calls.length, 2);
        assert.equal(calls[0].url, "https://sheets.googleapis.com/v4/spreadsheets/S:batchUpdate");
        assert.deepEqual(JSON.parse(calls[0].init.body), {
            requests: [{ addSheet: { properties: { title: "DRAWS", gridProperties: { frozenRowCount: 1 } } } }],
        });
        assert.equal(calls[0].init.headers.Authorization, "Bearer sa-token");
        assert.deepEqual(JSON.parse(calls[1].init.body), { valueInputOption: "RAW", data: [{ range: "DRAWS!A1:O1", values: [DRAWS_HEADERS] }] });
    });

    it("carries on to the header when the tab is already there", async () => {
        reply(json({ error: { code: 400, message: 'A sheet with the name "DRAWS" already exists. Please enter another name.' } }, 400), json({}));
        await client().createDrawsTab("S");
        assert.equal(calls.length, 2);
    });

    it("fails on any other 400", async () => {
        reply(json({ error: { code: 400, message: "Invalid requests[0].addSheet" } }, 400));
        await assert.rejects(client().createDrawsTab("S"), /Invalid requests/);
        assert.equal(calls.length, 1);
    });
});

describe("SheetsClient.writeDrawRows", () => {
    it("writes each whole row, A to O, in one request, as RAW, so a leading = stays text", async () => {
        reply(json({}));
        await client().writeDrawRows("S", [{ row: 7, values: DRAW_ROW }, { row: 8, values: DRAW_ROW }]);
        assert.equal(calls.length, 1);
        assert.equal(calls[0].url, "https://sheets.googleapis.com/v4/spreadsheets/S/values:batchUpdate");
        assert.equal(calls[0].init.headers.Authorization, "Bearer sa-token");
        assert.deepEqual(JSON.parse(calls[0].init.body), { valueInputOption: "RAW", data: [
            { range: "DRAWS!A7:O7", values: [DRAW_ROW] }, { range: "DRAWS!A8:O8", values: [DRAW_ROW] },
        ] });
    });

    for (const row of [1, 0, -1, 2.5, "7", null]) {
        it(`throws before any request for row ${String(row)}`, async () => {
            reply(json({}));
            await assert.rejects(client().writeDrawRows("S", [{ row: 2, values: DRAW_ROW }, { row, values: DRAW_ROW }]));
            assert.equal(calls.length, 0);
        });
    }

    it("throws before any request for a row that is not 15 cells, or for no rows", async () => {
        reply(json({}));
        await assert.rejects(client().writeDrawRows("S", [{ row: 2, values: DRAW_ROW.slice(1) }]), /a DRAWS row is 15 cells/);
        await assert.rejects(client().writeDrawRows("S", []), /no DRAWS rows/);
        assert.equal(calls.length, 0);
    });

    it("still refuses DRAWS through updateValues", async () => {
        reply(json({}));
        await assert.rejects(client().updateValues("S", [{ range: "DRAWS!A2:O2", values: [DRAW_ROW] }]), /write outside the allowlist/);
        assert.equal(calls.length, 0);
    });
});
```

### 7.6 Edits to existing API files

Don't edit these files by hand: **run the two scripts** from the `sage-tools-api` root, in order. Each
finds every anchor exactly once or stops with an error, and each keeps CRLF.

`edit-api2.mjs` makes these changes:
- **`src/clients/SheetsClient.mjs`:**
  - adds `writeDrawRows` and `createDrawsTab`;
  - folds `createAttendanceTab`'s `addSheet` into a shared private `#addTab`, with the same behaviour.
- **`src/shared/ports.mjs`:** the `DrawSheets` port.
- **`src/app.mjs`:** constructs `DrawService` and mounts `drawV3Routes`.
- **`src/docs/openapiSpec.mjs`:** the `draws` tag, and the routes file.
- **The test helpers and guards:** `test/helpers/routeManifest.mjs`, `test/helpers/coveredElsewhere.mjs`,
  `test/unit/guards/rest-conventions.test.mjs` (the M2 exception and the route counts) and
  `test/unit/guards/dependency-rules.test.mjs` (the `draws` feature).

```js
// Phase 2's anchored edits to existing sage-tools-api files (prize-draw-spec §7.6). Run from the repo root.
import fs from 'node:fs';
function edit(p, pairs){
  let t = fs.readFileSync(p, 'utf8'); const crlf = t.includes('\r\n'); t = t.replace(/\r\n/g, '\n');
  for (const [a, b] of pairs){
    const n = t.split(a).length - 1;
    if (n !== 1) throw new Error(`${p}: anchor found ${n} times: ${a.slice(0, 80)}`);
    t = t.replace(a, () => b);
  }
  fs.writeFileSync(p, crlf ? t.replace(/\n/g, '\r\n') : t);
}

edit('src/clients/SheetsClient.mjs', [
  ['import { isCourtCell } from "../courts/domain/courtControlGrid.mjs";\n',
   'import { isCourtCell } from "../courts/domain/courtControlGrid.mjs";\n' +
   'import { DRAWS_TAB, HEADERS as DRAWS_HEADERS, HEADER_RANGE as DRAWS_HEADER_RANGE, isDrawRowRange, rowRange as drawRowRange } from "../draws/domain/drawsTab.mjs";\n'],
  ["// match's two SCHEDULE score cells (and their note), or one Court Control match\n// cell in column C, each through its own allowlist.\n",
   "// match's two SCHEDULE score cells (and their note), one Court Control match\n// cell in column C, or whole DRAWS rows (A to O, row 2 on), each through its own allowlist.\n"],
  ['    // addSheet, the header row, a frozen row 1, and the "Not yet in" list in\n    // column J. The only tab this client ever creates is ATTENDANCE.\n',
`    // A spin's winners, one whole DRAWS row each (A to O, row 2 or below), in one request, as
    // typed (RAW, so a name starting "=" stays text). rows: [{ row, values }].
    // See sage-docs/docs/specs/.../prize-draw-spec.md §7.4.
    async writeDrawRows(sheetId, rows) {
        if (!Array.isArray(rows) || rows.length === 0) throw new Error("SheetsClient: no DRAWS rows to write");
        const data = rows.map(({ row, values }) => {
            const range = drawRowRange(row);
            if (!isDrawRowRange(range)) throw new Error(\`SheetsClient: write outside the draws allowlist: \${range}\`);
            if (!Array.isArray(values) || values.length !== DRAWS_HEADERS.length) {
                throw new Error(\`SheetsClient: a DRAWS row is \${DRAWS_HEADERS.length} cells\`);
            }
            return { range, values: [values] };
        });
        return this.#batchUpdateValues(sheetId, data, "RAW");
    }

    // addSheet DRAWS with a frozen row 1 (or nothing, when it is there already), then its
    // header row. One of the two tabs this client ever creates.
    async createDrawsTab(sheetId) {
        await this.#addTab(sheetId, DRAWS_TAB);
        await this.#batchUpdateValues(sheetId, [{ range: DRAWS_HEADER_RANGE, values: [DRAWS_HEADERS] }], "RAW");
    }

    // addSheet with a frozen row 1. A tab of that name already there (another instance made it
    // a moment ago, or a person did) is not an error: the header write that follows is identical.
    async #addTab(sheetId, title) {
        const url = \`\${BASE}/\${encodeURIComponent(sheetId)}:batchUpdate\`;
        const res = await this.#send(url, {
            method: "POST",
            body: JSON.stringify({
                requests: [{ addSheet: { properties: { title, gridProperties: { frozenRowCount: 1 } } } }],
            }),
        }, { write: true });
        if (res.status === 400) {
            const message = await errorMessage(res.clone());
            if (!/already exists/i.test(message)) throw new UpstreamError(\`Google Sheets: HTTP 400 \${message}\`);
        } else {
            await this.#assertOk(res, { write: true });
        }
    }

    // addSheet, the header row, a frozen row 1, and the "Not yet in" list in
    // column J. One of the two tabs this client ever creates (DRAWS is the other).
`],
  [`    async createAttendanceTab(sheetId) {
        const url = \`\${BASE}/\${encodeURIComponent(sheetId)}:batchUpdate\`;
        const res = await this.#send(url, {
            method: "POST",
            body: JSON.stringify({
                requests: [{ addSheet: { properties: { title: "ATTENDANCE", gridProperties: { frozenRowCount: 1 } } } }],
            }),
        }, { write: true });
        if (res.status === 400) {
            // Another instance created it a moment ago: the header write below is identical.
            const message = await errorMessage(res.clone());
            if (!/already exists/i.test(message)) throw new UpstreamError(\`Google Sheets: HTTP 400 \${message}\`);
        } else {
            await this.#assertOk(res, { write: true });
        }
        await this.updateValues`,
`    async createAttendanceTab(sheetId) {
        await this.#addTab(sheetId, "ATTENDANCE");
        await this.updateValues`],
]);

edit('src/shared/ports.mjs', [
  [' * @property {(...args: any[]) => Promise<any>} clearCourtMatch\n *\n',
   ' * @property {(...args: any[]) => Promise<any>} clearCourtMatch\n *\n' +
   ' * The part of the Sheets client prize draws use.\n * @typedef {object} DrawSheets\n' +
   ' * @property {(...args: any[]) => Promise<any>} readValues\n * @property {(...args: any[]) => Promise<any>} createDrawsTab\n' +
   ' * @property {(...args: any[]) => Promise<any>} writeDrawRows\n *\n'],
]);

edit('src/app.mjs', [
  ['import { CourtService } from "./courts/CourtService.mjs";\n',
   'import { CourtService } from "./courts/CourtService.mjs";\nimport { DrawService } from "./draws/DrawService.mjs";\n'],
  ['import { courtV3Routes } from "./courts/routes.mjs";\n',
   'import { courtV3Routes } from "./courts/routes.mjs";\nimport { drawV3Routes } from "./draws/routes.mjs";\n'],
  ['        syncService,\n        logger: courtsLogger,\n    });\n',
`        syncService,
        logger: courtsLogger,
    });

    // Prize draws write each spin's winners as rows of a facility workbook's DRAWS tab, as the same
    // service account. Nothing is published. See sage-docs/docs/specs/.../prize-draw-spec.md.
    const drawsLogger = logger.child("draws");
    const drawService = new DrawService({
        configStore: syncConfigStore,
        sheets: new SheetsClient({ auth: sheetsReadAuth, accessToken: googleAccessToken, timeoutMs: 60000, logger: drawsLogger }),
        logger: drawsLogger,
    });
`],
  ['        services: { syncController, authService, attendanceService, scoreService, courtService },',
   '        services: { syncController, authService, attendanceService, scoreService, courtService, drawService },'],
  ['        server, syncService, syncConfigStore, authService, attendanceService, scoreService, courtService, githubPublisher, livePublisher,',
   '        server, syncService, syncConfigStore, authService, attendanceService, scoreService, courtService, drawService, githubPublisher, livePublisher,'],
  ['        { path: "/v3", router: courtV3Routes({ courtService: services.courtService, auth }) },\n',
   '        { path: "/v3", router: courtV3Routes({ courtService: services.courtService, auth }) },\n        { path: "/v3", router: drawV3Routes({ drawService: services.drawService, auth }) },\n'],
  [' *   /v3 and have no twin (court control).', ' *   /v3 and have no twin (court control, prize draws).'],
]);

edit('src/docs/openapiSpec.mjs', [
  ['                { name: "courts" },\n', '                { name: "courts" },\n                { name: "draws" },\n'],
  ['            path.join(ROOT, "src/courts/routes.mjs"),\n            path.join(ROOT, "src/server/Server.mjs"),',
   '            path.join(ROOT, "src/courts/routes.mjs"),\n            path.join(ROOT, "src/draws/routes.mjs"),\n            path.join(ROOT, "src/server/Server.mjs"),'],
]);

edit('test/helpers/routeManifest.mjs', [
  ['    { route: "PUT /v3/days/:day/facilities/:facility/courts/:court/match", test: "test/integration/court-routes.test.mjs" },\n',
   '    { route: "PUT /v3/days/:day/facilities/:facility/courts/:court/match", test: "test/integration/court-routes.test.mjs" },\n' +
   '    { route: "GET /v3/days/:day/facilities/:facility/prize-draws", test: "test/integration/draw-routes.test.mjs" },\n' +
   '    { route: "PUT /v3/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin", test: "test/integration/draw-routes.test.mjs" },\n'],
]);

edit('test/helpers/coveredElsewhere.mjs', [
  ['    "src/courts/routes.mjs":               ["test/integration/court-routes.test.mjs", "route handlers"],\n',
   '    "src/courts/routes.mjs":               ["test/integration/court-routes.test.mjs", "route handlers"],\n' +
   '    "src/draws/routes.mjs":                ["test/integration/draw-routes.test.mjs", "route handlers"],\n'],
]);

edit('test/unit/guards/rest-conventions.test.mjs', [
  [`        reason: "the live court is read from the published day snapshot, which pages already hold; a Sheets read per GET would spend Google quota for no reader",
    },
`, `        reason: "the live court is read from the published day snapshot, which pages already hold; a Sheets read per GET would spend Google quota for no reader",
    },
    "PUT /v3/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin": {
        rule: "M2",
        reason: "a draw's spins are read with GET …/facilities/{facility}/prize-draws, the list the page reads; a per-spin GET would have no reader",
    },
`],
  ['        assert.equal(v3Routes.length, 18);', '        assert.equal(v3Routes.length, 20);'],
  ['    it("names the eighteen operationIds the spec lists", async () => {', '    it("names the twenty operationIds the spec lists", async () => {'],
  ['            "getScoreEntry", "getSyncDiagnostics", "listFacilityPeople", "runDaySync",',
   '            "getScoreEntry", "getSyncDiagnostics", "listFacilityPeople", "listPrizeDraws", "recordPrizeDrawSpin", "runDaySync",'],
  ['        assert.equal(seen, 18);', '        assert.equal(seen, 20);'],
]);

edit('test/unit/guards/dependency-rules.test.mjs', [
  ['(sync|attendance|scores|courts|scoresheets)', '(sync|attendance|scores|courts|draws|scoresheets)'],
  ['a feature folder (sync, attendance, scores, courts, scoresheets)', 'a feature folder (sync, attendance, scores, courts, draws, scoresheets)'],
]);
console.log('edits ok');
```

`edit-api3.mjs` makes these changes:
- appends the errors to `src/shared/errors.mjs`;
- updates the tests that pin the API's shape. They are client-visible, so they change with it:
  - `v3.test.mjs` and `routes.test.mjs`: 18 → 20 `/v3` routes, 15 → 17 paths, 33 → 35 routes in all;
  - `openapi.test.mjs`: 32 → 34 probes;
  - `app.test.mjs`: a seventh `/v3` router;
  - `openapiSpec.test.mjs`: the `draws` tag, and the two paths;
- adds the Sheets client tests;
- sets `package.json` to 3.9.0.

```js
// Phase 2's edits to the tests that pin the API's shape (prize-draw-spec §7.6), the error classes and the
// version. Run from the repo root, after edit-api2.mjs. Reads errors-tail.part.mjs and
// sheetsclient-draws.test.part.mjs from the folder this script is in.
import fs from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';
const HERE = path.dirname(fileURLToPath(import.meta.url));
function edit(p, pairs){
  let t = fs.readFileSync(p, 'utf8'); const crlf = t.includes('\r\n'); t = t.replace(/\r\n/g, '\n');
  for (const [a, b] of pairs){
    const n = t.split(a).length - 1;
    if (n !== 1) throw new Error(`${p}: anchor found ${n} times: ${a.slice(0, 80)}`);
    t = t.replace(a, () => b);
  }
  fs.writeFileSync(p, crlf ? t.replace(/\n/g, '\r\n') : t);
}
function append(p, file){
  let t = fs.readFileSync(p, 'utf8'); const crlf = t.includes('\r\n'); t = t.replace(/\r\n/g, '\n').replace(/\n*$/, '\n');
  t += '\n' + fs.readFileSync(path.join(HERE, file), 'utf8').replace(/\r\n/g, '\n').replace(/^\n+/, '');
  fs.writeFileSync(p, crlf ? t.replace(/\n/g, '\r\n') : t);
}

append('src/shared/errors.mjs', 'errors-tail.part.mjs');

const COURT = '    "PUT /v3/days/:day/facilities/:facility/courts/:court/match",\n';
const DRAWS = '    "GET /v3/days/:day/facilities/:facility/prize-draws",\n    "PUT /v3/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin",\n';
edit('test/integration/v3.test.mjs', [
  ['// The /v3 API (architecture spec §4.11, Phase 5): the eighteen routes, sixteen against their frozen\n// twin (the attendance list and the court match, added in 3.8.0, have none),',
   '// The /v3 API (architecture spec §4.11, Phase 5): the twenty routes, sixteen against their frozen\n// twin (the attendance list, the court match, added in 3.8.0, and the two prize-draw routes, added in 3.9.0, have none),'],
  [COURT, COURT + DRAWS],
  ['    it("registers exactly the eighteen routes, on fifteen paths", async () => {', '    it("registers exactly the twenty routes, on seventeen paths", async () => {'],
  ['            assert.equal(new Set(v3.map(r => r.split(" ")[1])).size, 15);', '            assert.equal(new Set(v3.map(r => r.split(" ")[1])).size, 17);'],
]);
edit('test/unit/helpers/routes.test.mjs', [
  ['    it("finds the thirty-three routes registered on the real app today: fifteen frozen, eighteen on /v3", async () => {',
   '    it("finds the thirty-five routes registered on the real app today: fifteen frozen, twenty on /v3", async () => {'],
  ['            "PUT /v3/days/:day/facilities/:facility/courts/:court/match",\n        ].sort());',
   '            "PUT /v3/days/:day/facilities/:facility/courts/:court/match",\n            "GET /v3/days/:day/facilities/:facility/prize-draws",\n            "PUT /v3/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin",\n        ].sort());'],
  ['        assert.equal(registeredRoutes(started.server.app).length, 33);', '        assert.equal(registeredRoutes(started.server.app).length, 35);'],
]);
edit('test/integration/openapi.test.mjs', [
  ['        assert.equal(probed, 32);   // 14 frozen paths and 15 /v3 paths, three of them with two methods',
   '        assert.equal(probed, 34);   // 14 frozen paths and 17 /v3 paths, three of them with two methods'],
]);
edit('test/unit/app.test.mjs', [
  ['    it("returns the routers in mount order: the frozen scoresheets, sync, auth and /v1 (attendance, scores), then the six /v3 surfaces", () => {',
   '    it("returns the routers in mount order: the frozen scoresheets, sync, auth and /v1 (attendance, scores), then the seven /v3 surfaces", () => {'],
  ['attendanceService: {}, scoreService: {}, courtService: {} },', 'attendanceService: {}, scoreService: {}, courtService: {}, drawService: {} },'],
  ['["/scoresheets", "/sync", "/auth", "/v1", "/v1", "/v3", "/v3", "/v3", "/v3", "/v3", "/v3"]',
   '["/scoresheets", "/sync", "/auth", "/v1", "/v1", "/v3", "/v3", "/v3", "/v3", "/v3", "/v3", "/v3"]'],
]);
edit('test/unit/docs/openapiSpec.test.mjs', [
  ['    it("has the seven tags", async () => {', '    it("has the eight tags", async () => {'],
  ['["health", "scoresheets", "sync", "auth", "attendance", "scores", "courts"]', '["health", "scoresheets", "sync", "auth", "attendance", "scores", "courts", "draws"]'],
  ['    it("documents exactly these 29 paths: the 14 frozen ones and the 15 /v3 ones", async () => {',
   '    it("documents exactly these 31 paths: the 14 frozen ones and the 17 /v3 ones", async () => {'],
  ['            "/v3/days/{day}/facilities/{facility}/people/{personKey}/attendance",\n',
   '            "/v3/days/{day}/facilities/{facility}/people/{personKey}/attendance",\n' +
   '            "/v3/days/{day}/facilities/{facility}/prize-draws",\n' +
   '            "/v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}",\n'],
]);
const imp = 'import { HEADERS } from "../../../src/attendance/domain/attendanceTab.mjs";\n';
edit('test/unit/clients/SheetsClient.test.mjs', [[imp, imp + 'import { HEADERS as DRAWS_HEADERS } from "../../../src/draws/domain/drawsTab.mjs";\n']]);
append('test/unit/clients/SheetsClient.test.mjs', 'sheetsclient-draws.test.part.mjs');
edit('package.json', [['"version": "3.8.0"', '"version": "3.9.0"']]);
console.log('test edits ok');
```

```bash
node /c/temp/prize-draw-edits/edit-api2.mjs
node /c/temp/prize-draw-edits/edit-api3.mjs
```

Each prints `edits ok` / `test edits ok`. Then refresh the OpenAPI snapshot, which pins the whole generated
document:

```bash
node --test --test-update-snapshots test/unit/docs/openapiSpec.test.mjs
git diff --numstat test/unit/docs/openapiSpec.test.mjs.snapshot
```

The diff must only **add** lines (the dry-run added 451 and removed 0). A removed line means an existing
route's documentation changed: **stop and ask**.

### 7.7 New tests

`test/unit/draws/domain/drawsTab.test.mjs`:

```js
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import {
    DRAWS_TAB, HEADERS, READ_RANGE, HEADER_RANGE, rowRange, isDrawRowRange, readDrawsTab, drawRows, pickRowValues, groupDraws,
} from "../../../../src/draws/domain/drawsTab.mjs";
import { DrawsLayoutError } from "../../../../src/shared/errors.mjs";

const pick = (drawId, spin, place, winner, extra = {}) => pickRowValues({
    drawId, title: "Door prizes", spin, place, prize: "Paddle", ticket: "", winner, result: "won", drawnAt: "2026-11-22T07:00:00.000Z",
    seed: "HXTE7Q4M", fingerprint: "f".repeat(64), rule: "remove-winners", entries: 3,
    list: ["Ana Reyes", "Ben Cruz", "Carla Santos"], continues: "", ...extra,
});

describe("the DRAWS tab's shape", () => {
    it("has fifteen columns, A to O", () => {
        assert.equal(DRAWS_TAB, "DRAWS");
        assert.equal(HEADERS.length, 15);
        assert.equal(READ_RANGE, "DRAWS!A:O");
        assert.equal(HEADER_RANGE, "DRAWS!A1:O1");
    });

    it("writes one whole data row at a time", () => {
        assert.equal(rowRange(2), "DRAWS!A2:O2");
        assert.throws(() => rowRange(1));
        assert.throws(() => rowRange(2.5));
        for (const r of ["DRAWS!A2:O2", "DRAWS!A140:O140"]) assert.equal(isDrawRowRange(r), true, r);
        for (const r of ["DRAWS!A1:O1", "DRAWS!A2:O3", "DRAWS!A2:N2", "DRAWS!B2:O2", "ATTENDANCE!A2:O2", "DRAWS!A0:O0", "'DRAWS'!A2:O2"]) {
            assert.equal(isDrawRowRange(r), false, r);
        }
    });

    it("writes the list and continues on spin 1, place 1 only", () => {
        const first = pick("K7Q4M2XA", 1, 1, "Ana Reyes", { continues: "ZZZZ2222" });
        assert.deepEqual([first[13], first[14]], ["Ana Reyes\nBen Cruz\nCarla Santos", "ZZZZ2222"]);
        const second = pick("K7Q4M2XA", 1, 2, "Ben Cruz", { continues: "ZZZZ2222" });
        assert.deepEqual([second[13], second[14]], ["", ""]);
        assert.equal(pick("K7Q4M2XA", 2, 1, "Ben Cruz")[13], "");
        assert.equal(second.length, 15);
    });
});

describe("readDrawsTab", () => {
    it("reads no tab, and an empty tab, as nothing recorded yet", () => {
        assert.deepEqual(readDrawsTab(null), { exists: false, rows: [], nextRow: 2 });
        assert.deepEqual(readDrawsTab([]), { exists: true, needsHeader: true, rows: [], nextRow: 2 });
    });

    it("refuses a tab whose header row is not HEADERS, or rows under an empty header", () => {
        const wrong = [HEADERS.map((h, i) => (i === 6 ? "name" : h))];
        assert.throws(() => readDrawsTab(wrong), DrawsLayoutError);
        assert.throws(() => readDrawsTab([[], ["x"]]), DrawsLayoutError);
    });

    it("reads each winner's row, and puts the next one after the last non-blank row", () => {
        const tab = readDrawsTab([HEADERS, pick("K7Q4M2XA", 1, 1, "Ana Reyes"), [], pick("K7Q4M2XA", 2, 1, "Ben Cruz")]);
        assert.equal(tab.exists, true);
        assert.equal(tab.nextRow, 5);
        assert.deepEqual(tab.rows.map(r => [r.row, r.spin, r.place, r.winner]), [[2, 1, 1, "Ana Reyes"], [4, 2, 1, "Ben Cruz"]]);
        assert.equal(tab.rows[0].entries, 3);
        assert.equal(tab.rows[0].list, "Ana Reyes\nBen Cruz\nCarla Santos");
    });
});

describe("groupDraws", () => {
    it("lists each draw once, in first-row order, its winners in spin and place order, its list and continues from spin 1, place 1", () => {
        const tab = readDrawsTab([
            HEADERS,
            pick("K7Q4M2XA", 1, 2, "Ben Cruz"),
            pick("K7Q4M2XA", 1, 1, "Ana Reyes", { continues: "ZZZZ2222" }),
            pick("ZZZZ2222", 1, 1, "Ben Cruz", { title: "Raffle", rule: "keep-winners" }),
            pick("K7Q4M2XA", 2, 1, "Carla Santos", { prize: "Cap", result: "redrawn" }),
        ]);
        const draws = groupDraws(tab);
        assert.deepEqual(draws.map(d => d.drawId), ["K7Q4M2XA", "ZZZZ2222"]);
        assert.deepEqual(draws[0].picks.map(p => [p.spin, p.place, p.winner, p.result]),
            [[1, 1, "Ana Reyes", "won"], [1, 2, "Ben Cruz", "won"], [2, 1, "Carla Santos", "redrawn"]]);
        assert.deepEqual(draws[0].list, ["Ana Reyes", "Ben Cruz", "Carla Santos"]);
        assert.equal(draws[0].continues, "ZZZZ2222");
        assert.equal(draws[1].rule, "keep-winners");
        assert.deepEqual(drawRows(tab, "K7Q4M2XA").map(r => [r.spin, r.place]), [[1, 1], [1, 2], [2, 1]]);
    });

    it("gives a draw whose spin 1, place 1 row was cut by hand an empty list", () => {
        const draws = groupDraws(readDrawsTab([HEADERS, pick("K7Q4M2XA", 2, 1, "Ben Cruz")]));
        assert.deepEqual([draws[0].list, draws[0].continues], [[], ""]);
    });
});
```

`test/unit/draws/domain/drawRequest.test.mjs`:

```js
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { parseDrawIdParam, parseSpinParam, parseSpinRequest, MAX_ENTRIES } from "../../../../src/draws/domain/drawRequest.mjs";
import { ValidationError } from "../../../../src/shared/errors.mjs";

const bad = (fn, message) => assert.throws(fn, err => err instanceof ValidationError && err.message === message);
const LIST = ["Ana Reyes", "Ben Cruz", "Carla Santos"];
const WON = (winner, place = 1, o = {}) => ({ place, prize: "Paddle", ticket: "", winner, result: "won", ...o });
const body = (o = {}) => ({ title: "Door prizes", seed: "HXTE7Q4M", fingerprint: "a".repeat(64), rule: "remove-winners",
    entries: 3, list: LIST, winners: [WON("Ana Reyes")], ...o });
const ID_MSG = "must be 8 characters from 2-9 and A-Z without I, L, O or U";

describe("parseDrawIdParam and parseSpinParam", () => {
    it("reads an 8-character draw id from the seed alphabet", () => assert.equal(parseDrawIdParam("K7Q4M2XA"), "K7Q4M2XA"));
    for (const id of ["K7Q4M2X", "K7Q4M2XAB", "k7q4m2xa", "K7Q4M2X0", "K7Q4M2XI", "K7Q4M2XL", "K7Q4M2XO", "K7Q4M2XU", undefined]) {
        it(`refuses draw id ${JSON.stringify(id)}`, () => bad(() => parseDrawIdParam(id), `drawId ${ID_MSG}`));
    }
    it("reads a spin from 1 to 9999", () => {
        assert.equal(parseSpinParam("1"), 1);
        assert.equal(parseSpinParam("9999"), 9999);
    });
    for (const s of ["0", "01", "10000", "-1", "1.5", "a", "", undefined]) {
        it(`refuses spin ${JSON.stringify(s)}`, () => bad(() => parseSpinParam(s), "spin must be a spin number, such as 3"));
    }
});

describe("parseSpinRequest", () => {
    it("reads spin 1 with its list, its winners and the draw it continues", () => {
        assert.deepEqual(parseSpinRequest(body({ continues: "ZZZZ2222", winners: [WON("Ana Reyes"), WON("Ben Cruz", 2, { ticket: "002" })] }), 1), {
            title: "Door prizes", seed: "HXTE7Q4M", fingerprint: "a".repeat(64), rule: "remove-winners", entries: 3, list: LIST,
            continues: "ZZZZ2222",
            winners: [{ place: 1, prize: "Paddle", ticket: "", winner: "Ana Reyes", result: "won" },
                      { place: 2, prize: "Paddle", ticket: "002", winner: "Ben Cruz", result: "won" }],
        });
    });

    it("ignores a later spin's list and continues, and takes an absent title, prize or ticket as empty", () => {
        const out = parseSpinRequest(body({ list: undefined, continues: "ZZZZ2222", title: undefined,
            winners: [{ place: 1, winner: "Ben Cruz", result: "redrawn" }] }), 2);
        assert.deepEqual([out.list, out.continues, out.title, out.winners[0].prize, out.winners[0].ticket, out.winners[0].result],
            [[], "", "", "", "", "redrawn"]);
    });

    const cases = [
        [null, "Body must be a JSON object"],
        [[], "Body must be a JSON object"],
        [body({ seed: "kite7q4m" }), `seed ${ID_MSG}`],
        [body({ seed: "MANGO" }), `seed ${ID_MSG}`],
        [body({ fingerprint: "A".repeat(64) }), "fingerprint must be 64 lowercase hex characters"],
        [body({ entries: 1 }), `entries must be a whole number from 2 to ${MAX_ENTRIES}`],
        [body({ entries: MAX_ENTRIES + 1 }), `entries must be a whole number from 2 to ${MAX_ENTRIES}`],
        [body({ winners: [] }), "winners must list the spin's 1 to 50 winners"],
        [body({ winners: [null] }), "each winner must be an object"],
        [body({ winners: [WON("Ana Reyes", 2)] }), "winners must be in place order: 1, 2, 3 …"],
        [body({ winners: [WON("Ana Reyes", 1, { ticket: "12345" })] }), "ticket must be up to 4 digits, or empty"],
        [body({ winners: [WON("Ana Reyes", 1, { prize: "x".repeat(81) })] }), "prize is at most 80 characters"],
        [body({ winners: [WON(" ")] }), "winner is required"],
        [body({ winners: [WON("Ana Reyes", 1, { result: "absent" })] }), 'result must be "won" or "redrawn"'],
        [body({ winners: [WON("Ana Reyes"), WON("Ana Reyes", 2)] }), "a spin can't draw the same entry twice"],
        [body({ title: "x".repeat(81) }), "title is at most 80 characters"],
        [body({ rule: "once" }), 'rule must be "remove-winners" or "keep-winners"'],
        [body({ list: ["Ana Reyes", "Ben Cruz"] }), "list must be the draw's entries, one string each, as many as entries"],
        [body({ list: ["Ana Reyes", " Ben Cruz", "Carla Santos"] }), 'each list entry is 1 to 40 characters, trimmed, with no line break or "|"'],
        [body({ list: ["Ana Reyes", "Ben|Cruz", "Carla Santos"] }), 'each list entry is 1 to 40 characters, trimmed, with no line break or "|"'],
        [body({ list: ["Ana Reyes", "Ana Reyes", "Carla Santos"] }), "list has an entry twice"],
        [body({ winners: [WON("Nobody")] }), "winner is not in the list"],
        [body({ continues: "nope" }), `continues ${ID_MSG}, or empty`],
    ];
    for (const [b, message] of cases) {
        it(`refuses: ${message}`, () => bad(() => parseSpinRequest(b, 1), message));
    }
});
```

`test/unit/draws/DrawService.test.mjs`:

```js
import { describe, it, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { createHash } from "node:crypto";
import { DrawService } from "../../../src/draws/DrawService.mjs";
import { HEADERS, readDrawsTab } from "../../../src/draws/domain/drawsTab.mjs";
import { SyncConfigSnapshot } from "../../../src/registry/SyncConfigSnapshot.mjs";
import {
    DrawConflictError, DrawsLayoutError, ForbiddenError, UnknownFacilityError, UnknownSyncDayError, ValidationError,
} from "../../../src/shared/errors.mjs";
import { capturingLogger } from "../../helpers/logger.mjs";

const fac = (name, sheetId) => ({ name, sheetId });
const RAW = {
    version: 1,
    events: {
        ev: { type: "standard", days: { "ev-day1": { label: "Day 1", date: "2026-11-22", facilities: [fac("Alpha", "SA"), fac("Beta", "SB")] } } },
        done: { type: "standard", archived: true, days: { "done-day1": { label: "Day 1", date: "2026-10-03", facilities: [fac("Alpha", "DA")] } } },
    },
};
const snapshot = new SyncConfigSnapshot(RAW, { sha: "abc", source: "remote", loadedAt: new Date().toISOString() });
const NOW = new Date("2026-11-22T07:05:00.000Z");

// The three calls DrawService makes, over in-memory tabs (no entry = no DRAWS tab).
class DrawSheetsFake {
    constructor() { this.tabs = {}; this.created = []; this.writes = []; }
    async readValues(sheetId, range) {
        assert.equal(range, "DRAWS!A:O");
        const t = this.tabs[sheetId];
        return t === undefined ? null : structuredClone(t);
    }
    async createDrawsTab(sheetId) { this.created.push(sheetId); this.tabs[sheetId] = [HEADERS.slice()]; }
    async writeDrawRows(sheetId, rows) {
        this.writes.push({ sheetId, rows: structuredClone(rows) });
        const t = this.tabs[sheetId];
        for (const { row, values } of rows) {
            while (t.length < row) t.push([]);
            t[row - 1] = values.slice();
        }
    }
}

const LIST = ["Ana Reyes", "Ben Cruz", "Carla Santos", "Dante Lim"];
const FP = createHash("sha256").update(LIST.join("\n"), "utf8").digest("hex");
const W = (winner, place = 1, o = {}) => ({ place, prize: "Paddle", ticket: "", winner, result: "won", ...o });
const spin1 = (o = {}) => ({ title: "Door prizes", seed: "HXTE7Q4M", fingerprint: FP, rule: "remove-winners", entries: 4, list: LIST,
    continues: "", winners: [W("Ana Reyes")], ...o });
const later = o => spin1({ list: [], continues: "", ...o });

let sheets, service, logger;
beforeEach(() => {
    sheets = new DrawSheetsFake();
    logger = capturingLogger();
    service = new DrawService({ configStore: { get: async () => snapshot }, sheets, logger, now: () => NOW });
});
const record = (spin, input, o = {}) => service.recordSpin({ day: "ev-day1", facilityName: "Alpha", drawId: "K7Q4M2XA", spin, input, actor: { kind: "operator" }, ...o });
const tabRows = () => readDrawsTab(sheets.tabs.SA).rows.map(r => [r.row, r.spin, r.place, r.winner, r.result]);

describe("DrawService.recordSpin", () => {
    it("creates the DRAWS tab on the first spin and writes its winner, with the list, to row 2", async () => {
        const res = await record(1, spin1({ continues: "ZZZZ2222" }));
        assert.deepEqual(res, { day: "ev-day1", facility: "Alpha", drawId: "K7Q4M2XA", spin: 1, rows: [2], created: true, drawnAt: NOW.toISOString() });
        assert.deepEqual(sheets.created, ["SA"]);
        assert.deepEqual(sheets.writes[0].rows[0].values, ["K7Q4M2XA", "Door prizes", 1, 1, "Paddle", "", "Ana Reyes", "won", NOW.toISOString(),
            "HXTE7Q4M", FP, "remove-winners", 4, LIST.join("\n"), "ZZZZ2222"]);
    });

    it("writes a spin of three winners as three rows in one call", async () => {
        const res = await record(1, spin1({ winners: [W("Ana Reyes", 1, { prize: "Ball pack (1 of 3)" }), W("Ben Cruz", 2, { prize: "Ball pack (2 of 3)" }), W("Dante Lim", 3, { prize: "Ball pack (3 of 3)" })] }));
        assert.deepEqual(res.rows, [2, 3, 4]);
        assert.equal(sheets.writes.length, 1);
        assert.deepEqual(tabRows(), [[2, 1, 1, "Ana Reyes", "won"], [3, 1, 2, "Ben Cruz", "won"], [4, 1, 3, "Dante Lim", "won"]]);
        assert.equal(sheets.tabs.SA[2][13], "", "the list sits on place 1 only");
    });

    it("adds later spins on the next rows, and logs each", async () => {
        await record(1, spin1());
        await record(2, later({ winners: [W("Ben Cruz", 1, { result: "redrawn" })] }));
        const res = await record(3, later({ winners: [W("Carla Santos")] }));
        assert.deepEqual(res.rows, [4]);
        assert.deepEqual(tabRows(), [[2, 1, 1, "Ana Reyes", "won"], [3, 2, 1, "Ben Cruz", "redrawn"], [4, 3, 1, "Carla Santos", "won"]]);
        assert.ok(logger.lines.some(([, m]) => m === "draw ev-day1/Alpha K7Q4M2XA spin 3 rows 4 by=operator"));
    });

    it("answers a retried spin with its rows, writing nothing", async () => {
        await record(1, spin1());
        const res = await record(1, spin1());
        assert.deepEqual([res.rows, res.created, res.drawnAt], [[2], false, NOW.toISOString()]);
        assert.equal(sheets.writes.length, 1);
    });

    it("refuses other winners for a spin already recorded", async () => {
        await record(1, spin1());
        await assert.rejects(record(1, spin1({ winners: [W("Ben Cruz")] })), DrawConflictError);
        await assert.rejects(record(1, spin1({ winners: [W("Ana Reyes"), W("Ben Cruz", 2)] })), DrawConflictError);
    });

    it("marks a winner of the latest spin drawn again on its own row, keeping its list and time", async () => {
        await record(1, spin1({ winners: [W("Ana Reyes"), W("Ben Cruz", 2)] }));
        const res = await record(1, spin1({ winners: [W("Ana Reyes"), W("Ben Cruz", 2, { result: "redrawn" })] }));
        assert.deepEqual([res.rows, res.created, res.drawnAt], [[3], false, NOW.toISOString()]);
        assert.deepEqual(tabRows(), [[2, 1, 1, "Ana Reyes", "won"], [3, 1, 2, "Ben Cruz", "redrawn"]]);
        await record(1, spin1({ winners: [W("Ana Reyes", 1, { result: "redrawn" }), W("Ben Cruz", 2, { result: "redrawn" })] }));
        assert.equal(sheets.tabs.SA[1][13], LIST.join("\n"));
        assert.equal(sheets.tabs.SA[1][7], "redrawn");
    });

    it("refuses to redraw an earlier spin, or to make a drawn-again winner a winner again", async () => {
        await record(1, spin1());
        await record(2, later({ winners: [W("Ben Cruz", 1, { result: "redrawn" })] }));
        await record(3, later({ winners: [W("Carla Santos")] }));
        await assert.rejects(record(1, spin1({ winners: [W("Ana Reyes", 1, { result: "redrawn" })] })), DrawConflictError);
        await assert.rejects(record(2, later({ winners: [W("Ben Cruz")] })), DrawConflictError);
    });

    it("refuses a spin out of order, or with other settings than spin 1", async () => {
        await assert.rejects(record(2, later({ winners: [W("Ben Cruz")] })), DrawConflictError);
        await record(1, spin1());
        await assert.rejects(record(3, later({ winners: [W("Ben Cruz")] })), DrawConflictError);
        await assert.rejects(record(2, later({ winners: [W("Ben Cruz")], seed: "OTHER222" })), DrawConflictError);
        await assert.rejects(record(2, later({ winners: [W("Ben Cruz")], rule: "keep-winners" })), DrawConflictError);
        assert.equal(sheets.writes.length, 1);
    });

    it("refuses a later winner who is not in the stored list", async () => {
        await record(1, spin1());
        await assert.rejects(record(2, later({ winners: [W("Nobody")] })), ValidationError);
    });

    it("refuses a spin 1 whose fingerprint is not the list's", async () => {
        await assert.rejects(record(1, spin1({ fingerprint: "0".repeat(64) })), ValidationError);
        assert.equal(sheets.created.length, 0);
    });

    it("refuses a DRAWS tab in another shape, an archived event, an unknown day or facility", async () => {
        sheets.tabs.SA = [["name"]];
        await assert.rejects(record(1, spin1()), DrawsLayoutError);
        await assert.rejects(record(1, spin1(), { day: "done-day1" }), ForbiddenError);
        await assert.rejects(record(1, spin1(), { day: "nope" }), UnknownSyncDayError);
        await assert.rejects(record(1, spin1(), { facilityName: "Gamma" }), UnknownFacilityError);
    });

    it("writes a header into an empty DRAWS tab made by hand", async () => {
        sheets.tabs.SA = [];
        await record(1, spin1());
        assert.deepEqual(sheets.created, ["SA"]);
        assert.deepEqual(sheets.writes[0].rows.map(r => r.row), [2]);
    });
});

describe("DrawService.listDraws", () => {
    it("lists nothing for a workbook with no DRAWS tab", async () => {
        assert.deepEqual(await service.listDraws({ day: "ev-day1", facilityName: "Beta" }), { day: "ev-day1", facility: "Beta", draws: [] });
    });

    it("lists the recorded draw with its list and winners, archived events included", async () => {
        await record(1, spin1());
        await record(2, later({ winners: [W("Ben Cruz"), W("Carla Santos", 2)] }));
        const { draws } = await service.listDraws({ day: "ev-day1", facilityName: "Alpha" });
        assert.equal(draws.length, 1);
        assert.deepEqual(draws[0].list, LIST);
        assert.deepEqual(draws[0].picks.map(p => [p.spin, p.place, p.winner]), [[1, 1, "Ana Reyes"], [2, 1, "Ben Cruz"], [2, 2, "Carla Santos"]]);
        sheets.tabs.DA = [HEADERS.slice()];
        assert.deepEqual((await service.listDraws({ day: "done-day1", facilityName: "Alpha" })).draws, []);
    });
});
```

`test/integration/draw-routes.test.mjs`:

```js
import { describe, it, before, after, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { createHash, randomBytes, scryptSync } from "node:crypto";
import { startApp } from "../helpers/http.mjs";
import { AuthService } from "../../src/auth/AuthService.mjs";
import { DrawService } from "../../src/draws/DrawService.mjs";
import { HEADERS } from "../../src/draws/domain/drawsTab.mjs";
import { SyncConfigSnapshot } from "../../src/registry/SyncConfigSnapshot.mjs";
import { silentLogger } from "../helpers/logger.mjs";

// The real Server (real routes, real CORS and auth middleware) on an ephemeral port, with the real
// DrawService over an in-memory DRAWS tab: GET …/prize-draws and PUT …/prize-draws/{drawId}/spins/{spin}.
const salt = randomBytes(16);
const passwordHash = `${salt.toString("hex")}:${scryptSync("operator:pw", salt, 32).toString("hex")}`;
const authService = new AuthService({ passwordHash, tokenSecret: "route-test-secret" }, silentLogger);

const RAW = {
    version: 1,
    events: {
        e: { type: "standard", days: { day1: { label: "Day 1", date: "2099-12-31", facilities: [{ name: "Court 1", sheetId: "S1" }] } } },
        old: { type: "standard", archived: true, days: { "old-day1": { label: "Day 1", date: "2026-01-01", facilities: [{ name: "Court 1", sheetId: "S2" }] } } },
    },
};

class DrawSheetsFake {
    constructor() { this.tabs = {}; this.writes = []; }
    async readValues(sheetId) { const t = this.tabs[sheetId]; return t === undefined ? null : structuredClone(t); }
    async createDrawsTab(sheetId) { this.tabs[sheetId] = [HEADERS.slice()]; }
    async writeDrawRows(sheetId, rows) {
        this.writes.push({ sheetId, rows });
        const t = this.tabs[sheetId];
        for (const { row, values } of rows) {
            while (t.length < row) t.push([]);
            t[row - 1] = values.slice();
        }
    }
}

let app, baseUrl, sheets;
before(async () => {
    sheets = new DrawSheetsFake();
    const snapshot = new SyncConfigSnapshot(RAW, { sha: "abc", source: "remote", loadedAt: new Date().toISOString() });
    const drawService = new DrawService({ configStore: { get: async () => snapshot }, sheets, logger: silentLogger, now: () => new Date("2099-12-31T02:00:00.000Z") });
    app = await startApp({ services: { authService, drawService } });
    baseUrl = app.baseUrl;
});
after(() => app.close());
beforeEach(() => { sheets.tabs = {}; sheets.writes = []; });

const operatorToken = () => authService.login("operator", "pw").token;
const deskToken = () => authService.issueDeskToken({ day: "day1", expiresAt: Date.now() + 60_000 }).token;
const scorerToken = () => authService.issueScorerToken({ day: "day1", expiresAt: Date.now() + 60_000 }).token;

async function request(method, path, { token, body, headers = {}, raw } = {}) {
    const res = await fetch(baseUrl + path, {
        method,
        headers: {
            ...(token ? { Authorization: `Bearer ${token}` } : {}),
            ...(body !== undefined ? { "Content-Type": "application/json" } : {}),
            ...headers,
        },
        body: raw !== undefined ? raw : (body === undefined ? undefined : JSON.stringify(body)),
    });
    const text = await res.text();
    let json = null;
    try { json = JSON.parse(text); } catch { /* not JSON */ }
    return { status: res.status, json, headers: res.headers };
}

const isProblem = (res, status, code) => {
    assert.equal(res.status, status);
    assert.match(res.headers.get("Content-Type"), /^application\/problem\+json/);
    assert.equal(res.json.error, res.json.detail, "error repeats detail");
    if (code) assert.equal(res.json.code, code);
};

const LIST = ["Ana Reyes", "Ben Cruz", "Carla Santos"];
const SPIN1 = { title: "Door prizes", seed: "HXTE7Q4M", fingerprint: createHash("sha256").update(LIST.join("\n"), "utf8").digest("hex"),
    rule: "remove-winners", entries: 3, list: LIST, winners: [{ place: 1, prize: "Paddle", ticket: "001", winner: "Ana Reyes", result: "won" }] };
const LIST_PATH = ({ day = "day1", facility = "Court%201" } = {}) => `/v3/days/${day}/facilities/${facility}/prize-draws`;
const SPIN_PATH = (spin = 1, { day = "day1", facility = "Court%201", drawId = "K7Q4M2XA" } = {}) => `${LIST_PATH({ day, facility })}/${drawId}/spins/${spin}`;
const put = (spin, body, opts = {}) => request("PUT", SPIN_PATH(spin, opts), { token: operatorToken(), body });

describe("PUT /v3/days/:day/facilities/:facility/prize-draws/:drawId/spins/:spin", () => {
    it("records spin 1 to row 2 of a new DRAWS tab and answers where it went", async () => {
        const res = await put(1, SPIN1);
        assert.equal(res.status, 200);
        assert.deepEqual(res.json, { day: "day1", facility: "Court 1", drawId: "K7Q4M2XA", spin: 1, rows: [2], created: true, drawnAt: "2099-12-31T02:00:00.000Z" });
        assert.equal(sheets.tabs.S1[1][13], LIST.join("\n"));
    });

    it("answers a retry with created: false", async () => {
        await put(1, SPIN1);
        const res = await put(1, SPIN1);
        assert.deepEqual([res.status, res.json.created, sheets.writes.length], [200, false, 1]);
    });

    it("400 for a bad draw id, spin or body", async () => {
        isProblem(await put(1, SPIN1, { drawId: "k7q4m2xa" }), 400, "validation_error");
        isProblem(await request("PUT", SPIN_PATH("0"), { token: operatorToken(), body: SPIN1 }), 400, "validation_error");
        isProblem(await put(1, { ...SPIN1, winners: [{ ...SPIN1.winners[0], result: "absent" }] }), 400, "validation_error");
        isProblem(await put(1, { ...SPIN1, fingerprint: "0".repeat(64) }), 400, "validation_error");
    });

    it("409 draw_conflict for a spin out of order", async () => {
        isProblem(await put(2, { ...SPIN1, list: undefined, winners: [{ place: 1, winner: "Ben Cruz", result: "won" }] }), 409, "draw_conflict");
    });

    it("409 draws_layout for a DRAWS tab in another shape", async () => {
        sheets.tabs.S1 = [["name", "prize"]];
        isProblem(await put(1, SPIN1), 409, "draws_layout");
    });

    it("403 for an archived event, 404 for an unknown day or facility", async () => {
        isProblem(await put(1, SPIN1, { day: "old-day1" }), 403, "forbidden");
        isProblem(await put(1, SPIN1, { day: "nope" }), 404);
        isProblem(await put(1, SPIN1, { facility: "Court%209" }), 404);
    });

    it("401 without an operator token", async () => {
        isProblem(await request("PUT", SPIN_PATH(), { body: SPIN1 }), 401, "unauthorized");
        isProblem(await request("PUT", SPIN_PATH(), { token: deskToken(), body: SPIN1 }), 401, "unauthorized");
        isProblem(await request("PUT", SPIN_PATH(), { token: scorerToken(), body: SPIN1 }), 401, "unauthorized");
    });

    it("415 for a body that is not JSON, 405 with Allow: PUT on GET", async () => {
        isProblem(await request("PUT", SPIN_PATH(), { token: operatorToken(), raw: "x", headers: { "Content-Type": "text/plain" } }), 415, "unsupported_media_type");
        const res = await request("GET", SPIN_PATH(), { token: operatorToken() });
        isProblem(res, 405, "method_not_allowed");
        assert.equal(res.headers.get("Allow"), "PUT");
    });
});

describe("GET /v3/days/:day/facilities/:facility/prize-draws", () => {
    it("lists nothing before a draw is recorded, with Cache-Control: no-store", async () => {
        const res = await request("GET", LIST_PATH(), { token: operatorToken() });
        assert.equal(res.status, 200);
        assert.deepEqual(res.json, { day: "day1", facility: "Court 1", draws: [] });
        assert.equal(res.headers.get("Cache-Control"), "no-store");
    });

    it("lists a recorded draw with its list and winners", async () => {
        await put(1, SPIN1);
        await put(2, { ...SPIN1, list: undefined, winners: [{ place: 1, prize: "Cap (1 of 2)", ticket: "002", winner: "Ben Cruz", result: "won" },
            { place: 2, prize: "Cap (2 of 2)", ticket: "003", winner: "Carla Santos", result: "won" }] });
        const res = await request("GET", LIST_PATH(), { token: operatorToken() });
        assert.equal(res.json.draws.length, 1);
        assert.deepEqual(res.json.draws[0].list, LIST);
        assert.deepEqual(res.json.draws[0].picks.map(p => [p.spin, p.place, p.winner, p.prize]),
            [[1, 1, "Ana Reyes", "Paddle"], [2, 1, "Ben Cruz", "Cap (1 of 2)"], [2, 2, "Carla Santos", "Cap (2 of 2)"]]);
    });

    it("401 without an operator token; 404 for an unknown day; 405 with Allow on POST", async () => {
        isProblem(await request("GET", LIST_PATH()), 401, "unauthorized");
        isProblem(await request("GET", LIST_PATH(), { token: deskToken() }), 401, "unauthorized");
        isProblem(await request("GET", LIST_PATH({ day: "nope" }), { token: operatorToken() }), 404);
        const res = await request("POST", LIST_PATH(), { token: operatorToken(), body: {} });
        isProblem(res, 405, "method_not_allowed");
        assert.equal(res.headers.get("Allow"), "GET, HEAD");
    });
});
```

**README.** In `README.md`'s Changelog, add at the top an entry in the same heading style as the ones
already there:

```markdown
### 3.9.0 — prize draws

- `GET /v3/days/{day}/facilities/{facility}/prize-draws` lists the prize draws recorded in a facility
  workbook's `DRAWS` tab, and `PUT /v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}`
  records one spin's winners there, one row each, creating the tab the first time. Operator token only.
  Nothing is published.
- `SheetsClient` writes one more range: whole `DRAWS` rows (A to O, row 2 on), through their own allowlist,
  and creates the `DRAWS` tab.
- **Tests:** `test/unit/draws/domain/drawsTab.test.mjs`, `test/unit/draws/domain/drawRequest.test.mjs`,
  `test/unit/draws/DrawService.test.mjs`, `test/integration/draw-routes.test.mjs`,
  `test/unit/clients/SheetsClient.test.mjs`, `test/integration/v3.test.mjs`, `test/unit/helpers/routes.test.mjs`,
  `test/unit/guards/rest-conventions.test.mjs`, `test/unit/guards/dependency-rules.test.mjs`,
  `test/integration/openapi.test.mjs`, `test/unit/app.test.mjs`, `test/unit/docs/openapiSpec.test.mjs` (and its
  snapshot), `test/helpers/routeManifest.mjs`, `test/helpers/coveredElsewhere.mjs`.
```

Also add the new routes and test files to the README's **Testing** table and route list, the same way the
3.8.0 court route was added (`grep -n "courts" README.md` shows each place).

Then run `npm test` (all pass) and `npm run verify`.

### 7.8 The workspace `CLAUDE.md` (`D:\Personal\SAGE\CLAUDE.md`)

Make these edits, each to text that occurs once.

**The repos table**, `sage-tools-api` row:
- after `and score entry (\`src/scores/\`)`, add `, and prize draws (\`src/draws/\`)`;
- change `Four features` to `Five features`.

**Layout**
- After the `src/courts/` bullet, add:

  > `src/draws/` — prize draws (`sage-docs/docs/specs/.../prize-draw-spec.md`): `domain/drawsTab.mjs` (the
  > `DRAWS` tab, one row per winner, the list in spin 1, place 1's column N), `domain/drawRequest.mjs` (input
  > validation), `DrawService` (writes one spin's rows through `SheetsClient`, lists a workbook's draws;
  > publishes nothing) and the `/v3` routes. Operator token only.

- In the `src/attendance/` bullet, after
  `and, for court control, one \`Court Control\` match cell in column C`, add
  `, and, for prize draws, whole \`DRAWS\` rows`.

**Endpoints**
- Change `Eighteen routes; thirteen of them` to `Twenty routes; thirteen of them`.
- Change `and the court match, have none:` to `the court match and the two prize-draw routes, have none:`.
- After the `PUT /v3/days/{day}/facilities/{facility}/courts/{court}/match` bullet, add:

  > - `GET /v3/days/{day}/facilities/{facility}/prize-draws`, `PUT /v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}` —
  >   list a workbook's prize draws, record one spin's winners to its `DRAWS` tab; operator token only; no
  >   frozen twin.

**`sage-match-control.github.io`, `tools/` bullet:** after `` `bracket-generator.html`, ``, add
`` `prize-draw.html` (Lucky Dink, the prize draw: `sage-docs/docs/specs/.../prize-draw-spec.md`), ``.

### 7.9 The page in Phase 2

All of this goes in `tools/prize-draw.html`. Add three ids to §5.16's id list: `recordStatus`, `pastDraws` and
`pastList`.

**Recording**
- It is on when a draw is locked with an event and day picked, and an operator sign-in held (§5.7).
- Every spin, and every Draw again, is sent with `fetchJson(url, { method: 'PUT', token, body })`.
- The URL:
  `${CLOUD_RUN_BASE_URL}/v3/days/${enc(day)}/facilities/${enc(facility)}/prize-draws/${state.drawId}/spins/${spin}`
- The body:

  ```js
  {
    title, seed, fingerprint, rule, entries: entries.length,
    winners: picksOfThisSpin.map(p => ({ place, prize, ticket, winner, result })),
  }
  ```

  On spin 1 it adds `list: entries` and `continues` (the previous part's `drawId`, or `''`). `title` is `''`
  when empty. A Draw again re-sends the whole spin, with that winner's `result: 'redrawn'`.

**The queue**
- Sends go through one queue, in order, one at a time.
- A spin's picks are marked `recorded: true` only once its answer arrives, and the queue survives a reload
  because `state` is saved.
- **A failure** leaves the spin unrecorded and stops the queue. `#recordStatus` reads *Spin 3 isn't in the
  workbook yet. [Retry]*, and Retry resumes.
- **A `409`** shows the answer's `error` text and stops the queue without retrying, because the workbook
  disagrees and a person has to look.
- Spinning never waits for recording.

**`#recordStatus`**, a line under `#seedLine`, reads one of:
- *Recording to the <facility> workbook* (when on);
- *Recorded: 4 of 4 spins* (each answer updates it);
- *Not recorded: sign in to Control Center in this browser* (no sign-in);
- *Not recorded: pick an event and day before locking* (no event).

Locking without a sign-in still works, and the draw is then local only. A sign-in that arrives later starts
recording from spin 1: the queue sends every unrecorded spin of every part, oldest first.

**Past draws**
- `#pastDraws` is a card under Winners, shown when an event and day are picked and a sign-in is held.
- **Load past draws** runs `GET …/prize-draws` for the day's facility. `#pastList` then has one row per draw:
  - the title (or *Untitled draw*);
  - *part 2* when `continues` is set;
  - the first pick's `drawnAt`, as local time;
  - *4 winners*;
  - and two buttons, **Check** and **Continue**.
- **Check** first checks `sha256Hex(listText(d.list)) === d.fingerprint`. When that fails, it shows *The
  list in the workbook doesn't match its fingerprint.* Otherwise it runs
  `verifyDraw({ entries: d.list, seed: d.seed, rule: d.rule, picks: d.picks, sha256Hex })` and shows the
  result as in §5.13.
- **Continue** asks `confirm('Continue this draw here? The draw open now is replaced.')`, then loads it into
  `state`:
  - locked, with `entries = d.list`;
  - the picks with `recorded: true`;
  - and the earlier parts, found by following `continues` back through the list.

  The page can't know prizes not yet drawn, so the operator adds any remaining prize rows.

## 8. Checks

### 8.1 Phase 1, in a browser

Serve the site from its root with any static server, for example `npx http-server -p 8080 -c-1`, and open
`http://localhost:8080/tools/prize-draw`. Check at 1280px and at 375px:

1. **No sideways scroll** at 375px, and no console error on load. There is no seed box anywhere.
2. **A draw with prizes.**
   - Paste §4.4's six names, add prizes *Paddle* × 1 and *Cap* × 2, and Lock.
   - The seed line reads *Draw code ######## · 6 entries · list 3de62d1c · spin 1*.
   - Spin three times. The prizes read *Paddle*, *Cap (1 of 2)*, *Cap (2 of 2)*, and no winner repeats.
3. **Reorder.** On a new draw, press **Next** on *Cap* before the first spin. Spin 1 is then for
   *Cap (1 of 2)*.
4. **Draw all at once.** Tick it on *Cap*. One spin draws both caps, the reveal lists two winners, and the
   reel lands on place 1.
5. **Draw again** on the latest spin's winner. It shows *(drawn again)*, and the next spin is for the same
   prize. No button or text says "absent".
6. **Unlock** shows before the first spin and hides after it.
7. **Edit list** after a spin:
   - Add a name and Lock again. The toast names how many earlier winners were left out.
   - The winners list shows *Part 1* and *Part 2*.
   - **Check this draw** ticks every winner in both parts.
8. **Copy results**, **Download .txt** and **Save image** each work.
9. **The Pickle Jar:**
   - With 6 names, labelled balls.
   - With 120 names, small balls without labels, and the winner's ball still drops into the tray.
   - With 301 names in the box, the Jar button is disabled, with its reason.
10. **Open display.**
    - A second window opens, showing *Waiting for the controls…* only until the controls answer.
    - Spin: the animation plays in the display window and the controls show *Showing on the display window*.
    - Close the display window: within 5 s the controls show the stage again.
11. **Numbers, 200:** Fill, Lock and spin. The reel and the reveal show *#NNN* only.
12. **Big screen** hides the setup, and Escape leaves it. **Reload mid-draw** restores the draw.
13. **From Control Center** (`/tools/control-center`, any event): **Pages and tools** shows **Open prize
    draw**, which opens the page with the event and day picked.
14. **Players on a day:** pick `upap-2026` and its day, and Load players: the box fills.
    - With Control Center open and signed in in another tab, **Load checked in** enables.
    - Don't press it against a live event's data unless the owner agrees.

### 8.2 Phase 2

1. Run `npm test` and `npm run verify` in the API: all pass.
2. After the deploy, `GET /ping` answers `X-App-Version: 3.9.0`. Use Node's `http2`, as in the README's
   sanity check.
3. **Ask the owner for a copy of a facility workbook to test on,** registered to a test day. Never use a live
   event's workbook. Then:
   - Lock a 6-name draw with *Cap* × 2 drawn all at once. Spin, then Draw again on one cap winner, and spin
     again.
   - The copy gains a `DRAWS` tab:
     - spin 1 has two rows (places 1 and 2), the list in N2, and one result reading `redrawn`;
     - spin 2 has one row.
   - **Load past draws** lists the draw, **Check** ticks every winner, and **Continue** in another browser
     carries on from spin 3.

## 9. Docs

### 9.1 Phase 1 (`docs 2.78.0`)

- **`docs/features/prize-draw.md` (new).** It covers, with no code:
  - what it is, and the four name sources;
  - prizes, with quantities, Draw all at once and reordering;
  - the rule;
  - spinning, Draw again, Unlock and Edit list;
  - the two styles and the display window;
  - the exports, Check this draw and Big screen;
  - the No Rigging Zone copy.

  Its last line links to the technical page.
- **`docs/technical/prize-draw.md` (new).** It covers:
  - the file, and its imports from `lib/v1/`;
  - `domain/prize-draw.js`'s rule (§4.1, stated as there);
  - the state object and `sage.prizeDraw`;
  - the reel and jar parameters;
  - the display window's messages (§5.15);
  - the operator-session sign-in;
  - the tests.

  Its last line links to the features page.
- **`docs/features/README.md`:** under **Running the day**, after Control Center, add *[Prize draw]
  (prize-draw.md) — Lucky Dink: raffle and door-prize winners drawn on a big screen, with a code anyone can
  check.*
- **`docs/technical/README.md`:** the same entry, in its matching list.
- **`mkdocs.yml`:**
  - add `Prize draw: features/prize-draw.md` and `Prize draw: technical/prize-draw.md` beside the Bracket
    Generator entries;
  - bump the `copyright` line's version.
- **`docs/features/control-center.md`**, the **Pages and tools** bullet:
  - after the **Open scoresheet generator** sentence, add *; and **Open prize draw**, which opens the [Prize
    draw](prize-draw.md) on the selected event and day*;
  - change *three or four of them* to *four or five of them*.
- **`docs/technical/control-center.md`:** where the **Pages and tools** tiles are listed, add the prize-draw
  tile and its `?event=&day=` link.
- **`docs/technical/site-engine.md`:** add `domain/prize-draw.js` where the domain modules are described, in
  one line: the prize draw's rules, imported by `tools/prize-draw.html`.
- **This spec:** move it from `not-started/` to `in-progress/` (Phase 1 built, Phase 2 not). Follow
  `docs/specs/README.md`'s "Moving a spec between folders", and update its status line.
- **`docs/changelog.md`:** a `## [2.78.0]` entry (Manila date) under **Added**, naming both new pages and the
  status move.

### 9.2 Phase 2 (`docs 2.79.0`)

- **`docs/technical/api.md`:**
  - add the two routes to the `/v3` routes table;
  - add a row to the exceptions table:

    | Route | Rule | Why |
    | --- | --- | --- |
    | `PUT …/prize-draws/{drawId}/spins/{spin}` | a `GET` for every `PUT` | a draw's spins are read with `GET …/prize-draws`, the list the page reads; a per-spin `GET` would have no reader |

  - add `draws_layout` (409) and `draw_conflict` (409) to the codes table;
  - add both routes to **Auth, by route**, under operator only.
- **The two prize-draw pages:** recording, the `DRAWS` tab (§7.2's table), Past draws, Check and Continue.
- **This spec:** move it to `implemented/`, as in §9.1.
- **`docs/changelog.md`:** a `## [2.79.0]` entry.

## 10. Out of scope

- Weighted entries, and more than one ticket per person.
- Typing a seed.
- Sound.
- Event colours.
- A public results page.
- Controlling the display from another device (a phone, say).
- More than 300 balls in the Pickle Jar.
- Drawing from several facilities' workbooks into one draw.
- Editing a recorded winner in any way but Draw again on the latest spin.
- Deleting a draw from the workbook: a person deletes its rows by hand.
- Any change to `event-data`, the live Worker or the snapshot.
