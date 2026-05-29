# Testing: What to Test in React

> Sources: Kent C. Dodds, 2019-04-13
> Raw: [How to Know What to Test](../../../raw/react/2019-04-13-how-to-know-what-to-test.md)

## Overview

The hardest part of testing is not learning tools or syntax — it's knowing **what** to test. The guiding principle: test use cases, not code. Think less about the code being tested and more about the use cases that code supports. The more tests resemble how software is actually used, the more confidence they provide.

---

## Use Case Coverage vs Code Coverage

### The Core Distinction

Code coverage measures which lines execute during tests. Use case coverage measures how many real user scenarios are verified. They are related but not interchangeable:

- **Code coverage** answers: "Which lines ran?"
- **Use case coverage** answers: "Which user behaviors are protected?"

When looking at uncovered lines, the productive question is not "how do I cover this branch?" but rather:

> What use cases are these lines of code supporting, and what tests can I add to support those use cases?

### 100% Code Coverage ≠ 100% Use Case Coverage

Code coverage can reach 100% while missing important use cases. Consider two implementations of the same function — one with three branches, another with two that implicitly handles the third case via `.filter(Boolean)`. The second can achieve full coverage without ever testing the falsy-input scenario explicitly.

The danger: someone removes the implicit handling, tests still pass, but dependent code breaks.

**Key principle:** Tests exist to ensure code _continues_ to support intended use cases as things change. Missing a use case means silent regressions can slip through even with perfect coverage numbers.

---

## What to Test in React Components

React components have two users:
1. **End users** — people interacting with the UI
2. **Developer users** — other developers rendering and configuring the component

### Don't Test (Implementation Details)

Testing these directly couples tests to internals and creates a fragile "third user" of the code:

- Lifecycle methods
- Element event handlers (the handler function itself)
- Internal component state

### Do Test (Observable Behavior)

Test the **observable effects** that matter to your two real users:

| What to Test | Tool | Question It Answers |
|---|---|---|
| User interactions | `userEvent` (`@testing-library/user-event`) | Can the end user interact with rendered elements? |
| Changing props | `rerender` (React Testing Library) | What happens when a developer re-renders with new props? |
| Context changes | `rerender` (React Testing Library) | What happens when context changes trigger re-render? |
| Subscription changes | (varies) | What happens when an external source (store, router, media query, firebase) emits? |

Each of these can change the DOM, make HTTP requests, call callback props, or produce other observable side effects — those effects are what tests should assert on.

---

## Where to Start Testing an App

When facing a large application with no tests, use this prioritization strategy:

### Step 1: Identify Critical Use Cases

Ask the team:

> "What would be the worst thing to break in this app?"

Make a prioritized list of features based on user impact. This exercise also builds organizational buy-in for testing investment.

### Step 2: Write E2E Tests for Happy Paths

Start with a single E2E test covering the "happy path" most users follow. This often touches several top-priority features at once, delivering maximum confidence for the investment.

Don't aim for 100% use case coverage or code coverage with E2E tests.

### Step 3: Layer in Integration and Unit Tests

After E2E coverage exists for critical paths:
- Add **integration tests** for edge cases not covered by E2E
- Add **unit tests** for complex business logic

This builds up a balanced testing portfolio over time without the paralysis of trying to test everything at once.

---

## Testing Prioritization Mental Model

```
1. E2E for critical happy paths (highest confidence per test)
2. Integration for important edge cases
3. Unit for complex logic
```

The goal is confidence, not coverage numbers. Add tests incrementally, focused on the use cases that matter most.

---

## See Also

- [The Testing Trophy](testing-trophy.md) — the Testing Trophy model, four test type classifications, trade-offs (cost/speed/confidence), and the confidence coefficient
- [React Re-renders](../react-re-renders.md) — understanding what triggers re-renders helps identify what state/prop changes to test
- [Error Handling in React](../error-handling.md) — error boundaries are key observable behaviors to test

