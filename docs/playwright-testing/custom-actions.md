# Assertions in Page Objects

Page objects use Playwright's `expect` and `expect.soft` directly, imported from `@playwright/test`. Both are called inside page object methods — never on inline selectors.

## `expect.soft` — non-blocking assertions

`expect.soft` is used for groups of related assertions. All assertions in the group run even if one fails — the test is marked as failed at the end, not at the first failure.

Use `expect.soft` when checking multiple properties of the same element or component together:

```ts
async checkH1(text: string): Promise<this> {
  await expect.soft(this.h1).toBeVisible()
  await expect.soft(this.h1).toHaveCount(1)
  await expect.soft(this.h1).toHaveText(text)
  return this
}
```

```ts
async checkLogoExpandedVisible(): Promise<this> {
  await expect.soft(this.logoTitle).toBeVisible()
  await expect.soft(this.logoTitle).toHaveText(this.logoTitleText)
  await expect.soft(this.logoSubtitle).toBeVisible()
  await expect.soft(this.logoSubtitle).toHaveText(this.logoSubtitleText)
  return this
}
```

## `expect` — blocking assertions

`expect` is used for single, critical assertions where failure should stop the test immediately. Also used for network/request assertions.

```ts
async checkNewTaskButtonIsVisible(): Promise<this> {
  await expect(this.newTaskButton).toBeVisible()
  return this
}

async checkCreateTaskPostRequest(): Promise<this> {
  const requestPromise = this.page.waitForRequest(/api\/tasks/)
  await this.createTaskButton.click()
  const request = await requestPromise
  expect(request.method()).toBe('POST')
  return this
}
```

## Decision guide

| Situation                                                | Use                                         |
| -------------------------------------------------------- | ------------------------------------------- |
| Multiple properties of the same element or component     | `expect.soft` — report all failures at once |
| Single assertion, or a prerequisite before the next step | `expect` — stop immediately on failure      |
| Network request method or URL check                      | `expect` — single check                     |

## `expect` in test files

Use `expect` directly in a test file only when a page object method is not sufficient — typically for count or attribute checks on a public locator:

```ts
// ok — count cannot be easily encapsulated
await expect(loginPage.contactIcons).toHaveCount(3)
```

All other assertions belong inside page object methods.
