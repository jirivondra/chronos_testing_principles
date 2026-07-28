# Project Structure (template)

This layout assumes the project depends on `@jirivondra/chronos-test-toolkit-ts` for the reusable
HTTP/SOAP client logic and response wrappers, keeping only project-specific glue and data locally.

## Directory Overview

```
config/       # constants and static configuration
docs/         # project documentation
helpers/      # thin wiring around @jirivondra/chronos-test-toolkit-ts — no request/response logic
schemas/      # schema definitions for response body validation (per entity)
testData/     # centralized test input data, one file per protocol
types/        # TypeScript interfaces (per entity)
test/
  rest/
    regression/   # regression tests — detailed assertions per endpoint
    smoke/        # smoke tests — full happy-path flows
  soap/           # only if the project has a SOAP API
    regression/   # SOAP regression tests — detailed assertions per operation
    smoke/        # SOAP smoke tests — basic functional verification
```

## config/

Static, reusable constants shared across the project.

- `httpStatus.ts` — `export { HTTP_STATUS } from '@jirivondra/chronos-test-toolkit-ts';`. Always use `HTTP_STATUS.*` instead of raw numbers.
- `urls.ts` — named URL path constants, specific to this project's API. Always use these instead of inline strings.
- `soapOperations.ts` — named SOAP operation constants (only if the project has a SOAP API). Always use these instead of raw strings.

## helpers/

Project-local wiring around the shared toolkit. No request/response logic lives here — that's the toolkit's job.

- `apiClient.ts` — wires `createHttpClients({ baseURL, auth })` from the toolkit using this project's `.env` vars.
- `makeRequest.ts` — wires `createRequestHelpers(clients)` from the toolkit, re-exports `get`/`post`/`put`/`del`. Each accepts an `authenticated` flag (default `true`).
- `soapClient.ts` — wires `createSoapClient(wsdlUrl)` from the toolkit (only if the project has a SOAP API).
- `makeSoapRequest.ts` — wires `createSoapRequestHelper(clientPromise)` from the toolkit, re-exports `callOperation`.

See the toolkit's README for the `ApiResponse`/`SoapResponse` fluent assertion APIs (`expectStatus`, `expectJsonSchema`, `expectResult`, `expectFaultContains`, …) — they don't need to be imported directly since `makeRequest`/`makeSoapRequest` return them.

## testData/

Centralized test input data shared across all test files. One file per protocol.

- `restTestData.ts` — input data for REST tests, organized by endpoint. Dynamic values are generated with `faker`.
- `soapTestData.ts` — input data for SOAP tests, organized by operation (only if the project has a SOAP API). Contains `common` for shared invalid-input cases used across all operations.

Test files import the named export and reference data by key — never define input values inline in test files.

## schemas/

Schema definitions used by `ApiResponse.expectJsonSchema()` to validate response body shape and types. Written as standard JSON Schema objects, validated at runtime with `ajv`.

One file per entity, e.g. `todo.schema.ts` — adapt to this project's actual resources.

## types/

TypeScript interfaces for domain entities used in tests and helpers. One file per entity, e.g. `todo.ts`.

## test/

All test files live here, organized by protocol and test type.

### test/rest/regression/

One file per endpoint. Each file covers all scenarios for that endpoint: happy path, authentication, validation errors, not found, etc.

### test/rest/smoke/

End-to-end flows that verify the system works as a whole. A single test typically creates, reads, updates, and deletes a resource in sequence.

### test/soap/regression/ and test/soap/smoke/ (if applicable)

Same split as REST: one file per operation for regression, full-flow smoke tests separately. Cross-operation tests that verify mathematical or structural properties (round-trip, commutativity) live in a dedicated `combined-operations.spec.ts`.

## .env

Runtime secrets and environment-specific config. Never committed to git.

```
BASE_URL=http://localhost:8000
API_USERNAME=your_username
API_PASSWORD=your_password
SOAP_URL=http://localhost:8001   # only if the project has a SOAP API
```

## package.json

Add `@jirivondra/chronos-test-toolkit-ts` as a dependency and a project-local `.npmrc` pointing `@jirivondra` at GitHub Packages — see the toolkit's README for the exact snippet.
