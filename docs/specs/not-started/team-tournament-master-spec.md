# Spec — Team Tournament Master (the team workbook generator)

> **Status: not started.** Nothing here is built. Written 2026-10-05, after the
> first team event ran, from what is known of the PickleDrive workbook
> (**2026-10-03 PickleDrive Club One Year Celebration**) and of the two existing
> generators. **A proposal, not a settled brief:** §8 lists the questions the
> owner has to answer first, and the work is large enough that it should be
> phased (§6) the way the dual-meet generator was.
>
> Split out of [Team tournament](../implemented/pickledrive-club-anniversary-team-tournament-spec.md)
> §15 item 5 ("consider a *Team Tournament Master* sheet generator. This event's
> workbook was built by hand."). Its sibling,
> [Team tournament event-site template](team-tournament-template-spec.md), is the
> website half and is separate.

Build the third generator: a **SAGE Team Tournament Master** workbook whose
`SAGE -> Generate event tabs` menu builds a team event's whole scoring workbook,
as [`sheet-generator.gs`](../implemented/dual-meet-sheet-generator-spec.md) does for
a dual meet and [`standard-generator.gs`](../implemented/standard-tournament-master-spec.md)
does for a standard tournament. Today the team workbook is built by hand and carries a formula load that made
Sheets API reads take up to 58 s on event day.

---

## 1. Why a generator, and why now

- The format has been run once. Its workbook is the only specification of itself.
- A hand-built workbook cannot carry the fixes in
  [Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md) (the hidden
  `StackCache` tab, the matchup family reading `MatchLookup`, the 15 unused named functions removed)
  unless every future workbook repeats them by hand. A generator builds them in once.

## 2. What the generator must produce

From the PickleDrive workbook, a team workbook has these parts (the names are the
ones `sage-tools-api` and the site depend on):

| Tab | Role | Read by |
| --- | --- | --- |
| `CSV` | one row per match: `matchNumber`, `matchUp`, `court`, `Schedule`, `CourtAssignment`, `teamCode1`, `team1Player1`, `team1Player2`, `team1Score`, `teamCode2`, … | the sync (`matchesSheetName`) |
| `STANDINGSCSV` | one row per team and per playoff slot: `teamCode`, `teamName`, `totalPoints`, `totalOpponentPoints`, `quotient`, `bracket` | the sync (`standingsSheetName`) |
| `Teams` | the roster: eight players per team, with level and gender | the sync (`rosterSheetName`, default `Teams`) |
| `MatchUps` | the organiser's entry tab: team names, the group draw, and the playoff seeds (the workbook's entry **is** the decision about who advances) | formulas |
| `SCHEDULE` | the court-block grid scorers type scores into: 8-column court blocks, two rows per slot, the same geometry every other workbook has ([Control Center score entry](../in-progress/control-center-score-entry-spec.md) §3) | scorers, score entry, `sheets-sync.gs` |
| `Court Control` | which match is on which court now | operators, the schedule board |
| `Standings`, `MatchLookup` | the workbook's own working tabs; the site ignores `Standings` | formulas |
| `StackCache` (hidden) | `SCHEDULE`'s stacked columns built once | the named functions |
| `ATTENDANCE` | empty, with the header `src/attendance/attendanceTab.mjs` requires (both existing generators add it) | attendance |

plus the workbook-level **named functions** (`STACKBLOCKS`, the `GET…BYMATCH…`
and matchup families) that the formulas call.

## 3. The hard constraint: named functions

Named functions cannot be created, edited or deleted through the Sheets API or
Apps Script. So, as with the dual meet's `Named Function library`
([technical page](../../technical/named-function-library.md)), they live **in the
master workbook** and reach a generated workbook by copying the master, never by
the script writing them. That fixes the shape:

- The master is a normal Google Sheet carrying the optimised named functions, a
  pristine prototype of each rebuilt-in-place tab, `sheets-sync.gs` and
  `team-generator.gs`, and no event data.
- An event is made by copying the master (the calculator-style **/copy** handoff the
  other two masters use), running the generator, which fills the tabs.
- The master keeps the `SAGE … Master` name so `sheets-sync.gs`'s **Replace shared
  secret** offer works (`secretMenuItem_`).
- Changing a named function is a person in the Sheets UI, followed by a new master:
  document that in the generator's technical page, as the dual-meet one does.

## 4. Inputs

What the generator needs that a person decides. The standard and dual-meet generators
take a Tournament Calculator CSV; the calculator has no team format today, so either
it grows one ([Calculator team format](calculator-team-format-spec.md); §8, Q1) or the sidebar takes everything.

