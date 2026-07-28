# ApiHelper

`ApiHelper` is the root class of the page object inheritance chain. It holds HTTP client logic with no Playwright dependency, so it can be used independently of the browser.

## Purpose

- Provides a single place for API credentials and base URL configuration.
- Exposes typed HTTP methods (`get`, `post`, `put`, `delete`) that every page object inherits.
- Allows tests to set up or tear down server state via API without going through the UI.

## Available methods

| Method       | Signature                               | Description                                                       |
| ------------ | --------------------------------------- | ----------------------------------------------------------------- |
| `get`        | `get(endpoint: string)`                 | Sends a GET request                                               |
| `post`       | `post(endpoint: string, body?: object)` | Sends a POST request with an optional JSON body                   |
| `put`        | `put(endpoint: string, body?: object)`  | Sends a PUT request with an optional JSON body                    |
| `delete`     | `delete(endpoint: string)`              | Sends a DELETE request                                            |
| `apiRequest` | `apiRequest(method, endpoint, body?)`   | Generic dispatcher — use when the method is determined at runtime |

All methods return `Promise<Response>` (native `fetch` response). `get`, `post`, `put`, and `delete` are `protected` — accessible only within page objects. `apiRequest` is `public`.

Authentication and `Content-Type: application/json` are added automatically from environment variables (`API_BASE_URL`, `API_USERNAME`, `API_PASSWORD`).

## How to use inside a page object

Define endpoints as named class properties, then call the inherited methods — the same rule as for selectors: no inline strings scattered across methods.

```ts
export class UserPage extends ToTopButton {
  private readonly usersEndpoint = '/api/users'

  async createUser(payload: { name: string; email: string }) {
    const response = await this.post(this.usersEndpoint, payload)
    return response.json()
  }

  async deleteUser(id: number) {
    await this.delete(`${this.usersEndpoint}/${id}`)
  }

  async getUser(id: number) {
    const response = await this.get(`${this.usersEndpoint}/${id}`)
    return response.json()
  }
}
```

The `endpoint` parameter is appended to `API_BASE_URL`, so the property holds only the path:

```ts
// correct — named property, path only
private readonly usersEndpoint = '/api/users'

// incorrect — duplicates the base URL
private readonly usersEndpoint = 'https://api.example.com/api/users'
```

## When to call API through a page object vs. directly in a test

**Through a page object** — when the API call is part of setting up a reusable precondition that multiple tests need, or when the call logically belongs to the page's domain:

```ts
// page object method — reusable setup
private readonly usersEndpoint = '/api/users'

async seedUser(payload: object) {
  await this.post(this.usersEndpoint, payload)
}
```

```ts
// test — calls the page object, does not know the endpoint
test('display user profile', async ({ userPage }) => {
  await userPage.seedUser({ name: 'Alice' })
  await userPage.visit()
  await userPage.checkUserName('Alice')
})
```

**Directly via `apiRequest`** — when the call is a one-off teardown or verification that does not belong to any page object and would not be reused. Store the endpoint in a named constant, not as an inline string:

```ts
test('delete user', async ({ userPage }) => {
  const userEndpoint = '/api/users/42'
  await userPage.deleteUser(42)
  const response = await userPage.apiRequest('GET', userEndpoint)
  expect(response.status).toBe(404)
})
```

## Rules

- Never hardcode `API_BASE_URL`, credentials, or auth headers — they come from environment variables automatically.
- `get`, `post`, `put`, `delete` are `protected` — call them only from within page object methods, not from test files.
- Use `apiRequest` in tests only when no suitable page object method exists and creating one would not be reused.
- Assert the response status or body in the test, not inside the page object method — page objects prepare data, tests verify outcomes.
