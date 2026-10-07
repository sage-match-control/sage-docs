# 6. Print

**Who:** coordinator (developer for the QR panel) · **When:** the last few days; the
scoresheets once the schedule has stopped moving · **You need:** the event synced
([step 5](connect-the-workbooks.md)) and a printer, or a print shop

Four things go on paper: the schedule, the scoresheets, the hub board's QR panel,
and the bracket images.

## The schedule PDF

The event's **schedule page** (`/events/<event-key>/schedule`, or **Open schedule**
in Mission Control) carries a **PDF** button. It lays the board out one sheet per
group of courts and opens the browser's print dialog.

1. Open the schedule page. On a day with several venues, pick the venue first.
2. Click **PDF**.
3. Choose **Save as PDF** for a file, or **Print** for paper for the desk, the
   noticeboard and each court.

Export it once the schedule is packed, numbered and synced, so the PDF, the website
and the workbook all say the same thing. Re-export it if the schedule changes. Keep
a copy in the event's folder in **1. TOURNAMENTS**. See
[Schedule board](../features/schedule-board.md#pdf-and-paper).

## Scoresheets

Print the match slips with the [Scoresheet Generator](../features/scoresheet-generator.md)
**once the schedule is final**. A scoresheet carries its match number, court and
time, so a reshuffle after printing means printing again.

1. In the workbook choose **SAGE → Generate Scoresheets**. It opens the generator
   with this workbook's day and venue already selected. (Or open the generator and
   pick the **event**, **day** and **venue**.) The card shows a match count and how
   long ago the venue last synced.
2. Check the **event name**, which prints on every sheet's header.
3. Pick a **scoresheet type**: **Standard** (with a referee column), **No Referee**
   (self-officiated), **No Referee, Wide** (room for scoring notes) or **Best of 3**.
4. Optionally **add blank scoresheets** for walk-up matches or backups.
5. Click **Generate PDF**, and print.

Do this for every venue and day. If the event isn't registered, or you are printing
before the first sync, upload the matches CSV instead.

## The hub board's QR panel

The venue's **hub board** is a 2 × 3 ft sintra print that tells players the
[Tournament Hub](../features/tournament-hub.md) exists and how to open it. Everything
but the QR panel is the same at every event, so the board is printed once and only the
panel changes.

!!! note "Developer task — needs the site repo (and Chrome) on the developer's computer"
    Render the event's panel with `node _templates/hub-pubmat/render.mjs <event-key>`.
    It reads the panel from the event's own page, so the event page must be built
    first ([step 4](register-and-build-the-site.md)). It writes `qr-panel.pdf` and
    `board.pdf` to `_templates/hub-pubmat/out/<event-key>/`. The mechanics are in
    [Adding a new event](../technical/adding-a-new-event.md).

- **The panel** is an 8 × 8.75 in sticker with the event's name, date and venue, its
  QR code and its short link. Stick it over the last event's panel.
- **The whole board**, with the panel in place, is there too, for when you'd rather
  reprint than re-sticker, or the board is lost.

**Scan the printed panel with a phone and check that it opens this event's page**
before mounting it. If a line on the panel is wrong, ask the developer to fix it on the
event page and make the panel again.

## Bracket images

The images saved in [step 3](draw-and-rosters.md) are what you post and print. Print
each category's image for the noticeboard, and keep the text files.

## Check it worked

- [ ] The schedule PDF matches what the Hub and the workbook show.
- [ ] The scoresheets for every venue and day are printed, with spares.
- [ ] The QR panel, scanned with a phone, opens this event's Hub.
- [ ] Bracket images are printed for the noticeboard.

## If it doesn't

- **The scoresheet card says the day hasn't synced.** Run
  [step 5](connect-the-workbooks.md)'s first sync, or upload the CSV.
- **The PDF shows the wrong venue or courts.** Check the venue picker and the court
  buttons on the page you printed from.
- **The QR panel refuses.** The event page still has `{{TOKENS}}` in it. Fix the page
  and render again.

## Where the formats differ

- **Team:** scoresheets carry the matchup's four matches. Print once lineups are
  known, or print blanks.
- **Dual meet:** one workbook, so one set of scoresheets.
- **Standard:** one set per venue per day.
- **Team:** one workbook, so one set.

**Next:** [7. Rehearse](rehearse.md).
