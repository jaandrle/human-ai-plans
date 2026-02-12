# Setting up Biome

This document outlines the steps to set up Biome for this project, with a focus on enforcing existing `.editorconfig` settings.

## 1. Installation

Install Biome as a dev dependency:

```bash
npm i -D --save-exact @biomejs/biome
```

…remove `editorconfig-checker` to avoid conflicts with Biome's formatter:
```terminal
npm uninstall -D editorconfig-checker
```
…remove (`reviewdog/action-eclint`) from `.github/workflows/pr.yml`.

## 2. Configuration

Create a `biome.json` file in the root of the project by running:

```bash
npx @biomejs/biome init
```

This will create a default `biome.json` file. The configuration below is based on Biome v2.3.4. If you have a different version installed, you may need to run `npx @biomejs/biome migrate --write` after creating the file.

To align with the project's `.editorconfig`, replace the content of `biome.json` with the following:
```json
{
	"$schema": "https://biomejs.dev/schemas/2.3.4/schema.json",
	"vcs": {
		"enabled": true,
		"clientKind": "git",
		"useIgnoreFile": true
	},
	"assist": { "actions": { "source": { "organizeImports": "on" } } },
	"formatter": {
		"enabled": true,
		"formatWithErrors": false,
		"indentStyle": "tab",
		"lineWidth": 120
	},
	"linter": {
		"enabled": true,
		"rules": {
			"recommended": true,
			"correctness": {
				"noConstructorReturn": "off"
			},
			"style": {
				"useTemplate": "info"
			}
		}
	},
	"files": {
		"includes": ["**/src/**/*.ts", "**/src/**/*.tsx", "**/bs/**/*.js", "!**/src/routeTree.gen.ts"]
	},
	"overrides": [
		{
			"includes": ["**/*.json"],
			"formatter": {
				"lineWidth": null
			}
		},
		{
			"includes": ["**/*.yml"],
			"formatter": {
				"indentStyle": "space",
				"indentWidth": 2
			}
		}
	]
}
```

### Notes on `.editorconfig` compatibility:

- **`max_line_length = unset`**: In the `.editorconfig`, some files have `max_line_length = unset`. There isn't a direct equivalent for this in Biome's `biome.json`. For `*.json` I've set it to 80, which is a common practice. For other files with `unset`, the default of 120 will be inherited.
- **`trim_trailing_whitespace = false` for Markdown**: The `.editorconfig` specifies not to trim trailing whitespace in Markdown files. Biome's formatter trims trailing whitespace by default, and there is no configuration option to disable this for specific file types. This means that running Biome's formatter will trim trailing whitespace from `.md` files.
- **`end_of_line = unset` for `gradlew.bat`**: Biome will use the default `lf` line ending. This should not be an issue.

## 3. Usage

You can run Biome from the command line to check for issues and format your code.

### Checking for issues

To check the entire project for formatting and linting errors, run:

```bash
npx @biomejs/biome check .
```

To apply safe fixes (including formatting), use the `--write` flag:

```bash
npx @biomejs/biome check --write .
```

Some fixes are considered "unsafe" and require explicit approval. To apply them, use the `--unsafe` flag along with `--write`:

```bash
npx @biomejs/biome check --write --unsafe .
```

### Formatting

To format the entire project, run:

```bash
npx @biomejs/biome format --write .
```

## 4. Integration with `bs`

To integrate Biome with the `bs` build system, you can modify the existing `lint.js` and `biome.js` scripts.

### Update `bs/dev/lint.js`

Modify `bs/dev/lint.js` to use Biome for linting, removing the `editorconfig-checker`.

```javascript
#!/usr/bin/env -S npx nodejsscript
import { describeFromReadme } from "../.common.js";
import { biome } from "./biome.js";

if ($.isMain(import.meta))
	$.api("", true)
		.option("--fix", "Apply fixes", false)
		.describe(describeFromReadme())
		.action(function main({ fix } = {}) {
			$.is_verbose = true;
			lintFE(fix);
			$.exit(0);
		})
		.parse();

export function lintFE(fix = false) {
	if (!$.is_verbose) echo.use("-R", "Linting…");
	try {
		s.run`npx tsc --noEmit`;
		biome("Linting", fix);
	} catch (e) {
		echo(e.toString());
		$.exit(e.exitCode || 1);
	}
}
```

Now you can run `./bs/dev/lint.js` to check for issues and `./bs/dev/lint.js --fix` to apply fixes.

### Create `bs/dev/biome.js`

Create a new file `bs/dev/biome.js` to handle code formatting/linting with Biome.

```javascript
#!/usr/bin/env -S npx nodejsscript

if ($.isMain(import.meta))
	$.api("[method]", true)
		.describe([
			"Formats/Lints the codebase using Biome.",
			"",
			"Methods:",
			"– Formatting (default)",
			"– Linting",
			"– All",
		])
		.option("--fix", "Apply fixes", false)
		.action(function main(method = "Formatting", { fix }) {
			$.is_verbose = true;
			biome(method, fix);
			$.exit(0);
		})
		.parse();

/**
 * @param {"Linting" | "Formatting" | "All"} method
 * @param {boolean} fix
 */
export function biome(method = "Formatting", fix = false) {
	const echoEnd = $.is_verbose ? () => ({}) : (ok = false) => echo(`${ok ? "✓" : "✗"} ${method}`);
	if (!$.is_verbose) echo.use("-R", method + " (`biome`)…");
	try {
		const m = method === "Linting" ? "lint" : method === "Formatting" ? "format" : "check";
		const level = method === "Linting" ? "--diagnostic-level=error" : "";
		const f = fix ? "--write" : "";
		s.run`npx @biomejs/biome ${m} ${f} --colors=force ${level}`;
		echoEnd(true);
	} catch (e) {
		echoEnd(false);
		echo(e.toString());
		$.exit(e.exitCode || 1);
	}
}
```

Make the script executable:
```bash
chmod u+x bs/dev/biome.js
```

The script is now more powerful. You can still run `./bs/dev/biome.js Formatting --fix` to format, but now just `./bs/dev/biome.js --fix` will also work.

## 5. GitHub Action for Formatting and Linting

To ensure consistent code style on pull requests, you can integrate Biome into your GitHub Actions workflow. This involves running both the linting and formatting scripts in your `.github/workflows/pr.yml` file.

Here's how you can add the steps:

```yaml
	- run: ./bs/dev/lint.js
	- name: Format code with Biome
		run: ./bs/dev/biome.js Formatting --fix
	- name: Commit formatted files
		uses: stefanzweifel/git-auto-commit-action@v5
		with:
			commit_message: ":tv: format code"
			branch: ${{ github.head_ref }}
```

This ensures that all code in pull requests is linted and formatted, adhering to the defined code style without any manual intervention.
