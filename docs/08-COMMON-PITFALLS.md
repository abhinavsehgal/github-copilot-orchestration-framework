# 08 — Common Pitfalls

GitHub Copilot has its own customization quirks layered on top of generic multi-agent pitfalls. Read this before bootstrapping.

## Pitfall 1: Putting the orchestrator's persona in `.github/copilot-instructions.md`

`.github/copilot-instructions.md` is auto-loaded into EVERY Copilot interaction — including inline completions and code review. Putting the orchestrator's full delegation prose there means:
- Inline completions get slower (the persona loads on every keystroke)
- Code review gets polluted with "delegate this to backend-api" instructions
- Every plain Chat session forces the orchestrator pattern on simple tasks

**Right answer:** Keep `.github/copilot-instructions.md` short (under 200 lines). It's a router. The orchestrator persona lives in `.github/agents/<project>-orchestrator.md` and only loads when explicitly selected.

## Pitfall 2: Code review reads only first 4,000 chars

Copilot code review reads only the **first 4,000 characters** of `.github/copilot-instructions.md` and each `.github/instructions/<NAME>.instructions.md`.

If you put critical-for-code-review rules at the bottom of a long instruction file, code review won't see them.

**Right answer:**
- Front-load code-review-relevant rules in each instruction file
- Or use `excludeAgent: "code-review"` for instructions that aren't review-relevant (so you can use the full file body for Chat without confusing code review)
- If code review really needs detailed guidance, split into multiple smaller instruction files

## Pitfall 3: Forgetting `applyTo:` makes the file useless

An instruction file with no `applyTo:` field in its frontmatter doesn't get auto-loaded by anything — it's just an orphan markdown file.

**Right answer:** Every `.github/instructions/*.instructions.md` file MUST have a non-empty `applyTo:` glob in frontmatter. If you have rules with no clear path scope, they belong in `.github/copilot-instructions.md` (the auto-loaded router) instead.

## Pitfall 4: `tools:` allowlist is your strongest defense — don't omit it

If you don't declare `tools:` in an agent's frontmatter, the agent inherits ALL tools available in the current Copilot session — including edit tools, terminal tools, and any installed MCP servers.

For REVIEW-ONLY agents (legal-compliance, security-privacy, architecture-review), this is a hidden disaster. The agent can write to your repo even though its instructions say it can't.

**Right answer:** Always declare `tools:` explicitly. For REVIEW-ONLY agents, list ONLY read-side tools. Verify by checking your IDE's agent dropdown — the agent should not be able to invoke edit tools.

## Pitfall 5: Cloud Agent vs IDE Chat agent — different contexts

The same `.github/agents/<NAME>.md` file works in both, BUT:
- The cloud agent runs in true context isolation (fresh environment per task)
- The IDE chat agent runs within the active Chat session's context — it can see prior messages

If your specialist's body assumes the cloud agent's isolation (e.g. "you have no prior context, start fresh"), it'll behave oddly in IDE chat where prior context exists.

**Right answer:** Write specialist bodies that work in BOTH contexts:
- Don't assume isolation; the agent might inherit Chat context
- Don't assume Chat context; the agent might run cleanly in cloud
- Use explicit handoff schema (`handoff_id`, `previous_specialist`, etc.) so the agent isn't relying on implicit conversation memory

## Pitfall 6: Prompt files are public preview — features may shift

`.github/prompts/*.prompt.md` is in **public preview** as of late 2025 / early 2026. Frontmatter fields and invocation behavior may change in future Copilot releases.

**Right answer:** Stick to documented frontmatter fields (`description`, `agent`). Avoid IDE-specific extensions you can't verify. When prompt files leave preview, check the [release notes](https://docs.github.com/en/copilot/whats-new) for any breaking changes.

## Pitfall 7: `.github/chatmodes/*.chatmode.md` is the older format

Custom chat modes (`.chatmode.md`) predate custom agents (`.agent.md`). `.agent.md` is the newer/preferred format with more metadata fields (mcp-servers, target, user-invocable, disable-model-invocation).

**Right answer:** For new projects, use `.github/agents/<NAME>.md`. Don't create new `.chatmode.md` files unless you specifically need a chat-mode-only feature.

## Pitfall 8: Treating documentation enforcement as runtime enforcement

The handoff schema, the universal evidence rule, the `failure_condition` observation, the soft hop limits — these are **documentation discipline**. They work because agents follow their instructions. They do NOT physically prevent a misbehaving agent from emitting a malformed handoff.

