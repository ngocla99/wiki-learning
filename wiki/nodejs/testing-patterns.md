# Testing: Patterns and Best Practices

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Testing is a fundamental part of software development that provides **fast feedback**, maximizes **collaboration**, and serves as **living documentation**. A well-tested codebase makes it easier to reproduce issues, onboard new contributors, and ship changes with confidence.

This article covers the core concepts of software testing, the types of tests, and how to write unit, integration, and E2E tests using Node.js's built-in test runner and Playwright.

---

## Foundations

### System Under Test (SUT)

The **System Under Test** is the specific component, module, function, or application being evaluated in a particular test case. It can range from a single function to a complex multi-tiered application.

### Arrange, Act, Assert (AAA)

Every test follows three high-level steps:

| Phase     | Purpose                                             |
| --------- | --------------------------------------------------- |
| **Arrange** | Set up pre-conditions for the SUT                  |
| **Act**     | Execute the SUT with those preconditions           |
| **Assert**  | Verify the results match expected behavior         |

### Code Coverage

Code coverage measures the percentage of your codebase executed during tests. Levels include:

- **Line coverage** -- percentage of lines executed
- **Branch coverage** -- percentage of decision points (if/switch) executed
- **Statement coverage** -- percentage of executable statements executed
- **Function coverage** -- percentage of functions invoked

> A high coverage percentage does not guarantee quality. You can hit 100% with zero assertions. Prioritize **focused tests** that verify behavior; treat coverage as a byproduct, not the goal.

### Test Doubles: Stubs, Spies, and Mocks

Test doubles are stand-in components used to isolate the SUT from its dependencies.

| Type     | Behavior                                                        |
| -------- | --------------------------------------------------------------- |
| **Stub** | Provides predetermined, static responses regardless of input    |
| **Spy**  | A stub that also records interactions (call count, arguments, order) |
| **Mock** | A spy with predefined expectations that actively fails the test if violated |

> Many testing frameworks provide a single `mock` abstraction that can serve as a stub, spy, or mock depending on how you configure it.

**Warning:** Mocks tightly couple tests to implementation details. If you refactor code (rename a method, adjust parameters), mock-based tests break even if overall behavior is unchanged. Avoid over-relying on mocks.

### TDD (Test-Driven Development)

The **Red-Green-Refactor** cycle:

1. **Red** -- Write a failing test that describes desired behavior
2. **Green** -- Write just enough code to make the test pass
3. **Refactor** -- Clean up the code while keeping the test green

### BDD (Behavior-Driven Development)

BDD expresses tests in a structured, human-readable format (typically **Gherkin**) to bridge communication between developers and non-technical stakeholders:

```gherkin
Feature: User Authentication
  Scenario: Successful Login
    Given I am on the login page
    When I enter valid username "user123" and password "password123"
    And I click the "Login" button
    Then I should be redirected to the dashboard page
```

