# LocalGPT Agent — Personal-Use Ergonomics Amendment

**Status:** Approved normative design amendment  
**Date:** 2026-08-17  
**Applies to:**
- `docs/superpowers/specs/2026-08-17-localgpt-agent-design.md`
- `docs/superpowers/specs/2026-08-17-localgpt-agent-approval-authority-amendment.md`

## 1. Purpose and precedence

LocalGPT is a personal-use system running on a trusted developer workstation. The security model should therefore avoid needless repetitive interaction while preserving the boundaries that protect against compromised repositories, indirect instructions, opaque execution, accidental destructive actions and compromised MCP clients.

This amendment refines three UX areas:

1. local MCP transport defaults
2. Autonomous-mode activation ergonomics
3. workspace-specific shell/classifier tuning

This amendment is **normative**. Where the main design or prior amendment is ambiguous about the behaviors below, this document takes precedence. It does **not** weaken `SEC-R3-001..003`, `SEC-AUTHORITY-001`, `SEC-CONTENT-001`, `SEC-EXEC-001`, `SEC-AUTH-001` or the canonical path-policy model.

The design principle is:

> **Reduce friction in transport setup, mode activation and classifier tuning before removing security boundaries. Personal use is a reason to optimize interaction cost, not a reason to make opaque or destructive execution silently trusted.**

## 2. Decision summary

Phase 1 adopts the following personal-use ergonomics decisions:

- persistent local MCP integrations should prefer the `mcp-stdio-bridge` + current-user Windows Named Pipe path
- localhost HTTP remains supported, authenticated and useful for debugging/programmatic clients, but its bearer credential still rotates every Agent session
- Phase 1 does not add a static global localhost bearer token
- every new Agent session still starts in `Guarded`
- the Control Center adds a one-click way to re-enter Autonomous for the current session
- Autonomous activation may use a bounded lease (`1 hour`, `4 hours`, or `until Agent exits`)
- no Autonomous lease survives Agent restart/crash/relaunch
- a user may persist a preferred lease duration, but not the active Autonomous state itself
- Phase 1 does not add blanket `Trusted Workspace = unrestricted shell`
- instead, a workspace may own narrow, reviewed **Workspace Command Rules** for direct command shapes
- workspace command rules may reduce repeated `direct_unknown` friction, but cannot make `opaque_indirect` or R3 self-approvable
- classifier/dogfood data remains local-only and is used for human-reviewed rule tuning, never automatic trust promotion

## 3. Local MCP transport ergonomics

### 3.1 Preferred transport by use case

The product should guide local clients toward the transport that best matches their lifecycle:

```text
Persistent local MCP client integration
→ preferred: stdio bridge → current-user named pipe → AgentCore

Temporary/debug/programmatic local client
→ supported: authenticated localhost HTTP
```

Examples of persistent integrations may include editor/desktop MCP clients that normally launch a stdio server process and expect configuration to survive application restarts.

The stdio bridge remains thin and owns no filesystem, shell, PolicyEngine, ApprovalManager or SQLite logic.

### 3.2 HTTP token policy remains session-scoped

`SEC-AUTH-001` remains unchanged for HTTP:

- loopback-only binding
- bearer authentication on every HTTP MCP request
- CSPRNG token with at least 256 bits of entropy
- token exists in AgentCore memory only
- token rotates every Agent session
- previous-session token becomes invalid
- token is never persisted in SQLite, logs or audit payloads

The UX should not position manual token copy as the primary way to connect persistent local MCP clients.

### 3.3 Phase 1 static-token non-goal

Phase 1 must not add a single static global HTTP bearer token merely to avoid copy/update friction.

Reason:

```text
per-session secret
→ bounded lifetime

static global bearer secret
→ long-lived credential copied into client configuration
→ larger persistence/leak/reuse window
```

If Phase 1 dogfood identifies an important local client that cannot use stdio and cannot refresh a session credential, a future design may evaluate **paired-client credentials** rather than a single global token.