What IS runtime-enforced (by the Copilot harness):
- `tools:` allowlists per agent
- `mcp-servers:` per-agent MCP scoping
- `applyTo:` globs in instruction files (auto-loading conditional on path match)
- `target:` field (environment scoping for `vscode` vs `github-copilot`)
- `disable-model-invocation` and `user-invocable` (agent visibility/routing)
- Body length cap (30,000 chars)

What is NOT runtime-enforced:
- The handoff YAML block being present
- The evidence rule being followed
- Refusal of vague delegations
- Hop limits ("3rd same-specialist delegation → escalate")
- Definition of Done completeness

**Right answer:** Be honest about which layer enforces what. Documentation discipline catches ~95% of value at ~5% of complexity. If you need hard enforcement of a documentation rule, you'd need a custom CI check, a webhook, or wait for Copilot to expose pre-tool-use hooks.

## Pitfall 9: Cross-agent invocation has no allowlist

Unlike some other AI tools, Copilot doesn't expose a per-agent allowlist for which other agents the orchestrator can invoke. The orchestrator's body says "I delegate to backend-api / frontend-ui / etc." — but if it tries to invoke a non-existent or unintended agent name, there's no harness-level rejection.

**Right answer:**
- Document the allowed specialist list explicitly in the orchestrator's body
- If the orchestrator tries to invoke an unknown specialist, treat it as a documentation bug
- For hard enforcement (Business / Enterprise tiers), use organization-level agent definitions to control which agents exist at all

## Pitfall 10: MCP server auth lives in `${{ secrets.* }}`

Copilot MCP server config supports environment variable substitution from secrets:

```yaml
mcp-servers:
  some-mcp:
    type: 'local'
    command: 'some-tool'
    env:
      API_KEY: ${{ secrets.SOME_MCP_API_KEY }}
```

Don't hardcode API keys in `.github/agents/*.md` files. Use the secrets reference syntax. Configure secrets at the user, repository, or organization level depending on scope.

## Pitfall 11: `AGENTS.md` is a "lowest common denominator" file

`AGENTS.md` (or `CLAUDE.md`, `GEMINI.md`) is recognized by Copilot as a fallback/cross-AI metadata file. Per the docs, it's "currently not supported by all Copilot features."

**Right answer:** Don't put critical Copilot-specific config in `AGENTS.md`. Use `.github/copilot-instructions.md` for Copilot-specific routing. Use `AGENTS.md` only for shared cross-AI metadata that you want all AI tools (Copilot + Claude + Gemini) to read.

## Pitfall 12: Inline completions only see the first ~few thousand chars of context

Inline completions are latency-sensitive. The model context for inline completions is intentionally smaller than Chat. Don't expect inline completions to know about complex orchestration patterns — they only see:
- The current file's open content
- Some surrounding files (depending on IDE configuration)
- `.github/copilot-instructions.md` (if present, but truncated for latency)
- Path-matching `.github/instructions/*.instructions.md` (truncated)

