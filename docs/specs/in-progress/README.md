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
| [Verifiable draw](bracket-generator-verifiable-draw-spec.md) | The whole feature, in the working tree — seed field, SHA-256 fingerprint sort, auto-seed generator, verification payload on both exports, *How it works* dialog, and both doc pages | **Committed and deployed.** `tools/bracket-generator.html` on the live site has none of it; the committed copy contains no `seedInput` and no SHA-256. The change is ~294 uncommitted lines in `sage-match-control.github.io`, alongside uncommitted edits to `docs/features/bracket-generator.md` and `docs/technical/bracket-generator.md` |
