# Spec — Verifiable draw

Make the bracket draw **provable**, not just fair. Replace the unrepeatable
`Math.random()` shuffle with a deterministic hash sort driven by a seed, stamp
the verification payload onto both exports, and add a **How it works** dialog
written for a player, not a developer.

**Every draw has a seed and every draw is reproducible.** Typing one is
optional — leave the field blank and the tool generates one and shows it (§3.3)
— so a quick practice draw costs no ceremony while still producing an artifact
that can be checked later.

The draw is already fair — `shuffle()` is a correct, unbiased Fisher-Yates.
What it cannot do is *prove* that to a skeptical player. This spec changes
nothing about the odds and everything about the evidence.

**Status: written and working, but not committed or deployed.** The working
copy of `tools/bracket-generator.html` carries the seed field, the SHA-256
fingerprint sort, the auto-seed generator with its read-aloud-safe alphabet,
the verification payload on both exports, and the *How it works* dialog, and
both doc pages (§10) carry the algorithm and the draw ceremony.

None of it is committed, so **the live tool has none of it** — the committed
copy contains no `seedInput` and no SHA-256. Shipping this is a commit and a
push, not more building. Verify against §11's acceptance checklist first,
since it has never run anywhere but a working tree.

| File | Repo | Change |
| --- | --- | --- |
| `tools/bracket-generator.html` | `sage-match-control.github.io` | the whole feature — design in §3–§8, **exact edits in §13** |
| `docs/technical/bracket-generator.md` | `sage-docs` | the algorithm, canonically stated — **copy in §10.1** |
| `docs/features/bracket-generator.md` | `sage-docs` | the draw ceremony + a required wording fix — **copy in §10.2** |
| `docs/specs/README.md` | `sage-docs` | index entry (already added) |

> **Implementing this?** §1–§9 are the reasoning; **§13 is the executable
> version** — every edit given as an exact find/replace against the current
> file, in order. §10 carries the doc changes as ready-to-paste prose. Read the
> reasoning if a decision looks odd, but you can build from §13 and §10 alone.

**No backend change.** No endpoint, no `event-data` read, no dependency, no
network call at run time. `crypto.subtle` is built into the browser, so the
tool stays one self-contained file that fetches nothing.

## Build order

1. **§4 — the hash sort.** Swap the shuffle, absorb the async ripple. The page
   still draws exactly as before; only the source of the ordering changes.
2. **§3 — the seed field**, with auto-seed fallback.
3. **§5 — the exports.** Verification payload on both, plus the on-page seed line.
4. **§6 — the draw counter.**
5. **§7 — the How it works dialog.** Last, because it documents the finished
   behaviour and its copy has to match what actually shipped.
6. **§10 — the docs.** Part of the work, not a follow-up.
7. **§11 — acceptance.**

§13 walks these same steps as exact edits, in this order. Following §13
top to bottom satisfies this list.

### Conventions to match

Unchanged from the [bracket generator spec](../implemented/bracket-generator-spec.md): one
self-contained file, inline `<style>`/`<script>`, no bundler, no CDN, no new
`<link>`. Every colour resolves from the `THEME` block (§4.1 there) — the
dialog is not an exception. `escapeHtml()` on anything reaching `innerHTML`.

---

## 1. What today's draw can and cannot prove

`generateBrackets()` locks the result in before the animation starts, and the
comment says so outright:

> Lock in the real result up front so the animation is purely cosmetic —
> what the user sees settle on is exactly what gets rendered/exported.

The frames that flicker during those three seconds are cosmetic re-deals —
decoy data, not the draw. So the most visible part of the tool is the part that
proves the least. That is not dishonest, but it must never be presented as the
draw, and after this spec it does not have to be: the result becomes provable
by other means and the animation becomes an honest *reveal* of something the
announced seed already determined.

Four suspicions a player can reasonably hold, and where each is answered:

| Suspicion | Answered by |
| --- | --- |
| "You re-rolled until you liked it." | §3.2 (a fixed seed makes re-pressing a no-op), §6 (the counter) |
| "The tool is rigged." | §4 (deterministic, externally reproducible) |
| "You edited the list first." | §9.1 (the list is committed before the seed exists) |
| "You doctored the image." | §5 (a doctored bracket will not reproduce from the seed) |

---

## 2. Why a hash sort rather than a seeded PRNG

A seeded PRNG (mulberry32 and friends) would also make the draw reproducible.
Hashing wins on **external** verification, which is the actual requirement:

- **SHA-256 is universal.** It is in Python's `hashlib`, in every language's
  standard library, and on hundreds of free calculator sites. A custom PRNG is
  something a verifier has to implement from a description, correctly, before
  they can check anything.
- **One pair can be checked alone.** A player verifies *their own* placement
  without reproducing the whole draw or understanding it: hash two strings,
  compare which sorts first. That is a thirty-second check on a phone.
- **No floating-point semantics to argue about.** PRNG output can differ subtly
  between languages; hex strings cannot.
- **Input order stops mattering.** A seeded PRNG shuffle only reproduces if the
  pair list is in the identical order, making the export fragile. Each pair's
  hash depends only on the seed and its own name, so order-independence falls
  out for free.

**Uniformity is unchanged.** Sorting by SHA-256 of a per-pair string yields a
uniformly random permutation for any purpose this tool has. The draw is exactly
as random as it was; it is now also checkable.

---

## 3. The seed

### 3.1 Markup

A new field directly under **Event name**, above **Category name**:

```html
<div class="field">
  <label class="field-label" for="seedInput">Seed <span class="field-optional">optional</span></label>
  <input id="seedInput" class="field-input" type="text"
         placeholder="e.g. MANGO, 42, 7-3" autocomplete="off" maxlength="40" />
  <div class="field-meta">
    <span class="field-hint">For a public draw, ask someone in the room for a word or number — it decides the draw, and anyone can check it afterwards. Leave blank and one is generated for you. <button type="button" class="link-btn" id="howItWorksLink">How it works</button></span>
  </div>
</div>
```

`.field-optional` is the existing chip from the bracket generator spec §3.1 —
same treatment as **Event name**, since both fields are genuinely optional and
should read that way at a glance.

The **How it works** entry point sits here because this is the field that needs
explaining. §7.1 adds a second entry point on the results.

### 3.2 Normalization — get this exactly right

**The seed is trimmed and uppercased before hashing, and the tool displays the
normalized form.** What is on screen is what was hashed.

```js
const normalizeSeed = s => String(s).trim().toUpperCase();
```

`Mango`, `mango ` and `MANGO` must be one draw, not three. If the displayed
seed and the hashed seed can ever differ, every external check fails and the
whole feature is decorative.

