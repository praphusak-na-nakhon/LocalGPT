# LocalGPT Agent — Windows Coding Agent Core Design

**Status:** Approved design for implementation planning  
**Date:** 2026-08-17  
**Working product name:** LocalGPT Agent  
**Primary target:** Windows developer workstation  

## 1. Purpose

สร้าง Local Desktop Agent บน Windows ที่เปิดผ่าน Electron Control Center แบบ elevated Administrator แล้วให้ MCP clients สั่งงาน coding workflow บนเครื่องจริงได้ เช่น อ่าน/ค้น/แก้ source code, inspect Git, รัน test/lint/typecheck/build, เปิด dev process, อ่าน stdout/stderr และรัน PowerShell/cmd แบบ raw shell โดยมี policy classification, approval lifecycle, process lifecycle และ audit trail กลางชุดเดียว

Phase 1 ต้องเป็น **Coding Agent Core** เท่านั้น ไม่ใช่ full desktop automation platform และไม่ใช่ hardened sandbox

ระบบต้องรักษา product intent สำคัญว่า normal coding work เช่น create/update/edit ภายใน workspace ทำได้โดยอัตโนมัติในขอบเขตที่เหมาะสม ขณะที่การลบหรือ destructive action ต่อ user/project data ต้อง “ถามก่อน” ผ่าน human-authorized approval ที่ requester agent ไม่สามารถอนุมัติให้ตัวเองได้

เอกสารฉบับนี้เพิ่ม explicit security boundaries สำหรับ **untrusted content / indirect instructions**, **repository-controlled code execution**, **opaque shell execution**, **session-scoped Autonomous mode**, **audit retention**, และ **resource governance** เพื่อไม่ให้ implementation ต้องตัดสิน security policy กลางทาง

## 2. Source baseline and design ownership

แนวทางจาก `text.txt` และ screenshots/reference materials ใช้เป็น baseline ของ capability/product intent ดังนี้:

- Agent มีสิทธิ์สูงและทำ coding workflow บนเครื่องจริงได้
- create/update/edit สามารถทำอัตโนมัติได้ในขอบเขตที่เหมาะสม
- destructive action โดยเฉพาะการลบ user/project data ต้องถามก่อน
- `workspace_*` สำหรับหลายโปรเจกต์, structure และ snapshot
- `read_file`, `search_text`, `apply_patch`
- `git_status`, `git_diff`, `git_log`
- `project_dev`, `test`, `lint`, `typecheck`, `build`
- `process_*`, `shell`
- Live Logs / MCP activity / process visibility
- Control Center มีแนวคิด diagnostics/`Doctor`
- secure outbound-oriented MCP tunnel เป็น product intent สำหรับ remote access
- capability ขั้นสูง เช่น `codex_run`, `dom_cdp`, `accessibility`, `input_event`, `vision`, `window`, `office`, `clipboard`, `file_dialog`, `screen_record`, `audio`, `notification`, `scheduler`, `web_fetch`, `system_info`, `health`

Source baseline ใช้เพื่อยืนยัน capability และ product intent เท่านั้น เอกสารนี้ **ไม่อ้างว่าเราทราบ internal architecture, authentication protocol หรือ security implementation ของระบบต้นแบบ** จาก screenshots/reference materials หากหลักฐานไม่ได้ระบุไว้

รายละเอียด architecture, schemas, permission levels, approval state machine, SQLite model, HTTP authentication, content-provenance model, resource limits, phase decomposition และ UI/IPC ในเอกสารนี้เป็น design ของ LocalGPT Agent เอง

## 3. Approved product decisions

Phase 1 ล็อก decision ต่อไปนี้แล้ว:

1. **Scope:** Coding Agent Core
2. **Stack:** Electron + React + TypeScript + Node.js
3. **OS:** Windows only
4. **Privilege:** Elevated Administrator / unrestricted runtime capability
5. **Destructive policy:** Destructive Guard with human-authorized R3 approval
6. **Path policy:** Full create/edit inside registered workspaces; read outside workspace; external writes require approval
7. **Shell:** Raw PowerShell/cmd capability with best-effort semantic inspection; opaque raw-shell execution is not silently treated as safe in Guarded mode
8. **MCP transports:** authenticated localhost HTTP + stdio bridge
9. **HTTP authentication:** cryptographically random per-session bearer token, rotated every Agent session
10. **Lifecycle:** Elevated Desktop App owns Agent lifecycle; closing the app shuts down Agent Core
11. **Project intelligence:** Node.js/TypeScript first; generic shell/file/Git fallback for other stacks
12. **Approval channels:** Electron Control Center + MCP approval API
13. **R2 approval authority:** requester self-approval is allowed only in explicit session-scoped `Autonomous / Unrestricted` mode
14. **R3 approval authority:** requester MCP client cannot self-approve; final allow requires an independent human-authorized channel
15. **Phase 1 human approval channel:** explicit user action in Electron Control Center; MCP `approval_decide` cannot final-allow R3
16. **Security mode lifetime:** every new Agent session starts in `Guarded`; Autonomous / Unrestricted never survives app restart
17. **Workspace model:** Multi-workspace registry keyed by `workspaceId`
18. **Workspace content trust:** registered workspace content may be read/edited automatically but is not automatically trusted as instruction authority
19. **Repository execution:** project scripts/hooks are arbitrary-code execution boundaries even when invoked through known commands such as `npm test`
20. **State store:** SQLite
21. **Audit:** operational append-only event history during normal execution, with explicit bounded retention/archive maintenance authority
22. **Control Center:** Operational Dashboard, not an IDE; navigation includes Dashboard, Projects/Workspaces, Git, Activity, Processes, Approvals, Tunnel, Doctor, Settings
23. **Remote access:** not part of Phase 1 Core release gate; Secure Remote Access is the immediate milestone after Phase 1 Core
24. **Resource governance:** tool execution, process spawning, approval queues and transport admission are bounded
25. **Architecture style:** Modular monolith inside the Electron host, with a thin stdio bridge

## 4. Goals

Phase 1 must prove this end-to-end chain on a packaged Windows app:

```text
Authenticated MCP Client / trusted stdio client
   ↓
McpGateway
   ↓
ToolDispatcher
   ↓
PolicyEngine
   ├─ PathPolicy
   ├─ ShellInspector
   ├─ SecurityModePolicy
   └─ ExecutionTrustPolicy
   ↓
ApprovalManager (when required)
   ↓
ResourceGovernor
   ↓
Workspace / File / Git / Project / Shell / Process execution
   ↓
Windows
   ↓
Audit/EventBus + SQLite
   ↓
Electron Control Center
```

A successful Phase 1 allows a valid client to:

1. register a Node/TypeScript workspace
2. inspect workspace snapshot/tree
3. read and search files
4. patch one or multiple source files
5. run tests, lint, typecheck and build with explicit repository-execution semantics
6. inspect Git diff/status/log
7. launch a managed dev process
8. read incremental process output
9. request an R2 sensitive operation and follow current security-mode policy
10. request an R3 destructive operation and receive mandatory independent human approval
11. verify frozen-request hash and exactly-once execution semantics
12. verify untrusted-content provenance is preserved for tool results where applicable
13. verify opaque raw-shell execution follows Guarded/Autonomous rules
14. verify a complete activity/audit timeline
15. run Doctor diagnostics without exposing secrets
16. restart the app and observe a fresh HTTP token plus `Guarded` security mode

## 5. Non-goals for Phase 1 Core

Phase 1 Core must **not** implement these capabilities:

- active secure remote tunnel transport
- `codex_run`
- `dom_cdp`
- Windows UI Automation / accessibility control
- keyboard/mouse input fallback
- screen vision/OCR automation
- window automation
- Word/Excel COM automation
- clipboard automation as an Agent capability
- native file-dialog automation as an Agent capability
- screen recording
- microphone/audio
- notifications
- Windows Scheduled Tasks
- generic `web_fetch`
- macOS/Linux support
- Windows Service mode
- background tray persistence after the Control Center is closed
- IDE/code-editor features inside Electron
- malware-resistant sandboxing or multi-user host isolation
- semantic interpretation of arbitrary workspace text as trusted policy instructions
- sandboxing arbitrary project scripts, package lifecycle hooks or child processes

The Phase 1 UI may include a **Tunnel status page** that reports remote capability as unavailable/not configured until the post-Core remote milestone. This does not mean secure tunnel transport is implemented in Phase 1.

Interfaces may be designed so future adapters can plug in, but Phase 1 code must not implement Phase 2/3 capabilities “เผื่อไว้”.

## 6. Security posture and threat model

### 6.1 Explicit posture

Phase 1 is designed for:

```text
Trusted developer workstation
+
Administrator-level local automation
+
authenticated localhost HTTP clients
+
same-user stdio/named-pipe clients
+
strong structured-tool path policy
+
human-gated R3 destructive actions
+
session-scoped Guarded/Autonomous policy
+
untrusted-content provenance
+
bounded resource admission
+
audit + best-effort shell inspection
```

Phase 1 is **not** designed for:

```text
Hostile multi-user machine
Malware resistance
Privilege isolation
Strong shell/process containment
Automatic safety of arbitrary repository scripts
Tamper-proof forensic logging
```

The HTTP session token reduces the chance that an arbitrary local process can trivially call the privileged HTTP MCP endpoint. It does not protect against malware or a process that can steal the token from the same user session, process memory, explicit clipboard exposure or another compromised trusted client.

### 6.2 Canonical security invariants

Security-sensitive requirements use stable identifiers so enforcement can be referenced from policy, persistence, UI and tests without relying on duplicated prose as the source of truth.

**SEC-R3-001 — Human authorization**  
Successful R3 execution requires independent human-authorized final approval.

**SEC-R3-002 — No requester self-approval**  
Requester agent/client credentials can never satisfy `SEC-R3-001` for the same R3 request.

**SEC-R3-003 — Autonomous does not downgrade R3**  
`Autonomous / Unrestricted` mode cannot convert R3 into R2/R1 and cannot bypass human approval.

**SEC-MODE-001 — Session reset**  
Every new Agent session starts in `Guarded`. Autonomous mode is session-scoped and is never restored automatically after restart/crash/relaunch.

**SEC-AUTH-001 — Loopback + authentication**  
Local HTTP MCP requires both loopback-only binding and the current per-session bearer credential.

