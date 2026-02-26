# 🚀 Playwright Testing (Rollout) Plan

**From Zero → Structured Coverage**

# Overview

This plan defines a **single-tool, browser-first testing strategy** using Playwright for:

- Pure business logic
- React component testing
- Critical-path E2E
- CI enforcement

It prioritizes:

- Real browser behavior (no jsdom)
- Observable outcomes over implementation details
- High ROI first (logic → components → E2E)
- Low flakiness
- CI-backed regression safety

The goal is not 100% coverage.
The goal is **stable, meaningful protection of business-critical behavior**.

# Inspiration

This approach aligns with the principles demonstrated by:

### Chris Ferdinandi
- https://github.com/cferdinandi/tdd
- “You’re Doing JS Testing Wrong”
- His TDD repo walkthrough (the disclosure demo)

Key idea: test behavior users can observe — not internal mechanics.

### Playwright Official Guidance
- https://playwright.dev/docs/intro
- “Playwright Component Testing”
- “Testing Web Apps with Playwright”
- “Isolation and Fixtures in Playwright”

### Kent C. Dodds
- https://kentcdodds.com/blog/testing-implementation-details
- “Testing Implementation Details (Why Not To)”
- “Write Tests. Not Too Many. Mostly Integration.”

Core alignment:

> Test behavior. Avoid implementation coupling.
> Treat the browser as the public API.

# Core Testing Philosophy

### Test:

- Visible UI
- ARIA roles + attributes
- User-triggered state changes
- Navigation outcomes
- Business rules

### Do NOT test:

- Internal state
- Private functions directly (unless pure logic)
- Implementation details
- CSS selectors (unless unavoidable)

If a user cannot observe it, it likely does not belong in a behavioral test.

# Phase 0 — Baseline Assessment (1–2 Days)

## Objective

Define what must be protected first.

## Checklist

- [ ] Identify critical flows (auth, payments, creation flows)
- [ ] Identify high-risk modules (complex logic, bug-prone)
- [ ] Identify external boundaries (APIs, storage, 3rd-party)
- [ ] Categorize:

	* Pure logic
	* UI components
	* Feature flows
	* Full journeys
- [ ] Define business-critical failure cases

## Output

- Prioritized test target list
- Agreed initial scope
	(Do not attempt full coverage immediately)

# Phase 1 — Foundation Setup (Day 2–3)

## Install

```bash
npm install -D @playwright/test
npx playwright install
```

If React component testing:

```bash
npm install -D @playwright/experimental-ct-react
```

## Minimal Config

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
	testDir: './src',
	retries: process.env.CI ? 2 : 0,
	use: {
		trace: 'on-first-retry',
	},
});
```

## CI Requirements (Mandatory)

- [ ] Run on every PR
- [ ] Retries enabled in CI only
- [ ] Trace artifacts stored on failure
- [ ] Build fails on test failure

No CI = no regression protection.

# Phase 2 — Start with High-ROI Tests

Do **not** start with large E2E.

Begin with the lowest reliable layer.

## Layer 1 — Pure Logic Tests (Fastest ROI)

### Test:

- Utility functions
- Business rules
- Validation logic
- Data transformations
- Config builders

### Why:

- Deterministic
- No flakiness
- Immediate regression protection

### Example

```ts
import { test, expect } from '@playwright/test';
import { calculateTotal } from './pricing';

test('adds tax correctly', () => {
	expect(calculateTotal(100, 0.2)).toBe(120);
});