Captured into `currentSeed` at draw time alongside `currentCategory` and
`currentEventName`, for the same reason (§3.2 of the bracket generator spec):
what the operator saw settle is what exports.

### 3.3 Typing one is optional — having one is not

Leaving the field blank generates a seed (8 characters, `crypto.getRandomValues`,
Crockford-style alphabet omitting `I`/`O`/`0`/`1` so it can be read aloud and
written down without ambiguity), fills the field with it, and draws.

**There is always a seed, so every draw is always reproducible.** A bracket
drawn in thirty seconds with nobody watching can still be re-checked a month
later, which is exactly the provability gain: the artifact carries its own
evidence whether or not anyone thought to ask for it at the time.

What differs is *who chose the seed*, and the exports must not blur that:

| Seed | Export reads | What it earns |
| --- | --- | --- |
| Operator or room typed it | `Seed: MANGO (entered)` | reproducible **and** witnessed |
| Left blank | `Seed: 8F2K7PXR (auto)` | reproducible, **not** witnessed |

An auto-seed is genuinely verifiable — a posted bracket reproduces from it, and
a doctored one does not. What it cannot do is defeat re-rolling, since the
operator can simply press again for a new one. Labelling it honestly is what
stops an auto-seeded bracket from implying a ceremony that never happened, and
it is why §7.2's dialog explains the difference in the player's own terms rather
than letting a printed seed speak for itself.

This also keeps the tool fast for the cases §9.1 of the bracket generator spec
protects — a practice or dry-run draw, a club drawing an internal ladder —
without making them perform a ceremony they do not need.

---

## 4. The draw itself

### 4.1 The algorithm, stated normatively

For each pair, in the list as entered (trimmed, blank lines dropped — unchanged
from `parsePairs()`):

```
fingerprint(pair) = SHA-256( normalizedSeed + "|" + pair )   → lowercase hex
```

Sort ascending by that hex string. Ties break by the pair string, then by
original input position. Then deal round-robin into brackets exactly as
`dealBrackets()` does today, so sizes still differ by at most one.

**The separator is a single `|` with no surrounding spaces.** The pair string is
exactly the text the export lists, character for character. Any ambiguity here
breaks external verification, so both the dialog and the text export quote the
exact string that was hashed.

### 4.2 The code change is small

`dealBrackets()` keeps its whole shape; only its ordering source changes.

```js
const sha256Hex = async (s) => {
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s));
  return [...new Uint8Array(buf)].map(b => b.toString(16).padStart(2, '0')).join('');
};

// Fingerprints are computed once per draw and carried through to the exports,
// so the artifact can show exactly what the ordering was derived from.
async function fingerprintPairs(pairs, seed){
  const hashes = await Promise.all(pairs.map(p => sha256Hex(seed + '|' + p)));
  return pairs
    .map((pair, i) => ({ pair, hash: hashes[i], i }))
    .sort((a, b) => a.hash < b.hash ? -1 : a.hash > b.hash ? 1 :
                    a.pair < b.pair ? -1 : a.pair > b.pair ? 1 : a.i - b.i);
}
```

`generateBrackets()` becomes `async`: await the fingerprints, then hand the
ordered list to the existing synchronous path. `runShuffleAnimation()`,
`renderBracketGrid()` and both exports are untouched by this step.

**Sort on the full 64-character hash.** The 8-character form in §5.2 is a
display convenience only. A truncated sort key starts colliding around 65k
items — far beyond anything this tool will see, but it costs nothing to get
right and is expensive to discover later.

#### `shuffle()` stays — for the animation only

`shuffle()` is called from two places today, and this spec changes exactly one
of them:

| Call site | What it feeds | After |
| --- | --- | --- |
| `dealBrackets()` (`:697`) | the real draw | **replaced** by `fingerprintPairs()` |
| `runShuffleAnimation()`'s `step()` (`:840`) | the cosmetic per-tick re-deals | **unchanged** |

So **do not delete `shuffle()`**, and do not leave `dealBrackets()` calling it.
Both mistakes are easy and only one is loud: deleting the function breaks the
animation immediately and visibly, while leaving `dealBrackets()` wired to it
ships a draw that prints a seed it was not produced from — the false proof §4.3
exists to prevent, arriving silently.

The animation's frames are thrown away and never exported, so they need no
reproducibility and `Math.random()` remains the right tool for them.

**Why sorting by a precomputed key is a real shuffle**, and not the broken
`arr.sort(() => Math.random() - 0.5)` it superficially resembles: that
antipattern fails because its comparator is *inconsistent* — asked about the
same pair twice it can answer differently, violating the sort contract and
producing biased, engine-dependent output. Here each pair carries one fixed key
computed before the sort begins, so the comparator is consistent and total.
That is the standard decorate–sort–undecorate shuffle, and with distinct random
keys every ordering is equally likely — uniform, exactly like the Fisher-Yates
it replaces.

### 4.3 No silent fallback

`crypto.subtle` requires a secure context. GitHub Pages is HTTPS, so this never
bites in production — but a local `file://` open might.

**If `crypto.subtle` is unavailable, fail loudly**: disable the draw button on
load and state that verifiable draws need the page served over HTTPS.

What must never happen is a draw quietly falling back to `Math.random()` while
still printing a seed. An export that names a seed it was not produced from is
worse than having no feature at all — it is a false proof, and a false proof is
the one outcome this whole spec exists to prevent. Refuse to draw instead.

---

## 5. What the exports have to carry

Every draw carries this, auto-seeded or not — that is what makes the artifact
self-proving rather than dependent on someone remembering to switch a mode on.

### 5.1 On-page, before exporting

A `#resultSeed` line joins `#resultEvent` in the results header, showing the
seed and draw number. Same reasoning as `#resultEvent` (§3.3 there): the
operator sees what the export will say before committing it. Always shown —
unlike `#resultEvent`, there is no blank state to hide.

### 5.2 Text export — the full audit trail

The text file is **self-describing**: it carries everything needed to re-check
the draw, including the instructions. No hosted verification page to maintain,
nothing to keep online, and the artifact still works years later. This fits what
the tool already is — it exports files and publishes nothing.

Appended before the existing footer:

