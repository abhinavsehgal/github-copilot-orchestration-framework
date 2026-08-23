# 10 — Mechanical Enforcement (hooks, skills, and what still needs discipline)

> **Rewritten for v1.2.0.** The v1.1 version of this chapter opened with *"GitHub Copilot does not
> expose programmable lifecycle hooks."* That was true when written and is false now: Copilot has
> hooks on the cloud agent, the Copilot CLI and VS Code. Every platform fact below was verified
> against the official hooks reference and the VS Code hooks page on **2026-08-22**. Re-verify
> quarterly (Pitfall 20 — the platform moves under your conventions).

This chapter is **optional**. Skip it until you have concrete evidence that documentation
discipline is being skipped under real-world pressure (the same triggers as the Claude edition:
the same correction repeated across sessions, bugs shipping despite a rule that forbade them,
builds skipped on commits, a production push with stale docs behind it). Then come back.

---

## What Copilot actually offers (verified 2026-08-22)

| Surface | Mechanism | Where |
|---|---|---|
| Declarative rule loading | `.github/instructions/*.instructions.md` with `applyTo:` globs — auto-loaded when a matching file is in play | All surfaces |
| Lifecycle hooks | `.github/hooks/*.json` — events `sessionStart`, `sessionEnd`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `postToolUseFailure`, `agentStop`, `subagentStart`, `subagentStop`, `errorOccurred`, `preCompact`, `permissionRequest`, `notification` | Cloud agent (reads only `.github/hooks/*.json`; `bash`/`command` fields only) · Copilot CLI (all events; also `~/.copilot/hooks/`) · VS Code (reads `.github/hooks/*.json` **and** a Claude-style `.claude/settings.json`) |
| Skills | `.github/skills/<name>/SKILL.md` — invoked as `/name` or auto-loaded by relevance | Cloud agent, code review, CLI, VS Code, JetBrains |
| Prompt files | `.github/prompts/*.prompt.md` | IDEs only — not the cloud agent, not the CLI |

### The hook contract (the part you must get right)

```json
{
  "version": 1,
  "hooks": {
    "preToolUse":  [ { "type": "command", "bash": "node .github/scripts/pre-tool.mjs",  "timeoutSec": 10 } ],
    "postToolUse": [ { "type": "command", "bash": "node .github/scripts/post-tool.mjs", "timeoutSec": 30 } ],
    "agentStop":   [ { "type": "command", "bash": "node .github/scripts/stop-gate.mjs", "timeoutSec": 600 } ]
  }
}
```

- **Input:** JSON on stdin. `agentStop` receives `session_id`, `cwd`, `transcript_path`,
  `stop_reason`, `stop_hook_active`. `preToolUse` / `postToolUse` receive `tool_name`,
  `tool_input` (and `tool_result` after). `userPromptSubmitted` receives `prompt`. Field names
  also arrive camelCase on some surfaces (`sessionId`, `toolName`, `transcriptPath`) — read both.
- **Output:** JSON on **stdout**. `preToolUse` → `{"permissionDecision": "allow|deny|ask",
  "permissionDecisionReason": "…"}`; `agentStop` → `{"decision": "block", "reason": "…"}` to
  force another turn; `postToolUse` → `{"additionalContext": "…"}`.
- **Exit codes:** `0` = parse stdout. For `preToolUse`, `2` and any other non-zero = **deny**
  (fail-closed). For other events a non-zero exit is a warning; `stderr` is shown to the user and the
  run continues.
- **Timeouts always fail OPEN**, whatever the event. A gate that needs to *block* must finish inside
  `timeoutSec` or it silently allows. Size the timeout to the slowest honest run and, inside the
  script, treat your own internal timeout as *inconclusive* (Rule 9 below).
- **Loop guard:** `agentStop` passes `stop_hook_active: true` on the forced continuation. Exit
  `{"decision":"allow"}` when you see it, or you trap the agent.

This contract differs from Claude Code's in two ways that matter if you run both editions: the
reminder goes on **stdout as JSON** (Claude: stderr as text), and `preToolUse` is **fail-closed**
(Claude: fail-silent). Do not copy a hook script between the two without re-reading this table.

---

## Translation table — the five patterns on Copilot's surface

