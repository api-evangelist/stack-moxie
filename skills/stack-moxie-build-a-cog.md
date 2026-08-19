---
name: Build a Crank Cog for a new platform
description: >-
  Scaffold, implement, install and document a Cog - a gRPC service implementing
  automaton.cog.CogService that exposes test steps and assertions for one SaaS
  platform. Use this when Crank has no Cog for a system you need to test.
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
  grpc/stack-moxie-cog.proto,
  https://github.com/run-crank/cli/blob/master/README.md,
  https://github.com/run-crank/typescript-cog-utilities
---

# Build a Crank Cog for a new platform

A Cog is **any gRPC service that implements `automaton.cog.CogService`**. There
are exactly three RPCs to implement. The full contract is in
`grpc/stack-moxie-cog.proto`.

## 1. Scaffold

```sh
crank cog:scaffold --language=typescript --org=myorg --name="My System" \
  --include-example-step --include-mit-license
```

For TypeScript, take the shared helpers:

```sh
npm install @run-crank/utilities
```

(`@run-crank/utilities` 0.5.2, last published 2022-12-30 — the closest thing to a
first-party SDK; expect to fill gaps yourself.)

## 2. Implement `GetManifest`

`GetManifest(ManifestRequest) returns (CogManifest)` — takes an empty request and
must return enough for `crank` to run you:

- `name` — e.g. `myorg/my-system-cog`
- `label`, `version` (semver), `homepage`
- `auth_fields[]` — a `FieldDefinition` per credential you need. The `key` you
  choose **is the gRPC metadata key** `crank` will set on every call
  (e.g. `mySystemAuthToken`). Set `optionality` to `REQUIRED` or `OPTIONAL` and a
  `type` from `ANYSCALAR | STRING | BOOLEAN | NUMERIC | DATE | DATETIME | EMAIL |
  PHONE | URL | ANYNONSCALAR | MAP`. Write real `description`/help text — it is
  what the user sees at `crank cog:auth`.
- `auth_help_url` — where a user learns how to get those credentials.
- `step_definitions[]` — one `StepDefinition` per step: `step_id`, `name`, `help`,
  `type` (`ACTION` or `VALIDATION`), `expression` (the sentence scenario authors
  write), `expected_fields[]`, `expected_records[]`.

Capability discovery is this live RPC — there is no static schema document to
publish, so anything you leave out of the manifest is invisible to users.

## 3. Implement `RunStep`

`RunStep(RunStepRequest) returns (RunStepResponse)`.

Read `step.step_id` to dispatch, `step.data` (a `google.protobuf.Struct`) for
arguments, and pull credentials from the **gRPC call metadata** under the keys you
declared. `request_id`, `scenario_id` and `requestor_id` are for logging and
correlation — do not treat them as idempotency keys.

Return a `RunStepResponse`, and set `outcome` honestly:

- `PASSED` — ran and met expectations
- `FAILED` — ran fine, but the assertion did not hold (a finding about the user's
  stack)
- `ERROR` — could not run: bad credentials, upstream unreachable, bad input

Do **not** raise a gRPC error for a failed assertion; that collapses the
distinction between "your stack is broken" and "my Cog is broken". Put detail in
`message_format` + `message_args`, attach evidence as `records[]`
(`key_value` Struct, `table`, or `binary`), and structured output in
`response_data`.

## 4. Implement `RunSteps` (streaming)

`RunSteps(stream RunStepRequest) returns (stream RunStepResponse)` — bidirectional
streaming. The proto's stated reason for it is systems with onerous authentication
or expensive setup: authenticate or open a session once per stream instead of once
per step. Emit one response per request, in order.

## 5. Install locally and exercise it

```sh
crank cog:install --source=local --local-start-command="npm start"
crank cog:auth myorg/my-system-cog
crank cog:step myorg/my-system-cog --step=MyStepId
crank cog:steps myorg/my-system-cog
crank registry:steps
```

Use `--use-ssl` on `cog:step`/`cog:steps`/`run` to verify your TLS support.

## 6. Document and publish

```sh
crank cog:readme myorg/my-system-cog
```

This injects generated auth and step docs into `README.md` at the
`<!-- authenticationDetails -->` and `<!-- stepDetails -->` markers, so the docs
are generated from the manifest rather than hand-maintained.

Publish as a Docker image so `crank cog:install org/cog:version` works — that is
how the nineteen first-party Cogs under `automatoninc` on Docker Hub ship. **Tag a
real semantic version**, not just `latest`, and keep it moving: the first-party
images are a cautionary example, with `latest` frozen in 2019–2020 and seven of
nineteen carrying only rolling `dev-*` tags no consumer can pin.