Any paired-client design must have at least:

- per-client credential identity
- explicit user pairing
- revocation
- credential rotation/replacement
- secure local storage using a Windows-appropriate credential mechanism
- audit-safe client identity without logging plaintext credentials

Paired-client credentials are intentionally deferred; they are not required to ship Phase 1 Core.

### 3.4 Control Center transport UX

Settings / Doctor should present guidance such as:

```text
Local client recommendation

Persistent editor/desktop integration:
  Preferred → MCP stdio bridge

HTTP MCP:
  Best for temporary/programmatic/debug clients
  Credential rotates when Agent restarts
```

The existing **Copy session credential** action remains available for explicit HTTP use.

Doctor should be able to detect and explain:

- stdio bridge available/unavailable
- named pipe healthy/unhealthy
- HTTP healthy/unhealthy
- current HTTP auth enabled state
- that HTTP credential rotation is expected behavior, not an error

## 4. Autonomous-mode activation ergonomics

### 4.1 `SEC-MODE-001` remains fixed

Every new Agent session starts:

```text
security.mode = guarded
```

No setting may automatically start a new Agent session in Autonomous mode.

A prior session ending in Autonomous may be remembered as historical/audit context, but it does not initialize the next session's active mode.

### 4.2 One-click session resume

To reduce restart friction, if the immediately previous session ended while Autonomous was active, the Dashboard may prominently offer:

```text
Previous session used Autonomous / Unrestricted

[ Resume Autonomous for this session ]
```

This button is an explicit human action in the current session. It does not automatically activate Autonomous.

The UI must not hide the mode change behind a generic toggle with unclear consequences. The action should state that R2 `human_or_autonomous` operations may become MCP/requester self-approvable while human-only and R3 gates remain unchanged.

### 4.3 Autonomous lease

Autonomous activation should support a bounded lease:

```text
1 hour
4 hours
Until Agent exits
```

The selected lease controls when the current session automatically returns to Guarded.

Rules:

- lease activation requires explicit human Control Center action
- lease expiry automatically switches the current session back to Guarded
- app restart/crash/relaunch always ends the lease
- system clock changes must not silently extend an already established lease; implementation should use a monotonic runtime deadline where practical
- mode changes and lease expiry are audited
- pending approvals keep their frozen `approvalAuthority`; toggling mode cannot downgrade `human_only`
- R3 and `opaque_indirect` remain human-only during the entire lease

### 4.4 Persisted preference vs active state

The system may persist a convenience preference such as:

```text
security.preferredAutonomousLease = 4h
```

or:

```text
security.preferredAutonomousLease = until_exit
```

This preference may preselect the UI option when the user explicitly chooses Autonomous.

It must **not** be interpreted as permission to activate Autonomous automatically at startup.

The following remains forbidden:

```text
security.startInAutonomous = true
```

for Phase 1.

### 4.5 Dashboard visibility

When Autonomous is active, Dashboard must show:

- `AUTONOMOUS / UNRESTRICTED`
- lease expiration or `until Agent exits`
- action to return to Guarded immediately
- explicit note that `opaque_indirect` and R3 still require human approval

When the lease has less than a reasonable warning interval remaining, UI may show a non-blocking countdown/warning.

## 5. Workspace Command Rules instead of blanket Trusted Workspaces

### 5.1 No blanket shell trust

Phase 1 must not implement a switch equivalent to:

```text
Trust this workspace
→ allow all shell commands in this workspace
```

Workspace registration/access does not make repository text, scripts, dependencies or generated files trusted instruction authority (`SEC-CONTENT-001`) and does not make repository-controlled execution safe (`SEC-EXEC-001`).

### 5.2 Workspace Command Rule concept

A workspace may have an explicit set of human-reviewed rules for narrow direct command shapes that repeatedly appear as `direct_unknown`.

Conceptually:

