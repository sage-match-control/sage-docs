# Spec — Tournament Hub player experience

> **Status: not started — proposals for the owner to choose from.** Written
> 2026-10-10 against `lib/v1` and the three event-site templates in
> `sage-match-control.github.io` as of commit `d68acf8`. Each proposal in §4–§11
> stands alone: it can be kept or dropped without changing the others, except
> where its **Depends on** line says so. §3 is the table to mark up. Nothing
> here is built.

Make the main thing a player opens the [Tournament Hub](../../features/tournament-hub.md)
for, *where and when do I play next?*, quicker on a phone. Nothing moves
off the page and no new sections are added. The Hub keeps its three tabs (four at a team event).

---

## 1. Why

On 2026-10-10 the Hub was compared with another organiser's player portal for
a one-day event (Purple Pickle XP, "Powered by Qourts"). It is one 1.8 MB HTML file
reading a Supabase table every 15 seconds. That portal does three things better on
a phone:

- It opens on the player's **next match** (court and time in large type) once the
  player has picked their name, on every later visit too.
- Its section tabs are a **bar along the bottom** of the screen, always one tap
  away.
- The first screen is **the task**, not the event's banner.

Behind those it has more sections than it needs: eight on a computer, five plus a
menu on a phone, several overlapping. It also has a sign-in-style welcome
screen in front of everything, and it shows organiser jargon (`P001`, `A-A1`,
`PE · PC · PQ`). The Hub is already simpler, and these proposals keep it so.
They take the three things above and leave the rest (§12).

### 1.1 What the Hub does today, measured

