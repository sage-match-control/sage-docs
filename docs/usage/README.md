# Usage guide

A training guide for someone who has never run a S.A.G.E. event. Read front to
back, it takes you from planning the event, through the day of the tournament,
to what you do after it. Every step says **who** does it, **when**, **what it
needs**, and **how to tell it worked**. The guide links to
[Features](../features/README.md) for the detail of each tool and does not repeat
it.

## What S.A.G.E. is

S.A.G.E. Match Control Experts runs pickleball tournaments. The tooling has one
idea at its centre: **the scoring workbook is the source of truth**. A person
types a score into a Google Sheet, and a few seconds later that score is on every
phone, screen and console at the venue. Nothing else needs to be typed.

```text
                       Google Sheet (one per venue per day)
                       scores · Court Control · rosters
                                     │
                                     │  live sync (on every edit)
                                     ▼
                              the sync service
                                     │
              ┌──────────────────────┼───────────────────────┐
              ▼                      ▼                       ▼
     Tournament Hub          Schedule board            Control Center
  (players' phones: Match   (wall display: courts   (operators: resync,
  Finder, live scores,       by time slots, live     go-live switch, awards,
  standings)                 matches ringed)         attendance, score entry)
```

Around that chain sit the tools you use to get ready: the
[Tournament Calculator](../features/tournament-calculator.md) to plan, the
[sheet generators](../features/scoring-workbook.md) to build the workbook, the
[Bracket Generator](../features/bracket-generator.md) to draw, and the
[Scoresheet Generator](../features/scoresheet-generator.md) to print.

## The three kinds of event

Decide which one yours is before anything else. It decides which master workbook
you copy, which site template the developer uses, and which parts of the guide apply.

| Kind | How to tell | Workbooks |
| --- | --- | --- |
| **Standard tournament** | An open-entry bracket event. Players enter in categories (division × event, e.g. Beginner Men's Doubles). Any number of days and venues. | One per venue per day, from the **Standard Tournament Master**. |
| **Dual meet** | Two clubs face off. Every category has the same number of pairs from each club. | One workbook, from the **Dual Meet Master**. |
| **Team** | Named teams of several players meet in matchups of four matches, won on total points. A group stage, then playoffs. | One workbook, copied from PickleDrive's. There is no Team Master yet. |

Where the kinds differ, the step says so in a **Where the formats differ**
section.

## The journey

| Step | Page | Who | When |
| --- | --- | --- | --- |
| Before | [Roles, access and kit](before-you-start.md) · [Glossary](glossary.md) | every SAGE member | read first |
| 1 | [Plan it](plan-the-event.md) | coordinator | 1–2 weeks before |
| 2 | [Build the workbooks](build-the-workbooks.md) | coordinator | 1–2 weeks before |
| 3 | [Draw and fill rosters](draw-and-rosters.md) | coordinator | once entries close |
| 4 | [Register and build the site](register-and-build-the-site.md) | coordinator + developer | about a week before |
| 5 | [Connect the workbooks](connect-the-workbooks.md) | coordinator | after rosters and schedule are final |
| 6 | [Print](printables.md) | coordinator (developer for the QR panel) | the last few days |
| 7 | [Rehearse](rehearse.md) | coordinator + operators | the last few days |
| 8 | [Run the day](run-the-day.md) | operators, scorers, desk staff | the day |
| 9 | [When something's wrong](troubleshooting.md) | operators | open it on the day |
| | [Scorer handout](scorer-handout.md) · [Desk handout](desk-handout.md) | scorers · desk staff | send with their links |
| After | [After the event](after-the-event.md) | coordinator + developer | the days after |

### The order, and why it is that order

Some steps cannot start until another has finished:

- **Register before you build the site and connect.** The Hub reads its days,
  venues and labels from the event's entry in `events.json`, and **Set up live
  sync** checks the day key against it. So the registration comes first.
- **The workbook comes before the draw.** The workbook says how many brackets each
  category has and how many pairs go in each. The Bracket Generator opens from a
  category tab.
- **Registering needs the workbooks.** Each venue's entry carries its workbook's
  ID, so build the workbooks first. (An entry with an empty `sheetId` is allowed,
  to add a day before its workbook exists.)
- **Connect last.** Run **Set up live sync** after the roster and schedule fixes,
  so the editing you do before the event is not publishing half-finished data.
- **Print scoresheets once the schedule stops moving.** A scoresheet carries its
  match number, court and time.

### A compressed calendar

Ideally the work starts **1 to 2 weeks before the event**. Planning and the
workbooks come first, and the rehearsal and printing in the last days.

| When | Do |
| --- | --- |
| 10–14 days before | **1. Plan it.** **2. Build the workbooks**, including filing and sharing. |
| 7–10 days before | **3. Draw and fill rosters** as soon as entries close. **4. Register and build the site**; send the developer their list early. |
| 4–7 days before | Fix the roster and schedule. Then **5. Connect the workbooks**. |
| 2–4 days before | **6. Print** the schedule PDF and the QR panel. **7. Rehearse**. |
| 1–2 days before | Print the scoresheets once the schedule has stopped moving. Brief the scorers and desk staff with their [handouts](scorer-handout.md). |
| The day | **8. Run the day.** Keep [9. When something's wrong](troubleshooting.md) open. |
| After | [After the event](after-the-event.md). |

If you have less than a week, keep the order and compress the gaps. Do not drop
the rehearsal.

## How to read the guide

Every step page has the same frame, so you always know where to look:

> **Who:** coordinator / developer / operators · **When:** how far ahead · **You need:**
> what has to exist first
>
> *Steps*, numbered, one action each, with the exact menu or button name in bold.
>
> **Check it worked:** what you should see.
>
> **If it doesn't:** the common failures, with links to
> [When something's wrong](troubleshooting.md).
>
> **Where the formats differ:** standard / dual meet / team, only when they do.
>
> **Next:** the next step.

A step that needs repo access is set off like this:

!!! note "Developer task — needs GitHub access to `event-data` / the site repo"
    One line naming the change. The step gives you a list to hand the developer, and
    what to check once they have done it. The mechanics are in
    [Adding a new event](../technical/adding-a-new-event.md).

If you are the coordinator and not the developer, the box tells you what to ask for.
Menu names in **bold** are exactly as they appear in the workbook's **SAGE** menu
or on the page.

**Start with [Roles, access and kit](before-you-start.md).**
