# Spec — Bracket Generator × scoring workbook handoff

Connect `tools/bracket-generator.html` to the scoring workbooks that need a
draw: a **SAGE menu item** that deep-links into the tool with the event and
category already filled in, and — for dual meets — an output mode that emits
what the workbook's roster scaffold actually asks for.

> **Status: not implemented. Deliberately deferred until a standard
> tournament exists** — see §2. Nothing in this document has been built; it is
> written now so the reasoning survives the wait.

| File | Repo | Change |
| --- | --- | --- |
| `scripts/sheets-sync.gs` | `sage-tools-api` | one menu item + its dialog (§3) |
| `tools/bracket-generator.html` | `sage-match-control.github.io` | `?category=` (§4), and the shuffled-codes mode if §5 is taken |
| `docs/features/bracket-generator.md` | `sage-docs` | the menu route, once it exists |
| `docs/technical/bracket-generator.md` | `sage-docs` | same |

**No backend change.** No endpoint, no auth, no `event-data` read, no
`package.json` bump, no Cloud Run deploy. `sheets-sync.gs` ships by being
pasted into a workbook's own script project, so changing it is not a deploy
either.

---

## 1. Current state — the STEP 3 hole

Every generated dual-meet category tab carries a roster scaffold, built by
`writeRosterScaffold_` (`sheet-generator.gs:1826`), once per club:

| Step | Column (club A / club B) | State on a fresh workbook |
| --- | --- | --- |
| **STEP 1 · NAMES** | `AB` / `AQ` | blank — operator pastes pair names, 2 rows per pair |
| **STEP 2 · CODES** | `AF` / `AU` | pre-filled `<CLUB>_<CATKEY>_1` … `_<teamsA>` |
| **STEP 3 · RANDOMIZED** | `AG` / `AV` | **blank on purpose** — operator pastes the codes back shuffled |

STEP 3 is the draw, and **nothing fills it today.** The operator shuffles the
codes by hand, or with a throwaway `SORT(…, RANDARRAY(…))` off to one side.

The blankness is deliberate and load-bearing. From the generator's own comment
(`sheet-generator.gs:1869`):

> STEP 3 ("RANDOMIZED") is left BLANK … It used to be pre-seeded with the codes
> in roster order, which was wrong twice over: it made a step that hasn't been
> done yet look done, and an unshuffled STEP 3 is not a no-op — it maps every
> pair to its own roster slot, quietly defeating the blinding the shuffle
> exists to provide.

So the workbook has a hole shaped like a shuffle tool, and the shuffle tool
has no route into the workbook. That is the gap this spec closes.

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
a shuffle — but not in *shape*, which is what §5 is about.

---

## 2. Why this waits for a standard tournament

The tool has one output today — bracket cards, a PNG or text list of pairs
grouped under lettered brackets — and that artifact fits the two event shapes
very differently:

| | Standard tournament | Dual meet |
| --- | --- | --- |
| What the draw produces | pairs grouped into brackets | a shuffled column of team codes |
| Bracket count | arbitrary | 1 or 2 |
| Club dimension | none | two clubs, drawn separately |
| Does the tool's current output fit? | **yes, exactly** | no — wrong artifact |
| Is there a generator workbook to link from? | **not yet** | yes |

The irony is exact: the event shape the tool's output already serves has no
workbook to put a menu in, and the one with the workbook needs an output the
tool doesn't have.

