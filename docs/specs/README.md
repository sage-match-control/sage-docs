# Specs

Design specs written before (or while) a feature was built — the reasoning
behind non-obvious decisions, worked examples, and the divergences section
each one keeps up to date when the built feature departs from what's written
here. These are reference material, not onboarding reading: start with
[Features & Usage](../features/README.md) or [Technical](../technical/README.md)
instead, and come here when you need the *why* behind a specific piece.

## How this folder is organised

By **build status**, one subfolder each:

| Folder | Means |
| --- | --- |
| [`implemented/`](implemented/README.md) | Built and in use. Read these to understand something that exists |
| [`in-progress/`](in-progress/README.md) | Partly built — some of it is running, some isn't. Read the spec's own status line for which |
| [`not-started/`](not-started/README.md) | Nothing built. These are proposals and plans |

### Citing a spec

A spec's status folder is **not part of its identity.** Filenames are unique
across all three folders, and that is what lets a spec change status without
breaking every reference to it.

| Citing from | Write |
| --- | --- |
| Anywhere **outside** `sage-docs` — code comments, CLAUDE.md files, READMEs | `sage-docs/docs/specs/.../<name>-spec.md` |
| **Inside** `sage-docs` | The real path, e.g. `implemented/<name>-spec.md` — mkdocs has to resolve it |

The `...` stands in for whichever status folder the spec is in today. It keeps
the reference correct forever, still greps by filename, and avoids `*/`, which
would terminate the `/** */` block comments these references sit in.

### Moving a spec between folders

Because of the rule above, this is confined to `sage-docs`:

1. `git mv` it into the new folder.
2. Update its **own status line** — the folder is filing, the spec is the
   record.
3. Update this index, and the READMEs of both the folder it left and the one
   it joined.
4. Update `mkdocs.yml`'s nav.
5. Fix any real markdown link to it — `grep -rn "<name>-spec.md" docs/`. A link
   from another folder needs `../<folder>/`.

Nothing outside `sage-docs` should need touching. If a grep of the other three
repos turns up a folder-qualified path, that reference is the bug — rewrite it
to the `...` form rather than repointing it.

---

## Implemented

### Sync & live data

- **[Runtime-fetched sync config](implemented/sync-config-runtime-spec.md)** —
  why `event-data/config/events.json` is fetched at runtime instead of
  committed to `sage-tools-api`.
- **[Sync script configuration](implemented/sync-script-configuration-spec.md)**
  — why `sheets-sync.gs` keeps its per-workbook config in Script Properties,
  and the validation the setup dialog runs before saving it.

### Control Center

- **[Match Control console](implemented/match-control-console-spec.md)** — the
  central operator console (written before it was renamed Control Center).
- **[Awards tab](implemented/awards-podium-tab-spec.md)** — podium finishers
  and image export.
- **[Schedule screen](implemented/schedule-screen-spec.md)** — the venue wall
  display.

### Scoresheet Generator

- **[Event picker](implemented/scoresheet-event-picker-spec.md)** — sourcing
  the matches CSV from a published `event-data` snapshot instead of a file
  upload.

### Tournament Calculator

- **[Dual-meet fixes](implemented/calculator-dual-meet-spec.md)** — the
  calculator's dual-meet-specific timing math.
- **[PWA](implemented/calculator-pwa-spec.md)** — installable, offline setup.

### Bracket Generator

- **[Bracket Generator](implemented/bracket-generator-spec.md)** — promoting
  the per-event bracket draw page into one evergreen tool, with an optional
  event name.

### Dual Meet Sheet Generator

- **[Sheet generator](implemented/dual-meet-sheet-generator-spec.md)** —
  Phase 1: category tabs, `Variables`, `Title`, `Reference for Players`.
- **[Schedule generator](implemented/dual-meet-schedule-generator-spec.md)** —
  Phase 2: the `SCHEDULE` tab.
- **[Readouts generator](implemented/dual-meet-readouts-generator-spec.md)** —
  Phase 3: `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`.

### Events & templates

- **[Event site templates](implemented/event-templates-spec.md)** — the
  dual-meet and standard-tournament templates new events are instantiated
  from.
- **[Pickle & Friends × 1Bataan United Picklers dual meet](implemented/pnf-x-bup-dual-meet-spec.md)**
  — the event that drove the dual-meet template's first real run.

---

## In progress

- **[Verifiable draw](in-progress/bracket-generator-verifiable-draw-spec.md)**
  — proving a draw wasn't rigged: a seed, a hash sort anyone can re-check on
  any SHA-256 site, and a plain-language *How it works* dialog. **Written and
  working, but uncommitted** — the deployed tool does not have it yet.

---

## Not started

- **[Standard Tournament Master](not-started/standard-tournament-master-spec.md)**
  — the dual-meet generator's counterpart for open-entry tournaments: one
  workbook per facility per day, uneven round-robin brackets, and a
  forward-propagating single-elimination ladder.
- **[Fast data delivery](not-started/fast-data-delivery-spec.md)** — replacing
  the GitHub Pages build step in the live data path with Cloudflare R2, and
  full-payload polling with pointer polling.
  - **[Plain-language explainer](not-started/fast-data-delivery-explainer.md)**
    — the same, without the implementation detail.
- **[Bracket Generator workbook handoff](not-started/bracket-generator-workbook-handoff-spec.md)**
  — a SAGE menu route from a scoring workbook into the tool, and the roster
  scaffold's unfilled STEP 3. Was deferred until a standard tournament
  existed; one now does.
