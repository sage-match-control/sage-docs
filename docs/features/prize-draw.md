# Prize draw (Lucky Dink)

**Prize draw** draws raffle and door-prize winners in front of a crowd. On screen it is headed
**S.A.G.E. Lucky Dink**. It lives at `/tools/prize-draw`, opens from Control Center's **Pages and
tools** row (on the selected event and day) and from the tools roster, and is linked from no public page.
Anyone with the link can open it; loading who has checked in and recording to the workbook need a
Control Center sign-in in the same browser.

## Setting up a draw

- **Title, event and day.** Optional. Picking an event and day lets the page load that day's people and
  record winners to its workbook. A day with several facilities asks which one to record to.
- **Names.** One per line, from one of four places: **paste** a list; **Players on a day** (everyone in the
  day's standings, or a team event's rosters); **Checked in** (the people who have checked in, needs the
  sign-in); or **Numbers** (001 to N, for physical ticket stubs). The same person twice is kept once and
  reported. A draw takes 2 to 1000 names.
- **Prizes.** Each prize has a name and a quantity. Each unit gets its own spin, labelled *Ball pack
  (1 of 3)*, unless **Draw all at once** is ticked, which draws every remaining unit in one spin. Prizes
  can be reordered at any time with the arrows and **Next**: the next spin is for the first prize not yet won.
- **The rule.** By default **each winner leaves the draw**. Untick it and winners stay in.

**Lock the list and start** fixes the names and rule, makes a random **draw code** and shows the list's
**fingerprint**. **Unlock** reopens the list until the first spin.

## Drawing

- **Spin** (the button, Space or Enter) draws the next prize. The winners are decided before the animation
  starts; the animation only plays the result out.
- **Draw again.** If a winner isn't there, **Draw again** beside the latest spin's winner takes them out and
  the same prize is drawn on the next spin. Earlier spins can't be changed.
- **Edit list** (after spins) closes this part of the draw and starts a linked continuation: earlier winners
  stay out, prizes not yet won carry over, and each part is checked on its own.
- The draw ends when every prize has a winner, or nobody is left.

## Two styles, one display window

The **Reel** (the default) lands on the winner's name; the **Pickle Jar** mixes one ball per name (up to 300)
and rolls the winners out. **Open display** shows either on a second window in the same browser, for the
TV, while the controls stay on the first. **Big screen** fills the controls' own window instead.

## Afterwards

- **Copy results**, **Download .txt** and **Save image** (a winners board); **Copy numbered list** posts the
  locked list with ticket numbers before a draw.
- **Check this draw** works every spin out again and ticks each winner.
- On an event day with a sign-in, every spin is recorded as rows of a `DRAWS` tab in the facility workbook,
  and **Past draws** lists, checks and continues recorded draws.

## No Rigging Zone

The **How it works** dialog says: the list is locked before anything is drawn, and its fingerprint changes
if one letter does; a random code made by the page decides every spin, so nobody can steer it, and the
prizes never enter it; and anyone with the list, the code and the rule can check every winner.

---
**Technical counterpart:** [Prize draw](../technical/prize-draw.md).
