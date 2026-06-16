---
name: write-an-issue
description: >
  Produces a plain, intention-first Linear issue: a title plus a description of what to build, do, update, or fix, stated concretely enough to know when it's done. Use when writing, drafting, filing, or rewriting any Linear issue or ticket — feature, fix, update, or one-off task.
---

# Linear Issue: plain, intention-first

A ticket states the **intention** — what to build, do, update, or fix — concretely enough that done is unambiguous. Not *how* to implement it (that's the PR), not the full business case (that's the project description).

## Style

- Plain, direct language. Lead with the point; a reader should get the gist from the first sentence.
- Every line must add something new: a behavior, a value, a surface, a constraint. Cut filler, repetition, and implementation detail.
- Length follows the change: a small fix can be two lines; a large issue takes as much space as its real requirements need. Never pad it with *how*.
- If one issue describes several independent slices of work, split it.

## Title

`<Service Name> | <action>` when one service owns the change, with the casing the repo uses: `Investment Service | Fix ABA decimal rounding`. For non-service work, a plain action statement is fine.

## Description

What to build / do / update / fix, in business language. It's the only section, so it needs no heading — the body follows the title directly.

- Bullets when several surfaces or behaviors are involved; prose when it's one simple change.
- Backticked identifiers from real code are good anchors (`bankCode`, `ListAccounts`). File paths and code syntax (SQL, proto, DDL, declarations) are not — that's *how*.
- Copy decided values verbatim from the source (spec, `Decision Register`, or the user's words): widths, limits, field names, defaults, fallbacks. Don't paraphrase, don't re-derive from memory, never revive a decision the source dropped. If something is undecided, ask the user instead of guessing.

## Example

```markdown
Ranger API | Let admin account listing include archived accounts

Admins need to see archived accounts when looking up an investor's history. Today `ListAccounts` only returns active accounts and there is no way around it.

- Add an `include_archived` option to `ListAccounts`. Default stays as today: archived accounts are excluded, so existing callers see no change.
- When the option is on, archived accounts come back in the same shape as active ones, with their existing `archivedAt` timestamp populated.
- Expose the option in the Admin App account search as an "Include archived" toggle, off by default.
```

This works: the first two lines state the intention in plain words; each bullet adds one decision — the option, its default, its surfaces; backticked identifiers anchor it to real code without any proto or SQL.
