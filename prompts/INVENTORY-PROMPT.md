# INVENTORY PROMPT — GitHub Copilot version

Paste this into Copilot Chat at the root of the project you want to bootstrap. Adjust `<framework path>` to match your install location.

---

I want to adopt the GitHub Copilot Orchestration Framework on this project. Before we generate any files, do a read-only inventory.

The framework lives at `<framework path>` (default: `/Users/<you>/Desktop/github-copilot-orchestration-framework/`). Read `docs/01-PRINCIPLES.md`, `docs/02-ARCHITECTURE.md`, and `docs/03-AGENTS-GUIDE.md` from there before answering. Do not modify any files in this project yet.

## ⚠ Two universal rules for this entire pass

1. **Evidence-first.** Every claim you make must cite a real file path, command output, or filename. The phrase "I infer" or "appears to be" without a specific file:line citation is not acceptable.
2. **Ask, don't guess.** If a fact cannot be confirmed by reading a specific file or running a specific command, mark it `<NEEDS USER CONFIRMATION: <one-line question for me>>` and EXPLAIN what you tried before giving up. Examples:
   - `<NEEDS USER CONFIRMATION: Is this project named "acme" or "acme-store"? package.json says "@acme/store-monorepo" — slug is ambiguous.>`
   - `<NEEDS USER CONFIRMATION: Are EU users in scope? No GDPR-related env vars or consent UI found, but README mentions "international expansion".>`

Surface ambiguity. Never invent a fact to fill a gap.

---

## Discovery commands — run these in order

Before writing anything, execute this discovery sequence and capture the results in your context. If a command/file is not present, note it as missing and move on.

### Step A — Top-level inventory

```bash
pwd
ls -la
find . -maxdepth 1 -type f | sort
find . -maxdepth 2 -type d | sort
git remote -v
git branch -a
git log --oneline -5
```

### Step B — Project name + tech stack manifest discovery

Read whichever of these exist (in this order); stop after the first one with a clear `name`:

| File | Looks for |
|---|---|
| `package.json` | `name`, `description`, `dependencies`, `scripts` (Node/JS/TS) |
| `pyproject.toml` | `[project] name`, `dependencies` (Python) |
| `setup.py` | `name=`, `install_requires=` (legacy Python) |
| `Cargo.toml` | `[package] name`, `[dependencies]` (Rust) |
| `go.mod` | `module <path>`, `require` (Go) |
| `Gemfile` + `*.gemspec` | gem name, deps (Ruby) |
| `composer.json` | `name`, `require` (PHP) |
| `pubspec.yaml` | `name:`, `dependencies:` (Flutter/Dart) |
| `*.podspec` / `Podfile` | iOS native dependencies |
| `build.gradle` / `build.gradle.kts` | Android / Kotlin |
| `*.csproj` | .NET project name |
| `package.swift` | Swift Package Manager |
| `mix.exs` | Elixir |

If MULTIPLE manifests exist (monorepo), list all and ask: `<NEEDS USER CONFIRMATION: Which manifest is the canonical project name source?>`

### Step C — Framework + service detection

Read these files if present:

| File | Tells you |
|---|---|
| `next.config.js` / `next.config.ts` | Next.js |
| `nuxt.config.ts` | Nuxt |
| `vite.config.ts` / `vite.config.js` | Vite |
| `svelte.config.js` | SvelteKit |
| `astro.config.mjs` | Astro |
| `remix.config.js` | Remix |
| `gatsby-config.js` | Gatsby |
| `angular.json` | Angular |
| `vue.config.js` | Vue CLI |
| `app.json` / `metro.config.js` | React Native / Expo |
| `pubspec.yaml` (lib: flutter) | Flutter |
| `tsconfig.json` | TypeScript target/strict mode |
| `prisma/schema.prisma` | Prisma ORM |
| `drizzle.config.ts` | Drizzle ORM |
| `*.sql` files in `migrations/` or `db/` | Raw SQL migrations |
| `Dockerfile` / `docker-compose.yml` | Container setup |
| `vercel.json` | Vercel deploy |
| `netlify.toml` | Netlify deploy |
| `wrangler.toml` | Cloudflare Workers |
| `serverless.yml` | Serverless framework |
| `fly.toml` | Fly.io |
| `firebase.json` | Firebase |
| `terraform/*.tf` / `cdk.json` | IaC |
| `.github/workflows/*.yml` | GitHub Actions CI/CD |
| `.gitlab-ci.yml` | GitLab CI |
| `.circleci/config.yml` | CircleCI |
| `playwright.config.ts` | Playwright E2E tests |
| `cypress.config.ts` | Cypress E2E tests |
| `vitest.config.ts` / `jest.config.js` | Unit test runner |
| `pytest.ini` / `tox.ini` | Python test config |

