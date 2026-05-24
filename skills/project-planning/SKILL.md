---
name: project-planning
description: |
  Use when the user asks to "plan a project", "break down this project", "create an engineering plan", or "plan the work for [project name]". Also use when reviewing or refining an existing engineering plan.
---

# Engineering Project Planning

Convert a refined Linear project scope into a delivery plan — milestones, vertically-sliced issues, story-point estimates — then publish to Linear. Designed for 1-3 engineers picking up issues in parallel without stepping on each other.

This skill orchestrates 4 phases that map directly onto a real planning session: gather context (project + code evidence + open-decision gate), shape milestones + issues, draft issue bodies, publish to Linear. Each phase has its own file under `phases/` — load only what the current phase needs.

The skill does **not** produce a written brief. Linear already holds the project description; re-stating it would be redundant work for the user. Phase 1 leaves the Linear description, grep evidence, and (optional) explorer findings in chat context for downstream phases to read directly.

## Startup

On skill activation, do exactly this:

1. Read `phases/01-context.md` and follow it (it owns project identification, ref loading, grep verification, optional deep exploration, and the Open decisions gate). Do **not** read other phase files yet.
2. After each phase's exit conditions are met (the user approves the current artefact), read the next phase file and continue.

---

## Phase overview

| Phase | Goal | Output |
|---|---|---|
| 1 — Context | Read project + load refs + grep-verify surface area + optional deep code exploration + conditional Open-decisions gate | Linear description + grep evidence + optional explorer findings, all in chat context |
| 2 — Shape | Milestone outline → per-milestone issue walk → final sign-off | locked shape (in chat) |
| 3 — Draft | Per-milestone issue bodies, invoking the `linear-issue` skill once | drafts (in chat) |
| 4 — Publish | Pre-publish review → write to Linear | Linear issues |

**Phase files**:
- `phases/01-context.md`
- `phases/02-shape.md`
- `phases/03-draft.md`
- `phases/04-publish.md`

Read the current phase file at phase entry. Do not pre-load all phase files.

---

## Grilling style

Cross-cutting philosophy for **every** user-facing interaction in this skill — Phase 1 Open decisions gate (conditional), Phase 2 shape review (outline + per-milestone + sign-off), Phase 3 draft review per milestone, and the Phase 4 publish gate.

Interview the user relentlessly until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

- **One question at a time.** Wait for the user's reply before posing the next. Never batch.
- **Walk the design tree depth-first.** Resolve dependencies between decisions before branching — a downstream question is wasted if an upstream answer reframes it.
- **Recommended answer + 1-2 alternatives + a one-line rationale**, grounded in the Linear project description, grep / explore evidence in chat, or a shared reference. Never offer choices without a recommendation.
- **Explore before asking.** If a question can be answered by reading the repo, shared references, or Linear project — do that instead of asking the user. Only ask when human judgment is required (preference, team, priority, product trade-off).
- **Accept short replies** (`A`, `B`, `yes`, `no`, free text). Push back is feedback — re-pose with revised options; do not advance.
- **Relentless, not exhausting.** Stop a branch when it's resolved. Add follow-ups only when an answer surfaces a real ambiguity, not to pad. Each phase has a fixed floor of mandatory checkpoints; the ceiling is the user's tolerance, not a quota.

---

## Phase transitions

This skill talks to a senior engineer. At the end of each phase, ask one open question (phase-specific — see each phase file for the wording) and read the user's natural-language reply. Decide what to do based on the reply's intent — do not match a fixed keyword vocabulary.

- **Approval** (user is satisfied, no complaints raised) → advance to the next phase.
- **Single-point feedback on the current artefact** → patch the artefact in place and re-present.
- **Multiple structural complaints in one reply** → judge in context: patch if it's a series of within-step tweaks, redraw if it's a structural restructure. If unsure, ask once.
- **Request to go back** (user wants to revisit an earlier phase or step, or change a Phase 1 answer) → roll back to the phase or step the user names. Confirm once before rolling back.
- **Ambiguous reply** (could be either acknowledgement or partial approval) → ask one clarifying question before acting. Do not silently advance.

**The Phase 4 publish gate is the one phase-transition that does not use natural language.** Publish writes to Linear and is hard to undo, so it uses `AskUserQuestion` with explicit `Publish` / `Revisit` / `Other` options.

`AskUserQuestion` may also appear mid-phase for stakeful binary tool-dispatch decisions (the Phase 1 optional deep exploration trigger is the only current example). It is never used for phase-end approval — those always read a natural-language reply.

---

## External dependencies

The only Skill this skill invokes via the `Skill` tool is **`linear-issue`** in Phase 3 (Draft). Everything else needed by this skill — grilling style, vertical-slice rules, shape format, phase-transition rules — is internalised in this file and the `phases/` files.

Linear read/write operations (project read in Phase 1, milestone + issue create in Phase 4) use whichever Linear interface is available in the workspace: a `linear` command-line tool, a Linear MCP server, or equivalent. The phase files describe Linear operations in terms of intent (read project, create milestone, create issue) — use whatever tool is actually available; don't depend on a specific binary.

**Agent tool (Phase 1 Step 4 only).** The optional deep code exploration step in Phase 1 dispatches a subagent via the `Agent` tool (`subagent_type: feature-dev:code-explorer`, `model: sonnet`). If the runtime platform does not expose an `Agent`-equivalent (or the `feature-dev:code-explorer` subagent is unavailable), present the AskUserQuestion option only as `No — skip` and proceed directly to the Open decisions gate (Step 5). Do not block the skill on the absence of this optional capability.

---

## Session recovery

This skill is **stateless across sessions**. If a session is interrupted before Phase 4 publish completes, restart from Phase 1. Linear publish (Phase 4) is the only durable artefact; after publish, Linear is the source of truth.

Pre-Phase-4 work lives only in chat. The user is responsible for staying in one session or accepting restart.

---

## Conversational style

Pragmatic. The user (project lead) knows the codebase and the team. This skill saves them time by digesting the Linear project, asking sharp questions, structuring the plan, and writing it to Linear.

- Save time — Linear already holds the project description; the skill reads it in and uses it as input, without making the user re-read a restatement.
- Sharp questions — single Q at a time, recommended answer + 1-2 alternatives + reasoning. User answers with one word or pushes back.
- Structure their thinking — Phase 2 makes the plan visible (milestone outline, then per-milestone issue trees, then a locked sign-off view).
- Challenge gently — "That milestone is heavy, split at X?" not "I think you should consider splitting at X."
- Defer to judgment — on technical approach, estimation, sequencing, the lead's call.
