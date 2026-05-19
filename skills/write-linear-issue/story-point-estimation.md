# Story Point Estimation

Fibonacci: **0, 1, 2, 3, 5, 8, 13**. `13` must be split before sprint — it is allowed as a diagnostic signal, not a shippable size.

## Meta rule — estimate from scratch

Estimate the total work to ship from zero, as if no one has started. Do NOT discount for:

- Investigation already done
- A PR that already exists (reverse-engineered tickets)
- Decisions made elsewhere

The estimate represents *total work to ship*, not *work remaining*. This is the only basis on which team velocity stays self-consistent.

## Step 1 — Pick the row

Find the row whose **shape** matches the work. The time anchor is for a senior engineer unfamiliar with the work, excluding review / QA / meetings.

| Points | Shape | Time anchor |
|---|---|---|
| **0** | Story-label parent with no work beyond child tasks | — |
| **1** | Copy / config / comment / single trivial flag flip | minutes |
| **2** | Bounded logic OR trivially reversible config / schema flag in one file area, clear requirements | hours |
| **3** | One PR's worth in one service: a single endpoint OR a single schema field OR a single UI surface | ~1 day |
| **5** | Multi-surface in one service (schema + API + UI together), OR a second service owned by the same team | ~2–3 days |
| **8** | Cross-cutting: third or more services, migration with backfill or dual-write, or a genuinely novel architectural pattern | ~1 week |
| **13** | Initiative — MUST be split. Do not ship as one issue. | >1 week |

Rules:

- **Pick the lowest row that fits.** When torn between two rows, pick the lower one and let Step 2 decide whether to escalate.
- **Service count is a row signal, not an escalator** — already captured by the row's shape.
- **Schema / contract changes are NOT inherently +1** — already absorbed into row shape (3 covers one schema field; 5 covers schema + API + UI together).
- **Content volume is a row signal**, not an escalator — ≥5 genuinely independent sub-deliverables (would ship in separate PRs in normal practice) pushes the row from 3 to 5 or from 5 to 8.

## Step 2 — Escalate one row if ANY trigger fires

Cap: **single bump**. Multiple triggers do not stack. Each trigger requires a concrete signal in the issue text.

| Trigger | Fires when |
|---|---|
| **External-spec authority** | Work must conform to a *named* external spec (AIIR record widths; ITAA s276; ISO 4217; WCAG 2.2 AA; Stripe API; OAuth 2.1; RFC 7234; named design system), AND the ticket does not merely extend a pre-existing spec-conformant pattern. Triggers when the engineer must reason directly from the spec to make new design decisions. |
| **Cross-team coordination** | A PR or owner outside your team is in the critical path. |
| **High unknowns** | Core problem unsolved here before; sparse docs; no adjacent precedent in the repo. |
| **Legally / financially observable output** | Wrong output is externally visible — statements, tax filings, money movement, regulator-facing files, customer-visible totals. |

If an escalation would push the result above 8, **stop and split the issue**. Do not score 13 from an escalation — 13 is reserved for inherently month-scale initiatives that need decomposition.

"Industry best practice" or "looks like good design" does NOT qualify as external-spec authority. The authority must be nameable.

## Output format

```
**Story point: N** — row: <shape>; <escalator if any with concrete signal>.
```

Banned phrases: `effort severe`, `complexity moderate`, `feels like a 5`. The rationale must name the row shape and the concrete trigger that moved it (or the absence of a trigger).

### Examples

`**Story point: 3** — row: one schema field on accounts; no escalator.`

`**Story point: 5** — row: schema + API + UI on accounts in one service; no escalator.`

`**Story point: 5** — row: one schema field; external-spec authority fired (AIIR record widths drive the column definitions and valid code sets).`

`**Story point: 8** — row: migration with backfill in document-service; cross-team escalator fired (Voyager team owns the downstream consumer and must deploy in lockstep).`

`**Story point: 2** — row: trivially reversible config flag flip in one service.`

`**Story point: 1** — row: pure copy change in one component.`

`**Story point: 13** — row: initiative; MUST split before sprint. Suggested split: schema PR + backfill PR + reader migration + writer migration.`

## Anti-patterns — do NOT use these as escalation signals

| ✗ Don't | Reason |
|---|---|
| Description length | Long ticket = clear writing, not heavy work. |
| Heading / sub-heading count | Structure ≠ scope. |
| Domain-term density (`AMIT`, `s276`, `ITAA`) | Only counts if the term names an external spec the engineer must conform to — see Step 2. |
| Bundled sub-issues | Bundling is an organizational choice. Score the actual work. |
| Listed Scope items in one PR | Items inside one PR are not independent sub-deliverables. |
| Field count within one schema change | Captured by the row (3 = single field; 5 = multi-surface including multi-field). |
| "Why this design" sentence count | Rewards verbose authors and punishes terse-but-correct ones. |

## Story-label parent issues

Parent containers whose work lives in child task issues. Each child is scored individually, so the parent must not double-count.

- Default the parent to **0 or 1**.
- Score higher only when the parent itself carries coordination overhead beyond what the child tasks capture (cross-team kickoff, launch sync, planning effort).

## How to estimate — process

1. **Read Context and Scope.** Internalize what is being built (not how it's described).
2. **Pick the lowest row that fits** by matching the shape — service count, surface count, presence of migration.
3. **Check the four escalator triggers.** Bump one row if any fires. Stop at the first match — cap is one bump.
4. **If the result exceeds 8**, stop and recommend a split — do not snap to 13 from an escalation.
5. **Output the score with a one-sentence rationale** that names the row shape and the concrete trigger (or its absence).

## When you cannot estimate confidently

Say so explicitly and name what is missing. Do **not** guess.

Examples of missing information that block estimation:

- Number of services in scope is unclear
- External-spec authority is referenced but not named
- Migration strategy (dual-write? backfill?) is unspecified
- Cross-team dependencies exist but aren't enumerated

Hoist these into `[CONFIRM]` items in the issue draft, resolve them, then estimate.
