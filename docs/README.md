# S.A.G.E. Documentation

S.A.G.E. Match Control Experts is a team that runs pickleball tournaments.
This repo documents the tooling they built to do it: live match/standings
sync from a Google Sheet to a public event site, an operator console for
running the day of, and five generator tools (scoresheets, bracket draws,
tournament timing, and the dual-meet and standard-tournament sheet generators).
It's split into four sections for different readers.

## [Usage](usage/README.md)

**A training guide.** For someone who has never run a S.A.G.E. event. Read front
to back, it takes you from planning the event, through the day of the tournament,
to what you do after it. It says who does each step, what it needs, and how to tell
it worked. Start here if you want to *run* a tournament with S.A.G.E.

## [Features](features/README.md)

**A reference.** What each tool is, what it shows, and every option it has, one page
per tool. Start here if you're looking up a tool, or you're a player or spectator
trying to understand what a page is showing you. The Usage guide links here for the
detail.

## [Technical](technical/README.md)

**Architecture, code, deployment.** How the pieces fit together, what each
repo owns, the data flow between them, and the reasoning behind non-obvious
decisions. Start here if you're maintaining or extending S.A.G.E. itself.

## [Specs](specs/README.md)

**Design specs.** The reasoning behind non-obvious decisions, written before or
while a feature was built, by build status. Come here for the *why* behind a specific
piece.

## [Changelog](changelog.md)

**What changed in these docs**, release by release, and how they are versioned. The
version is in the footer of every page.

---

## The three repos

S.A.G.E. is three repos, not one. This doc repo is a fourth, alongside them:

| Repo | What it is |
| --- | --- |
| [`sage-tools-api`](https://github.com/sage-match-control/sage-tools-api) | Node/Express backend on Google Cloud Run. Four features: scoresheet PDF generation, the Google Sheets → GitHub live-data sync, event attendance, and score entry. Its `live-worker/` folder holds the Cloudflare Worker that pushes snapshots to open pages. |
| [`sage-match-control.github.io`](https://github.com/sage-match-control/sage-match-control.github.io) | GitHub Pages static site. Tournament Hub (the public event pages), Control Center (the operator console), and the standalone tools. |
| [`event-data`](https://github.com/sage-match-control/event-data) | Shared data store. Every event's live snapshots and the registry (`config/events.json`) that says which events/days/facilities exist. |

Full picture: [technical/architecture.md](technical/architecture.md).
