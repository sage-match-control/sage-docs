# Spec — Usage guide and docs restructure

> **Status: in progress.** Built and merged to `sage-docs` `main` on 2026-10-08 as
> docs 2.0.0 (see the [changelog](../../changelog.md)): the Usage tab (§4), the
> Features changes (§5), the corrections in §2.2 inside `sage-docs`, and §6. Not
> yet: the read-through by someone who has never run an event (§7 step 5), and
> D5 with the two site-repo fixes in §2.2 (the dry-run checklist template's
> labels and scorer links, and `_templates/CLAUDE.md` step 13), which need a push
> to `sage-match-control.github.io`. When both land, this moves to
> `implemented/`.
>
> Written 2026-10-08 from an audit of
> every page in `features/`, `technical/README.md`, `technical/adding-a-new-event.md`,
> the root `README.md`, and the site repo's `_templates/CLAUDE.md` and
> `_templates/dry-run-checklist-template.md` as of that date. The owner's
> decisions are in §8.

## 1. Goal

Split today's **Features & Usage** tab into two:

- **Usage** — a training guide for someone who has never run a S.A.G.E. event.
  Read front to back, it takes them from planning the event to the day of the
  tournament and through to after it. It covers every step in order, says who
  does each step and what the step needs, and shows how to tell the step worked.
  It links to Features for detail and does not repeat it.
- **Features** — a reference. Each page covers one tool: what it is, what it
  shows, and every option it has. These pages carry no event-setup sequence.
  Each one links to the Usage step where the tool comes in.

Technical and Specs stay as they are, apart from the corrections in §6.

## 2. What the audit found

### 2.1 Procedure is spread over five places, in three different orders

The steps for getting an event ready are written up in:

| Where | Order it uses |
| --- | --- |
| `features/preparing-an-event.md` | plan → workbook → draw → **site** → **register** → sync → PDF → rehearse → scoresheets → QR panel |
| `technical/adding-a-new-event.md` | site → images → tokens → board → **register** → sync script → data folder → checklist → attendance → QR |
| `_templates/CLAUDE.md` §2 (site repo) | the same as above, plus scorer page (12) and the service-account share (13) |
| `_templates/dry-run-checklist-template.md` (site repo) | rehearsal, then day of |
| Each generator's features page | its own how-to, including the SAGE menu |

**The ordering problem.** Every one of these builds the site *before*
registering the event. But the Hub reads its days, venues and labels from
`events.json`, and **Set up live sync** checks the day key against it. In
practice, then, registering comes first. Usage should register first, then build
the site, then connect the workbooks.

### 2.2 Stale or wrong statements

