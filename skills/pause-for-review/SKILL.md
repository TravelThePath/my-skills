---
name: pause-for-review
description: >
    Establishes a slice-and-pause rhythm for substantial changes: break the work into independently reviewable pieces and pause at each boundary for the user to review before continuing. Use when carrying out a multi-step or multi-file change (feature, refactor, migration, multi-part fix), or before any hard-to-reverse or externally-visible action (API/schema/proto/migration change, deleting data, deploying, force-push). Skip for single-step edits, read-only exploration, or answering questions.
---

# Pause for Review

## The rhythm

Break work into **vertical slices** — coherent changes someone could review on their own. Slice by *function*, not by size or by layer.

Finish one slice → pause → let the user review → take the next. The slice boundary is the review point. That rhythm is the whole skill.

## When to pause

- **A slice is done** — a functionally-independent piece works end to end.
- **A genuine fork** — a decision the user should own: a direction with several valid answers, a commitment that's costly to walk back, or an external contract others depend on.

**Don't pause inside a slice.** Once you're executing an agreed slice, finish it — happy path, error cases, tests, cleanup, fixing what verification surfaces. The user can interrupt anytime, so you never pause "to be safe."

**When the plan breaks.** If you learn the plan is wrong, don't ask permission to keep walking a broken path. Stop, say what you found and why it voids the plan, and propose new slices. The user is choosing a direction, not approving a step.
