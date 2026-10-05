# Decision record — the site's duplicated code

> **Status: not started.** A decision for the owner, not an implementation
> plan. This is Phase 8 of the
> [`sage-tools-api` architecture spec](../in-progress/sage-tools-api-architecture-spec.md),
> and it builds nothing: it sets three options side by side so one can be chosen.

## What is duplicated

The site is one self-contained HTML file per page, with no build step. Code that
more than one page needs is therefore copied by hand, and the copies are kept
identical by discipline and a `diff`.

| Code | Copies |
| --- | --- |
| the `LIVE CHANNEL` block (`createLiveChannel`) | 12 files: Control Center, both templates' `index.html` and `schedule.html`, the scorer template, and those pages of every event that has not finished |
| the `ATTENDANCE CLIENT` block | 4: Control Center, the attendance template and two event pages |
| the `SCORE CLIENT` block | 2 today: Control Center and the scorer template (more as events get a `scorer.html`) |
| the played / BYE / series-final rules | `sage-tools-api`'s `facilityCompletion.mjs`, Control Center, both templates' `index.html` and `schedule.html`, and the scorer template |
| the team-event rules and the team rosters | 2 files each: PickleDrive's `index.html` and Control Center |

This is the largest maintenance risk in the system: a fix made in one copy and
missed in another shows up as a page that disagrees with the console during an
event. Fixing it changes the site's "one self-contained file per page" rule,
which is why it is the owner's decision.

## The options

**1. Shared files** at root-absolute `/assets/js/*.js`, loaded with
`<script src>`. No build step, and no byte-identical copies to keep in step.

- A page is no longer one file.
- `tools/sw.js` caching and cache-busting (`?v=`) need care, or a cached page
  keeps loading an old script.
- An archived page that keeps loading a shared file can break when that file
  changes.

**2. A generation script** that stamps each shared block into its pages. Pages
stay self-contained.

- A script to run, and a diff check to keep, so a forgotten run is a stale page.
- The source of each block still has to live somewhere that is not a page.

**3. Keep the copies**, and rely on the
[site test suite](site-test-suite-spec.md)'s consistency and parity checks to
catch drift. Nothing changes in the pages.

- The copies still have to be edited by hand, every one.
- That spec has to be built first, and catches drift only after it exists.

## Recommendation to evaluate

Option 1 for the live channel, the attendance client and the score client,
leaving archived events untouched; option 3 for the rule copies that also live
in `sage-tools-api` (the played / BYE / series rules, the team rules), because
those are checked against the server's own copy by parity tests rather than by
a `diff`.

## What the decision unlocks

Nothing in this record changes code. The choice decides whether a later spec
adds an `assets/` folder and its caching rules (option 1), a stamping script
and its check (option 2), or only the site test suite's checks (option 3).
