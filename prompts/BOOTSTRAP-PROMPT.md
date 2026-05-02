# BOOTSTRAP PROMPT — GitHub Copilot version

Paste this AFTER you have completed the INVENTORY-PROMPT pass and approved the proposed specialists. Adjust `<framework path>` to match your install location.

---

I've reviewed the inventory. Now bootstrap the GitHub Copilot Orchestration Framework on this project. Use the templates from `<framework path>` (default: `/Users/<you>/Desktop/github-copilot-orchestration-framework/templates/`). Read those template files before generating anything.

## What to create (order matters)

### Step 1 — Create the directory skeleton

```
.github/agents/
.github/instructions/
.github/prompts/
docs/ai-context/
docs/_archive/
```

(`.github/chatmodes/` is OPTIONAL — only create if the project specifically needs the older chat-mode format alongside agents.)

### Step 2 — Create `docs/ai-context/HANDOFF_SCHEMA.md`

Use `templates/HANDOFF_SCHEMA.md.template`. Substitute:
- `<PROJECT_NAME>` — project's full name from inventory section 1
- `<PROJECT_SLUG>` — lowercase-hyphenated slug

Customize the worked examples to use realistic file paths from THIS project's codebase. Show me before saving.

### Step 3 — Create `docs/_archive/README.md`

Use `templates/archive-README.md.template`. No placeholder substitution needed. Show me before saving.

### Step 4 — Create `docs/ai-context/INDEX.md`

Use `templates/INDEX.md.template`. Populate the routing table from inventory section 3 (specialists) and section 5 (orientation candidates). Show me before saving.

### Step 5 — Create the orchestrator agent

`.github/agents/<project-slug>-orchestrator.md` — use `templates/orchestrator-agent.md.template`. Substitute:
- `<PROJECT_NAME>` — project's identity
- `<PROJECT_SLUG>` — slug
- `<PROTECTED_BRANCH>` — confirmed protected branch name from inventory
- `<PROJECT_SPECIFIC_ANTI_PATTERNS>` — 3-5 anti-patterns specific to this project

For the `tools:` field, propose a read-only set:
```yaml
tools: ['codebase', 'search', 'usages', 'problems', 'changes', 'fetch']
```

⚠ Verify the exact tool names supported by the user's IDE/Copilot version. The list above is a common default; some IDEs may use different identifiers. If unsure, ASK the user (`<NEEDS USER CONFIRMATION: My proposed tools array uses [codebase, search, usages, ...]. Are these the correct identifiers for your Copilot version?>`).

Show me the file before saving.

### Step 6 — Create each specialist agent

For each specialist from inventory section 3:
- If REVIEW-ONLY: use `templates/review-only-agent.md.template`
- If implementation: use `templates/specialist-agent.md.template`

Substitute placeholders. The `Incoming handoff validation` and `Return schema (required)` sections are IDENTICAL across every specialist — paste verbatim from the template.

For the `tools:` field:
- REVIEW-ONLY agents: read-only tools only (`['codebase', 'search', 'usages', 'problems', 'changes', 'fetch']` — adjust per IDE)
- Implementation agents: read tools + edit tools (`['codebase', 'search', 'usages', 'problems', 'changes', 'edit_files', 'apply_patch', 'runCommands']` — adjust per IDE)

For specialists that need MCP servers (e.g. browser MCP for QA), add the `mcp-servers:` block per Copilot docs.

Show me each file. Do them in this order so I can check the pattern, then approve subsequent ones in batch:
1. First implementation specialist: full review
2. First REVIEW-ONLY specialist: full review
3. Remaining specialists: batch review (just confirm correct frontmatter + correct cross-links)

### Step 7 — Create path-globbed instruction files

For each instruction file from inventory section 4:
- Create `.github/instructions/<NAME>.instructions.md`
- Use `templates/instructions.md.template`
- Substitute the actual `applyTo:` glob and the 3-5 rules