The standard template on the Piggleball fixture, mid-day, on a 375 × 812 phone
screen (the harness's `phone` size):

| | Top of it, from the top of the page |
| --- | --- |
| The tabs (Match Finder, Live Matches, Standings) | 346 px |
| The search box | 535 px |
| A pair's results, after a search | 755 px: 57 px of it on the first screen |

- **A search shows nothing new on the first screen.** Picking a suggestion, tapping
  **Find matches** or pressing Enter runs the search, and the page stays where it
  is. The results begin 57 px above the bottom edge, so a player sees the box
  change and has to scroll to find out whether anything happened.
- **Find matches leaves the suggestions open** over the results: the page closes
  them on a click outside `.autocomplete`, and the button is inside it. On a
  phone, Enter leaves the keyboard up as well.
- **The last search comes back on reload** (`<event>.search` in the browser's
  storage), with the same layout: the results are below the first screen again.
- **A pair's results list every match** in schedule order, with a **Next Up** pill on
  one ticket. The player scans the list for it.
- **Each side of a ticket starts with its team code** (`ND_5`, `ND_QF_4`), in
  Control Center's small muted style, above the names.

## 2. Scope

- **The engine Hub only:** `lib/v1` (`apps/hub.js`, `views/finder.js`,
  `views/ticket.js`, `views/standings.js` and their CSS) and the three templates'
  `index.html`, so every event made from a template from then on. Every event
  page built so far, under `events/` and `events/archives/`, is self-contained
  and frozen and imports nothing from `lib/v1`. They do not change.
- **Control Center does not change.** It shares the finder and the ticket, so every
  change it should not get is an option the Hub passes, off by default
  ([site engine](../../technical/site-engine.md#changing-the-engine): "add an
  option rather than a flag").
- **The schedule board, scorer page and desk page do not change.**
- No new data. Every proposal reads what the snapshot and `events.json` already
  carry, except P7's one optional setting.

## 3. The proposals, to mark up

| # | Proposal | Size | Recommendation | Keep? |
| --- | --- | --- | --- | --- |
| [P1](#4-p1-show-the-results-after-a-search) | Show the results after a search | Small | Keep | |
| [P2](#5-p2-a-shorter-banner-on-phones) | A shorter banner on phones | Small | Keep | |
| [P3](#6-p3-your-next-match-card) | "Your next match" card | Medium | Keep | |
| [P4](#7-p4-names-not-team-codes-on-the-hubs-tickets) | Names, not team codes, on the Hub's tickets | Small | Keep | |
| [P5](#8-p5-the-tabs-along-the-bottom-on-phones) | The tabs along the bottom on phones | Small–medium | Try on a real phone first | |
| [P6](#9-p6-say-when-standings-are-provisional) | Say when standings are provisional | Small | Keep | |
| [P7](#10-p7-a-rules-link) | A rules link | Small | Keep if organisers supply rules | |
| [P8](#11-p8-follow-more-than-one-pair) | Follow more than one pair | Medium | Drop for now | |

P1–P4 together give the Hub the other portal's best feature (open, see your next
match) without its welcome screen. They are the core of this spec. P5–P8 are
independent extras.

---

## 4. P1. Show the results after a search

**Problem.** §1.1: a search changes nothing on a phone's first screen, the
suggestions can stay open over the results, and the keyboard stays up.

**Proposal.** When a person runs a search (a suggestion picked, **Find matches**,
Enter, or a team chip at a team event):

1. Close the suggestions.
2. On a phone (below 900 px, where the search box is already pinned to the top
   of the screen), take the focus out of the box so the keyboard closes.
3. Scroll the page so the results' first line sits just below the pinned search
   box. The scroll is instant under *reduce motion*, smooth otherwise.

Nothing scrolls when the Hub redraws the same results for new data (every push
and every poll), when the box is cleared, or on a computer screen. A
**restored** search on page load is P3's open decision D3.

**Where.** `views/finder.js`: a new `createFinder` option, `revealResults`
(default `false`), applied in the event handlers rather than in `run()`, because
`refresh()` calls `run()` on every data update. `apps/hub.js` passes `true`.
The team event's finder (`views/teams.js`, the finder's `delegate`) goes through
the same handlers and gets it too. Closing the suggestions on **Find matches**
is a fix and applies everywhere, Control Center included.

**Tests.** A `finder.test.mjs` case per trigger (what scrolls, and that a refresh
does not). The harness's phone `finder-search` cases change (accepted
differences, each with its reconciliations row).

**Open decisions.** None.

## 5. P2. A shorter banner on phones

**Problem.** The banner (event logo, tagline, date and venue, title, headline,
"Find your pair's full match schedule…", tabs, divider, sync line) fills the
first 500 px of a phone screen, so the search box starts at 535 px.

**Proposal.** On a phone, once a day has loaded, drop the explanatory subtitle and
the divider and tighten the rest. The logos, the date and venue line and the
title stay, smaller. The target is the search box's top **within the first
350 px**, so after P1 the first result ticket is on the first screen. The banner
before a day loads (the day picker, a "Loading…" or error message) and every
computer-sized screen stay as they are.

**Where.** `css/hub.css`, under a phone media query and a class `apps/hub.js`
puts on `.hero` once the tabs show (new class names only). Each template's `THEME`
block is untouched, so an event's look carries over.

**Tests.** Every phone Hub case's screenshot changes: one accepted difference per
template.

**Open decisions.**

- **D1.** How much of the banner stays: the title at a smaller size (proposed), or
  only the logos and title, or the whole banner collapsing to one line as the
  page scrolls (more work, and it moves under the reader's thumb).

## 6. P3. "Your next match" card

**Problem.** A pair's results are every match of the day. The one the player
wants is marked by a small **Next Up** pill somewhere in the list.

**Proposal.** Above the ticket list, one card that answers the question. Its
state follows the rules the tickets already use (domain `matches.js`,
`series.js`, `byes.js`):

| State | When | The card shows |
| --- | --- | --- |
| **Playing now** | One of the pair's matches is on a court (`court` is set) | *Playing now · Court 3*, the opponent, the score if one is in |
| **Next match** | Otherwise, the first match not played, not live, not a game that won't be needed and not a BYE (the **Next Up** ticket's rule) | *Next match · 10:15 AM · Court 1*, the opponent's names, the round (*Round robin*, *Semifinal*…) |
| **Waiting on a result** | The next match is a playoff slot whose opponent isn't decided | As *Next match*, with the opponent reading *To be decided* |
| **All done** | Every match played | *All 7 matches played*, the pair's wins and losses |

The court in *Next match* is the scheduled `CourtAssignment`. When the match is
called to a court it becomes *Playing now* with that court.

The list underneath stays as it is, **Next Up** pill included. The card is only
the summary.

With P1, a search lands on this card. With the saved search (§1.1), so does a
returning visit, which is what the other portal's "My Games" gives its players,
with no welcome screen and no extra step.

**Where.** `views/finder.js`: a new export, `nextMatchCardHTML(team, matches,
model)`, called from `pairResultsHTML`, and its styles in `css/finder.css`. It
is a pure function, unit-tested on the fixtures' `pre`, `mid` and `final`
states. Control Center does not show it (an option, as in §2).

**Depends on.** Nothing; it is most useful with P1 and P2.

**Open decisions.**

- **D2.** A team event: the same card for the team's next **matchup** (both time
  slots and courts) and for a player's next match within it, or standard and
  dual-meet events first and the team card later (proposed).
- **D3.** On a page load with a saved search, scroll to the card (as the other
  portal opens on its *My Games* screen) or leave the page at the top (today).
  Scrolling is proposed, but only when the saved tab is Match Finder.

## 7. P4. Names, not team codes, on the Hub's tickets

**Problem.** Every ticket side begins with the pair's workbook code (`ND_5`,
`ND_QF_4`). It means nothing to a player, and the round is already on the ticket
as its own pill. The team event's Hub already hides its codes (its
`teamLetters: false`).

**Proposal.** A ticket option, `showCodes` (default `true`), that the Hub sets to
`false`. A side with a player's name shows the names alone, with the **You**
badge beside the first name. A side with no names yet (an undecided playoff
slot) keeps the code, since nothing else on the ticket says which slot it
is. Control Center keeps its codes. Standings keep the code for a round-robin
pair whose names aren't in the sheet yet, as the Hub's page describes.

**Where.** `views/ticket.js`, `css/ticket.css`, the option passed through
`createFinder`'s `ticket` and `apps/hub.js`.

**Tests.** `ticket.test.mjs` cases for both values and the no-names side. Harness
accepted differences on the Hub's finder cases.

**Open decisions.** None.

## 8. P5. The tabs along the bottom on phones

**Problem.** After scrolling down a long list (the day's matches, a category's
standings), switching tab means scrolling back up to the banner.

**Proposal.** On a phone, the tab row becomes a bar fixed to the bottom of the
screen, inside the phone's safe area (above the home indicator). It has the same
tabs and labels, text only. It shows when the tabs show today, once a day has
loaded. The page gains bottom padding the bar's height, so the footer is never
under it. Computer screens are unchanged.

**Where.** `css/hub.css` (new rules for the existing `.view-tabs` under a phone
query, plus a new class so nothing a page uses is renamed). Each template's
viewport tag adds `viewport-fit=cover`, which the safe-area inset needs.

**Cost.** The pinned search box at the top and the bar at the bottom together
take about 200 px of an 812 px screen while Match Finder is open. iOS Safari's
own bottom toolbar moves as the page scrolls, and a fixed bar has to be seen
on a real iPhone and Android phone before an event, not only in the
harness.

**Open decisions.**

- **D4.** Keep the tabs in the banner as well (two ways to the same thing), or
  move them (proposed, with P2).
- **D5.** Icons above the labels, as the other portal has: proposed **no**,
  since it means an icon set in every shell for little gain.

## 9. P6. Say when standings are provisional

**Problem.** The Hub ranks a round-robin table by wins, then head-to-head, then
quotient. The workbook picks who advances by wins, then quotient, so on a
head-to-head tie the two can differ. The [Hub's page](../../features/tournament-hub.md)
says so. The Hub itself does not, and a table mid-round-robin reads as final.

**Proposal.** Text only; the Hub never works out who would advance.

- Each round-robin table's header gets a status line: *In progress · 6 of 10
  matches played* (the played rule of `matches.js`, BYEs left out), then *Round
  robin complete*.
- One line under the tables: *Ranked by wins, head-to-head, then quotient. The
  organisers confirm who advances.*
- A playoff slot still reads **TBD** until the workbook fills it (today's
  behaviour), so the bracket is never a projection.

**Where.** `views/standings.js`, `css/standings.css`. Control Center's Standings
use the same builder and get it too, which is proposed: operators answer this
question at the desk.

**Tests.** `standings.test.mjs` for the counts on `pre`, `mid` and `final`. Harness
accepted differences on every Standings case.

**Open decisions.**

- **D6.** Team events: their Standings are ranked by the organiser's
  tiebreakers in the workbook's order and seeded by hand, so only the status line
  applies. Include it (proposed) or leave team Standings alone.

## 10. P7. A rules link

**Problem.** An organiser's rules (scoring, time limits, tiebreakers) are handed
out on paper or in a group chat, and the Hub has nowhere to point at them.

**Proposal.** An optional `mountHub` setting, `rulesUrl` (default `''`, meaning no
link). When set, a **Rules** link shows under the tabs and in the footer and opens
the file in a new tab. The rules file goes in the event's own folder
(`events/<event-key>/assets/rules.pdf`), like its QR image. The Hub does not
render the rules or give them a tab.

**Where.** `apps/hub.js` (a new known setting: an unknown setting is already an
error, so this is the compatible way to add one), the templates' markup and the
instantiation runbook (`_templates/CLAUDE.md` §2), as an optional step.

**Open decisions.**

- **D7.** A shell setting (proposed: the file is in the site repo, beside the page)
  or a `display.rulesUrl` in `events.json` (changeable without touching the site
  repo, but a site file named from the data repo).

## 11. P8. Follow more than one pair

**Problem.** A player entered in two categories, or a parent with two children
playing, searches each time. The Hub remembers one search.

**Proposal.** Match Finder keeps up to four followed pairs on the device, as
chips above the results, each opening its results and P3 card.

**Recommendation: drop for now.** It is the other portal's most complex screen.
It adds a list to manage (add, remove, a limit) and the saved-search
format changes. Most players are in one category on a day. Revisit if
organisers report multi-category players asking for it.

---

## 12. Not taken from the other portal

| Their feature | Why not |
| --- | --- |
| A welcome screen asking "player or spectator" before anything shows | An extra step for every spectator. P3 gives players the same result after one search |
| The event poster as the home screen | The banner already carries the event's identity; P2 makes it smaller, not larger |
| Eight sections (Home, Overview, Schedule, Standings, Live Courts, Championship, My Matches, Rules) | They overlap. The Hub's three tabs cover the same ground |
| A venue TV view inside the player portal | The [schedule board](../../features/schedule-board.md) is a separate, unlisted page on purpose |
| A qualification projection ("Live projection") | The workbook decides who advances; P6 says so instead of guessing |
| Polling a database row every 15 s | The Hub's live push is faster and archives every publish |
| A 1.8 MB page with its images inlined | The Hub's shell loads the engine's modules and the event's own images as separate files, and nothing here adds an image |

## 13. Building it

- **Engine rules apply** ([site engine](../../technical/site-engine.md)): only
  `lib/v1` is edited, compatibly (new options, exports and classes, nothing
  removed or renamed), and a module only starts using another's new export in a
  later push.
- **Each proposal is its own push**, in the order of §3. Each runs
  `npm run verify` and adds its accepted differences to
  `_tests/engine/accepted.mjs`, each with a new row in the site engine spec's
  reconciliations table (§12).
- **Not within three days of an event**, as for any push to the site's `main`.
- **Docs, with each one built:** [Tournament Hub](../../features/tournament-hub.md)
  (Match Finder, Standings), [site engine](../../technical/site-engine.md) for new
  options and settings, the runbook for P7, and this spec's status line and a
  divergences section once anything departs from what is written here.

## 14. Open decisions, in one place

| # | Proposal | Decision | Proposed |
| --- | --- | --- | --- |
| D1 | P2 | How much of the banner stays on a phone | Logos, date and venue, title (smaller) |
| D2 | P3 | Team events' next-matchup card now or later | Later |
| D3 | P3 | Scroll to the card when a saved search is restored on load | Yes, when the saved tab is Match Finder |
| D4 | P5 | Tabs in the banner as well as the bottom bar | Move them |
| D5 | P5 | Icons in the bottom bar | No |
| D6 | P6 | Team Standings get the status line | Yes |
| D7 | P7 | Rules link as a shell setting or in `events.json` | Shell setting |

---
**Features:** [Tournament Hub](../../features/tournament-hub.md) · **Technical:** [site engine](../../technical/site-engine.md)
