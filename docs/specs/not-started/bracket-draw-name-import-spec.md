# Spec — Bracket draw name import

Fill a generated workbook's rosters by uploading the Bracket Generator's
exported **text files**, one per category, instead of typing every pair into
each category tab's `STEP 1 · NAMES` column and then hand-shuffling
`STEP 3 · RANDOMIZED CODES`.

> **Status: written, not yet in any workbook.** §9, §11 and §12 are all
> applied in `sage-tools-api/scripts/` and every harness is green — §9.8's 31
> checks, §12's 13, and both verify scripts. §9 remains a ready-to-apply guide:
> exact insertions, anchored on quoted text, with a runnable check after each.
>
> What that does **not** prove is anything a real workbook does with the
> values, because the mock evaluates no formulas and nothing here has been
> pasted into a master yet. §9.9's step 3 is the gap: until it passes against a
> real generated workbook, treat this as unproven in production.

| | |
| --- | --- |
| Where the code goes | `standard-generator.gs` (§9, the import), `sheets-sync.gs` + `tools/bracket-generator.html` (§11, the menu route), `sheet-generator.gs` (§12, the dual-meet shuffle) |
| How it ships | Bound Apps Script in both masters — paste, not deploy. §11's tool half is a static-site commit |
| New infra | none |
| New credentials | none |
| `sage-tools-api` change | none — no version bump, no Cloud Run deploy. Everything here is in `scripts/*.gs`, which is not part of the service |
| Reads | the `.txt` files [`tools/bracket-generator.html`](../../features/bracket-generator.md) already exports |
| Writes | each category tab's `AD` (names), `AI` (codes) and `AB2:AB4` (draw provenance); §12 writes a dual meet's `AG`/`AV` — nothing else |

**Read first:**
[`standard-tournament-master-spec.md`](../implemented/standard-tournament-master-spec.md)
§7.2 (group pairs and bracket order), §7.3 (the code ladder) and §7.6 (the two
scaffolds).

This replaces a spec that went the other way round. `bracket-generator-workbook-handoff-spec.md`
proposed a SAGE menu route *into* the tool plus a new tool output shaped like
`STEP 3`, and was deferred until a standard-tournament generator existed. Once
one did, the import direction turned out to need nothing new from the tool —
the exported text file already carries everything — so that spec was retired
and its two surviving ideas are recorded here, in §11 and §12.

---

## 0. Who does what

**Implementer (§9).** Three files, all in `sage-tools-api/scripts/`:

| File | Change |
| --- | --- |
| `standard-generator.gs` | the menu item, the sidebar and nine new functions (§9.1-§9.6) |
| `mock-apps-script.mjs` | one method, `MockRange.getValues` (§9.7) |
| `verify-standard-generator.mjs` | one new scenario (§9.8) |

**Implementer (§11, §12).** Three more, two of them outside `standard-generator.gs`:

| File | Change |
| --- | --- |
| `sheets-sync.gs` | `BRACKET_GENERATOR_URL`, one menu item, `showBracketGeneratorLink` (§11) |
| `tools/bracket-generator.html` | `initCategoryFromQuery` (§11) — the only change outside `sage-tools-api` |
| `sheet-generator.gs` | the menu item and the shuffle functions (§12) |
| `verify-sheet-generator.mjs` | one new scenario (§12) |

Nothing is deployed and no version is bumped: `scripts/*.gs` is not part of the
Cloud Run service, and the tool is a static page.

**Operator, afterwards.** Paste the edited `standard-generator.gs` into the
SAGE Standard Tournament Master's Apps Script project, replacing the file
there. Workbooks generated *before* that paste do **not** get the feature:
each copy carries the script it was made from. Only copies taken after it do.

**Do not touch:**

- Anything in `src/`. `sheet-generator.gs` and `sheets-sync.gs` are off limits
  to §9, which is confined to `standard-generator.gs`; §11 and §12 are the
  only reason either is edited at all, and only where those sections say.
- `generateEventTabs` and everything it calls. This runs *after* generation and
  shares only the trace and progress-log helpers.
- The `onOpen` block — it is byte-identical across three `.gs` files with a
  "Change one, change all three" note. §9.5 and §12 edit
  `addGeneratorMenuItems_` and §11 edits `addSyncMenuItems_`, all of which
  `onOpen` already calls.
- Any formatting, any other tab, and any cell outside `AD`, `AI`, `AB2:AB4`
  (§9) or `AG`/`AV` (§12). The generator already laid the scaffold out; this
  writes values into it.

**Stop and ask** rather than inventing an answer if:

- the plan block is not at `Variables!I11:L…` in the master you test against
  (§3.4);
- a real exported draw file does not parse under §2 — the tool's export format
  may have changed, and then §2's table is the thing to fix first;
- you want to write to a cell this spec does not name.

## 1. The gap this closes

Preparing one standard-tournament venue-day today means, per category:

1. Draw the brackets in the Bracket Generator, download the image and the
   text file.
2. Read the text file and type each pair's two players into `AD`, two rows per
   pair, in bracket order.
3. Paste the codes from `AH` into `AI` in a shuffled order, by hand.

Step 2 is transcription — the names are already in a file — and step 3 is
busywork the draw has already done (§8.1). The import does both, and records
which draw it did them from.

## 2. Input — the Bracket Generator's text export

Exactly what `exportAsText()` in `tools/bracket-generator.html` writes today,
unchanged. One file per category, named `<event>-<category>-brackets.txt`:

```
PICKLE FOR SIGHT TOURNAMENT
HIGH INTERMEDIATE MEN'S DOUBLES
Drawn 9/23/2026, 7:41:02 PM
3 brackets, 13 pairs
============================================

BRACKET A
------------------------
1. Juan Dela Cruz / Maria Santos
2. Mark Limjuco / Allen Dizon
…

BRACKET B
------------------------
1. …

DRAW VERIFICATION
------------------------
Seed:   BUNUTAN  (typed by the organiser)
Draw:   #1
Method: SHA-256(seed + "|" + pair), sorted ascending, dealt round-robin

Fingerprints, in draw order:
  1f0a3c9d...  Juan Dela Cruz / Maria Santos
…
TO CHECK THIS DRAW YOURSELF
…
```

`parseDrawText_` (§9.1) reads exactly these lines and ignores every other:

| Pattern | Read as |
| --- | --- |
| the first line among the first six matching `^Drawn\s+` | the **anchor**; everything else is positioned from it |
| the line **before** the anchor | **category**, as the operator typed it in the tool, **upper-cased by the export** — so §4 compares case-insensitively |
| the line **two before** the anchor, when there is one | event name — logged, never matched on |
| `^BRACKET\s+(\S+)$` | a bracket, in file order. Its label (`A`, `B`, …) is positional only |
| `^(\d+)\.\s+(.+)$` after a `BRACKET` line | one **pair**, in file order (§5) |
| `(no pairs assigned)` | nothing — the bracket stays empty, which §6 then rejects |
| `^Seed:\s+(.*?)\s*\((.*)\)\s*$` after `DRAW VERIFICATION` | the seed and its source |
| `^Draw:\s+#(\d+)$` after `DRAW VERIFICATION` | the draw number |

