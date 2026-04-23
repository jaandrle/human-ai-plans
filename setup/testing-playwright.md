# 🚀 Playwright Testing Guide

**A practical single-tool testing strategy using Playwright across logic, components, hooks, and critical E2E flows**

## Overview

This guide presents a **behavior-first testing strategy** built around Playwright as the **single testing tool** for modern frontend projects.

It covers:
- **Logic tests** for business rules and pure utilities
- **React component tests** in a real browser
- **React hooks tests** using wrapper components
- **Critical E2E journeys** for regression protection
- **CI/CD integration** for pull-request safety

The approach is based on **real project experience** and is designed to be **portable across multiple React/Vite codebases**.

> **Core philosophy:** test what users can observe, not implementation internals.

## Testing Philosophy

This strategy combines ideas from:

### Chris Ferdinandi[^cf]
[^cf] https://github.com/cferdinandi/tdd
- TDD workflow examples
- Behavior-driven UI testing
- Progressive enhancement mindset

### Playwright Guidance[^pg]
[^pg] https://playwright.dev/docs/intro
- Component Testing
- Isolation and fixtures
- Browser-native execution

### Kent C. Dodds[^kcd]
[^kcd] https://kentcdodds.com/blog/testing-implementation-details
- “Testing Implementation Details (Why Not To)”
- “Write Tests. Not Too Many. Mostly Integration.”

### Guiding Principles
- Test **observable behavior**
- Treat the **browser as the public API**
- Prefer **high-confidence tests over broad shallow coverage**
- Protect **critical regressions in CI**

## Why Playwright?

**Key benefits:**
- Real browser testing (no jsdom)
- Single tool for all test types
- Excellent React support
- Low flakiness

**Core principles:**
- Test user-observable behavior
- Prioritize high-value tests
- Use CI for regression protection

## Installation
Use the interactive installer — it sets up dependencies, config files, browser binaries, Vite/React integration, and TypeScript automatically.

```bash
npm init playwright@latest

# For React component testing
npm init playwright@latest -- --ct
```

## Configuration

### Unit Testing Configuration

**File:** `playwright.unit.config.ts`

**Purpose:** Unit and component tests for React

**Key Features:**
- Uses `@playwright/experimental-ct-react`
- Real Chromium browser environment
- Vite configuration for module resolution
- Runs on port 3100

**Usage:**
```bash
# Run tests
npx playwright test -c playwright.unit.config.ts

# UI mode
npx playwright test -c playwright.unit.config.ts --ui
```

**Key Settings:**
- `testDir: "./src/"` - Test files in src directory
- `fullyParallel: true` - Parallel execution
- `ctViteConfig` - Custom Vite config with path aliases
- **Note:** Uses separate Vite config, not the main project config

### E2E Testing Configuration

**File:** `playwright.e2e.config.ts`

**Purpose:** End-to-end tests for user flows

**Key Features:**
- Tests complete application behavior
- Runs on port 5173
- Supports multiple browsers

**Usage:**
```bash
# Run E2E tests
npx playwright test -c playwright.e2e.config.ts

# Debug mode
npx playwright test -c playwright.e2e.config.ts --headed
```

**Key Settings:**
- `testDir: "./tests"` - E2E tests directory
- `baseURL: "http://localhost:5173"` - App URL
- `webServer.timeout: 120000` - 2 minute timeout


### Project Structure
```
src/
  components/
    Button.tsx
    Button.test.tsx

  core/
    __components.tsx      # hook wrappers for tests
    useMyHook.ts
    useMyHook.test.tsx
    math.ts
    math.test.ts

  app/ # pages/endpoint

tests/
  e2e/
    login.spec.ts

playwright.unit.config.ts
playwright.e2e.config.ts
```

> This structure scales well across multiple projects because the testing intent is obvious from the file layout.

## Testing Strategy

### 1. Logic Tests (Fastest ROI)
Test pure functions, business rules, and validation logic. No browser needed — highest confidence per test written.

```ts
import { test, expect } from '@playwright/test';
import { calculateTotal } from './pricing';

test('adds tax correctly', () => {
  expect(calculateTotal(100, 0.2)).toBe(120);
});
```

### 2. Components tests
Focus on:
- user interactions
- accessibility roles
- visible state changes
- form behavior

**Portability Rule**, always prefer:
- `getByRole`
- `getByLabel`
- `getByText`

Avoid CSS selectors unless unavoidable.

This keeps tests resilient across redesigns.

