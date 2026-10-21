# Generated Validation Artifacts

Applies regardless of stack — this is about how to keep a committed validation artifact (a JSON
Schema, a set of generated types, anything that describes a contract) in sync with its source of
truth, not about any particular testing framework or schema library.

## Generate from the source of truth, don't hand-write

When the thing a test validates against (a response shape, a contract) is already described
somewhere authoritative and live — an API's own OpenAPI document, a database schema, a protobuf
definition, anything the system itself exposes — generate the validation artifact from that
source instead of hand-writing and hand-maintaining a parallel description of the same shape.
Hand-written copies drift from reality silently; a generator reading the live source can't.

## The committed artifact and the generated one must stay physically separate

A generator writes its output somewhere scratch (gitignored, reproducible on demand — the
equivalent of a `dist/` directory). The artifact actually used by tests is a **separate,
committed file**. Never let the generator write directly to the committed file as part of a
check — see the next section for why.

```
schemas/todo.schema.ts            ← committed, what tests actually import, stable until
                                     someone deliberately updates it
schemas/generated/todo.schema.ts  ← gitignored, reproducible, exists only to diff against
                                     the committed file
```

Only generate the artifacts actually adopted (imported by a committed file), not everything the
source happens to expose — a generator that dumps every entity a live API defines produces a lot
of output nobody asked for and nothing imports.

## A drift check must only ever compare, never write

A CI step that verifies the committed artifact still matches the live source must regenerate
into the scratch location and diff — it must never overwrite the committed file before
comparing. If it did, the live source would end up compared against itself, which always
"passes" by construction and defeats the entire point of the check. Concretely, this also means
a drift check doesn't need to run the actual test suite — it only needs the diff.

```
check-drift:  generate → diff against committed → fail with a readable diff if they differ
update:       generate → overwrite committed → re-verify (e.g. run the dependent tests) →
              human reviews and commits
```

The "update" command is a separate, local/manual-only tool — never run it in CI for the same
reason the drift check must never write: running it there would make every drift check trivially
pass.

## Why this two-command split matters

Schema validation that only checks for **required** fields being present typically can't catch a
source adding a new field — most schema validators accept unknown extra properties by default, so
an additive change to the source passes every existing test silently. A dedicated drift check
(comparing the generated description against the committed one, not real response data) is what
actually catches that: it compares what the source *documents* now against what was last
approved, independently of whether any individual test happens to exercise the new field.
