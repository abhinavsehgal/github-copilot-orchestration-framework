# 04 — Handoff Schema (GitHub Copilot)

The bidirectional schema for orchestrator ↔ specialist communication. This is the framework's central artifact.

The schema is **identical** to the equivalent in the Claude Code framework — it's tool-agnostic. What changes is how it's invoked: in Copilot, the orchestrator emits the YAML block when it `@`-mentions a specialist or when the cloud agent is delegating work.

## Why a schema

Without structure, handoffs are free-form prompts. Specialists guess what's expected. Orchestrators don't commit to specific claims. Cascading hallucinations compound across hops because nothing forces evidence-to-claim binding at the boundary.

The schema closes both gaps with two YAML blocks (one per direction) and one universal evidence rule.

## Universal evidence rule

> **No evidence, no claim.** If a claim cannot be tied to a file, line, table, column, rule, doc, test, or command output, it must be marked as an assumption (`confidence: low` outbound, or moved to `unverified_claims` inbound) and cannot be passed downstream as fact.

This rule applies to both directions of every handoff. It is repeated verbatim in the orchestrator's outbound-discipline section and in every specialist's incoming-handoff-validation section.

## Outbound: orchestrator → specialist

When the orchestrator delegates to a specialist (via `@<specialist-name>` in Chat or by spawning the specialist in the cloud agent flow), the prompt body MUST include this YAML block at the top:

```yaml
handoff:
  schema_version: 1
  handoff_id: <short unique id, e.g. h-2026-05-02-a3f>
  from_agent: <orchestrator-name>
  to_agent: <specialist-name>
  goal: <one sentence — what done looks like>
  task_class: <bug | feature | review | qa | refactor | audit>
  affected_roles: [<role1>, <role2>, ...]
  claims:
    - claim: <factual statement the orchestrator commits to>
      evidence: <path/to/file.ts:42 OR instruction:.github/instructions/foo.instructions.md OR doc:docs/ARCHITECTURE.md#section OR table:column OR test:path OR command:output>
      confidence: <high | medium | low>
  verify_before_acting:
    - <claim or assumption the specialist MUST re-verify against source before editing>
  failure_condition: <one sentence — observable evidence that proves the orchestrator's hypothesis or delegation premise is wrong. If the specialist observes this, it MUST stop and return status: claim_rejected (or needs_clarification) instead of editing on a false premise.>
  in_scope:
    - <path or component the specialist may edit>
  out_of_scope:
    - <path or component the specialist must NOT touch — even if "while we're in there">
  instructions_to_read:
    - <.github/instructions/<NAME>.instructions.md path that applies to in-scope files, OR "specialist will discover from applyTo: globs">
  expected_artifacts:
    - <file changed, test added, summary section, etc.>
  hop_context:
    parent_task: <one line — why the orchestrator is delegating>
    previous_specialist: <agent name or "none">
    previous_findings_summary: <one line — what came back, or "n/a">
```

### Field requirements (outbound)

| Field | Required | Notes |
|---|---|---|
| `schema_version` | yes | Currently `1`. Bump on breaking changes. |
| `handoff_id` | yes | Short unique id. Specialist echoes it on return so they pair. |
| `from_agent` | yes | Always the orchestrator's name. |
| `to_agent` | yes | The specialist's `name` field from its `.github/agents/<NAME>.agent.md` frontmatter. |
| `goal` | yes | One sentence with measurable outcome. "Fix" or "improve" without an outcome is too vague. |
| `task_class` | yes | One of the six values. |
| `affected_roles` | yes | Non-empty list. |
| `claims` | yes (may be empty list) | Every entry MUST have `evidence`. Use `confidence: low` for assumptions. |
| `verify_before_acting` | yes if task touches code | Empty allowed only for pure-research handoffs. |
| `failure_condition` | yes | One sentence. The inverse of `verify_before_acting`. |
| `in_scope` | yes | Non-empty list. |
| `out_of_scope` | yes | Empty list `[]` is valid. Missing key is not. |
| `instructions_to_read` | yes | Use `["specialist will discover from applyTo globs"]` if you don't know which instructions apply. |
| `expected_artifacts` | yes | Drives the specialist's `files_changed` / `tests_run` later. |
| `hop_context` | yes | All three sub-fields required. Use `none` / `n/a` for first hop. |

