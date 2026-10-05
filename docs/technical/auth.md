# Auth

One shared operator login, not per-user accounts — `sage-tools-api/src/auth/`.

## How it works

`AuthService.mjs` hashes `"username:password"` **together** as one string,
never a separate username field — there's exactly one operator identity,
shared by whoever's running the console. `POST /auth/login` checks the
submitted credentials against `AUTH_PASSWORD_HASH` (an env var — never the
plaintext credentials themselves) and, on success, issues a short-lived
HMAC-signed token (no JWT library) instead of handing back the sync secret
directly. It expires `AUTH_TOKEN_TTL_MS` after it is issued — 12 hours, one
tournament day, when that variable is unset.

The hash itself is generated **locally**, offline, via
`node scripts/hash-password.mjs '<username>' '<password>'` — the plaintext
credentials never touch an env var or the deployed service; only the hash
does.

## Where the checks live

Every Express auth check is in `src/auth/middleware.mjs`; no route file has its
own. `createAuthMiddleware({ authService, syncSharedSecret })` returns four
checks, and each one calls `next()` on success, setting `req.actor`, and
passes an `UnauthorizedError` to the error handler otherwise (the 401 body and
its `WWW-Authenticate` header come from there):

| Check | Accepts | `req.actor` |
| --- | --- | --- |
| `requireOperator` | an operator bearer token | `{ kind: "operator" }` |
| `requireOperatorOrSyncSecret` | an operator token, or the `X-Sync-Secret` header | `{ kind: "operator" }` for a token |
| `requireOperatorOr("desk")` | an operator token, or a desk token | `{ kind: "desk", day }` for a desk token |
| `requireOperatorOr("scorer")` | an operator token, or a scorer token | `{ kind: "scorer", day }` for a scorer token |

A desk or scorer token is not an operator token, and an operator token is not a
scoped one, so the wrong kind gets the same 401 as no token. A new token scope is
one entry in the middleware's scope map beside its `AuthService` issue and verify
pair.

The shared secret is compared in constant time (`src/shared/safeEqual.mjs`: both
sides are hashed with SHA-256 and compared with `timingSafeEqual`, so neither
the content nor the length leaks). A secret that is not configured never matches,
not even an empty header.

## Two auth paths, by design

`POST /sync/:day` accepts **either** the raw shared secret
(`X-Sync-Secret` header) **or** an operator's bearer token. This is
deliberate, not redundant: Apps Script (the thing calling this endpoint on
every sheet edit) has no browser to sign into, so it authenticates with the
raw secret directly; Control Center authenticates with the
token it got from signing in. Same for `GET /sync/config`.

`POST /sync/:day/live` (the go-live override) and `POST /sync/live-push`
(the live push switch) accept the **operator token only** — no shared-secret
fallback, since Apps Script never calls these endpoints. This is what makes
the shared secret safe to bake into 15+ installed Apps Script projects: it
can only ever trigger a data sync, never flip the public site's visibility or
turn live push off.

All of these routes **fail closed** if neither auth path is configured or valid —
there's no unauthenticated fallback.

## Desk tokens

Attendance adds a second, narrower kind of token for desk staff
(`issueDeskToken` / `verifyDeskToken` in `AuthService`). A desk token
carries a day, expires at the end of that day in Manila time, and is accepted
only by the attendance mark route, and only while the event's `attendance`
setting is `"desks"`.

- It is signed with a key derived from `AUTH_TOKEN_SECRET`, so its signature
  never verifies as an operator token. Its payload also carries a `scope`,
  and `verify()` rejects any token with one, as a second guard.
- It is therefore a 401 on every `/sync/*` route and on the operator-only
  attendance routes (issuing desk links, updating the roster).
- Only an operator can issue one: `POST /v1/days/:day/attendance/desk-links`.

See [event attendance](event-attendance.md).

## Scorer tokens

Score entry adds a third kind of token, for scorer staff (`issueScorerToken` /
`verifyScorerToken`). It is built like a desk token with its own scope,
`"score-desk"`, and so its own derived signing key (`#signScoped(scope, payload)`
derives the key from the scope, for desk and scorer tokens alike). A scorer token
carries a day and expires 24 hours after it is issued, not at the end of the day.

- It never verifies as an operator token or a desk token, and neither of those
  verifies as a scorer token: the signatures differ by key, and `verify()` also
  rejects any payload carrying a `scope`.
- The score route (`PUT /v1/days/:day/facilities/:facility/matches/:matchNumber/score`)
  is the only route that accepts it. Every other route, including attendance's and
  every `/sync/*` route, answers 401. A scorer can enter, correct, clear and replace on
  a conflict, exactly as an operator can, and nothing else.
- Only an operator can issue one: `POST /v1/days/:day/scores/scorer-links`. It is refused
  once the day is over (06:00 Manila the morning after its `date`) and while the event's
  scoring setting is `"console"`.
- **The switch.** Each use of the score route checks the event's `scoreEntry`: a scorer
  token is accepted only while it is `"links"`, and only for its own day. Mission Control's
  **Accepting / Stopped** switch (`PUT /v1/events/:event/score-entry`, operator token only)
  moves the event between `"links"` and `"console"` in `events.json`, so **Stopped**
  refuses every scorer save with a 403 and every new link, within `SYNC_CONFIG_TTL_MS`
  (about a minute), without waiting for tokens to expire. Moving back to **Accepting**
  makes links issued earlier work again until they expire. The switch never turns score
  entry on or off: an event without a `scoreEntry` setting is refused.
- Nothing accepts the sync secret on these routes.

See [Scorer page](scorer-page.md) and [sync pipeline § Score entry
writes](sync-pipeline.md#score-entry-writes).

## What the console does with it

Signing in exchanges the operator's password for a session token held only
in that browser tab — the password itself is never stored. See [Mission
Control usage](../features/control-center.md#mission-control).

---
**Related env vars:** `AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`,
`SYNC_SHARED_SECRET` — see [Deployment](deployment.md).
