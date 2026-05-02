# 09 — Runbook: Bootstrap a New Project

Step-by-step guide for adopting this framework on a real codebase using GitHub Copilot. Plan for ~2-4 hours of focused work.

## Prerequisites

- **GitHub Copilot subscription** (Individual / Business / Enterprise) and signed in
- **Copilot extension** installed in your IDE (VS Code / Visual Studio / JetBrains / Eclipse / Xcode)
- **Recent extension version** that supports custom agents (`.github/agents/*.md`)
- **Git** installed; **GitHub CLI** (`gh`) optional for PR creation
- **Read access** to this framework at `~/Desktop/github-copilot-orchestration-framework/`
- **2-4 hours** of focused time

## Phase 0 — Decide if you need this framework (10 min)

Skip this framework if:
- Your project is a solo prototype
- Your project has fewer than 4 distinct domain areas
- Your team uses Copilot only for inline completions (not Chat or Cloud Agent)
- Your project is < 5,000 lines of code

Use this framework if:
- Your project has multiple roles
- Multiple data systems are in play
- Real compliance/security/payments concerns exist
- A team uses Copilot daily on the same codebase
- You've experienced cascading hallucinations or context drift

## Phase 1 — Inventory your project (30 min)

Open your IDE in the project root. Open Copilot Chat.

Paste `prompts/INVENTORY-PROMPT.md` (from this framework). Copilot will:
- Scan your codebase structure
- Propose a list of specialists based on your domain boundaries
- Surface clutter that should be archived
- Identify your protected branches
- Identify your tech stack and which MCP servers will be useful

You answer: confirm/adjust the proposed specialist list.

**Common gotcha:** Copilot may propose 10-15 specialists for a medium project. That's too many. Aim for 5-8.

## Phase 2 — Bootstrap the framework files (45 min)

In the same Copilot Chat session, paste `prompts/BOOTSTRAP-PROMPT.md`. Reference this framework's path so Copilot can read templates:

```
The framework lives at /Users/<you>/Desktop/github-copilot-orchestration-framework/.
Read templates from there. Customize for this project: <project-name>.
```

Copilot will:
1. Create `.github/agents/<project>-orchestrator.md` from the orchestrator template
2. Create one `.github/agents/<specialist>.md` per specialist
3. Create `.github/instructions/<domain>.instructions.md` files (with `applyTo:` globs)
4. Create `.github/prompts/investigate-bug.prompt.md` and `.github/prompts/build-feature.prompt.md` from skill templates
5. Create `docs/ai-context/HANDOFF_SCHEMA.md`
6. Create `docs/ai-context/INDEX.md` with task → docs map
7. Create `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md`
8. Create skeleton `docs/ai-context/<area>-experience.md` files
9. Create `.github/copilot-instructions.md` (the router)
10. Create `docs/_archive/README.md`
11. Update `.gitignore`

Verify each file before saving.

## Phase 3 — Add your first instructions (30 min)

The hardest part of this framework is writing useful instruction files. They take real domain knowledge.

In the Copilot Chat session, ask:

```
Based on the codebase scan, propose 3-5 instruction files for .github/instructions/.
For each, list:
  - applyTo: glob list (verify the globs match real files in this project)
  - 2-3 hard rules with WHY + HOW TO APPLY
  - excludeAgent: if relevant

Do not propose any rule that's just style preference — only invariants where
breaking them caused or could cause a real production issue. Cite file:line
where the gotcha currently lives.

Wait for my approval before creating any files.
```

Common instruction files to start with:
- `backend-api.instructions.md` (or your equivalent server-side concern)
- `frontend-ui.instructions.md` (or `mobile-ui.instructions.md`)
- `database.instructions.md`
- `auth-security.instructions.md`
- `testing.instructions.md`

Aim for **3-5 rules per file** at first.

## Phase 4 — Add 1-2 starter prompt files (20 min)

Most projects benefit from at least these:
- `.github/prompts/investigate-bug.prompt.md`
- `.github/prompts/build-feature.prompt.md`

Ask Copilot:

```
Create .github/prompts/investigate-bug.prompt.md and
.github/prompts/build-feature.prompt.md based on
templates/prompt.md.template, customized for this project's
tech stack and roles.

These should be invokable via /investigate-bug and /build-feature in Chat.
```

You can add more prompt files later (qa-flow, compliance-review, audit-pipeline, context-refactor).

## Phase 5 — Verify (15 min)

Run these checks:

```bash
# 1. Files in expected locations
ls .github/agents/
ls .github/instructions/
ls .github/prompts/
ls docs/ai-context/

# 2. Doc-link sweep — find broken refs
grep -rohE "(docs|\.github)/[A-Za-z0-9_./-]+\.md" .github/copilot-instructions.md docs/ .github/agents/ .github/instructions/ .github/prompts/ 2>/dev/null | sort -u | while read p; do [ -f "$p" ] || echo "BROKEN: $p"; done

# 3. Build still passes (whatever your build command is)
npm run build  # or: pytest, go build, cargo build, etc.
```