| Input | Notes |
| --- | --- |
| Event title, date, venue/facility | as the other generators; the workbook is renamed `<date> <title> - <FACILITY>` on success |
| Teams | a name each, and a group (bracket) |
| Rosters | eight players per team: name, level, gender. Pasted from a sheet or CSV |
| Pairs per matchup | the pair table (today: MD, WD, XD, XD) |
| Courts and slot length | today 10 courts, 25-minute slots; pairs 1–2 of a matchup in one slot and 3–4 in the next |
| Group stage | round robin within each group: the matchup list and its order |
| Playoffs | stages (QF, SF, Bronze, Final), how many advance, and that the organiser types the qualifiers into `MatchUps` |
| Sync | the facility name and day key, set through **SAGE → Set up live sync** as in every workbook |

## 5. Design principles (taken from the two existing generators)

- **Rebuild in place, refuse a rerun.** Like the dual-meet generator, a run that dies
  halfway cannot be retried, so a refusing-to-run check on the master's pristine
  prototype shapes comes first, and a verify script exercises the whole run.
- **A verify script against a mocked Sheets API** (`scripts/verify-team-generator.mjs`,
  on the same `mock-apps-script.mjs`), checked against the PickleDrive workbook's
  published numbers: 152 matches, 38 matchups, QF/SF/Bronze/Final numbering.
  `verify-attendance.mjs` also gets a check that no top-level name collides with the
  other `.gs` files, which share one script project per workbook.
- **Pinned tab GIDs.** `SCHEDULE` and `Court Control` GIDs are pinned in
  `sheets-sync.gs`; the master must keep them.
- **The template's data contract.** The generated `CSV` and `STANDINGSCSV` columns are
  exactly what [the site template](team-tournament-template-spec.md) and Control
  Center's `team` type read; the generator ships with a fixture snapshot of its own
  output so both can be tested without Google.
- **Written as phases**, like the dual-meet generator's three.

## 6. Phases (proposed)

| Phase | Builds | Done when |
| --- | --- | --- |
| 1 | The master workbook (optimised named functions, `StackCache`, prototype tabs) built by hand from the PickleDrive workbook, with `team-workbook-stack-cache-spec` applied. No script yet | A copy of the master, filled by hand, syncs in under 6 s |
| 2 | `team-generator.gs`: `Teams`, `MatchUps`, `Standings`, `MatchLookup`, `STANDINGSCSV` and `ATTENDANCE` from the sidebar inputs | The verify script reproduces PickleDrive's team rows and standings columns |
| 3 | The schedule: `SCHEDULE`, `Court Control`, `CSV`, with playoffs | The verify script reproduces PickleDrive's 152 matches, slots and playoff seeds |
| 4 | The calculator's team format and **/copy** handoff, if Q1 is yes | A plan from the calculator opens the master ready to run |

## 7. Documentation and runbook

Feature and technical pages like the other generators have
(`features/team-tournament-generator.md`, `technical/team-tournament-generator.md`),
the specs index, `sage-tools-api`'s `README.md` and `CLAUDE.md` script notes, and
`_templates/CLAUDE.md` §2 step 8 (the workbook step) for team events. A change to the
generator is not a deploy and does not bump `package.json`.

## 8. Decisions for the owner

| # | Question | Leaning |
| --- | --- | --- |
| Q1 | Does the Tournament Calculator get a team format, or does the sidebar take every input? | Sidebar only at first (phases 1–3); the calculator after |
| Q2 | Is the format fixed (4 pairs, 8 players, round-robin groups, then playoffs) or configurable? | Configurable pairs and group sizes; fixed playoff shape QF/SF/Bronze/Final |
| Q3 | Who draws the groups and seeds the playoffs: the generator, or the organiser (as today)? | The organiser, as today; the site never computes who advances |
| Q4 | Do rosters come in through the sidebar or are they typed into `Teams` afterwards? | Typed or pasted into `Teams`, since captains change them through the day |
| Q5 | Must it work before [Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md) is applied? | No longer open: it is applied to the PickleDrive workbook, the generator's starting point |

## 9. Blocked on

- Nothing hard. [Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md) is applied to the PickleDrive
  workbook, which is the generator's starting point; the master is built from it by hand in the Sheets UI.
- Nothing from the website: the [template](team-tournament-template-spec.md) reads
  the published tabs and can be built first or in parallel.

## 10. Out of scope

- The site template, Control Center and `sage-tools-api` (the sync, score entry and
  attendance already work on a team workbook).
- Bracket drawing and the Bracket Generator's name import, which are per-category.
- Scoresheets for team codes.
