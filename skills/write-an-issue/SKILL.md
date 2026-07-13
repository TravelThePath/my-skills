---
name: write-an-issue
description: Use when writing, drafting, filing, or rewriting a Linear issue or ticket — a feature, fix, update, or one-off task — or when an issue's scope, structure, deliverable, or definition of done is unclear.
---

# Linear Issue: a self-contained build spec

An issue is a **self-contained build spec**: it names the deliverable, bounds it, and says when it's done. Not *how* to build it (that's the PR / the code), not the business case (that's the project description). Three properties make one good:

- **Self-contained** — it reads on its own, with no pointers to planning docs, decision registers, or decision codes (`D7`, `D12`…). Inline the *outcome* of a decision in plain words; background carries just enough *why* to stand alone, without reproducing the project's business case.
- **Deliverable explicit** — the reader knows the concrete thing that will exist when it's done.
- **Bounded & self-consistent** — it owns exactly one slice; adjacent layers and cross-cutting concerns are named as out of scope, not silently absorbed.

## Skeleton

```
Title: <Layer / Service> | <the deliverable, in one line>

1. Background — why this is needed (1–2 sentences, conclusion first, no decision codes).
2. Deliverable — the concrete thing that will exist (for a fix, the behaviour that will hold):
     backend   → the artifact: Ent table + migration, store method, RPC, GraphQL query/mutation…
     frontend  → the user-facing surface: a page, settings section, toggle (the component /
                 route name is a PR detail)
     fix       → the corrected behaviour, as the rule that will hold ("amounts round half-up")
     spike     → a written recommendation and the decision it unblocks
     other     → the concrete output: a test suite, signed-off wording, a backfill
3. Behaviour & constraints — the rules the deliverable must satisfy
                 (tenant isolation, idempotent / unique, default empty, a content contract…).
4. Out of scope — the adjacent layers and cross-cutting concerns cut away
                 (feature flag, audit, upstream projects, neighbouring layers).
5. Done when — observable conditions that prove the constraints, each named once (don't restate
                 the rule). For a spike: the recommendation is written and accepted.

Dependencies: express as Linear relations (blocked-by / related), not prose. Prefer filing the
relation; only when you truly cannot (text-only, no Linear access) add one "Blocked by <id>" line
and flag it to the user.
```

**Title left side:** the owning service (`Message Service`), or the app for frontend (`Admin App`, `Web App`); for work with no single owner, the functional area (`Rollout`, `Local Dev`).

Use all five parts by default. Collapse to background + deliverable + done-when only when there is nothing to add beyond the deliverable — one slice, no cross-cutting concern to cut, and no constraint that isn't already obvious from the deliverable — otherwise keep all five. If one issue holds several independent slices, split it; a requester's step-by-step ("add X, then read it in Y") usually hides more than one slice.

## Name the deliverable, not the how

The most common failure is describing a *capability* ("a store that holds each tenant's block") instead of naming the *artifact* ("an Ent table + migration + tenant-scoped store methods"). **Name the artifact** — the kind of thing that will exist. That is the "what", and it is exactly what the reader needs; stop before the "how".

The test: name *which* artifact and what it is for — never its shape. A table, type, component, or single field *name* is a fine anchor; the full column list, field types, widths, nullability, and routes are not — those are the migration and the wiring, and they live in the PR. Use one or two identifiers as anchors, not an enumeration of the schema. This shape ban holds in every section: in Behaviour & constraints, name a field only to state a rule about it (opacity, uniqueness, a copied limit), never its type or width.

| The *what* — include it | The *how* — leave it for the PR |
|---|---|
| Ent schema / table, migration, store method | file paths, DDL / SQL, struct or proto declarations |
| RPC, GraphQL query / mutation, endpoint | code snippets, function bodies, codegen steps |
| A settings section, a toggle, a test suite | component names, framework wiring, commit / branch steps |
| Backticked identifiers as anchors (`ListAccounts`, `bankCode`) | step-by-step implementation order |

## Values and decisions

- Copy decided values **verbatim** from the source (spec, the user's words): widths, limits, field names, defaults, fallbacks. Don't paraphrase or re-derive from memory.
- Inline the **outcome** of a decision; never cite its code or its planning doc. `owned by message-service (D7)` becomes `owned by message-service`. If the source still lists a decision as open, it is not decided — ask the user, don't guess. Never revive a decision the source dropped.

## Out of scope earns its section

Cross-cutting concerns quietly absorbed by the wrong issue are how scope leaks and how committed requirements vanish. If a feature flag, audit logging, or an upstream dependency touches this work but *is not* this work, name it under Out of scope and let it live in its own issue.

## Example

```markdown
Message Service | Store each tenant's editable email-footer block

## Background
The system email footers are hardcoded and identical for every tenant, so any change is a
code change and a deploy. Making them tenant-editable needs somewhere to persist the block
each tenant authors. This issue creates that storage only — nothing reads or writes it yet.

## Deliverable
- A new Ent schema for the tenant footer block in message-service, with the generated code
  and a MySQL migration that creates the table.
- Tenant-scoped store methods: read one block, save (upsert) a block, clear a block.

## Behaviour & constraints
- One row per (tenant, variant); the three variants are Transactional, Access & Compliance,
  and Marketing.
- Every read and write is scoped to the calling tenant.
- Save is idempotent per (tenant, variant): re-saving overwrites in place via a unique
  constraint, never a second row.
- `content` is stored opaquely and round-trips verbatim; the store never parses its shape.
- No per-tenant seeding — an unset block reads as empty.

## Out of scope
- The API to read / edit blocks, the feature flag, and audit logging — each its own issue.
- The shape of the content — owned by the frontend content contract.

## Done when
- The migration creates the table with the (tenant, variant) unique index.
- MySQL integration tests prove: an unset block reads empty; save-then-read returns identical
  content; re-save overwrites with no duplicate row; clear reads empty; one tenant never sees
  another's block.
```

Every part earns its place: the title names the artifact; background says why in two sentences with no decision codes; the deliverable is a concrete table + methods; constraints are rules, not implementation (`content` and the variant set appear only to state opacity and cardinality — not as a column layout); out-of-scope cuts the flag / audit / API and hands the content shape to its owner; done-when states the tests that prove the constraints, not the constraints again.

## Common mistakes

| Mistake | Fix |
|---|---|
| "A store that holds…" (a capability) | Name the artifact: "an Ent table + migration + store methods" |
| `(D7)`, "per the Decision Register", milestone codes | Inline the decision's outcome in plain words |
| A schema / column dump called "anchors" | Name the artifact and its purpose; one or two identifiers, not its shape |
| "Depends on ABC-123" written in the body | Use a Linear blocked-by / related relation |
| No "done when", or a spike forced into "testable" checks | Add observable conditions; for a spike, "recommendation written and accepted" |
| A flag / audit / upstream concern folded into an unrelated issue | Name it under Out of scope; give it its own issue |
| Reviving a decision the source dropped | Use only what the current source decides; ask if unclear |

**Estimating?** See `story-point-estimation.md` in this folder.
