# Spec — Bracket Generator: keep-apart groups

> **Status: implemented** (1 October 2026). Built in `tools/bracket-generator.html`.
> Builds on the [Verifiable draw](bracket-generator-verifiable-draw-spec.md); read
> its §4 first.

Let an organiser say "these pairs must not land in the same bracket", without
losing the property that anyone can re-check the draw from the seed and the
exported file.

## 1. The problem

A category has 20 pairs and 4 brackets, and 4 of those pairs should be in
different brackets. The draw sorts pairs by SHA-256(seed | pair) and deals them
round-robin, so any four pairs have only a small chance of ending up in four
different brackets (about 13 draws in 100 for these numbers).

## 2. The rule

Define a **keep-apart group** as a set of pairs. A draw takes zero or more
groups, numbered in the order given, with no pair in two groups.

1. Fingerprint every pair exactly as before: `SHA-256(seed + "|" + pair)`.
2. Build the list to deal: group 1's pairs sorted ascending by fingerprint, then
   group 2's, and so on, then every pair in no group, sorted ascending.
3. Deal that list round-robin, as before: 1st to Bracket A, 2nd to B, …

**Why it works.** In a round-robin deal into N brackets, any N consecutive
positions land in N different brackets. A group of at most N members occupies
consecutive positions, so its members are in different brackets. Bracket sizes
still differ by at most one, and no pair is moved by hand, so nothing the
verifier has to take on trust is added.

**Constraints.** A group has at least 2 and at most N members, where N is the
bracket count actually used (capped at the pair count). A member is a pair
entered on exactly one line; the draw cannot tell identical lines apart.

With no groups, step 2 returns the plain sorted list, so the draw is identical to
the one in the verifiable-draw spec.

## 3. Decisions

- **Groups first, not a re-hash.** Redrawing until a group is spread (hashing
  `seed|pair|attempt`) is also verifiable, but it needs an attempt counter in
  the proof and has no bound that is easy to explain. One extra sentence in the
  method is the cheaper proof.
- **Not moving pairs by hand after the draw.** It would make the printed result
  disagree with the seed, which is what the whole feature exists to prevent.
- **A picker, not a prefix or a second text box.** Members are chosen from the
  entered pairs, so a name cannot fail to match. A prefix on the pair line would
  change the string that is hashed and needs a label per group.
- **No overlapping constraints.** "A not with B, B not with C, A and C may meet"
  is not expressible; a pair belongs to one group.
- **A group smaller than N leaves the last brackets without it.** The group
  occupies the first positions after the groups before it. The bracket labels
  are interchangeable, so this gives no pair an advantage, but it is
  deterministic rather than random. A rotation chosen by hash would remove it
  at the cost of a second hash in the proof.

## 4. What changed in the tool

- A **Keep apart** field under *Number of brackets*: **+ Add a keep-apart group**
  adds a group card; each card has a picker over the entered pairs not yet in a
  group, a chip per member, a status line, and a remove button. Editing the pair
  list drops members that no longer match and says so.
- The draw is refused, before a seed is generated or the draw count moves, when a
  group has fewer than 2 members or more members than brackets.
- The draw counter's input key includes the groups.
- Result cards tag grouped pairs `G1`, `G2`…; the header says how many groups.
- **Text export:** the `DRAW VERIFICATION` block lists fingerprints group by
  group, then "Everyone else", and the method and check instructions describe
  the order. With no groups the export is byte-identical to before. `BRACKET`
  lines are never annotated, because
  [bracket draw name import](bracket-draw-name-import-spec.md) reads every
  numbered line under them as a pair name; the new lines sit in the verification
  block, which the importer skips.
- **Image export:** the verification line gains the group count.
- The *How it works* dialog gains a short section.

## 5. Checked

In a browser: 400 random cases (2–40 pairs, 1–8 brackets, 0–3 groups) all had
every group apart, bracket sizes within one, the same result for the same seed,
and the same result for a shuffled input list. The text export with and without
groups was run through the workbook's `parseDrawText_`, which read the event,
category, seed and brackets correctly, and the no-groups export matched the
pre-change tool's byte for byte (apart from the timestamp). The 20-pair,
4-bracket, one-group case from §1 came out 5/5/5/5 with the group in four
different brackets. The refused cases (group too large, group too small, bracket
count lowered, a member's line deleted or duplicated) behaved as described.

## 6. Out of scope

- Overlapping constraints, or "not with this one pair" rules.
- Remembering groups between visits (category and pairs are not remembered
  either).
- Importing groups from a file or the workbook.
- Rotating which brackets a short group skips.
