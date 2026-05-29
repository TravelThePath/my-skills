---
name: write-linear-issue
description: Use when creating, writing, drafting, opening, filing, or rewriting a Linear issue or ticket for engineering scope work — feature, refactor, schema change, migration, or infra task. Not for bugs, incidents, or spikes (those use different templates). Also use when an existing issue's What to build has drifted into implementation detail.
---

# Linear Issue — What to build + AC

An engineering issue is an _intent contract_, not a merge contract. It tells the implementer **what** to build and how to recognize it's done. **Why** (motivation, stakeholder, urgency) lives in the Linear project description, not the issue. **How** (file paths, code syntax) lives in the PR.

The issue body has exactly two sections: **What to build** and **Acceptance Criteria**. Both are mandatory. No `## Context` / `## Background` / motivation section — that's **why**, and it lives in the project. Dependencies are expressed as Linear blocked-by relations, not as a `Blocked by` list or dependency prose in the body.

## Ground every claim in the source of truth

The issue **restates** its requirement so it reads self-contained — never "see Section 3." But every behavioral claim must come from the source of truth, not from memory. For project work the source is the Linear project description — its `Decision Register`, `Product Acceptance Criteria`, and the relevant key-flow section when present. When the description is thin or lacks those sections, fall back to the most authoritative statement it does contain; for a one-off issue with no spec at all, the user's stated requirement is the source.

Re-derivation from a half-remembered spec is how issues drift. Guard against it:

- **Re-read before drafting.** Open the source sections this issue implements and read them verbatim — do not reconstruct decided behavior from earlier context.
- **Carry decided values verbatim.** Exact widths, limits, field names, strings, fallbacks — copy them, don't paraphrase. Paraphrase is where drift enters.
- **Cite the decision inline.** Name the source decision as provenance while restating the requirement: "… per the project's `Decision Register` row on the reconciliation reference." This keeps the issue self-contained *and* traceable — and naming the decision forces you to actually find it.
- **Never revive a dropped decision.** If the source marks an option dropped / superseded / out of scope, describe only what replaced it. A value the spec explicitly retired must never reappear.
- **`[CONFIRM:]` covers behavior, not just names.** Any behavioral claim you can't trace to a source line gets `[CONFIRM: <question>]`, exactly like an unverifiable identifier.

## Title

Use `<Service Name> | <action statement>` (space-pipe-space). Write the service name with the casing it actually uses in its repo/docs — `Voyager`, `Ranger API`, `Document Service`, `Atlas`, `Jasper API`. When a change spans multiple services, pick the one that owns the primary entity or where the controlling change lands; cross-service follow-ups become separate issues with their own prefixes.

| ✓ Good  | ✗ Bad  |
| ------- | ------ |
| `Document Service \| Reject AU tax statement batch when the period is not an Australian financial year` | `Reject AU tax statement batch when …` (missing prefix) |
| `Ranger API \| Add include_archived flag to listAccounts`    | `document-service: Reject …` (wrong separator and casing) |

## What to build

Describe **what** the issue produces — entities, fields, surfaces, behaviors — in business language. Do not write **how** (file paths or code syntax: DDL, SQL, proto, function or field declarations) and do not write **why** (motivation, stakeholder ask, urgency).

Prose, bullets, or both — pick whatever reads clearest for the change. Default to bullets when the change touches several surfaces; they scan better than dense prose. No required sub-headings; add ad-hoc headings only when they earn their place.

Keep it tight — concise, not thin:

- **One new fact per line.** Every sentence or bullet must add something the previous lines and the anchors don't already carry. Cut the rest.
- **Restate the requirement, drop the noise.** Keep behavior, fields, surfaces, decided values, anchors. Drop motivation, PR-level how, invented detail, and filler ("can start immediately", "None —").
- **Don't echo the AC.** What to build says what to produce; AC says how you'd observe it — don't write the same sentence twice.
- **No length cap** — but if What to build balloons, it's usually a tell: you're restating *why* / *how*, duplicating the AC, or covering more than one service / slice (split it).

### Code anchors are allowed; code snippets are not

The rule is "no file paths, no code snippets," **not** "no identifiers." Naming the real entity, type, field, RPC, or page — in the casing each surface uses — anchors the implementer to the right place in the code. The boundary:

| Writing   | Allowed? | Example   |
| ------- | -------- | ----------- |
| Backticked identifier from existing code | ✓  | `BankAccountLocationDetails`, `ListAccounts` |
| Backticked identifier being introduced | ✓  | `bankCode`, `directEntryUserId`  |
| Qualified identifier path  | ✓        | `BankAccountLocationDetails.bank_code`, `ListAccountsRequest.include_archived`  |
| Multiple surfaces, each in its own casing  | ✓  | gRPC `bank_code`, GraphQL `bankCode`   |
| Behavior / contract sentence (no identifier)  | ✓  | "Default excludes archived rows", "Non-admin callers receive `PermissionDenied`"  |
| Current-state fact (existing gap, not motivation)  | ✓  | "Ranger API's admin gRPC client does not currently expose `bank_code`"  |
| File path  | ✗        | `internal/bank/store.go`   |
| Code syntax — DDL, SQL, proto, function or field declarations (inline or block) | ✗ | `ALTER TABLE accounts ADD COLUMN …`, `bool include_archived = 5;`, `BankCode string`, `func ListAccounts(ctx, *Request) (*Response, error)` |
| Motivation / stakeholder / urgency | ✗ | "Operations need BSB codes to reconcile statements"   |

