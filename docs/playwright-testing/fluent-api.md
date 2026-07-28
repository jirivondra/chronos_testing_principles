# Fluent API

Every page object method returns `Promise<this>`, allowing calls to be chained via `.then()` instead of writing separate `await` expressions.

## Purpose

- Makes test code more readable — a sequence of actions is written as a single expression.
- Removes repeated `await pageObject` lines.

## How to write chainable methods in a page object

Every method that should support chaining must declare `Promise<this>` as its return type and explicitly `return this`.

```ts
async fillUserName(userName: string): Promise<this> {
  await this.page.locator(this.userName).fill(userName)
  return this
}

async checkH1(text: string): Promise<this> {
  await expect.soft(this.h1).toBeVisible()
  await expect.soft(this.h1).toHaveCount(1)
  await expect.soft(this.h1).toHaveText(text)
  return this
}
```

## Example usage in a test

```ts
await loginPage
  .visit()
  .then((p) => p.fillUserName('admin'))
  .then((p) => p.fillPassword('secret'))
  .then((p) => p.clickSubmit())
```

When a method returns a different page object (e.g. after navigation), the chain ends and the new object is used separately:

```ts
const dashboardPage = await loginPage.login(username, password)
await dashboardPage.checkDashboardUrl()
```

## Rules

- Custom page object methods must explicitly return `return this`, otherwise chaining will not work.
- Chaining works via `.then()` on the returned `Promise<this>`.
- Methods that navigate to a new page return a new page object, not `this` — the chain ends there.
