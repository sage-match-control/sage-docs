# Spec — lean deploy image

> **Status: not started.** Written 2026-10-26 against `sage-tools-api` 3.4.0;
> rewritten 2026-11-23 against 3.9.0 (commit `38bab1a`) and the build trigger's
> own build config, after two test builds on Cloud Build (§11). Every decision is
> taken (§3). The cleanup policy in §2.3 is already set (`keepCount` 1); nothing
> else here is built. **Build it only when the owner says so**, and never within
> 3 days of an event. The runbook (§7) is written to be followed without any
> other context.

Make each deploy of `sage-tools-api` store about 16 MB in Artifact Registry
instead of 382 MB, keep the last **2** deploys so a bad deploy can be rolled back
from the Cloud Run console, stay inside the free 0.5 GB, and make each deploy
about 80 seconds faster. Nothing the service does changes.

---

## 1. Why

Every push to `main` makes a Cloud Build trigger build a new image of
`sage-tools-api` and store it in the Artifact Registry repository
`cloud-run-source-deploy` (project `sage-tools-api`, region `us-central1`).
Artifact Registry gives 0.5 GB of storage free per month and charges about
$0.10 per GB-month above it.

The trigger's build runs `docker build --no-cache` (§2.5). Without the cache, the
`apt-get install chromium` step runs again on every deploy and makes a **new**
269 MB layer, byte-for-byte different even when nothing in it changed, so no two
deploys share it. That is how 74 deploys reached 25.5 GB before the cleanup
policy (§2.3) existed. The policy now keeps one deploy (382 MB), which fits the
free allowance only because nothing else is kept, so the only rollback is
reverting the commit and pushing.

With the cache on, a deploy reuses the previous image's Node and Chromium layers
(350 MB, stored once) and adds only its npm packages and code.

## 2. Where things stand

### 2.1 What is in the image

The compressed layers of the live image (3.9.0, built 2026-11-22):

| Layers | Size | Changes when |
| --- | --- | --- |
| `node:22-slim` (Debian bookworm and Node) | 81 MB | Docker publishes a new `node:22-slim` (last: 2026-11-11) |
| `apt-get install chromium fonts-liberation ca-certificates` | 269 MB | every deploy today, because of `--no-cache` |
| `npm install --omit=dev` | 31.8 MB | every deploy: `package.json`'s version is bumped with every code change |
| `index.mjs`, `src/`, `templates/` | 0.8 MB | every deploy |
| **Total** | **382 MB** | |

The npm layer holds 85 MB (unpacked) of `node_modules` and 24 MB of npm's own
download cache, `/root/.npm`, which the service never reads.

### 2.2 What one deploy stores

Each deploy adds **one image** to the package `sage-tools-api/sage-tools-api`,
tagged with the commit SHA. Cloud Build attaches two provenance records to it
(in-toto attestations, under 20 KB each). A version listing shows all three, but
a cleanup rule's `keepCount` counts only the image, and the attachments are
deleted with it.

A deploy made by hand with the `Dockerfile`'s fallback command
(`gcloud run deploy … --source .`) stores its image in a different package,
`sage-tools-api`, which the policy deletes within a day.

### 2.3 The cleanup policy

Set on the repository, not a dry run:

```json
[
  { "name": "delete-all", "action": { "type": "Delete" }, "condition": { "tagState": "any" } },
  { "name": "keep-last-deploy", "action": { "type": "Keep" },
    "mostRecentVersions": { "packageNamePrefixes": ["sage-tools-api/sage-tools-api"], "keepCount": 1 } }
]
```

It keeps the newest image of the trigger's package with its attachments, and
deletes everything else. Artifact Registry applies it about once a day.
`keepCount` is a number of deploys, since attachments are not counted. A tag does
not protect an image (`tagState` is `any`).

### 2.4 What Cloud Run needs from the registry

