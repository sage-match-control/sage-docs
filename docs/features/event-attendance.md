# Event attendance

A check-in page for the staff at an event's desk. Open it on any phone, find a
player, and flip their switch: they're marked in, with the time they arrived.
No app, no sign-in.

Pickle for Sight is the first event with one, at
`/events/pickle-for-sight-2026/attendance`. It isn't linked from the public
event page — share the link with desk staff only.

## Using it

- **Pick your venue** at the top. The page remembers it on that phone.
- **Find the player** by scrolling to their category, or type part of a name
  or a team code into the search box.
- **Flip their switch.** It shows *Saving…* for a moment, then *In 08:14*.
  Each player has their own switch, so a pair with a partner still on the way
  shows exactly who is missing. Once both are in, the pair is marked
  **Ready**.
- **Marked the wrong person?** Flip it back. That clears the time.
- The count at the top shows how many players are in at that venue.

Several phones can mark at once. Each picks up the others' marks within about
30 seconds, straight away when you come back to the page, or when you press
**Refresh**.

## When something goes wrong

- **A switch flips back with a message at the top.** That mark was not saved
  — usually a dropped connection. Try again.
- **"… is not connected yet."** That venue's workbook hasn't been set up for
  attendance. Tell whoever runs the event's workbooks.
- **A player is missing or misspelled.** The list comes from the scoring
  workbook's roster. Fix it there; the page picks it up on the next refresh.

## What it records

Each mark lands in an **ATTENDANCE** tab of that venue's scoring workbook:
team code, which player, their name, whether they're in, and the time. None
of it appears on the public event page.

Anyone who has the link can change marks, so share it with desk staff only.

---
**Technical:** [event attendance](../technical/event-attendance.md)
