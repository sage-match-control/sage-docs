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

Runs `node index.mjs` on `PORT` (default 8080) over h2c (cleartext HTTP/2:
Cloud Run talks h2c to the container, and a cleartext HTTP/1.1 client cannot
connect, so a local check needs an h2c client such as Node's `http2`). Copy
`.env.example` to `.env` first — it documents every variable. Secrets (`GITHUB_TOKEN`, `SYNC_SHARED_SECRET`,
`GOOGLE_SHEETS_API_KEY`, `AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`) are
Cloud Run env vars in production and must never be committed.
`AUTH_PASSWORD_HASH` is generated locally with
`node scripts/hash-password.mjs '<username>' '<password>'` — see
[Auth](auth.md).

ESM throughout (`"type": "module"`, `.mjs` files, classes, constructor
injection). No TypeScript, no build step, no linter.

### Startup validation

`src/config/loadConfig.mjs` is the only reader of the environment. A value is
trimmed, and a blank one counts as unset. A number must be a whole number inside
its range (`PORT` 1–65535; `SCORESHEET_CONCURRENCY`, `AUTH_TOKEN_TTL_MS` and
`LIVE_PUSH_TIMEOUT_MS` at least 1; `SYNC_CONFIG_TTL_MS` and
`SHEETS_FETCH_TIMEOUT_MS` at least 0, where 0 disables the timeout), and
`LIVE_PUSH_URL` an `http` or `https` URL. A variable that is set but unusable
stops the service before it listens, with a message that names every bad one:

```
Invalid environment:
  - SHEETS_FETCH_TIMEOUT_MS: "abc" must be a whole number at least 0
```

An unset secret does not stop it: it turns its feature off (or makes it fail
closed), and one `warn` line at startup, `Environment: not set, so what they
enable is off or fails closed: …`, names what is unset. Check the Cloud Run
logs for that line after a deploy and confirm it names nothing unexpected. A new
variable is added in `loadConfig.mjs`, in `.env.example` and in
`test/unit/config/loadConfig.test.mjs`.

### Tests

From `sage-tools-api/`:

```bash
npm test
```

runs the unit and integration suite (Node's built-in runner, no network).
`npm run verify` runs it plus the per-folder coverage thresholds
(`npm run test:coverage`) and the three Apps Script harnesses
(`npm run test:appscript`). Run `npm test` before every commit and
`npm run verify` before every push to `main`; a pre-push hook runs the suite.

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

### Which pushes build

The build trigger rebuilds the image only for a change that is part of the
service. A change to Apps Script, the live Worker, the tests, the
scripts, the git hooks, or any markdown, `.env.example` or `jsconfig.json` does
not redeploy Cloud Run, because none of them is in the image: the `Dockerfile`
copies only `package*.json`, `index.mjs`, `src/` and `templates/`. A change to
any of those, or to the `Dockerfile` itself, still builds.

The owner sets the filter once, on the existing trigger. List the triggers to
find its name:

```bash
gcloud builds triggers list --project=sage-tools-api
```

```bash
gcloud builds triggers update github <TRIGGER_NAME> --project=sage-tools-api --ignored-files="apps-script/**,live-worker/**,test/**,scripts/**,.githooks/**,**/*.md,.env.example,jsconfig.json"
```

To check it, push a README-only commit to a branch the trigger watches and
confirm no build starts. (Pushing `main` deploys, so do not use `main` to test
it, and never push near an event.)

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
- `GET /v3/diagnostics/sync` (secret- or token-gated; legacy `GET /sync/config`,
  which workbooks made before 3.0.0 call during setup) — full diagnostic view of the
  currently-loaded event registry, plus `live: { enabled, baseUrl }` for live
  push.
- `GET <worker>/health` — the live Worker's health check.
