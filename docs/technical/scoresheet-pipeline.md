# Scoresheet pipeline

CSV in, PDF out: `CSV → Handlebars → Chromium → per-chunk PDFs → merged
PDF`, in `sage-tools-api/src/scoresheets/`. Entirely independent of the
sync pipeline — no shared state, no shared endpoints.

## Composition

- `ScoresheetConfig.mjs` is the type registry — each registered type maps to
  a `templates/<type>.html` + `<type>.css` pair. Four types ship today:
  Standard (with a referee column), No Referee, No Referee Wide, and Best
  of 3. Adding a new type is a new template pair **and** a `ScoresheetConfig.mjs`
  entry — nothing else needs touching.
- `index.mjs` composes the pipeline: parse the uploaded CSV, render each
  match through the selected type's Handlebars template, render the
  populated HTML through headless Chromium (Puppeteer) into per-chunk PDFs,
  merge into one final PDF.

**The pipeline is wired lazily**, not eagerly, in `index.mjs`'s
composition root. Puppeteer is only imported on the *first* scoresheet
request — the sync pipeline is wired eagerly, so a sync-only cold start
(the common case for this service) never pays Puppeteer's import cost at
all. Don't move that import to the top level.

## Endpoints

- **`POST /scoresheets/generate`** — multipart (`csv` file + `evt`, `type`,
  `out`, `blanks`), returns a PDF directly.
- **`POST /scoresheets/generate/stream`** — same inputs, but returns NDJSON
  progress lines as each chunk renders, then a final
  `{"phase":"done","pdfBase64":...}` line. This is what
  `tools/scoresheet-generator.html`'s progress bar is reading. Once
  streaming has started, the HTTP status is locked at 200 — a failure
  midstream arrives as a `{"phase":"error"}` line in the body, not an HTTP
  error status, since headers are already sent.

## Deployment

Runs on `node:22-slim` (Puppeteer requires Node ≥22.12), with system
Chromium installed via `apt` rather than Puppeteer's own bundled download —
`PUPPETEER_SKIP_DOWNLOAD` + `PUPPETEER_EXECUTABLE_PATH` in the Dockerfile.
See [Deployment](deployment.md) for the full Cloud Run setup.

---
**Features:** [Scoresheet Generator usage](../features/scoresheet-generator.md)
