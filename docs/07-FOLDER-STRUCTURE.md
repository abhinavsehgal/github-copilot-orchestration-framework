# 07 — Folder Structure

How to organize Copilot config and documentation so the AI (and humans) always know where to look.

## The Copilot config tier

```
.github/
├── copilot-instructions.md             ← repo-wide router (auto-loaded everywhere)
├── agents/                             ← orchestrator + specialists
│   ├── <project>-orchestrator.md
│   └── <specialist>.md
├── instructions/                       ← path-globbed invariants (frontmatter has applyTo:)
│   └── <domain>.instructions.md
├── prompts/                            ← repeatable workflows (slash commands)
│   └── <workflow>.prompt.md
├── skills/                             ← repeatable workflows (cross-surface) — v1.2
│   └── <name>/SKILL.md
└── hooks/                              ← optional mechanical enforcement — v1.2
    └── framework.json                  (chat modes are retired: rename .chatmode.md → .agent.md)
```

## The documentation tier

```
docs/
├── ai-context/                         ← TIER 1: orientation maps (read by Copilot agents)
│   ├── INDEX.md
│   ├── HANDOFF_SCHEMA.md
│   ├── ORCHESTRATION_SPOONFEEDER.md
│   └── <area>-experience.md
│
├── ARCHITECTURE.md                     ← TIER 2: canonical references
├── API.md
├── TECH_STACK.md
├── <UPPERCASE>.md
│
└── _archive/                           ← TIER 3: frozen historical
    ├── README.md
    └── <YYYY-MM>/
```

## Tier 1 — Orientation maps (`docs/ai-context/`)

**Purpose:** Per-area guides Copilot reads at the start of any task in that area.

**Format:** 50-150 lines per file.

**Audience:** Copilot agents (primary), humans (secondary).

**Naming:** `<area>-experience.md` or `<area>-<concern>.md`. Examples:
- `customer-experience.md`
- `admin-experience.md`
- `auth-and-sessions.md`
- `payments.md`
- `realtime-sync.md`
- `data-pipeline.md`

**Body sections:**
1. **2-sentence orientation.** What this area is, what it isn't.
2. **Key file paths.**
3. **3-5 gotchas worth knowing.**
4. **Cross-links** to canonical refs and to related instruction files.

**Special files:**
- `INDEX.md` — task-type → docs + agent map
- `HANDOFF_SCHEMA.md` — bidirectional handoff schema
- `ORCHESTRATION_SPOONFEEDER.md` — human-facing usage guide
- `MIGRATION_LEDGER.md` (optional) — tracks doc/structure migration progress
- `LEGACY_BACKUP.md` (optional) — frozen pre-refactor snapshot

> **Why `docs/ai-context/` and not `docs/copilot-context/`?** The orientation maps are tool-agnostic. If you later add Claude Code, Gemini Code Assist, or other AI tools, they'll all read from the same place. Don't tie this folder to a single AI vendor.

## Tier 2 — Canonical references (`docs/<UPPERCASE>.md`)

**Purpose:** Authoritative source of truth. Full detail. Updated when underlying truth changes.

**Format:** As long as needed. No artificial line cap.

**Audience:** Humans (primary), agents (when deeper detail needed).

**Naming:** `UPPERCASE.md`. Convention signals "this is canonical."

**Common files:** `ARCHITECTURE.md`, `API.md`, `TECH_STACK.md`, `CHANGELOG.md`, `PRODUCT_REQUIREMENTS.md`, `BUSINESS_DOCUMENT.md`.

**Cross-referencing:**
- Orientation maps in `ai-context/` link DOWN to canonical docs.
- Canonical docs do NOT link UP to orientation maps.
- Agents link to either, depending on depth needed.

## Tier 3 — Frozen archive (`docs/_archive/`)

**Purpose:** Historical material no longer authoritative but worth preserving for audits.

**Format:** Whatever the original was — markdown, HTML, PNG, CSV.

**Audience:** Audits only. Active docs never link here.

**Naming:** Date-prefixed subdirectories: `<YYYY-MM>/<file>`.

**Required:** `_archive/README.md` documenting:
- What this directory is
- Three rules: don't link from active docs, don't delete, don't move back to root
- "Known link rot" section if you migrated from a structure with cross-links

## What goes where — decision tree