```ts
type WorkspaceCommandRule = {
  id: string;
  workspaceId: string;
  shell: "powershell" | "cmd" | "any";
  commandShape: string;
  classification: "direct_known";
  reviewedRiskCeiling: "R0" | "R1" | "R2";
  createdAt: string;
  updatedAt: string;
  enabled: boolean;
  note?: string;
};
```

The exact persistence schema may vary, but the behavioral rules below are fixed.

### 5.3 What a Workspace Command Rule may do

A reviewed rule may:

- recognize a previously `direct_unknown` command shape as `direct_known`
- reduce repeated classifier uncertainty for that narrow shape
- preserve a reviewed semantic risk result for that shape when no stronger runtime indicator is present
- make dogfood tuning workspace-specific when a CLI is meaningful only inside one project

Examples of potentially reviewable shapes:

```text
acme-cli status <bounded-args>
acme-cli inspect <bounded-args>
internal-tool --version
internal-tool metadata --json
```

A rule is not a general executable-name allowlist.

### 5.4 What a Workspace Command Rule must never do

A workspace rule cannot:

- convert `opaque_indirect` to `direct_known`
- bypass `SEC-AUTHORITY-001`
- lower recognized R3 semantics
- bypass `SEC-R3-001..003`
- authorize download-and-execute
- authorize encoded/dynamic evaluation merely by pattern familiarity
- grant trust to arbitrary repository scripts
- convert `indirect_repository_controlled` into direct/sandboxed execution
- make an external-path destructive action safe merely because cwd is inside the workspace
- auto-create itself from approval history

Runtime detection of stronger semantics always wins over a workspace rule.

Examples:

```text
workspace rule matches command prefix
+
actual invocation contains -EncodedCommand
→ opaque_indirect
→ human_only

workspace rule matches known CLI
+
runtime arguments contain recognized destructive target
→ R3
→ human_only
```

### 5.5 Rule creation UX

The system may surface repeated `direct_unknown` shapes from Guarded Friction diagnostics as **rule candidates**.

The user may explicitly choose an action such as:

```text
Review command shape
→ inspect normalized/redacted shape + examples
→ create workspace command rule
```

Creating/enabling/changing/removing a Workspace Command Rule is a human Control Center action and is audited.

The UI must clearly state that the rule changes classifier handling for a narrow shape; it does not mark the whole workspace trusted.

### 5.6 Rule scope and matching

Rules should be intentionally narrow.

The matcher should prefer parsed structure over arbitrary regex supplied by the user. A rule may include reviewed fields such as:

- executable identity/name
- subcommand/verb
- allowed flag names
- argument positions/categories
- shell type
- workspace-relative cwd requirement

Avoid a default UI that encourages rules such as:

```text
powershell *
node *
*.ps1
anything in C:\Projects\MyApp
```

because these effectively recreate blanket trust.

## 6. Interaction with approval authority amendment

The prior approval-authority amendment remains authoritative.

The combined behavior is:

| Operation | Guarded | Autonomous lease |
|---|---|---|
| R0/R1 normal structured work | Auto | Auto |
| reviewed workspace `direct_known` R0/R1 shape | Auto | Auto |
| `direct_unknown` R2 | Human | MCP/requester eligible |
| ordinary R2 `human_or_autonomous` | Human | MCP/requester eligible |
| `opaque_indirect` R2 minimum | **Human** | **Human** |
| R3 destructive | **Human** | **Human** |

A Workspace Command Rule may only affect the `direct_unknown → reviewed direct_known` tuning path. It cannot alter the `opaque_indirect` or R3 rows.

## 7. Persistence

### 7.1 Settings

Persistent convenience settings may include:

```text
mcp.preferredLocalTransport = stdio
security.preferredAutonomousLease = 1h | 4h | until_exit
```

The setting `mcp.preferredLocalTransport` is UI/config guidance; HTTP remains independently configurable.

The active security mode remains session state and is not restored automatically.