**SEC-CONTENT-001 — Content is data, not authority**  
Text/content returned from workspaces, Git, process output, future browser/network sources or other external data cannot itself grant permission, change security mode, authorize approval, or alter policy boundaries.

**SEC-EXEC-001 — Repository scripts are code execution**  
Known project commands do not imply trusted side effects. Repository-controlled scripts/hooks execute arbitrary code under Agent privileges and must be modeled as an execution boundary.

**SEC-AUDIT-001 — Operational audit boundary**  
Normal application execution never mutates/deletes historical audit events. Only the dedicated retention/archive maintenance path may prune eligible audit data under explicit retention rules.

**SEC-RESOURCE-001 — Bounded admission**  
Transport requests, concurrent tool executions, managed processes and pending approvals must have bounded admission/backpressure.

### 6.3 Security boundaries that must remain explicit

- **Loopback binding ≠ authentication.** HTTP must satisfy `SEC-AUTH-001`.
- **Authentication ≠ sandbox.** A client with valid credentials can invoke exposed tools; raw shell remains privileged execution.
- **Structured path policy is deterministic application control.** Structured tools resolve canonical paths and apply workspace policy before execution.
- **Shell inspection is best effort.** PowerShell/cmd can invoke scripts, encoded commands, downloaded executables and child processes whose behavior cannot be statically determined reliably.
- **Workspace registration ≠ instruction trust.** A registered repository may contain malicious README text, comments, generated files, commit messages or scripts.
- **Known command name ≠ known behavior.** `npm test`, `pnpm build`, Git hooks and package lifecycle scripts can execute arbitrary repository-controlled code.
- **R3 approval is a human gate.** `SEC-R3-001..003` are fixed Phase 1 guarantees.
- **R2 autonomous approval is mode-gated.** Requester self-approval is allowed only after explicit human opt-in for the current session.
- **Audit is operational, not forensic-grade.** An Administrator can still modify the local SQLite database outside the application.
- **Retention does not apply to user/project data.** Audit/process-log maintenance authority never grants permission to delete source/user files.

### 6.4 Untrusted content / indirect-instruction threat model

LocalGPT expects the upstream AI client to read data and decide subsequent tool calls. Data read by the client may contain adversarial text intended to influence that decision.

Examples include:

- README or source comments saying “run this command”
- generated files that contain instructions aimed at the model
- Git commit messages/history containing tool-like instructions
- shell/process output containing deceptive instructions
- files read outside the workspace
- future browser/CDP/network content

The Agent host cannot reliably determine whether natural-language content is a malicious prompt injection. Instead it enforces a boundary:

> **Tool output and workspace content are data, not authority.** Content provenance may inform the orchestrator, but content cannot directly modify AgentCore policy, security mode, approval authority or transport credentials.

Tool result schemas that surface potentially model-consumed text should carry bounded provenance metadata where practical, for example:

```ts
type ContentProvenance = {
  source:
    | "workspace_file"
    | "external_file"
    | "git_metadata"
    | "process_output"
    | "system_generated"
    | "agent_generated"
  trust: "untrusted_content" | "local_operational" | "human_authoritative"
  workspaceId?: string
  path?: string
}
```

Rules:

- `workspace_file`, `external_file`, `git_metadata` and process stdout/stderr default to `untrusted_content` for instruction-authority purposes
- `human_authoritative` is reserved for explicit Control Center actions/approvals, not arbitrary text in a project file
- provenance is an orchestration/security hint, not a sandbox and not a substitute for PolicyEngine checks
- a file being inside a registered workspace does not upgrade its instruction trust
- future adapters must map their returned content into the same provenance concept

### 6.5 Repository-controlled execution trust boundary

Structured project operations such as `test`, `lint`, `typecheck`, `build`, `project_dev`, dependency installation and Git commit may execute repository-controlled code via:

- `package.json` scripts
- package lifecycle hooks
- Git hooks
- local executables/binaries
- child scripts/programs
- toolchain plugins/configuration

Therefore `SEC-EXEC-001` applies even when the top-level command is familiar.

Phase 1 intentionally does **not** require approval before every test/build because autonomous coding would become unusable. Instead:

1. structured project commands remain R1 when they match the detected project adapter contract
2. tool result/audit must record that execution is repository-controlled, for example `executionTrust="repository_controlled"`
3. the UI must not describe these executions as sandboxed or side-effect-free
4. network/secret isolation is not guaranteed by Phase 1
5. if a structured project command expands into an obviously R2/R3 direct action before launch, normal policy escalation still applies
6. arbitrary raw-shell execution that is opaque/unrecognized follows the stricter rules in Section 11

This is an explicit product tradeoff: normal coding scripts run automatically, but the user is informed that repositories themselves are executable trust boundaries.

## 7. Top-level architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Electron Desktop Control Center                                      │
│                                                                      │
│  ┌──────────────────── React Renderer ─────────────────────────────┐ │
│  │ Dashboard │ Projects │ Git │ Activity │ Processes              │ │
│  │ Approvals │ Tunnel │ Doctor │ Settings                         │ │
│  └────────────────────────┬─────────────────────────────────────────┘ │
│                           │ typed preload IPC                         │
│  ┌────────────────────────▼─────────────────────────────────────────┐ │
│  │ Agent Core                                                       │ │
│  │                                                                  │ │
│  │ McpGateway                                                       │ │
│  │ ├─ localhost HTTP → SessionTokenAuth                             │ │
│  │ └─ named-pipe RPC target for stdio bridge                        │ │
│  │          │                                                       │ │
│  │          ▼                                                       │ │
│  │ ToolDispatcher → ToolRegistry                                    │ │
│  │          │                                                       │ │
│  │          ▼                                                       │ │
│  │ PolicyEngine                                                     │ │
│  │ ├─ PathPolicy                                                    │ │
│  │ ├─ ShellInspector                                                │ │
│  │ ├─ SecurityModePolicy                                            │ │
│  │ └─ ExecutionTrustPolicy                                          │ │
│  │          │                                                       │ │
│  │          ├─ allow ───────────────► ResourceGovernor ─► Execution │ │
│  │          └─ approval ─► ApprovalManager ──────────────► Execution │ │
│  │                                                                  │ │
│  │ Execution Layer                                                  │ │
│  │ ├─ Workspace/File                                                │ │
│  │ ├─ Git                                                           │ │
│  │ ├─ Project Adapter                                               │ │
│  │ ├─ Shell                                                         │ │
│  │ └─ ProcessManager                                                │ │
│  │                                                                  │ │
│  │ DoctorService                                                    │ │
│  │ AuditRetentionService                                            │ │
│  │          │                                                       │ │
│  │          ▼                                                       │ │
│  │ EventBus ───────────────► AuditWriter ─► SQLite                  │ │
│  │    └────────────────────► UiBroadcaster                          │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

MCP client ──stdio──► mcp-stdio-bridge ──Windows named pipe──► Agent Core
```

Future remote transport must connect to the same `McpGateway → ToolDispatcher → PolicyEngine → ApprovalManager → ResourceGovernor → Execution → Audit` pipeline. It must not create a parallel policy/process/audit stack.

### 7.1 Core rules

- MCP transports contain no tool business logic.
- HTTP authentication occurs before HTTP requests reach tool dispatch.
- Tool implementations do not make their own permission decisions.
- React renderer never touches filesystem, shell, SQLite or MCP server internals directly.
- Every tool invocation follows one lifecycle owned by `ToolDispatcher`.
- Every privileged action is auditable through the same event model.
- Security mode changes are owned by AgentCore, initiated through explicit Control Center action and audited.
- Security mode is runtime session state, not a persistent “last used mode”.
- Doctor diagnostics consume health/state services but do not bypass policy or become a second execution layer.
- `ResourceGovernor` is admission/backpressure infrastructure, not an authorization bypass.
- `AuditRetentionService` is the only application path allowed to prune eligible historical audit/output data.

## 8. Standard tool execution lifecycle

Every MCP tool call follows this conceptual path:

```text
HTTP only: authenticate session token
        ↓
1. Transport admission / rate and concurrency check
2. Receive request
3. Assign toolCallId
4. Runtime schema validation
5. Resolve client/session/workspace/path context
6. Attach content/execution provenance where applicable
7. Policy classification
8. Write policy/audit events
9a. ALLOW → ResourceGovernor → execute
9b. APPROVAL_REQUIRED → freeze request → create approval → return APPROVAL_REQUIRED
9c. DENY → return stable policy error
10. Capture result or failure
11. Persist final state + audit event
12. Map to MCP result
```

Authentication failures are rejected before executable tool calls are created. They may create bounded security audit events that never contain the supplied token.

Tool implementations must never send MCP responses directly.

## 9. Permission model

### 9.1 Risk levels

| Level | Meaning | Default behavior |
|---|---|---|
| `R0 READ_ONLY` | No host state mutation | Auto allow |
| `R1 NORMAL_WRITE_EXECUTE` | Normal coding write/execute inside workspace or structured project execution | Auto allow |
| `R2 SENSITIVE` | External/system/high-impact/opaque execution | Approval required; requester self-approval only in current-session Autonomous mode |
| `R3 DESTRUCTIVE` | Likely user/project data or state loss | Independent human-authorized approval required; requester self-approval prohibited |

### 9.2 Examples

`R0`:

- `read_file`
- `search_text`
- `workspace_tree`
- `workspace_snapshot`
- `git_status`
- `git_diff`
- `git_log`
- `process_output`
- `health`
- `system_info`

`R1`:

- create/edit source files inside workspace
- normal line additions/removals through `apply_patch`
- structured project `test/lint/typecheck/build/dev`
- package-manager operation selected by the project adapter when not otherwise escalated
- `git add`
- `git commit` while acknowledging Git hooks as repository-controlled execution
- known direct non-destructive shell commands with transparent semantics

`R2`:

- create/edit outside workspace
- registry/system configuration mutation
- service/firewall/scheduled-task mutation
- system-wide install
- ACL/ownership change
- terminating a process the Agent did not create
- unregistering workspace metadata
- unknown/opaque raw-shell command shapes in Guarded mode
- encoded/interpreted/download-and-execute patterns not already classified R3
- other high-impact actions that are not direct data destruction

`R3`:

- deleting a user/project file or directory
- deleting a file through `apply_patch`
- truncating/clearing user/project content in a way classified as data loss
- `git reset --hard`
- `git clean -fd`
- destructive `git restore` / `git checkout -- <path>` patterns
- force push when classified as destructive repository-state mutation
- destructive disk/filesystem operations

### 9.3 Path policy matrix

| Target | Read | Create/Edit | Delete |
|---|---:|---:|---:|
| Registered workspace | Allow | Allow | R3 human approval |
| Outside workspace | Allow | R2 approval | R3 human approval |
| Windows/system-sensitive path | Allow if OS ACL allows | R2/R3 by semantics | R3 human approval |

Agent-owned ephemeral runtime artifacts under the Agent data directory are a special operational class. Bounded retention/rotation/pruning of Agent-generated temporary process logs and expired operational audit data may occur according to Section 18 without user approval. This exception must never apply to source files, registered workspace data, user documents or other user/project data.

### 9.4 Security modes

Phase 1 defines two local operating modes.

**Guarded — mandatory session default**

- every new Agent session starts here (`SEC-MODE-001`)
- R0 auto allow
- R1 auto allow
- R2 requires explicit human approval in Control Center
- unknown/opaque raw-shell execution is R2
- R3 requires explicit human approval in Control Center
- MCP `approval_decide(... allow ...)` cannot final-allow R2 or R3

**Autonomous / Unrestricted — explicit session-scoped opt-in**

- user must enable it again for each new Agent session
- mode is never persisted as an auto-restored runtime mode
- R0 auto allow
- R1 auto allow
- R2 may be final-approved by an authenticated MCP client, including the requester itself
- R2 requester self-approval must set `selfApproved=true` and `autonomousMode=true` in audit/UI
- unknown/opaque raw-shell execution remains at least R2; Autonomous permits R2 self-approval but does not relabel it R1
- R3 still requires independent human authorization under `SEC-R3-001..003`

Changing mode must require explicit Control Center action with warning/confirmation and audit. Phase 1 MCP tools must not provide an operation that enables Autonomous mode.

## 10. Canonical path policy

Structured filesystem operations must not rely on string-prefix checks.

Required pipeline:

```text
requested path
   ↓