Building the dual-meet half first would mean designing the deep-link contract
against the only generator that exists, then retrofitting it when the
standard-tournament sheet generator lands — which is already planned (see the
calculator's dual-meet handoff note in the root `CLAUDE.md`: "A standard-
tournament equivalent is planned, at which point the show/hide becomes a
swap"). Worse, the *natural* dual-meet-first implementation is "menu item →
bracket cards", which ships something that looks finished and doesn't actually
close the operator's loop. A feature that appears done and isn't is more
expensive than one that doesn't exist yet.

Waiting costs nothing: STEP 3 has been filled by hand for every event so far,
and one menu contract designed against both generators at once is cheaper than
two.

---

## 3. The menu item

### 3.1 Where it goes — and where it must not

Add it to **`addSyncMenuItems_` in `sheets-sync.gs`**, not to
`addGeneratorMenuItems_` in `sheet-generator.gs`.

`addGeneratorMenuItems_` removes itself once the workbook has been generated
(`sheet-generator.gs:750`, guarding on `PROP_TABS_GENERATED_FOR`), because
`Generate event tabs` can only error after that. But roster pasting — and
therefore the draw — happens *after* generation. An item placed there would
disappear at exactly the moment it becomes useful.

`addSyncMenuItems_` persists for the life of the workbook, and the Dual Meet
Master carries both scripts, so the item shows up before and after generation
alike.

Contributing from one builder also sidesteps the hazard the shared block warns
about — `onOpen` is duplicated verbatim across both files with a "Change one,
change both" note (`sheets-sync.gs:741-748`). This adds nothing to that block.

### 3.2 Ungated, unlike its neighbour

`Generate Scoresheets` sits behind `readSyncConfig_()` because it deep-links a
day key and facility name that only exist once sync is set up. A draw needs
neither. The bracket item is available in any workbook that carries the script,
configured or not — including the Master, and including a copy on the day the
rosters arrive but before sync is wired.

### 3.3 The dialog

Copy the `showScoresheetLink` pattern verbatim (`sheets-sync.gs:869`), for the
reason its own docstring gives:

> Apps Script can't open a URL from server-side code, and a `window.open()`
> fired on dialog load is blocked as a popup inside the sandboxed iframe — so
> the navigation has to come from a real user click on an anchor, which is what
> this dialog provides.

Same shape: a small modal, the prefilled values shown as a `<dl>` so the
operator can see what they're about to get, one green anchor with
`target="_blank" rel="noopener"` and `onclick="google.script.host.close()"`.
Makes no HTTP request and needs no additional OAuth scope.

Add `BRACKET_GENERATOR_URL` beside `SCORESHEET_GENERATOR_URL`
(`sheets-sync.gs:94`), pointing at
`https://sage-match-control.github.io/tools/bracket-generator.html`.

### 3.4 What it prefills

| Parameter | Source | Note |
| --- | --- | --- |
| `?event=` | `Title!B6` | the event title the generator wrote there (`sheet-generator.gs:2465`) |
| `?category=` | the active category tab's **`A1`** | the display label, uppercased (`sheet-generator.gs:1241`) |

**`A1`, not the tab name.** Tabs are named by the raw category key — `LIWD`,
matching `^[A-Z0-9]{2,8}$` — while `A1` carries the human label
("LOW INTERMEDIATE WOMEN'S DOUBLES"). The raw key is the wrong thing to print
on a bracket card handed to players.

**Open question:** how the item behaves when the active sheet isn't a category
tab. Options are to fall back to no `?category=` at all, or to offer a picker.
The cheap version — prefill from the active sheet when it looks like a category
tab, omit the parameter otherwise — is probably right, but it needs deciding
against the standard-tournament generator's tab naming, which doesn't exist
yet.

---

## 4. Tool-side — the `?category=` parameter

Mirrors `?event=` (bracket generator spec §3.7) and costs about four lines.

**With one deliberate difference: `?category=` must not persist.** `?event=` is
stored in `localStorage` and survives to the next visit because one operator
draws many categories for one event in a sitting (§3.6). Category is the
opposite, and that same section is explicit about why:

> Category and pairs are **not** persisted. They change every draw, and a
> prefilled pair list is a live hazard: an operator who does not notice it draws
> the wrong category's brackets.

A category arriving in a fresh deep link is fine — it came from the tab the
operator is looking at. A category resurrected from storage on an unrelated
later visit is the exact hazard above. So: read it, fill the field, never write
it back.

---

## 5. The shuffled-codes output mode

This is the part that actually closes the dual-meet loop, and the part with the
most left to decide.

### 5.1 What STEP 3 wants

A single column of `teamsA` team codes in shuffled order, for one club, ready
to paste. Not names, not brackets — codes. The tool's current exports (a PNG of
bracket cards, a text list of names under bracket headings) are neither.

### 5.2 Shape

