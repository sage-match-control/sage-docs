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
| [CLSO Pickle for Sight](pickle-for-sight-spec.md) | The site (§3–§9), registered in `event-data` (§10), both venue workbooks (§12.1), and live sync publishing for both PCPH Main and PCPH Annex (§12.2) | The dry run (§12.4), due 25 September. `isLive` also needs to move from its current hardcoded `true` back to `"auto"` before the event |
