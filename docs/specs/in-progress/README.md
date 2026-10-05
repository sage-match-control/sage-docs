# In-progress specs

Specs where **part of the work is live and part is not**. The folder is a
filing decision; each spec's own status line is where you find out which half
you are reading.

A spec belongs here when a reader could otherwise be misled — a phased build
partway through, a feature shipped on one surface but not the other, or
something written and working locally but not committed or deployed. It does
not stay here: when the rest lands it moves to
[`../implemented/`](../implemented/README.md); if the work is abandoned, the
built parts get their own spec and the rest goes to
[`../archived/`](../archived/README.md).

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Built | Not yet |
| --- | --- | --- |
| [Live push delivery](durable-object-push-spec.md) | `sage-tools-api` 2.4.0 (`LivePublisher`, the publish-then-archive flow in `SyncService`, Live/Hide through the push path), the `live-worker/` Cloudflare Worker and Durable Object with `smoke.mjs`, the live-channel block in Control Center and both templates and the Piggleball and PickleDrive pages, and the docs. Deployed: the Worker, Cloud Run 2.5.0 with `LIVE_PUSH_URL` and `LIVE_PUSH_SECRET` set, and `LIVE_BASE_URL` set in all nine pages (emptied in Piggleball's and PickleDrive's four since both finished). A sync from each event publishes to the Worker and archives to GitHub, and the three kinds of page hold an open socket | The real-workbook checks in §11 (two-workbook race, stopwatch timings, **Force hidden**, blocked-Worker fallback, long-open tab, ping count, the **Sync method** switch with a real sign-in) |
| [Score entry and scorer links](control-center-score-entry-spec.md) | `sage-tools-api` 2.8.0 (`src/scores/`: the score route, scorer links, the `scoreEntry` switch, `SheetsClient`'s score allowlist), Control Center's Match Finder dialog and Scorer links section, `_templates/scorer/scorer.html`, the docs. Tested against fakes and local fixtures | The real-workbook checks in §11.2 (C1–C8: a Dual Meet, Standard and team workbook copy, no onEdit run from the API write, clearing, a conflict, a missing Editor share, a scorer link on a phone, the Stopped switch), then setting `"scoreEntry"` on a real event and instantiating its scorer page |
| [sage-tools-api architecture hardening](sage-tools-api-architecture-spec.md) | Phases 1–4 (2.8.1–2.8.4) on the `arch-refactor` branch: `apps-script/` split from `scripts/`, `src/clients/`, `src/registry/` and per-feature `domain/` folders; one error handler with stable codes, one auth-middleware module, a validated `loadConfig`, `createApp` shared by `index.mjs` and the tests, a constant-time secret check, reserved day keys; `withConflictRetry`, the registry split into load/cache, validation and pure edits, the `SnapshotStore` port with its two adapters and `mergeIntoStore`; `GitHubOnlyDelivery`, `LiveFirstDelivery` and `SnapshotPublisher`, the `SyncService` split into three services and a controller, and the dependency-rules guard (R0–R10) | Phase 5 (the `/v3` API as 3.0.0), 6 (every client on `/v3`), 7 (build trigger, docs, workspace) and 8 (a decision record). Nothing is merged or deployed |
| [Live push delivery — explainer](durable-object-push-explainer.md) | The same, in plain language. No instructions to implement | — |
