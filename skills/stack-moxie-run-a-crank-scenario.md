---
name: Run a Crank scenario against a SaaS stack
description: >-
  Install the Crank CLI, install and authenticate a Cog for the platform under
  test, then run a BDD scenario file and interpret the result. Use this to QA a
  marketing/RevOps stack (Salesforce, Marketo, Pardot, Eloqua, HubSpot, web,
  inbox, DNS) from the command line.
api: grpc/stack-moxie-cog.proto
contract: automaton.cog.CogService (proto3, gRPC)
operations:
  - GetManifest
  - RunStep
  - RunSteps
cli: cli/stack-moxie-cli.yml
generated: '2026-08-14'
method: generated
source: >-
  https://github.com/run-crank/cli/blob/master/README.md,
  grpc/stack-moxie-cog.proto, https://crank.run/intro
---

# Run a Crank scenario against a SaaS stack

This skill covers the **open-source, local** path. Everything below runs on your
machine: `crank` is a gRPC client, and each Cog is a gRPC server you run as a
Docker container — there is no Stack Moxie endpoint in this flow and no account
required.

If you want the **hosted** product instead — scenarios stored server-side, run on
a schedule, driven over HTTP — use the REST API at `https://app.stackmoxie.com/api/`
and the skill `stack-moxie-run-a-scenario-via-rest-api.md`.

## Preconditions

- Docker installed and running (Cogs are Docker images).
- `/usr/local/bin` on `PATH`; the installer uses `sudo`.
- Credentials for the third-party system you intend to test. **Those credentials
  authenticate the Cog to that system, not to Stack Moxie.**

## 1. Install the CLI

```sh
curl -s https://get.crank.run/install.sh | sh
crank --version
```

There is no npm/Homebrew/apt package — the install script fetches an oclif tarball
from `get.crank.run`. The npm package named `crank` is unrelated third-party code;
do not install it.

## 2. Install a Cog and authenticate it

```sh
crank cog:install automatoninc/salesforce
crank cog:auth automatoninc/salesforce
```

`cog:install` defaults to `--source=docker`. `cog:auth` prompts for exactly the
fields the Cog declares in `CogManifest.auth_fields` (returned by the
**`GetManifest`** RPC) — each one is a `FieldDefinition` with a `key`, a `type`
(`STRING`, `EMAIL`, `URL`, `BOOLEAN`, `NUMERIC`, `DATE`, `DATETIME`, `PHONE`,
`MAP`, …) and an `optionality` of `REQUIRED` or `OPTIONAL`. The CLI sends each
value as **gRPC call metadata** keyed by that `key` on every subsequent step.

Pin the version when you can (`org/cog:1.0.0`). Be aware that most public Cog
images last published a `latest` tag in 2020, and seven of the nineteen images
have no `latest` tag at all — see `packages/stack-moxie-packages.yml`.

## 3. Discover what the Cog can do

```sh
crank registry:cogs          # what is installed
crank registry:steps         # every step across all installed Cogs
crank cog:step automatoninc/salesforce --step=SomeStepId   # try one interactively
```

Each step comes from a `StepDefinition`: a `step_id`, a human `name`, `help`,
an `expression` (the sentence your scenario matches), a `type` of **`ACTION`**
(mutates the system under test) or **`VALIDATION`** (asserts), plus
`expected_fields` and `expected_records`.

## 4. Run the scenario

```sh
crank run ./scenario.crank.yml
crank run ./scenarios/                       # a whole folder
crank run scenario.crank.yml --token utmSource=Email -t "utmCampaign=Test Campaign"
crank run --use-ssl ./scenario.crank.yml     # TLS between crank and the Cogs
crank run --debug ./scenario.crank.yml
```

## 5. Read the result correctly

This is the part agents get wrong. **The gRPC call succeeds even when the test
fails.** `RunStep`/`RunSteps` return a `RunStepResponse` and you must branch on
`outcome`, not on the RPC status:

| `outcome` | Meaning | What to do |
|---|---|---|
| `PASSED` (0) | Step ran and met expectations | continue |
| `FAILED` (1) | Step ran, assertion did not hold | a real finding about the customer's stack — report it, do not retry |
| `ERROR` (2) | Step could not run at all | check credentials, reachability of the system under test, and step input |

Detail arrives only as `message_format` + `message_args` (a printf-style human
string), with structured evidence in `records[]` (`KEYVALUE`, `TABLE`, or
`BINARY` — e.g. a screenshot from the web Cog) and `response_data`. There is no
error-code registry; see `errors/stack-moxie-outcome-codes.yml`.

## Retry rule — important

The contract defines **no idempotency key**. `RunStepRequest` carries
`request_id`, `scenario_id` and `requestor_id`, but these are correlation
identifiers only — the proto specifies no de-duplication or replay behaviour.
Re-running an `ACTION` step repeats its mutation in the third-party system
(creating a second lead, sending a second email). Retry `VALIDATION` steps freely;
never blind-retry `ACTION` steps. See `conventions/stack-moxie-conventions.yml`.

## Where limits come from

Any throttling you hit belongs to the SaaS under test and surfaces as `ERROR` with
a message. The CLI is free and unmetered; only the hosted product meters "runs"
per month by plan (`plans/stack-moxie-plans-pricing.yml`).
