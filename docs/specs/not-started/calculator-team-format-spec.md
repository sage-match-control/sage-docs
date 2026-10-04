# Spec — Tournament Calculator: team tournament format

> **Status: not started.** Nothing here is built. Written 2026-10-05 against
> `sage-match-control.github.io/tools/tournament-calculator.html` as of that date
> and the first team event,
> [PickleDrive Club One Year Celebration](../implemented/pickledrive-club-anniversary-team-tournament-spec.md).
> The rules and the model are settled by that event; §9 lists the few questions
> that are the owner's.

Add a third option to the calculator's **Format** menu, **Team tournament**, so a
team event can be planned the way a standard tournament or a dual meet already
is: how many matchups and matches, how many time slots, and what time the last
match finishes, before the draw is final.

Independent of the [Team Tournament Master](team-tournament-master-spec.md): this
changes only the calculator. It does give that generator an input to read later
(§6), but the calculator ships first without it.

---

## 0. How to use this spec

- **One file changes:** `sage-match-control.github.io/tools/tournament-calculator.html`
  (a single self-contained page: inline `<style>`, inline `<script>`, no build step,
  2-space indent, single quotes, classic script). Docs change in `sage-docs`.
- **Find code by function name**, not line number. The functions named below exist
  today: `calcAll`, `calcCategory`, `calcDualCategory`, `catTime`, `readForm`,
  `recalc`, `buildPlanCsv`, `importCSVText`, `exportXLSX`, `toggleDualClubsVisibility`,
  `splitEven`, `GENERATOR_MASTERS`.
- **Don't touch** the standard or dual-meet maths, their CSV columns, or `sw.js`
  (§7).
- Working copies use CRLF line endings in some repos; keep each file's.

## 1. Why the model differs from the other two formats

| | Standard | Dual meet | **Team** |
| --- | --- | --- | --- |
| Unit that plays | a pair, in a category | a pair vs a pair from the other club | a **team** meets a team: one **matchup** of several matches |
| Categories | many, each with its own count and brackets | many | **none**: one competition |
| Group stage | round robin inside a bracket, one match per pairing | cross-club, one match per pairing | round robin inside a group, one **matchup** per pairing, each *P* matches |
| Playoffs | per category, from the top *k* of each group | per category | one shared playoff: QF, SF, Bronze, Final, each a matchup |
| What limits the clock | courts, and that a pair can't be on two courts | the same | courts, **and** that a team plays one matchup at a time, which takes several slots |

So the calculator treats the team format as **one competition with no category
cards**, and works in matchups rather than matches.

## 2. The model

Inputs (all new, team format only):

| Input | Default | Meaning |
| --- | --- | --- |
| Teams, *T* | 15 | |
| Groups, *G* | 3 | Teams are split as evenly as possible with `splitEven(T, G)`, bigger groups first (15 into 3 is 5, 5, 5) |
| Matches per matchup, *P* | 4 | One match per pair of the two teams |
| Matches on court at once, *S* | 2 | How many of a matchup's matches share a slot. PickleDrive played pairs 1–2 in one slot and pairs 3–4 in the next |
| Teams to the playoffs, *A* | 8 | 2, 4 or 8 (§9 Q2). The organiser chooses who; the calculator only counts |
| Break before the playoffs (min) | 0 | Idle time between the last group match and the first playoff slot |

The existing **Courts**, **Match duration**, **Buffer** and **Start time** inputs
are reused as they are.

Derived:

- **Group matchups** = Σ over groups of *n(n−1)/2*. **Group matches** = that × *P*.
- **Playoff matchups** by *A*: 8 → QF 4 + SF 2 + Bronze 1 + Final 1; 4 → SF 2 +
  Bronze 1 + Final 1; 2 → Final 1. **Playoff matches** = matchups × *P*.
- **Total matchups** and **total matches** are the sums.
- **Block** = `ceil(P / S)` slots: how long one matchup occupies the courts it
  uses. **Lanes** = `floor(courts / S)`: how many matchups can run at once.
  (Courts below *S* is an error: §5.)

Time, in slots (each slot is `dur + buf` minutes):

- **Group stage** = `max( ceil(groupMatchups / lanes), R ) × block`, where *R* is
  the biggest group's round count (*n − 1* if *n* is even, *n* if odd, since a team
  can't be in two matchups at once, so a group can't finish faster than its rounds).
  Groups run side by side.
