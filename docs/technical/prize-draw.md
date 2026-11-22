# Prize draw (Lucky Dink)

`tools/prize-draw.html` is one self-contained page: an inline `<style>` and one `<script type="module">`.
Its rules are the engine module `lib/v1/domain/prize-draw.js`; the page imports `platform.js`,
`data/registry.js`, `data/snapshot.js`, `data/api.js`, `data/operator-session.js` and that module, by
root-absolute paths, and declares nothing the module exports. Control Center's **Open prize draw** tile and the
tools roster link to it. The same file is the display window, with `?display=1`.

## The rule (`domain/prize-draw.js`)

- **Entries.** One per line, white space collapsed, the same person (by `attendanceKey`) dropped. 2 to 1000
  entries, each at most 40 characters, with no `|`.
- **Draw code and id.** 8 characters from `23456789ABCDEFGHJKMNPQRSTVWXYZ`, from `crypto.getRandomValues`
  at Lock; the draw id is made the same way. Nobody types one.
- **Fingerprint.** SHA-256, lowercase hex, of the entries joined by `\n`.
- **Spin n's winners.** Each entrant still in gets the key SHA-256(`code|n|entry`); a spin that draws k winners
  takes the k smallest keys, smallest first. The prize never enters the hash. `Math.random()` only fills the
  reel's cosmetic rows and moves the jar's balls.
- **Prize slots.** One per unit, id `<row id>:<k>`. A spin draws the first slot not yet won, or every unwon slot
  of its row when the row is **Draw all at once**. A slot is won by a pick with result `won`; `redrawn` frees it.
- **Who is in.** A winner drawn again is always out; a prize winner is out under `remove-winners`, in under
  `keep-winners`.
- **The check** (`verifyDraw`) replays every spin and compares winners place by place.

## State

One object in `localStorage['sage.prizeDraw']` (version 2): title, event, day, facility, names, rule,
`showTickets`, `style`, `prizeRows` (`{ id, name, quantity, together }`, ids from a counter, `rowSeq`), then,
once locked, `drawId`, `entries`, `seed`, `fingerprint`, `lockedAt`, `picks` (`spin`, `place`, `slotId`, `prize`,
`ticket`, `winner`, `result`, `drawnAt`, `recorded`) and `parts` (earlier draws after **Edit list**). The display
window never reads or writes it.

## The reel and the jar

- **Reel.** Five rows (64px, 52px on a phone, 96px on the big screen and display), the winner's row third; 46
  filler rows, then the winner, over 4.8 s (`cubic-bezier(.08,.72,.12,1)`); with reduced motion the result
  shows after 400 ms.
- **Jar.** A 640 by 400 canvas, drum centre (190, 190), radius 150, tray at y = 372. Ball radius
  `max(6, min(19, 19·√(40/n)))`; labels up to 40 balls; the jar is unavailable above 300 names. Gravity .38,
  damping .996, wall restitution .75, drum spin up to .07 while mixing (2.8 s), then each winner leaves 0.6 s
  after the last, drops 0.7 s and rolls 1.1 s along the tray. Above 120 balls collisions use a grid.

## The display window

Both windows share `BroadcastChannel('sage.prize-draw-display')`. The display sends `hello`, `here` (every
2 s), `played` (`spin`) and `bye`; the controls send `scene` (title, event line, style, tickets, rows, entrants,
last reveal, next prize), `play` (`spin`, `style`, `entrants`, `picks`) and `close`. A display counts as connected
until 5 s pass without `hello` or `here`. The results, **Draw again** and the exports stay on the controls.

## Sign-in and recording

The page asks other tabs of this browser for the operator sign-in over `data/operator-session.js`
(`answer: false`), keeps it in memory only, and uses it for the people list
(`GET /v3/days/{day}/facilities/{facility}/people`, **Load checked in**) and for recording.

Each spin and each **Draw again** is sent in order, one at a time, to
`PUT /v3/days/{day}/facilities/{facility}/prize-draws/{drawId}/spins/{spin}`; a spin's picks are marked
`recorded` when the answer arrives, so the queue survives a reload. A failure stops the queue (**Retry**
resumes it); a `409` stops it without retry. Spinning never waits. **Past draws** reads
`GET …/prize-draws`, checks a draw (list against fingerprint, then `verifyDraw`) and continues one.

### The `DRAWS` tab

One row per winner, columns A to O: `drawId`, `title`, `spin`, `place`, `prize`, `ticket`, `winner`, `result`
(`won` or `redrawn`), `drawnAt` (the API's clock), `seed`, `fingerprint`, `rule`, `entries`, then, on spin 1,
place 1 only, `list` (every entry, one per line in the cell) and `continues`. Rows are written whole, as typed
(RAW), by the API's service account; the API creates the tab on the first spin and refuses another header row
(`409 draws_layout`). `src/draws/` holds `domain/drawsTab.mjs`, `domain/drawRequest.mjs`, `DrawService.mjs` and
the routes; nothing is published, and the routes take an operator token only. See
[API](api.md).

## Tests

`_tests/unit/domain/prize-draw.test.mjs` (the rule, with pinned winners) and
`_tests/unit/tools/prize-draw-page.test.mjs` (one module script, no copy of the rules, every named element) in
the site repo; `test/unit/draws/`, `test/integration/draw-routes.test.mjs` and the
`SheetsClient` tests in the API.

---
**Features counterpart:** [Prize draw](../features/prize-draw.md).
