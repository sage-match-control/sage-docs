# API

The HTTP surface of `sage-tools-api`. There is one REST surface, **`/v3`**, and
the older URLs, which are kept exactly as they are and never removed. A new
client uses `/v3`. `GET /openapi.json` (importable into Postman with **Import →
Link**) documents all of it, generated from the route files, so it cannot drift
from what is deployed.

## The surfaces

| Surface | Paths | Status |
| --- | --- | --- |
| Health | `GET /ping` | outside any version, never changes |
| `/v3` | resource-named routes, the table below | the one to build against; released in 3.0.0 |
| `/v1` | attendance and score entry, from 2.6.0 and 2.8.0 | frozen |
| legacy | `/sync/...`, `/scoresheets/...`, `/auth/login` | frozen |

**Frozen means identical**: same URL, same method, same auth, same status, same
body. The routes exist because Apps Script (pasted into every installed
workbook), archived event pages and a cached copy of Control Center call them.
Each of them answers two extra headers (RFC 9745) that point at its `/v3` twin:

```
Deprecation: @<unix seconds of the 3.0.0 release>
Link: </v3/days/day1/visibility>; rel="successor-version"
```

The `Link` is the twin's path with that request's parameters filled in. No
`Sunset` header is sent, because these routes never go away. OpenAPI marks each
`deprecated: true`.

**There is no `/v2`.** The URL version is the package's major version, so the
surface of 3.x is `/v3`; `/v1` is the frozen partial REST surface of the
2.x releases. Nothing answers under `/v2`.

### The versioning rule

- Adding a route, an optional request member or a response member is not
  breaking: it is a minor release, and stays on `/v3`.
- Renaming or removing a route or member, changing a type, or changing the
  status code for the same outcome is breaking. It is allowed only as a new
  major release with a new URL surface (4.0.0 and `/v4`) that serves beside
  `/v3`. A major release never ships without its new surface, so the package
  major and the URL version stay equal.
- An old URL is never removed.

## `/v3` routes

Auth: **secret** is the `X-Sync-Secret` header (Apps Script), **operator** is a
signed-in operator's bearer token, **desk** and **scorer** are the narrower
day-scoped tokens (see [Auth](auth.md)).

| Method and path | What it does | Auth | Success | Frozen twin |
| --- | --- | --- | --- | --- |
| `POST /v3/days/{day}/syncs` | sync every facility of the day | secret or operator | `200` sync report | `POST /sync/:day` |
| `POST /v3/days/{day}/facilities/{facility}/syncs` | sync one facility (what Apps Script calls on an edit) | secret or operator | `200` sync report | `POST /sync/:day?facility=` |
| `GET /v3/days/{day}/visibility` | read the day's go-live setting | operator | `200 { day, label, isLive }` | — |
| `PUT /v3/days/{day}/visibility` | set it, and republish | operator | `200 { day, label, isLive, republished, live?, archive? }` | `POST /sync/:day/live` |
| `GET /v3/settings/live-push` | read the live-push switch | operator | `200 { enabled, available }` | — |
| `PUT /v3/settings/live-push` | set it | operator | `200 { enabled, changed, available }` | `POST /sync/live-push` |
| `GET /v3/diagnostics/sync` | sync config and live-push diagnostics | secret or operator | `200` | `GET /sync/config` |
| `POST /v3/scoresheets` | generate a scoresheet PDF | none | `200` PDF or NDJSON | `POST /scoresheets/generate`, `…/generate/stream` |
| `POST /v3/sessions` | sign in | none | **`201`** `{ token, expiresAt }` | `POST /auth/login` (`200`) |
| `GET /v3/events/{event}/score-entry` | read the event's score-entry mode | operator | `200 { event, scoreEntry }` (`"console"`, `"links"` or `null`) | — |
| `PUT /v3/events/{event}/score-entry` | set it | operator | `200 { event, scoreEntry, changed }` | `PUT /v1/events/:event/score-entry` |
| `PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance` | mark one person present or not | operator or desk | `200` the row | `PUT /v1/days/:day/facilities/:facility/attendance/:key` |
| `POST /v3/days/{day}/attendance/desk-links` | issue a desk link | operator | `201` | `POST /v1/days/:day/attendance/desk-links` |
| `POST /v3/days/{day}/attendance/reconciliations` | update the `ATTENDANCE` roster | operator | `200` | `POST /v1/days/:day/attendance/reconciliations` |
| `PUT /v3/days/{day}/facilities/{facility}/matches/{matchNumber}/score` | enter, correct or clear a score | operator or scorer | `200` | `PUT /v1/days/:day/facilities/:facility/matches/:matchNumber/score` |
| `POST /v3/days/{day}/scores/scorer-links` | issue a scorer link | operator | `201` | `POST /v1/days/:day/scores/scorer-links` |

