# Conditional Skip via API State

Some UI behaviours only appear under specific data conditions (e.g. an "expand" button is only visible when there are more than 10 tasks). Tests for such behaviours must first check the current state of the data before deciding whether to run or skip.

## Pattern

1. Query the API in `beforeEach` and store the result in a `let` variable scoped to the `describe` block.
2. Use `test.skip(condition)` at the top of each test that depends on that state.
3. Define the threshold values as named constants in the test-data file.

```ts
test.describe('Atomic Tests For Dashboard', () => {
  let openTaskCount: number

  test.beforeEach(async ({ dashboardPage }) => {
    openTaskCount = await dashboardPage.countOpenTasks()
  })

  test('Check Empty Open Section', async ({ dashboardPage }) => {
    test.skip(openTaskCount > dashboardPageData.emptyListCount)
    await dashboardPage.checkEmptyOpenSection()
  })

  test('Check Expand Button Is Visible', async ({ dashboardPage }) => {
    test.skip(openTaskCount <= dashboardPageData.taskPreviewLimit)
    await dashboardPage.checkExpandButtonVisible()
  })

  test('Check Expand Button Is Not Visible', async ({ dashboardPage }) => {
    test.skip(openTaskCount > dashboardPageData.taskPreviewLimit)
    await dashboardPage.checkExpandButtonNotVisible()
  })
})
```

## Where the API call lives

The method that fetches state belongs in the page object, not in the test. It uses the inherited `this.get()` from `ApiHelper`:

```ts
// open_task.ts
async countOpenTasks(): Promise<number> {
  const response = await this.get(this.todosEndpoint)
  const todos = (await response.json()) as { completed: boolean }[]
  return todos.filter((t) => !t.completed).length
}
```

The test only calls the method and stores the result — it does not know about endpoints or response shapes.

## Where the thresholds live

Store threshold constants in the relevant test-data file alongside other page data:

```ts
// support/test-data/dashboard_page_data.ts
export const dashboardPageData = {
  emptyListCount: 0,
  taskPreviewLimit: 10,
}
```

This keeps magic numbers out of the spec file and in one place to update.

## Why not `test.skip` at describe level

`test.skip` at the `describe` level skips all tests unconditionally. Using it inside individual tests allows each test to apply its own condition — mutually exclusive cases (button visible vs. not visible) can coexist in the same describe block and exactly one will run depending on current data.

## Rules

- The API call goes in `beforeEach`, not inside each test body.
- Store the result in a `let` variable at `describe` scope, not at module scope.
- The page object method does the fetching and filtering — the test only stores a number.
- Threshold constants go in the test-data file, not inline in `test.skip()`.
- `test.skip(condition)` must be the first statement in the test body, before any page object calls.
