# Test Patterns

## Layers

Tests are composed from five shared layers plus the test files themselves:

- **testData/** — all test input values and expected results. Never define data inline in test files.
- **config/** — named constants for URLs, HTTP status codes, and SOAP operations. Never use raw strings or numbers in tests.
- **helpers/** — thin project-local wiring (`makeRequest`, `makeSoapRequest`) around `@jirivondra/chronos-test-toolkit-api-ts`, which owns the actual HTTP/SOAP client logic and response wrappers (`ApiResponse`, `SoapResponse`).
- **schemas/** — response body shape definitions used for JSON validation.
- **types/** — TypeScript interfaces for domain entities.

## Making requests

`makeRequest` exposes `get`, `post`, `put`, `del`. Each returns `ApiResponse`. Pass `authenticated = false` to send an unauthenticated request (used for 401 test cases).

## Asserting responses

`ApiResponse` provides two chainable assertion methods:

- `expectStatus(code)` — asserts the HTTP status code
- `expectJsonSchema(schema)` — validates response body field types, required fields, and nullability

To read a value from the response body, cast `response.json` to the appropriate type.

## Test data

All test input data lives in `testData/`, one file per protocol:

- `restTestData.ts` — organized by endpoint (`postTodo`, `putTodo`, …), with scenario keys within (`valid`, `create`, `update`, `invalidDescription`). Dynamic values use `faker`.
- `soapTestData.ts` — organized by operation (`add`, `subtract`, …), if the project has a SOAP API. Shared invalid-input cases live under `common`. Values are static for deterministic results.

Test files import the named export and never define data inline:

```ts
import { restTestData } from '../../../testData/restTestData';
import { soapTestData } from '../../../testData/soapTestData';
```

Smoke tests reuse the first entry from regression cases instead of duplicating data:

```ts
const { a, b, expected } = soapTestData.add.cases[0]!;
```

## Data-driven tests

When the same operation is verified across many input combinations, use `forEach` over a `cases` array from `testData/`. Each entry in the array produces one `it` block.

```ts
soapTestData.add.cases.forEach(({ a, b, expected }) => {
    it(`Add(${a}, ${b}) = ${expected}`, async function () {
        const response = await callOperation(SOAP_OPERATIONS.ADD, { a, b });
        response.expectStatus(HTTP_STATUS.OK).expectResult(expected);
    });
});
```

The `it` name is generated from the actual values using a template literal — this makes every failing test immediately identifiable in the output without inspecting the data file.

The same pattern applies to invalid-input (rainy day) cases:

```ts
soapTestData.common.invalidCases.forEach(({ a, b, description }) => {
    it(`Add with ${description} → SOAP Fault`, async function () {
        const response = await callOperation(SOAP_OPERATIONS.ADD, { a, b });
        response
            .expectStatus(HTTP_STATUS.INTERNAL_SERVER_ERROR)
            .expectFaultContains(soapTestData.common.invalidTypeFault);
    });
});
```

Each `forEach` iteration is still a single independent `it` block — the same rules apply: one concern per test, AAA structure, no shared mutable state between iterations.

## Regression tests

One file per endpoint or operation. Each `it` block covers exactly one scenario. Resources created during a test are cleaned up in `after`.

## Smoke tests

One file per flow. Sequential `it` blocks share state via a `let` variable in the `describe` scope. Only the status code is asserted — detailed validation belongs in regression tests.

---

For step-by-step guidance when writing new tests, use:

- `/add-regression-test`
- `/add-smoke-test`
