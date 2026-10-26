# Spec — lean deploy image

> **Status: not started.** Written 2026-10-26 against `sage-tools-api` 3.4.0, its
> `Dockerfile` and the image Cloud Run serves (the build of commit `93e090d`).
> The Artifact Registry cleanup policy in §2.3 is already set; nothing else here
> is built. Build it **after** the Angeles City Employees Sportsfest (13 November
> 2026) and never within 3 days of an event: it changes how the API is built and
> deployed. §8 lists the owner's open decisions.

Make each deploy of `sage-tools-api` add about 20 MB to Artifact Registry instead
of 361 MB, so it costs nothing to keep deploys for rollback again, and builds get
faster. Nothing the service does changes.

---

## 1. Why

Every push to `main` makes the build trigger build a new image and store it in
the `cloud-run-source-deploy` repository. Artifact Registry gives 0.5 GB of
storage free per month and charges about $0.10 per GB-month above it.

The build starts from nothing each time, so the `apt-get install chromium` step
runs again on every deploy and produces a **new** 254 MB layer, even when nothing
in it changed. No two deploys share it. That is how 74 deploys reached 25.5 GB
before the cleanup policy (§2.3) existed. The policy now keeps one deploy, 361 MB,
which fits the free allowance only because nothing else is kept: keeping three
for rollback would take about 1.06 GB, twice the allowance.

If Chromium and Node sit in a **base image** built once, every deploy shares
those layers, and adds only its npm packages and code.

## 2. Where things stand

### 2.1 What is in the image

The compressed layers of the live image:

| Layers | Size | Share |
| --- | --- | --- |
| `node:22-slim` (Debian bookworm and Node) | 77 MB | 21% |
| `apt-get install chromium fonts-liberation ca-certificates` | 254 MB | 70% |
| `npm install --omit=dev` | 29 MB | 8% |
| `index.mjs`, `src/`, `templates/` | 0.7 MB | <1% |
| **Total** | **361 MB** | |

The `Dockerfile` is already lean: a slim base, `--no-install-recommends`, the apt
lists removed, no dev dependencies, and only the service's own files copied in.
Shrinking any one layer saves little. Sharing the big ones is what saves.

### 2.2 What one deploy stores

Each deploy adds **three versions** to the package `sage-tools-api/sage-tools-api`:
the image (361 MB, tagged with the commit SHA) and two OCI manifests the build
attaches to it (0 MB each). A cleanup rule that counts versions counts all three.

A deploy made by hand with the `Dockerfile`'s fallback command
(`gcloud run deploy … --source .`) stores its image in a different package,
`sage-tools-api`.

### 2.3 The cleanup policy

Set on the repository on 2026-10-26, not a dry run:

```json
[
  { "name": "delete-all", "action": { "type": "Delete" }, "condition": { "tagState": "any" } },
  { "name": "keep-last-deploy", "action": { "type": "Keep" },
    "mostRecentVersions": { "packageNamePrefixes": ["sage-tools-api/sage-tools-api"], "keepCount": 3 } }
]
```

It keeps the newest 3 versions of the trigger's package, which is the last
deploy, and deletes everything else, the manual-deploy package included.
Artifact Registry applies it about once a day.

`keepCount` counts versions, not deploys. A `keepCount` of 1 would keep only the
newest 0 MB manifest and delete every image.

### 2.4 What Cloud Run needs from the registry

