# LocalGPT Agent — Approval Authority Amendment

**Status:** Approved normative design amendment  
**Date:** 2026-08-17  
**Applies to:** `docs/superpowers/specs/2026-08-17-localgpt-agent-design.md`  

## 1. Purpose and precedence

This amendment tightens the raw-shell execution model after review of the existing `opaque_or_unknown` behavior.

The current design correctly separates semantic risk (`R0`–`R3`) from execution transparency, but it previously allowed every R2 action to become requester-self-approvable when the current Agent session was in `Autonomous / Unrestricted` mode. That rule is too broad for execution shapes whose semantics the AgentCore cannot inspect with sufficient confidence.

This document is **normative**. Where it conflicts with the current text in Sections 3, 6, 9, 11, 12, 14, 16, 23, 26, 29–31, 36 or 39 of the main design, this amendment takes precedence until the same wording is folded into the main design document.

The architectural model remains unchanged:

```text
McpGateway
  → ToolDispatcher
  → PolicyEngine
  → ApprovalManager
  → ResourceGovernor
  → Execution
  → Audit/EventBus
```

The change is limited to execution-transparency classification and approval authority.

## 2. Decision summary

Phase 1 now separates two questions that must not be conflated:

1. **Risk level:** what kind of impact is known or reasonably inferred? (`R0`–`R3`)
2. **Approval authority:** who is allowed to final-authorize the request?

`R3` remains human-only because the system recognizes destructive/data-loss semantics.

Some `R2` requests are also human-only, not because they are known to be destructive, but because their execution semantics are too opaque to permit requester self-approval safely.

The key decision is:

> **Truly opaque/indirect raw-shell execution is human-only even in `Autonomous / Unrestricted` mode. Unknown-but-direct execution may remain R2 and may use Autonomous R2 self-approval.**

This preserves autonomous coding UX for unfamiliar local/internal CLIs while preventing an agent from self-authorizing execution techniques that deliberately hide or dynamically construct behavior.

## 3. Canonical invariant

Add the following canonical invariant to Section 6.2 of the main design:

**SEC-AUTHORITY-001 — Opaque execution requires human authorization**  
A request classified as `opaque_indirect` cannot be final-approved by the requesting MCP client or by Autonomous-mode R2 self-approval. Final allow requires a human-authorized approval channel. If known R3 semantics are also present, R3 rules continue to apply.

Relationship to existing invariants:

- `SEC-R3-001..003` remain unchanged.
- `SEC-MODE-001` remains unchanged; every new session starts Guarded.
- `SEC-CONTENT-001` remains unchanged; content is data, not authority.
- `SEC-EXEC-001` remains unchanged; repository scripts are arbitrary-code boundaries.
- `SEC-AUTHORITY-001` adds an approval-authority floor based on execution opacity, not destructive semantics.

## 4. Execution transparency model

Replace the previous three-value `ExecutionTransparency` model with four values:

```ts
type ExecutionTransparency =
  | "direct_known"
  | "direct_unknown"
  | "indirect_repository_controlled"
  | "opaque_indirect";
```

### 4.1 `direct_known`

The command shape is recognized and sufficiently transparent at the command-line level for a reviewed classifier rule.

Examples may include narrowly scoped forms such as:

```text
git status
git diff
git log
node --version
npm --version
```

This classification does not imply sandboxing and does not override known R2/R3 semantics.

### 4.2 `direct_unknown`

The executable/command family is not yet covered by a reviewed `direct_known` rule, but the invocation remains structurally direct and does not contain material opacity/indirection indicators.

Typical characteristics:

- executable target can be identified
- arguments can be tokenized/inspected normally
- no encoded command payload
- no dynamic evaluation primitive
- no download-and-execute pipeline
- no meaningful shell indirection that hides the executed payload

Example:

```text
internal-cli inspect --format json
```

when `internal-cli` is unfamiliar to the classifier but is directly invoked with visible arguments.

Default policy:

```text
risk = R2
approvalAuthority = human_or_autonomous
```

Guarded requires human approval. Autonomous may permit requester self-approval under normal R2 rules.

### 4.3 `indirect_repository_controlled`

Execution delegates to repository-controlled scripts/hooks/plugins through a structured ProjectAdapter or another explicitly modeled project boundary.

Examples:

```text
npm test
npm run build
pnpm run lint
project_dev
Git hook triggered by a modeled operation
```

The existing product tradeoff remains:

```text
risk = R1 by explicit ProjectAdapter contract unless other semantics escalate it
executionTrust = repository_controlled
approvalAuthority = none for the normal R1 path
```

This does not mean the code is sandboxed or side-effect free; `SEC-EXEC-001` remains in force.

### 4.4 `opaque_indirect`

