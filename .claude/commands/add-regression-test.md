# Add Regression Test

Create a regression test file for a REST API endpoint following the project conventions.

## Inputs to gather first

Before writing anything, identify:

- HTTP method and endpoint path (e.g. `GET /todos/{id}`)
- Which status codes to cover (200/201/204, 401, 404, 422, …)
- Whether the test needs to create a resource (→ needs `after` cleanup)
- Whether the response body should be validated with a schema

## File location

`test/rest/regression/<method>-<resource>.spec.ts`

Examples: `get-todos.spec.ts`, `post-todo.spec.ts`, `delete-todo.spec.ts`

## Test data

Add all input values for this endpoint to `testData/restTestData.ts` under a new top-level key named after the endpoint (e.g. `postTodo`, `putTodo`). Use `faker` for dynamic values.

```ts
// in testData/restTestData.ts
$endpoint: {
  valid: { title: faker.lorem.words(3), completed: false },
  // invalid variants for 422 cases
},
```

## Template

```ts
import { $method } from '../../../helpers/makeRequest';
import { $schema } from '../../../schemas/$resource.schema';   // if body validation needed
import type { $Type } from '../../../types/$resource';         // if reading response body
import { HTTP_STATUS } from '../../../config/httpStatus';
import { $resourceUrls } from '../../../config/urls';
import { restTestData } from '../../../testData/restTestData';

describe('$METHOD /$resource', function () {
  let resourceId: number; // only if cleanup is needed

  after(async function () {              // only if a resource was created
    await del($resourceUrls.byId(resourceId));
  });

  it('Test for $METHOD - $STATUS_CODE', async function () {
    const response = await $method($resourceUrls.$url, restTestData.$endpoint.$scenario, $authenticated?);
    // if you need the id from the response:
    // resourceId = (response.json as $Type).id;
    response
      .expectStatus(HTTP_STATUS.$STATUS)
      .expectJsonSchema($schema); // omit if no body validation
  });

  // one it() per status code
});
```

## Rules to follow

- One `it` block per status code — never combine two assertions into one test.
- Use `HTTP_STATUS.*` constants — never raw numbers.
- Use `$resourceUrls.*` constants — never raw strings.
- All input values belong in `testData/restTestData.ts` — never inline in the test file.
- Pass `false` as the last argument to `get`/`post`/`put`/`del` for unauthenticated requests (401 cases).
- Cleanup with `after`, not `afterEach` — runs once after the whole suite.
- Only validate the schema on success responses (2xx). Error responses need only `expectStatus`.
