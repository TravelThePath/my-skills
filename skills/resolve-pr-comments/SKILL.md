---
name: resolve-pr-comments
description: >
  Use when a PR has reviewer comments or threads to address, resolve,
  respond to, fix, or work through — from AI bots (CodeRabbit, Copilot,
  Cursor, etc.) or human reviewers. Covers both review-only passes and
  publish flows that commit fixes, post replies, and close GitHub
  threads.
---

# Resolve PR Comments

Work through unresolved PR review feedback. The core loop is **code-first**: for each thread, read the current code, reach your own verdict, then say whether the bot was right. The bot's claim is a hypothesis to verify, not a conclusion to defend.

## Prerequisites

- `gh` CLI installed and authenticated.
- Current branch has an open PR, or the user provides a PR URL/number.

## Platform Rules

- Use `gh` for all GitHub calls. Do not use GitHub MCP tools.
- Blocking choices: `AskUserQuestion` in Claude Code; plain text + stop elsewhere.
- Keep output compact: show verdicts, blockers, previews, outcomes. Keep raw API output out of responses.
- An explicit `resolve` / `fix` / `publish` request authorizes the publish lane (see `publish.md`) once every item has a recorded verdict. A review-only request (`pr review`, `review comments`) does not — ask once before any GitHub write.

## Verdicts

One vocabulary for both your recommendation and the user's choice.

| Verdict | Meaning | Requires |
| --- | --- | --- |
| **Fix** | Change code in this PR, reply `Fixed in <commit>`, resolve. | Concrete evidence: a `file:line` + quote, or a grep/diff/test result confirming the issue in current code. |
| **Reply** | No code change. Post an explanation, resolve. Covers stale, already-handled, convention-mismatch, and noise. | Evidence the bot's specific claim does not match current code. |
| **Defer** | Like Reply, but the concern deserves tracked follow-up — also create a follow-up issue/note. Use Reply, not Defer, when acknowledging without tracking. | Evidence the concern is real but belongs in separate work. |
| **Ask** | You cannot judge within this repo (cross-service contract, production/migration state, product tradeoff). Hand it to the user. | Evidence field reads `none; bot's claim is about <cross-file \| ownership \| process \| product>`. |

**The one hard rule:** `Fix` requires concrete evidence. If the Evidence field would be empty or a paraphrase of the bot's text, you have not verified it — pick `Reply`, `Defer`, or `Ask`. Never invent evidence to justify a Fix. `Ask` is non-terminal: the user must convert it to Fix/Reply/Defer before publish.

**Reviewer signal** affects scrutiny and order, never the verdict. Human comments: never auto-resolve; if unclear, `Reply` or `Ask`. Bots (CodeRabbit/Codex/Cursor/Copilot): credible hypotheses, verify against current code before `Fix`. High signal does not earn a `Fix` — evidence does.

## Step 1 — Fetch

Run from the target repo (keep the shell cwd there):

```bash
python3 <skill-dir>/scripts/fetch-comments.py
```

For explicit PRs: `--repo OWNER/REPO --pr 123` or `--url <pr-url>`. The script drops resolved threads and prior skill replies at the source, and fail-fast checks the PR `head_sha` against local `git rev-parse HEAD`. On a mismatch, stop and ask the user to checkout/update the branch — do not analyze against a stale checkout. If the script cannot run, fetch unresolved threads, PR-level comments, and review bodies with `gh api graphql` directly.

The output can be large — bots embed walkthroughs and base64 state. Read it with `jq`/grep to pull out threads and review bodies; do not dump the whole file into your response.

**Actionable** = unresolved inline threads — including security-scanner threads (CodeQL/Snyk) that point at a `file:line`, which are findings, not gates — plus review-body findings, including a nitpick that lives only in a review body with no thread of its own. **Drop as non-actionable** (do not card, reply, or resolve): CI/coverage gate status comments, Linear/Jira linkbacks, bot walkthrough/summary comments, the content-free "review" envelope a bot posts alongside its inline threads (the real findings are the threads, not the envelope), `@bot` trigger comments, and PR-author status updates (use the last only as a staleness signal).

If nothing actionable remains, output `No review comments found` and stop.

## Step 2 — Verdict (code-first)

For each thread, before forming a verdict, gather:

1. The focused diff: `git diff $(gh pr view --json baseRefName -q .baseRefName)...HEAD -- <path>`
2. The function/method containing the flagged line, plus nearby conventions (`CLAUDE.md`/`AGENTS.md`, lints, peer code).
3. For PR-level comments: the PR description and changed-file summary.

Then judge **the current code**, not the bot's text: does the issue exist here, right now? Assign a verdict from the table above and an impact:

- **`high`** — security, data loss, panic/crash on normal input, state-corrupting race, broken error handling, API-contract break, deploy/migration risk, resource leak.
- **`normal`** — everything else (edge cases, missing tests, readability, naming, style, nits).

