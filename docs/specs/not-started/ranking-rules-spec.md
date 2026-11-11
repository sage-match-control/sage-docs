# Spec — ranking rules as a list

> **Status: not started.** Written 2026-11-12 against the site engine `lib/v1/` at
> `sage-match-control.github.io` commit `c4f8ae9` (`domain/standings.js`, `domain/teams.js`,
> `domain/awards.js`, `views/standings.js`, `views/teams.js`, `data/registry.js`) and
> `event-data/config/events.json` as of that date. Nothing here is built. It has **no build
> date**: it waits for an event whose organiser needs rules the presets don't cover (D5). It
> changes the engine, so it is never merged within 3 days of an event. The owner's decisions are
> recorded in §10.

An event's standings are ordered by a list of named rules in `events.json`, such as
`["wins", "h2hWins", "quotient"]`, instead of by one of three rankers written into the engine.
A new organiser's tiebreakers become a line of config, not an engine change. Every event that
exists today ranks exactly as it does now, because each of today's rankers becomes a named
preset that its events get without a `ranking` key.

---

## 1. Why

The engine has three rankers, each written for the event that needed it, and two settings that
pick between them or trim one:

| Ranker | Order | Events |
| --- | --- | --- |
| `rankStandings` (`domain/standings.js`) | wins → head-to-head wins among the pairs level on wins → quotient | every standard and dual-meet event |
| `teamRankBracket` (`domain/teams.js`), `"points"` | points scored → quotient → head-to-head points → pair wins | PickleDrive |
| `teamRankBracketByGames` (`domain/teams.js`), `"games"` | matches won → for two teams: head-to-head wins, head-to-head quotient, tiebreaker games; for three or more: quotient, tiebreaker games, two still level going back to the two-team rules | Millennium Invitational; Ladies' Dink with `headToHeadQuotient: false` |

Each is a list of tiebreakers with the list written as code. `headToHeadQuotient: false` already
removes one item from a list by flag. The next organiser who orders the same tiebreakers
differently ("quotient before head-to-head", "points scored before wins") would need a fourth
ranker or a second flag, an engine change in `lib/v1/` and a test change, in an event's build
week. With a list, it is a line in `events.json`.

## 2. Where things stand

### 2.1 Where ranking is called

| Caller | Ranker | Shown as |
| --- | --- | --- |
| `views/standings.js` (round-robin brackets of a standard or dual-meet event) | `rankStandings(group, model.matches)` | the Hub's and Control Center's Standings tables |
| `domain/awards.js` `standingsTop3ForCategory` | `rankStandings` | the podium of a category with no playoff bracket |
| `views/teams.js` `groupTableHTML` | `teamRankBracket(rows, matches)` (points) | a team event's bracket tables |
| `views/teams.js` `groupTableByGamesHTML` | `teamRankBracket(rows, matches, 'games', { headToHeadQuotient })` | the same, with **Tied · playoff** pills from the result's `ties` |

`domain/bracket.js`'s `rankByWinsThenQuotient` is **not** a ranker in this sense. It reproduces
the workbook's `SORTBYWINS` (wins, then quotient), so that a playoff seat's source tag
("Br 2 · #1") says the place the sheet seated the pair from. It follows the sheet, not the
organiser's tiebreakers, and this spec leaves it alone.

### 2.2 The numbers each ranker reads

The rankers differ in their tiebreakers and also in where their numbers come from:

- **Standard and dual meet:** `wins` and `quotient` from the `STANDINGSCSV` row (the sheet's
  numbers); head-to-head from the day's matches, a best-of-3 round-robin meeting counting once,
  for the pair that took it (`headToHeadWins`).
- **Team, `"points"`:** `pf` and `quotient` from the `STANDINGSCSV` row; head-to-head points and
  pair wins from the played group-stage matches, a `_TB` game included.
- **Team, `"games"`:** every number from the played group-stage matches (`teamGameRecords`), a
  `_TB` game left out; the `_TB` games count only as the last tiebreaker.

`teamScoring` decides which numbers a team event uses, and also who wins a matchup
(`teamMatchupResult`) and which columns the table shows. It stays a setting of its own (§3.3).

### 2.3 What decides who advances