### Step D — External service detection (.env / dependencies)

Read `.env.example` (or `.env.sample` / `.env.template`). For each env var, infer the service:

| Env var pattern | External service |
|---|---|
| `STRIPE_*` | Stripe (payments) |
| `RESEND_*`, `SENDGRID_*`, `MAILGUN_*`, `POSTMARK_*` | Transactional email |
| `TWILIO_*` | Twilio (SMS / voice) |
| `OPENAI_*`, `ANTHROPIC_*`, `GEMINI_*` | LLM provider |
| `SUPABASE_*` | Supabase |
| `FIREBASE_*` | Firebase |
| `CLERK_*`, `NEXTAUTH_*`, `AUTH0_*`, `WORKOS_*` | Auth provider |
| `POSTHOG_*`, `MIXPANEL_*`, `SEGMENT_*`, `AMPLITUDE_*` | Analytics |
| `SENTRY_*` | Error monitoring |
| `DATADOG_*`, `NEW_RELIC_*` | APM / observability |
| `CLOUDFLARE_*`, `R2_*`, `S3_*`, `GCS_*` | Storage / CDN |
| `ALGOLIA_*`, `MEILISEARCH_*`, `ELASTICSEARCH_*` | Search |
| `REDIS_URL`, `UPSTASH_*` | Cache / queue |
| `DAILY_*`, `LIVEKIT_*`, `AGORA_*`, `PUSHER_*`, `ABLY_*` | Real-time / video |

Cross-check by reading the `dependencies` section of the manifest from Step B. The two should agree.

### Step E — Existing Copilot configuration

```bash
ls .github/ 2>/dev/null
cat .github/copilot-instructions.md 2>/dev/null | head -50
ls .github/agents/ 2>/dev/null
ls .github/instructions/ 2>/dev/null
ls .github/prompts/ 2>/dev/null
ls .github/chatmodes/ 2>/dev/null   # retired format — any file here will be RENAMED to .github/agents/<name>.agent.md
ls .github/skills/ .github/hooks/ 2>/dev/null
cat AGENTS.md 2>/dev/null | head -50
```

If the project ALREADY has any of these, document what's there. The bootstrap will need to merge with existing config rather than overwriting it.

### Step F — Source-tree shape

```bash
find . -maxdepth 4 -type d \
  -not -path './node_modules/*' \
  -not -path './.git/*' \
  -not -path './.next/*' \
  -not -path './dist/*' \
  -not -path './build/*' \
  -not -path './target/*' \
  -not -path './venv/*' \
  -not -path './__pycache__/*' \
  | sort | head -100
```

For the apparent source root (`src/`, `app/`, `lib/`, `internal/`, `cmd/`, etc.):
```bash
find <source-root> -maxdepth 3 -type d | sort
```

### Step G — Documentation discovery

```bash
find docs -type f -name "*.md" 2>/dev/null | sort
ls docs/ai-context/ 2>/dev/null
find . -maxdepth 1 -type f -name "*.md" | sort   # README, CONTRIBUTING, AGENTS, etc.
```

### Step H — Hygiene scan

```bash
find . -maxdepth 1 -type f -name "_*" -o -name "check-*" -o -name "test-*" 2>/dev/null
find . -maxdepth 1 -type f \( -name "*.png" -o -name "*.jpg" -o -name "*.html" \) 2>/dev/null
find . -maxdepth 2 -type f -name ".env*" 2>/dev/null
git ls-files | grep -E "(\\.env|secret|credential)" | head
```

### Step I — Branch and remote check

```bash
git branch -a | head -20
git remote -v
cat .github/workflows/*.yml 2>/dev/null | grep -E "branches:|push:" | head
```

This identifies protected branches by what CI runs against.

---

