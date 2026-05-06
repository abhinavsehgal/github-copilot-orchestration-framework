# 10 — Mechanical Enforcement Patterns

GitHub Copilot does not expose programmable lifecycle hooks (no equivalent of Claude Code's `PreToolUse` / `PostToolUse` / `Stop`). What it has instead is **declarative auto-loading via `applyTo:` frontmatter** — which already covers the most important of the four hook patterns from the Claude side: rule surfacing.

This chapter:

1. Maps the Claude Code hook patterns onto Copilot's surface area, calling out which translate cleanly and which don't.
2. Provides ready-to-paste `.github/prompts/*.prompt.md` templates for the patterns that DO translate (correction-capture, commit-push-pr).
3. Explains how the orchestrator's Definition of Done section can carry the build-gate enforcement that has no native trigger.

Read after `docs/05-INSTRUCTIONS-AND-PROMPTS.md` (which covers the basic mechanics of `applyTo:` and prompt files).

This chapter is **optional**. Skip it if your project hasn't yet experienced documentation discipline failing in real-world use. Come back when concrete evidence shows agents are skipping rules, corrections aren't hardening, or builds are landing broken.

---

## Translation table — Claude Code hook patterns → Copilot equivalents

| Claude Code pattern | Copilot equivalent | Translation quality |
|---|---|---|
| `PreToolUse` rule-surfacing (auto-inject matching rule body before edit) | `.github/instructions/<NAME>.instructions.md` with `applyTo:` glob | **Native and strict.** Copilot auto-loads matching instruction files when editing matching paths. This is built into the platform — no scripting required. The framework's existing `instructions.md.template` IS this pattern. |
| `Stop` correction-capture (block stop + draft rule patch when user corrects) | `.github/prompts/correction-capture.prompt.md` (manual workflow) | **Partial.** Copilot has no Stop event, so the workflow must be invoked explicitly. The prompt template below codifies the same checklist for use after a correction. |
| `Stop` build-gate (run build before stop; block if broken) | Orchestrator's Definition of Done + `.github/prompts/verify-build.prompt.md` | **Partial.** No automatic trigger. Enforcement lives in the agent's instructions ("the build must be green before claiming done") and in a prompt the orchestrator runs as the last step. |
| `PostToolUse` lint-fix (auto-fix linter after every edit) | IDE auto-fix on save / pre-commit hook | **Out of scope.** This belongs in the IDE config or in a Husky/lint-staged pre-commit, not in Copilot's surface area. |
| `/commit-push-pr` slash command (daily commit→PR loop) | `.github/prompts/commit-push-pr.prompt.md` | **Native.** Slash prompts in Copilot map 1:1 to slash commands in Claude Code. |

The takeaway: Copilot's design favours declarative auto-loading over imperative event hooks. Two of the four patterns translate cleanly; one is partial; one belongs elsewhere on the stack.

---

## Pattern 1 — Rule surfacing (already in the framework)

The `templates/instructions.md.template` you got at install IS this pattern. Recap:

```yaml
---
applyTo: "src/components/**/*.tsx,src/lib/avatars.ts"
---

# Frontend / UI Instructions

## Hard rules

### Always use `<img>` + `resolveAvatarSrc` for `profile.avatar` — never `next/image`

**Why:** ...

**How to apply:** ...
```

When a developer (or Copilot's autonomous agent) edits any matching file, Copilot auto-loads the instruction body into context. **No script. No hook. No restart.** This is what makes Copilot's instructions surface qualitatively different from Claude Code's static `CLAUDE.md` — the auto-loading is built into the platform.

What you DO need to discipline:

- Keep instruction files ≤ 150 lines / ~3,000 chars (Copilot code-review reads only the first ~4,000 chars)
- Use `applyTo:` globs that actually match the files being edited (verify by grepping your codebase against each glob)
- Front-load code-review-relevant rules (per the 4,000-char cap)
- Have a `context-librarian` specialist whose job includes auditing instruction `applyTo:` overlap quarterly

---

## Pattern 2 — Correction-capture (manual prompt)

In Claude Code, a Stop hook can detect strong correction signals in the user's message and force a rule patch. Copilot has no Stop event, so the equivalent is a **prompt the developer (or orchestrator) invokes explicitly after a correction**.

A template ships at `templates/correction-capture.prompt.md.template`. Copy it to `.github/prompts/correction-capture.prompt.md` in your project. After any correction that feels like a recurring pattern, type:

```
/correction-capture
```

The prompt walks Copilot through the same 3-step checklist the Claude `Stop` hook auto-injects:
1. Was this a correction on a recurring domain pattern?
2. If yes — draft a one-line `.github/instructions/<file>.instructions.md` patch under "Hard rules" or "What NOT to do." Name the exact file. Get user approval.
3. If no — acknowledge briefly.

The discipline is "no `I'll remember`, only patches." Memory is not a substitute for an instruction file.

The prompt is opt-in (the user types it) rather than opt-out (a hook fires automatically). That's the platform's reality. The trade-off:

- ✅ No false positives from regex misfires
- ✅ Works the same across every Copilot surface (VS Code, JetBrains, Cloud Agent, CLI)
- ❌ Requires the developer to remember to invoke it
- ❌ Easier to skip than the Claude Code Stop hook

To minimize the discipline cost, encode the `correction-capture` invocation step into your orchestrator agent's "incoming feedback" instructions: "if the user issues a correction, immediately run the `correction-capture` prompt before continuing."

---

## Pattern 3 — Build-gate (orchestrator Definition of Done)

Copilot has no Stop hook, but every specialist agent's Definition of Done can include a "build is green" requirement. The orchestrator enforces this on receipt of a `return:` block — if `tests_run` doesn't include the build command and its result, the orchestrator pushes back instead of proceeding.

Concrete enforcement:

1. **In the orchestrator's incoming-validation step** (per `docs/04-HANDOFF-SCHEMA.md`):
   ```yaml
   - if return.status == "completed" but tests_run does not include `npm run build` (or your project's build command), reject with status: incomplete
   ```

2. **In every implementation specialist's Definition of Done section:**
   ```markdown
   ## Definition of Done
   1. ...
   N. Run the build: `npm run build`. Include the command + result in `tests_run`. Fail the handoff if the build doesn't pass.
   ```

3. **As a `/verify-build.prompt.md`** the orchestrator can invoke explicitly when in doubt:
   ```
   /verify-build
   ```
   Walks Copilot through running the build, capturing the output tail, and refusing to mark the parent task done if it fails.

The pattern relies on the orchestrator catching the violation on the return path rather than on a runtime trigger. Less mechanical than the Claude Code Stop hook, but enforceable as long as the orchestrator's validation step is taken seriously.

---

## Pattern 5 — `/commit-push-pr` (native to Copilot prompts)

This pattern translates 1:1. A template ships at `templates/commit-push-pr.prompt.md.template`. Copy to `.github/prompts/commit-push-pr.prompt.md` and customize the project's golden rules (base branch, commit trailer, allowed paths).

In any Copilot Chat surface, type:

```
/commit-push-pr
```

The prompt will collect the inputs (optional commit summary override) and walk through the 8-step workflow:

1. Branch safety (refuse main/master/develop)
2. Build gate (run build if build-relevant files dirty)
3. Diff review (verify instruction files for changed paths were applied)
4. Selective staging (NEVER `git add -A`; deny `.env*`, secrets)
5. Conventional Commit message with project trailer
6. Push without `--force` / `--no-verify`
7. Open or update PR via `gh pr create`
8. Compact final report

Hard NOs are codified inline in the prompt body so the workflow can't drift.

---

## What does NOT translate

### `PostToolUse` lint-fix has no Copilot equivalent

Copilot has no event for "after the file was just edited, run X." This belongs at the IDE layer:

- **VS Code:** ESLint extension's "Fix on Save" (`"editor.codeActionsOnSave": { "source.fixAll": true }`)
- **JetBrains:** "Reformat code on save" + ESLint plugin
- **Pre-commit:** Husky + lint-staged in `package.json`

If you don't have an editor-level auto-fix yet, fix that before adopting any other parts of this chapter. Copilot will write code at whatever style your editor formats to; if your editor doesn't format on save, every PR will have formatting noise and Copilot has no way to fix it server-side.

### Lesson from the Claude framework's v1.1.0 → v1.1.2 sequence

If you're running this Copilot framework alongside [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework), one cautionary note from the Claude side that *can't apply here* but is worth knowing:

Claude Code's Stop hooks have a counterintuitive IO contract — `stdout` is captured but NOT surfaced into the model's next turn; only `stderr` is. The Claude framework's v1.1.0 shipped Stop hook templates that wrote reminders to stdout, which meant the reminders correctly produced + the exit code was correct, but the model never saw them. v1.1.2 fixed it by switching to stderr.

This whole class of bug doesn't exist on the Copilot side because Copilot has no programmable hook events. The Pattern 1 / Pattern 2 / Pattern 3 / Pattern 5 translations above are all manual or declarative, not driven by an event runtime, so there's no "wrong IO channel" failure mode.

If you author hook scripts for Claude Code based on Copilot patterns you've translated mentally, remember that on the Claude side: **Stop hooks → stderr; PreToolUse hooks → stdout.**

### Hooks-on-Cloud-Agent

The Copilot Cloud Agent (autonomous agent) runs in GitHub Actions with no access to per-developer settings. Hooks-style runtime enforcement on the Cloud Agent would require GitHub Actions workflow steps — outside the framework's surface area. The framework's documentation discipline applies fully to the Cloud Agent because instruction files and prompt files are repository state, not local config.

---

## Verification

For Copilot, "did the rule fire?" is a different question than for Claude Code:

1. **`applyTo:` rule surfacing test** — open a file matching an instruction's `applyTo:` glob in Copilot Chat ("@workspace what instruction files apply here?"). The matching instruction should be listed. If not, the glob doesn't match — fix it.
2. **Correction-capture test** — issue a correction in Chat, then invoke `/correction-capture`. The prompt should produce a draft instruction-file patch, not a "noted" acknowledgment.
3. **Build-gate test** — break the build, run a specialist task, confirm the specialist's `return:` block fails the build gate and the orchestrator rejects the return.
4. **`/commit-push-pr` test** — run on a trivial doc change with a `--dry-run` style abort instruction. Confirm the workflow is interruptible at each step.

If any of these fail in real-world use, treat as P0 — the Copilot enforcement layer is thinner than Claude Code's, and the few mechanical pieces it has must work.

---

## Cross-links

- `docs/01-PRINCIPLES.md` § Principle 5 — what's runtime-enforced vs documented (the line is different on Copilot than on Claude Code)
- `docs/05-INSTRUCTIONS-AND-PROMPTS.md` — `applyTo:` mechanics and prompt-file basics
- `docs/04-HANDOFF-SCHEMA.md` — how the orchestrator's incoming-validation enforces Pattern 3 (build-gate via Definition of Done)
- `docs/08-COMMON-PITFALLS.md` § Pitfall 17 — Copilot has no Stop event; correction-capture and build-gate are manual on Copilot
- `templates/correction-capture.prompt.md.template`
- `templates/commit-push-pr.prompt.md.template`
