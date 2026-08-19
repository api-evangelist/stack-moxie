---
name: Run and monitor a Stack Moxie test scenario over the REST API
description: >-
  Authenticate to the Stack Moxie REST API with a JWT, connect a platform, create
  a test Scenario, run it synchronously or asynchronously, and read the outcome.
  Use this to drive stack QA from CI, a workflow tool, or an agent.
api: openapi/stack-moxie-rest-api-openapi.yml
base_url: https://app.stackmoxie.com/api/
operations:
  - POST /v1/organizations/{org}/connections
  - GET /v1/organizations/{org}/registry
  - POST /v1/organizations/{org}/scenarios
  - POST /v1/organizations/{org}/scenarios/{id}/runs
  - GET /v1/organizations/{org}/scenarios/{scenarioId}/runs/{id}
  - GET /v1/organizations/{org}/scenarios/{scenarioId}/mostRecentRun
  - GET /v1/organizations/{org}/scenarios/{scenarioId}/runs/{runId}/log/{id}
  - POST /v1/organizations/{org}/asyncScenario
  - GET /v1/organizations/{org}/job/{jobId}
  - POST /v1/organizations/{org}/schedules
generated: '2026-08-14'
method: generated
source: openapi/stack-moxie-rest-api-openapi.yml, https://api.stackmoxie.com/
---

# Run and monitor a Stack Moxie test scenario over the REST API

Base URL: `https://app.stackmoxie.com/api/`. Reference: <https://api.stackmoxie.com/>.
The spec has **no `operationId`s**, so operations are addressed by method + path
throughout.

## Authenticate

Every one of the 43 operations requires the same header — the security requirement
is declared once at the document root and no endpoint is anonymous:

```
Authorization: Bearer <jwt>
```

Mint and revoke the JWT on your account settings page at
<https://app.stackmoxie.com>. There is **no token endpoint, no refresh flow and no
introspection endpoint** in the API — token lifecycle is a UI-only operation, so an
unattended agent needs a human to rotate its credential.

Distinguish the two auth failures:
- `401` — "there may be a problem with your API token" → the token is bad or expired.
- `403` — "the authenticated user isn't allowed to perform this action" → the token
  is fine, the user lacks rights in this Organization. There are no scopes to widen.

## Know your Organization UUID

Almost every path is tenant-scoped: `/v1/organizations/{org}/...` where `{org}` is
the Organization **UUID** (`format: uuid`). There is no "current organization"
default and no list-my-organizations operation in the spec — obtain the UUID from
the app.

## 1. Connect the platform you want to test

Read what Cogs are available and what each one needs:

```http
GET /v1/organizations/{org}/registry
```

Returns `RegistryEntry` objects — `name`, `label`, `version`, `homepage`,
`authHelpUrl`, `stepDefinitionsList`, `authFieldsList`. This is the REST projection
of the Cog gRPC `CogManifest`; `authFieldsList` tells you exactly which keys the
next call needs.

```http
POST /v1/organizations/{org}/connections
{ "cog": "automatoninc/salesforce", "auth": { ...fields from authFieldsList... }, "profile": "default" }
```

`auth` is **writeOnly** — you can never read credentials back, only overwrite them.
Check `isValid` on a later `GET` to see whether the credentials still work.

Failure modes here are specific and worth handling: `404` unknown `cog`, `409` a
Connection already exists for this cog/profile (PATCH the existing one instead of
retrying the POST), `422` the `auth` keys do not match the Cog's required fields.

## 2. Create a Scenario

```http
POST /v1/organizations/{org}/scenarios
{
  "name": "...",
  "definition": {
    "scenario": "...",
    "steps": [
      { "cog": "automatoninc/salesforce", "stepId": "<from stepDefinitionsList>",
        "data": { ... }, "waitFor": 0, "failAfter": 0 }
    ]
  }
}
```

Each step requires `cog` and `stepId`. `waitFor` delays the step (seconds);
`failAfter` is how long to keep retrying it before the run is considered failed.
`422` means the definition is malformed — validate against `ScenarioDefinition`
before resending, and confirm each `stepId` exists in the registry.

Organize with folders (`POST /v1/organizations/{org}/folders`) and notify humans
with notification groups (`POST /v1/organizations/{org}/notification-groups`,
which deliver to an **email alias** — there are no webhooks).

## 3. Run it

Synchronous:

```http
POST /v1/organizations/{org}/scenarios/{id}/runs
```

Asynchronous — for definitions that need it:

```http
POST /v1/organizations/{org}/asyncScenario     → { "jobId": "<uuid>" }
GET  /v1/organizations/{org}/job/{jobId}       → status: Processing | Complete | Failed
```

Poll until `status` leaves `Processing`, then read `data` — note it is a **JSON
string**, so parse it a second time. There is no callback, no webhook and no event
stream; polling is the only completion signal. The extra constraints the async
endpoint enforces (its `400` says the definition "does not match async
requirements") are not documented — treat a `400` here as "resubmit synchronously".

Or schedule it to repeat:

```http
POST /v1/organizations/{org}/schedules
{ "interval": 3600, "scheduledModel": "scenario", "scheduledScenario": <scenarioId> }
```

`interval` is in **seconds**.

## 4. Read the result — do not confuse HTTP with test outcome

```http
GET /v1/organizations/{org}/scenarios/{scenarioId}/runs/{id}
GET /v1/organizations/{org}/scenarios/{scenarioId}/mostRecentRun
GET /v1/organizations/{org}/scenarios/{scenarioId}/runs/{runId}/log/{id}
```

**A failing test returns HTTP 200.** The finding is `Run.outcome`:
`Created | Running | Passed | Failed | Error | Waiting`. `Failed` is a real defect
in the customer's stack; `Error` means the run could not complete (usually an
invalid Connection). `duration` is in milliseconds; fetch the `RunLog` for detail.

Filter run history with `?outcome=Passed,Failed`, `?ranAfter=<unix ms>`,
`?ranBefore=<unix ms>`.

## Retry rule — important

There is **no `Idempotency-Key`** and no client request id anywhere in the API. A
retried `POST .../runs` or `POST .../asyncScenario` is a **second real execution**:
it re-runs actions against live Salesforce/Marketo/inbox systems and consumes a
second metered "run" against your monthly plan quota. On a timeout, do **not**
blind-retry — call `GET .../mostRecentRun` first and only resubmit if nothing
landed.

## Paging and filtering

List operations return a **bare JSON array** with no total, no next link and no
`Link` header, so you cannot tell whether more records exist except by asking for
another page and seeing it come up short. Two idioms are offered without stated
precedence: `?limit=&skip=` and `?page=&perPage=`. Also available: `?sort=name ASC`,
`?where=` (Waterline/Sails blueprint criteria), `?select=a,b`, `?populate=` to
expand relations. Because relation fields are `oneOf: [integer, object]`, whether
`scenario`, `log` or `members` comes back as an id or a full object depends on
`populate` — handle both shapes.

## Limits

No rate limits are documented, no `429` is declared, and no `RateLimit-*` or
`Retry-After` header is defined — so there is no runtime backoff signal. The real
ceiling is commercial: metered runs per month per plan
(`plans/stack-moxie-plans-pricing.yml`). Behaviour when the quota is exhausted is
not specified in the contract.
