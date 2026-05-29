# Phase 3 — Draft

Per-milestone issue bodies. The issue-writer skill is the **source of truth** for everything about an issue body: shape, sections, grounding, estimation, quality gates. Invoke it once at phase entry and follow whatever it says — Phase 3 never re-describes its rules. Phase 3 owns the surrounding orchestration: single invocation, plan-metadata wrapping, per-milestone batching, review loop.

## Entry conditions

- Phase 2 shape locked (Step C sign-off approved).
- Per-milestone issue trees available in chat context.
- **If Phase 1 Step 4 ran**, code-explorer findings remain in chat context. Use them for issue-body technical grounding (existing patterns to extend, prior implementations to cite, subtle dependencies) and as input to the issue-writer skill's estimation rationale.

---

## Steps

### 1. Invoke the issue-writer skill once

At phase entry, invoke the skill via the `Skill` tool with `skill: write-linear-issue` — this is the one place the issue-writer is named literally; everywhere else in this skill refers to it as "the issue-writer skill." One invocation only — its loaded content stays in chat context for the rest of Phase 3. Do **not** re-invoke per issue.

If the issue-writer skill requires per-issue choices that should be consistent across this plan, resolve them once at phase entry and apply uniformly to every issue — do not let it ask per invocation.

### 2. Batch issue drafts by milestone

Process milestones in order (M1 → M2 → ...). For each milestone:

1. **Announce**: "Drafting M<N> issues."
2. **Draft every issue in the milestone in one response.** For each issue, first re-read from the Linear project description the exact `Decision Register` rows, `Product Acceptance Criteria` bullets, and key-flow section this issue implements — the issue-writer skill grounds against this verbatim source, not Phase 1's summarized excerpts, so do not rely on memory of the spec. Then run the issue-writer skill end to end on that issue, wrap its output in the Phase 3 plan-metadata block (template below), and apply whatever quality gates it defines before presenting.
3. **End the response** with one open question: "M<N> drafts above. Anything to revise before I move on?"
4. **Wait for user reply.** Read it and act per phase-transition rules in `SKILL.md` (approval → next milestone; feedback → patch; shape push-back → roll back to Phase 2; ambiguous → clarify).

### 3. Per-issue chat presentation

```markdown
### I<N.M> — <semantic short tag>

**Type**: AFK / HITL · **Blocked by**: <issue codes> | none · **Story points**: <N>
**Estimate rationale**: base <N> + <adjuster, e.g. "+2 new schema"> - <reducer, e.g. "-1 reused calc helper">; <one-line summary>. (Defer to the issue-writer skill's estimation rules for the actual values.)

#### Linear issue (uploaded in Phase 4)

<the issue-writer skill's output verbatim — whatever Title + body it produced>
```

The plan-metadata block (Type, Blocked by, Story points, Estimate rationale, I<N.M> tag) stays in chat only — it threads Phase 2 decisions into Phase 4 publish and never enters the Linear issue body. Everything under the `#### Linear issue (uploaded in Phase 4)` heading is what Phase 4 writes to Linear verbatim; the I<N.M> tag identifies which `/tmp/i<N.M>_body.md` Phase 4 should write.

---

## Loop rules in this phase

Cross-cutting **Interaction style** rules in `SKILL.md` apply. Phase-specific additions:

- **Approval per milestone** → advance to the next milestone (or to Phase 4 if this was the last).
- **Single- or multi-issue feedback within the current milestone** → patch flagged issues in one response; re-present.
- **Push back on the shape itself** (milestone boundaries change, issue addition or removal across milestones) → roll back to Phase 2 after confirming once.
- **Ambiguous reply** → ask one clarifying question before acting.

---

## Exit conditions

- All milestones drafted.
- User approves the last milestone's drafts.
- Advance to `phases/04-publish.md`.
