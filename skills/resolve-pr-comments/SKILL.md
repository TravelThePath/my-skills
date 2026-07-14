---
name: resolve-pr-comments
description: >
  Use when a PR has reviewer comments or threads to address, resolve, respond to, fix, or work through — from AI bots (CodeRabbit, Copilot, Cursor, etc.) or human reviewers. Covers both review-only passes and publish flows that commit fixes, post replies, and close GitHub threads. Do not use for general code review without open PR review threads.
---

# Resolve PR Comments

Work through unresolved PR review feedback with two lenses, in order: **first frame the PR yourself** — what it changes, why, and where each change sits in the architecture — from the diff and code *before* reading any bot's claim, so the bots don't set your frame; **then judge each thread** against that frame and the current code. The loop is **code-first**: read the code, reach your own verdict, then say whether the bot was right — the bot's claim is a hypothesis to verify, not a conclusion to defend.

Your own frame is not license to disagree. Direction comes from evidence and the frame, never a reflex for or against bots: the frame can confirm a bot more strongly than it argued, or show a locally-true claim doesn't matter here. Trading a habit of agreeing for a habit of pushing back is the same failure.

## Prerequisites

- `gh` CLI installed and authenticated.
- Current branch has an open PR, or the user provides a PR URL/number.

## Platform Rules

- Use `gh` for all GitHub calls. Do not use GitHub MCP tools.
- Blocking choices: `AskUserQuestion` in Claude Code; plain text + stop elsewhere.
- Keep output compact: verdicts, blockers, previews, outcomes, and raw API output stay out of responses. But *compact* means cutting noise — never compressing a verdict's reasoning. Never make the user decode dense shorthand or jargon to see why; explain in as many plain words as it takes.
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

## Step 2 — Frame (before reading any thread's claim)

Once, at PR level and **outside-in** — start from the system, not the diff. Read the wider context first, then locate the PR within it, drawing on the code, repo conventions (`CLAUDE.md`/`AGENTS.md`), the domain model, and cross-service contracts — **not** the bot comments:

- **Problem & approach:** what real need this change serves, and whether its *shape* is the right move for that need — the design-level question a line-scoped bot cannot ask.
- **System context:** the patterns, boundaries, and contracts it lives within — which service owns it, what it calls and is called by, which consistency/tenancy boundary it crosses, how peer code already solves the same thing.
- **Already guaranteed:** what the architecture, types, or existing code enforce elsewhere — so a comment demanding it again is moot.

This is the coordinate system every verdict is measured against, and it is deliberately wider than any one thread: a bot sees a single `file:line`; you see the whole. Keep it to a handful of lines — a lens, not an audit. Build it before you open the threads, so a bot's chosen granularity never becomes your frame.

## Step 3 — Verdict (code-first, against the frame)

For each thread, before forming a verdict, gather:

1. The focused diff: `git diff $(gh pr view --json baseRefName -q .baseRefName)...HEAD -- <path>`
2. The function/method containing the flagged line, plus nearby conventions (`CLAUDE.md`/`AGENTS.md`, lints, peer code).
3. For PR-level comments: the PR description and changed-file summary.

Then judge the comment on **two questions, both required**:

1. **Is it true?** Does the issue exist in the current code, right now?
2. **Does it matter, given the frame?** Is the bot pointing at the right layer? Does the architecture (Step 2) already guarantee this elsewhere? Does the fix fit the PR's intent, or is it a locally-true nitpick that pulls the change off-course?

A claim can be true but architecturally moot (→ `Reply`/`Defer`), or aimed at the wrong layer (→ `Reply`, and say where it actually belongs). The frame cuts both ways: when it shows the bot caught a real structural problem, that's a `Fix` with a harder rationale than the bot gave. Assign a verdict and an impact:

- **`high`** — security, data loss, panic/crash on normal input, state-corrupting race, broken error handling, API-contract break, deploy/migration risk, resource leak.
- **`normal`** — everything else (edge cases, missing tests, readability, naming, style, nits).

Impact is your read of the code, not the bot's label. A bot-labelled "Critical" the code disproves is `normal` + `Reply`. Drop an item only when the code shows the issue cannot exist — and even then, `Reply` to say so. When unsure, include it: over-including is recoverable, dropping real feedback is invisible.

**Dedupe** only when comments hit the same file/topic, point at the same function/path, ask for the same action category, and share ≥2 meaningful keywords. List all reviewers in the header (`[coderabbit/cursor]`), combine their points in `Wants`, reply to each thread.

