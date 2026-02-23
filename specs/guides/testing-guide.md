# Testing Guide

This is the consolidated guide for testing, covering unit tests, integration tests, and E2E tests.

---

## Table of Contents

1. [Overview](#overview)
2. [Unit & Integration Tests (@effect/vitest)](#unit--integration-tests-effectvitest)
3. [E2E Tests (Playwright)](#e2e-tests-playwright)
4. [Test Data Patterns](#test-data-patterns)
5. [CI Integration](#ci-integration)
6. [Running Tests](#running-tests)

---

## Overview

| Test Type | Framework | Location | Purpose |
|-----------|-----------|----------|---------|
| Unit | @effect/vitest | `src/test/*.test.ts`, `convex/test/*.test.ts` | Test individual functions, services |
| Integration | @effect/vitest | `src/test/*.test.ts` | Test services with Convex |
| E2E | Playwright | `tests/*.spec.ts` | Test full user flows in browser |

---

## Unit & Integration Tests (@effect/vitest)

For comprehensive Effect testing patterns, see [effect-guide.md](./effect-guide.md#testing-patterns-effectvitest).

### Quick Reference

```typescript
import { describe, expect, it, layer } from "@effect/vitest"
import { Effect, TestClock, Fiber, Duration } from "effect"
```

### Test Variants

| Method | TestServices | Use Case |
|--------|--------------|----------|
| `it.effect` | TestClock | Most tests - deterministic time |
| `it.live` | Real clock | Tests needing real time/IO |
| `it.scoped` | TestClock | Tests with resources |

### Basic Example

```typescript
it.effect("processes after delay", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.fork(
      Effect.sleep(Duration.minutes(5)).pipe(Effect.map(() => "done"))
    )
    yield* TestClock.adjust(Duration.minutes(5))
    const result = yield* Fiber.join(fiber)
    expect(result).toBe("done")
  })
)
```

### Shared Layers

```typescript
layer(ItemServiceLive)("ItemService", (it) => {
  it.effect("finds item by id", () =>
    Effect.gen(function* () {
      const service = yield* ItemService
      const item = yield* service.findById(testItemId)
      expect(item.name).toBe("Test")
    })
  )
})
```

### Testing Services with Layers

```typescript
const TestLayer = ItemServiceLive.pipe(
  Layer.provide(ConvexClientTest)  // Test Convex client
)

layer(TestLayer, { timeout: "30 seconds" })("ItemService", (it) => {
  it.effect("creates and retrieves", () =>
    Effect.gen(function*() {
      const service = yield* ItemService
      const item = yield* service.create({ name: "Test" })
      const found = yield* service.findById(item.id)
      expect(found.name).toBe("Test")
    })
  )
})
```

---

## E2E Tests (Playwright)

### Project Structure

```
project/
├── playwright.config.ts
├── tests/
│   ├── fixtures/
│   │   └── auth.ts           # Authentication helpers
│   ├── auth/
│   │   ├── login.spec.ts
│   │   └── register.spec.ts
│   └── items/
│       └── crud.spec.ts
└── test-results/
```

### Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: "./tests",
  fullyParallel: false,  // Sequential for shared DB
  workers: 1,
  retries: process.env.CI ? 2 : 0,

  use: {
    baseURL: "http://localhost:5173",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
    actionTimeout: 10000,
    navigationTimeout: 30000,
  },

  webServer: {
    command: `pnpm dev`,
    url: "http://localhost:5173",
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
})
```

### CRITICAL: Use data-testid for Element Selection

**ALL element selection in E2E tests MUST use `data-testid` attributes.**

```typescript
// CORRECT: Always use data-testid
await page.click('[data-testid="submit-button"]')
await page.fill('[data-testid="login-email-input"]', email)
await expect(page.locator('[data-testid="user-menu"]')).toBeVisible()

// WRONG: CSS class selectors - fragile
await page.click('button.primary-btn.submit')

// WRONG: Text selectors - break on copy changes
await page.click('text=Submit')

// WRONG: Structural selectors - break on DOM changes
await page.click('form > div:nth-child(2) > button')
```

**Naming conventions:**
- Use kebab-case: `data-testid="submit-button"`
- Be descriptive: `data-testid="item-list"`
- Include context: `data-testid="login-email-input"`
- For lists: `data-testid="item-row-{id}"`

### Test Patterns

#### Authentication Flow

```typescript
test("should login with valid credentials", async ({ page, testUser }) => {
  await page.goto("/login")
  await page.fill('[data-testid="login-email-input"]', testUser.email)
  await page.fill('[data-testid="login-password-input"]', testUser.password)
  await page.click('[data-testid="login-submit-button"]')

  await page.waitForURL("/")
  await expect(page.locator('[data-testid="user-menu"]')).toBeVisible()
})
```

#### Use API for Setup, UI for Assertions

```typescript
test("should display created item", async ({ authenticatedPage, testUser }) => {
  // Create via API (fast)
  await authenticatedPage.request.post("/api/v1/items", {
    data: { name: "New Item", categoryId: "cat_123" }
  })

  // Verify in UI
  await authenticatedPage.goto("/items")
  await expect(authenticatedPage.getByText("New Item")).toBeVisible()
})
```

---

## Test Data Patterns

### Unique Identifiers

Always use timestamps and random strings for test data:

```typescript
const timestamp = Date.now()
const random = Math.random().toString(36).slice(2, 8)

const testEmail = `test-${timestamp}-${random}@example.com`
const testName = `Item-${timestamp}-${random}`
```

### Test Isolation

Each test should be independent:

```typescript
// GOOD: Create fresh data for each test
test("should create item", async ({ authenticatedPage }) => {
  // Fresh data created in fixture
})

// BAD: Relying on data from previous test
test("should list items", async ({ authenticatedPage }) => {
  // Assumes items were created in previous test
})
```

---

## CI Integration

### GitHub Actions Workflow

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install
      - run: pnpm typecheck
      - run: pnpm test

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

---

## Running Tests

### Unit/Integration Tests

```bash
# Run all tests (minimal output)
pnpm test

# Run with full output
pnpm test:verbose

# Run with coverage
pnpm test:coverage

# Run specific test file
pnpm test src/test/Item.test.ts
```

### E2E Tests

```bash
# Run all E2E tests
pnpm test:e2e

# Run with interactive UI
pnpm test:e2e:ui

# Run specific test file
pnpm test:e2e --grep "login"

# Run in headed mode (see browser)
pnpm test:e2e --headed

# Debug specific test
pnpm test:e2e --debug tests/auth/login.spec.ts
```
