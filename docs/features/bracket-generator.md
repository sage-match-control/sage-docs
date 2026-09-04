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
itself is uniformly random with no seeding and no protected pairings: it
won't keep training partners or club-mates apart, and it doesn't try to.

---
**Technical:** [bracket generator](../technical/bracket-generator.md)
