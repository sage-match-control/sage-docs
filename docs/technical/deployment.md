# Deployment

Every repo deploys the same way: **push to `main`.** No manual deploy step
for any of them. `sage-match-control.github.io` and `event-data` are plain
GitHub Pages, so a push is their entire story with no build. `sage-tools-api`
runs on Google Cloud Run behind a build trigger watching the repo — pushing
to `main` builds and deploys the new revision automatically; nobody runs
`gcloud run deploy` by hand day to day.

## Local development

```bash
npm start
```

Runs `node index.mjs` on `PORT` (default 8080) over h2c with HTTP/1.1
fallback. Copy `.env.example` to `.env` first — it documents every
variable. Secrets (`GITHUB_TOKEN`, `SYNC_SHARED_SECRET`,
`GOOGLE_SHEETS_API_KEY`, `AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`) are
Cloud Run env vars in production and must never be committed.
`AUTH_PASSWORD_HASH` is generated locally with
`node scripts/hash-password.mjs '<username>' '<password>'` — see
[Auth](auth.md).

ESM throughout (`"type": "module"`, `.mjs` files, classes, constructor
injection). No TypeScript, no build step, no test suite, no linter.

## Cloud Run deploy

The build/deploy trigger runs the equivalent of:

```bash
gcloud run deploy sage-tools-api --source . --use-http2 --region us-central1 \
  --memory 2Gi --cpu 2 --timeout 900 --concurrency 4 --min-instances 0 \
  --allow-unauthenticated
```

— this exact command is kept as a comment at the top of the `Dockerfile`,
both as the documentation of what the trigger does and as the manual
fallback if the trigger itself ever needs bypassing.

Runs on `node:22-slim` (Puppeteer requires Node ≥22.12), with system
Chromium installed via `apt` rather than Puppeteer's own bundled download
(`PUPPETEER_SKIP_DOWNLOAD` + `PUPPETEER_EXECUTABLE_PATH` in the
Dockerfile). See [Scoresheet pipeline](scoresheet-pipeline.md) for why the
Puppeteer import itself is lazy despite this.

## Versioning

Bump `package.json`'s version for **any** code change, however small —
patch for a fix/tweak, minor for a new endpoint or feature, major for a
breaking change. It's the only way to tell what's actually running on
Cloud Run: `GET /ping`'s `X-App-Version` header reads straight from it. Add
a matching Changelog entry in `sage-tools-api/README.md` alongside the
bump.

## What does *not* need a redeploy

Changing anything in `event-data/config/events.json` — adding an event, a
day, or fixing a sheet ID — is a commit to that repo, picked up by every
running Cloud Run instance within `SYNC_CONFIG_TTL_MS` (~60s default). See
[sync pipeline](sync-pipeline.md) for why this moved out of source code.

## Diagnostics

- `GET /ping` — health check. `X-App-Version` (package version),
  `X-Sync-Config` (cached config's short SHA + source). Never triggers a
  config fetch itself, so it stays fast even if `event-data` or GitHub is
  unreachable.
- `GET /openapi.json` — the full API spec, generated at request time from
  `@openapi` JSDoc blocks above each route handler — never hand-maintained,
  so it can't drift from the actual routes. Public, no auth. Importable
  straight into Postman via **Import → Link**.
- `GET /sync/config` (secret- or token-gated) — full diagnostic view of the
  currently-loaded event registry.
