# Taskfile Conventions

Applies regardless of stack — how a `Taskfile.yml` is structured is a tooling convention, not a
testing one.

## Referencing another task

Use the `task:` key, not a bare `task <name>` string:

```yaml
# Correct
cmds:
    - task: build
    - task: lint
```

```yaml
# Wrong — spawns a new `task` subprocess instead of letting Task call it directly
cmds:
    - task build
```

The `task:` key is Task's own mechanism for calling another task — it runs within the same
process, participates in the run's dependency graph, and shares variables with it. A bare
`task <name>` string just re-invokes the `task` CLI binary as a new OS process: it happens to
work, but loses all of that and is slower for no benefit.

## Deferred cleanup

Same rule for `defer:` — wrap the task reference in the object form, not a bare string:

```yaml
cmds:
    - task: wiremock-up
    - defer: { task: wiremock-down }
    - npx mocha --spec 'dist/test/**/*.spec.js'
```

## Plain shell commands

A command that isn't a reference to another task is just a bare string — no `cmd:` wrapper
needed:

```yaml
cmds:
    - rm -rf dist
    - npx tsc
```

Reach for the `cmd:` key only when that command needs to sit alongside another modifier
(`platforms:`, `silent:`, …) that the bare-string form can't carry:

```yaml
cmds:
    - cmd: echo "building..."
      silent: true
    - platforms: ['darwin']
      cmd: npm install @esbuild/darwin-arm64 --no-save
```

## Why this matters

A Taskfile that mixes both styles for the same kind of reference (`task: lint` in one task,
`task lint-fix` in another) reads as if the two forms were interchangeable stylistic choices.
They aren't — one calls Task's own API, the other shells out to a new process — so picking one
and using it consistently keeps that distinction visible instead of accidental.
