# Bracket Generator

**Used in:** [3. Draw and fill rosters](../usage/draw-and-rosters.md) ·
[6. Print](../usage/printables.md)

Settle who plays in which bracket, in front of the room if you want, without
wrestling a spreadsheet into a diagram. A standalone tool, no sign-in, nothing
saved anywhere but your own browser.

## How you use it

Type the category name, paste the pairs one per line, say how many brackets
you want, and press **Randomize Brackets**. The draw runs on screen — pairs
shuffle for a few seconds, then land in their brackets — and the result stays
up for you to read off or export. **Randomize again** re-draws the same pairs
from scratch if you want a different split.

**Opening it from the workbook.** If you're drawing for an event you've
already built a scoring workbook for, open the category's tab and choose
**SAGE → Open Bracket Generator**. The tool opens with the event name and
that category already filled in, so the names on the export match the tab
they're going back into. The category is filled in for that visit only and is
never remembered — a category carried over to a later visit is how someone
draws the wrong one.

## Keeping certain pairs apart

Some categories have pairs that shouldn't meet in the round robin: say four
strong pairs in a category drawn into four brackets. Add them as a
**keep-apart group** and the draw puts every pair of the group in a different
bracket.

1. Paste the pairs first. The group pickers list the pairs you've entered, so a
   name can never be mistyped.
2. Under **Keep apart**, press **+ Add a keep-apart group**, then choose the
   pairs from the **Add a pair…** list. Press × on a pair to take it out again.
3. Add more groups if you need them. A pair can be in only one group.
4. To name a group (say "Top seeds" or "Club A"), press the pencil beside its
   title, type the name and press Enter. Escape cancels, and clearing the name
   goes back to "Group 2" and so on. Names are up to 40 characters.
5. Draw as usual. Each group's pairs are tagged on the result cards with the
   group's name, or `G1`, `G2`… if it has none.

A group can't have more pairs than there are brackets, and needs at least two.
The tool says so on the group and won't draw until it's fixed. If you edit or
remove a pair that is in a group, it drops out of the group and the group tells
you. A pair entered on two lines can't be put in a group.

Which of the group's pairs lands in which bracket is still decided by the seed,
and the draw is just as checkable: the text export lists every group, by name
if you gave it one. The names are only labels for you; they have no effect on
the draw. Groups
are dealt first, in the order they're numbered, then everyone else, so the
sizes stay even (within one pair). When a group has fewer pairs than there are
brackets, the last brackets get none of it. The brackets are otherwise
interchangeable, so that is not an advantage to anyone.

## Verifiable Draw or Random Draw

The **Draw type** switch above the Seed field picks how the draw is made:

- **Verifiable Draw** (the default) — the draw comes from a seed you type or
  leave blank. Pressing again with the same seed gives the same brackets.
  This is the one for a draw people are watching — see below.
- **Random Draw** — nothing to type, and every press gives a fresh split. The
  tool still makes up a seed for each draw behind the scenes and the exports
  mark it `(auto)`, so a Random Draw can be checked afterwards too.

## The event name

An optional field above Category name. It's remembered between draws, so
type it once for the first category of the day and every draw after that
carries it too. It changes what the *export* says — the exported image and
text file are stamped with it — not the tool itself; the page you're looking
at always just says "Bracket Generator." Leaving it blank is fine: the export
comes out with plain S.A.G.E. branding instead of an event name.

## Running a draw people can trust

Every draw is checkable afterwards, and for a draw people are watching there is
a way to run it that makes that obvious. Use **Verifiable Draw** for this.

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
never implies a ceremony that didn't happen. The made-up seed stays in the
field, so pressing again redraws the same brackets (still marked `(auto)`);
clear the field, or switch to **Random Draw**, for a fresh one.

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

## What comes out

Two export options, once a draw has landed:

- **Export as image** — a printable PNG, laid out as a card per bracket.
- **Export as text** — a plain-text list of every bracket and its pairs,
  followed by the seed and every pair's fingerprint.

**Save both, for every category.** The image is what you post and print; the
text file is what fills the scoring workbook, and what anyone re-checking the
draw needs. A standard tournament takes those text files straight into the
workbook — every category at once — with
[**SAGE → Import bracket draws**](standard-tournament-generator.md#filling-the-rosters-from-the-draw),
which reads the names, the bracket order and the seed off them, so nothing is
typed twice. Where this sits in the setup sequence is
[Usage step 3](../usage/draw-and-rosters.md).

Filenames include the category (and the event name, if you set one), so
they're easy to find again after exporting eight categories in a row —
`beginner-18-mens-doubles-brackets.png`, or with an event name set,
`bkl-pickleball-cup-2026-beginner-18-mens-doubles-brackets.png`.

## What it doesn't do

It doesn't know your event. Pairs are typed in by hand — nothing is pulled
from a registration list or `event-data` — and nothing the tool produces is
published anywhere; you export a file and post or print it yourself. Apart
from the [keep-apart groups](#keeping-certain-pairs-apart) you set up, the draw
is uniformly random with no rankings: it won't keep training partners or
club-mates apart unless you tell it to.

---
**Usage:** [3. Draw and fill rosters](../usage/draw-and-rosters.md) ·
**Technical:** [bracket generator](../technical/bracket-generator.md)
