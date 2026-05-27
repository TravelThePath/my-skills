# Story Point Estimation

Reference for estimating story points on Linear issues. Use Fibonacci values only: **0, 1, 2, 3, 5, 8, 13**. Points are relative effort, not exact time — they combine effort, time, complexity, and risk/uncertainty.

**Estimate from scratch.** Score the total work to ship as if no one had started — don't discount for investigation already done, a PR that already exists, or decisions made elsewhere. The estimate represents *total work to ship*, not *work remaining*. This is the only basis on which team velocity stays self-consistent.

## The scale

| Points | Time      | Shape                                                          |
| ------ | --------- | -------------------------------------------------------------- |
| **0**  | —         | Parent issue with no own work; all work lives in child tasks   |
| **1**  | minutes   | Copy, config, or a single trivial flag                         |
| **2**  | hours     | Bounded logic in one area with clear requirements              |
| **3**  | ~1 day    | Single PR in one service                                       |
| **5**  | ~2-3 days | Multi-surface in one service, or work crossing into a second   |
| **8**  | ~1 week   | Cross-service, migration with backfill, or notable unknowns    |
| **13** | >1 week   | Initiative — must split before sprint                          |

The shape column blends effort, time, complexity, and risk into one judgement. The output format below breaks them back out so reviewers can see what drove the score.

## Process

1. **Read the issue.** Understand the total work to ship — not "what's left after the PR I already wrote."
2. **Find the row.** Match the shape holistically; if torn between two rows, prefer the lower.
3. **Sanity-check by walking the four dimensions** — effort, time, complexity, risk/uncertainty. If any of them pushes you to the higher row, take the higher row.
4. **Decide split or write.** If you land at 13 — directly from step 2, or because step 3 pushed you there — stop and recommend a split. Otherwise, write the output in the format below.

## Output

```
**Estimate: N points**

- **Effort**: <level> — <concrete anchor in 1 short clause>
- **Time**: <duration>
- **Complexity**: <level> — <concrete anchor>
- **Risk / uncertainty**: <level> — <concrete anchor>
- **Rationale**: <1-2 sentences naming the row shape; mention the dominant driver if any>
```

Effort, Complexity, and Risk lines must each include a concrete anchor — not just a level word. `Effort: moderate` alone is not enough; `Effort: moderate — schema + API + UI in one service` is. The Rationale line must name the row shape so the reviewer can locate it in the scale table.

If information is missing, say so in the Risk line (e.g., `Risk: high — migration strategy unspecified`) and explain what blocks confident estimation.

### Examples

**Pure row match:**

```
**Estimate: 3 points**

- **Effort**: mild — one field on the accounts schema
- **Time**: ~1 day
- **Complexity**: low — no auth changes, no new pattern
- **Risk / uncertainty**: low — adjacent fields use the same shape
- **Rationale**: Single PR in one service.
```

**Row pushed up by additional factors:**

```
**Estimate: 8 points**

- **Effort**: moderate — schema + API + UI in document-service plus a backfill
- **Time**: ~1 week
- **Complexity**: medium — backfill must run before reader cutover
- **Risk / uncertainty**: moderate — Voyager team owns the downstream consumer; deploy must be lockstep
- **Rationale**: Multi-surface in one service plus migration; cross-team coordination is the dominant driver.
```

**Must split:**

```
**Estimate: 13 points** — must split before sprint.

- **Effort**: severe — touches 4 services and requires dual-write
- **Time**: ~1 month
- **Complexity**: high — new event format must be consumer-first
- **Risk / uncertainty**: high — no precedent for dual-write in this domain
- **Rationale**: Initiative scope. Natural cut points: schema PR, backfill PR, reader migration, writer migration.
```

## Story-label parents

Issues with the `Story` label are parent containers; each child task is estimated individually, so the parent must not double-count. Default the parent to **0 or 1**. Score higher only when the parent itself carries coordination overhead — cross-team kickoff, launch sync, planning effort — beyond what the child tasks capture.