```
DRAW VERIFICATION
------------------------
Seed:   MANGO  (entered)
Draw:   #1
Method: SHA-256(seed + "|" + pair), sorted ascending, dealt round-robin

Fingerprints, in draw order:
  07a3f91c...  Player 1 / Player 2
  1b4e88d0...  Player 3 / Player 4
  ...

TO CHECK THIS DRAW YOURSELF
Search the web for "SHA-256 calculator" and paste in exactly:
  MANGO|Player 1 / Player 2
The result should start with the fingerprint above. Pairs are sorted
from lowest fingerprint to highest, then dealt one at a time around
the brackets: 1st to Bracket A, 2nd to Bracket B, and so on.

If two fingerprints match on the digits shown, compare more of them —
the order is decided by the full code, not the first eight characters.
```

Fingerprints are shown to 8 hex characters — long enough that matching one by
chance is not a thing, short enough to compare by eye. The full hash is
recomputable by anyone who wants it, and is what the sort actually ran on
(§4.2).

**The one edge case a verifier could trip on**, and why the last two lines of
that block exist. Because only 8 characters are shown, two pairs could in
principle share the displayed prefix while sorting on digits that aren't
visible — making the printed order look wrong to someone checking by eye. For
100 pairs the odds are roughly one in a million, but the fix is one sentence in
the export rather than a puzzled player.

It stays **out** of the How it works dialog (§7.2) deliberately: that copy is
for legibility, and a one-in-a-million caveat buys a player nothing but doubt
about whether they followed the instructions correctly. The text export is the
precise protocol; the dialog is the explanation.

### 5.3 PNG export — enough to check, not enough to clutter

The poster gets one added line above the existing S.A.G.E. footer:

```
Seed: MANGO (entered) · Draw #1 · SHA-256 verifiable — see the text export to check
```

Per-pair fingerprints stay out of the image; they live in the text file. Grow
`bottomH` (currently 56) to fit the second line, and run the new line through
`truncateToWidth` like every other canvas string (§3.4 there). Colours resolve
through `cssVar` like everything else (§6.2 there).

---

## 6. The draw counter

`Draw #N` on both exports and the on-page seed line.

It counts **draws against the current input** — it resets whenever the category
name or the pair list changes, and increments otherwise.

Because a fixed seed produces a fixed result, the counter only ever advances
meaningfully when the *seed* changed too. So `Draw #4` reads precisely as "four
seeds were tried on this list", which is exactly the number a skeptic wants and
exactly the thing that was invisible before.

The counter does not prevent re-rolling. It makes silence about it impossible,
which is the part that matters socially.

---

## 7. The How it works dialog

The feature most likely to be judged by a player rather than a developer, so
its copy is specified here rather than left to implementation.

### 7.1 Behaviour

- A native `<dialog>` opened with `showModal()` — focus trapping, `Esc` to
  close, and a `::backdrop` for free, with no dependency and no custom
  focus-management code to get wrong.
- Two entry points: the **How it works** button in the seed field's hint row
  (§3.1), and a second on the results action bar, so someone looking at a
  finished draw can find out how it was made without scrolling back up.
- Styled from the `THEME` tokens. Scrolls internally on a short viewport;
  readable at 360px wide.
- Content is static markup — no interpolation of the current draw, so nothing
  in it can be an injection sink.

### 7.2 The copy

Written for a player. "Fingerprint" carries the whole idea; "hash" appears once,
parenthetically, so a technical reader is not confused and a non-technical one
is not stopped.

---

> ### How this draw works
>
> **The short version:** the brackets are decided by a single short word or
> number — the *seed* — and anyone can check afterwards that the result really
> came from it. At a public draw, that word comes from someone in the room.
>
> **1. Everyone sees the list first**
> Before anything is drawn, the full list of pairs is up on screen. Nothing can
> be added or taken away after that without it being obvious.
>
> **2. A seed is set**
> A word, a number, anything. For a public draw it should be called out by a
> player rather than the organiser — then typed in and shown on screen, so you
> can photograph it. If nobody supplies one, the app generates one and displays
> it, and the sheet says which of the two happened. (More on that below.)
>
> **3. Every pair gets a fingerprint**
> The computer takes the seed and a pair's names together and turns them into a
> long code — a fingerprint (technically a *hash*) that looks like `07a3f91c…`
>
> The same names with the same seed always give the same fingerprint. But change
> a single letter of the seed and every fingerprint changes completely. Nobody
> can work out ahead of time what fingerprint a pair will get.
>
> It is not a code we invent. It is a standard calculation used all over the
> world, and there are free websites that will do it for you — which is what
> makes it possible for you to check us.
>
> **4. Pairs are sorted by fingerprint, then dealt out**
> Lowest fingerprint first, the way you would sort a list alphabetically. Then
> they are dealt around the brackets one at a time — first pair to Bracket A,
> second to Bracket B, and so on — so every bracket ends up the same size, or
> within one.
>
> **5. That is the whole draw**
> Because the fingerprints come from the seed, the result was already settled
> the moment the seed was set. Pressing the button does not roll anything — it
> just shows what the seed had already decided. The few seconds of shuffling on
> screen are there to make the reveal watchable, not to pick the winner.
>
> ### Where did this seed come from?
>
> Look at the sheet. Next to the seed it says one of two things.
>
> **`(entered)`** — a person typed it in. At a public draw that should be a
> player calling it out, not the organiser.
>
> **`(auto)`** — nobody supplied one, so the app made one up at the moment of
> the draw.
>
> Either way the result can still be checked afterwards, and a bracket that has
> been altered will not match. The difference is what it proves about *how the
> seed was chosen* — only a seed called out in the room shows the organiser
> didn't go hunting for one they liked.
>
> ### Why this is hard to rig
>
> - **Nobody can predict what a seed will produce.** There is no way to look at
>   a word and know which bracket it will hand you — you have to run it.
> - **Pressing again changes nothing.** The same seed always gives the same
>   brackets. A different result needs a different seed — and at a public draw
>   that means asking the room again, out loud, in front of everyone.
> - **The sheet counts the draws.** If it says Draw #3, three seeds were tried
>   on this list. That number cannot be hidden.
> - **The exported file records everything** — the seed, every pair, and every
>   fingerprint — so the draw can still be checked long after the day.
>
> ### Check it yourself
>
> You do not have to take our word for any of this, and you do not need to
> understand all of it to check your own pair.
>
> 1. Search the web for **"SHA-256 calculator"** and open any of them.
> 2. Type in the seed, a `|` character, then your pair exactly as it appears on
>    the sheet — for example `MANGO|Player 1 / Player 2`
> 3. Compare the start of the result to the fingerprint printed next to your
>    name in the exported list. They should match.
>
> Pairs with lower fingerprints were dealt first. If your fingerprint is in the
> right place in that order, your bracket is exactly where the draw put it.

---

### 7.3 One explanation, two renderings