In JavaScript/Node.js, [Cucumber](https://cucumber.io/) is a widely used BDD framework.

### CI/CD

- **Continuous Integration (CI)** -- Developers push code frequently, triggering automated builds and tests to catch issues early.
- **Continuous Delivery (CD)** -- Every change that passes CI is automatically built, tested, and deployed to a staging environment. A human decides when to push to production.
- **Continuous Deployment** -- Removes the manual step; if automated tests pass, changes go straight to production.

---

## Types of Tests

### Unit Tests

Examine the **smallest testable units** (functions, methods, classes) in complete isolation. Key qualities:

- **Precision** -- Pinpoint failures to specific lines of code
- **Speed** -- Run in milliseconds; suites of hundreds finish in seconds
- **Design discipline** -- Force modular code with clear inputs/outputs
- **Living documentation** -- Well-named tests document expected behavior

### Integration Tests

Verify that **components work together** as intended (e.g., business logic + database, payment module + gateway).

Challenges:
- Setup overhead (databases, APIs, containers)
- State management (reset between tests)
- Reproducibility ("works on my machine")
- Speed trade-offs

> Prioritize testing critical user workflows. Aim for dozens, not hundreds.

### End-to-End (E2E) Tests

Simulate **full user journeys** from start to finish, validating the entire experience. They bridge the gap between theoretical correctness and real-world reliability.

Challenges:
- Complex setup (full environments, browser automation)
- Brittleness (UI changes break tests)
- Ownership ambiguity in large teams
- Slow execution (minutes to hours)

> Focus on a handful of well-crafted E2E tests covering high-stakes workflows (checkout, signups, payments).

### Other Test Types

Performance, usability, security, regression, fuzz, property, and mutation testing each address unique risks. See the book for details.

### The Testing Pyramid

The pyramid guides test distribution for a balanced strategy:

```
        /  E2E  \        -- A handful (critical user flows)
       /----------\
      / Integration \    -- Fewer (key component connections)
     /----------------\
    /    Unit Tests     \ -- Many (fast, focused, catch bugs early)
   /--------------------\
```

> The **testing trophy** (Kent C. Dodds) offers an alternative: "Write tests. Not too many. Mostly integration." It prioritizes ROI per test.

---

## The Node.js Test Runner

Since Node.js 18, the standard library includes `node:test`, a built-in test runner with mocking, assertions, coverage, and more.

### First Test

```js
// calculateBasketTotal.test.js
import { equal } from 'node:assert/strict'
import { test } from 'node:test'
import { calculateBasketTotal } from './calculateBasketTotal.js'

test('Calculates basket total', () => {
  // Arrange
  const basket = {
    items: [
      { name: 'Croissant', unitPrice: 2, quantity: 2 },
      { name: 'Olive bread', unitPrice: 3, quantity: 1 },
    ],
  }
  // Act
  const result = calculateBasketTotal(basket)
  // Assert
  equal(result, 7, `Expected total to be 7, but got ${result}`)
})
```

Run with:

```bash
node --test
```

**Assertions:**
- `equal()` -- compares primitives or strict object identity
- `deepEqual()` -- compares objects by structural equivalence
- Always import from `node:assert/strict` (enforces strict mode, no type coercion)

### Subtests and Hierarchical Organization

```js
import { test } from 'node:test'

test('Top level test', { concurrency: true }, t => {
  t.test('Subtest 1', () => { /* ... */ })
  t.test('Subtest 2', () => { /* ... */ })
})
```

> **Pattern:** Always enable `concurrency: true` on parent tests with subtests to keep tests fast.

### Parametrized Test Cases

Generate tests from data instead of repeating logic:

```js
test('Calculates basket total', { concurrency: true }, t => {
  const cases = [
    { name: 'Empty basket', basket: { items: [] }, expectedTotal: 0 },
    { name: 'One croissant', basket: { items: [{ name: 'Croissant', unitPrice: 2, quantity: 1 }] }, expectedTotal: 2 },
    { name: 'Mixed items', basket: { items: [
      { name: 'Croissant', unitPrice: 2, quantity: 2 },
      { name: 'Olive bread', unitPrice: 3, quantity: 1 },
    ]}, expectedTotal: 7 },
  ]

  for (const { name, basket, expectedTotal } of cases) {
    t.test(name, () => {
      equal(calculateBasketTotal(basket), expectedTotal)
    })
  }
})
```

### Test Suites with `suite()` / `describe()`

```js
import { describe, it } from 'node:test'

describe('Top level suite', { concurrency: true }, () => {
  it('Test 1', () => {})
  it('Test 2', () => {})
})
```

`suite()` = `describe()`, `test()` = `it()` -- pick whichever naming resonates with your team.

### Watch Mode

```bash
node --test --watch
```

Tests re-execute automatically when source files change. Creates a tight feedback loop.

### Glob Patterns and Targeted Execution

Default discovery patterns include `**/*.test.{cjs,mjs,js}`, `**/*-test.js`, `**/test/**/*.js`, etc.

Run specific subsets:

```bash
node --test "shoppingCart/**/*.test.js" "checkout/**/*.test.js"
```

### Filtering Tests

```bash
# By name pattern
node --test --test-name-pattern="suite 2"

# Skip by pattern
node --test --test-skip-pattern="Test 2"
```

In-code options: `skip`, `only`, `todo`:

```js
test('Skipped test', { skip: true }, () => {})
test.todo('Planned test')
suite.only('Exclusive suite', () => { /* ... */ })
```

> Use `--test-only` flag to activate the `only` option.

### Test Reporters

| Reporter | Description                                          |
| -------- | ---------------------------------------------------- |
| `spec`   | Default. Human-readable with colors and symbols      |
| `tap`    | Test Anything Protocol; machine- and human-readable  |
| `dot`    | Compact dots for large suites                        |
| `junit`  | JUnit XML for CI services (Jenkins, CircleCI)        |
| `lcov`   | Coverage reports in LCOV format                      |

```bash
# Multiple reporters simultaneously
node --test \
  --test-reporter=spec \
  --test-reporter=junit \
  --test-reporter-destination=console \
  --test-reporter-destination=results.xml
```

### Code Coverage

```bash
node --test --experimental-test-coverage
```

Filter coverage scope:

```bash
node --test --experimental-test-coverage --test-coverage-include="src/*.js"
```

Disable coverage for specific code sections:

```js
/* node:coverage disable */
if (falsyCondition) { console.error('unreachable') }
/* node:coverage enable */

/* node:coverage ignore next */
if (falsyCondition) { console.log('never executed') }
```

Generate HTML reports with `c8`:

```bash
npx c8 -r html node --test --experimental-test-coverage
```

### TypeScript Support

The test runner natively supports `.test.ts` files. Run with a custom glob:

```bash
node --test '**/*.test.ts'

# With coverage (exclude test files from coverage)
node --test --experimental-test-coverage --test-coverage-exclude='**/*.test.ts' '**/*.test.ts'
```

---

## Writing Unit Tests

### Testing Asynchronous Code

The FUT (function under test) can be:
1. **Synchronous** -- fails if it throws
2. **Promise-returning / async** -- fails if the promise rejects
3. **Callback-based** -- receives `done`; fails if `done(err)` is called with a truthy value

Example testing an async `TaskQueue` class (see [Async Control Flow](async-control-flow.md)):

```js
import { suite, test, mock } from 'node:test'
import assert from 'node:assert/strict'
import { once } from 'node:events'
import { setImmediate } from 'node:timers/promises'
import { TaskQueue } from './TaskQueue.js'

suite('TaskQueue', { concurrency: true, timeout: 500 }, () => {
  test('Respect the concurrency limit', async () => {
    const queue = new TaskQueue(4)
    let runningTasks = 0, maxRunningTasks = 0, completedTasks = 0

    const task = async () => {
      runningTasks++
      maxRunningTasks = Math.max(maxRunningTasks, runningTasks)
      await setImmediate()
      runningTasks--
      completedTasks++
    }

    queue.pushTask(task).pushTask(task).pushTask(task).pushTask(task).pushTask(task)
    await once(queue, 'empty')

    assert.equal(maxRunningTasks, 4)
    assert.equal(completedTasks, 5)
  })
})
```

> Use `timeout` on async suites to catch tests that hang due to missed `await` or logic bugs.

### Creating Spies with `mock.fn()`

`mock.fn()` creates a spy that tracks calls while preserving original behavior:

```js
import { test, mock } from 'node:test'
import assert from 'node:assert/strict'

test('All tasks executed', async () => {
  const queue = new TaskQueue(2)

  const task1 = mock.fn(async () => { await setImmediate() })
  const task2 = mock.fn(async () => { await setImmediate() })

  queue.pushTask(task1).pushTask(task2)
  await once(queue, 'empty')

  assert.equal(task1.mock.callCount(), 1)
  assert.equal(task2.mock.callCount(), 1)
})
```

Spy tracking capabilities:
- `mock.callCount()` -- how many times called
- `mock.calls[i].arguments` -- arguments per call
- `mock.calls[i].result` / `mock.calls[i].error` -- return values or thrown errors

### Mocking HTTP Requests with Built-in Mock

Use `t.mock.method()` to replace `fetch` scoped to a single test:

```js
test('Fetches internal links from a page', async t => {
  const mockHtml = `
    <html><body>
      <a href="https://example.com/blog">Blog</a>
      <a href="/about">About</a>
      <a href="https://external.com">External</a>
    </body></html>
  `

  t.mock.method(global, 'fetch', async _url => ({
    ok: true,
    status: 200,
    headers: {
      get: key => key === 'content-type' ? 'text/html; charset=utf-8' : null,
    },
    text: async () => mockHtml,
  }))

  const links = await getInternalLinks('https://example.com')
  assert.deepEqual(links, new Set([
    'https://example.com/blog',
    'https://example.com/about',
  ]))
})
```

> Using `t.mock.method()` (instead of top-level `mock.method()`) scopes the mock to the test and automatically restores it when done.

### Mocking HTTP Requests with Undici

Undici's `MockAgent` provides fine-grained control over mocked HTTP responses:

```js
import { MockAgent, setGlobalDispatcher } from 'undici'

test('Fetches internal links (undici)', async () => {
  const agent = new MockAgent()
  agent.disableNetConnect()     // Prevent real network calls
  setGlobalDispatcher(agent)

  agent
    .get('https://example.com')
    .intercept({ path: '/', method: 'GET' })
    .reply(200, mockHtml, {
      headers: { 'content-type': 'text/html; charset=utf-8' },
    })

  const links = await getInternalLinks('https://example.com')
  // ...assertions...
})
```

Use `beforeEach` / `afterEach` hooks to share agent setup across a suite:

```js
suite('with undici mock', () => {
  let agent
  const originalDispatcher = getGlobalDispatcher()

  beforeEach(() => {
    agent = new MockAgent()
    agent.disableNetConnect()
    setGlobalDispatcher(agent)
  })

  afterEach(() => {
    setGlobalDispatcher(originalDispatcher)
  })

  test('a test...', () => { /* configure agent.intercept() here */ })
})
```

### Mocking Node.js Core Modules

Use `t.mock.module()` to replace built-in modules like `node:fs/promises`:

```js
test('Creates folder if needed', async t => {
  const mockMkdir = mock.fn()
  const mockAccess = mock.fn(async () => { throw new Error('ENOENT') })

  t.mock.module('node:fs/promises', {
    cache: false,
    namedExports: {
      access: mockAccess,
      mkdir: mockMkdir,
      writeFile: mock.fn(),
    },
  })

  // Dynamic import AFTER mock is in place
  const { saveConfig } = await import('./saveConfig.js')
  await saveConfig('./path/to/configs/app.json', { port: 3000 })

  assert.equal(mockMkdir.mock.callCount(), 1)
})
```

Run with:

```bash
node --test --experimental-test-module-mocks
```

> Module mocks are global state. You cannot mock the same module concurrently in parallel tests. Set `concurrency: false` when mocking the same module from multiple tests.

### Mocking Other Dependencies

Replace internal or third-party modules the same way:

```js
const queryMock = mock.fn(async () => sampleRecords)

mock.module('./dbClient.js', {
  cache: false,
  namedExports: {
    DbClient: class DbMock { query = queryMock },
  },
})

const { canPayWithVouchers } = await import('./payments.js')

suite('canPayWithVouchers', { concurrency: false, timeout: 500 }, () => {
  beforeEach(() => { queryMock.mock.resetCalls() })
  after(() => { queryMock.mock.restore() })

  test('Returns true if balance is enough', async () => {
    const result = await canPayWithVouchers('user1', 18)
    assert.equal(result, true)
    assert.equal(queryMock.mock.callCount(), 1)
  })
})
```

### Import Mocking vs. Dependency Injection

**Problems with mocking imports:**
- Tight coupling between tests and implementation details
- Global scope -- mocks affect all imports in the process
- Module must be imported *after* mock is applied (dynamic imports required)
- Cannot mock the same module concurrently

**Dependency Injection** solves these issues by passing dependencies as parameters:

```js
// payments.js -- DI version
export async function canPayWithVouchers(db, userId, amount) {
  const vouchers = await db.query(/* ... */, [userId])
  const balance = vouchers.reduce((acc, v) => acc + v.balance, 0)
  return balance >= amount
}
```

```js
// payments.test.js -- clean test with DI
import { canPayWithVouchers } from './payments.js'  // static import works!

suite('canPayWithVouchers', { concurrency: true }, () => {  // concurrency enabled!
  test('Returns true if balance is enough', async t => {
    const dbMock = {
      query: t.mock.fn(async () => sampleRecords),
    }
    const result = await canPayWithVouchers(dbMock, 'user1', 18)
    assert.equal(result, true)
    assert.equal(dbMock.query.mock.callCount(), 1)
  })
})
```

Benefits of DI over import mocking:
- Static imports (no dynamic `import()`)
- Full concurrency support
- Each test creates its own isolated mock (no `beforeEach`/`after` cleanup)
- Easier to read, faster to run, less fragile

See [Creational Design Patterns](creational-design-patterns.md) for a deeper look at the DI pattern.

---

## Writing Integration Tests

### Testing with a Local Database

Use an **in-memory SQLite database** for fast, isolated integration tests without external server setup:

```js
import { suite, test } from 'node:test'
import assert from 'node:assert/strict'
import { DbClient } from './dbClient.js'
import { createTables } from './dbSetup.js'
import { getActiveVouchers, canPayWithVouchers } from './payments.js'

suite('activeVouchers', { concurrency: true, timeout: 500 }, () => {
  test('queries for active vouchers', async () => {
    const db = new DbClient(':memory:')
    await createTables(db)

    await addTestUser(db, 'user1', 'Test User 1')
    const expected = []
    expected.push(await addTestVoucher(db, 'v1', 'user1', 10))
    expected.push(await addTestVoucher(db, 'v2', 'user1', 5))

    // These should be filtered out:
    await addTestVoucher(db, 'v3', 'user1', 10, expiredDate)  // expired
    await addTestVoucher(db, 'v4', 'user2', 10)               // different user
    await addTestVoucher(db, 'v5', 'user1', 0)                // zero balance

    const result = await getActiveVouchers(db, 'user1')
    db.close()
    assert.deepEqual(result, expected)
  })
})
```

Key pattern: each test creates a **fresh in-memory database**, ensuring full isolation and safe concurrency.

### Testing a Web Application

Using **Fastify's `inject()` method** to simulate HTTP requests without starting a real server:

```js
import { createApp } from './app.js'
import { DbClient } from './dbClient.js'
import { createTables } from './dbSetup.js'

suite('Booking integration tests', { concurrency: true }, () => {
  test('Reserving a seat works until full', async () => {
    const db = new DbClient(':memory:')
    await createTables(db)
    const app = await createApp(db)

    // Create event with 2 seats
    const createRes = await app.inject({
      method: 'POST', url: '/events',
      payload: { name: 'Event 1', totalSeats: 2 },
    })
    assert.equal(createRes.statusCode, 201)
    const { eventId } = createRes.json()

    // Reserve both seats successfully
    const res1 = await app.inject({
      method: 'POST', url: `/events/${eventId}/reservations`,
      payload: { userId: 'u1' },
    })
    assert.equal(res1.statusCode, 201)

    const res2 = await app.inject({
      method: 'POST', url: `/events/${eventId}/reservations`,
      payload: { userId: 'u2' },
    })
    assert.equal(res2.statusCode, 201)

    // Third reservation should fail (fully booked)
    const res3 = await app.inject({
      method: 'POST', url: `/events/${eventId}/reservations`,
      payload: { userId: 'u3' },
    })
    assert.equal(res3.statusCode, 403)
    assert.deepEqual(res3.json(), { error: 'Event is fully booked' })

    await db.close()
    await app.close()
  })
})
```

> For server-based databases (PostgreSQL, MySQL), consider [testcontainers](https://node.testcontainers.org/) to spin up ephemeral Docker containers for each test worker.

---

## Writing E2E Tests

E2E testing validates the product **from the user's perspective** -- simulating actual user behavior through the UI without peeking inside the system.

### Playwright Setup

Initialize a project:

```bash
npm init playwright@latest
```

This generates `playwright.config.ts`, example tests, and downloads browser binaries (Chromium, Firefox, WebKit).

Run tests:

```bash
npx playwright test              # headless, all browsers
npx playwright test --headed     # opens real browser windows
npx playwright test --ui         # interactive UI mode with time-travel debugger
```

### Playwright API Essentials

**Navigation:**

```js
await page.goto('http://localhost:3000')
```

**Locator API** (finds elements reliably across re-renders):

| Locator                      | Targets                          |
| ---------------------------- | -------------------------------- |
| `page.getByRole()`          | Accessibility roles and attributes |
| `page.getByText()`          | Visible text content             |
| `page.getByLabel()`         | Form controls by label           |
| `page.getByPlaceholder()`   | Inputs by placeholder            |
| `page.getByTestId()`        | Elements by `data-testid`        |

**Actions on locators:**

```js
await locator.click()
await locator.fill('some text')
await locator.check()
await locator.hover()
await locator.selectOption('value')
```

**Assertions** (web-first -- automatically wait and retry):

```js
await expect(page).toHaveTitle(/Playwright/)
await expect(locator).toBeVisible()
await expect(locator).toHaveText('Expected text')
await expect(locator).toBeDisabled()
await expect(page).toHaveURL(/dashboard/)
```

### Timeouts

| Operation             | Default Timeout | Behavior                                |
| --------------------- | --------------- | --------------------------------------- |
| Web-first assertions  | 5 seconds       | Retry until true or timeout             |
| Sync assertions       | Immediate       | No retry; fails instantly               |
| Actions (click, fill) | 0 (no limit)    | Wait for actionability; hangs until test timeout |
| Navigation (goto)     | 0 (no limit)    | Waits for load; hangs until test timeout |

Configure globally in `playwright.config.ts` with `timeout`, `actionTimeout`, and `navigationTimeout`.

### Example: Full User Flow

```js
import { test, expect } from '@playwright/test'

test('A user can sign up and book an event', async ({ page }) => {
  // 1. Navigate to home page
  await page.goto('http://localhost:3000')

  // 2. Go to Sign In -> Sign Up
  await page.getByRole('link', { name: 'Sign In' }).click()
  await page.getByRole('link', { name: 'Sign up' }).click()

  // 3. Fill registration form with unique data
  const seed = Date.now().toString()
  await page.getByRole('textbox', { name: 'name' }).fill(`TestUser ${seed}`)
  await page.getByRole('textbox', { name: 'email' }).fill(`test${seed}@example.com`)
  await page.getByRole('textbox', { name: 'password' }).fill(`password${seed}`)
  await page.getByRole('button', { name: 'Create account' }).click()

  // 4. Click on an event
  await page.getByRole('link', { name: 'Marathon City Run' }).click()

  // 5. Reserve a spot
  const capacity = Number.parseInt(
    await page.getByTestId('available-capacity').textContent()
  )
  await page.getByRole('button', { name: 'Reserve your spot' }).click()

  // 6. Verify booking
  await expect(page.getByTestId('badge').first()).toHaveText('Booked')
  const bookBtn = page.getByRole('button', { name: 'You have booked this event!' })
  await expect(bookBtn).toBeDisabled()
  const newCapacity = Number.parseInt(
    await page.getByTestId('available-capacity').textContent()
  )
  expect(newCapacity).toBeLessThan(capacity)

  // 7. Check dashboard
  await page.getByRole('link', { name: 'My Reservations' }).click()
  await expect(page.getByRole('heading', { name: 'My Reservations' })).toBeVisible()
  expect(await page.getByRole('heading', { name: 'Marathon City Run' })).toBeVisible()
})
```

### E2E Best Practices

- Run tests in a **dedicated staging environment** seeded with predictable data
- Use `data-testid` attributes for stable element selection alongside accessible locators
- Generate unique test data (timestamps, UUIDs) to avoid conflicts between runs
- Use sandbox environments for payment integrations (Stripe test mode, etc.)
- Keep E2E tests focused on **critical user flows** -- not every edge case
- E2E tests require cross-functional collaboration (dev, QA, product) to maintain

---

## Quick Reference

| Concept | Tool / API |
| --- | --- |
| Test runner | `node --test` |
| Assertions | `node:assert/strict` (`equal`, `deepEqual`, `ok`) |
| Suites | `suite()` / `describe()` from `node:test` |
| Test cases | `test()` / `it()` from `node:test` |
| Spies | `mock.fn()` from `node:test` |
| Method mocking | `t.mock.method(obj, 'method', impl)` |
| Module mocking | `t.mock.module('module', { namedExports })` |
| HTTP mocking | `Undici MockAgent` or `t.mock.method(global, 'fetch')` |
| Watch mode | `node --test --watch` |
| Coverage | `node --test --experimental-test-coverage` |
| E2E testing | Playwright (`@playwright/test`) |
| Coverage HTML | `npx c8 -r html node --test --experimental-test-coverage` |

---

## See Also

- [Async Control Flow](async-control-flow.md) -- TaskQueue and concurrency patterns tested in this chapter
- [Creational Design Patterns](creational-design-patterns.md) -- Dependency Injection pattern
- [Behavioral Design Patterns](behavioral-design-patterns.md) -- Observer/EventEmitter patterns used in test examples
- [Advanced Recipes](advanced-recipes.md) -- Delayed initialization, request batching, cancellation
