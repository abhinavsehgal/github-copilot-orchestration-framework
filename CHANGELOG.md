# Changelog

All notable changes to the GitHub Copilot Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
