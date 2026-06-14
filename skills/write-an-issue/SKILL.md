---
name: write-an-issue
description: >
  Use when creating, writing, drafting, filing, or rewriting any Linear issue or ticket: feature, fix, update or one-off task.
---

# Linear Issue: plain, intention-first

A ticket states the **intention** (what to build, do, update, or fix) and how we'll know it's done. Not *how* to implement it (that's the PR), and not the full business case (that's the project description).

## Style

- Plain, direct language. Lead with the point; a reader should get the gist from the first sentence.
- Every line must add something new: a behavior, a value, a surface, a constraint. Cut filler, repetition, and implementation detail.
- Length follows the change: a small fix can be two lines; a large issue takes as much space as its real requirements need. Never pad it with *how*.
- If one issue describes several independent slices of work, split it.

## Title

`<Service Name> | <action>` when one service owns the change, with the casing the repo uses: `Investment Service | Fix ABA decimal rounding`. For non-service work, a plain action statement is fine.

## Body

Two sections:

**1. Description**: what to build / do / update / fix, in business language.

- Bullets when several surfaces or behaviors are involved; prose when it's one simple change.
- Backticked identifiers from real code are good anchors (`bankCode`, `ListAccounts`). File paths and code syntax (SQL, proto, DDL, declarations) are not. That's *how*.
- Copy decided values verbatim from the source (spec, `Decision Register`, or the user's words): widths, limits, field names, defaults, fallbacks. Don't paraphrase, don't re-derive from memory, never revive a decision the source dropped. If something is undecided, ask the user instead of guessing.

**2. AC**: observable outcomes from the user's / caller's / data's point of view. As many as the change has, no more. Not test steps (no Run / Verify / Check), and not a reworded copy of the Description. If the change is additive or optional, existing callers staying unaffected is itself an outcome. Give it its own AC rather than leaving it implied.

## Example

```markdown
Ranger API | Let admin account listing include archived accounts

## Description

Admins need to see archived accounts when looking up an investor's history. Today `ListAccounts` only returns active accounts and there is no way around it.

- Add an `include_archived` option to `ListAccounts`. Default stays as today: archived accounts are excluded, so existing callers see no change.
- When the option is on, archived accounts come back in the same shape as active ones, with their existing `archivedAt` timestamp populated.
- Expose the option in the Admin App account search as an "Include archived" toggle, off by default.

## AC

- Existing callers that don't set the option get exactly the results they get today.
- A call with the option on returns both active and archived accounts, and each archived account carries its `archivedAt` value.
- In the Admin App, turning the toggle on shows archived accounts in search results; turning it off hides them again.
```

This example works: the first two lines say the intention in plain words; each bullet adds one decision (the option, its default, its surfaces); identifiers anchor it to real code without any proto or SQL; AC describes what a caller or admin would observe.