That is sixteen routes on thirteen paths. Details a twin does not show:

- **Sync input moves into a body.** `?facility=` becomes a path segment, and
  `?method=` and the `X-Edit-At` header become `{ "method"?: "sheets" | "csv",
  "editedAt"?: <ISO 8601> }`. A `method` other than `sheets` or `csv` is a `400`
  (the legacy route silently uses `sheets`), as is an `editedAt` that does not
  parse; one outside the last hour (or in the future) is ignored, as `X-Edit-At`
  is. A day-level sync takes no `editedAt`. The body is optional.
- **Scoresheets** negotiate on `Accept` before the upload is read:
  `application/pdf` (or no `Accept`, or `*/*`) returns the PDF;
  `application/x-ndjson` streams progress lines ending with the PDF
  base64-encoded; anything else is `406`. The body must be `multipart/form-data`.
- **Attendance** is a singular sub-resource of the person
  (`…/people/{personKey}/attendance`), replacing `/v1`'s `…/attendance/{key}`. A
  roster update names its facility in a body, `{ "facility": "…" }`; no body
  means every facility.
- **The score-entry body** is `{ "scoreEntry": "console" | "links" }` where `/v1`
  takes `{ "mode": … }`.

### Where `/v3` differs from the `/v1` routes it replaces

A day, event or facility named in a `/v3` path that does not exist is a `404`
(`/v1` answers `400`); an unsupported method on a known path is a `405` with
`Allow`; a body that is not JSON is a `415`; errors are problem details (below).
Success bodies are the same.

## Conventions

Every `/v3` route follows these. A guard test
(`test/unit/guards/rest-conventions.test.mjs`) checks the naming and method rules
on every registered route and the documentation rules on every OpenAPI operation.

**URLs.**

- Nouns, never verbs. An action becomes a noun for its result: `syncs`,
  `reconciliations`, `sessions`.
- A path segment directly before an identifier is a plural: `days/{day}`,
  `facilities/{facility}`, `matches/{matchNumber}`, `events/{event}`.
- A one-per-parent thing is a singular sub-resource: a day's `visibility`, a
  match's `score`, an event's `score-entry`, the global `settings/live-push`, the
  `diagnostics/sync` report.
- Literal segments are lowercase kebab-case, parameters camelCase. No trailing
  slash, no file extension.
- Nest only for real containment, from the shortest unique parent: day keys are
  unique across events, so a day is `/v3/days/{day}`, not under its event.
- A `GET` is filtered by its query string; a `POST` or `PUT` takes a JSON body.

**Methods.** `GET` reads (and Express answers `HEAD`). `PUT` replaces a
resource's state and is idempotent, and every `PUT` resource also has a `GET` with
the same representation (the exceptions are below). `POST` to a collection
creates something or runs an action that returns a report. `PATCH` and `DELETE`
are allowed by CORS and used by no route yet.

**Bodies.** JSON, camelCase, no envelope: a response is the resource or the
report itself. The `PUT` body and the `GET` response use the same member names. A
`PUT` response is that representation plus side-effect members (`changed`,
`republished`, `live`, `archive`). Enumerations are lowercase strings. A request
with a body must say `Content-Type: application/json`, or it is a `415`.

**Status codes.**

| Status | When |
| --- | --- |
| `200` | a successful `GET`, `PUT`, or action `POST` that returns a report |
| `201` | a `POST` that creates something: a session, a desk link, a scorer link |
| `400` | malformed JSON, or a body or query value that fails validation |
| `401` | no credentials, or invalid or expired ones; always with `WWW-Authenticate: Bearer realm="sage-tools-api"` |
| `403` | valid credentials not allowed this action (wrong day, feature switched off) |
| `404` | an unknown path, or an unknown day, event, facility or match named **in the path** |
| `405` | a known path with an unsupported method, with `Allow` |
| `406` | an `Accept` the route cannot satisfy |
| `409` | the target's state conflicts: conflicting writes (`conflict`), a score that changed (`score_conflict`), an `ATTENDANCE` tab in the wrong shape (`attendance_layout`) |
| `415` | a body that is not `application/json` (`multipart/form-data` for scoresheets) |
| `422` | a well-formed request the workbook's state cannot take (`schedule_layout`) |
| `500` / `502` / `503` | our failure / Google's or GitHub's / temporarily unavailable |

### Exceptions

