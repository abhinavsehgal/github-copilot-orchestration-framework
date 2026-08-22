# 02 — Architecture

The physical layout of files in a project that uses this framework.

## The full picture

```
your-project/
│
├── .github/
│   ├── copilot-instructions.md             ← repo-wide router (auto-loaded everywhere)
│   ├── agents/                             ← orchestrator + specialists
│   │   ├── <project>-orchestrator.agent.md
│   │   ├── <specialist-1>.agent.md
│   │   ├── <specialist-2>.agent.md
│   │   └── ...
│   ├── instructions/                       ← path-globbed invariants
│   │   ├── <domain-1>.instructions.md      (with applyTo: "src/api/**/*.ts")
│   │   ├── <domain-2>.instructions.md
│   │   └── ...
│   ├── skills/                             ← repeatable workflows, cross-surface (Agent Skills)
│   │   ├── <project>-engineering/SKILL.md  (invoked as /<project>-engineering, or auto-loaded)
│   │   ├── <workflow-1>/SKILL.md
│   │   └── ...
│   ├── prompts/                            ← repeatable prompts, IDE-only (slash commands)
│   │   ├── <workflow-1>.prompt.md          (invoked as /<workflow-1>)
│   │   └── ...
│   ├── hooks/                              ← optional: lifecycle hooks (Chapter 10)
│   │   └── <name>.json                     (preToolUse / postToolUse / agentStop …)
│   └── chatmodes/                          ← RETIRED — rename *.chatmode.md → agents/*.agent.md
│
├── AGENTS.md                                ← optional cross-AI metadata file
│                                              (also recognized by other AI tools)
│
├── docs/
│   ├── ai-context/                          ← orientation maps (read by agents)
│   │   ├── INDEX.md
│   │   ├── PROJECT.md                       ← current truth: what is live where (Chapter 11)
│   │   ├── LEARNINGS.md                     ← decisions, failures, corrections (Chapter 11)
│   │   ├── GLOSSARY.md                      ← one name per concept (Chapter 11)
│   │   ├── HANDOFF_SCHEMA.md
│   │   ├── ORCHESTRATION_SPOONFEEDER.md
│   │   └── <area>-experience.md             ← per-domain orientation, 50-150 lines each
│   ├── ARCHITECTURE.md                      ← canonical refs (full detail)
│   ├── API.md
│   ├── <YOUR-OTHER-CANONICAL-DOCS>.md
│   ├── <AREA>_BACKLOG.md                    ← deferred work, one per area (Chapter 11)
│   └── _archive/                            ← frozen historical
│       ├── README.md
│       └── <YYYY-MM>/
│
└── <your application code>/
```

## What lives where, and why

### `.github/copilot-instructions.md` (root router)

Auto-loaded by Copilot for EVERY interaction in this repository — including inline completions, Chat, code review, and cloud agent. Should be **under 200 lines**.

Contains:
- Identity (one sentence about the project)
- Golden rules (security, deployment, branching)
- Mandatory workflow (numbered steps for every non-trivial task)
- Routing guidance ("for X tasks, use the @<specialist> agent")
- Cross-links to the spoonfeeder, INDEX, and canonical docs

Does NOT contain:
- Long technical detail (lives in canonical docs)
- Per-domain gotchas (lives in `.github/instructions/`)
- The orchestrator's full persona (lives in `.github/agents/<project>-orchestrator.agent.md`)
- Sprint history or audit content (lives in `docs/_archive/`)

⚠ **Size budget.** The current docs say repository instructions "must be no longer than 2 pages" (and any single instruction file should stay under about 1,000 lines). Front-load the rules that matter most for code review — good practice, not a hard cap; the older "code review reads only the first 4,000 characters" sentence is no longer on the docs (Pitfall 2).

### `.github/agents/`

