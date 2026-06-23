---
name: resolve-pr-comments
description: >
  Use when a PR has reviewer comments or threads to address, resolve, respond to, fix, or work through — from AI bots (CodeRabbit, Copilot, Cursor, etc.) or human reviewers. Covers both review-only passes and publish flows that commit fixes, post replies, and close GitHub threads. Do not use for general code review without open PR review threads.
---

# Resolve PR Comments

Work through unresolved PR review feedback. The core loop is **code-first**: for each thread, read the current code, reach your own verdict, then say whether the bot was right. The bot's claim is a hypothesis to verify, not a conclusion to defend.

## Prerequisites

- `gh` CLI installed and authenticated.
- Current branch has an open PR, or the user provides a PR URL/number.

## Platform Rules

- Use `gh` for all GitHub calls. Do not use GitHub MCP tools.
- Blocking choices: `AskUserQuestion` in Claude Code; plain text + stop elsewhere.
- Keep output compact: verdicts, blockers, previews, outcomes. Keep raw API output out of responses.
- An explicit `resolve` / `fix` / `publish` request authorizes the publish lane (see `publish.md`) once every item has a recorded verdict. A review-only request (`pr review`, `review comments`) does not — ask once before any GitHub write.

## Verdicts

One vocabulary for both your recommendation and the user's choice.

| Verdict | Meaning | Requires |
| --- | --- | --- |
| **Fix** | Change code in this PR, reply `Fixed in <commit>`, resolve. | A concrete current-code citation in `Problem`: a `file:line` + quote, or a grep/diff/test result confirming the issue exists now. |
| **Reply** | No code change. Post an explanation, resolve. Covers stale, already-handled, convention-mismatch, and noise. | A reason the bot's specific claim does not match current code. |
| **Defer** | Like Reply, but the concern deserves tracked follow-up — also create a follow-up issue/note. Use Reply when acknowledging without tracking. | The concern is real but belongs in separate work. |
| **Ask** | You cannot judge within this repo (cross-service contract, production/migration state, product tradeoff). Hand it to the user. | The blocker is about cross-file / ownership / process / product, not current code. |

**The one hard rule:** no concrete current-code citation in `Problem` → you haven't verified it → not a `Fix`; use `Reply`, `Defer`, or `Ask`. Never invent evidence. `Ask` is non-terminal: the user must convert it to Fix/Reply/Defer before publish.

**Reviewer signal** sets scrutiny and order, never the verdict. Humans: never auto-resolve; if unclear, `Reply` or `Ask`. CodeRabbit / Codex / Cursor: credible hypotheses — verify before `Fix`. Copilot and unknown bots: low signal — demand concrete evidence, review last. Signal never earns a `Fix`; evidence does.

## Step 1 — Fetch

Run from the target repo (keep the shell cwd there):

```bash
python3 <skill-dir>/scripts/fetch-comments.py
```

For explicit PRs: `--repo OWNER/REPO --pr 123` or `--url <pr-url>`. The script drops resolved threads and prior skill replies at the source, and checks the PR `head_sha` against local `git rev-parse HEAD` — on a mismatch, stop and ask the user to checkout/update the branch rather than analyze a stale checkout. If the script can't run, fetch unresolved threads, PR-level comments, and review bodies with `gh api graphql` directly. Output can be large (bots embed walkthroughs and base64 state) — read it with `jq`/grep; don't dump it into your response.

