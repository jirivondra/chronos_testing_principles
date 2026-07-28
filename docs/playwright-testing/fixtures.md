# Fixtures

Fixtures extend Playwright's built-in `test` with page object instances. Each fixture instantiates a page object, navigates to the page before the test, and clears the cache after. Tests receive a ready-to-use page object without any setup boilerplate.

## Purpose

- Removes repetitive `new PageObject(page)` and `visit()` calls from every test.
- Guarantees a clean browser state after each test via `clearCache()`.
- Keeps test files focused on assertions, not setup.

## File structure

```
support/fixture/
  index.ts            — merges all fixture groups and re-exports test + expect
  auth-fixtures.ts    — fixtures for authenticated pages (injects session token before visit)
  noauth-fixtures.ts  — fixtures for pages tested without authentication (redirect testing)
```

## How to use in a test

Import `test` and `expect` from `support/fixture`, not from `@playwright/test` directly. The page object is injected by name as a destructured parameter:

```ts
import { test, expect } from '../support/fixture'

test('Check H1 On Page Login', async ({ loginPage }) => {
  await loginPage.checkH1('Sign in')
})
```

The fixture handles `visit()` before the test body runs and `clearCache()` after — the test itself only interacts with the page.

## Fixture groups

### Authenticated fixtures (`auth-fixtures.ts`)

Used for pages that require the user to be logged in. Instead of going through the login form on every test, the fixture injects a session token directly into `sessionStorage` via `addInitScript` before `visit()` is called:

```ts
dashboardPage: async ({ page }, use) => {
  const token = Buffer.from(
    `${loginCredentials.validUser.username}:${loginCredentials.validUser.password}`
  ).toString('base64')

  await page.context().addInitScript((t) => {
    sessionStorage.setItem('auth', t)
  }, token)

  const dashboardPage = new DashboardPage(page)
  await dashboardPage.visit()
  await use(dashboardPage)
  await dashboardPage.clearCache()
},
```

`addInitScript` runs before any page script, so the token is in `sessionStorage` by the time the app initialises — the app sees an already-authenticated session without any UI login step.

### Unauthenticated fixtures (`noauth-fixtures.ts`)

Used specifically for testing redirect behaviour when a user without a valid session tries to access a protected page. No token injection — the fixture just visits the page directly:

```ts
unAuthDashboardPage: async ({ page }, use) => {
  const dashboardPage = new DashboardPage(page)
  await dashboardPage.visit()
  await use(dashboardPage)
  await dashboardPage.clearCache()
},
unAuthNewTaskPage: async ({ page }, use) => {
  const newTaskPage = new NewTaskPage(page)
  await newTaskPage.visit()
  await use(newTaskPage)
  await newTaskPage.clearCache()
},
```

Fixture names are prefixed with `unAuth` to make the intent clear at a glance in the test.

### Special case: fixture depending on another fixture

Some fixtures are composed from an existing fixture rather than `page`. `newTaskPage` depends on `dashboardPage` — it navigates to the new task page via a UI action and tears down by deleting the task via API:

```ts
newTaskPage: async ({ dashboardPage }, use) => {
  const newTaskPage = await dashboardPage.clickButtonNewTask()

  await use(newTaskPage)

  await dashboardPage.deleteTaskByTitle(newTaskPage.taskName)
},
```

This fixture does not call `clearCache()` — the parent `dashboardPage` fixture handles that. Teardown uses `deleteTaskByTitle()` via API to remove any task created during the test.

## How to add a new fixture

1. Decide which group the fixture belongs to: `auth-fixtures.ts` for authenticated pages, `noauth-fixtures.ts` for unauthenticated access tests.
2. Extend `base` from `@playwright/test` with a typed fixture object.
3. For authenticated pages, inject the session token via `addInitScript` before `visit()`.
4. Export the fixture group and make sure it is included in `mergeTests()` in `index.ts`.

