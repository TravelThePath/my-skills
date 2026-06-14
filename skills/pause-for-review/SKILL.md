---
name: pause-for-review
description: >
  Use before crossing a decision boundary: a direction choice (multiple valid implementations), a contract commitment (public API, database schema, proto/GraphQL field, migration, auth/tenant boundary, event format), a costly-to-reverse action (delete data, force push, deploy to shared env), after a discovery that invalidates the plan, or after completing an independently reviewable slice. Not for routine work inside the agreed slice. Never pre-emptively, never based on size.
---

# Pause for Review

## The principle

A pause is a costly signal. It is worth something *only* because it is rare. Pause unnecessarily and you teach the user to reply "yes" without looking; once that habit forms, the user rubber-stamps the real pause too. An unnecessary pause costs more than the turn it wastes: it spends down the protective value of every future pause.

This yields one test, and every boundary below follows from it:

> **Pause only where human judgment is load-bearing and being wrong is costly or hard to reverse.**

The boundaries proxy that test; they catch the costly cases and no more. So once you are confident you are inside an agreed slice, extra caution buys nothing and costs signal: the boundaries already cover what matters. When you hit a case the list doesn't name, decide by the test, not by the list, and not by size.

Two ways to fail:

- **Over-pause**: the common failure for an eager agent. Its damage compounds: it degrades every later pause. Guard against this hardest.
- **Under-pause**: commit a contract, delete data, or push past a dead plan with no human input. Rarer, but the boundaries exist to catch it.

## Continue without pausing

Stay inside the agreed slice. "Same slice" means the same behaviour under the same contract. If no slice was agreed, treat the user's last instruction as the slice. The user can interrupt at any time, so you never need to pause "to be safe."

- Happy path → error cases for the same behaviour
- More tests or edge cases under the same contract
- Fixing failures the current verification command uncovers
- Local refactor inside the slice (extract a helper, rename a private symbol)
- Mechanical cleanup: formatting, imports, narrow renames
- Running more verification

A new helper module, a new public function, or a code path that wasn't in scope is a *new* slice. See the boundaries below.

## Stop and surface

Pause when about to cross one of these. Cross only after the user acknowledges.

1. **Direction** (multiple valid paths exist): choosing between materially different strategies, naming an abstraction other code will depend on, resolving an ambiguous spec line, deciding what "done" means when the spec is silent.
2. **Contract** (committing to something external): public API, GraphQL/REST schema, gRPC proto message, DB schema / migration / index, CLI flag / env var / config format, a permission / tenant / data-ownership boundary, an event format other services consume.
3. **Reversibility** (costly to undo): data migration with non-trivial backfill, deleting code / data / branches / files (`git reset --hard`, `rm -rf`), deploying to a shared env, force push, history rewrite, dependency downgrade.
4. **Discovery**: the plan turned out wrong (see below; this one is different).
5. **Slice complete** (an independently reviewable unit is done): a vertical slice works end to end, all tests for the agreed scope pass, or a new architectural path runs end to end for the first time.

## Discovery is a re-plan, not a request

The other four boundaries ask *"may I cross this line?"* Discovery is different: the map is wrong, so the current plan is already void. Do not ask permission to continue down a path you now know is broken. Stop, state what you found and why it breaks the plan, and propose the new plan. The user is choosing a direction, not approving a step.

## How to pause well

A pause is high-signal only if it carries your thinking forward. A pause that hands the decision back untouched (*"should I continue?"*) is itself noise: it makes the user reconstruct the context and do the work from scratch, which is what trains rubber-stamping. So when you pause:

- State the decision in one line.
- Give the real options: the actual forks, not yes/no.
- Recommend one, with the reason. You did the analysis; share the conclusion.
- Make the default action clear, so a one-word reply unblocks you.

Batch related decisions into a single pause instead of ping-ponging across several. While waiting, stop work that depends on the pending decision; you may continue independent work, but default to stopping to keep the review surface small.
