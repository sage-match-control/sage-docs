# Team workbook

**Used in:** [2. Build the workbooks](../usage/build-the-workbooks.md) ·
[3. Draw and fill rosters](../usage/draw-and-rosters.md) ·
[8. Run the day](../usage/run-the-day.md)

The scoring workbook for a **team event**: named teams meet in **matchups** of
four matches (men's doubles, women's doubles and two mixed doubles). A group
stage is followed by playoffs: Quarterfinals, Semifinals, Bronze and Final.

> **There is no Team Tournament Master yet.** Its
> [spec](../specs/not-started/team-tournament-master-spec.md) is separate work. A
> team event's workbook is made today by copying PickleDrive's workbook and
> clearing its inputs, and **only an event of PickleDrive's shape can be made
> that way**: 15 teams `A`–`O` in three brackets of five, four pairs per matchup,
> 152 matches, one venue and 10 courts. Any other shape waits for the master.
> The steps are in [Usage step 2](../usage/build-the-workbooks.md#team-the-pickledrive-copy).

It is one workbook for the whole event, and it carries the same
[tabs and SAGE menu](scoring-workbook.md) as the others, with the differences
below.

## The tabs a person types in

| Tab | What goes in it |
| --- | --- |
| **`Title`** | The event's title and date. |
| **`Teams`** | The roster: each team's name, and its players with their level and gender. The site's Teams tab, and the roster on Match Finder, read it. Captains can change it through the day. |
| **`MatchUps`** | The organizer's entry tab, with two kinds of input: each matchup's **lineup** (the players for each pair) and the **playoff seed cells**. |
| **`SCHEDULE`** | Scores, as in every workbook. |
| **`Court Control`** | The match number on each court, as in every workbook. |

Every other tab is computed. `Standings`, `FINAL RANK`, `Awards`, `CSV`,
`STANDINGSCSV`, `MatchLookup` and the hidden `StackCache` are formulas. Leave them
alone. `StackCache` is why the workbook answers in seconds rather than a minute.
`Raffle` is a leftover input tab from the first event. `Brackets` is a leftover
to ignore.

### Lineups

A team captain gives the organizer each matchup's players before it is played.
The organizer types them into `MatchUps`. Until a matchup's lineup is in, its
match rows read **Lineup not set** on the site (the schedule board says **Lineup
TBD**). A player can still be found by name from `Teams`. The same player may
play a different pair in each matchup. A match with no lineup can still be scored
in the score dialog, with a warning.

### Playoff seed cells

The site never works out who advances. The organizer types it.

Each playoff slot has a **seed cell** in `MatchUps` that starts as a seed number.
When the group stage ends the organizer types the team's letter (`A`–`O`) over the
seed number. The site then shows that team in the slot and marks the team
**Advances**. Until a seed is filled in, its card reads *Seed 3 · TBD*.

| Slot | Seeds | Cells |
| --- | --- | --- |
| Quarterfinal seeds | 3, 6, 1, 8, 2, 7, 4, 5 | `D604`, `D614`, `D624`, `D634`, `D644`, `D654`, `D664`, `D674` |
| Semifinal seeds | 1–4 | `D684`, `D694`, `D704`, `D714` |
| Bronze | 1–2 | `D724`, `D734` |
| Final | 1–2 | `D744`, `D754` |

Eight teams reach the quarterfinals, seeded 1–8 and paired 3 v 6, 1 v 8, 2 v 7
and 4 v 5. After each playoff round the organizer types the next round's teams
the same way. When rehearsing, put each seed cell back to its seed number
afterwards.

## Why a matchup is won on points

The team with more **total points** across the four matches wins the matchup,
not the team that wins more matches. A team can win three matches 11–9 and lose
the fourth 0–11, and still lose the matchup 33–38. The pair count (for example
*pairs 3–1*) is secondary. If both teams finish on the same total, the matchup
is a tie. See [Tournament Hub § Team events](tournament-hub.md#team-events) for
how a matchup card reads, and the standings tiebreakers.

## Team codes

A team code is `<SIDE>_<PAIR>`: `A_3`, `QF-3_4`, `SF-A_2`, `Fi-J_1`. `PAIR` is the
pair number in the matchup. `SIDE` is a team letter in the group stage and
`<stage>-<slot>` in a playoff, where the stage is `QF`, `SF`, `Br` or `Fi` and
the slot is a seed number until the organizer types a team letter over it.

---
**Features:** [The scoring workbook](scoring-workbook.md) · [Tournament Hub § Team events](tournament-hub.md#team-events)
**Technical:** [Control Center § The team type](../technical/control-center.md#the-team-type) ·
[team tournament spec](../specs/implemented/pickledrive-club-anniversary-team-tournament-spec.md) ·
[Team Tournament Master spec](../specs/not-started/team-tournament-master-spec.md)
