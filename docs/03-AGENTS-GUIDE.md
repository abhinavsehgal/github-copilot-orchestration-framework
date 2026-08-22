# 03 — Agents Guide (GitHub Copilot)

How to design the orchestrator and specialists for your project using GitHub Copilot's [custom agents](https://docs.github.com/en/copilot/reference/custom-agents-configuration) feature.

## The orchestrator

Every project has exactly **one** orchestrator. Naming convention: `<project-name>-orchestrator` (e.g. `acme-orchestrator`, `payments-platform-orchestrator`).

### Responsibilities

- Read incoming task. Restate it.
- Identify affected roles / domains.
- Read minimum docs (`docs/ai-context/INDEX.md` → 1-3 orientation maps).
- Pick the right specialists (typically 1-3 per task).
- Issue structured handoffs with claims + evidence + failure_condition.
- Receive returns. Verify rejected_claims. Re-verify unverified_claims before propagating.
- Aggregate findings into a final summary with the project's "Definition of Done."

### Constraints

- **Restricted tool set** — primarily Read/Search tools, NOT edit tools. The orchestrator coordinates; specialists implement.
- **`disable-model-invocation: false`** — auto-routable when the task is complex.
- **`user-invocable: true`** — explicitly selectable from agent dropdown.

### Frontmatter template

```yaml
---
name: <project>-orchestrator
description: Main task dispatcher for <Project>. Use for any medium/complex task. Picks the minimum docs and specialists; never loads everything. Reference other agents using the agent tool alias when delegating.
tools: ['codebase', 'search', 'usages', 'problems', 'changes']
agents: ['<specialist-1>', '<specialist-2>', 'qa-functional', 'security-privacy']   # VS Code subagent allowlist; ignored by the cloud agent
target: github-copilot
user-invocable: true
disable-model-invocation: false
---
```

File name: `.github/agents/<project>-orchestrator.agent.md`. (Field reference: see [Custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration) — exact tool names depend on Copilot extension version. `agents:` is documented by VS Code only — it names the agents this one may invoke as subagents, `*` meaning all; the cloud agent ignores the field, so keep the same list in the body too.)

## Specialists

A specialist owns one domain. Specialists are domain-shaped, not technology-shaped.

Bad: `react-specialist`, `typescript-specialist`. Good: `frontend-ui`, `payments`, `realtime-sync`, `compliance`.

### How many do you need?

| Project size | Specialist count |
|---|---|
| Solo project / prototype | 0-2 (just use plain Copilot Chat) |
| Small team / single product | 4-6 |
| Medium codebase / multiple roles | 6-10 |
| Large codebase / multi-product | 10-15 |

More than 15 is a smell — you're probably making them too narrow.

### Common specialists by project type

#### Web app (any stack)

| Specialist | Domain |
|---|---|
| `frontend-ui` | UI components, routing, state management, client-side interactions |
| `backend-api` | API routes, server logic, request handling |
| `database` | Schema, migrations, query layer, indexes |
| `auth-security` | Authentication, authorization, sessions, secrets |
| `qa-functional` | Browser/E2E testing, regression checks |
| `release-devops` | Build, deploy, env config, observability |

#### Mobile app

| Specialist | Domain |
|---|---|
| `mobile-ui` | Screens, navigation, gestures, platform-specific UX |
| `mobile-state` | State management, persistence, offline sync |
| `mobile-native` | Platform bridges (iOS/Android native or Flutter plugins) |
| `backend-api` | Server-side endpoints |
| `qa-mobile` | Device testing, simulator/emulator flows |

#### Backend / API service

| Specialist | Domain |
|---|---|
| `api-routes` | HTTP/gRPC route handlers |
| `domain-logic` | Core business logic |
| `data-layer` | DB access, repositories, ORM patterns |
| `integrations` | Third-party API clients |
| `observability` | Logging, metrics, tracing |
| `release-devops` | Deploy, env, secrets |

#### Cross-cutting (add to most projects)

| Specialist | Domain |
|---|---|
| `product-flow` | Acceptance criteria, cross-role behavior (LIGHT EDITS ONLY) |
| `qa-functional` | Tests across the system |
| `security-privacy` | Auth/sensitive-data review (REVIEW-ONLY) |
| `legal-compliance` | Regulatory review (REVIEW-ONLY) |
| `release-devops` | Build/deploy/cron/env |
| `context-librarian` | Maintains `.github/`, `docs/ai-context/`, archives drift |