Cloud Run copies an image when it deploys a revision and "keeps this copy while a
serving revision uses it"; instances never pull from Artifact Registry
([Cloud Run: deploying](https://docs.cloud.google.com/run/docs/deploying)). The
docs promise nothing for a revision that has stopped serving. So:

- Deleting the **live** revision's image from Artifact Registry does not stop it
  serving or block a cold start.
- Routing traffic back to an **older** revision is only safe while that
  revision's image is still in the registry. With `keepCount` 2, that is the one
  deploy before the live one.

### 2.5 The trigger today

| | |
| --- | --- |
| Name | `rmgpgab-sage-tools-api-us-central1-sage-match-control-sage-tjfv` (region `global`) |
| Fires on | a push to `main` of `sage-match-control/sage-tools-api`, except one touching only the ignored files (`apps-script/**`, `live-worker/**`, `test/**`, `scripts/**`, `.githooks/**`, `**/*.md`, `.env.example`, `jsconfig.json`) |
| Runs as | `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com` |
| Build config | **inline**, the one Cloud Run's "continuous deployment" setup generated (Appendix A) |

Three steps:

| Step | Runs | Time (last 12 builds) |
| --- | --- | --- |
| `Build` | `docker build --no-cache -t <image>:$COMMIT_SHA . -f Dockerfile` | 71–93 s |
| `Push` | `docker push <image>:$COMMIT_SHA` | 43–55 s |
| `Deploy` | `gcloud run services update sage-tools-api --image=<image>:$COMMIT_SHA …` with every environment variable | 27–82 s |

`<image>` is `$_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME`,
which resolves to
`us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api/sage-tools-api`.
`$REPO_NAME` (`sage-tools-api`), `$COMMIT_SHA` and `$BUILD_ID` are Cloud Build's
own; the `$_…` values are the trigger's **substitution variables**: `_AR_HOSTNAME`,
`_AR_PROJECT_ID`, `_AR_REPOSITORY`, `_SERVICE_NAME`, `_DEPLOY_REGION`, `_PLATFORM`,
`_TRIGGER_ID`, and the production secrets and settings (`_GITHUB_TOKEN`,
`_SYNC_SHARED_SECRET`, `_GOOGLE_SHEETS_API_KEY`, `_SHEETS_FETCH_TIMEOUT_MS`,
`_AUTH_PASSWORD_HASH`, `_AUTH_TOKEN_SECRET`, `_AUTH_TOKEN_TTL_MS`,
`_LIVE_PUSH_URL`, `_LIVE_PUSH_SECRET`), stored **in plaintext**.

The inline build config carries a second, older set of substitutions of its own.
Its `_GITHUB_TOKEN` and `_SYNC_SHARED_SECRET` differ from the trigger's, and the
live service runs with the **trigger's** values (checked 2026-11-23 by comparing
hashes): trigger values override a build config's. Moving the build config into
the repository (§5.2) drops the stale inline set and changes nothing the service
gets.

The `docker` builder image runs Docker 20.10.24 with `DOCKER_BUILDKIT` unset, so
`docker build` uses the **classic builder** (`buildx` is installed but unused).
The classic builder's `--cache-from` reuses a layer when the pulled image has the
same parent layers and the same instruction, or, for a `COPY`, the same file
contents.

## 3. Decisions (taken 2026-11-23)

| # | Decision | Taken |
| --- | --- | --- |
| D1 | How many deploys the policy keeps | **2**: the live one and the one before, for a one-step rollback from the console |
| D2 | Where the build config lives | **In the repository**, as `sage-tools-api/cloudbuild.yaml`, with the trigger pointing at it, so a change to it is reviewed and versioned. It names the substitutions and holds no secret |
| D3 | Clear npm's download cache in the image | **Yes**: `npm cache clean --force` after `npm install`, halving what each deploy stores (31.8 → 15.4 MB) |
| D4 | Who changes the trigger | **The owner, by hand in the Cloud Console** (§7 step 6). Nobody exports or imports the trigger with `gcloud`: that writes its plaintext secrets to a file |
| D5 | When it is built | **When the owner says so**, and never within 3 days of an event |

## 4. Goals and non-goals

**Goals**

- Each deploy stores about 16 MB; two deploys total about 383 MB, inside 0.5 GB.
- A deploy builds and pushes about 80 s faster.
- The previous deploy stays in the registry, so a bad deploy is rolled back from
  the Cloud Run console.
- The service behaves exactly as before. Every scoresheet type renders the same PDF.

**Not doing**

| Option | Why not |
| --- | --- |
| A hand-built base image (`sage-tools-api-base`, Node and Chromium, dated tags) that the `Dockerfile` builds `FROM` | Stores the same layers as the cache but adds a second image, a new folder, a keep rule for the base and a monthly rebuild by hand. The cache shares the layers with no extra image and refreshes Chromium on its own (§9). It stays the fallback if the cache doesn't hit (§8). |
| Keeping deploys without the cache (≈ 0.76 GB for 2) | About $0.03 a month, and builds stay slow. |
| `puppeteer-core` instead of `puppeteer` | Would shave a few MB off the npm layer at the price of a code change (the browser path through `loadConfig`). D3 saves more for one line. |
| Chrome's headless-only build (`chrome-headless-shell`) | About 100 MB smaller, but a different browser build can render fonts and page breaks differently. With the layer shared, its size stops mattering. |
| An Alpine base | Alpine's Chromium is built against musl and lags Puppeteer's supported version. Same rendering risk, no saving once layers are shared. |
| Scoresheets in a service of their own | Stores the same Chromium layer anyway and adds a second service to deploy. |

## 5. Design

### 5.1 The `Dockerfile`

One change, in `sage-tools-api/Dockerfile`: the npm step also clears npm's cache.
Replace

```dockerfile
RUN npm install --omit=dev
```

with

```dockerfile
# npm's download cache (/root/.npm, about 24 MB) is never read at runtime;
# clearing it halves the layer each deploy stores.
RUN npm install --omit=dev && npm cache clean --force
```

and add one line under the existing deploy-command comment at the top:

```dockerfile
# The build trigger runs cloudbuild.yaml (cached from the last deploy); the
# command above is the manual fallback only.
```

Nothing else in the `Dockerfile` changes: in particular the `FROM` line and the
`apt-get` line must stay exactly as they are, or the first build can't reuse the
live image's Chromium layer.

### 5.2 `cloudbuild.yaml`

A new file, `sage-tools-api/cloudbuild.yaml`. Its `Deploy` step, `images`,
`tags` and `options` are the inline config's (Appendix A), unchanged:

```yaml
# Cloud Build config for the sage-tools-api trigger (a push to main). See
# sage-docs/docs/specs/.../lean-deploy-image-spec.md.
# The $_… values are the trigger's substitution variables; the secrets live there,
# never in this file. _CACHE_TAG=none forces a build without the cache.
substitutions:
  _CACHE_TAG: latest
options:
  logging: CLOUD_LOGGING_ONLY
  substitutionOption: ALLOW_LOOSE
tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
  - sage-tools-api
images:
  - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
steps:
  - id: Pull cache
    name: gcr.io/cloud-builders/docker
    entrypoint: bash
    args:
      - -c
      - docker pull $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$_CACHE_TAG || exit 0
  - id: Build
    name: gcr.io/cloud-builders/docker
    args:
      - build
      - --cache-from
      - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$_CACHE_TAG
      - --build-arg
      - BUILDKIT_INLINE_CACHE=1
      - -t
      - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
      - -t
      - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:latest
      - .
      - -f
      - Dockerfile
  - id: Push
    name: gcr.io/cloud-builders/docker
    args:
      - push
      - --all-tags
      - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME
  - id: Deploy
    name: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
    entrypoint: gcloud
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - --platform=managed
      - --image=$_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
      - --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID
      - --region=$_DEPLOY_REGION
      - --quiet
      - --set-env-vars=SCORESHEET_CONCURRENCY=8,CORS_ORIGIN=https://sage-match-control.github.io,GITHUB_OWNER=sage-match-control,GITHUB_REPO=event-data,GITHUB_BRANCH=main,GITHUB_TOKEN=$_GITHUB_TOKEN,SYNC_SHARED_SECRET=$_SYNC_SHARED_SECRET,GOOGLE_SHEETS_API_KEY=$_GOOGLE_SHEETS_API_KEY,SHEETS_FETCH_TIMEOUT_MS=$_SHEETS_FETCH_TIMEOUT_MS,AUTH_PASSWORD_HASH=$_AUTH_PASSWORD_HASH,AUTH_TOKEN_SECRET=$_AUTH_TOKEN_SECRET,AUTH_TOKEN_TTL_MS=$_AUTH_TOKEN_TTL_MS,LIVE_PUSH_URL=$_LIVE_PUSH_URL,LIVE_PUSH_SECRET=$_LIVE_PUSH_SECRET
```

What each new piece does:

- **`Pull cache`** fetches the last deploy's image, tagged `latest` (about 20 s).
  `|| exit 0` lets the build go on without it (a forced refresh, §9, or a missing
  `latest`), which is then a full build like today's.
- **`Build`** has no `--no-cache` and reuses every layer whose parents and
  instruction match the pulled image: `node:22-slim` and the Chromium install,
  unless `node:22-slim` has moved since. The `COPY package.json` layer and
  everything after it rebuild every time, because the version is bumped with
  every code change. `BUILDKIT_INLINE_CACHE=1` keeps the cache working if Cloud
  Build's Docker ever switches to BuildKit (from the second build after the
  switch). Today it prints a harmless line:
  `[Warning] One or more build-args [BUILDKIT_INLINE_CACHE] were not consumed`.