```ts
// support/fixture/index.ts
import { mergeTests } from '@playwright/test'
import { authFixtures } from './auth-fixtures'
import { noAuthFixtures } from './noauth-fixtures'

export const test = mergeTests(authFixtures, noAuthFixtures)

export { expect } from '@playwright/test'
```

## Lifecycle

Standard fixture (e.g. `loginPage`, `logoutPage`) — no auth, no dependency:

```
[before test]  new PageObject(page)  →  visit()
[test body]    use(pageObject)       ← test receives instance here
[after test]   clearCache()
```

Authenticated fixture (e.g. `dashboardPage`):

```
[before test]  addInitScript (token → sessionStorage)  →  new PageObject(page)  →  visit()
[test body]    use(pageObject)                          ← test receives instance here
[after test]   clearCache()
```

Dependent fixture (e.g. `newTaskPage`):

```
[before test]  [dashboardPage fixture runs]  →  clickButtonNewTask()
[test body]    use(newTaskPage)              ← test receives instance here
[after test]   deleteTaskByTitle()           ← API teardown, clearCache handled by parent
```

`clearCache()` runs even if the test fails, so the next test always starts with a clean state.

## Fixtures vs. hooks

Playwright provides two mechanisms for test setup and teardown: fixtures and hooks (`beforeAll`, `beforeEach`, `afterAll`, `afterEach`). They serve different purposes.

**Use a fixture** when the setup is reusable — it will be used in multiple test files or multiple `describe` blocks. A fixture is defined once and injected wherever it is needed.

```ts
// Reusable across all tests that need an authenticated dashboard
dashboardPage: async ({ page }, use) => {
  // sets up auth token, navigates to dashboard
  await use(dashboardPage)
  await dashboardPage.clearCache()
}
```

**Use a hook** when the setup is specific to one `describe` block and would not be reused anywhere else.

```ts
// Specific to "E2E Test For Task Toggle" — other tests don't need this
test.describe('E2E Test For Task Toggle', () => {
  let taskName: string

  test.beforeEach(async ({ dashboardPage }) => {
    const newTaskPage = await dashboardPage.clickButtonNewTask()
    await newTaskPage.fillTaskTitle()
    taskName = newTaskPage.taskName
    await newTaskPage.clickCreateTaskButton()
  })

  test.afterEach(async ({ dashboardPage }) => {
    await dashboardPage.deleteTaskByTitle(taskName)
  })
})
```

Another valid use for `beforeEach` is querying state needed for conditional skip — the result is stored in a `let` variable at `describe` scope and read by `test.skip()` inside each test:

```ts
test.describe('Atomic Tests For Dashboard', () => {
  let openTaskCount: number

  test.beforeEach(async ({ dashboardPage }) => {
    openTaskCount = await dashboardPage.countOpenTasks()
  })

  test('Check Expand Button Is Visible', async ({ dashboardPage }) => {
    test.skip(openTaskCount <= dashboardPageData.taskPreviewLimit)
    await dashboardPage.checkExpandButtonVisible()
  })
})
```

### Decision rule

| Situation                                                | Use                                                              |
| -------------------------------------------------------- | ---------------------------------------------------------------- |
| Setup needed in multiple test files or `describe` blocks | **Fixture** — define once, inject anywhere                       |
| Setup or teardown specific to one `describe` block       | **Hook** (`beforeEach` / `afterEach` / `beforeAll` / `afterAll`) |
| Querying state for `test.skip()` conditions              | **Hook** (`beforeEach`)                                          |

## Rules

- Always import `test` and `expect` from `support/fixture`, never from `@playwright/test`.
- Each fixture file contains one group of related page fixtures (`auth-fixtures.ts` or `noauth-fixtures.ts`).
- Authenticated fixtures must inject the session token via `addInitScript` before `visit()`.
- Unauthenticated fixtures do not inject any token — the missing auth is the condition being tested.
- Dependent fixtures (those that take another fixture instead of `page`) handle teardown via API, not `clearCache()`.
- Do not put test logic or assertions inside a fixture — fixtures only prepare state.
- One fixture = one page object. Do not instantiate multiple page objects in a single fixture.
