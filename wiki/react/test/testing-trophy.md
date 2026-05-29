# The Testing Trophy

> Sources: Kent C. Dodds, 2021-06-03; Kent C. Dodds, 2021-06-03; Kent C. Dodds, 2019-07-13
> Raw: [The Testing Trophy and Testing Classifications](../../../raw/react/2021-06-03-the-testing-trophy-and-testing-classifications.md); [Static vs Unit vs Integration vs E2E Tests](../../../raw/react/2021-06-03-static-vs-unit-vs-integration-vs-e2e-tests.md); [Write tests. Not too many. Mostly integration.](../../../raw/react/2019-07-13-write-tests-not-too-many-mostly-integration.md)
> Updated: 2026-05-29

## Overview

The Testing Trophy is a model for thinking about the **return on investment** of different forms of testing in JavaScript applications. Introduced by Kent C. Dodds as a frontend-focused alternative to the traditional Testing Pyramid, it visualizes four test types — Static, Unit, Integration, and End-to-End — with their relative sizes indicating how much focus each should receive. The trophy's central insight: integration tests offer the best confidence-to-cost ratio for most UI codebases.

---

## Origin and Intent

The Testing Trophy grew from Guillermo Rauch's maxim:

> "Write tests. Not too many. Mostly integration."

Kent C. Dodds adopted this philosophy and added **Static** analysis as a foundational layer — something implicit in statically-typed languages but an active choice in JavaScript (ESLint, TypeScript, Flow). The trophy was originally conceived for **frontend monolith codebases** viewed in isolation — it was never designed for microservices or serverless architectures, though the principles can apply to backend monoliths as well.

The term "integration" in this model means testing multiple units of your own code working together, not system-level integration between services. A "unit" is a single function, class, or object containing logic.

---

## The Four Levels

From bottom to top of the trophy:

| Level | What It Tests | Key Tools |
|---|---|---|
| **Static** | Typos, type errors, syntax mistakes | ESLint, TypeScript, Flow |
| **Unit** | Individual isolated parts (no or mocked dependencies) | Jest, Vitest, Testing Library |
| **Integration** | Multiple units working together in harmony | Testing Library + MSW |
| **End-to-End** | Full user flows through the real application | Cypress, Playwright |

### Static