- **`latest`** always names the last deploy, so the next build pulls it. `Deploy`
  still deploys `:$COMMIT_SHA`, so each Cloud Run revision still names its commit.
- **`Push --all-tags`** pushes both tags. The registry already has the shared
  layers (`Layer already exists`), so only the npm and code layers upload.
- The file's `_CACHE_TAG: latest` is a default. A run that passes
  `_CACHE_TAG=none` overrides it (§9).

`cloudbuild.yaml` is not in the trigger's ignored files, so a push that changes it
builds and deploys, which is what a change to the build should do.

### 5.3 The trigger

The owner changes one setting in the Console (§7 step 6): the build config moves
from **Inline** to **Cloud Build configuration file**, location **Repository**,
path `cloudbuild.yaml`. Its event, branch, ignored files, substitution variables
and service account stay as they are.

### 5.4 The cleanup policy

`keepCount` goes from 1 to **2**, before the first cached deploy (§7 step 2), so
the 3.9.0 image (or whatever is live then) stays for rollback. The next image
shares its 350 MB, so this costs nothing.

### 5.5 Versioning

The `Dockerfile` and `cloudbuild.yaml` changes are one **patch** release of
`sage-tools-api`: the next patch version in `package.json` and a Changelog entry
in `README.md`. No test covers the build, so `test/` does not change; this is the
one kind of change the "failing test first" rule can't apply to, and the entry
says so (§7 step 4 gives its text).

