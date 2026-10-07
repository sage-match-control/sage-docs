# 4. Register and build the site

**Who:** organizer hands over the list below; **admin** makes the changes ·
**When:** about a week before · **You need:** the keys from
[step 1](plan-the-event.md#choose-the-keys), each workbook filed in
[step 2](build-the-workbooks.md#file-it) with its link, and the event's images

The event is inert until two things exist: its **registration** (the entry in
`event-data/config/events.json`) and its **event site** (the pages in the site
repo). Register first. The Hub reads its days, venues and labels from the
registration, and [step 5](connect-the-workbooks.md)'s **Set up live sync** checks
the day key against it.

## What to hand the admin

Send this list, filled in, in one message.

- [ ] The **event key** and every **day key**.
- [ ] The event's **type**: standard, dual meet or team.
- [ ] The **title**, the **dates**, and the **venue** name.
- [ ] For each day, each **venue name** (spelled and capitalised exactly as you will
  type it in **Set up live sync**) and the **link to its workbook**.
- [ ] The **labels** the Hub shows: a name for every division, every event, and (dual
  meet) both clubs; or, for a team event, the label of each pair (MD, WD, XD…).
- [ ] Whether the event uses **attendance** (`console` or `desks`) and **score entry**
  (`console` or `links`). See [who enters scores](before-you-start.md#who-enters-scores).
- [ ] The images: a **QR image** that opens the event's page, its **short link**, the
  event's **logo**, and for a dual meet **both clubs' logos**.
- [ ] The schedule board's **category colours**, read off the organizer's
  colour-coded `SCHEDULE` tab (standard and dual meet).

## Register the event

!!! note "Admin task — needs GitHub access to `event-data`"
    Add the event's entry to `event-data/config/events.json` and commit it to
    `main`. No redeploy: it takes effect within about a minute. The mechanics are in
    [Adding a new event](../technical/adding-a-new-event.md).

The entry carries:

- **`type`** — `"standard"`, `"dual-meet"` or `"team"`. Required and explicit.
- **`title`** — the masthead label.
- **`days`** — one entry per day. Each has a unique **day key**, a `label`, a `date`,
  and its **`facilities`**: each venue's `name` and the workbook's `sheetId`.
- **`display`** — labels for divisions, events and clubs (code → label; the order of
  the keys is the display order). A team event adds `display.pairs`.
- **`attendance`** — `"console"` or `"desks"`, only if the event uses it.
- **`scoreEntry`** — `"console"` or `"links"`, only if the event uses it.

Day keys must be unique across every event in the file. Venue names are compared
**exactly, with capitals**. The entry stays in the file for as long as any of the
event's pages are up: removing a finished event's entry blanks its Hub.

## Build the site

!!! note "Admin task — needs GitHub access to the site repo"
    Copy the matching template to `events/<event-key>/`, replace its tokens, add
    the images, and commit. The mechanics are in
    [Adding a new event](../technical/adding-a-new-event.md).

1. Copy the template that matches the type: `standard-tournament-template/`,
   `dual-meet-template/` or `team-tournament-template/`.
2. Replace every `{{TOKEN}}` (event key, title, tagline, date range, venue, QR image
   and short link, logo; a dual meet adds club codes, names and logos). A search for
   `{{` in the folder must come back empty.
3. Put the images in the folder.
4. On the **schedule board** (`schedule.html`), set the **day key** it shows and its
   `CAT_META`: each category's wall-display colour, read off the organizer's
   `SCHEDULE` tab. A team event's board is coloured by bracket and has one colour
   for playoffs. Nothing detects it if the organizer recolours the sheet later.
5. Leave the theme alone unless the event genuinely needs its own.

There are no days, venues or categories to fill in on the pages: they come from the
registration.

## Add the desk page and the scorer page, if used

!!! note "Admin task — needs GitHub access to the site repo"
    Copy `_templates/attendance/attendance.html` to
    `events/<event-key>/attendance.html` if the event is `"desks"`, and
    `_templates/scorer/scorer.html` to `events/<event-key>/scorer.html` if it is
    `"links"`. Replace `{{EVENT_KEY}}` and `{{EVENT_TITLE}}` in each.

Without the desk page, a desk link opens a missing page. Without the scorer page,
Mission Control's **Scorer links** switch and **Issue scorer link** are off, with a
note. `"console"` score entry needs neither page.

## Check it worked

Once the admin says it is done, **give it a minute**, then check:

- [ ] The event appears in Control Center's event picker, with its days.
- [ ] The Hub (`/events/<event-key>/`) loads and shows the right **days** and
  **venues**, with the right labels.
- [ ] The schedule board page (`/events/<event-key>/schedule`) loads.
- [ ] For `"desks"`, `/events/<event-key>/attendance` loads. For `"links"`,
  `/events/<event-key>/scorer` loads.
- [ ] Mission Control's **Check connection** shows the registry copy it is using.

The Hub shows no scores yet. That is [step 5](connect-the-workbooks.md).

## If it doesn't

- **The Hub loads but renders empty.** Check the spreadsheet's column names (`CSV`
  and `STANDINGSCSV` are exact and case-sensitive) and that the event's `display`
  labels exist. See [When something's wrong](troubleshooting.md).
- **The event is missing from Control Center.** The commit has not landed, or the
  file failed validation. A file that fails any rule is rejected as a whole: ask the
  admin to check the commit.
- **A day key is rejected.** It is already used by another event. Choose a new one
  and use it everywhere.

## Where the formats differ

- **Standard:** `standard-tournament-template/`; `display.divisions` and
  `display.events`.
- **Dual meet:** `dual-meet-template/`; `display.clubs`; both clubs' logos and codes.
- **Team:** `team-tournament-template/`; `type` is `"team"`; `display.pairs`. Team
  names come from the workbook, so no division or club labels are needed.

**Next:** [5. Connect the workbooks](connect-the-workbooks.md).
