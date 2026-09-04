# Auth

One shared operator login, not per-user accounts — `sage-tools-api/src/auth/`.

## How it works

`AuthService.mjs` hashes `"username:password"` **together** as one string,
never a separate username field — there's exactly one operator identity,
shared by whoever's running the console. `POST /auth/login` checks the
submitted credentials against `AUTH_PASSWORD_HASH` (an env var — never the
plaintext credentials themselves) and, on success, issues a short-lived
HMAC-signed token (no JWT library) instead of handing back the sync secret
directly.

The hash itself is generated **locally**, offline, via
`node scripts/hash-password.mjs '<username>' '<password>'` — the plaintext
credentials never touch an env var or the deployed service; only the hash
does.

## Two auth paths, by design

`POST /sync/:day` accepts **either** the raw shared secret
(`X-Sync-Secret` header) **or** an operator's bearer token. This is
deliberate, not redundant: Apps Script (the thing calling this endpoint on
every sheet edit) has no browser to sign into, so it authenticates with the
raw secret directly; Control Center authenticates with the
token it got from signing in. Same for `GET /sync/config`.

`POST /sync/:day/live` (the go-live override) accepts the **operator
token only** — no shared-secret fallback, since Apps Script never calls
this endpoint. This is what makes the shared secret safe to bake into 15+
installed Apps Script projects: it can only ever trigger a data sync, never
flip the public site's visibility.

Both routes **fail closed** if neither auth path is configured or valid —
there's no unauthenticated fallback.

## What the console does with it

Signing in exchanges the operator's password for a session token held only
in that browser tab — the password itself is never stored. See [Mission
Control usage](../features/control-center.md#mission-control).

---
**Related env vars:** `AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`,
`SYNC_SHARED_SECRET` — see [Deployment](deployment.md).