## Step 4 — Present & decide

Sort all items by impact: `high` → `ask` → `normal`, numbered globally (so `why 7` and cross-references work across both tiers). Present in **two tiers**, worked in order — every `high` one-by-one, then every `ask` one-by-one, then the `normal` pages:

- **`high` and `ask` — one at a time.** Show a single card, state the recommended verdict, and **stop for the user's decision.** Don't render the next card until the current one has a recorded verdict.
- **`normal` — paginated.** 5 per page, recommended verdict on each; `ok` / `ok all` accepts the defaults. `ok all` never touches a `high`/`ask` item.
- **Nitpick / info cards — display only.** Render inside the `normal` pages, tagged `info` in place of a verdict, with no `Decision?` prompt. They need no decision, `ok` skips past them, and publish never replies to or resolves them.

Card body — `Problem`, then either `Wants` (for a `Fix`) or `Why <verdict>` (for `Reply`/`Defer`/`Ask`), plus a `Signals:` line for PR-level comments. Don't add other fields:

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

`normal` items render the same card — sized for clarity, not squeezed (see the tier budget below) — 5 per page under a `── PR … — normal, page 1 of N ──` header, closing each page with the literal prompt `Reply: ok · ok all · fix N · reply N · defer N · ask N · why N (combinable: reply 3, defer 4)`.

A `normal` `Reply`/`Defer` card, written to be understood in one read — jargon spelled out, reasoning in full sentences:

```text
── PR OWNER/REPO#123 — normal, page 1 of 1 ──────────

[normal] #4 store.go:162  [cursor]  → Reply
Problem:   The bot says a stored email with a trailing space won't match the
           lookup, so the merge skips those rows. That half is real. The other
           half it implies — different capitalisation breaking the match — is
           already handled: the column compares text case-insensitively, so
           "John@x.com" and "john@x.com" already match today.
Why Reply: The one real gap, a stored trailing space, shouldn't be patched
           inside the lookup query. Wrapping the query in TRIM() would stop the
           database from using the email index, making every lookup slow. The
           right fix is a one-off cleanup of the existing email values — its own
           task, not this PR. I'll post that reasoning as the reply.

Reply: ok · ok all · fix N · reply N · defer N · ask N · why N (combinable: reply 3, defer 4)
```

Write every field — `Problem`, `Wants`, `Why <verdict>` — in plain, simple, direct words. Say what you found as fact. Don't reuse the bot's wording. Every card must land one concrete case — real code and real values — not just a description of the mechanism.

- **`Problem`** — three parts, in order:
  1. One plain line: what breaks. Lead with the effect, not the code location.
  2. The offending code — the exact lines, so the reader sees it.
  3. A concrete failure: real input → what the code does → what it should do. Use real values ("30s timeout", "1000 requests"), not "a request".
  For `Ask`, replace parts 2–3 with what you can't judge and why it's out of repo scope.
- **`Wants`** (for a `Fix`) — the change as code, not prose. Show the line(s) to add or a before→after, not "add a ctx.Done() branch".
- **`Why <verdict>`** (for `Reply`/`Defer`/`Ask`, in place of `Wants`) — the plain-language reason the code doesn't change: what the bot assumed, why the current code doesn't match that, and for `Defer` the follow-up to track. This is the whole payload of a non-`Fix` card, so give it real room — full sentences, spelled out, no shorthand.
- **Ration code, never explanation — every tier.** The only thing kept tight is *code*: `high`/`ask` get the full block, `normal` show just the one line that matters, and never paste a wall of diff. Explanation is never rationed at any tier — if being clear takes five sentences, write five. When a term is repo- or DB-specific (`utf8_general_ci`, sargable, NO PAD), say what it does in plain words instead of just naming it. A `high`/`ask` card's full code block is not licence to go terse around it: the prose still carries the point and the code only illustrates it — never make the reader reverse-engineer the code to find out why it matters.
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

## Step 5 — Publish

Once every item has a verdict (no open `high`/`ask`), read `publish.md` and follow the single publish flow: preview → apply fixes + commit locally → **explicit `yes` confirmation gate** → push + reply + resolve. The push/reply/resolve step never runs without the user's explicit confirmation, even for a `fix` / `resolve` request.

## Step 6 — Learn (opt-in)

Triggered only when the user types `learn` after publish. Read `learn.md` and follow it.