### 7.2 Workspace command rules

Add a logical persistence boundary such as:

```text
workspace_command_rules
- id
- workspace_id
- shell_type
- command_shape
- rule_json
- reviewed_risk_ceiling
- enabled
- created_at
- updated_at
- note
```

Rules must be validated when loaded. Malformed or unsupported rules fail closed and are reported by Doctor; they must not silently broaden matching.

### 7.3 Session mode lease metadata

Session/runtime metadata may record:

```text
security_mode
mode_lease_kind
mode_lease_started_at
mode_lease_expires_at NULL
```

This supports audit/UI. It does not authorize restoration in a future session.

## 8. Audit events

Add or retain bounded audit events/reasons for:

```text
SECURITY_MODE_CHANGED
AUTONOMOUS_LEASE_STARTED
AUTONOMOUS_LEASE_EXPIRED
AUTONOMOUS_LEASE_CANCELLED
WORKSPACE_COMMAND_RULE_CREATED
WORKSPACE_COMMAND_RULE_UPDATED
WORKSPACE_COMMAND_RULE_DISABLED
WORKSPACE_COMMAND_RULE_DELETED
WORKSPACE_COMMAND_RULE_MATCHED
```

Audit payloads use normalized/redacted command shapes and must not leak credentials or sensitive raw arguments.

## 9. Guarded-friction dogfood updates

Dogfood should now answer two separate questions:

1. Is the user spending time maintaining local transport credentials unnecessarily?
2. Is Guarded producing repeated `direct_unknown` friction that can be safely reduced through reviewed rules?

Recommended local-only metrics include:

```text
stdio_client_sessions
http_client_sessions
http_session_credential_copy_count

direct_unknown_rate
direct_unknown_human_allow_rate
repeat_approved_direct_unknown_shape_count
workspace_rule_candidate_count
workspace_rule_match_count

autonomous_activation_count
autonomous_resume_count
autonomous_lease_expiry_count
guarded_return_count
```

Metrics remain local operational diagnostics, not external telemetry.

Important interpretations:

- repeated HTTP credential copy suggests transport UX friction; prefer stdio or evaluate future paired-client credentials
- repeated approved `direct_unknown` shapes suggest review candidates, not automatic trust
- frequent Autonomous activation suggests Guarded friction worth investigating; it is not itself a reason to persist Autonomous across sessions
- `opaque_indirect` frequency remains primarily a security signal and is not eligible for automatic trust tuning

## 10. Control Center changes

### Dashboard

Add/clarify:

- preferred local MCP transport guidance
- one-click `Resume Autonomous for this session` when contextually relevant
- current Autonomous lease countdown/state
- immediate `Return to Guarded` action
- reminder that human-only execution remains gated

### Settings

Add/clarify:

- preferred local transport: `stdio` default recommendation
- preferred Autonomous lease duration
- HTTP remains per-session credential mode
- no `persist Autonomous across restart` switch
- no `static global HTTP token` switch in Phase 1

### Projects / Workspaces

Add:

- Workspace Command Rules panel
- candidate rules sourced from local dogfood data
- explicit statement: `This does not trust all commands in this workspace`

### Activity

Allow filtering by:

- workspace command rule matched / unmatched
- direct_unknown
- direct_known
- opaque_indirect
- security mode / lease state where useful

### Doctor

Add checks/warnings for:

- persistent local client using HTTP where stdio is available (`INFO`, not failure)
- repeated manual HTTP credential copies if locally observable (`INFO`)
- malformed/disabled workspace command rules
- unusually broad command-rule patterns rejected by validation
- Autonomous lease active / expiry state

Doctor must not recommend enabling Autonomous to bypass `opaque_indirect` or R3 approval.

## 11. Testing requirements

### Transport UX

- stdio bridge remains usable across Agent restarts without editing an HTTP bearer token
- HTTP token still rotates every Agent session
- previous HTTP token fails after restart
- no static global token setting exists in Phase 1
- token never enters normal renderer state/log/audit

