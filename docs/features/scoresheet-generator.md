# Scoresheet Generator

Turns a matches CSV into a ready-to-print PDF of paper scoresheets — one per
match. A standalone tool, no sign-in, nothing saved: your CSV goes straight
to the generator and nothing is stored afterward.

## How it works

Four steps on one page:

1. **Name the event** — prints on every scoresheet's header.
2. **Pick a scoresheet type** — four layouts:
   - **Standard** — includes a referee column
   - **No Referee** — for self-officiated matches
   - **No Referee, Wide** — extra room for scoring notes
   - **Best of 3** — tracks three games per match
3. **Upload your matches CSV** — drag-and-drop or browse. This is the same
   kind of matches file the rest of S.A.G.E. reads, listing every matchup for
   the event.
4. **Add blank scoresheets** *(optional)* — extra unfilled sheets, shuffled
   in with the rest, handy for walk-up matches or backups.

Click **Generate PDF** and a progress bar tracks the render as it happens —
useful for a large event where generating hundreds of sheets takes a moment.
The finished PDF downloads when it's done.

---
**Technical:** [scoresheet pipeline](../technical/scoresheet-pipeline.md)