Parsing stops at `TO CHECK THIS DRAW YOURSELF`. The fingerprint list is **not**
re-verified (§8.2).

## 3. What it writes

Per category, into that category's tab only. `teams` below is that category's
pair count from the plan block (§3.4).

### 3.1 `AD` — STEP 1 · NAMES

`AD5:AD<4 + 2 × teams>`, two rows per pair, in **import order**: bracket A's
pairs first, then bracket B's, and so on; each pair's first player on the
pair's first row, its second player on the second (§5).

That order is load-bearing. It is the order the tab's group pairs run in
(master spec §7.2), so bracket A's pairs take codes `<KEY>_1 …`, which is what
puts them in bracket 1 on the tab, in `STANDINGSCSV!G` and in the score grids.

### 3.2 `AI` — STEP 3 · RANDOMIZED CODES

`AI5:AI<4 + teams>`: `<KEY>_1` … `<KEY>_<teams>`, in order, as plain strings.

An identity mapping, on purpose — §8.1 is why that is right here and wrong by
hand.

### 3.3 `AB2:AB4` — the draw's provenance

Three plain strings in the notes column beside the roster, which nothing
reads:

```
AB2   Draw seed: BUNUTAN (typed by the organiser)
AB3   Draw #1, drawn 9/23/2026, 7:41:02 PM
AB4   Imported 2026-09-27T11:04:22.134Z from pfs-himd-brackets.txt
```

Built exactly as §9.3 builds them, `new Date().toISOString()` included. A
missing seed or draw number (a hand-edited file) writes what it has and leaves
the rest out; never a failure.

### 3.4 Where the category's own numbers come from

`Variables`, as the generator wrote it (master spec §10.1): one row per plan
category from **row 11**, with `I` = key, `J` = display value, `K` = teams,
`L` = brackets. `readPlanBlock_` (§9.2) reads `I11:L<lastRow>` and keeps rows
with a non-empty key.

A plan row is a **generated tab** only when `ss.getSheetByName(key)` exists.
`Variables` lists every category in the plan CSV, including the other venue's
(master spec §6.3), and those have no tab here.

## 4. Matching a file to a category tab

The tool's category is free text typed by the operator; the workbook's is a
key (`HIMD`) with a display name in `Variables!J`. `resolveDrawCategory_`
(§9.2) tries, in order:

1. `norm(file category)` equals `norm(key)` — the key itself;
2. `norm(file category)` equals `norm(value)` — the display name;
3. nothing — the sidebar shows a dropdown of this workbook's category tabs
   beside that file, and the operator picks.

`norm` is: trim, upper-case, collapse runs of whitespace to one space. An
operator-picked assignment always wins over both.

A file left unassigned is **skipped**, not guessed. Two files resolving to the
same tab is a validation failure (§6).

## 5. Splitting a pair into two players

The tool takes one pair per line as free text; the workbook wants two rows.
`splitPairNames_` (§9.1) splits on the **earliest** occurrence of any of:

| Separator | Example | Gives |
| --- | --- | --- |
| `/` | `Juan Dela Cruz / Maria Santos` | `Juan Dela Cruz`, `Maria Santos` |
| `&` | `Juan & Maria` | `Juan`, `Maria` |
| `+` | `Juan + Maria` | `Juan`, `Maria` |
| `,` | `Dela Cruz, Santos` | `Dela Cruz`, `Santos` |
| ` and ` (whitespace either side, any case) | `Juan and Maria` | `Juan`, `Maria` |

Rules:

- A separator at position 0 does not count, so `/ Maria` stays one name.
- Only the earliest separator splits: `A / B / C` gives `A` and `B / C`.
- **No separator:** the whole string goes on the pair's first row, the second
  row is written as `''`, and the import **warns**, naming the category and
  the pair. Some events enter a team name rather than two players, so this is
  a warning, not a failure.
- Both halves are trimmed. Nothing is reordered or reformatted.

## 6. Validation

`validateDraws_` (§9.2) runs every check below across **all** uploaded files
before anything is written, and returns every failure at once — the master
spec's §9 rule. Each message names the file.

| Rule | Message names |
| --- | --- |
| the file parses: an anchor line, a category, ≥1 `BRACKET`, ≥1 pair | the file |
| its category resolves to exactly one generated tab (§4), or the operator assigned one | the file and its category line |
| no two files resolve to the same tab | both files and the tab |
| the tab's `AD5:AD<4 + 2 × teams>` is entirely empty, unless `replace` is set | the tab |
| total pairs equals the plan's `teams` | both counts |
| bracket count equals `groupSizes_(teams, brackets).length` | both counts |
| per-bracket sizes equal `groupSizes_(teams, brackets)` exactly | both shapes, e.g. `5-5-3, but tab "HIMD" was built for 5-4-4` |
| no bracket is empty | the bracket label |
| the same pair text does not appear twice in one file | the pair |

`groupSizes_` already applies the master spec §5.6 reduction, so comparing
against it compares against the tab as built. The tool deals round-robin,
largest bracket first, so a correct draw matches element for element.

## 7. The sidebar

`SAGE → Import bracket draws`, below `Generate event tabs`:

- A drop zone and a file picker taking **several `.txt` files at once**, read
  in the browser with `FileReader` exactly as the generator's CSV box does, so
  nothing is uploaded anywhere.
- One row per file: filename, the category it resolved to or a dropdown (§4),
  pairs, bracket sizes, seed. A file that failed to parse shows why in place.
- A **Replace existing names** checkbox, off by default.
- **Import** writes every assigned file, then reports per category: how many
  pairs, plus every §5 warning.

The menu item **stays** after a successful import — unlike `Generate event
tabs`, this is safely repeatable with `Replace` ticked, which is how a re-draw
gets applied.

## 8. Decisions

### 8.1 Why `AI` is filled in order

Master spec §7.6 ships `AI` blank and says pre-seeding it in order defeats the
blinding. That holds for a **hand** roster, where roster order is entry order
— alphabetical, or as-registered — so an identity mapping tells everyone which
code is whose.

It does not hold here. The rows arrive in the order a seeded, published,
re-checkable draw produced (bracket generator verifiable-draw spec §1), so the
randomisation has already happened and the file proves it. Shuffling again
would only break the bracket assignment the draw made.

Consequence for the implementation: `AI` is written **only** together with
`AD`, in the same call, from the same file (§9.3). Never add a path that fills
`AI` on its own.

### 8.2 Why the fingerprints are not re-verified

They are what an outsider checks the draw with, on any SHA-256 site. Re-doing
that inside the workbook would prove only that the file is internally
consistent, which a hand-edited file can also be. §3.3's provenance is the
useful half: it ties this workbook to that file.

### 8.3 Why the qualifier draw is untouched

`AM` (master spec §7.6) is a second draw, made after the round robin from the
standings, on the day. There is no file to import.

---

## 9. Implementation guide

Every edit is to `sage-tools-api/scripts/standard-generator.gs` unless the step
says otherwise. **Apply them in order and do not paraphrase the code.**
Anchors are verbatim strings from the current file.

