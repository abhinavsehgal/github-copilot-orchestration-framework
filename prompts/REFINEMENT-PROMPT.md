# REFINEMENT PROMPT — GitHub Copilot version

Use this prompt AFTER you've run 2-3 real tasks through the orchestrator (`@<project-slug>-orchestrator` in Copilot Chat) and want to harden the setup.

Adjust `<framework path>` to match your install location.

---

The orchestration framework has been live for [N] tasks. Now I want a hardening review. Read `<framework path>/docs/08-COMMON-PITFALLS.md` first.

## Review the following

### 1. Schema compliance

Inspect the most recent 3 orchestrated task transcripts (you can ask me for them or look at recent git history / PR descriptions). Check:

- Did the orchestrator emit the outbound `handoff:` YAML block on every delegation?
- Did each specialist return the inbound `return:` YAML block?
- Were `failure_condition` items articulated meaningfully (not just "task fails")?
- Did any specialist refuse a vague delegation? If yes, what triggered it?
- Were `do_not_pass_downstream_without_verification` items honored on subsequent hops?

For each violation found, propose a fix:
- Tighten the orchestrator's outbound discipline section
- Tighten a specialist's incoming validation section
- Add a missing instruction file
- Add a missing orientation map

### 2. Tool boundary enforcement

Check each agent file in `.github/agents/`:
- Each agent's `tools:` field is explicit (no missing field = inheriting everything = bad)
- REVIEW-ONLY agents have NO edit tools (no `edit_files`, `apply_patch`, `runCommands`)
- The tool identifiers used are valid for the current Copilot extension version
- Browser-testing specialists scope MCP via `mcp-servers:` — not by enumerating individual tools

If a specialist is doing things outside its declared `tools:` boundary, investigate whether the IDE/extension is still respecting the allowlist correctly. (Bug reports possible — file with GitHub Copilot.)

### 3. Instruction file usage