The dialog and the text export's `TO CHECK THIS DRAW YOURSELF` block say the
same thing in HTML and in plain text. They must stay in step — if the algorithm
or the separator ever changes, both change. Worth a comment at each site
pointing at the other.

---

## 8. What this does not prove

Stated here so nobody oversells it on a poster:

- It does **not** protect against the operator and the seed-caller colluding.
- It does **not** make an auto-seeded draw (§3.3) witnessed — only reproducible.
- It does **not** verify that the pair list was complete or correct. It proves
  the bracket follows from *that* list, not that the list was right.

Defeating collusion needs commit-reveal against a public future event (publish
the list, announce that the seed will be a named lottery draw, draw afterwards).
That is a championship-final ceremony, not a Saturday-club one, and it is out of
scope here — §9.3.

---

## 9. Decisions

### 9.1 The list is committed before the seed is taken — always in that order

Not a UI constraint but a real one, and it belongs in the operator-facing docs.

If the seed is taken *first*, an operator who can compute fingerprints could
adjust the list — adding, removing or renaming a pair — to steer the result,
because with a known seed every candidate list has a knowable outcome. Showing
the list first and taking the seed second removes that move entirely: at the
moment the list is fixed, the seed does not exist yet; at the moment the seed
exists, the list can no longer change unnoticed.

### 9.2 Rejected — `crypto.getRandomValues()` for the draw itself

The instinct on "prove it's random" is stronger entropy. It makes things worse:
a cryptographically random draw is *less* reproducible and therefore less
provable. Verifiability wants the randomness moved out of the draw and into a
public seed, not strengthened inside it.

`getRandomValues` still appears, but only to generate an auto-seed (§3.3) —
where unpredictability is the requirement and reproducibility is provided by
recording the value.

### 9.3 Rejected — a public randomness beacon (drand, NIST)

Technically the strongest option and practically the worst fit. It needs network
at the venue, breaks the tool's "fetches nothing" property, adds an availability
dependency to a draw that must happen on time in a gym, and no player would ever
verify it. A seed called out by a player is weaker on paper and far stronger in
the room.

### 9.4 Rejected — hiding *Randomize again* during a public draw

Considered, since it is the re-roll button. Rejected: re-drawing is sometimes
legitimate (a pair was missing from the list), and a hidden button pushes that
into an untracked reload instead of an on-the-record `Draw #2`. Making re-rolls
*visible* beats making them awkward.

### 9.5 Rejected — showing fingerprints on the PNG

They belong in the audit trail, not on a poster on a wall. §5.3 keeps the image
readable and points at the text export for the detail.

---

## 10. Docs

Both edits are written out in full below. Paste them; do not paraphrase. The
technical page is the document a third-party verifier works from, so its
wording about the separator and the seed casing is load-bearing.

### 10.1 `docs/technical/bracket-generator.md`

**Insert this section immediately after the `## The deal` section, before
`## One palette source`:**

```markdown
## The draw is verifiable

The ordering is not `Math.random()` — it is a deterministic function of the
seed and the pair list, so anyone can reproduce it without this tool.

For each pair, in the list as entered (trimmed, blank lines dropped):

    fingerprint = SHA-256( seed + "|" + pair )      lowercase hex

Pairs are sorted ascending by that hex string, then dealt round-robin as above.
Ties — which only arise from two identical pair strings, since identical input
hashes identically — break by the pair text, then by input position.

Three details are load-bearing for anyone re-checking a draw:

- **The separator is a single `|`, with no surrounding spaces.**
- **The seed is trimmed and uppercased before hashing** (`normalizeSeed`), and
  the field displays the normalized form, so what is on screen is what was
  hashed.
- **The sort runs on the full 64-character hash.** The 8-character form in the
  text export is display only; sorting on a truncated key would start colliding
  around 65k items.

Because each pair's fingerprint depends only on the seed and its own name, the
order of the pasted list does not affect the result — which is what lets the
export stay verifiable without recording the input order.

`shuffle()` still exists and is still Fisher-Yates, but now feeds only
`runShuffleAnimation()`'s cosmetic per-tick frames. Those are thrown away and
never exported, so they need no reproducibility.

### The seed

Typing one is optional; having one is not. A blank field generates an
8-character seed (`crypto.getRandomValues`, alphabet omitting `I`/`O`/`0`/`1`
so it survives being read aloud), fills the field with it and draws — so every
draw is reproducible whether or not anyone asked for a ceremony. Both exports
label the source, `(entered)` or `(auto)`, because only a seed supplied by a
person shows the organiser did not go looking for one they liked.

### Secure context required

`crypto.subtle` is only available in a secure context. GitHub Pages is HTTPS so
this never bites in production, but a local `file://` open can hit it. The page
**fails closed**: the draw button is disabled and the reason stated. It never
falls back to `Math.random()` while still printing a seed — an export naming a
seed it was not produced from is a false proof, which is worse than no feature.
```

**Also update the existing `## Replaced a per-event copy` section's closing** —
no change needed to its content; leave it as the last section.

### 10.2 `docs/features/bracket-generator.md`

**Edit 1 — one wording fix, required.** The page says the draw has "no
seeding", meaning the *sports* sense (ranking-based placement). With a
cryptographic seed now on the same page, that reads as a contradiction.

Find:

```
The draw itself is uniformly random with no seeding and no protected pairings: it
won't keep training partners or club-mates apart, and it doesn't try to.
```

Replace with:

```
The draw itself is uniformly random with no rankings and no protected pairings:
it won't keep training partners or club-mates apart, and it doesn't try to.
```

**Edit 2 — insert this section immediately after `## The event name`, before
`## What comes out`:**

```markdown
## Running a draw people can trust

Every draw is checkable afterwards, and for a draw people are watching there is
a way to run it that makes that obvious.

**Show the list first, then take the seed.** Put the pairs up on screen before
anyone supplies a seed. That order matters: once the list is visible it can't be
quietly changed, and at that moment the seed doesn't exist yet, so nothing can
be tuned to suit it.

**Ask the room for the seed.** Any word or number — ideally called out by a
player rather than by you, and best of all by someone from the club most likely
to be unhappy with the result. Type it in, say it out loud, and let people
photograph the screen.

**Then draw.** The few seconds of shuffling on screen are a reveal, not the
draw itself — the seed already decided the answer. Pressing again with the same
seed gives the same brackets, so there is nothing to gain by re-pressing.

**Leaving the seed blank is fine.** The tool makes one up and shows it, and the
draw is still checkable later. The exported files say which happened —
`(entered)` if a person supplied it, `(auto)` if the app did — so a bracket
never implies a ceremony that didn't happen.

If you do have to re-draw — a pair was missing, the list was wrong — say so out
loud, fix it, and ask for a new seed. The exports carry a draw number, so a
second attempt shows as `Draw #2` rather than passing unnoticed.