### 9.1 Pure parsing — `parseDrawText_`, `splitPairNames_`

Find:

```js
function byNumber_(a, b) { return a.number - b.number; }
```

Insert immediately **after** it:

```js
// ============================================================================
// BRACKET DRAW IMPORT — pure parsing
// Spec: sage-docs/docs/specs/.../bracket-draw-name-import-spec.md §2, §5
// ============================================================================

// Where the generator writes the plan block on Variables (master spec §10.1):
// one row per plan category from row 11, I=key J=value K=teams L=brackets.
var PLAN_FIRST_ROW = 11;
var VARS_COL = { I: 9, J: 10, K: 11, L: 12 };

/** Trimmed, upper-cased, inner whitespace collapsed — how §4 compares category text. */
function normalizeCategoryText_(s) {
  return String(s === null || s === undefined ? '' : s).trim().toUpperCase().replace(/\s+/g, ' ');
}

/**
 * §2 — reads one exported draw file. Never throws: a file it cannot make
 * sense of comes back with `errors` filled in, which §6 reports.
 */
function parseDrawText_(text, filename) {
  var out = {
    file: String(filename || ''), eventName: '', category: '', drawnAt: '',
    seed: '', seedSource: '', drawNumber: '', brackets: [], errors: [],
  };
  var lines = String(text === null || text === undefined ? '' : text).split(/\r?\n/);

  // The "Drawn ..." line is the anchor: the category is the line above it.
  var anchor = -1;
  for (var i = 0; i < lines.length && i < 6; i++) {
    if (/^Drawn\s+/.test(lines[i].trim())) { anchor = i; break; }
  }
  if (anchor < 1) {
    out.errors.push('no "Drawn ..." line in the first six lines — is this a Bracket Generator text export?');
    return out;
  }
  out.drawnAt = lines[anchor].trim().replace(/^Drawn\s+/, '');
  out.category = lines[anchor - 1].trim();
  if (anchor >= 2) out.eventName = lines[anchor - 2].trim();

  var current = null, inVerification = false;
  for (var j = anchor + 1; j < lines.length; j++) {
    var line = lines[j].trim();
    if (line === 'DRAW VERIFICATION') { inVerification = true; continue; }
    if (/^TO CHECK THIS DRAW YOURSELF/.test(line)) break;

    if (!inVerification) {
      var bracket = /^BRACKET\s+(\S+)$/.exec(line);
      if (bracket) { current = { label: bracket[1], pairs: [] }; out.brackets.push(current); continue; }
      var pair = /^(\d+)\.\s+(.+)$/.exec(line);
      if (pair && current) current.pairs.push(pair[2].trim());
      continue;
    }
    var seed = /^Seed:\s+(.*?)\s*\((.*)\)\s*$/.exec(line);
    if (seed) { out.seed = seed[1].trim(); out.seedSource = seed[2].trim(); continue; }
    var draw = /^Draw:\s+#(\d+)$/.exec(line);
    if (draw) out.drawNumber = draw[1];
  }

  if (!out.category) out.errors.push('no category line above "Drawn ..."');
  if (!out.brackets.length) out.errors.push('no "BRACKET x" section');
  else if (!out.brackets.reduce(function (n, b) { return n + b.pairs.length; }, 0)) {
    out.errors.push('no pairs under any bracket');
  }
  return out;
}

/** Bracket sizes in file order, e.g. [5, 4, 4]. */
function drawSizes_(draw) {
  return draw.brackets.map(function (b) { return b.pairs.length; });
}

/** Every pair in import order: bracket A's, then bracket B's (§3.1). */
function drawPairs_(draw) {
  var out = [];
  draw.brackets.forEach(function (b) { b.pairs.forEach(function (p) { out.push(p); }); });
  return out;
}

/**
 * §5 — one pair line into two players. Returns { one, two, split }; `split`
 * false means no separator was found, which the caller warns about.
 */
function splitPairNames_(pair) {
  var s = String(pair === null || pair === undefined ? '' : pair).trim();
  var at = -1, len = 0;
  ['/', '&', '+', ','].forEach(function (sep) {
    var i = s.indexOf(sep);
    if (i > 0 && (at < 0 || i < at)) { at = i; len = sep.length; }
  });
  var and = /\sand\s/i.exec(s);
  if (and && and.index > 0 && (at < 0 || and.index < at)) { at = and.index; len = and[0].length; }
  if (at < 0) return { one: s, two: '', split: false };
  return { one: s.slice(0, at).trim(), two: s.slice(at + len).trim(), split: true };
}
```

**Check:**

```bash
cd sage-tools-api && node -e "new Function(require('fs').readFileSync('scripts/standard-generator.gs','utf8'))" && echo "GS OK"
```

### 9.2 The workbook side — plan block, matching, validation

Find:

```js
// ============================================================================
// COLOURS (§10.2.4 "Category colours", dual-meet schedule spec §6)
// ============================================================================
```

Insert immediately **before** it:

```js
// ============================================================================
// BRACKET DRAW IMPORT — resolving and validating against the workbook
// Spec: sage-docs/docs/specs/.../bracket-draw-name-import-spec.md §3.4, §4, §6
// ============================================================================

/** §3.4 — the plan block, one entry per plan category, generated tabs flagged. */
function readPlanBlock_(ss) {
  var sheet = ss.getSheetByName(VARIABLES_SHEET_NAME);
  if (!sheet) throw new Error('"' + VARIABLES_SHEET_NAME + '" tab not found.');
  var last = sheet.getLastRow();
  if (last < PLAN_FIRST_ROW) return [];
  var rows = sheet.getRange(PLAN_FIRST_ROW, VARS_COL.I, last - PLAN_FIRST_ROW + 1, 4).getValues();
  var out = [];
  rows.forEach(function (row) {
    var key = String(row[0] === null || row[0] === undefined ? '' : row[0]).trim();
    if (!key) return;
    out.push({
      key: key,
      value: String(row[1] === null || row[1] === undefined ? '' : row[1]).trim(),
      teams: Math.round(Number(row[2])) || 0,
      brackets: String(row[3] === null || row[3] === undefined ? '' : row[3]).trim(),
      hasTab: !!ss.getSheetByName(key),
    });
  });
  return out;
}

/** The category tabs this workbook actually has, in plan order. */
function generatedCategories_(plan) {
  return plan.filter(function (row) { return row.hasTab; });
}

/** §6 — the bracket sizes the tab was built for. */
function planBracketSizes_(cat) {
  return groupSizes_(cat.teams, cat.brackets === '' ? 4 : Number(cat.brackets));
}

/** §4 — key, then display name, then nothing. A non-empty `assigned` key always wins. */
function resolveDrawCategory_(draw, plan, assigned) {
  var cats = generatedCategories_(plan);
  if (assigned) {
    return cats.filter(function (c) { return c.key === assigned; })[0] || null;
  }
  var want = normalizeCategoryText_(draw.category);
  var byKey = cats.filter(function (c) { return normalizeCategoryText_(c.key) === want; })[0];
  if (byKey) return byKey;
  return cats.filter(function (c) { return c.value && normalizeCategoryText_(c.value) === want; })[0] || null;
}

/**
 * §6 — every check, over every file, before anything is written. Returns
 * { errors, warnings, jobs }; `jobs` is what §9.3 writes, one per assigned
 * file, and is only trustworthy when `errors` is empty.
 */
function validateDraws_(ss, draws, options) {
  options = options || {};
  var plan = readPlanBlock_(ss);
  var errors = [], warnings = [], jobs = [], takenBy = {};

  draws.forEach(function (draw) {
    var label = '"' + draw.file + '"';
    if (draw.errors.length) {
      draw.errors.forEach(function (e) { errors.push(label + ': ' + e); });
      return;
    }
    var assigned = (options.assign || {})[draw.file] || '';
    var cat = resolveDrawCategory_(draw, plan, assigned);
    if (!cat) {
      // Not an error: the sidebar offers a dropdown, and an unassigned file is
      // skipped (§4). Reported so the operator sees that it was skipped.
      warnings.push(label + ': category "' + draw.category +
        '" matches no category tab — skipped. Pick a tab for it in the sidebar.');
      return;
    }
    if (takenBy[cat.key]) {
      errors.push(label + ' and "' + takenBy[cat.key] + '" both resolve to tab "' + cat.key + '".');
      return;
    }
    takenBy[cat.key] = draw.file;

    var sizes = drawSizes_(draw);
    var pairs = drawPairs_(draw);
    var want = planBracketSizes_(cat);
    var ok = true;

    var empty = draw.brackets.filter(function (b) { return !b.pairs.length; });
    if (empty.length) {
      errors.push(label + ': bracket ' + empty.map(function (b) { return b.label; }).join(', ') + ' has no pairs.');
      ok = false;
    }
    if (pairs.length !== cat.teams) {
      errors.push(label + ': ' + pairs.length + ' pairs, but tab "' + cat.key + '" was built for ' + cat.teams + '.');
      ok = false;
    }
    if (sizes.length !== want.length) {
      errors.push(label + ': ' + sizes.length + ' brackets, but tab "' + cat.key + '" was built for ' + want.length + '.');
      ok = false;
    } else if (sizes.join('-') !== want.join('-')) {
      errors.push(label + ': bracket sizes ' + sizes.join('-') + ', but tab "' + cat.key +
        '" was built for ' + want.join('-') + '.');
      ok = false;
    }
    var seen = {};
    pairs.forEach(function (p) {
      var k = normalizeCategoryText_(p);
      if (seen[k]) { errors.push(label + ': pair "' + p + '" appears twice.'); ok = false; }
      seen[k] = true;
    });

    var sheet = ss.getSheetByName(cat.key);
    var filled = sheet.getRange(5, COL.AD, 2 * cat.teams, 1).getValues()
      .filter(function (r) { return String(r[0] === null || r[0] === undefined ? '' : r[0]).trim() !== ''; });
    if (filled.length && !options.replace) {
      errors.push('Tab "' + cat.key + '" already has names in AD. Tick "Replace existing names" to overwrite them.');
      ok = false;
    }

    pairs.forEach(function (p) {
      if (!splitPairNames_(p).split) {
        warnings.push(cat.key + ': "' + p + '" has no name separator — written to one row, its second row left blank.');
      }
    });

    if (ok) jobs.push({ draw: draw, cat: cat, pairs: pairs, sizes: sizes });
  });

  return { errors: errors, warnings: warnings, jobs: jobs };
}
```

**Check:** the same `new Function(...)` syntax check as §9.1.

### 9.3 The write — `writeDrawNames_`

Insert immediately **after** `validateDraws_`, in the same block:

```js
/**
 * §3 — writes one job's names, codes and provenance. Three setValues calls and
 * no formatting: the generator already built the scaffold (master spec §7.6).
 *
 * AD and AI are written together, always, from the same file — §8.1.
 */
function writeDrawNames_(ss, job, stamp) {
  var cat = job.cat;
  var sheet = ss.getSheetByName(cat.key);
  var names = [], codes = [];
  job.pairs.forEach(function (pair, idx) {
    var split = splitPairNames_(pair);
    names.push([split.one], [split.two]);
    codes.push([cat.key + '_' + (idx + 1)]);
  });
  sheet.getRange(5, COL.AD, names.length, 1).setValues(names);
  sheet.getRange(5, COL.AI, codes.length, 1).setValues(codes);

  var draw = job.draw;
  var seedLine = draw.seed
    ? 'Draw seed: ' + draw.seed + (draw.seedSource ? ' (' + draw.seedSource + ')' : '')
    : 'Draw seed: (not in the file)';
  var drawLine = (draw.drawNumber ? 'Draw #' + draw.drawNumber : 'Draw #?') +
    (draw.drawnAt ? ', drawn ' + draw.drawnAt : '');
  sheet.getRange(2, COL.AB, 3, 1).setValues([
    [seedLine],
    [drawLine],
    ['Imported ' + stamp + ' from ' + draw.file],
  ]);

  trace_('imported ' + cat.key + ': ' + job.pairs.length + ' pairs, sizes ' + job.sizes.join('-') +
    ', seed ' + (draw.seed || '(none)') + ', from ' + draw.file);
  return { key: cat.key, pairs: job.pairs.length };
}
```

**Check:** syntax check, then

```bash
grep -n "getRange(5, COL.AD\|getRange(5, COL.AI\|getRange(2, COL.AB" scripts/standard-generator.gs
```

→ `COL.AD` appears twice (the emptiness read in §9.2, the write here), `COL.AI`
and `COL.AB` once each, all inside the import functions.

### 9.4 The entry points — `previewDraws`, `importBracketDraws`

Insert immediately **after** `writeDrawNames_`:

```js
/**
 * Called by the sidebar whenever files are dropped, so the operator sees what
 * was read before anything is written. `files` is [{ name, text }].
 */
function previewDraws(files) {
  var ss = SpreadsheetApp.getActive();
  var plan = readPlanBlock_(ss);
  var cats = generatedCategories_(plan);
  return {
    categories: cats.map(function (c) {
      return { key: c.key, value: c.value, teams: c.teams, sizes: planBracketSizes_(c).join('-') };
    }),
    files: (files || []).map(function (f) {
      var draw = parseDrawText_(f.text, f.name);
      var cat = draw.errors.length ? null : resolveDrawCategory_(draw, plan, '');
      return {
        name: draw.file,
        category: draw.category,
        resolved: cat ? cat.key : '',
        pairs: drawPairs_(draw).length,
        sizes: drawSizes_(draw).join('-'),
        seed: draw.seed,
        errors: draw.errors,
      };
    }),
  };
}

/**
 * Entry point. `files` is [{ name, text }]; `options` is
 * { assign: { filename: key }, replace: boolean }. Validates everything first
 * (§6) and returns { errors: [...] } having written nothing, or
 * { message: "..." } once every job is written — never both.
 */
function importBracketDraws(files, options) {
  options = options || {};
  resetLog_();
  trace_('=== importBracketDraws START ' + new Date().toISOString() + ' ===');
  var ss = SpreadsheetApp.getActive();
  var draws = (files || []).map(function (f) { return parseDrawText_(f.text, f.name); });
  logStep_('Read ' + draws.length + ' file(s).');

  var checked = validateDraws_(ss, draws, options);
  if (checked.errors.length) {
    checked.errors.forEach(function (e) { trace_('VALIDATION ERROR: ' + e); });
    logStep_('Rejected: ' + checked.errors.length + ' problem(s) — nothing was written.');
    return { errors: checked.errors };
  }
  if (!checked.jobs.length) {
    return { errors: ['No file was assigned to a category tab — nothing to import.'].concat(checked.warnings) };
  }

  var stamp = new Date().toISOString();
  var done = [];
  try {
    checked.jobs.forEach(function (job, i) {
      logStep_('Importing "' + job.draw.file + '" into ' + job.cat.key +
        ' (' + (i + 1) + '/' + checked.jobs.length + ')...');
      done.push(writeDrawNames_(ss, job, stamp));
    });
  } catch (e) {
    var madeIt = done.map(function (d) { return d.key; }).join(', ') || '(none)';
    trace_('EXCEPTION after importing [' + madeIt + ']: ' + e.message);
    logStep_('Failed: ' + e.message);
    throw new Error('Failed after importing: ' + madeIt + '. Error: ' + e.message);
  }

  logStep_('Done.');
  return {
    message: 'Imported ' + done.length + ' categor' + (done.length === 1 ? 'y' : 'ies') + ': ' +
      done.map(function (d) { return d.key + ' (' + d.pairs + ' pairs)'; }).join(', ') + '.\n\n' +
      'Names are in STEP 1 and the codes in STEP 3, in draw order, and each tab\'s ' +
      'AB2:AB4 records the seed and draw they came from.' +
      (checked.warnings.length ? '\n\nWarnings:\n' + checked.warnings.join('\n') : ''),
  };
}
```

**Check:** syntax check, then
`grep -n "function importBracketDraws\|function previewDraws" scripts/standard-generator.gs`
→ one each.

### 9.5 The menu item

The import must survive generation, so it goes **outside** the
`PROP_TABS_GENERATED_FOR` guard.

Find:

```js
function addGeneratorMenuItems_(menu, state) {
  if (PropertiesService.getScriptProperties().getProperty(PROP_TABS_GENERATED_FOR)
      === SpreadsheetApp.getActive().getId()) {
    return;
  }
  if (state.count > 0) menu.addSeparator();
  menu.addItem('Generate event tabs', 'showGenerateSidebar');
  state.count += 1;
}
```

Replace with:

```js
function addGeneratorMenuItems_(menu, state) {
  var generated = PropertiesService.getScriptProperties().getProperty(PROP_TABS_GENERATED_FOR)
      === SpreadsheetApp.getActive().getId();
  if (state.count > 0) menu.addSeparator();
  // Generate event tabs contributes nothing once this workbook has been
  // generated — a second run is refused anyway. Import bracket draws stays:
  // rosters arrive after generation, and a re-draw is imported over the top
  // with "Replace existing names" (spec §7).
  if (!generated) {
    menu.addItem('Generate event tabs', 'showGenerateSidebar');
    state.count += 1;
  }
  menu.addItem('Import bracket draws', 'showImportSidebar');
  state.count += 1;
}
```

Then find:

```js
function showGenerateSidebar() {
  var html = HtmlService.createHtmlOutput(GENERATE_SIDEBAR_HTML_).setTitle('Generate event tabs');
  SpreadsheetApp.getUi().showSidebar(html);
}
```

Insert immediately **after** it:

```js
function showImportSidebar() {
  var html = HtmlService.createHtmlOutput(IMPORT_SIDEBAR_HTML_).setTitle('Import bracket draws');
  SpreadsheetApp.getUi().showSidebar(html);
}
```

**Check:** syntax check. The separator logic is the subtle part: in a generated
workbook the menu must read `… | Import bracket draws | Help`, with one
separator above the item and no stray divider.

### 9.6 The sidebar

Find the last two lines of the `GENERATE_SIDEBAR_HTML_` constant:

```js
  '});' +
  '</script>';
```

Insert immediately **after** them:

```js
var IMPORT_SIDEBAR_HTML_ =
  '<style>' +
  'body{font-family:Arial,sans-serif;font-size:13px;padding:4px 8px;}' +
  'label{display:block;margin-top:12px;font-weight:bold;}' +
  'button{margin-top:14px;padding:8px 14px;}' +
  '#drop{margin-top:6px;border:2px dashed #aaa;border-radius:4px;padding:14px;text-align:center;color:#555;font-weight:normal;}' +
  '#drop.dragging{border-color:#0b6b0b;background:#f3fbf3;}' +
  '#status{margin-top:12px;white-space:pre-wrap;font-size:12px;}' +
  '.err{color:#b00020;} .ok{color:#0b6b0b;}' +
  'table{border-collapse:collapse;margin-top:8px;width:100%;font-size:12px;}' +
  'td{padding:2px 4px;vertical-align:top;} td.bad{color:#b00020;}' +
  '.hint{font-weight:normal;color:#666;font-size:11px;}' +
  '</style>' +
  '<label>Bracket Generator text files <span class="hint">— one per category; drop several at once</span>' +
  '<input type="file" id="files" accept=".txt,text/plain" multiple>' +
  '<div id="drop">Drop .txt files here</div></label>' +
  '<div id="list"></div>' +
  '<label style="font-weight:normal"><input type="checkbox" id="replace"> Replace existing names</label>' +
  '<button id="go" disabled>Import</button>' +
  '<div id="status"></div>' +
  '<script>' +
  'var loaded=[], cats=[];' +
  'function esc(s){return String(s==null?"":s).replace(/[&<>"]/g,function(c){return {"&":"&amp;","<":"&lt;",">":"&gt;","\\"":"&quot;"}[c];});}' +
  'function render(res){' +
  '  cats=res.categories; var html="<table>";' +
  '  res.files.forEach(function(f){' +
  '    html+="<tr><td><b>"+esc(f.name)+"</b></td>";' +
  '    if(f.errors.length){ html+="<td class=bad colspan=2>"+esc(f.errors.join(" "))+"</td></tr>"; return; }' +
  '    html+="<td>"+esc(f.pairs)+" pairs · "+esc(f.sizes)+(f.seed?" · seed "+esc(f.seed):"")+"</td><td>";' +
  '    if(f.resolved){ html+="<b>"+esc(f.resolved)+"</b>"; }' +
  '    else {' +
  '      html+="<select class=assign data-file=\\""+esc(f.name)+"\\"><option value=\\"\\">pick a tab…</option>";' +
  '      cats.forEach(function(c){ html+="<option value=\\""+esc(c.key)+"\\">"+esc(c.key)+" ("+esc(c.teams)+" pairs)</option>"; });' +
  '      html+="</select>";' +
  '    }' +
  '    html+="</td></tr>";' +
  '  });' +
  '  document.getElementById("list").innerHTML=html+"</table>";' +
  '  document.getElementById("go").disabled = !res.files.length;' +
  '}' +
  'function readFiles(fileList){' +
  '  loaded=[]; var pending=fileList.length;' +
  '  if(!pending) return;' +
  '  Array.prototype.forEach.call(fileList,function(file){' +
  '    var r=new FileReader();' +
  '    var settle=function(){ if(--pending===0) google.script.run.withSuccessHandler(render).previewDraws(loaded); };' +
  '    r.onload=function(e){ loaded.push({name:file.name,text:String(e.target.result||"")}); settle(); };' +
  '    r.onerror=settle;' +
  '    r.readAsText(file);' +
  '  });' +
  '}' +
  'document.getElementById("files").addEventListener("change",function(){ readFiles(this.files); });' +
  'var drop=document.getElementById("drop");' +
  '["dragenter","dragover"].forEach(function(ev){ drop.addEventListener(ev,function(e){ e.preventDefault(); e.stopPropagation(); drop.classList.add("dragging"); }); });' +
  '["dragleave","dragend"].forEach(function(ev){ drop.addEventListener(ev,function(e){ e.preventDefault(); e.stopPropagation(); drop.classList.remove("dragging"); }); });' +
  'drop.addEventListener("drop",function(e){ e.preventDefault(); e.stopPropagation(); drop.classList.remove("dragging");' +
  '  if(e.dataTransfer && e.dataTransfer.files) readFiles(e.dataTransfer.files); });' +
  '["dragover","drop"].forEach(function(ev){ document.addEventListener(ev,function(e){ e.preventDefault(); }); });' +
  'document.getElementById("go").addEventListener("click",function(){' +
  '  var btn=this; btn.disabled=true;' +
  '  var status=document.getElementById("status"); status.className=""; status.textContent="Importing...";' +
  '  var assign={};' +
  '  Array.prototype.forEach.call(document.querySelectorAll("select.assign"),function(s){' +
  '    if(s.value) assign[s.getAttribute("data-file")]=s.value; });' +
  '  google.script.run' +
  '    .withSuccessHandler(function(res){' +
  '      btn.disabled=false;' +
  '      if(res.errors && res.errors.length){ status.className="err"; status.textContent="Rejected — nothing was written:\\n\\n"+res.errors.join("\\n"); }' +
  '      else { status.className="ok"; status.textContent=res.message; }' +
  '    })' +
  '    .withFailureHandler(function(err){ btn.disabled=false; status.className="err";' +
  '      status.textContent="Import stopped partway through: "+(err.message||err); })' +
  '    .importBracketDraws(loaded, { assign: assign, replace: document.getElementById("replace").checked });' +
  '});' +
  '</script>';
```