The site's ranking is **display only**. The workbook decides who reaches the playoffs: the
standard generator's `SORTBYWINS` and `fixedSeating_`, and a team workbook's playoff slots the
organiser fills by hand. The two can already disagree on a head-to-head tie; `rankStandings`'s
comment says so. A `ranking` list changes what the Hub and Control Center show, not the sheet. An
event whose organiser changes the rules that pick playoff feeders needs the sheet's formulas
changed to match, by hand or by the generator (§11).

### 2.4 How the rankers break ties

The three rankers share a shape that a single sort can't express, and the list has to keep it:

1. **Head-to-head is computed among the teams still level at that step,** not among everyone. A
   comparator chained over the whole table (`a.wins - b.wins || h2h(a) - h2h(b) || …`) would
   count matches against teams that are not tied, and rank three-way ties wrongly.
2. **The games ranker's rules depend on how many are level:** two teams go to head-to-head,
   three or more to the overall quotient.
3. **The games ranker goes back to the start when a tie shrinks:** three teams level, one split
   off by quotient, leaves two that go back to the two-team rules, which recompute head-to-head
   between just those two.
4. **`rankStandings` and the points ranker do not go back:** pairs level on head-to-head wins go
   straight to quotient, without recomputing head-to-head between just themselves.

`teamRankBracketByGames` already implements 1–3 with its `runs` and `byKey` helpers. This spec
generalises those helpers into a module of their own.

## 3. The design

### 3.1 The `ranking` setting

`events.<event-key>.ranking` is optional. It is either a list of rules (the short form) or an
object:

```json
"ranking": ["wins", "h2hWins", "quotient"]
```

```json
"ranking": {
  "restart": true,
  "rules": [
    "wins",
    { "when": "3+", "rules": ["quotient", "tiebreakerWins"] },
    { "when": 2,    "rules": ["h2hWins", "h2hQuotient", "tiebreakerWins"] }
  ]
}
```

- A **rule** is a name from the registry (§3.2), always ranking higher values first.
- A **block** `{ "when": 2 | "3+", "rules": [...] }` applies its rules only to a group of that
  size. Blocks do not nest.
- **`restart`** (default `false`): when a rule splits a group, each smaller group still level
  starts again from the first rule (§3.4). The short form is `restart: false`.

The setting holds names, never code. `events.json` is data that `sage-tools-api` fetches,
caches and serves, and that a person edits by hand in a commit; the functions live in the engine,
where they are tested.

### 3.2 The rules

`lib/v1/domain/ranking.js` holds the registry: each name, the number it ranks by, and whether that
number depends on which teams are in the group.

| Rule | Ranks by | Depends on the group | Available to |
| --- | --- | --- | --- |
| `wins` | matches won | no | every event |
| `pf` | points scored | no | team events |
| `quotient` | points scored ÷ points conceded; a missing quotient lowest, two missing level | no | every event |
| `h2hWins` | matches won in played matches between two members of the group (a best-of-3 meeting counting once) | **yes** | every event |
| `h2hPoints` | points scored in played matches between two members of the group | **yes** | team events |
| `h2hQuotient` | points scored ÷ conceded in played matches between two members of the group | **yes** | team events |
| `tiebreakerWins` | `_TB` games won against other members of the group | **yes** | team events |

`pf` and `h2hPoints` could serve standard events too, but `STANDINGSCSV` carries no points for a
pair, so they are team-only until a standard event needs them. A rule that is not available to an
event's type is a configuration error (§3.6).

### 3.3 Where the numbers come from

Each rule reads its numbers through the event's **numbers source**, chosen by the event's type
and `teamScoring`, exactly as today (§2.2):

