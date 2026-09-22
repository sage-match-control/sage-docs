# Spec — Bracket draw name import

Fill a generated workbook's rosters by uploading the Bracket Generator's
exported **text files**, one per category, instead of typing every pair into
each category tab's `STEP 1 · NAMES` column and then hand-shuffling
`STEP 3 · RANDOMIZED CODES`.

> **Status: not built.** Nothing in this document exists yet.

| | |
| --- | --- |
| Where the code goes | `sage-tools-api/scripts/standard-generator.gs` — a second menu item and sidebar |
| How it ships | Bound Apps Script in the SAGE Standard Tournament Master — paste, not deploy |
| New infra | none |
| New credentials | none |
| `sage-tools-api` change | none — no version bump, no Cloud Run deploy |
| Reads | the `.txt` files [`tools/bracket-generator.html`](../../features/bracket-generator.md) already exports |
| Writes | each category tab's `AD` (names), `AI` (codes) and `AB` (draw provenance) — nothing else |

**Read first:**
[`standard-tournament-master-spec.md`](../implemented/standard-tournament-master-spec.md)
§7.2 (group pairs and bracket order), §7.3 (the code ladder) and §7.6 (the two
scaffolds), and
[`bracket-generator-workbook-handoff-spec.md`](bracket-generator-workbook-handoff-spec.md),
which asked for a route in the other direction — workbook out to the tool —
and deferred the shuffled-codes half until a standard tournament existed. This
is that half, arriving as an import rather than an export.

---

## 1. The gap this closes

Preparing one standard-tournament venue-day today means, per category:

1. Draw the brackets in the Bracket Generator, download the image and the
   text file.
2. Read the text file and type each pair's two players into `AD`, two rows per
   pair, in bracket order.
3. Paste the codes from `AH` into `AI` in a shuffled order, by hand.

Step 2 is transcription — the names are already in a file — and step 3 is
busywork that the draw has already done: the Bracket Generator's draw is
verifiable (seed, SHA-256 fingerprint per pair, dealt round-robin), so the
pair-to-bracket assignment it produces is the randomisation `STEP 3` exists
to provide. Doing it again by hand adds nothing and can silently undo it: an
unshuffled `STEP 3` maps every pair to its own roster slot.

So the import writes both columns, and the draw file becomes the record of
how the assignment was made.

## 2. Input — the Bracket Generator's text export

Exactly what `exportAsText()` writes today, unchanged. One file per category,
named `<event>-<category>-brackets.txt`:

```
PICKLE FOR SIGHT TOURNAMENT
HIGH INTERMEDIATE MEN'S DOUBLES
Drawn 9/23/2026, 7:41:02 PM
3 brackets, 13 pairs
============================================

BRACKET A
------------------------
1. Juan Dela Cruz / Maria Santos
2. …
5. …

BRACKET B
------------------------
1. …

DRAW VERIFICATION
------------------------
Seed:   bunutan  (typed by the organiser)
Draw:   #1
Method: SHA-256(seed + "|" + pair), sorted ascending, dealt round-robin
…
```

What the parser reads, and nothing else:

| Line | Read as |
| --- | --- |
| line 1, when line 2 is not `Drawn …` | event name (ignored except in the log) |
| the line before `Drawn …` | **category**, as the operator typed it in the tool (§4) |
| `^BRACKET (\S+)$` | a bracket, in file order; its label (`A`, `B`, …) is positional only |
| `^\s*(\d+)\.\s+(.+)$` under a bracket | one **pair**, in file order (§5) |
| `(no pairs assigned)` | an empty bracket — a validation failure (§6) |
| `^Seed:\s+(.+?)\s+\((.+)\)$` | the seed and where it came from (§3.3) |
| `^Draw:\s+#(\d+)$` | the draw number |

Everything after `TO CHECK THIS DRAW YOURSELF` is ignored. A file whose
category line, brackets or pairs cannot be found is rejected by name; the
fingerprint list is not re-verified (the file is its own audit trail, and
re-checking it would mean hashing in Apps Script for no gain).

## 3. What it writes

Per category, into that category's tab only.

### 3.1 `AD` — STEP 1 · NAMES

Two rows per pair, from `AD5`, in **import order**: bracket A's pairs first,
then bracket B's, and so on, each pair's first player on the pair's first row
and its second player on the second (§5). This is the same order the tab's
group pairs run in (master spec §7.2), so bracket A's pairs take codes
`<KEY>_1 …`, which is what puts them in bracket 1 on the tab, in
`STANDINGSCSV!G`, and in the score grids.

### 3.2 `AI` — STEP 3 · RANDOMIZED CODES

`<KEY>_1 … <KEY>_<teams>` in order, one row per pair from `AI5`, as plain
values.

That is an identity mapping on purpose: the shuffle happened in the draw. The
`AE` link formulas the generator already wrote resolve each roster row to its
code, so `B` on every group row picks up the names with no further step.

**This is the one place the import departs from the master spec's §7.6**,
which ships `AI` blank and says pre-seeding it in order defeats the blinding.
That reasoning holds for a *hand* roster, where roster order is entry order,
usually alphabetical or as-registered. It does not hold here: the rows arrive
in an order a seeded, published draw produced. The import must therefore
refuse to write `AI` when it did not also write `AD` from a draw file (§6), so
the two columns can never disagree about where the randomisation came from.

### 3.3 `AB` — the draw's provenance

`AB2`, `AB3`, `AB4` on the category tab (the notes column beside the roster,
which nothing reads):

```
AB2   Draw seed: bunutan (typed by the organiser)
AB3   Draw #1, drawn 9/23/2026, 7:41:02 PM
AB4   Imported <timestamp> from hiwd-brackets.txt
```