**Check:** syntax check, then extract the sidebar's own inline JS and check
that too:

```bash
node -e "const fs=require('fs');const s=fs.readFileSync('scripts/standard-generator.gs','utf8');const i=s.indexOf('var IMPORT_SIDEBAR_HTML_ =');const lit=s.slice(i+s.slice(i).indexOf('=')+1, i+s.slice(i).indexOf(\"</script>';\")+\"</script>'\".length);const html=eval(lit);fs.writeFileSync('import_sidebar_tmp.js',html.match(/<script>([\s\S]*)<\/script>/)[1]);" && node --check import_sidebar_tmp.js && rm import_sidebar_tmp.js && echo "SIDEBAR JS OK"
```

### 9.7 The mock needs `getValues`

`readPlanBlock_` and §9.2's emptiness check read ranges, and
`scripts/mock-apps-script.mjs` only implements the single-cell `getValue`. Add
the plural.

In `scripts/mock-apps-script.mjs`, find:

```js
    getValue() {
        const cell = this.sheet.cells.get(key(this.row, this.col));
        return cell && cell.value !== undefined ? cell.value : "";
    }
```

Insert immediately **after** it:

```js
    // Row-major, like the real API: a cell holding a formula reads back as its
    // formula text (the mock evaluates nothing), and an untouched cell as "".
    getValues() {
        const out = [];
        for (let r = 0; r < this.numRows; r++) {
            const row = [];
            for (let c = 0; c < this.numCols; c++) {
                const cell = this.sheet.cells.get(key(this.row + r, this.col + c));
                row.push(cell ? (cell.value !== undefined ? cell.value : (cell.formula ?? "")) : "");
            }
            out.push(row);
        }
        return out;
    }
```

**Check:** `node scripts/verify-sheet-generator.mjs` and
`node scripts/verify-standard-generator.mjs` both still pass — the method is
additive, so nothing existing should move.

### 9.8 The verify-script scenario

In `sage-tools-api/scripts/verify-standard-generator.mjs`, find:

```js
// -------------------------------------------------------------------- reject
```

Insert immediately **before** it:

