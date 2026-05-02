# 05 — Instructions and Prompts

Two patterns for capturing knowledge that doesn't belong in agents: **instruction files** (path-globbed invariants) and **prompt files** (repeatable workflows invoked via slash command).

## Instruction files

Path-globbed invariants. Each file in `.github/instructions/` has YAML frontmatter declaring which file paths it applies to. When Copilot is editing a matching file, the instruction file is auto-loaded into the prompt.

This is the **direct equivalent of "rules"** in other frameworks — and arguably cleaner because the path globbing is in YAML frontmatter rather than in body text.

### When to write an instruction file

Write one when:
- A bug shipped to production once and must not recur
- A pattern is non-obvious from reading the code (e.g. "this column looks like a string but it's a JSON-encoded string in legacy rows")
- A constraint crosses multiple files but isn't enforced by types/tests
- An anti-pattern is tempting but wrong

Don't write one for:
- Things obvious from the code
- Style preferences (lint config does that)
- One-off decisions (PR descriptions do that)
- Anything covered by an existing instruction file (extend the existing one)

### Instruction file structure

```markdown
---
applyTo: "src/api/**/*.ts"
excludeAgent: "code-review"
---

# <Domain Name> Instructions

## Hard rules

### <Rule title — short imperative>

**Why:** <one-sentence reason — usually a past incident or invariant>

**How to apply:** <2-4 sentences — what to do, what helper to use, what to check>

### <Next rule>

...

## What NOT to do

- Bullet list of anti-patterns

## Cross-links

- `<related orientation map>`
- `<related canonical doc>`
- `<related instruction file>`
```

### Frontmatter fields

| Field | Required | Notes |
|---|---|---|
| `applyTo` | yes | Glob pattern. Multiple patterns separated by commas: `"**/*.ts,**/*.tsx"`. |
| `excludeAgent` | no | Prevent specific Copilot features from using this. Values: `"code-review"`, `"cloud-agent"`. |

### Glob patterns

Standard shell globs. Examples:

```yaml
applyTo: "src/api/**/*.ts"
applyTo: "src/db/migrations/**/*.sql"
applyTo: "src/{lib,utils}/auth-*.ts"
applyTo: "components/payment/**/*.{tsx,jsx}"
applyTo: "ios/Runner/**/*.swift"
applyTo: "lib/screens/**/*.dart"
applyTo: "internal/handlers/**/*.go"
applyTo: "app/models/*.rb"
```

Globs are matched against the relative path from project root.

### Limit on code review

⚠ **Code review reads only the first 4,000 characters of each instruction file.** If you need code-review-relevant rules, put them at the top of the file.

To exclude an instruction from code review entirely:
```yaml
applyTo: "src/api/**/*.ts"
excludeAgent: "code-review"
```

### Examples by tech

#### Backend / API

```markdown
---
applyTo: "src/api/**/*.ts,src/server/**/*.ts"
---

### Always validate request body before reading database

**Why:** Three production incidents traced to unvalidated body data flowing to SQL parameters.

**How to apply:** Use `validateRequest(schema)` middleware in `src/lib/validation.ts`. Never read `req.body.<field>` directly in a handler.
```

#### Database / migrations

```markdown
---
applyTo: "**/migrations/**/*.sql,prisma/schema.prisma"
---

### Migration order is sacred — never reorder, never edit applied migrations

**Why:** Migrations apply in filename order. Editing an applied migration silently diverges staging from production.

**How to apply:** New migrations get the next sequential timestamp prefix. Bug fixes to applied migrations require a NEW migration that supersedes the old behavior.
```

#### Frontend / UI

```markdown
---
applyTo: "src/components/**/*.{tsx,jsx},src/app/**/*.{tsx,jsx}"
---

### Never trust component props for security checks — re-validate on the server

**Why:** A user can edit any prop via React DevTools.

**How to apply:** Server-side handlers re-validate authorization. Props are for rendering, not authorization.
```

#### Mobile

```markdown
---
applyTo: "src/screens/**/*.tsx,src/components/**/*.tsx"
---

### iOS auto-zooms on input focus when font-size < 16px

**Why:** iOS Safari WebView feature; jarring UX.

**How to apply:** Set `font-size: 16px` minimum on all `<input>`, `<select>`, `<textarea>`. For visual size adjustment, scale via padding/transform, not font-size.
```

### Anti-patterns in instruction files

❌ **Long rationale paragraphs.** Rules should be ≤ 4 sentences each. Move long stories to the canonical doc.

❌ **Aspirational rules.** "We should always..." — if it's not enforced, it's not a rule.

