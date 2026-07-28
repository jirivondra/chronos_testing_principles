# chronos-testing-principles

Language-agnostic testing conventions (DRY/YAGNI/AAA, complexity limits, data-driven test patterns, …) and
Claude Code slash commands (`/add-regression-test`, `/add-smoke-test`), packaged as a **GitHub template repository**
so every new Chronos testing project starts with them already in place.

## Bootstrap a new project from this template

```
gh repo create <new-project-name> --template jirivondra/chronos-testing-principles --clone --public
cd <new-project-name>
```

Then:

1. Rename `# <Project Name>` at the top of `CLAUDE.md`.
2. If the project is TypeScript, add `@jirivondra/chronos-test-toolkit-ts` as a dependency (see that repo's README) and follow `docs/structure.md`.
3. If the project uses a different language, `docs/principles.md` and `docs/test-patterns.md` still apply conceptually — reimplement the equivalent of `@jirivondra/chronos-test-toolkit-ts` in that language's ecosystem, keeping the same shape (fluent response wrapper, facade over the HTTP/SOAP client, named test data, etc).
4. Adjust `docs/structure.md` for any layout differences (e.g. no SOAP API, different test runner).

## What's in here

- `docs/principles.md` — DRY, YAGNI, AAA, complexity ≤ 2, no hardcoded values, data-driven tests, and more.
- `docs/structure.md` — template directory layout assuming a toolkit-package + project-specific split.
- `docs/test-patterns.md` — concrete code patterns (making requests, asserting responses, data-driven `forEach` tests).
- `.claude/commands/add-regression-test.md`, `.claude/commands/add-smoke-test.md` — Claude Code slash commands that scaffold new test files following these conventions.
- `CLAUDE.md` — skeleton that `@`-imports the three docs above.
