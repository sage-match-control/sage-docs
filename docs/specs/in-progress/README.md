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
| [sage-tools-api architecture hardening](sage-tools-api-architecture-spec.md) | Phases 1–5 (2.8.1–3.0.0), merged and deployed: the folder moves, hardening and one composition root, the retry loop and store interface, the delivery strategies and the `SyncService` split, the dependency-rules guard (R0–R10), and the `/v3` API with its conventions guard, problem-details errors and deprecation headers. Phase 6's code: the site and the repo's `sheets-sync.gs` call `/v3`. Phase 7's docs and rules | Phase 6: a fresh copy of each master checked, the production acceptance. Phase 7: the build-trigger filter and the workspace's GitHub repo. Phase 8: the [site engine](../not-started/site-engine-spec.md), decided and specified, not built |