Cloud Run copies an image when it deploys a revision and keeps that copy for as
long as the revision is serving; instances never pull from Artifact Registry
([Cloud Run: deploying](https://docs.cloud.google.com/run/docs/deploying)). So:

- Deleting the **live** revision's image from Artifact Registry does not stop it
  serving or block a cold start.
- Routing traffic back to an **older** revision needs that revision's image to
  still be in the registry. The deploys the policy keeps are the ones a rollback
  can reach. Keeping one deploy, a bad deploy is undone by reverting the commit
  and pushing, which takes a build (about 3 minutes), not from the Cloud Run
  console.
- A manual deploy's image lands in the `sage-tools-api` package, which the policy
  deletes within a day. The revision keeps serving from Cloud Run's copy, but
  can't be rolled back to.

## 3. Goals and non-goals

**Goals**

- The deploys kept for rollback, plus the base image, total under 0.5 GB.
- A deploy builds faster, because it no longer installs Chromium.
- The service behaves exactly as before. Every scoresheet type renders the same PDF.
- Cloud Run's settings and the deploy command don't change.

**Not doing**

| Option | Why not |
| --- | --- |
| Chrome's headless-only build (`chrome-headless-shell`) instead of Debian's Chromium | About 100 MB smaller, but a different browser build can render fonts and page breaks differently, and every scoresheet type would need checking. With a shared base layer its size stops mattering. |
| An Alpine base | Smaller, but Alpine's Chromium is built against musl and lags Puppeteer's supported version. Same rendering risk, for no saving once layers are shared. |
| Scoresheets in a service of their own | Would make the sync image small, but stores the same Chromium layer anyway and adds a second service to deploy and keep in step. |
| Docker layer caching in Cloud Build (`--cache-from`) | Would reuse the Chromium layer only while the cache hits, and means replacing the trigger's generated build with a `cloudbuild.yaml`. A base image is simpler and certain. |

## 4. Design

### 4.1 The base image

A new folder, `sage-tools-api/base-image/`, holds one file, `Dockerfile`:

```dockerfile
FROM node:22-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    chromium \
    fonts-liberation \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ENV PUPPETEER_SKIP_DOWNLOAD=true
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
```

The lines and their comments move here from today's `Dockerfile` unchanged. It is
built by hand, from `sage-tools-api/`, and tagged with the date it was built:

```bash
gcloud builds submit base-image --project=sage-tools-api --region=global --tag=us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api-base:2026-11-20
```

It lives in the same repository as the app images, as its own package,
`sage-tools-api-base`. Artifact Registry stores a layer once per repository, so
every app image built `FROM` it shares its 331 MB.

The trigger's ignored-files filter
([Deployment → Which pushes build](../../technical/deployment.md#which-pushes-build))
gains `base-image/**`: a change to the base image's `Dockerfile` deploys nothing
until the app's `Dockerfile` points at the new tag.

### 4.2 The app image

The `Dockerfile` becomes:

```dockerfile
FROM us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api-base:2026-11-20

# gcloud run deploy sage-tools-api --source . … (the deploy command, unchanged)

WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci --omit=dev && npm cache clean --force

COPY index.mjs ./
COPY src ./src
COPY templates ./templates

EXPOSE 8080

CMD ["node", "index.mjs"]
```

- `FROM` names a dated tag, never `latest`, so a build is repeatable and moving
  to a new base is a visible one-line change.
- `npm ci` installs exactly `package-lock.json`. `npm cache clean --force` stops
  npm's download cache being stored in the layer.
- The trigger's build service account already reads this project's Artifact
  Registry, so pulling the base needs no new permission. The manual fallback
  deploy pulls it as the owner's account.

### 4.3 `puppeteer-core` (optional, D2)

`src/scoresheets/BrowserManager.mjs` imports `puppeteer`, which also brings in the
code that downloads and manages browsers. The service never uses it:
`PUPPETEER_SKIP_DOWNLOAD` is set and Chromium comes from apt. `puppeteer-core` is
the same API without it.

`puppeteer-core` does not read `PUPPETEER_EXECUTABLE_PATH`, so the browser's path
has to reach `launch()` as `executablePath`. Under the API's rules that means a
config value read in `src/config/loadConfig.mjs` (the only reader of the
environment), added to `.env.example` and to `loadConfig.test.mjs`, and passed to
`BrowserManager` from the composition root. The variable keeps its name,
`PUPPETEER_EXECUTABLE_PATH`, so the base image doesn't change and a local run
that has no browser path set still fails the way it does today.

Measure the saving before building this: if the `npm` layer shrinks by less than
5 MB, drop §4.3 and leave `puppeteer` as it is.

### 4.4 The cleanup policy

Add a keep rule for the base package, and set the app's rule (§2.3) to the deploy
count decided in D1: `keepCount` 9 for three deploys.

```json
{ "name": "keep-current-base", "action": { "type": "Keep" },
  "mostRecentVersions": { "packageNamePrefixes": ["sage-tools-api-base"], "keepCount": 1 } }
```

Check how many versions one `gcloud builds submit --tag` makes before setting
`keepCount`, the way §2.2 counted the trigger's three. If it makes more than one,
`keepCount` is that number.

The rule's prefix also matches nothing else: the app's package is
`sage-tools-api/sage-tools-api`, and the manual-deploy package is
`sage-tools-api`, neither of which starts with `sage-tools-api-base`.

### 4.5 Versioning

The `Dockerfile` change is a code change: a PATCH bump in `package.json` and a
Changelog entry, with a **Tests:** line. Without §4.3 it touches no `src/` file,
so `test/` doesn't change, and the entry says the change is covered by the
production checks in §6. With §4.3, `BrowserManager` and `loadConfig` change, and
their unit tests change first (a failing test that the launcher passes the
configured `executablePath`).

## 5. Size and cost

Estimates from the layer sizes in §2.1. The real numbers come from §6's checks.

| | Stored | Free allowance | Per month |
| --- | --- | --- | --- |
| Before the cleanup policy | 25.5 GB | 0.5 GB | ≈ $2.50 and growing |
| Now: last deploy only, no base image | 361 MB | 0.5 GB | $0 |
| Last 3 deploys, no base image | ≈ 1.06 GB | 0.5 GB | ≈ $0.06 |
| This spec: base + last 3 deploys | ≈ 331 MB + 3 × 20–30 MB ≈ 0.4 GB | 0.5 GB | $0 |

So this spec buys back rollback from the console, at no cost, and faster builds.

While the deploys kept still include one built on the **previous** base, that
base's layers stay stored even after its version is deleted, because a kept image
still points at them. Storage sits at about 0.65 GB until three deploys have been
made on the new base, a few cents for that month.

Each deploy should also be faster by the time the Chromium install takes now.
Measure it from `gcloud builds list` before and after.

## 6. Checks

Before pushing, on a machine with Docker:

1. Build the base, then the app image from it, and run it with a `.env`. `GET
   /ping` answers, and one scoresheet of each registered type generates.
2. `RUN_PDF_E2E=1 npm run test:e2e` passes with `PUPPETEER_EXECUTABLE_PATH`
   pointing at a local Chromium.
3. `npm run verify` passes.

After the push deploys (outside event weeks):

4. `X-App-Version` on `GET /ping` reads the new version.
5. One scoresheet of each type from `tools/scoresheet-generator.html` matches one
   made from the same CSV before the change, page for page.
6. A sync (Control Center's **Resync**) publishes as before.
7. The new image's layers (as in §2.1) are about 20–30 MB above the base.
8. A day later, the repository's size
   (`gcloud artifacts repositories describe cloud-run-source-deploy --location=us-central1`)
   is under 0.5 GB once three deploys have been made on the base.

## 7. Keeping the base current

Chromium and Debian's security fixes reach the service only when the base is
rebuilt. Rebuild it about once a month, and when a Chromium or Node security
release matters, never in an event week:

1. Build a new dated tag (§4.1).
2. Point the `Dockerfile`'s `FROM` at it, bump the PATCH version, add the
   Changelog entry.
3. Push, then run §6's checks 4–6.

A scoresheet that renders differently after a rebuild is a Chromium change.
Point `FROM` back at the previous tag while it's still kept, or rebuild that tag
from the same `base-image/Dockerfile` if it isn't.

When this is built, these pages describe the base image instead of today's
`Dockerfile`: [Deployment](../../technical/deployment.md) (its "Runs on
`node:22-slim`" paragraph, the ignored-files filter, and this section's rebuild
steps), [Scoresheet pipeline](../../technical/scoresheet-pipeline.md), and the
workspace `CLAUDE.md`'s `sage-tools-api` section.

## 8. Open decisions

| # | Decision | Default if not decided |
| --- | --- | --- |
| D1 | How many app deploys the policy keeps once the base exists: 3 (rollback two steps from the console) or 1, as now (rollback only by reverting and pushing) | 3. With the base image both are free |
| D2 | Build §4.3 (`puppeteer-core`) | Only if it saves 5 MB or more |
| D3 | Keep the previous base as well as the current one, for a faster rollback across a base rebuild | No. Rebuilding an old tag takes a few minutes |
| D4 | How often the base is rebuilt | Monthly, outside event weeks |
