# chronos-testing-principles

Language-agnostic testing conventions (DRY/YAGNI/AAA, complexity limits, data-driven test patterns, page
object patterns, fixtures, …) and Claude Code slash commands, packaged as a **GitHub template repository**
so every new Chronos testing project starts with them already in place.

Two stacks are covered so far:

- **`docs/api-testing/`** — REST/SOAP testing with Mocha (DRY/YAGNI/AAA, complexity ≤ 2, facade pattern, data-driven `forEach` tests).
- **`docs/playwright-testing/`** — Playwright UI testing (page object inheritance/composition, fixtures, fluent `Promise<this>` API, data-driven tests, conditional skip via API state).

## Bootstrap a new project from this template

```
gh repo create <new-project-name> --template jirivondra/chronos-testing-principles --clone --public
cd <new-project-name>
```

Then:

1. Rename `# <Project Name>` at the top of `CLAUDE.md`.
2. Run `git config core.hooksPath .githooks` once — enforces `docs/git-workflow.md`'s branch naming convention locally.
3. Follow the instructions in `CLAUDE.md` to activate the docs block matching this project's stack (REST/SOAP or Playwright), and delete the other one.
4. **REST/SOAP (TypeScript) projects:** add `@jirivondra/chronos-test-toolkit-api-ts` as a dependency (see that repo's README) and follow `docs/api-testing/structure.md`. If the project uses a different language, `docs/api-testing/principles.md` and `docs/api-testing/test-patterns.md` still apply conceptually — reimplement the equivalent of the toolkit in that language's ecosystem, keeping the same shape (fluent response wrapper, facade over the HTTP client, named test data, etc).
5. **Playwright projects:** follow `docs/playwright-testing/*.md` for the page object inheritance chain, fixture, and test-data conventions. There's no published npm toolkit for Playwright yet — these are documented patterns to reimplement per project, not an installable package.
6. Copy `.claude/commands/add-regression-test.md`/`add-smoke-test.md` for REST/SOAP projects, or write Playwright-flavored equivalents (not included yet).

## What's in here

- `docs/api-testing/principles.md` — DRY, YAGNI, AAA, complexity ≤ 2, no hardcoded values, data-driven tests, and more.
- `docs/api-testing/structure.md` — template directory layout assuming a toolkit-package + project-specific split.
- `docs/api-testing/test-patterns.md` — concrete code patterns (making requests, asserting responses, data-driven `forEach` tests).
- `docs/playwright-testing/page-objects.md` — page object model, inheritance vs. composition, selector conventions.
- `docs/playwright-testing/fixtures.md` — fixture architecture (`mergeTests`, auth vs. no-auth fixtures, dependent fixtures, fixtures vs. hooks).
- `docs/playwright-testing/fluent-api.md` — the `Promise<this>` chaining convention for page object methods.
- `docs/playwright-testing/custom-actions.md` — when to use `expect` vs. `expect.soft` in page objects.
- `docs/playwright-testing/api-helper.md` — using an API client from within a page object for setup/teardown.
- `docs/playwright-testing/test-data.md` — static vs. dynamic (`faker`/`dayjs`) test data conventions.
- `docs/playwright-testing/test-structure.md` — spec file template, `test.step` usage, naming conventions.
- `docs/playwright-testing/data-driven-tests.md` — `forEach`-based data-driven test pattern.
- `docs/playwright-testing/conditional-skip.md` — skipping tests based on API-queried state.
- `.claude/commands/add-regression-test.md`, `.claude/commands/add-smoke-test.md` — Claude Code slash commands for REST/SOAP test files.
- `docs/git-workflow.md` — branch naming (`<type>/<description>`) and commit message convention, stack-agnostic.
- `.githooks/prepare-commit-msg` — enforces the above; install with `git config core.hooksPath .githooks`.
- `CLAUDE.md` — skeleton with both stacks' `@`-import blocks; pick one.
