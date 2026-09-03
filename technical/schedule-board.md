# Schedule board

A standalone page per event (`events/<event-key>/schedule.html`, public URL
`/events/<event-key>/schedule`) — a wall display for the venue: courts as
columns, time slots as rows, one face-off card per match, color-coded by
category, live matches highlighted and results filling in as they're
played. Deliberately **unlisted** — nothing on the public site links to it;
Mission Control is its only entry point (see [Control
Center](control-center.md)).

## Why a separate page, not a tab

- **Opposite audience needs.** The public `index.html` is a phone lookup
  tool for a player finding their own match — hero, search, day picker, tab
  bar. The board is a passive wall display where every one of those costs a
  match row; its masthead is 48px.
- **Theme collision.** This page uses the SAGE *tools* theme
  (`scoresheet-generator.html` / `tournament-calculator.html`'s palette),
  not the event pages' dark theme — both define `--paper`, `--ink`, `.cell`,
  `.legend`, `.dot`, so sharing a document risks cross-contaminating them.
- **Weight.** The event `index.html` is already substantial; the board is a
  fraction of that standalone. Folding it in would tax every phone visitor
  for a view they'll never open.

It needs almost none of the shared config surface (`DIVISIONS`, `EVENTS`,
`CLUBS`, `STAGE_META`, search, standings, or bracket logic) — just the
event/day key, the data-repo location, and a category color map.

## Match cell

Symmetric: team 1 left, score centered, team 2 right, mirrored. Player names
stack two lines, truncating with an ellipsis. Club is read **per side** from
the team-code prefix, never assumed by position — semifinals in a dual-meet
bracket are intra-club (e.g. `PNF_LIWD_SF_1` vs. `PNF_LIWD_SF_2`), so a cell
can legitimately be the same club on both sides; hardcoding "club A always
left" would be wrong.

A **stage pill** beside the category chip reads the ladder position off the
same `roundKeyword()` classification the rest of the site uses — lifted
verbatim, not reimplemented, so this page classifies a code exactly as
Standings and Live Matches do. Pool play renders as a quiet outline `RR`
(it's the large majority of cells); playoff rounds (`R16`/`QF`/`SF`/
`BRONZE`/`FINAL`) invert to solid navy so they're findable at a glance
across a wall of cells.

Three states: **upcoming** (no scores, full opacity, `VS` centered),
**live** (green ring + glow, `● COURT n` pill, driven by the `court` column
being non-empty), **completed** (dimmed to 84% opacity, shows the score —
84% specifically, since a lower value pushed the darkest category's text
under the 4.5:1 AA contrast floor).

## Court filter — state lives in the URL

`?courts=1-5` / `?courts=1,2,7-9`. This is the point: a two-screen venue
setup bookmarks one URL per screen (`?courts=1-5` on screen A,
`?courts=6-9` on screen B), and each boots straight into its range after a
refresh or power cycle with nobody touching the machine. Clicking a court
pill rewrites the URL live (`history.replaceState`). The **legend** follows
the filter (it keys what's visible); the **match count** does not — it's
event-wide on every screen, so the same label doesn't mean different things
on two split screens; only the scope text changes.

A collapsible header (chevron, state also carried in the URL as
`?compact=1`, composable with `?courts=`) hides operator chrome — the court
filter and the PDF button — while keeping the color legend, since a viewer
still needs that to read the board at all.

## Category colors are organizer-owned, not invented

Read directly off the color-coded SCHEDULE tab of the source spreadsheet so
the wall display and the organizer's own printed schedule agree. These are
**not exportable** — cell fills are formatting, absent from both the CSV
and gviz exports, and Sheets paints its own grid to a `<canvas>` with
nothing recoverable from the DOM — so they're recovered by sampling
rendered pixels and cross-checked, and **must be re-read by hand if the
organizer ever recolors the sheet**; nothing detects that drift. Applied as
a 40% tint over white (the measured contrast ceiling for navy body text on
the darkest category) with the solid hue as a left rule; chip text color is
derived per-category from luminance rather than hardcoded, since the
palette spans light berry to near-black berry.

## Print / PDF export

A `PDF` button builds print-only pages and calls `window.print()` — the
browser's own "Save as PDF" covers the PDF ask, and the same output prints
on paper. Deliberately **not** a canvas render (unlike
`bracket-generator.html`'s export): re-implementing every face-off cell in
Canvas 2D and hand-chunking pages would be a second layout implementation
that drifts from the CSS the moment either changes, whereas printing gets
page breaking, paper size, and DPI from the browser for free, off the same
grid-rendering function the live board already uses.

Both axes (courts, time slots) can overflow a sheet independently, so
pagination chunks both and takes the cross product — courts as the outer
loop, so one court group's whole day lands on consecutive sheets, chunked
**balanced** (9 courts at 4/page → 3+3+3, never 4+4+1). `print-color-adjust:
exact` is required or the category tints — the whole point of the board —
print blank; the "completed" dimming is forced back to full opacity on
paper, since ink is cheap and a dimmed printed row just reads muddy.

## Data & live polling

Reads the same published snapshot the event pages already fetch. Court
placement is defensive: a valid, parseable `CourtAssignment` places
directly; anything blank, unparseable, or a duplicate falls to the first
free lane; a genuinely full slot **surfaces rather than drops** a match — a
`+n` badge on the time cell lists the overflow match numbers in a tooltip.
Row order must never be trusted as an implicit court assignment. It can
appear to work — row order often does line up with courts — and that is
exactly what makes it dangerous: an undocumented contract that breaks
silently the first time someone reorders a row.

Polls on the same interval and cache-busting convention as the rest of the
site (see [sync pipeline](sync-pipeline.md)); a failed poll leaves the last
good board on screen rather than showing an error — a wall display showing
stale data beats one showing an error message. Re-rendering preserves the
current court filter and scroll position.

---
**Features:** [The schedule board, for spectators/operators](../features/tournament-hub.md#at-the-venue-the-schedule-board) · [launching it from Mission Control](../features/control-center.md#mission-control)
**Spec:** `specs/schedule-screen-spec.md` in the site repo (full layout spec, pagination math, acceptance checklist).