absolute resolution
   ↓
Windows normalization
   ↓
symlink/junction/reparse-point resolution where applicable
   ↓
canonical target
   ↓
canonical workspace comparison
```

Must cover regression cases for:

- `..` traversal
- mixed `/` and `\`
- case-insensitive Windows paths
- nested workspaces
- sibling prefix confusion (`C:\Project` vs `C:\ProjectFake`)
- symlinks
- NTFS junctions / reparse points
- invalid/unresolvable paths

All structured filesystem tools use one shared API such as:

```ts
PathPolicy.resolve({
  workspaceId,
  requestedPath,
  operation: "read" | "write" | "delete"
})
```

## 11. Shell and execution-trust policy

### 11.1 Shell capability

`shell` accepts PowerShell or cmd commands and may run foreground or background processes with Administrator privileges.

There is no complete command allowlist in Phase 1. The system instead combines semantic risk with execution transparency.

### 11.2 Execution transparency classes

`ShellInspector` returns both risk and transparency:

```ts
type ExecutionTransparency =
  | "direct_known"
  | "indirect_repository_controlled"
  | "opaque_or_unknown"
```

**`direct_known`**  
Command behavior is recognizable at the command-line level and does not contain known indirect/opaque indicators. This does not prove child behavior is safe.

**`indirect_repository_controlled`**  
Execution delegates to repository-controlled scripts/hooks/plugins, including structured project commands. ProjectAdapter may still classify this as R1 by explicit product rule while recording the execution trust boundary.

**`opaque_or_unknown`**  
The command shape cannot be classified with sufficient confidence, invokes encoded/interpreted content, unknown executables/scripts, or otherwise hides meaningful behavior from inspection.

Rules:

- Guarded mode: `opaque_or_unknown` raw-shell execution is at least R2
- Autonomous mode: `opaque_or_unknown` remains R2 but requester self-approval may be permitted under R2 rules
- known R3 semantics always override transparency and route to human-only R3 approval
- structured ProjectAdapter operations can remain R1 while recording `executionTrust=repository_controlled`
- `unknownRisk=true` may be retained as an audit flag, but it is not sufficient by itself; Guarded must gate the execution as R2

### 11.3 Inspector coverage

Before execution, `ShellInspector` performs best-effort classification for at least:

**Deletion/data loss**

- `Remove-Item`
- `del`, `erase`
- `rd`, `rmdir`
- `Clear-Content`
- destructive filesystem utilities

**Git destructive operations**

- `reset --hard`
- `clean`
- destructive restore/checkout patterns
- `push --force` / `-f`

**System/process operations**

- `Stop-Process`
- `taskkill`
- `sc delete`
- registry mutation/deletion
- shutdown/reboot tooling
- disk/boot utilities

**Opaque/high-risk execution indicators**

- `Invoke-Expression` / `iex`
- encoded PowerShell
- download-and-execute patterns
- unknown executable/script target
- shell indirection that prevents reliable target extraction

The UI and documentation must never claim shell inspection is containment.

When the inspector detects R3 semantics, the request is frozen and routed to the human-only R3 approval path. Autonomous mode cannot downgrade or bypass it.

### 11.4 CWD

If `workspaceId` is supplied and no `cwd` is provided, default to workspace root.

Relative `cwd` resolves from workspace root.

Absolute `cwd` outside the workspace is allowed by raw-shell capability but must be marked in audit context, e.g. `externalCwd=true`. External write classification remains best effort for raw shell.

### 11.5 Environment variables and secret exposure

Shell may receive additional environment variables, but audit views must not persist/display sensitive plaintext values by default.

At minimum redact values for key patterns containing:

- `TOKEN`
- `SECRET`
- `PASSWORD`
- `API_KEY`
- `AUTH`
- `PRIVATE_KEY`

Store key names and redacted values for operational visibility.

The per-session HTTP MCP token is a transport secret and must never be included in shell audit arguments, normal operational logs or generic environment dumps.

Phase 1 does not guarantee that arbitrary repository scripts cannot read environment variables or access the network. That limitation is part of `SEC-EXEC-001` and must be visible in Security/Doctor documentation.

## 12. Approval model

### 12.1 Approval guarantee

Fixed Phase 1 guarantee:

> **Delete/destructive actions require an independent human-authorized approval and cannot be silently self-approved by the requesting agent.**

This is `SEC-R3-001..003`.

For Phase 1, the human-authorized final-approval channel is an explicit user action in Electron Control Center. Future remote milestones may add authenticated human approval channels but must preserve the same R3 guarantees.

### 12.2 Approval semantics

The original tool request does not stay blocked on an open MCP connection.

```text
1. Client invokes risky tool.
2. Policy returns APPROVAL_REQUIRED.
3. Agent freezes exact tool name + arguments and creates approvalRequestId.
4. Original call returns APPROVAL_REQUIRED + approvalRequestId.
5. A decision arrives through Control Center or MCP approval API.
6. ApprovalManager validates risk-specific approver rules + current session mode.
7. If final ALLOW is authorized, Agent validates frozen request hash.
8. Agent atomically transitions the request and executes that exact request once.
9. Decision call / UI receives execution outcome.
```

### 12.3 Risk-specific approval authority

**R2 — SENSITIVE**

- Guarded: final allow requires explicit human approval in Control Center
- Autonomous: authenticated MCP client may final-allow R2
- if `requesterClientId === approverClientId`, allowed only in Autonomous and visibly audited

**R3 — DESTRUCTIVE**

- successful execution requires `humanAuthorized=true` (`SEC-R3-001`)
- Phase 1 produces `humanAuthorized=true` only from explicit Control Center user action
- MCP `approval_decide(... allow ...)` for R3 returns `HUMAN_APPROVAL_REQUIRED`
- requester cannot be final approver of its own R3 request (`SEC-R3-002`)
- Autonomous does not alter this (`SEC-R3-003`)

Any otherwise-authorized actor may deny; deny never executes the frozen request.

### 12.4 State machine

```text
PENDING
   ├─ deny ─────────────► DENIED
   ├─ expire ───────────► EXPIRED
   ├─ shutdown ─────────► CANCELLED
   └─ authorized allow
          ▼
       EXECUTING
          ├─ success ───► COMPLETED
          └─ failure ───► FAILED