In your IDE:
4. **Reload the IDE** so Copilot picks up the new `.github/` files.
5. Open Copilot Chat → click the agent dropdown → confirm your specialists appear in the list.
6. Type `/` in Chat → confirm your prompt files appear in the picker.

Fix any breakage before moving on. Common issues:
- Agent file YAML invalid (it won't appear in the dropdown)
- `applyTo:` glob points to a path that doesn't match anything (instruction won't load)
- Markdown link in `.github/copilot-instructions.md` to a file that doesn't exist (typo)

## Phase 6 — Run a real task through the orchestrator (30 min)

Pick a small real task — ideally a bug fix or a small feature.

Open Copilot Chat. Select your `<project>-orchestrator` agent from the agent dropdown (or type `@<project>-orchestrator` at the start of your message).

Verify the active agent shown in Chat is your orchestrator.

Give it the real task. Watch carefully:
- Does the orchestrator emit the outbound `handoff:` YAML block when it proposes to delegate?
- When you accept the delegation (or invoke the specialist), does the specialist return the inbound `return:` YAML block?
- Does the orchestrator catch any rejected_claims correctly?
- Does verification (tests, build) actually run?

If anything is off → tighten the relevant agent's body. Common fixes:
- Orchestrator skipping the YAML block → tighten its "outbound discipline" section
- Specialist accepting vague delegations → tighten its "incoming handoff validation" section
- Specialist not running tests → add explicit "tests_run MUST be non-empty" to its Definition of Done

## Phase 7 — Document for your team (20 min)

Add to your project's existing onboarding docs:
- "We use GitHub Copilot with a custom orchestration setup. See `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md`."
- "Always pick the right invocation mode — see `docs/06-INVOCATION-MODES.md` (or the spoonfeeder's invocation modes section)."
- "Don't push to main without project-owner approval."

Optional: announce in your team chat with a 30-second demo of `@<project>-orchestrator` handling a multi-step task.

## Phase 8 — Add hardening as needed (optional, after 2 weeks)

After running real tasks for a couple weeks, decide if you need:

### Tighter `tools:` allowlists

If a specialist is doing things outside its domain (e.g. running terminal commands when it shouldn't), tighten its `tools:` array.

### Organization-level custom instructions (Business / Enterprise)

If you have multiple repos using this framework, define shared cross-repo conventions at the organization level (UI at github.com org settings + `.github-private/agents/` for shared agents).

### CI check for handoff schema compliance

If specialists keep emitting malformed return blocks, add a CI script that grep's PRs for the schema and warns when missing. (Custom CI, not a Copilot feature.)

### `excludeAgent: "code-review"` on noisy instructions

If code review is over-applying instructions (e.g. flagging style preferences), add `excludeAgent: "code-review"` to those instruction files.

## Phase 9 — Quarterly maintenance

Every 3 months, dedicate a session to:
1. Run the `context-refactor` prompt (or have the `context-librarian` specialist do a sweep)
2. Move stale dated reports to `docs/_archive/<YYYY-MM>/`
3. Reconcile any instruction conflicts
4. Update orientation maps for areas where reality has drifted
5. Verify all agents still appear in the IDE Chat dropdown after dependency / IDE updates

## Common time-sinks to avoid

- **Don't try to make every instruction file perfect on day 1.** Ship 5; add 1 per month as production teaches you.
- **Don't over-specialize.** 6-8 specialists is plenty.
- **Don't write canonical docs from scratch in this framework.** Use what you already have.
- **Don't manually merge agent files when adding a rule.** Always edit the instruction file.
- **Don't forget to reload the IDE after `.github/` changes.** New agents won't appear until reload.

## When to stop and ask

- If specialists don't appear in the IDE Chat dropdown → YAML frontmatter syntax error
- If instruction files don't auto-load → check `applyTo:` glob matches the file you're editing
- If specialist outputs don't include the return YAML block → the specialist's body needs the "Return schema" section
- If the orchestrator keeps delegating in a loop → check `failure_condition` is articulated
- If a specialist refuses a delegation → check the orchestrator's outbound block has all required fields

## End state

You have:
- A `<project>-orchestrator` and 4-10 specialists in `.github/agents/`
- 3-5 path-globbed instruction files in `.github/instructions/`
- 2-4 prompt files (slash commands) in `.github/prompts/`
- Three-tier docs (orientation maps + canonical refs + archive)
- A team that knows how to pick invocation modes
- Auditable handoffs for every cross-domain task

Time invested: 2-4 hours setup + ~30 min/quarter maintenance.

Time saved: every cascading-hallucination incident you DIDN'T have. Every cross-domain task that got the right specialists without you thinking about it. Every new engineer who could navigate the codebase without 2 weeks of pairing.
