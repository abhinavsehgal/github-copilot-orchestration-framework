# 06 — Invocation Modes

GitHub Copilot has more invocation modes than typical AI tools because it spans inline completions, Chat (in IDE and on github.com), Cloud Agent (autonomous), and CLI. Each surface has different rules for what gets loaded.

## The seven modes

### Mode 1: Inline completions (autocomplete)

> **Not re-verified on 2026-08-22.** Whether inline completions consult `.github/instructions/*.instructions.md` (as opposed to only `.github/copilot-instructions.md`) was not part of the v1.2/v1.1 verification pass and the two editions of this framework historically disagreed. Treat the bullet below as `documented-unverified` (Chapter 11/12) until you confirm it against the current Copilot docs for your IDE.


How: ghost text in the editor as you type.

**Loads:**
- `.github/copilot-instructions.md` (auto)
- `.github/instructions/<NAME>.instructions.md` matching the current file's path (auto)

**Does NOT load:**
- Custom agents (`.github/agents/`)
- Skills or prompt files
- Hooks

Use for: typing assistance. Keep `.github/copilot-instructions.md` short for fast inline completions.

### Mode 2: Chat in IDE (default)

How: Copilot Chat sidebar in VS Code / JetBrains / Visual Studio.

**Loads:**
- `.github/copilot-instructions.md` (auto)
- `.github/instructions/<NAME>.instructions.md` matching files in current chat context (auto)

**Available on demand:**
- Custom agents (selectable from agent dropdown or @-mention)
- Skills (`/<name>`, or auto-loaded by relevance)
- Prompt files (invokable via `/<name>`; IDE-only)
- Hooks from `.github/hooks/*.json` (VS Code also reads `.claude/settings.json` hooks) — Chapter 10

(Custom chat modes are retired — rename `.chatmode.md` files to `.github/agents/<name>.agent.md`, Pitfall 7.)

Use for: most interactive work. Combine `@<specialist>` + `/<workflow>` for power.

### Mode 3: Chat with a specific agent (specialist mode)

How: Select a specific agent from the agent dropdown, OR type `@<agent-name>` in chat.

**Loads:**
- The selected agent's `.github/agents/<name>.agent.md` body as system prompt
- The agent's `tools:` allowlist is enforced
- `.github/copilot-instructions.md` (still auto-loaded)
- `.github/instructions/` matching paths (still auto-loaded)

**Use for:** narrow domain work. Examples:
- `@frontend-ui` for UI session
- `@backend-api` for API work
- `@qa-functional` for QA pass with browser MCP
- `@legal-compliance` for review-only session

The agent's `tools:` allowlist physically restricts what it can do (e.g. `legal-compliance` literally cannot edit files because it has no edit tools).

### Mode 4: Chat with the orchestrator (multi-domain mode)

How: Select your `<project>-orchestrator` agent OR type `@<project>-orchestrator <task>` in chat.

**Loads:** the orchestrator's body as system prompt + everything from Mode 3.

**Use for:**
- Anything spanning more than one role
- Anything spanning more than one feature area
- Data integrity, payments, security/privacy, compliance work
- Bug investigations where root cause may cross layers
- Feature builds where acceptance criteria need writing before implementation
- Pre-release QA across multiple roles
- Anything where the scope is ambiguous

The orchestrator delegates to specialists by `@<specialist>` mentions in its responses (you may see it propose to invoke a specialist; confirm or paste the recommended next step). In VS Code the orchestrator can also invoke a specialist directly as a subagent, restricted to the names in its `agents:` frontmatter (Chapter 3).

### Mode 5: Cloud Agent (autonomous)

How: Assign a Copilot agent to a GitHub issue or PR. The cloud agent runs autonomously on github.com infrastructure, makes changes, and opens a PR for review.

**Loads:**
- `.github/copilot-instructions.md` (auto) and `AGENTS.md` (nearest in the directory tree wins)
- `.github/instructions/<NAME>.instructions.md` matching files it edits (auto)
- The custom agent assigned (from `.github/agents/<NAME>.agent.md`)
- Skills from `.github/skills/*/SKILL.md`
- Hooks from `.github/hooks/*.json` — and **only** from there (Chapter 10)
- For org/enterprise: org-level custom agents from `/agents/` in the org's `.github` or `.github-private` repository