### How a player checks their own pair

They don't have to trust the tool, and they don't need to understand the whole
draw:

1. Search the web for "SHA-256 calculator" and open any of them.
2. Type the seed, a `|`, then their pair exactly as the exported list shows it —
   for example `MANGO|Player 1 / Player 2`
3. Compare the start of the result to the fingerprint printed beside their name.

Pairs with lower fingerprints were dealt first. The **How it works** button on
the tool explains all of this in the same terms, and the exported text file
carries the instructions with it.
```

---

## 11. Acceptance checklist

**The draw**

- [ ] The same seed and the same pair list always produce the same brackets,
      across reloads and across browsers.
- [ ] Changing one character of the seed changes the result.
- [ ] `Mango`, `mango ` and `MANGO` all produce the identical draw, and the
      field displays the normalized form.
- [ ] Re-ordering the pasted pair list does **not** change the result.
- [ ] Bracket sizes still differ by at most one; the count still clamps to the
      pair count with its inline message.
- [ ] A blank seed auto-generates one, fills the field, and both exports label
      it `(auto)`; a typed seed labels it `(entered)`.
- [ ] `dealBrackets()` no longer calls `shuffle()`, **and `shuffle()` still
      exists** and is still called by `runShuffleAnimation()` — the animation
      cycles names during the draw exactly as it does today.
- [ ] The sort runs on the full hash, not the 8-character display form.

**Verification**

- [ ] A fingerprint from the text export reproduces on an unrelated third-party
      SHA-256 site from the documented string, character for character.
- [ ] The pairs in the text export, sorted by their listed fingerprints, produce
      the bracket assignment shown.
- [ ] A pair name containing `|`, `<`, `&` or `"` verifies correctly and renders
      literally everywhere.

**Exports**

- [ ] Both exports carry seed, source label, draw number and method.
- [ ] The PNG's new footer line is truncated rather than painted past either
      edge, and its colours follow a `:root` re-skin.
- [ ] The text export's instructions quote the exact string that was hashed.

**Counter**

- [ ] Drawing twice with the same seed gives the identical bracket and shows
      `Draw #2`.
- [ ] Editing the category or the pair list resets the counter to `#1`.

**Dialog**

- [ ] Opens from both entry points, closes on `Esc`, on the backdrop and on its
      own close button; focus returns to the button that opened it.
- [ ] Readable at 360px wide and on a short viewport; follows a `:root` re-skin.
- [ ] Its explanation matches the text export's, and both match §4.1.
- [ ] **No claim in it is false for an auto-seeded draw.** It never says the
      seed came from the room unconditionally; it explains `(entered)` vs
      `(auto)` and what each one does and does not prove.
- [ ] It states plainly that the on-screen shuffle is a reveal, not the draw —
      the animation is never presented as the thing that decides the result.

**Failure mode**

- [ ] With `crypto.subtle` unavailable, the page loads, the draw button is
      disabled, and the reason is stated. No draw is produced by any path.

---

## 12. Out of scope

- Commit-reveal against a public future event (§8). Worth specifying separately
  if a championship ever wants it.
- Seeded or otherwise non-random draws in the *sports* sense — snake seeding,
  protected pairings, byes. Still a different feature.
- Publishing a draw anywhere. The tool exports files; it publishes nothing.
- Any change to the shuffle animation's timing, curve or reduced-motion branch.
  Its meaning changes (§1); its code does not.
- The workbook handoff — see
  [workbook handoff](../not-started/bracket-generator-workbook-handoff-spec.md), which is
  independent of this and still waiting on a standard tournament.

---

## 13. Implementation guide

Every edit is given exactly. **Apply them in order and do not paraphrase the
code or the copy.** Anchors are verbatim strings from the current
`tools/bracket-generator.html`; the line numbers are pre-edit hints only and
will shift as you go — match on the string, not the number.

Nothing outside these steps changes. Do not reformat, re-indent or "tidy"
surrounding code.

### 13.1 CSS — three insertions

**(a) The inline link-button.** Find (`:205`):

```css
  .field-hint{ text-align:left; }
```

Insert immediately after it:

```css
  /* Inline "How it works" trigger. A <button>, not an <a> — it opens a dialog
     rather than navigating — styled to read as a link inside the hint row. */
  .link-btn{
    appearance:none;
    background:none;
    border:none;
    padding:0;
    font:inherit;
    color:var(--green-dark);
    font-weight:700;
    text-decoration:underline;
    cursor:pointer;
  }
  .link-btn:hover{ color:var(--navy); }
```

**(b) The results seed line.** Find (`:321`):

```css
  .result-title{
```

Insert immediately **before** it:

```css
  #resultSeed{
    font-family:'Barlow Condensed',sans-serif;
    font-size:11.5px;
    letter-spacing:.06em;
    text-transform:uppercase;
    color:var(--ink-soft);
    margin-top:6px;
  }
```

**(c) The dialog.** Find (`:502`):

```css
  @media (max-width:600px){
```

Insert immediately **before** it:

```css
  /* ---------- HOW IT WORKS DIALOG ---------- */
  dialog.how-dialog{
    max-width:560px;
    width:calc(100% - 32px);
    max-height:80vh;
    padding:0;
    border:none;
    border-radius:16px;
    background:var(--white);
    color:var(--ink);
    box-shadow:0 30px 70px -30px rgba(11,24,38,.6);
  }
  dialog.how-dialog::backdrop{ background:rgba(11,24,38,.55); }
  .how-head{
    display:flex;
    align-items:center;
    justify-content:space-between;
    gap:12px;
    padding:18px 22px;
    background:var(--navy);
  }
  .how-head h2{
    font-family:'Archivo Black',sans-serif;
    font-weight:400;
    font-size:17px;
    text-transform:uppercase;
    letter-spacing:.01em;
    color:var(--white);
    margin:0;
  }
  .how-close{
    appearance:none;
    background:none;
    border:none;
    color:var(--white);
    font-size:22px;
    line-height:1;
    padding:2px 8px;
    border-radius:6px;
    cursor:pointer;
  }
  .how-close:hover{ background:rgba(255,255,255,.15); }
  .how-body{
    padding:20px 22px 26px;
    overflow-y:auto;
    max-height:calc(80vh - 62px);
    font-size:14px;
    line-height:1.6;
  }
  .how-lead{
    background:var(--paper-dim);
    border-left:3px solid var(--green);
    border-radius:0 8px 8px 0;
    padding:12px 14px;
    margin:0 0 18px;
  }
  .how-body h3{
    font-family:'Barlow Condensed',sans-serif;
    font-size:15px;
    font-weight:700;
    letter-spacing:.06em;
    text-transform:uppercase;
    color:var(--green-dark);
    margin:22px 0 8px;
  }
  .how-body p{ margin:0 0 12px; }
  .how-body ol, .how-body ul{ margin:0 0 12px; padding-left:20px; }
  .how-body li{ margin-bottom:8px; }
  .how-body code{
    font-family:ui-monospace,Menlo,Consolas,monospace;
    font-size:12.5px;
    background:var(--paper-dim);
    padding:1px 5px;
    border-radius:4px;
  }
```

