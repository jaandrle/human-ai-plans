# Playwright testing
Below is a **complete, production-grade guide** to testing React (and vanilla JS) using **only Playwright**, aligned with the philosophy demonstrated in Chris Ferdinandi’s TDD demo and the disclosure example you referenced.

## Based on
To reinforce this architecture:

### Chris Ferdinandi
- https://github.com/cferdinandi/tdd
- “You’re Doing JS Testing Wrong”
- His TDD repo walkthrough (the disclosure demo)

### Playwright Official
- https://playwright.dev/docs/intro
- “Playwright Component Testing”
- “Testing Web Apps with Playwright”
- “Isolation and Fixtures in Playwright”

### Kent C. Dodds
- https://kentcdodds.com/blog/testing-implementation-details
- “Testing Implementation Details (Why Not To)”
- “Write Tests. Not Too Many. Mostly Integration.”


## Architectural Summary
This setup gives you:

- Real browser behavior
- Co-located tests
- Zero jsdom
- Single toolchain
- Behavioral confidence
- Safer refactors

It scales from small components to large apps cleanly.


# 1. Testing Philosophy (From the Talk & Demo)

The disclosure example in the repo follows these principles:

- Test what users can observe
- Interact via real DOM
- Avoid testing private functions
- Avoid mocking internal logic
- Use the browser as the source of truth

In short:

> Treat the browser as your public API.

That means:

- Click buttons
- Assert visible text
- Assert ARIA attributes
- Assert behavior changes

Not:

- Checking internal state
- Importing private functions
- Spying on implementation


# 2 Install Playwright (Single Tool Setup)

```Bash
npm init -y
npm install -D @playwright/test @playwright/experimental-ct-react
npx playwright install
```

We will use:

- `@playwright/test` → E2E + logic tests
- `@playwright/experimental-ct-react` → React component tests

No Vitest. No Jest. No jsdom.

# 3 Project Structure (Co-Located Tests)

Recommended:

```code
src/
	components/
		Disclosure/
			Disclosure.tsx
			Disclosure.spec.tsx
		Counter/
			Counter.tsx
			Counter.spec.tsx
	utils/
		formatDate.ts
		formatDate.spec.ts
playwright.config.ts
playwright-ct.config.ts
```

Tests live beside code.


# 4 Configurations

## A) playwright.config.ts (Logic + E2E)

```TypeScript
import { defineConfig } from "@playwright/test";

export default defineConfig({
	testDir: "./src",
	testMatch: /.*\.spec\.ts$/,
	use: {
		baseURL: "http://localhost:3000",
		headless: true,
	},
});
```

This runs:

- Pure logic tests
- Browser navigation tests


## B) playwright-ct.config.ts (React Components)

```TypeScript
import { defineConfig } from "@playwright/experimental-ct-react";

export default defineConfig({

	testDir: "./src",

	testMatch: /.*\.spec\.tsx$/,

});
```

This mounts React components in real Chromium.


# 5 Example: React Disclosure Component

### Component

```TypeScript
// src/components/Disclosure/Disclosure.tsx

import { useState } from "react";

export function Disclosure({ title, children }) {
	const [open, setOpen] = useState(false);

	return (
		<div>
			<button
				aria-expanded={open}
				onClick={() => setOpen(o => !o)}
			>
				{title}
			</button>
			{open && (
				<div role="region">
					{children}
				</div>
			)}
		</div>
	);
}
```

# 6 Component Test (Like Chris’s Disclosure Demo)

```TypeScript
// src/components/Disclosure/Disclosure.spec.tsx

import { test, expect } from "@playwright/experimental-ct-react";
import { Disclosure } from "./Disclosure";

test("disclosure toggles content", async ({ mount }) => {
	const component = await mount(
		<Disclosure title="More info">
			Hidden content
		</Disclosure>
	);

	const button = component.getByRole("button", { name: "More info" });
	await expect(button).toHaveAttribute("aria-expanded", "false");
	await button.click();
	await expect(button).toHaveAttribute("aria-expanded", "true");
	await expect(component.getByRole("region")).toContainText("Hidden content");
});
```

Notice:

- We check ARIA attributes
- We check visible content
- We do not inspect state

This mirrors the structure of the disclosure example you linked.


# 7 Vanilla JS Example (Matching Demo Style)

If you’re testing a non-React disclosure like in the repo:

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

No special HTML file required.


# 8 Pure Logic Test (Still Playwright)

```TypeScript
// formatDate.ts
export function formatDate(date: Date) {
	return date.toISOString().split("T")[0];
}
```

```TypeScript
// formatDate.spec.ts
import { test, expect } from "@playwright/test";
import { formatDate } from "./formatDate";
test("formats ISO date", () => {
	expect(formatDate(new Date("2024-01-01")))
		.toBe("2024-01-01");
});
```

Runs in Node.


# 9 Testing API Calls (Without Mocking Everything)
- Mock boundaries
- Not internal logic

```TypeScript
test("loads data", async ({ mount, page }) => {
	await page.route("/api/user", route =>
		route.fulfill({
			json: { name: "Chris" }
		})
	);

	const component = await mount(<UserProfile />);

	await expect(component).toContainText("Chris");
});
```

# 10 CI Setup (GitHub Actions)
For example:

`package.json`:
```JSON
{
	"scripts": {
		"test": "playwright test",
		"test:ct": "playwright test -c playwright-ct.config.ts"
	}
}
```

`.github/workflows/tests.yaml`
```YAML
name: Tests
on: [push]

jobs:
	test:
		runs-on: ubuntu-latest
		steps:
			- uses: actions/checkout@v4
			- uses: actions/setup-node@v4
				with:
					node-version: 20
			- run: npm ci
			- run: npx playwright install --with-deps
			- run: npm run test
			- run: npm run test:ct
```


# 11 When To Use Each Level

| **Test Type** | **Tool** | **Scope** |
| :-: | :-: | :-: |
| Pure logic | Playwright | Node only |
| React component | Playwright CT | Real browser, isolated |
| Feature | Playwright | Full navigation |
| Full app | Playwright | E2E |

Single ecosystem.
