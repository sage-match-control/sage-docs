# Control Center

One operator console (`tools/control-center.html`) covering every registered
event. The alternative it displaces is a separate copy of the console per
event, each carrying its own hand-duplicated configuration block — which is
why the console carries **no per-event configuration of its own**: it picks
an event from
`event-data/config/events.json` (see [event registry
schema](event-data-config.md)), reads that event's registry entry, and
renders. Its views are the [site engine](site-engine.md)'s: the Match
Finder, tickets, Live Matches, Standings, team views, score dialog and
attendance list come from `lib/v1/views/` (and their CSS from `lib/v1/css/`),
the rules from `lib/v1/domain/`, so the console and an event's Tournament Hub
draw the same markup. The page itself keeps what only an operator has: the
event picker, Mission Control, sign-in and the extension points that attach
score entry, the match-number search and the facility progress cards to the
shared views.

> `tools/match-control.html` is a redirect stub to `/tools/control-center`, so a
> bookmarked link to it keeps working. [`match-control-console-spec.md`](../specs/implemented/match-control-console-spec.md)
> and [`awards-podium-tab-spec.md`](../specs/implemented/awards-podium-tab-spec.md)
> call the console *Match Control* throughout; they are a record of how it was
> built.

## Config resolution

Selecting an event resolves three things fresh, every time:

- `CURRENT_EVENT_KEY` / `EVENTS_REGISTRY` entry
- `CURRENT_TYPE` — `"dual-meet"`, `"standard"` or `"team"`, **required and explicit**,
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

**Live Matches, Match Finder, and Standings** have no club logos anywhere
(no `.score-logo`/`.live-team-logo`/`.cs-logo` treatment); the console
**ignores `isLive` and always shows live data** (it builds its day model with
`alwaysLive: true`), so an operator can preview a day before it's public;
and labels degrade to raw codes when `display` config is absent rather
than erroring.