Impact is your read of the code, not the bot's label. A bot-labelled "Critical" that the code disproves is `normal` + `Reply`. Drop items only when the code shows the issue cannot exist (e.g. the line moved and the claim is now false) — and even then, `Reply` to say so. When genuinely unsure, include it; over-including is recoverable, dropping real feedback is invisible.

**Dedupe** only when comments hit the same file/topic, point at the same function/path, ask for the same action category, and share ≥2 meaningful keywords. List all reviewers in the header (`[coderabbit/cursor]`), synthesize their angles in `Wants`, reply to each thread.

## Step 3 — Present & decide

Sort all items by impact: `high` → `ask` → `normal`, numbered globally (so `why 7` and cross-references work across both tiers). One card template — do not add fields. Present in **two tiers**:

- **`high` and `ask` — one at a time.** Show a single card, state the recommended verdict, and **stop for the user's decision.** Do not render the next card, or ask about several items at once, until the current one has a recorded verdict.
- **`normal` — paginated.** 5 per page, recommended verdict on each; `ok` / `ok all` accepts the normal defaults. `ok all` never touches a `high`/`ask` item.

Work the tiers in order: every `high` one-by-one, then every `ask` one-by-one, then the `normal` pages.

A `high` / `ask` item, presented alone — stop after it:

```text
── PR OWNER/REPO#123 — high 1 of 2 ──────────

[high] #1 worker.go:42  [coderabbit]  → Fix
Problem:  the worker loop never watches ctx.Done(), so when a request times out the goroutine just keeps running with nothing left to stop it.
Wants:    add a ctx.Done() branch to the select.
Evidence: worker.go:42 — `select { case w := <-work: …; case r := <-results: … }`, no ctx.Done() case.
Reason:   confirmed in current code; the leak is real, so Fix.

Decision? fix · reply · defer · ask · why  — I'll stop here until you choose.
```

The `normal` tier, batched 5 per page:

```text
── PR OWNER/REPO#123 — normal, page 1 of 1 ──────────

[norm] #3 model.go:33  [coderabbit]  → Reply
Problem:  the reviewer wants an option dropped because it looks unused, but the handlers next to it deliberately keep the same one for interface symmetry.
Wants:    delete the unused option here.
Evidence: model.go:33 — declared, unused; peers keep it for interface symmetry (model_a.go:41).
Reason:   removing it would break the shared interface shape; explain in the reply.

[norm] #4 …

ok / ok all confirms these normal defaults; `high`/`ask` stay open.
Reply with: ok · ok all · fix N · reply N · defer N · ask N · why N  (combinable: reply 3, defer 4)
```

Writing style for `Problem`/`Wants`/`Evidence`/`Reason`: natural language, **as if explaining to a colleague sitting next to you — in your own words, not the reviewer's.** Lead with the conclusion, name the exact function/field/line, and write as much as the point needs — a clause or several sentences, no length quota. `Problem` is your own read of what actually breaks in the current code, not a paraphrase of the bot; `Wants` is the concrete change the reviewer is after, as a brief action. No reviewer echo ("Consider adding…"), no hedging ("might", "could potentially"), no vague nouns ("logic", "handling", "issue"). "This breaks because…" / "This is safe because…" when it helps.

Render each field's value as one logical line — do **not** hand-wrap or indent-align continuation lines to the label column. Let the terminal soft-wrap; hand-wrapping at a narrow width wastes horizontal space and leaves the right side of the screen empty.

| Input | Behavior |
| --- | --- |
| `fix` / `reply` / `defer` / `ask` / `why` (no number) | Decide the `high`/`ask` item shown right now; then advance to the next one. |
| `ok` / `done` | Confirm this page's `normal` defaults. `high` and `ask` items stay open. |
| `ok all` | Same, across all remaining pages. `high`/`ask` still stay open. |
| `fix N` / `reply N` / `defer N` / `ask N` | Set the verdict for item N. Ranges/lists ok: `fix 1,3`, `reply 1-4`. |
| `why N` | Explain the verdict without changing it. |

For PR-level comments, add a `Signals:` line before `Problem` (positive reactions from the reviewer/author, later author follow-ups, whether the PR was updated after the comment). Show them; never auto-resolve on signals alone.

## Step 4 — Publish

Once every item has a verdict (no open `high`/`ask`), read `publish.md` and follow the single publish flow: preview → authorization → commit+push if any Fix → reply+resolve.

## Common Mistakes

1. Picking `Fix` with an empty or paraphrased Evidence field. No concrete artifact → not Fix.
2. Inheriting impact from the bot's label instead of reading the code.
3. Letting `ok all` clear `high` or `ask` items. It clears `normal` only.
4. Batching `high`/`ask` items — showing several cards together or asking for one combined decision. Show one, ask, stop; don't advance without a recorded verdict for it.
5. Adding fields to the card. Render the template as written.