> The `rgba(11,24,38,…)` shadow and backdrop are literals, matching the shadow
> convention already used throughout this file (`.hero`, `.input-panel`). The
> one-palette-source rule in §4.1 of the bracket generator spec governs *fills
> and text*, which here all resolve from tokens. Do not "fix" these two.

### 13.2 HTML — four insertions

**(a) The seed field.** Find (`:530`–`:531`) — the end of the Event name field
and the start of the Category field:

```html
    </div>
    <div class="field">
      <label class="field-label" for="categoryInput">Category name</label>
```

Replace with:

```html
    </div>
    <div class="field">
      <label class="field-label" for="seedInput">Seed <span class="field-optional">optional</span></label>
      <input id="seedInput" class="field-input" type="text"
             placeholder="e.g. MANGO, 42, 7-3" autocomplete="off" maxlength="40" />
      <div class="field-meta">
        <span class="field-hint">For a public draw, ask someone in the room for a word or number &mdash; it decides the draw, and anyone can check it afterwards. Leave blank and one is generated for you. <button type="button" class="link-btn" id="howItWorksLink">How it works</button></span>
      </div>
    </div>
    <div class="field">
      <label class="field-label" for="categoryInput">Category name</label>
```

**(b) The results seed line.** Find (`:561`):

```html
      <div class="result-meta" id="resultMeta"></div>
```

Replace with:

```html
      <div class="result-meta" id="resultMeta"></div>
      <div id="resultSeed"></div>
```

**(c) The second dialog entry point.** Find (`:567`):

```html
      <button type="button" class="action-btn" id="exportTextBtn">Export as text</button>
```

Replace with:

```html
      <button type="button" class="action-btn" id="exportTextBtn">Export as text</button>
      <button type="button" class="action-btn" id="howItWorksBtn">How it works</button>
```

**(d) The dialog itself.** Find (`:577`):

```html
<script>
```

Insert immediately **before** it (after the `</footer>` line):

```html
<dialog class="how-dialog" id="howDialog" aria-labelledby="howDialogTitle">
  <div class="how-head">
    <h2 id="howDialogTitle">How this draw works</h2>
    <button type="button" class="how-close" id="howCloseBtn" aria-label="Close">&times;</button>
  </div>
  <div class="how-body">
    <p class="how-lead"><strong>The short version:</strong> the brackets are decided by a single short word or number &mdash; the <em>seed</em> &mdash; and anyone can check afterwards that the result really came from it. At a public draw, that word comes from someone in the room.</p>

    <p><strong>1. Everyone sees the list first.</strong> Before anything is drawn, the full list of pairs is up on screen. Nothing can be added or taken away after that without it being obvious.</p>

    <p><strong>2. A seed is set.</strong> A word, a number, anything. For a public draw it should be called out by a player rather than the organiser &mdash; then typed in and shown on screen, so you can photograph it. If nobody supplies one, the app generates one and displays it, and the sheet says which of the two happened.</p>

    <p><strong>3. Every pair gets a fingerprint.</strong> The computer takes the seed and a pair's names together and turns them into a long code &mdash; a fingerprint (technically a <em>hash</em>) that looks like <code>07a3f91c&hellip;</code></p>

    <p>The same names with the same seed always give the same fingerprint. But change a single letter of the seed and every fingerprint changes completely. Nobody can work out ahead of time what fingerprint a pair will get.</p>

    <p>It is not a code we invent. It is a standard calculation used all over the world, and there are free websites that will do it for you &mdash; which is what makes it possible for you to check us.</p>

    <p><strong>4. Pairs are sorted by fingerprint, then dealt out.</strong> Lowest fingerprint first, the way you would sort a list alphabetically. Then they are dealt around the brackets one at a time &mdash; first pair to Bracket A, second to Bracket B, and so on &mdash; so every bracket ends up the same size, or within one.</p>

    <p><strong>5. That is the whole draw.</strong> Because the fingerprints come from the seed, the result was already settled the moment the seed was set. Pressing the button does not roll anything &mdash; it just shows what the seed had already decided. The few seconds of shuffling on screen are there to make the reveal watchable, not to pick the winner.</p>

    <h3>Where did this seed come from?</h3>
    <p>Look at the sheet. Next to the seed it says one of two things.</p>
    <p><strong>(entered)</strong> &mdash; a person typed it in. At a public draw that should be a player calling it out, not the organiser.</p>
    <p><strong>(auto)</strong> &mdash; nobody supplied one, so the app made one up at the moment of the draw.</p>
    <p>Either way the result can still be checked afterwards, and a bracket that has been altered will not match. The difference is what it proves about <em>how the seed was chosen</em> &mdash; only a seed called out in the room shows the organiser didn't go hunting for one they liked.</p>

    <h3>Why this is hard to rig</h3>
    <ul>
      <li><strong>Nobody can predict what a seed will produce.</strong> There is no way to look at a word and know which bracket it will hand you &mdash; you have to run it.</li>
      <li><strong>Pressing again changes nothing.</strong> The same seed always gives the same brackets. A different result needs a different seed &mdash; and at a public draw that means asking the room again, out loud, in front of everyone.</li>
      <li><strong>The sheet counts the draws.</strong> If it says Draw #3, three seeds were tried on this list. That number cannot be hidden.</li>
      <li><strong>The exported file records everything</strong> &mdash; the seed, every pair, and every fingerprint &mdash; so the draw can still be checked long after the day.</li>
    </ul>

    <h3>Check it yourself</h3>
    <p>You do not have to take our word for any of this, and you do not need to understand all of it to check your own pair.</p>
    <ol>
      <li>Search the web for <strong>"SHA-256 calculator"</strong> and open any of them.</li>
      <li>Type in the seed, a <code>|</code> character, then your pair exactly as it appears on the sheet &mdash; for example <code>MANGO|Player 1 / Player 2</code></li>
      <li>Compare the start of the result to the fingerprint printed next to your name in the exported list. They should match.</li>
    </ol>
    <p>Pairs with lower fingerprints were dealt first. If your fingerprint is in the right place in that order, your bracket is exactly where the draw put it.</p>
  </div>
</dialog>

```

