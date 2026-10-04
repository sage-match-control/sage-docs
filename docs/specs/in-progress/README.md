# In-progress specs

Specs where **part of the work is live and part is not**. The folder is a
filing decision; each spec's own status line is where you find out which half
you are reading.

A spec belongs here when a reader could otherwise be misled — a phased build
partway through, a feature shipped on one surface but not the other, or
something written and working locally but not committed or deployed. It does
not stay here: when the rest lands it moves to
[`../implemented/`](../implemented/README.md); if the work is abandoned, the
built parts get their own spec and the rest goes back to
[`../not-started/`](../not-started/README.md).

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Built | Not yet |
| --- | --- | --- |
| [Team tournament](pickledrive-club-anniversary-team-tournament-spec.md) | The public site, schedule board, Control Center `team` type, registry entry and operator runbook (§5–§11). Used for the event on 3 October 2026 | The team template (§15), extracted from this event; its workbook's slow Sheets reads to fix first |
| [Live push delivery](durable-object-push-spec.md) | `sage-tools-api` 2.4.0 (`LivePublisher`, the publish-then-archive flow in `SyncService`, Live/Hide through the push path), the `live-worker/` Cloudflare Worker and Durable Object with `smoke.mjs`, the live-channel block in Control Center and both templates and the Piggleball and PickleDrive pages, and the docs. Deployed: the Worker, Cloud Run 2.5.0 with `LIVE_PUSH_URL` and `LIVE_PUSH_SECRET` set, and `LIVE_BASE_URL` set in all nine pages (emptied in Piggleball's and PickleDrive's four since both finished). A sync from each event publishes to the Worker and archives to GitHub, and the three kinds of page hold an open socket | The real-workbook checks in §11 (two-workbook race, stopwatch timings, **Force hidden**, blocked-Worker fallback, long-open tab, ping count, the **Sync method** switch with a real sign-in) |
| [Live push delivery — explainer](durable-object-push-explainer.md) | The same, in plain language. No instructions to implement | — |