test('returns zero when base is zero', () => {
	expect(calculateTotal(0, 0.2)).toBe(0);
});
```

Goal: protect core business rules first.

## Layer 2 — Component Tests (React)

Render in real browser via Playwright CT.

### Test:

- Disabled/enabled states
- Conditional rendering
- Validation feedback
- ARIA behavior
- Keyboard interaction

### Rules

- Prefer `getByRole`
- One behavior per test
- Assert observable outcomes only
- Avoid brittle selectors

### Example

```ts
test('submit enables when form becomes valid', async ({ mount }) => {
	const component = await mount(<LoginForm />);

	const email = component.getByLabel('Email');
	const password = component.getByLabel('Password');
	const submit = component.getByRole('button', { name: 'Submit' });

	await expect(submit).toBeDisabled();

	await email.fill('user@test.com');
	await password.fill('password123');

	await expect(submit).toBeEnabled();
});
```

Goal: lock UI behavior without full app complexity.

# Phase 3 — Critical Path E2E (Minimal, Strategic)

Only after lower layers exist.

## Select 3–5 journeys:

- [ ] Login
- [ ] Core workflow
- [ ] Payment / submission
- [ ] Logout
- [ ] One high-risk edge case

## Hard Rules

- ❌ No `waitForTimeout`
- ❌ No CSS selectors by default
- ❌ No testing internal state
- ✅ Use role/label queries
- ✅ Mock external APIs at boundaries
- ✅ Assert user-visible outcomes

### Example

```ts
test('user logs in and sees dashboard', async ({ page }) => {
	await page.goto('/login');

	await page.getByLabel('Email').fill('user@test.com');
	await page.getByLabel('Password').fill('password');
	await page.getByRole('button', { name: 'Login' }).click();

	await expect(page).toHaveURL(/dashboard/);
	await expect(
		page.getByRole('heading', { name: 'Dashboard' })
	).toBeVisible();
});
```

Goal: protect revenue-critical flows — not every screen.

## Layer 2 — Component Tests (Vanilla JS)

If you’re testing a non-React components (Web Components) like:

```JavaScript
// disclosure.js

export function initDisclosure(button) {
		const content = document.getElementById(button.getAttribute("aria-controls"));
		button.addEventListener("click", () => {
				const expanded = button.getAttribute("aria-expanded") === "true";
				button.setAttribute("aria-expanded", String(!expanded));
				content.hidden = expanded;
		});
}
```

Test:

```JavaScript
import { test, expect } from "@playwright/test";

test("vanilla disclosure toggles", async ({ page }) => {
		await page.setContent(`
				<button aria-expanded="false" aria-controls="content">
						Toggle
				</button>
				<div id="content" hidden>
						Hello
				</div>
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

No special HTML file required. You can follow the same rules as above for React.

# Phase 4 — Stability Hardening

Audit for flakiness.

## Must Fix Immediately

- [ ] `waitForTimeout`
- [ ] Missing `await`
- [ ] Shared state between tests
- [ ] Uncontrolled real network calls
- [ ] Cross-test data leakage

## Replace With

- Auto-waiting assertions
- `page.route()` for API boundary mocks
- Isolated fixtures
- Deterministic test data

# Phase 5 — Structure & Organization

## Suggested Layout

```
src/
	utils/
		math.ts
		math.test.ts
	components/
		Button.tsx
		Button.test.tsx
tests/
	e2e/
		login.spec.ts
```

## Principles

- One behavior per test
- Small setup blocks
- Clear naming
- Reusable fixtures
- No duplicate coverage across layers

Bad:

```ts
test('button works', ...)
```

Good:

```ts
test('submit button enables when form is valid', ...)
```

# Phase 6 — Intentional Edge Expansion

After core stability.

Add coverage for:

- [ ] Error states
- [ ] Loading states
- [ ] Empty states
- [ ] Permission variations
- [ ] Keyboard-only navigation
- [ ] Boundary inputs

Expand based on risk — not coverage metrics.

# Phase 7 — CI & Performance Scaling

When suite grows:

- [ ] Enable parallelization
- [ ] Shard large test sets
- [ ] Run headless in CI
- [ ] Monitor flakiness weekly
- [ ] Fail build on repeated regressions

# Recommended Execution Order

1. Define critical flows
2. Add 10–20 high-value logic tests
3. Add 5–10 component tests
4. Add 3–5 E2E flows
5. Integrate into CI
6. Eliminate flakiness
7. Expand intentionally

# What NOT To Do

- ❌ Start with 100 E2E tests
- ❌ Chase 100% coverage
- ❌ Test implementation details
- ❌ Duplicate behavior across layers
- ❌ Skip CI enforcement

# Realistic 2-Week Milestone

By end of week 2:

- [ ] CI running Playwright
- [ ] Core logic protected
- [ ] Primary workflow covered
- [ ] No flaky tests
- [ ] Clear repository structure

Not maximum coverage.
Stable, business-aligned protection.