### 13.3 JS — the verifiable-draw helpers

Find (`:587`):

```js
function shuffle(arr){
```

Insert immediately **before** it:

```js
// ==================== Verifiable draw ====================
// The draw is a deterministic function of (seed, pair list): each pair is
// fingerprinted with SHA-256 and the list sorted by that fingerprint, so anyone
// can reproduce it without this tool — including on a third-party SHA-256 site,
// which is the whole point. Spec: sage-docs/docs/specs/
// bracket-generator-verifiable-draw-spec.md
const SEED_ALPHABET = '23456789ABCDEFGHJKMNPQRSTVWXYZ'; // no I/O/0/1 — read aloud safely
const AUTO_SEED_LENGTH = 8;

// Trim + uppercase, and the field is rewritten to match, so what is displayed
// is exactly what was hashed. If those two can ever differ, every external
// check fails and the feature is decorative.
const normalizeSeed = s => String(s).trim().toUpperCase();

function generateAutoSeed(){
  const bytes = new Uint8Array(AUTO_SEED_LENGTH);
  crypto.getRandomValues(bytes);
  return [...bytes].map(b => SEED_ALPHABET[b % SEED_ALPHABET.length]).join('');
}

async function sha256Hex(s){
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s));
  return [...new Uint8Array(buf)].map(b => b.toString(16).padStart(2, '0')).join('');
}

// Sorts on the FULL hash — the 8-char form in the export is display only, and a
// truncated sort key would start colliding around 65k items. Ties can only come
// from two identical pair strings (identical input hashes identically), so they
// fall back to the pair text and then input position, keeping the order defined.
async function fingerprintPairs(pairs, seed){
  const hashes = await Promise.all(pairs.map(p => sha256Hex(seed + '|' + p)));
  return pairs
    .map((pair, i) => ({ pair, hash: hashes[i], i }))
    .sort((a, b) => a.hash < b.hash ? -1 : a.hash > b.hash ? 1 :
                    a.pair < b.pair ? -1 : a.pair > b.pair ? 1 : a.i - b.i);
}

```

Then, **immediately after** that insertion, amend the comment on `shuffle()`
itself. Find:

```js
function shuffle(arr){
```

Replace with:

```js
// Animation only. The real draw is fingerprintPairs() above — this feeds
// runShuffleAnimation()'s cosmetic per-tick re-deals, which are thrown away and
// never exported, so they need no reproducibility. Do not wire dealBrackets()
// back to this.
function shuffle(arr){
```

### 13.4 JS — state and elements

**(a)** Find (`:627`):

```js
let currentBrackets = []; // [{ label, pairs: [] }]
```

Replace with:

```js
let currentBrackets = []; // [{ label, pairs: [] }]
let currentSeed = '';
let currentSeedSource = '';   // 'entered' | 'auto'
let currentFingerprints = []; // [{ pair, hash }] in draw order
let drawCount = 0;
let lastDrawInputKey = '';    // drawCount resets when the category or pairs change
```

**(b)** Find (`:630`):

```js
const eventNameInput = document.getElementById('eventNameInput');
```

Replace with:

```js
const eventNameInput = document.getElementById('eventNameInput');
const seedInput = document.getElementById('seedInput');
```

**(c)** Find (`:646`):

```js
const exportTextBtn = document.getElementById('exportTextBtn');
```

Replace with:

```js
const exportTextBtn = document.getElementById('exportTextBtn');
const resultSeedEl = document.getElementById('resultSeed');
const howDialog = document.getElementById('howDialog');
const howItWorksLink = document.getElementById('howItWorksLink');
const howItWorksBtn = document.getElementById('howItWorksBtn');
const howCloseBtn = document.getElementById('howCloseBtn');
```

### 13.5 JS — the secure-context guard and dialog wiring

Find (`:663`):

```js
eventNameInput.addEventListener('change', () => saveEventName(eventNameInput.value.trim()));
```

Insert immediately after it:

```js

// ==================== How it works dialog ====================
const openHowDialog = () => howDialog.showModal();
howItWorksLink.addEventListener('click', openHowDialog);
howItWorksBtn.addEventListener('click', openHowDialog);
howCloseBtn.addEventListener('click', () => howDialog.close());
// Clicking the backdrop lands on the <dialog> itself rather than its contents.
howDialog.addEventListener('click', e => { if(e.target === howDialog) howDialog.close(); });

// ==================== Secure-context guard ====================
// SubtleCrypto needs a secure context. GitHub Pages is HTTPS, so this only bites
// a local file:// open. Fail closed rather than falling back to an unverifiable
// shuffle that still prints a seed — an export naming a seed it wasn't produced
// from is a false proof, which is worse than having no feature at all.
const CRYPTO_OK = !!(window.crypto && window.crypto.subtle && window.crypto.getRandomValues);
if(!CRYPTO_OK){
  generateBtn.disabled = true;
  seedInput.disabled = true;
  showFormError('This page has to be served over HTTPS to run a verifiable draw. Open it at https://sage-match-control.github.io/tools/bracket-generator.html');
}
```

### 13.6 JS — `dealBrackets` and `generateBrackets`

Find (`:696`–`:728`) — both functions, entire:

```js
function dealBrackets(pairs, numBrackets){
  const shuffled = shuffle(pairs);
  const buckets = Array.from({ length: numBrackets }, () => []);
  shuffled.forEach((pair, i) => buckets[i % numBrackets].push(pair));
  return buckets.map((list, i) => ({ label: bracketLabel(i), pairs: list }));
}

function generateBrackets(){
```

Replace with:

```js
// Takes an ALREADY-ORDERED list — the ordering is the draw, and it comes from
// fingerprintPairs(). Deals round-robin so bracket sizes differ by at most one.
function dealBrackets(orderedPairs, numBrackets){
  const buckets = Array.from({ length: numBrackets }, () => []);
  orderedPairs.forEach((pair, i) => buckets[i % numBrackets].push(pair));
  return buckets.map((list, i) => ({ label: bracketLabel(i), pairs: list }));
}

async function generateBrackets(){
```

Then find (`:720`–`:727`) — the tail of `generateBrackets`:

```js
  currentCategory  = categoryInput.value.trim();
  currentEventName = eventNameInput.value.trim();

  // Lock in the real result up front so the animation is purely cosmetic —
  // what the user sees settle on is exactly what gets rendered/exported.
  const finalDeal = dealBrackets(pairs, numBrackets);

  runShuffleAnimation(pairs, finalDeal);
}
```

