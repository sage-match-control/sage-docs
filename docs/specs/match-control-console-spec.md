# Spec — Match Control console (central operator page)

Replace the per-event `match-control.html` with **one** operator console at
`tools/match-control.html`, driven entirely by `event-data/config/events.json`.
It keeps all four original tabs (Live Matches, Match Finder, Standings, Mission
Control) but carries no per-event configuration of its own: it picks an event,
reads that event's registry entry, and renders. A fifth tab, Awards, was added
later per `awards-podium-tab-spec.md` — a read-only podium/export view
derived the same way, sitting between Standings and Mission Control.

Per-event `match-control.html` then disappears from both templates and from
the instantiation runbook.

**Status: built, not yet deployed.** `tools/match-control.html` exists and
the go-live override is implemented end to end in `sage-tools-api`
(committed and pushed). Still needed: the Cloud Run deploy itself, and
setting `AUTH_PASSWORD_HASH`/`AUTH_TOKEN_SECRET` there.

Operators sign in (§4) rather than pasting a shared secret, and the `auto`
go-live threshold (§2.5, §4.1) is computed from the schedule rather than a
per-event config value — both described in place below, not as a config
field named `LIVE_GO_LIVE_HOUR_PH`.

---

## 1. Why central, not per-event

| | |
|---|---|
| Path | `tools/match-control.html` |
| Public URL | `/tools/match-control` |
| Discoverability | Unlisted — not linked from the tools index (§9.1) |

Three findings made this worth doing, all checked against the shipped code and
the live snapshot rather than assumed:

1. **The organizer panel is already event-agnostic.** Grepping the whole
   Mission Control region for the presentation config returns nothing: it
   touches none of `DIVISIONS`, `EVENTS`, `CLUBS`, `STAGE_META`, `parseCode`
   or `divisionLabel`. Its entire dependency surface is `EVENT_KEY`,
   `DAYS[].{key,label}`, `FACILITIES[].name` and `CLOUD_RUN_BASE_URL`.
2. **The registry it needs already exists, and is publicly readable.**
   `GET https://sage-match-control.github.io/event-data/config/events.json`
   returns `200 application/json` and lists every registered event, its days
   and its facility names. **Read is public; only the actions are
   secret-gated.** So the console needs no new config surface to enumerate
   events — and no secret to *show* anything.
3. **The shared secret is global.** `SYNC_SHARED_SECRET` is a single Cloud Run
   env var (`sage-tools-api/index.mjs`), checked in `src/sync/routes.mjs`.
   Today it is stored per page as `${EVENT_KEY}.orgSecret`, so an operator
   running two events pastes the same secret twice. One console = paste once.

### 1.1 What it replaces

`match-control.html` is `index.html` plus the Mission Control tab, minus the
QR panel, with `computeDayIsLive()` hardcoded to `true`. There are four copies
(~13,700 lines) and each shares a ~106-line configuration block with its
sibling `index.html` — the duplication CLAUDE.md already names as this repo's
known weak spot. All four go away.

> `index.html` is **not** affected. It stays per-event, keeps its own config
> block, and keeps its club logos. It is the public, spectator-facing page;
> this is the operator's.

---

## 2. Configuration — `events.json` is the only source

### 2.1 New fields

```jsonc
"pnf-x-bup-dual-meet": {
  "type": "dual-meet",                 // NEW — picks layout + team-code shape
  "archived": false,                   // NEW — true hides it from the picker (§9)
  "title": "PNF × BUP Dual Meet",      // NEW — masthead
  "days": {
    "pnf-x-bup-day1": {
      "label": "Sep 12",               // unchanged, already present
      "date": "2026-09-12",            // NEW — see §2.5
      "isLive": "auto",                // NEW — operator-controlled, see §4.1
      "facilities": [ ... ]            // unchanged
    }
  },
  "display": {                         // NEW — entirely optional (§2.3)
    "divisions": { "LI": "Low Intermediate", "HI": "High Intermediate", "A": "Advanced" },
    "events":    { "WD": "Women's Doubles", "MD": "Men's Doubles", "XD": "Mixed Doubles" },
    "clubs":     { "PNF": "Pickle & Friends Community", "BUP": "1Bataan United Picklers" }
  }
}
```