| Source | Event | `wins`, `pf`, `quotient` | Head-to-head rules |
| --- | --- | --- | --- |
| `pairs` | standard, dual meet | the `STANDINGSCSV` row | the day's played matches between the rows' codes; best-of-3 meetings once (`headToHeadWins`'s rule) |
| `team-points` | team, `teamScoring: "points"` | the `STANDINGSCSV` row; `wins` is the pair-wins count of every played group-stage match, `_TB` included | played group-stage matches, `_TB` included |
| `team-games` | team, `teamScoring: "games"` | `teamGameRecords`: played group-stage matches, `_TB` left out | the same matches; `tiebreakerWins` reads only the `_TB` games |

So a rule name means the same thing at every event ("head-to-head wins") while the engine keeps
reading the numbers each event's tables already show. A source is never named in `events.json`.

### 3.4 How a list is applied

```
rank(group, list):
  if group has one row: done
  for each rule in list (a block's rules in place, if its "when" fits the group's size):
    compute the rule's number for each row in the group
      (a rule that depends on the group computes it among exactly these rows)
    sort the group by that number, highest first, keeping the existing order among equals
    split the group into runs of equal numbers
    if there is more than one run:
      for each run:
        if restart: rank(run, the whole list)
        else:       rank(run, the rest of the list after this rule)
      return the runs in order
  no rule split the group: it is unresolved; it keeps its STANDINGSCSV order
                           and is recorded as a tie (§3.5)
```

The size check happens each time a rule is reached, so a group that shrinks from three to two
partway through the list passes over `"3+"` rules from then on, without needing a restart.
Restarting is what makes a smaller group recompute the rules that depend on the group from the
top, and it is the only way to express "two still level go back to the two-team rules".

Under `restart`, rules that don't depend on the group never split a group the second time round,
since it is already level on them. Only rules that depend on the group can split it again.

Numbers are compared with `===` after the same coercions as today (a missing quotient is
`-Infinity`). Equal ratios divide to the same double, so `22/11` and `44/22` are level.

### 3.5 What the ranking returns

`rankRows(rows, matches, event)` returns a new array in ranked order (the input is not changed),
carrying `ties`: `[{ codes, playoff }]`, one entry per unresolved group, `playoff` `'game'` for
two rows and `'round-robin'` for three or more. As today:

- `ties` is `[]` until every group-stage match between the rows is played (a `_TB` game not
  counting as one), so a level pair mid-bracket is never shown as **Tied**.
- The `pairs` source always returns `ties: []`. Standard and dual-meet tables show no tie pills
  today, and this spec doesn't add them (D3).
- The `team-points` source computes `ties`, and its view keeps ignoring them, as it does now
  (D3).

### 3.6 Presets, and reading the setting

Each of today's rankers is a named preset in `ranking.js`:

| Preset | Equals | List |
| --- | --- | --- |
| `pairs` | `rankStandings` | `["wins", "h2hWins", "quotient"]` |
| `team-points` | `teamRankBracket(…, 'points')` | `["pf", "quotient", "h2hPoints", "wins"]` |
| `team-games` | `teamRankBracket(…, 'games')` | restart; `"wins"`, then `"3+"`: `quotient`, `tiebreakerWins`; then `2`: `h2hWins`, `h2hQuotient`, `tiebreakerWins` |
| `team-games-no-h2h-quotient` | `teamRankBracket(…, 'games', { headToHeadQuotient: false })` | `team-games` without `h2hQuotient` |

`data/registry.js`'s `eventConfigFrom` gains a `ranking` field, read by a new `rankingFrom(raw,
eventKey)`:

1. With a `ranking` key, it is parsed and checked: every name in the registry and available to
   the event's type (§3.2), every `when` either `2` or `"3+"`, no nested blocks, no empty list.
   `ranking` may also be the name of a preset (`"ranking": "team-games"`).
2. Any error makes the whole setting fall back to the preset below, with a `console.warn` naming
   the event and the first problem, the way `teamScoringFrom` warns now. Control Center also shows
   it on screen, in the day prompt where it reports a missing `type`, as a warning that doesn't
   block the event: an operator never opens the browser console. A half-applied list would rank
   silently wrong, while the preset is at least the rule the type had before.
3. Without a `ranking` key, the event gets its preset: `pairs` for standard and dual meet;
   `team-points`, `team-games` or `team-games-no-h2h-quotient` from `teamScoring` and
   `headToHeadQuotient` for a team event. **The two existing settings keep working forever.**
   Millennium and Ladies' Dink are finished and never get a `ranking` key.
4. With both `ranking` and `headToHeadQuotient`, `ranking` wins and the registry warns that
   `headToHeadQuotient` is ignored.

### 3.7 The module

`lib/v1/domain/ranking.js` is a new file of pure functions with no DOM access. It imports only
`domain/` modules (`standings.js`'s `headToHeadWins`, `series.js`, `teams.js`'s `sideOf`,
`stageOf` and `teamGameRecords`, `deciders.js`'s `isDecider`):

| Export | Does |
| --- | --- |
| `RANKING_RULES` | the registry: name → `{ label, appliesTo, groupDependent }`, labels for any text that describes a list |
| `RANKING_PRESETS` | the four presets of §3.6 |
| `rankingFrom(raw, event)` | §3.6's parse, check and fallback; returns `{ ranking, problem }` |
| `presetFor(event)` | the preset an event gets without a `ranking` key |
| `rankRows(rows, matches, event)` | §3.4 and §3.5 with `event.ranking` (or `presetFor(event)` when it is absent) |

It takes the `EventConfig` that `eventConfigFrom` builds, not a raw registry entry.

### 3.8 The callers

`lib/v1/` only allows compatible changes, and a browser may hold a module for 10 minutes, so:

- `rankStandings(rows, matches)` and `teamRankBracket(rows, matches, scoring, options)` keep
  their exports and signatures. Their bodies become `rankRows` with the matching preset, so a
  shell or archived page that still imports them ranks as before.
- `views/standings.js`, `views/teams.js` and `domain/awards.js` call `rankRows(rows,
  model.matches, model.event)` from the new `ranking.js`. A new file can't be held stale by a
  browser, so this can ship in the same push as the module. `rankRows` falls back to
  `presetFor(event)` when `event.ranking` is missing, in case a cached `registry.js` built the
  config.
- `_tests/helpers/fixtures.mjs`, which mirrors `eventConfigFrom` for the harness, gains the same
  `ranking` field.

`events/archives/pickledrive-anniversary-2026/` has its own inline copy of the points ranker. It
is frozen with that event and is not touched.

## 4. What does not change

- The workbook, its `SORTBYWINS`, the generator's seating, and who advances (§2.3).
- `rankByWinsThenQuotient` and the playoff seats' source tags (§2.1).
- `teamScoring`: who wins a matchup, the table's columns, the numbers source.
- Any visible text or layout, apart from the one corner case in D1 and a new Control Center
  warning when a `ranking` setting is invalid. No new tie pills and no line describing the rules
  (D3, D4).
- `sage-tools-api`. It never reads ranking settings, and `facilityCompletion` doesn't rank.

## 5. Worked examples

**A three-way cycle at a standard event** (preset `pairs`, no restart). A, B and C are 2–1. A beat
B, B beat C, C beat A. `wins` leaves all three level; `h2hWins` among them is 1, 1, 1, so they
stay level; `quotient` orders them. As today.

**Four level at a standard event** (preset `pairs`). A, B, C and D are 2–2. Among the four,
`h2hWins` is A 2, B 1, C 1, D 0 (B beat C). Without restart, B and C go to `quotient` and C may
finish above B despite losing to B. That is today's `rankStandings`. An organiser who wants their
direct result to count writes `{ "restart": true, "rules": ["wins", "h2hWins", "quotient"] }`.
B and C then restart, `h2hWins` between just the two of them is B 1, C 0, and B goes above C.

**Three level at Millennium** (preset `team-games`). X, Y and Z have 2 wins each. `wins` doesn't
split them; the `"3+"` block's `quotient` splits off X; Y and Z restart: `wins` and the `"3+"`
block pass over them, and the `2` block ranks them by `h2hWins` between just the two. As today.

**A new event** whose organiser ranks by wins, then quotient, then head-to-head:

```json
"ranking": ["wins", "quotient", "h2hWins"]
```

A commit to `event-data`, no engine change. The Hub and Control Center pick it up when they next
read `events.json`.

## 6. Parity

The presets must rank exactly as the old functions do. To prove it:

1. Before changing the old functions, copy their bodies as they are into
   `_tests/helpers/legacy-ranking.mjs`, test-only, so the comparison outlives the delegation.
2. `_tests/unit/domain/ranking-parity.test.mjs` runs both on:
   - every fixture's standings, bracket by bracket, under every preset its type allows;
   - generated brackets: a seeded generator of 2–6 teams with random results, level wins, missing
     quotients, best-of-3 meetings, `_TB` games, unplayed matches. A few thousand cases per preset,
     the seed printed on failure.
   It asserts the same order and, for the team sources, the same `ties`.
3. The comparison harness, baseline against branch, shows no difference on any page.

The one known difference is D1, decided in favour of the recount. Its case is pinned by its own
test, and the generator leaves it out of the parity comparison.

## 7. Tests

In `_tests/unit/domain/ranking.test.mjs`:

- each rule's number on a small hand-built bracket, for each source it is available to;
- §3.4's walk: a split continues with the rest of the list; a restart goes back to the top; a
  `when` block is skipped for the wrong size and applies once a group shrinks to the right size; an
  unresolved group keeps its input order and is recorded in `ties` only once the round robin is
  over;
- §5's four examples, as written.

In `_tests/unit/data/data-modules.test.mjs`, beside the `teamScoring` tests: `rankingFrom` reads
the short form, the object form and a preset name; rejects an unknown name, a rule not available
to the type, a bad `when`, a nested block and an empty list, each falling back to the preset with
one warning naming the event; and `ranking` overrides `headToHeadQuotient` with a warning.

The existing `rankStandings` and `teamRankBracket` tests stay as they are and keep passing. They
now exercise the presets.

## 8. Build order

**Phase 1: the engine, with no visible change.** `legacy-ranking.mjs` and the parity test first,
passing against the old functions. Then `ranking.js` with the registry, the walk and the presets;
the old functions delegate; the callers move to `rankRows`. `npm run verify` in `_tests/`, with
the harness showing no difference. No `events.json` change. One push.

**Phase 2: the setting.** `rankingFrom` in `registry.js` and `fixtures.mjs`, the Control Center
warning, their tests, and the documentation (§9). A later push than Phase 1, so that
Phase 1 can be reverted on its own if the harness or an event day turns something up.

An event's first use of `ranking` is part of that event's own spec and its own `event-data` commit,
never this one.

## 9. Documentation

When built, in the present tense:

- `sage-docs/docs/technical/event-data-config.md` and `event-data/config/README.md`: the `ranking`
  key, its forms, the rule table, the presets, and that `teamScoring` and `headToHeadQuotient`
  pick the preset without it.
- `sage-docs/docs/technical/site-engine.md`: `domain/ranking.js` in the module list, the walk,
  and that `rankStandings` and `teamRankBracket` are presets.
- `sage-docs/docs/features/tournament-hub.md` and `features/control-center.md`: each says the
  standings follow "the event's ranking rules", giving the default order per type, rather than
  each spelling out a fixed list.
- `sage-docs/docs/specs/event-spec-template.md`: the **[Team]** line that asks for
  `teamScoring` and `headToHeadQuotient`, and the gather-first list for every type, ask for the
  organiser's tiebreakers as a `ranking` list. Every event type can now differ.
- The workspace `CLAUDE.md`'s team-event paragraph: ranking follows `ranking`, or the preset from
  `teamScoring`.

## 10. Decisions

Decided by the owner on 2026-11-12.

| # | Question | Decided |
| --- | --- | --- |
| D1 | **The `team-games` corner case.** Four or more teams level on wins and quotient, split by `tiebreakerWins` into a group of three or more still level. The old ranker flags that group **Tied · playoff** at once. Under restart, `tiebreakerWins` is recomputed among just those three, which can split them. | **Recount.** The restart behaviour stands, with no `"stop"` rule. It is what "still level go back to the rules" means, and it needs a round of `_TB` games among four teams first. A test pins it (§6). |
| D2 | **The setting's name.** | **`ranking`.** The first rule (`wins`, `pf`) isn't a tiebreaker. |
| D3 | **Tie pills for standard events and `"points"` team events.** The engine can report unresolved ties for every source. | **Not in this spec.** It is a visible change and belongs with [Hub player experience](hub-player-experience-spec.md)'s P13 (the colours and the ties explained). |
| D4 | **Describe the rules on the page.** `RANKING_RULES`'s labels could print "Ranked by wins, then head-to-head, then quotient" under each Standings table, from the event's own list. | **Not in this spec**, for the same reason as D3. The labels are built in Phase 1, so it is cheap later. |
| D5 | **When to build.** | **No date.** The spec waits until an organiser needs rules the presets don't cover, and is built in good time before that event, never within 3 days of one. Phase 1 changes the ranking every standard event uses, so it goes in at least one event before the one that needs it, or is dry-run on that event's fixtures. |

## 11. Out of scope

- **The sheet's side.** The generators could write the playoff feeders' sort from the same list,
  so the sheet and the site always agree. That would mean the list reaching Apps Script and the
  sheet formula for each rule; it is its own spec if an organiser's rules ever change who
  advances.
- **Per-category rules.** One list per event. An event whose categories rank differently would
  need `ranking` keyed by category; nothing has needed it.
- **New rules** beyond §3.2, such as points conceded, fewest losses or a coin toss. Each one is a
  registry entry and a test, added by the event that needs it.
