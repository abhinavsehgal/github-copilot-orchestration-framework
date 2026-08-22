# 01 — Principles

The seven principles this framework rests on. Every other doc, prompt, and template implements these.

## 1. Orchestrator-worker pattern, documented not coded

One coordinator agent reads tasks, classifies them, picks the right specialists, and aggregates findings. Specialists do focused work in their own context (where the surface allows isolation).

The orchestration is **prescriptive documentation**, not runtime code. The "orchestrator" is a `.github/agents/<project>-orchestrator.md` file with instructions. There is no orchestration service, no message queue, no central process.

This matters because:
- It works with any Copilot surface that recognizes custom agents (Chat in VS Code / JetBrains / Visual Studio, the cloud agent on github.com, the Copilot CLI)
- It's fully transparent — every rule is in version control as plain markdown
- It's reversible — delete `.github/agents/`, `.github/instructions/`, `.github/prompts/` and you're back to default Copilot
- Hard runtime enforcement (Copilot hooks — `.github/hooks/*.json`, v1.2) can be added later as a separate hardening pass

## 2. Each specialist runs with focused tool scope

Each specialist agent declares a `tools:` array in its frontmatter. Copilot enforces this at runtime — the agent physically cannot use tools not listed.

For specialists that need MCP server access (e.g. browser testing, database introspection), declare them in the `mcp-servers:` field with explicit scope.

Examples:
- A `legal-compliance` specialist gets `tools: ['codebase', 'search', 'usages']` — no edit tools at all
- A `frontend-ui` specialist gets edit tools plus a Chrome DevTools MCP for verification
- A `backend-api` specialist gets edit tools plus a database MCP for schema introspection

This is the **single most important runtime defense** the framework provides. A REVIEW-ONLY specialist cannot hallucinate code into your repo because the harness blocks the tool call.

> **Note on context isolation:** Custom agents in Copilot Chat run within the active Chat session's context (unlike Claude Code subagents which run in fully isolated context windows). The cloud agent runs in true isolation. See `docs/06-INVOCATION-MODES.md` for the practical implications.

## 3. Universal evidence rule — "no evidence, no claim"

If a claim cannot be tied to a file:line, table:column, rule, doc anchor, test, or command output, it must be marked as an assumption (`confidence: low` outbound, or moved to `unverified_claims` inbound) and cannot be passed downstream as fact.

This rule appears verbatim in:
- The handoff schema (both directions)
- The orchestrator's "outbound discipline"
- Every specialist's "incoming handoff validation"
- Every path-globbed instruction file's preamble

It is the **core defense against cascading hallucinations** across hops. An orchestrator that hallucinates "the bug is in `foo.ts:42`" must cite that path; the specialist re-verifies before editing; if wrong, the specialist returns `status: claim_rejected` with the evidence of what it actually found.

## 4. Failure condition — articulate what would prove you wrong

Every outbound handoff includes a `failure_condition` field: one sentence stating what observable evidence would prove the orchestrator's hypothesis or delegation premise wrong. If the specialist observes that condition, it stops, returns `claim_rejected`, and includes the triggering evidence.

This is the **inverse of `verify_before_acting`**. Where `verify_before_acting` says "check these before acting," `failure_condition` says "if you observe this, STOP." Together they bracket the work in falsifiable claims.

Forcing the orchestrator to articulate `failure_condition` is itself a thinking discipline — if you can't name what would falsify your hypothesis, you don't have a hypothesis, you have a guess.

## 5. Tools are runtime-enforced; everything else is documentation discipline

Copilot's harness enforces:
- `tools:` allowlists (specialist physically cannot use tools not listed in the agent file)
- `disable-model-invocation:` (prevents auto-routing to this agent)
- `user-invocable:` (controls manual selection)
- `target:` (scopes agent to specific environments — `vscode` or `github-copilot`)
- `mcp-servers:` per-agent MCP server scoping
- The `applyTo:` glob in `.github/instructions/<NAME>.instructions.md` (auto-loaded only when matching files are touched)
- Body length cap (30,000 chars)

Everything else is **documentation discipline** — the agent follows its instructions because they're in the system prompt:
- Handoff schema fields being present
- Refusal of vague delegations
- Evidence rule being followed
- `failure_condition` observation rule
- Hop limits ("3rd same-specialist delegation → escalate")
- "Read the right instruction file before editing"
- Definition of Done completeness

Don't conflate the two layers. When you say "the orchestrator can only delegate to project specialists," that's documentation in Copilot (no runtime allowlist for cross-agent invocation in current versions). When you say "the legal-compliance agent cannot edit files," that IS runtime-enforced via the `tools:` field.

## 6. Three-tier documentation

Project documentation should split into three clearly-labeled tiers:

| Tier | Location | Purpose | Read by |
|---|---|---|---|
| Orientation maps | `docs/ai-context/<topic>.md` | 50-150 lines per area; gotchas; cross-links to canonical | Agents (per task routing) |
| Canonical references | `docs/<UPPERCASE>.md` | Architecture, API, schema, business — full detail | Humans + agents (when deeper) |
| Frozen archive | `docs/_archive/<date>/` | Sprint reports, post-mortems, migration logs, snapshots | Audits only — never linked from active docs |

This separation makes drift impossible. Active docs live in the first two tiers. The archive is append-only and never referenced by agents/instructions/active docs.

The orientation maps live in `docs/ai-context/` even in a Copilot-only project — they're tool-agnostic, and using a tool-named directory (like `docs/copilot-context/`) would tie you to a single AI assistant.

## 7. Default Copilot stays default for inline completions

**Do not put the orchestrator's persona in `.github/copilot-instructions.md`.** The repository-wide instructions file is auto-loaded into EVERY Copilot interaction — including inline completions and code review. Loading the orchestrator's full persona there would:
- Slow down inline completions
- Pollute code-review prompts with delegation instructions
- Force every interaction through the orchestrator's "delegate everything" body

`.github/copilot-instructions.md` should be a **thin router** — golden rules + workflow + cross-links. The orchestrator agent (`.github/agents/<project>-orchestrator.md`) is invoked explicitly when the user picks it from the agent dropdown or types `@<project>-orchestrator` in chat.

Three invocation modes coexist:
- **Default** — inline completions + plain Chat (uses repository-wide instructions only)
- **Specialist mode** — Chat with a specific specialist agent selected
- **Orchestrator mode** — Chat with `@<project>-orchestrator` for cross-domain work

See `docs/06-INVOCATION-MODES.md`.

---

## What these principles get you

A team can work in different IDEs (VS Code / JetBrains / Visual Studio) and on github.com (cloud agent, code review) with consistent behavior. Specialists never inherit baggage from unrelated work. Orchestrator decisions are auditable (every claim cites evidence). Hallucinations don't compound across hops. New engineers see a clean `.github/` layout that tells them where to look.

The cost is upfront discipline: you spend a day writing the agent contracts, instruction files, and orientation maps. The payback is permanent — every future task benefits.
