# In-progress specs

Specs where **part of the work is live and part is not**. The folder is a
filing decision; each spec's own status line is where you find out which half
you are reading.

A spec belongs here when a reader could otherwise be misled — a phased build
partway through, a feature shipped on one surface but not the other, or
something written and working locally but not committed or deployed. It does
not stay here: when the rest lands it moves to
[`../implemented/`](../implemented/README.md); if the work is abandoned, the
built parts get their own spec and the rest goes to
[`../archived/`](../archived/README.md).

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Built | Not yet |
| --- | --- | --- |
| [Double round robin, best-of-3 round robin and one-bracket finals](double-round-robin-and-finals-spec.md) | Built and live: the calculator's round-robin choice (single, double, best-of-3 meetings) and four one-bracket playoffs, Standard Tournament Generator 1.5.0 (`rr_legs`, `rr_best_of`, `final`, `final_bronze`, `CSV` scores by match number, a `bestOf` column, series W and L), `sage-tools-api` 3.7.0, the site engine's best-of-3 rule in two pushes, their tests and the docs. The Standard Tournament Master runs 1.5.0, and the UPAP workbook was generated from it | The owner's real-workbook checks (§9 step 2) and dry run (§9 step 5). The first event to use them is the UPAP 3rd Annual Tournament, 22 November 2026 |
| [UPAP 3rd Annual Tournament](upap-3rd-annual-tournament-spec.md) | The 22 November 2026 standard event's site, in three phases, each pushed to the site's `main`: the schedule board with the `events.json` entry (`b5ce561`, `event-data` `c1a02d4e`), the Tournament Hub (`03bb1e3`) and the desk page with the dry-run checklist (`c32ace0`). The standard template re-skinned from the pubmat, with the board's category colours chosen for the event. The workbook syncs `upap-day1` | The QR image for `tinyurl.com/SAGExUPAP2026` and the QR panel, the rest of the workbook checks, the dry run and the event. A fix for the Hub's QR panel covering the right end of the title between 900 and about 1120px wide, waiting on the owner |