## 6. Size and time

Measured on Cloud Build (§11). Every image built with the cache shares 350 MB
(`node:22-slim` 81 MB plus Chromium 269 MB). With D3, each deploy adds 16.3 MB
of its own (npm 15.4 MB, code 0.8 MB, plus a few KB).

| | Stored | Per month |
| --- | --- | --- |
| Now: last deploy only, no cache | 382 MB | $0 |
| First cached deploy, beside the 3.9.0 image | 382 + 16.3 ≈ 399 MB | $0 |
| Steady state: 2 cached deploys | 350 + 2 × 16.3 ≈ 383 MB | $0 |
| For up to 2 deploys after `node:22-slim` moves or a forced refresh (§9) | 2 × 350 + 2 × 16.3 ≈ 733 MB | under a cent |

After a change to the shared layers, the old 350 MB stays until both kept deploys
have been built on the new layers, which is two deploys.

| | Pull cache | Build | Push | Build + push |
| --- | --- | --- | --- | --- |
| Today (last 12 builds) | — | 71–93 s | 43–55 s | 115–148 s |
| With the cache (§11, build 1) | 22 s | 19 s | 8 s | 49 s |
| Forced refresh, no cache (§11, build 2) | 2 s | 104 s | 50 s | 156 s |

`Deploy` (27–82 s) doesn't change.

## 7. Runbook

**Owner** marks the steps the owner does; every other step is the implementer's.
Run the commands in PowerShell, from the workspace root `D:\Personal\SAGE`,
signed in to `gcloud` as the owner's account with project `sage-tools-api`
(`gcloud config list` shows it). Stop at any step whose result differs from what
it says, and tell the owner.

1. **Ask the owner to go ahead**, and for the date of the next event. Do nothing
   further unless the owner says go and the event is more than 3 days away.

2. **Keep 2 deploys.** Write this to a file outside the repositories, for example
   `$env:TEMP\cleanup-policy.json`:

   ```json
   [
     { "name": "delete-all", "action": { "type": "Delete" }, "condition": { "tagState": "any" } },
     { "name": "keep-last-deploy", "action": { "type": "Keep" },
       "mostRecentVersions": { "packageNamePrefixes": ["sage-tools-api/sage-tools-api"], "keepCount": 2 } }
   ]
   ```

   ```powershell
   gcloud artifacts repositories set-cleanup-policies cloud-run-source-deploy --location=us-central1 --policy="$env:TEMP\cleanup-policy.json" --no-dry-run
   ```

   ```powershell
   gcloud artifacts repositories describe cloud-run-source-deploy --location=us-central1 --format=json
   ```

   The second command's `cleanupPolicies` shows `keepCount: 2`.

