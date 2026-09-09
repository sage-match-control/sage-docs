# Spec — Tournament Calculator dual-meet fixes

Two changes to dual-meet mode in `tools/tournament-calculator.html`: collapse
the two club-pair inputs into one, and give dual meets their own bracket
default.

Independent of `calculator-pwa-spec.md` — same file, unrelated concern. Either
can ship first.

---

## 1. Current state

Default category object (line ~692):

```js
{id:nextId++, code:free[0], name:free[1], teams:16, group:4, adv:2, fill:'bye',
 teamsA:8, teamsB:8, poFormat:'semis'}
```

The dual-meet card renders three inputs: `Club A pairs` (`teamsA`, default 8),
`Club B pairs` (`teamsB`, default 8), and `Number of brackets` (`group`).

`group` is **shared with standard mode**. The dual card's `value="${c.group||1}"`
only falls back to 1 when `group` is falsy — and the default is `4`, so a new
dual category opens showing **4 brackets**, not 1.

The maths in `calcDualCategory` / `dualGroupStage` already handles unequal club
sizes correctly and clamps the bracket count:

```js
const kEff = Math.max(1, Math.min(30, Math.round(k)||1, a, b));
```

That clamp is correct — a bracket needs at least one pair from each club — and
is not in scope to change.

---

## 2. One pairs input, default 5

Replace the two club-pair fields with a single **`Pairs per club`** input,
default **5**.

### 2.1 UI

In `catCard()`'s dual branch, the `grid3` becomes:

```html
<label class="f"><span class="lab">Pairs per club</span>
  <input type="number" data-k="pairsPerClub" min="1" max="50" value="${c.teamsA||5}"></label>
<label class="f"><span class="lab">Number of brackets</span>
  <input type="number" data-k="groupDual" min="1" max="30" value="${c.groupDual||1}"></label>
```

Two fields now, not three — adjust the wrapper from `grid3` to `grid2`.

### 2.2 Keep `teamsA` / `teamsB` internally

**Do not collapse the model to a single field.** The `pairsPerClub` handler
writes the same value to both:

```js
else if(k==='pairsPerClub'){
  const v = Math.min(50, Math.max(1, +e.target.value||1));
  c.teamsA = v; c.teamsB = v;
}
```

Reasons to keep both:

- `calcDualCategory`, `dualGroupStage`, and `splitEven` are already written and
  correct for asymmetric inputs. Collapsing the model means rewriting working
  maths for no gain.
- The CSV schema's `teams_a` / `teams_b` columns stay stable, so previously
  exported plans still import.
- **Asymmetry remains reachable via CSV import** (§2.3), so the code paths are
  not dead.

Change the default to `teamsA: 5, teamsB: 5`.

### 2.3 Keep the unequal-pairs note

```js
if(a!==b) notes.push(`Club pair counts differ (${a} v ${b}) — pairs play unequal numbers of matches this category.`);
```

**Keep this.** It can no longer be triggered from the UI, but importing a CSV
written before this change can still produce `teamsA !== teamsB`, and the note
is exactly what tells the user why their imported numbers look odd. Removing it
would make that case silent.

On import, populate the single input from `teamsA` (falling back to `teamsB`).
Do not silently equalise them — let the note fire, and let the user correct it
by typing in the field, which then writes both.

---

## 3. Dual meets default to 1 bracket

Add a **separate** `groupDual` field rather than reusing `group`:

```js
{... teams:16, group:4, adv:2, groupDual:1, teamsA:5, teamsB:5, poFormat:'semis'}
```

- Dual card binds to `groupDual` (§2.1).
- `calcDualCategory` reads `c.groupDual` instead of `c.group`:
  ```js
  const wantedK = Math.min(30, Math.max(1, Math.round(c.groupDual)||1));
  ```
- Standard mode keeps using `group` (default 4), untouched.

**Why a separate field rather than resetting `group` on mode switch:** the
format toggle is global and flips every category at once. Overwriting `group`
would destroy a carefully set standard-mode bracket layout the moment someone
peeks at dual mode, with no undo. Two fields means toggling back and forth is
lossless.

**CSV:** the existing `brackets` column carries whichever field is active for
the current `format` setting — `group` when `format=standard`, `groupDual` when
`format=dual`. The `format` setting row already appears earlier in the file, so
the importer knows which field to populate. This keeps the column count
unchanged.

---

## 4. Acceptance checklist

- [ ] A new dual-meet category opens with **Pairs per club = 5** and
      **Number of brackets = 1**, and two inputs, not three.
- [ ] Typing in `Pairs per club` updates both clubs; the summary line reads
      `1 bracket (5 v 5)`.
- [ ] Switching format standard → dual → standard leaves the standard-mode
      bracket count untouched (not reset to 1).
- [ ] Setting brackets above the pair count still clamps, and still shows the
      "Only N brackets possible" note.
- [ ] Export CSV → Import CSV round-trips a dual plan with the right bracket
      count landing in `groupDual`, not `group`.
- [ ] The exported Breakdown sheet still opens and its `Brackets` column matches
      the on-screen bracket count — this spec does not change that sheet, so it
      is a regression check, not a new behaviour.
- [ ] Importing a **pre-change** CSV with `teams_a=8, teams_b=3` loads, shows 8
      in the single input, and still emits the "Club pair counts differ" note.

## 5. Out of scope

- The `min(k, a, b)` bracket clamp and the dual-meet match maths generally —
  both are correct.
- Re-introducing asymmetric club sizes as a UI feature.
- Any change to standard-mode defaults (`teams: 16`, `group: 4`, `adv: 2`).
- The PWA work in `calculator-pwa-spec.md`.
- **The exported Breakdown sheet's bracket columns.** Deliberately deferred, not
  overlooked. For the record, so it isn't rediscovered as a fresh bug: the
  `Brackets` column (`p.groups`) is correct — it reports the effective count
  after the clamp, and §3's default fix is what stops it opening at 4. The
  adjacent `Bracket sizes` column is the questionable one: `bracketShorthand()`
  renders `p.sizes`, which for dual meets is the *combined* A+B per bracket, so
  5 pairs a side over 2 brackets prints `5-5` rather than `3v2, 2v3`. Real, but
  cosmetic and export-only.