The invocation materially prevents AgentCore/ShellInspector from inspecting the execution semantics with sufficient confidence, or dynamically obtains/constructs code to execute.

Indicators include at least:

- PowerShell `-EncodedCommand`
- `Invoke-Expression` / `iex` with dynamic content
- download-and-execute chains such as `irm ... | iex`
- piping downloaded content to PowerShell/cmd/interpreter execution
- `node -e` / `python -c` or equivalent with dynamically generated/opaque payload when the payload cannot be safely classified
- nested shell invocation whose meaningful payload cannot be extracted reliably
- unknown script/interpreter chains combined with substantial redirection/pipeline/evaluation indirection
- other classifier conditions where the important executed payload is hidden from normal inspection

Default policy floor:

```text
risk >= R2
approvalAuthority = human_only
```

Autonomous mode cannot reduce this approval-authority floor.

Known R3 semantics take precedence:

```text
opaque_indirect + destructive semantics
→ risk = R3
→ approvalAuthority = human_only
```

`opaque_indirect` is **not automatically R3** solely because it is opaque. R3 continues to mean recognized destructive/data-loss semantics; opacity is represented by a separate approval-authority dimension.

## 5. Approval authority model

Add a first-class policy field:

```ts
type ApprovalAuthority =
  | "none"
  | "human_or_autonomous"
  | "human_only";
```

Update the conceptual policy decision:

```ts
type PolicyDecision = {
  decision: "allow" | "approval_required" | "deny";
  riskLevel: "R0" | "R1" | "R2" | "R3";
  approvalAuthority: ApprovalAuthority;
  reasons: string[];
  inspectorFlags: string[];
  executionTransparency?: ExecutionTransparency;
  executionTrust?: "direct" | "repository_controlled" | "opaque";
  unknownRisk?: boolean;
  contentProvenance?: ContentProvenance[];
};
```

`riskLevel` and `approvalAuthority` are intentionally independent.

Examples:

| Operation | Risk | Transparency | Approval authority |
|---|---|---|---|
| `git status` | R0 | `direct_known` | `none` |
| normal workspace patch | R1 | n/a | `none` |
| external process kill | R2 | `direct_known` where applicable | `human_or_autonomous` |
| unfamiliar direct internal CLI | R2 | `direct_unknown` | `human_or_autonomous` |
| encoded/dynamic/download-and-execute shell | R2 minimum | `opaque_indirect` | `human_only` |
| `git reset --hard` | R3 | usually `direct_known` | `human_only` |
| opaque command with recognized deletion/data loss | R3 | `opaque_indirect` | `human_only` |

## 6. Security-mode semantics

### Guarded

- R0/R1 normal paths remain auto-allowed.
- R2 `human_or_autonomous` requires human approval.
- R2 `human_only` requires human approval.
- R3 requires human approval.

### Autonomous / Unrestricted

- R0/R1 normal paths remain auto-allowed.
- R2 with `approvalAuthority="human_or_autonomous"` may be final-approved by an authenticated MCP client, including requester self-approval where existing rules allow it.
- R2 with `approvalAuthority="human_only"` cannot be final-approved by MCP, including the requester.
- R3 remains human-only under `SEC-R3-001..003`.
- Autonomous does not relabel `direct_unknown` as R1 and does not relabel `opaque_indirect` as direct/known.

Therefore the old blanket statement:

```text
Autonomous: R2 may be self-approved by requester
```

must be read as:

```text
Autonomous: R2 may be requester-self-approved only when
approvalAuthority == human_or_autonomous
```

## 7. ApprovalManager requirements

ApprovalManager must authorize by both `riskLevel` and `approvalAuthority`.

For an MCP `allow` decision:

```text
approvalAuthority = none
→ no approval object should normally exist

approvalAuthority = human_or_autonomous
→ Guarded: reject MCP final allow
→ Autonomous: allow valid R2 MCP/self-approval

approvalAuthority = human_only
→ reject MCP final allow in every local security mode
→ request remains PENDING
→ return HUMAN_APPROVAL_REQUIRED
```

An unauthorized MCP allow attempt must not mutate the request into executable state.

The existing frozen-request hash, atomic state transition and exactly-once execution guarantees remain unchanged.

A deny decision still never executes the frozen request.

## 8. Persistence and audit

### 8.1 `tool_calls`

Add/retain bounded policy metadata:

```text
execution_transparency TEXT NULL
execution_trust TEXT NULL
approval_authority TEXT NOT NULL
```

Allowed `approval_authority` values:

```text
none
human_or_autonomous
human_only
```

### 8.2 `approvals`

Persist the authority determined when the frozen request is created:

```text
approval_authority TEXT NOT NULL
```

The frozen approval must not be downgraded later merely because the session switches from Guarded to Autonomous.

For `human_only` approvals:

