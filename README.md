# S.A.G.E. Documentation

S.A.G.E. (Match Control Experts) is pickleball tournament tooling: live
match/standings sync from a Google Sheet to a public event site, an operator
console for running the day of, and a couple of standalone generator tools
(scoresheets, bracket graphics, tournament timing). This repo is the
documentation for all of it, split into two sections for two different
readers.

## [Features & Usage](features/README.md)

**No code, no deployment steps.** What each tool does, who it's for, and how
to use it — written for tournament organizers, day-of operators, and anyone
just trying to find their match. Start here if you want to *run* a tournament
with S.A.G.E., or you're a player/spectator trying to understand what a page
is showing you.

## [Technical](technical/README.md)

**Architecture, code, deployment.** How the pieces fit together, what each
repo owns, the data flow between them, and the reasoning behind non-obvious
decisions. Start here if you're maintaining or extending S.A.G.E. itself.

---

## The three repos

S.A.G.E. is three repos, not one. This doc repo is a fourth, alongside them:

| Repo | What it is |
| --- | --- |
| [`sage-tools-api`](https://github.com/sage-match-control/sage-tools-api) | Node/Express backend on Google Cloud Run. Scoresheet PDF generation + the Google Sheets → GitHub live-data sync. |
| [`sage-match-control.github.io`](https://github.com/sage-match-control/sage-match-control.github.io) | GitHub Pages static site. The public event pages, the operator console, and the standalone tools. |
| [`event-data`](https://github.com/sage-match-control/event-data) | Shared data store. Every event's live snapshots and the registry (`config/events.json`) that says which events/days/facilities exist. |

Full picture: [technical/architecture.md](technical/architecture.md).
