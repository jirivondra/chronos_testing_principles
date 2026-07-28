# Git Workflow

Applies regardless of stack — branch naming and commit message convention are process rules, not
testing conventions.

## Branch naming

```
<type>/<description>
```

Allowed types (Conventional Commits): `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `ci`, `build`, `perf`, `style`.
`description` is a short kebab-case summary.

Examples: `feat/due-date-coverage`, `fix/soap-fault-message`, `refactor/cleanup-page-objects`.

`main`/`master`/`develop` are exempt from this check.

## Commit messages

Start with the same type as the branch, followed by a colon and a short description:

```
feat: add due date coverage to todo schema
fix: correct SOAP fault message assertion
```

A pre-commit hook auto-prepends the branch's type if you forget it — you don't have to type it
by hand every time, but the message still ends up in this format.

## Installing the hook

This repo ships a `prepare-commit-msg` hook in `.githooks/`. It is not installed automatically —
git only looks in `.githooks/` if you point it there once per clone:

```
git config core.hooksPath .githooks
```

After that, every commit on a branch that doesn't match `<type>/<description>` is rejected with
an explanation, and commit messages missing the type prefix get it added automatically.
