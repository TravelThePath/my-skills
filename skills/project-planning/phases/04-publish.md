# Phase 4 — Publish

Pre-publish review (multi-block materialisation in chat), `AskUserQuestion` gate, then publish to Linear.

## Entry conditions

- Phase 3 drafts complete; all milestones have full issue bodies in chat.

---

## Steps

### 1. Pre-publish review

Materialise the **complete plan** in chat. This is the only checkpoint where the whole plan appears together — there's no SSOT file behind it. The user uses this view to catch anything before Linear writes happen.

Produce 6 blocks in one response, in this order:

#### Block 1 — Project recap

Synthesize the project recap from two sources:

- **Step 4 explorer findings** (if Step 4 ran) — the deepest source; code-level context, data flows, prior implementations, services involved.
- **Linear project description content** (if it has substance) — pull goal, scope, and intent. Linear descriptions are free-form and may be sparse or empty; treat as optional supplement rather than required input.

**Length is adaptive** — typically 2-4 paragraphs for a multi-service project, shorter for trivial scopes. Write only as much as a returning reviewer needs to re-engage with what this plan is for before reviewing milestones / issues / settings. If both sources are thin, write a brief stub rather than padding.

Conclude with a bulleted list of verified repos from Phase 1 Step 3 (technical surface area).

This is the only block that synthesizes prose; it is generated at publish time from chat context, not stored anywhere upstream.

#### Block 2 — Milestones

| Milestone | Title | Locked issues | Total points |
|---|---|---|---|
| M1 | ... | ... | ... |
| ... | ... | ... | ... |
| Project total | — | <T> | <total> |

#### Block 3 — Issues by milestone

For each milestone, list every issue with: Title, Type (AFK/HITL), Blocked by, Points. Do **not** repeat the full Linear issue body here — it already lives in the Phase 3 drafts further up the chat.

```
M1 — <title> (<count> issues, <pts> pts)
  I1.1 [AFK] Investment Service | <verb> <object> — <pts>, blocked by: none
  I1.2 [AFK] ...
  ...

M2 — <title> (<count> issues, <pts> pts)
  I2.1 [AFK] ...
  I2.2 [HITL] ...
```

#### Block 4 — Linear publish settings

State explicitly what will be written:

```
State: Todo
Labels: AI (always) + per-issue layer labels (Front-end / Back-end), inferred at write time from each issue's body — see step 2b
Estimate: per Phase 3
Milestone: per shape
Priority: NOT set (default)
Blocked-by relations: NOT set (default)

Override: say "set priority" or "add blocking" before approving.
```

#### Block 5 — Risks & open Qs (residual)

Pull from two sources:
- The Phase 1 Step 5 chat record — every Open decision that ended marked `PM follow-up required` must be visible here. If Step 5 didn't run (Decision Register had no Open rows), this source is empty.
- Any risk or unresolved question raised during Phase 2 shape or Phase 3 drafts that didn't get resolved.

Write "None" only if both sources are genuinely empty.

#### Block 6 — Gate

Use `AskUserQuestion` here — publishing writes to Linear and is hard to undo, so the gate is an explicit click, not a free-text reply.

- Header: `Publish`
- Question: "Publish this plan to Linear?"
- Options:
  - **Publish** — Write milestones + issues to Linear with the settings above
  - **Revisit** — Roll back to a specific phase

If the user picks `Other`, treat it as targeted feedback and decide: roll back to a phase or patch in place.

---

### 2. Publish to Linear

Only after the `AskUserQuestion` gate returns `Publish`.

#### 2a. Create milestones (parallel)

For each milestone, call the Linear interface available in the workspace (CLI command, MCP, or equivalent) with:

- Linear project slug or ID (from Phase 1)
- Milestone name in the form `M<N> — <title>`
- Description: 2-3 sentences summarising what this milestone delivers — derive from the milestone's title plus the slice of the Linear project description that describes this deliverable. Append total points and issue count.