`type` is `"dual-meet"` or `"standard"`, matching the two templates. It decides
both the standings layout (per-club columns + cross-club Bronze/Final, vs flat
category cards) and the team-code shape (`<CLUB>_<DIV><EVT>_<REST>` vs
`<DIV><EVT>_<REST>`).

> **`type` must be explicit, not inferred.** Distinguishing `A_B_C` from
> `A_B` by counting segments works most of the time and fails *silently* —
> unmatched codes currently fall back to "Other" with no warning, which is the
> dangerous kind of failure. An unknown or missing `type` is an error the
> console shows, not a guess.

All three `display` maps are **the same shape: code → label.** One resolver
handles all three. Ordering comes from JSON key order, so `DIVISION_ORDER` /
`EVENT_ORDER` disappear entirely — a simplification over today.

**No logos.** `display.clubs` is a plain string map, not an object. The club is
already on every row as its 3-letter code; the live table would otherwise
render 18 logos at once (9 courts × 2 sides) carrying no information the
adjacent tag doesn't. Dropping them is also what keeps `clubs` the same shape
as `divisions`/`events`, and it sidesteps the asset-path question entirely
(logos live in `events/<key>/assets/`, which moves when an event is archived).

### 2.2 Derived from the snapshot, not configured

Verified against the live `pnf-x-bup-day1.json`:

| Thing | Today | Derivation |
|---|---|---|
| Facility → courts | hand-written `FACILITIES[].courts: [1,9]` | **Each facility ships its own CSV**, so its court set is knowable. Recovered `courts 1..9` with no config. |
| Club set | `CLUBS` keys | 1st code segment — `BUP, PNF` (2 distinct across 126 matches) |
| Category set | `DIVISIONS` × `EVENTS` | 2nd code segment — 7 distinct |
| Stage (RR/QF/SF/…) | `STAGE_META` | code tail; the regexes are generic, not per-event |
| Standings grouping | `divisionEventLabel()` | group on the whole category token |
| Day label | `DAYS[].label` | snapshot's own `label` field |
| Courts in play, scores, players, live state | — | data |

> Facility → courts is the one that is *better* centralised. A hand-written
> range can drift from what the sheet actually contains; a derived one cannot.

### 2.3 Not derivable — and what happens without it

Only four things genuinely need `display`:

1. `LIWD` → "Low Intermediate Women's Doubles"
2. `PNF` → "Pickle & Friends Community"
3. Deliberate category order
4. The event title

The first cannot be inferred: splitting `LIWD` into `LI`+`WD` requires the
vocabulary, since `LIW`+`D` is equally valid to a parser. That is exactly why
`CODE_REGEX` is built from the config today.

**But the split is needed only for labels and ordering, never for structure.**
Grouping on the whole `LIWD` token produces byte-identical groups to today.

So: **no `display` block → the console still works**, showing raw codes
(`LIWD`, `PNF`) and first-appearance ordering. A newly registered event is
usable immediately; `display` is polish added when convenient.

### 2.4 Resolution failure must be loud

With `display.divisions` present, split a category token by **longest-prefix
match** against the division keys, then map both halves.

This is ambiguous only when one division key prefixes another *and* both
remainders are valid events (divisions `A`/`AM` with events `MD`/`D`). Rare,
but:

> After resolving, assert every code mapped. If any did not, show a visible
> warning naming them. Do **not** inherit the current silent "Other" fallback —
> a mislabelled board that looks fine is worse than one that says it is wrong.

### 2.5 Day scheduling — `date` and `isLive`

`date` moves here because the console needs it: `label: "Sep 12"` is not
machine-sortable, and the console has to order days chronologically and default
to the one being played.

`isLive` moves here for a different reason — it stops being a config value
edited by hand and becomes **an operator control in the console** (§4.1). This
section defines where the value lives; §4.1 defines the button.

