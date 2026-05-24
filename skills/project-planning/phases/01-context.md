# Phase 1 — Context

Read project, load shared references, gather code evidence (grep + optional deep exploration), and gate on any unresolved Open product decisions. Phase 2 (Shape) consumes the Linear project description + chat-context evidence directly — **this phase does not produce a brief artefact**. Linear already holds the project description; re-stating it in chat is redundant work.

## Entry conditions

- User invoked the skill (matched trigger phrase or asked for a plan).
- Linear project URL / name / ID identified.

If no Linear project identified, ask once:

> "Which Linear project should I plan? Give me the project name, URL, or ID."

Do not proceed without a valid Linear project.

---

## Steps

### 1. Load must-read Caruso shared references

Locate and read the three Caruso shared references by name (they live in the workspace's Caruso skill set, typically in a `shared-references/` directory):

- `caruso-code-repositories`
- `caruso-data-model-overview`
- `caruso-terminology`

If the location is not obvious, search the workspace for `shared-references/caruso-code-repositories.md` and use the directory of the first hit. Skip if already loaded earlier in the session.

### 2. Fetch the Linear project

Read the Linear project description and Decision Register using the Linear interface available in the workspace (Linear CLI command, Linear MCP server, or equivalent). Extract: project name, description, Decision Register table (if present), and any `Open` decisions.

For very long descriptions, fetch and read the full text but print only the highest-signal excerpts in chat (project name, intent, key entities, Decision Register rows). The model holds the full description in working context across this session; downstream phases re-read or re-fetch as needed.

**Scope edits to the Linear project description are only allowed during Phase 1 (any step).** If a scope edit is needed, propose the surgical change in chat (show old → new wording), get explicit user approval, then write back via the available Linear interface. Once Phase 1 exits, the description is frozen for the rest of the session — downstream phases work with it as-is.

### 3. Grep-verify technical surface area

For every repo / service mentioned in the project description, run a quick `grep` / `find` to verify the entity / endpoint exists. Repo selection is **evidence-based** — every repo the model treats as in-scope must have at least one grep-confirmed code path.

Report each verification as a one-liner in chat:

```
- <repo-name> — verified: <path/to/file:line> (what's there)
- <repo-name> — verified: <path/to/file:line> (what's there)
- <repo-name> — NOT FOUND locally; surface to user
```

If a repo cannot be located locally (not cloned to the workspace), state so explicitly and ask the user how to handle it before proceeding.

The grep evidence stays in chat context. Phase 2 / 3 / 4 reference it directly.

### 4. Optional deep code exploration

Offer the user a deeper pass via a Sonnet-powered code-explorer subagent. Use `AskUserQuestion`:

- **Header**: `Deep exploration`
- **Question**: `"Dispatch a code-explorer subagent (Sonnet) to deep-dive the involved codebases before milestone shaping? Typically 1-2 min; findings inform downstream Phase 2 / Phase 3 decisions."`
- **Options**:
  - `Yes — run exploration` — dispatch the subagent and wait for it to return.
  - `No — skip` — proceed directly to Step 5.

**If Yes**, invoke the `Agent` tool with:

- `subagent_type: feature-dev:code-explorer`
- `model: sonnet`
- `description`: a short label such as `Phase 1 code exploration`.
- `prompt`: assemble in the template below — interpolate placeholders before invoking. The `<one-paragraph summary>` is composed on the fly by you from the Linear description Step 2 fetched (Step 2 does not pre-produce a summary). The verified surface area list comes from Step 3. Do not pass the template verbatim with `<...>` still in place.

  ```
  Context: <one-paragraph summary of the Linear project from Step 2 — what's changing, primary entities>.

  Verified technical surface area (from Step 3 grep):
  - <repo-name> — <path/file:line — what's there>
  - <repo-name> — <path/file:line — what's there>

  Task: Deep-explore these codebases to ground downstream planning.
  Focus on the listed repos, but follow imports / RPC / event consumers into adjacent repos when tracing data flow.

  Usage of your findings: They stay in chat context to inform:
  (a) Phase 2 milestone shaping — where to draw milestone boundaries, what vertical slices to cut per milestone, whether a `Foundations` or `Cleanup` milestone is warranted, and which files siblings must avoid overlapping;
  (b) Phase 3 issue body drafting — technical context per issue, story-point grounding, and references to prior implementations.

  Angles to cover (hints, not a checklist):
  - Current state of the primary entities and which services own them.
  - Cross-service data flow (gRPC / GraphQL / events) for the affected workflows.
  - Similar prior implementations that this project can extend or mirror.
  - Shared infrastructure (schema migration, RBAC permission, gRPC contract, event topic) that multiple upcoming features will depend on.
  - Legacy code or flows this project may retire.

  Return structured findings (markdown, headed sections per repo + a cross-cutting summary). Cite paths as `repo/path/to/file.go:LINE` where useful.
  ```

The subagent's full response stays in chat context. Do **not** edit it; later steps re-read it. If the subagent fails or times out, treat it as if the user had chosen `No` — surface a single-line note in chat (`Code exploration unavailable — proceeding without it.`) and continue to Step 5.

**If No**, proceed directly to Step 5.

### 5. Open decisions gate (conditional)

This step exists **only when Step 2 surfaced one or more `Open` decisions in the Decision Register**. If the register has no Open rows, skip this step entirely — Phase 1 ends after Step 4 and the skill advances to Phase 2 without any further user touchpoint.

When Open decisions exist, surface them in one chat response:

```
## Open product decisions to resolve before shaping

- [ ] <decision 1 — concise summary of the question>
- [ ] <decision 2 — ...>
```

End with one open question:

> "Resolve these inline (give me the answers), or say `flag all` to mark them as PM follow-up so we can proceed without blocking?"

**Loop rules**:

- **User resolves one or more inline** → record the resolution in chat (e.g., `Decision 1 resolved: <user's answer>`), re-present any remaining Open items, repeat until all are resolved or flagged.
- **`flag all` / `flag back to PM` / "I can't decide these"** → mark every remaining Open decision as `PM follow-up required` in a chat record. Phase 4 Block 5 reads this list. Accept advancement.
- **User signals approval while Opens remain raw** → do not advance; re-prompt with "resolve each, or `flag all` to bulk-flag".

---

## Exit conditions

- Step 3 grep verification complete (every in-scope repo has a verified path or is explicitly flagged not-found-locally to the user).
- Step 4 either ran or was skipped.
- Step 5 either skipped (no Opens) or every Open decision resolved/flagged.
- Advance to `phases/02-shape.md`.
