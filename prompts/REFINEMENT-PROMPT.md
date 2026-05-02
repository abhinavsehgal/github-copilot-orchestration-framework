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
- Is the file < 4,000 chars (so code review reads all of it)? If > 4,000 chars, suggest splitting OR adding `excludeAgent: "code-review"`.
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

- Is the file under 200 lines?
- Is critical-for-code-review content in the first 4,000 characters?
- Has it accumulated debug notes or sprint history that should move out?

### 8. Hardening recommendations (yes/no per item)

Based on what you've observed, recommend whether to add:

- **`excludeAgent: "code-review"` on noisy instructions** — if code review is over-applying instructions that aren't review-relevant.
- **Tighter `tools:` allowlists** — if any specialist is using tools outside its declared scope.
- **Organization-level configuration** (Business / Enterprise) — if you have multiple repos using this framework and want shared agents.
- **`AGENTS.md` at root** — if the project uses multiple AI tools (Copilot + Claude + Gemini) and would benefit from a shared cross-AI metadata file.
- **CI check for handoff schema compliance** — custom CI script that grep's PRs for the schema and warns when missing.
- **Scheduled context-refactor** — automatic quarterly cleanup pass via a calendar reminder.

For each recommendation, give a one-paragraph justification.

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

## Proposed PR
- Files to update (paths)
- Risk assessment per file
- Verification steps
```

## Constraints

- **Do not modify any files in this pass.** Read-only review.
- **Wait for my approval per finding.** I'll say which fixes to land vs. defer.
- **Do not propose features GitHub Copilot doesn't currently support.** If you suggest hooks, organization-level features, or new frontmatter fields, verify against the [current Copilot docs](https://docs.github.com/en/copilot) first.

---

(End of prompt.)
