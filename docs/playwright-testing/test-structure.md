# Test structure (spec files)

## Template

```typescript
import { test, expect } from '../support/fixture'
import { dashboardPageData } from '../support/test-data/dashboard_page_data'

test.describe('Test Dashboard page', () => {
  test.describe('Atomic Tests For Header', () => {
    test('Check H1 On Page Dashboard', async ({ dashboardPage }) => {
      await dashboardPage.checkH1(dashboardPageData.h1)
    })

    test('Check Submit Button Visibility', async ({ dashboardPage }) => {
      await dashboardPage.checkSubmitButtonVisible()
    })
  })

  test.describe('E2E Test For Dashboard', () => {
    test('Full User Flow', async ({ dashboardPage }) => {
      await test.step('Step one description', async () => {
        await dashboardPage.doStep1()
      })

      await test.step('Step two description', async () => {
        await dashboardPage.doStep2()
        await dashboardPage.checkResult()
      })
    })
  })
})
```

## `test.step` in E2E tests

Use `test.step` in every E2E test that has more than one logical phase. Each step has a short label describing what happens in that phase — this label appears in the Playwright report and makes failures easy to locate.

```ts
test('Toggle Task Between Open And Finish Sections', async ({ dashboardPage }) => {
  await test.step('Task is in open section', async () => {
    await dashboardPage.checkTaskInOpenSection(taskName)
  })

  await test.step('Toggle to finished', async () => {
    await dashboardPage.toggleTask(taskName)
    await dashboardPage.checkTaskInFinishSection(taskName)
  })

  await test.step('Toggle back to open', async () => {
    await dashboardPage.toggleTask(taskName)
    await dashboardPage.checkTaskInOpenSection(taskName)
  })
})
```

When a step returns a page object that the next step needs, return the value from the `test.step` callback:

```ts
test('Full Application Flow', async ({ loginPage }) => {
  const dashboardPage = await test.step('Login', async () => {
    return loginPage.login(username, password)
  })

  await test.step('Verify dashboard', async () => {
    await dashboardPage.checkDashboardUrl()
  })

  const { taskName, dashboardAfterCreate } = await test.step('Create new task', async () => {
    const newTaskPage = await dashboardPage.clickButtonNewTask()
    await newTaskPage.fillTaskTitle()
    const dashboardAfterCreate = await newTaskPage.clickCreateTaskButton()
    return { taskName: newTaskPage.taskName, dashboardAfterCreate }
  })
})
```

Do not use `test.step` in atomic tests or in simple E2E tests that perform a single action and check a single result — one-liners don't need wrapping.

## Rules

- Always import from `'../support/fixture'`, never directly from `'@playwright/test'`.
- Import `expect` from `'../support/fixture'` as well.

### Describe block structure

| Level          | Format                         | Example                     |
| -------------- | ------------------------------ | --------------------------- |
| Outer          | `'Test [Name] page'`           | `'Test Login page'`         |
| Inner — atomic | `'Atomic Tests For [Section]'` | `'Atomic Tests For Footer'` |
| Inner — E2E    | `'E2E Test For [Page]'`        | `'E2E Test For Login Page'` |

### Test naming

- Starts with a capital letter.
- Describes WHAT is being tested, not how.
- Examples: `'Check H1 On Page Login'`, `'Show And Hide Password'`, `'Login With Correct Credentials'`

### Direct `expect` in a spec file

Use `expect` directly in a spec file only when a page object method is not sufficient:

```typescript
// ok — count cannot be easily encapsulated
await expect(loginPage.contactIcons).toHaveCount(3)
```

All other assertions belong inside page object methods.
