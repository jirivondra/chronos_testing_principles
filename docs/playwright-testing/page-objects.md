# Page Objects

Page Object Model (POM) is a design pattern that separates page logic from the tests themselves. Each page of the application is represented by its own class that encapsulates selectors and actions.

## Purpose

- Prevents duplication — a selector is defined in one place.
- Tests are more readable; they work with user actions, not CSS selectors.
- When the UI changes, only the page object needs updating, not every test individually.

## Inheritance vs. Composition

Common classes (`AppBar`, `SiteBarMenu`, `OpenTask`, …) can be brought into a page object in two ways.

### Inheritance — full class chain

Use when a page object needs the majority (~80%) of the functionality from the common chain. The concrete page extends the deepest class it needs; the entire chain above it comes along automatically.

Current chain:

```
ApiHelper → BasePage → Header → Footer → ToTopButton → AppBar → SiteBarMenu → OpenTask → DashboardPage
```

Each class in the chain is a self-contained unit — it defines its own selectors and methods and passes `page` and `path` upward via `super()`. Pages that share the same UI structure (app bar, sidebar menu, task list) use this pattern.

```ts
// DashboardPage needs the full stack — extend the deepest class required
export class DashboardPage extends OpenTask {
  // ...
}
```

### Composition — single component

Use when a page object needs only **one** specific component from commons. Pulling in the full inheritance chain just to use one class is unnecessary coupling — instantiate that class as a property instead.

```ts
// Only AppBar is needed — compose it as a property, do not extend the full chain
export class SpecialPage extends BasePage {
  private readonly appBar: AppBar

  constructor(page: Page) {
    super(page, '/special')
    this.appBar = new AppBar(page, '/special')
  }

  async clickLogout() {
    return this.appBar.clickLogout()
  }
}
```

> **Limitation:** methods on a composed object return that object's `this`, not the outer page's `this`. Fluent chaining does not flow across the boundary without wrapping each delegation method.

### Decision rule

| Situation                                                        | Approach                                                       |
| ---------------------------------------------------------------- | -------------------------------------------------------------- |
| Page uses most of the common UI (app bar, sidebar, task list, …) | **Inheritance** — extend the deepest class needed              |
| Page only needs one specific common component                    | **Composition** — instantiate that class as a private property |

## How to create a new page object

1. Decide whether to use inheritance or composition (see above).
2. Define selectors as private/protected class properties.
3. Each public method corresponds to a single user action or assertion.
4. Page objects do not contain test logic (`test`, `expect` belong in tests).

## Assertions and interactions

Assertions and element interactions inside page objects use `expect` and `expect.soft` directly, imported from `@playwright/test`. See [custom-actions.md](custom-actions.md) for when to use each.

## Selectors: string vs Locator

A class property can hold either a CSS/selector **string** or a resolved **Locator**. The choice follows where the property is consumed:

| Type      | When to use                                                                                              | Visibility                                               |
| --------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `string`  | Used as a source to build a `Locator` via `this.page.locator()`, or when a subclass needs to override it | `protected` when subclasses need it, otherwise `private` |
| `Locator` | Passed to `expect`/`expect.soft`, or used directly with `.click()`, `.fill()` etc.                       | `private readonly`                                       |

A `string` property can serve as the source for a sibling `Locator` when both are needed:

```ts
protected submitButton: string = 'button[type="submit"]'
private readonly signInButton: Locator = this.page.locator(this.submitButton)
```

Use `protected` on a `string` selector only when a subclass must reference or override it. Default to `private readonly` for `Locator` properties — they are never inherited.

## Rules

- Selectors are not magic strings scattered across methods — they are named class properties.
- One method = one user action or assertion.
- A new page object inherits from the closest existing class with shared functionality.
- Test logic does not belong in page objects.