```

Unauthorized allow attempts do not transition out of `PENDING`; they return stable policy errors and may create audit events.

Terminal approval states are not revivable. A new execution attempt requires a new tool call and approval request.

### 12.5 Exactly-once protection

The Agent must prevent:

- double approval causing double execution
- replay of completed approvals
- mutation of command/arguments after approval
- execution of expired/denied/cancelled approvals
- unauthorized R3 MCP allow becoming executable state

Before enqueueing approval, canonicalize/hash frozen tool request. Before execution, the hash must still match.

Approval decision uses an atomic database transition so concurrent UI/MCP decisions cannot both execute the request.

### 12.6 Self-approval audit

`selfApproved=true` is valid only for R2 in current-session Autonomous mode.

For R3, successful approvals must never have `selfApproved=true`. A requester self-approval attempt is rejected/audited without execution.

### 12.7 Expiration

Default approval expiration: **15 minutes**.

Settings may support 5, 15, 30 or 60 minutes.

## 13. Workspace model

Use a multi-workspace registry keyed by opaque `workspaceId`.

```ts
Workspace {
  id
  name
  rootPath
  canonicalRootPath
  projectType
  packageManager
  gitRoot
  createdAt
  updatedAt
  lastOpenedAt
  enabled
}
```

All structured file/Git/project tools receive `workspaceId`; they do not accept unrestricted project roots as the main addressing model.

`workspace_snapshot` is a logical project snapshot, not a backup.

Workspace registration grants filesystem/project scope, **not instruction trust**. Text read from a registered workspace remains `untrusted_content` for `SEC-CONTENT-001`.

## 14. MCP tool catalog — Phase 1

Exact runtime schemas must be defined once in shared runtime-validatable schemas and reused for MCP, IPC and internal types where appropriate.

### 14.1 Workspace tools

#### `workspace_list`
- Policy: R0
- Lists registered workspaces

#### `workspace_get`
- Policy: R0
- Input: `workspaceId`

#### `workspace_register`
- Policy: R1/CONTROL
- Canonicalizes path and runs project detection
- Does not upgrade content trust

#### `workspace_update`
- Policy: R1/CONTROL

#### `workspace_unregister`
- Policy: R2 according to security mode
- Removes registry metadata only; does not delete project files

#### `workspace_tree`
- Policy: R0
- Bounded directory tree

#### `workspace_snapshot`
- Policy: R0
- Bounded tree/project/Git summary

### 14.2 File/search tools

#### `read_file`
Reads bounded text content.

- Policy: R0
- Input: `workspaceId`, `path`, optional line/byte bounds
- Binary input returns metadata/unsupported result
- content returned to model-facing clients carries `ContentProvenance`

#### `search_text`
Searches project text using an implementation-selected backend such as ripgrep.

- Policy: R0
- Results are bounded
- result text carries `ContentProvenance`

#### `apply_patch`
Creates/modifies text files, including multi-file patches.

- normal content edits inside workspace: R1
- external target: R2
- file deletion/destructive truncation: R3 human approval

Requirements:

- validate all targets before modification
- compute edits before final writes where practical
- use temporary files + atomic replace per file where practical
- report partial failure explicitly
- ordinary source-line deletions are normal edits; deleting/truncating the file itself is destructive

No generic `write_file` is required in Phase 1.

### 14.3 Git tools

#### `git_status`
- Policy: R0

#### `git_diff`
- Policy: R0
- Bounded staged/unstaged/path filtering
- returned text carries `git_metadata` provenance

#### `git_log`
- Policy: R0
- Bounded history/range
- commit messages are `untrusted_content` for instruction authority

No dedicated `git_commit`, `git_push`, `git_reset` is required in Phase 1. Raw shell routes through the same ShellInspector/R2/R3 policy.

### 14.4 Project tools

#### `project_info`
Detects Node/TypeScript metadata, package manager, scripts, Git root and capabilities.

- Policy: R0

#### `project_dev`
Starts detected dev script as Agent-managed background process.

- Policy: R1 by ProjectAdapter contract
- `executionTrust="repository_controlled"`

#### `test`
#### `lint`
#### `typecheck`
#### `build`

- Policy: R1 by ProjectAdapter contract
- use detected package manager/script only
- if absent: `PROJECT_SCRIPT_NOT_FOUND`; do not invent a command
- mark `executionTrust="repository_controlled"`
- return actual command, exit code, duration, bounded stdout/stderr and truncation flags

The R1 classification is an explicit product choice and does not mean project scripts are sandboxed.

For non-Node repos, file/Git/raw shell remain usable; no stack-specific adapter is promised in Phase 1.

### 14.5 Shell tool

#### `shell`
Runs PowerShell or cmd.

```ts
{
  command: string
  workspaceId?: string
  cwd?: string
  shell?: "powershell" | "cmd"
  env?: Record<string, string>
  timeoutMs?: number
  mode?: "foreground" | "background"
}
```

Policy is dynamic and includes `riskLevel`, `executionTransparency`, `executionTrust`, `unknownRisk` and inspector flags.

- opaque/unknown raw-shell in Guarded → at least R2
- R3 → human-authorized approval only

### 14.6 Process tools

#### `process_list`
Lists Agent-managed processes by default.

#### `process_get`
Returns process metadata/state.

#### `process_output`
Returns bounded incremental stdout/stderr with cursor/offset semantics and `process_output` provenance.

#### `process_stop`
- Agent-managed process: auto allow
- external process: normally R2; destructive semantics may elevate to R3

#### `process_restart`
Restarts an Agent-managed process using frozen launch metadata.

Agent `processId` is primary identity; Windows PID is metadata only.

### 14.7 Approval tools

#### `approval_list`
Lists pending/recent approvals.

#### `approval_get`
Returns frozen request, risk reasons, requester, expiry, security mode at request time/current mode, whether self-approval is permitted and whether human approval is required.

#### `approval_decide`

```ts
{
  approvalRequestId: string
  decision: "allow" | "deny"
  note?: string
}
```

- deny: transitions valid pending request to DENIED
- R2 allow: succeeds only if current session mode/approval source permits it
- R2 requester self-allow: only Autonomous, visibly audited
- R3 allow from MCP: rejected with `HUMAN_APPROVAL_REQUIRED`
- Control Center human approval invokes the same ApprovalManager/hash/exactly-once path

### 14.8 Operational tools

#### `health`
Bounded machine-readable Agent/SQLite/transport/process/resource-governor summary.

#### `system_info`
Bounded coding-relevant Windows/system info such as OS, architecture, CPU/RAM summary, disk summary and Node/Git availability.

Doctor is a first-class Control Center service/page, not required as a broad privileged MCP tool in Phase 1.

## 15. Result and error contracts

Use stable internal result envelopes/error codes. MCP mapping may adapt the envelope without forcing clients to parse English error text.

```ts
type ToolResult<T> =
  | { ok: true; toolCallId: string; data: T }
  | {
      ok: false
      toolCallId: string
      error: {
        code: string
        message: string
        details?: unknown
      }
    }
