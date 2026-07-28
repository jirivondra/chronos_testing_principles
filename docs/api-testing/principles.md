# Testing Principles

## DRY (Don't Repeat Yourself)

All code must follow the DRY principle — no duplicated logic or code. Repeated parts belong in helpers.

## YAGNI (You Aren't Gonna Need It)

Only add code that is currently needed. Add new helpers, functions, or abstractions only when they are actually used.

## Prefer existing libraries

Before writing non-trivial logic by hand — parsing, encoding, validation, or standardized values — check whether a library the project already depends on (or already has installed) provides it. Recreating what a library already solves adds risk (missed edge cases, bugs) and ongoing maintenance for no benefit.

This does not mean adding a dependency for every small thing — a handful of stable, project-specific constants (URLs, SOAP operation names) are fine to own directly. It targets logic: parsing, validation, encoding, or standardized values that a mature library already implements correctly.

**Example:** `config/httpStatus.ts` re-exports `HTTP_STATUS` from `@jirivondra/chronos-test-toolkit-api-ts`, which sources its values from axios's `HttpStatusCode` enum instead of hardcoding the same numbers by hand.

## Documentation stays in sync

Whenever a change affects tooling, dependencies, or the shape of a shared file (schema format, config structure, helper behavior), update the relevant `docs/` file in the same change. Documentation describing an implementation that no longer exists is misleading and contradicts the code it claims to describe.

**Example:** switching a schema file from a custom field format to standard JSON Schema validated by `ajv` also requires updating its description in `docs/structure.md`.

## AAA (Arrange – Act – Assert)

Each `it` block follows the AAA pattern in a fixed order:

**Arrange** — prepare everything the test needs. Where this lives depends on the scope:

- `testData/` — centralized input values imported into the test file
- `before` — one-time setup for the whole `describe` (e.g. create a resource that all tests will read)
- `beforeEach` — setup that must repeat before every test (e.g. create a fresh resource when each test will destroy it)

**Act** — one `await` call to `makeRequest`. A single HTTP request per test. Exception: tests that verify mathematical or structural properties across operations (round-trip, commutativity) may make multiple calls within a single `it` block when the property under test inherently requires it.

**Assert** — `response.expectStatus(...)` and optionally `.expectJsonSchema(...)`, chained on the response.

```ts
// Arrange (testData imported from testData/, before() created the resource)
// Act
const response = await get(resourceUrls.byId(resourceId));
// Assert
response.expectStatus(HTTP_STATUS.OK).expectJsonSchema(resourceSchema);
```

## before vs beforeEach

Choose based on whether the test consumes the resource:

- `before` + `after` — the resource survives all tests (GET, PUT scenarios: they read or update, never delete)
- `beforeEach` + `afterEach` — a fresh resource is needed for every test (DELETE scenarios: each test deletes the resource)

## Fluent assertions

Response wrapper methods return `this`, allowing assertions to be chained. Always chain — never split assertions into separate statements on the same response.

```ts
response.expectStatus(HTTP_STATUS.CREATED).expectJsonSchema(resourceSchema);
```

## Named test data

All test input values live in `testData/`, one file per protocol. Test files import from there — never define input values inline.

```ts
import { restTestData } from '../../../testData/restTestData';
```

Each file is a single named export organized by endpoint or operation at the top level, with named scenario keys below:

```ts
export const restTestData = {
    postResource: {
        valid: { title: faker.lorem.words(3), completed: false },
        invalidDescription: { title: faker.lorem.words(3), description: 'x'.repeat(5001), completed: false },
    },
    // ...
};
```

**Naming conventions:**

- Top-level key — name of the endpoint or operation (`postResource`, `add`, `smoke`)
- Scenario key — intent of the data (`valid`, `create`, `update`, `invalidDescription`)
- Shared data across operations — group under `common`

**Dynamic vs. static data:**

- Use `faker` for REST data where values must be unique across runs (titles, descriptions)
- Use static values for SOAP data where results must be deterministic (mathematical inputs and expected outputs)

**Smoke tests** reference the first item from regression cases (`cases[0]`) rather than maintaining a separate copy of the same data.

## Facade

Tests never import the HTTP/SOAP client library directly. `makeRequest`/`makeSoapRequest` are project-local facades wiring `@jirivondra/chronos-test-toolkit-api-ts` that hide the underlying HTTP client and translate raw responses into a fluent response wrapper. This means transport-level changes (auth, base URL, error handling) stay confined to one place and never leak into test files.

```
test file → makeRequest (facade) → toolkit http client → API
```

## Cyclomatic complexity

Every function must have a cyclomatic complexity of at most **2**. Complexity starts at 1 and increases by 1 for each branch: `if`, ternary `?:`, `&&`, `||`, `for`, `while`, `switch case`.

A complexity above 2 is a signal that the function is doing too much or contains logic that should be split or expressed differently.

Enforce automatically by ESLint — the build should fail if the limit is exceeded (see the shared `complexity: 2` rule in `@jirivondra/chronos-test-toolkit-api-ts/eslint-config`).

**Allowed (complexity 2):**

```ts
function getClient(authenticated: boolean) {
    return authenticated ? authenticatedClient : unauthenticatedClient; // +1
}
```

**Not allowed (complexity 3):**

```ts
function getClient(authenticated: boolean) {
    if (authenticated) {      // +1
        return authenticatedClient;
    } else if (someOther) {   // +1 → complexity 3
        ...
    }
}
```

## No if conditions in helpers

Instead of `if/else` blocks, use a functional approach — `filter`, `forEach`, private methods. Conditions reduce readability and testability.

## Test isolation

Each `it` block must be fully independent — it must not rely on state left by another test or assume a specific execution order. Setup belongs in `before`/`beforeEach`, cleanup in `after`/`afterEach`.

## Single concern per test

Each `it` block tests exactly one thing — one status code, one scenario. Do not combine multiple independent assertions into a single test.

## Data-driven tests

When the same operation must be verified across multiple input combinations, use a `forEach` loop over a `cases` array instead of repeating `it` blocks manually. Each array entry produces one independent test.

Use data-driven tests when:

- The same endpoint or operation is tested with many input variants (positive, negative, float, zero, …)
- The test body is identical across all variants — only the inputs and expected result differ

Do not use data-driven tests when:

- The scenarios differ structurally (different setup, different assertions) — write separate `it` blocks instead
- There is only one or two variants — `forEach` over a single-element array adds noise without benefit

## No hardcoded static values

No static text or value may appear directly in code. Every static value must be assigned to a named variable. Allowed locations:

- **`testData/`** — test input values, expected results, error message strings
- **`config/httpStatus.ts`** — re-exported from the shared toolkit
- **`config/urls.ts`** — REST URL paths
- **`config/soapOperations.ts`** — SOAP operation names (if applicable)

This applies to: URLs, HTTP status codes, test input values, error messages, timeouts, and any other literal that would otherwise appear inline.