❌ **Rules that contradict other instruction files.** Have the context-librarian agent reconcile.

❌ **Files with no `applyTo`.** Without the glob, Copilot has no signal to load it.

❌ **Putting too many rules in one file.** Group by domain, not by "all rules in one file." If a file passes ~150 lines or 3,000 chars, split it.

## Prompt files

Repeatable workflows invoked via slash command. Each file in `.github/prompts/` has frontmatter + body.

### When to write a prompt file

Write one when:
- The same kind of task arrives 3+ times
- The task has multiple steps with verification between them
- The cost of skipping a step is high (e.g. forgot to run security review before launching a new auth flow)
- A junior team member would benefit from a guided checklist

Don't write one for:
- One-off tasks
- Tasks where the steps vary too much per instance
- Things that are really just a single agent invocation

### Prompt file structure

```markdown
---
description: <one-sentence description of when to use this — appears in /command picker>
---

<the prompt body — instructions, may include ${input:variable:hint} placeholders>

## Steps

### 1. <Step name>

<2-4 sentences>

### 2. <Next step>

...

## Definition of Done

1. <Artifact>
2. <Verification>
```

### Frontmatter fields (current public preview)

| Field | Required | Notes |
|---|---|---|
| `description` | yes | One sentence shown in the `/command` picker. |
| `agent` | optional | Specific agent to associate with this prompt. |

(More fields may be supported by your IDE — check the [Copilot prompt files documentation](https://docs.github.com/en/copilot/tutorials/customization-library/prompt-files).)

### Input placeholders

Inside the body, use `${input:NAME:HINT}` for prompted inputs:

```markdown
Explain the following code in a clear, beginner-friendly way:

Code to explain: ${input:code:Paste your code here}
Target audience: ${input:audience:Who is this explanation for?}
```

When invoked via `/<filename>`, Copilot prompts the user for each input.

### Invocation

In Copilot Chat, type `/` to see the picker. Pick from the list, or type the prompt name: `/<filename-without-extension>`. Example: `.github/prompts/investigate-bug.prompt.md` → `/investigate-bug`.

Available in: VS Code, Visual Studio, JetBrains. Currently in **public preview** — features may evolve.

### Common prompt files (most projects benefit from these)

#### `.github/prompts/investigate-bug.prompt.md`

Steps: restate bug → identify affected roles → read minimum docs → trace UI → state → API → DB → tests → match touched paths to instruction files → reproduce → classify severity → propose smallest fix → verify.

#### `.github/prompts/build-feature.prompt.md`

Steps: restate feature → identify roles → read minimum docs → run pre-feature checklist → write acceptance criteria per role → identify instruction-covered paths → plan smallest implementation → implement incrementally.

#### `.github/prompts/qa-flow.prompt.md`

Steps: identify flows in scope → read testing strategy + relevant orientation map → cover golden path + edge cases + cross-role → verify functional + data correctness → severity-rank findings.

#### `.github/prompts/compliance-review.prompt.md`

Steps: identify data involved → identify if regulated subjects involved → read compliance orientation map → apply standing checklist → flag risks per regulation → produce attorney checklist.

#### `.github/prompts/context-refactor.prompt.md`

For maintaining the framework itself: read current INDEX → categorize suspect content → move to correct home → consolidate duplicates → flag conflicts → keep `.github/copilot-instructions.md` small.

## Instructions vs prompt files vs agents — when to use which

| If the knowledge is... | Put it in... |
|---|---|
| A path-scoped invariant ("don't do X when editing Y") | Instruction file |
| A multi-step workflow that recurs (invoked manually) | Prompt file |
| A persona with its own scope and Definition of Done | Custom agent |
| A one-time decision rationale | PR description |
| Long-form architecture explanation | Canonical doc |
| Quick orientation for an area | Orientation map (`docs/ai-context/<area>.md`) |

When something feels like it could be 2 of those: prefer instruction file > prompt file > agent (in that order). Instruction files are most precise (path-globbed, auto-loaded). Prompt files are next (named workflow, manual). Agents are heaviest (full persona).

## Naming and lifecycle

### Instruction files
- Lowercase with hyphens, `.instructions.md` suffix
- File name = domain name
- New gotcha from a new domain: create new file
- New gotcha from existing domain: append to existing file
- Stale instruction (the underlying constraint was fixed): delete it, don't leave it

### Prompt files
- Lowercase with hyphens, `.prompt.md` suffix
- Filename becomes the slash command (`investigate-bug.prompt.md` → `/investigate-bug`)
- Used < 3x per quarter: consider deleting; the workflow probably wasn't recurrent enough