```

Required errors:

**Authentication/transport**

- `AUTH_REQUIRED`
- `AUTH_INVALID`
- `HTTP_BIND_NOT_LOOPBACK`
- `PORT_IN_USE`
- `AGENT_NOT_RUNNING`
- `RATE_LIMITED`
- `TOO_MANY_CONCURRENT_REQUESTS`

**Validation**

- `INVALID_ARGUMENT`
- `UNKNOWN_TOOL`
- `INVALID_WORKSPACE`
- `INVALID_PATH`

**Policy/approval**

- `APPROVAL_REQUIRED`
- `POLICY_DENIED`
- `HUMAN_APPROVAL_REQUIRED`
- `SELF_APPROVAL_NOT_ALLOWED`
- `AUTONOMOUS_MODE_REQUIRED`
- `APPROVAL_EXPIRED`
- `APPROVAL_ALREADY_DECIDED`
- `REQUEST_HASH_MISMATCH`
- `OPAQUE_EXECUTION_REQUIRES_APPROVAL`

**Resource governance**

- `PROCESS_LIMIT_REACHED`
- `APPROVAL_QUEUE_FULL`
- `OUTPUT_LIMIT_REACHED`

**Execution**

- `COMMAND_FAILED`
- `COMMAND_TIMEOUT`
- `PROCESS_START_FAILED`
- `PROCESS_NOT_FOUND`
- `FILE_READ_FAILED`
- `PATCH_FAILED`
- `GIT_FAILED`
- `PROJECT_SCRIPT_NOT_FOUND`

**System**

- `DATABASE_ERROR`
- `INTERNAL_ERROR`
- `AGENT_SHUTTING_DOWN`

Authentication errors must never echo supplied bearer tokens.

## 16. SQLite persistence

### 16.1 Storage location

Use a local per-user application-data directory such as:

```text
%LOCALAPPDATA%\LocalGPT Agent\
├─ agent.db
├─ logs\
├─ exports\
└─ runtime\
```

Exact path derives through Windows/Electron application-data APIs.

Source files remain in workspaces. Plaintext HTTP session token is memory-only and never persisted in SQLite.

### 16.2 SQLite requirements

- WAL mode
- foreign keys enabled
- bounded busy timeout
- schema migrations from first release
- renderer never opens SQLite directly
- repositories own persistence access

Logical repositories:

- `WorkspaceRepository`
- `SessionRepository`
- `ClientRepository`
- `ToolCallRepository`
- `ApprovalRepository`
- `ProcessRepository`
- `AuditRepository`
- `SettingsRepository`

### 16.3 `workspaces`

```text
id TEXT PK
name TEXT
root_path TEXT
canonical_root_path TEXT
project_type TEXT
package_manager TEXT NULL
git_root TEXT NULL
created_at DATETIME
updated_at DATETIME
last_opened_at DATETIME NULL
is_enabled INTEGER
```

### 16.4 `sessions`

```text
id TEXT PK
started_at DATETIME
ended_at DATETIME NULL
app_version TEXT
windows_user TEXT
windows_session_id TEXT
elevated INTEGER
http_auth_method TEXT
http_auth_enabled INTEGER
security_mode TEXT
shutdown_reason TEXT NULL
status TEXT
```

Statuses:

- `STARTING`
- `RUNNING`
- `SHUTTING_DOWN`
- `STOPPED`
- `CRASHED`

Rules:

- new row always starts with `security_mode='guarded'` (`SEC-MODE-001`)
- historical mode changes may be reflected in session/audit state
- previous session Autonomous state is never used to initialize a new session
- authentication method/state may be recorded, bearer token may not

### 16.5 `clients`

```text
id TEXT PK
session_id TEXT
name TEXT
transport TEXT
authenticated INTEGER
auth_method TEXT NULL
connected_at DATETIME
last_seen_at DATETIME
disconnected_at DATETIME NULL
metadata_json TEXT
```

Transport values include `stdio`, `http`, `internal_ui`.

HTTP `authenticated=1` means possession of current session token was verified. It is not a human identity.

stdio/named-pipe trust relies on local process/user boundary plus named-pipe ACL design.

### 16.6 `tool_calls`

```text
id TEXT PK
session_id TEXT
client_id TEXT
workspace_id TEXT NULL
tool_name TEXT
arguments_json TEXT
arguments_hash TEXT
risk_level TEXT
policy_decision TEXT
policy_reasons_json TEXT
inspector_flags_json TEXT
execution_transparency TEXT NULL
execution_trust TEXT NULL
content_provenance_json TEXT NULL
status TEXT
requested_at DATETIME
execution_started_at DATETIME NULL
execution_finished_at DATETIME NULL
duration_ms INTEGER NULL
result_summary_json TEXT NULL
error_code TEXT NULL
error_message TEXT NULL
```

Statuses:

- `RECEIVED`
- `AWAITING_APPROVAL`
- `EXECUTING`
- `SUCCEEDED`
- `FAILED`
- `DENIED`
- `EXPIRED`
- `CANCELLED`

Audit-safe arguments may be redacted. Approval integrity hash derives from canonical frozen request, not display-redacted representation.

Transport auth headers/tokens must never be stored in `arguments_json` or provenance metadata.

### 16.7 `approvals`

```text
id TEXT PK
tool_call_id TEXT UNIQUE
requester_client_id TEXT
approver_client_id TEXT NULL
approver_kind TEXT NULL
approval_source TEXT NULL
risk_level TEXT
reasons_json TEXT
frozen_arguments_json TEXT
request_hash TEXT
status TEXT
created_at DATETIME
expires_at DATETIME
decided_at DATETIME NULL
execution_started_at DATETIME NULL
completed_at DATETIME NULL
decision_note TEXT NULL
human_authorized INTEGER
self_approved INTEGER
autonomous_mode INTEGER
```

`approver_kind`: `HUMAN_UI`, `MCP_CLIENT`  
`approval_source`: `control_center`, `mcp`

Invariant requirements:

- successful R3 requires `human_authorized=1` (`SEC-R3-001`)
- successful R3 has `approver_kind=HUMAN_UI` in Phase 1
- successful R3 never has `self_approved=1` (`SEC-R3-002`)
- R2 `self_approved=1` requires `autonomous_mode=1`

Statuses: `PENDING`, `DENIED`, `EXPIRED`, `CANCELLED`, `EXECUTING`, `COMPLETED`, `FAILED`.

### 16.8 `processes`

```text
id TEXT PK
session_id TEXT
workspace_id TEXT NULL
pid INTEGER
parent_pid INTEGER NULL
process_start_timestamp DATETIME NULL
owner_type TEXT
command TEXT
cwd TEXT
shell_type TEXT
execution_trust TEXT NULL
status TEXT
started_at DATETIME
exited_at DATETIME NULL
exit_code INTEGER NULL
restart_count INTEGER
created_by_tool_call_id TEXT
```

`owner_type`: `AGENT_MANAGED`, `EXTERNAL`.

### 16.9 Process output storage

Do not keep unbounded stdout/stderr in one SQLite row.

```text
runtime\processes\<processId>\stdout.log
runtime\processes\<processId>\stderr.log
```

SQLite chunk/index metadata:

```text
process_output_chunks
- id
- process_id
- stream
- sequence
- byte_offset
- byte_length
- created_at
- preview_text
```

Default MCP response cap for foreground stdout/stderr: **256 KB per stream**. Larger output returns truncation metadata and remains accessible through `process_output` while retained.

### 16.10 `audit_events`

```text
id TEXT PK
session_id TEXT
tool_call_id TEXT NULL
client_id TEXT NULL
workspace_id TEXT NULL
event_type TEXT
severity TEXT
payload_json TEXT
created_at DATETIME
```

Minimum vocabulary:

- `SESSION_STARTED`
- `CLIENT_CONNECTED`
- `HTTP_AUTH_REJECTED`
- `SECURITY_MODE_CHANGED`
- `TOOL_REQUEST_RECEIVED`
- `POLICY_DECIDED`
- `OPAQUE_EXECUTION_CLASSIFIED`
- `APPROVAL_CREATED`
- `APPROVAL_ALLOW_REJECTED`
- `APPROVAL_APPROVED`
- `APPROVAL_DENIED`
- `APPROVAL_EXPIRED`
- `APPROVAL_CANCELLED`
- `EXECUTION_STARTED`
- `EXECUTION_SUCCEEDED`
- `EXECUTION_FAILED`
- `PROCESS_STARTED`
- `PROCESS_EXITED`
- `PROCESS_STOP_REQUESTED`
- `WORKSPACE_REGISTERED`
- `WORKSPACE_UPDATED`
- `WORKSPACE_UNREGISTERED`
- `DOCTOR_RUN_COMPLETED`
- `RESOURCE_ADMISSION_REJECTED`
- `AUDIT_RETENTION_RUN`
- `AGENT_SHUTDOWN_STARTED`
- `AGENT_SHUTDOWN_COMPLETED`

`HTTP_AUTH_REJECTED` never contains submitted token/authorization header.

`tool_calls` represents current/final state; `audit_events` reconstructs timeline. Activity UI may render Live Logs / MCP Activity without changing the source-of-truth model.

Audit is operational, not tamper-proof. An Administrator can modify SQLite outside the app.

### 16.11 Settings and session-only state

Persistent settings include:

- `mcp.http.enabled`
- `mcp.http.port`
- `mcp.http.authMode = session_bearer`
- `mcp.stdio.enabled`
- `shell.default`
- `shell.defaultTimeoutMs`
- `approval.expirationMinutes`
- `process.shutdownGraceMs`
- resource-governance limits
- audit/process-output retention controls

The following are **not persistent runtime-restoration settings**:

- plaintext HTTP token
- current `security.mode`

Every session initializes:

```text
security.mode = guarded
```

The UI may display the current mode and history, but must not persist Autonomous in a way that auto-restores it after restart.

Default shutdown grace: **5 seconds**.

## 17. HTTP session authentication

### 17.1 Token generation and lifetime

At every new Agent session:

1. generate a cryptographically random secret using CSPRNG
2. minimum equivalent entropy: **256 random bits**
3. keep plaintext token only in AgentCore memory
4. use as bearer credential for localhost HTTP MCP
5. rotate by generating a new token every session/restart
6. clear old references on shutdown as best effort

### 17.2 HTTP contract

```text
Authorization: Bearer <session-token>
```

- missing → `AUTH_REQUIRED`
- malformed/incorrect → `AUTH_INVALID`
- current token → proceed
- previous-session token → `AUTH_INVALID`

Use timing-safe comparison where practical.

### 17.3 Secret exposure rules

Token must not appear in:

- SQLite plaintext fields
- audit payloads
- normal logs
- errors
- Doctor output
- Dashboard normal state
- process env dumps

Control Center may provide explicit **Copy session credential** action. It must be intentional, sensitive, and routed through privileged main/preload IPC. Normal renderer state must not continuously receive the secret.

Clipboard copy should warn that other applications may read clipboard contents.

## 18. Audit retention and operational maintenance

### 18.1 Retention defaults

Phase 1 defaults:

```text
Active audit-event retention: 30 days
Full process-output retention: 7 days
Manual audit export: JSONL
```

Retention values may be configurable, but a fresh installation uses these defaults.

### 18.2 Append-only semantics

`audit_events` are append-only for all normal Agent execution and business services (`SEC-AUDIT-001`).

A dedicated `AuditRetentionService` is the only application component allowed to archive/prune eligible historical audit/process-output data. This is maintenance authority, not general deletion authority.

Rules:

- retention service never deletes workspace/source/user files
- retention run produces its own summary audit event before/after pruning as implementation permits
- manual JSONL export is available before pruning for users who need longer history
- retention never rewrites historical event payloads to change meaning
- active pending/executing approvals/tool calls/process metadata must not be pruned
- retention maintenance is not exposed as an arbitrary MCP delete tool

This preserves operational “append-only during execution” semantics without allowing SQLite/runtime storage to grow without bound indefinitely.

## 19. Resource governance and backpressure

`SEC-RESOURCE-001` applies even to local-only Phase 1 because uncontrolled local clients can exhaust memory, processes or approval queues.

Default Phase 1 limits:

```text
Global concurrent tool executions: 8
Concurrent executions per MCP client: 4
Agent-managed process limit: 32
Pending approval limit: 100
Authenticated HTTP request admission: 120 requests/minute per client session, burst 30
```

Failed authentication receives separate throttling/backoff; implementation should cap repeated failures and apply bounded delay without logging the token.

Requirements:

- reaching a limit returns a stable error rather than silently dropping requests
- queued work must be bounded
- output remains bounded independently of request-rate limits
- approval creation is rejected with `APPROVAL_QUEUE_FULL` when full; no risky action executes
- process start is rejected with `PROCESS_LIMIT_REACHED` when full
- UI/Doctor expose resource pressure/limit configuration without secrets
- local defaults may be configurable within safe validated ranges

Release 1.1 remote access must add transport/provider-specific abuse controls on top of these local limits rather than replacing them.

## 20. Event bus and live UI

Core modules publish typed domain events to a single in-process EventBus.

```text
Domain Event
   ├─► AuditWriter ─► SQLite
   └─► UiBroadcaster ─► typed Electron IPC ─► React
```

Renderer loads initial state then subscribes to typed events such as:

- `activity:event`
- `approval:changed`
- `process:changed`
- `workspace:changed`
- `agent:status`
- `doctor:result`
- `security:modeChanged`
- `resource:status`
- `tunnel:status`

UI never polls SQLite directly.

## 21. Session and process lifecycle

### 21.1 One app launch = one Agent session

Startup creates one session, one fresh HTTP bearer token and sets security mode to **Guarded** regardless of previous session state.

If previous session remains `RUNNING` without clean shutdown, mark it `CRASHED` and expose to Doctor.

### 21.2 Managed process lifecycle

States include `STARTING`, `RUNNING`, `EXITED`, `START_FAILED`, `ORPHANED`.

Agent `processId` is opaque/UUID primary identity. PID is metadata only; record PID + process start timestamp + command for identity checks.

### 21.3 Shutdown

```text
Agent → SHUTTING_DOWN
↓
reject new executions
↓
cancel pending approvals according to policy
↓
allow foreground work bounded grace
↓
stop Agent-managed background/dev processes
↓
leave external processes untouched
↓
flush audit/state
↓
stop HTTP MCP and named-pipe RPC
↓
invalidate/clear HTTP credential
↓
discard Autonomous state
↓
close SQLite
↓
STOPPED
```

### 21.4 Crash recovery

Do not automatically adopt old process records after restart. If an old managed process cannot be safely re-identified, mark old record `ORPHANED`.

New session always gets:

```text
new HTTP token
security.mode = guarded
```

regardless of clean or unclean previous shutdown.

## 22. MCP transports

### 22.1 Localhost HTTP

Endpoint concept:

```text
http://127.0.0.1:<configured-port>/mcp
```

Requirements:

- loopback only
- explicitly supported IPv4/IPv6 loopback representations only
- no `0.0.0.0`
- no LAN bind setting Phase 1
- bearer authentication every MCP HTTP request
- random per-session token
- no plaintext persistence/logging
- bounded request admission under Section 19
- do not enable permissive browser CORS as convenience
- invalid non-loopback binding rejected/flagged

HTTP port failure may leave Agent Core running with transport `ERROR: PORT_IN_USE`; UI/Doctor show degraded state.

### 22.2 stdio bridge

Do not start second AgentCore.

```text
Codex/MCP Client
    │ stdio
    ▼
mcp-stdio-bridge
    │ framed local RPC
    ▼
Windows Named Pipe
    │
    ▼
