# 06 — Invocation Modes

GitHub Copilot has more invocation modes than typical AI tools because it spans inline completions, Chat (in IDE and on github.com), Cloud Agent (autonomous), and CLI. Each surface has different rules for what gets loaded.

## The five modes

### Mode 1: Inline completions (autocomplete)

How: ghost text in the editor as you type.

**Loads:**
- `.github/copilot-instructions.md` (auto)
- `.github/instructions/<NAME>.instructions.md` matching the current file's path (auto)

**Does NOT load:**
- Custom agents (`.github/agents/`)
- Prompt files
- Custom chat modes

Use for: typing assistance. Keep `.github/copilot-instructions.md` short for fast inline completions.

### Mode 2: Chat in IDE (default)

How: Copilot Chat sidebar in VS Code / JetBrains / Visual Studio.

**Loads:**
- `.github/copilot-instructions.md` (auto)
- `.github/instructions/<NAME>.instructions.md` matching files in current chat context (auto)

**Available on demand:**
- Custom agents (selectable from agent dropdown or @-mention)
- Prompt files (invokable via `/<name>`)
- Custom chat modes (selectable from mode dropdown)

Use for: most interactive work. Combine `@<specialist>` + `/<workflow>` for power.

### Mode 3: Chat with a specific agent (specialist mode)

How: Select a specific agent from the agent dropdown, OR type `@<agent-name>` in chat.

**Loads:**
- The selected agent's `.github/agents/<name>.md` body as system prompt
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

The orchestrator delegates to specialists by `@<specialist>` mentions in its responses (you may see it propose to invoke a specialist; confirm or paste the recommended next step).

### Mode 5: Cloud Agent (autonomous)

How: Assign a Copilot agent to a GitHub issue or PR. The cloud agent runs autonomously on github.com infrastructure, makes changes, and opens a PR for review.

**Loads:**
- `.github/copilot-instructions.md` (auto)
- `.github/instructions/<NAME>.instructions.md` matching files it edits (auto)
- The custom agent assigned (from `.github/agents/<NAME>.md`)
- For org/enterprise: org-level custom agents from `.github-private/agents/<NAME>.md`

**Strong context isolation:** the cloud agent runs in a fresh environment per task. No leakage from previous sessions.

Use for: well-defined, self-contained tasks where you'd otherwise hand off to a junior engineer. Not for ambiguous work that needs back-and-forth.

### Mode 6: Copilot CLI

How: `gh copilot suggest` / `gh copilot explain` in the terminal.

**Loads:** Copilot CLI has its own instruction system (separate from IDE). See [Copilot CLI custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions).

**Use for:** terminal command suggestions and explanations.

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

Is this a terminal-related question?
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

Instead, `.github/copilot-instructions.md` is a **thin router** — golden rules + workflow + cross-links. The orchestrator persona is in `.github/agents/<project>-orchestrator.md` and only loads when explicitly selected.

## Mode comparison table

| Mode | Loads orchestrator persona? | Specialists invokable? | Tool allowlist enforced? | Context isolation |
|---|---|---|---|---|
| Inline completions | No | No | N/A | n/a |
| Plain Chat | No | Yes (manual `@`) | Default Chat tools | Within Chat session |
| Specialist Chat | No | (just this one) | Yes (per agent's `tools:`) | Within Chat session |
| Orchestrator Chat | Yes | Yes (via orchestrator's `@` calls) | Yes (per agent's `tools:`) | Within Chat session |
| Cloud Agent | If assigned | If multi-agent | Yes (per agent's `tools:`) | Strong (fresh env) |
| Copilot CLI | n/a (separate) | n/a | n/a | Separate context |

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

| Feature | VS Code | JetBrains | Visual Studio | github.com Chat | Cloud Agent | CLI |
|---|---|---|---|---|---|---|
| `.github/copilot-instructions.md` | ✅ | ✅ | ✅ | ✅ | ✅ | (separate) |
| `.github/instructions/*.instructions.md` (auto-load on path match) | ✅ | ✅ | ✅ | ✅ | ✅ | (separate) |
| `.github/prompts/*.prompt.md` (slash commands) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `.github/agents/*.md` (custom agents) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `.github/chatmodes/*.chatmode.md` (chat modes) | ✅ | ✅ (older) | ✅ (older) | ❌ | ❌ | ❌ |
| MCP servers per agent | ✅ | ✅ | ✅ | partial | partial | ✅ |

(Surface support evolves rapidly — verify against current Copilot release notes.)
