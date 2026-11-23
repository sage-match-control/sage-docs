# Spec — Tournament Hub player experience

> **Status: not started — the owner has picked what to build.** Written
> 2026-10-10 against `lib/v1` and the three event-site templates in
> `sage-match-control.github.io` as of commit `d68acf8`; P9–P17 added
> 2026-10-25 against commit `d7ed8ed`; revisited 2026-11-23 against commit
> `b89fda5`, where Match Finder's pair summary card and other changes cover parts
> of P3, P6, P12, P14 and P15 (§1.3). On 2026-11-23 the owner kept nine
> proposals and dropped eight (§3), had P2, P5 and P6 redone and approved the new
> designs, and answered every open decision (§23). P6's tie pills, added at the
> owner's request after that and redone to say how each tie was broken, are
> approved too. The spec is ready to build. Each proposal in §4–§20 stands
> alone, except where its **Depends on** line says so. Nothing here is built.

Make the two things a player opens the [Tournament Hub](../../features/tournament-hub.md)
for, *where and when do I play next?* and *where do we stand?*, quicker on a phone.
Nothing moves off the page and no new sections are added. The Hub keeps its three
tabs (four at a team event).

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
They take the three things above and leave the rest (§21).

### 1.1 What the Hub does today, measured

The standard template on the Piggleball fixture, mid-day, on a 375 × 812 phone
screen (the harness's `phone` size):

| | Top of it, from the top of the page |
| --- | --- |
| The tabs (Match Finder, Live Matches, Standings) | 346 px |
| The search box | 541 px |
| A pair's results, after a search | 763 px: the pair's name, 49 px of it on the first screen |
| The pair summary card (§1.3) | 845 px, 326 px tall |
| The pair's first ticket | 1,223 px |

Measured again on 2026-11-23 at `b89fda5`.

- **A search shows nothing new on the first screen.** Picking a suggestion, tapping
  **Find matches** or pressing Enter runs the search, and the page stays where it
  is. Only the top of the pair's name shows above the bottom edge, so a player sees
  the box change and has to scroll to find out whether anything happened.
- **Find matches leaves the suggestions open** over the results: the page closes
  them on a click outside `.autocomplete`, and the button is inside it. On a
  phone, Enter leaves the keyboard up as well.
- **The last search comes back on reload** (`<event>.search` in the browser's
  storage), with the same layout: the results are below the first screen again.
- **A pair's results list every match** in Schedule time order under the pair
  summary card, with a **Next Up** pill on one ticket. The card says where the pair
  stands, not when it plays next in the round robin. The player scans the list for
  the pill.
- **Each side of a ticket starts with its team code** (`ND_5`, `ND_QF_4`), in
  Control Center's small muted style, above the names. The ticket also has its
  match number as a tab in its corner and its category and round as pills.

### 1.2 A second comparison: Dinktator, 2026-10-25

On 2026-10-25 the Hub was compared with Dinktator's spectator pages for Paddle Up! vs
Bangers 4.0 (`pvo.dinktator.club/t/pvo`), a two-club, two-day event with the same
shape as a dual meet. Pages are drawn on the server and updated over a WebSocket.
Umpires score every match from their own devices. It has five pages, each with its
own address: Courts, Standings, Bracket, Schedule, and a page per court. Their
standings work better on a phone:

- **They open on a division.** A dropdown picks one, and the first is chosen already.
- **They explain themselves.** The rule for who advances is printed at the cut line.
  A pair placed by a tiebreak says so ("Tied on win–loss — placed by quotient"). A
  pair's row opens its games and its quotient worked out (`51 ÷ 31 = 1.645`). A line
  counts the division's games ("50 of 50 games final").
- **A pool has a cross-club grid.** One club's pairs down the side, the other's across,
  every score in its cell.
- **A finished division shows its podium**: champions, runners-up and bronze, and the
  final's score.

Their pages show no club-against-club total anywhere, and they lead with pair codes
(`PU-HMD-04`) before names. The Hub does both of these better already.

Measured on the Hub, on a 375 × 812 phone screen: Piggleball (standard, engine) and
PNF × BUP (dual meet, built before the engine, whose Standings the engine's dual-meet
Standings follow), both on 2026-10-25:

- **Standings open empty on a phone.** The phone's category search starts blank:
  "Search and select a category above to view its standings."
- **A dual meet's Standings are one long column.** Every category is stacked under the
  club bar (`dualMeetStandingsHTML`'s phone branch). At PNF × BUP that is 7 categories
  and about 6,300 px, with no way to jump to one.
- **The sync line says *Live* for good.** `apps/hub.js` writes "*day* · Live · last
  synced *time*" whenever a snapshot loads. Piggleball reads "Oct 3 · Live · last
  synced 8:25:50 AM" three weeks later.
- **A tab has no address.** Switching tab changes the page but not its URL (the
  view is kept in the browser's storage only), so a QR code or a shared link can't
  open on Standings.
- **The row colours aren't explained.** A table row is green when the pair has more
  wins than losses and pink otherwise, a level record such as 2–2 included
  (`standingRowClass`, `views/ticket.js`). A reader takes green for *advancing*,
  which it isn't.
- **The counts call pairs teams.** Match Finder's first screen reads "126 Matches ·
  56 Teams · 7 Divisions" at PNF × BUP, where the teams are the two clubs and the
  56 are pairs.
- **A dual meet's club score is only in Standings.** The club bar (`clubSummaryHTML`)
  is drawn at the top of Standings and nowhere else.

P9–P17 take these. §21 lists what is not taken from Dinktator.

### 1.3 What the engine has gained since

Between 2026-11-10 and 2026-11-22 these changes reached the engine Hub, built for
their own reasons rather than from this spec. Each covers part of a proposal here,
and the proposals below are written against them.

| Change | Commit | What it covers here |
| --- | --- | --- |
| **The pair summary card** (`views/pair-summary.js`) above a searched pair's tickets: its category and bracket, its round-robin rank with W, L, PF, PA and Q (the Standings tab's numbers and order), a row per playoff round reached (the opponent or the seat's label, then the match number and time, *Live · Court n* or the result) and a status pill (*In the Semifinals*, *Playing now*, *Champions*, *Runners-up*, *Bronze medal*, *Did not advance*). Standard and dual-meet events, once the day is live; the Hub and Control Center both | `476dbd7` | **P3** for a pair in the playoffs: its next playoff match is already on the card. Not its next round-robin match, and nothing before the day is live. **P12** for the searched pair's own medal. **P14**'s quotient line, which the card's W, L, PF, PA and Q replace. **P6**, for one pair: the rank line adds *round robin final* once its category's round robin is over |
| **PF and PA columns** in every Round Robin table, the numbers in bold | `16aa049`, `2c01b15` | **P14**: the points behind each quotient are on the table |
| A pair's matches in **Schedule time order**, so **Next Up** is the earliest unplayed match | `ccc6c60` | **P3**'s *Next match* rule, which is the **Next Up** ticket's |
| The first screen's third count reads **Categories**, not *Divisions* | `2435ef6` | **P15**, half of it: the middle count still reads *Teams* |
| **Match numbers as corner tabs**, and a ticket's **category as a pill** before the round | `85907e1` | Nothing: **P4** still applies, as the code is still above the names |

None of P1, P2, P4, P5, P7, P9, P10, P11, P13, P16 or P17 is covered.

## 2. Scope

- **The engine Hub only:** `lib/v1` (`apps/hub.js`, `views/finder.js`,
  `views/ticket.js`, `views/standings.js` and their CSS) and the three templates'
  `index.html`, so every event made from a template from then on. Every event
  page built so far, under `events/` and `events/archives/`, is self-contained
  and frozen and imports nothing from `lib/v1`. They do not change.
- **Control Center does not change** unless a proposal says so. It shares the
  finder, the ticket, the pair summary card and Standings, so every change it should
  not get is an option the Hub passes, off by default
  ([site engine](../../technical/site-engine.md#changing-the-engine): "add an
  option rather than a flag").
- **The schedule board, scorer page and desk page do not change.**
- No new data. Every kept proposal reads what the snapshot and `events.json`
  already carry. (P7, dropped, would have added one optional setting.)

## 3. The proposals, and the owner's picks

The owner marked each proposal on 2026-11-23. *Redone* means the owner kept the
aim and asked for another design, which the section now describes.

| # | Proposal | Size | Covered so far (§1.3) | Owner's pick |
| --- | --- | --- | --- | --- |
| [P1](#4-p1-show-the-results-after-a-search) | Show the results after a search | Small | — | **Keep** |
| [P2](#5-p2-a-banner-that-folds-into-a-bar-on-phones) | A banner that folds into a bar on phones | Small–medium | — | **Keep**, redone |
| [P3](#6-p3-the-next-match-on-the-pair-summary-card) | The next match on the pair summary card | Small | A playoff's next match | **Keep** |
| [P4](#7-p4-names-not-team-codes-on-the-hubs-tickets) | Names, not team codes, on the Hub's tickets | Small | — | Drop |
| [P5](#8-p5-the-tabs-in-a-menu-on-the-folded-bar) | The tabs in a menu on the folded bar | Small | — | **Keep**, redone |
| [P6](#9-p6-a-status-pill-on-each-round-robin-bracket) | A status pill on each round-robin bracket | Small | One pair's, on its card | **Keep**, redone |
| [P7](#10-p7-a-rules-link) | A rules link | Small | — | Drop |
| [P8](#11-p8-follow-more-than-one-pair) | Follow more than one pair | Medium | — | Drop |
| [P9](#12-p9-a-tabs-own-address) | A tab's own address | Small | — | Drop |
| [P10](#13-p10-standings-open-on-a-category-on-phones) | Standings open on a category on phones | Small | — | Drop |
| [P11](#14-p11-say-final-once-the-day-is-over) | Say *Final* once the day is over | Small | — | Drop |
| [P12](#15-p12-a-podium-on-a-finished-category) | A podium on a finished category | Small–medium | One pair's medal, on its card | **Keep** |
| [P13](#16-p13-explain-the-colours-and-the-ties) | Explain the colours and the ties | Small | — | Drop |
| [P14](#17-p14-a-standings-row-opens-that-pairs-matches) | A Standings row opens that pair's matches | Small | The quotient line (the card, PF and PA) | **Keep** |
| [P15](#18-p15-a-dual-meets-club-score-first-and-pairs-not-teams) | A dual meet's club score first, and *Pairs*, not *Teams* | Small | *Categories*, not *Divisions* | **Keep** |
| [P16](#19-p16-jump-to-a-category-in-a-dual-meets-standings) | Jump to a category in a dual meet's Standings | Small | — | **Keep** |
| [P17](#20-p17-a-cross-club-grid-per-category) | A cross-club grid per category | Medium | — | Drop |

The nine kept: P1 and P3 land a search on the pair's next match, which is Purple
Pickle XP's best feature without its welcome screen. P2 and P5 keep the banner as it
is at the top of the page and fold it into a slim bar with a menu once it scrolls
away. P6, P12 and P14 make Standings answer *where do we stand?*, and P15 and P16
do the same for a dual meet. A dropped proposal's section stays below, marked
**Dropped**, as the record of what was considered.

**The pictures.** Each proposal that changes what the page shows has a picture:
the page today beside a mockup of the proposal, or two mockups for a change that
happens in two steps. They are the engine's own template pages, served by the
site's comparison harness on its fixture data with the clock held at 1 PM on a
375 × 812 phone screen, and photographed after the proposal's new elements were
drawn into the page. So everything outside the change is the real page.
The new elements are a sketch of the proposal, not its final styling, and every
mockup is labelled **Mockup**. P7, P8 and P9 have none: P7 is one link, P8 is
dropped and P9 changes only the address.

The pictures for P1, P2, P3, P5, P6, P13, P14 and P15 were taken on 2026-11-23 at
`b89fda5`, with the pair summary card, the PF and PA columns and the corner match
numbers. In them the card's playoff rows and status pill are left out: the
fixtures' mid-day state keeps the finished day's playoff names, so mid-round-robin
the card would show playoff seats the workbook hasn't filled yet. For the same
reason the mid-day Standings tables show the finished day's W and L, with PF and PA
(counted from the scores) still at 0. The rest (P10, P11, P12, P16, P17) date from
2026-10-25 and show the page from before those changes. Their proposals are
unchanged by them.

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

![Match Finder on a phone after searching for Josh Manarang / Angelo Manarang at Pickle for Sight, 1 PM. Today, first screen: the banner and the search box fill it and the results start below. Today, scrolled down: the pair summary card (Novice Men's Doubles, Bracket 2, #2 of 5, W 3, L 1, PF 20, PA 12, Q 3.0000), then the tickets, each with the pair codes above the names. Mockup: the page has scrolled to the results, and the card opens with a navy Next match block reading 2:50 PM · Court 4, vs Tony Hecker / Angelo Culala · PCPH Main, Round robin · Match 1060.](../../images/hub-player-experience/p1-p3-search.png)

P1 and P3 together, as a player sees them after one search. The middle panel
is today's page scrolled by hand to the results. With P2, the scroll also folds the
banner into its bar, and the results sit under the bar and the pinned search box
(the picture under P2).

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

## 5. P2. A banner that folds into a bar on phones

**Problem.** The banner (event logo, tagline, date and venue, title, headline,
"Find your pair's full match schedule…", tabs, divider, sync line) fills the
first 541 px of a phone screen. Once a reader scrolls down, it is gone: the
event's name, the tabs, the day and the sync line are all back at the top.

**Proposal.** The banner stays as it is at the top of the page: it is the event's
identity, and the first thing a player sees on opening the Hub. On a phone, once
it has scrolled out of view, a slim bar pinned to the top of the screen takes its
place. The bar carries the event's logo, its date and venue in small type and its
title (D1's answer: the logos, the date and venue line and the title, smaller), and
on the right a button naming the open tab that opens P5's menu. It goes when the
banner scrolls back into view.

Match Finder's search box, already pinned on a phone, pins under the bar instead of
at the top of the screen. P1's scroll lands the results under both, so a search
folds the banner as it lands.

The bar shows only once a day has loaded and the tabs show. Computer-sized screens,
and the banner before a day loads (the day picker, a "Loading…" or error message),
stay as they are.

![Three mockups on a phone at Pickle for Sight, 1 PM. First, the top of the page: the banner as it is today, with its logos, title, tabs and sync line. Second, after a search: a slim navy bar pinned at the top with the S.A.G.E. logo, "Sep 27 · PCPH Main" above "Pickle for Sight", and a "Match Finder ☰" button; the search box pinned under it, then the pair's name and summary card with its next match. Third, the menu open: Match Finder (marked as open), Live Matches and Standings, a Day row reading Sep 27, and the sync line, over a dimmed page.](../../images/hub-player-experience/p2-p5-menu-bar.png)

P2 and P5 together.

**Where.** `apps/hub.js` draws the bar (new markup inside the shell's header, from
what the banner already shows) and an `IntersectionObserver` on `.hero` shows it,
so nothing runs on scroll. Styles in `css/hub.css` under a phone media query, new
class names only. `css/finder.css`'s pinned `.search-panel` gains an offset of the
bar's height when the bar shows. The bar's colours are the shell's `THEME` tokens,
so an event's look carries over. The bar adds `env(safe-area-inset-top)` to its
padding.

**Tests.** A `hub` case that the bar is hidden at the top and shown once the banner
has scrolled away; the harness's phone cases that scroll (every `finder-search`)
change, one accepted difference per template.

**Open decisions.** None: D1 is answered.

## 6. P3. The next match on the pair summary card

**Problem.** The pair summary card (§1.3) answers *where do we stand?* For a pair in
the playoffs it also answers *when do we play next?*, in its round rows. During the
round robin, which is most of a day, it doesn't: the next match is marked only by a
small **Next Up** pill on one ticket further down the list. And a round-robin match
on court today has no *Playing now* pill, since the card's pill reads only the
playoff rows.

**Proposal.** One block at the top of the card, above its pills, that answers the
question. Its state follows the rules the tickets already use (domain `matches.js`,
`series.js`, `byes.js`):

| State | When | The block shows |
| --- | --- | --- |
| **Playing now** | One of the pair's matches is on a court (`liveCourt` is set) | *Playing now · Court 3*, the opponent, the score if one is in |
| **Next match** | Otherwise, the first match not played, not live, not a game that won't be needed and not a BYE (the **Next Up** ticket's rule) | *Next match · 2:50 PM · Court 4*, the opponent's names, the facility when the day has more than one, the round and the match number |
| **Waiting on a result** | The next match is a playoff slot whose opponent isn't decided | As *Next match*, with the opponent reading the seat's label (*Winner of Semifinal 2*), as the card's playoff rows already do, or *To be decided* |
| **All done** | Every match played | No block: the card's rank, record and status pill already say it |

The court in *Next match* is the scheduled `CourtAssignment`. When the match is
called to a court it becomes *Playing now* with that court.

The rest of the card and the list underneath stay as they are, **Next Up** pill
included. For a pair in the playoffs the block repeats the round row it points at.
That is deliberate: the block is the one place to look, whatever the stage.

With P1, a search lands on this block. With the saved search (§1.1), so does a
returning visit, which is what Purple Pickle XP's "My Games" gives its players,
with no welcome screen and no extra step.

The picture under P1 shows the block in the *Next match* state.

**Where.** `views/pair-summary.js`: a new export, `nextMatchHTML(team, model)`,
called from `pairSummaryHTML`, and its styles beside the card's `.ps-*` rules in
`css/finder.css`. It is a pure function, unit-tested in `pair-summary.test.mjs` on
the fixtures' `pre`, `mid` and `final` states. It reuses `pairResultsHTML`'s
schedule-time order and its **Next Up** rule, moved into a shared function so the
block and the pill can't disagree.

**Depends on.** Nothing; it is most useful with P1 and P2.

**Open decisions.**

- **D2.** A team event, which has no summary card: the same block for the team's
  next **matchup** (both time slots and courts) and for a player's next match
  within it, or standard and dual-meet events first and the team card later
  (proposed).
- **D3.** On a page load with a saved search, scroll to the card (as the other
  portal opens on its *My Games* screen) or leave the page at the top (today).
  Scrolling is proposed, but only when the saved tab is Match Finder.
- **D15.** Before the day goes live the card shows nothing (no scores yet). Show the
  block alone then, as the time and court are already known (proposed), or keep the
  card's rule.
- **D16.** Control Center: the card is shared, so the block shows there too
  (proposed: an operator asked where a pair plays next answers from it), or the Hub
  turns it on with an option.

## 7. P4. Names, not team codes, on the Hub's tickets

**Dropped** by the owner on 2026-11-23.

**Problem.** Every ticket side begins with the pair's workbook code (`ND_5`,
`ND_QF_4`). It means nothing to a player, and the category and round are already
on the ticket as pills, its match number as the tab in its corner. The team
event's Hub already hides its codes (its `teamLetters: false`).

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

## 8. P5. The tabs in a menu on the folded bar

**Problem.** After scrolling down a long list (the day's matches, a category's
standings), switching tab means scrolling back up to the banner. So does changing
day at a multi-day event, or checking when the page last synced.

**Proposal.** P2's bar ends with a button naming the open tab (*Match Finder ☰*). A
tap opens a menu that drops from the bar, over a dimmed page:

- the tabs, the open one marked;
- the day picker, at a multi-day event (the banner's own `select`, moved, so there
  is one);
- the sync line.

A tab tap switches the tab, closes the menu and scrolls to the top of that tab's
content, under the bar. A tap on the dimmed page or the button closes it, as does
Escape. The banner keeps its own tabs, so at the top of the page nothing changes.
This replaces the bottom bar first proposed here, which the owner turned down for a
menu at the top: it takes no space at the bottom of the screen, where iOS Safari's
own toolbar moves.

The picture under P2 shows the button and the menu open.

**Where.** `apps/hub.js` (the menu's markup and its open, close and tab-switch
handlers, which call the tabs' existing switch) and `css/hub.css`. The button is a
real `button` with `aria-expanded` and `aria-controls`; the menu's tabs are
buttons.

**Depends on.** P2, whose bar carries the button.

**Tests.** `hub` cases: the menu opens and closes, a tab tap switches and closes
it, and the open tab is marked. A harness phone case with the menu open.

**Decisions.**

- **D17.** What the menu holds. **Answered:** the tabs, the day picker and the sync
  line.

## 9. P6. A status pill on each round-robin bracket

**Problem.** A round-robin table mid-day reads as final: nothing says whether its
bracket has finished. Only a searched pair's summary card says anything: its rank
line adds *round robin final* once its category's round robin is over (§1.3).

**Proposal.** One pill in each round-robin bracket's coloured header, beside
*Bracket 2*, in the style of a team event's matchup pills (`.tm-pill`):

| State | When | The pill |
| --- | --- | --- |
| **In progress** | Some of the bracket's round-robin matches are unplayed | *In progress · 6/10*, the bracket's played matches out of its total, on the green fill |
| **Final** | Every one is played | *Final*, on navy |

No pill before the day is live. A match counts as played by the rule of
`matches.js`, BYEs left out, and a bracket is the `bracket` column's group, as the
table already draws it. A table without brackets (one group per category) carries
the pill in its *Round robin* heading instead. The Hub never works out who would
advance, and a playoff slot still reads **TBD** until the workbook fills it, so
nothing here is a projection. This replaces the status line and the "organisers
confirm who advances" line first proposed here, which the owner found too long.

![Pickle for Sight's Novice Men's Doubles on a phone. Today, mid-day: three round-robin tables, Brackets 1 to 3, with nothing saying whether each has finished. Mockup, mid-day: Bracket 1's and Bracket 2's headers each carry a green "In progress · 6/10" pill, and Bracket 3's a navy "Final" pill. Mockup, finished: every bracket reads "Final".](../../images/hub-player-experience/p6-status-pill.png)

**Ties.** Pairs level on wins each carry a small pill under their names, in the
same style, saying how the tie was broken. The Hub's order (`rankStandings`,
`domain/standings.js`) separates pairs level on wins by their wins against each
other, then by quotient:

| The tie | How it is broken | The pill on each row |
| --- | --- | --- |
| Two pairs level on wins | The match between them | *Tie · won head-to-head 11–5* on one, *Tie · lost head-to-head 5–11* on the other |
| Three or more, and one or more of them beat the rest | Head-to-head | *3-way tie · head-to-head* on each pair it placed |
| Three or more who beat each other in a circle | Quotient | *3-way tie · on quotient* |
| Two pairs who never met: a dual meet's, from one club | Quotient | *Tie · on quotient* |
| Level on wins, head-to-head and quotient, as after a drawn match | Nothing | *Tied*, on yellow, as a team event's tied teams read |

Every pill but *Tied* is on a grey fill. A pair not level with any other has no
pill. The pills say only what the Hub's order did. The workbook picks who advances
by wins, then quotient, so on a head-to-head tie the two can differ (the
[Hub's page](../../features/tournament-hub.md) says so). This is P13's tie note,
dropped as a sentence, back as a pill.

![Three mockups on a phone, every bracket header reading Final. First, Pickle for Sight's Novice Mixed Doubles, Bracket 2: Chun Sa / Ralph, Mark Luz / Amcee Sumaya and Charles Casas / Joy Galdo on 2 wins, each having beaten one of the others, each with a grey "3-way tie · on quotient" pill. Second, the same bracket with one result flipped: Chun Sa / Ralph "Tie · won head-to-head 11–5" above Mark Luz / Amcee Sumaya "Tie · lost head-to-head 5–11", then Jairus Villanueva / Abigail Sanchez, quotient 0.5313, "Tie · won head-to-head 11–10" above Charles Casas / Joy Galdo, quotient 0.8125, "Tie · lost head-to-head 10–11". Third, Novice Men's Doubles, Bracket 3: Jet De Guzman / Jay Dayrit and Ian Velasquez / Dave Mabañes on 0 wins, who drew 11–11, with the same quotient 0.3333, each with a yellow "Tied" pill.](../../images/hub-player-experience/p6-tie-pills.png)

No event so far, fixture or real, has a two-way tie that head-to-head settled. So
the middle picture is the fixture with one result flipped: Jairus Villanueva /
Abigail Sanchez beat Charles Casas / Joy Galdo 11–10 (match 1085), and the two
pairs' standings rows are recounted from the scores. The engine drew it as it
draws every table, and placed Jairus Villanueva's pair above Charles Casas's on
head-to-head despite the lower quotient.

**Where.** `domain/standings.js`: two new exports, `bracketProgress(rows, matches)`
(played and total per bracket) and `tieBreaks(rows, matches)` (for each pair in a
group level on wins, the group's size, the rule that placed it, `h2h`, `quotient`
or `none`, and for a two-way tie settled by head-to-head the pair's score in that
match), worked out the way `rankStandings` and `headToHeadWins` already order them,
so the pill never disagrees with the order. A best-of-3 meeting's score is its games
(*won head-to-head 2–1*). Both unit-tested on
`pre`, `mid` and `final`. `views/standings.js` draws the bracket pill in
`.rr-bracket-label` and the tie pill in the pair cell, and `css/standings.css` styles
them. The card's *round robin final* comes from `categoryRoundRobinDone`, so a test
checks a category whose brackets all read *Final* is one the card calls final.
Control Center's Standings use the same builder and get both too: operators answer
these questions at the desk.

**Tests.** `standings.test.mjs` for the counts and both states on `pre`, `mid` and
`final`, and for each tie case: a circular three-way split (Pickle for Sight's
Novice Mixed Doubles, Bracket 2), the same bracket with match 1085 flipped (two
two-way ties settled by head-to-head), an unbroken tie (its Novice Men's Doubles,
Bracket 3), and a dual meet's tie (PNF × BUP). Harness accepted differences on every
Standings case.

**Decisions.**

- **D6.** Team events' bracket tables, which already carry *Advances* and *Tied*
  pills. **Answered:** they get the bracket pill too. Their own *Tied* pill stays;
  they get no new tie pills, as their ranking is the organiser's (`teams.js`).
- **D18.** The pill's count. **Answered:** *In progress · 6/10*.

## 10. P7. A rules link

**Dropped** by the owner on 2026-11-23.

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

**Dropped** by the owner on 2026-11-23.

**Problem.** A player entered in two categories, or a parent with two children
playing, searches each time. The Hub remembers one search.

**Proposal.** Match Finder keeps up to four followed pairs on the device, as
chips above the results, each opening its results and P3 card.

**Recommendation: drop for now.** It is Purple Pickle XP's most complex screen.
It adds a list to manage (add, remove, a limit) and the saved-search
format changes. Most players are in one category on a day. Revisit if
organisers report multi-category players asking for it.

## 12. P9. A tab's own address

**Dropped** by the owner on 2026-11-23.

**Problem.** §1.2: a tab has no URL of its own. A QR code at the venue, the Hub board
or a link sent to a group chat always opens on Match Finder (or on whatever tab that
phone last had open), never on Standings.

**Proposal.** Each tab gets an address fragment: `#finder`, `#live`, `#standings`,
and `#teams` at a team event. A tab tap replaces the fragment (`history.replaceState`,
so Back still leaves the page rather than stepping through tabs). On load, a known
fragment wins over the saved view. An unknown fragment, or a tab the event doesn't
have (`#teams` at a standard event), is ignored and the saved view applies as today.

**Where.** `apps/hub.js`, beside `saveView` and `loadSavedView`. Nothing in the
shells changes.

**Tests.** A `hub` case per fragment, and one that an unknown fragment falls back to
the saved view.

**Open decisions.**

- **D8.** A tab tap replaces the fragment (proposed) or adds a history entry, so Back
  returns to the previous tab.

## 13. P10. Standings open on a category on phones

**Dropped** by the owner on 2026-11-23.

**Problem.** §1.2: on a phone a standard event's Standings start empty until a
category is searched for, even at an event with three.

**Proposal.** On a phone, Standings open on a category:

1. the category of the pair in the saved search, when there is one;
2. otherwise the first category in the event's category order.

The search box shows the chosen category and works as today. A category the reader
picks is kept as today (`selectCategory`).

![The Standings tab on a phone. Today, under the banner, a "Search a category" box and an empty card reading "Search and select a category above to view its standings." In the mockup the box reads Intermediate Men's Doubles and the category's round-robin brackets are drawn under it.](../../images/hub-player-experience/p10-standings-open.png)

**Where.** `views/standings.js`: a `createStandings` option, `openOnCategory` (default
`false`), with `apps/hub.js` passing `true` and a way to read the saved search's
category. Control Center keeps its own behaviour.

**Tests.** `standings.test.mjs` for both rules and for an event with no categories yet.
Harness accepted differences on the phone Standings cases.

**Open decisions.**

- **D9.** At an event with few categories (four or fewer), chips on phones as on a
  computer, instead of the search. Proposed **no**: one control per screen size is
  simpler, and P10 removes the empty first screen either way.

## 14. P11. Say *Final* once the day is over

**Dropped** by the owner on 2026-11-23.

**Problem.** §1.2: the sync line reads *Live* whenever a snapshot loads, weeks after
the event too.

**Proposal.** Once every facility in the day's snapshot has a `completedAt` (the stamp
`facilityCompletion` puts on a facility when its last match is played), the line reads
"*day* · Final · results as of *time*", the time being the snapshot's `generatedAt`,
since a score corrected after the last match still counts. Until then it reads as
today. The *Live (partial)* line for a facility that couldn't be reached is unchanged.

![The banner's sync line on a finished day. Today: "Oct 3 · Live · last synced 2:54:44 AM". Mockup: "Oct 3 · Final · results as of 2:54:44 AM".](../../images/hub-player-experience/p11-final.png)

The time is the fixture snapshot's own `generatedAt`, a resync after the day.

**Where.** `apps/hub.js`, where `statusEl` is written.

**Tests.** A `hub` case on the `final` fixture state, one on `mid`.

**Open decisions.**

- **D10.** A day that ends with a facility short of `completedAt` (a match cancelled
  and never scored) keeps reading *Live*. Proposed: accept that, rather than also
  using the day's date, which `events.json` carries only as the label.

## 15. P12. A podium on a finished category

**Problem.** A category's result is the Final's row in its bracket. A searched pair's
summary card names its own medal (*Champions*, *Runners-up*, *Bronze medal*, §1.3),
but nothing on the Hub says who won a category without searching for each pair.

**Proposal.** Once a category's Final and Bronze are both scored, its Standings card
opens with a podium: gold, silver and bronze, with the names, and under gold
"Beat *names* *score* in the final". The rules are Control Center's Awards tab's,
`buildPodiums(model)` in `domain/awards.js`, so a podium on the Hub never disagrees
with the one the organisers export. The bracket stays underneath unchanged. The card's
status pill works out the same medals from the pair's own rounds; a test checks the two
agree on the `final` fixture.

![Piggleball's Novice Open Doubles card on a phone, finished. Today it opens on the round-robin brackets. In the mockup a Podium card comes first: 1, Joshua Dizon / Jericho Dizon, "Beat Makeen Fernandez / Noelle Mercado 11–7 in the final"; 2, Makeen Fernandez / Noelle Mercado; 3, Arlene Gacho / Earl Andreu Acabal.](../../images/hub-player-experience/p12-podium.png)

A team event gets its podium too (D11): once its Final and Bronze matchups are
settled, its Standings view opens with the three teams, from `buildTeamPodium(model)`,
the rules of Control Center's Awards tab for a team event, and the Final's matchup
score under gold.

**Where.** `views/standings.js`: a new export, `podiumHTML(podium)`, and a
`createStandings` option, `podium` (default `false`), that `apps/hub.js` sets.
`views/teams.js` draws the team podium at the top of its Standings view, behind an
option of its own, also off by default and set by the Hub. Control Center has its
Awards tab and keeps both options off. Styles in `css/standings.css`, shared by both.

**Tests.** `standings.test.mjs` on the `final` state (a podium) and on `mid` (none);
`teams.test.mjs` the same on the team fixtures. Harness accepted differences on the
final-state Standings cases of all three templates.

**Decisions.**

- **D11.** Team events, through `buildTeamPodium`, now or later. **Answered:** now.
  (P3's team block, D2, stays later.)

## 16. P13. Explain the colours and the ties

**Dropped** by the owner on 2026-11-23.

**Problem.** §1.2: green and pink rows read as *advancing* and *out*. They mean a
winning record and anything else. And where pairs are level on wins, nothing says what
put one above the other.

**Proposal.** Text only.

- Under each round-robin table, with P6's line: *Green: more wins than losses.
  Pink: as many losses as wins, or more.*
- A pair placed above another level on wins gets a short note under its names, from
  the rule of the Hub's order that separated them. Among the pairs level on wins the
  Hub counts each one's wins against the others in that group, then compares quotients
  (`rankStandings`, `domain/standings.js`). So the note is *Level on wins, ahead on
  head-to-head* or *Level on wins and head-to-head, ahead on quotient*.

Two pairs level in a standard bracket have played each other, so head-to-head settles
it. Three or more can beat each other in a circle, as in the picture, and then the
quotient does. In a dual meet one club's pairs never meet, so a tie there is always
settled by quotient.

![Pickle for Sight's Novice Mixed Doubles, Bracket 2, on a phone. Today it is a W, L, PF, PA, Q table with green and pink rows. In the mockup the bracket's header is followed by "Round robin complete" (P6's first design). The pairs placed by a tiebreak carry a note. Under the table are a two-line colour legend and "Ranked by wins, head-to-head, then quotient. The organisers confirm who advances."](../../images/hub-player-experience/p6-p13-standings-explained.png)

The picture shows both parts: three pairs on two wins, each having beaten one of the
others, placed by quotient. It is drawn beside P6's first design, a status line,
which P6's pill has replaced. The tie note is back, as a pill, in P6.

**Where.** `domain/standings.js`: `rankStandings` also reports, for each pair, which
rule put it above the next one (a new export beside it, so its callers don't change).
`views/standings.js`'s row template shows the note. Control Center gets it too, as
with P6.

**Depends on.** P6 for the line the legend joins. Without P6 the legend is a line of
its own.

**Tests.** `standings.test.mjs` with a head-to-head split and a circular three-way
split (Pickle for Sight's Novice Mixed Doubles, Bracket 2) on the fixtures. Harness
accepted differences on every Standings case.

**Open decisions.**

- **D12.** Keep the colours with a legend (proposed), or drop them, since a legend is
  needed only because they look like something else.

## 17. P14. A Standings row opens that pair's matches

**Problem.** A table shows W, L, PF, PA and Q but not the games behind them. A pair
checking its place has to switch to Match Finder and type its names.

**Proposal.** Tapping a pair's row runs Match Finder's search for that pair: the tab
switches and, with P1 and P3, the page lands on its summary card, the next match at
its top, and its matches. Each row ends with a **›** so it reads as something to tap.
A row with no names yet (an unseeded playoff slot) has none and does nothing.

The card already shows the pair's round-robin record as Standings counts it (the
same `rankStandings` order and `pointsFromMatches` points, §1.3), so the line this
proposal first added to the results (*Round robin: 4 won · 0 lost · quotient 44 ÷ 7
= 6.2857*) is no longer part of it.

![Two mockups on a phone. First, Pickle for Sight's Novice Men's Doubles brackets, each row ending in a › and Josh Manarang / Angelo Manarang's row in Bracket 2 outlined as tapped. Second, where the tap lands: Match Finder on that pair, the summary card opening with its next match (2:50 PM · Court 4 against Tony Hecker / Angelo Culala), then #2 of 5 in Bracket 2 with W 3, L 1, PF 20, PA 12, Q 3.0000, then its tickets.](../../images/hub-player-experience/p14-row-opens-matches.png)

**Where.** `views/standings.js` (a row's `data-team` and a click handler calling a
`createStandings` option, `onPairSelect`, default none) and `apps/hub.js`, which runs
the search as a suggestion pick would.

**Depends on.** P1, for the search to land on the results.

**Tests.** A `standings.test.mjs` case for the handler. Harness: a phone case tapping
a row.

**Open decisions.** None.

## 18. P15. A dual meet's club score first, and *Pairs*, not *Teams*

**Problem.** §1.2: at a dual meet the question everyone asks, *which club is
winning?*, is answered only at the top of Standings. And Match Finder's first screen
counts pairs as *Teams*, at every standard and dual-meet event. (Its third count
already reads *Categories*, §1.3.)

**Proposal.**

- At a dual meet, the club bar (`clubSummaryHTML`: each club's round-robin wins, with
  the leader picked out) is also drawn at the top of Match Finder's first screen, above
  the counts. It is the same bar, so it reads the same in both places.
- The middle count reads *Pairs*. The rule that counts them (a code ending in a bare
  number) is unchanged.

![PNF × BUP's Match Finder first screen on a phone. Today, under the search box, tiles for 126 Matches, 56 Teams and 7 Categories. In the mockup the club bar comes first, Pickle & Friends Community 64 vs 48 1Bataan United Picklers with both logos, and the middle tile reads 56 Pairs.](../../images/hub-player-experience/p15-dual-meet-first-screen.png)

**Where.** `views/finder.js` (`introHTML`), importing `clubSummaryHTML` from
`views/standings.js`, which needs its club grouping, now inside
`dualMeetStandingsHTML`, as an export of its own. Per the engine rules the finder
starts using the new export one push after the export lands. The label change reaches
Control Center's Match Finder too, which is proposed: *Pairs* is right there as well.

**Tests.** `finder.test.mjs` for the bar at a dual meet and its absence at a standard
event, and the label. Harness accepted differences on every Match Finder first screen.

**Open decisions.** None.

## 19. P16. Jump to a category in a dual meet's Standings

**Problem.** §1.2: a dual meet's Standings on a phone are every category in one
column, about 6,300 px at PNF × BUP's seven.

**Proposal.** A row of category chips under the club bar, one per category in the
category order (*LI WD*, *LI MD*…, the short labels the schedule board already uses).
A tap scrolls to that category's card. It is a jump, not a filter: the page stays whole
and can still be scrolled through. The club bar and chips stay pinned at the top while
Standings scroll, as the club bar already is at PNF × BUP.

![PNF × BUP's Standings on a phone, scrolled to the third category. Today the pinned club bar sits over the middle of a category card. In the mockup a row of chips (LI WD, LI MD, LI XD, HI WD, HI MD and on) is pinned under the club bar, LI XD highlighted, and the page sits at the top of Low Intermediate Mixed Doubles.](../../images/hub-player-experience/p16-dual-meet-jump.png)

**Where.** `views/standings.js` (`dualMeetStandingsHTML`'s phone branch, and an id on
each category column) and `css/standings.css`. The computer layout, a grid of columns,
is unchanged.

**Tests.** `standings.test.mjs` for the chips' order and targets. Harness accepted
differences on the dual-meet phone Standings case.

**Open decisions.**

- **D13.** Jump (proposed), or a filter to one category as a standard event's phone
  search does (shorter, but the whole picture of both clubs goes).

## 20. P17. A cross-club grid per category

**Dropped** by the owner on 2026-11-23.

**Problem.** A dual meet's category card has a table per club (W, L, Q) and the
cross-club Bronze and Final. The round-robin games between the two clubs, which decide
both tables, aren't shown.

**Proposal.** Under each category's two club tables, a grid: one club's pairs down the
side, the other's across, each cell that match's score read row pair first, marked won
or lost, and an empty cell where the two didn't meet. Each row ends with its pair's W–L
and points. It is drawn from the day's matches (`model.matches`, round-robin stage
only), so it needs no new data. On a phone it scrolls sideways inside its card, with
the row names pinned, rather than shrinking the cells.

![PNF × BUP's Low Intermediate Women's Doubles card on a phone, finished. Today: a W, L, Q table per club, then Bronze Battle and Finals. In the mockup a Head-to-head grid sits between the tables and the Bronze Battle. It has the four PNF pairs down the side and the four BUP pairs across, every cell a score read row first and shaded won or lost, and a W–L column: Keren / Shaira 4–0, Dynes / Daisy 2–2, Nichole / Joy 2–2, Anne / Ellica 1–3.](../../images/hub-player-experience/p17-cross-club-grid.png)

The grid is shown open; on a phone D14 proposes it folded.

**Where.** `views/standings.js`: a new export, `crossClubGridHTML(cat, perClubRows,
model)`, called from `categoryColumnHTML`, and styles in `css/standings.css`. Control
Center gets it too (proposed): it is the operators' quickest check of a disputed score.

**Depends on.** Nothing. Most useful with P16.

**Tests.** `standings.test.mjs` on the PNF × BUP fixture
(`_tests/fixtures/pnf-x-bup-dual-meet/`): every played pairing in its cell, a missing
pairing empty, the row-first reading. Harness accepted differences on every dual-meet
Standings case.

**Open decisions.**

- **D14.** The grid open (proposed on a computer) or folded under a *Head-to-head*
  heading (proposed on a phone, where it doubles the card's height).

---

## 21. Not taken from the other portals

### 21.1 From Purple Pickle XP (2026-10-10)

| Their feature | Why not |
| --- | --- |
| A welcome screen asking "player or spectator" before anything shows | An extra step for every spectator. P3 gives players the same result after one search |
| The event poster as the home screen | The banner already carries the event's identity; P2 keeps it and folds it into a bar on scrolling, rather than making it larger |
| Eight sections (Home, Overview, Schedule, Standings, Live Courts, Championship, My Matches, Rules) | They overlap. The Hub's three tabs cover the same ground |
| A venue TV view inside the player portal | The [schedule board](../../features/schedule-board.md) is a separate, unlisted page on purpose |
| A qualification projection ("Live projection") | The workbook decides who advances; P6 says so instead of guessing |
| Polling a database row every 15 s | The Hub's live push is faster and archives every publish |
| A 1.8 MB page with its images inlined | The Hub's shell loads the engine's modules and the event's own images as separate files, and nothing here adds an image |

### 21.2 From Dinktator (2026-10-25)

| Their feature | Why not |
| --- | --- |
| A page per court (umpire, up next, the rest of the court's day) | A new page, outside this spec's scope (§2). The [schedule board](../../features/schedule-board.md)'s court filter covers part of it. A court view for a QR code at each court would be a spec of its own |
| A schedule grid that becomes stacked cards on a phone | The schedule board doesn't change here (§2). Worth its own change to the board: at PNF × BUP's nine courts a phone shows two at a time and cuts long names short |
| Players signing in, so their own matches are marked and the phone buzzes when they're called | A sign-in step for every player. P3 and the saved search give the same *your match* without one, and iOS Safari has no vibration |
| Pair codes before names (`PU-HMD-04`) | The Hub's tickets lead with the names; their codes are small, above them |
| Overlapping initials circles for each pair | No photos to show, and the second circle clips the first one's initials |
| An umpire's name on each court, PAUSED / NEXT states and a call-to-court forfeit clock | The workbook carries none of it. Calling a match to a court belongs to Control Center ([Court Control](../in-progress/control-center-court-control-spec.md)) |
| A sponsor's logo on every court card | Organiser artwork per court that no event has asked for. The event's own banner carries its partners |
| Five separate pages behind a menu | The Hub keeps its tabs on one page. P5's menu reaches them from anywhere on it |

## 22. Building it

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
  options and settings, and this spec's status line and a
  divergences section once anything departs from what is written here.

## 23. Decisions, in one place

The owner answered all ten still asked on 2026-11-23. The rest belonged to dropped
proposals, or to P5's bottom bar, which the menu replaced.

| # | Proposal | Decision | Answer |
| --- | --- | --- | --- |
| D1 | P2 | How much of the banner stays on a phone | **Answered:** the logos, date and venue and title, smaller: what the folded bar carries. The full banner stays at the top (the owner's note) |
| D2 | P3 | Team events' next-matchup block now or later | **Answered:** later |
| D3 | P3 | Scroll to the card when a saved search is restored on load | **Answered:** yes, when the saved tab is Match Finder |
| D15 | P3 | Show the next-match block before the day goes live, when the card itself shows nothing | **Answered:** yes, the block alone |
| D16 | P3 | The next-match block in Control Center's card too | **Answered:** yes |
| D13 | P16 | Category chips jump to a category or filter to it | **Answered:** jump |
| D17 | P5 | What the menu holds | **Answered:** the tabs, the day picker and the sync line |
| D6 | P6 | Team Standings' bracket tables get the pill | **Answered:** yes |
| D18 | P6 | The pill's count (*In progress · 6/10*) or *In progress* alone | **Answered:** with the count |
| D11 | P12 | Team events' podium now or later | **Answered:** now |
| D4, D5 | P5 | Tabs and icons in a bottom bar | No longer asked: the bottom bar is replaced by the menu |
| D7–D10, D12, D14 | P7, P9, P10, P11, P13, P17 | | No longer asked: the proposals are dropped |

---
**Features:** [Tournament Hub](../../features/tournament-hub.md) · **Technical:** [site engine](../../technical/site-engine.md)
