# Fast data delivery — plain-language explainer

*Companion to `fast-data-delivery-spec.md` (the implementation spec). This
file is for understanding what the project does and why; it has no
instructions an implementer should follow — read the spec for that.*

*Archived — superseded: Live push delivery was chosen and built instead; this plan will not be built.*

*This is one of two alternative plans for the same problem. The other is
[Live push delivery](../implemented/durable-object-push-explainer.md), which pushes each
update to open pages instead of having them check for one. Only one gets
built.*

---

## What it does, in plain terms

Right now, when someone updates a score on a facility's Google Sheet, it takes
**about 35–40 seconds** before anyone watching the site sees it, and over a
minute and a half in bad moments. At Pickle for Sight (27 September 2026) the
biggest measured part was GitHub Pages rebuilding the site after each data
commit: 25 seconds typically, 52 seconds one time in ten. When commits come
quickly, each new one cancels the build in progress, so busy stretches are the
slowest.

This spec replaces that step with Cloudflare R2 (an S3-like file store) as the
live data path, aiming for **~5–7 seconds** — and it's not a rewrite of the
sync logic, just a change to where the data lands and how the client asks for
it.

### The three pipelines side by side

![Three ways a score reaches a screen: the old delayed-trigger and GitHub Pages pipeline at about 105 seconds, the proposed Cloudflare R2 pipeline at about 5 to 7 seconds (not built), and the running Cloudflare Worker push at about 2 to 5 seconds, with a bar chart of the three drawn to one scale.](../../images/sync-pipelines-compared.svg)

How to read it: each row is one score's journey from the sheet edit to a screen.
**Before** is measured (Pickle for Sight and Piggleball). **Proposed** is the
Fast data delivery plan's estimates; it was never built. **Running now** is the
Worker push: the Cloud Run leg and the edit-to-published time are measured, and
the push leg is expected to be well under a second, to be confirmed by the
real-use checks. In the two newer rows Cloud Run also saves the snapshot to
GitHub afterwards as the archive, which is off the viewers' path, and a page
whose live connection is down falls back to polling GitHub at the old speed.
The old row's biggest delays were the delayed Apps Script trigger and the GitHub
Pages build; the two newer plans both remove them.

## One thing it doesn't fix on its own

Before the sync even starts, Google Apps Script waits for a delayed trigger
that Google fires "about" three seconds after an edit, but sometimes much
later. Nobody has measured how late yet. If it's often 20–60 seconds, R2 alone
can't reach its target. So the first step of either plan is to measure that
delay and replace the delayed trigger with one that syncs straight away. That
work has its own spec, [Immediate sync](../implemented/immediate-sync-spec.md), which both
plans build on.

## The two mechanisms

**1. Pointer + payload, instead of one growing file.** Today the client
re-downloads the whole day's match data (4–6 KB) on every poll, whether
anything changed or not. The new design splits that into:

- a tiny **pointer** (~100 bytes: just a fingerprint of the data plus a
  timestamp) the client checks every 3 seconds
- the actual **payload** (match/standings data), fetched only when the
  pointer's fingerprint changes

Since the payload's filename *is* its fingerprint, it never changes once
written — so browsers and Cloudflare's CDN cache it forever with zero
staleness risk.

**2. R2 replaces GitHub as the "hot" store; GitHub becomes a pure backup.**
Cloud Run still does the same work (fetch sheets, merge facilities) but writes
to R2 instead of waiting on a GitHub Pages deploy. GitHub still gets a commit
after, as a durability archive and a fallback if R2 is ever down — the client
automatically falls back to the old GitHub Pages URL on any R2 failure.

## Why it's not just "swap the store" — four real problems this had to solve

1. **A custom domain in front of R2 isn't decoration — it's what makes this
   affordable.** With Cloudflare's CDN caching the pointer and payload, all
   those 3-second checks get answered at the edge; R2 only sees a trickle of
   reads no matter how many people are watching. Skip the domain (use R2's
   free `r2.dev` address instead) and every single check becomes a billed read
   that grows with your audience — the difference between ~882K reads a month
   and ~40M at 200 viewers. The domain costs about $10 a year.

2. **Concurrent writes could silently lose scores.** Several facilities sync
   at once, and Control Center's Live/Hide switch rewrites the same file too.
   Today GitHub accidentally protects against two writers clobbering each
   other. Moving to R2 loses that protection unless it's explicitly rebuilt —
   so the spec requires "only write if nobody else has since" checks with a
   redo-the-whole-merge retry. Without them, one facility's scores could just
   vanish.

3. **The fingerprint has to include the Live/Hide setting.** If it only
   covered the scores, flipping **Force hidden** would change nothing the
   pages check for, and the public site would keep showing scores — at
   exactly the moment an operator is trying to hide a mistake.

4. **Skipping duplicate writes would quietly break Control Center's
   staleness warnings.** If nothing changed, no new payload should be
   written — but Mission Control's "last synced" time and its amber "aging
   data" warning come from that same data. The fix: always refresh the
   pointer's timestamp even when data hasn't changed, so "is the pipeline
   alive" and "did the score change" stay separate signals.

## Structure of the spec document

- **§1** is a hard gate — six things that must be verified *before* any code
  is written, because two of them could invalidate the whole design. It also
  holds the Pickle for Sight measurements.
- **§2–§10** are the implementation details: data formats, fingerprint rules,
  concurrency handling, the exact backend/client code changes (including the
  full list of pages that poll today), infrastructure setup, and the cost
  model.
- **§11** is the build order, starting with the shared Apps Script fix.
- **§12** is an acceptance checklist covering write-path correctness, client
  behavior, the Live/Hide switch, and infra (including catching a
  misconfigured cache rule via metrics after the first real event).
- **§13** explicitly excludes things like migrating old data or retiring
  GitHub Pages — this is additive, not a rip-and-replace.

## Bottom line

It's a well-scoped performance upgrade to the live-scoring path, not a
rewrite of the sync system — but it does add new failure modes (R2 outages,
cache misconfiguration, concurrent-write races) that the old GitHub-only path
didn't have, which is why §1 and §12 are as detailed as they are. Compared
with Live push delivery it is slower (seconds of polling instead of an
instant push) and needs a paid domain, but it has no always-open connections
and uses only standard, portable storage.