- successful execution requires a human-authorized source
- `approver_kind=MCP_CLIENT` cannot produce a successful final allow
- successful requester `self_approved=1` is invalid

For R2 `human_or_autonomous` self-approval:

- current session must be Autonomous
- `self_approved=1`
- `autonomous_mode=1`

### 8.3 Audit

Policy/audit detail should include:

```text
riskLevel
approvalAuthority
executionTransparency
executionTrust
inspectorFlags
```

Add/retain an event or reason that distinguishes:

```text
DIRECT_UNKNOWN_REQUIRES_R2
OPAQUE_INDIRECT_REQUIRES_HUMAN
```

Do not log opaque payloads in a way that bypasses the existing secret-redaction rules.

## 9. Tool and error contracts

The `shell` tool result/policy detail should expose bounded, audit-safe classification metadata sufficient for Activity/Approvals UI:

```text
riskLevel
executionTransparency
approvalAuthority
reasons
```

`HUMAN_APPROVAL_REQUIRED` is the canonical error for an MCP attempt to final-allow a `human_only` approval.

`OPAQUE_EXECUTION_REQUIRES_APPROVAL` may remain as a classification/request error where useful, but it must not imply that Autonomous MCP self-approval is valid for `opaque_indirect`.

## 10. Control Center UX

### 10.1 Approvals

Display approval authority explicitly.

For `direct_unknown` R2 in Guarded:

```text
UNRECOGNIZED DIRECT COMMAND
R2 — HUMAN APPROVAL REQUIRED IN GUARDED MODE
Autonomous eligibility: Yes
```

For `opaque_indirect`:

```text
OPAQUE / INDIRECT EXECUTION
HUMAN APPROVAL REQUIRED
Autonomous eligibility: No
```

The UI must not describe `opaque_indirect` as merely “unknown command”. The distinction matters because the former is intentionally non-self-approvable while the latter may be a normal unfamiliar CLI.

### 10.2 Dashboard / Activity

Activity filters should support the four transparency classes.

When `opaque_indirect` is pending, Dashboard/Activity should make the human-only nature visible and must not suggest enabling Autonomous as remediation.

### 10.3 Doctor

Doctor/security summary should include:

```text
Direct unknown raw shell:
  R2; human in Guarded; Autonomous-eligible

Opaque indirect raw shell:
  R2 minimum; human-only in every mode

R3 destructive:
  human-only in every mode
```

## 11. Dogfood/classifier tuning changes

Section 39 metrics must distinguish `direct_unknown` from `opaque_indirect`.

Recommended local metrics:

```text
direct_unknown_rate
  = direct_unknown raw-shell calls / all raw-shell calls

direct_unknown_human_allow_rate
  = human-approved direct_unknown R2 / decided direct_unknown R2

direct_unknown_self_approve_rate
  = Autonomous MCP/self-approved direct_unknown R2 / decided direct_unknown R2

opaque_indirect_rate
  = opaque_indirect raw-shell calls / all raw-shell calls

opaque_indirect_human_allow_rate
  = human-approved opaque_indirect / decided opaque_indirect

repeat_approved_direct_unknown_shape_count
  = repeated approved direct_unknown calls grouped by audit-safe shape
```

Dogfood tuning priorities:

1. repeated `direct_unknown` shapes are candidates for human-reviewed promotion to narrowly scoped `direct_known` rules
2. `opaque_indirect` is primarily a security signal, not a familiarity-training bucket
3. repeated approval never auto-mutates classifier rules
4. no metric automatically enables Autonomous or changes approval authority
5. a rule may move a command from `direct_unknown` to `direct_known` only after review/regression tests demonstrate that the shape is actually transparent
6. a rule must not promote `opaque_indirect` merely because the user has approved similar commands repeatedly

The previous generic `opaque_rate` metric may be retained only as a compatibility aggregate; operational decisions should use the split metrics above.

## 12. Testing requirements

Update unit/security/integration coverage to include at least:

### ShellInspector / ExecutionTrustPolicy

```text
git status
→ direct_known
→ normal risk semantics

unknown-internal-cli --version
→ direct_unknown
→ R2
→ human_or_autonomous

unknown-internal-cli inspect foo
→ direct_unknown when no opacity indicators are present
→ R2
→ human_or_autonomous

powershell -EncodedCommand <fixture>
→ opaque_indirect
→ >=R2
→ human_only

Invoke-Expression <dynamic fixture>
→ opaque_indirect
→ >=R2
→ human_only

download | interpreter fixture
→ opaque_indirect
→ >=R2
→ human_only

git reset --hard
→ R3
→ human_only
```

### Security modes