Given a code stem (`PNF_LIWD`) and a pair count, emit `PNF_LIWD_1` …
`PNF_LIWD_n` shuffled. Clipboard, not download: STEP 3 is a paste target, and
the precedent is the calculator's **Copy plan & open generator** button, which
copies rather than exporting a file for exactly this reason.

One draw per club per category, since the two clubs' columns are independent
(`AF`/`AG` for A, `AU`/`AV` for B). Whether that means two invocations, a club
toggle, or one run emitting both columns is undecided.

### 5.3 The open design question

How this coexists with the bracket-card output. It could be a second export
button, a mode switch, or an argument that it belongs in a different tool
entirely. Deciding it before the standard-tournament generator exists means
deciding it with half the information.

---

## 6. The competing design: shuffle in the sheet instead

Worth stating plainly, because it is a genuinely strong alternative and this
spec should not pretend otherwise.

A `SAGE → Shuffle roster codes` item could write the shuffled codes straight
into `AG`/`AV` with one click — no browser, no clipboard, no round trip, no
paste errors, and the operator never leaves the workbook. For the mechanical
job STEP 3 describes, that is simpler than everything above.

What it loses is the reason the bracket generator is shaped the way it is. From
the bracket generator spec §4:

> this page is a *stage*, not a form … the ~3s shuffle exists so a room can
> watch a draw land

An in-sheet shuffle is invisible. It produces no artifact to post, and nobody
watches it happen. Where a draw needs to be *witnessed* — players in the room,
a photo of the bracket on a wall — the tool is the point and the sheet write is
not.

These may simply be two features for two situations: the in-sheet shuffle for a
routine roster blind, the tool for a draw with an audience. **Not decided
here.** The standard-tournament case is expected to clarify it, since a
standard tournament has the audience and (today) no sheet to write into.

---

## 7. Decisions

### 7.1 Rejected — prefilling pair names from STEP 1

Tempting, and the timing genuinely works: names land at STEP 1 *before* STEP 3,
so reading them out is not circular the way pulling from `event-data` would be.
(That is rejected separately, and for a different reason — the published
snapshot holds matches and standings, not a pre-draw roster; bracket generator
spec §9.5. It stays rejected.)

It fails on output shape instead. Feed names in and the tool returns bracket
cards, which is still not the column STEP 3 wants — the operator has read their
roster into a tool and gotten back something they cannot paste. Name prefill
becomes worth doing **together with** §5's shuffled-codes mode, and is
pointless without it.

### 7.2 Rejected — a club-aware bracket card

Grouping a dual meet's cards by club would make the card output fit the dual
meet. Rejected: the tool knows nothing about clubs by design (bracket generator
spec §12, "One tool serves both event shapes because nothing in it knows about
clubs"), and teaching it would fork the one thing that is currently shared.

### 7.3 Rejected — putting the item in the generator's menu builder

See §3.1. It would vanish after generation, which is before the draw.

---

## 8. Out of scope

- Any change to how `sheet-generator.gs` builds the roster scaffold. STEP 3
  stays blank; this spec fills it, it does not redesign it.
- Reading rosters or pairs from `event-data` (§7.1).
- Seeded or non-random draws — still a different feature with its own spec.
- A standard-tournament sheet generator. This spec *waits* on one; it does not
  specify one.

---

## 9. Acceptance checklist

*(To be worked through when this is built, not before.)*

- [ ] The SAGE menu shows the bracket item in the Master, in a fresh copy, and
      in a generated workbook — before and after `Generate event tabs`.
- [ ] It appears in a workbook with no sync configured.
- [ ] The dialog's link opens the tool in a new tab with `?event=` from
      `Title!B6` and `?category=` from the active category tab's `A1`.
- [ ] Opening it from a non-category tab still works, with whatever §3.4's open
      question settles on.
- [ ] `?category=` fills the field and is **not** written to `localStorage`;
      a later visit with no parameter comes back with the category empty and
      the event name still remembered.
- [ ] `onOpen`'s shared block is byte-identical in both `.gs` files.
- [ ] `node scripts/verify-sheet-generator.mjs` still passes (it covers
      `sheet-generator.gs` only, but the shared menu block lives in both).

---

## 10. Divergences

*(None — nothing here is built. Record departures when it is.)*