For each `.github/instructions/<NAME>.instructions.md` file:
- Is the `applyTo:` glob still matching real files in the codebase? (`find . -path '<glob>'`)
- Are any "hard rules" describing fixed bugs (the constraint no longer exists)? → propose deletion
- Did the instruction file actually auto-load during recent tasks? (Specialists should report which instructions applied — if your instructions file isn't appearing in their reports, the glob may be wrong.)
- Is the file comfortably inside the documented size budget (about 2 pages; well under 1,000 lines)? If it is growing past that, suggest splitting by domain OR adding `excludeAgent: "code-review"` for the implementation-only parts. (The old "first 4,000 characters" cap is no longer on the docs — Pitfall 2.)
- Do new instruction files need to be added based on recent production incidents?

### 4. Specialist scope drift

For each specialist, check the last 3 tasks it handled:
- Did it routinely refuse work that "almost" fit its domain? → its scope is too narrow
- Did it routinely accept work that the description doesn't really cover? → its scope is too broad
- Did it consistently delegate aspects it could have handled inline? → consider broadening
- Did it consistently get re-delegated to (orchestrator delegating to it 2-3x in a row)? → soft hop limit triggered; investigate root cause

Propose specific edits to agent definitions, NOT broader restructuring.

### 5. Doc tier hygiene

Check `docs/`:
- Are there new dated reports / sprint summaries / audit reports at `docs/` root that should move to `_archive/<YYYY-MM>/`?
- Are any orientation maps in `docs/ai-context/` over 200 lines? (Move detail to canonical doc.)
- Is the `docs/_archive/README.md` current with any new "known link rot" entries?

### 6. Cloud Agent vs IDE Chat behavior

If the team uses both:
- Are agent files written to work in BOTH contexts (no assumptions about isolation)?
- Are any `target:` fields scoping agents unnecessarily to one environment?
- Has the cloud agent been assigned tasks that turned out to need back-and-forth (those should have been IDE Chat)?

### 7. `.github/copilot-instructions.md` health

- Is the file under 200 lines (the docs' "no longer than 2 pages")?
- Is critical-for-code-review content front-loaded (good practice, not a cap)?
- Has it accumulated debug notes or sprint history that should move out?
- Does it still carry golden rules 9–12 (corrections → instruction files, deferred work written, production push freshens docs, one name per concept) and the "read `PROJECT.md` §3 + skim `LEARNINGS.md` §D" workflow step?

### 8. Hardening recommendations (yes/no per item)

Based on what you've observed, recommend whether to add:

- **`excludeAgent: "code-review"` on noisy instructions** — if code review is over-applying instructions that aren't review-relevant.
- **Tighter `tools:` allowlists** — if any specialist is using tools outside its declared scope.
- **Organization-level configuration** (Business / Enterprise) — if you have multiple repos using this framework and want shared agents.
- **`AGENTS.md` at root** — if the project uses multiple AI tools (Copilot + Claude + Gemini) and would benefit from a shared cross-AI metadata file.
- **CI check for handoff schema compliance** — custom CI script that grep's PRs for the schema and warns when missing. (A `preToolUse` hook can now check the block before the agent call proceeds — Chapter 10 — but CI remains the backstop for surfaces that don't load your hooks.)
- **Scheduled context-refactor** — automatic quarterly cleanup pass via a calendar reminder.

For each recommendation, give a one-paragraph justification.

### 9. Platform drift (v1.2.0)

The platform moves under the framework's conventions (Pitfall 20). Re-read the official pages for custom agents, Agent Skills, hooks and the Copilot CLI, and diff them against what this project's agents, instruction files, skills and `.github/hooks/*.json` assume. Three claims this framework itself made in v1.0/v1.1 are now retracted — check the project has not inherited them:

- "Copilot has no hooks" — it does (`.github/hooks/*.json`; `preToolUse` can deny, `agentStop` can block). Is any enforcement still living only in a pre-commit hook or IDE setting because of the old claim?
- "Cross-agent invocation has no allowlist" — VS Code's `agents:` is one; the cloud agent still has none. Does the orchestrator declare `agents:`? Does any specialist?
- "Prompt files are the Copilot equivalent of skills" — they are IDE-only; Agent Skills are cross-surface. Is any workflow the cloud agent or CLI needs still a prompt file only?

Also check: no `.chatmode.md` files remain (retired → `.agent.md`); the instruction-file size guidance quoted anywhere is the current one (about 2 pages / ~1,000 lines, not "4,000 characters"); any `copilot -p` delegation runs from the right directory and is granted the tools it needs. Report each stale assumption with the doc URL that contradicts it. Where the docs say nothing, write "not documented" — never guess.

### 10. Project-truth freshness (v1.2.0)

- `docs/ai-context/PROJECT.md` §3: is every row's environment state still true? Diff against the changelog since the header's verified-on date; re-stamp the header after fixing.
- `docs/ai-context/LEARNINGS.md`: any §D correction that has been violated in the last month despite being written down? That is the signal to promote it to a hook (Chapter 10).
- Backlogs: any `docs/*_BACKLOG.md` item older than a quarter with no revisit signal → propose archive or delete. Any "later" in recent PR descriptions that never reached a backlog?
- Glossary: any new name for an existing concept introduced since the last pass?
- The engineering skill (`.github/skills/<project-slug>-engineering/SKILL.md`): is it still loading (appears under `/` in chat; used by the cloud agent)? Are reports still tagging claims with the confidence classes?

## Output format

Produce a structured report:

```
# Refinement Review — <date>

## Tasks reviewed
1. <task 1 description, outcome>
2. <task 2 description, outcome>
3. <task 3 description, outcome>

## Schema compliance
[findings + proposed fixes]

## Tool boundary enforcement
[findings + proposed fixes]

## Instruction file usage
[findings + proposed fixes]

## Specialist scope drift
[findings + proposed fixes]

## Doc tier hygiene
[findings + proposed fixes]

## Cloud Agent vs IDE Chat
[findings + proposed fixes]

## .github/copilot-instructions.md health
[findings + proposed fixes]

## Hardening recommendations
- [ ] excludeAgent: "code-review" on noisy instructions — recommended? Why?
- [ ] Tighter tools: allowlists — recommended for which agents?
- [ ] Organization-level config — recommended? Why?
- [ ] AGENTS.md at root — recommended?
- [ ] CI handoff schema check — recommended?
- [ ] Scheduled context-refactor — recommended?

## Platform drift
[stale assumptions, each with the contradicting doc URL]

## Project-truth freshness
[PROJECT.md §3 diffs, violated §D corrections, stale backlog items, glossary drift]

## Proposed PR
- Files to update (paths)
- Risk assessment per file
- Verification steps
```

## Constraints

- **Do not modify any files in this pass.** Read-only review.
- **Wait for my approval per finding.** I'll say which fixes to land vs. defer.
- **Do not propose features GitHub Copilot doesn't currently support.** If you suggest a hook event, an organization-level feature, or a frontmatter field, verify it against the [current Copilot docs](https://docs.github.com/en/copilot) (and the VS Code docs for IDE-only fields) first, and quote the page. Where the docs are silent, say "not documented".

---

(End of prompt.)