**Standings layout is chosen by `type`.** `dual-meet` gets per-club columns
inside each category with a shared club win-total summary bar and the
cross-club Bronze/Final shown as one block spanning both clubs (see
`CROSS_CLUB_STAGE_KEYS`); `standard` gets flat category cards with a
desktop toggle bar and a mobile category-search filter; `team` gets
group tables, playoff matchup cards and collapsible bracket matchups (see
[the team type](#the-team-type) below).

**Unnamed pairs.** `isEmptyStanding` marks a row whose `player1` is blank or
still equals its `teamCode` (the sheet fills names by formula, defaulting to
the code). A twice-to-beat game's code carries its game number
(`IXD_F_1_(2)`) but its placeholder name doesn't (`IXD_F_1`), so the name is
also compared with the code minus that suffix. `renderStageTables` keeps such an RR row when
`isUnnamedScheduledPair` holds: it is an RR code and appears in
`MATCH_BY_CODE`, so it's a real pair on the schedule, and `pairCell` prints
its code. A code with no match, an unused slot in a hand-built workbook with
fixed-size rosters, is skipped. Playoff cards and the Awards podium keep using `isEmptyStanding`
alone, so they show TBD and never award a code. The same pair of functions
is in every standard and dual-meet event page and both templates.

**Round Robin brackets.** `renderStageTables` groups a category's Round
Robin rows by bracket label. With no labels it draws one table; with labels
it draws one mini table per bracket, each under a colored `Bracket <n>`
label, in `.rr-bracket-grid` — at most two per row, a third wraps. It
returns `{ html, hasMultiBracket }`; a `standard` category card with two or
more brackets gets `.standings-col--wide` on desktop, exactly two columns'
width (740px = 360 × 2 + the 20px gap). Its header carries a desktop-only
`.bracket-cols-toggle` button; clicking it adds the category to
`stackedCategories`, which swaps `--wide` for `.standings-col--stacked` (a
normal 360px column with the brackets in one grid column). The set lives in
memory, so the choice survives each poll's re-render but not a page reload.
A `dual-meet` club subsection uses
the same grid but never widens its column. The standard template's
`index.html` renders standings the same way; the dual-meet template does not
split by bracket.

**Desktop standings** need a window at least 900px wide **and** 600px tall
(`DESKTOP_STANDINGS_QUERY`, `(min-width:900px) and (min-height:600px)`).
The height half keeps big phones turned sideways, up to about 956px wide but
under about 450px tall, on the mobile layout. `isDesktopStandings()` reads
the query through `matchMedia`, and the stylesheet's `@media` rules use the
same query, with `(max-width:899px), (max-height:599px)` for the mobile
side; change them together. The public event pages and both templates use
the same query. Desktop standings break out of the page's `.wrap` to the
full viewport width. Every category is one fixed 360px column
(`flex:0 0 360px`) in a single row that scrolls horizontally, always
`justify-content:flex-start` — a centred flex row that overflows cuts off
its first columns in Chromium. Each column is capped at `92vh` and scrolls
on its own. These sizes match the public event pages' standings board.
Pair names never wrap or truncate: each table sits in a `.br-table-wrap`
that scrolls horizontally if a name is wider than the card. A row whose names
aren't in yet (`TBD` or its code) keeps the two lines a pair's names take, so
it doesn't grow when they arrive. Playoff matchup cards hold the same height
the same way, but there each name keeps to one line, cut with … when too long
(full names on hover). The public event pages do the same in both places,
cutting long names in the round-robin tables too. The desktop
row and every `.br-table-wrap` are in `SCROLLABLE_SELECTOR`, so their
scroll positions survive the re-render that every update triggers (a pushed snapshot, or the 10-second poll).

**Awards** is the second tab — full derivation and export architecture in its
own section below.

**Mission Control** — see [Mission Control usage](../features/control-center.md#mission-control)
for what it does; scoped to whichever event is selected. Every call the console
makes goes to the API's `/v3` surface ([API](api.md)): sign-in is
`POST /v3/sessions`, **Resync** is `POST /v3/days/{day}/syncs` (or
`…/facilities/{facility}/syncs`, with `{ "method": "csv" }` in the body for the
CSV fallback) and the go-live switch is `PUT /v3/days/{day}/visibility`. The
legacy and `/v1` routes it called before 3.0.0 stay served, frozen, so a cached
copy keeps working. The go-live states are `auto`
(live 4 hours before the day's earliest scheduled match time, computed live
from the synced `Schedule` column — no hour to configure), `true` (force
live), and `false` (Live/Standings hidden, scores suppressed on Tournament
Hub only — this console's own tabs stay live regardless, so an operator can
verify a fix before un-hiding). **Check connection** also reports whether Cloud Run is publishing to live push
(`GET /v3/diagnostics/sync`'s `live`, which needs the sign-in; a `404` there
reads as "Cloud Run is older than 3.0.0"), and **Sync method** is the emergency
switch between live push and GitHub only (`PUT /v3/settings/live-push`, which
writes `livePush` into `config/events.json`). The page
itself holds the same live channel block as the public pages, and the
**Live updates** row in Facility sync status reports whether its own socket is
open. The connection-check line also surfaces the
currently-cached sync config's short SHA (see [sync
pipeline](sync-pipeline.md) § Observability), so an operator can see at a
glance whether a just-committed `events.json` change has actually taken
effect yet.

Each facility row in the Facility Sync Status list links directly to that
facility's actual Google Sheet (`docs.google.com/spreadsheets/d/<sheetId>/edit`).
`FACILITIES` (resolved per day-select from `EVENTS_REGISTRY`) carries
`sheetId` alongside `name` for exactly this. This exposes nothing extra:
sheet IDs are already public in the
same `config/events.json` fetch the console already makes (see [event
registry schema](event-data-config.md) § Sheet IDs are effectively
public). The link is omitted for a facility with no `sheetId` yet, the
same "not set up" condition the sync pipeline itself skips.

**Tab order and landing.** The tabs run Mission Control, Awards, Attendance,
Match Finder, Live Matches, Standings, Teams. Teams shows only for a `"team"`
event whose snapshot carries `rosterCsv` (`syncTeamsTab()`). Attendance is hidden unless the event's
`attendance` is `"console"` or `"desks"` (`syncAttendanceTab()`, called on
every event change, day change and tab reveal). `showView(view)` is the one place that switches
views (the tab buttons call it), and `revealLiveTabsAfterLoad()` calls
`showView('organizer')` once a day loads, so the console opens on Mission
Control.

**Match Finder's empty state and number search.** With nothing searched,
`renderIntro()` adds `allMatchesHTML()`: every non-bye match as a ticket in
match-number order, built with `ticketHTML(m, null, false)`, where a `null`
team code means a neutral ticket (team 1 on the left, no **You** badge). A
team event's `teamRenderIntro()` lists every matchup card after its team
chips through `allMatchupsHTML()`, sorted by `matchupFirstNum()` (the
matchup's lowest match number, so #3/#8/#9/#17 sorts as 3) rather than
trusting `TEAM_MATCHUPS`' insertion order, which follows the CSV's row order.
PickleDrive's event page carries the same two functions.

**A refresh redraws the search that was run, not the box's text.**
`runSearch()` records what it shows (`shownQuery`, and the picked
`shownTeam`/`shownEntry`). Every data refresh calls `refreshFinder()`, which
re-runs that search, or redraws the all-matches list when nothing was
searched, then puts back whatever is half-typed in the box. Running the box's
text instead would swap the list for a short "Several pairs match" message while
someone pauses mid-word, shrinking the page so the pinned search box has nothing
left to scroll against. A new day clears
`shownQuery`. The event pages work the same way. `runSearch()` first checks `MATCH_NUMBER_SEARCH_RE` (`42` or `#42`);
a match number goes to `renderMatchByNumber()` (the ticket, or the matchup
card with `onlyMatch`) and is not saved as the remembered search. The event
pages carry the same `allMatchesHTML()` and neutral ticket, but not the
number search.

**How outcomes are shown.** Short outcomes go to `showToast(kind, text, key)`,
a stack pinned to the top of the screen (newest first) (`#toastStack`, `aria-live`), so
they are seen wherever the button sits: the go-live and sync-method switches
(their current state is already on screen), sign-in and sign-out, and
"pick a day first" / "sign in first". A toast with the same `key` replaces
the last one; errors stay about nine seconds and can be closed. Results worth
reading go in a box directly under their button, rendered as labelled rows by
`showResultRows()` and scrolled into view: **Check connection**
(`#orgConnResultBox`) and the resyncs (`#orgResultBox`, one row per facility
from the sync response via `resyncResultRows()`). Error text from
`sage-tools-api` embeds the upstream service's raw reply (`HTTP 403 {"error":
{...}}` from Google, `{"message": ...}` from GitHub, `{"error": ...}` from the
live Worker); `friendlyApiMessage()` turns each `HTTP <status> <json>` into
words plus the JSON's own message before anything is shown.

Every finished result box gets a close button (`addResultClose()`); one still
showing its `loading` message doesn't. `clearResultBoxes()` hides and empties
every `.organizer-result` and bumps `resultEpoch`; `selectEvent()`,
`selectDay()` and `showView()` (only when the view actually changes) call it.
A box records the epoch on its `loading` message, and a result arriving under
a later epoch (a resync still running when the operator switched day) goes to
`showToast()` instead, keyed by the box's id, so it is neither lost nor shown
under the wrong day. Toasts are made visible by flushing styles
(`void toast.offsetWidth`) before adding `show`, not with
`requestAnimationFrame`, which never fires while the page isn't painting.

## Teams tab

Team events only. The day model parses each facility's `rosterCsv`
(`teamCode,player,level,gender`) into `model.team.roster`, and
`renderRosters()` (`createTeams` in `lib/v1/views/teams.js`) draws one collapsible card per team, grouped by
bracket from the standings, players sorted by level. The finder's index adds
roster players to Match Finder, and the team and player results carry a
roster card. The rules (`lib/v1/domain/teams.js`) and the cards are a port of the event page's
Teams tab (`events/pickledrive-anniversary-2026/index.html`), whose copy is
frozen with that finished event.

## Attendance tab

The list is `createAttendanceView` in `lib/v1/views/attendance-view.js`, shared
with the desk page (`lib/v1/apps/attendance-desk.js`); its styles are
`lib/v1/css/attendance.css`. It reads each
facility's `ATTENDANCE` tab through the gviz CSV export every 10 s while
visible, and marks through `PUT /v3/…/people/{personKey}/attendance` with the operator
token. In this console it loads every facility of the day, so the counts are
complete.

Around it, the console's own `attConsole*` section adds the per-facility
counts, **Update roster** (`POST /v3/days/{day}/attendance/reconciliations`),
**Issue desk link** (`POST …/desk-links`, QR drawn in a canvas from
`qrcode-generator` 1.4.4 on cdnjs, SRI-pinned, loaded on first click) and
**Needs attention**. `showView()` mounts the view on entering the tab and
destroys it (and its poll) on leaving. Messages go through `showToast` and
`showResultRows` like the rest of the console. With `?fixture=` on localhost
everything reads `_fixtures/` and makes no API call; `?attfail` makes the first
mark fail.

Full design: [event attendance](event-attendance.md).

## Score entry

An operator enters, corrects or clears one match's score from Match Finder
(`PUT /v3/days/{day}/facilities/{facility}/matches/{matchNumber}/score`, see [sync
pipeline § Score entry writes](sync-pipeline.md#score-entry-writes)). The
feature is on for an event whose `events.json` entry has `"scoreEntry":
"console" | "links"`, and the console only offers it while an operator is signed
in. Usage: [Entering a score](../features/score-entry.md).

### The score dialog

The dialog is `createScoreDialog` in `lib/v1/views/score-dialog.js`, shared with
the scorer page (`lib/v1/apps/scorer.js`); its `.score-*` rules are
`lib/v1/css/score-dialog.css`. Each page carries the `<dialog
id="scoreDialog">` markup, and the dialog finds its parts inside it by id. It uses no global of either
page: everything that differs is passed in through `opts`.

| Option | What it is |
| --- | --- |
| `dialog`, `apiBase`, `getToken()`, `fixture` | the element, Cloud Run's base URL, the bearer token (the operator's here, the scorer's on the scorer page), and the localhost fixture name |
| `describe(m)` | the title, the sub-title and the two sides (`name`, `code`, `players` or `null`, and `missing`, the text for a side with no players) |
| `readOnlyReason(m)` | a reason to open read-only, or `null` |
| `extraWarnings(m)` | the page-specific warnings (an unneeded series game, a missing lineup) |
| `findMatch(facility, num)` | the page's current copy of a match, for `refresh()` |
| `notify`, `friendlyError`, `unreachable` | outcomes as toasts, and error text in plain words |
| `expiredText`, `resyncHint` | the sentences that differ between an operator and a scorer |
| `onFixtureSave` | patches the page's own data when a fixture save "succeeds" |

It returns `{ open, close, refresh, isOpen }`.

### Which matches are clickable

`scoreEntryAvailable()` reads the registry entry's `scoreEntry` and the sign-in.
`scoreAttrs(m)` returns the `data-score-*`, `role` and `tabindex` attributes, or
nothing for a BYE or when entry is not available; `scoreHintHTML(m)` is the
pencil hint. `ticketDecorate(m)` packs them into the `decorate` option the
shared ticket and the team matchup rows take (`decorateRow`). Only Match Finder
asks for it, so the console's other tabs are read-only: the team views' Match
Finder callers (the team and player results, the all-matchups list and the
match-number search) pass `interactive: true` and Standings' calls leave it off. Two delegated listeners on the results
container (click, and Enter or space on a focused match) open the dialog, because
Match Finder redraws all its markup on every poll and push, which would drop any
listener bound to a single match. Signing in or out redraws Match Finder, so the
hint appears and disappears with it. Changing the event or the day closes the
dialog.

### States

One state object lives inside the closure: `readonly`, `enter`, `review`,
`saving` or `conflict`. It holds **a copy** of the match
and the `expected` values the person was shown (the two team codes and the two
scores). It copies because every poll and push rebuilds the page's match list, and
a dialog that read the live object would change under the operator mid-entry.
`expected` is what the API's check compares the sheet with. `refresh()`, called
after every snapshot, compares the page's current match with `expected` and puts
a note under the boxes; it never touches the inputs and never updates `expected`,
so saving after the sheet changed gets a 409 and the person chooses between
**Keep the sheet's score** and **Replace with yours** (which saves with the
sheet's current values as `expected`).

Esc and the × button are blocked while saving. A second Enter or space within 400
ms of Review appearing is swallowed in the capture phase, so a fast double Enter
from the last box stops on Review. The save request has a two-minute timeout.
After five seconds the dialog says the workbook is slow. A response that arrives
after the dialog closed is ignored. The 200, 409, 401 and other responses map to
toasts and steps as described in the spec's save table.

### Fixtures

On localhost, `?fixture=<name>` simulates a save: after 700 ms it patches the
page's match, redraws and says nothing was sent; the next poll restores the
fixture's scores. Adding `&scoreConflict=1` makes the first save of a dialog
answer with a conflict instead (the sheet "now reads" 11 – 9), so the conflict
panel can be exercised. `_fixtures/config.json` carries `"scoreEntry": "links"`
for both fixture events.

### The Scorer links section

Mission Control's **Scorer links** section (`renderScorerLinks()`) reads the
registry entry's `scoreEntry` and is hidden unless it is `"console"` or
`"links"`. It is called from `renderAuthStatus()`, from the top of
`renderOrganizerStatus()` and when the event or day changes, so it follows
sign-in and the selected day. The **Accepting** / **Stopped** toggle (a `role="switch"` button, the same `.switch` style as
the **Use CSV export fallback** toggle) calls `PUT
/v3/events/{event}/score-entry` (body `{ scoreEntry }`) with the operator token and then copies the new
value into the page's own registry entry, which would otherwise stay stale until
a reload. **Issue scorer link** calls `POST /v3/days/{day}/scores/scorer-links` and
`renderScorerLink` draws the result like a desk link: a read-only input, **Copy**,
**Share**, **Show QR** (`attConsoleShowQr`, which takes a label for the canvas's
`aria-label`) and the validity line. In a fixture no request is made; the issued
token has a real payload (`scope: "score-desk"` and the day) so the scorer page
can decode it, and the link points at the local `scorer.html`.

**The scorer page must exist.** `scorerPageKnown(eventKey)` does a same-origin `HEAD` request for
`/events/<key>/scorer.html` (a fixture also accepts `scorer-beta.html`), remembers the answer
per event for a minute and re-checks after that. Until it is `true`, `renderScorerLinks()` disables the
toggle and **Issue scorer link** and says why, and `setScorerLinksMode` and `issueScorerLink`
refuse to run even if the button is forced. A failed check counts as "no page". This is only a
console guard: the API cannot see the site, and a scorer link simply 404s without the page.

The dialog's side labels follow `describe()`: a side whose name equals its code (a pair) is led by its
players and shows the code as a chip; a side with a real name (a team) keeps the name as its
headline.

Full design of the page the link opens: [Scorer page](scorer-page.md).

## Facility progress

`computeFacilityProgress(matches, isEventDay, nowMin)` turns one facility's
matches into done/left counts, a schedule grid (slot length, rows, courts,
blank cells) and an estimated finish. `renderFacilityProgress` draws a card
per facility on Live Matches (called from `renderLiveMatches`), and
`facilityStatusProgressHTML` adds a line to each Mission Control sync row
(called from `renderOrganizerStatus`). Both go through `facilityProgressFor`
and `facilityFinishParts`, so the two can't disagree.

Times are *event-day minutes*: minutes since midnight at the start of the
day's `date`, which keep counting past 1440 after midnight. Schedule times
before `DAY_ROLLOVER_MIN` (6:00 AM) are read as the night after the event
day. The stale test in `facilityDataIsStale` is the same rule as the sync
row's `isWarn`; change both or neither. Design and worked examples:
[facility progress spec](../specs/implemented/facility-progress-spec.md).

### Left and In play

`computeFacilityProgress` returns `left` as every unplayed match, in-play ones
included, and `inPlay` separately. The estimate (`unitsLeft` counts an in-play
match as half), the "all matches done" test (`left === 0`), the stale check
and the server's `completedAt` all depend on that meaning. The card prints
`left - inPlay` as **Left**, so Done + Left + In play equals the total. Change
the printed number in `facilityProgressCardHTML`, never `left`.

### Actual end

A finished facility (`left === 0`) shows when play actually wrapped, beside
its scheduled end. `facilityActualEnd` supplies it, preferring the snapshot's
own `facilities[].completedAt`, which `SyncService` stamps server-side (see
[sync pipeline](sync-pipeline.md) § Facility completion). That stamp is set
once and carried forward, so it survives a manual resync and reads the same
on every device. `eventDayMinutesFromISO` converts it into event-day minutes
for `facilityFinishParts`, which rounds it to 5 minutes and reports the
difference from `plannedEnd` as "late", "early" or "on schedule". (An
unfinished facility's estimate keeps "behind" and "ahead".)

For a snapshot with no `completedAt`, the fallback is the
earliest `syncedAt` this browser has seen while the facility was complete,
kept in `localStorage` under `sage.facilityEnds` (keyed
`event|day|facility`) and dropped when the facility reopens. It is
per-device, so a device that first opens the page after a resync reads that
resync's time instead. That is why the server stamp takes precedence.

The "played", "BYE" and series-final rules here (`rowsToMatches`,
`sideIsBye`, `seriesGameOf`/`unneededSeriesGames`,
`computeFacilityProgress`'s `left`) are duplicated in
`sage-tools-api/src/sync/domain/facilityCompletion.mjs`. If one copy changes and the
other doesn't, the recorded end time disagrees with the card that announces
it.

## Installability

Installable via its own manifest, `tools/control-center.webmanifest`, the
same pattern [Tournament Calculator](tournament-calculator.md) established —
`scope` is the page path (`/tools/control-center.html`), not the `/tools/`
directory, so installing this doesn't sweep in `scoresheet-generator.html`
or the calculator itself. Icons and theme/background colors reuse the
shared site assets and palette, same as the calculator's manifest.

**Deliberately no service worker.** Unlike the calculator (whose whole
justification for offline support is that nothing it shows can go stale),
Control Center's entire value is live data — scores, sync status, court
assignments — so caching any of it risks showing an operator something
stale during a live event, exactly the failure mode the calculator's own
spec ruled out for pages like this. The manifest alone is enough for
"Add to Home Screen" / an install prompt and a standalone window; it adds
no caching and changes no runtime behavior. An installed shortcut still
needs a live connection to do anything, identical to a regular tab.

## Narrow screens

The console gets opened on a phone at a venue, so it holds up at 375px, and
at 320px on the smallest common handsets. Three pieces carry that:

- **The view-tab row** is a horizontal scroller below the breakpoint. Five
  pills need ~400px in a row, more than a phone has, so they scroll rather
  than wrap — wrapping costs ~30px of vertical space in a hero that is
  already tall. An edge fade appears only on the side that still has content
  past it, and activating a tab scrolls it fully into view, so nothing is
  ever hidden without a cue.

  `justify-content` is `flex-start` while scrolling, not `center`. A centred
  flex row that overflows makes its *leading* items unreachable in every
  browser.

  The row is measured when it is **revealed** (after a day loads), not at
  script load — while `display:none` its `scrollWidth`/`clientWidth` both
  read 0. Anything that reveals those tabs by another path must call
  `syncViewTabsScroll()`.

- **The standings club-summary boxes** let their identity and stat blocks
  shrink (`min-width:0`) below 520px. They sit in a `flex:1` box, so
  `flex:0 0 auto` contents would have nothing to give and would spill past
  their own border.

- **Mission Control's status rows** wrap their detail onto its own line on
  small screens, rather than truncating the facility name.

Tap targets key off `@media (pointer:coarse)` rather than a width
breakpoint, so a tablet gets them too — it is wide but still driven by a
thumb.

## Theme

The console is a **tool** — it lives in `tools/` beside
`scoresheet-generator.html` and `tournament-calculator.html`, which are
where the S.A.G.E. palette (navy structure, green accent, off-white paper;
Archivo Black / Barlow Condensed / Inter) originated, and it uses that
palette directly rather than porting a converted theme from the old
per-event pages. Two consequences:

- **It does not take on per-event theming.** An event may re-skin its own
  Tournament Hub, but the console looks identical regardless of which event
  is selected — it's one tool pointed at different data, and an operator
  switching events shouldn't see the furniture move.
- Contrast rules worth knowing if touching the CSS: **green is a fill
  colour, not a text colour** (`--green` on white is ~2.3:1, fails AA at any
  size — use it as a background with navy text on top, or on a navy panel).
  The masthead is a navy bezel on paper; controls sitting on it need their
  own light-on-navy overrides rather than reusing the paper-ground base
  rules.

## Awards tab

A fifth tab showing each category's podium and exporting it as
SAGE-branded PNGs. Entirely **read-only and derived** — it writes nothing,
needs no auth, and adds no new sync surface or `sage-tools-api` dependency.
Everything it shows comes from `MATCHES` (the day's parsed match rows) and
`STANDINGS`, already loaded by the console for the other tabs.

### Podium derivation

`buildPodiums()` is the single pure function everything else consumes:
`(MATCHES, STANDINGS) -> [{ category, source, gold, silver, bronze,
bronzeWalkover, warning }]`, ordered to match Standings' own category order.

For each category:

1. **Find the decisive Final.** A Final played as a series (twice-to-beat,
   best-of-3 — see below) is decided by the game where a seat reaches its
   wins. Otherwise, among all matches tagged `F` for that category, the
   decisive one is the **resolved instance with the highest instance
   number**, where *resolved* means played, or decided by a bye walkover. A
   category with no `F` match at all (pure round robin) falls back to its
   top-3 standings rows instead, tagged `source: 'standings'`.
2. **Find the decisive Bronze**, the same way. A category with no Bronze
   match at all can still have its bronze decided in the standings: see
   below.
3. **Gold/silver** = winner/loser of the decisive Final. **Bronze** = winner
   of the decisive Bronze, or the standings walkover's pair.
4. Anything that can't be resolved cleanly — a tied score, or both sides of
   a match reading `BYE` — produces a `warning` naming the match number and
   **suppresses the whole category's podium** rather than guessing. Nothing
   here ever fabricates a winner.

#### Series finals: twice-to-beat and best-of-3

A Final played as a series of games carries the seat and the game in each
team code, `<prefix>_F_<seat>_(<game>)` (`IXD_F_1_(2)`); `seriesGameOf()`
reads them. Two games is **twice-to-beat**: seat 1, the round robin's #1,
needs one win and seat 2 needs two. Three is **best-of-3**: two wins each.
`walkSeries()` counts wins game by game; the game where a seat reaches its
wins decides the series, and a tied game counts for neither seat and raises
the usual tie warning. So a challenger who wins twice-to-beat's game 1 has
forced game 2, not won the Final: gold stays Pending until game 2.

Every game after the decider is never played. `unneededSeriesGames()`
returns them, and `computeFacilityProgress` leaves them out of the total, so
a twice-to-beat Final whose #1 won game 1 doesn't hold "matches left" at 1.
The public hub and the schedule board carry the same rule and grey such a
game out (`.not-needed`, with a "Not needed" tooltip and no label): its
finals matchup, its Match Finder ticket, which is never flagged **Next Up**,
and its schedule board cell.

#### Byes, and why they're detected the way they are

A twice-to-beat bracket often has no Bronze match to actually *play* — third
place is already determined by the bracket structure. Rather than special-
case that in code, it's encoded in the data: the operator types `BYE` into
the opponent's team code or either player-name cell, making the real side a
winner by walkover. `matchByeSide(m)` checks **all three cells** (both team
codes, both pairs of player names) — deliberately, since the sheet
autofills these by formula, so whichever cell the operator overrides has to
be the one that's checked. A bye match is decisive **without scores** —
`rowsToMatches()` sets `played` from the score columns, so a bye match has
`played === false`, and the podium derivation explicitly does not gate on
`played` when a bye side is present.

A bye-decided bronze shows its medalist like any other, with no score and
no label (the Awards tab and its images never write "walkover"), and the bye
side is never rendered as a person anywhere.

**A bronze decided in the standings alone.** A workbook can also leave the
Bronze off `SCHEDULE` entirely, so no Bronze match reaches `CSV`, and set
it up only in `STANDINGSCSV`: one bronze slot holds the round robin's #3,
the other reads `BYE`. `standingsBronzeWalkover()` recognises that shape
(exactly one real `B` row and one `BYE` row). When the category has no
Bronze match at all, Awards gives bronze to that pair once its names are
in, and Standings drops the Bronze
block the same as for a bye-decided match.

#### Byes must not surface where a match looks playable

A bye is never a real match, so it's filtered out of every surface that
presents match data as something to watch or play, not just the podium:

| Surface | What's filtered |
| --- | --- |
| Standings | Any row whose team code/player1 is `BYE`; if a category's *decisive* Bronze was bye-decided, or its bronze slots are a pair and a `BYE` with no Bronze match, the whole Bronze block is dropped (not just that row) — a one-sided "Bronze Battle" showing one team with no opponent is worse than showing nothing |
| Live Matches | Bye matches excluded from court grouping entirely |
| Match Finder | A team's own bye match never appears in their schedule, and can never be flagged "Next Up" |
| Team index / autocomplete | `BYE` is never indexed as a searchable name |

### The team type

`"team"` is for events of named teams meeting in four-match matchups (spec:
`sage-docs/docs/specs/.../pickledrive-club-anniversary-team-tournament-spec.md`). It is additive: every
change to a shared function is either inside a `CURRENT_TYPE === 'team'`
branch or an extra parsed field nothing else reads, so `dual-meet` and
`standard` are unaffected by it. The team logic lives in one bannered
block (`// ---- team type (...) ----`, just below `pairCell`), ported from the
event page's `index.html` so the two agree on every rule.

Data comes only from the two published CSVs. `rowsToMatches` reads a
`matchUp` column and `rowsToStandings` reads `teamName`, `totalPoints` and
`totalOpponentPoints` when present; matches are grouped into matchups by
`matchUp` alone. `teamMatchupResult` decides a matchup on total points (equal
points is a tie), and `sideLabel` / `teamNameOf` / `baseTeamOf` turn a code
such as `SF-A_2` into a team name and letter, reading a playoff slot as its
team once an operator has typed the letter into the workbook. `STAGES`
maps a playoff side's prefix to its label and order: `QF` Quarterfinal, `SF`
Semifinal, `Br` Bronze, `Fi` Final. A team is marked **Advances** when its
letter fills any playoff slot, which with quarterfinals means the eight
`QF-*` qualifiers.

Functions that branch on `team`:

| Function | Team behaviour |
| --- | --- |
| `selectEvent` | Accepts `type: "team"`; resets the team state |
| `parseCode` | Returns `{ club: null, category: null, rest }` so no shared caller breaks |
| `renderStandings` | Calls `renderTeamStandings()` before any category code runs; the category toggle bar and filter stay hidden. Each bracket table is ordered by `teamRankBracket`: points scored, quotient, head-to-head points, pair wins |
| `liveTableRowsHTML` | `teamLiveTableRowsHTML`: both team names, stage and pair label, running matchup score |
| `rebuildTeamIndex`, `resolveTeam`, `runSearch`, `renderAutocomplete`, `selectAcItem`, `renderIntro` | The team-and-player index, its search and the intro's team list; a saved search is `team:<letter>` or `player:<name>`, never a playoff code |
| `renderAwards` | `buildTeamPodium()` instead of `buildPodiums()` |
| `categoryLabel` | `'__team__'` reads *Team Championship* |
| `medalRowHTML`, `awardsCardHTML` | Show the team name, with the roster under it |
| `drawPodiumCard`, `renderWholeTournamentCard` | Draw the team name with the roster beneath (`wrapCanvasList`, breaking only between names); the podium card's fixed-height row uses `fitRosterLines` (24 → 20 → 17px, then "+N more") |

`buildTeamPodium()` returns one podium in the usual shape. Gold is the Final
matchup's winner, silver its loser and bronze the Bronze matchup's winner; each
is `null` (Pending) until that matchup is final. A tied Final or Bronze sets a
`warning` naming the matchup's lowest match number and leaves all three
placings null. `computeOverallChampion` still returns `null` outside a dual
meet.

On localhost, `?fixture=<name>` loads `/_fixtures/config.json` as the registry
and `/_fixtures/<eventKey>/<name>.json` as the snapshot, for testing without
publishing anything; the hostname check keeps it inert on the live site. The
console keeps the house theme — only the event pages take a per-event palette.

### Overall Champion (dual-meet only)

Deliberately **the same metric as Standings' own club win-total
summary** — total wins across Round Robin rows only, per club — not a
podium-derived count (e.g. gold medals), so the two tabs can never disagree
about who's ahead. A tie is shown explicitly as "Tied," never resolved by an
invented tiebreak.

### Image export — Canvas 2D, no library

`tools/control-center.html` has zero JS dependencies (Google Fonts is the
only external resource), so the export draws directly with the Canvas 2D
API rather than pulling in `html2canvas` or a similar library. Cost: the
card design lives in drawing code, not CSS.

**Per-category card:** fixed 1080×1350 logical canvas, rendered at 2× (with
a fallback to 1× if that would exceed a safe canvas-area budget — iOS
Safari caps total canvas area near 16.7M pixels). Each medal row fills its
whole box in the medal tint once decided (dark navy text — checked for
contrast: ~7.3:1 on gold, ~8.9:1 on silver, ~4.7:1 on bronze); an undecided
placing stays on the plain navy panel, so the row's own color signals
decided-vs-not.

**Whole-tournament sheet:** every category in a grid (column count picked
by category count, to keep the image from becoming an ever-taller single
column) rather than one row per category. Nothing is ever truncated —
category names, player names, and club labels all word-wrap; `wrapCanvasText`
force-breaks at the character level for the rare single "word" (a long
hyphenated surname, say) that's wider than its column on its own, so no
line can ever overflow its box.

**The measure-then-draw pattern.** Both the per-card and whole-tournament
drawing functions take a `draw` boolean and run through the *exact same*
layout logic either way — called once with `draw: false` (nothing painted,
just wrapping/line-counting to get a height), then again with `draw: true`
at the real position once the canvas is correctly sized. This is what
makes it possible for content to be genuinely dynamic-height (wrap to
however many lines it needs) without measuring and drawing ever disagreeing
about where something lands — they're driven by identical code, not two
hand-synced implementations.

Fonts are explicitly `document.fonts.load()`-ed and awaited before the
first stroke — `ctx.font` silently falls back to a system font if the
family hasn't been requested by *some* DOM node yet, with no error to
catch, and this is the single most likely way a cold-load export comes out
wrong while the on-screen tab looks fine.

---
**Features:** [Control Center](../features/control-center.md) · [Awards tab usage](../features/control-center.md#awards)
**Specs:** [`match-control-console-spec.md`](../specs/implemented/match-control-console-spec.md) and [`awards-podium-tab-spec.md`](../specs/implemented/awards-podium-tab-spec.md) (full build history, acceptance checklists — both call the console Match Control).
