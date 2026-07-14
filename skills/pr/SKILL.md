---
name: pr
description: >
  Use when the user asks to create, update, synchronize, merge, close, reopen, mark draft, or mark ready a pull request, or when PR title, body, branch, merge, or authorization conventions matter.
---

# PR Conventions

PR workflow conventions for Caruso/JasperLabs repos: authorization boundaries, conventions per lifecycle stage, and non-obvious gotchas. Assume you know normal git/gh mechanics.

## Authorization

A user request to create, update, synchronize, or merge a PR is explicit confirmation for the scoped git/gh actions that request needs (commit, push, PR edit, or PR merge). *Scoped* means limited to the PR's own branch and the files relevant to the request.

A user request to close, reopen, mark draft, or mark ready a PR is explicit confirmation for that requested PR state change.

These always need a separate ask, even when they would accomplish the requested action faster:

- Force push.
- Pushing directly to `main` / `master`.
- Pushing to protected branches outside the PR branch.
- Staging unrelated files.
- Enabling auto-merge (off by default; see Merge).

## Conventions

### Branch

- Format: `app-XXXXX-*` or `*-APP-XXXXX-*` (where `*` is a wildcard).
- On PR create, if the branch doesn't match: proceed but warn once that Linear auto-link won't fire.
- Issue ID inference: args -> branch -> commit messages -> current session's explicit issue context. Do not infer from stale memory.

### Title & Body

- Title: `<ISSUE-ID> | <conventional commit subject>`, imperative, under 70 chars; omit issue prefix if no issue. `<ISSUE-ID>` is uppercase (e.g. `APP-21005`), even though the branch prefix is lowercase.
- Body: intent-focused, not detail-focused.
  - State the problem the PR solves, in 1-3 logical points.
  - Do NOT list files changed, manual test steps, or version numbers.
- Keep the body flat unless a `## Summary` heading aids reading for a multi-point change. No `## Test plan` section. Change details belong in commits and automated tooling, not in a body that drifts on follow-up pushes.

### Quality gate

The code a commit or a PR-updating push carries must pass the repo's formatter, linter, and static analysis. A PR must never introduce a new failure.

- Discover the repo's own commands — Makefile targets (`make format`, `make lint`), `.golangci.yml`, `pre-commit` config, `package.json` scripts, the CI workflow, `CLAUDE.md` — and run those. Do not guess or hand-roll a formatter.
- Run format first, then lint / vet / static analysis, then commit. Formatting rewrites files, so re-stage only the target files afterwards — never `git add -A`.
- You own failures in the files your change touches; fix them before committing. You need not fix pre-existing failures in untouched files, but never add new ones. If a required fix falls outside the change's scope, surface it rather than commit a dirty diff.
- If the repo defines no such tooling, say so and fall back to the language default (e.g. `gofmt`, `go vet`).

### Create

- Draft by default; create as ready (non-draft) only when the user explicitly says so (e.g. passes `ready` or asks for a non-draft PR).
- Commit only after the Quality gate passes.
- If a PR already exists for the current branch: report it and treat further work as update/sync, not a new PR.
- After creating a draft PR, post two separate review-trigger comments on it via `gh pr comment <pr> --body '<text>'`: one with body `@codex review`, another with body `BugBot run`. Two distinct comments, not a single combined one (each handle listens for its own standalone comment). Skip when the PR is created as ready.

### Update / Sync

- Regenerate and replace both PR title and body from the current branch state, even when an existing title is present. PR descriptions drift; treat each update as a fresh write. The regenerated title and body still follow the Title & Body conventions above. Do not paste the diff.
- Push updates only after the Quality gate passes.

### Merge

- Squash merge only.
- Subject: PR title as-is.
- Body: one why sentence + themed bullets (intent, not file-by-file).
- Preconditions: all CI checks pass, including non-required ones. Do not skip a failing non-required check without explicit user sign-off.
- Auto-merge: off by default; enable only on explicit user request (and confirm per Authorization).

### Close

- Never use `--delete-branch` when closing a PR.

## Gotchas

- Worktree merge cleanup: if CWD is inside a worktree, use `git worktree remove`; do not checkout base inside the worktree.
- `gh pr diff --stat` does not exist; use `gh pr view --json files`.
- `git branch -d` can fail after squash merge; use `-D` only when the PR is confirmed merged.
- CODEOWNERS auto-assigns reviewers, so do NOT pass `--reviewer` on create unless the user names a specific reviewer. Manual assignment can break round-robin distribution within the owning team.
