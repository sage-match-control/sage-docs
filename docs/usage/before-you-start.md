# Roles, access and kit

**Who:** the coordinator, before the work starts · **When:** before step 1 ·
**You need:** the organizer's brief, and who from SAGE is free on the day

Four things to settle before the first step: who does what, what each person
needs access to, what gear the day needs, and who enters scores.

## Roles

The **organizer** is the client who hires SAGE to run their tournament. Everyone else
in this guide is a SAGE member, except scorers and desk staff, who can be SAGE members
or volunteers the organizer provides.

| Role | Who | What they do | Steps |
| --- | --- | --- | --- |
| **Organizer** | The client | Hires SAGE and owns the tournament's decisions: the format, the categories and entries, the venue and hours, how a qualifier draw is run, and each late change. Does nothing in the workbooks or the system. | Gives the brief below |
| **Coordinator** | A SAGE member | Prepares the event: the plan, the workbooks, the draw, the rosters, connecting the workbooks, the printing, the rehearsal. Hands the developer what the site needs. | 1–7, [After the event](after-the-event.md) |
| **Developer** | A SAGE member | Everything technical: the event's registration, its website pages, the hub board's QR panel, and the changes marked **Developer task**. Can also be the coordinator. | 4, 6 (the QR panel), [After the event](after-the-event.md) |
| **Operators** | SAGE members on site | Run the day: Control Center, the workbook, Court Control, the screens, the desks, players' questions. | 7, 8, 9 |
| **Scorers** | SAGE members or the organizer's volunteers | Enter match scores on their phones from a scorer link, if the event uses them. | [Scorer handout](scorer-handout.md) |
| **Desk staff** | SAGE members or the organizer's volunteers | Mark people in from a desk link, if the event has attendance with desks. | [Desk handout](desk-handout.md) |

### What SAGE needs from the organizer

The coordinator collects this before step 1, and checks it again when entries close:

- the event's **name**, **dates**, and **venues**, with each venue's **courts** and
  the hours SAGE has them;
- the **format** (standard, dual meet or team) and the **categories**;
- the **entry lists**: every pair per category, or each club's or team's players;
- the event's **logo**, and for a dual meet both clubs' logos;
- whether the event wants **attendance** (check-in), and who staffs the desks;
- whether **scorers** enter scores on their phones, and who they are;
- how a standard tournament's **qualifier draw** is run on the day;
- who on the organizer's side decides **late changes** on the day.

### How many operators

It depends on who is available: **at least 2 operators, ideally 3.** There are three
jobs on the day:

1. **Console and workbook.** Control Center, Court Control, the scores if an
   operator enters them, resyncs, awards.
2. **Screens and desks.** Wall displays, the hub board, attendance desks, scorers.
3. **Players' questions.** Match Finder, the desk, "where is my match?".

With three operators, each takes one. With two, one takes job 1 and the other takes
jobs 2 and 3. Do not fold job 1 onto someone with another full-time job: the court
rhythm needs someone looking at the sheet all day.

### Who enters scores

This depends on the event. There are three ways, and they all write the same two
cells of the match in the workbook, so they can be mixed freely. The coordinator
chooses with the organizer, using these questions:

| Way | Good when | It needs |
| --- | --- | --- |
| **An operator types it in the sheet** | Few courts, one venue, and an operator who is in the sheet anyway. | Nothing extra. |
| **An operator enters it in Control Center** (Match Finder) | The operator would rather not have the sheet open for scores, or wants the review step that names the winner. | `"scoreEntry": "console"` or `"links"` in the registration. Each workbook shared with the service account. |
| **Scorers enter it on their phones** with a scorer link | Many courts or several venues, and people to put at each court. | `"scoreEntry": "links"`, the event's scorer page, and each workbook shared with the service account. Step 4 and step 2 cover these. |

If two people change the same match, the second to save is shown what the sheet
now reads and chooses, instead of overwriting it. See
[Score entry](../features/score-entry.md).

## Access checklist

Check this a week ahead. Most delays on the day are one of these missing.

**Coordinator**

- [ ] Google access to the **masters** (SAGE Dual Meet Master, SAGE Standard
  Tournament Master) so you can make a copy, and to the shared drive folder
  **SAGE → 1. TOURNAMENTS** so you can create the event's folder in it.
- [ ] The Tournament Calculator, Bracket Generator and Scoresheet Generator open
  in a browser. None needs a sign-in.
- [ ] The organizer's brief, and a way to reach the developer.

**Developer**

- [ ] GitHub write access to the site repo (`sage-match-control.github.io`) and
  to `event-data`.
- [ ] A local copy of the site repo, with `sage-tools-api` installed alongside it
  and Chrome on the machine, to render the hub board's QR panel.

**Operators**

- [ ] The Control Center operator **username and password**. You sign in under
  Mission Control. Only operators need it.
- [ ] Edit access to every venue's workbook, from your own Google account.
- [ ] Control Center opened once on the device you will use, signed in, on the
  event and day.

**Scorers and desk staff**

- [ ] A phone with a browser. Nothing to install and nothing to sign into: an
  operator sends a link.

**Only for a workbook made by hand**

- [ ] The sync **shared secret**. A workbook copied from a master carries it
  and never asks. Only a workbook made by hand, with the sync script pasted in,
  asks for it. It is the same value as `SYNC_SHARED_SECRET` in the API's
  settings, so ask the developer. Do not send it in a chat group.

## Equipment

For every event:

- [ ] A **laptop or tablet** for the operators, charged, with the charger.
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