Catches bugs at write-time without running code. In JavaScript this is an active investment (unlike Java/C# where the compiler handles it):

```js
// ESLint's for-direction rule catches this infinite loop
for (var i = 0; i < 10; i--) {
  console.log(i)
}

// TypeScript catches this type mismatch
const two = '2'
const result = add(1, two)
```

### Unit

Tests individual, isolated pieces. Can test React components or pure functions:

```js
import { render, screen } from '@testing-library/react'
import ItemList from '../item-list'

test('renders "no items" when the item list is empty', () => {
  render(<ItemList items={[]} />)
  expect(screen.getByText(/no items/i)).toBeInTheDocument()
})

test('renders the items in a list', () => {
  render(<ItemList items={['apple', 'orange', 'pear']} />)
  expect(screen.getByText(/apple/i)).toBeInTheDocument()
  expect(screen.getByText(/orange/i)).toBeInTheDocument()
  expect(screen.getByText(/pear/i)).toBeInTheDocument()
})
```

Pure function unit tests are ideal for `jest-in-case` / parametrized testing:

```js
import cases from 'jest-in-case'
import fizzbuzz from '../fizzbuzz'

cases(
  'fizzbuzz',
  ({ input, output }) => expect(fizzbuzz(input)).toBe(output),
  [
    [1, '1'], [2, '2'], [3, 'Fizz'],
    [5, 'Buzz'], [9, 'Fizz'], [15, 'FizzBuzz'], [16, '16'],
  ].map(([input, output]) => ({
    title: `${input} => ${output}`, input, output,
  })),
)
```

### Integration

Mock as little as possible — typically only network requests (via MSW) and animation components. Render with all the app's providers to test components in a realistic context:

```js
import { render, screen, waitForElementToBeRemoved } from 'test/app-test-utils'
import userEvent from '@testing-library/user-event'
import { rest } from 'msw'
import { setupServer } from 'msw/node'
import { handlers } from 'test/server-handlers'
import App from '../app'

const server = setupServer(...handlers)
beforeAll(() => server.listen())
afterAll(() => server.close())
afterEach(() => server.resetHandlers())

test(`logging in displays the user's username`, async () => {
  await render(<App />, { route: '/login' })
  const { username, password } = buildLoginForm()

  userEvent.type(screen.getByLabelText(/username/i), username)
  userEvent.type(screen.getByLabelText(/password/i), password)
  userEvent.click(screen.getByRole('button', { name: /submit/i }))

  await waitForElementToBeRemoved(() => screen.getByLabelText(/loading/i))
  expect(screen.getByText(username)).toBeInTheDocument()
})
```

### End-to-End

Run the full application (frontend + backend) and interact like a real user. Keep these focused on critical user flows:

```js
describe('todo app', () => {
  it('should work for a typical user', () => {
    const user = generate.user()
    const todo = generate.todo()

    cy.visitApp()
    cy.findByText(/register/i).click()
    cy.findByLabelText(/username/i).type(user.username)
    cy.findByLabelText(/password/i).type(user.password)
    cy.findByText(/login/i).click()

    cy.findByLabelText(/add todo/i)
      .type(todo.description)
      .type('{enter}')

    cy.findByTestId('todo-0').should('have.value', todo.description)
    cy.findByLabelText('complete').click()
    cy.findByTestId('todo-0').should('have.class', 'complete')
  })
})
```

---

## Trade-offs

Three dimensions change as you move up the trophy:

| Dimension | Bottom (Static) | Top (E2E) |
|---|---|---|
| **Cost** | Cheap (runs in editor, fast CI) | Expensive (full environments, slow CI, more maintenance) |
| **Speed** | Instant (no code execution) | Slow (entire app stack running) |
| **Confidence** | Catches simple problems (typos, types) | Catches big problems (real user flows breaking) |

### The Confidence Coefficient

As you move up the trophy, you increase the **confidence coefficient** — the relative confidence each test provides. Above the trophy sits manual testing (highest confidence, highest cost). Below sits no testing at all.

The key insight that separates the trophy from the pyramid: **if cost and speed were the only trade-offs, you'd write 100% unit tests.** The reason you don't is that lower-level tests simply cannot catch certain classes of bugs:

- **Static** cannot validate business logic
- **Unit** cannot verify you're calling dependencies correctly (you can assert on mock calls, but can't prove the real dependency behaves as expected)
- **Integration** cannot ensure correct data reaches your backend or that you handle real API errors properly
- **E2E** is the most capable but runs in non-production environments, trading some confidence for practicality

Conversely, using a high-level test for something a lower level handles better is wasteful — an E2E test for a coupon code edge case requires spinning up the entire app when a unit test would suffice.

---

## The Guiding Principle

> "The more your tests resemble the way your software is used, the more confidence they can give you."

This is the guiding principle behind Testing Library and the Testing Trophy. Testing is a **return on investment** problem where "return" is confidence and "investment" is time. The trophy helps allocate that investment: heavy on integration (best ROI for frontend), supported by static analysis and unit tests, anchored by focused E2E tests for critical paths.

---

## Coverage: Not Too Many

The "not too many" part of the maxim is about diminishing returns. Mandating 100% code coverage for applications is counterproductive — the value of each additional test declines sharply after roughly 70% coverage. Chasing the last 30% often means:

- Testing code with no logic (pure wiring that ESLint or TypeScript already validates)
- Testing implementation details just to reach hard-to-reproduce branches
- Building a maintenance burden that slows the team during refactors

> You should very rarely have to change tests when you refactor code. If tests break during a refactor that preserves behavior, the tests were testing implementation details.

**Applications vs libraries:** 100% coverage can make sense for small, widely-reused open-source libraries where a single bug cascades into many consumers. For applications, the goal is **confidence in use cases**, not coverage numbers. See [Testing: What to Test](testing-what-to-test.md) for the use case coverage model.

---

## Writing More Integration Tests

The single most impactful change to write more integration tests: **stop mocking so much.** Every mock removes confidence in the integration between the code under test and the mocked dependency. You prove your code works with the mock, not with the real thing.

Legitimate reasons to mock:
- Network boundaries (use MSW to intercept HTTP, not to replace your data layer)
- Side effects with real-world consequences (emails, payments, SMS)

Everything else — internal modules, sibling components, utility functions — should be exercised for real. The deeper the mock boundary, the more false confidence the test provides.

**React-specific:** avoid shallow rendering. Shallow rendering tests a component in isolation from its children, which is precisely the integration you need to verify. If `<A />` renders `<B />` with props `c` and `d`, but `<B />` breaks without prop `e`, a shallow test of `<A />` will pass while the real app fails.

---

## Trophy vs Pyramid

The traditional Testing Pyramid (Martin Fowler / Mike Cohn) recommends many unit tests, fewer integration tests, and very few E2E tests. The pyramid assumed that integration tests were inherently slow and expensive — an assumption that modern tools (Testing Library, MSW, Vitest) have largely invalidated. The Testing Trophy shifts focus toward integration tests because:

1. A single integration test covers more code paths than multiple unit tests combined
2. Frontend units (components) rarely work in isolation — their value comes from composition
3. Mocking everything (as unit tests require) can create a false sense of security
4. Modern tools (Testing Library, MSW) have made integration tests nearly as fast and reliable as unit tests

The pyramid remains valid for contexts where integration tests are expensive (microservices, distributed systems). The trophy is optimized for the reality of frontend monolith codebases.

> "People love debating what percentage of which type of tests to write, but it's a distraction. Nearly zero teams write expressive tests that establish clear boundaries, run quickly & reliably, and only fail for useful reasons. Focus on that instead." — Justin Searls

---

## See Also

- [Testing: What to Test in React](testing-what-to-test.md) — use case coverage vs code coverage, observable behavior, and prioritization strategy
- [Testing Patterns](../../nodejs/testing-patterns.md) — the Testing Pyramid model, Node.js test runner, mocking strategies, and E2E with Playwright