#### Vanilla JS component tests
For Web Components or plain JS — inject HTML with `page.setContent()` and load your module with `page.addScriptTag()`.

```js
import { test, expect } from '@playwright/test';
test("disclosure toggles", async ({ page }) => {
  await page.setContent(`
    <button aria-expanded="false" aria-controls="content">Toggle</button>
    <div id="content" hidden>Hello</div>
  `);

  await page.addScriptTag({ path: "src/disclosure.js", type: "module" });
  await page.evaluate(() => {
    const button = document.querySelector("button");
    window.initDisclosure(button);
  });

  const button = page.getByRole("button");
  await button.click();
  await expect(button).toHaveAttribute("aria-expanded", "true");
  await expect(page.getByText("Hello")).toBeVisible();
});
```

#### React/Vite/…?
Test component behavior using `getByRole` and `getByLabel`.

**Example:**
```typescript
import { test, expect } from "@playwright/experimental-ct-react";
import { LoginForm } from "./LoginForm";
test('submit enables when form is valid', async ({ mount }) => {
  const component = await mount(<LoginForm />);
  
  await component.getByLabel('Email').fill('user@test.com');
  await component.getByLabel('Password').fill('password123');
  
  await expect(component.getByRole('button', { name: 'Submit' })).toBeEnabled();
});
```

#### TanStack Router, …
Still mostly not working.

### 3. React Hooks Testing (Reusable Workaround)

Direct hook calls inside Playwright tests are unreliable.

The most portable solution is a **dedicated wrapper component pattern**.

### Wrapper Component
```typescript
// src/core/__components.tsx
export function TestUsePartialIp({ ip, onResult }) {
  const result = usePartialIp(ip);

  useEffect(() => {
    onResult(result);
  }, [result]);

  return null;
}
```

### Hook Test
```typescript
import { test, expect } from "@playwright/experimental-ct-react";
import { TestUsePartialIp } from "./__components";
test('transforms IP correctly', async ({ mount }) => {
  let result = '';

  await mount(
    <TestUsePartialIp
      ip="192.168.1.100"
      onResult={(r) => {
        result = r;
      }}
    />
  );

  expect(result).toBe('192.168.1.');
});
```

**File Structure Recommendation:**
```
src/
  core/
    __components.tsx      # Test wrapper components
    useMyHook.ts          # Original hook
    useMyHook.test.tsx    # Tests using wrapper components
```

This gives every project a standard place for hook wrappers.

### Known issues
**At least for now can’t test hooks/components that use TanStack Query or Tanstack Router hooks!!!**

## 4. E2E Tests (Critical Paths Only)

Limit E2E to **business-critical workflows**.

Recommended first targets:
- login
- checkout
- onboarding
- report generation
- settings save flow

```typescript
test('user logs in and sees dashboard', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password');
  await page.getByRole('button', { name: 'Login' }).click();
  await expect(page).toHaveURL(/dashboard/);
});
```

### Rule of Thumb
Start with **3–5 flows max**.

Too many E2E tests early = slower teams and fragile pipelines.

## Best Practices

### Do

- Behavior assertions
- Accessibility queries
- Use auto-waiting assertions (no manual timeouts)
- Mock APIs with `page.route()`
- Keep tests isolated with deterministic data
- Write descriptive test names that describe behavior

```ts
// ❌ Bad
test('button works', ...)

// ✅ Good
test('submit button enables when form is valid', ...)
```

### Don't

- Start with too many E2E tests
- Chase 100% coverage
- Test implementation details
- Use `waitForTimeout()`
- Skip CI integration

## CI Integration

> **No CI = No protection.** Tests only matter if they block broken code from shipping.

Minimum CI requirements:

- Run on every PR
- Store trace artifacts for post-failure debugging
- Fail the build on any test failure

As your test suite grows:

- Enable parallelization
- Monitor and track flakiness
- Fail the build on regressions

### Scaling
When tests grow:
- Enable parallelization
- Monitor flakiness
- Fail build on regressions

## Suggested 2-Week Rollout Plan

A reusable adoption plan for new projects:

### Week 1
- install Playwright
- create both configs
- protect core utilities
- add first component tests

### Week 2
- add 3 critical E2E journeys
- connect CI
- enable traces
- remove flaky legacy tests

## Resources
- [Playwright Docs](https://playwright.dev/docs/intro)
- [Component Testing](https://playwright.dev/docs/test-components)

**Goal:** Stable protection of critical behavior, not 100% coverage.
