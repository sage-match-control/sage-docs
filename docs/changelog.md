# Changelog

What changed in these docs, release by release. The other pages describe the
system as it is; the history of how the docs got there lives here.

## How the docs are versioned

The docs carry a version, `MAJOR.MINOR.PATCH`, shown in the footer of every page.
What a reader would notice decides which number goes up:

| Bump | When | Examples |
| --- | --- | --- |
| **MAJOR** | A page or nav section is removed, renamed or moved, so a bookmark or a link from another repo stops working. | Splitting a tab in two; deleting a page and sharing its content out. |
| **MINOR** | Something new to read: a new page or section, a page updated for a change in the product, or a new spec or a spec moving status folder. | A features page for a new tool; a feature's new behaviour; a spec moving to `implemented/`. |
| **PATCH** | A correction that changes nothing in the product: a wrong fact, a broken link, a typo, clearer wording. | Fixing a menu name; repointing a link. |

A spec's URL includes its status folder, so a status move is MINOR, not MAJOR.
Cite specs by filename, as the [Specs index](specs/README.md#citing-a-spec)
describes, and a status move breaks nothing.

### Releasing a version

Every change that reaches `main` carries its version, in the same commit (or the
last commit of its pull request):

1. Add an entry at the top of this page: `## [x.y.z] — YYYY-MM-DD`, dated in Manila
   time. A pull request with several changes gets one entry, with the largest bump
   among them.
2. Set the same version in `mkdocs.yml`'s `copyright` line.

The deploy workflow refuses to publish when the two disagree.

An entry groups its lines under **Added**, **Changed**, **Removed** and **Fixed**.
**History** records facts taken out of the pages because they describe the past
rather than the system: renames, earlier versions, measurements of a design that
has been replaced.

---

## [2.4.0] — 2026-10-09

Desk and scorer links name who they were issued to, and the workbook records it.

### Added

- [Who a link was issued to](features/control-center.md#who-a-link-was-issued-to):
  the optional **Issued to** and **Note** fields when issuing a desk or scorer link.
- [Who marked someone in](features/event-attendance.md#who-marked-someone-in): the
  `markedBy` column of the `ATTENDANCE` tab.
- [Who entered a score](features/score-entry.md#who-entered-a-score): the note on both
  score cells of a match.
- [Technical](technical/auth.md#who-a-scoped-token-was-issued-to): the `to` and `note` keys
  of a scoped token, the optional body of the two `/v3` issue routes, how `markedByColumn`
  picks a column, and the one-cell widening of the attendance write allowlist.

### Changed

- The scorer and desk pages show the label after the day.
- [Run the day](usage/run-the-day.md): issue one link per person, with their name and gate or
  courts. The [scorer](usage/scorer-handout.md) and [desk](usage/desk-handout.md) handouts say
  the page shows who the link was issued to, and not to pass it on.
- Spec: [Link attribution](specs/implemented/link-attribution-spec.md) moves from Not started
  to **Implemented**.

## [2.3.0] — 2026-10-08

### Added

- Spec: [Link attribution](specs/implemented/link-attribution-spec.md). Desk and scorer
  links name who they were issued to, and the workbook records it on every mark and
  score they make. Not started.

## [2.2.1] — 2026-10-08

### Fixed

- The [scorer handout](usage/scorer-handout.md) and [desk handout](usage/desk-handout.md)
  stand alone, with no way to reach another page. They no longer end with links to the
  Features pages, and a page template, `overrides/handout.html`, drops the menu, tabs,
  search, repository and edit links, and the footer. A scorer or desk volunteer sees
  only the one page.

## [2.2.0] — 2026-10-08

The roles match how SAGE works: the organizer hires SAGE and decides, and SAGE members
do the work.

### Changed

- [Roles, access and kit](usage/before-you-start.md) defines four roles. The
  **organizer** hires SAGE and decides; they do nothing in the workbooks or the
  system. The **coordinator**, a SAGE member, prepares the event: plan, workbooks,
  draw, rosters, printing. The **developer**, a SAGE member, does everything
  technical. The **operators** are the SAGE members on site on the day, and they set
  up and run the venue screens. Scorers and desk staff can be SAGE members or the
  organizer's volunteers.
- The operators' three jobs on the day: **calling matches** (Court Control, calling
  matches to courts, the qualifier draws), **scores and playoff names**, and **players
  and the organizer**. Jobs can overlap, but the operator calling matches never also
  coordinates with the organizer. [Run the day](usage/run-the-day.md) says which job
  does each part of play.
- New section: [What SAGE needs from the organizer](usage/before-you-start.md#what-sage-needs-from-the-organizer).
- Every Usage step names the coordinator, developer or operators as its owner. *Admin
  task* boxes are **Developer task** boxes.
- On the day, an operator types the qualifier draw, team lineups, playoff seeds and
  late changes into the workbook; the organizer decides them. Features and Technical
  say the same.
- Glossary: Coordinator, Developer, Operator and Organizer.

## [2.1.1] — 2026-10-08

### Fixed

- Stray headings in the middle of paragraphs. A wrapped line starting with `#1` or
  `#2` rendered as a page title (Run the day, Glossary, Tournament Hub, Tournament
  Calculator, the Standard Tournament Master spec), and a sentence directly above a
  `---` rule rendered as a section heading (the Sync script configuration spec).
- Technical → Event attendance: *Pickle for Sight's attendance* is a section, not a
  second page title.

## [2.1.0] — 2026-10-08

Spec statuses brought in line with what is built.

### Changed

- [Usage guide and docs restructure](specs/in-progress/docs-usage-guide-spec.md)
  moves from Not started to **In progress**: merged as 2.0.0, with the newcomer
  read-through and the site repo's checklist fixes left.
- [Site test suite](specs/not-started/site-test-suite-spec.md) stays Not started,
  with its status revised: the site engine built the `_tests/` runner, fixtures and
  unit tests it shares, and its page tests are not built.
- Nine implemented specs that had no status line carry one: Bracket Generator,
  Calculator dual-meet fixes, Calculator PWA, Dual Meet Schedule Generator, Event
  site templates, PNF × BUP dual meet, Scoresheet event picker, Runtime-fetched sync
  config and Sync script configuration.

## [2.0.0] — 2026-10-08

The Features & Usage tab splits in two: a **Usage** guide that trains a new
operator, and **Features** as a per-tool reference. Built from the
[usage guide spec](specs/in-progress/docs-usage-guide-spec.md).

### Added

- **Usage** tab, first after Home: Start here, Roles, access and kit, Glossary,
  steps 1–9 (Plan it, Build the workbooks, Draw and fill rosters, Register and build
  the site, Connect the workbooks, Print, Rehearse, Run the day, When something's
  wrong), the scorer and desk handouts, and After the event.
- Features pages: [Schedule board](features/schedule-board.md),
  [Score entry](features/score-entry.md),
  [The scoring workbook](features/scoring-workbook.md) (every tab and the whole SAGE
  menu) and [Team workbook](features/team-workbook.md).
- A **Used in:** line at the top of every features page, linking the Usage steps
  where the tool comes in.
- This changelog, and the docs version in the footer.
- Checklists render as task lists.

### Changed

- The **Features & Usage** tab is now **Features**, a reference grouped as Public
  pages, Running the day, Scoring workbooks, and Planning and print tools.
- Control Center's sections follow the order of its tabs. *Entering a score* is on
  its own page, Score entry.
- The Dual Meet and Standard Tournament generator pages keep what each generator
  builds, its rules and its refusals. The SAGE menu moves to The scoring workbook,
  and the step-by-step to Usage step 2.
- Tournament Hub's schedule board section moves to its own page.
- The scorer page and event attendance pages are reference only. Their step-by-step
  is the scorer and desk handouts.
- [Adding a new event](technical/adding-a-new-event.md) is the admin's mechanics page
  for Usage steps 4–6, in the order register → site → workbooks, with the scorer
  page and the service-account share as steps of their own.
- Technical overview: Event attendance is listed under Backend.
- Home: four sections, Usage first.
- Usage, Features and Technical pages are written in the present tense. Their change
  history is recorded below.

### Removed

- `features/preparing-an-event.md`. Its content is in Usage steps 1–7.
- `features/running-an-event-day.md`. Its content is in Usage step 8 and After the
  event.
- The sync pipeline's *Baseline: the time-based trigger* section. Recorded below.

### Fixed

- Features overview: three kinds of event (standard, dual meet, team), not two.
- Adding a new event: both masters carry the sync script; only a hand-made workbook
  needs it pasted in.
- Technical overview: three Apps Script files live in `sage-tools-api/apps-script/`;
  event attendance is `src/attendance/`, part of the API.
- Home: `sage-tools-api` also runs attendance and score entry, and holds the live
  push Worker (on Cloudflare); there are five generator tools.

### History

- **Control Center** was first called **Match Control**. The file, the URL and every
  reference moved together; `tools/match-control.html` stays as a redirect stub. The
  [console](specs/implemented/match-control-console-spec.md) and
  [Awards tab](specs/implemented/awards-podium-tab-spec.md) specs predate the rename.
- **Pickle for Sight's attendance** is the earlier, per-workbook version: an
  `attendance.gs` web app in each workbook, one row per pair slot, no desk links and no
  Attendance tab. It was removed from `sage-tools-api`; its last version is
  `apps-script/attendance.gs` at commit `198f02c`. Every later event uses the API's
  `ATTENDANCE` tab, which the API creates in a workbook made before the generators
  added it.
- **The Bracket Generator** replaced four near-identical copies: two event-site
  templates and one live event's copy, which had drifted to a pre-S.A.G.E. palette.
  That event's copy became a redirect stub; archived events' copies are frozen. Adding
  keep-apart groups left a draw without groups byte-for-byte the same.
- **The generators** replaced duplicating the last event's workbook and
  find-replacing every category key, court number and team code by hand. The dual-meet
  generator's spec was written before the workbook was read closely, hence its §13;
  every bug found while building the generator was a block one row off, which is why it
  logs its layout.
- **The time-based sync trigger.** Before the lock-based sync, `sheets-sync.gs`
  scheduled a one-shot `runIfSettled` trigger from every edit. Measured on the
  Piggleball workbook on 1 October 2026, the delay from the last edit to the sync
  starting was 119, 70, 74, 22 and 108 s across five bursts: a median of about 74 s
  against the 3 s intended. Each `onEditInstallable` run took 1.3–3.3 s, mostly
  deleting and recreating triggers, and parallel edits raced on that and left stray
  triggers, so a workbook could sync twice. The lock-based sync replaced it; a
  non-holder run takes 0.6–2.2 s.
- **Live push** replaced polling GitHub Pages as the main delivery path. With polling
  alone, a published score took about 35–40 s to reach a viewer.
- **The PickleDrive workbook** was trimmed on 5 October 2026: the `StackCache` tab, a
  rewritten matchup family and 15 named functions removed, with every published value
  unchanged.
- **`completedAt`** is newer than the first published snapshots. A snapshot from
  before it uses the per-browser `syncedAt` fallback.
- **The team template.** PickleDrive Club One Year Celebration, the first team event,
  was hand-built as `events/pickledrive-anniversary-2026/` before the team template
  existed.
- **`sage-tools-api` 3.0.0** restructured the service. Before it, the service was a
  feature-sliced, layered monolith with hand-wired constructor injection: each service
  was coded against the concrete object it received, routes did their own auth and
  error mapping, and `Server` imported every feature. `/v1` was the partial REST
  surface of the 2.x releases.
- **Control Center's Match Finder refresh** used to re-run the search box's text, so
  pausing mid-word swapped the list for "Several pairs match" and the pinned search box
  lost its scroll. It redraws the search that was run.
- **The Tournament Calculator's legend** used to infer extra rounds from `plan.byes`,
  and said "no preliminary rounds needed" above a chart showing two.
- **The service-account share** inherited from `1. TOURNAMENTS` was confirmed on
  2026-10-08: Piggleball's workbook there lists the account as Editor.

## [1.0.0] — 2026-10-08

The docs as they stood when versioning began, at commit `cf8a4ae`: Home and the
Features & Usage, Technical and Specs tabs. Earlier changes are in the repo's git
history.