Written as plain text. It is what lets anyone holding the exported file
confirm the workbook was filled from *that* draw, months later.

## 4. Matching a file to a category tab

The tool's category is free text typed by the operator; the workbook's is a
key (`HIMD`) with a display name in `Variables!J`. Resolve in this order:

1. the file's category, trimmed and upper-cased, equals a generated tab's
   name — the key itself;
2. it equals a generated category's display name (`Variables!J`), compared
   upper-cased and with runs of whitespace collapsed;
3. otherwise **ask**: the sidebar shows a dropdown of this workbook's
   category tabs beside that file, defaulting to unset.

A file the operator leaves unset is skipped, not guessed. Two files resolving
to the same tab is a validation failure (§6).

## 5. Splitting a pair into two players

The tool takes one pair per line as free text; the workbook needs two rows.
Split on the **first** occurrence of any of these, then trim both halves:

| Separator | Example |
| --- | --- |
| `/` | `Juan Dela Cruz / Maria Santos` |
| `&` | `Juan & Maria` |
| `+` | `Juan + Maria` |
| `and`, as a whole word | `Juan and Maria` |
| `,` | `Dela Cruz, Santos` |

- No separator: the whole string goes on the pair's first row, the second row
  is left blank, and the import **warns**, naming the category and the pair.
  Some events enter a team name rather than two players, so this is a warning,
  not a failure.
- More than one separator: only the first splits. `A / B / C` becomes
  `A` and `B / C`, and warns the same way.

The import never reorders the two players and never reformats a name.

## 6. Validation

Everything below runs across **every** uploaded file before anything is
written, and every failure is reported at once — the master spec's §9 rule.

| Rule | Reason |
| --- | --- |
| the file parses: a category line, at least one `BRACKET`, at least one pair | §2 |
| its category resolves to exactly one generated tab (§4) | — |
| no two files resolve to the same tab | the second would overwrite the first |
| the tab's `AD5:AD` is empty, unless **Replace existing names** is ticked | a filled roster means the event is under way |
| total pairs equals that category's `teams` (`Variables!K`) | a draw of a different field |
| bracket count equals its bracket count (`Variables!L`, after the master spec §5.6 reduction) | — |
| per-bracket sizes equal `groupSizes(teams, brackets)` — the sizes the tab was built for | the tool deals round-robin, largest first, so a correct draw matches exactly |
| no bracket is empty, and no pair line is blank | §2 |
| the same pair text does not appear twice in one file | a duplicated pair means a mis-paste into the tool |

A failure names the file and the category and writes nothing anywhere. A
per-bracket size mismatch reports both shapes (`file 5-4-4, tab 5-5-3`),
since the usual cause is a different bracket count typed into the tool.

## 7. The sidebar

`SAGE → Import bracket draws`, beside `Generate event tabs`:

- A drop zone and a file picker taking **several `.txt` files at once**, read
  in the browser with `FileReader` exactly as the generator's CSV box does, so
  nothing is uploaded anywhere.
- One row per file: filename, the category it resolved to (or the dropdown of
  §4), pairs, bracket sizes, and the seed. A row that fails §6 shows its
  reason in place.
- A **Replace existing names** tick, off by default.
- **Import** writes every resolvable file, then reports per category: names
  written, codes written, and any §5 warnings.

The item stays in the menu after a successful import — unlike
`Generate event tabs`, this is safely repeatable with `Replace` on, which is
how a re-draw gets applied.

## 8. Implementation notes

- `parseDrawText_(text)` is **pure** — string in, `{ eventName, category,
  brackets: [[pair, …], …], seed, seedSource, drawNumber, drawnAt }` out — and
  is where every §2 and §5 rule lives. `resolveCategory_` and the §6 checks
  are pure too, taking the workbook's plan block as data.
- The writes are three `setValues` calls per category (`AD`, `AI`, `AB`), so a
  nine-category venue-day is 27 calls. No formatting is touched: the generator
  already laid the scaffold out.
- `scripts/verify-standard-generator.mjs` gains a scenario: generate the PFS
  Annex workbook, import a text file built from `exportAsText()`'s real
  output for each of its five categories, then assert `AD`, `AI` and `AB`, the
  §5 split on `/`, `&` and a no-separator pair, and each §6 refusal.

## 9. Acceptance

- A `HIMD` draw of 13 pairs in 3 brackets (5-4-4) imports into the PFS Annex
  workbook: `AD5:AD30` holds 26 names in bracket order, `AI5:AI17` holds
  `HIMD_1 … HIMD_13`, and every group row's `B` shows its pair's names.
- The same file imported twice is refused, and accepted with **Replace**.
- A draw of 12 pairs, or of 4 brackets, is refused by name with both shapes
  reported, and the workbook is untouched.
- A file whose category reads `HIMD`, one whose category reads
  `High Intermediate Men's Doubles`, and one whose category reads
  `Something Else` all behave per §4 — the third asks.
- Five files import in one pass, each into its own tab.
- A pair line with no separator lands whole on its first row and warns.
- Nothing else on the tab changes: no formats, no `AM` draw, no playoff rows.

## 10. Out of scope

- The **qualifier draw** (`AM`, master spec §7.6). It is a second draw, made
  after the round robin, and has no file to import from yet.
- The Bracket Generator's **image** export, which stays a human artifact.
- Any change to the Bracket Generator itself, including
  `bracket-generator-workbook-handoff-spec.md`'s menu route *into* the tool.
  The two are independent: this spec needs only the file the tool already
  writes.
- The **Dual Meet Master**. A dual meet's rosters are per club and its draw is
  not a bracket draw.
- Re-verifying the draw's fingerprints (§2).

---

## 11. Divergences

*(None — nothing here is built. Record departures when it is.)*
