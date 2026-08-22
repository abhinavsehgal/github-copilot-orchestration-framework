# templates/workspace/ — multi-repo workspace layer (framework Chapter 12, Copilot edition)

Copy into a NEW repo (the workspace), fill the `<PLACEHOLDERS>`, run `scripts/sync-repos.sh`, open
the `.code-workspace` in VS Code.

| File | Goes to |
|---|---|
| `copilot-instructions.md.template` | `.github/copilot-instructions.md` |
| `workspace.code-workspace.template` | `<ws>.code-workspace` |
| `workspace.json.template` | `workspace.json` |
| `gitignore.template` | `.gitignore` |
| `orchestrator-agent.md.template` | `.github/agents/<ws>-orchestrator.agent.md` |
| `contract-guardian-agent.md.template` | `.github/agents/contract-guardian.agent.md` |
| `service-mapper-agent.md.template` | `.github/agents/service-mapper.agent.md` |
| `cross-repo-contracts.instructions.md.template` | `.github/instructions/cross-repo-contracts.instructions.md` |
| `delegate-skill/SKILL.md.template` | `.github/skills/delegate/SKILL.md` |
| `sync-repos.sh.template`, `delegate.sh.template` | `scripts/` |
| `SERVICE_MAP.md.template`, `CONTRACTS.md.template` | `docs/ai-context/` |
| (from the main templates) `HANDOFF_SCHEMA.md.template` | `docs/ai-context/HANDOFF_SCHEMA.md` |

Prerequisite: every child repo already has its own layer-1 install with a working
`<repo>-orchestrator` (`.github/agents/<repo>-orchestrator.agent.md`). The workspace cannot help a
repo that has nothing to delegate to.