## Inbound: specialist → orchestrator (return schema)

Every specialist response MUST end with this YAML block:

```yaml
return:
  schema_version: 1
  handoff_id: <copy from inbound — pairs the response>
  from_agent: <specialist-name>
  to_agent: <orchestrator-name>
  status: <completed | blocked | needs_clarification | claim_rejected>
  verified_claims:
    - claim: <orchestrator claim that was confirmed against source>
      evidence: <path:line or command output the specialist saw>
  rejected_claims:
    - claim: <orchestrator claim that turned out to be wrong>
      evidence: <what the specialist actually found>
  unverified_claims:
    - claim: <orchestrator claim that was not checked>
      reason: <why — out of scope, infra not available, etc.>
  evidence_used:
    - <path:line, instruction file, doc anchor, or command output>
  files_changed:
    - <path:line range and one-line summary>
  tests_run:
    - <command + result>
  risks:
    - <risk + mitigation or "none">
  recommended_next_agent:
    agent: <specialist-name or "none">
    why: <one line>
  do_not_pass_downstream_without_verification:
    - <claim or finding the orchestrator must NOT propagate as fact>
```

### Field requirements (inbound)

| Field | Required | Notes |
|---|---|---|
| `schema_version` | yes | Must match outbound. |
| `handoff_id` | yes | Must echo inbound id verbatim. |
| `from_agent` / `to_agent` | yes | Self-explanatory. |
| `status` | yes | `completed` only if all `verify_before_acting` items checked. Use `claim_rejected` when an orchestrator claim was wrong. |
| `verified_claims` | yes (may be `[]`) | Empty list is valid. Missing key is not. |
| `rejected_claims` | yes (may be `[]`) | Empty list is valid. Non-empty implies `status: claim_rejected` or `blocked`. |
| `unverified_claims` | yes (may be `[]`) | Anything you accepted without re-checking goes here. |
| `evidence_used` | yes | Real paths/commands. "I read the codebase" is not evidence. |
| `files_changed` | yes (may be `[]`) | Empty list valid for review/audit handoffs. |
| `tests_run` | yes (may be `[]`) | Empty list valid for pure-doc changes. |
| `risks` | yes | Use `["none"]` if there are genuinely none. |
| `recommended_next_agent` | yes | `agent: "none"` if work is complete. |
| `do_not_pass_downstream_without_verification` | yes (may be `[]`) | Cascading-hallucination defense. |

## Refusal

If the inbound handoff fails the field requirements, the specialist responds with ONLY this block, does no other work, and waits for re-issue:

```yaml
return:
  schema_version: 1
  handoff_id: <copy from inbound, or "missing" if absent>
  from_agent: <specialist-name>
  to_agent: <orchestrator-name>
  status: needs_clarification
  reason: handoff_schema_invalid
  missing_or_invalid_fields:
    - field: <field name>
      problem: <what's wrong — missing, vague, no evidence, etc.>
  required_action:
    - <what the orchestrator must add or fix before re-issuing>
```

## Failure-condition observation rule

If at any point during the work the specialist observes the inbound `failure_condition`, it STOPS immediately and responds with the return schema using `status: claim_rejected` (or `needs_clarification` if ambiguous). It cites the triggering evidence in `evidence_used`. It does NOT push through a falsified premise.

## Worked examples

### Example 1 — Backend bug investigation

**Outbound** (orchestrator → backend-api):

