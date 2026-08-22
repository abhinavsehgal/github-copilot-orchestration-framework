# templates/workspace/ — multi-repo workspace layer (framework Chapter 12, Copilot edition)

Copy into a NEW repo (the workspace). **Short path:** copy `workspace.json.template` to `workspace.json`, fill it, then run `bash bootstrap.sh.template <framework path>` in the new repo — it copies every file below into place, fills the per-repo placeholders, generates `.gitignore` and the `.code-workspace` folder list, fills the orchestrator's `agents:` allowlist and the service-map rows, and lists what is left to fill by hand. Then `scripts/sync-repos.sh` and open the `.code-workspace` in VS Code. **Long path:** the table below.

| File | Goes to |
|---|---|
| `bootstrap.sh.template` | (run it — creates everything below) |
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