---

## 3. The tabs

Live Matches, Match Finder and Standings behave as they do today, with three
differences:

- **No club logos** (§2.1) — the `.score-logo`, `.live-team-logo` and
  `.cs-logo` treatments are not carried over. The hero's `.club-logos` pair is
  not carried over either.
- **Always live.** The console ignores `isLive` and the go-live hour, as
  today's `match-control.html` does (`computeDayIsLive()` returns `true`), so
  an organizer can preview a day before it opens publicly.
- **Labels degrade to codes** when `display` is absent (§2.3).

Standings layout is chosen by `type`: `dual-meet` gets the per-club columns,
the club win summary and cross-club Bronze/Final; `standard` gets flat
category cards with the toggle bar and mobile category filter.

### 3.1 Theme — SAGE, and built that way from the start

The console is a **tool**, and it lives in `tools/` beside
`scoresheet-generator.html` and `tournament-calculator.html`, which are where
the S.A.G.E. palette came from in the first place. It uses that palette
directly: navy structure, green accent, off-white paper, Archivo Black /
Barlow Condensed / Inter, the same `:root` token block those two share with
`schedule.html` and (since the recent conversion) both event templates.

Two consequences worth stating up front:

- **Build it SAGE-native, don't port and convert.** The per-event
  `match-control.html` it replaces was originally a dark page that was
  converted; converting a second time would inherit the artefacts of the
  first pass. The console is a new file — write it against the tokens.
- **It does not take on per-event theming.** An event may re-skin its own
  `index.html` (runbook §2 step 5), but the console looks the same whichever
  event is selected. It is one tool that happens to be pointed at different
  data, and an operator switching events should not see the furniture move.

Carry over the two rules the conversion established, since they are easy to
get wrong on a fresh page:

> **Green is a fill, not a text colour.** `--green` on white is ~2.3:1 and
> fails AA at any size; `--green-dark` is ~4.0:1 and still misses the 4.5:1
> body floor. Use green as a background with `--navy` text on it (~5.4:1), or
> on a navy panel. Small text on paper is `--ink` or `--ink-soft`.

> **The masthead is a navy bezel on paper**, matching the tools and the
> schedule board. Controls sitting on it are light-on-navy and need their own
> scoped rules — the shared control styles are written for the paper ground of
> the body. `schedule.html`'s `HERO-SCOPED CONTROL OVERRIDES` block is the
> worked example.

Responsiveness gets the same treatment as the schedule board: the console will
be opened on a phone at a venue, so the masthead wraps rather than forcing the
document wider than the viewport, and the live table stays readable at 375px.
See `schedule-screen-spec.md` and the `NARROW SCREENS` block in
`schedule.html`.

That held for the live table but not for everything around it. Three things
were overflowing their containers on a phone and have since been fixed:

- **The view-tab row.** Five pills need ~400px in a row, so under roughly a
  440px viewport the first and last tabs were clipped by the hero and could
  not be tapped at all — Live Matches and Mission Control were simply
  unreachable. It is now a horizontal scroller below the breakpoint, with
  the leading edge fading only while there is more to scroll toward and the
  newly activated tab scrolled into view. `justify-content` has to flip to
  `flex-start` there: a centred flex row that overflows makes its *leading*
  items unreachable in every browser, which is the same bug at the other end.
- **The standings club-summary boxes**, whose contents were `flex:0 0 auto`
  inside a `flex:1` box, so nothing could give and each box spilled past its
  own border.
- **Mission Control's status rows** at 320px, which now wrap the detail onto
  its own line rather than truncating the facility name.

Tap targets are sized off `@media (pointer:coarse)` rather than a width
breakpoint, so a tablet gets them too. The organizer secondary buttons had
`padding:0 16px` with no vertical padding, leaving them 16px tall — awkward
with a mouse and unusable with a thumb.

