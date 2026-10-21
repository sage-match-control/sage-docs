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
| [Millennium Invitational](millennium-invitational-spec.md) | The 24 October 2026 team event's site, pushed to the site's `main`: an opt-in `teamScoring: "games"` in the engine (a matchup won on games won, a bracket ranked by games won, total points, then head-to-head wins and points), the team template re-skinned from the pubmat and logo, and its `events.json` entry in `event-data` | The short link and QR image, the workbook fixes still open, and the dry run |
| [Angeles City Employees Sportsfest](acso-employees-sportsfest-spec.md) | The 13 November 2026 standard event's site, pushed to the site's `main`: the standard template in the house theme with the ACSO logo, the organiser's category colours on the schedule board, team codes without a division (`"divisions": { "": "" }`), and its `events.json` entry. The workbook syncs all 96 matches with the players' names | The QR image for `tinyurl.com/SAGExACSO2026` and the QR panel, the rest of the workbook checks (`Court Control`, the start time, players in two categories) and the dry run |
| [Code and access protection](code-and-access-protection-spec.md) | Built 2026-10-12, and deployed (all of it is in `sage-tools-api` 3.4.0 and the site's `main`): the played/series fix (`sage-tools-api` 3.1.1), the server reading workbooks as its service account (3.2.0), the attendance list through the API (3.3.0), the site's attendance view with its harness, the runbook and docs for Restricted facility workbooks, and `LICENSE` files | The owner's tasks (spike S1, quota checks O1–O2, then O3), and the Google, GitHub and Cloud settings of §2, §5, §6 and §7 |
| [Playoff bracket chart](standings-bracket-chart-spec.md) | The List / Bracket switch and bracket chart on the Hub and in Control Center, with the round strip and Full tree on phones; the Standard Tournament Generator 1.3.0 (the `STANDINGSCSV` `match` and `from` columns, **SAGE → Add bracket links**); `display.playoffSeats`; the unit tests, a fixture event and the browser checks; the docs. All on a branch of each repo | Merging the site to `main` (not before 25 October, by 9 November for the Sportsfest), pasting generator 1.3.0 into the masters and the Sportsfest workbook and running **Add bracket links**, and the push of each branch |