## Now produce the structured inventory in 7 sections

Use the discovery results above. Cite SPECIFIC files for every claim.

### 1. Project identity
- Project name (one phrase) — cite which manifest you read it from
- Project slug (lowercase-hyphenated)
- One-sentence description
- Primary tech stack — list each component with the file that confirmed it
- Deployment target — cite the deploy config file
- CI/CD — cite the workflow file(s)
- Protected branches — list with confirmation source
- Test runners — unit + E2E (cite config files)
- **Existing Copilot config** — what already exists in `.github/copilot-instructions.md`, `.github/agents/`, `.github/instructions/`, `.github/prompts/`, `.github/skills/`, `.github/hooks/`, `.github/chatmodes/` (retired — rename to `.agent.md`), `AGENTS.md`. Bootstrap will need to merge with these, not overwrite.

### 2. Roles and audiences
List every distinct user type. Infer from route group structure, permission-model files, README. For each role:
- Name
- One-sentence description
- Evidence (file path or doc)

### 3. Domain boundaries (proposed Copilot agents)
Propose specialists. For each:
- `name:` (lowercase-hyphenated, domain-shaped not technology-shaped)
- One-sentence description
- Whether it should be REVIEW-ONLY or implementation
- The path globs it will primarily edit (must match real directories — verify with `ls`)
- Suggested `tools:` allowlist (read-only set for REVIEW-ONLY; full set for implementation)
- Whether it needs MCP server access (e.g. browser MCP for QA)
- Evidence: which signals from Steps C/D/F led you to propose this

Aim for 5-10 specialists. Always include:
- An orchestrator named `<project-slug>-orchestrator`
- A `qa-functional` if user-facing surface (test config exists in Step C)
- A `release-devops` if deploy automation exists
- A `security-privacy` (REVIEW-ONLY) if user data exists
- A `legal-compliance` (REVIEW-ONLY) ONLY if regulatory exposure is confirmed (NOT speculatively)

### 4. Path-globbed instruction files (proposed)
For each specialist's domain, propose 1-2 instruction files in `.github/instructions/`. For each:
- File name (`<NAME>.instructions.md`)
- `applyTo:` glob list (verify the globs match real files)
- Whether to use `excludeAgent: "code-review"` (if rules aren't review-relevant)
- Top 3-5 hard rules — each WITH source-confirmed evidence (cite file:line)

If you cannot find evidence for a proposed rule:
- Either: don't propose it
- Or: mark `<NEEDS USER CONFIRMATION: I think rule X applies because Y, but I can't find a code example. Is this rule real?>`

### 5. Existing documentation tier
Categorize every file in `docs/`:
- **Canonical** — full-detail, authoritative, currently accurate
- **Orientation candidate** — would benefit from being condensed into `docs/ai-context/<area>.md`
- **Archive candidate** — dated reports, sprint snapshots, post-mortems
- **Active workflow** — currently-in-progress plans

### 6. Clutter / hygiene findings
Surface (don't fix yet):
- Tracked tool caches (build artifacts, IDE caches)
- Orphan scripts at root
- Empty stub directories
- Loose screenshots / images
- Env file backups (`.env.local.bak*`, `.env.staging_tmp`) — **flag CRITICAL if tracked in git**
- Files with stale paths
- Obvious security concerns (env files in git history, hardcoded secrets)

### 7. Invocation guidance for the team
Propose a one-paragraph snippet for the team's onboarding doc explaining when to use:
- Plain Copilot Chat (no agent)
- `@<project-slug>-orchestrator` for cross-domain / production-sensitive work
- `@<specialist>` for narrow single-domain work
- Cloud Agent for well-defined, autonomous tasks
- `/<workflow>` slash commands for repeatable workflows

Customize for THIS project's actual specialist list and protected branches.

---

## Output format rules

- Use the section headings above EXACTLY. Numbered 1-7.
- Cite a specific file path or command output for every claim.
- For anything you can't verify, mark `<NEEDS USER CONFIRMATION: <specific question>>`.
- Do NOT create any files in this pass. Read-only investigation.
- After producing the inventory, end with a section called `## Open questions` listing every `<NEEDS USER CONFIRMATION>` from above as a numbered list.
- Then ask which sections I want to adjust before bootstrapping.

---

(End of prompt.)