| Page | Says | Actually |
| --- | --- | --- |
| `features/README.md` § "Two kinds of event" | two event types | three: standard, dual meet, **team** |
| `features/preparing-an-event.md` | covers dual meet and standard only | team events (copying the PickleDrive workbook, entering `Teams`, `MatchUps` seeds) are missing entirely |
| `features/preparing-an-event.md` | — | **score entry** setup is missing: `scoreEntry` in `events.json`, the scorer page, sharing with the service account for score entry |
| `technical/adding-a-new-event.md` step 8 | "a dual meet's workbook carries [the sync script] already" | both masters carry it; only a hand-made workbook needs it pasted |
| `technical/adding-a-new-event.md` | — | no scorer-page step and no service-account share for `scoreEntry` (the runbook's steps 12–13) |
| `technical/README.md` § "Bound Apps Script" | "All four live in `sage-tools-api/apps-script/`" and lists Event attendance there | attendance is `src/attendance/`, part of the API; there are three Apps Script files |
| root `README.md` | `sage-tools-api` = "Scoresheet PDF generation + the Google Sheets → GitHub live-data sync"; "a couple of standalone generator tools" | it also runs attendance and score entry, and the live Worker is on Cloudflare; there are five tools |
| `specs/README.md` | "start with Features & Usage" | needs to point at the new tabs |

**Outside `sage-docs`** (out of scope here, but should be fixed in the site
repo at the same time): the dry-run checklist template still says
**Live/Hide** set to `false`/`auto`, while the console now shows **Auto /
Force live / Force hidden**. It also mentions an **Open match finder** button,
which is now **Open Tournament Hub**, and it never mentions scorer links.
Also, `_templates/CLAUDE.md` step 13 says the service-account share is
inherited from the Drive folder but "not confirmed yet". It is confirmed: on
2026-10-08, Piggleball's workbook in `1. TOURNAMENTS` lists the account as
Editor. Step 13 should now say to move the workbook into the event's folder.

### 2.3 Pages missing or misplaced in Features

- **The schedule board** has a technical page but no features page. Its
  description is buried in `tournament-hub.md`, which breaks the rule that
  every page has a counterpart in the other section.
- **The scoring workbook itself has no page.** The full SAGE menu is described
  only on the *dual-meet* generator page, so a standard or team operator never
  finds it there. Neither are the tabs every workbook shares (`SCHEDULE`,
  `Court Control`, `CSV`, `STANDINGSCSV`, `Timeline`, `ATTENDANCE`).
- **The team workbook** (`Teams`, `MatchUps`, lineups and seed cells) is
  explained only in `_templates/CLAUDE.md` and the dry-run checklist, which are
  both repo files.
- **Score entry** (the dialog) lives inside `control-center.md` and is linked
  from the scorer page. It is shared, so it should be its own page.
- **`control-center.md` sections are in a different order from the tabs.** The
  tabs run Mission Control, Awards, Attendance, Match Finder, Live Matches,
  Standings, Teams. The page runs Mission Control, Attendance, Teams, Awards,
  Live Matches, Match Finder, Standings.

### 2.4 What a newcomer needs that no page has

- **Who does what.** Roles and what each needs: the organizer or admin (GitHub
  access to the site and `event-data`, the masters), the operator (Control
  Center login), scorers, desk staff.
- **Access checklist.** Google access to the masters and the event Drive
  folder, the operator login, GitHub access, and the shared secret (only for
  a workbook made by hand).
- **A glossary.** Event key, day key, facility/venue, division/event/category
  codes, team code, match number, bracket, round robin, playoff, BYE, walkover,
  twice-to-beat, Court Control, snapshot, sync, go-live, live push.
- **A timeline.** How far ahead each step happens, and what can't start until
  something else is done.
- **Equipment.** Laptop or tablet for the operator, venue screens and their
  cables or casting, wifi, the printer, the hub board.
- **The playoff hand-offs on the day.** A standard tournament's qualifier draw
  ("you still fill it in on the day", with no instructions anywhere) and a team
  event's seed cells (only in the dry-run checklist).
- **Late changes.** A withdrawal or substitution, a schedule change after the
  scoresheets are printed, and when to use **Pause live sync**.
- **Handouts.** A one-page brief for scorers and one for desk staff, to send
  with their links.

## 3. New navigation

```
Home
Usage                         <- new, first after Home
  Start here                  usage/README.md
  Roles, access and kit       usage/before-you-start.md
  Glossary                    usage/glossary.md
  Preparing the event
    1. Plan it                usage/plan-the-event.md
    2. Build the workbooks    usage/build-the-workbooks.md
    3. Draw and fill rosters  usage/draw-and-rosters.md
    4. Register and build site usage/register-and-build-the-site.md
    5. Connect the workbooks  usage/connect-the-workbooks.md
    6. Print                  usage/printables.md
    7. Rehearse               usage/rehearse.md
  On the day
    8. Run the day            usage/run-the-day.md
    9. When something's wrong usage/troubleshooting.md
  Handouts
    Scorers                   usage/scorer-handout.md
    Attendance desks          usage/desk-handout.md
  After the event             usage/after-the-event.md
Features                      <- reference
  Overview                    features/README.md
  Public pages
    Tournament Hub            features/tournament-hub.md
    Schedule board            features/schedule-board.md        (new, split out)
  Running the day
    Control Center            features/control-center.md
    Score entry               features/score-entry.md           (new, split out)
    Scorer page               features/scorer-page.md
    Event attendance          features/event-attendance.md
  Scoring workbooks
    The scoring workbook      features/scoring-workbook.md      (new)
    Dual Meet Sheet Generator features/dual-meet-sheet-generator.md
    Standard Tournament Gen.  features/standard-tournament-generator.md
    Team workbook             features/team-workbook.md         (new)
  Planning and print tools
    Tournament Calculator     features/tournament-calculator.md
    Bracket Generator         features/bracket-generator.md
    Scoresheet Generator      features/scoresheet-generator.md
Technical                     (unchanged nav; §6 fixes)
Specs                         (+ this spec under Not started)
```

Filenames carry no step numbers, so a link survives if the steps get reordered;
the order comes from the nav alone. The headings use numbers ("1. Plan it") so
readers can say "step 4".

`features/preparing-an-event.md` and `features/running-an-event-day.md` are
**deleted**, and their content is shared out among the Usage pages. Nothing outside
`sage-docs` links to either (checked: no hits in the other three repos).

## 4. The Usage pages

Every step page has the same frame, so a trainee always knows where to look:

> **Who:** organizer / admin / operator · **When:** e.g. "2–3 weeks before" ·
> **You need:** what has to exist first
>
> *Steps* (numbered, one action each, the exact menu or button name in bold)
>
> **Check it worked:** what you should see
>
> **If it doesn't:** the common failures, with links to Troubleshooting
>
> **Where the formats differ:** standard / dual meet / team (only when they do)
>
> **Next:** the next step

Steps that need repo access are set off in an admonition: **Admin task —
needs GitHub access to `event-data` / the site repo**. Each one names the
change in one line and links to `technical/adding-a-new-event.md` for how to
make it (see decision D1).

| Page | Covers | Built from |
| --- | --- | --- |
| **Start here** | What S.A.G.E. is in a few paragraphs and one diagram (workbook → sync → Hub, Control Center, board). The three event types and how to tell which yours is. The whole journey on one screen: the step list with links, as `preparing-an-event`'s Quickstart is now. How to read the guide. | root README, `preparing-an-event` intro and Quickstart, `features/README` |
| **Roles, access and kit** | The roles (organizer/admin, operator, scorer, desk staff, screen runner) and who can double up. An access checklist for each role. Equipment for the day. | new (§2.4), with the staffing and equipment in §8 D4 |
| **Glossary** | The terms in §2.4, each a sentence or two, linking to where it matters. | new; terms drawn from every features page |
| **1. Plan it** | Choose the format, then pick the **event key** and the **day keys** (they must be unique across all events). The calculator: categories, courts, match length, single-bracket options, one plan per day. Check the finish time. **Copy plan & open generator**. Team events: the calculator has no team format yet, so planning is done by hand to PickleDrive's shape. | `preparing` §1, calculator page, `_templates/CLAUDE.md` step 7 (day-key uniqueness) |
| **2. Build the workbooks** | Copy the master and generate, separately for each format. **Dual meet**: one workbook. **Standard**: one per venue per day, with categories, court labels and waves, then **packing `SCHEDULE`** from `MATCHES` (the rules, Paste special). **Team**: copy the PickleDrive workbook and clear its inputs, PickleDrive's shape only. Then **file it**: move each workbook into the shared drive folder **SAGE → 1. TOURNAMENTS →** the event's own folder, named `<YYYY-MM-DD> <event name>` (e.g. `2026-10-03 PiggleBall Tournament`). Create that folder if it doesn't exist; it also holds the event's `BRACKETS` folder and schedule PDF. `1. TOURNAMENTS` is shared with the API's service account as Editor, and a workbook inside it inherits that share. Checked on Piggleball's workbook on 2026-10-08, so no separate share step is needed. Then set **General access** to *Anyone with the link: Viewer*, on every workbook, every time. The folder doesn't set it, and the sync reads the workbook with an API key, which needs it. Then, for every format, **Fill match numbers**. Each venue of a day gets its own range (1000, 2000…), because a day's venues show together and no two matches may share a number. For a standard tournament, checking `Timeline` comes after numbering. The run-once rule and what to do when a run fails. | `preparing` §2 and §5, both generator pages (the standard page's "Packing the schedule" step 4), `_templates/CLAUDE.md` "Team events: the workbook" and step 8 |
| **3. Draw and fill rosters** | Running a draw people can trust (seed from the room). Save both exports. **Standard**: Import bracket draws. **Dual meet**: paste the names, then Shuffle roster codes. **Team**: type the `Teams` tab. Check that names show. | `preparing` §3, bracket page's ceremony section, team workbook |
| **4. Register and build the site** | *Admin task.* Add the event to `events.json`: type, title, days and venues, `display` labels, `attendance`, `scoreEntry`. Copy the template, add images and tokens, and the schedule board's `CAT_META` colours. The desk page and scorer page if used. Each step says what to hand the admin, and what to check once they've done it (the Hub loads with the right days and venues). | `preparing` §4–5, `adding-a-new-event`, `_templates/CLAUDE.md` 1–7, 11–12 |
| **5. Connect the workbooks** | In each workbook, check that the Share dialog lists the service account. It is inherited from `1. TOURNAMENTS` (step 2); share it by hand only for a workbook kept outside that folder. **Set up live sync** (do it last, after roster and schedule fixes). Then the first sync from the workbook or Control Center, and checking the site page by page. | `preparing` §5–6, `_templates/CLAUDE.md` 8, 13 |
| **6. Print** | The schedule PDF from the schedule page. Scoresheets: when to print them and which type. The hub board's QR panel (*admin task* to render; scan it before mounting). Bracket images for the noticeboard. | `preparing` §7, 9, 10 |
| **7. Rehearse** | The dry run, written out here as a walkthrough rather than a pointer to a repo file: pick test rows, edit, check the console, check the public screens, clean up. Includes the team checks (lineup, seed cell), scorer links and attendance. Ends with the **"Ready" checklist**. | dry-run checklist Part 1, `preparing` §8 and "What ready looks like" |
| **8. Run the day** | Before doors open, then the screens, go-live, attendance desks and scorer links. During play: the court rhythm (Court Control, then the score), the three ways to enter a score, and what to watch. **Playoff hand-offs**: the standard qualifier draw, team seeds and the twice-to-beat game 2. Late changes. End of day: Awards and exports. | `running-an-event-day`, dry-run checklist Part 2, team seeds, and §8 D4 |
| **9. When something's wrong** | A symptom → cause → fix table. It covers: a venue stale or **Sync failed** (check first that the workbook's General access is still *Anyone with the link: Viewer*); a score not showing after 2 minutes; *polling GitHub*; a bad score already public (Force hidden); the Hub empty (spreadsheet columns); the unmapped-code warning; a generator refusing to run; a score save refused or *sheet changed*; an attendance switch flipping back; a scorer link expired or stopped; a desk link stopped. Point to **SAGE → Help** for workbook messages. | the "If it refuses", "When something goes wrong" and "What to watch for" sections across features; dry-run checklist 2.4 |
| **Scorer handout** | One page to send with the link: open it, pick your venue, find a match, enter, review, save, and what each message means. Written to be readable on a phone. | `features/scorer-page.md` |
| **Desk handout** | The same for desk staff: open the link, pick the venue, search, flip, undo. | `features/event-attendance.md` "Using it", "Desk links" |
| **After the event** | Final results. The Hub stays up, so keep the `events.json` entry. *Admin*: `LIVE_BASE_URL = ''`, and archiving later. What to keep (draw text files, the workbook). | `running-an-event-day` "After the last day", `adding-a-new-event` "After the event", `_templates/CLAUDE.md` §7 |

## 5. Changes to Features

Every features page keeps the existing rule (a counterpart link at the bottom)
and adds one line near the top: **Used in:** a link to the Usage step(s).

| Page | Change |
| --- | --- |
| `README.md` | Rewrite as a reference index grouped as in §3. "Two kinds of event" becomes **Three kinds**, adding team. Point newcomers to Usage. |
| `tournament-hub.md` | Move "At the venue: the schedule board" out to `schedule-board.md`. Point the QR-panel link at Usage → Print. |
| `schedule-board.md` (new) | Venue picker, court split, header collapse, PDF, bookmarks, the team-event cards. Counterpart: `technical/schedule-board.md`. |
| `control-center.md` | Reorder the sections to match the tabs. Move "Entering a score" to `score-entry.md`, leaving two lines and a link. Keep the Mission Control reference. Its day-of usage is already in Usage. |
| `score-entry.md` (new) | The dialog: Enter, Review, Save, corrections, *sheet changed*, publishing failed, setup. Linked from Control Center and the scorer page. Counterpart: `technical/control-center.md#score-entry`. |
| `scorer-page.md` | Reference only. Its step-by-step becomes the scorer handout. |
| `event-attendance.md` | Keep "Turning it on" as reference; Usage steps 4–5 carry the procedure. "Using it" moves to the desk handout, leaving a summary. |
| `scoring-workbook.md` (new) | What a workbook is, one per venue per day. The tabs every format shares, and which ones a person edits (`SCHEDULE` scores, `Court Control`, names) and which they leave alone. **The SAGE menu, every item**, moved here from the dual-meet page. The run-once rule. Counterparts: the generator and sync technical pages. |
| `dual-meet-sheet-generator.md` | Remove the SAGE-menu paragraphs (now in `scoring-workbook`) and the numbered how-to (now in Usage step 2). Keep what it builds, playoff shapes and the refusals. |
| `standard-tournament-generator.md` | Same. "Packing the schedule" and "Filling the rosters" move their procedure to Usage steps 2–3 and keep the rules as reference. |
| `team-workbook.md` (new) | The team workbook's tabs and the inputs a person types (`Teams`, lineups on `MatchUps`, seed cells), and why a matchup is won on points. It says the Team Tournament Master isn't built yet. Counterpart: the team spec and `technical/control-center.md#the-team-type`. |
| `tournament-calculator.md`, `bracket-generator.md`, `scoresheet-generator.md` | Only the **Used in:** line. Change the "step 9 of preparing an event" and "step 3 of…" links to the Usage pages. The bracket page keeps the ceremony section, and Usage links to it. |

## 6. Changes elsewhere in `sage-docs`

- Root `README.md`: four sections instead of two (Usage first). Correct the repo
  table (§2.2) and the tool count.
- `technical/README.md`: fix "Bound Apps Script". It lists three Apps Script
  files, and Event attendance moves under Backend.
- `technical/adding-a-new-event.md`: make it the admin's mechanics page for
  Usage steps 4–6. Reorder to register → site → workbooks. Fix step 8 (both
  masters carry the sync script). Add the scorer page and service-account
  steps. It keeps the full detail, and Usage links in.
- `specs/README.md`: the "start with" line, and an index entry for this spec.
- `mkdocs.yml`: the nav in §3.
- Every link to `features/preparing-an-event.md` and
  `features/running-an-event-day.md`, repointed (`grep -rn` over `docs/`).

## 7. Order of work

1. Create Features' new pages and moves (§5). Usage links into them, so they
   come first.
2. Write Usage: Start here, Glossary and the step pages in order, then
   Troubleshooting, the handouts and After.
3. Delete the two old pages. Repoint links; update the nav and READMEs (§6).
4. `mkdocs build --strict` with no warnings (every link resolves).
5. Read-through: someone who has never run an event follows Usage start to
   finish against `attendance-demo-2026` or a fixture, noting every place
   they had to guess.

Documentation is written in the present tense, as CLAUDE.md requires.

## 8. Owner decisions

All decided 2026-10-08.

- **D1. Repo work in Usage: name it and link Technical.** Each step that
  needs GitHub is an **Admin task** callout. It says what changes in one
  line, what to check afterwards, and links to
  `technical/adding-a-new-event.md` for the git mechanics.
- **D2. Usage comes first:** Home, Usage, Features, Technical, Specs.
- **D3. Handouts are separate pages,** short and readable on a phone, meant
  to be sent with the scorer link or the desk link. Each stands alone: no
  links or references to other pages, and the `handout.html` template takes
  away the menu, tabs, search and footer, so nothing leads off the page.
- **D4. The operational knowledge**, as Usage writes it:
  - **Qualifier draw (standard).** It varies by event. Step 8 describes it
    as a hand-off: it happens after the round robin, before the first playoff
    slot. Whoever runs it (the organizer's call), the result is typed into the
    category tab's qualifier draw area. The step then gives the checks that
    the playoff matches show the right names. It does not prescribe one way
    to draw.
  - **Withdrawals and substitutes.** The organizer decides each case.
    Usage describes the options and their effect, and does not prescribe one:
    - a substitute, or a replacement for a pair that hasn't played yet:
      overwrite the names in `STEP 1 · NAMES`, and the codes and schedule
      stay;
    - a walkover: the opponent's win is entered as a score.
    - Either way, tell the operator, and check that the change reaches the
      Hub.
  - **Schedule change after printing.** For a minor change, correct the
    slips by hand. For a major one, generate the scoresheets again and
    reprint.
  - **Pause live sync** is rarely used, because live sync is set up last.
    Usage mentions it once, in step 5, as an option for a big edit to a
    workbook that is already live.
  - **Staffing** depends on who is available: **at least 2 people, ideally
    3.** "Roles, access and kit" describes the roles (console and workbook,
    screens and desks, players' questions) and how they fold onto two
    people.
  - **Who enters scores** depends on the event. Usage presents all three
    ways as options and says how to choose: the operator in the sheet, the
    operator in Control Center, or scorers with scorer links.
  - **Equipment, every event:** laptop or tablet, venue screens and their
    cables or casting gear, a mobile hotspot, and a printer or a set printed
    beforehand.
  - **Timeline: 1 to 2 weeks before the event, ideally.** "Start here"
    gives the order steps depend on, and a compressed calendar inside that
    window. Planning and the workbooks come first, and the rehearsal and
    printing in the last days.
- **D5. Fix the site repo's dry-run checklist in the same push.** Correct the
  stale labels (**Auto / Force live / Force hidden**, **Open Tournament
  Hub**) and add scorer links.