3. **Seed the cache.** Find the live image and tag it `latest`:

   ```powershell
   gcloud run services describe sage-tools-api --region us-central1 --format='value(spec.template.spec.containers[0].image)'
   ```

   It prints `…/sage-tools-api/sage-tools-api:<sha>`. Then, with that `<sha>`:

   ```powershell
   gcloud artifacts docker tags add us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api/sage-tools-api:<sha> us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api/sage-tools-api:latest
   ```

   Record the repository's size now, for check 2:

   ```powershell
   gcloud artifacts repositories describe cloud-run-source-deploy --location=us-central1 --format='value(sizeBytes)'
   ```

4. **Make the repository change** in `sage-tools-api`, on `main`, up to date with
   `origin/main` and with a clean working tree:
   - `Dockerfile` as in §5.1.
   - `cloudbuild.yaml` as in §5.2, character for character.
   - `package.json`: the next patch version.
   - `README.md`'s Changelog, at the top of the entries:

     ```markdown
     ### <version> — cached builds

     - The build trigger now runs `cloudbuild.yaml` from this repository instead of an inline config.
       It pulls the last deploy's image (`latest`) and builds `--cache-from` it, so a deploy reuses the
       Node and Chromium layers instead of reinstalling Chromium (`--no-cache` is gone), and pushes both
       its commit tag and `latest`. The `Deploy` step and every environment variable are unchanged.
       `_CACHE_TAG=none` forces a build without the cache.
     - The `Dockerfile`'s npm step clears npm's download cache, halving the layer each deploy stores.
     - **Tests:** none. No test covers the `Dockerfile` or the build config, so this change has no
       failing test first; it is checked by the production checks in lean-deploy-image-spec §8.
     ```

   - `README.md`'s paragraph that starts "A push to `main` deploys Cloud Run
     through a Cloud Build trigger": add a sentence saying the trigger runs
     `cloudbuild.yaml`, and that a change to it builds and deploys.

   Run `npm test`, then `npm run verify`; both pass. Commit (message
   `<version>: cached builds (cloudbuild.yaml, --cache-from latest, npm cache cleared)`).
   **Do not push yet.**

5. **Tell the owner the commit is ready** and ask for step 6.

6. **Owner**, in the Cloud Console (project `sage-tools-api`): **Cloud Build →
   Triggers →** `rmgpgab-sage-tools-api-us-central1-sage-match-control-sage-tjfv`
   **→ Edit**.
   - **Configuration**: Type **Cloud Build configuration file (yaml or json)**,
     Location **Repository**, file location `cloudbuild.yaml`.
   - Leave **Event**, **Source**, the ignored files, the **Substitution
     variables** and the **Service account** as they are.
   - **Save**, and tell the implementer it is done.

   Until the push in step 7, nobody pushes to `sage-tools-api`'s `main`: the
   trigger now expects a file that isn't there yet.

7. **Push** the commit from step 4. The pre-push hook runs the tests again. The
   push starts a build; find it and wait for it to finish (about 2 minutes):

   ```powershell
   gcloud builds list --region=global --limit=1 --format='table(id,createTime,status)'
   ```

   Repeat until `STATUS` is `SUCCESS` (or `FAILURE`: go to §10). Then run the
   checks in §8.