Elevated Electron AgentCore
```

Bridge responsibilities:

- expose MCP stdio transport
- forward lifecycle/tool requests
- return responses/events
- maintain client metadata

Bridge must not own shell/filesystem/PolicyEngine/ApprovalManager/SQLite.

If Desktop App is not running: `AGENT_NOT_RUNNING`.

Named pipe ACL should be current-user scoped where practical. This is defense-in-depth, not a hostile-host sandbox.

## 23. Electron Control Center

Electron is control/observability/approval plane, not IDE.

### 23.1 Renderer security

Required:

- `nodeIntegration = false`
- `contextIsolation = true`
- narrow typed preload API
- renderer sandbox where compatible

Never expose generic `invoke(channel,args)`.

Renderer does not directly access `fs`, `child_process`, SQLite, shell/process APIs or MCP internals.

### 23.2 Navigation/pages

- Dashboard
- Projects / Workspaces
- Git
- Activity / Live Logs
- Processes
- Approvals
- Tunnel
- Doctor
- Settings

### 23.3 Dashboard

Show:

- Agent status/uptime
- elevation
- current security mode with prominent `Guarded` or `Autonomous / Unrestricted`
- explicit note that Autonomous resets next session
- raw-shell status
- HTTP endpoint/auth health (never token)
- stdio bridge
- clients
- active processes
- pending approvals
- resource-pressure summary
- recent activity

When an opaque R2 request is waiting, Activity/Dashboard should make `OPAQUE EXECUTION` visible rather than hiding it only in detailed logs.

### 23.4 Projects / Workspaces

Show name, root, type, Git branch/root, package manager, last opened, enabled.

Unregister copy explicitly says project files are not deleted.

A visible security note should explain that workspace registration grants project access but **does not mark repository text/scripts as trusted instructions or sandboxed code**.

### 23.5 Git page

Operational view only:

- branch
- status
- bounded diff
- recent log
- changed files
- related Activity filter/link

Git mutation continues through ToolDispatcher/Policy/Approval. Commit messages/diff content displayed to AI-facing paths are untrusted content.

### 23.6 Activity / Live Logs

Filters:

- time
- workspace
- tool
- client
- risk
- status
- execution transparency/trust

Detail shows timeline, policy, audit-safe args, content provenance, execution trust, output and events.

`tool_calls` remains state source; `audit_events` remains timeline source. “Live Logs/MCP Activity” is visualization only.

### 23.7 Processes

Show Agent process ID, PID, workspace, command/cwd, uptime/state, execution trust, recent output, restart/stop.

Output viewer is bounded text, not terminal emulator.

### 23.8 Approvals

Show:

- requester
- workspace
- tool
- frozen arguments
- risk/reasons
- execution transparency/trust
- expiry
- current security mode
- whether MCP self-approval is permitted
- human approval requirement
- execution result

R3 prominently shows `HUMAN APPROVAL REQUIRED` and references the invariant behavior.

Opaque raw-shell R2 requests show `OPAQUE EXECUTION — HUMAN APPROVAL REQUIRED IN GUARDED MODE`.

### 23.9 Tunnel

Phase 1: status/unavailable/not configured.

Release 1.1 activates remote status/auth/health without exposing credentials.

### 23.10 Doctor

Structured diagnostics return `PASS`, `WARN`, `FAIL` with actionable explanations and no secrets.

Checks at least:

- AgentCore health
- elevation/Admin
- current security mode and whether session reset policy is active
- SQLite open/migrations/WAL
- audit-retention configuration/service health
- HTTP MCP status/auth
- port availability
- named-pipe RPC
- stdio bridge
- workspace accessibility
- Git/version
- Node/npm/pnpm/yarn
- PowerShell/cmd
- ProcessManager
- ResourceGovernor status/pressure
- writable app data/runtime/log dirs
- stale/crashed session
- security warnings including repository-controlled execution boundary
- Tunnel once remote transport exists

Doctor must never print bearer token/secrets.

### 23.11 Settings

At minimum:

- default shell
- command timeout
- shutdown grace
- HTTP enabled/port/auth status
- stdio enabled/status
- approval expiry
- resource-governance limits
- audit retention (default 30 days)
- process output retention (default 7 days)
- read-only security posture summary

Current Autonomous state is controlled from an explicit security-mode action, not persisted as a “remember my last mode” setting.

Security summary includes:

```text
Elevation: Administrator
Shell: Raw / privileged
Default session mode: Guarded
Current session mode: Guarded | Autonomous / Unrestricted
Autonomous persistence: Never; resets every session
Opaque raw shell in Guarded: R2 approval
Repository scripts: Arbitrary-code boundary; not sandboxed
External writes: Approval required
R3 destructive actions: Independent human approval required
HTTP authentication: Per-session bearer token
Approval self-allow: R2 Autonomous only; never R3
```

## 24. Typed IPC boundary

Conceptual preload surface:

```ts
window.agent.dashboard.getStatus()
window.agent.workspaces.list()
window.agent.workspaces.register(...)
window.agent.workspaces.update(...)
window.agent.git.getOverview(...)
window.agent.activity.list(...)
window.agent.activity.subscribe(...)
window.agent.processes.list()
window.agent.processes.stop(...)
window.agent.approvals.list()
window.agent.approvals.decideHuman(...)
window.agent.doctor.run()
window.agent.security.getMode()
window.agent.security.enableAutonomousForSession(...)
window.agent.security.returnToGuarded()
window.agent.auth.copySessionCredential()
window.agent.resources.getStatus()
window.agent.settings.get()
window.agent.settings.update(...)
window.agent.tunnel.getStatus()
```

Renderer UX restrictions are not security authority. AgentCore validates/classifies/audits every mutation.

`enableAutonomousForSession` requires explicit UI confirmation and does not persist across session restart.

## 25. Internal module boundaries

```text
AgentCore
│
├─ McpGateway
│  ├─ HttpTransport
│  ├─ SessionTokenAuth
│  └─ NamedPipeRpcServer
│
├─ ToolDispatcher
├─ ToolRegistry
│
├─ PolicyEngine
│  ├─ PathPolicy
│  ├─ ShellInspector
│  ├─ SecurityModePolicy
│  ├─ ExecutionTrustPolicy
│  └─ ContentProvenancePolicy
│
├─ ApprovalManager
├─ ResourceGovernor
├─ WorkspaceManager
├─ ProjectDetector / ProjectAdapter
├─ ProcessManager
├─ DoctorService
├─ AuditRetentionService
│
├─ EventBus
│  ├─ AuditWriter
│  └─ UiBroadcaster
│
└─ Persistence
   ├─ SQLite adapter
   ├─ migrations
   └─ repositories
