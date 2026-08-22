# 08 — Common Pitfalls

Twenty-eight hard-won lessons. GitHub Copilot has its own customization quirks layered on top of generic multi-agent pitfalls. Read this before bootstrapping.

> Pitfalls 1–19 date from v1.0/v1.1. Pitfalls 20–28 were added in v1.2.0 after three further months of production use; Pitfalls 2, 7, 9 and 19 were rewritten in v1.2.0 because the platform facts they rested on changed (verified against the official Copilot and VS Code docs on 2026-08-22), and Pitfall 20 records which v1.0/v1.1 claims the platform has since made false.

## Pitfall 1: Putting the orchestrator's persona in `.github/copilot-instructions.md`

`.github/copilot-instructions.md` is auto-loaded into EVERY Copilot interaction — including inline completions and code review. Putting the orchestrator's full delegation prose there means:
- Inline completions get slower (the persona loads on every keystroke)
- Code review gets polluted with "delegate this to backend-api" instructions
- Every plain Chat session forces the orchestrator pattern on simple tasks

**Right answer:** Keep `.github/copilot-instructions.md` short (under 200 lines). It's a router. The orchestrator persona lives in `.github/agents/<project>-orchestrator.agent.md` and only loads when explicitly selected.

## Pitfall 2: Instruction files have a documented size budget — long files get ignored or skimmed

The current GitHub docs give two size limits for instructions (verified 2026-08-22): the code-review tutorial says to **limit any single instruction file to about 1,000 lines**, and the repository-instructions page says **instructions must be no longer than 2 pages**. Earlier versions of this framework (v1.0/v1.1) cited a "code review reads only the first 4,000 characters" rule; that sentence is **no longer on the current docs** and should not be quoted as a hard cap.

The practical failure is unchanged: a long instruction file dilutes the rules that matter, and whichever reader is under the tightest budget (code review, inline completions) sees the least of it.

**Right answer:**
- Keep each `.github/instructions/<NAME>.instructions.md` well inside the documented budget (~2 pages; never anywhere near 1,000 lines). Split by domain when a file grows.
- Front-load the rules that matter most for code review — good practice, not a hard cap.
- Use `excludeAgent: "code-review"` for instructions that aren't review-relevant, so the review reader isn't spending its budget on implementation guidance.
- Re-check the two size sentences quarterly (Pitfall 20); they have already changed once.

## Pitfall 3: Forgetting `applyTo:` makes the file useless

An instruction file with no `applyTo:` field in its frontmatter doesn't get auto-loaded by anything — it's just an orphan markdown file.

**Right answer:** Every `.github/instructions/*.instructions.md` file MUST have a non-empty `applyTo:` glob in frontmatter. If you have rules with no clear path scope, they belong in `.github/copilot-instructions.md` (the auto-loaded router) instead.

## Pitfall 4: `tools:` allowlist is your strongest defense — don't omit it

If you don't declare `tools:` in an agent's frontmatter, the agent inherits ALL tools available in the current Copilot session — including edit tools, terminal tools, and any installed MCP servers.

For REVIEW-ONLY agents (legal-compliance, security-privacy, architecture-review), this is a hidden disaster. The agent can write to your repo even though its instructions say it can't.

**Right answer:** Always declare `tools:` explicitly. For REVIEW-ONLY agents, list ONLY read-side tools. Verify by checking your IDE's agent dropdown — the agent should not be able to invoke edit tools.

## Pitfall 5: Cloud Agent vs IDE Chat agent — different contexts

The same `.github/agents/<NAME>.agent.md` file works in both, BUT:
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

Note also that prompt files are **IDE-only** (VS Code, Visual Studio, JetBrains) — the cloud agent and the Copilot CLI do not use them. A workflow that must run on those surfaces belongs in an Agent Skill (`.github/skills/<name>/SKILL.md`, Chapter 5), which is the cross-surface primitive; VS Code's own guidance for a prompt file that agents don't pick up is "convert it to an agent skill".

## Pitfall 7: `.github/chatmodes/*.chatmode.md` is RETIRED — rename to `.agent.md`

