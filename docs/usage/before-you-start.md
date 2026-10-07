# Roles, access and kit

**Who:** the organizer, before assigning anyone a job · **When:** before step 1 ·
**You need:** your event's staff list

Four things to settle before the first step: who does what, what each person
needs access to, what gear the day needs, and who enters scores.

## Roles

| Role | What they do | Steps |
| --- | --- | --- |
| **Organizer** | Owns the event: the plan, the workbooks, the draw, the registration list, the printing. Decides each late change. | 1–7, [After the event](after-the-event.md) |
| **Admin** | Has GitHub access to the site repo and `event-data`. Makes the changes marked **Admin task**. Often the same person as the organizer, but often not. | 4, [After the event](after-the-event.md) |
| **Operator** | Runs the console and the workbook on the day. Signs in to Control Center, watches sync, flips the go-live switch, puts matches on courts, resyncs. | 7, 8, 9 |
| **Scorers** | Enter match scores on their phones from a scorer link, if the event uses them. | [Scorer handout](scorer-handout.md) |
| **Desk staff** | Mark people in from a desk link, if the event has attendance with desks. | [Desk handout](desk-handout.md) |
| **Screen runner** | Sets up the wall displays and keeps them running. Needs only the schedule board's page address and the venue's cables. | 8 |

### How many people

It depends on who is available: **at least 2 people, ideally 3.** There are three
jobs on the day:

1. **Console and workbook.** The operator: Control Center, Court Control, the
   scores if the operator enters them, resyncs, awards.
2. **Screens and desks.** Wall displays, the hub board, attendance desks, scorers.
3. **Players' questions.** Match Finder, the desk, "where is my match?".

With three people, each takes one. With two, the operator takes job 1 and the
other person takes jobs 2 and 3. Do not fold job 1 onto someone with another
full-time job: the court rhythm needs someone looking at the sheet all day.

### Who enters scores

This depends on the event. There are three ways, and they all write the same two
cells of the match in the workbook, so they can be mixed freely. The organizer
chooses, using these questions:

| Way | Good when | It needs |
| --- | --- | --- |
| **The operator types it in the sheet** | Few courts, one venue, and an operator who is in the sheet anyway. | Nothing extra. |
| **The operator enters it in Control Center** (Match Finder) | The operator would rather not have the sheet open for scores, or wants the review step that names the winner. | `"scoreEntry": "console"` or `"links"` in the registration. Each workbook shared with the service account. |
| **Scorers enter it on their phones** with a scorer link | Many courts or several venues, and spare staff to put at a court. | `"scoreEntry": "links"`, the event's scorer page, and each workbook shared with the service account. Step 4 and step 2 cover these. |

If two people change the same match, the second to save is shown what the sheet
now reads and chooses, instead of overwriting it. See
[Score entry](../features/score-entry.md).

## Access checklist

Check this a week ahead. Most delays on the day are one of these missing.

**Organizer**

- [ ] Google access to the **masters** (SAGE Dual Meet Master, SAGE Standard
  Tournament Master) so you can make a copy, and to the shared drive folder
  **SAGE → 1. TOURNAMENTS** so you can create the event's folder in it.
- [ ] The Tournament Calculator, Bracket Generator and Scoresheet Generator open
  in a browser. None needs a sign-in.
- [ ] The name of the admin, and a way to reach them.

**Admin**

- [ ] GitHub write access to the site repo (`sage-match-control.github.io`) and
  to `event-data`.
- [ ] A local copy of the site repo, with `sage-tools-api` installed alongside it
  and Chrome on the machine, to render the hub board's QR panel.

**Operator**

- [ ] The Control Center operator **username and password**. You sign in under
  Mission Control. Nobody else needs it.
- [ ] Edit access to every venue's workbook, from your own Google account.
- [ ] Control Center opened once on the device you will use, signed in, on the
  event and day.

**Scorers and desk staff**

- [ ] A phone with a browser. Nothing to install and nothing to sign into: the
  operator sends a link.

**Only for a workbook made by hand**

- [ ] The sync **shared secret**. A workbook copied from a master carries it
  and never asks. Only a workbook made by hand, with the sync script pasted in,
  asks for it. It is the same value as `SYNC_SHARED_SECRET` in the API's
  settings, so ask the person who maintains S.A.G.E. Do not send it in a chat
  group.

## Equipment

For every event:

- [ ] A **laptop or tablet** for the operator, charged, with the charger.
- [ ] **Venue screens**, with the **cables or casting gear** for them. The schedule
  board runs in a browser, so any screen that can show a web page will do.
- [ ] A **mobile hotspot**, as a fall-back for the venue's wifi.
- [ ] A **printer**, or the printed set made beforehand: scoresheets, the schedule
  PDF, bracket images.
- [ ] The **hub board**, with this event's QR panel stuck on.

Also useful: a phone charger for each scorer, tape for the schedule on the
noticeboard, and spare scoresheets (the Scoresheet Generator can add blank
ones).

**Next:** [Glossary](glossary.md), then [1. Plan it](plan-the-event.md).