| Pattern | Copilot mechanism | Quality |
|---|---|---|
| 1 Rule-surfacing (load matching rules before an edit) | **Native.** `applyTo:` on instruction files. No script. | Strict |
| 2 Correction-capture (a correction must become an instruction-file patch) | `userPromptSubmitted` hook detects the correction signal and writes a flag file; `agentStop` hook blocks the stop while the flag exists | Native (v1.2) — previously a manual `/correction-capture` prompt; keep the prompt for IDEs without hooks |
| 3 Build-gate (no stop with build-relevant files dirty and a failing build) | `agentStop` hook runs the build; `{"decision":"block"}` on a real failure; timeout = inconclusive | Native (v1.2) — plus Definition-of-Done discipline as before |
| 4 Lint-fix after edit | `postToolUse` hook on edit tools, or editor format-on-save / pre-commit | Native (v1.2); IDE config still preferred |
| 5 Doc-freshness gate (a production push must be followed by a changelog edit in the same session) | `postToolUse` hook records the last push and the last changelog edit in a state file; `agentStop` blocks when push > changelog | Native (v1.2) |
| `/commit-push-pr` daily workflow | `.github/skills/commit-push-pr/SKILL.md` (works on every surface; the v1.1 prompt-file version is IDE-only) | Native |

Templates for all of these ship at `templates/hooks/` (Copilot JSON contract) and
`templates/skills/`. Each script is stack-agnostic with `<UPPERCASE>` placeholders for the build
command, changelog path and protected branch.

### Why the flag-file design instead of parsing the transcript

Claude Code's Stop hooks parse the session transcript (a documented JSONL format). Copilot's
`agentStop` also hands you a `transcript_path`, but its format is not documented in the hooks
reference and may differ per surface. The templates therefore never parse it: `userPromptSubmitted`
and `postToolUse` see the events directly and record what the `agentStop` gate needs in a small
state file under `.github/hooks/.state/` (gitignored). Same behaviour on every surface, no
dependency on an undocumented format.

---

## Pattern 1 — Rule surfacing (native)

Unchanged from v1.0. `applyTo:` globs auto-load the instruction body when a matching file is being
edited or reviewed. Discipline that still matters: keep each file short (current guidance: ~1,000
lines max; two pages for the repo-wide file), make the globs actually match (verify with a `find`),
and audit `applyTo:` overlap quarterly.

## Pattern 2 — Correction-capture (two hooks + a flag)

1. `userPromptSubmitted` → `correction-detect.mjs`: strips code fences and inline code from the
   prompt, runs the anchored correction regexes (Rule 11), and on a match writes
   `.github/hooks/.state/correction-pending`.
2. `agentStop` → `stop-gate.mjs`: if the flag exists and `stop_hook_active` is false, deletes the
   flag and returns `{"decision":"block","reason":"<the §9 reminder: draft a one-line patch to the
   right .github/instructions/<file>.instructions.md and ask for approval — never 'I'll remember'>"}`.

The `/correction-capture` skill remains for surfaces where you have not installed hooks.

## Pattern 3 — Build-gate (`agentStop`)

`stop-gate.mjs` checks `git status --porcelain` for dirty build-relevant files; if any, runs
`<BUILD_COMMAND>` with its own internal cap **below** the hook's `timeoutSec`. Real non-zero exit →
`{"decision":"block","reason":"<build tail>"}`. Internal cap hit → allow (inconclusive, CI builds
every PR). Loop-guarded.

## Pattern 4 — Lint-fix (`postToolUse`)

`lint-fix.mjs` runs the project's auto-fix linter on the file in `tool_input` when `tool_name` is an
edit tool. Always exits 0 and outputs nothing — a lint problem must never block.

## Pattern 5 — Doc-freshness gate (`postToolUse` + `agentStop`)

`doc-freshness-track.mjs` (postToolUse) records two timestamps in the state file: the last shell
command that was a *real* production push (its own top-level command segment starting with
`git push` / `gh pr merge` and naming `<PROTECTED_BRANCH>`; heredoc bodies and quoted text never
count — Rule 10) and the last edit to `<CHANGELOG_PATH>`. `stop-gate.mjs` blocks when the push is
newer than the changelog edit, with the full doc-update checklist (changelog → affected orientation
maps → affected instruction files → `PROJECT.md` §3 if production state changed — Chapter 11).

---

## Design rules for any Copilot hook