Verify each `applyTo:` glob matches real files (`find . -path '<glob>'` — should return matches).

Show me each file before saving.

### Step 8 — Create starter prompt files (slash commands)

Create:
- `.github/prompts/investigate-bug.prompt.md`
- `.github/prompts/build-feature.prompt.md`

Both based on `templates/prompt.md.template`, customized for this project's tech stack and roles.

Optionally create more (`/qa-flow`, `/compliance-review`, `/context-refactor`) if the project needs them.

Show me each file before saving.

### Step 9 — Create skeleton orientation maps

For each major domain in inventory section 5, create `docs/ai-context/<area>.md` with a 50-150 line skeleton:
- 2-sentence orientation
- "Key file paths" (extracted from codebase scan)
- "Top gotchas" (start with placeholders, mark `<TODO: fill from real bugs>`)
- "Cross-links" (link to canonical docs that already exist)

### Step 10 — Create `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md`

Use `templates/SPOONFEEDER.md.template`. Customize the invocation modes section with the project's specific orchestrator name and specialist list.

### Step 11 — Create / update `.github/copilot-instructions.md`

If the file doesn't exist, create from `templates/copilot-instructions.md.template`. If it exists, MERGE — preserve any existing content, add the framework's golden rules + workflow + routing tables.

⚠ **Keep this file UNDER 200 lines.** Code review reads only the first 4,000 characters; inline completions are latency-sensitive. The file should be a tight router.

Show me before saving.

### Step 12 — Update `.gitignore`

Append (don't overwrite) these entries:

```gitignore
# Tool caches that should never be committed
test-results/
logs/
coverage/

# IDE caches
.vscode/.history
.idea/workspace.xml

# Vercel/CLI env exports — broader pattern
.env*.bak*
.env*staging_tmp
.env*tmp
```

Adjust based on the project's actual tech stack.

## Constraints

- **Do not commit anything.** Show me each file. I'll commit at the end.
- **Do not modify existing application code.** Only create/update files in `.github/`, `docs/ai-context/`, `docs/_archive/`, and `.gitignore`.
- **Cite evidence for every gotcha.** If you can't find evidence, mark `<NEEDS USER CONFIRMATION>` rather than guessing.
- **Verify Copilot tool identifiers.** Before generating agent files with `tools:` arrays, confirm the tool names are valid for the user's Copilot extension version. ASK if unsure.
- **REVIEW-ONLY specialists must NOT have `memory:`** if Copilot adds that field — per general AI tooling pitfalls, memory often auto-grants Edit/Write. Currently this isn't a documented Copilot field, but check before adding.
- **Don't put the orchestrator persona in `.github/copilot-instructions.md`.** That file is auto-loaded for inline completions and code review — would pollute everything.

## After all files are created — verify

Run these commands and report results:

```bash
# 1. Files in expected locations
ls .github/agents/
ls .github/instructions/
ls .github/prompts/
ls docs/ai-context/

# 2. Doc-link sweep — find broken refs
grep -rohE "(docs|\.github)/[A-Za-z0-9_./-]+\.md" .github/copilot-instructions.md docs/ai-context/ .github/agents/ .github/instructions/ .github/prompts/ 2>/dev/null | sort -u | while read p; do [ -f "$p" ] || echo "BROKEN: $p"; done

# 3. Build still passes (use whatever the project's build command is)
<project's build command>
```

Then ask the user to:
4. **Reload their IDE** so Copilot picks up the new `.github/` files.
5. Open Copilot Chat → click the agent dropdown → confirm specialists appear.
6. Type `/` in Chat → confirm prompt files appear in the picker.

Report:
- Number of agent files created (should match what the inventory proposed)
- Any broken doc links (should be zero)
- Build pass/fail
- Any `<NEEDS USER CONFIRMATION>` items still pending

After verification passes: I'll review the diff, commit, and push to a branch + open a PR.

---

(End of prompt.)
