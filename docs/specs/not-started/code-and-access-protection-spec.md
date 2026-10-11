# Spec — code and access protection

> **Status: not started.** A to-do list, not a design: each item says what to
> change and where, and leaves the details to whoever builds it. Written
> 2026-10-11. Everything here is free or close to it. Items are ordered by
> value; each is independent and can be done on its own.
>
> This page is public. It states the rules to follow, not which of them are
> broken today: check each one when you pick it up.

## 1. What is protected, and what cannot be

The code worth protecting runs on the server: `sage-tools-api` (sync, auth,
score and attendance writes, the scoresheet pipeline and the Apps Script) is a
private repo. Keep it that way.

Anything a browser runs (`lib/v1/`, the tools, Control Center) is readable by
whoever opens the page, whatever repo it comes from. So:

- **No obfuscation or minification of the site.** It is reversed with free
  tools, costs a build step the site deliberately does not have, breaks the
  comparison harness and the `lib/v1` compatibility rule, slows pages on venue
  Wi-Fi and makes event-day debugging harder.
- **The Bracket Generator's draw stays in the browser on purpose.** Its
  verifiable draw ([spec](../implemented/bracket-generator-verifiable-draw-spec.md))
  depends on anyone being able to re-run it.
- **The pages and the engine stay in the browser.** They render live from
  snapshots, and the rules they hold are tournament rules players should see.
- **Control Center stays as it is.** It holds no secret and enforces nothing
  itself; the API decides every action.

Protection is therefore about **access**: who can edit the workbooks, who can
push, what a leaked credential can do, and what an attacker can cost us.

## 2. Workbook sharing (Google Drive)

Workbook IDs are public: the masters' IDs are in the Tournament Calculator and
every facility workbook's is in `event-data/config/events.json`.

- **Every link-shared workbook is "Anyone with the link: Viewer", never
  Editor.** That covers both masters (SAGE Dual Meet Master, SAGE Standard
  Tournament Master) and every event's facility workbooks, finished events
  included. Staff who edit are added by name. The calculator's handoff still
  works: **Make a copy** needs only view access.
- **Why it matters:** an editor can change scores (the onEdit sync publishes
  them), change the bound Apps Script that every future workbook is copied
  from, and read Script Properties, where `sheets-sync.gs` keeps the sync
  secret.
- **If a workbook holding the secret was ever editor-shared:** run **SAGE →
  Replace shared secret** in it, and if exposure can't be ruled out rotate
  `SYNC_SHARED_SECRET` in Cloud Run (every syncing workbook then needs the new
  one, so never close to an event).
- **Add it to the dry-run checklist**
  (`_templates/dry-run-checklist-template.md`): check each facility workbook's
  sharing before the event.

## 3. Cap what Cloud Run can cost

- Add `--max-instances` (e.g. `3`) to the deploy command in the `Dockerfile`
  comment and to the Cloud Run build trigger, and update
  [Deployment](../../technical/deployment.md). With `--concurrency 4`, check the
  cap still covers an event's peak sync load.
- Set a Google Cloud **budget alert** (e.g. $5–10/month) on the project. It
  only warns; it does not stop spending.
- Check the Cloudflare account for `sage-live` is on the free plan with no
  card-backed overage.

## 4. Rate-limit sign-in

- `POST /v3/sessions` and its frozen twin `POST /auth/login` throttle failed
  attempts, e.g. 5 per IP per 15 minutes, answering `429` (problem details on
  `/v3`, `{ error, code }` on the legacy route) with `Retry-After`.
- In-memory is enough: no new service, no new dependency. Per-instance counts
  are acceptable with the instance cap above.
- `sage-tools-api` rules apply: failing test first, a `package.json` bump
  (minor), a Changelog entry with its **Tests:** line, and the
  [API](../../technical/api.md) and [Auth](../../technical/auth.md) pages
  updated.

## 5. GitHub settings

Org `sage-match-control`, all repos:

- **Require two-factor authentication** for members.
- **Secret scanning with push protection** and **Dependabot alerts** on.
- **A ruleset on `main` of `sage-tools-api`**: no force-push, no deletion. A
  push there deploys to production, and the pre-push hook is skippable with
  `--no-verify`. Optionally the same for the site and `event-data`.
- Review who has write access to each repo.

## 6. Least-privilege credentials

- `GITHUB_TOKEN` (Cloud Run) is a **fine-grained token**: repository
  `event-data` only, Contents read/write, nothing else, with an expiry date and
  a calendar reminder to renew it outside event weeks.
- The `sage-tools-api-runtime` service account is an Editor only on facility
  workbooks it writes (attendance and score entry), not on the masters.
- `GOOGLE_SHEETS_API_KEY` is restricted to the Sheets API in the Google Cloud
  console.

## 7. Where the docs live

`sage-docs` is public and published to GitHub Pages, so the architecture, auth
design and API are readable by anyone. GitHub Pages on a private repo needs a
paid plan. Owner's decision, one of:

1. **Leave it public** (no cost, no change).
2. **Make the repo private and host the site on Cloudflare Pages** (free from a
   private repo; the account already exists for `sage-live`). The site URL
   changes, so every link to `sage-match-control.github.io/sage-docs/` has to
   follow.
3. **Split it:** publish `features/` and `usage/`, keep `technical/` and
   `specs/` in a private repo.

## 8. Licence notice

Add a `LICENSE` file (all rights reserved, no permission to copy or reuse) to
each public repo: `sage-match-control.github.io`, `event-data` and, if it stays
public, `sage-docs`. Unlicensed code is already all rights reserved; the file
makes that explicit.

## 9. Deferred

- **Tournament Calculator logic on the server.** Its planning core (tiered
  byes, wildcard fill, slot-aware finish times) is the one piece of front-end
  logic with real originality. It could move behind an endpoint on the
  `sage-live` Worker (free, no cold start), at the cost of offline use and a
  network call on every change. Not worth it unless it is actually copied.
- **Private site repo.** Moving the site to Cloudflare Pages would let
  `sage-match-control.github.io` go private, but only hides its history,
  templates and tests: the served code stays readable.