**Right answer:** Optimize `.github/copilot-instructions.md` for inline completions: lead with the most important constraints in the first 500 chars. The orchestration / agent routing details belong in agent files (which inline completions don't load anyway).

## Pitfall 13: Ignoring `.env*` history

If env files with secrets are already in git history, removing them from the working tree doesn't remove the secrets from history.

**Right answer:**
1. Rotate every credential that was in any leaked env file.
2. Use `git-filter-repo` or BFG Repo-Cleaner to scrub history.
3. Force-push the scrubbed history (coordinate with the team).
4. Add broad gitignore patterns: `.env*.bak*`, `.env*staging_tmp`, `.env*tmp`.

## Pitfall 14: Letting `docs/` rot without a librarian

Without active maintenance, `docs/` accumulates: sprint reports, audit reports referenced by no one, outdated decisions, etc. After a year, engineers can't tell which are authoritative.

**Right answer:** Have a `context-librarian` specialist whose job is exactly this. Schedule a quarterly cleanup pass. Move stale material to `docs/_archive/<YYYY-MM>/`.

## Pitfall 15: Reorganizing `.github/` config without checking IDE behavior

If you rename `.github/agents/foo.md` → `.github/agents/bar.md` mid-task, IDE Copilot Chat may not pick up the change until you reload the IDE. The cloud agent picks up new agent files on the next assignment.

**Right answer:**
- Reload your IDE after structural changes to `.github/`
- Verify with the agent dropdown — the renamed agent should appear
- For team-wide changes, communicate so others reload too
- Avoid renames mid-PR; do structural changes in their own PR

## Pitfall 16: One huge instructions file instead of scoped files

Putting all rules into `.github/copilot-instructions.md` defeats the purpose of `.github/instructions/`. Repository-wide instructions load on every interaction; path-globbed instructions load only when relevant.

**Right answer:** Refactor a long `.github/copilot-instructions.md` into:
- Keep router content (golden rules + workflow + cross-links) in `.github/copilot-instructions.md` (target < 200 lines)
- Move per-area gotchas to `.github/instructions/<domain>.instructions.md` (with `applyTo:` globs)
- Move long architecture explanations to `docs/<UPPERCASE>.md`
- Move sprint history to `docs/_archive/`

## Pitfall 17: Bootstrap on a repo with existing Copilot config

If you ran VS Code's `/init` previously, or your team has been adding to `.github/copilot-instructions.md` for months, BOOTSTRAP-PROMPT.md must NOT silently overwrite that work.

**Right answer:** the prompt now includes mandatory pre-flight checks at the top — snapshot to `.github-pre-bootstrap-backup/`, naming-collision detection, `applyTo:` glob conflict detection, drift detection on existing instructions, and a decision gate that STOPS if any pre-flight raised a `<NEEDS USER CONFIRMATION>` flag.

**Specific risks:**
- Existing `.github/copilot-instructions.md` with team rules → Pre-flight 4 (drift detection) flags stale content; Step 11 (merge step) shows a 3-pane diff before writing.
- Existing `.github/agents/<name>.md` with same name as a proposed specialist → Pre-flight 2 (naming collision check) STOPS for explicit user decision per file.
- Existing `.github/instructions/` with overlapping `applyTo:` → Pre-flight 3 detects glob overlap; both files would load creating contradictory rules.
- Existing `.github/chatmodes/` heavily used → Pre-flight 5 surfaces the parallel system; user decides migrate vs coexist.

**The pre-flight workflow is non-optional** — even on apparent greenfield projects, run it. Cost is 30 seconds; benefit is never silently destroying team work.

## Pitfall 18: Forgetting that custom agents are per-repository

`.github/agents/<NAME>.md` is per-repo. If you have multiple repos and want shared agents, you have two options:
- Copy agent files into each repo (drift risk)
- For Business/Enterprise: use `.github-private/agents/<NAME>.md` at the org level

Personal-tier Copilot doesn't support cross-repo agents.

**Right answer:** For multi-repo orgs on Business/Enterprise, define shared infrastructure in `.github-private/`. For personal/single-repo, just keep agents in the one `.github/`.

## Pitfall 19: Expecting a Stop hook or PostToolUse hook in Copilot

Copilot does **not** expose programmable lifecycle events. There is no equivalent of Claude Code's `PreToolUse` / `PostToolUse` / `Stop` hooks. If you've come from Claude Code and are looking for "the place to wire correction-capture / build-gate / lint-fix automation," it doesn't exist on Copilot's surface.

What Copilot has instead:

- **Declarative auto-loading** via `applyTo:` frontmatter on `.github/instructions/*.instructions.md` — the platform-native equivalent of Claude Code's `PreToolUse` rule-surfacing hook. Strictly better in one way (zero scripting, can't fail silently).
- **Manually-invoked prompts** via `.github/prompts/*.prompt.md` — these cover what would be slash commands on Claude Code, and they substitute for some Stop-hook patterns when invoked explicitly (see `docs/10-MECHANICAL-ENFORCEMENT.md`).
- **Definition-of-Done discipline** in agent instructions — substitutes for the build-gate Stop hook, but requires the orchestrator to actually validate the return-block contract.
- **IDE-level auto-fix and pre-commit hooks** — substitute for the `PostToolUse` lint-fix pattern.

**Right answer:** read `docs/10-MECHANICAL-ENFORCEMENT.md` for the per-pattern translation. Don't try to build Claude Code-style hook scripts and wire them into Copilot — there's no event to wire them to. The right place for "after every edit, run X" is the IDE config or a Husky/lint-staged pre-commit hook, not the Copilot framework.

This pitfall most often hits teams who adopted the Claude Orchestration Framework first and are now adopting the Copilot one alongside it for engineers who prefer Copilot. The two frameworks share the documentation discipline (rules, agents, handoff schema) but diverge on enforcement layer.