**Actionable** = unresolved inline threads (including security-scanner threads like CodeQL/Snyk that point at a `file:line` — they're findings, not gates), substantive review-body findings, and **nitpick-labelled comments** (treat as `normal` impact; split a review-body nitpick `<details>` block into one item per finding).

**Drop as non-actionable** (don't card, reply, or resolve): CI/coverage gate status comments, Linear/Jira linkbacks, bot walkthrough/summary comments, the content-free "review" envelope a bot posts alongside its inline threads, `@bot` trigger comments, and PR-author status updates (use the last only as a staleness signal).

**Outdated threads** (`is_outdated`, kept by the fetch script): not actionable — list each as a one-line summary so the user sees them, and in publish **resolve them directly without a reply** (the code they referenced is gone). This is the one exception to reply-before-resolve.

If nothing actionable remains, output `No review comments found` and stop.

## Step 2 — Verdict (code-first)

For each thread, before forming a verdict, gather:

1. The focused diff: `git diff $(gh pr view --json baseRefName -q .baseRefName)...HEAD -- <path>`
2. The function/method containing the flagged line, plus nearby conventions (`CLAUDE.md`/`AGENTS.md`, lints, peer code).
3. For PR-level comments: the PR description and changed-file summary.

Then judge **the current code**: does the issue exist here, right now? Assign a verdict and an impact:

- **`high`** — security, data loss, panic/crash on normal input, state-corrupting race, broken error handling, API-contract break, deploy/migration risk, resource leak.
- **`normal`** — everything else (edge cases, missing tests, readability, naming, style, nits).

Impact is your read of the code, not the bot's label. A bot-labelled "Critical" the code disproves is `normal` + `Reply`. Drop an item only when the code shows the issue cannot exist — and even then, `Reply` to say so. When unsure, include it: over-including is recoverable, dropping real feedback is invisible.

**Dedupe** only when comments hit the same file/topic, point at the same function/path, ask for the same action category, and share ≥2 meaningful keywords. List all reviewers in the header (`[coderabbit/cursor]`), synthesize their angles in `Wants`, reply to each thread.

## Step 3 — Present & decide

Sort all items by impact: `high` → `ask` → `normal`, numbered globally (so `why 7` and cross-references work across both tiers). Present in **two tiers**, worked in order — every `high` one-by-one, then every `ask` one-by-one, then the `normal` pages:

- **`high` and `ask` — one at a time.** Show a single card, state the recommended verdict, and **stop for the user's decision.** Don't render the next card until the current one has a recorded verdict.
- **`normal` — paginated.** 5 per page, recommended verdict on each; `ok` / `ok all` accepts the defaults. `ok all` never touches a `high`/`ask` item.

Card template — two fields, do not add more:

```text
── PR OWNER/REPO#123 — high 1 of 2 ──────────

[high] #1 worker.go:42  [coderabbit]  → Fix
Problem:  worker.go:42's select has no ctx.Done() case (`select { case w := <-work: …; case r := <-results: … }`), so a timed-out request leaks the goroutine with nothing left to stop it.
Wants:    add a ctx.Done() branch to the select.

Decision? fix · reply · defer · ask · why  — I'll stop here until you choose.
```

`normal` items render the same card, 5 per page under a `── PR … — normal, page 1 of N ──` header, closing each page with the literal prompt `Reply: ok · ok all · fix N · reply N · defer N · ask N · why N (combinable: reply 3, defer 4)`.

Writing `Problem` and `Wants`: **plain, direct, natural language — the way you'd explain it out loud to a colleague.** Never restate the bot's review or reuse its phrasing; describe what you found yourself, as fact.

- **`Problem`** — what actually breaks in the current code, conclusion first, naming the exact function/field/line. For a `Fix`, this is where the concrete evidence lives (`file:line` + quote / grep / diff). For `Ask`, say what you can't judge and why it's out of repo scope.
- **`Wants`** — the concrete change being asked for, as a brief action.
- No bot echo, no hedging ("might", "could"), no vague nouns ("logic", "handling", "issue"). One logical line per field; let the terminal soft-wrap.

```text
Bad (echoes the bot): Problem: Consider adding a ctx.Done() case to prevent the potential goroutine leak the reviewer flagged.
Good (plain, your own): Problem: worker.go:42's select has no ctx.Done() branch, so a timed-out request leaves the goroutine running with nothing to stop it.
```

For PR-level comments, add a `Signals:` line before `Problem` (positive reactions from reviewer/author, later author follow-ups, whether the PR was updated after the comment). Show them; never auto-resolve on signals alone.

| Input | Behavior |
| --- | --- |
| `fix` / `reply` / `defer` / `ask` / `why` (no number) | Decide the `high`/`ask` item shown now; then advance to the next. |
| `ok` / `done` | Confirm this page's `normal` defaults. `high`/`ask` stay open. |
| `ok all` | Same, across all remaining pages. `high`/`ask` still stay open. |
| `fix N` / `reply N` / `defer N` / `ask N` | Set the verdict for item N. Ranges/lists ok: `fix 1,3`, `reply 1-4`. |
| `why N` | Explain the verdict (and its evidence) without changing it. |

## Step 4 — Publish

Once every item has a verdict (no open `high`/`ask`), read `publish.md` and follow the single publish flow: preview → authorization → commit+push if any Fix → reply+resolve.

## Step 5 — Learn (opt-in)

Triggered only when the user types `learn` after publish. Read `learn.md` and follow it.