```

Boundaries are requirements even if exact source layout differs.

## 26. Runtime schemas

MCP inputs, IPC inputs and domain data crossing boundaries require runtime validation, not TypeScript types only.

Use shared schemas and derive TS types where practical.

Policy results should be able to represent:

```ts
type PolicyDecision = {
  decision: "allow" | "approval_required" | "deny"
  riskLevel: "R0" | "R1" | "R2" | "R3"
  reasons: string[]
  inspectorFlags: string[]
  executionTransparency?: ExecutionTransparency
  executionTrust?: "direct" | "repository_controlled" | "opaque"
  unknownRisk?: boolean
  contentProvenance?: ContentProvenance[]
}
```

## 27. Startup sequence

Required startup order:

```text
Electron starts elevated
↓
single-instance lock
↓
initialize app directories
↓
open SQLite
↓
run migrations
↓
recover/mark previous crashed session
↓
create new Agent session with security.mode=guarded
↓
generate fresh HTTP token
↓
initialize repositories/EventBus
↓
initialize PolicyEngine
↓
initialize ResourceGovernor
↓
initialize ToolRegistry/Dispatcher
↓
start Named Pipe RPC
↓
start authenticated localhost HTTP MCP
↓
initialize Doctor state
↓
renderer ready
↓
Agent RUNNING
↓
run bounded maintenance/retention when appropriate
```

Critical failures in SQLite, PolicyEngine or ToolRegistry prevent RUNNING.

Non-critical transport failure may yield degraded RUNNING state surfaced in Dashboard/Doctor.

Second launch foregrounds existing instance instead of spawning another privileged AgentCore.

## 28. Failure handling requirements

### 28.1 `apply_patch`

- parse/validate full patch before modification
- resolve all targets before writes
- compute edits in memory where practical
- per-file temp + atomic replace where practical
- multi-file atomicity not guaranteed
- partial failure lists changed/failed files
- audit partial failures

### 28.2 Foreground shell timeout

On timeout:

1. mark timeout
2. best-effort terminate Agent-owned process tree
3. capture bounded final output
4. audit timeout
5. return `COMMAND_TIMEOUT`

Do not claim perfect child containment.

### 28.3 Output bounds

No MCP response may dump unbounded stdout/stderr/file/tree/search data.

### 28.4 Resource saturation

When limits are reached:

- reject deterministically with stable error
- never bypass approval/policy because queue is full
- do not silently drop audit events
- UI/Doctor show degraded/pressure state

## 29. Testing strategy

Phase 1 requires:

```text
Static / Typecheck
Unit
Contract
Integration
Security regression
E2E / packaged Windows smoke
```

### 29.1 Unit tests

**PathPolicy**

- inside/outside
- traversal
- case variants
- sibling prefix
- symlink/junction escape
- invalid paths

**ShellInspector / ExecutionTrustPolicy**

```text
npm test via ProjectAdapter      → R1 + repository_controlled
pnpm build via ProjectAdapter    → R1 + repository_controlled
del foo.txt                      → R3
Remove-Item foo                  → R3
git reset --hard                 → R3
git clean -fd                    → R3
taskkill external               → R2/R3 by semantics
reg add                          → R2
unknown-tool foo, Guarded        → R2 + opaque_or_unknown
unknown-tool foo, Autonomous     → R2 + opaque_or_unknown
encoded PowerShell               → >=R2, never silent R1
```

**ApprovalManager**

- R2 Guarded human allow
- R2 Guarded self-allow reject
- R2 Autonomous self-allow
- R3 MCP allow reject
- R3 human allow
- expiry
- shutdown cancel
- concurrent double approval
- replay
- hash mismatch
- exactly-once

**SecurityModePolicy**

- new session always Guarded
- enable Autonomous requires explicit UI path
- restart resets Guarded
- crash recovery resets Guarded
- no MCP tool can enable Autonomous

**ContentProvenancePolicy**

- workspace file → untrusted_content
- Git commit message → untrusted_content
- process output → untrusted_content
- explicit Control Center approval → human_authoritative
- registration does not upgrade trust

**ResourceGovernor**

- per-client concurrency
- global concurrency
- process cap
- approval cap
- request rate/burst
- stable error codes

### 29.2 MCP tool contract tests

Every tool tests:

- schema validation
- policy classification
- success/failure shape
- bounded output
- audit event creation
- content/execution provenance when applicable

### 29.3 Persistence integration tests

Fresh temp SQLite DB covers:

- migrations
- foreign keys
- WAL/busy
- session lifecycle
- session starts Guarded
- previous Autonomous not restored
- tool-call lifecycle
- approval atomic transition
- R3 invariants
- audit append
- process records
- crash recovery
- bearer token absent from persisted data
- retention service excludes active records/user data

### 29.4 Filesystem/Git integration tests

Use temp Windows fixture repos, never real developer repo.

Test:

- read/search/patch
- canonical paths
- Git inspection
- project detection
- package-manager/script detection
- file-delete route creates R3 approval without deleting real user data
- repository content provenance

### 29.5 Project execution integration tests

Use controlled fixture scripts to verify:

- structured `test/lint/typecheck/build/dev` remain R1
- `executionTrust=repository_controlled` recorded
- stdout/stderr provenance
- documented arbitrary-code boundary is visible in Activity/Doctor security summary

Tests must not imply structured project scripts are sandboxed.

### 29.6 Shell/process integration tests

Use deterministic fixtures:

- stdout/stderr
- exit code
- timeout
- background
- incremental output
- stop/restart
- shutdown cleanup
- Guarded unknown command → approval instead of execution
- Autonomous unknown command remains R2 and can follow R2 approval policy

Automated suite must not perform real destructive system commands.

### 29.7 HTTP security suite

- no auth → rejected
- invalid token → rejected
- current token → succeeds
- rotation after restart
- old token fails
- token absent SQLite/audit/log/errors/Doctor
- normal renderer state does not expose token
- explicit copy action only normal UI exposure
- non-loopback bind rejected
- `0.0.0.0` rejected
- request admission/rate limits enforced
- repeated auth failures throttled without secret leakage

### 29.8 Security regression suite

Maintain dedicated tests for:

- path traversal/junction escape
- destructive Git
- PowerShell/cmd aliases
- mixed quoting
- external writes
- approval mutation/replay/double execution
- R2 Guarded self-approval rejection
- R2 Autonomous self-approval audit
- R3 requester self-approval rejection
- R3 MCP allow rejection
- R3 human execution exactly once
- token leakage
- Autonomous reset on restart/crash
- unknown/opaque raw-shell escalation
- repository-controlled execution metadata
- untrusted-content provenance
- audit retention boundaries
- resource admission exhaustion

Every permission/auth/trust-boundary bug requires a regression test before considered fixed.

### 29.9 Electron UI tests

Cover:

- Dashboard/security mode + reset copy
- Projects/Workspaces trust notice
- Git operational page
- Activity filters/Live Logs/provenance/transparency
- process output
- R2 both modes
- R3 human approval
- opaque execution warning
- explicit credential copy
- Tunnel states
- Doctor PASS/WARN/FAIL
- resource pressure
- transport errors
- degraded states

### 29.10 Doctor acceptance tests

**Healthy**

- AgentCore PASS
- elevation PASS
- SQLite/migrations/WAL PASS
- HTTP auth/loopback PASS
- named pipe PASS
- writable dirs PASS
- security mode Guarded at new session PASS
- retention configuration PASS
- ResourceGovernor PASS

**Failure/warnings**

- port conflict → actionable WARN/FAIL
- SQLite failure → FAIL
- non-elevated when required → FAIL
- missing Git/Node → capability-specific diagnostics
- missing pnpm/yarn → contextual WARN
- stale/crashed session → WARN
- insecure bind attempt → FAIL
- Autonomous active → visible security WARN/INFO, not silently treated as normal
- repository-controlled execution boundary → persistent explanatory security notice

### 29.11 Audit retention tests

- default active audit retention = 30 days
- default full process-output retention = 7 days
- active/pending/executing records never pruned
- retention cannot target workspace paths
- manual JSONL export produces bounded valid records
- maintenance run produces retention summary event

### 29.12 Critical E2E flows

**Flow A — Normal coding**

```text
launch
→ verify Guarded
→ obtain current HTTP credential through explicit test/user-equivalent path
→ authenticated connection
→ register fixture workspace
→ snapshot/read/search
→ apply_patch
→ test/lint/typecheck/build
→ verify repository_controlled execution metadata
→ git status/diff/log
→ project_dev/process_output
→ verify Activity/Live Logs + provenance
```

**Flow B — R2 Guarded**

```text
sensitive or opaque raw-shell request
→ APPROVAL_REQUIRED
→ requester MCP allow rejected
→ Control Center human allow
→ exact frozen request executes once
```

**Flow C — R2 Autonomous**

```text
human explicitly enables Autonomous for current session
→ R2 request
→ requester MCP self-approves
→ selfApproved=true + autonomousMode=true
→ restart Agent
→ verify mode returns Guarded
```

**Flow D — R3 destructive**

```text
destructive request
→ APPROVAL_REQUIRED
→ requester MCP allow rejected
→ remains PENDING
→ HUMAN APPROVAL REQUIRED
→ human allows
→ frozen hash matches
→ exact request executes once
→ replay/double approval cannot re-execute
→ audit humanAuthorized=true, selfApproved=false
```

**Flow E — HTTP auth/rotation**

```text
no token → reject
wrong token → reject
current token → success
restart
old token → reject
new token → success
verify no token persisted/logged
```

**Flow F — Doctor**

```text
healthy → PASS
port conflict → actionable WARN/FAIL
SQLite failure → FAIL
missing dependency → actionable diagnostic
resource pressure → visible diagnostic
```

**Flow G — Resource governance**

```text
exceed per-client/global concurrency → stable rejection
hit process cap → PROCESS_LIMIT_REACHED
hit approval cap → APPROVAL_QUEUE_FULL
verify no unauthorized execution and audit remains coherent
```

### 29.13 Release smoke tests

Packaged Windows test verifies:

- actual elevation
- PowerShell/cmd
- Git
- authenticated HTTP MCP
- token rotation
- stdio/named-pipe ACL behavior
- Guarded session default/restart reset
- R3 Control Center approval
- opaque-shell Guarded gate
- Doctor
- resource limits
- shutdown cleanup

## 30. Phase 1 implementation roadmap

### Phase 1.1 — Foundation

Deliver:

- Electron/React/TypeScript shell
- elevated lifecycle
- single instance
- SQLite/migrations
- Agent session lifecycle
- mandatory Guarded session initialization
- typed preload foundation
- EventBus/logger
- basic Dashboard health

Acceptance:

- starts elevated
- `RUNNING`
- SQLite healthy
- fresh session is Guarded
- clean STOPPED shutdown

### Phase 1.2 — Workspace/file core + provenance

Deliver:

- WorkspaceRegistry
- canonical PathPolicy
- ProjectDetector
- ContentProvenance model
- `workspace_*`
- `read_file`, `search_text`, `apply_patch`

Acceptance:

- fixture registration/detection/tree/read/search/patch
- traversal/junction tests
- every call audited
- delete path classified R3
- workspace/Git text is untrusted content, not instruction authority

### Phase 1.3 — Git + Node ProjectAdapter + execution trust

Deliver:

- `git_status`, `git_diff`, `git_log`
- `project_info`, `project_dev`, `test`, `lint`, `typecheck`, `build`
- npm/pnpm/yarn detection
- execution-trust metadata
- Git operational service

Acceptance:

- configured scripts run
- absent script error
- Git bounded/audited
- Git page operational
- project scripts visibly recorded as `repository_controlled`

### Phase 1.4 — Shell + ProcessManager + opaque execution policy

Deliver:

- raw `shell`
- `ExecutionTransparency`
- foreground/background
- `process_*`
- bounded output
- timeout/restart/shutdown cleanup

Acceptance:

- known normal direct command succeeds
- Guarded unknown/opaque raw shell creates R2 approval
- Autonomous unknown/opaque remains R2
- destructive shell fixture R3
- process lifecycle works

### Phase 1.5 — Policy, security modes and approvals

Deliver:

- Risk classifier
- ShellInspector
- SecurityModePolicy
- ExecutionTrustPolicy
- external path classification
- ApprovalManager
- frozen hash/exactly-once
- R2 Guarded/Autonomous
- R3 human-only

Acceptance:

- normal project R1
- external write R2
- Guarded self-allow rejected
- Autonomous R2 self-allow audited
- restart/crash resets Guarded
- R3 requester self-allow rejected
- Control Center human R3 succeeds
- replay cannot re-execute

### Phase 1.6 — Authenticated transports + ResourceGovernor

Deliver:

- 256-bit-equivalent per-session token
- HTTP auth middleware
- thin stdio bridge
- current-user named-pipe ACL
- client auth metadata
- credential-copy action
- resource limits/backpressure

Acceptance:

- HTTP/stdio same policy behavior
- bridge app-not-running behavior
- loopback only
- missing/invalid token rejected
- rotation
- no secret leakage
- concurrency/rate/process/approval limits work

### Phase 1.7 — Operational Control Center + Doctor

Deliver pages:

- Dashboard
- Projects/Workspaces
- Git
- Activity/Live Logs
- Processes
- Approvals
- Tunnel status
- Doctor
- Settings

Acceptance:

User can determine without backend terminal logs:

- Agent action/client/workspace
- command/process/Git state
- risk/approval reason
- execution transparency/trust
- content provenance where relevant
- current session mode and reset semantics
- R2 self-approval eligibility
- R3 human requirement
- HTTP auth health
- dependency/retention/resource health
- Tunnel state

### Phase 1.8 — Hardening, retention and Core release gate

Deliver:

- unit/contract/integration/security/auth suites
- trust-boundary/provenance regressions
- resource-governance suite
- AuditRetentionService + JSONL export
- Electron E2E
- Doctor suite
- packaged elevated smoke

Phase 1 Core completes only when the full Definition of Done below passes on a packaged Windows build.

## 31. Phase 1 Core Definition of Done

Release gate:

```text
1. Launch packaged elevated Control Center
2. Verify new session is Guarded
3. Verify Doctor baseline
4. Verify fresh HTTP credential exists only through explicit secret path
5. No token request → rejected
6. Invalid token → rejected
7. Valid token → connect
8. Register Node/TS fixture repo
9. snapshot/read/search
10. verify content provenance marks repo text as untrusted_content
11. apply_patch inside workspace
12. test/lint/typecheck/build
13. verify executionTrust=repository_controlled
14. git status/diff/log + Git page
15. project_dev/process_output
16. unknown/opaque raw shell in Guarded → R2 approval
17. requester MCP R2 allow rejected
18. Control Center human R2 allow succeeds
19. enable Autonomous explicitly
20. R2 requester self-approval succeeds with visible flags
21. R3 request → requester MCP allow rejected
22. R3 human Control Center approval succeeds
23. verify frozen hash + exactly once
24. replay/double approval cannot execute again
25. verify Activity/Live Logs/Audit timeline
26. exercise resource limits without unauthorized execution
27. verify token absent SQLite/audit/log/Doctor
28. verify retention defaults/config and safe maintenance boundaries
29. close app
30. verify pending approvals cancelled/handled
31. verify managed processes cleaned; external untouched
32. restart app
33. old token fails; new token succeeds
34. security mode is Guarded again (Autonomous did not persist)
35. Doctor reports clean new-session state
```

Secure Remote Access must not begin until this gate is stable and unresolved permission/authentication/trust-boundary/resource bugs are closed.

## 32. Release 1.1 — Secure Remote Access

Secure Remote Access is the immediate milestone after Phase 1 Core because remote operation is central to source product intent, while still deliberately excluded from the Core release gate.

### 32.1 Secure outbound-oriented tunnel

```text
Remote Client
   ↓