Example using a Linear CLI command (use your actual tool's equivalent syntax):

```bash
linear milestone create \
  --project <slug> \
  --name "M<N> — <title>" \
  --description "<summary + total points + issue count>"
```

Run all milestones in a single parallel batch — they are independent and milestone creation is fast. Capture each milestone UUID from the response and record in chat (needed if user later asks to add blocking relations).

#### 2b. Create issues (per milestone, parallel within milestone)

For each issue:

1. Extract the Linear issue body from the Phase 3 chat draft — everything under the `#### Linear issue (uploaded in Phase 4)` heading for that issue, verbatim. Write it to `/tmp/i<N.M>_body.md` via the `Write` tool. Do **not** include the plan-metadata block (Type / Blocked by / Story points / Estimate rationale) — those values are extracted separately and applied to Linear fields, not the body.
2. **Infer layer labels** from the Linear project description, Phase 1 Step 3 verified repos, and the issue's body content — an issue can have any combination of:
   - `Front-end` — touches voyager / admin-app / web-app
   - `Back-end` — touches a Go service (investment-service, ranger-api, entity-service, etc.)
   - both, if the issue spans layers

   These labels must already exist in the workspace (like `AI`). If issue creation fails because a label is missing, ask the user to create it once, then retry — do not silently drop the label.
3. Call the Linear interface (CLI command, MCP, or equivalent) to create the issue with:
   - Team: `APP` (Caruso's single Linear team — do not change without explicit user instruction)
   - Project: project slug or ID from Phase 1
   - Milestone: the matching `M<N> — <title>` from step 2a
   - Title: full title from Phase 3
   - Description: contents of the tempfile created in step 1
   - Estimate: story points from Phase 3
   - State: `Todo`
   - Labels: `AI` (always) + any inferred layer labels from step 2

   Example using a Linear CLI command (use your actual tool's equivalent syntax):

   ```bash
   linear issue create \
     --team APP \
     --project <slug> \
     --milestone "M<N> — <title>" \
     --title "<full title from Phase 3>" \
     --description-file /tmp/i<N.M>_body.md \
     --estimate <pts> \
     --label AI \
     --label "Front-end" \   # only if inferred
     --label "Back-end" \    # only if inferred
     --state Todo
   ```
4. Capture the returned issue identifier (e.g. `APP-XXXXX`).

Run issues in parallel within a milestone. Order across milestones does not matter when blocked-by is off (default).

Clean up tempfiles after all writes succeed — remove only the files this run created, not a broad `/tmp/i*` glob that could hit another session's tempfiles:

```bash
rm /tmp/i1.1_body.md /tmp/i1.2_body.md   # … list the exact files written in step 1
```

#### 2c. Report results

Output a final summary in chat:

```
Published.

M<N> — <title> (UUID: <id>)
  APP-XXXX  I<N>.1  <title>  · <pts> pts
  APP-XXXX  I<N>.2  <title>  · <pts> pts
  ...

Total: <T> issues, <total> pts.
```

---

## Overrides

If the user said any of these before approving:

- **"set priority"** → before publish, ask priority (1-4) for each issue or each milestone; pass the priority value to your Linear interface when creating each issue.
- **"add blocking"** → after all issues are created, make a second pass to set `blocked-by` relations using the Phase 2 / 3 dep info. This requires the real issue identifiers returned from the first pass, which is why it runs separately.

---

## Loop rules in this phase

- `AskUserQuestion` returns **Publish** → execute publish steps.
- `AskUserQuestion` returns **Revisit** → ask which phase to roll back to.
- `AskUserQuestion` returns **Other** → treat as feedback; decide patch in place or roll back.
- Publish failure (Linear API error) → report exact error, do not retry silently, ask user how to proceed.

---

## Exit conditions

- All milestones + issues live in Linear.
- Final summary printed in chat.
- Linear is now the source of truth. Skill is done.
