# Spec — Bracket Generator × scoring workbook handoff

Connect `tools/bracket-generator.html` to the scoring workbooks that need a
draw, in the outbound direction: a **SAGE menu item** that deep-links into the
tool with the event and category already filled in. Plus the decision about
how a dual meet's roster blind gets done, which used to be the bulk of this
document (§4).

> **Status: not implemented, and rescoped.** Nothing here has been built.
>
> It was written while it still waited on a standard-tournament generator, and
> most of it was about giving the tool a second output shape so a dual meet
> could paste its `STEP 3` codes. Both premises have moved:
>
> - the standard tournament's **inbound** half — draw files going back into the
>   workbook — is its own spec,
>   [`bracket-draw-name-import-spec.md`](bracket-draw-name-import-spec.md),
>   and needs nothing new from the tool;
> - the dual meet's `STEP 3` is answered in the sheet instead of in the tool
>   (§4), which was already this spec's own competing design.
>
> What is left is the menu route (§2, §3) — small, useful to both formats, and
> buildable now — and §4's decision. The designs that were considered and
> dropped are kept in §5, so they don't come back unexamined.

| File | Repo | Change |
| --- | --- | --- |
| `scripts/sheets-sync.gs` | `sage-tools-api` | the bracket menu item + its dialog (§2), and §4's shuffle item |
| `tools/bracket-generator.html` | `sage-match-control.github.io` | `?category=` (§3) |
| `docs/features/bracket-generator.md` | `sage-docs` | the menu route, once it exists |
| `docs/technical/bracket-generator.md` | `sage-docs` | same |

**No backend change.** No endpoint, no auth, no `event-data` read, no
`package.json` bump, no Cloud Run deploy. `sheets-sync.gs` ships by being
pasted into a workbook's own script project, so changing it is not a deploy
either.

---

## 1. Where a draw meets a workbook

Both masters' category tabs carry a three-step roster scaffold. A dual meet
has one per club (`writeRosterScaffold_`, `sheet-generator.gs:1826`); a
standard tournament has one (standard master spec §7.6):

| Step | Dual meet (club A / club B) | Standard | State on a fresh workbook |
| --- | --- | --- | --- |
| **STEP 1 · NAMES** | `AB` / `AQ` | `AD` | blank — pair names, 2 rows per pair |
| **STEP 2 · CODES** | `AF` / `AU` | `AH` | pre-filled `…_1` … `…_<teams>` |
| **STEP 3 · RANDOMIZED** | `AG` / `AV` | `AI` | **blank on purpose** — the codes, shuffled |

The blankness is deliberate and load-bearing. From the generator's own comment
(`sheet-generator.gs:1869`):

> STEP 3 ("RANDOMIZED") is left BLANK … It used to be pre-seeded with the codes
> in roster order, which was wrong twice over: it made a step that hasn't been
> done yet look done, and an unshuffled STEP 3 is not a no-op — it maps every
> pair to its own roster slot, quietly defeating the blinding the shuffle
> exists to provide.

Who fills STEP 3 now differs by format, and that is the whole shape of this
spec:

- **Standard tournament** — the Bracket Generator's draw does it. Its output
  (pairs grouped under lettered brackets) is exactly the artifact a standard
  category needs, and the import spec reads the exported text file into
  STEP 1 and STEP 3 together.
- **Dual meet** — nothing does. The operator shuffles the codes by hand, or
  with a throwaway `SORT(…, RANDARRAY(…))` off to one side. The tool cannot
  help: its output is bracket cards, and a dual meet wants a column of codes.

### 1.1 Bracket membership is positional, not labelled

A dual meet's brackets are a subdivision *within each club*, derived from where
a code lands rather than from any bracket column: with `n = teamsA / brackets`,
codes `_1`…`_n` are bracket 1 and `_n+1`…`_2n` are bracket 2.

The generator validates that shape hard (`sheet-generator.gs:375-388`):

- `brackets` must be **1 or 2** — "a club playoff needs exactly two bracket
  winners to pair off"
- `teams_a` must equal `teams_b` — "bracket blocks must be square"
- `teams_a` must divide evenly by `brackets`
- `n` must land in 2..12

None of that matches the tool's own model, which deals an arbitrary pair count
into an arbitrary bracket count and knows nothing about clubs (bracket
generator spec §12). The two models are compatible in *outcome* — a shuffle is
a shuffle — but not in *shape*. That mismatch is why §4 stays in the sheet.

---

## 2. The menu item

### 2.1 Where it goes — and where it must not

Add it to **`addSyncMenuItems_` in `sheets-sync.gs`**, not to a generator's own
menu builder.

`addGeneratorMenuItems_` removes itself once the workbook has been generated
(`sheet-generator.gs:750`, `standard-generator.gs`, both guarding on
`PROP_TABS_GENERATED_FOR`), because `Generate event tabs` can only error after
that. But roster pasting — and therefore the draw — happens *after*
generation. An item placed there would disappear at exactly the moment it
becomes useful.