- **Each playoff round** = `ceil(matchupsInRound / lanes) × block`. **Bronze and
  Final are one round**, played at the same time. Rounds run one after another.
- **Total slots** = group slots + Σ playoff round slots.
- **Play time** = `slots × (dur + buf) − buf + break`. **Projected finish** = start +
  play time, formatted by the existing `fmtTime`, with its "+1d" handling.

### Worked examples (the verification targets)

15 teams, 3 groups, *P* 4, *S* 2, 10 courts, 25-minute matches, no buffer, 9:00 AM
start. Lanes 5, block 2, *R* 5, 30 group matchups = 120 matches, group stage
`max(6, 5) × 2` = **12 slots**.

| Plan | Matchups | Matches | Playoff slots | Slots | Break | Finish |
| --- | --- | --- | --- | --- | --- | --- |
| *A* 4 (PickleDrive's published plan: SF, Bronze, Final) | 34 | 136 | 4 | 16 | 25 min | **4:05 PM** |
| *A* 8 (the quarterfinal plan the workbook was changed to) | 38 | 152 | 6 | 18 | 0 | 4:30 PM |
| *A* 8 | 38 | 152 | 6 | 18 | 25 min | 4:55 PM |

The first row matches the event's published schedule: the group stage ends at
2:00 PM, the semifinals start at 2:25 PM after a one-slot break, and the Final and
Bronze start at 3:15 PM and 3:40 PM. The counts (136 and 152 matches, 34 and 38
matchups) are the event's own.

## 3. UI

- **Format** gains `<option value="team">Team tournament</option>`. Standard and
  dual are unchanged.
- **Team format shows:** the existing Tournament details panel (title, date, start,
  courts, duration, buffer), plus a new **Team tournament** panel with the six
  inputs of §2, in place of the **Categories** panel. The club-name fields stay
  hidden, as for standard. Category data is kept in memory, so switching format
  and back loses nothing.
- **Results** keep the big clock, the stat grid (matches, slots, play time) and
  the alerts. The breakdown table becomes **Breakdown** with one row per stage:

  | Stage | Matchups | Matches | Slots | Time |
  | --- | --- | --- | --- | --- |
  | Group stage (3 groups: 5-5-5) | 30 | 120 | 12 | 5h |
  | Quarterfinals | 4 | 16 | 2 | 50m |
  | Semifinals | 2 | 8 | 2 | 50m |
  | Bronze + Final | 2 | 8 | 2 | 50m |
  | **Total** | **38** | **152** | **18** | **7h 30m** |

  The header text under the clock reads `Team tournament`, with the date, start
  and courts as the other formats do.
- **Playoff chips** (like the other formats' `tagchip`s) list the rounds and their
  matchup counts. No bracket chart in this version.
- Everything recalculates on input, as now, and persists through `save()` under
  the same `ttc-config` key, with the team inputs added to the saved object. A saved
  object without them loads with the defaults.

## 4. Export and import

**CSV** keeps one file format and adds a value to it. For `format` = `team`:

- the `setting` rows are the existing ones (`title`, `date`, `start`, `courts`,
  `duration_min`, `buffer_min`, `format`) plus six more: `teams`, `groups`,
  `matches_per_matchup`, `matches_per_slot`, `advance_teams`, `break_min`;
- there are **no `category` rows**;
- `club_a` and `club_b` are written empty.

The header line is unchanged. `importCSVText` reads these keys when `format` is
`team`, clamps them like the form does, switches the format menu, and shows the
team panel. A file that is not `team` imports exactly as today. A team file
imported by an older copy of the page is not supported and says "no categories
found", which is acceptable.

**`.xlsx`** (`exportXLSX`) gets a team layout: the header lines, then the stage
table of §3 with its totals row, on the same **Breakdown** sheet. The existing
`fileBase` naming is unchanged.

## 5. Alerts

Reuse the three existing finish-time alerts (over midnight, after 10 PM,
comfortable) unchanged, and add warnings, which are notes and never block:

| Condition | Message |
| --- | --- |
| `courts < S` | Needs at least *S* courts: a matchup puts *S* of its matches on court at once. Results show 0 until fixed |
| `T < 2`, or `G > floor(T / 2)` | Each group needs at least 2 teams: using *G′* groups |
| `A > T` | Only *T* teams: the playoff is capped at the largest of 2, 4, 8 that fits |
| `A < G` | Fewer playoff places than groups: not every group can send its winner |
| a group of size 2 | A 2-team group plays one matchup |
| *R* × block > group slots from capacity | The biggest group's rounds, not the courts, set the length of the group stage |
| `lanes` > a playoff round's matchups | Courts idle during the *round name* |
| `A` not 2, 4 or 8 | Not supported: choose 2, 4 or 8 |

Plus one fixed note, always shown: *Who advances is the organiser's choice: the
calculator only counts the places.*

## 6. Generator handoff

`GENERATOR_MASTERS` has `dual` and `standard`. For `team` there is no master yet
([Team Tournament Master](team-tournament-master-spec.md) builds it), so:

- while `GENERATOR_MASTERS.team` is absent, the **Build the event workbook** box is
  hidden for the team format, rather than offering a master that cannot read the
  plan;
- when the Master exists, add its entry (file ID and help text) and the box returns;
  nothing else in the handoff changes, because `buildPlanCsv()` already feeds both
  the file and the clipboard.

Both existing generators already refuse a plan whose `format` is not theirs, with
a message naming the format they found, so a team CSV pasted into the wrong master
fails clearly with no change to either script.

## 7. What does not change

The standard and dual-meet calculations and their cards, `PRESETS`, `buildChart`,
the CSV columns, `GENERATOR_MASTERS`' existing entries, and `sw.js`. The service
worker serves the document network-first and its cached set does not change, so
`CACHE` is not bumped ([PWA spec](../implemented/calculator-pwa-spec.md)).
Nothing in `sage-tools-api` changes.

## 8. Acceptance checklist

Serve the site and open the calculator (a git-ignored `tournament-calculator-beta.html`
copy is enough while building).

- [ ] **Format** lists three options. Standard and Dual meet behave exactly as before
  (compare a saved plan's numbers before and after).
- [ ] Team format shows the team panel and no category cards; switching back restores
  the categories.
- [ ] The three rows of §2's table reproduce: 136 / 34 / 16 slots / 4:05 PM; 152 / 38 /
  18 slots / 4:30 PM; 4:55 PM with the break.
- [ ] Stage table totals equal the stat grid (matches and slots).
- [ ] Each alert of §5 appears for a plan that triggers it and is absent otherwise.
- [ ] Export CSV, then Import CSV (and Reset first): the team plan round-trips with the
  same numbers. Open the file: no `category` rows, the six new `setting` rows.
- [ ] Export `.xlsx` opens and shows the stage table and totals.
- [ ] A saved team plan survives a reload; an old saved plan (no team keys) loads.
- [ ] The handoff box is hidden for the team format and back for the others.
- [ ] No console errors, at desktop and 375 px, and the page still installs and works
  offline (the PWA).

## 9. Decisions for the owner

| # | Question | Leaning |
| --- | --- | --- |
| Q1 | Default for *break before the playoffs*: 0, or 25 minutes as PickleDrive ran? | 0: it is the organiser's choice, and the worked examples show both |
| Q2 | Which playoff sizes: 2, 4, 8 only, or any number? | 2, 4, 8, which covers the quarterfinal-semifinal-final shape the format has used |
| Q3 | Should *S* default to 2, or be derived from the roster (men and women needed per pair)? | 2, as an input: roster rules are the Master's concern |
| Q4 | Include a playoff bracket chart for teams? | Not in this version |
| Q5 | Should the calculator take team names, so the CSV can carry them to the generator? | No: names and rosters are the generator's inputs ([Master](team-tournament-master-spec.md) Q4) |

## 10. Out of scope

- Building the schedule (which matchup is on which court in which slot) and any
  tiebreak or ranking rule: those belong to the workbook.
- Team names and rosters.
- Categories inside a team event.
- A team calculator for formats other than group stage then QF/SF/Bronze/Final.
- Any change to the Team Tournament Master, the event-site template or `sage-tools-api`.

---
**Related:** [Tournament Calculator usage](../../features/tournament-calculator.md) ·
[technical](../../technical/tournament-calculator.md) ·
[Team Tournament Master](team-tournament-master-spec.md) ·
[Team tournament event-site template](team-tournament-template-spec.md)
