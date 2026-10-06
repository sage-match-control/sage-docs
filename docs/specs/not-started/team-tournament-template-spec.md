# Spec — Team tournament event-site template

> **Status: not started.** Nothing here is built. Written 2026-10-05, after the
> first team event ran, against `sage-match-control.github.io`'s
> `events/pickledrive-anniversary-2026/` (the prototype), `tools/control-center.html`
> and `_templates/` as of that date. **A proposal, not a settled brief:** §7 lists
> the decisions the owner still has to make, and §6 what blocks it.
>
> Split out of [Team tournament](../implemented/pickledrive-club-anniversary-team-tournament-spec.md)
> §15, which asked for this "after the event". Its sibling,
> [Team Tournament Master](team-tournament-master-spec.md), is the workbook
> generator and is a separate, larger piece of work.
>
> **Revise before building: the [site engine](../implemented/site-engine-spec.md) changes how
> this template is made.**
>
> - Once the engine is in place, a team event's Hub is a shell on it, with a
>   team branch in `apps/hub.js` built from the `views/teams.js` modules
>   Control Center already uses.
> - It does not copy PickleDrive's inline code or the shared blocks, so the
>   "carry the blocks byte-identical" steps below no longer apply.
>
> Revise this spec after the engine's Phase 5.

Make a third event-site template, `_templates/team-tournament-template/`, so a
team tournament is a copy-and-fill job like the two existing templates
([Event site templates](../implemented/event-templates-spec.md)) instead of a
copy-the-last-event-and-hunt-for-hardcoded-strings job.

**The deliverable is the template, not an event.** No real event is created by
this work; the runbook step in `_templates/CLAUDE.md` is what a later session
follows to stand one up.

---

## 1. What exists

The first team event, PickleDrive Club One Year Celebration, was built by hand
in `events/pickledrive-anniversary-2026/` from the **standard** template, then
rewritten for the team format. It is the prototype:

| Piece | Where | Reusable as it is? |
| --- | --- | --- |
| Public page | `events/pickledrive-anniversary-2026/index.html` | The team rules and views are generic; the event values, pubmat theme and several constants are the event's |
| Schedule board | `…/schedule.html` | Same |
| Attendance desk page | `…/attendance.html` | Already a template: `_templates/attendance/` |
| Scorer page | none yet | Already a template: `_templates/scorer/` |
| Operator runbook | `…/dry-run-checklist.md` | A copy of `_templates/dry-run-checklist-template.md` with team-event edits |
| Control Center's `team` type | `tools/control-center.html` | Already generic: it serves every `"type": "team"` event, so it needs no template |
| Registry entry | `event-data/config/events.json` | One entry per event, by hand; see `_templates/CLAUDE.md` §2 step 7 |