```yaml
handoff:
  schema_version: 1
  handoff_id: h-2026-05-02-bug-001
  from_agent: acme-orchestrator
  to_agent: backend-api
  goal: Find why /api/orders returns 500 for orders with > 100 line items.
  task_class: bug
  affected_roles: [customer]
  claims:
    - claim: The 500 originates in the order serialization step.
      evidence: src/api/orders/handler.ts:88
      confidence: high
    - claim: Order serialization runs synchronously and may exceed function timeout.
      evidence: instruction:.github/instructions/api.instructions.md
      confidence: medium
  verify_before_acting:
    - That src/api/orders/handler.ts:88 is still the serialization line.
    - That the timeout is actually exceeded (logs / metric, not just suspected).
  failure_condition: If logs show the 500 happens in the database query rather than serialization, STOP — the orchestrator's hypothesis about the failure location is wrong.
  in_scope:
    - src/api/orders/handler.ts
    - src/api/orders/serializer.ts
  out_of_scope:
    - The order schema (database concern; separate specialist if migration needed).
  instructions_to_read:
    - .github/instructions/api.instructions.md
    - .github/instructions/observability.instructions.md
  expected_artifacts:
    - Root cause statement (file:line)
    - Smallest fix or "no bug — load test issue" verdict
  hop_context:
    parent_task: Customer reports cart > 100 items always errors at checkout.
    previous_specialist: none
    previous_findings_summary: n/a
```

**Return** (backend-api → orchestrator):

```yaml
return:
  schema_version: 1
  handoff_id: h-2026-05-02-bug-001
  from_agent: backend-api
  to_agent: acme-orchestrator
  status: claim_rejected
  verified_claims:
    - claim: src/api/orders/handler.ts:88 is the serialization line.
      evidence: src/api/orders/handler.ts:88 (confirmed via codebase search + read)
  rejected_claims:
    - claim: The 500 originates in serialization.
      evidence: Stack trace from production logs shows the failure is in the database query at src/db/repositories/orderRepo.ts:142, not serialization. The query loads ALL line items in a single SELECT without pagination, exceeding the 30s function timeout for orders with > ~140 line items.
  unverified_claims: []
  evidence_used:
    - src/api/orders/handler.ts:80-95
    - src/db/repositories/orderRepo.ts:140-150
    - usages search for "loadOrderLineItems" in src/
    - production logs (sampled 3 failing requests)
  files_changed: []
  tests_run: []
  risks:
    - Real fix requires pagination of the query. Out of this handoff's scope.
  recommended_next_agent:
    agent: database
    why: Pagination of orderRepo.loadOrderLineItems is the correct fix.
  do_not_pass_downstream_without_verification:
    - That the timeout value is actually 30s (read from env or config; I inferred from observed behavior, did not check).
```

### Example 2 — Mobile feature build

**Outbound** (orchestrator → mobile-ui):

```yaml
handoff:
  schema_version: 1
  handoff_id: h-2026-05-02-feat-002
  from_agent: acme-orchestrator
  to_agent: mobile-ui
  goal: Add a pull-to-refresh gesture on the OrderHistoryScreen.
  task_class: feature
  affected_roles: [customer]
  claims:
    - claim: OrderHistoryScreen lives at src/screens/OrderHistoryScreen.tsx.
      evidence: src/screens/OrderHistoryScreen.tsx
      confidence: high
    - claim: The app uses React Native's RefreshControl for pull-to-refresh elsewhere.
      evidence: instruction:.github/instructions/mobile-ui.instructions.md
      confidence: high
  verify_before_acting:
    - That OrderHistoryScreen uses a ScrollView or FlatList that can host RefreshControl.
  failure_condition: If OrderHistoryScreen renders a non-scrollable view (e.g. a Map or a custom virtualized list), STOP — the standard RefreshControl pattern won't apply and a different solution is needed.
  in_scope:
    - src/screens/OrderHistoryScreen.tsx
    - src/screens/__tests__/OrderHistoryScreen.test.tsx (add test)
  out_of_scope:
    - The order data fetching logic (already exists; just call refetch).
  instructions_to_read:
    - .github/instructions/mobile-ui.instructions.md
  expected_artifacts:
    - Edit to OrderHistoryScreen.tsx adding RefreshControl
    - New test asserting onRefresh triggers refetch
  hop_context:
    parent_task: Product wants pull-to-refresh on order history per ACME-1234.
    previous_specialist: none
    previous_findings_summary: n/a
```