`addSyncMenuItems_` persists for the life of the workbook, and every workbook
carries `sheets-sync.gs`, so the item shows up in both masters, in a copy, and
before and after generation alike.

Contributing from one builder also sidesteps the hazard the shared block warns
about — `onOpen` is duplicated verbatim across all three `.gs` files with a
"Change one, change all three" note. This adds nothing to that block.

### 2.2 Ungated, unlike its neighbour

`Generate Scoresheets` sits behind `readSyncConfig_()` because it deep-links a
day key and facility name that only exist once sync is set up. A draw needs
neither. The bracket item is available in any workbook that carries the script,
configured or not — including a master, and including a copy on the day the
rosters arrive but before sync is wired.

### 2.3 The dialog

Copy the `showScoresheetLink` pattern verbatim (`sheets-sync.gs`), for the
reason its own docstring gives:

> Apps Script can't open a URL from server-side code, and a `window.open()`
> fired on dialog load is blocked as a popup inside the sandboxed iframe — so
> the navigation has to come from a real user click on an anchor, which is what
> this dialog provides.

Same shape: a small modal, the prefilled values shown as a `<dl>` so the
operator can see what they're about to get, one green anchor with
`target="_blank" rel="noopener"` and `onclick="google.script.host.close()"`.
Makes no HTTP request and needs no additional OAuth scope.

Add `BRACKET_GENERATOR_URL` beside `SCORESHEET_GENERATOR_URL`, pointing at
`https://sage-match-control.github.io/tools/bracket-generator.html`.

### 2.4 What it prefills

