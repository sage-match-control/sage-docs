# Tournament Calculator

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

## Exporting

A **Breakdown** sheet exports to `.xlsx` — a full match-by-match schedule you
can hand to whoever's running registration, or use as a starting point for
the actual matches CSV.

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