### Example 3 — Compliance review (REVIEW-ONLY specialist)

**Outbound** (orchestrator → legal-compliance):

```yaml
handoff:
  schema_version: 1
  handoff_id: h-2026-05-02-rev-003
  from_agent: acme-orchestrator
  to_agent: legal-compliance
  goal: Review the new email opt-in flow for GDPR/CASL/CAN-SPAM compliance before launch.
  task_class: review
  affected_roles: [customer]
  claims:
    - claim: The opt-in flow records consent timestamp + IP + opt-in source.
      evidence: src/db/migrations/2026-05-01-add-consent-log.sql
      confidence: high
  verify_before_acting:
    - That the migration has been applied to staging.
    - That the consent log columns are actually written by the new opt-in handler.
  failure_condition: If the consent log table exists but the opt-in handler does NOT write to it, STOP — the consent capture is non-functional and the compliance review is moot.
  in_scope:
    - REVIEW-ONLY. No code edits.
    - src/email/**
    - src/db/migrations/2026-05-01-*
  out_of_scope:
    - Any code changes (escalate to email-templates or backend-api specialists).
  instructions_to_read:
    - .github/instructions/security-privacy.instructions.md
  expected_artifacts:
    - Severity-ranked findings (blocker / high / medium / low)
    - Attorney-checklist of questions for counsel
  hop_context:
    parent_task: Pre-launch gate for new email opt-in feature.
    previous_specialist: none
    previous_findings_summary: n/a
```

## Optional fields added in v1.2.0 (additive — `schema_version` stays 1)

**Outbound**, for multi-repo workspaces (Chapter 12) — omit in single-repo projects:

```yaml
  repo: <repo name from workspace.json — the ONLY repo this handoff may edit>
  contract_impact:
    level: <none | additive | breaking>
    contracts: [<names from CONTRACTS.md>]
    consumers_to_update: [<repo names — required when level != none>]
```

**Outbound**, any project — the evidence-confidence class from Chapter 11 may be used in place of the
three-value `confidence:` on a claim (`verified-code` / `verified-schema` / `verified-test` /
`verified-git` / `documented-unverified` / `historical` / `unknown`). Specialists treat anything
that is not `verified-*` exactly as they treat `confidence: low`.

**Inbound**, for multi-repo workspaces:

```yaml
  contracts_changed:
    - contract: <name>
      change: <one line>
      backward_compatible: <true | false>
      consumers_grepped: [<repo>:<path>, …]
```

**Inbound**, any project — `deferred_work:` lists anything the specialist is *not* doing that
someone must (each item with the backlog file it was appended to). A return that names deferred work
without a backlog path is incomplete (Chapter 11).

## Versioning

- `schema_version: 1` is current.
- Additive changes (new optional fields) keep version 1.
- Breaking changes (renaming fields, changing field semantics) bump the version.
- Bumps require updating HANDOFF_SCHEMA.md, the orchestrator, and every specialist file in the same PR.

## Why this schema is enough (hooks are optional, not required)

The schema is enforced by:
1. The orchestrator's instructions to ALWAYS emit the outbound block.
2. Every specialist's instructions to refuse delegations missing the block.
3. The orchestrator's instructions to consume the return block before merging work.

This is documentation enforcement — agents follow the rules because their instructions tell them to. It works because:
- The Copilot harness at least enforces the `tools:` allowlist (so a misbehaving agent can't write where it shouldn't)
- The orchestrator catches return-block violations (missing fields, wrong types)
- Refusal is a one-shot rejection: if the specialist returns the refusal template, the orchestrator must re-issue with the missing fields

Copilot now has `preToolUse` / `agentStop` hooks (v1.2, `docs/10-MECHANICAL-ENFORCEMENT.md`), so a hook that validates the outbound YAML block before an `agent` tool call is possible — but the documentation enforcement still covers the majority of value at minimal complexity, and the framework ships no schema-validation hook by default.