Replace with:

```js
  currentCategory  = categoryInput.value.trim();
  currentEventName = eventNameInput.value.trim();

  // A blank seed gets one generated and written back to the field, so there is
  // always a seed and every draw is reproducible. Only *who chose it* varies,
  // which is what the (entered)/(auto) label on the exports records.
  const typedSeed = normalizeSeed(seedInput.value);
  currentSeedSource = typedSeed ? 'entered' : 'auto';
  currentSeed = typedSeed || generateAutoSeed();
  seedInput.value = currentSeed;

  // Counts seeds tried against THIS input, so a re-roll is on the record rather
  // than invisible. Same seed re-drawn gives the same brackets, so the number
  // only advances meaningfully when the seed changed too.
  // JSON.stringify rather than a delimiter: no separator can collide with a
  // category name or a pair, so the key is unambiguous by construction.
  const inputKey = JSON.stringify([currentCategory, pairs]);
  if(inputKey !== lastDrawInputKey){ drawCount = 0; lastDrawInputKey = inputKey; }
  drawCount += 1;

  // Disabled across the await so a double-press can't start two overlapping
  // draws; runShuffleAnimation's setControlsDisabled takes over from here.
  generateBtn.disabled = true;
  let ordered;
  try {
    ordered = await fingerprintPairs(pairs, currentSeed);
  } catch(e){
    generateBtn.disabled = false;
    showFormError('Could not compute the draw fingerprints. Reload the page and try again.');
    return;
  }
  currentFingerprints = ordered.map(o => ({ pair: o.pair, hash: o.hash }));

  // Lock in the real result up front so the animation is purely cosmetic —
  // what the user sees settle on is exactly what gets rendered/exported.
  const finalDeal = dealBrackets(ordered.map(o => o.pair), numBrackets);

  runShuffleAnimation(pairs, finalDeal);
}
```

### 13.7 JS — the results header

Find (`:730`), the opening of `renderResultsHeader`:

```js
function renderResultsHeader(){
```

Replace with:

```js
function renderResultsHeader(){
  resultSeedEl.textContent =
    `Seed ${currentSeed} (${currentSeedSource}) \u00b7 Draw #${drawCount}`;
```

### 13.8 JS — the text export

Find (`:889`–`:892`):

```js
  out += currentEventName
    ? `${currentEventName} \u00b7 Bracket Draw\n`
    : 'S.A.G.E. Bracket Draw\n';
  out += 'Powered by S.A.G.E. Match Control Experts\n';
```

Replace with:

```js
  // The audit trail travels with the artifact: seed, every fingerprint, and the
  // instructions to re-check it. Self-describing on purpose — there is no hosted
  // verification page to keep alive, and the file still works years later.
  out += 'DRAW VERIFICATION\n';
  out += `${'-'.repeat(24)}\n`;
  out += `Seed:   ${currentSeed}  (${currentSeedSource})\n`;
  out += `Draw:   #${drawCount}\n`;
  out += 'Method: SHA-256(seed + "|" + pair), sorted ascending, dealt round-robin\n\n';
  out += 'Fingerprints, in draw order:\n';
  currentFingerprints.forEach(f => { out += `  ${f.hash.slice(0, 8)}...  ${f.pair}\n`; });
  out += '\n';
  out += 'TO CHECK THIS DRAW YOURSELF\n';
  out += 'Search the web for "SHA-256 calculator" and paste in exactly:\n';
  if(currentFingerprints.length) out += `  ${currentSeed}|${currentFingerprints[0].pair}\n`;
  out += 'The result should start with the fingerprint above. Pairs are sorted\n';
  out += 'from lowest fingerprint to highest, then dealt one at a time around\n';
  out += 'the brackets: 1st to Bracket A, 2nd to Bracket B, and so on.\n\n';
  out += 'If two fingerprints match on the digits shown, compare more of them —\n';
  out += 'the order is decided by the full code, not the first eight characters.\n\n';

  out += currentEventName
    ? `${currentEventName} \u00b7 Bracket Draw\n`
    : 'S.A.G.E. Bracket Draw\n';
  out += 'Powered by S.A.G.E. Match Control Experts\n';
```

### 13.9 JS — the image export

**(a) Make room.** Find (`:1106`):

```js
  const bottomH = 56;
```

Replace with:

```js
  const bottomH = 74; // two footer lines: the verification stamp, then branding
```

**(b) The stamp.** Find (`:1191`):

```js
  ctx.fillText(truncateToWidth(ctx, footerText, width - 120), width / 2, height - 24);
```

Replace with:

```js
  ctx.fillText(truncateToWidth(ctx, footerText, width - 120), width / 2, height - 24);

  // Verification stamp. Per-pair fingerprints stay out of the image — they live
  // in the text export — so the poster stays readable on a wall.
  ctx.font = '500 10.5px "Barlow Condensed", sans-serif';
  const verifyText =
    `Seed: ${currentSeed} (${currentSeedSource}) \u00b7 Draw #${drawCount} \u00b7 ` +
    'SHA-256 verifiable — see the text export to check';
  ctx.fillText(truncateToWidth(ctx, verifyText, width - 80), width / 2, height - 42);
```

### 13.10 Verify before calling it done

Syntax-check the inline script. `node --check` refuses a `.html` file and
refuses an extensionless process substitution, so extract to a real `.js` first:

```bash
node -e "const fs=require('fs');const m=fs.readFileSync('tools/bracket-generator.html','utf8').match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync('bg_check_tmp.js',m[1]);" && node --check bg_check_tmp.js && rm bg_check_tmp.js && echo "JS OK"
```

Then **serve it over HTTP** — opening the file directly trips the §4.3 guard and
disables the draw, which is correct behaviour, not a bug:

```bash
python -m http.server 8934
```

and open `http://localhost:8934/tools/bracket-generator.html`. Note that
GitHub Pages resolves extensionless URLs and `http.server` does not, so use the
full `.html` path locally.

Walk §11's checklist. The four that catch the most likely mistakes:

1. Same seed + same list, drawn twice → identical brackets, `Draw #2`.
2. Re-order the pasted list → **same** brackets.
3. `grep -n "shuffle(" tools/bracket-generator.html` → the definition and
   exactly one call, inside `runShuffleAnimation`. **Not** in `dealBrackets`.
4. A fingerprint from the text export reproduces on a third-party SHA-256 site
   from the exact documented string.

---

## 14. Divergences

*(None — this spec is written before the work. Record here anything the built
tool deliberately does differently.)*