Custom chat modes (`.chatmode.md`) were the earlier name for what are now custom agents. As of the current docs (verified 2026-08-22) chat modes are **retired, not "older but fine"**: the documented location is `.github/agents/<NAME>.agent.md` (a bare `.md` name still works; `.agent.md` is current), and existing `.chatmode.md` files should be **renamed** to `.agent.md`. The agent format carries the fields chat modes lacked (`target`, `mcp-servers`, `user-invocable`, `disable-model-invocation`, `metadata`, and in VS Code `agents`, `handoffs`, `argument-hint`, `hooks`).

v1.0/v1.1 of this framework shipped a `chatmode.md.template` and told you to "keep chat modes only if you have legacy files you don't want to migrate". Don't keep them.

**Right answer:**
- New agents: `.github/agents/<NAME>.agent.md`.
- Existing `.github/chatmodes/<NAME>.chatmode.md`: `git mv` to `.github/agents/<NAME>.agent.md`, keep the frontmatter (`description`, `tools`, `model` carry over), reload the IDE (Pitfall 15), and confirm the agent appears in the dropdown.
- Don't create new `.chatmode.md` files for any reason.

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

Also runtime-enforced, since v1.2.0 (Chapter 10):
- `preToolUse` hooks in `.github/hooks/*.json` (`permissionDecision: allow | deny | ask`; exit 2 = deny)
- `agentStop` hooks that return `{"decision":"block"}` to refuse ending a turn
- `agents:` subagent allowlists — VS Code only (Pitfall 9)

**Right answer:** Be honest about which layer enforces what. Documentation discipline catches ~95% of value at ~5% of complexity. If you need hard enforcement of a documentation rule, write a `preToolUse` hook that validates the handoff block before the agent call proceeds, or an `agentStop` hook that checks the return block — `docs/10-MECHANICAL-ENFORCEMENT.md` has the contract. A custom CI check remains the backstop for surfaces that don't load your hooks.

## Pitfall 9: The agent-to-agent allowlist exists in VS Code and does not exist on the cloud agent

v1.0 of this chapter said "Copilot doesn't expose a per-agent allowlist for which other agents the orchestrator can invoke". That is now **half wrong** (verified 2026-08-22):

- **VS Code:** the `agents:` frontmatter property on a custom agent lists which agents it may invoke as **subagents** (`*` = all). That list **is** an allowlist — an agent not named cannot be invoked through the `runSubagent` tool. Nested invocation (a subagent invoking a subagent) is off unless `chat.subagents.allowInvocationsFromSubagents` is enabled.
- **Cloud agent (github.com):** GitHub's `agent` tool alias "allows a different custom agent to be invoked to accomplish a task", but the docs publish **no per-agent allowlist** for it. `agents:` is a VS Code property; the cloud agent ignores it. If the orchestrator invokes an unintended agent name there, there is no harness-level rejection.

So an `agents:` field on your orchestrator gives you real enforcement in one surface and documentation in the other. Write it either way — it costs one line and it is the only place the allowed specialist list is machine-readable.

**Right answer:**
- Put `agents: [<specialist-1>, <specialist-2>, …]` on the orchestrator (VS Code allowlist; ignored by the cloud agent). Never `*` on an orchestrator — that reopens every built-in and plugin agent.
- Keep the allowed specialist list in the orchestrator's body as well, so the cloud agent has the same routing table as prose.
- Specialists still return `recommended_next_agent` rather than invoking each other — a flat tree is auditable and the handoff schema survives every hop (Chapter 3).
- For the cloud agent, the only hard control is which agents exist at all: repository `.github/agents/` plus, on Business/Enterprise, org-level agents in the org's `.github` or `.github-private` repository.

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
- Path-matching `.github/instructions/*.instructions.md` (truncated; whether inline completions load these at all is not re-verified — Chapter 6, Mode 1)

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

If you rename `.github/agents/foo.agent.md` → `.github/agents/bar.agent.md` mid-task, IDE Copilot Chat may not pick up the change until you reload the IDE. The cloud agent picks up new agent files on the next assignment.

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
- Existing `.github/agents/<name>.agent.md` (or `<name>.md`, or a `.chatmodes/<name>.chatmode.md`) with same name as a proposed specialist → Pre-flight 2 (naming collision check) STOPS for explicit user decision per file.
- Existing `.github/instructions/` with overlapping `applyTo:` → Pre-flight 3 detects glob overlap; both files would load creating contradictory rules.
- Existing `.github/chatmodes/` heavily used → Pre-flight 5 surfaces the parallel system; user decides migrate vs coexist.

