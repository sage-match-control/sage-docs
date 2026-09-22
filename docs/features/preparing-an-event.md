# Preparing an event

Everything that has to exist before tournament day, in the order it has to
exist in. This is the setup half; the other half, once the doors open, is
[Running an event, day of](running-an-event-day.md).

Read this first if you're setting up an event for the first time, or if
you're handing the job to someone else. Each step links to the page that
covers it properly — the value here is the *sequence*, and knowing what
depends on what.

## Quickstart

The whole sequence on one screen, for when you've done this before. Each
step links to its full section below.

1. **Pick the event key** — e.g. `pnf-x-bup-dual-meet` — and use it
   byte-identically everywhere from here on.
2. **[Plan it](#1-plan-it)** in the Tournament Calculator: format,
   categories, courts, match duration, one plan per day. Then click
   **Copy plan & open generator**.
3. **[Build the scoring workbook](#2-build-the-scoring-workbook)**, one per
   venue per day: copy the format's master, run **SAGE → Generate event
   tabs**, paste the plan. A standard tournament's `SCHEDULE` is then packed
   by hand from its `MATCHES` tab.
4. **[Draw the brackets](#3-draw-the-brackets-and-enter-the-names)** in the
   Bracket Generator, one category at a time, saving both the image and the
   text file. Enter each category's names from its text file, then paste the
   codes back shuffled.
5. **[Build the event site](#4-build-the-event-site)**: copy the matching
   template into `events/<event-key>/`, replace the tokens, add the QR code
   (and both clubs' logos for a dual meet).
6. **[Register it and wire up the sync](#5-register-it-and-wire-up-the-sync)**:
   add the event to `event-data/config/events.json`, then in every workbook
   run **SAGE → Fill match numbers** (a separate range per venue) and
   **SAGE → Set up live sync**.
7. **[Sync, and check the site](#6-sync-and-check-the-site)** — from the
   workbook's SAGE menu or from Control Center.
8. **[Export the schedule PDF](#7-export-the-schedule-pdf)** from the event's
   schedule page, for the desk and the noticeboard.
9. **[Rehearse it](#8-rehearse-it)** with the dry-run checklist: a fake score
   per venue reaches Control Center and the public page, then gets reverted.
10. **[Generate the scoresheets](#9-generate-the-scoresheets)** once the
    schedule is final.
11. **Check it against [what "ready" looks like](#what-ready-looks-like).**

## The four resources

Every event, whatever its shape, needs these four things. They are separate
artifacts owned in separate places, which is why the order matters.

| # | Resource | Where it lives | What it's for |
|---|---|---|---|
| 1 | **The plan** | Tournament Calculator, exported as a CSV | How many matches, how many courts, what time it ends |
| 2 | **The scoring workbook** | Google Sheets, one per event | Where scores get typed and standings are computed |
| 3 | **The event site** | `sage-match-control.github.io/events/<event-key>/` | What players and spectators see |
| 4 | **The registration** | `event-data/config/events.json` | What tells the sync this event exists |

They chain: the plan sizes the workbook, the workbook feeds the site, and the
registration is what connects the two. Skip ahead and you'll be back-filling.

> **Pick the event key once.** `<event-key>` (e.g. `pnf-x-bup-dual-meet`) is
> the event's folder name in *both* the site repo and `event-data`, and it
> ends up in the workbook's `Title` tab. Choose it at step 1 and keep it
> byte-identical everywhere after that.

## 1. Plan it

In the [Tournament Calculator](tournament-calculator.md): set the format,
add the categories, set courts and match duration, and check the finish time
is one the venue will tolerate.

This is the cheapest place to discover the event doesn't fit — moving a
number here costs nothing, moving it after the workbook is generated means
regenerating.

Settle these before moving on, because everything downstream is sized from
them:

- **Format** — dual meet, or standard tournament. This decides which site
  template you use and which master workbook and generator build the
  scoring workbook.
- **Categories**, with pairs per club (dual meet) or team counts (standard).
- **Courts** and **match duration**, including any buffer between matches.
- **Days and venues** — one plan per tournament day, if the event runs
  across several.

Export the plan when it's right. Use **Copy plan & open generator** rather
than the file export — it puts the plan on your clipboard and opens the copy
dialog of the master for the plan's format, dual meet or standard, in one
step.

The draw comes later, in step 3: the workbook is what tells you how many
brackets each category ended up with, and how many pairs are in each.

## 2. Build the scoring workbook

**Dual meet:** use the
[Dual Meet Sheet Generator](dual-meet-sheet-generator.md). Copy the master,
run `SAGE → Generate event tabs`, paste the plan. You get every category tab,
the schedule, and the four readout tabs the live sync publishes from.

**Standard tournament:** use the
[Standard Tournament Generator](standard-tournament-generator.md), once per
venue per day. Copy the master, run `SAGE → Generate event tabs`, paste the
plan, then tick that venue's categories and enter its court labels. You get
every category tab with its playoff ladder, a `MATCHES` tab holding every
match, an empty `SCHEDULE`, and the readout tabs. **Pack `SCHEDULE` now**, by
copying court blocks from `MATCHES` into its slots — that page has the rules
and the paste-special step, and the generator's log suggests a layout you can
follow.

Either way the generator renames the workbook after the event and venue when
it finishes, so a folder of fifteen of them stays readable.

Two things to know before you run it:

- **Work in your copy, never the master.** The generator builds several tabs
  in place, so it can only run once per workbook — if a run fails partway, or
  you want to change the plan, you start again from a fresh copy of the
  master. That is deliberate: a half-regenerated workbook mid-event is the
  worst outcome available.
- **The names come next, not now.** The plan has no roster in it. The
  generator leaves the name-entry columns blank and tells you so when it
  finishes; until they're filled, the workbook is structurally complete but
  has nobody in it.

A generated workbook carries the live-sync script already — it's built into
the master — so it's ready for step 5's sync setup with no code-pasting step
of its own.

One workbook per venue per day. An event with three venues across five days
has fifteen of them.

## 3. Draw the brackets and enter the names

Now that the workbook exists, it says how many brackets each category has and
how many pairs go in each. Draw them, one category at a time, in the
[Bracket Generator](bracket-generator.md):

1. Put the event name in, so every export is stamped with it.
2. Paste that category's pairs, set the bracket count to the one the workbook
   used, and run the draw — ideally with a seed called out in the room, which
   is what makes the draw checkable afterwards.
3. **Save both exports:** the **image**, which is what you post and print, and
   the **text file**, which is the one you work from next. The text file also
   carries the seed and every pair's fingerprint, so the draw can be
   re-checked months later.
4. Repeat for every category. Eight categories means eight images and eight
   text files.

Then fill each category tab in the workbook, from that category's text file:

- **`STEP 1 · NAMES`** — each pair's two players, two rows per pair, in the
  order the text file lists them: bracket A's pairs first, then bracket B's,
  and so on. That order is what puts each pair in the right bracket on the
  tab, in the standings, and in its score grid.
- **`STEP 3 · RANDOMIZED CODES`** — the codes from `STEP 2`, pasted back in a
  shuffled order. It ships blank on purpose: pasting them in order maps every
  pair to its own slot and undoes the blinding.

Check as you go that each category tab's `B` column fills in with names once
a pair's code is linked. A tab still showing codes instead of names means the
roster and the codes haven't met.

> **Coming later:** uploading those text files straight into the workbook,
> so the names and the shuffled codes are filled in from the draw itself. The
> design is written up in
> [bracket draw name import](../specs/not-started/bracket-draw-name-import-spec.md);
> until it's built, this step is typing and pasting.

## 4. Build the event site

Copy the matching template — `dual-meet-template/` or
`standard-tournament-template/` — into `events/<event-key>/`, replace the
tokens, and fill in the days, facilities and categories.

The step-by-step is [adding a new event](../technical/adding-a-new-event.md);
the exhaustive version, with the full token table, is `_templates/CLAUDE.md`
in the site repo. You'll also need the event's images ready: a QR code, and
both clubs' logos for a dual meet.

## 5. Register it, and wire up the sync

Three connections, and the event is inert until all of them exist:

- **Register the event** in `event-data/config/events.json` — its type,
  title, and one entry per day with that day's venues. This takes effect
  within minutes of the commit; nothing needs redeploying.
- **Number the matches** in each venue's spreadsheet with **SAGE → Fill match
  numbers**. It numbers every match on SCHEDULE, starting after the number
  you enter. When a day has more than one venue, give each venue's workbook
  its own range, for example 1000 for one (1001, 1002, …) and 2000 for the
  other. All of a day's venues show together on the site, so two matches
  must never share a number.
- **Wire up the sync** in each venue's spreadsheet, once per workbook: reload
  it and run **SAGE → Set up live sync**, entering that workbook's day key
  and facility name. This is what makes typing a score publish it.

The facility names in the config and in each spreadsheet's own settings are
compared exactly and are **case-sensitive** — but setup doesn't leave that to
chance: it confirms the entered day key and facility name against the
registry (and runs a real test sync) before saving, so a mismatch is caught
on the spot rather than discovered as a venue that silently never publishes.

## 6. Sync, and check the site

Scores publish on their own during the event, about ten seconds after the
typing stops. Before the event there is nothing to wait for, so push the
first copy up yourself. Either way works:

- **From the workbook** — **SAGE → Sync now**, in each venue's spreadsheet.
  It reports what it sent.
- **From [Control Center](control-center.md)** — Mission Control's **Resync
  this day now**, which pulls a fresh copy from every venue of that day at
  once. Handy when you have several workbooks open and don't want to visit
  each.

Then open the event page and confirm what you'd expect: every day and
category present, every venue listed, the schedule showing the matches you
packed, and the standings listing every pair by name. This is the first point
at which a wrong facility name or a missing column shows itself.

## 7. Export the schedule PDF

The event's **schedule page** carries a `PDF` button. It lays the board out
one sheet per group of courts and hands it to the browser's print dialog, so
"Save as PDF" gives you a file and "Print" gives you the paper copy for the
desk, the noticeboard and each court.

Export it once the schedule is packed, numbered and synced, so the PDF, the
website and the workbook all say the same thing. Re-export it if the schedule
changes.

## 8. Rehearse it

Copy the dry-run checklist into the event's folder and work through it, with
the real workbooks and the real site, before the day.

The rehearsal is the same shape as the day itself: enter a few fake scores,
confirm they reach [Control Center](control-center.md) and the public page,
mark a match live and confirm it shows as live, then revert. It is the only
step that exercises the whole chain end to end, and the only place a
case-sensitive facility name or a missing column is cheap to find.

## 9. Generate the scoresheets

Last, once the schedule is final and the rehearsal has proven a sync reaches
the public page: print the match slips with the
[Scoresheet Generator](scoresheet-generator.md). Because the event is
registered and synced by now, the generator can pull the day's matches
straight from that event/day/venue instead of you downloading a CSV out of
the workbook.

Generate them after the schedule stops moving. A scoresheet carries its match
number, court and time, so a reshuffle after printing means printing again.

## What "ready" looks like

- The plan's finish time is one the venue will accept.
- Every category has a drawn bracket, with its image and text file saved.
- Every category tab has its rosters pasted in, its codes shuffled, and no
  tab shows an error.
- A standard tournament's `SCHEDULE` is packed and numbered in every venue's
  workbook.
- The event page loads, shows the right days and categories, and lists every
  venue.
- A test score typed into each venue's workbook reaches Control Center and
  the public page, and has been reverted.
- Every venue reports a recent successful sync.
- The schedule PDF matches the site, and the scoresheets are printed.

At that point the event is ready, and the rest is
[day-of operations](running-an-event-day.md).

## Where dual meets differ

- The scoring workbook comes from the **Dual Meet Master** and its
  generator, which also places every match on `SCHEDULE`. A standard
  tournament's generator leaves `SCHEDULE` for you to pack from `MATCHES`.
- The site uses `dual-meet-template/`, which frames every category, the
  standings, and the Awards tab's "Overall Champion" line as Club A vs Club B.
- The plan needs **pairs per club** and a bracket count per category, and the
  calculator accounts for the cross-club Bronze and Final matches that
  produces.
- You need both clubs' logos among the event images.

---
**Technical:** [adding a new event](../technical/adding-a-new-event.md) ·
[dual meet sheet generator](../technical/dual-meet-sheet-generator.md) ·
[standard tournament generator](../technical/standard-tournament-generator.md)