Still outstanding, and **desktop** rather than mobile: the standings grid
wants ~1100px inside a 920px `.wrap`, which squeezes the pair-name cells to
16px so player names overflow them. Pre-existing, unrelated to the mobile
work, not yet fixed.

---

## 4. Mission Control

Unchanged in behaviour, but scoped to the selected event:

- Per-facility sync status (`Synced 4h ago` / `Never synced` / `Last attempt
  failed`), read from the loaded snapshot — no extra fetch.
- `Check connection`, `Resync this day now`, optional per-facility resync
  (shown only when the event has more than one facility).
- `Use CSV export fallback instead of Sheets API` toggle.
- **`Open schedule`** launcher → `/events/<event-key>/schedule.html`. This is
  the schedule board's only entry point and it moves here from the per-event
  page.

**One sign-in covers every event.** The console has an operator sign-in form
(username + password, one shared combo for the whole team) that calls `POST
/auth/login`. The API checks the pair against a stored hash and returns a
short-lived signed token; the console holds that token in `sessionStorage`
under a single `sage.authToken` key (not per event), never the credentials
or the raw `SYNC_SHARED_SECRET`. Signed-out operators still get read access —
schedule, live matches, standings, match finder — and only the actions
(resync, go-live override) are gated on the token. See
`sage-tools-api/src/auth/AuthService.mjs`.

### 4.1 Go-live override — the kill switch as a button

`isLive` is documented in the code today as "a kill switch if something needs
to come down mid-day", but pulling it means editing `index.html`, committing,
and waiting for Jekyll to rebuild the site repo. That is the wrong shape for
something you reach for under pressure. It becomes a control here instead.

**Three states, per day** — not a binary. Returning to `auto` is the normal
end state after an incident, so the UI must offer it explicitly:

| State | Public site shows | Use |
|---|---|---|
| `auto` | live 4 hours before the day's earliest scheduled match time (read from the synced `Schedule` column, `date` for the day) | the default |
| `true` | live now, regardless of schedule | opening early |
| `false` | Live/Standings hidden, scores suppressed | the kill switch |

#### How it takes effect

The write path already exists and needs no new plumbing:

- `GitHubPublisher.publish(path, json, message, knownSha)` is generic — it is
  already the shared writer for both `config/events.json` and the day
  snapshots.
- `SyncService` already re-reads config on every sync and rebuilds the
  snapshot from it (`label` comes from `config.getDay(day)` this way).

So: **`events.json` is the source of truth, and the published snapshot is a
mirror.** `SyncService` stamps the day's current `isLive` into every snapshot
it writes, exactly as it already does with `label`. `index.html` then gets the
flag from the file it already polls — no second fetch, no second poll, no new
failure mode.

A new secret-gated endpoint does both halves:

```
POST /sync/:day/live   { "isLive": true | false | "auto" }
```

1. Write the new value into `config/events.json`.
2. **Immediately republish that day's snapshot** so the change lands on the
   public site within one 10s poll.

> Step 2 is the whole point. Without it the flag would only reach the public
> site the next time somebody edited the spreadsheet — a kill switch that
> waits for an unrelated event is not a kill switch. This is also why the flag
> cannot live *only* in the snapshot: a snapshot is a derived artifact, and a
> rebuild from scratch would lose the operator's intent.

Because the value is sourced from config and re-stamped on every sync, a later
sync **cannot silently revert an override** — a property worth an explicit test
(§8).

#### UI requirements

- Show the **current effective state** prominently, and for `auto`, when it
  will flip (`"Auto — - goes live 7:00 AM, 12 Sep"`). A forgotten `false` that
  is not visible is the main failure mode of this feature.
- Say plainly that this controls **the public site**, not the console. The
  console ignores `isLive` for its own display (§3), so an operator who
  hides a day and sees no change in front of them will otherwise assume the
  button is broken.
- Confirm before `false`. It is the one destructive state.

> **This feature does nothing until `index.html` reads the flag from the
> snapshot.** The console button and that change ship together or not at all;
> a toggle that silently has no effect is worse than no toggle.

---

## 5. Event and day selection

