# Bracket Generator

Settle who plays in which bracket, in front of the room if you want, without
wrestling a spreadsheet into a diagram. A standalone tool, no sign-in, nothing
saved anywhere but your own browser.

## How you use it

Type the category name, paste the pairs one per line, say how many brackets
you want, and press **Randomize Brackets**. The draw runs on screen — pairs
shuffle for a few seconds, then land in their brackets — and the result stays
up for you to read off or export. **Randomize again** re-draws the same pairs
from scratch if you want a different split.

## The event name

An optional field above Category name. It's remembered between draws, so
type it once for the first category of the day and every draw after that
carries it too. It changes what the *export* says — the exported image and
text file are stamped with it — not the tool itself; the page you're looking
at always just says "Bracket Generator." Leaving it blank is fine: the export
comes out with plain S.A.G.E. branding instead of an event name.

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

## What comes out

Two export options, once a draw has landed:

- **Export as image** — a printable PNG, laid out as a card per bracket.
- **Export as text** — a plain-text list of every bracket and its pairs.

Filenames include the category (and the event name, if you set one), so
they're easy to find again after exporting eight categories in a row —
`beginner-18-mens-doubles-brackets.png`, or with an event name set,
`bkl-pickleball-cup-2026-beginner-18-mens-doubles-brackets.png`.

## What it doesn't do

It doesn't know your event. Pairs are typed in by hand — nothing is pulled
from a registration list or `event-data` — and nothing the tool produces is
published anywhere; you export a file and post or print it yourself. The draw
itself is uniformly random with no rankings and no protected pairings:
it won't keep training partners or club-mates apart, and it doesn't try to.

---
**Technical:** [bracket generator](../technical/bracket-generator.md)
