# Adding a new event

Instantiating a new tournament site is a copy-and-fill job against one of
three reusable templates in `sage-match-control.github.io/_templates/`, not a
copy-an-old-event-and-hunt-for-hardcoded-strings job. This page is the admin's
mechanics page for [Usage steps 4 to 6](../usage/register-and-build-the-site.md):
what to change in the repos, in the order it has to happen. It is a condensed
overview; the full step-by-step (with the complete token table)
is `_templates/CLAUDE.md` in that repo.

## Choosing a template

- **Two clubs facing off** → `dual-meet-template/`
- **Named teams meeting in matchups** (a team tournament) → `team-tournament-template/`.
  Its `type` is `"team"` in `events.json`, and its pair labels (MD, WD, XD…) are
  the event's `display.pairs` there. The rules and the first event, PickleDrive
  Club One Year Celebration (hand-built as `events/pickledrive-anniversary-2026/`
  before the template existed), are in the
  [team tournament spec](../specs/implemented/pickledrive-club-anniversary-team-tournament-spec.md);
  the template is the
  [team tournament template spec](../specs/implemented/team-tournament-template-spec.md).
- **Everything else** (open-entry bracket tournament) → `standard-tournament-template/`

Day count and category count don't affect this choice — all three templates
handle any number of tournament days. The standard and dual-meet templates
handle any number of divisions/events. `dual-meet-template/` additionally
handles exactly two clubs; it's not a general multi-club template.

> **A team event's workbook, for now.** There is no Team Tournament Master
> yet (its [spec](../specs/not-started/team-tournament-master-spec.md) is
> separate work), so a team event of PickleDrive's shape (15 teams in three
> brackets of five, four pairs per matchup, 152 matches, one facility) is
> made by copying PickleDrive's workbook and clearing its inputs. The cells
> to clear are in the runbook, `_templates/CLAUDE.md`, "Team events: the
> workbook", and in [Usage step 2](../usage/build-the-workbooks.md#team-the-pickledrive-copy).
> Any other shape waits for the master.

> **Generate the scoring workbook, don't copy last event's.** These steps
> instantiate the *registration* and the *site*, and connect the event's
> Google Sheets to them. They assume the workbooks already exist. Build them
> from the Tournament Calculator plan with the format's generator: the
> [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) for a dual meet,
> which produces every tab the event needs, or the
> [Standard Tournament Generator](standard-tournament-generator.md), one
> workbook per venue per day, which leaves `SCHEDULE` to be packed from its
> `MATCHES` tab. The full sequence, from planning to the day, is the
> [Usage guide](../usage/README.md).

## The steps, in order

The order is register → site → workbooks. The Hub reads its days, venues and
labels from the event's entry in `events.json`, and **Set up live sync** checks
the day key against it, so the registration comes first.

1. **Register the event** in `event-data/config/events.json` — `type`,
   `title`, one entry per day (a globally unique day key, each venue's `name`
   and `sheetId`), and the `display` labels the Hub shows (code to label for
   divisions, events and clubs; key order is display order). Add `"attendance":
   "console"` or `"desks"` and `"scoreEntry": "console"` or `"links"` here if the
   event uses them. See [event registry schema](event-data-config.md). The event
   stays in the file for as long as any page shows it: removing a finished
   event's entry blanks its Hub. Commit; no `sage-tools-api` redeploy needed,
   live within `SYNC_CONFIG_TTL_MS` (about a minute). A day key that is not live
   yet fails a sync with `UnknownSyncDayError`, so give it that minute before
   step 11. Usage step: [4. Register and build the site](../usage/register-and-build-the-site.md).
2. **Copy the template folder** into `events/<event-key>/` in the site
   repo. `<event-key>` becomes this event's folder name in *both* that repo
   and `event-data`, and its key in `events.json` — pick it once, keep it
   identical everywhere.
3. **Add the event's images** — a QR PNG, the event's logo, and (dual-meet
   only) both clubs' logos.
4. **Replace every `{{TOKEN}}`** — event identity, dates, venue, club
   names/codes. `grep -r '{{' events/<event-key>/` must come back empty
   when done.
5. **There is no config to fill in.** The pages are shells of the [site
   engine](site-engine.md): the days, facilities and the division, event and
   club labels come from the event's entry in `events.json` (step 1) at run
   time. A dual meet's `index.html` adds `CLUB_LOGOS`, filled from the club
   tokens.
6. **Leave the theme alone** unless the event genuinely needs its own — every
   page ships the shared S.A.G.E. palette and re-skinning means changing
   one `:root` block, nothing else.
