# Scorer page

The scorer page is for the staff who enter match scores on tournament day. It
does one thing: it lists a venue's matches and lets you type in a match's score.
It has no standings, no schedule board and nothing to sign into. The operator
gives you a **scorer link**, from [Control Center's Mission
Control](control-center.md#scorer-links), and the link is all you need.

## Opening it

Open the link on your phone. The page keeps the link for next time, so you can
close the tab and come back to the same page later without the link, until it
expires. A newer link replaces an older one.

- A scorer link works for **one day**, at **every venue** of that day, for **24
  hours** from when the operator issued it.
- **"Ask the operator for a scorer link."** There is no link on this device, or
  it is not a scorer link.
- **"This scorer link has expired. Ask the operator for a new one."** It has run
  past its 24 hours. Open a newer link.
- **"Scorer links are stopped for this event. Ask the operator."** The operator
  has stopped scorer links. Nothing can be saved until they accept them again,
  and the page checks every time it loads, so reload once they have.

## Picking a venue

If the day has several venues, the page asks **Which venue are you at?** and
shows no matches until you pick one. A day with one venue picks it for you.

The venue buttons stay at the top of the page the whole time. Tap another venue
whenever you need to, and you see that venue's matches straight away, with no
new link and no confirmation. The page remembers your venue on this device, and
the title line under the page name always says which venue you are scoring.

## Finding a match

Below the venues:

- a **search box**: a match number (`42` or `#42`), a team code, a team name or a
  player's name
- **court buttons**: **All courts**, or one court
- **Hide scored**, on by default, which leaves out matches that already have a
  score so you only see what is left

Each venue keeps its own search, court and **Hide scored** choices, so switching
away and back finds them as you left them. Above the list, a line reads **7 of 24
scored** for the venue.

Each match is a card: its number, time and court, both sides in the order the
sheet has them, and the score on the right, or a dash while it has none. A card
that says **To be decided** has players that aren't known yet and opens read-only.
In a team event the cards show team names and **Lineup not set** where a lineup is
missing. A playoff side whose team isn't known yet reads *Seed 1 · TBD*. The
score dialog names the stage and the pair under the title, for example
*Semifinal · Men's Doubles*.

## Entering a score

Tap a match. The dialog works as it does in Control Center:
[Entering a score](control-center.md#entering-a-score). In short: type both scores,
check the **Winner** line and the score on the **Review** step, then **Save**. You
can correct a score that is already there, or clear it with **Clear score**.

If the sheet changed while you had the match open (someone typed into the sheet,
or another scorer saved), you are shown what it reads now and can **Keep the
sheet's score** or **Replace with yours**.

If saving is refused you are told why, for example that the link is for another
day, or that scorer links have been stopped. If the score reached the sheet but
could not be published, the page says so and to **tell the operator** so they can
resync.

---
**Technical:** [Scorer page](../technical/scorer-page.md) · [auth](../technical/auth.md#scorer-tokens)
