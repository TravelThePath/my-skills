# Learn — turn Fix items into durable rules

Goal: turn `Fix` items where the **agent wrote the bad code** into rules that prevent the same mistake from recurring. Rules land in the right config file — global or per-repo — so Step 2's convention-reading and this step's convention-writing stay symmetric.

## Flow

1. **List Fix items.** Show each `Fix` from this session — file, line, one-line Problem summary, and Evidence snippet. Do not include `Reply`/`Defer`/`Ask` items.

2. **Ask the user to curate.** Which items were agent-authored mistakes (vs pre-existing bugs or one-off edge cases)? Accept item numbers: `1 3` or `all`. Skip silently if none selected.

3. **Draft rules.** For each selected item, propose one rule entry for `~/.claude/CLAUDE.md`. Classify scope:

   - **General** — applies across repos (e.g., "always add `case <-ctx.Done()` to long-running select loops"). Place under the most fitting existing section.
   - **Project-specific** — conventions tied to one repo's contracts, naming schemes, or local abstractions. Place under a `## <repo-name>` section (create it if it doesn't exist), and open with a one-line scope note: `# Applies to <repo-name> only.`

   Never write to the repo's own `CLAUDE.md` or `AGENTS.md` — those are git-tracked and shared.

   Formatting rules:
   - Imperative phrasing: "Always include…", "Never shadow…", "Prefer X over Y when…".
   - One short comment only if the why isn't obvious from the rule itself.
   - Before drafting, grep `~/.claude/CLAUDE.md` for near-duplicates — if a similar rule exists, propose updating it rather than adding a new entry.

4. **Show and confirm.** Present proposed additions as a diff-style preview. Wait for the user to approve, edit, or skip each one.

5. **Append.** For each approved rule, append to `~/.claude/CLAUDE.md` under the correct section. Print `Saved N rule(s) to ~/.claude/CLAUDE.md.`

## Scope guard

Only concretely actionable rules belong here ("always add `case <-ctx.Done()` to long-running select loops"). Vague observations ("be more careful with errors") do not — skip them.
