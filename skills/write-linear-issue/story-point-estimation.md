# Story Point Estimation

Fibonacci only: **0, 1, 2, 3, 5, 8, 13**. Always estimate the **total work to ship** — never "work remaining". An existing PR or finished investigation doesn't lower the score.

Scoring is two steps: read the **scope** off the scale, then **bump** for risk.

## Scale (scope)

The row measures pure scope — how much the change touches. Risk is handled separately below.

| Points | Time      | Scope                                                         |
| ------ | --------- | ------------------------------------------------------------- |
| **0**  | —         | Parent issue with no own work; children carry the points      |
| **1**  | minutes   | Copy, config, a single trivial flag                           |
| **2**  | hours     | Bounded logic in one area, clear requirements                 |
| **3**  | ~1 day    | Single PR in one service                                      |
| **5**  | ~2-3 days | Multi-surface in one service, or touching a second service    |
| **8**  | ~1 week   | Cross-service change                                          |
| **13** | >1 week   | Too big — split before sprint                                 |

## Risk bumps

Take the base row, then move **one Fibonacci step up** for each that is true:

- **Stateful work** — migration, backfill, or fixing existing production data. Code can be reverted; data can't.
- **Unknowns** — you couldn't start coding today without investigating or asking someone first.
- **Coordination** — another team or service owner must move in lockstep with you.

None true → base score stands. If bumps push you to 13, the issue isn't just big, it's risky — split it, or de-risk first (spike, clarify requirements) before estimating.

## Output

The score, plus one line showing base and bumps:

```
**Estimate: 8** — base 5 (schema + API + Admin App in one service), +1 step for the backfill.
```

```
**Estimate: 3** — single PR in investment-service, no bumps.
```

## Story-label parents

Issues with the `Story` label are containers — children are estimated individually, so the parent defaults to **0 or 1**. Score higher only when the parent itself carries coordination work no child captures.