8. **Docs**, once every check in §8 passes:
   - `sage-docs/docs/technical/deployment.md`, section **Cloud Run deploy**:
     replace "The build/deploy trigger runs the equivalent of: …" with a
     description of `cloudbuild.yaml`'s four steps (pull the last deploy's image,
     build with its cache, push the commit tag and `latest`, deploy), keep the
     `gcloud run deploy --source .` command as the **manual fallback** only, and
     add two short subsections: **Image storage and rollback** (the cleanup
     policy keeps 2 deploys; roll back one step from the Cloud Run console's
     Revisions tab) and **Refreshing Chromium** (§9's command). In **Which
     pushes build**, add that a change to `cloudbuild.yaml` builds.
   - The workspace `CLAUDE.md` (`D:\Personal\SAGE\CLAUDE.md`, repository
     `sage-workspace`), section `sage-tools-api`, the paragraph "Deploys
     automatically on push to `main` via a Cloud Run build trigger …": say the
     trigger runs `cloudbuild.yaml`, cached from the last deploy, and that the
     `Dockerfile` comment's command is the manual fallback.
   - Move this spec from `not-started/` to `implemented/` by the procedure in
     `sage-docs/docs/specs/README.md` ("Moving a spec between folders"), set its
     status line to **Implemented** with the version and date, and add a MINOR
     entry to `sage-docs/docs/changelog.md` (and the same version in
     `mkdocs.yml`) covering both pages.

9. **The next day**, check the size (§8 check 6).

## 8. Checks

After the build in step 7:

1. **It deployed.** The live image is the new commit, and `/ping` answers the new
   version:

   ```powershell
   gcloud run services describe sage-tools-api --region us-central1 --format='value(status.url,spec.template.spec.containers[0].image)'
   ```

   ```powershell
   curl.exe -sI <url>/ping
   ```

   `X-App-Version` is the version from step 4.

2. **The cache hit.** The repository grew by about 16 MB, not about 285 MB:

   ```powershell
   gcloud artifacts repositories describe cloud-run-source-deploy --location=us-central1 --format='value(sizeBytes)'
   ```

   Compare with the size from step 3: under 50 MB more means the Chromium layer
   was reused. (If a package named `cache-test` from the 2026-11-23 test is still
   listed, `gcloud artifacts packages list --repository=cloud-run-source-deploy --location=us-central1`,
   its size was in both readings.) The build's log in the Console (**Cloud Build →
   History →** the build **→ Build**) shows `Using cache` under
   `Step 2/12 : RUN apt-get update && apt-get install …`.

3. **It was faster.** `Build` plus `Push` took under a minute:

   ```powershell
   gcloud builds describe <build id> --region=global --format='json(steps[].id,steps[].timing)'
   ```

4. **`latest` moved.** Both tags are on the new image:

   ```powershell
   gcloud artifacts docker tags list us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api/sage-tools-api
   ```

   `latest` and the new commit SHA show the same digest.