**The pre-flight workflow is non-optional** — even on apparent greenfield projects, run it. Cost is 30 seconds; benefit is never silently destroying team work.

## Pitfall 18: Forgetting that custom agents — and cloud-agent runs — are per-repository

`.github/agents/<NAME>.agent.md` is per-repo. If you have multiple repos and want shared agents, you have two options:
- Copy agent files into each repo (drift risk)
- For Business/Enterprise: define org-level agents under `/agents/` in the organization's `.github` or `.github-private` repository

Personal-tier Copilot doesn't support cross-repo agents.

The cloud agent is per-repository in a stronger sense too: per the docs, "Copilot cannot make changes across multiple repositories in one run" — one branch, one PR per task. A cross-repo change is several cloud-agent tasks (or an IDE/CLI session over several checkouts), never one.

**Right answer:** For multi-repo orgs on Business/Enterprise, define shared infrastructure at the org level. For personal/single-repo, just keep agents in the one `.github/`. For a cross-repo *workflow* (shared contracts, one orchestrator over several services), see `docs/12-MULTI-REPO-WORKSPACES.md`.

## Pitfall 19: Assuming Copilot has no hooks — or assuming every surface loads them the same way

**Retraction.** The v1.1 text of this pitfall said "Copilot does not expose programmable lifecycle events" and told you to wire automation into the IDE or a pre-commit hook instead. That is **no longer true** (verified 2026-08-22). Copilot now has lifecycle hooks — `sessionStart`, `sessionEnd`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop`, `subagentStart`, `subagentStop`, `errorOccurred`, `preCompact`, `permissionRequest`, `notification` — configured as `.github/hooks/*.json` (plus `~/.copilot/hooks/` for personal hooks). `preToolUse` can return `permissionDecision: allow | deny | ask` (exit 2 = deny, fail-closed); `agentStop` can return `{"decision":"block","reason":"…"}` to force another turn. Every correction-capture / build-gate / lint-fix / doc-freshness pattern the Claude edition runs as a hook now has a Copilot mapping — **`docs/10-MECHANICAL-ENFORCEMENT.md` carries the per-pattern translation, the hook I/O contract and the templates**. Don't rebuild them from this paragraph.

The pitfall that survives is subtler — three ways a hook you installed silently isn't running:

- **The cloud agent loads only `.github/hooks/*.json`.** VS Code additionally reads Claude-style hooks from `.claude/settings.json` (PascalCase event names), and the CLI reads `~/.copilot/hooks/`. A hook that lives only in `.claude/settings.json` works on a laptop and is absent from every cloud-agent run. Put team hooks in `.github/hooks/`.
- **Timeouts fail OPEN.** A hook that exceeds `timeoutSec` (default 30) is treated as if it had allowed the action. A build-gate that sizes its build to the timeout is a gate that opens exactly when the build is slow — see Pitfall 25 and size the gate's *own* internal cap below the hook timeout.
- **The platform-native half is still native.** `applyTo:` auto-loading on `.github/instructions/*.instructions.md` remains the right tool for rule surfacing (zero scripting, can't fail silently). A `preToolUse` hook that re-implements it is a second, weaker copy.

**Right answer:** read `docs/10-MECHANICAL-ENFORCEMENT.md` before writing any hook. Install team hooks under `.github/hooks/`, verify each one fires on the surface you care about (cloud agent, CLI, VS Code are three separate checks), and treat a timeout as "inconclusive" in the hook's own logic rather than relying on the platform's fail-open default.

This pitfall most often hits teams running both editions of the framework: the documentation discipline (instruction files, agents, handoff schema) is shared, and the enforcement layer is now shared too — but the file locations and event names differ, so a hook ported by copy-paste lands in a directory one surface never reads.

## Pitfall 20: The platform moves under your conventions — re-verify the docs every quarter

Three claims this framework made in v1.0/v1.1 became false within a few months, and one number it
quoted disappeared from the docs entirely (all re-verified against docs.github.com and
code.visualstudio.com on 2026-08-22):

- **"Copilot has no hooks."** It does — `sessionStart` through `notification`, in
  `.github/hooks/*.json`, with `preToolUse` able to deny and `agentStop` able to block. The whole
  of `docs/10-MECHANICAL-ENFORCEMENT.md` was rewritten; Pitfall 19 above is a retraction. Every
  correction-capture / build-gate / lint-fix / doc-freshness pattern now has a hook mapping.
- **"Cross-agent invocation has no allowlist."** In VS Code the `agents:` property on a custom agent
  *is* the allowlist for what it may invoke as a subagent (Pitfall 9). The cloud agent still has
  none. The framework's *design* choice — specialists return `recommended_next_agent` rather than
  chaining — still stands, because a flat tree is auditable and a deep one is not; but it is a
  choice on VS Code now, not a platform gap.
- **"Prompt files are the Copilot equivalent of skills."** They are not. Agent Skills
  (`.github/skills/<name>/SKILL.md`) are the cross-surface primitive — cloud agent, code review,
  CLI, VS Code and JetBrains — and are the Copilot equivalent of Claude Code skills. Prompt files
  are an IDE-only convenience; the VS Code docs' own advice for a prompt file an agent won't use is
  "convert it to an agent skill". Chapters 2, 3, 5 and 6 were corrected.
- **"Code review reads only the first 4,000 characters."** That sentence is gone from the current
  docs. The documented limits are now "about 1,000 lines" per instruction file and "no longer than
  2 pages" for repository instructions (Pitfall 2). Front-loading stays good practice; it is no
  longer a cap.

Also changed, quietly: chat modes are retired (Pitfall 7); the agentic `copilot` CLI replaced
`gh copilot suggest` as the headless surface (Chapter 6); VS Code reads `.claude/agents/`,
`.claude/rules/` and `.claude/settings.json` hooks by default.

**Right answer:** every framework claim about the platform carries a *verified-on* date. The
REFINEMENT prompt now includes a "platform drift" pass: re-read the custom-agents reference, the
Agent Skills page, the hooks reference and the Copilot CLI page, and diff them against Chapters 3,
5, 6 and 10. Treat a claim older than a quarter as `documented-unverified` (Chapter 11) until
re-checked. Where a Copilot equivalent of something is *not* on the current docs, say "not
documented" rather than guessing.

## Pitfall 21: An instruction file is a claim, not evidence

An instruction file documented the wrong HTTP verb for a route. Three agents trusted it, shipped a
client that called the wrong verb, and every user of that flow saw a generic "check your
connection" message for a week. The fourth agent read the route's `export` line.

Instruction files are written by people and agents at a point in time. They rot like any other
document — faster, because they are short and confident, and because `applyTo:` auto-loading puts
them in front of the agent with the authority of a system prompt.

**Right answer:** an instruction file is `documented-unverified` until you have looked at the thing
it describes. Before *relying* on a documented fact to build something, check it at the source (the
route, the schema, the config). When an instruction file is found wrong, fix it in the same turn —
never leave a known-false rule standing because "it's not my task".

## Pitfall 22: Deferred work that lives only in prose vanishes

"We'll do the retry logic in a follow-up" was said at the end of a session. No follow-up happened.
Two weeks later the missing retry was reported as a bug, investigated from scratch, and fixed — with
none of the context the first session had.

**Right answer:** a turn that names deferred work may not end until that work is appended to
`docs/<AREA>_BACKLOG.md` with what / why / effort / revisit-when (Chapter 11). The backlog id is
checked against the remote and open PRs first — two concurrent sessions took the same id on the same
day. Verbal follow-ups are not follow-ups.

## Pitfall 23: A production push that does not freshen the docs strands the next agent

A feature went to production on Tuesday. Its orientation map still described it as staging-only
behind a flag. On Thursday a fresh agent "enabled" it — and re-opened a migration that had already
been applied.

**Right answer:** every production push updates the full doc set *in the same turn*:
`docs/CHANGELOG.md` (the anchor), the affected orientation maps, the affected instruction files, and
`PROJECT.md` §3 if the production *state* changed. Documentation discipline missed this about one
push in five; it is now an `agentStop` hook (doc-freshness gate, Chapter 10 Pattern 5). A future
agent sees only what is in git.

## Pitfall 24: Correction-capture regexes false-fire on benign phrases — and the cost is trust

The correction-capture hook (Chapter 10, Pattern 2) fired on *"You already have access to it"* — a
perfectly polite sentence — because its regex matched `you already`. It also fired on a test plan
that *quoted* a correction phrase, and on a framework doc that *explained* the hook.

**Right answer:** three guards, all now in the template. (1) Anchor the frustration verbs to what
follows them (`you already (did|changed|broke)`), never bare. (2) Strip code fences, inline code and
heredoc bodies before matching — quoted text is data, not a correction. (3) Keep the loop guard
(`stop_hook_active`, which `agentStop` receives on its stdin payload) so a false fire costs one reply
("not a correction"), never a trapped session. When a false fire does happen, tighten the pattern
the same day; a hook that cries wolf gets disabled by the team within a week.

## Pitfall 25: A killed check is inconclusive, not failed

The build-gate `agentStop` hook capped the build at five minutes. On a machine also running two
other builds, a legitimate four-minute build was killed every time — and reported as *failed*, with
a warnings-only tail the agent then tried to "fix". An alarm loop with nothing to fix.

On Copilot there is a second timer to respect: the hook's own `timeoutSec` (default 30), and a hook
that exceeds it **fails OPEN by design** — the platform treats it as if it had allowed the stop.
So a build-gate has two bad shapes: one that reports a kill as a failure (the alarm loop), and one
whose build runs longer than the hook timeout (a gate that is open precisely when the build is
slow, with no signal that it opened).

**Right answer:** distinguish "killed by timeout or signal" from a non-zero exit inside the script.
A killed run has produced **no failure evidence**; exit 0 silently and let CI (which builds every PR
anyway) be the authority. Only a real non-zero exit blocks the stop. Size the gate's *own* internal
cap below the hook's `timeoutSec`, and size both to the slowest honest build on a busy machine, not
to the fastest one on an idle machine.

## Pitfall 26: Many sessions, one working directory

Three agent sessions shared one checkout. One was launched into a stale, detached snapshot of the
tree and "fixed" code that had already been rewritten. Two ran the build at once and corrupted each
other's output directory — every route 500'd with an error that read like a code bug and was pure
directory collision. Two picked the same migration number.

**Right answer:** `git fetch` and compare to the remote base *before the first edit* — the tree you
were launched in is not trustworthy. Do feature work in an isolated git worktree branched from the
fresh-fetched base — use separate git worktrees per session, and symlink the dependency directory
rather than duplicating it. Never run a build and a dev server in the same directory at once. Before
taking any sequential id (migration number, backlog id, schema version), check the remote *and*
other sessions' open PRs. (The Copilot CLI has no `--cwd` flag — run it *from* the worktree.)

## Pitfall 27: Never report a negative from a reader you have not verified can see the whole set

"The generator refused these 336 items" turned out to mean "the reader that listed the items was
silently capped at 1,000 rows and never showed the generator 25% of them." A confident negative
result, wrong, because the *reader* had a ceiling nobody checked.

Most data-access layers cap a bare query (ORMs, REST layers, search APIs — 1,000 is a common
default), and an explicit `limit` above the cap often does not raise it. No error, no truncation
flag.

**Right answer:** before reporting "none found", "it refused", or "zero exist", establish the reader's
ceiling. Page with a stable order, aggregate in the database, or use a count query. A capped read is
worse than a failed one: it looks like a confident answer.

## Pitfall 28: Two words for one thing ships bugs

The same concept was called `domain` in the database, `topic` on one component and `chapter` on the
page — and `topic` *already meant something else* in a legacy subsystem. A gate took the new
meaning, looked it up where the old meaning lived, matched nothing, and an empty match silently
widened a filter to the whole dataset. Every user of that gate got off-target results for a day.

Vocabulary drift is not cosmetic. It is how a correct-looking lookup returns the wrong set.

**Right answer:** one glossary (`docs/ai-context/GLOSSARY.md` — a rule file, not a nicety) naming
each concept exactly once, with the DB column, the type field and the UI label that carry it. Adding
a fifth name for an existing concept is a defect. Rename only while already editing the file that
carries the wrong name, and say so in the commit.