Secure Remote Transport / Tunnel
   ↓
McpGateway
   ↓
ToolDispatcher
   ↓
PolicyEngine
   ↓
ApprovalManager
   ↓
ResourceGovernor
   ↓
Execution
   ↓
Audit
```

Requirements:

- reuse existing Dispatcher/Policy/Approval/Process/Audit/ResourceGovernor
- no remote-only policy stack
- explicit remote identity/auth design
- do not reuse/expose local Phase 1 bearer token as remote identity by default
- do not inherit localhost trust assumptions
- R3 remains human-authorized
- requesting remote agent cannot self-approve R3
- Tunnel page shows bounded health/auth state without secrets
- Doctor adds tunnel diagnostics
- add remote-specific request/rate/concurrency abuse controls
- update content provenance for remote responses/sources where applicable

Exact protocol/provider/security implementation of reference prototype is not inferred from screenshots.

### 32.2 Security review gate

Before shipping Release 1.1:

- write a threat-model delta for the remote transport
- review credential lifecycle/storage/revocation
- review remote human approval identity
- update security regression tests
- test rate limiting/backpressure under remote conditions
- verify R3 invariants remain unchanged

### 32.3 Acceptance target

```text
remote authenticated connection
→ workspace_snapshot
→ read_file
→ apply_patch
→ test
→ R2 policy behavior
→ R3 human-authorized approval
→ Activity visible
→ Tunnel/Doctor health visible
```

## 33. Phase 2 — Agentic Development Capabilities

Phase 2 extends development capability after Phase 1 Core and Secure Remote Access. It must not create a second policy/audit/process stack.

### 33.1 `codex_run`

Delegate to Local Codex CLI using existing workspace/process/policy/audit/resource infrastructure.

No separate process manager.

Exact schema is designed during Phase 2 after validated Phase 1 behavior.

### 33.2 Browser CDP

Add `dom_cdp` capabilities:

- navigate
- inspect/query DOM
- click/type
- evaluate JS
- screenshot

Target:

```text
start dev server
→ open Chrome
→ inspect DOM
→ interact
→ verify web UI
```

Browser-returned page text/DOM is untrusted content under `SEC-CONTENT-001`.

### 33.3 Additional project adapters

Add Python/.NET/Rust according to actual priority. All implement `ProjectAdapter` and explicit execution-trust metadata.

### 33.4 Remote/human approval refinement

Remote human approval requires explicit authenticated human identity/authorization. Agent executor credentials alone cannot satisfy R3.

### 33.5 Phase 2 security review gate

Before shipping any Phase 2 capability that adds delegation/browser execution/new code-execution surfaces:

- produce threat-model delta
- identify new credential/content boundaries
- map returned content to provenance
- review script/browser execution trust
- extend resource-governance limits if needed
- add security regression cases
- verify all canonical `SEC-*` invariants remain valid

This gate is mandatory, not optional documentation work.

### Phase 2 Definition of Done

A trusted client can request agentic coding, delegate appropriately, modify/test repo, launch app, validate through Chrome CDP, inspect Git state and preserve activity/provenance/R2/R3 guarantees; Phase 2 threat-model delta and security regressions are complete.

## 34. Phase 3 — Full Windows Desktop Agent

Phase 3 extends execution adapters beyond coding/browser workflows.

### 34.1 Windows UI Automation

Add semantic Microsoft UI Automation first:

- enumerate/find windows/controls
- accessibility tree
- invoke buttons/menus
- get/set control values/text

Prefer semantic UIA actions over coordinate clicks.

### 34.2 Window management

Add list/activate/move/resize/minimize/maximize/close. Closing apps with unsaved state passes policy classification.

### 34.3 Vision + input fallback

```text
UI Automation
↓ if unavailable
Vision
↓
Keyboard/Mouse events
```

Coordinate automation is fallback, not default.

### 34.4 Clipboard + file dialogs

Add bounded/audited clipboard and native Open/Save dialog automation.

### 34.5 Office

Structured Word/Excel COM automation first; do not initially expose arbitrary generic COM invocation.

### 34.6 Screen capture/recording

Add monitor/window/region selection, duration/storage bounds and visible privacy state.

### 34.7 Notifications/scheduler

Windows notifications + Scheduled Task management. Scheduled-task mutation defaults R2 or higher by semantics.

### 34.8 `web_fetch`

Add bounded local HTTP fetching with protocol/timeout/size/credential/local-network policy. Fetched content is untrusted under `SEC-CONTENT-001`.

### 34.9 Audio

Microphone/audio only with explicit visible privacy state and approval rules.

### 34.10 Phase 3 security review gate

Each new OS/UI/Office/vision/network/audio capability requires a threat-model delta covering:

- new input/content provenance
- credential/privacy boundary
- mutation/destructive mapping
- resource limits
- new regression tests
- effect on canonical `SEC-*` invariants

### Phase 3 Definition of Done

Cross-application E2E:

```text
modify project
→ build/run
→ interact with Windows UI
→ open Excel/Word
→ write verification result
→ save
→ capture evidence
→ preserve complete audit/provenance timeline
```

and required security-review deltas/regressions are complete.

## 35. Cross-phase architecture and security rules

Permanent unless later approved design explicitly changes them:

1. Phase 1 is the platform; later phases extend transports/adapters.
2. One `ToolDispatcher` lifecycle for all tools.
3. One `PolicyEngine`/`ApprovalManager` authority.
4. One process-management abstraction where applicable.
5. One audit/event model across phases.
6. One ResourceGovernor admission/backpressure layer across transports where applicable.
7. Renderer remains UI; privileged backend stays outside renderer.
8. Do not implement speculative future features in Phase 1.
9. Do not introduce cross-platform abstraction in Phase 1.
10. Never advertise raw shell/project scripts as sandboxed.
11. `SEC-R3-001..003` remain fixed unless user explicitly redesigns destructive policy.
12. `SEC-MODE-001`: every new local Agent session starts Guarded.
13. `SEC-CONTENT-001`: content/data never becomes policy/approval authority merely because an AI reads it.
14. `SEC-EXEC-001`: known repository command names do not imply safe side effects.
15. Remote mode requires its own security review; local trust assumptions are not inherited.
16. Any milestone introducing a **new transport, delegation mechanism, code-execution path, browser execution surface, OS automation adapter, credential boundary, or external data source** requires a threat-model delta and security-regression update before shipping.
17. Audit retention maintenance is narrowly scoped and never grants deletion authority over user/project data.

## 36. Implementation constraints for Codex

Treat as fixed requirements:

- Windows-only Phase 1
- Electron owns privileged Agent lifecycle
- elevated Administrator runtime
- modular monolith AgentCore
- thin stdio bridge, not second AgentCore
- HTTP loopback + per-session auth token
- token memory-only; rotate session
- multi-workspace registry
- strong structured path resolution
- raw shell is best-effort inspected, not sandboxed
- unknown/opaque raw-shell in Guarded is at least R2
- project scripts can remain R1 via ProjectAdapter but must be marked `repository_controlled`
- workspace/Git/process content is not instruction authority
- R0–R3
- external structured write R2
- user/project deletion R3
- R3 independent human approval; no requester self-approval
- Autonomous only R2 and session-scoped
- every new session Guarded
- SQLite operational source of truth
- application-level append-only audit with dedicated retention exception
- default active audit retention 30 days
- default full process output retention 7 days
- bounded outputs
- bounded request/concurrency/process/approval admission
- Node/TS adapter first
- Doctor first-class Control Center page
- Git/Tunnel operational pages
- no Phase 2/3 implementation in Phase 1
- Phase 1 packaged release gate must pass
- each later attack-surface expansion requires security-review delta

## 37. Decisions intentionally deferred to implementation planning

Behavioral/security policy above is fixed. Implementation-level choices remain deferred:

- exact Electron/Node/package versions
- exact runtime schema library
- exact SQLite driver
- exact test framework/bundler
- exact repository/package-manager layout
- exact UI component library
- exact named-pipe framing format
- exact algorithms/data structures used by ShellInspector pattern matching
- exact remote tunnel provider/protocol and remote identity storage
- exact future remote human-approval identity mechanism
- exact future sandbox/secret-isolation strategy, if one is later added

Any implementation selection must preserve contracts/invariants in this design.

## 38. Final Phase 1 success statement

Phase 1 succeeds when LocalGPT Agent behaves as a local, elevated Windows coding execution platform that an MCP client can use autonomously for normal coding work while:

- structured paths are deterministically bounded by policy
- repository/user content is treated as data, not authority
- repository-controlled scripts are explicitly recognized as arbitrary-code execution boundaries
- unknown/opaque raw shell is gated in Guarded mode
- Autonomous mode is explicit and session-scoped
- destructive R3 actions require independent human approval
- transport secrets rotate and remain out of audit/log state
- runtime resources and historical logs are bounded
- every important action can be reconstructed from Control Center audit/activity

It must be powerful, observable and predictable on a trusted developer workstation. It must not misrepresent raw shell, project scripts, authentication, audit or content-provenance controls as a hardened sandbox or malware-resistant isolation boundary.
