# Fast data delivery — plain-language explainer

*Companion to `fast-data-delivery-spec.md` (the implementation spec). This
file is for understanding what the project does and why; it has no
instructions an implementer should follow — read the spec for that.*

---

## What it does, in plain terms

Right now, when someone updates a score on a facility's Google Sheet, it takes
**40–60 seconds** before anyone watching the site sees it. Almost all of that
time is GitHub Pages rebuilding the site after the data commits. This spec
replaces that with Cloudflare R2 (an S3-like object store) as the live data
path, cutting the delay to **~5–7 seconds** — and it's not a rewrite of the
sync logic, just a change to where the data lands and how the client asks for
it.

## The two mechanisms

**1. Pointer + payload, instead of one growing file.** Today the client
re-downloads the whole day's match data (4–6 KB) on every poll, whether
anything changed or not. The new design splits that into:

- a tiny **pointer** (~100 bytes: just a hash + timestamp) the client polls
  every 3 seconds
- the actual **payload** (match/standings data), fetched only when the
  pointer's hash changes

Since the payload's filename *is* its content hash, it never changes once
written — so browsers and Cloudflare's CDN cache it forever with zero
staleness risk.

**2. R2 replaces GitHub as the "hot" store; GitHub becomes a pure backup.**
Cloud Run still does the same work (fetch sheets, merge facilities) but writes
to R2 instead of waiting on a GitHub Pages deploy. GitHub still gets a commit
after, as a durability archive and a fallback if R2 is ever down — the client
automatically falls back to the old GitHub Pages URL on any R2 failure.

## Why it's not just "swap the store" — three real problems this had to solve

1. **A custom domain in front of R2 isn't decoration — it's what makes this
   affordable.** With Cloudflare's CDN caching the pointer/payload, all those
   3-second polls get absorbed at the edge; R2 only sees a trickle of "origin"
   reads regardless of how many people are watching. Skip the domain (use
   R2's free `r2.dev` instead) and every single poll becomes a billed read
   that scales with your audience — the difference between ~882K reads/month
   and ~40M at 200 viewers.

2. **Concurrent writes could silently lose scores.** Multiple facilities sync
   at once. Today GitHub's compare-and-swap accidentally protects against two
   syncs clobbering each other. Moving to R2 loses that protection unless
   it's explicitly rebuilt — so the spec requires conditional writes
   (`If-Match` on the pointer) with a retry-the-whole-merge loop. Without
   this, one facility's scores could just vanish.

3. **Skipping duplicate writes would quietly break the organizer's staleness
   warnings.** If nothing changed, no new payload should be written — but the
   "last synced" timestamp the organizer console relies on comes from that
   same data. The fix: always refresh the pointer's timestamp even when data
   hasn't changed, so "is the pipeline alive" and "did the score change" stay
   separate signals.

## Structure of the spec document

- **§1** is a hard gate — six things (mostly: does R2 actually support
  conditional writes here, does the Cache Rule actually cache JSON) that must
  be verified *before* any code is written, because two of them could
  invalidate the whole design.
- **§2–§10** are the implementation details: data formats, hashing rules,
  concurrency handling, the exact backend/client code changes,
  infrastructure setup, and the cost model.
- **§11** is a 16-step build order.
- **§12** is an acceptance checklist covering write-path correctness, client
  behavior, and infra (including catching a misconfigured cache rule via
  metrics after the first real event).
- **§13** explicitly excludes things like migrating old data or retiring
  GitHub Pages — this is additive, not a rip-and-replace.

## Bottom line

It's a well-scoped performance upgrade to the live-scoring path, not a
rewrite of the sync system — but it does add new failure modes (R2 outages,
cache misconfiguration, concurrent-write races) that the old GitHub-only path
didn't have, which is why §1 and §12 are as detailed as they are.
