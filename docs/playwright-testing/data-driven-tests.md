# Data-Driven Tests

Data-driven testing runs the same test logic against multiple input datasets. Instead of duplicating a test for each case, a single test template iterates over an array of data and generates one test per entry.

## Purpose

- Eliminates test duplication — the assertion logic is written once.
- Adding a new case means adding one object to the data array, not a new test.
- Each generated test has its own name and result in the report.

## How to define test case data

Store the array in the relevant test-data file alongside the other page data. Each entry is a plain object with fields that describe the case.

```ts
// support/test-data/login_page_data.ts
export const negativeLoginCases = [
  { description: 'Invalid Credentials', username: 'wrong', password: 'wrong' },
  { description: 'Empty Credentials', username: '', password: '' },
]
```

- `description` — a short label used to build the test name; starts with a capital letter.
- Other fields — the values passed into the test body.

## How to use in a test

Import the array and call `.forEach()` inside a `test.describe` block. Destructure the entry fields directly in the callback parameter list.

```ts
import { negativeLoginCases } from '../../support/test-data/login_page_data'

test.describe('Login Page - Negative Scenarios', () => {
  negativeLoginCases.forEach(({ description, username, password }) => {
    test(`Login With ${description} Stays On Login Page`, async ({ loginPage }) => {
      await loginPage
        .fillUserName(username)
        .then((l) => l.fillPassword(password))
        .then((l) => l.clickSubmit())
        .then((l) => l.checkUrl(loginPageData.urlLoginPage))
    })
  })
})
```

The test name is built from the template literal — each generated test appears separately in the report with its own `description` value.

## Naming the describe block and test

| Part       | Format                                                   | Example                                               |
| ---------- | -------------------------------------------------------- | ----------------------------------------------------- |
| `describe` | `'[Page] - [Scenario group]'`                            | `'Login Page - Negative Scenarios'`                   |
| `test`     | template literal with `description` as the variable part | `` `Login With ${description} Stays On Login Page` `` |

The static part of the test name describes the shared behaviour; `description` identifies what differs between cases.

## When to use

Use data-driven iteration when:

- Two or more tests share identical logic but different inputs.
- The number of cases is likely to grow (e.g. validation error messages, permission combinations).

Do not use it when:

- Each case requires meaningfully different assertions or setup steps — write separate named tests instead.
- There is only one case — a plain test with a descriptive name is clearer.

## Rules

- The case array lives in `support/test-data/`, not inline in the spec file.
- Each entry must have a `description` field used to build a unique test name.
- Destructure fields in the `.forEach()` callback — do not reference `testCase.username` inside the test body.
- The describe block that wraps the loop is always a nested inner block, not the outer `test.describe`.
- Follow the fluent API chaining style (`.then()`) inside the generated test body, the same as any other test.