| Parameter | Source | Note |
| --- | --- | --- |
| `?event=` | `Title!B6` | the event title, which both generators write there (`sheet-generator.gs:2465`, `standard-generator.gs`'s `writeTitleTab_`) |
| `?category=` | the active category tab's **display label** — see below | uppercased |

**The display label, not the tab name.** Tabs are named by the raw category
key — `LIWD`, matching `^[A-Z0-9]{2,8}$` — and the raw key is the wrong thing
to print on a bracket card handed to players. Which cell holds the label
differs by master:

| Master | Label cell | What the other cell holds |
| --- | --- | --- |
| Dual Meet | `A1` — the label, uppercased (`sheet-generator.gs:1241`) | no key cell; the tab name is the key |
| Standard Tournament | `B1` — `=FILTER(Variables!J:J, Variables!I:I=A1)`, the plan's `value` (standard master spec §7.1) | `A1` is the raw key |

Read `B1` when it holds a non-empty value different from `A1`, else `A1`. That
covers both masters without the script knowing which one it is in, and a
standard tab whose `B1` hasn't resolved degrades to the key rather than sending
an empty parameter.

**When the active sheet isn't a category tab:** both masters name category tabs
by the raw key, so "looks like a category tab" is testable — the sheet name
matches `^[A-Z0-9]{2,8}$` and appears in `Variables`' plan block. Prefill from
the active sheet when it passes that test, omit `?category=` otherwise. A
picker is the alternative if operators turn out to run the item from
`SCHEDULE`.

### 2.5 Why it is worth the ~40 lines

Saving two fields of typing is not the point. The
[import spec](bracket-draw-name-import-spec.md) matches each draw file to a
category tab **by the category string the operator typed into the tool** (its
§4). Typed freehand, that string is `HIMD` one day and `Hi Int Men's` the
next, and the import falls back to asking. Prefilled from the tab, it matches
exactly, every time — and the exported filenames come out consistent too. The
menu route is what makes the import's happy path the common one.

---

## 3. Tool-side — the `?category=` parameter

Mirrors `?event=` (bracket generator spec §3.7) and costs about four lines.

**With one deliberate difference: `?category=` must not persist.** `?event=` is
stored in `localStorage` and survives to the next visit because one operator
draws many categories for one event in a sitting (§3.6 there). Category is the
opposite, and that same section is explicit about why:

> Category and pairs are **not** persisted. They change every draw, and a
> prefilled pair list is a live hazard: an operator who does not notice it draws
> the wrong category's brackets.

A category arriving in a fresh deep link is fine — it came from the tab the
operator is looking at. A category resurrected from storage on an unrelated
later visit is the exact hazard above. So: read it, fill the field, never write
it back.

---

## 4. A dual meet's STEP 3: shuffle in the sheet

**Decided: the shuffle belongs in the workbook, not in the tool.**

A `SAGE → Shuffle roster codes` item writes that tab's STEP 3 column(s)
directly: read `STEP 2`, shuffle, write. No browser, no clipboard, no round
trip, no paste errors, and the operator never leaves the workbook.

What it gives up is the reason the Bracket Generator is shaped the way it is.
From the bracket generator spec §4:

> this page is a *stage*, not a form … the ~3s shuffle exists so a room can
> watch a draw land

An in-sheet shuffle is invisible: no artifact to post, nobody watching it
happen. That trade is right here and wrong for a bracket draw, and the
difference is what the two are *for*:

| | Bracket draw (standard) | Roster blind (dual meet) |
| --- | --- | --- |
| Who is watching | players, in the room | nobody — it is bookkeeping |
| What it decides | which bracket a pair plays in | which code a pair is recorded under |
| Needs an artifact | yes: a card to post and a file to re-check | no |
| Where it happens | the tool, then imported | in the sheet |

Two dual meets exist in the system's history and both hand-shuffled without
complaint, so this is a convenience, not a hole to rush. Sketch:

- One item in `addSyncMenuItems_`, on the same terms as §2.1 and §2.2.
- Acts on the **active category tab**, and refuses with a clear message
  anywhere else.
- Refuses when STEP 3 already holds anything, unless the operator confirms a
  replace — a filled STEP 3 mid-event means the workbook is live, and
  reshuffling it would re-point every pair.
- A dual-meet tab has two columns (`AG`, `AV`) and they are independent draws;
  whether one click does both or asks which club is the one thing left open.
- A standard tab has one (`AI`), but its normal route is the import, so the
  item is the fallback for a category drawn on paper.
- No audit trail, by design: an unwitnessed shuffle has nothing to prove. A
  draw that needs proving goes through the tool.

---

## 5. Decisions

### 5.1 Rejected — a shuffled-codes output mode in the tool

The original plan: teach the tool to emit `PNF_LIWD_1` … `_n` shuffled, to the
clipboard, for pasting into STEP 3. Rejected in favour of §4.

It would mean a second output shape, a club dimension the tool deliberately
lacks (§5.2), and a mode switch beside the bracket cards — all to move codes
through the operator's clipboard, for the rarer of the two event shapes, when
the sheet can write them itself. The pull to build it was that the tool is
where draws happen; the answer is that this particular draw is not the kind
anybody watches (§4's table).

### 5.2 Rejected — a club-aware bracket card

Grouping a dual meet's cards by club would make the card output fit the dual
meet. Rejected: the tool knows nothing about clubs by design (bracket generator
spec §12, "One tool serves both event shapes because nothing in it knows about
clubs"), and teaching it would fork the one thing that is currently shared.

### 5.3 Rejected — prefilling pair names from STEP 1

Tempting, and the timing genuinely works: names land at STEP 1 *before* STEP 3,
so reading them out is not circular the way pulling from `event-data` would be.
(That is rejected separately, and for a different reason — the published
snapshot holds matches and standings, not a pre-draw roster; bracket generator
spec §9.5. It stays rejected.)

It fails on output shape instead. Feed names in and the tool returns bracket
cards, which is still not the column STEP 3 wants — the operator has read their
roster into a tool and gotten back something they cannot paste.

For a standard tournament the direction is reversed, and the rejection stands
for a second reason: names travel *out* of the draw and into the sheet
([`bracket-draw-name-import-spec.md`](bracket-draw-name-import-spec.md)), so
prefilling the tool from STEP 1 would feed it the roster it is about to
produce.

### 5.4 Rejected — putting the item in a generator's menu builder

See §2.1. It would vanish after generation, which is before the draw.

---

## 6. Out of scope

- Any change to how either generator builds the roster scaffold. STEP 3 stays
  blank; this spec fills it, it does not redesign it.
- The standard tournament's inbound half — draw files going back into its
  workbook. That is
  [`bracket-draw-name-import-spec.md`](bracket-draw-name-import-spec.md), and
  it needs nothing from here.
- Reading rosters or pairs from `event-data` (§5.3).
- Seeded or non-random draws — a different feature with its own spec.

---

## 7. Acceptance checklist

*(To be worked through when this is built, not before.)*

- [ ] The SAGE menu shows the bracket item in each master, in a fresh copy, and
      in a generated workbook — before and after `Generate event tabs`.
- [ ] It appears in a workbook with no sync configured.
- [ ] The dialog's link opens the tool in a new tab with `?event=` from
      `Title!B6` and `?category=` from the active category tab's display
      label — `A1` in a dual-meet workbook, `B1` in a standard one (§2.4).
- [ ] Run from `SCHEDULE`, it still opens the tool, with no `?category=`.
- [ ] `?category=` fills the field and is **not** written to `localStorage`;
      a later visit with no parameter comes back with the category empty and
      the event name still remembered.
- [ ] A category drawn through the menu route exports a file whose category
      line matches its tab exactly, and the import spec's §4 resolves it with
      no dropdown.
- [ ] `Shuffle roster codes` fills the active tab's STEP 3, refuses on a
      non-category tab, and refuses a filled STEP 3 without confirmation.
- [ ] `onOpen`'s shared block is byte-identical in all three `.gs` files.
- [ ] `node scripts/verify-sheet-generator.mjs` and
      `node scripts/verify-standard-generator.mjs` still pass.

---

## 8. Divergences

*(None — nothing here is built. Record departures when it is.)*