5. **Scoresheets render.** Save this as `$env:TEMP\check.csv`:

   ```csv
   matchNumber,teamCode1,team1Player1,team1Player2,teamCode2,team2Player1,team2Player2
   1,AAA,Ann Cruz,Ben Diaz,BBB,Cy Eng,Di Fox
   2,CCC,Gus Hall,Ivy Jin,DDD,Kai Lee,Mo Ng
   3,EEE,Ola Pak,Qin Ray,FFF,Sol Tan,Uma Vo
   ```

   and post it once for each registered type, `scoresheet`, `scoresheet-no-ref`
   and `scoresheet-no-ref-wide` (no sign-in needed), changing `type=` and the
   output file's name each time:

   ```powershell
   curl.exe -s -o "$env:TEMP\check-scoresheet.pdf" -w '%{http_code} %{content_type} %{size_download}' -H 'Accept: application/pdf' -F evt=Check -F type=scoresheet -F blanks=0 -F "csv=@$env:TEMP\check.csv" <url>/v3/scoresheets
   ```

   Each answers `200 application/pdf` and more than 1,000 bytes. The first may
   take 20–30 s (Chromium's cold start).

6. **The next day**, the size is about 383–400 MB and the package
   `sage-tools-api/sage-tools-api` lists two images (step 3's command and
   `gcloud artifacts docker images list us-central1-docker.pkg.dev/sage-tools-api/cloud-run-source-deploy/sage-tools-api/sage-tools-api`).

## 9. Keeping Chromium current

Today every deploy installs the newest Chromium. With the cache, Chromium changes
only when `node:22-slim` moves (Docker republishes it for Node and Debian
updates, every few weeks) or on purpose:

- **To refresh on purpose**, outside event weeks, run the trigger on `main`
  without the cache; it rebuilds and redeploys the same commit with the newest
  Chromium:

  ```powershell
  gcloud builds triggers run rmgpgab-sage-tools-api-us-central1-sage-match-control-sage-tjfv --region=global --branch=main --substitutions=_CACHE_TAG=none
  ```

  Or in the Console: **Cloud Build → Triggers → Run** on the trigger, with the
  substitution `_CACHE_TAG` set to `none`. The `Pull cache` step then fails
  harmlessly (`manifest unknown`) and the build starts from nothing.
- A scoresheet that renders differently after a refresh is a Chromium change:
  roll back one revision (§10) and report it.

The service renders only its own templates, never a page from outside, so a
Chromium security fix rarely matters to it; one that does is a reason to refresh.

## 10. If something goes wrong

| What | Effect | Do |
| --- | --- | --- |
| The build fails in `Pull cache`, `Build` or `Push` | Nothing deploys; the live revision keeps serving | Read the build log in the Console. Fix `cloudbuild.yaml` and push. If that can't be done quickly, the **owner** switches the trigger back to **Inline** and pastes Appendix A; the trigger's substitution variables are still there |
| The build fails in `Deploy` | The new image is stored, the live revision keeps serving | Same as above |
| It deploys but the service misbehaves | The new revision serves | **Cloud Run → sage-tools-api → Revisions →** the previous revision **→ Manage traffic → 100%**. Its image is still kept (`keepCount` 2). Then revert the commit and push |
| Check 2 shows about 285 MB of growth | The cache missed: storage and time are as today, nothing is broken | Confirm in the build log which step stopped using the cache. If it was `FROM` (`node:22-slim` moved between step 3 and step 7), the next deploy will hit. If not, report it; the base-image design in §4 is the fallback |

## 11. Tested 2026-11-23

Two builds with `gcloud builds submit`, from commit `38bab1a`'s files with
`package.json`'s version bumped as the next deploy's would be, ran §5.2's
`Pull cache`, `Build` and `Push` steps. They pushed to a throwaway package,
`cache-test/sage-tools-api`, which the cleanup policy deletes, and deployed nothing.

1. **Cache from the live image** (`:38bab1a…`): the `Build` log showed
   `Using cache` for the Chromium install and every step before the
   `package.json` copy. The new image's `node:22-slim` and Chromium layers have
   the live image's digests (Chromium `49de7d1b4336`). `Push` reported them as
   `Layer already exists` and uploaded the other 32.7 MB, and the repository grew
   by 32.6 MB. The cache hit although Docker Hub's copy of `node:22-slim`'s first
   layer has a different compressed digest from the stored one, because the cache
   compares contents.
2. **A forced refresh** (`_CACHE_TAG=none`), with D3's `npm cache clean --force`:
   the pull failed (`manifest unknown`) and the build went on, installed Chromium
   156.0.8078.79 afresh and stored a new 269 MB layer. The npm layer came to
   15.4 MB, with `/root/.npm` down to 1 MB.

Untested: the trigger reading `cloudbuild.yaml` from the repository, the `Deploy`
step from that file, and `gcloud builds triggers run --substitutions`. Checks 1–4
cover the first two on the first real deploy.

## Appendix A — the inline build config before this change

What the trigger ran up to this change, without its substitutions (the trigger
keeps those). Pasting it back as the trigger's **Inline** config undoes §7 step 6.

```yaml
steps:
  - id: Build
    name: gcr.io/cloud-builders/docker
    args: [build, --no-cache, -t, '$_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA', ., -f, Dockerfile]
  - id: Push
    name: gcr.io/cloud-builders/docker
    args: [push, '$_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA']
  - id: Deploy
    name: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
    entrypoint: gcloud
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - --platform=managed
      - --image=$_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
      - --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID
      - --region=$_DEPLOY_REGION
      - --quiet
      - --set-env-vars=SCORESHEET_CONCURRENCY=8,CORS_ORIGIN=https://sage-match-control.github.io,GITHUB_OWNER=sage-match-control,GITHUB_REPO=event-data,GITHUB_BRANCH=main,GITHUB_TOKEN=$_GITHUB_TOKEN,SYNC_SHARED_SECRET=$_SYNC_SHARED_SECRET,GOOGLE_SHEETS_API_KEY=$_GOOGLE_SHEETS_API_KEY,SHEETS_FETCH_TIMEOUT_MS=$_SHEETS_FETCH_TIMEOUT_MS,AUTH_PASSWORD_HASH=$_AUTH_PASSWORD_HASH,AUTH_TOKEN_SECRET=$_AUTH_TOKEN_SECRET,AUTH_TOKEN_TTL_MS=$_AUTH_TOKEN_TTL_MS,LIVE_PUSH_URL=$_LIVE_PUSH_URL,LIVE_PUSH_SECRET=$_LIVE_PUSH_SECRET
images:
  - $_AR_HOSTNAME/$_AR_PROJECT_ID/$_AR_REPOSITORY/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
options:
  substitutionOption: ALLOW_LOOSE
  logging: CLOUD_LOGGING_ONLY
tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
  - sage-tools-api
```