```js
// -------------------------------------------------------------------- import
if (!only || only === "import") {
    // Builds a file byte-shaped like tools/bracket-generator.html's
    // exportAsText() output, so the parser is checked against the real format.
    const drawFile = (category, brackets, { seed = "BUNUTAN", drawNo = 1,
        event = "PICKLE FOR SIGHT TOURNAMENT" } = {}) => {
        const total = brackets.reduce((n, b) => n + b.length, 0);
        let out = `${event}\n${category.toUpperCase()}\nDrawn 9/23/2026, 7:41:02 PM\n` +
            `${brackets.length} brackets, ${total} pairs\n${"=".repeat(44)}\n\n`;
        brackets.forEach((pairs, i) => {
            out += `BRACKET ${String.fromCharCode(65 + i)}\n${"-".repeat(24)}\n`;
            pairs.forEach((p, k) => { out += `${k + 1}. ${p}\n`; });
            out += "\n";
        });
        out += `DRAW VERIFICATION\n${"-".repeat(24)}\n`;
        out += `Seed:   ${seed}  (typed by the organiser)\nDraw:   #${drawNo}\n`;
        out += `Method: SHA-256(seed + "|" + pair), sorted ascending, dealt round-robin\n\n`;
        out += "Fingerprints, in draw order:\n";
        brackets.flat().forEach((p, i) => { out += `  ${String(i).padStart(8, "0")}...  ${p}\n`; });
        out += "\nTO CHECK THIS DRAW YOURSELF\nSearch the web for \"SHA-256 calculator\"\n";
        return out;
    };
    const pairs = (n, tag) => Array.from({ length: n }, (_, i) => `${tag}${i + 1}A / ${tag}${i + 1}B`);
    const file = (name, category, brackets, opts) => ({ name, text: drawFile(category, brackets, opts) });

    const out = run("import into PFS ANNEX", fixture("pfs-annex-plan.csv"), {
        facility: "PCPH Annex", courtLabels: "5,6,7,8,9",
        selected: ["LIMD", "LIXD", "HIWD", "HIMD", "HIXD"], waves: {},
    });
    if (out) {
        const { cell, sandbox } = out;
        const himd = [pairs(5, "P"), pairs(4, "Q"), pairs(4, "R")];   // 13 pairs, 5-4-4

        // ---- §2 parsing, on its own
        const parsed = sandbox.parseDrawText_(drawFile("High Intermediate Men's Doubles", himd), "himd.txt");
        // exportAsText() upper-cases the category line, which is why §4
        // compares case-insensitively.
        check("parse: category", parsed.category, "HIGH INTERMEDIATE MEN'S DOUBLES");
        check("parse: sizes", sandbox.drawSizes_(parsed).join("-"), "5-4-4");
        check("parse: seed and draw", `${parsed.seed} #${parsed.drawNumber}`, "BUNUTAN #1");
        check("parse: no errors", parsed.errors.length, 0);
        check("parse: junk file reports an error", sandbox.parseDrawText_("hello\nworld", "x.txt").errors.length, 1);

        // ---- §5 splitting
        const split = (s) => { const r = sandbox.splitPairNames_(s); return `${r.one}|${r.two}|${r.split}`; };
        check("split: slash", split("Juan Dela Cruz / Maria Santos"), "Juan Dela Cruz|Maria Santos|true");
        check("split: ampersand", split("Juan & Maria"), "Juan|Maria|true");
        check("split: and", split("Juan and Maria"), "Juan|Maria|true");
        check("split: comma", split("Dela Cruz, Santos"), "Dela Cruz|Santos|true");
        check("split: earliest separator wins", split("A / B / C"), "A|B / C|true");
        check("split: none", split("Team Thunder"), "Team Thunder||false");

        // ---- a good import, matched by display name (§4 rule 2)
        const res = sandbox.importBracketDraws([file("himd.txt", "High Intermediate Men's Doubles", himd)], {});
        check("import accepted", !!res.message, true);
        check("HIMD AD5:AD6 = first pair", `${cell("HIMD", 5, 30)}|${cell("HIMD", 6, 30)}`, "P1A|P1B");
        // Pair i occupies rows 5 + 2(i-1) and 6 + 2(i-1), so bracket 2's first
        // pair — pair 6 of 13 — is on rows 15-16.
        check("HIMD AD15 = bracket 2's first pair", cell("HIMD", 15, 30), "Q1A");
        check("HIMD AD30 = last name", cell("HIMD", 30, 30), "R4B");
        check("HIMD AI5:AI17 = codes in order",
            [5, 6, 17].map((r) => cell("HIMD", r, 35)).join(","), "HIMD_1,HIMD_2,HIMD_13");
        check("HIMD AB2 = seed", cell("HIMD", 2, 28), "Draw seed: BUNUTAN (typed by the organiser)");
        check("HIMD AB3 = draw", cell("HIMD", 3, 28), "Draw #1, drawn 9/23/2026, 7:41:02 PM");
        check("HIMD AB4 names the file", String(cell("HIMD", 4, 28)).endsWith("from himd.txt"), true);
        check("the qualifier draw is untouched", cell("HIMD", 5, 39), "-");

        // ---- §6 refusals
        const bad = (files, options) => (sandbox.importBracketDraws(files, options).errors ?? []).join("\n");
        check("refuses a second import", bad([file("himd.txt", "HIMD", himd)], {}).includes("Replace existing names"), true);
        check("accepts with replace",
            !!sandbox.importBracketDraws([file("himd.txt", "HIMD", himd)], { replace: true }).message, true);
        check("refuses the wrong pair count",
            bad([file("a.txt", "LIMD", [pairs(6, "S"), pairs(6, "T")])], {}).includes("built for 19"), true);
        check("refuses the wrong bracket sizes",
            bad([file("b.txt", "HIMD", [pairs(5, "S"), pairs(5, "T"), pairs(3, "U")])], { replace: true })
                .includes('5-5-3, but tab "HIMD" was built for 5-4-4'), true);
        check("refuses two files for one tab",
            bad([file("c.txt", "HIMD", himd), file("d.txt", "HIMD", himd)], { replace: true }).includes("both resolve"), true);
        check("refuses a duplicate pair",
            bad([file("e.txt", "HIMD", [["X / Y", "X / Y", "C / D", "E / F", "G / H"], pairs(4, "Q"), pairs(4, "R")])],
                { replace: true }).includes("appears twice"), true);
        check("an unmatched category is skipped, not failed",
            bad([file("f.txt", "Something Else", himd)], { replace: true }).includes("No file was assigned"), true);
        check("an assignment overrides the category line", (() => {
            const r = sandbox.importBracketDraws([file("g.txt", "Something Else", himd)],
                { assign: { "g.txt": "HIMD" }, replace: true });
            return !!r.message && cell("HIMD", 5, 30) === "P1A";
        })(), true);

        // ---- §5's warning reaches the operator, and the pair is still written
        const warned = sandbox.importBracketDraws(
            [file("h.txt", "HIXD", [["Team Thunder", "A / B", "C / D"], pairs(3, "Z")])], { replace: true });
        check("a no-separator pair warns", String(warned.message).includes("no name separator"), true);
        check("and is written whole", cell("HIXD", 5, 30), "Team Thunder");
        check("with a blank second row", cell("HIXD", 6, 30), "");
    }
}
```

**Check:**

```bash
node scripts/verify-standard-generator.mjs --only=import   # every line OK
node scripts/verify-standard-generator.mjs                 # the other five scenarios still pass
```

`HIXD` is 6 pairs over 3-3, so the `h.txt` file above is a correct draw for it.

### 9.9 Verify before calling it done

```bash
cd sage-tools-api
node -e "new Function(require('fs').readFileSync('scripts/standard-generator.gs','utf8'))" && echo "GS OK"
node scripts/verify-standard-generator.mjs
node scripts/verify-sheet-generator.mjs
```

The last one matters because §9.7 touches the shared mock: the dual-meet
harness must be unaffected.

Then check the menu, which no harness covers — a fresh copy offers both items,
a generated workbook only the import, and neither gets a stray separator:

```bash
node -e "
const fs=require('fs'),vm=require('vm');
const items=[];const menu={addItem:(l)=>{items.push(l);return menu;},addSeparator:()=>{items.push('---');return menu;}};
const props={};
const ctx={Logger:{log(){}},SpreadsheetApp:{getActive:()=>({getId:()=>'WB'})},
  PropertiesService:{getScriptProperties:()=>({getProperty:(k)=>props[k]||null})}};
vm.createContext(ctx);vm.runInContext(fs.readFileSync('scripts/standard-generator.gs','utf8'),ctx);
ctx.addGeneratorMenuItems_(menu,{count:2});console.log('fresh copy :',items.join(' | '));
items.length=0;props['TABS_GENERATED_FOR']='WB';
ctx.addGeneratorMenuItems_(menu,{count:2});console.log('generated  :',items.join(' | '));"
```

Expected exactly:

```
fresh copy : --- | Generate event tabs | Import bracket draws
generated  : --- | Import bracket draws
```

All three must pass. The mock does **not** evaluate formulas, so it proves the
values and the geometry, not that the tab recalculates. Closing that gap needs
a real workbook:

1. Copy the master and generate the PFS Annex workbook
   (`SAGE → Generate event tabs`).
2. Draw `HIMD`'s 13 pairs over 3 brackets in the Bracket Generator, export the
   text file, and import it.
3. Confirm each group row's `B` column now shows names rather than codes —
   that is the `AE`/`AI` link resolving, which the mock cannot show.

Report honestly if step 3 fails: it means `AI`'s values are not matching the
`AE` link formulas, and the fix belongs in this spec, not in a workaround.

## 10. Acceptance

- A `HIMD` draw of 13 pairs in 3 brackets (5-4-4) imports into the PFS Annex
  workbook: `AD5:AD30` holds 26 names in bracket order, `AI5:AI17` holds
  `HIMD_1 … HIMD_13`, `AB2:AB4` records seed, draw and file, and **every group
  row's `B` shows its pair's names**.
- The same file imported twice is refused; accepted with **Replace**.
- A draw of 12 pairs, or of 4 brackets, or of 5-5-3, is refused by name with
  both shapes reported, and the workbook is untouched.
- Files whose category reads `HIMD`, `High Intermediate Men's Doubles` and
  `Something Else` behave per §4 — the third asks, and importing it after
  picking `HIMD` from the dropdown works.