**Does NOT load:** prompt files (IDE-only — convert to a skill).

**Strong context isolation:** the cloud agent runs in a fresh environment per task. No leakage from previous sessions. **Per-repository:** "Copilot cannot make changes across multiple repositories in one run" — one branch, one PR per task (Chapter 12 for cross-repo work).

Use for: well-defined, self-contained tasks where you'd otherwise hand off to a junior engineer. Not for ambiguous work that needs back-and-forth.

### Mode 6: Copilot CLI, headless (`copilot -p`) — the current CLI

How: the agentic `copilot` command in the terminal. Interactive by default; non-interactive with `copilot -p "<prompt>"`.

**Loads** (from the working directory): `.github/copilot-instructions.md`, instruction files, `AGENTS.md` / `CLAUDE.md`, custom agents, skills, and hooks (`.github/hooks/*.json` plus `~/.copilot/hooks/`). There is **no `--cwd` flag** — run it from the target directory (or worktree). `--add-dir=<path>` grants access to additional directories.

**Flags that matter for orchestration:**
- `--agent=<name>` — run as a specific custom agent (the orchestrator, for a scripted multi-domain task).
- `--allow-tool=<tool>` / `--deny-tool=<tool>` / `--allow-all-tools` — tool permissions for a non-interactive run. A headless run that needs to edit or run commands must be granted them explicitly; a review-only agent should be run with nothing beyond read tools allowed.
- `-s` / `--silent` — suppress non-output chatter for scripting.
- `--output-format` is **not documented** — parse the plain text output, or have the agent write a file.

