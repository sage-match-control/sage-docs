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
| [Double round robin, best-of-3 round robin and one-bracket finals](double-round-robin-and-finals-spec.md) | Built and live: the calculator's round-robin choice (single, double, best-of-3 meetings) and four one-bracket playoffs, Standard Tournament Generator 1.5.0 (`rr_legs`, `rr_best_of`, `final`, `final_bronze`, `CSV` scores by match number, a `bestOf` column, series W and L), `sage-tools-api` 3.7.0, the site engine's best-of-3 rule in two pushes, their tests and the docs | Pasting the generator into the Standard Tournament Master (it still runs 1.4.1), the owner's real-workbook checks (§9 step 2) and dry run (§9 step 5). The first event to use them is the UPAP 3rd Annual Tournament, 22 November 2026 |
| [Angeles City Employees Sportsfest](acso-employees-sportsfest-spec.md) | The 13 November 2026 standard event's site, pushed to the site's `main`: the standard template in the house theme with the ACSO logo, the organiser's category colours on the schedule board, team codes without a division (`"divisions": { "": "" }`), and its `events.json` entry. The workbook syncs all 96 matches with the players' names | The QR image for `tinyurl.com/SAGExACSO2026` and the QR panel, the rest of the workbook checks (`Court Control`, the start time, players in two categories) and the dry run |
| [Code and access protection](code-and-access-protection-spec.md) | Built 2026-10-12, and deployed (all of it is in `sage-tools-api` 3.4.0 and the site's `main`): the played/series fix (`sage-tools-api` 3.1.1), the server reading workbooks as its service account (3.2.0), the attendance list through the API (3.3.0), the site's attendance view with its harness, the runbook and docs for Restricted facility workbooks, and `LICENSE` files | The owner's tasks (spike S1, quota checks O1–O2, then O3), and the Google, GitHub and Cloud settings of §2, §5, §6 and §7 |
| [UPAP 3rd Annual Tournament](upap-3rd-annual-tournament-spec.md) | The 22 November 2026 standard event's site, in three phases, each pushed to the site's `main`: the schedule board with the `events.json` entry (`b5ce561`, `event-data` `c1a02d4e`), the Tournament Hub (`03bb1e3`) and the desk page with the dry-run checklist (`c32ace0`). The standard template re-skinned from the pubmat, with the board's category colours chosen for the event. The workbook syncs `upap-day1` | The QR image for `tinyurl.com/SAGExUPAP2026` and the QR panel, the rest of the workbook checks, the dry run and the event. A fix for the Hub's QR panel covering the right end of the title between 900 and about 1120px wide, waiting on the owner |
| [Team names on scoresheets](scoresheet-team-names-spec.md) | A team event's sheets print each team's name beside its pair code, on all four layouts: `sage-tools-api` 3.6.0 and 3.6.1 and the site's `94fa9c7`, live | §7's live-page checks C1–C5 (C5 is the owner's call) and §8's docs |
