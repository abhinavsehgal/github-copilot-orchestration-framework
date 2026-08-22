# Changelog

All notable changes to the GitHub Copilot Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.2.0] — 2026-08-22

### Retracted — the platform moved (verified against docs.github.com + code.visualstudio.com, 2026-08-22)

Three claims this framework made in v1.0/v1.1 are no longer true, and one number is gone:

- **"Copilot has no programmable lifecycle hooks."** It does: `.github/hooks/*.json` with `sessionStart`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop`, `subagentStart/Stop`, `errorOccurred`, `preCompact`, `permissionRequest`, `notification` — on the cloud agent (reads only `.github/hooks/*.json`), the Copilot CLI and VS Code (which also reads a Claude-style `.claude/settings.json`). `preToolUse` returns `permissionDecision` (fail-closed on non-zero exit); `agentStop` can return `{"decision":"block"}` and receives `stop_hook_active`; timeouts fail open. **Chapter 10 is rewritten** around this contract. Pitfall 19 is retracted and rewritten.
- **"Cross-agent invocation has no allowlist."** VS Code's `agents:` frontmatter *is* an allowlist of subagents (`*` = all), invoked via `runSubagent`, with nesting behind `chat.subagents.allowInvocationsFromSubagents`; GitHub's `agent` tool alias exists on the cloud agent (no allowlist there). Pitfall 9 rewritten.
- **"Prompt files are Copilot's skills."** Agent skills (`.github/skills/<name>/SKILL.md`, also `.claude/skills/`) are the documented cross-surface primitive — cloud agent, code review, CLI, VS Code, JetBrains. Prompt files are IDE-only, and VS Code says to convert a prompt to a skill for the Agent Host. Chapters 2, 5, 6 and the README corrected.
- The "code review reads only the first 4,000 characters" cap is no longer on the current docs; current guidance is "about 1,000 lines per instruction file" / "no longer than two pages". Pitfall 2 rewritten. Chat modes are retired (rename `.chatmode.md` → `.agent.md`); Pitfall 7 rewritten.

### Added — what three more months of production use taught

- **`docs/11-PROJECT-TRUTH-AND-LEARNINGS.md`** — three knowledge stores a fresh agent reads first (`PROJECT.md` with a date-stamped "what is live where" table; `LEARNINGS.md` with decisions / failed approaches / bug patterns / agent corrections; per-area backlogs), "deferred work must be written, not spoken", "every production push freshens the docs in the same turn", the six-gate engineering playbook, the evidence-confidence taxonomy, the proof ladder, and the multi-client parity rules.
- **`docs/12-MULTI-REPO-WORKSPACES.md`** — web + mobile + microservices in separate repos: three layers (per-repo install → org-level agents → workspace repo with a `.code-workspace`, a manifest and gitignored clones, no CI), the verified per-surface table of what loads from several folders, three delegation mechanisms (the child's own `copilot -p` session; VS Code subagents; one cloud-agent task per repo), the name-collision hazard and how the design avoids it, two additive handoff fields, the cross-repo contract protocol, a one-afternoon POC recipe. Answers "do we need a fourth framework?" — no.
- **Pitfalls 20–28:** platform drift (re-verify quarterly); an instruction file is a claim, not evidence; deferred work in prose vanishes; production push without doc freshening; correction-regex false positives; a killed check is inconclusive (and Copilot hook timeouts fail open); many sessions on one working directory; never report a negative from a capped reader; two words for one thing ships bugs.
- **Templates:** `PROJECT.md`, `LEARNINGS.md`, `BACKLOG.md`, `GLOSSARY.md`, `engineering-playbook-skill.md` (as an agent skill); `hooks/` (`hooks.json`, `hook-io.mjs`, `correction-detect.mjs`, `doc-freshness-track.mjs`, `lint-fix.mjs`, `stop-gate.mjs` — Patterns 2–5 on the real Copilot contract, flag-file design so no undocumented transcript format is parsed); `skills/` (`commit-push-pr`, `correction-capture`, `verify-build` — cross-surface successors of the v1.1 prompt files); `workspace/` (the whole layer 3: `.code-workspace`, `workspace.json`, router, orchestrator + `contract-guardian` + `service-mapper` agents, `cross-repo-contracts.instructions.md`, `/delegate` skill, `sync-repos.sh`, `delegate.sh`, service map, contracts doc).
- **Chapter 4:** optional additive fields `repo`, `contract_impact`, `contracts_changed`, `deferred_work`; evidence-confidence class on claims. `schema_version` stays 1.
- **Chapter 6:** the agentic Copilot CLI (`copilot -p`, `--agent=`, `--allow-tool=`) as a mode; surface matrix gains Skills and Hooks rows.
- **Prompts:** BOOTSTRAP Step 11 generates the project-truth set; REFINEMENT gains §8 platform drift and §9 project-truth freshness.
- **Root router template** gains golden rules 9–12 (corrections → instruction files; deferred work written; production push freshens docs; one name per concept).

### Changed

- `templates/*-agent.md.template` — file name `.agent.md`; the orchestrator template carries `agents:` (VS Code subagent allowlist; ignored by the cloud agent).
- Chapter 3 — specialists still return `recommended_next_agent` rather than chaining: a framework convention for auditability now that nesting is possible, not a platform limit.
- README — comparison table with Claude Code rebuilt (hooks, rules, skills, headless, multi-repo rows); FAQ corrected (the repo is public; bootstrap does not replace an existing router; chat modes retired; prompt files vs skills).

### Known gap

- `GitHub-Copilot-Orchestration-Framework.pdf` is still the v1.1.2 render. Chapters 10 (rewritten), 11 and 12 are markdown-only until regenerated.

### Provenance

Same production codebase as v1.0/v1.1, which runs this framework and the Claude Code edition side by side on one corpus of rules/agents/skills (VS Code reads `.claude/*` locations natively — a supported dual-tool setup). Every platform claim was re-verified on 2026-08-22 and carries that date, because Pitfall 20 is the lesson that made this release necessary. Everything project-specific was stripped: if a lesson could not be restated as "any team, any stack, any domain hits this", it stayed out.

---

## [1.1.2] — 2026-05-06

### Companion release to claude-orchestration-framework v1.1.2

The Copilot framework has **no behavioral changes** in this release because GitHub Copilot doesn't expose programmable lifecycle hooks (Stop / PreToolUse) — see Pitfall 19. The fix that triggered the matched-version bump applies to `claude-orchestration-framework`'s `templates/hooks/correction-capture-prompt.mjs.template` and `build-gate.mjs.template` only.

### Added — cross-framework lesson

- **`docs/10-MECHANICAL-ENFORCEMENT.md`** updated with a sidebar note: *"If you're working on the Claude side of a dual-framework setup, the Stop hook IO contract matters — stderr surfaces, stdout doesn't. The Copilot side has no equivalent because Copilot has no programmable hook events."* Reduces the chance that an adopter using both frameworks ports a buggy pattern between them.

### Why bump the version anyway

Matched version numbers across the two companion frameworks make it easier to communicate compatibility ("if you're on Claude framework v1.1.2, use Copilot framework v1.1.2"). No behavioral change here, just a sync-point release.

---

## [1.1.0] — 2026-05-06

### Added — mechanical enforcement patterns

- **`docs/10-MECHANICAL-ENFORCEMENT.md`** — new chapter mapping the four Claude Code hook patterns (`PreToolUse` rule-surfacing, `Stop` correction-capture, `Stop` build-gate, `PostToolUse` lint-fix) onto Copilot's surface area. The honest answer per pattern:
  - **Rule surfacing** translates **natively and strictly** via `applyTo:` frontmatter on instruction files (already in the framework).
  - **Correction capture** translates **partially** as a manually-invoked prompt — Copilot has no Stop event.
  - **Build-gate** translates **partially** via Definition-of-Done discipline + an explicit verification prompt.
  - **Lint-fix on edit** does **not** translate — belongs in the IDE config or pre-commit hooks.

- **`templates/correction-capture.prompt.md.template`** — drop-in `/correction-capture` slash prompt. Walks Copilot through the 5-step "is this a recurring rule? draft a `.github/instructions/<file>.instructions.md` patch, get user approval, apply" workflow. Forbids the assistant from saying "I'll remember" — only patches.
- **`templates/commit-push-pr.prompt.md.template`** — drop-in `/commit-push-pr` slash prompt. 9-step workflow that codifies the project's golden rules (no main pushes, build before commit, no force, no auto-merge, no `.env*` staging) at the slash-prompt boundary. Stack-agnostic with explicit `<DEFAULT_BASE_BRANCH>` / `<BUILD_RELEVANT_GLOB>` / `<BUILD_COMMAND>` / `<PROJECT_TRAILER>` placeholders.

- **Pitfall 19** in `docs/08-COMMON-PITFALLS.md` — Copilot has no programmable lifecycle hooks. Teams arriving from Claude Code expecting a `Stop` / `PostToolUse` event will not find one. The pitfall enumerates what Copilot has instead and where each Claude Code hook pattern goes on the Copilot surface.

### Changed

- **README** — bumped to v1.1.0; refreshed "What's in the box" tree to include chapter 10 + the two new prompt templates; "8 lessons → 19 lessons" notation in the pitfalls reference.

### Why this is a smaller update than the Claude framework's v1.1

GitHub Copilot's design favours declarative auto-loading over programmable hooks. Two of the four Claude Code hook patterns translate; one is partial; one belongs at a different layer of the stack (IDE config / pre-commit). The framework reflects that — we ship two new prompt templates and one new chapter, not a `templates/hooks/` directory.

The shared philosophy across both frameworks remains: documentation discipline first; mechanical enforcement layered on top **only when documentation has demonstrably failed**. The two enforcement layers diverge by platform, but the discipline is the same.

### Provenance

The hook patterns originated on Claude Code (creator: Boris Cherny, Anthropic) and were field-tested on a production K-12 SaaS codebase that runs both Claude Code and GitHub Copilot side by side. The "deterministic scaffolding" framing is from Boris's [Lenny's Podcast appearance](https://www.lennysnewsletter.com/p/head-of-claude-code-what-happens) (Feb 2026). The translation to Copilot's surface was tested against Copilot's documented features — `applyTo:` frontmatter, prompt files, custom agents, and the Cloud Agent — without using any undocumented or extension-based functionality.

---

## [1.0.0] — 2026-05-01

### Added

- Initial public release.
- Nine-chapter doc set covering principles, architecture, agents, handoff schema, instructions + prompts, invocation modes, folder structure, common pitfalls, runbook.
- Templates for orchestrator agent, specialist agent, REVIEW-ONLY agent, handoff schema, INDEX, spoonfeeder, copilot-instructions root router, instructions file, prompt file, chatmode, archive README.
- Three bootstrap prompts: INVENTORY, BOOTSTRAP, REFINEMENT.
- Pre-flight safety pass for brownfield bootstrap on repos with existing Copilot configuration.
- Coverage of all five Copilot customization surfaces — `.github/copilot-instructions.md`, `.github/instructions/`, `.github/prompts/`, `.github/agents/`, `.github/chatmodes/` — with no Marketplace or extension dependencies.