So the template covers only `index.html` and `schedule.html`. The team rules
(who wins a matchup, team names, bracket ranking, who advances, the pair labels,
the team rosters) exist in the prototype's two pages and in Control Center's
`team type` block, and are kept in step by hand
([`CLAUDE.md`](https://github.com/sage-match-control)'s "Things that must be
kept in sync by hand"). The template adds a third copy of the pages' half of
them.

## 2. What the prototype hard-codes

Everything in this list has to become a `{{TOKEN}}`, a marked
`// EXAMPLE — replace` value, or something derived from the data:

| Hard-coded | Today | Template |
| --- | --- | --- |
| Event values | name, date, venue, logo, tagline, short link, QR | `{{TOKENS}}`, the same set the standard template uses |
| `PAIRS` | four pairs: MD, WD, XD, XD, numbered `XD 1`/`XD 2` by `numberRepeatedPairs` | `// EXAMPLE — replace`. Any number of pairs per matchup; the function that numbers repeats stays |
| `STAGES` | `QF`, `SF`, `Br`, `Fi` | `// EXAMPLE — replace`; a stage in the data but not in `STAGES` must not break a page |
| `FACILITIES` | one facility, 10 courts | `// EXAMPLE — replace` |
| Groups | three brackets of five, teams `A`–`O` | Derived from the `bracket` column and the team rows of `STANDINGSCSV`: any number of brackets of any size |
| Who advances | eight quarterfinalists, marked **Advances** when a letter fills a playoff slot | Derived from the slots in the matches, so a four-team playoff or an eight-team one both work, with no constant |
| Theme and fonts | the pubmat's palette, `Archivo Black`, hard-coded copies of the old colours in canvas code and meta tags | **The S.A.G.E. house theme**, exactly as the standard and dual-meet templates ship it (§3): the `THEME` block, navy and green on off-white paper, Archivo Black, Barlow Condensed and Inter. The pubmat palette does not go in the template |
| Pair abbreviation | `MXD` on PickleDrive's pages, `XD` in Control Center | `XD` by default; one constant, since it is the organiser's choice |
| Hero copy | rewritten by hand (§5.3 of the prototype spec) | Neutral wording driven by `{{EVENT_HEADLINE}}` and the tagline |
| Fixture override | `FIXTURE` and `snapshotUrlFor` added to the page | In the template from the start. Check both existing templates for it and add it where it is missing, so all three read `/_fixtures/` the same way |

## 3. Design

```
_templates/team-tournament-template/
  index.html
  schedule.html
  assets/            (placeholder logo and QR, as the other templates have them)
```

- Start from the prototype's two pages, not from the standard template: the
  prototype already has the team views (matchup cards, bracket standings,
  playoffs, Teams tab, Match Finder by team or player, the schedule board's
  team chips).
- Carry the shared blocks byte-identical: `LIVE CHANNEL` (`LIVE_BASE_URL` a
  literal, not a token). The `ATTENDANCE CLIENT` and `SCORE CLIENT` blocks live in
  their own page templates and are not in these two pages.
- Keep the template's example *event* values equal to PickleDrive's, so instantiating it
  with those values must reproduce the prototype's behaviour (§5) apart from the theme.
- `{{TOKEN}}`s: the standard template's set (`EVENT_KEY`, `EVENT_TITLE`,
  `EVENT_TAGLINE`, `EVENT_HEADLINE`, `EVENT_DATE_RANGE`, `VENUE`, `EVENT_LOGO`,
  `SCHEDULE_DAY_KEY`, `QR_IMAGE`, `QR_URL`), and no more unless a page needs one.
- **S.A.G.E. themed, like the other two templates.** `index.html` and `schedule.html` open
  with the same `:root` `THEME` banner and tokens as the standard and dual-meet
  templates (`--navy`, `--navy-deep`, `--green`, `--green-dark`, `--paper`, `--paper-dim`,
  `--line`, `--ink`, `--ink-soft`, `--white` and the role aliases beneath them), the same
  contrast rules (green is a fill, never small text) and the same fonts, so a new team
  event matches the tools, the schedule board and the rest of the platform until someone
  re-skins it. Everything in the pages resolves through those tokens; the one place that
  does not (the schedule board's stage colour map, like the standard template's `CAT_META`)
  is called out in `_templates/CLAUDE.md` step 5 with the others. The prototype's
  pubmat theme is the worked example of a re-skin, not the default.
- A `type: "team"` event takes team names from `STANDINGSCSV` and needs no
  `display` block, so the template carries no `DIVISIONS`/`EVENTS` config.

## 4. Steps

1. Create `_templates/team-tournament-template/` from the prototype's
   `index.html` and `schedule.html`.
2. Replace event values with the tokens in §3 and mark `PAIRS`, `STAGES`,
   `FACILITIES` and the theme block `// EXAMPLE — replace`.
3. Remove the constants of §2 that the data can supply: the group count and size,
   the advancing count.
4. Make `PAIRS`/`STAGES` tolerant: an unknown stage or pair renders its raw
   code and a visible warning, as Control Center does for an unknown category.
5. Add the fixture override to the two existing templates if either lacks it.
6. Document it in `_templates/CLAUDE.md`: §1 (choosing a template), §3 (tokens),
   §5 (team code format), §5.1 (the sync table gains the template's copy of the
   team rules and pair labels), and the instantiation steps. Update the repo's
   `CLAUDE.md` site section to match.
7. Update the sage-docs pages that point at "no template yet":
   `technical/adding-a-new-event.md` and the specs index.

## 5. Verification

- The pages carry the same `THEME` banner and `:root` tokens as the standard template
  (compare the blocks), and no pubmat colour or font survives: a grep for the pubmat's
  hex values finds nothing.
- Instantiate the template into a scratch `events/` folder (a git-ignored
  `*beta*` copy is enough) with PickleDrive's values and diff the result against
  the prototype's pages: only the intentional differences remain (the theme, the tokens, the tolerant `PAIRS`/`STAGES`).
- Load it against `_fixtures/pickledrive-anniversary-2026/` `?fixture=pre`,
  `finished`, `edge` and `qf-pre`: every view reads as the prototype's does.
- Instantiate it again with different values (different pair count, two groups
  of four, a four-team playoff) against hand-built fixtures: standings, advancement
  and the schedule board follow the data.
- `grep -rn '{{' ` on the instantiated folder prints nothing; the `LIVE CHANNEL`
  block `diff`s clean against Control Center's.
- No console errors, at desktop and 375 px.

## 6. Blocked on

- Nothing hard. The template reads `CSV` and `STANDINGSCSV` in the format the
  prototype's workbook publishes ([Team tournament](../implemented/pickledrive-club-anniversary-team-tournament-spec.md)
  §3), so it can be extracted without the generator.
- **Better after** [Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md):
  a template whose first event syncs in 10–58 s per edit is a poor start, and
  that work decides whether the format's published tabs change at all.

## 7. Decisions for the owner

The theme is settled: the S.A.G.E. house theme, as in §3.

| # | Question | Leaning |
| --- | --- | --- |
| Q2 | `MXD` or `XD` as the default abbreviation? | `XD`, with one constant to change |
| Q3 | Is "any number of groups" required, or is three groups of five the format? | Derive it from the data: the code is already group-agnostic where it matters |
| Q4 | Does the template ship `attendance.html` and `scorer.html`, or leave them to their own templates? | Leave them: they are already templates with their own runbook steps |
| Q5 | Should it ship a `dry-run-checklist` variant for team events? | Yes: the prototype's copy already has the team edits |

## 8. Out of scope

- The workbook, and anything that builds it: [Team Tournament Master](team-tournament-master-spec.md).
- Control Center. Its `team` type is already shared by every team event.
- Team logos, scoresheets for team codes, and any change to `sage-tools-api`.