### REVIEW-ONLY specialists

Some specialists should explicitly **not** edit code. Tag them in the description, give them only read tools (no `edit_files` / `create_file` / `apply_patch`), and document the constraint in their I CANNOT section.

Examples:
- `legal-compliance` — flags risks; never claims final legal advice
- `security-privacy` — identifies vulnerabilities; doesn't ship fixes
- `architecture-review` — reviews proposed designs; doesn't implement

Frontmatter for REVIEW-ONLY:
```yaml
---
name: legal-compliance
description: REVIEW-ONLY. Flags <list of regulations> risks. Produces attorney-checklist content. Never claims final legal advice.
tools: ['codebase', 'search', 'usages', 'problems', 'changes', 'fetch']
target: github-copilot
user-invocable: true
disable-model-invocation: false
---
```

Notably absent: edit-file tools, terminal/run tools. The harness physically prevents this agent from modifying files.

⚠ **Verify your IDE's exact tool names.** The `tools:` array names are tool identifiers Copilot recognizes — they vary slightly between extension versions and IDEs. The list above (`codebase`, `search`, `usages`, `problems`, `changes`, `fetch`) is a common read-only set. Check your IDE's agent configuration UI to see the current valid identifiers, or consult the [Copilot documentation](https://docs.github.com/en/copilot/reference/custom-agents-configuration).

### Implementation specialists

Get edit/write/terminal access plus relevant MCP server scope.

Frontmatter for an implementation specialist:
```yaml
---
name: backend-api
description: Builds and fixes API routes, server logic, request handling. Coordinates with database and auth-security specialists.
tools: ['codebase', 'search', 'usages', 'problems', 'changes', 'edit_files', 'apply_patch', 'runCommands']
target: github-copilot
user-invocable: true
disable-model-invocation: false
---
```

For specialists that need MCP server access (e.g. browser testing, database introspection), declare via `mcp-servers:`:

```yaml
---
name: qa-functional
description: Browser-based E2E testing and regression checks across roles.
tools: ['codebase', 'search', 'usages', 'changes', 'edit_files', 'apply_patch', 'runCommands']
mcp-servers:
  playwright:
    type: 'local'
    command: 'npx'
    args: ['-y', '@modelcontextprotocol/server-playwright']
    tools: ['*']
---
```

(Exact MCP server config syntax may vary — check the [Copilot MCP documentation](https://docs.github.com/en/copilot/concepts/mcp).)

## Agent body structure (every agent)

Every agent file body should include these sections in order:

```markdown
# <Agent Display Name>

<2-sentence description of the agent's domain.>

## When to use

- Bullet list of task types this agent owns

## When NOT to use

- Bullet list of task types that belong to other agents

## Required reading

1. `docs/ai-context/INDEX.md` — task → docs map
2. <area-specific orientation map>
3. **`.github/instructions/<your-instructions>.instructions.md`** is auto-loaded when editing matching paths
4. <other rules / canonical refs>

## Incoming handoff validation

<paste the standard incoming handoff validation block — see template>

## Return schema (required)

<paste the standard return schema block — see template>

## I CAN

- Bullet list of what this agent is allowed to do

## I CANNOT

- Bullet list of what this agent must refuse
- Always include: "Push to main" (or your equivalent protected branch)

## Definition of Done

1. Files changed (paths)
2. Instruction files referenced before editing
3. Tests run
4. Risks remaining
5. (any agent-specific Definition of Done items)

## Cross-links

- `<related instruction file>`
- `<related orientation map>`
- `<canonical doc>`
```

The "Incoming handoff validation" and "Return schema" sections are **identical across all specialists** — they enforce the universal handoff contract. See `templates/specialist-agent.md.template`.

## Cross-agent invocation

In Copilot, one agent can invoke another. On the cloud agent this is the `agent` tool alias, which "allows a different custom agent to be invoked to accomplish a task"; in VS Code it is the `runSubagent` tool, gated by the invoking agent's `agents:` frontmatter list:

```
@<other-specialist-name> <task-with-handoff-yaml-block>
```

Whether there is an **allowlist** depends on the surface (verified 2026-08-22):

- **VS Code:** `agents: ['<specialist-1>', …]` on the orchestrator is a real allowlist — an agent not named cannot be invoked as a subagent (`*` = all; never use `*` on an orchestrator). A subagent invoking its own subagents is off unless `chat.subagents.allowInvocationsFromSubagents` is enabled. This is the equivalent of Claude Code's `Agent(specialist-1, specialist-2)` allowlist.
- **Cloud agent (github.com):** no per-agent allowlist is documented. The `agents:` field is ignored; the orchestrator's body is the only routing table, and respecting it is documentation discipline.

Either way, declare `agents:` on the orchestrator and repeat the list in its body — one line gives you enforcement on one surface and a machine-readable contract on both.

**Specialists do not chain.** Even where the platform allows a specialist to invoke another specialist, by framework convention a specialist returns to the orchestrator with `recommended_next_agent` and lets the orchestrator issue the next handoff. A specialist that invokes another has become an un-audited orchestrator: the handoff schema, the evidence rule and the orchestrator's return-validation are all bypassed one level down. Leave `agents:` off specialists entirely (or empty). The only sanctioned nesting is orchestrator → orchestrator (Chapter 12, multi-repo), where both hops carry the full schema.

## Naming conventions

- Lowercase with hyphens: `backend-api`, `frontend-ui`, `legal-compliance`
- Filename matches `name`, with the `.agent.md` suffix: `backend-api` → `backend-api.agent.md` (a bare `backend-api.md` still loads; `.agent.md` is the current convention and is what a renamed `.chatmode.md` becomes — Pitfall 7)
- Be specific but not over-narrow: `payments` good, `stripe-webhook-handler` too narrow
- Cross-cutting concerns get their own agent: `qa-functional`, `security-privacy`, not folded into another specialist

## When to add a new specialist vs. extend an existing one

Add a new specialist when:
- The domain has its own gotchas (would warrant its own instruction file)
- Tasks in that domain frequently arrive
- The existing specialists' "I CANNOT" sections would all reject this work

Extend an existing specialist when:
- The work overlaps with an existing specialist's domain
- The new domain is small (fits in 1-2 paragraphs of the existing agent's body)
- You haven't seen 3+ tasks in this domain yet

## Anti-patterns

❌ **Technology-named specialists.** `typescript-specialist` doesn't have a clear domain.

❌ **Layer-named specialists.** `database-specialist` is fine if "database" means a domain (schema, migrations); not fine if it means "anyone touching SQL anywhere should consult me."

❌ **Specialists with overlapping `tools` and overlapping descriptions.** Confuses the auto-routing logic.

❌ **Specialists with no `tools:` field.** They inherit ALL tools, defeating defense-in-depth. Always declare `tools` explicitly.

❌ **Specialists that invoke other specialists.** VS Code allows it (via `agents:`) and the cloud agent's `agent` tool does not stop it, but a specialist that does so has become an un-audited orchestrator — the handoff schema, the evidence rule and the orchestrator's return-validation are all bypassed one level down. By framework convention a specialist returns `recommended_next_agent` and lets the orchestrator chain the work (see "Cross-agent invocation").

❌ **Specialists whose "I CAN" includes "approve production deploys."** That's the project owner's call.

❌ **Putting the orchestrator's persona in `.github/copilot-instructions.md`.** That file is loaded for inline completions and code review too — pollutes everything. Keep the orchestrator persona in `.github/agents/<project>-orchestrator.md`, separate.

## A note on cloud agent vs IDE chat agents

The same `.github/agents/<NAME>.agent.md` file works for both:
- **IDE Chat custom agents** (VS Code, JetBrains, etc.)
- **Cloud agent on github.com** (autonomous agent that takes issue/PR assignments)

Some properties may behave differently between environments:
- The cloud agent runs in true context isolation
- The IDE chat agent runs within the active Chat session's context
- The `target:` field can scope an agent to one or the other (`vscode` vs `github-copilot`)
- Some properties are VS Code-only (`agents`, `handoffs`, `argument-hint`, `hooks`): `argument-hint` and `handoffs` are explicitly not supported on the cloud agent per the VS Code docs, and `agents:` is ignored there (Pitfall 9)

For most projects, write agents that work in BOTH environments by avoiding IDE-specific properties. Use `target:` only when an agent genuinely needs to be exclusive to one environment.
