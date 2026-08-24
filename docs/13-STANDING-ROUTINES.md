# 13 — Standing Routines (scheduled autonomy)

## The question this chapter answers

Everything before this chapter runs when a person asks. The orchestrator decomposes a task
(Chapter 2), hooks enforce discipline while a session runs (Chapter 10), and the truth files brief
the next session (Chapter 11) — but nothing happens between sessions. This chapter adds the third
leg: **routines** — narrow agent jobs that run on a schedule, produce small reviewable pull
requests, and get *tuned* instead of babysat.

The pattern is not speculative. The Claude Code team at Anthropic ran it publicly on their own repos
in Aug 2026: a fleet of daily routines (crash fuzzing, duplicate-abstraction unification, dead-code
removal, flaky-test root-causing) opened **388 PRs in a few weeks, 180 of which were merged** after
automated review plus human review, at roughly 1-in-50 noise. Nothing in the pattern is
vendor-specific — Copilot's cloud agent and headless CLI are exactly the surfaces it needs.
Everything below restates the practice stack- and domain-agnostically, folded into this framework's
existing conventions.

## Where routines sit

| Leg | Runs when | Chapter |
|---|---|---|
| Interactive orchestration | a person asks | 02, 03, 06 |
| Mechanical enforcement | a session does something | 10 |
| **Standing routines** | **the clock says so** | **this one** |

A routine is *not* a cron script with an LLM inside. It is a specialist (Chapter 3) with a written
charter, an output contract, a review gate, and a tuning log — the same discipline the framework
applies to interactive agents, applied to unattended ones. If a job needs no judgment (bump a
version, rotate a log), use plain Actions automation; a routine earns its cost only where the work
needs reading, reasoning, or writing code.

## The seven conventions

**1. One routine, one narrow charter.** "Maintain the codebase" is not a charter. "Find statically
unreachable code and remove it" is. A routine that needs the word "and" in its charter is usually
two routines. The charter lives in a checked-in file (`templates/routine.md.template`) so it can be
diffed, reviewed, and tuned like any other framework file.

