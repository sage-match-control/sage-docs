# Scorer page

The scorer page is a per-event page, `events/<event-key>/scorer.html`, made from
the template `_templates/scorer/scorer.html`. It lets scorer staff, who hold a
scorer link and never see Control Center, enter scores for any venue of one day.
Usage: [Scorer page](../features/scorer-page.md). The write it makes is
described in [Sync pipeline § Score entry writes](sync-pipeline.md#score-entry-writes)
and the token it carries in [Auth § Scorer tokens](auth.md#scorer-tokens).

## The template and its tokens

Like the attendance desk page, the template is copied into an event's folder with
two tokens replaced, `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`, for an event whose
`events.json` entry has `"scoreEntry": "links"`. It is linked from nowhere public:
Mission Control's **Issue scorer link** produces the address,
`https://sage-match-control.github.io/events/<key>/scorer?scorer=<token>`. The
page's head, palette and header come from the attendance desk page. The
instantiation step is in `_templates/CLAUDE.md`.

The file is a shell of the [site engine](site-engine.md): its markup, its
palette and one settings script (`EVENT_KEY`, `LIVE_BASE_URL`) that calls
`mountScorer` in `lib/v1/apps/scorer.js`. The page's own code is that app; the
score dialog is `lib/v1/views/score-dialog.js`, and the page's styles are
`lib/v1/css/scorer.css` and `lib/v1/css/score-dialog.css`.

## Start-up

`scorerStart` runs these checks in order, and each failure puts one sentence in
the message line and stops:

1. **The token.** A `?scorer=<token>` in the address is stored in
   `localStorage` under `sage.scorer.<event-key>` and removed from the address
   bar with `history.replaceState`, other parameters kept, so it does not show in
   screenshots or reshared links. A newer link replaces an older one. The token is
   read back from storage, so a reload keeps working.
2. **Decoding**, for display only: the payload must carry `scope: "score-desk"`, a
   string `day` and a numeric `exp`. The API decides what the token may do. No
   token or a bad one, or one past `exp`, stops the page.
3. **The registry.** `events.json` is fetched (a fixture reads
   `/_fixtures/config.json`). The event must exist with `scoreEntry: "links"`;
   `"console"` or no setting reads as "Scorer links are stopped". It is checked on
   every load. The token's day must be one of the event's days.
4. **Venues.** The day's facilities that have a `sheetId`. The title line reads
   `<event> · <day> · <venue>` and follows the chosen venue.

A timer checks `exp` every minute; once it passes, the page closes the dialog,
empties the list, stops its polling and shows the expired message.

## Data

The page wires the live channel as the event pages do
(`lib/v1/data/live-channel.js`): `createLiveChannel`, a `fetchDaySnapshot` that prefers the pushed snapshot and
otherwise fetches `<event>/data/<day>.json` from GitHub Pages and keeps the newer
copy, `liveChannel.follow(EVENT_KEY, day)`, a 10-second poll, and a pause while
the tab is hidden. See [Sync pipeline § Pages](sync-pipeline.md#pages).

`loadData` keeps the whole snapshot. One snapshot holds every venue of the day, so
switching venue rebuilds the match list from `lastSnapshot` with no request: the
chosen facility's CSV goes through `rowsToMatches`, BYEs are dropped, and a team
event's names come from that facility's standings (`sideLabel` over the
standings rows by team code): a playoff side whose slot holds a team letter
shows that team's name, an open one reads "Seed n · TBD". The dialog's sub
line for a team match is the stage and the pair, from the event's
`display.pairs` (for example "Bracket 2 · Mixed Doubles 1"), not the matchup key. A 404 reads "schedule isn't
published yet" and a failure reads "Retrying"; both keep polling. Each rebuild
redraws the list and calls the dialog's `refresh()`.

## Venue picker, filters, list

The picker is rebuilt on each change and always shows its buttons when there is
more than one venue, with the chosen one `aria-pressed`. The choice is stored
under `sage.scorer.<event-key>.facility`; a remembered venue that is not on this
day counts as none. The filters (search, court, **Hide scored**) are stored per
venue under `sage.scorer.<event-key>.filters.<venue>` and restored on a switch.
The court buttons are rebuilt only when the set of courts or the selection
changes, so typing in the search box keeps its focus across a poll. Cards are
redrawn on every snapshot, which is fine because the dialog is modal.

## The engine's rules

The CSV parsers, the played/BYE and series rules, the API's error wording and
`escapeHtml` are the engine's (`lib/v1/domain/`, `lib/v1/data/api.js`,
`lib/v1/views/html.js`), the same modules Control Center and the event pages
import, so there is nothing to keep in step by hand. The BYE and played rules
also exist in `sage-tools-api`'s `facilityCompletion.mjs`; the parity test in
`_tests/unit/parity-server.test.mjs` fails if the two disagree.

## Testing locally

Serve the site folder with any static server. A test copy of the template with
the tokens replaced, named `scorer-beta.html`, is ignored by git (`**/*beta*`).
`?fixture=<name>` on localhost reads `/_fixtures/`, simulates saves and
`&scoreConflict=1` exercises the conflict panel. Take the link from Control
Center's **Issue scorer link** on a fixture, which builds a token the page can
decode.
