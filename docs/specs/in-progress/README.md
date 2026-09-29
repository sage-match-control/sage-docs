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
| [Team tournament](team-tournament-spec.md) | The public site, schedule board, Control Center `team` type, registry entry and operator runbook (§5–§11), checked against the test fixtures. Live sync installed and publishing for Kingcourts | The QR image and the short link (both the organiser's), the dry run, setting `isLive` back to `"auto"`, and the template (§15), which follows the event |
| [Piggleball Chairman's Cup](piggleball-chairmans-cup-spec.md) | The site (§3–§9) and its registry entry (§11) | The QR image (the organiser's), the workbook fixes (§13.1), live sync setup (§13.2) and the dry run (§13.4), due 2 October |
