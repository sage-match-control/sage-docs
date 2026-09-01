# Control Center

Control Center is the operator's console for running a tournament day —
one page, covering every registered event, always showing live data (unlike
Tournament Hub, it never hides scores while previewing a day before it's
publicly live). Open it, pick an event and a day, and five tabs appear.

No sign-in is needed to view any of the first four tabs — they're read-only.
Only the actions inside Mission Control (resyncing, forcing the public site
live or hidden) need an operator to sign in.

## Live Matches

A board of every court currently in use, grouped by venue if the event has
more than one. Each court shows the category and round of whatever's playing
there, both teams' names, and the live score — or "No match playing" for an
idle court. This is the same view a wall-mounted screen at the venue would
show.

## Match Finder

Identical to Tournament Hub's Match Finder — search a pair's name, see
their full day of matches with Live/Next Up flags and scores. Useful for an
operator fielding "where's my match" questions without having to also pull up
Tournament Hub itself.

## Standings

Win/loss records and rankings by category. For a **dual meet**, standings are
split by club within each category (with a shared win-count summary at the
top), and the Bronze/Final rounds — since those pit one club against the
other — are shown as a single head-to-head block rather than nested under
either club.

If a match's team code doesn't match any configured category, a visible
warning banner names it rather than silently lumping it into an "Other"
bucket — so a data problem in the spreadsheet gets noticed instead of hidden.

## Awards

The podium tab: once a category's Final and Bronze matches are scored, this
tab shows who won gold, silver, and bronze — derived automatically from the
match data, nothing to type in separately. Before those matches are played,
every placing just reads **Pending**; nothing is ever guessed.

**Overall Champion** *(dual-meet events only)* — a banner at the top naming
whichever club has more Round Robin wins overall, with the runner-up's win
count shown underneath. If the two clubs are tied, it says so rather than
picking one.

**Exporting for the ceremony** — every category card has an **Export image**
button that downloads a SAGE-branded PNG of that category's podium, ready to
hand to an emcee or post on social media. An **Export whole tournament**
button at the top produces a single combined image covering every category
at once, arranged as a grid so it stays a reasonable shape regardless of how
many categories the event has.

A few things the Awards tab is careful about:

- A category decided by walkover (no bronze match actually needed to be
  played) shows a **Walkover** tag on that medalist instead of pretending a
  match happened.
- If a match's scores look wrong (e.g. tied, which shouldn't be possible), the
  card shows a warning naming the match number instead of guessing a winner.
- A category with no playoff bracket at all (pure round robin) falls back to
  its top-three standings, tagged **By standings** so it's clear where that
  podium came from.

## Mission Control

The operator-only control panel — the one part of Control Center that needs
signing in.

- **Sign in** with the shared operator username/password. This exchanges your
  password for a short-lived session token held only in that browser tab —
  the password itself is never stored anywhere.
- **Public site status** — the kill switch for Tournament Hub's Live
  Matches/Standings (this console's own tabs are unaffected and always show
  live data, for preview). Three states: `Auto` (goes live automatically
  ~4 hours before the day's first scheduled match), `Force live`, or
  `Force hidden` — for quietly correcting a bad score before anyone sees it.
- **Facility sync status** — whether each venue's spreadsheet is syncing
  cleanly, and when it last succeeded. Each row has a **Google Sheet**
  link that opens that facility's actual Google Sheet for this event, in
  a new tab — for checking or correcting a score straight at the source
  without having to go find it in Drive. Whether it opens editable
  depends on that sheet's own Google sharing settings, not on Control
  Center.
- **Resync this day now** — pulls a fresh copy from the spreadsheet(s)
  immediately, instead of waiting for the next automatic sync.
- **Public pages** — launchers for the venue's schedule board and the
  event's Tournament Hub, both opening in a new tab so the console stays
  where it is.

---
**Technical:** [Control Center architecture, incl. Awards tab internals](../technical/control-center.md) · [sync pipeline](../technical/sync-pipeline.md) · [auth](../technical/auth.md)
