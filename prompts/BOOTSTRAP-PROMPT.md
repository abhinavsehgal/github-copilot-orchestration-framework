# BOOTSTRAP PROMPT — GitHub Copilot version

Paste this AFTER you have completed the INVENTORY-PROMPT pass and approved the proposed specialists. Adjust `<framework path>` to match your install location.

---

I've reviewed the inventory. Now bootstrap the GitHub Copilot Orchestration Framework on this project. Use the templates from `<framework path>` (default: `/Users/<you>/Desktop/github-copilot-orchestration-framework/templates/`). Read those template files before generating anything.

---

## ⚠ PRE-FLIGHT SAFETY CHECKS (run BEFORE creating anything)

This project may already have some Copilot configuration (e.g. from running VS Code's `/init`, or from prior team work). Bootstrap is designed to coexist with existing config, but ONLY if you detect collisions first. Run these checks in order. Do NOT skip even if the inventory said the project was greenfield.

### Pre-flight 1 — Auto-snapshot (mandatory)

Before any write, create a backup of any existing Copilot config:

```bash
mkdir -p .github-pre-bootstrap-backup
[ -f .github/copilot-instructions.md ] && cp .github/copilot-instructions.md .github-pre-bootstrap-backup/
[ -d .github/agents ] && cp -r .github/agents .github-pre-bootstrap-backup/
[ -d .github/instructions ] && cp -r .github/instructions .github-pre-bootstrap-backup/
[ -d .github/prompts ] && cp -r .github/prompts .github-pre-bootstrap-backup/
[ -d .github/chatmodes ] && cp -r .github/chatmodes .github-pre-bootstrap-backup/
[ -f AGENTS.md ] && cp AGENTS.md .github-pre-bootstrap-backup/
ls -la .github-pre-bootstrap-backup/
```

Report what was backed up. If the backup directory is empty, the project had no prior Copilot config — proceed without further pre-flight concerns.

If the backup directory has content, continue with pre-flight 2-5.

### Pre-flight 2 — Naming collision check (per file)

For EACH file you plan to create, check if the destination path already exists:

```bash
# For each proposed agent
for agent in <project-slug>-orchestrator <specialist-1> <specialist-2> <...>; do
  for f in ".github/agents/$agent.agent.md" ".github/agents/$agent.md" ".github/chatmodes/$agent.chatmode.md"; do
    [ -f "$f" ] && echo "COLLISION: $f already exists"
  done
done

# For each proposed instruction file
for inst in <name-1> <name-2> <...>; do
  if [ -f ".github/instructions/$inst.instructions.md" ]; then
    echo "COLLISION: .github/instructions/$inst.instructions.md already exists"
  fi
done

# For each proposed prompt file
for prompt in investigate-bug build-feature <...>; do
  if [ -f ".github/prompts/$prompt.prompt.md" ]; then
    echo "COLLISION: .github/prompts/$prompt.prompt.md already exists"
  fi
done
```

For EACH collision found, STOP and ASK:
> `<NEEDS USER CONFIRMATION: .github/agents/<name>.md already exists. Options: (a) overwrite (existing content goes to backup), (b) skip this agent, (c) merge content. Which?>`

Do NOT silently overwrite ANY collision. If the user picks (c) merge, propose the merged content and get explicit approval before writing.

### Pre-flight 3 — `applyTo:` glob conflict check

For each new instruction file you plan to create, check if existing instruction files have an OVERLAPPING `applyTo:` glob:

```bash
# Read all existing instruction file frontmatters
for f in .github/instructions/*.instructions.md; do
  [ -f "$f" ] && echo "=== $f ===" && head -5 "$f" | grep -E "^applyTo:"
done
```

If your proposed `applyTo: "src/api/**/*.ts"` overlaps with an existing instruction's glob, BOTH instructions will load when editing matching files — leading to potentially contradictory rules. STOP and ASK:
> `<NEEDS USER CONFIRMATION: My proposed .github/instructions/api.instructions.md (applyTo: "src/api/**/*.ts") overlaps with existing .github/instructions/server.instructions.md (applyTo: "src/api/**,src/server/**"). Options: (a) merge into the existing file, (b) tighten my proposed glob to a non-overlapping subset, (c) accept the overlap (both will load). Which?>`

### Pre-flight 4 — Drift detection on existing `.github/copilot-instructions.md`

If `.github/copilot-instructions.md` exists, READ IT and compare against current project reality:

```bash
# Re-read current tech stack from manifest
cat package.json 2>/dev/null | head -30
cat Cargo.toml 2>/dev/null | head -30
cat go.mod 2>/dev/null | head -10
cat pyproject.toml 2>/dev/null | head -30

# Compare to what existing copilot-instructions.md says
cat .github/copilot-instructions.md
```

Look for stale claims in the existing file:
- Tech stack mentions that don't match current dependencies (e.g. file says "Vue 3" but package.json has React)
- Directory paths that don't exist anymore (e.g. file says `src/components/` but the project moved to `app/components/`)
- Conventions referencing removed features (e.g. mentions of `/api/v1/legacy/` routes that were deleted)

For EACH stale entry found, STOP and ASK:
> `<NEEDS USER CONFIRMATION: Existing .github/copilot-instructions.md says "<stale claim>" but current reality is "<actual>". Should I drop / update / preserve as-is?>`

Do NOT silently drop content. The existing rules may be valuable context the team added intentionally.

### Pre-flight 5 — Existing chatmodes / agents differ in style

If `.github/chatmodes/` or `.github/agents/` already has files, check whether they follow a similar persona/contract style. Two parallel conventions in the same repo confuses the team.

```bash
ls .github/chatmodes/ 2>/dev/null
ls .github/agents/ 2>/dev/null
```

If existing agents exist with persona contracts that overlap with what you'd create, STOP and ASK:
> `<NEEDS USER CONFIRMATION: Existing agent .github/agents/code-reviewer.md exists with its own persona. Should the new <project-slug>-orchestrator coordinate WITH it (treat it as another specialist), REPLACE it, or IGNORE it? If you have invested in the existing agent system, replacement risks losing team knowledge.>`

### Decision gate — STOP if ANY pre-flight check raised an issue

If pre-flight 1-5 raised even one `<NEEDS USER CONFIRMATION>` flag, STOP and present ALL flags as a numbered list. Wait for the user to answer EVERY flag before proceeding to Step 1 below. Do not partially proceed.

If pre-flight 1-5 raised zero issues (truly greenfield Copilot setup), proceed to Step 1.

---

## What to create (order matters)

### Step 1 — Create the directory skeleton

```
.github/agents/
.github/instructions/
.github/skills/
.github/prompts/
docs/ai-context/
docs/_archive/
```

(Do NOT create `.github/chatmodes/` — chat modes are retired. If pre-flight 5 found existing `.chatmode.md` files, propose renaming them to `.github/agents/<name>.agent.md` as a separate, user-approved step. `.github/hooks/` is optional and comes later — see `docs/10-MECHANICAL-ENFORCEMENT.md`.)

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

`.github/agents/<project-slug>-orchestrator.agent.md` — use `templates/orchestrator-agent.md.template`. Substitute:
- `<PROJECT_NAME>` — project's identity
- `<PROJECT_SLUG>` — slug
- `<PROTECTED_BRANCH>` — confirmed protected branch name from inventory
- `<PROJECT_SPECIFIC_ANTI_PATTERNS>` — 3-5 anti-patterns specific to this project

For the `tools:` field, propose a read-only set:
```yaml
tools: ['codebase', 'search', 'usages', 'problems', 'changes', 'fetch']
```

⚠ Verify the exact tool names supported by the user's IDE/Copilot version. The list above is a common default; some IDEs may use different identifiers. If unsure, ASK the user (`<NEEDS USER CONFIRMATION: My proposed tools array uses [codebase, search, usages, ...]. Are these the correct identifiers for your Copilot version?>`).

Fill `agents:` with the exact specialist names from Step 6 (VS Code subagent allowlist; ignored by the cloud agent — never `*`).

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

**If the file doesn't exist:** create from `templates/copilot-instructions.md.template`. Substitute placeholders.

**If the file EXISTS** (verified during pre-flight 1):

1. The original is already snapshotted in `.github-pre-bootstrap-backup/copilot-instructions.md`.
2. **Read the existing file in full.** Identify which sections are: (a) project-specific rules the team added intentionally, (b) auto-generated `/init` content that may be stale, (c) generic best-practice content the framework's template covers anyway.
3. Produce a PROPOSED MERGED version that:
   - Preserves ALL section (a) content (project-specific team rules)
   - Drops or updates section (b) content if pre-flight 4 (drift detection) flagged it as stale
   - Replaces section (c) content with the framework's structured router
4. **Show the user a 3-pane diff** before writing:
   - LEFT: existing file (from backup)
   - RIGHT: proposed merged file
   - MIDDLE: a per-section disposition list ("preserved", "updated for drift", "replaced by template router", "dropped — covered by template")
5. Wait for explicit user approval ("yes, write the merged file") before writing. If the user pushes back on any section, revise and re-show the diff.

⚠ **Keep this file UNDER 200 lines** post-merge. The docs say repository instructions must be no longer than about 2 pages; inline completions are latency-sensitive. The file should be a tight router.

⚠ **Front-load the most important rules** so code review and inline completions see them first (good practice — the old "first 4,000 characters" figure is no longer on the docs).

The template's golden rules 9–12 (corrections → instruction files, deferred work written to a backlog, production push freshens the docs, one name per concept) and the workflow step "read `docs/ai-context/PROJECT.md` §3 + skim `LEARNINGS.md` §D" are the v1.2.0 additions — keep them in the merged file; they point at the files Step 13 creates.

If the merged file would exceed 200 lines: propose moving sections to `.github/instructions/<NAME>.instructions.md` (path-globbed) or `docs/ai-context/<area>.md` (orientation map) instead of bloating the router. Get user approval per moved section.

### Step 12 — Update `.gitignore`

Append (don't overwrite) these entries:

```gitignore
# Bootstrap backup — keep local-only, do not commit
.github-pre-bootstrap-backup/

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

Adjust based on the project's actual tech stack. The `.github-pre-bootstrap-backup/` line is critical — that directory contains the original Copilot config from BEFORE the bootstrap and should NEVER be committed (it would create a confusing parallel set of files in the repo).

### Step 13 — Create the project-truth set (v1.2.0)

These files are what a FRESH agent with no transcript reads first (`docs/11-PROJECT-TRUTH-AND-LEARNINGS.md`). Generate them from the codebase with the same evidence discipline as the instruction files — every state claim date-stamped, every command copied from the real build config, never guessed:

- `docs/ai-context/PROJECT.md` from `<framework path>/templates/PROJECT.md.template`. Fill §2 (environments + **what the deploy pipeline does NOT do**), §3 (what is live where — if you cannot verify an environment, write `unknown`, never a guess), §6 (sources of truth: the canonical helper per concept, found by grep), §7 (commands, verified against the build config file).
- `docs/ai-context/LEARNINGS.md` from `<framework path>/templates/LEARNINGS.md.template`. On a brownfield repo, seed §A/§B from the git log and any existing post-mortems or ADRs; seed §D with the generic corrections already in the template; leave §E for the owner to fill.
- `docs/ai-context/GLOSSARY.md` from `<framework path>/templates/GLOSSARY.md.template` — list every domain concept you found called by more than one name, with the file:field evidence.
- `.github/skills/<project-slug>-engineering/SKILL.md` from `<framework path>/templates/engineering-playbook-skill.md.template` — an Agent Skill (frontmatter `name` + `description`), so it loads on the cloud agent and the CLI as well as in the IDE. The directory name must equal the `name:` field.
- One `docs/<AREA>_BACKLOG.md` from `<framework path>/templates/BACKLOG.md.template` for the largest area, seeded with any TODO/FIXME clusters you found (each with what / why / effort / revisit-when).

Show me each file before saving.

## Constraints

- **Do not commit anything.** Show me each file. I'll commit at the end.
- **Do not modify existing application code.** Only create/update files in `.github/`, `docs/ai-context/`, `docs/_archive/`, `docs/<AREA>_BACKLOG.md`, and `.gitignore`.
- **Pre-flight checks are mandatory** (see top of prompt). Do not skip even on apparent greenfield. If pre-flight raises ANY `<NEEDS USER CONFIRMATION>` flag, STOP and present all flags before any file write.
- **Snapshot first, write second.** Pre-flight 1 creates `.github-pre-bootstrap-backup/` — that backup is your safety net. If you have NOT created the backup, do not proceed to Step 1.
- **Never silently overwrite.** Any pre-existing file at a destination path you'd write requires explicit user approval (per pre-flight 2). "Merge" is not the default — ASK whether to overwrite, skip, or merge.
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
ls .github/skills/
ls .github/prompts/
ls docs/ai-context/

# 2. Doc-link sweep — find broken refs
grep -rohE "(docs|\.github)/[A-Za-z0-9_./-]+\.md" .github/copilot-instructions.md docs/ai-context/ .github/agents/ .github/instructions/ .github/skills/ .github/prompts/ 2>/dev/null | sort -u | while read p; do [ -f "$p" ] || echo "BROKEN: $p"; done

# 3. If existing copilot-instructions.md was merged: diff against backup
diff .github-pre-bootstrap-backup/copilot-instructions.md .github/copilot-instructions.md | head -100

# 4. If existing agents/instructions/prompts were preserved or merged: diff each
for f in .github-pre-bootstrap-backup/agents/*.md 2>/dev/null; do
  [ -f "$f" ] && [ -f ".github/agents/$(basename "$f")" ] && diff "$f" ".github/agents/$(basename "$f")" | head -30
done

# 5. Build still passes (use whatever the project's build command is)
<project's build command>
```

Then ask the user to:
6. **Reload their IDE** so Copilot picks up the new `.github/` files.
7. Open Copilot Chat → click the agent dropdown → confirm specialists appear.
8. Type `/` in Chat → confirm prompt files AND the `<project-slug>-engineering` skill appear in the picker.
9. Spot-check a previously-working flow to confirm no regression (e.g. existing slash command still works, existing instruction file still loads on its `applyTo:` paths).

Report:
- Number of agent files created (should match what the inventory proposed)
- Number of pre-existing files preserved / merged / overwritten (per user's pre-flight decisions)
- Any broken doc links (should be zero)
- Build pass/fail
- Any `<NEEDS USER CONFIRMATION>` items still pending
- Confirmation that `.github-pre-bootstrap-backup/` was created and contains the original files

After verification passes:
- I'll review the diff against `.github-pre-bootstrap-backup/`, commit, and push to a branch + open a PR.
- The backup directory should be gitignored (add `.github-pre-bootstrap-backup/` to `.gitignore` if not already).
- Once the PR is merged and verified in real use for ~1 week, the backup directory can be deleted locally.

---

(End of prompt.)