One markdown file per agent, named `<agent-name>.agent.md` (a bare `.md` name still works; `.agent.md` is the current convention). Frontmatter declares fields per the [Copilot custom agents spec](https://docs.github.com/en/copilot/reference/custom-agents-configuration):

```yaml
---
name: <agent-name>                    # optional, defaults to filename
description: <required string>        # how the agent decides when to delegate
tools: ['tool-a', 'tool-b']           # optional allowlist
mcp-servers: { ... }                  # optional per-agent MCP scoping
model: <model-name>                   # optional, IDE-specific
target: vscode | github-copilot       # optional environment scoping
disable-model-invocation: true|false  # optional, prevents auto-routing
user-invocable: true|false            # optional, controls manual selection
metadata: { key: value }              # optional annotations
agents: ['<specialist-1>', ...]       # VS Code only: subagent allowlist (`*` = all); ignored by the cloud agent
---
```

VS Code additionally documents `handoffs`, `argument-hint` and `hooks` (preview, agent-scoped); `argument-hint` and `handoffs` are not supported on the cloud agent. User-level agents live in `~/.copilot/agents/`, and VS Code also reads `.claude/agents/` by default (`chat.agentFilesLocations`).

Body is the agent's system prompt — up to 30,000 characters.

Always includes exactly one orchestrator (`<project-slug>-orchestrator`). Specialists are added based on the project's natural domain boundaries (typically 5-12 specialists for a medium-complexity codebase).

See `03-AGENTS-GUIDE.md` for how to design agents.

### `.github/instructions/`

Path-globbed invariants — Copilot's direct equivalent of "rules" in other frameworks. Each file:

```markdown
---
applyTo: "src/api/**/*.ts,src/server/**/*.ts"
excludeAgent: "code-review"          # optional: don't apply during code review
---

# API Layer Rules

## Hard rules

### <Rule title>
**Why:** <reason>
**How to apply:** <how>
```

Frontmatter `applyTo:` accepts glob patterns (comma-separated for multiples). When Copilot is editing a matching file, this instruction file is auto-loaded into the relevant prompts.

These are the **hard-won gotchas** of your codebase — things that broke production once and must not break again.

### `.github/skills/`

Agent Skills — the **cross-surface** home for repeatable multi-step workflows, and the Copilot equivalent of Claude Code skills. One directory per skill, `SKILL.md` inside:

```markdown
---
name: <skill-name>                    # required; lowercase-hyphen; equals the directory name
description: <one sentence — when to use this>   # required
---

<the procedure — steps, checks, Definition of Done>
```

Invoked as `/<skill-name>`, or loaded automatically when the task is relevant. Supported by the cloud agent, code review, the Copilot CLI, and agent mode in VS Code and JetBrains. Also discovered from `.claude/skills/` and `.agents/skills/`; personal skills live in `~/.copilot/skills/`. See Chapter 5 for the full frontmatter.

Use skills for bug investigation, feature build, QA pass, compliance review, and the project's engineering playbook (`templates/engineering-playbook-skill.md.template`).

### `.github/prompts/`

IDE-only repeatable prompts invoked via slash command. Each file:

```markdown
---
description: <one sentence — appears in /command picker>
---

<the prompt body — instructions, may include ${input:variable:hint} placeholders>
```

In Copilot Chat, type `/<filename-without-extension>` to invoke. Example: `.github/prompts/investigate-bug.prompt.md` is invoked as `/investigate-bug`.

Available in: VS Code, Visual Studio, JetBrains only. Currently in **public preview**. The cloud agent and the Copilot CLI do not use prompt files — VS Code's advice for a prompt an agent must run is "convert it to an agent skill". A prompt file is a convenience for a workflow you only ever start by hand in the IDE; anything else belongs in `.github/skills/`.

### `.github/hooks/` (optional)

Lifecycle hooks — `<name>.json` files declaring commands to run on `sessionStart`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop` and other events. The cloud agent loads only `.github/hooks/*.json`; the CLI also reads `~/.copilot/hooks/`; VS Code also reads `.claude/settings.json`. This is where the framework's mechanical enforcement (build-gate, doc-freshness gate, correction-capture) lives — see `docs/10-MECHANICAL-ENFORCEMENT.md` and `templates/hooks/`.

### `.github/chatmodes/` — RETIRED

Custom chat modes (`*.chatmode.md`) were the earlier name for custom agents and are retired. Rename any existing file to `.github/agents/<name>.agent.md`; the `description`, `tools` and `model` fields carry over unchanged (Pitfall 7). Do not create new chat-mode files.

### `AGENTS.md` (optional, recommended for cross-AI projects)

A cross-AI standard file recognized by GitHub Copilot, Claude, Gemini, and other AI assistants. If your project uses multiple AI tools, put shared cross-AI metadata here.

Format: plain markdown describing the project, agents, and conventions.

### `docs/ai-context/`

Orientation maps. Per-area guides Copilot reads at the start of any task in that area. Cheaper than reading thousand-line canonical docs.

Format: 50-150 lines per file.

Audience: AI agents (primary), humans (secondary).

Naming: `<area>-experience.md` or `<area>-<concern>.md`.

Special files:
- `INDEX.md` — task-type → docs + agent map
- `HANDOFF_SCHEMA.md` — bidirectional handoff schema
- `ORCHESTRATION_SPOONFEEDER.md` — human-facing usage guide

The `docs/ai-context/` location is tool-agnostic so the same orientation maps work if you later add Claude Code, Gemini Code Assist, or other AI tools.

### `docs/<UPPERCASE>.md`

Canonical references. Full detail. Updated when the underlying truth changes. As long as needed (no artificial line cap).

Common files: `ARCHITECTURE.md`, `API.md`, `TECH_STACK.md`, `CHANGELOG.md`, `PRODUCT_REQUIREMENTS.md`, `BUSINESS_DOCUMENT.md`.

### `docs/_archive/`

Frozen historical material. Date-prefixed subdirectories (`<YYYY-MM>/`). Active docs never link here.

Required: `_archive/README.md` documenting the archive convention.

## Why this layout works

### Predictability for Copilot

Every agent knows where to find:
- Its own contract: `.github/agents/<self>.agent.md`
- Repository-wide rules: `.github/copilot-instructions.md` (auto-loaded)
- Path-specific rules: `.github/instructions/*.instructions.md` (auto-loaded when editing matching files)
- Reusable workflows: `.github/skills/*/SKILL.md` (invoked via `/<name>` on every surface) — plus IDE-only `.github/prompts/*.prompt.md`
- Mechanical enforcement: `.github/hooks/*.json` (if installed)
- Current truth and learnings: `docs/ai-context/PROJECT.md`, `docs/ai-context/LEARNINGS.md`
- The handoff schema: `docs/ai-context/HANDOFF_SCHEMA.md`
- Orientation per topic: `docs/ai-context/<area>.md`
- Canonical references: `docs/<UPPERCASE>.md`

### Predictability for humans

A new engineer joining the project sees:
- `.github/copilot-instructions.md` — tells them how Copilot is configured
- `.github/agents/`, `.github/instructions/`, `.github/skills/`, `.github/prompts/`, `.github/hooks/` — Copilot config files
- `docs/` — documentation
- Application code in its standard tech-stack location

They can ignore the `.github/` agents/instructions/skills/prompts/hooks if they don't use Copilot.

### Minimal coupling to tech stack

Nothing in `.github/` or `docs/ai-context/` cares about your framework, language, or deploy target. The agents reference paths in your actual codebase, but the agent FILES themselves are pure markdown. You can reorganize your codebase entirely and only need to update the `applyTo:` globs and the orientation maps' cross-links.

## What does NOT belong in this framework

- **Application code.** Stays where your tech stack puts it.
- **Build config.** Stays at root.
- **CI workflows.** Stays in `.github/workflows/`.
- **Test files.** Stays in your tests directory.
- **Generated artifacts.** Gitignored.

The framework is **purely additive** to whatever else lives in your repo.

## Variations

### Monorepo

For monorepos, you can either:
- Have one `.github/` at the root that knows about all packages, or
- Have one `AGENTS.md` per package + a root `.github/copilot-instructions.md` that orchestrates across packages

The first is simpler. The second is more isolated. Pick based on whether tasks routinely cross package boundaries. Note that `AGENTS.md` can exist at any depth and the nearest one in the directory tree wins (cloud agent, code review, CLI); VS Code reads the root one automatically and nested ones only behind the experimental `chat.useNestedAgentsMdFiles` setting.

### Multiple repositories (web + mobile + microservices)

When the units are separate repos rather than packages, see **`12-MULTI-REPO-WORKSPACES.md`** — the cloud agent cannot change more than one repository in a run, so cross-repo work is a workspace of per-repo installs plus shared contracts, not one agent over everything.

### Mobile + web split

If your project has separate mobile and web codebases in one repo, your orchestrator typically delegates to a `mobile-ui` specialist OR a `web-ui` specialist. They can share a `backend-api` specialist for the API-side work.

### Multi-tenancy / multi-product

If you serve multiple products from one codebase, consider per-product orientation maps and per-product instructions. The orchestrator can be one or per-product depending on whether tasks routinely cross products.

### Heavy infra / DevOps focus

Add specialists for `infrastructure`, `observability`, `release-automation`. Their instruction files cover Terraform/CDK/Kubernetes/Helm conventions.

### ML / data heavy

Add specialists for `data-pipeline`, `model-training`, `inference-serving`. Their orientation maps cover dataset locations, experiment tracking, model versioning.

The architecture flexes; the principles don't.

## Organization-level customization (Business / Enterprise)

If you have GitHub Copilot Business or Enterprise, you can also configure:

- **Organization custom instructions** (UI at organization settings) — applies to Chat on github.com, code review, and cloud agent across all org members
- **Organization-level custom agents** — `/agents/<NAME>.agent.md` in the organization's `.github` or `.github-private` repository

Use organization-level configuration for company-wide policies (e.g. "always cite the file:line for security-sensitive code"). Use repository-level for project-specific specialists and rules.

Precedence (highest to lowest): Personal > Repository > Organization. All sets are provided to Copilot when relevant.
