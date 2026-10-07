# 1. Plan it

**Who:** coordinator · **When:** 1–2 weeks before · **You need:**
[the organizer's brief](before-you-start.md#what-sage-needs-from-the-organizer): the
date or dates, the venues and how many courts each has, the hours each venue gives
you, and a count of entries per category (an estimate is fine)

The cheapest place to find out an event doesn't fit is here. Moving a number in
the calculator costs nothing. Moving it after the workbook is generated means
starting the workbook again.

## Choose the format

Pick the format first. It decides which master workbook you copy and which site
template the developer uses. See [the three kinds of event](README.md#the-three-kinds-of-event).

## Choose the keys

Two kinds of key, both chosen now and used **byte-identically** from here on.

- **The event key**, for example `pnf-x-bup-dual-meet`. Lowercase letters, digits
  and hyphens. It becomes the event's folder name in the site repo and in
  `event-data`, its key in `events.json`, and it is typed into the workbook's
  `Title` tab.
- **A day key for each tournament day**, for example `pnf-bup-day1`. The same
  characters. **Day keys must be unique across every event ever registered**, not
  just this one, because a day key is also part of the address each sync is sent
  to. Prefix it with something specific to the event (`<event-key>-day1`).

Write both down in a place the coordinator and the developer can see. A key typed two
ways is the most common cause of an event that "looks right and does nothing".

## Steps

1. Open the [Tournament Calculator](../features/tournament-calculator.md) and set
   the **format**: **Standard tournament** or **Dual meet**.
2. Add the **categories**. For a dual meet give each a **pairs per club** count
   and a bracket count. For a standard tournament give each a team count, a
   bracket group size and how many advance to playoffs.
3. Set the **courts** and the **match duration**, with any buffer between matches.
4. For a category with a single bracket, choose how the medals are decided:
   **Twice-to-beat final**, or **Round robin only**. The choice is per category,
   and the finish time updates as you switch.
5. If the event runs across several days or venues, make **one plan per day** (and
   per venue, for a standard tournament).
6. Check the **finish time** against the hours the organizer has the venue. If it is
   too late, agree a change with the organizer (courts, match length or bracket
   sizes) and make it here, not later.
7. When the plan is right, click **Copy plan & open generator**. It puts the plan on
   your clipboard and opens the copy dialog of the master for the plan's format.
   Keep that tab open for [step 2](build-the-workbooks.md).

**Check it worked:** the calculator shows a total match count and a finish time you
accept, and a new tab has opened on the master workbook's **Make a copy** dialog.
If the clipboard is blocked (some browsers block it on non-secure pages), the
button says so: use **Export CSV** and paste the file's contents into the
generator instead.

**If it doesn't:** nothing in the live system is touched by the calculator, so
there is nothing to undo. The calculator remembers your plan between visits.

## Where the formats differ

- **Dual meet:** the plan needs **pairs per club**, and the calculator accounts
  for the cross-club Bronze and Final matches that format produces.
- **Standard:** one plan per venue per day, and you tick each venue's categories
  in the generator later.
- **Team:** the calculator has **no team format yet** (its
  [spec](../specs/not-started/calculator-team-format-spec.md) is not started). Plan
  by hand to PickleDrive's shape: 15 teams `A`–`O` in three brackets of five, four
  pairs per matchup, 152 matches and 10 courts. There is no plan to copy. Skip to
  [step 2](build-the-workbooks.md#team-the-pickledrive-copy).

**Next:** [2. Build the workbooks](build-the-workbooks.md).
