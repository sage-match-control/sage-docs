# Schedule board

**Used in:** [4. Register and build the site](../usage/register-and-build-the-site.md) ·
[6. Print](../usage/printables.md) · [8. Run the day](../usage/run-the-day.md)

A wall display meant for a screen at the venue, not a phone: courts as columns,
time slots as rows, one card per match, colour-coded by category. Live matches
get a highlighted ring; finished ones dim and show their score. It updates on
its own, with no refreshing.

The board is unlisted. Nothing on [Tournament Hub](tournament-hub.md) links to
it. An operator launches it with **Open schedule** in
[Control Center's Mission Control](control-center.md#mission-control) and hands
the address to whoever runs the venue's screen.

## Reading a card

Each card shows the two sides facing each other, the score in the middle, the
match number, and a small pill for the stage: `RR` for pool play, and a solid
pill for a playoff round (`R16`, `QF`, `SF`, `BRONZE`, `FINAL`). A card is in
one of three states:

- **Upcoming** shows `VS` between the sides.
- **Live** has a green ring and a `COURT n` pill. A match is live when its
  number is in the workbook's `Court Control` tab.
- **Completed** is dimmed and shows its score.

If two matches land in the same court and time slot, or a match has no usable
court, the board never drops one: the time cell shows a `+n` badge, and hovering
it lists the match numbers that did not fit.

## Venue picker and court split

When a day is played at more than one venue, a **Venue** row at the top (**All**
and one button per venue) narrows the board to that venue's matches and courts.
Each venue's screen then shows only its own. With one venue the row is hidden.

A row of court buttons splits the board by court, so a venue with two screens
can show courts 1–5 on one and 6–9 on the other. Court numbers run on across a
day's venues (Main 1–4, Annex 5–9), and a venue's view offers only its own
courts.

Both choices live in the page address (`?venue=` and `?courts=`), so
**bookmark each screen's page once** and it comes back the same after a restart,
with nobody touching the machine.

## Collapsing the header

A chevron at the top collapses the header to the essentials, for a screen that
needs every pixel. It hides the venue and court filters and the **PDF** button
and keeps the colour legend, since a viewer needs that to read the board. The
choice is carried in the address too (`?compact=1`).

## PDF and paper

The **PDF** button lays the board out one sheet per group of courts and opens the
browser's print dialog. **Save as PDF** gives a file; **Print** gives paper for
the desk, the noticeboard and each court. The category colours print. Finished
matches print at full strength rather than dimmed.

## Team events

At a team event each card names both teams above their players and carries the
pair label (`MD`, `WD`, `XD`). A side whose lineup is not in yet reads **Lineup
TBD**. Cards are coloured by bracket, not by category, with every playoff match
in its own colour.

## What it needs

The board shows exactly one day. The event's page is set to that day, and the
colour key for the categories is set by hand from the organizer's own workbook.
Both are part of
[Usage step 4](../usage/register-and-build-the-site.md). If the organizer
recolours the workbook, the board's colours are re-set by hand: nothing detects
the change.

---
**Features:** [Tournament Hub](tournament-hub.md) · [Control Center](control-center.md)
**Technical:** [schedule board](../technical/schedule-board.md)