### Autonomous lease

- every new session starts Guarded
- one-click resume requires explicit Control Center action
- 1h lease returns to Guarded at expiry
- 4h lease returns to Guarded at expiry
- until-exit lease ends when Agent exits
- crash/restart never restores active Autonomous state
- persisted preferred lease only preselects UX; it does not activate mode
- R3 remains human-only throughout lease
- `opaque_indirect` remains human-only throughout lease
- switching to Autonomous does not mutate frozen approval authority

### Workspace Command Rules

- reviewed narrow `direct_unknown` fixture can become `direct_known`
- rule is workspace-scoped
- same command in another workspace does not inherit rule
- encoded/dynamic/opaque indicators override matching rule
- R3 indicators override matching rule
- repository-controlled ProjectAdapter commands remain `indirect_repository_controlled`
- malformed/broad unsupported rule fails closed
- approval history never auto-creates a rule
- creating/updating/disabling/deleting a rule is audited
- command-shape audit remains secret-redacted

## 12. Phase 1 roadmap updates

### Phase 1.6 — MCP transports

Add acceptance:

```text
persistent local MCP integration uses stdio bridge without session-token maintenance
HTTP remains authenticated and session-scoped
```

### Phase 1.7 — Operational Dashboard

Add:

- transport recommendation/guidance
- one-click Autonomous resume
- lease selector/state/countdown
- Workspace Command Rules UI

### Phase 1.8 — Dogfood/hardening

Review:

- HTTP credential-copy friction
- stdio adoption for persistent clients
- direct_unknown repeated shapes
- Autonomous activation/resume frequency
- workspace rule candidates and actual match behavior

Tune interaction/classifier rules before considering weaker persistent security defaults.

## 13. Core Definition of Done additions

Phase 1 release gate should demonstrate:

```text
1. Configure a persistent local client through stdio bridge.
2. Restart LocalGPT Agent.
3. Client reconnects through stdio without HTTP token editing.
4. Verify HTTP token changed and old token is rejected.
5. Start new session and verify Guarded.
6. Explicitly activate Autonomous with a bounded lease.
7. Verify eligible R2 behavior follows Autonomous rules.
8. Verify opaque_indirect remains human-only.
9. Verify R3 remains human-only.
10. Let/cancel lease and verify return to Guarded.
11. Create a narrow workspace command rule from a reviewed direct_unknown shape.
12. Verify that shape is recognized only in that workspace.
13. Add an opaque/destructive indicator and verify the rule cannot bypass stronger classification.
14. Verify audit timeline contains mode/lease/rule changes without secrets.
```

## 14. Implementation constraints for Codex

Treat these as fixed Phase 1 requirements:

- stdio bridge is the preferred persistent local-client transport
- HTTP bearer credential still rotates every Agent session
- no static global HTTP token in Phase 1
- every new Agent session starts Guarded
- Autonomous may be resumed quickly only through explicit current-session human action
- Autonomous may use bounded leases of 1h, 4h or until Agent exits
- active Autonomous state never persists across restart/crash/relaunch
- preferred lease duration may persist as convenience only
- no blanket Trusted Workspace shell bypass
- Workspace Command Rules are narrow, human-reviewed and workspace-scoped
- runtime opacity/destructive detection overrides workspace rules
- `opaque_indirect` remains human-only
- R3 remains human-only
- repeated approval/dogfood evidence never creates trust automatically

## 15. Final rule

The normative personal-use ergonomics rule is:

> **Make the safe path cheap: use stdio for persistent local clients, make Autonomous easy to re-enter explicitly for a bounded session lease, and tune repeated direct-command friction with narrow workspace rules. Do not solve personal-use friction by introducing a long-lived global bearer token, auto-restoring Autonomous after restart, or trusting every shell command merely because its cwd is inside a registered workspace.**
