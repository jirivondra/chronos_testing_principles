# Test Data

Test data files live in `support/test-data/`. They contain plain TypeScript objects — no logic, no imports from page objects or Playwright. Tests and page objects import from here, never the other way around.

## File naming

| Situation                         | Convention            | Example              |
| --------------------------------- | --------------------- | -------------------- |
| Data specific to one page         | `<page-name>_data.ts` | `login_page_data.ts` |
| Data shared across multiple pages | descriptive noun      | `general.ts`         |

## Static data

Use plain exported `const` objects. Group related values under one export; split into multiple exports when the groups have clearly different purposes.

```ts
// support/test-data/login_page_data.ts
export const loginPageData = {
  h1: 'Chronos',
  h2: 'Welcome',
  urlDashboard: '/dashboard.html',
  urlLoginPage: '/login',
}

export const loginCredentials = {
  validUser: {
    username: 'admin',
    password: 'secret',
  },
  invalidUser: {
    username: 'wrong',
    password: 'wrong',
  },
}
```

Import wherever needed — tests or page object methods:

```ts
import { loginPageData, loginCredentials } from '../support/test-data/login_page_data'

test('Check H1 On Page Login', async ({ loginPage }) => {
  await loginPage.checkH1(loginPageData.h1)
})
```

## Dynamic data — faker

Use [faker](https://fakerjs.dev/) for values that must be unique per run or that simulate realistic user input. Call faker inside a factory function so each call returns a fresh object.

```ts
// support/test-data/user_data.ts
import { faker } from '@faker-js/faker'

export function generateUser() {
  return {
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    email: faker.internet.email(),
    password: faker.internet.password({ length: 12 }),
  }
}
```

```ts
// in a test
import { generateUser } from '../support/test-data/user_data'

test('Register new user', async ({ registrationPage }) => {
  const user = generateUser()
  await registrationPage.fillForm(user)
  await registrationPage.checkSuccessMessage()
})
```

Do not call faker at the module level — that would produce the same value for all tests in a run.

## Date data — dayjs

Use [dayjs](https://day.js.org/) when a test needs a specific or relative date. Keep the dayjs call inside a factory function for the same reason as faker.

```ts
// support/test-data/booking_data.ts
import dayjs from 'dayjs'

export function generateBookingDates() {
  return {
    checkIn: dayjs().add(7, 'day').format('YYYY-MM-DD'),
    checkOut: dayjs().add(14, 'day').format('YYYY-MM-DD'),
  }
}
```

```ts
// in a test
import { generateBookingDates } from '../support/test-data/booking_data'

test('Create booking', async ({ bookingPage }) => {
  const dates = generateBookingDates()
  await bookingPage.fillDates(dates.checkIn, dates.checkOut)
  await bookingPage.checkConfirmation()
})
```

For fixed dates that must not drift (e.g. a past event), use a plain string constant instead of dayjs.

## Rules

- Test data files contain only data — no `expect`, no Playwright calls, no page object imports.
- Static values → plain `const` object.
- Unique or realistic values → `faker` inside a factory function.
- Relative or formatted dates → `dayjs` inside a factory function.
- Never call faker or dayjs at the module top level.
- One file per page (or per logical domain for shared data).