- Five files import in one pass, each into its own tab.
- A pair line with no separator lands whole on its first row, leaves the
  second blank, and warns.
- Nothing else changes: no formats, no `AM` draw, no playoff rows, no other
  tab.
- `SAGE → Import bracket draws` is present in a freshly generated workbook,
  where `Generate event tabs` has removed itself.

## 11. Built: a menu route into the tool

`SAGE → Open Bracket Generator` opens the tool with `?event=` from `Title!B6`
and `?category=` from the active tab's own label, in a dialog copying
`showScoresheetLink`'s pattern — Apps Script cannot open a URL from server
code, so the navigation comes from a click on an anchor. `?category=` is
**not** persisted to `localStorage`, unlike `?event=`: a category resurrected
on a later visit is how someone draws the wrong one.

Where it lives:

| File | Change |
| --- | --- |
| `sage-tools-api/scripts/sheets-sync.gs` | `BRACKET_GENERATOR_URL`, the menu item in `addSyncMenuItems_`, and `showBracketGeneratorLink` |
| `sage-match-control.github.io/tools/bracket-generator.html` | `initCategoryFromQuery`, beside `initEventName` |

The item is in `sheets-sync.gs`, so it appears in **every** workbook, dual
meets included. That is deliberate: the tool is still the way a dual meet's
organiser draws anything they want bracket cards for, and gating the item on
which master it is in would mean `sheets-sync.gs` having to know, which it
otherwise never does.

The original reasoning for parking it stands and is worth keeping: a standard
tournament's roster comes from registration, not from the workbook, so the
prefill saves typing a category name the tool needs anyway. What it buys this
spec is that the file's category line then matches its tab exactly, so §4
rarely reaches its dropdown.

## 12. Built: a dual meet's STEP 3

A dual meet gets no benefit from the import itself: its draw is a per-club
roster blind, not a bracket draw, and the tool's bracket cards are the wrong
artifact for it. So its `STEP 3` (`AG`/`AV`, two independent columns) is drawn
in the sheet instead.

`SAGE → Shuffle roster codes`, in `sage-tools-api/scripts/sheet-generator.gs`,
reads `STEP 2` and writes a shuffled permutation straight into `STEP 3` on the
active category tab — no browser, no clipboard, no paste errors. Each club's
column is drawn independently; they are separate rosters that never mix. It
refuses a tab with no roster scaffold, and refuses a `STEP 3` that already
holds anything unless the operator confirms a replace, since reshuffling a
live workbook re-points every pair.

The scaffold is located by its own `CODES` header rather than by recomputing
the generator's geometry, so a tab edited since generation still works. Like
`Import bracket draws` in the Standard master, the item sits **outside** the
`PROP_TABS_GENERATED_FOR` guard: the roster is filled after generation, which
is exactly when the shuffle is needed.

What it gives up is an audience: an in-sheet shuffle produces no artifact and
nobody watches it land, which is the whole point of the tool's ~3s shuffle
(bracket generator spec §4). That trade is right for a roster blind, which is
bookkeeping, and wrong for a bracket draw, which is a moment in a room — which
is why this shuffles a dual meet's roster and nothing else. In particular it
is never offered in the Standard master, where filling `STEP 3` without a draw
file would defeat §8.1.

Rejected along the way, and worth not re-proposing:

- **A shuffled-codes output mode in the tool** — a second export shape, plus a
  club dimension the tool deliberately lacks (bracket generator spec §12), to
  move codes through the clipboard for the rarer event shape, when the sheet
  can write them itself.
- **A club-aware bracket card** — teaching the tool about clubs forks the one
  thing both event shapes currently share.
- **Prefilling the tool's pairs from `STEP 1`** — the timing works (names land
  before the shuffle), but the tool would hand back bracket cards, still not
  the column `STEP 3` wants.

## 13. Out of scope

- The **qualifier draw** (`AM`, master spec §7.6) — §8.3.
- The Bracket Generator's **image** export, which stays a human artifact.
- Any change to the Bracket Generator beyond §11's `?category=` prefill. The
  import itself needs only the file the tool already writes.
- The Dual Meet Master's **rosters and brackets**. §12 adds a roster-code
  shuffle there and nothing else; no dual-meet tab is ever filled from a draw
  file.
- Re-verifying the draw's fingerprints (§8.2).
- Filling `AI` on its own, without names from a draw file (§8.1).

---

## 14. Divergences

§9 was applied as written; the departures below are all in §11 and §12, which
were prose sketches rather than anchored steps.

**§11 reads `B1` with an `A1` fallback, not `B1` alone.** §11 named `B1` as
the category label, which is right for the Standard master (`writeChrome_`
puts the key in `A1` and the display name in `B1`) but wrong for the dual-meet
master, which writes the uppercased display value to `A1` and never touches
`B1`. Since the item ships in both, `showBracketGeneratorLink` reads `B1` and
falls back to `A1`, which covers each master without having to know which one
it is in.

**§11's dialog shows both values before the click.** Not in the sketch. It is
what makes a wrong active tab visible: on a non-category tab the category
simply reads as something the operator can see is wrong, so no tab-type
refusal is needed.

**§12 guards on `getMaxColumns()`.** `findRosterScaffold_` reads column `AU`
for club B, and a narrow tab — `SCHEDULE`, `Timeline`, any readout — has no
such column, so `getRange` throws instead of returning blanks. Without the
guard the "refuse on a non-category tab" rule crashed rather than refusing.
Caught by the verify scenario, not by inspection.

**§12 shuffles both clubs in one action.** The sketch said "the active
category tab" without saying how many of its columns. Both, independently: a
tab carries two rosters and the operator thinks in tabs, not clubs.

**Menu placement follows §9.5's pattern in both generators.** `sheet-generator.gs`'s
`addGeneratorMenuItems_` got the same guard-move `standard-generator.gs` got,
so the shuffle survives generation.

Harness coverage added with them: `verify-sheet-generator.mjs` gains a
`shuffle` scenario (13 checks). §11's tool half has no harness — it was
verified in a browser against a local static server: `?event=` and
`?category=` both prefill, and after a reload with no query string the event
name returns from `localStorage` while the category comes back empty.