**2. Output is small, self-contained PRs — never direct pushes.** A routine's writes reach the
integration branch only through a PR (Pitfall 13 applies doubly to unattended writers — and the
cloud agent's own PR-only workflow already enforces this shape). Small and self-contained is what
makes ten routine PRs a five-minute review instead of an afternoon.

**3. Every fix-PR carries a repro and a truth table.** "No evidence, no claim" (Principle 3) does
not relax when nobody is watching — it tightens. A crash fix shows the reproduction; a behavior
change shows the before/after truth table, verified end-to-end on the real system, not a mock. A
routine PR without evidence is closed without debate, and that closure feeds convention 6.

**4. All routine activity reports to one place.** One issue thread, one channel, or one dated log
file per fleet — not scattered notifications. The point is that a person can read one surface each
morning and know what ran, what it produced, and what died. Routines that fail silently are worse
than no routines (see Pitfall 30).

**5. A routine never merges its own PR.** The gate is: automated review first (Copilot code review
on every routine PR; a stronger review pass for anything touching money, auth, data deletion, or
another repo's contract), then a human merge. The human's cost stays low *because* of conventions
2 and 3.

**6. Wrong output tunes the routine, not just the output.** When a routine PR is wrong, fix the PR
if it's worth fixing — but always patch the routine's charter/prompt so tomorrow's run doesn't
repeat it, and date the change in the charter's tuning log. This is the correction-capture loop
(Chapter 10) applied to unattended work. Expect a real noise rate — the public reference fleet ran
at ~1 in 50 — and budget it; a routine whose noise stays above its budget after two tuning passes
gets retired, not tolerated.

**7. Attempt caps and a verified retire path.** Every routine carries a per-run budget (time or
premium-request spend), an attempt cap after which it parks instead of grinding, and — critically —
a *checked* completion write. One production team's generator pipeline ran for 17 days without a
single job completing: the "done" write violated a database constraint, the error return was never
read, and a reaper kept resurrecting the same satisfied jobs forever. The lesson generalizes to any
unattended loop: **a best-effort write whose failure changes control flow must have its error
read**, and "the routine ran" must never be reported as "the routine worked" (Pitfall 14, unattended
edition).

## Model and effort tiering

Match the model to the routine, not the fleet. The public reference fleet ran mostly on a mid-size
frontier model, reserving the largest for a few hard routines — and noted that a smaller model is
workable if you "spend a bit more time auditing PRs and iterating on routines' prompts, then adding
in checks and guardrails." That trade is general: **cheaper model ⇒ tighter output contract, more
automated checks, more sampling in review.** Where Copilot exposes model choice (cloud agent, CLI),
set it per routine, and re-tier quarterly as models change (Pitfall 20).

## A starter catalog

Charters that have earned their keep, restated generically — pick two, not ten:

- **Dead-code remover** — remove statically unreachable code. For *suspected* (dynamically
  reachable?) dead code: add logging first, check the logs next run, remove only then. The
  two-step is the difference between a janitor and a hazard.
- **Duplicate-abstraction unifier** — find similar-yet-slightly-divergent helpers/components and
  unify them behind one, with a truth table showing call-site behavior is unchanged.
- **Flaky-test root-causer** — pick the week's flakiest test, find the *root cause* (order
  dependence, time, shared state), fix it or delete it with a written justification. Never
  auto-retry as the fix.
- **Doc-drift checker** — diff Tier-1/Tier-2 docs (Chapter 7) against reality: dead links, stale
  counts, commands that no longer exist. Companion to the librarian (Pitfall 16), running daily
  instead of quarterly.
- **Crash fuzzer** (repos that ship an app) — drive the real app, find crashes, root-cause, open a
  fix PR with the repro attached.
- **Contract-drift checker** (workspaces) — see Chapter 12 § Scheduled workspace routines.
- **Workspace janitor** — prune stale worktrees, merged branches, dead clone directories
  (Pitfall 26's cleanup crew).
- **Hill-climber** — a routine wrapper around the hill-climb skill
  (`templates/skills/hill-climb/SKILL.md.template`): one metric, one target, one bounded iteration
  per run.

## Running them on Copilot

Three mechanisms, in order of preference:

1. **Scheduled workflow → cloud agent (Mode 5).** A GitHub Actions `schedule:` workflow creates
   (or re-opens) the routine's tracking issue and assigns it to Copilot; the cloud agent works in
   its own environment and opens the PR. This is the most native fit — the cloud agent's
   PR-per-task shape *is* convention 2. Keep the charter file as the issue body's single source.
2. **Scheduled workflow → headless CLI (Mode 6).** `copilot -p` with `--agent=<routine-agent>`
   inside a cron workflow — the portable floor when you need a custom toolchain in the runner
   image. Note: whether `.github/hooks/*.json` fire in headless CI sessions is **not re-verified**
   — test it in your install and stamp a verified-on date before relying on hook enforcement
   inside routine runs; until then, treat the PR gate (convention 5) as the enforcement layer.
3. **Org-level automation** — where an organization schedules agents centrally, the same charter
   and gates apply; the mechanism changes, the conventions do not.

Give every mechanism a kill switch a human can flip without a deploy (disable the workflow, close
the tracking issue with a label, an environment variable) — you will want it during an incident.

## What NOT to do

- Don't let a routine merge, push to the integration branch, or touch production state directly —
  PRs only, gates always.
- Don't launch a fleet before the first routine has run for two weeks and had its noise tuned
  down — routines compound, and so does their noise.
- Don't give a routine agent a broad toolset "to be safe" — charter-narrow `tools:`, same as any
  specialist (Principle 5); an unattended agent with wide write access is your largest blast
  radius.
- Don't run a routine without a budget and an attempt cap — a runaway loop discovered on the
  monthly premium-request bill is the canonical failure (see also REFINEMENT check 12).
- Don't report "ran" as "worked" — completion means the *verified* completion write landed.
- Don't skip the schedule and run "when someone remembers" — that is not a routine, that is
  Chapter 6.

## Cross-links

- `templates/routine.md.template` — the charter file every routine checks in.
- `templates/skills/hill-climb/SKILL.md.template` — the metric-loop skill routines can wrap.
- `docs/06-INVOCATION-MODES.md` § Scheduled runs — where routines sit among the modes (Mode 5/6 on
  a clock).
- `docs/10-MECHANICAL-ENFORCEMENT.md` — the correction-capture pattern is the tuning loop's
  interactive twin.
- `docs/11-PROJECT-TRUTH-AND-LEARNINGS.md` — routine tuning logs follow the LEARNINGS entry shape.
- `docs/12-MULTI-REPO-WORKSPACES.md` § Scheduled workspace routines — the contract guardian on a
  clock.
- `docs/08-COMMON-PITFALLS.md` — Pitfalls 29 (context weight) and 30 (unattended jobs need a
  verified retire path).