## Acceptance Criteria

List the **observable end-state** that must be true once the work is done — from the caller's, user's, or database's point of view. For project work, select the `Product Acceptance Criteria` rows this issue covers and restate each from an observable angle — don't invent criteria the source doesn't have.

Not a test plan: no `Run`, `Verify`, `Check`.

AC may overlap with What to build, but must restate from an observable angle, not repeat its wording. If What to build says "exposes `bankCode`", AC says "querying `AUDBankAccountBankAccountLocationDetails` on an existing AUD bank account returns the persisted `bankCode` value."

## Process

1. **Classify.** Bug / incident retro / spike / product scoping → use the matching template, not this one.
2. **Ground, then draft.** Re-read the source verbatim per the **Ground every claim** rules above, then draft Title + What to build + AC. Grep the repo for every entity / API / page / type name and use the real name from code. Mark anything you can't trace to the source or the code — a value *or* a behavior — `[CONFIRM: <question>]` inline; never silently guess.
3. **Resolve & deliver.** Clear every `[CONFIRM:]` with the user before delivering. (When an orchestrating skill drives drafting in batches, surface unresolved `[CONFIRM:]` in that flow's review checkpoint rather than blocking each issue.) If they ask you to create the issue in Linear, use the Linear interface available in the workspace (CLI command, MCP, or equivalent) to create it with `state=Todo`, the estimate, and the `AI` label. The `AI` label must already exist in the workspace; if creation fails because the label is missing, ask the user to create it once.

Story-point estimation runs by default — produce an estimate for every issue using the framework in [`story-point-estimation.md`](./story-point-estimation.md). Skip it only when the user opts out, or for a pure parent / `Story` container scored from its children. An orchestrating skill (e.g. `project-planning`) may set this choice once for a whole batch.

## Pre-delivery checklist

- [ ] **Title** uses `<Service Name> | <action>`; service name matches the casing the repo/docs use.
- [ ] **What to build** says only **what**, never **how** or **why**. Backticked identifiers are OK; file paths and code syntax are not.
- [ ] Every behavioral claim traces to a source line (`Decision Register` / `Product Acceptance Criteria` / key-flow) or the user's stated requirement; decided values are carried verbatim and the source decision is cited inline.
- [ ] No decision the source marks dropped / superseded / out of scope appears in the draft.
- [ ] Self-contained: no "see Section X" pointer standing in for content; no `## Context` / motivation section; no `Blocked by` / dependency list in the body.
- [ ] Every entity / API / page / type name in What to build is grep-verified in the repo or explicitly flagged as new.
- [ ] **AC** describes observable end-state from the caller / user / DB perspective — no `Run`, `Verify`, `Check` — and doesn't repeat What to build's wording.
- [ ] No `[CONFIRM:` remains in the draft.

## Worked example — ABA detail record Lodgement Reference

This field has a value decided in the project's `Decision Register` *and* a wrong version the register explicitly dropped — the cleanest test of grounding vs improvisation. `APP-22835` is a live in-repo issue with the same self-contained, provenance-cited shape.

**Title:** `Investment Service | Set ABA Lodgement Reference from the payee IE payment reference`

### ✗ Bad — revives a dropped decision, conflates limits, invents behavior

```markdown
## What to build

The ABA detail record Lodgement Reference is the last 6 digits of the IE ID zero-padded, followed by the description truncated to 12 characters, 18 characters total. If two IE IDs share the same last 6 digits, emit a warn-level log.

## Acceptance Criteria

- Run a payment pack and verify the Lodgement Reference is populated.
```

What went wrong:

- Revives the `{IE_ID}{description}` composite the `Decision Register` **explicitly drops** — rebuilt from memory instead of re-reading the decided value.
- `truncated to 12` is the Other Payment File `Code` rule; the ABA field is 18 — two decided values conflated.
- The last-6-collision warn log is nowhere in the source — invented.
- AC is a test step (`Run … verify`) and restates What to build instead of giving an observable outcome.

### ✓ Good — verbatim decided value, provenance cited inline, anchored to code

```markdown
## What to build

Populate the ABA detail record **Lodgement Reference** (18 chars) per the project's `Decision Register` row on the reconciliation reference: primary value is the payee Investing Entity's `paymentReference`, truncated to 18 if longer; when unset, fall back to the Investing Entity ID per that same register row. Left-aligned, space-padded to 18.

## Acceptance Criteria

- A payee whose Investing Entity has `paymentReference` set produces a Lodgement Reference equal to that value, truncated to 18 and space-padded to 18.
- A payee whose Investing Entity has no `paymentReference` falls back to the Investing Entity ID per the register row, space-padded to 18.
```

Why this works:

- Carries the decided values verbatim (18-char width, primary `paymentReference`, register-defined fallback) — nothing paraphrased, nothing invented; the exact fallback is whatever the register currently says, re-read at draft time, not a value memorised from an earlier version.
- Cites the source decision inline (`Decision Register` row) — traceable, yet self-contained: the reader never opens the spec.
- Anchors to the real field `paymentReference` and the Investing Entity ID from code.
- AC restates each branch as an observable outcome, not a test step, and doesn't echo What to build's wording.
