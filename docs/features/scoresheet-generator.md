# Scoresheet Generator

Turns a matches CSV into a ready-to-print PDF of paper scoresheets — one per
match. A standalone tool, no sign-in, nothing saved: your CSV goes straight
to the generator and nothing is stored afterward.

## How it works

Four steps on one page:

1. **Choose your matches** — pick an event or upload a CSV:
   - **Registered event** — pick an **event**, **day**, and **venue**
     registered in `event-data/config/events.json`, and the matches come
     straight from that day's last published sync. The card shows a match
     count and how long ago that venue last synced, so you can confirm a
     roster paste actually landed before printing. A venue with no matches
     yet is listed but disabled. If the day hasn't synced at all, the card
     says so and points you at uploading a CSV instead.
   - **Upload a CSV** — drag-and-drop or browse. This is the same kind of
     matches file the rest of S.A.G.E. reads, listing every matchup for the
     event. Always available, and the only option for an event that isn't
     registered, a hand-edited CSV, or printing before the day's first sync.
2. **Name the event** — prints on every scoresheet's header. Auto-filled from
   the registered event's title when you pick one; still editable, and a name
   you type yourself is never overwritten by changing day or venue.
3. **Pick a scoresheet type** — four layouts:
   - **Standard** — includes a referee column
   - **No Referee** — for self-officiated matches
   - **No Referee, Wide** — extra room for scoring notes
   - **Best of 3** — tracks three games per match
4. **Add blank scoresheets** *(optional)* — extra unfilled sheets, shuffled
   in with the rest, handy for walk-up matches or backups.

Click **Generate PDF** and a progress bar tracks the render as it happens —
useful for a large event where generating hundreds of sheets takes a moment.
The finished PDF downloads when it's done.

Every registered facility spreadsheet's **SAGE** menu also has a
**Generate Scoresheets** item that opens this page with that workbook's day
and venue already selected.

---
**Technical:** [scoresheet pipeline](../technical/scoresheet-pipeline.md)
