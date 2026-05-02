# GitHub Copilot Orchestration Framework

> **Purpose.** A reusable multi-agent orchestration setup for [GitHub Copilot](https://docs.github.com/en/copilot) that prevents cascading hallucinations, enforces evidence-based handoffs between Copilot custom agents, and makes Copilot usable on production codebases by teams. Tech-stack agnostic — drops into any project (web, mobile, backend, ML, infra) in 2-4 hours.

This framework uses GitHub Copilot's **own** customization surface — `.github/copilot-instructions.md`, `.github/instructions/`, `.github/prompts/`, `.github/agents/`, `.github/chatmodes/` — and adds documentation conventions on top. No external tools, no extensions, no Marketplace dependencies.

---

## Who this is for

- Engineering teams using GitHub Copilot on real production codebases
- Projects with multiple roles, multiple data systems, or compliance/security/payments concerns
- Anyone who has experienced Copilot drifting on rules, hallucinating cross-file claims, or producing inconsistent suggestions across team members
- Teams who use Copilot in a mix of: VS Code Chat, JetBrains, Visual Studio, Copilot CLI, GitHub.com Chat, and the Copilot cloud agent (autonomous coding agent)

If your project is a solo prototype or under ~5,000 lines, you probably don't need this yet — default Copilot is enough. Adopt this when complexity passes the threshold.

## The 30-second pitch

Default Copilot is excellent for inline suggestions and quick chat. For complex projects, you need:

- **One orchestrator custom agent that delegates to specialist custom agents** (instead of asking one chat session to do everything and flooding context)
- **Bidirectional structured handoffs with evidence-bound claims** (instead of free-form prompts that propagate hallucinations across hops)
- **Per-agent runtime constraints** — REVIEW-ONLY agents physically can't write; tool allowlists scope MCP access; `target` field scopes agents to specific environments
- **A `failure_condition` field** that forces specialist agents to STOP if the orchestrator's premise turns out wrong (instead of pushing through on a false hypothesis)
- **Three-tier docs** — orientation maps for Copilot, canonical refs for humans, frozen archive for history
- **Path-globbed instruction files** — `.github/instructions/<NAME>.instructions.md` with `applyTo:` glob (auto-loaded when the matching file is being edited)

This framework gives you all of that using Copilot's documented features. No runtime hooks, no external dependencies — just markdown + Copilot's built-in customization mechanisms.

## Copilot customization surfaces this framework uses

| Mechanism | File | Loaded when | Best for |
|---|---|---|---|
| **Repository-wide instructions** | `.github/copilot-instructions.md` | Every repository request | Project-wide router (golden rules + workflow + agent map) |
| **Path-specific instructions** | `.github/instructions/<NAME>.instructions.md` (with `applyTo:` glob) | Editing files matching the glob | Domain-scoped invariants ("rules") |
| **Prompt files** | `.github/prompts/<NAME>.prompt.md` | Manually invoked via `/<name>` in Chat | Repeatable workflows ("skills") |
| **Custom agents** | `.github/agents/<NAME>.md` | Selected manually OR auto-routed | Orchestrator + specialists |
| **Custom chat modes** | `.github/chatmodes/<NAME>.chatmode.md` | Selected from Chat dropdown | Personas (optional, secondary to agents) |

All five mechanisms are documented Copilot features. See `docs/02-ARCHITECTURE.md` for which to use when.

## What's in the box

```
github-copilot-orchestration-framework/
├── README.md                            ← this file
├── GitHub-Copilot-Orchestration-Framework.pdf   ← consolidated printable
├── LICENSE
│
├── docs/                                 ← framework documentation (9 chapters)
│   ├── 01-PRINCIPLES.md                  ← seven core principles
│   ├── 02-ARCHITECTURE.md                ← .github/ layout (instructions / prompts / agents / chatmodes)
│   ├── 03-AGENTS-GUIDE.md                ← how to design orchestrator + specialists for Copilot
│   ├── 04-HANDOFF-SCHEMA.md              ← bidirectional schema + worked examples
│   ├── 05-INSTRUCTIONS-AND-PROMPTS.md    ← path-globbed instructions + prompt files
│   ├── 06-INVOCATION-MODES.md            ← Chat vs Edit vs Cloud Agent vs CLI
│   ├── 07-FOLDER-STRUCTURE.md            ← three-tier doc organization
│   ├── 08-COMMON-PITFALLS.md             ← Copilot-specific lessons + framework lessons
│   └── 09-RUNBOOK.md                     ← step-by-step bootstrap (~2-4 hours)
│
├── prompts/                              ← ready-to-paste Chat prompts for bootstrapping
│   ├── INVENTORY-PROMPT.md
│   ├── BOOTSTRAP-PROMPT.md
│   └── REFINEMENT-PROMPT.md
│
└── templates/                            ← drop-in templates with placeholders
    ├── orchestrator-agent.md.template       ← .github/agents/<project>-orchestrator.md
    ├── specialist-agent.md.template         ← .github/agents/<specialist>.md
    ├── review-only-agent.md.template        ← REVIEW-ONLY specialist
    ├── HANDOFF_SCHEMA.md.template           ← docs/ai-context/HANDOFF_SCHEMA.md
    ├── INDEX.md.template                    ← docs/ai-context/INDEX.md
    ├── SPOONFEEDER.md.template              ← docs/ai-context/ORCHESTRATION_SPOONFEEDER.md
    ├── copilot-instructions.md.template     ← .github/copilot-instructions.md (root router)
    ├── instructions.md.template             ← .github/instructions/<name>.instructions.md
    ├── prompt.md.template                   ← .github/prompts/<name>.prompt.md
    ├── chatmode.md.template                 ← .github/chatmodes/<name>.chatmode.md
    └── archive-README.md.template
```

---

## Quick start — pick your scenario

### Scenario A — Brand new project (greenfield)

```bash
# 1. Verify Copilot is installed in your IDE (VS Code / JetBrains / Visual Studio)
#    and you're signed in to a GitHub account with Copilot access

# 2. Open the new project in your IDE
cd ~/path/to/new-project

# 3. Open Copilot Chat (Ctrl+Cmd+I in VS Code)
#    Paste prompts/BOOTSTRAP-PROMPT.md
#    Replace <framework path> with the absolute path to this repo on your machine
```

For greenfield, you can skip the inventory step and tell Copilot what specialists you want directly:

```
Set up the GitHub Copilot Orchestration Framework for a new
<Next.js + Postgres + Stripe> project named "<your-project-name>".

Specialists I want (each becomes .github/agents/<name>.md):
  - <project-slug>-orchestrator
  - frontend-ui
  - backend-api
  - database
  - payments
  - auth-security
  - qa-functional
  - release-devops
  - security-privacy (REVIEW-ONLY)

Use templates from <absolute path to this repo>/templates/.
Show me each generated file before saving.
```

Estimated time: **45 min - 1 hour**.

### Scenario B — Existing project (brownfield)

⚠ **If your project ALREADY has Copilot configuration** (e.g. you ran VS Code's `/init`, or your team has been adding to `.github/copilot-instructions.md` for months), the bootstrap will detect this and ask before overwriting. The framework includes mandatory pre-flight safety checks:

- **Pre-flight 1** — auto-snapshot existing config to `.github-pre-bootstrap-backup/`
- **Pre-flight 2** — naming-collision check (per file — STOP for explicit user decision)
- **Pre-flight 3** — `applyTo:` glob conflict check
- **Pre-flight 4** — drift detection on existing `.github/copilot-instructions.md`
- **Pre-flight 5** — existing chatmode/agent style detection
- **Decision gate** — STOP if any pre-flight raised a `<NEEDS USER CONFIRMATION>` flag

You can also do an extra manual snapshot before starting:
```bash
mkdir -p .github-pre-bootstrap-backup
cp -r .github .github-pre-bootstrap-backup/ 2>/dev/null
[ -f AGENTS.md ] && cp AGENTS.md .github-pre-bootstrap-backup/
```

The framework's BOOTSTRAP-PROMPT will repeat the snapshot inside the prompt — this is just belt-and-suspenders.

```bash
# 1. Verify Copilot is installed and you're signed in

# 2. Open the existing project in your IDE
cd ~/path/to/existing-project

# 3. Make sure git is clean
git status
git checkout -b setup/copilot-orchestration

# 4. Open Copilot Chat
#    Paste prompts/INVENTORY-PROMPT.md
#    Replace <framework path> with the absolute path to this repo on your machine

# 5. Review/adjust Copilot's proposed inventory + answer Open Questions
#    INVENTORY scans for existing .github/agents/, /instructions/, /prompts/, /chatmodes/, AGENTS.md

# 6. Paste prompts/BOOTSTRAP-PROMPT.md (in the same Chat session)
#    BOOTSTRAP runs pre-flight safety checks BEFORE creating any files
#    If existing config is detected, you'll be asked to confirm per-file decisions

# 7. Verify in terminal
ls .github/agents/                # should list your project specialists
ls .github/instructions/          # should list your path-globbed instructions
ls .github/prompts/               # should list your prompt files
diff .github-pre-bootstrap-backup/copilot-instructions.md .github/copilot-instructions.md  # if existed

# 8. Test in Chat — pick a real task
#    In VS Code Chat: select your orchestrator agent from the agent picker
#    Or: type @<project-slug>-orchestrator <your task>

# 9. Commit and PR (note: .github-pre-bootstrap-backup/ is gitignored)
git add .github/ docs/ai-context/ docs/_archive/ AGENTS.md .gitignore
git commit -m "chore: bootstrap GitHub Copilot orchestration framework"
git push -u origin setup/copilot-orchestration
gh pr create --base <your-base-branch> --title "Bootstrap Copilot orchestration framework"
```

Estimated time: **2-4 hours** (split across phases — see `docs/09-RUNBOOK.md`).

### Scenario C — Just want to read the framework

Read the **PDF** (`GitHub-Copilot-Orchestration-Framework.pdf`). Or browse the markdown files in `docs/` for clickable cross-links.

The most actionable single chapter is **`docs/09-RUNBOOK.md`**.

---

## Prerequisites

- **GitHub Copilot subscription** (Individual / Business / Enterprise)
- **Copilot extension** installed in your IDE (VS Code, Visual Studio, JetBrains, Eclipse, or Xcode)
- **Custom agents support** — verify in your IDE settings (some features require recent Copilot extension versions)
- **Git** + optional **GitHub CLI** (`gh`) for PR creation

Note: prompt files and custom agents are currently in **public preview** in some IDEs. Functionality may evolve. Check the [Copilot release notes](https://docs.github.com/en/copilot/whats-new) when adopting new features.

## What this framework does NOT include

- **Runtime hooks.** Copilot doesn't expose pre-tool-use hooks the way some AI agents do. Documentation enforcement is the framework's primary discipline layer.
- **`maxTurns` enforcement.** Copilot agents don't currently support a `maxTurns` field. The orchestrator's "soft hop limit" is documentation-only.
- **Code generation.** This is purely a documentation + agent-config framework. Your application code stays untouched.
- **Vendor lock-in.** No external services, no SaaS, no API keys beyond your existing Copilot subscription.

## Customizing the framework for your project

Three things change per project:

1. **Project name + slug** (used to name the orchestrator: `<slug>-orchestrator`)
2. **Specialist list** (5-10 specialists matching your project's domain boundaries)
3. **`applyTo:` globs in path-specific instructions** (matched to your actual code paths)

Everything else (the handoff schema, the universal evidence rule, the failure_condition pattern, the three-tier docs structure) stays the same across projects.

See **`docs/03-AGENTS-GUIDE.md`** for how to pick your specialist list and **`docs/05-INSTRUCTIONS-AND-PROMPTS.md`** for how to write path-globbed instructions and prompt files.

## Maintenance

After bootstrapping, the framework needs minimal upkeep:

- **Per task:** instruction files accumulate naturally as production teaches you new gotchas
- **Per quarter:** run `prompts/REFINEMENT-PROMPT.md` to audit specialist scope drift, archive stale docs
- **Per major refactor:** the `context-librarian` specialist (if you spawned one) handles docs reorganization

## How this differs from the Claude Code version

If you've seen the [Claude Code Orchestration Framework](https://github.com/abhinavsehgal/claude-orchestration-framework), the patterns are similar but the implementation differs because Copilot's customization model differs:

| Concept | Claude Code | GitHub Copilot |
|---|---|---|
| Project router | `CLAUDE.md` | `.github/copilot-instructions.md` (or `AGENTS.md` for cross-AI) |
| Specialists | `.claude/agents/<name>.md` | `.github/agents/<NAME>.md` |
| Path-globbed rules | `.claude/rules/<name>.md` (with `applies_to:` in body) | `.github/instructions/<NAME>.instructions.md` (with `applyTo:` in frontmatter — direct equivalent, cleaner) |
| Repeatable workflows | `.claude/skills/<name>/SKILL.md` | `.github/prompts/<NAME>.prompt.md` (invoked via `/<name>`) |
| Personas (optional) | n/a | `.github/chatmodes/<NAME>.chatmode.md` |
| Tool allowlists | `tools:` field | `tools:` field (same syntax) |
| MCP scoping | `mcpServers:` field | `mcp-servers:` field (note hyphen, not camelCase) |
| Cross-agent invocation | `Agent(name1, name2)` allowlist | `agent` tool alias + cross-references in body |
| `maxTurns` | Supported | NOT supported (documentation discipline only) |
| Subagent context isolation | Strong (separate context windows) | Varies by surface — see `docs/06-INVOCATION-MODES.md` |
| Invocation modes | `claude` / `claude --agent <name>` | IDE Chat / Cloud Agent / CLI / Code Review (4+ surfaces) |

Both frameworks share: the bidirectional handoff schema, the universal evidence rule, the failure_condition pattern, three-tier docs, and the principle of documentation-discipline-over-runtime-enforcement.

## Frequently asked

**Q: Do custom agents work in inline completions?**
No. Custom agents apply to Chat, Cloud Agent, and (depending on IDE) Edits. Inline completions use only repository-wide instructions (`.github/copilot-instructions.md`) and path-specific instructions (`.github/instructions/`).

**Q: My team uses both VS Code and JetBrains. Will this framework work for both?**
Yes — the `.github/` files are shared across IDEs. Some features (like prompt files) work identically in VS Code, Visual Studio, JetBrains. The cloud agent on github.com uses the same files plus `target:` field scoping if needed.

**Q: What's the difference between custom agents and custom chat modes?**
Custom agents (`.github/agents/<NAME>.md`) are the newer format with full metadata (tools, mcp-servers, model, target, user-invocable). Custom chat modes (`.github/chatmodes/<NAME>.chatmode.md`) are the older format with similar but more limited fields. The framework recommends agents over chat modes for new projects — `.agent.md` is the more future-proof format.

**Q: Will this conflict with my existing `.github/copilot-instructions.md`?**
The framework will REPLACE your existing file. Back it up first if it has content you want to preserve. The new router includes a section for project-specific instructions you can keep.

**Q: Can I share this with another engineer who isn't on my team?**
This repo is private. Add them as a collaborator on GitHub, or clone the framework folder and share. The framework itself is MIT-licensed.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.

## Sources

This framework's design is grounded in the official GitHub Copilot documentation:
- [About customizing GitHub Copilot Chat responses](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses)
- [Adding repository custom instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
- [Custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [About custom agents (cloud agent)](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-custom-agents)
- [Prompt files tutorial](https://docs.github.com/en/copilot/tutorials/customization-library/prompt-files)
- [Creating custom agents for Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)
