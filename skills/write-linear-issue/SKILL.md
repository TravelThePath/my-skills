---
name: write-linear-issue
description: Use when creating, writing, drafting, opening, filing, or rewriting a Linear issue or ticket for engineering scope work — feature, refactor, schema change, migration, or infra task. Not for bugs, incidents, or spikes (those use different templates). Also use when an existing issue's What to build has drifted into implementation detail.
---

# Linear Issue — What to build + AC

An engineering issue is an _intent contract_, not a merge contract. It tells the implementer **what** to build and how to recognize it's done. **Why** (motivation, stakeholder, urgency) lives in the Linear project description, not the issue. **How** (file paths, code syntax) lives in the PR.

The issue body has exactly two sections: **What to build** and **Acceptance Criteria**. Both are mandatory.

## Title

Use `<Service Name> | <action statement>` (space-pipe-space). Write the service name with the casing it actually uses in its repo/docs — `Voyager`, `Ranger API`, `Document Service`, `Atlas`, `Jasper API`. When a change spans multiple services, pick the one that owns the primary entity or where the controlling change lands; cross-service follow-ups become separate issues with their own prefixes.

| ✓ Good  | ✗ Bad  |
| ------- | ------ |
| `Document Service \| Reject AU tax statement batch when the period is not an Australian financial year` | `Reject AU tax statement batch when …` (missing prefix) |
| `Ranger API \| Add include_archived flag to listAccounts`    | `document-service: Reject …` (wrong separator and casing) |

## What to build

Describe **what** the issue produces — entities, fields, surfaces, behaviors — in business language. Do not write **how** (file paths or code syntax: DDL, SQL, proto, function or field declarations) and do not write **why** (motivation, stakeholder ask, urgency).

Prose, bullets, or both — pick whatever reads clearest for the change. No required sub-headings; if the change touches enough surfaces that ad-hoc headings would help readability, add them, but don't add a heading just to fill space.

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

List the **observable end-state** that must be true once the work is done — from the caller's, user's, or database's point of view.

Not a test plan: no `Run`, `Verify`, `Check`.

AC may overlap with What to build, but must restate from an observable angle. If What to build says "exposes `bankCode`", AC says "querying `AUDBankAccountBankAccountLocationDetails` on an existing AUD bank account returns the persisted `bankCode` value."

## Process

1. **Classify.** Bug / incident retro / spike / product scoping → use the matching template, not this one.
2. **Draft** Title + What to build + AC. When you write any entity / API / page / type name, grep the repo and use the real name from code. For anything you can't verify, mark `[CONFIRM: <question>]` inline — never silently guess.
3. **Resolve & deliver.** Clear every `[CONFIRM:]` with the user. If they ask you to create the issue in Linear, call `save_issue` with `state=Todo` and `labels=["AI"]`. The `AI` label must already exist in the workspace; if `save_issue` fails because the label is missing, ask the user to create it once.

Story-point estimation is opt-in. Run it only when the user explicitly asks or when `save_issue` is invoked with an `estimate` parameter. The framework lives in [`story-point-estimation.md`](./story-point-estimation.md).

## Pre-delivery checklist

- [ ] **Title** uses `<Service Name> | <action>`; service name matches the casing the repo/docs use.
- [ ] **What to build** says only **what**, never **how** or **why**. Backticked identifiers are OK; file paths and code syntax are not.
- [ ] Every entity / API / page / type name in What to build is grep-verified in the repo or explicitly flagged as new.
- [ ] **AC** describes observable end-state from the caller / user / DB perspective — no `Run`, `Verify`, `Check`.
- [ ] No `[CONFIRM:` remains in the draft.

## Worked example — Expose AUD bank fields on Ranger API

**Title:** `Ranger API | Expose bank_code and direct_entry_user_id on AUD bank accounts`

### ✗ Bad — file paths, motivation, code syntax, test-plan AC

```markdown
## What to build

Operations need BSB codes so they can reconcile direct debit batches before quarter-end. Add a `BankCode string` field and a `DirectEntryUserID string` field to the AUD struct in `ranger-api/internal/graph/resolver/bank_account.go`, then run `ALTER TABLE bank_accounts ADD COLUMN bank_code VARCHAR(6)` in investment-service, and add `bool bank_code = 12` to the proto.

## Acceptance Criteria

- Run a GraphQL query that fetches `bankCode`; verify the response is not null.
- Check that the new column exists in production.
```

What went wrong:

- Opens with motivation ("Operations need BSB codes so they can reconcile…"). Motivation belongs in the Linear project, not the issue.
- Names a file path (`ranger-api/internal/...`), inline DDL (`ALTER TABLE …`), proto syntax (`bool bank_code = 12`), and Go field types (`BankCode string`) — all PR concerns.
- AC uses imperative `Run` / `Verify` / `Check` — that's a test plan, not an end-state.

### ✓ Good — code anchors, observable AC, no motivation, no code syntax

```markdown
## What to build

Ranger API's admin gRPC client surfaces `bank_code` and `direct_entry_user_id` on the AUD branch of `BankAccountLocationDetails`. Ranger API GraphQL exposes `bankCode` and `directEntryUserId` on the existing shared `AUDBankAccountBankAccountLocationDetails` type and its input counterpart `AUDBankAccountInput`. Both fields are optional; non-AUD branches are unaffected.

## Acceptance Criteria

- Querying `AUDBankAccountBankAccountLocationDetails` on an existing AUD bank account returns the persisted `bankCode` and `directEntryUserId` values, or `null` when not yet set.
- Submitting `AUDBankAccountInput` with `bankCode` and `directEntryUserId` persists both fields, and a subsequent query returns the submitted values.
- Non-AUD `BankAccountLocationDetails` variants are unchanged in both gRPC and GraphQL schemas.
```

Why this works:

- Names every identifier in the casing each surface uses (`bank_code` for gRPC, `bankCode` for GraphQL) and anchors them to existing types (`BankAccountLocationDetails`, `AUDBankAccountInput`) — the implementer goes straight to the right files without the issue prescribing them.
- No motivation, no file paths, no DDL, no proto syntax — those live in the Linear project and the PR respectively.
- AC restates the contract from the caller's vantage point ("Querying … returns …", "Submitting … persists …") instead of mirroring What to build or listing test steps.