```text
Guarded + direct_unknown R2 MCP allow
→ rejected
→ human UI allow succeeds

Autonomous + direct_unknown R2 requester self-allow
→ allowed
→ selfApproved=true
→ autonomousMode=true

Guarded + opaque_indirect MCP allow
→ rejected HUMAN_APPROVAL_REQUIRED

Autonomous + opaque_indirect requester self-allow
→ rejected HUMAN_APPROVAL_REQUIRED
→ remains PENDING

Autonomous + opaque_indirect human UI allow
→ succeeds exactly once when frozen hash matches
```

### Regression invariants

- known R3 semantics override transparency class
- `opaque_indirect` never becomes self-approvable by toggling Autonomous
- `direct_unknown` does not silently become R1 in Autonomous
- familiar executable name alone does not guarantee `direct_known`
- repository ProjectAdapter commands remain `indirect_repository_controlled`, not `direct_unknown`
- dogfood approval history cannot auto-promote `opaque_indirect`
- unauthorized MCP allow leaves approval PENDING
- request hash/exactly-once/replay guarantees remain unchanged

## 13. Phase 1 roadmap / release-gate updates

### Phase 1.4 — Shell + ProcessManager + execution-transparency policy

Deliver:

- `direct_known`
- `direct_unknown`
- `indirect_repository_controlled`
- `opaque_indirect`
- first-class `ApprovalAuthority`

Acceptance:

- known direct inspection command follows normal risk semantics
- unfamiliar structurally direct CLI becomes R2 `human_or_autonomous`
- opaque/indirect execution becomes at least R2 `human_only`
- destructive semantics remain R3 human-only

### Phase 1.5 — Policy / approvals

Acceptance must include:

```text
Guarded direct_unknown → human approval
Autonomous direct_unknown → valid R2 requester self-approval
Guarded opaque_indirect → human-only
Autonomous opaque_indirect → still human-only
R3 → human-only in every mode
```

### Phase 1.8 — Dogfood/hardening

Use split transparency metrics. Tune repeated `direct_unknown` friction first. Treat `opaque_indirect` as a security signal rather than a command-familiarity problem.

### Core Definition of Done additions

Add the following release-gate checks:

```text
unknown but direct raw-shell fixture in Guarded
→ R2 human approval

same direct_unknown fixture in Autonomous
→ requester R2 self-approval succeeds

opaque_indirect fixture in Guarded
→ human-only approval

same opaque_indirect fixture in Autonomous
→ MCP/requester self-allow rejected
→ human Control Center approval required

verify Activity/Audit records risk + transparency + approvalAuthority
```

## 14. Implementation constraints for Codex

Treat these as fixed Phase 1 requirements:

- `ExecutionTransparency` has four classes: `direct_known`, `direct_unknown`, `indirect_repository_controlled`, `opaque_indirect`
- `ApprovalAuthority` is first-class and independent from `riskLevel`
- `direct_unknown` raw shell is R2 by default and `human_or_autonomous`
- `opaque_indirect` raw shell is at least R2 and always `human_only`
- Autonomous may authorize eligible R2 but cannot bypass a `human_only` authority decision
- R3 remains human-only and retains its destructive/data-loss meaning
- do not turn all opacity into R3 solely to achieve human gating
- do not permit repeated approvals/dogfood familiarity to auto-promote `opaque_indirect`
- persistence/audit/UI/tests must carry `approvalAuthority`

## 15. Examples of intended behavior

| Request | Guarded | Autonomous |
|---|---|---|
| `git status` (`direct_known`, R0) | Auto | Auto |
| normal workspace patch (R1) | Auto | Auto |
| unfamiliar direct internal CLI (`direct_unknown`, R2) | Human approval | MCP/requester may approve |
| external process kill (R2 `human_or_autonomous`) | Human approval | MCP/requester may approve |
| encoded/dynamic/download-and-execute (`opaque_indirect`, R2 minimum) | Human approval | **Human approval** |
| destructive `git reset --hard` (R3) | Human approval | **Human approval** |

## 16. Non-changes

This amendment does **not** change:

- Windows-only Phase 1
- Electron/React/TypeScript/Node stack
- elevated AgentCore lifecycle
- authenticated loopback HTTP + stdio bridge
- session-scoped Autonomous mode/reset-to-Guarded behavior
- canonical path policy
- repository-controlled structured project R1 tradeoff
- R3 human-only guarantees
- frozen approval request/hash
- exactly-once execution
- SQLite/EventBus audit architecture
- resource governance
- Phase 1/Release 1.1/Phase 2/Phase 3 ordering

## 17. Final rule

The normative Phase 1 rule is:

> **Autonomous mode is permission to self-authorize eligible sensitive R2 actions; it is not permission for the requester to self-authorize execution whose meaningful behavior AgentCore cannot inspect. `direct_unknown` may remain Autonomous-eligible R2, while `opaque_indirect` is human-only.**
