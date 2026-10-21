# Cross-Repo CI Validation

Applies regardless of stack — this is about CI architecture when two repositories depend on each
other, not about any particular test framework.

## The setup

A "producer" repo (an API, a service) and a "consumer" repo (a separate test suite that validates
it against a live instance) are common in this project family — the test repo isn't vendored
into the API repo, it's checked out by the API repo's own CI.

## The blind spot

The producer's CI typically checks out the consumer repo to validate changes against it — but a
plain `actions/checkout` with no `ref:` always pulls the consumer's **default branch**, never an
open PR branch there. That gives one direction of coverage for free: when the producer changes,
its own CI re-validates the consumer's current main against that change, catching contract
breaks introduced by the producer.

It does **not** give the other direction. A PR opened in the consumer repo — updating a committed
test fixture, a generated schema, anything that's supposed to match the producer's current live
behavior — has nothing automatically checking it before merge. The producer's CI only ever looks
at the consumer's main, never its open PRs; nothing in the consumer repo checks itself, unless it
has CI of its own.

```
producer CI:  checks out consumer's main  → validates it against the producer's own PR branch
                                              (coverage: producer PR → consumer main, only)

consumer's own PR: nothing checks it automatically, unless the consumer repo has its own CI
                    that checks out the producer to validate against
```

## The fix

The consumer repo needs its own CI, triggered on its own PR/push, that checks out the producer's
live main (or whatever instance is appropriate) to validate **its own branch** — independent of
whatever the producer's CI happens to do. The two pipelines end up checking each other from
opposite directions; neither one alone covers both repos' own PRs.

## Why this is easy to miss

Everything appears to work: the producer's CI is green, its own PRs get validated against the
consumer's current state. The gap only shows up as a problem once someone opens a PR in the
*consumer* repo that's actually wrong against the producer's current live behavior — and nothing
flags it until it's already merged, because nobody's CI was ever looking at that specific branch.
