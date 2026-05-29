# Phase 2 — Shape

Design milestones + issue tree. Walked as three sequential checkpoints — outline, per-milestone review, final sign-off — each one its own user approval. Never collapse them into a single batched response.

## Entry conditions

- Phase 1 complete: every Open decision (if any) resolved inline or marked `PM follow-up required`.
- Linear project description + Step 3 grep evidence in chat context (these ground every shape decision).
- **If Phase 1 Step 4 ran**, the code-explorer's full findings are also in chat context. Treat them as a deeper grounding layer for Step A's Foundation/Cleanup heuristic, Step B's vertical-slice cuts (esp. file-overlap detection across siblings), and any Blocked-by lines that point at shared infra.

---

## Shape philosophy

```
Vertical slice rules

- Each issue is a tracer bullet — cuts through ALL relevant layers
  end-to-end (schema + API + UI + tests as applicable), never a horizontal
  slice of one layer.
- Issues are narrow but COMPLETE: a finished slice is demoable / verifiable
  on its own.
- Prefer many thin slices over few thick ones.
- Mark each slice AFK (autonomous) or HITL (needs human decision). Prefer
  AFK; flag HITL early so blockers surface.
- Within a milestone, no two issues may overlap on the same file — if they
  would, extract a foundation issue or restructure.
```

---

## Walk structure

Phase 2 has three sequential checkpoints. Each one is its own response with one open question at the end. Do not jump ahead — the next step starts only after the current step's approval.

### Step A — Milestone outline

In one response, produce:

1. **Milestone overview table** (planned, not yet locked):

   | Milestone | Title | Planned issues | Total points |
   |---|---|---|---|
   | M1 | <semantic name> | <N> | TBD |
   | ... | ... | ... | TBD |
   | Project total | — | <T> | TBD |

   Milestone titles name the **deliverable**, not the team or layer (`Distribution Payment Pack`, not `Backend work`). Planned-issues column carries an estimate; the real count locks at Step C. Total points stays TBD throughout Phase 2 — Phase 3 fills it via `write-linear-issue`.

2. **One open question**, e.g.: "Milestone outline above. Before I shape each one's issues — does this split look right?"

**Foundation / Cleanup heuristic for the initial outline:**

Read from chat context: the Linear project description (free-form prose Phase 1 Step 2 fetched), Step 3 grep evidence, and — if it ran — Step 4 explorer findings. Apply:

- **Foundation milestone** — if those inputs describe shared infra (schema migration, RBAC permission, new gRPC contract, new event topic) that multiple feature milestones will depend on → draft it as `M1 — Foundations` with the feature milestones numbered M2+. Feature milestones list its issues as Blocked-by.
- **Cleanup milestone** — if those inputs mention decommissioning legacy, retiring an old flow, or removing dead code after the new path is in production → draft it as the last milestone `M<last> — Cleanup`.
- **Otherwise** — start with 2-3 feature-only milestones and let the user push back if a Foundation or Cleanup is needed.

Loop within Step A: within-outline tweaks (rename, swap one milestone in/out, reorder) are patched in place and re-presented. If the reply signals a bigger restructure (multiple boundary moves, splits, scope shifts), re-read the Linear description + chat-context evidence and redraw the outline from zero before re-presenting. When in doubt about which the user means, ask once — never silently advance.

### Step B — Per-milestone issue review

After Step A is approved, walk M1 → M2 → ... → M<last> **in order**, one milestone per response. Do not batch.

For each milestone:

1. **Announce + present**: "Shaping M<N> — <title>." Then list every issue in that milestone — and nothing else. No milestone overview table, no preview of upcoming milestones, no recap of locked ones.

2. **Issue line format**:

   ```
   I<N.M> — <semantic title>
     Type: AFK or HITL (pick one)
     Blocked by: <issue codes> | none
     Points: TBD
   ```

   Issue titles use the convention `<Service> | <verb> <object>` (matches `write-linear-issue` skill's Title rule). Service prefix uses Title Case: `Investment Service`, `Ranger API`, `Voyager`.

   **Cross-milestone Blocked-by references** get an inline tag so the user does not have to scroll back. Format: `Blocked by: I<x.y> (M<x>: <2-4 word tag from the referenced issue's title>)` — e.g. `Blocked by: I1.2 (M1: Calc API)` when I1.2's title is `Investment Service | Build distribution calc API`. Same-milestone references stay bare: `Blocked by: I3.1`.

3. **End with one open question**, e.g.: "M<N> issues above. Granularity, AFK/HITL, deps — look right?"

4. **Wait for the user's reply.** Read it and act:
   - Approval → advance to the next milestone, or to Step C if this was the last.
   - Within-milestone feedback (rename / split / merge an issue here, flip its AFK/HITL, edit its Blocked-by within this milestone) → patch in one response and re-present.
   - Anything that touches a locked milestone, the outline itself (add/remove a milestone, move a boundary), or otherwise leaves the current milestone's scope → judge in context. Read the reply, propose the cleanest path back (which step to roll back to and why), and confirm once before acting. Cross-cutting Grilling style rules in `SKILL.md` cover the rest.

### Step C — Final sign-off

After every milestone in Step B is approved, produce the locked overview in one response:

1. **Locked overview table** — same columns as Step A's table but with the real count in `Locked issues`, plus a `Δ vs planned` column when any milestone's count changed:

   | Milestone | Title | Locked issues | Δ vs planned | Total points |
   |---|---|---|---|---|
   | M1 | Foundations | 3 | — | TBD |
   | M2 | Statement Pack | 4 | -1 (merged I2.3 into I2.1) | TBD |
   | M3 | Cleanup | 2 | — | TBD |
   | Project total | — | 9 | — | TBD |

   Use `—` in `Δ vs planned` when nothing changed. The annotation in changed rows should fit on one line.

2. **One open question**, e.g.: "Shape locked, <T> issues across <M> milestones — ready to draft?"

Step C has no in-place patch surface — it shows only milestone names and counts, which were set in earlier steps. If the user pushes back, judge in context: route the feedback back to Step A (outline-level) or to the relevant milestone's Step B (issue-level). Confirm once before rolling back.

---

## Hard rules

- **No two siblings in a milestone may touch the same file.** If shape would violate this, extract a foundation issue or restructure before presenting that milestone.
- **Every issue must have Type (AFK / HITL).** Don't leave blank.
- **Every issue must have Blocked by line** (write `none` if no blockers).
- **Story points are filled in Phase 3** after `write-linear-issue` is loaded. In Phase 2, every issue's Points line is `TBD`. Do not estimate yet — Phase 2's job is shape, not sizing.
- **No parallel timeline, no dep-graph ASCII.** Blocked-by lines per issue are the only dependency representation in this skill.

---

## Exit conditions

- Step A outline approved.
- Step B every milestone's issue tree approved, in order.
- Step C locked overview approved.
- Advance to `phases/03-draft.md`.
