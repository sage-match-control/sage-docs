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
| [Code and access protection](code-and-access-protection-spec.md) | Built 2026-10-12, and deployed (all of it is in `sage-tools-api` 3.4.0 and the site's `main`): the played/series fix (`sage-tools-api` 3.1.1), the server reading workbooks as its service account (3.2.0), the attendance list through the API (3.3.0), the site's attendance view with its harness, the runbook and docs for Restricted facility workbooks, and `LICENSE` files | The owner's tasks (spike S1, quota checks O1–O2, then O3), and the Google, GitHub and Cloud settings of §2, §5, §6 and §7 |
