# Story Point Estimation

Fibonacci only: **0, 1, 2, 3, 5, 8, 13**. A point estimates **relative size** — effort + complexity + uncertainty combined — **not duration**. Calendar time is *derived later* from your cycle's measured velocity, not read off the scale. Always estimate the **total work to ship**, never "work remaining": an existing PR or finished investigation doesn't lower the score.

Scoring is two steps: read the **scope** off the scale, then **bump** for risk. The result is one combined size.

## Scale (scope)

The row measures how much the change touches and how complex it is. The bumps below handle risk. The `≈ Time` column is a loose gut-check only — *not* the definition. If a change reads as a 5 by scope/complexity but feels quick, it's still a 5; your velocity, measured over cycles, is what turns points into dates. When the time hunch and scope/uncertainty disagree, trust scope + uncertainty.

| Points | ≈ Time (gut-check) | Scope / complexity                                            |
| ------ | ------------------ | ------------------------------------------------------------- |
| **0**  | none               | Parent/container issue with no own work; children carry the points |
| **1**  | minutes            | Copy, config, a single trivial flag                           |
| **2**  | hours              | Bounded logic in one area, clear requirements                 |
| **3**  | ~1 day             | Single PR in one service                                      |
| **5**  | ~2-3 days          | Multiple surfaces in one service, or reaching into a second   |
| **8**  | ~1 week            | Change spanning services                                      |
| **13** | >1 week            | Too big — split (or de-risk) before the sprint                |

Service count is a proxy for blast radius and complexity, not a literal driver — a trivial two-service edit can be lighter than deep logic in one. Read the row by effort, not surface tally.

## Risk bumps

Uncertainty and coordination grow faster than raw effort, so they belong in the number. Take the base row, then move **one Fibonacci step up** for each that is true:

- **Stateful work**: migration, backfill, or fixing existing production data. You can revert code; you can't revert data.
- **Unknowns**: you couldn't start coding today without investigating or asking someone first.
- **Coordination**: another team or service owner must move in lockstep with you.

None true → base score stands. If bumps push you to 13, the issue is risky; split it, or de-risk first (spike, clarify requirements) before estimating.

## Output

The score, plus one line showing base and bumps:

```
**Estimate: 8**. Base 5 (schema + API + Admin App in one service), +1 step for the backfill.
```

```
**Estimate: 3**. Single PR in investment-service, no bumps.
```