**Use for:** CI jobs, scheduled sweeps, and scripted delegation (a workspace orchestrator calling a repo's orchestrator — Chapter 12). Because hooks load here, a `copilot -p` run is still governed by the build-gate / doc-freshness hooks the repository ships.

### Mode 7: `gh copilot suggest` / `explain` — the older CLI extension

How: `gh copilot suggest` / `gh copilot explain` via the GitHub CLI extension.

**Loads:** its own instruction system, separate from the repository's `.github/` customizations. It does not run agents, skills or hooks.

**Use for:** one-line shell command suggestions and explanations only. For anything agentic, use Mode 6.

## Decision tree

```
Is this an editor/typing assist task?
  YES → Inline completions (no choice — that's what's running)
  NO ↓

Is this a quick question about code (read-only)?
  YES → Plain Chat (no agent selected)
  NO ↓

Is this narrow single-domain work (one specialist's territory)?
  YES → Chat with that specialist agent (@<specialist>)
  NO ↓

Is this cross-domain work, ambiguous scope, or production-sensitive?
  YES → Chat with orchestrator (@<project>-orchestrator)
  NO ↓

Is this a well-defined, self-contained task you'd assign to a junior?
  YES → Assign issue to Cloud Agent
  NO ↓

Is this a scripted / CI / scheduled run of a defined workflow?
  YES → copilot -p "…" --agent=<agent> from the target directory
  NO ↓

Is this a one-line terminal question?
  YES → gh copilot suggest / explain
```

## Why we don't put the orchestrator persona in `.github/copilot-instructions.md`

The repository-wide instructions file is auto-loaded into:
- Inline completions (every keystroke!)
- Chat sessions (every prompt)
- Code review (PR diff comments)
- Cloud agent (every task)

Putting the orchestrator's full delegation persona there would:
- **Slow down inline completions** (every keystroke loads the persona)
- **Pollute code review** with delegation instructions ("delegate this fix to backend-api")
- **Force every interaction** through the orchestrator's "delegate, don't implement" pattern

Instead, `.github/copilot-instructions.md` is a **thin router** — golden rules + workflow + cross-links. The orchestrator persona is in `.github/agents/<project>-orchestrator.agent.md` and only loads when explicitly selected.

## Mode comparison table

| Mode | Loads orchestrator persona? | Specialists invokable? | Tool allowlist enforced? | Context isolation |
|---|---|---|---|---|
| Inline completions | No | No | N/A | n/a |
| Plain Chat | No | Yes (manual `@`) | Default Chat tools | Within Chat session |
| Specialist Chat | No | (just this one) | Yes (per agent's `tools:`) | Within Chat session |
| Orchestrator Chat | Yes | Yes (via orchestrator's `@` calls) | Yes (per agent's `tools:`) | Within Chat session |
| Cloud Agent | If assigned | If multi-agent | Yes (per agent's `tools:`) | Strong (fresh env) |
| Copilot CLI (`copilot -p`) | If `--agent=` | Yes (agents load) | Yes (`tools:` + `--allow-tool`/`--deny-tool`) | Fresh per run |
| `gh copilot suggest` | n/a (separate) | n/a | n/a | Separate context |

## Practical recipe

A well-functioning team uses several modes:

```
# Quick fix (90% of casual work)
Open Copilot Chat, no agent selected
"Update the button label from 'Sign In' to 'Sign in'"

# Specialist deep-dive (focused domain work)
Select @frontend-ui in Chat
"Audit our Suspense boundaries for client-component leaks"

# Full orchestrated work (multi-domain, production-sensitive)
Select @<project>-orchestrator in Chat
"Add a new payment method that requires KYC review and cron-driven activation"

# Autonomous bug fix (well-defined, junior-level)
Assign GitHub issue to Copilot
The cloud agent investigates, fixes, opens PR

# Scripted run (CI, cron, cross-repo delegation)
cd <repo>   # no --cwd flag — the CLI loads customizations from the working directory
copilot -p "Run /qa-flow on the checkout page and write findings to docs/qa/latest.md" --agent=qa-functional --allow-tool=<read/run tools as needed> -s
```

## Anti-patterns

❌ **Using plain Chat for everything.** Cross-domain work flooded with context. Specialists never invoked.

❌ **Using `@<orchestrator>` for typos.** Forced delegation overhead. Specialist spawn for a one-line fix.

❌ **Not knowing which mode you're in.** Check the agent dropdown in Chat. If you're not sure, you're probably in the wrong mode.

❌ **Putting the orchestrator persona in `.github/copilot-instructions.md`.** See section above — it pollutes everything.

❌ **Letting the Cloud Agent take ambiguous tasks.** Cloud Agent works best on tasks with clear scope and acceptance criteria. Ambiguity → bad PRs → human cleanup.

## When you change projects

Each project has its own `.github/agents/`, so `@<project>-orchestrator` is project-specific. The orchestrator name should be `<project>-orchestrator` (e.g. `acme-orchestrator`, not just `orchestrator`) so that when you have multiple repos open simultaneously, you can tell at a glance which orchestrator is active.

## Surface support matrix (current)

| Feature | VS Code | JetBrains | Visual Studio | github.com Chat | Cloud Agent | Code review | CLI (`copilot`) |
|---|---|---|---|---|---|---|---|
| `.github/copilot-instructions.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `.github/instructions/*.instructions.md` (auto-load on path match) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `AGENTS.md` (nearest wins) | ✅ root (nested behind `chat.useNestedAgentsMdFiles`) | not documented | not documented | not documented | ✅ | ✅ | ✅ |
| `.github/skills/*/SKILL.md` (agent skills) | ✅ | ✅ | not documented | not documented | ✅ | ✅ | ✅ |
| `.github/prompts/*.prompt.md` (slash commands, IDE-only) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `.github/agents/*.agent.md` (custom agents) | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ (`--agent=`) |
| `agents:` subagent allowlist | ✅ | not documented | not documented | ❌ | ❌ (ignored) | — | not documented |
| `.github/hooks/*.json` (lifecycle hooks) | ✅ (also `.claude/settings.json`) | not documented | not documented | not documented | ✅ (`.github/hooks` only) | not documented | ✅ (also `~/.copilot/hooks/`) |
| `.github/chatmodes/*.chatmode.md` | RETIRED — rename to `.agent.md` | RETIRED | RETIRED | ❌ | ❌ | ❌ | ❌ |
| MCP servers per agent | ✅ | ✅ | ✅ | partial | partial | — | ✅ |

(Verified against docs.github.com and code.visualstudio.com on 2026-08-22. "not documented" means the current docs neither confirm nor deny it — do not assume either way. Surface support evolves rapidly; re-verify quarterly — Pitfall 20.)