| Route | Rule it departs from | Why |
| --- | --- | --- |
| `PUT …/people/{personKey}/attendance` and `PUT …/matches/{matchNumber}/score` | a `GET` for every `PUT` | the state is read from the published day snapshot and the `ATTENDANCE` tab's published CSV, which pages already poll; a Sheets read per `GET` would spend Google quota for no reader |
| `PUT …/matches/{matchNumber}/score` | concurrency through `If-Match` / `412` | the precondition is the values a person saw in two cells, not a version the API issues, so there is no `ETag`; `expected` travels in the body, and the `409` carries `current` |
| `201` without `Location` (sessions, desk links, scorer links) | `Location` on a created resource | the result is a signed credential, not a stored resource, so there is no URL to point at |
| `expiresAt` in the three token responses | timestamps are ISO 8601 | it is epoch milliseconds on every surface, and the pages compare it with `Date.now()` |
| `isLive: true \| false \| "auto"` | an enumeration is not a mixed type | the same field, with the same values, as `events.json` and the published snapshot every page reads |
| `422` for `schedule_layout` | — | stated so nobody "fixes" it to `409`: the request is fine, the workbook's content cannot take it |

## Errors

Two shapes, because two kinds of client read them.

**Frozen routes** answer the plain body, as `application/json`:

```json
{ "error": "Unknown sync day: zzz. Valid days: day1", "code": "unknown_day" }
```

Apps Script prints it to operators, so it never changes. A score conflict adds
`current`.

**`/v3`** answers RFC 9457 problem details, as `application/problem+json`:

```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "The sheet changed since this match was opened",
  "instance": "/v3/days/pd-day1/facilities/Main/matches/12/score",
  "code": "score_conflict",
  "error": "The sheet changed since this match was opened",
  "current": { "teamCode1": "A", "teamCode2": "B", "team1Score": 11, "team2Score": 9 }
}
```

`title` is the status's standard reason phrase; `detail` is the message;
`instance` is the request path without its query string. `error` repeats
`detail`, so the pages that read `body.error` work unchanged, and any extra member
the error carries (`current`) follows.

Both shapes carry a stable `code`:

| `code` | Status | When |
| --- | --- | --- |
| `validation_error` | 400 | a body or query value fails validation |
| `unknown_scoresheet_type` | 400 | a scoresheet type that is not registered |
| `unknown_day` | 400 (404 on `/v3`) | a day key that is not in the registry |
| `unknown_facility` | 400 (404 on `/v3`) | a facility name that is not one of the day's |
| `bad_request` | 400 | malformed JSON in a request body |
| `unauthorized` | 401 | no, invalid or expired credentials |
| `forbidden` | 403 | valid credentials that may not do this |
| `score_entry_off` | 403 | score entry is off for the event |
| `not_found` | 404 | an unknown path, or a named resource that is not there |
| `unknown_event` | 404 | an event key that is not in the registry |
| `method_not_allowed` | 405 | an unsupported method on a known `/v3` path |
| `not_acceptable` | 406 | an `Accept` the route cannot satisfy |
| `conflict` | 409 | three conflicting writes in a row |
| `score_conflict` | 409 | the sheet changed since the client showed the match |
| `attendance_layout` | 409 | the `ATTENDANCE` tab's header row is not the expected one |
| `unsupported_media_type` | 415 | a body of the wrong `Content-Type` |
| `schedule_layout` | 422 | no `SCHEDULE` tab, a duplicate match number, or a score cell holding a formula |
| `upstream_failure` | 502 | Google Sheets failed, or every facility's fetch did |
| `config_unavailable` | 503 | the event registry could not be loaded |
| `service_busy` | 503 | Google Sheets answered 429 twice |
| `internal_error` | 500 | anything unexpected |

An unknown path anywhere answers a JSON `404` (`not_found`), never Express's HTML
page.

## Auth, by route

| Needs | Routes |
| --- | --- |
| nothing | `GET /ping`, `GET /openapi.json`, `POST /v3/sessions` (and `/auth/login`), `POST /v3/scoresheets` (and `/scoresheets/...`) |
| secret **or** operator | the two sync `POST`s, `GET /v3/diagnostics/sync` |
| operator only | visibility, live-push and score-entry (`GET` and `PUT`), desk links, reconciliations, scorer links |
| operator or desk | the attendance `PUT` |
| operator or scorer | the score `PUT` |

The shared secret is never accepted where it says operator only. See
[Auth](auth.md) for the tokens and `src/auth/middleware.mjs`.

---
**Features counterpart:** [Control Center](../features/control-center.md).
**Related:** [Architecture](architecture.md), [Sync pipeline](sync-pipeline.md),
[Deployment](deployment.md).
