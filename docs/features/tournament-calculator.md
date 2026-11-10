# Tournament Calculator

**Used in:** [1. Plan it](../usage/plan-the-event.md) ·
[2. Build the workbooks](../usage/build-the-workbooks.md)

Set up your categories and court plan, and see instantly how many matches
you're running and what time the last match finishes — before the draw is
even final. A standalone planning tool; nothing here is live tournament data.

## What it's for

Answering "will this actually fit in the time and courts we have" during
planning, not on the day of. Add your categories, how many courts you have,
and match duration/buffer time, and it works out the full schedule math for
you — total matches, court-hours needed, and a finish time.

## Two formats

- **Standard tournament** — an open bracket event. Each category gets its own
  team count, bracket group size, and how many advance to playoffs.
- **Dual meet (club vs. club)** — two named clubs face off. Each category
  gets a **pairs per club** count (both clubs play the same number of pairs)
  and a bracket count, and the calculator accounts for the cross-club
  Bronze/Final matches that format produces.

## How the round robin is played

Every standard category with a round robin has a row of three choices:

- **Single round robin** — every pair plays every other pair once, as one game.
  This is the default.
- **Double round robin** — every pairing is played twice, as a first and a second
  meeting. It is for a category with **one bracket** only, so the button is
  disabled with two or more.
- **Best-of-3 meetings** — every pairing meets once, and the meeting is a
  best-of-3: the first pair to win two games takes it, and game 3 is played only
  at 1–1. It works with any number of brackets. In the standings, W and L count
  meetings won and lost, and the quotient counts the points of every game. The
  playoff is the one the category would have anyway, each match one game.

The choices exclude each other: pressing one releases the others. A 2-pair
category has no round robin (it plays a best-of-3 final), and a dual meet has
neither choice.

A best-of-3 round robin is planned at its full length, every meeting three games,
with the least (every meeting won 2–0) shown beside it as a range, for example
`28–40 matches`. When meetings end early the day finishes earlier than planned.
A double round robin of more than 6 pairs, and best of 3 in a bracket of more
than 5, carry a note suggesting more brackets.

## Single-bracket categories

A category with only one bracket has no semifinal to play, so it offers a
choice of how the medals get decided:

- **Twice-to-beat final** — the round robin's #1 and #2 meet in a final that #1
  only has to win once and #2 has to win twice. #3 takes bronze with no
  match. Adds up to two matches on top of the round robin.
- **Final + bronze match** — one game between the round robin's #1 and #2 for
  gold, and one between #3 and #4 for bronze. Needs 4 pairs; the button is
  disabled below that, and a category saved on it that drops to 3 pairs plays
  *Final, #3 bronze* instead, with a note.
- **Final, #3 bronze** — one game between #1 and #2 for gold. #3 takes bronze
  with no match.
- **Round robin only** — no playoff at all. The final standings decide gold,
  silver and bronze outright.

The choice is per category, so one division can run a final while another
settles on standings. Round robin only is the shortest option, and the
projected finish time updates as soon as you switch. All four formats go with
either a single or a double round robin, and with best-of-3 meetings.

## Exporting

A **Breakdown** sheet exports to `.xlsx` — a full match-by-match schedule you
can hand to whoever's running registration, or use as a starting point for
the actual matches CSV.

## Handing the plan to a sheet generator

Under the export buttons, **Copy plan & open generator** copies the plan to
your clipboard and opens the copy dialog for the master scoring workbook, so
you can make your own copy and paste the plan straight into its generator —
no file to download and re-open. Which master it opens follows the format:

- **Dual meet** — the Dual Meet Master, for the
  [Dual Meet Sheet Generator](dual-meet-sheet-generator.md).
- **Standard tournament** — the Standard Tournament Master, for the
  [Standard Tournament Generator](standard-tournament-generator.md). Make one
  copy per venue per day; in each, tick that venue's categories and enter its
  court labels in the sidebar.

If the clipboard is unavailable (some browsers block it on non-secure pages),
the button says so and you can fall back to **Export CSV** and paste the
file's contents instead.

## Works offline

The calculator can be installed like an app (from Chrome's install prompt on
Android, or "Add to Home Screen" on iOS Safari), and works with no network at
all once installed — useful for schedule planning at a venue with unreliable
wifi. Your plan is also remembered automatically between visits, even in a
plain browser tab.

This is the only page on the S.A.G.E. site that works this way. It's safe
here because nothing it shows is live tournament data that could go
stale — everywhere else on the site, showing an out-of-date score would be
actively misleading, so no other page is installable or offline-capable.

---
**Technical:** [dual-meet + PWA implementation](../technical/tournament-calculator.md)
