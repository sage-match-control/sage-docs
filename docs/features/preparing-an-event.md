# Preparing an event

Everything that has to exist before tournament day, in the order it has to
exist in. This is the setup half; the other half, once the doors open, is
[Running an event, day of](running-an-event-day.md).

Read this first if you're setting up an event for the first time, or if
you're handing the job to someone else. Each step links to the page that
covers it properly — the value here is the *sequence*, and knowing what
depends on what.

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
  template you use and whether the sheet generator can help you at all.
- **Categories**, with pairs per club (dual meet) or team counts (standard).
- **Courts** and **match duration**, including any buffer between matches.
- **Days and venues** — one plan per tournament day, if the event runs
  across several.

Export the plan when it's right. For a dual meet, use **Copy plan & open
generator** rather than the file export — it puts the plan on your clipboard
and opens the master workbook's copy dialog in one step.

## 2. Build the scoring workbook

**Dual meet:** use the
[Dual Meet Sheet Generator](dual-meet-sheet-generator.md). Copy the master,
run `SAGE → Generate event tabs`, paste the plan. You get every category tab,
the schedule, and the four readout tabs the live sync publishes from.

Two things to know before you run it:

- **Work in your copy, never the master.** The generator builds several tabs
  in place, so it can only run once per workbook — if a run fails partway, or
  you want to change the plan, you start again from a fresh copy of the
  master. That is deliberate: a half-regenerated workbook mid-event is the
  worst outcome available.
- **Paste the rosters afterwards.** The plan has no player names in it. The
  generator leaves the name-entry columns blank and tells you so when it
  finishes; until they're filled, the workbook is structurally complete but
  has nobody in it.

A generated workbook carries the live-sync script already — it's built into
the master — so it's ready for step 4's sync setup with no code-pasting step
of its own.

**Standard tournament:** there's no generator yet, so the workbook is built by
hand or copied from a previous event. Whichever way, it has to end up with the
exact column names the sync expects — see
[adding a new event](../technical/adding-a-new-event.md) for the list. Getting
one column name wrong is the single most common reason a new event's page
loads but renders empty.

Either way, one workbook per venue per day. An event with three venues across
five days has fifteen of them.

## 3. Build the event site

Copy the matching template — `dual-meet-template/` or
`standard-tournament-template/` — into `events/<event-key>/`, replace the
tokens, and fill in the days, facilities and categories.

The step-by-step is [adding a new event](../technical/adding-a-new-event.md);
the exhaustive version, with the full token table, is `_templates/CLAUDE.md`
in the site repo. You'll also need the event's images ready: a QR code, and
both clubs' logos for a dual meet.

## 4. Register it, and wire up the sync

Two connections, and the event is inert until both exist:

- **Register the event** in `event-data/config/events.json` — its type,
  title, and one entry per day with that day's venues. This takes effect
  within minutes of the commit; nothing needs redeploying.
- **Wire up the sync** in each venue's spreadsheet, once per workbook: reload
  it and run **SAGE → Set up live sync**, entering that workbook's day key
  and facility name. This is what makes typing a score publish it.

The facility names in the config and in each spreadsheet's own settings are
compared exactly and are **case-sensitive** — but setup doesn't leave that to
chance: it confirms the entered day key and facility name against the
registry (and runs a real test sync) before saving, so a mismatch is caught
on the spot rather than discovered as a venue that silently never publishes.

## 5. Rehearse it

Copy the dry-run checklist into the event's folder and work through it, with
the real workbooks and the real site, before the day.

The rehearsal is the same shape as the day itself: enter a few fake scores,
confirm they reach [Control Center](control-center.md) and the public page,
mark a match live and confirm it shows as live, then revert. It is the only
step that exercises the whole chain end to end, and the only place a
case-sensitive facility name or a missing column is cheap to find.

If you print scoresheets, this is also when to generate them — see the
[Scoresheet Generator](scoresheet-generator.md). Once the rehearsal has
proven a sync reaches the public page, the generator can pull matches
straight from that registered event/day/venue instead of downloading a CSV
from the workbook.

## What "ready" looks like

- The plan's finish time is one the venue will accept.
- Every category tab has its rosters pasted in, and no tab shows an error.
- The event page loads, shows the right days and categories, and lists every
  venue.
- A test score typed into each venue's workbook reaches Control Center and
  the public page, and has been reverted.
- Every venue reports a recent successful sync.

At that point the event is ready, and the rest is
[day-of operations](running-an-event-day.md).

## Where dual meets differ

- The scoring workbook can be **generated** from the plan; standard
  tournaments are still hand-built.
- The site uses `dual-meet-template/`, which frames every category, the
  standings, and the Awards tab's "Overall Champion" line as Club A vs Club B.
- The plan needs **pairs per club** and a bracket count per category, and the
  calculator accounts for the cross-club Bronze and Final matches that
  produces.
- You need both clubs' logos among the event images.

---
**Technical:** [adding a new event](../technical/adding-a-new-event.md) ·
[dual meet sheet generator](../technical/dual-meet-sheet-generator.md)
