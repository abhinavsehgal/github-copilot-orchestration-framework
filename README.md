# GitHub Copilot Orchestration Framework

> **Version 1.3.0** ([changelog](CHANGELOG.md)) · MIT license · `templates/` + `docs/` are tech-stack and domain agnostic
>
> **v1.3.0 (2026-08-24) — the third leg: scheduled autonomy.** New chapter **13 — Standing Routines**: narrow agent jobs on a schedule (a `schedule:` workflow assigning an issue to the cloud agent, or headless `copilot -p`) that open small PRs behind review gates — one charter per routine, repro + truth table on every fix, noise budgets, wrong output tunes the *routine*, attempt caps + a verified retire path. Ships `templates/routine.md.template` and a cross-surface hill-climb skill. Chapter 12's scheduled `contract-guardian` job is now fully specified. Two new pitfalls (a context system is a program; unattended jobs need a verified retire path) plus REFINEMENT checks 11–12.
>
> **v1.2.0 (2026-08-22) — three months of production use, folded back in, and three platform claims retracted.** New chapters: **11 — Project truth, learnings and the evidence ladder** and **12 — Multi-repo workspaces** (web + mobile + microservices across repos, with the `.code-workspace` + manifest + delegation pattern). Nine new pitfalls. **Retracted, because the platform moved:** Copilot *does* have lifecycle hooks now (`.github/hooks/*.json` — chapter 10 is rewritten around them, with five working templates); custom agents *can* invoke custom agents (VS Code `agents:` is an allowlist); and **agent skills** (`.github/skills/`) — not prompt files — are the cross-surface equivalent of Claude Code skills. Every platform claim in v1.2.0 carries a verified-on date.
>
> **Purpose.** A reusable multi-agent orchestration setup for [GitHub Copilot](https://docs.github.com/en/copilot) that prevents cascading hallucinations, enforces evidence-based handoffs between Copilot custom agents, and makes Copilot usable on production codebases by teams. Tech-stack agnostic — drops into any project (web, mobile, backend, ML, infra) in 2-4 hours.

This framework uses GitHub Copilot's **own** customization surface — `.github/copilot-instructions.md` (or `AGENTS.md`), `.github/instructions/`, `.github/agents/*.agent.md`, `.github/skills/`, `.github/hooks/`, and the IDE-only `.github/prompts/` — and adds documentation conventions on top. No external tools, no extensions, no Marketplace dependencies.

> **Read the onboarding guide online:** https://abhinavsehgal.github.io/github-copilot-orchestration-framework/ — all three editions, one page.

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

This framework gives you all of that using Copilot's documented features. No external dependencies — just markdown + Copilot's built-in customization mechanisms (lifecycle hooks are an optional later layer, chapter 10).

## Copilot customization surfaces this framework uses

| Mechanism | File | Loaded when | Best for |
|---|---|---|---|
| **Repository-wide instructions** | `.github/copilot-instructions.md` | Every repository request | Project-wide router (golden rules + workflow + agent map) |
| **Path-specific instructions** | `.github/instructions/<NAME>.instructions.md` (with `applyTo:` glob) | Editing files matching the glob | Domain-scoped invariants ("rules") |
| **Agent skills** (v1.2) | `.github/skills/<name>/SKILL.md` | `/<name>` or auto-loaded by relevance — cloud agent, CLI, code review, VS Code, JetBrains | Repeatable workflows — the cross-surface equivalent of Claude Code skills |
| **Prompt files** | `.github/prompts/<NAME>.prompt.md` | `/<name>` in IDE Chat only | IDE-only convenience prompts (not the cloud agent, not the CLI) |
| **Custom agents** | `.github/agents/<NAME>.agent.md` | Selected manually, `@`-mentioned, or invoked as a subagent (VS Code `agents:` allowlist) | Orchestrator + specialists |
| **Hooks** (v1.2) | `.github/hooks/*.json` | Lifecycle events (`preToolUse`, `postToolUse`, `agentStop`, …) — cloud agent, CLI, VS Code | Mechanical enforcement (chapter 10) |
| ~~Custom chat modes~~ | `.github/chatmodes/*.chatmode.md` | Retired — rename to `.agent.md` | — |

All of these are documented Copilot features (verified 2026-08-22). See `docs/02-ARCHITECTURE.md` for which to use when.

## What's in the box

```
github-copilot-orchestration-framework/
├── README.md                            ← this file
├── GitHub-Copilot-Orchestration-Framework.pdf   ← consolidated printable
├── LICENSE
│
├── docs/                                 ← framework documentation
│   ├── 00-QUICKSTART.md                  ← START HERE: step-by-step onboarding for any project, incl. many repos (VS Code + CLI)
│   ├── 00-QUICKSTART.html                ← the same guide as one offline page with tabs for all three editions (open in a browser)
│   │                                    live: https://abhinavsehgal.github.io/github-copilot-orchestration-framework/
│   ├── 01-PRINCIPLES.md                  ← seven core principles
│   ├── 02-ARCHITECTURE.md                ← .github/ layout (instructions / agents / skills / hooks / prompts; chat modes retired)
│   ├── 03-AGENTS-GUIDE.md                ← how to design orchestrator + specialists for Copilot
│   ├── 04-HANDOFF-SCHEMA.md              ← bidirectional schema + worked examples
│   ├── 05-INSTRUCTIONS-AND-PROMPTS.md    ← path-globbed instructions + prompt files
│   ├── 06-INVOCATION-MODES.md            ← Chat vs Edit vs Cloud Agent vs CLI
│   ├── 07-FOLDER-STRUCTURE.md            ← three-tier doc organization
│   ├── 08-COMMON-PITFALLS.md             ← Copilot-specific + framework lessons (30 as of v1.3)
│   ├── 09-RUNBOOK.md                     ← step-by-step bootstrap (~2-4 hours)
│   ├── 10-MECHANICAL-ENFORCEMENT.md      ← (rewritten v1.2) Copilot hooks: the contract, five patterns, twelve design rules
│   ├── 11-PROJECT-TRUTH-AND-LEARNINGS.md ← (v1.2) PROJECT.md / LEARNINGS.md / backlogs, the evidence ladder, the six-gate playbook
│   ├── 12-MULTI-REPO-WORKSPACES.md       ← (v1.2) web + mobile + microservices across repos: layers, three delegation mechanisms, contracts
│   └── 13-STANDING-ROUTINES.md           ← (v1.3) scheduled autonomy: routine fleets, output contracts, budgets, review gates
│
├── prompts/                              ← ready-to-paste Chat prompts for bootstrapping
│   ├── INVENTORY-PROMPT.md
│   ├── BOOTSTRAP-PROMPT.md
│   └── REFINEMENT-PROMPT.md
│
└── templates/                            ← drop-in templates with placeholders
    ├── orchestrator-agent.md.template       ← .github/agents/<project>-orchestrator.agent.md
    ├── specialist-agent.md.template         ← .github/agents/<specialist>.agent.md
    ├── review-only-agent.md.template        ← REVIEW-ONLY specialist
    ├── HANDOFF_SCHEMA.md.template           ← docs/ai-context/HANDOFF_SCHEMA.md
    ├── INDEX.md.template                    ← docs/ai-context/INDEX.md
    ├── SPOONFEEDER.md.template              ← docs/ai-context/ORCHESTRATION_SPOONFEEDER.md
    ├── copilot-instructions.md.template     ← .github/copilot-instructions.md (root router)
    ├── instructions.md.template             ← .github/instructions/<name>.instructions.md
    ├── prompt.md.template                   ← .github/prompts/<name>.prompt.md
    ├── chatmode.md.template                 ← RETIRED format — kept only to migrate legacy .chatmode.md files to .agent.md
    ├── archive-README.md.template
    ├── correction-capture.prompt.md.template  ← (v1.1) IDE-only prompt-file form — superseded by the skill below
    ├── commit-push-pr.prompt.md.template      ← (v1.1) IDE-only prompt-file form — superseded by the skill below
    ├── PROJECT.md.template · LEARNINGS.md.template · BACKLOG.md.template · GLOSSARY.md.template   ← (v1.2) the project-truth set
    ├── engineering-playbook-skill.md.template ← (v1.2) six gates + evidence ladder, as .github/skills/<slug>-engineering/SKILL.md
    ├── skill.md.template                      ← (v1.2) generic agent-skill shape for .github/skills/<name>/SKILL.md
    ├── skills/                                ← (v1.2) cross-surface skills: commit-push-pr, correction-capture, verify-build; (v1.3) hill-climb
    ├── routine.md.template                    ← (v1.3) standing-routine charter (Chapter 13)
    ├── hooks/                                 ← (v1.2) hooks.json + hook-io / correction-detect / doc-freshness-track / lint-fix / stop-gate
    └── workspace/                             ← (v1.2) the multi-repo layer: .code-workspace, manifest, router, orchestrator + 2 specialists, contract instructions, delegate skill + scripts
        ├── bootstrap.sh.template                 ← (v1.2) creates the whole layer from a filled workspace.json
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

Specialists I want (each becomes .github/agents/<name>.agent.md):
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
#    INVENTORY scans for existing .github/agents/, /instructions/, /skills/, /hooks/, /prompts/, /chatmodes/, AGENTS.md

# 6. Paste prompts/BOOTSTRAP-PROMPT.md (in the same Chat session)
#    BOOTSTRAP runs pre-flight safety checks BEFORE creating any files
#    If existing config is detected, you'll be asked to confirm per-file decisions

# 7. Verify in terminal
ls .github/agents/                # should list your project specialists
ls .github/instructions/          # should list your path-globbed instructions
ls .github/skills/                # should list the <project-slug>-engineering skill + starter skills
ls .github/prompts/               # optional IDE-only prompt files, if you created any
diff .github-pre-bootstrap-backup/copilot-instructions.md .github/copilot-instructions.md  # if existed

# 8. Test in Chat — pick a real task
#    In VS Code Chat: select your orchestrator agent from the agent picker
#    Or: type @<project-slug>-orchestrator <your task>

# 9. Commit and PR (note: .github-pre-bootstrap-backup/ is gitignored)
git add .github/ docs/ .gitignore            # plus AGENTS.md if you created one
git commit -m "chore: bootstrap GitHub Copilot orchestration framework"
git push -u origin setup/copilot-orchestration
gh pr create --base <your-base-branch> --title "Bootstrap Copilot orchestration framework"
```

Estimated time: **2-4 hours** (split across phases — see `docs/09-RUNBOOK.md`).

### Scenario C — Just want to read the framework

Read the **PDF** (`GitHub-Copilot-Orchestration-Framework.pdf`) — the **v1.3.0 render** (60 pages: quickstart + all 14 chapters, hooks rewrite included). Or browse the markdown files in `docs/` for clickable cross-links.

The most actionable single chapter is **`docs/00-QUICKSTART.md`** — every step with *what / why / paste this / you know it worked when*, through the multi-repo workspace. `docs/09-RUNBOOK.md` is its long form.

---

## Prerequisites

- **GitHub Copilot subscription** (Individual / Business / Enterprise)
- **Copilot extension** installed in your IDE (VS Code, Visual Studio, JetBrains, Eclipse, or Xcode)
- **Custom agents support** — verify in your IDE settings (some features require recent Copilot extension versions)
- **Git** + optional **GitHub CLI** (`gh`) for PR creation

Note: prompt files and custom agents are currently in **public preview** in some IDEs. Functionality may evolve. Check the [Copilot release notes](https://docs.github.com/en/copilot/whats-new) when adopting new features.

## What this framework does NOT include

- **Hooks in the default install.** Copilot *does* have lifecycle hooks as of v1.2 (`.github/hooks/*.json`), and five templates ship — but they are an explicit later-phase decision (chapter 10), not a setup-time default. Documentation discipline stays the primary layer.
- **`maxTurns` enforcement.** No documented per-agent turn cap on Copilot (verified 2026-08-22). The orchestrator's "soft hop limit" is documentation-only.
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
- **Per quarter:** run `prompts/REFINEMENT-PROMPT.md` to audit specialist scope drift, archive stale docs, re-check platform drift (Pitfall 20 — three v1.0/v1.1 claims were false within three months), and re-stamp `PROJECT.md`
- **Per major refactor:** the `context-librarian` specialist (if you spawned one) handles docs reorganization

## How this differs from the Claude Code version

If you've seen the [Claude Code Orchestration Framework](https://github.com/abhinavsehgal/claude-orchestration-framework), the patterns are similar but the implementation differs because Copilot's customization model differs:

| Concept | Claude Code | GitHub Copilot |
|---|---|---|
| Project router | `CLAUDE.md` | `.github/copilot-instructions.md` (or `AGENTS.md` for cross-AI) |
| Specialists | `.claude/agents/<name>.md` | `.github/agents/<NAME>.agent.md` |
| Path-globbed rules | `.claude/rules/<name>.md` (with `paths:` frontmatter — native) | `.github/instructions/<NAME>.instructions.md` (with `applyTo:` frontmatter — direct equivalent) |
| Repeatable workflows | `.claude/skills/<name>/SKILL.md` | `.github/skills/<name>/SKILL.md` (cross-surface); `.github/prompts/*.prompt.md` is IDE-only |
| Personas (optional) | n/a | ~~chat modes~~ retired — use `.agent.md` |
| Tool allowlists | `tools:` field | `tools:` field (same idea, Copilot tool names) |
| MCP scoping | `mcpServers:` field | `mcp-servers:` field (note hyphen) |
| Cross-agent invocation | `Agent(name1, name2)` allowlist; nesting to a configurable depth | VS Code `agents:` allowlist + `runSubagent` (nesting via `chat.subagents.allowInvocationsFromSubagents`); cloud agent `agent` tool alias, no allowlist |
| Lifecycle hooks | `.claude/settings.json` (`PreToolUse` / `PostToolUse` / `Stop`; stderr + exit 2) | `.github/hooks/*.json` (`preToolUse` / `postToolUse` / `agentStop`; JSON on stdout; timeouts fail open). VS Code also reads `.claude/settings.json` |
| Path-scoped rules | `.claude/rules/*.md` with `paths:` | `.github/instructions/*.instructions.md` with `applyTo:`. VS Code also reads `.claude/rules/` |
| `maxTurns` | Supported | Not documented (discipline only) |
| Headless / scripted | `claude -p … --agent … --output-format json` | `copilot -p … --agent=… --allow-tool=…` (no structured-output flag documented) |
| Multi-repo | Workspace dir; child hooks do NOT load from a parent (chapter 12) | VS Code multi-root loads every folder's agents + hooks — name collisions instead (chapter 12) |
| Invocation modes | `claude` / `claude --agent <name>` / `claude -p` | IDE Chat / Cloud Agent / CLI / Code Review (4+ surfaces) |

Both frameworks share: the bidirectional handoff schema, the universal evidence rule, the failure_condition pattern, three-tier docs, the v1.2 project-truth set (`PROJECT.md` / `LEARNINGS.md` / backlogs / glossary), the evidence-confidence taxonomy, and the principle of documentation-discipline-over-runtime-enforcement. **Running both in one repo** is a supported setup: VS Code reads `.claude/rules`, `.claude/agents`, `.claude/skills` and `.claude/settings.json` hooks by default, so one corpus of rules/agents/skills can serve two thin routers (`CLAUDE.md` + `.github/copilot-instructions.md`). The two hook contracts differ — never share a hook script without an adapter.

## Frequently asked

**Q: Do custom agents work in inline completions?**
No. Custom agents apply to Chat, Cloud Agent, and (depending on IDE) Edits. Inline completions use only repository-wide instructions (`.github/copilot-instructions.md`) and path-specific instructions (`.github/instructions/`) — the path-specific half is not re-verified; see chapter 6, Mode 1.

**Q: My team uses both VS Code and JetBrains. Will this framework work for both?**
Yes — the `.github/` files are shared across IDEs. Some features (like prompt files) work identically in VS Code, Visual Studio, JetBrains. The cloud agent on github.com uses the same files plus `target:` field scoping if needed.

**Q: What's the difference between custom agents and custom chat modes?**
Chat modes are retired: custom agents *were* chat modes, and the docs now say to rename `.chatmode.md` files to `.agent.md`. Use `.github/agents/<NAME>.agent.md`.

**Q: Prompt files or skills?**
Skills (`.github/skills/<name>/SKILL.md`). They work on the cloud agent, the CLI, code review and every IDE; prompt files are IDE-only and the VS Code docs say to convert a prompt to a skill for the Agent Host. The framework's v1.1 prompt-file templates are kept for IDE use; the v1.2 skills supersede them.

**Q: Will this conflict with my existing `.github/copilot-instructions.md`?**
No — the BOOTSTRAP prompt runs pre-flight checks, snapshots the existing file, and shows a 3-pane diff before writing a merged router. Nothing is replaced without your per-file approval.

**Q: We have one repo per microservice plus separate web and mobile repos. Agents at every level, or one top-level orchestrator?**
Both, in layers — and not a separate framework. Every repo keeps its own install; shared specialists move to the organisation's `.github-private/agents/`; a workspace repo with a `.code-workspace` file, a manifest and gitignored clones holds *only* the cross-repo orchestrator, the service map and the contract rules, and delegates writes to each child's own orchestrator (its own CLI session, or a VS Code subagent). Design, verified platform behaviour, and a one-afternoon POC recipe: `docs/12-MULTI-REPO-WORKSPACES.md` + `templates/workspace/`.

**Q: Can I share this with another engineer who isn't on my team?**
This repo is public and MIT-licensed. Send the link.

## Companion editions

All three share the handoff schema, the evidence rule, the three-tier docs, the project-truth set and the multi-repo workspace layer; they differ only in the tool's surfaces. Released together on 2026-08-22.

- [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) v1.2.0 — Claude Code.
- [`copilot-ios-orchestration-framework`](https://github.com/abhinavsehgal/copilot-ios-orchestration-framework) v1.1.0 — this edition, pre-filled for native iOS.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.

## Sources

This framework's design is grounded in the official GitHub Copilot documentation:
- [About customizing GitHub Copilot Chat responses](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses)
- [Adding repository custom instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
- [Custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [About custom agents (cloud agent)](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-custom-agents)
- [Prompt files tutorial](https://docs.github.com/en/copilot/tutorials/customization-library/prompt-files)
- [About agent skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) · [Hooks reference](https://docs.github.com/en/copilot/reference/hooks-reference) · [Copilot CLI programmatic reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-programmatic-reference)
- [VS Code: custom agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents) · [VS Code: subagents](https://code.visualstudio.com/docs/copilot/agents/subagents) · [VS Code: hooks](https://code.visualstudio.com/docs/copilot/customization/hooks)
- [Creating custom agents for Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)