```
You have a doc to write or move. Where?

Is it a per-area gotcha guide of 50-150 lines?
  → docs/ai-context/<area>.md

Is it the system architecture, API reference, or another full-detail doc?
  → docs/<UPPERCASE>.md

Is it a path-globbed invariant ("don't do X when editing Y")?
  → .github/instructions/<domain>.instructions.md (with applyTo: in frontmatter)

Is it a multi-step workflow that recurs (manually invoked)?
  → .github/prompts/<workflow>.prompt.md

Is it an agent persona definition?
  → .github/agents/<name>.agent.md

Is it a sprint report, audit, post-mortem, or dated snapshot?
  → docs/_archive/<YYYY-MM>/<file>.md

Is it a one-time decision rationale?
  → PR description or commit message — NOT docs/

Is it just routing ("for X tasks, use Y agent")?
  → .github/copilot-instructions.md (kept SHORT) or docs/ai-context/INDEX.md
```

## Folder-level conventions

### Root level

The root contains ONLY:
- Tech-stack config files (`package.json`, `tsconfig.json`, `Cargo.toml`, `go.mod`, etc.)
- Build/deploy config (`Dockerfile`, `docker-compose.yml`, etc.)
- CI config (`.github/workflows/`, `.gitlab-ci.yml`)
- Test runner config (`playwright.config.ts`, `pytest.ini`, etc.)
- The framework files (`AGENTS.md` if used, `.github/`, `docs/`)
- Application directories (`src/`, `app/`, `lib/`, `pkg/`, etc.)
- Static assets (`public/`, `assets/`)

The root does NOT contain:
- Loose orphan scripts (move to `scripts/_archive/` if not actively used)
- Screenshots or images (move to `docs/_archive/screenshots/` or `assets/`)
- Backup env files (gitignore + delete from history if secrets were committed)
- Generated artifacts (gitignore)
- Stub directories with one file (consolidate elsewhere)

### Tracked-cache cleanup

Common directories to gitignore (often tracked by accident):
- IDE caches (`.vscode/`, `.idea/` — at minimum gitignore the auto-generated parts)
- Test runner per-run state (`test-results/`, `coverage/`)
- Database CLI temp dirs
- Build artifacts (`.next/`, `dist/`, `build/`, `__pycache__/`)
- Local logs (`logs/`)

### Env file safety

Add broad gitignore patterns:

```gitignore
.env
.env.*
!.env.example
.env*.bak*
.env*staging_tmp
.env*tmp
```

If env files with secrets are already in git history, that's a separate workstream — rotate secrets first, then scrub history with `git-filter-repo` or BFG.

## Variants for non-standard projects

### Monorepo

```
monorepo-root/
├── .github/
│   ├── copilot-instructions.md         ← root router; mentions per-package files exist
│   ├── agents/                         ← orchestrator + cross-package specialists
│   ├── instructions/                   ← cross-package instructions
│   └── prompts/
├── docs/                               ← cross-package docs
│   └── ai-context/
└── packages/
    ├── package-a/
    │   └── AGENTS.md                   ← package-specific notes
    └── package-b/
        └── AGENTS.md
```

(Note: nested `.github/` directories don't get loaded the same way — use `applyTo:` globs in root `.github/instructions/` to scope per-package.)

### Multi-product in one repo

```
your-product-repo/
├── .github/agents/
│   ├── orchestrator.md
│   ├── product-a-frontend.md
│   ├── product-a-backend.md
│   ├── product-b-frontend.md
│   └── shared-database.md
├── .github/instructions/
│   ├── product-a.instructions.md       (applyTo: "src/products/a/**")
│   └── product-b.instructions.md       (applyTo: "src/products/b/**")
├── docs/ai-context/
│   ├── INDEX.md
│   ├── product-a-experience.md
│   └── product-b-experience.md
```

### Mobile + web shared

```
your-app/
├── apps/
│   ├── web/                            ← Next.js / Vite / etc.
│   └── mobile/                         ← React Native / Flutter / native
├── packages/                           ← shared libs
└── .github/agents/
    ├── web-ui.md                       ← only edits apps/web/
    ├── mobile-ui.md                    ← only edits apps/mobile/
    ├── shared-libs.md                  ← only edits packages/
    └── backend-api.md                  ← stays separate if hosted elsewhere
```

## Organization-level configuration

If you have GitHub Copilot Business or Enterprise:

```
your-org/
├── .github-private/                    ← special org-level repo
│   └── agents/
│       └── company-wide-agent.md       ← available across all org repos
└── (regular repos as above)
```

Organization custom instructions are configured in the GitHub UI at organization settings — not in files.
