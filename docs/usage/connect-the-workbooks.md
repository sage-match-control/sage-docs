# 5. Connect the workbooks

**Who:** organizer · **When:** 4–7 days before, **after** the roster and schedule
fixes are done · **You need:** the event registered and its site built
([step 4](register-and-build-the-site.md)), every workbook filed and numbered
([step 2](build-the-workbooks.md)), and rosters in
([step 3](draw-and-rosters.md))

This step makes typing a score publish it. Do it **last**, once the roster and
schedule are right, so the editing you do before the event isn't publishing
half-finished data. Do the steps once for every workbook.

## Steps

1. **Check the service account.** In each workbook choose **Share** and check that
   the dialog lists
   `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` as **Editor**.
   It is inherited from the folder **1. TOURNAMENTS** (see
   [step 2](build-the-workbooks.md#file-it)). Share it by hand only for a workbook
   kept outside that folder. This matters for any event with attendance or score
   entry: without it the roster update, every mark and every score save fail with a
   message naming the account.
2. **Reload the workbook** so the **SAGE** menu is current.
3. Choose **SAGE → Set up live sync**.
4. Enter the **day key** (exactly as registered) and the **venue name** (exactly as
   registered, with capitals). Confirm the tabs to watch. `SCHEDULE` and `Court
   Control` are ticked for you. For a team event, check that `MatchUps` is ticked as
   well, or lineups and playoff seeds won't publish.
5. The dialog reads **Secret stored** and goes ahead. Setup checks the day key and
   venue against the registry and runs a real test sync before saving anything, so a
   mismatch is caught on the spot.
6. Repeat for every other workbook. Each has its own day key and venue name.

Only a workbook made by hand, with the sync script pasted in, asks for the shared
secret. See [Roles, access and kit](before-you-start.md#access-checklist).

**If you must edit a workbook that is already live,** such as a late roster or
schedule change after this step, **SAGE → Pause live sync** stops automatic
publishing without losing the saved settings. **Resume live sync** restarts it.
You will rarely need it, because setup comes last.

## The first sync

Before the event there is nothing to wait for, so push the first copy yourself.
Either way works:

- **From the workbook:** **SAGE → Sync now**, in each venue's workbook. It reports
  what it sent: the venue and day, whether open pages got it straight away, and how
  long it took.
- **From Control Center:** sign in, open Mission Control and click **Resync this
  day now**, which pulls a fresh copy from every venue of that day at once.

## Attendance

If the event has attendance, open Control Center's **Attendance** tab and press
**Update roster**. It reports each venue's people.

## Check it worked

Open the event page and go through it page by page:

- [ ] Every **day** and **category** is present.
- [ ] Every **venue** is listed.
- [ ] The **schedule** shows the matches you packed or generated.
- [ ] **Standings** lists every pair by name.
- [ ] Mission Control's **Facility sync status** reads a fresh **Synced** for every
  venue.
- [ ] With attendance, **Update roster** reports every venue's people.

This is the first point at which a wrong venue name or a missing column shows
itself.

## If it doesn't

- **Setup says the day key or venue is not found.** The registration and the
  workbook disagree. Venue names are case-sensitive. Fix whichever is wrong. If you
  just changed the registration, wait a minute.
- **Sync failed.** The words after **Sync failed** say why, and **SAGE → Help**
  explains each one. First check that the workbook's **General access** is still
  **Anyone with the link: Viewer**.
- **The Hub is empty.** The column names in `CSV` or `STANDINGSCSV` have changed.
  See [When something's wrong](troubleshooting.md).
- **A venue shows no matches.** Its `SCHEDULE` is empty, or its matches have no
  numbers. Go back to [step 2](build-the-workbooks.md#fill-match-numbers).

## Where the formats differ

- **Standard:** one setup per venue per day, each with that day's key.
- **Dual meet:** one workbook, so one setup.
- **Team:** the copy keeps the sync script but **not its trigger**. This step creates
  it and replaces the copied day key.

**Next:** [6. Print](printables.md).
