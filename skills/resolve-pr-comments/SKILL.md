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
- An explicit `resolve` / `fix` / `publish` request authorizes applying fixes and committing them locally (see `publish.md`) once every item has a recorded verdict — but pushing, posting replies, and resolving threads **always** require a final explicit `yes` from the user, asked after every fix is committed. Never push or resolve on the strength of the original request alone. A review-only request (`pr review`, `review comments`) does not authorize the publish lane — ask once before any GitHub write.

## Verdicts

One vocabulary for both your recommendation and the user's choice.

| Verdict | Meaning | Requires |
| --- | --- | --- |
| **Fix** | Change code in this PR, reply `Fixed in <commit>`, resolve. | A concrete current-code citation in `Problem`: a `file:line` + quote, or a grep/diff/test result confirming the issue exists now. |
| **Reply** | No code change. Post an explanation, resolve. Covers stale, already-handled, convention-mismatch, and noise. | A reason the bot's specific claim does not match current code. |
| **Defer** | Like Reply, but the concern deserves tracked follow-up — also create a follow-up issue/note. Use Reply when acknowledging without tracking. | The concern is real but belongs in separate work. |
| **Ask** | You cannot judge within this repo (cross-service contract, production/migration state, product tradeoff). Hand it to the user. | The blocker is about cross-file / ownership / process / product, not current code. |

**The one hard rule:** no concrete current-code citation in `Problem` → you haven't verified it → not a `Fix`; use `Reply`, `Defer`, or `Ask`. Never invent evidence. `Ask` isn't a final answer: the user has to turn it into Fix/Reply/Defer before publish.

**Who the reviewer is** changes how hard you look and what order you work in — never the verdict. Humans: never auto-resolve; if unclear, `Reply` or `Ask`. CodeRabbit / Codex / Cursor: worth taking seriously — but verify before `Fix`. Copilot and unknown bots: low trust — demand hard evidence, review last. A reviewer's name never earns a `Fix`; evidence does.

## Step 1 — Fetch

Run from the target repo (keep the shell cwd there):

```bash
python3 <skill-dir>/scripts/fetch-comments.py
```

For explicit PRs: `--repo OWNER/REPO --pr 123` or `--url <pr-url>`. The script drops resolved threads and prior skill replies at the source, and checks the PR `head_sha` against local `git rev-parse HEAD` — on a mismatch, stop and ask the user to checkout/update the branch rather than analyze a stale checkout. If the script can't run, fetch unresolved threads, PR-level comments, and review bodies with `gh api graphql` directly. Output can be large (bots embed walkthroughs and base64 state) — read it with `jq`/grep; don't dump it into your response.

**Actionable** = unresolved inline threads (including security-scanner threads like CodeQL/Snyk that point at a `file:line` — they're findings, not gates) and substantive review-body findings.

**Nitpick-labelled comments** (a `nitpick` / `🧹 Nitpick` tag, or a review-body nitpick `<details>` block — split into one item per finding): **show only.** Render each as a compact `normal` card so the user sees it, but never reply to or resolve a nitpick thread — it's information, not a task. No verdict is required and it never gates publish. If the user asks to `fix` one, apply the code change but still leave the thread untouched (no reply, no resolve).

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

**Dedupe** only when comments hit the same file/topic, point at the same function/path, ask for the same action category, and share ≥2 meaningful keywords. List all reviewers in the header (`[coderabbit/cursor]`), combine their points in `Wants`, reply to each thread.

## Step 3 — Present & decide

Sort all items by impact: `high` → `ask` → `normal`, numbered globally (so `why 7` and cross-references work across both tiers). Present in **two tiers**, worked in order — every `high` one-by-one, then every `ask` one-by-one, then the `normal` pages:

- **`high` and `ask` — one at a time.** Show a single card, state the recommended verdict, and **stop for the user's decision.** Don't render the next card until the current one has a recorded verdict.
- **`normal` — paginated.** 5 per page, recommended verdict on each; `ok` / `ok all` accepts the defaults. `ok all` never touches a `high`/`ask` item.
- **Nitpick / info cards — display only.** Render inside the `normal` pages, tagged `info` in place of a verdict, with no `Decision?` prompt. They need no decision, `ok` skips past them, and publish never replies to or resolves them.

Card body — `Problem` and `Wants` only (plus a `Signals:` line for PR-level comments, shown below). Don't add other fields:

```text
── PR OWNER/REPO#123 — high 1 of 2 ──────────

[high] #1 worker.go:42  [cursor]  → Fix
Problem:  A cancelled request leaks a goroutine forever — nothing stops it.
          worker.go:42 selects on work/results but not ctx.Done():
              select {
              case w := <-work:    ...
              case r := <-results: ...
              }
          Caller times out at 30s → this goroutine stays blocked on the
          channels. 1000 cancelled requests → 1000 stuck goroutines.
Wants:    Add a ctx.Done() arm so cancel returns:
              case <-ctx.Done():
                  return ctx.Err()

Decision? fix · reply · defer · ask · why  — I'll stop here until you choose.
```

`normal` items render the same card but compact (see the tier budget below), 5 per page under a `── PR … — normal, page 1 of N ──` header, closing each page with the literal prompt `Reply: ok · ok all · fix N · reply N · defer N · ask N · why N (combinable: reply 3, defer 4)`.

Write `Problem` and `Wants` in plain, simple, direct words. Say what you found as fact. Don't reuse the bot's wording. Every card must land one concrete case — real code and real values — not just a description of the mechanism.

- **`Problem`** — three parts, in order:
  1. One plain line: what breaks. Lead with the effect, not the code location.
  2. The offending code — the exact lines, so the reader sees it.
  3. A concrete failure: real input → what the code does → what it should do. Use real values ("30s timeout", "1000 requests"), not "a request".
  For `Ask`, replace parts 2–3 with what you can't judge and why it's out of repo scope.
- **`Wants`** — the change as code, not prose. Show the line(s) to add or a before→after, not "add a ctx.Done() branch".
- **Budget by tier.** `high`/`ask` (one at a time) get the full code block. `normal` (5 per page) stay tight — one concrete line, code only when a single line makes it clear. Never paste a wall of diff; show the one line that matters.
- No hedging ("might", "could"), no vague nouns ("logic", "handling", "issue").

```text
Bad (mechanism only):  Problem: worker.go:42's select has no ctx.Done() case, so a timed-out request leaks the goroutine.
Good (concrete case):  Problem: A cancelled request leaks a goroutine forever — nothing stops it.
                           select { case w := <-work: ...; case r := <-results: ... }   // no ctx.Done()
                       Caller times out at 30s → goroutine stays blocked. 1000 cancelled → 1000 stuck.
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

Once every item has a verdict (no open `high`/`ask`), read `publish.md` and follow the single publish flow: preview → apply fixes + commit locally → **explicit `yes` confirmation gate** → push + reply + resolve. The push/reply/resolve step never runs without the user's explicit confirmation, even for a `fix` / `resolve` request.

## Step 5 — Learn (opt-in)

Triggered only when the user types `learn` after publish. Read `learn.md` and follow it.
