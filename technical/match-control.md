# Match Control console

One operator console (`tools/match-control.html`) covering every registered
event, replacing what used to be a separate `match-control.html` copy
per event — four copies at last count, ~13,700 lines total, each carrying
its own ~106-line hand-duplicated configuration block. The console carries
**no per-event configuration of its own**: it picks an event from
`event-data/config/events.json` (see [event registry
schema](event-data-config.md)), reads that event's registry entry, and
renders.

## Config resolution

Selecting an event resolves three things fresh, every time:

- `CURRENT_EVENT_KEY` / `EVENTS_REGISTRY` entry
- `CURRENT_TYPE` — `"dual-meet"` or `"standard"`, **required and explicit**,
  never inferred from the data. Guessing from team-code shape works most of
  the time and fails silently; an unrecognized/missing `type` shows a
  visible configuration error instead.
- `DISPLAY` — the optional `{ divisions, events, clubs }` label maps. Absent
  is fine; the console just shows raw codes (`LIWD`, `PNF`) instead of
  friendly labels.

Everything else the console needs maps onto that: the category set is the
second code segment (`DIVISIONS` × `EVENTS`), stage (RR/QF/SF/Bronze/Final)
comes from generic regexes in `STAGE_META` — not per-event — and the day
label comes straight from the published snapshot's own `label` field.

**Mission Control is event-agnostic.** Its dependency surface is just
`EVENT_KEY` — it touches none of `DIVISIONS`, `EVENTS`, `CLUBS`,
`STAGE_META`, `parseCode`, or `divisionLabel`. That's what makes it safe to
share unmodified across every event type.

## The tabs

**Live Matches, Match Finder, and Standings** behave as the old per-event
pages did, with three differences: club logos were dropped entirely (no
`.score-logo`/`.live-team-logo`/`.cs-logo` treatment); the console **ignores
`isLive` and always shows live data** (`computeDayIsLive()` is hardcoded
`true`), so an operator can preview a day before it's public; and labels
degrade to raw codes when `display` config is absent rather than erroring.

**Standings layout is chosen by `type`.** `dual-meet` gets per-club columns
inside each category with a shared club win-total summary bar and the
cross-club Bronze/Final shown as one block spanning both clubs (see
`CROSS_CLUB_STAGE_KEYS`); `standard` gets flat category cards with a
desktop toggle bar and a mobile category-search filter.

**Awards** is a fifth tab, added later — its own architecture page:
[Awards tab](awards-tab.md).

**Mission Control** — see [Mission Control usage](../features/match-control.md#mission-control)
for what it does; behaviorally unchanged from the old per-event pages, just
scoped to whichever event is selected. The go-live states are `auto`
(live 4 hours before the day's earliest scheduled match time, computed live
from the synced `Schedule` column — no hour to configure), `true` (force
live), and `false` (Live/Standings hidden, scores suppressed on the
*public* site only — this console's own tabs stay live regardless, so an
operator can verify a fix before un-hiding). The connection-check line also
surfaces the currently-cached sync config's short SHA (see [sync
pipeline](sync-pipeline.md) § Observability), so an operator can see at a
glance whether a just-committed `events.json` change has actually taken
effect yet.

## Theme

The console is a **tool** — it lives in `tools/` beside
`scoresheet-generator.html` and `tournament-calculator.html`, which are
where the S.A.G.E. palette (navy structure, green accent, off-white paper;
Archivo Black / Barlow Condensed / Inter) originated, and it uses that
palette directly rather than porting a converted theme from the old
per-event pages. Two consequences:

- **It does not take on per-event theming.** An event may re-skin its own
  public `index.html`, but the console looks identical regardless of which
  event is selected — it's one tool pointed at different data, and an
  operator switching events shouldn't see the furniture move.
- Contrast rules worth knowing if touching the CSS: **green is a fill
  colour, not a text colour** (`--green` on white is ~2.3:1, fails AA at any
  size — use it as a background with navy text on top, or on a navy panel).
  The masthead is a navy bezel on paper; controls sitting on it need their
  own light-on-navy overrides rather than reusing the paper-ground base
  rules.

---
**Features:** [Match Control](../features/match-control.md) · [Awards tab](../features/match-control.md#awards)
**Spec:** `specs/match-control-console-spec.md` in the site repo (full build history, acceptance checklist).