1. **Decide fail-open vs fail-closed per event, on purpose.** `preToolUse` is fail-closed by the
   platform (a crashing hook denies every edit). Wrap `main()` in try/catch and emit
   `{"permissionDecision":"allow"}` on unexpected errors unless the hook is a security gate.
2. **Scope narrowly.** Filter on `tool_name` / file path first; exit fast.
3. **Cap output.** `additionalContext` and `reason` land in the model's context.
4. **Loop guard** every `agentStop` hook on `stop_hook_active`.
5. **One `agentStop` script, ordered checks inside it.** Copilot runs the hooks array in order but
   the first `block` wins; keeping correction → build → doc-freshness inside one script makes the
   order explicit and the state file reads cheap.
6. **stdout is the channel, and it must be valid JSON.** Debug to stderr.
7. **Commit `.github/hooks/*.json`.** The cloud agent reads only that location; a hook in
   `~/.copilot/hooks/` enforces nothing for teammates or CI.
8. **Restart to load.** Hook files are read at session start on the CLI and VS Code; verify in a
   fresh session, not the one that installed them.
9. **A killed check is inconclusive, not failed.** Your script's internal build cap must be lower
   than the hook `timeoutSec` (which fails open anyway), and a cap hit must `allow`, never `block`.
10. **Quoted text is data.** Strip fences, inline code and heredoc bodies before pattern-matching;
    match shell commands per top-level segment with the verb anchored at the start.
11. **Anchor correction regexes.** Bare `you already` matched "you already have access". A false
    positive costs more trust than a miss.
12. **A push in another repository is not this repo's push.** `doc-freshness-track` tracks where the
    shell is per command segment (`cd`, subshell `cd`, `git -C`, `gh --repo`) and records a push only
    when its effective directory is inside the session's `cwd`; an unknowable directory (`cd $VAR`)
    never counts. Releasing sibling repos from scratchpad clones fired the Claude edition's gate twice
    before this guard existed.
13. **Content beats ordering.** Teams that write the changelog entry FIRST (changelog → integration
    branch → promote) ship it inside the promote merge, so no changelog edit follows the push. The
    doc-freshness gate also accepts a same-day release heading in the changelog (read from
    `origin/<PROTECTED_BRANCH>` or the working tree); customise `RELEASE_HEADING_RE` to your heading
    shape, and make sure another environment's heading or a stale date cannot match.

---

## What still needs discipline (no hook can do it)

- The handoff schema being present and evidence-bound on every delegation.
- A specialist refusing a vague delegation.
- The orchestrator actually validating `return:` blocks (`tests_run` includes the build; deferred
  work has a backlog path; `contracts_changed` is backward-compatible).
- Reading an instruction file *before planning*, not only when `applyTo:` fires on an edit.

These live in the agent bodies and the Definition of Done, exactly as in v1.0.

---

## Verification

1. **Hook files load** — start a fresh CLI session in the repo; a `sessionStart` hook that prints
   one line of `additionalContext` proves the file was read.
2. **Correction-capture** — type a real correction; attempt to stop; expect the block with the
   reminder; reply with the patch; second stop succeeds (loop guard).
3. **Build-gate** — break the build; attempt to stop; expect the block with the build tail; fix;
   stop succeeds.
4. **Doc-freshness** — run a (dry) `git push origin <PROTECTED_BRANCH>` in the session; attempt to
   stop; expect the block; edit the changelog; stop succeeds. Then paste a doc that *quotes* the push
   command and confirm no block.
5. **Negative test** — a docs-only turn must produce no output from any hook.
6. **Cloud agent** — assign a trivial issue; confirm in the run log that `.github/hooks/*.json`
   was loaded.

If any of these fail, treat it as P0 — a hook that silently allows is worse than no hook.

---

## Cross-links

- `docs/01-PRINCIPLES.md` § Principle 5 — runtime-enforced vs documented (the line moved in v1.2)
- `docs/05-INSTRUCTIONS-AND-PROMPTS.md` — `applyTo:` mechanics; skills vs prompt files
- `docs/08-COMMON-PITFALLS.md` § Pitfall 19 (hooks exist — retraction), § Pitfall 20 (platform drift)
- `docs/11-PROJECT-TRUTH-AND-LEARNINGS.md` — the rule Pattern 5 enforces
- `templates/hooks/` · `templates/skills/`
- Official: *GitHub Copilot hooks reference*, *VS Code → Hooks*, *About agent skills* (verified 2026-08-22)