7. **Set up the schedule board** (`schedule.html`) — the day key it shows,
   and `CAT_META` (each category's wall-display color, read off the source
   spreadsheet's own color-coding — see [schedule board](schedule-board.md)
   for why these can't be read any other way).
8. **Attendance desk page, if the event is `"desks"`.** Copy
   `_templates/attendance/attendance.html` to `events/<event-key>/attendance.html`
   and replace `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`. For `"console"`, or no
   attendance, skip it. See [event attendance](event-attendance.md).
9. **Scorer page, if the event is `"links"`.** Copy
   `_templates/scorer/scorer.html` to `events/<event-key>/scorer.html` and replace
   `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`. Mission Control's **Issue scorer link**
   and the **Scorer links** switch are off until the page exists. For `"console"`,
   or no `scoreEntry`, skip it. See [scorer page](scorer-page.md).
10. **Create the data folder** in `event-data` — `<event-key>/data/`, or let
    the first successful sync create it.
11. **Connect each workbook** (Usage step:
    [5. Connect the workbooks](../usage/connect-the-workbooks.md)). Both masters
    carry the sync script (`apps-script/sheets-sync.gs`, from `sage-tools-api`) already,
    so a generated workbook only needs to be reloaded. **Only a hand-made workbook**
    needs the script pasted in (Extensions → Apps Script) and the shared secret
    entered. Reload the spreadsheet and run **SAGE → Set up live sync**, entering the
    day key and facility name; setup verifies both against Cloud Run — including a
    real test sync — before saving. Reloading also adds a **SAGE → Generate
    Scoresheets** menu item deep-linking into the
    [Scoresheet Generator](../features/scoresheet-generator.md) with this
    workbook's day/venue preselected — no extra setup needed for it.
12. **Share every facility workbook with the service account** as Editor, for an
    event with any `attendance` or `scoreEntry` setting:
    `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com`. Without it the
    roster update, every mark and every score save fail with a message naming the
    account. **The share is inherited from the Drive folder:** the shared drive
    folder `1. TOURNAMENTS` is shared with the account as Editor, and a workbook
    moved into the event's folder under it inherits that share. This was confirmed
    on 2026-10-08, when Piggleball's workbook in `1. TOURNAMENTS` listed the account
    as Editor. So move the workbook into the event's folder, and check its Share
    dialog lists the account. Share it by hand only for a workbook kept outside that
    folder. A workbook also needs **General access: Anyone with the link: Viewer**,
    which the folder does not set; the sync reads the workbook with an API key.
13. **Copy the dry-run checklist** template into the event's own folder —
    a combined pre-event rehearsal (fake a few rows, verify the console
    catches them correctly, revert) and day-of runbook. The rehearsal is written
    out in [Usage step 7](../usage/rehearse.md), and the day-of half in
    [Usage step 8](../usage/run-the-day.md).
14. **Render the hub board's QR panel**, once step 4 is done:

    ```
    node _templates/hub-pubmat/render.mjs <event-key>
    ```

    `_templates/hub-pubmat/` is the venue's 24 × 36 in Tournament Hub board:
    a fixed design (`board.html`, phone screenshots in `shots/`) with a QR
    panel per event. The script reads the panel from the event's own
    `index.html` — the name from `<title>`, the date/venue line from the
    hero's `.eyebrow`, the QR image and the short link from its QR panel
    (`{{QR_IMAGE}}`, `{{QR_URL}}`) — and refuses a page with `{{TOKENS}}`
    left in it. Using puppeteer from `sage-tools-api/node_modules` and the
    installed Chrome, it prints `qr-panel.pdf` (8 × 8.75 in, a sticker for
    the board's slot), `board.pdf` (the whole board) and PNGs to the
    git-ignored `_templates/hub-pubmat/out/<event-key>/`, plus
    `out/base/board-blank.pdf`, the board with an empty slot. The folder's
    `README.md` covers re-taking the screenshots. Usage step:
    [6. Print](../usage/printables.md#the-hub-boards-qr-panel).

## Required spreadsheet columns

Check this first if a new event's page loads but renders empty — the most
common cause. Exact, case-sensitive:

- **Matches tab:** `matchNumber`, `teamCode1`, `team1Player1`,
  `team1Player2`, `teamCode2`, `team2Player1`, `team2Player2`, `Schedule`,
  `team1Score`, `team2Score`, `CourtAssignment`, `court`. `court` is the
  *live* court (distinct from the scheduled `CourtAssignment`) — without it
  every court sits on "No match playing" forever.
- **Standings tab:** `teamCode`, `player1`, `player2`, `wins`, `loss`,
  `quotient`, `bracket`.

## Team code format

- Standard: `<DIVISION><EVENT>_<REST>` (e.g. `B18MD_1`, `HI40XD_SF_2`,
  `B35XD_F_1_(2)`)
- Dual meet: `<CLUB>_<DIVISION><EVENT>_<REST>`
- Team: `<SIDE>_<PAIR>` (e.g. `A_3`, `QF-3_4`, `SF-A_2`, `Fi-J_1`): the pair
  number in the matchup, after a team letter or a playoff side

`_(N)` suffixes mark a twice-to-beat playoff instance. See [Control
Center, incl. Awards tab](control-center.md) for how that gets parsed and
resolved.

## Root-absolute asset paths — don't "fix" them to relative

Both templates load icons and images with root-absolute paths
(`/assets/logo.png`, one shared `assets/` folder at the repo root), not
relative ones — a relative path resolves differently depending on how deep
a page is nested, so root-absolute paths work regardless of nesting. Leave
them that way; archiving an event later is then a plain `mv` with nothing
to re-prefix.

## After the event

Once the event's last day is over, set `LIVE_BASE_URL = ''` in the settings
script of its `index.html`, `schedule.html` and `scorer.html`, and change
nothing else. Nothing is published for the event any more, so
an open socket would only cost Worker requests (a connect per visit and a
ping every 50 s) against the free plan's daily cap. With the constant empty
the pages read their snapshot from GitHub and show the final results as
before. The folder stays put, at the address the venue's QR code points
to; archiving it is a separate step.

## Things kept in sync by hand (no automatic check)

- `event-data/config/events.json`'s `days` ↔ the schedule board's `DAY_KEY`
  ↔ each spreadsheet's day key and facility name, set through
  **SAGE → Set up live sync** and stored in that workbook's Script
  Properties (not in `sheets-sync.gs`'s source, which is identical in every
  workbook). Facility names are compared exactly, case-sensitive.
- `EVENT_KEY` — identical across the site-repo folder name, the
  `event-data` folder name, and the registry key (and the attendance desk
  page's `EVENT_KEY`, when there is one).

---
**Full runbook:** `_templates/CLAUDE.md` in the site repo (complete token
table, required-column detail, archiving instructions).
**Related:** [event registry schema](event-data-config.md), [the Usage guide](../usage/README.md).
