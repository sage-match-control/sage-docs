# Event attendance

**Used in:** [4. Register and build the site](../usage/register-and-build-the-site.md) ·
[5. Connect the workbooks](../usage/connect-the-workbooks.md) ·
[8. Run the day](../usage/run-the-day.md) · [Desk handout](../usage/desk-handout.md)

Staff check-in for any event. A person is marked in once, and that covers
every category they play that day at that venue. Marks are made in
[Control Center](control-center.md), or on a desk page opened from a link, and
land in an **ATTENDANCE** tab of each venue's scoring workbook.

## Turning it on

Attendance is a setting on the event, in `event-data/config/events.json`:

| Setting | What it means |
| --- | --- |
| `"attendance": "console"` | Operators mark people in the **Attendance** tab of Control Center. |
| `"attendance": "desks"` | Operators can do that too, and desk staff can mark people from their own phones with a **desk link**. |
| *(absent)* | The event has no attendance. The tab does not appear. |

Each venue's workbook must be shared with the API's service account as
**Editor**, so it can write the tab. Workbooks filed in the event's folder under
**1. TOURNAMENTS** inherit the share; see
[Usage step 2](../usage/build-the-workbooks.md#file-it). The procedure is in
[Usage steps 4 and 5](../usage/register-and-build-the-site.md).

## The tab

The workbook generators add an empty **ATTENDANCE** tab to every workbook they
build, so it is there from the start with its header and the **Not yet in** list.
**Update roster** then only fills it in. A workbook without one gets the tab
created on the first Update roster.

## The roster

The roster is made for you. After every sync the API compares each venue's
players with its ATTENDANCE tab and brings the tab up to date: people missing
from it are added, a changed team or category is corrected, and someone who is
no longer on the roster is flagged **withdrawn**. A person who has been
withdrawn keeps their row and any check-in, and is hidden from the list. If
they come back, they are restored.

To do it right now instead of waiting for a sync, press **Update roster** in
the Attendance tab. It reports, for each venue, how many people were added,
updated or withdrawn.

Two names count as the same person when they match once spacing, capitals
and accents are ignored (`José Rizal` and `jose  rizal`). Names that are
*nearly* the same are never merged for you. They appear under **Needs
attention** so you can fix the spelling in the category tab; the roster
follows on the next sync. A person listed twice in the same category appears
there too.

Standard and dual-meet rosters come from the standings tab. A team event's
comes from its **Teams** tab.

## Using it

Desk staff open a desk link on a phone, pick the venue, find the person with the
search box or a category (or team) chip, and flip their switch. It shows
*Saving…* for a moment, then *In 8:14 AM*. Flipping it back clears the time. A
person who plays in two categories is one switch, shown in both places. When
everyone on a team is in, it shows **Ready**. Marks made on other devices appear
within about 10 seconds, or when you press **Refresh**.

The step-by-step is the [desk handout](../usage/desk-handout.md), a page to send
with the desk link.

## Control Center extras

The Attendance tab, for operators, also has:

- the count of people in at each venue
- **Update roster** (above)
- **Issue desk link** (events set to `"desks"`): a link for one day, with
  **Copy**, **Share** and **Show QR**. It works until the end of that day in
  Manila time. Send it to the desks, or let them scan the QR code.
- **Needs attention**: possible duplicate names, people listed twice in a
  category, and a **Show withdrawn** switch.

Sign in at the top of Mission Control first. Without signing in the list is
read-only.

## Desk links

A desk link opens that event's attendance page on a phone and can do one
thing: mark people in or out, on that day. It cannot reach anything else.
Switching the event to `"console"` stops every link already handed out, within
about a minute. A link that has stopped working shows *Ask the operator for a
new desk link.*

## When something goes wrong

- **A switch flips back and a message says it was not saved.** Usually a
  dropped connection. Try again. If the message names the service account,
  the venue's workbook is not shared with it.
- **"No roster yet."** Press **Update roster**.
- **"The ATTENDANCE tab's header row is not …"** Someone changed or replaced
  the tab's first row. Put it back, or delete the tab and press **Update
  roster** to build a new one.

Use a **filter view**, not Sort, to rearrange the ATTENDANCE tab by hand.
Anything you add in columns to the right of **withdrawn** (a `TShirt Size`
column, say) is left alone, and shows beside each name.

A tab the API creates also has a **Not yet in** list in column J: the names of
everyone who is not yet in and not withdrawn, shrinking as people are marked.
It is for the person running the workbook. A tab that already exists is left as
it is, so put the formula in `J2` by hand there
(`=IFNA(FILTER(TRIM(B2:B1000), E2:E1000=FALSE, G2:G1000=FALSE, B2:B1000<>""))`).

The tab can be read by anyone who has the workbook's ID, so keep phone numbers
and emails out of it.

## Pickle for Sight

Pickle for Sight 2026 has its own check-in page and a per-workbook Apps Script,
and nothing above applies to it. It keeps one row per pair slot, has its own page at `/events/pickle-for-sight-2026/attendance`
and does not use desk links or the Attendance tab.

---
**Technical:** [event attendance](../technical/event-attendance.md)