- Event list comes from `events.json` over plain HTTPS with the same
  `?t=${Date.now()}` cache-buster the snapshot fetches use — GitHub Pages
  caches for ~10 minutes otherwise.
- Selection rides in the URL (`?event=<key>&day=<key>`) so an operator can
  bookmark straight into the event they are running, matching how the schedule
  board carries `?courts=` and `?compact=`.
- Last selection also remembered in `localStorage` for a bare visit.

> `sage-tools-api`'s `SYNC_CONFIG_TTL_MS` does **not** apply here. That is the
> server's own cache of `events.json`; a browser fetching the file directly
> gets it fresh.

---

## 6. What leaves the templates

- Delete `_templates/dual-meet-template/match-control.html` and
  `_templates/standard-tournament-template/match-control.html`.
- `{{CLOUD_RUN_BASE_URL}}` disappears from the runbook's token table — it was
  used only by `match-control.html`, and the console is a single file that can
  hardcode it.
- Runbook §2 loses the match-control half of step 4 (config block) and the
  Mission Control references; the runbook's §3 token table loses one row; its §5.1 hand-sync
  table loses its `match-control.html` column.
- `index.html` gains a preview escape hatch (§9.1).

### 6.1 Cutover, given the console runs PNF × BUP on 2026-09-12

The decision (§9, decision 2) is to cut over before the event, so the console's first
real use is a live tournament. Two rules make that safe without slowing it
down:

1. **Keep `events/pnf-x-bup-dual-meet/match-control.html` in place through the
   event.** Deleting it is build step 7 and costs nothing to defer. Leaving it
   means that if the console misbehaves mid-event, the operator has a known-good
   page at a URL they already have — no rollback, no deploy, just a different
   bookmark. Delete it once the event is over.
2. **Delete the two template copies only after that.** They are what a *future*
   event instantiates from, so nothing forces their removal on this timeline,
   and keeping them costs only tidiness.

> This is not a hedge against the decision — the console still becomes the
> tool used to run the event. It just means the first live use has a fallback
> that requires no action to reach.

Rehearse before the day: run a real resync through the console against the
live event, confirm the sync status rows match what the per-event page shows,
and exercise the go-live override end to end (§8) while it still doesn't
matter.

---

## 7. Build steps

1. Add `type`, `title`, `date`, `isLive` and `display` to both events in
   `event-data`'s `config/events.json`; update `config/README.md` for the new
   shape.
2. Build `tools/match-control.html`: config resolution layer (§2) first,
   proven against both events, then the four tabs, SAGE-themed from the start
   (§3.1).
3. Verify against the live snapshot at both `type` values, and with the
   `display` block removed, to confirm the degradation path (§2.3).
4. **Go-live override (§4.1), as one unit:**
   1. `SyncService` stamps the day's `isLive` into every published snapshot,
      alongside `label`.
   2. `POST /sync/:day/live` in `sage-tools-api` — writes config, then
      republishes that day's snapshot.
   3. `index.html` (both templates) reads `isLive` from the snapshot instead
      of its own `DAYS` entry; there is no `LIVE_GO_LIVE_HOUR_PH` field —
      the `auto` threshold is computed from the Schedule column (§2.5, §4.1).
   4. The console's control.
5. Add the preview hatch to `index.html` in both templates (§9.1).
6. Delete the two template `match-control.html` copies; update the runbook (§6).
7. Delete `events/pnf-x-bup-dual-meet/match-control.html` once the console has
   been used successfully for that event.

---

## 8. Acceptance

- [ ] `/tools/match-control` lists both registered events with no per-event code.
- [ ] Selecting `pnf-x-bup-dual-meet` renders 130 matches, 84 standings rows,
      7 category columns, the Advanced overflow column, and 7 cross-club
      Bronze/Final blocks — matching today's per-event page.
- [ ] Facility → court mapping is derived, not configured, and matches
      `courts 1..9`.
- [ ] Removing the `display` block leaves everything working, showing `LIWD`
      and `PNF` as raw codes.
