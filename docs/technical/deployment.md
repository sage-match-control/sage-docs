# Deployment

Every repo deploys the same way: **push to `main`.** No manual deploy step
for any of them — except the live Worker in `sage-tools-api/live-worker/`,
which only ever deploys by hand (see [Live Worker](#live-worker-cloudflare)). `sage-match-control.github.io` and `event-data` are plain
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
  --allow-unauthenticated \
  --service-account=sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com
```

— this exact command is kept as a comment at the top of the `Dockerfile`,
both as the documentation of what the trigger does and as the manual
fallback if the trigger itself ever needs bypassing.

Runs on `node:22-slim` (Puppeteer requires Node ≥22.12), with system
Chromium installed via `apt` rather than Puppeteer's own bundled download
(`PUPPETEER_SKIP_DOWNLOAD` + `PUPPETEER_EXECUTABLE_PATH` in the
Dockerfile). See [Scoresheet pipeline](scoresheet-pipeline.md) for why the
Puppeteer import itself is lazy despite this.

### Runtime service account

The service runs as a dedicated service account, `sage-tools-api-runtime`,
with no project roles. [Event attendance](event-attendance.md) uses its
access token (from the metadata server) to write to each facility workbook,
so every workbook is shared with it as **Editor**, or sits in a Drive folder
that is. The `--service-account` flag above keeps it set on later deploys.
It is set once on the existing service with
`gcloud run services update sage-tools-api --service-account=…`; logging
needs no role. For local development, `GOOGLE_ACCESS_TOKEN` stands in for
the metadata server (see `.env.example`).

## Versioning

Bump `package.json`'s version for **any** code change, however small —
patch for a fix/tweak, minor for a new endpoint or feature, major for a
breaking change. It's the only way to tell what's actually running on
Cloud Run: `GET /ping`'s `X-App-Version` header reads straight from it. Add
a matching Changelog entry in `sage-tools-api/README.md` alongside the
bump.

## Live Worker (Cloudflare)

`sage-tools-api/live-worker/` is the Cloudflare Worker `sage-live` and its
`DayChannel` Durable Object (see [sync pipeline](sync-pipeline.md#live-push-delivery)).
The Cloud Run image never contains it and pushing to `main` never deploys it.
It needs a free Cloudflare account, with the `workers.dev` subdomain chosen
when prompted, so the Worker is at `https://sage-live.<subdomain>.workers.dev`.

From `sage-tools-api/live-worker/`:

```bash
npm install
```

```bash
npx wrangler login
```

```bash
npx wrangler secret put PUBLISH_SECRET
```

```bash
npx wrangler deploy
```

`PUBLISH_SECRET` is a long random value, and the same value is Cloud Run's
`LIVE_PUSH_SECRET`. Check a deploy with:

```bash
node smoke.mjs https://sage-live.<subdomain>.workers.dev <the publish secret>
```

which uses a throwaway `smoke-test/run-<timestamp>` key and exits non-zero on a
failed step. `npx wrangler dev` runs the Worker locally on
`http://localhost:8787`, reading secrets from `live-worker/.dev.vars`
(git-ignored).

Logs: `wrangler.jsonc` enables Workers Logs at 10 % sampling (one event per
sampled request: URL, method, status; never headers or snapshot bodies), read in
the Cloudflare dashboard under the Worker's **Logs**. It is included on the free
plan within a daily event allowance and is dropped, not billed, beyond it.
`npx wrangler tail` streams live requests for free without storing anything.

**Never deploy the Worker during an event, or in the few days before one.** A
deploy restarts every Durable Object and drops every WebSocket; pages
reconnect on their own, but it is the live data path.

Turning live push on, in order: deploy the Worker and run `smoke.mjs`; deploy
Cloud Run with `LIVE_PUSH_URL` unset, then set `LIVE_PUSH_URL` and
`LIVE_PUSH_SECRET` on the service (a new revision, no code push); then set
`LIVE_BASE_URL` (`wss://sage-live.<subdomain>.workers.dev`) in the pages. To
turn it off in an emergency, use Control Center's **Sync method** switch (GitHub
only), which needs no deploy; to remove it, empty `LIVE_BASE_URL` in the pages
and/or clear `LIVE_PUSH_URL` on Cloud Run. Pages fall back to polling GitHub.

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
  currently-loaded event registry, plus `live: { enabled, baseUrl }` for live
  push.
- `GET <worker>/health` — the live Worker's health check.