- [ ] A code that fails to resolve produces a visible warning, not "Other".
- [ ] An event with `type` missing or unknown shows an error, not a guess.
- [ ] The secret is entered once and works for every event in the session.
- [ ] `Open schedule` opens the selected event's board in a new tab.
- [ ] `?event=&day=` round-trips through a reload.
- [ ] No club logos anywhere on the console.
- [ ] Scores are visible for a day that has not reached its go-live hour.

Go-live override (§4.1):

- [ ] All three states are reachable, and the current one is shown; `auto`
      also states when it will flip.
- [ ] Setting `false` hides Live/Standings on the public `index.html` within
      one poll (~10s), without a manual resync or a sheet edit.
- [ ] Setting it back to `auto` restores the scheduled behaviour.
- [ ] **A subsequent sync does not revert the override** — set `false`, edit
      the sheet to trigger a real sync, confirm the day is still hidden.
- [ ] The console's own tabs stay visible while a day is force-hidden, and the
      UI says the control affects the public site.
- [ ] `POST /sync/:day/live` rejects a request with a missing or wrong
      `X-Sync-Secret`.
- [ ] Selecting an event whose snapshot is missing shows a clear "not
      published" state, not a blank board or a stuck spinner (§9, decision 4).
- [ ] Multi-facility rendering exercised. **No live data currently exists for
      this**: `bkl-cup-2026` is registered with days `day2`–`day6`, but every
      one of its snapshots returns 404, and `pnf-x-bup-dual-meet` has a single
      facility. Test against a hand-built fixture, or defer this until an
      event with more than one venue actually runs.

---

## 9. Decisions taken

1. **Full scope in v1.** The console *and* the go-live override ship together,
   including the `sage-tools-api` endpoint, the `SyncService` change, the Cloud
   Run deploy, and both templates' `index.html` reading the flag.
2. **Cut over before 2026-09-12**, so PNF × BUP is run from the console.
   See §6 for how that is de-risked.
3. **No preview hatch on `index.html`.** The console ignores `isLive` by
   design, so organizers preview there. This removes the `?preview=1` question
   entirely — no guessable URL undermining the embargo that `isLive` exists to
   enforce. `index.html`'s only change is reading the flag (§4.1).
4. **Dead registry entries get `"archived": true`** and are filtered out of the
   event picker. The "not published" state is still built, as the fallback for
   a day that is registered and current but has not synced yet.

### 9.1 Defaults applied without further asking

All reversible; say so if any is wrong:

- A snapshot with **no `isLive` field** (anything synced before this ships) is
  treated as `auto`, so existing events keep their scheduled behaviour rather
  than going dark.
- The console is **not linked** from any tools index.
- **No in-page audit trail.** Every override writes a commit to `event-data`,
  so `git log config/events.json` already records who changed what and when.
- `display` labels living in the data repo is **treated as confirmed** — it is
  already written into the runbook at your request.

## 10. Still open

1. **Multi-facility rendering has no live test case.** `pnf-x-bup-dual-meet`
   has one facility and `bkl-cup-2026`'s snapshots all 404, so the
   per-facility grouping on the Live board cannot be exercised against real
   data. Either build a fixture or accept that this path ships unverified
   until an event with more than one venue runs. Flagging rather than
   deciding, because it is a testing-cost question, not a design one.

---

## 11. Out of scope

- Any change to `index.html`'s look, config, or club logos, beyond the two
  functional ones it needs: reading `isLive` from the snapshot (§4.1) and the
  preview hatch (§9.1). Its `DIVISIONS`/`EVENTS`/`CLUBS` block, its layout and
  its logos all stay exactly as they are.
- The schedule board. It stays per-event: it needs `CAT_META`, which is
  organiser-owned presentation read off the source spreadsheet, and it is tied
  to one venue and one day.
- `bracket-generator.html`. Already self-contained, no config, no sheet access.
- Moving presentation config out of `index.html`. The public pages keep their
  own config block; only the console is data-driven.
