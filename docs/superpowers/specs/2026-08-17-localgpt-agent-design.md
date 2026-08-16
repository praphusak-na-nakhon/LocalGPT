# LocalGPT Agent — Windows Coding Agent Core Design

**Status:** Approved design for implementation planning  
**Date:** 2026-08-17  
**Working product name:** LocalGPT Agent  
**Primary target:** Windows developer workstation  

## 1. Purpose

สร้าง Local Desktop Agent บน Windows ที่เปิดผ่าน Electron Control Center แบบ elevated Administrator แล้วให้ MCP clients สั่งงาน coding workflow บนเครื่องจริงได้ เช่น อ่าน/ค้น/แก้ source code, inspect Git, รัน test/lint/typecheck/build, เปิด dev process, อ่าน stdout/stderr และรัน PowerShell/cmd แบบ raw shell โดยมี policy classification, approval lifecycle, process lifecycle และ audit trail กลางชุดเดียว

Phase 1 ต้องเป็น **Coding Agent Core** เท่านั้น ไม่ใช่ full desktop automation platform และไม่ใช่ hardened sandbox

ระบบต้องรักษา product intent สำคัญว่า normal coding work เช่น create/update/edit ภายใน workspace ทำได้โดยอัตโนมัติในขอบเขตที่เหมาะสม ขณะที่การลบหรือ destructive action ต่อ user/project data ต้อง “ถามก่อน” ผ่าน human-authorized approval ที่ไม่สามารถถูกข้ามด้วย requester self-approval

## 2. Source baseline and design ownership

แนวทางจากเอกสาร/ภาพที่ผู้ใช้ให้มาเป็น baseline ของ capability/product intent ดังนี้:

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

รายละเอียด architecture, schemas, permission levels, approval state machine, SQLite model, HTTP authentication, phase decomposition และ UI/IPC ในเอกสารนี้เป็น design ของ LocalGPT Agent ที่ตกลงร่วมกัน

## 3. Approved product decisions

Phase 1 ล็อก decision ต่อไปนี้แล้ว:

1. **Scope:** Coding Agent Core
2. **Stack:** Electron + React + TypeScript + Node.js
3. **OS:** Windows only
4. **Privilege:** Elevated Administrator / Unrestricted runtime capability
5. **Destructive policy:** Destructive Guard with human-authorized R3 approval
6. **Path policy:** Full create/edit inside registered workspaces; read outside workspace; external writes require approval
7. **Shell:** Raw unrestricted PowerShell/cmd with best-effort destructive inspection
8. **MCP transports:** authenticated localhost HTTP + stdio bridge
9. **HTTP authentication:** cryptographically random per-session bearer token, rotated every Agent session
10. **Lifecycle:** Elevated Desktop App owns Agent lifecycle; closing the app shuts down Agent Core
11. **Project intelligence:** Node.js/TypeScript first; generic shell/file/Git fallback for other stacks
12. **Approval channels:** Electron Control Center + MCP approval API
13. **R2 approval authority:** requester self-approval is allowed only when explicit `Autonomous / Unrestricted` mode is enabled; otherwise human approval is required
14. **R3 approval authority:** requester MCP client cannot self-approve; final allow requires a human-authorized approval channel
15. **Phase 1 human approval channel:** explicit user action in the Electron Control Center; MCP `approval_decide` cannot final-allow R3
16. **Workspace model:** Multi-workspace registry keyed by `workspaceId`
17. **State store:** SQLite
18. **Control Center:** Operational Dashboard, not an IDE; navigation includes Dashboard, Projects/Workspaces, Git, Activity, Processes, Approvals, Tunnel, Doctor, Settings
19. **Remote access:** not part of Phase 1 Core release gate; Secure Remote Access is the immediate milestone after Phase 1 Core
20. **Architecture style:** Modular monolith inside the Electron host, with a thin stdio bridge

## 4. Goals

Phase 1 must prove this end-to-end chain on a packaged Windows app:

```text
MCP Client
   ↓
authenticated localhost HTTP or stdio bridge
   ↓
McpGateway
   ↓
ToolDispatcher
   ↓
PolicyEngine
   ↓
ApprovalManager (when required)
   ↓
Workspace / File / Git / Project / Shell / Process execution
   ↓
Windows
   ↓
Audit/EventBus + SQLite
   ↓
Electron Control Center
```

A successful Phase 1 allows an authenticated MCP client to:

1. register a Node/TypeScript workspace
2. inspect workspace snapshot/tree
3. read and search files
4. patch one or multiple source files
5. run tests, lint, typecheck and build
6. inspect Git diff/status/log
7. launch a managed dev process
8. read incremental process output
9. request an R2 sensitive operation and follow the configured approval policy
10. request an R3 destructive operation and receive a mandatory human-approval requirement
11. verify frozen-request hash and exactly-once execution semantics
12. verify a complete activity/audit timeline
13. run Doctor diagnostics without exposing secrets

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

The Phase 1 UI may include a **Tunnel status page** that clearly reports the remote feature as unavailable/not configured until the post-Core remote milestone. This does not mean secure tunnel transport is implemented in Phase 1.

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
audit + best-effort shell inspection
```

Phase 1 is **not** designed for:

```text
Hostile multi-user machine
Malware resistance
Privilege isolation
Strong shell containment
Tamper-proof forensic logging
```

The HTTP session token materially reduces the risk that an arbitrary local process can accidentally or trivially call the privileged HTTP MCP endpoint. It does not protect against malware or a process that can steal the token from the same user session, process memory, explicit clipboard exposure or another compromised trusted client.

### 6.2 Security boundaries that must remain explicit

- **Loopback binding ≠ authentication.** HTTP must satisfy both loopback-only binding and bearer-token authentication.
- **Authentication ≠ sandbox.** A client that possesses the valid session token can invoke the tools exposed to it; raw shell remains privileged execution.
- **Structured path policy is deterministic application control.** Structured tools must resolve canonical paths and apply workspace policy before execution.
- **Shell inspection is best effort.** PowerShell/cmd can invoke scripts, encoded commands, downloaded executables and child processes whose behavior cannot be statically determined reliably.
- **R3 approval is a human gate.** Delete/destructive actions require an independent human-authorized approval and cannot be silently self-approved by the requesting agent.
- **R2 autonomous approval is mode-gated.** MCP requester self-approval is allowed only when a human has explicitly enabled `Autonomous / Unrestricted` mode in the Control Center.
- **Audit is operational, not forensic-grade.** An Administrator can still modify the local SQLite database outside the application.

## 7. Top-level architecture

```text
┌────────────────────────────────────────────────────────────────────┐
│ Electron Desktop Control Center                                    │
│                                                                    │
│  ┌──────────────────── React Renderer ───────────────────────────┐ │
│  │ Dashboard │ Projects │ Git │ Activity │ Processes            │ │
│  │ Approvals │ Tunnel │ Doctor │ Settings                       │ │
│  └───────────────────────┬────────────────────────────────────────┘ │
│                          │ typed preload IPC                        │
│  ┌───────────────────────▼────────────────────────────────────────┐ │
│  │ Agent Core                                                     │ │
│  │                                                                │ │
│  │ McpGateway                                                     │ │
│  │ ├─ localhost HTTP → SessionTokenAuth                           │ │
│  │ └─ named-pipe RPC target for stdio bridge                      │ │
│  │          │                                                     │ │
│  │          ▼                                                     │ │
│  │ ToolDispatcher → ToolRegistry                                  │ │
│  │          │                                                     │ │
│  │          ▼                                                     │ │
│  │ PolicyEngine                                                   │ │
│  │ ├─ PathPolicy                                                  │ │
│  │ ├─ ShellInspector                                              │ │
│  │ └─ SecurityModePolicy                                          │ │
│  │          │                                                     │ │
│  │          ├─ allow ───────────────► Execution                   │ │
│  │          └─ approval ─► ApprovalManager ─► Execution           │ │
│  │                                     │                          │ │
│  │ Execution Layer                     │                          │ │
│  │ ├─ Workspace/File                    │                          │ │
│  │ ├─ Git                               │                          │ │
│  │ ├─ Project Adapter                   │                          │ │
│  │ ├─ Shell                             │                          │ │
│  │ └─ ProcessManager                    │                          │ │
│  │                                      │                          │ │
│  │ DoctorService ───────────────────────┤                          │ │
│  │                                      ▼                          │ │
│  │                                  EventBus                       │ │
│  │                               ┌──────┴──────┐                   │ │
│  │                               ▼             ▼                   │ │
│  │                            AuditWriter   UiBroadcaster           │ │
│  │                               │                                 │ │
│  │                               ▼                                 │ │
│  │                             SQLite                              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘

MCP client ──stdio──► mcp-stdio-bridge ──Windows named pipe──► Agent Core
```

Future remote transport must connect to the same `McpGateway → ToolDispatcher → PolicyEngine → ApprovalManager → Execution → Audit` pipeline. It must not create a parallel policy/process/audit stack.

### 7.1 Core rules

- MCP transports contain no tool business logic.
- HTTP authentication occurs before an HTTP request reaches tool dispatch.
- Tool implementations do not make their own permission decisions.
- React renderer never touches filesystem, shell, SQLite or MCP server internals directly.
- Every tool invocation follows one lifecycle owned by `ToolDispatcher`.
- Every privileged action is auditable through the same event model.
- Security mode changes are owned by AgentCore, initiated through explicit Control Center action and audited.
- Doctor diagnostics consume health/state services but do not bypass policy or become a second execution layer.

## 8. Standard tool execution lifecycle

Every MCP tool call follows this exact conceptual path:

```text
HTTP only: authenticate session token
        ↓
1. Receive request
2. Assign toolCallId
3. Runtime schema validation
4. Resolve client/session/workspace/path context
5. Policy classification
6. Write policy/audit events
7a. ALLOW → execute
7b. APPROVAL_REQUIRED → freeze request → create approval → return APPROVAL_REQUIRED
7c. DENY → return stable policy error
8. Capture result or failure
9. Persist final state + audit event
10. Map to MCP result
```

Authentication failures are rejected before tool execution and must never create a valid tool call that can execute. They may create bounded security audit events that do not contain the supplied token.

Tool implementations must never send MCP responses directly.

## 9. Permission model

### 9.1 Risk levels

| Level | Meaning | Default behavior |
|---|---|---|
| `R0 READ_ONLY` | No state mutation | Auto allow |
| `R1 NORMAL_WRITE_EXECUTE` | Normal coding write/execute inside workspace | Auto allow |
| `R2 SENSITIVE` | External/system/high-impact mutation | Approval required; MCP requester self-approval only in explicit Autonomous / Unrestricted mode |
| `R3 DESTRUCTIVE` | Likely user/project data or state loss | Human-authorized approval required; requester MCP self-approval prohibited |

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
- package install in workspace
- test/lint/typecheck/build
- dev server start
- `git add`
- `git commit`
- normal non-destructive shell commands

`R2`:

- create/edit outside workspace
- registry/system configuration mutation
- service/firewall/scheduled-task mutation
- system-wide install
- ACL/ownership change
- terminating a process the Agent did not create
- unregistering workspace metadata
- other high-impact actions that are not classified as direct data destruction

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

Agent-owned ephemeral runtime artifacts under the Agent data directory are a special operational class. Bounded retention/rotation/pruning of Agent-generated temporary process logs may occur according to configured retention without user approval. This exception must never apply to source files, registered workspace data, user documents or other user/project data.

### 9.4 Security modes

Phase 1 defines two local operating modes:

**Guarded (default)**

- R0 auto allow
- R1 auto allow
- R2 requires explicit human approval in the Control Center
- R3 requires explicit human approval in the Control Center
- MCP `approval_decide(... allow ...)` cannot final-allow R2 or R3

**Autonomous / Unrestricted (explicit opt-in)**

- R0 auto allow
- R1 auto allow
- R2 may be approved by an authenticated MCP client, including the requester itself
- R2 requester self-approval must set `selfApproved=true` and `autonomousMode=true` in audit/UI
- R3 still requires explicit human-authorized approval and **cannot** be final-approved by the requesting MCP client or silently bypassed

Changing security mode must require an explicit Control Center action, must be audited, and must not be exposed as a normal MCP tool in Phase 1.

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

## 11. Shell policy

### 11.1 Shell capability

`shell` accepts arbitrary PowerShell or cmd commands and may run foreground or background processes with Administrator privileges.

There is no command allowlist in Phase 1.

### 11.2 Inspector

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

Known risky patterns become `R2` or `R3` based on semantics. Unknown commands remain executable under the selected raw-shell capability, with audit flags such as `unknownRisk=true` where appropriate.

The UI and documentation must never claim shell inspection is containment.

When the inspector detects R3 semantics, the request is frozen and routed to the human-only R3 approval path. `Autonomous / Unrestricted` mode does not downgrade or bypass R3.

### 11.3 CWD

If `workspaceId` is supplied and no `cwd` is provided, default to workspace root.

Relative `cwd` resolves from workspace root.

Absolute `cwd` outside the workspace is allowed under raw-shell mode but must be marked in audit context (for example `externalCwd=true`). External write classification remains best effort for raw shell.

### 11.4 Environment variables and redaction

Shell may receive additional environment variables, but audit views must not persist or display sensitive plaintext values by default.

At minimum redact values for key patterns containing:

- `TOKEN`
- `SECRET`
- `PASSWORD`
- `API_KEY`
- `AUTH`
- `PRIVATE_KEY`

Store key names and redacted values for operational visibility.

The per-session HTTP MCP token is a transport secret and must never be included in shell audit arguments, normal operational logs or generic environment dumps.

## 12. Approval model

### 12.1 Approval guarantee

The following guarantee is a fixed Phase 1 product/security requirement:

> **Delete/destructive actions require an independent human-authorized approval and cannot be silently self-approved by the requesting agent.**

For Phase 1, the human-authorized final-approval channel is an explicit user action in the Electron Control Center. Future remote milestones may add authenticated human approval channels, but they must preserve the same R3 guarantee.

### 12.2 Approval semantics

The original tool request does not stay blocked on an open MCP connection.

Flow:

```text
1. Client invokes risky tool.
2. Policy returns APPROVAL_REQUIRED.
3. Agent freezes exact tool name + arguments and creates approvalRequestId.
4. Original call returns APPROVAL_REQUIRED + approvalRequestId.
5. A decision arrives through Control Center or MCP approval API.
6. ApprovalManager validates risk-specific approver rules.
7. If final ALLOW is authorized, Agent validates frozen request hash.
8. Agent atomically transitions the request and executes that exact request once.
9. Decision call / UI receives the execution outcome.
```

### 12.3 Risk-specific approval authority

**R2 — SENSITIVE**

- Guarded mode: final allow requires explicit human approval in Control Center.
- Autonomous / Unrestricted mode: an authenticated MCP client may final-allow R2.
- If `requesterClientId === approverClientId` for an R2 allow, this is permitted only in Autonomous / Unrestricted mode and must be visibly audited as self-approved.

**R3 — DESTRUCTIVE**

- final allow requires `humanAuthorized=true`
- in Phase 1, `humanAuthorized=true` can be produced only by explicit Control Center user action
- MCP `approval_decide` with `decision="allow"` for R3 must return `HUMAN_APPROVAL_REQUIRED`
- requester MCP client cannot be the final approver for its own R3 request
- changing to Autonomous / Unrestricted mode does not alter this rule

Any actor may submit a deny decision if otherwise authorized to access the approval API; deny never executes the frozen request.

### 12.4 State machine

```text
PENDING
   ├─ deny ─────────────► DENIED
   ├─ expire ───────────► EXPIRED
   └─ authorized allow
          ▼
       EXECUTING
          ├─ success ───► COMPLETED
          └─ failure ───► FAILED
```

An unauthorized allow attempt does not transition the request out of `PENDING`; it returns a stable policy error and may create an audit event.

Terminal approval states are not revivable. A new execution attempt requires a new tool call and approval request.

### 12.5 Exactly-once protection

The Agent must prevent:

- double approval causing double execution
- replay of completed approvals
- mutation of command/arguments after approval
- execution of expired/denied approvals
- unauthorized R3 MCP allow attempts becoming executable state

Before enqueueing approval, canonicalize and hash the frozen tool request. Before execution, the hash must still match.

Approval decision must use an atomic database transition, so concurrent UI/MCP decisions cannot both execute the request.

### 12.6 Self-approval audit

`selfApproved=true` is valid only for R2 in explicit Autonomous / Unrestricted mode.

For R3, `selfApproved=true` must never be present on a successfully executed approval. A requester self-approval attempt must be rejected and audited without execution.

### 12.7 Expiration

Default approval expiration: **15 minutes**.

Settings may support 5, 15, 30 or 60 minutes.

## 13. Workspace model

Use a multi-workspace registry keyed by opaque `workspaceId`.

Conceptual record:

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

All structured file/Git/project tools receive `workspaceId`; they do not accept unrestricted absolute project roots as their main addressing model.

`workspace_snapshot` is a logical project snapshot (tree summary, project metadata, Git summary) and **not** a backup of file contents.

## 14. MCP tool catalog — Phase 1

The exact runtime schemas must be defined once in shared runtime-validatable schemas and reused for MCP, IPC and internal types where appropriate.

### 14.1 Workspace tools

#### `workspace_list`
Lists registered workspaces.

- Policy: `R0`
- Input: optional filters
- Output: workspace summaries

#### `workspace_get`
Returns full workspace metadata.

- Policy: `R0`
- Input: `workspaceId`

#### `workspace_register`
Adds a local project root to the registry after canonicalization and project detection.

- Policy: `R1/CONTROL`
- Input: `rootPath`, optional `name`
- Does not copy project files

#### `workspace_update`
Updates mutable workspace metadata/settings.

- Policy: `R1/CONTROL`

#### `workspace_unregister`
Removes registry metadata only; does not delete project files.

- Policy: `R2`, approval required according to security mode

#### `workspace_tree`
Returns a bounded directory tree.

- Policy: `R0`
- Input: `workspaceId`, optional relative path, depth, limits

#### `workspace_snapshot`
Returns a bounded logical project snapshot.

- Policy: `R0`

### 14.2 File/search tools

#### `read_file`
Reads a text file with bounded byte/line output.

- Policy: `R0`
- Input: `workspaceId`, `path`, optional line/byte bounds
- Binary input returns metadata/unsupported result rather than dumping binary data

#### `search_text`
Searches project text using implementation-selected search backend (for example ripgrep) without exposing backend-specific semantics as the MCP contract.

- Policy: `R0`
- Input: query, workspace, optional include/exclude/glob/case/limit
- Output: path, line, matched context

#### `apply_patch`
Creates or modifies text files, including multi-file patches.

- Normal content edits inside workspace: `R1`
- External target: `R2`
- File deletion or destructive truncation: `R3`, human approval required

Requirements:

- validate all targets before modification
- compute edits before final writes where practical
- use temporary files + atomic replace per file where practical
- report partial failure explicitly
- ordinary source-line deletions are normal edits; deleting/truncating the file itself is destructive

No generic `write_file` tool is required in Phase 1.

### 14.3 Git inspection tools

#### `git_status`
- Policy: `R0`

#### `git_diff`
- Policy: `R0`
- Supports staged/unstaged and optional path filtering

#### `git_log`
- Policy: `R0`
- Supports bounded history/range

No dedicated `git_commit`, `git_push` or `git_reset` tool is required in Phase 1. Those remain available through raw shell and therefore pass `ShellInspector` and the same R2/R3 approval policy.

### 14.4 Project tools

#### `project_info`
Detects Node/TypeScript project metadata, package manager, scripts, Git root and project capabilities.

- Policy: `R0`

#### `project_dev`
Starts the detected dev script as an Agent-managed background process.

- Policy: `R1`
- Returns Agent `processId`, Windows PID, command, cwd and detected local URL when reliably available

#### `test`
#### `lint`
#### `typecheck`
#### `build`

- Policy: `R1`
- Use detected package manager/script only
- If script is absent, return `PROJECT_SCRIPT_NOT_FOUND`; do not invent a command
- Return actual command, exit code, duration, stdout/stderr summaries and truncation flags

For non-Node repositories, file/Git/raw shell remain usable, but no stack-specific project adapter is promised in Phase 1.

### 14.5 Shell tool

#### `shell`
Runs arbitrary PowerShell or cmd commands.

Conceptual input:

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

Policy: classified dynamically by `ShellInspector`; R3 results always route to human-authorized approval.

### 14.6 Process tools

#### `process_list`
Lists Agent-managed processes by default.

#### `process_get`
Returns process metadata/state.

#### `process_output`
Returns incremental bounded stdout/stderr using cursor/offset semantics.

#### `process_stop`
- Agent-managed process: auto allow
- External process: normally `R2`; destructive semantics may elevate to `R3`

#### `process_restart`
Restarts an Agent-managed process using frozen launch metadata.

Agent `processId` is the primary identity; Windows PID is metadata only.

### 14.7 Approval tools

#### `approval_list`
Lists pending/recent approvals.

#### `approval_get`
Returns frozen request, risk reasons, requester, expiry, whether MCP self-approval is permitted, whether human approval is required and decision state.

#### `approval_decide`
Input:

```ts
{
  approvalRequestId: string
  decision: "allow" | "deny"
  note?: string
}
```

Policy:

- `deny`: does not execute and transitions pending request to `DENIED` when valid
- `allow` on R2: succeeds only if current security mode/approval source allows it
- requester self-allow on R2: allowed only in Autonomous / Unrestricted mode and audited
- `allow` on R3 from MCP: rejected with `HUMAN_APPROVAL_REQUIRED`
- Control Center human approval invokes the same `ApprovalManager` service, validates the same frozen hash and executes exactly once

### 14.8 Operational tools

#### `health`
Returns Agent version/uptime plus health of SQLite, transports and process manager. It is a bounded machine-readable summary, not a replacement for the richer Doctor UI.

#### `system_info`
Returns bounded coding-relevant Windows/system info such as OS version, architecture, CPU/RAM summary, disk summary and Node/Git availability.

Doctor is a first-class Control Center service/page, not required to be exposed as a broad privileged MCP tool in Phase 1.

## 15. Result and error contracts

Use a stable internal result envelope and stable error codes. MCP mapping may adapt this envelope without forcing clients to parse English error text.

Conceptual result:

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

Required stable error families:

**Authentication/transport**

- `AUTH_REQUIRED`
- `AUTH_INVALID`
- `HTTP_BIND_NOT_LOOPBACK`
- `PORT_IN_USE`
- `AGENT_NOT_RUNNING`

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

Authentication error responses must never echo the supplied bearer token.

## 16. SQLite persistence

### 16.1 Storage location

Use a local per-user application data directory such as:

```text
%LOCALAPPDATA%\LocalGPT Agent\
├─ agent.db
├─ logs\
├─ exports\
└─ runtime\
```

The exact path should be derived through the Windows/Electron application-data APIs rather than hard-coded string concatenation.

Source files stay in their real workspace; source code is not copied into SQLite.

The plaintext HTTP session token is **memory-only** and must not be persisted in SQLite.

### 16.2 SQLite requirements

- WAL mode
- foreign keys enabled
- bounded busy timeout
- schema migrations from first release
- renderer never opens SQLite directly
- repositories own persistence access

Required logical repositories:

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

The session may record authentication method/state but must never store the plaintext bearer token. A non-reversible diagnostic identifier/fingerprint may be added only if there is a demonstrated implementation need; it is not required by this design.

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

Transport values include:

- `stdio`
- `http`
- `internal_ui`

For HTTP, `authenticated=1` means possession of the current session bearer token was verified. This authenticates access to the local Agent session; it does not create a full user identity or prove a human identity.

For stdio/named pipe, trust is transport-specific and relies on the local process/user boundary plus named-pipe ACL design.

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

Audit-safe arguments may be redacted. Approval integrity hashes must be derived from the canonical frozen request, not from a display-redacted representation.

Transport authentication headers/tokens must never be stored in `arguments_json`.

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

`approver_kind` examples:

- `HUMAN_UI`
- `MCP_CLIENT`

`approval_source` examples:

- `control_center`
- `mcp`

Invariant requirements:

- successful R3 execution requires `human_authorized=1`
- successful R3 execution must have `approver_kind=HUMAN_UI` in Phase 1
- successful R3 execution must never have `self_approved=1`
- R2 `self_approved=1` requires `autonomous_mode=1`

Statuses:

- `PENDING`
- `DENIED`
- `EXPIRED`
- `EXECUTING`
- `COMPLETED`
- `FAILED`

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
status TEXT
started_at DATETIME
exited_at DATETIME NULL
exit_code INTEGER NULL
restart_count INTEGER
created_by_tool_call_id TEXT
```

`owner_type`:

- `AGENT_MANAGED`
- `EXTERNAL`

### 16.9 Process output storage

Do not keep unbounded stdout/stderr in one SQLite row.

Store full bounded streams in runtime files, for example:

```text
runtime\processes\<processId>\stdout.log
runtime\processes\<processId>\stderr.log
```

Use SQLite metadata/chunks for indexing and recent previews:

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

Default MCP response cap for foreground stdout and stderr: **256 KB per stream**. Larger output returns truncation metadata and remains accessible through `process_output` while retained.

### 16.10 `audit_events`

Append-only at application level:

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

Minimum event vocabulary:

- `SESSION_STARTED`
- `CLIENT_CONNECTED`
- `HTTP_AUTH_REJECTED`
- `SECURITY_MODE_CHANGED`
- `TOOL_REQUEST_RECEIVED`
- `POLICY_DECIDED`
- `APPROVAL_CREATED`
- `APPROVAL_ALLOW_REJECTED`
- `APPROVAL_APPROVED`
- `APPROVAL_DENIED`
- `APPROVAL_EXPIRED`
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
- `AGENT_SHUTDOWN_STARTED`
- `AGENT_SHUTDOWN_COMPLETED`

`HTTP_AUTH_REJECTED` must never contain the submitted token or authorization header.

`tool_calls` represents current/final invocation state; `audit_events` reconstructs the timeline. The Activity UI may render this as Live Logs / MCP Activity without changing the source-of-truth model.

The audit log is operational and inspectable, not a tamper-proof forensic ledger. An Administrator can still modify the local database outside the application.

### 16.11 Settings

Logical `settings` values include:

- `mcp.http.enabled`
- `mcp.http.port`
- `mcp.http.authMode = session_bearer`
- `mcp.stdio.enabled`
- `shell.default`
- `shell.defaultTimeoutMs`
- `approval.expirationMinutes`
- `security.mode = guarded | autonomous_unrestricted`
- `process.shutdownGraceMs`
- process-output limits/retention
- audit display/retention policy

The HTTP token itself is not a setting and must not be persisted.

Changing `security.mode` must be an explicit Control Center action with confirmation and audit. Phase 1 MCP tools must not provide an operation that enables Autonomous / Unrestricted mode.

Default shutdown grace: **5 seconds** before best-effort force termination of Agent-managed processes.

## 17. HTTP session authentication

### 17.1 Token generation and lifetime

At every new Agent session:

1. generate a cryptographically random session secret using a platform/runtime CSPRNG
2. keep the plaintext token only in AgentCore memory
3. use it as the bearer credential for localhost HTTP MCP
4. rotate it by generating a new token on every new Agent session/restart
5. clear references to the old token during shutdown as best effort

The token must have sufficient entropy for local bearer authentication; implementation planning should use a minimum equivalent to 256 random bits unless the selected runtime/library imposes a stronger standard.

### 17.2 HTTP authentication contract

HTTP clients send:

```text
Authorization: Bearer <session-token>
```

Requests with:

- no bearer token → `AUTH_REQUIRED`
- malformed/incorrect token → `AUTH_INVALID`
- valid current-session token → proceed to MCP request handling
- previous-session token after restart → `AUTH_INVALID`

Token comparisons should use a timing-safe strategy where practical.

### 17.3 Secret exposure rules

The token must not appear in:

- SQLite plaintext fields
- audit payloads
- normal operational logs
- error messages
- Doctor output
- Dashboard normal status payloads
- process environment dumps

The Control Center may provide an explicit **Copy session credential** action. The action must be intentional, clearly labeled as sensitive and handled through privileged main/preload IPC. Normal renderer state must not continuously receive the secret.

Copying to clipboard is an explicit user exposure action and should warn that clipboard contents may be readable by other applications.

## 18. Event bus and live UI

Core modules publish typed domain events to a single in-process EventBus.

At minimum:

```text
Domain Event
   ├─► AuditWriter ─► SQLite
   └─► UiBroadcaster ─► typed Electron IPC ─► React
```

Renderer pages load initial state through query APIs and then subscribe to live typed events such as:

- `activity:event`
- `approval:changed`
- `process:changed`
- `workspace:changed`
- `agent:status`
- `doctor:result`
- `security:modeChanged`
- `tunnel:status`

The UI must not poll SQLite directly.

## 19. Session and process lifecycle

### 19.1 One app launch = one Agent session

Startup creates one session record and one fresh HTTP bearer token. Closing the app closes that session and invalidates the in-memory token.

If a previous session remains `RUNNING` without a clean shutdown marker, mark it `CRASHED` on next startup and expose the condition to Doctor diagnostics.

### 19.2 Managed process lifecycle

States include:

- `STARTING`
- `RUNNING`
- `EXITED`
- `START_FAILED`
- `ORPHANED`

Agent `processId` is a UUID/opaque ID. Never use PID as the stable external identity because Windows may reuse PIDs.

Record PID + process start timestamp + command for identity checks.

### 19.3 Shutdown

When Electron exits:

```text
Agent → SHUTTING_DOWN
↓
reject new executions
↓
cancel pending approvals according to policy
↓
allow current foreground work a bounded grace period
↓
stop Agent-managed background/dev processes
↓
leave external processes untouched
↓
flush audit/state
↓
stop HTTP MCP and named-pipe RPC
↓
invalidate/clear session HTTP credential in memory
↓
close SQLite
↓
STOPPED
```

### 19.4 Crash recovery

Do not automatically adopt old process records after restart. If a previous managed process cannot be safely re-identified, mark the old record `ORPHANED`.

A new session must always receive a new HTTP credential even after an unclean previous shutdown.

## 20. MCP transports

### 20.1 Localhost HTTP

Endpoint concept:

```text
http://127.0.0.1:<configured-port>/mcp
```

Requirements:

- bind loopback only
- accept IPv4/IPv6 loopback representations only as explicitly supported by implementation
- no `0.0.0.0`
- no LAN binding setting in Phase 1
- bearer authentication required for every MCP HTTP request
- cryptographically random per-session token
- no plaintext token persistence
- no token in normal logs/audit/error output
- do not enable permissive browser CORS as a convenience feature
- reject/flag invalid non-loopback binding configuration

If token generation/auth initialization fails, HTTP MCP must not enter a healthy/running state.

If HTTP port startup fails, Agent Core may remain running with transport status `ERROR: PORT_IN_USE`; Dashboard and Doctor must expose the partial failure.

Loopback binding and HTTP authentication are separate controls and both are required.

### 20.2 stdio bridge

Do not start a second AgentCore for stdio.

Architecture:

```text
Codex/MCP Client
    │ stdio
    ▼
mcp-stdio-bridge
    │ local framed RPC
    ▼
Windows Named Pipe
    │
    ▼
Elevated Electron AgentCore
```

Bridge responsibilities:

- expose MCP stdio transport
- forward MCP lifecycle/tool requests to AgentCore
- return responses/events
- maintain client connection metadata

Bridge must not own:

- shell execution
- filesystem execution
- PolicyEngine
- ApprovalManager
- SQLite
- HTTP session token

If the Desktop App is not running, return stable `AGENT_NOT_RUNNING` behavior.

Named pipe ACL must be scoped to the current Windows user as far as the selected API permits. This is a transport trust boundary and defense-in-depth; it does not make raw shell sandboxed or make the host malware-resistant.

## 21. Electron Control Center

The Electron application is a control/observability/approval/diagnostics plane, not an IDE.

### 21.1 Electron renderer security

Required configuration:

- `nodeIntegration = false`
- `contextIsolation = true`
- renderer receives only a narrow typed preload API
- enable renderer sandboxing where compatible with the chosen Electron architecture

Never expose a generic `invoke(channel, args)` bridge to renderer code.

Renderer must not directly access:

- `fs`
- `child_process`
- SQLite
- shell/process APIs
- MCP server internals
- HTTP session token through normal state queries

### 21.2 Navigation / information architecture

Control Center navigation must cover:

1. Dashboard
2. Projects / Workspaces
3. Git
4. Activity
5. Processes
6. Approvals
7. Tunnel
8. Doctor
9. Settings

### 21.3 Dashboard

Must visibly show:

- Agent status and uptime
- Administrator/elevation state
- current security mode (`Guarded` or `Autonomous / Unrestricted`)
- raw-shell capability status
- HTTP endpoint/status
- `HTTP AUTH: SESSION TOKEN`
- stdio bridge status
- connected clients
- active processes
- pending R2/R3 approvals
- Doctor overall health
- recent activity
- Tunnel state (`Not available in Phase 1 Core`, `Disconnected`, `Connected`, etc. depending milestone)

The Dashboard must never display the session token by default.

### 21.4 Projects / Workspaces

Show:

- name
- root path
- project type
- Git branch/root
- package manager
- last opened
- enabled state

Actions:

- add
- edit metadata
- refresh detection
- open folder
- view activity
- unregister

Unregister copy must explicitly say project files are not deleted.

### 21.5 Git page

Git is an operational page, not a code editor.

It should show for a selected workspace:

- current branch
- Git status
- changed files
- bounded staged/unstaged diff
- recent bounded log
- repository/root metadata
- links/filters into related Activity events

Git mutations still go through the same tools/shell, PolicyEngine and ApprovalManager. Destructive Git operations remain R3 human-approved actions.

### 21.6 Live Activity / Live Logs

Activity remains backed by structured audit data:

- `tool_calls` = current/final invocation state
- `audit_events` = ordered timeline source of truth

UI may render this as:

- structured Activity table
- Live Logs stream
- MCP Activity view
- per-process output links

Filters:

- time
- workspace
- tool
- client
- risk
- status
- transport

Tool detail view must show timeline, policy decision, exact/audit-safe arguments, output and event history without exposing secrets.

### 21.7 Processes

Show Agent-managed process cards with:

- Agent process ID
- PID
- workspace
- command/cwd
- uptime/state
- recent output
- restart/stop actions

Process output viewer is bounded streaming text, not a full terminal emulator.

### 21.8 Approvals

Pending approvals must display:

- requester client
- workspace
- tool
- frozen command/arguments
- request hash or safe fingerprint representation
- risk level/reasons
- expiry
- current security mode
- whether MCP self-approval is permitted
- whether human approval is required
- decision source/approver kind
- execution result

R3 must be visually distinct and state clearly:

```text
HUMAN APPROVAL REQUIRED
Requester MCP self-approval is not permitted.
```

For R2 in Autonomous / Unrestricted mode, a self-approved request must show visible `SELF-APPROVED` and `AUTONOMOUS MODE` flags.

### 21.9 Tunnel

Phase 1 Core includes the information-architecture page/status only. Before the remote milestone it must report that secure remote transport is not active and must not imply remote connectivity exists.

After the remote milestone, the page should show only bounded operational state such as:

- tunnel enabled/disabled
- connecting/connected/error
- remote identity/auth status summary
- last connection/error time
- no secret material

The Tunnel UI never gets a separate execution/policy stack.

### 21.10 Doctor

Doctor is a first-class diagnostics page. It is not an IDE and must not expose secrets.

Minimum checks:

- AgentCore health
- elevation/Admin status
- SQLite open status
- migration state
- WAL mode
- HTTP MCP status
- HTTP authentication status
- configured HTTP port availability / conflict
- named-pipe RPC status
- stdio bridge availability
- registered workspace accessibility
- Git availability/version
- Node availability/version
- npm/pnpm/yarn availability/version when installed
- PowerShell availability/version
- cmd availability
- ProcessManager health
- writable app-data/runtime/log directories
- stale/crashed previous session state
- current security configuration warnings
- Tunnel status after remote transport is introduced

Structured diagnostic result:

```ts
type DoctorCheckResult = {
  id: string
  status: "PASS" | "WARN" | "FAIL"
  title: string
  summary: string
  remediation?: string
  metadata?: Record<string, unknown>
}
```

`metadata` must be bounded/redacted and must not contain bearer tokens, auth headers, secret environment values or other credentials.

Doctor overall status is the highest active severity required by policy. Critical inability to open SQLite or initialize required AgentCore components is `FAIL`; optional tool absence such as pnpm on a machine that uses npm is generally `WARN`/informational rather than automatically failing the entire Agent.

### 21.11 Settings

At minimum:

- default shell
- default command timeout
- shutdown grace
- HTTP enabled/port
- HTTP authentication status (`session_bearer`)
- explicit **Copy session credential** action
- stdio enabled/status
- approval expiry
- security mode (`Guarded` / `Autonomous / Unrestricted`)
- process-output limits
- audit/log retention controls
- read-only security posture summary

Security mode change requirements:

- user-initiated from Control Center only in Phase 1
- explicit confirmation when enabling Autonomous / Unrestricted
- audit event required
- does not weaken R3 human approval

Security summary must include:

```text
Elevation: Administrator
Shell: Unrestricted capability
HTTP binding: Loopback only
HTTP authentication: Per-session bearer token
External writes: R2 approval
R2 MCP self-approval: Only in Autonomous / Unrestricted mode
R3 destructive actions: Human approval required
R3 requester self-approval: Prohibited
```

## 22. Typed IPC boundary

Preload exposes narrow named APIs conceptually like:

```ts
window.agent.dashboard.getStatus()

window.agent.workspaces.list()
window.agent.workspaces.register(...)
window.agent.workspaces.update(...)

window.agent.git.getStatus(...)
window.agent.git.getDiff(...)
window.agent.git.getLog(...)

window.agent.activity.list(...)
window.agent.activity.subscribe(...)

window.agent.processes.list()
window.agent.processes.stop(...)

window.agent.approvals.list()
window.agent.approvals.decideAsHuman(...)

window.agent.tunnel.getStatus()

window.agent.doctor.run()
window.agent.doctor.getLatest()
window.agent.doctor.subscribe(...)

window.agent.settings.get()
window.agent.settings.update(...)
window.agent.security.getMode()
window.agent.security.setModeWithConfirmation(...)

window.agent.http.copySessionCredential()
```

`copySessionCredential()` should perform the sensitive copy through main-process functionality rather than returning the token in ordinary Dashboard/state payloads.

Renderer UX restrictions are not security decisions. AgentCore still validates, classifies and audits every mutation reached through IPC.

## 23. Internal module boundaries

Logical structure:

```text
AgentCore
│
├─ McpGateway
│  ├─ HttpTransport
│  │  └─ SessionTokenAuth
│  └─ NamedPipeRpcServer
│
├─ ToolDispatcher
├─ ToolRegistry
│
├─ PolicyEngine
│  ├─ PathPolicy
│  ├─ ShellInspector
│  └─ SecurityModePolicy
│
├─ ApprovalManager
├─ WorkspaceManager
├─ GitService
├─ ProjectDetector / ProjectAdapter
├─ ProcessManager
├─ DoctorService
├─ TunnelStatusService
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

`TunnelStatusService` in Phase 1 Core may only expose `NOT_AVAILABLE`/planned state; secure tunnel transport implementation belongs to the post-Core remote milestone.

Repository/project source layout may be selected in the implementation plan, but boundaries above are requirements. The implementation must preserve one-way dependencies so renderer/preload do not become backend execution layers.

## 24. Runtime schemas

MCP inputs, IPC inputs and domain data crossing module boundaries require runtime validation, not TypeScript compile-time types only.

Use one shared schema source per contract and derive TypeScript types from it where practical. The exact schema library and SQLite driver are implementation choices, but they must be isolated so contracts do not depend on a specific transport or persistence driver.

Authentication headers/tokens are transport concerns and must not be copied into generic tool schemas.

## 25. Startup sequence

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
create new Agent session
↓
generate fresh cryptographically random HTTP session token in memory
↓
initialize repositories/EventBus
↓
initialize PolicyEngine + SecurityModePolicy
↓
initialize ApprovalManager
↓
initialize ToolRegistry/Dispatcher
↓
initialize ProcessManager / DoctorService
↓
start Named Pipe RPC with current-user ACL
↓
start localhost HTTP MCP with SessionTokenAuth
↓
renderer ready
↓
run initial Doctor diagnostics
↓
Agent RUNNING
```

Critical failures in SQLite, PolicyEngine, ApprovalManager or ToolRegistry prevent `RUNNING` state.

HTTP token generation failure prevents HTTP transport from becoming healthy. Depending on implementation policy, AgentCore may remain partially operational over local IPC while Dashboard/Doctor clearly report HTTP failure.

Non-critical transport/tool availability failures may yield partial-running status surfaced in Dashboard and Doctor.

Use a single-instance lock so a second launch foregrounds the existing Control Center instead of spawning a second privileged AgentCore/SQLite/process manager or a second session credential.

## 26. Failure handling requirements

### 26.1 `apply_patch`

- parse/validate the full patch before modifying files
- resolve all targets before write
- compute edits in memory where practical
- use per-file temp + atomic replace where practical
- multi-file atomicity is not guaranteed
- partial failure must list which files changed and which failed
- audit partial failures explicitly
- file deletion/destructive truncation must not execute before valid R3 human approval

### 26.2 Foreground shell timeout

On timeout:

1. mark timeout state
2. best-effort terminate Agent-owned process tree
3. capture final bounded output
4. audit timeout
5. return `COMMAND_TIMEOUT`

Do not claim child-process containment is perfect.

### 26.3 Output bounding

No MCP response may dump unbounded stdout/stderr, file content, tree data, Git diff or search results. Every potentially large tool requires explicit bounds/truncation metadata.

### 26.4 Authentication failure handling

Authentication failures:

- must execute no tool
- must not create approval requests
- must not reveal whether a specific token prefix/length was close to correct beyond generic validation needs
- may create a bounded `HTTP_AUTH_REJECTED` event without secret values

## 27. Testing strategy

Phase 1 requires five test layers:

```text
Static / Typecheck
Unit
Contract
Integration
Security Regression
E2E / packaged Windows smoke
```

### 27.1 Unit tests

High-priority modules:

**PathPolicy**

- inside workspace
- outside workspace
- traversal
- case variants
- sibling-prefix confusion
- symlink/junction escape
- invalid paths

**ShellInspector**

Examples:

```text
npm test              → R1
pnpm build            → R1
del foo.txt            → R3
Remove-Item foo        → R3
git reset --hard       → R3
git clean -fd          → R3
taskkill               → R2/R3 by semantics
reg add                → R2
unknown-tool foo       → allowed raw-shell path + audit flag
```

**ApprovalManager / SecurityModePolicy**

- R2 Guarded → human required
- R2 Autonomous → MCP allow permitted
- R2 requester self-approval in Autonomous → succeeds + visible audit flag
- R2 requester self-approval in Guarded → rejected
- R3 requester MCP self-approval → rejected
- R3 non-requester MCP allow → still rejected in Phase 1
- R3 Control Center human allow → succeeds
- expiry
- concurrent double approval
- replay
- hash mismatch
- exactly-once execution
- Autonomous mode never downgrades R3

**SessionTokenAuth**

- token generation uses CSPRNG abstraction
- no token → reject
- invalid token → reject
- valid token → allow transport request
- previous-session token after rotation → reject
- logging/redaction helpers never emit token

**DoctorService**

- PASS/WARN/FAIL aggregation
- bounded/redacted metadata
- each required diagnostic maps to actionable result

### 27.2 MCP tool contract tests

Every tool must test:

- schema validation
- policy classification
- success shape
- known failure shape
- audit event creation
- risk-specific approval behavior where applicable

HTTP MCP contract tests must also validate authentication before tool dispatch.

### 27.3 Persistence integration tests

Use a fresh temporary SQLite DB per test group to cover:

- migrations
- foreign keys
- WAL/busy behavior
- session lifecycle
- security mode persistence for the session/config as designed
- tool-call lifecycle
- approval atomic transition
- R3 human-authorized invariant
- audit append
- process records
- crash recovery
- absence of bearer-token plaintext in persisted data

### 27.4 Real filesystem/Git integration tests

Use Windows temporary fixture repositories, never the developer's real repository.

Test:

- read/search/patch
- canonical paths
- Git inspection
- project detection
- package-manager/script detection
- R3 file-delete path requires human approval fixture path, without deleting real user data

### 27.5 Shell/process integration tests

Use deterministic fixture commands/processes to test:

- stdout
- stderr
- exit code
- timeout
- background process
- incremental output
- stop
- restart
- shutdown cleanup

Automated tests must not perform real system-destructive commands.

### 27.6 HTTP security regression/integration suite

Required cases:

- request without auth → rejected
- request with invalid token → rejected
- request with valid current token → succeeds
- token rotates after Agent restart/new session
- previous token fails after restart
- token is absent from SQLite
- token is absent from audit payloads
- token is absent from normal logs/error messages
- normal Dashboard/Doctor state does not expose token
- explicit copy action is the only normal UI credential exposure path
- non-loopback bind configuration is rejected
- `0.0.0.0` is rejected

### 27.7 Security regression suite

Maintain dedicated tests for:

- path traversal
- junction/reparse escape
- destructive Git patterns
- PowerShell/cmd destructive aliases
- mixed quoting cases
- external writes
- approval mutation
- approval replay
- double-execution races
- R2 Guarded requester self-approval rejection
- R2 Autonomous requester self-approval audit flag
- R3 requester self-approval rejection
- R3 MCP allow rejection regardless of client ID in Phase 1
- R3 human-authorized execution exactly once
- token leakage regressions

Every discovered permission/authentication bug requires a regression test before the fix is considered complete.

### 27.8 Electron UI tests

Cover:

- Dashboard states/security summary
- Projects/Workspaces
- Git operational page
- Activity filters/detail/Live Logs view
- process output/stop/restart
- R2 approval behavior in both security modes
- R3 human-approval copy and rejection of MCP self-approval
- Settings mode-change confirmation
- explicit credential-copy UX
- Tunnel unavailable/available states
- Doctor PASS/WARN/FAIL rendering and remediation
- transport errors
- Agent degraded/error states

### 27.9 Doctor acceptance tests

Required examples:

**Healthy system**

- AgentCore healthy → PASS
- elevation present → PASS
- SQLite open/migrations/WAL healthy → PASS
- HTTP MCP authenticated and bound to loopback → PASS
- named pipe healthy → PASS
- writable directories → PASS

**Failure/warning examples**

- configured HTTP port conflict → WARN or FAIL according to whether HTTP transport is required for current scenario, with actionable remediation
- SQLite open/migration failure → FAIL
- non-elevated state when elevated startup is required → FAIL
- missing Git → WARN/FAIL according to affected coding capability, with explicit explanation
- missing Node → WARN/FAIL according to workspace needs
- missing pnpm/yarn when npm is available → WARN/informational unless selected workspace requires them
- stale/crashed previous session → WARN with explanation
- insecure/misconfigured bind attempt → FAIL

### 27.10 Critical E2E flows

**Flow A — Normal coding**

```text
launch
→ obtain current session HTTP credential through explicit test harness/user-equivalent setup
→ authenticated MCP connection
→ register fixture workspace
→ workspace_snapshot
→ read_file
→ search_text
→ apply_patch inside workspace
→ test/lint/typecheck/build
→ git_status/git_diff/git_log
→ project_dev
→ process_output
→ verify Activity/Live Logs timeline
```

**Flow B — R2 Guarded**

```text
sensitive request
→ APPROVAL_REQUIRED
→ requester MCP attempts allow
→ rejected (human approval required)
→ Control Center human allow
→ exact frozen request executes once
```

**Flow C — R2 Autonomous / Unrestricted**

```text
human explicitly enables Autonomous / Unrestricted mode
→ sensitive request
→ requester MCP self-approves
→ allowed
→ selfApproved=true
→ autonomousMode=true
→ visible audit/UI flags
```

**Flow D — R3 destructive**

```text
destructive request
→ APPROVAL_REQUIRED
→ requester MCP attempts allow
→ HUMAN_APPROVAL_REQUIRED / rejected
→ request remains PENDING
→ Control Center shows HUMAN APPROVAL REQUIRED
→ human allows
→ frozen hash matches
→ exact request executes once
→ replay/double approval does not execute again
→ audit confirms humanAuthorized=true and selfApproved=false
```

**Flow E — HTTP authentication/rotation**

```text
no token → rejected
wrong token → rejected
current token → succeeds
restart Agent
old token → rejected
new token → succeeds
verify no token in SQLite/audit/log/Doctor
```

**Flow F — Doctor**

```text
healthy setup → PASS results
simulate port conflict → actionable WARN/FAIL
simulate SQLite failure → FAIL
simulate missing Git/Node fixture → actionable diagnostics
```

### 27.11 Elevation/release smoke tests

Core logic should be testable without repeatedly invoking UAC. A packaged Windows release smoke test must separately verify:

- actual elevation
- actual PowerShell/cmd
- actual Git
- actual authenticated HTTP MCP
- actual session-token rotation after restart
- actual stdio bridge/named pipe ACL behavior
- R3 Control Center human approval
- Doctor diagnostics
- app shutdown cleanup

## 28. Phase 1 implementation roadmap

Phase 1 is implemented in ordered slices so each layer becomes testable before the next.

### Phase 1.1 — Foundation

Deliver:

- Electron/React/TypeScript app shell
- elevated lifecycle
- single instance
- SQLite + migrations
- Agent session lifecycle
- typed preload IPC foundation
- EventBus + operational logger
- basic Dashboard health

Acceptance:

- app starts elevated
- session becomes `RUNNING`
- SQLite is healthy
- app closes cleanly and session becomes `STOPPED`

### Phase 1.2 — Workspace and file core

Deliver:

- WorkspaceRegistry
- canonical PathPolicy
- ProjectDetector
- `workspace_*`
- `read_file`
- `search_text`
- `apply_patch`

Acceptance:

- register fixture repo
- detect project
- tree/snapshot/read/search work
- multi-file patch works
- traversal/junction tests pass
- every call audited
- file deletion path is classified R3

### Phase 1.3 — Git and Node project adapter

Deliver:

- `git_status`, `git_diff`, `git_log`
- `project_info`, `project_dev`, `test`, `lint`, `typecheck`, `build`
- npm/pnpm/yarn script detection
- Git operational service for Control Center page

Acceptance:

- fixture project can run all configured scripts
- absent script returns `PROJECT_SCRIPT_NOT_FOUND`
- Git output is bounded and audited
- Git page can show branch/status/diff/log without being an editor

### Phase 1.4 — Shell and ProcessManager

Deliver:

- raw `shell`
- foreground/background execution
- `process_*`
- bounded stdout/stderr
- timeout/restart/shutdown cleanup

Acceptance:

- normal shell command succeeds
- dev process starts and streams output
- managed process stop/restart works
- closing app cleans managed processes
- destructive shell fixture is classified R3 rather than directly executed

### Phase 1.5 — Policy, security modes and approvals

Deliver:

- Risk classifier
- ShellInspector
- SecurityModePolicy
- external path classification
- ApprovalManager
- approval tools
- immutable request/hash
- exactly-once transition
- R2 Guarded and Autonomous mode rules
- R3 human-only final approval

Acceptance:

- normal test/build auto-run
- external structured write creates R2 approval
- Guarded requester MCP cannot self-allow R2
- Autonomous requester MCP may self-allow R2 and visible flags are recorded
- delete/destructive Git creates R3 approval
- requester MCP cannot self-approve R3
- no MCP client can final-allow R3 in Phase 1
- Control Center human approval can final-allow R3
- replay/double-approval cannot execute twice

### Phase 1.6 — Authenticated MCP transports

Deliver:

- cryptographically random per-session HTTP bearer token
- localhost HTTP MCP auth middleware
- thin stdio bridge
- current-user named-pipe RPC ACL
- client lifecycle/auth metadata
- explicit Control Center credential-copy action

Acceptance:

- same tool behavior through authenticated HTTP and stdio
- same policy/audit behavior for both transports
- bridge returns `AGENT_NOT_RUNNING` when app is closed
- HTTP never binds outside loopback
- missing/invalid HTTP token rejected
- current token succeeds
- token rotates after restart
- token never appears in SQLite/audit/normal logs

### Phase 1.7 — Operational Control Center + Doctor

Deliver full navigation/pages:

- Dashboard
- Projects / Workspaces
- Git
- Activity / Live Logs
- Processes
- Approvals
- Tunnel status
- Doctor
- Settings

Acceptance:

A user can determine from the UI, without backend terminal logs:

- what Agent is doing
- which client requested it
- which workspace is affected
- current command/process state
- Git operational state
- why approval was required
- whether R2 MCP self-approval is allowed in the current mode
- that R3 requires human approval
- whether HTTP authentication is healthy
- whether core dependencies pass Doctor diagnostics
- whether remote Tunnel is unavailable/connected according to current milestone
- whether the action succeeded/failed

### Phase 1.8 — Hardening and Core release gate

Deliver:

- unit/contract/integration/security/auth suites
- Electron E2E
- Doctor acceptance suite
- packaged elevated Windows smoke test
- audit export support if needed for release diagnosis

Phase 1 Core is complete only when the full release-gate workflow passes on a packaged Windows build.

## 29. Phase 1 Core Definition of Done

The following workflow is the release gate:

```text
1. Launch packaged elevated Control Center
2. Verify Doctor baseline diagnostics
3. Verify fresh HTTP session credential exists in memory and is not shown by default
4. Attempt HTTP request without token → rejected
5. Attempt HTTP request with invalid token → rejected
6. Connect using valid current-session token
7. Register Node/TypeScript fixture repo
8. workspace_snapshot
9. read_file / search_text
10. apply_patch inside workspace
11. test
12. lint
13. typecheck
14. build
15. git_status / git_diff / git_log and Git page verification
16. project_dev
17. process_output
18. process_stop/restart validation
19. R2 Guarded request → requester MCP allow rejected → human allow succeeds
20. Enable Autonomous / Unrestricted explicitly in UI
21. R2 request → requester MCP self-approval succeeds with visible audit flag
22. R3 destructive request → requester MCP allow rejected
23. R3 human-authorized Control Center approval succeeds
24. verify frozen request hash and exactly-once execution
25. verify replay/double approval does not execute again
26. verify complete Activity/Live Logs/Audit timeline
27. verify token absent from SQLite/audit/normal logs/Doctor output
28. close app
29. verify pending approvals handled according to shutdown policy
30. verify managed-process cleanup and external processes untouched
31. restart app
32. verify old HTTP credential fails and new credential succeeds
33. verify clean new session state and Doctor status
```

Secure Remote Access work must not begin until this Core release gate is stable and unresolved permission/authentication bugs are closed.

## 30. Release 1.1 — Secure Remote Access

Secure Remote Access is the **immediate milestone after Phase 1 Core** because remote operation from ChatGPT/another device is central to the source product intent. It is deliberately not pulled into the Phase 1 Core release gate.

### 30.1 Secure outbound-oriented tunnel

Add a secure outbound-oriented tunnel adapter so remote trusted clients can reach the same execution pipeline:

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
Execution
   ↓
Audit
```

Requirements:

- reuse existing ToolDispatcher/PolicyEngine/ApprovalManager/ProcessManager/Audit
- do not create a remote-only policy/process/audit stack
- design remote identity/authentication explicitly for the remote transport
- do not reuse or expose the Phase 1 local HTTP session bearer token as the remote identity system by default
- do not inherit localhost trust assumptions automatically
- R3 remote requests still require human-authorized final approval
- the requesting remote agent/client cannot silently self-approve R3
- Tunnel page becomes active and reports bounded connection/auth/health state without secrets
- Doctor adds tunnel diagnostics when the feature is enabled

The exact protocol/provider/internal security implementation of the reference prototype is not inferred from screenshots; Release 1.1 must choose and document its own remote security design.

### 30.2 Release 1.1 acceptance target

From another trusted device/client:

```text
remote authenticated connection
→ workspace_snapshot
→ read_file
→ apply_patch
→ test
→ R2 policy behavior
→ R3 request requiring human-authorized approval
→ activity visible in Control Center
→ Tunnel/Doctor health visible
```

Remote release must include its own threat-model review before shipping.

## 31. Phase 2 — Agentic Development Capabilities

Phase 2 extends development capability after Phase 1 Core and Secure Remote Access. It must not create a second policy/audit/process stack.

### 31.1 `codex_run`

Add delegation to Local Codex CLI using the existing workspace, process, policy and audit infrastructure.

Do not create a separate process manager for Codex.

Exact tool schema is intentionally designed during Phase 2 after Phase 1 behavior is validated.

### 31.2 Browser CDP

Add `dom_cdp` capabilities for:

- navigate
- inspect/query DOM
- click/type
- evaluate JS
- screenshot

Target workflow:

```text
start dev server
→ open Chrome
→ inspect DOM
→ interact
→ verify web UI
```

### 31.3 Additional project adapters

Add Python/.NET/Rust adapters only according to actual usage priority. All implement the same `ProjectAdapter` boundary.

### 31.4 Remote/human approval UX refinement

If remote human approval is introduced, it must have explicit authenticated human identity/authorization semantics. R3 may be approved remotely only through a channel that the system recognizes as human-authorized; an agent executor credential alone is insufficient.

### Phase 2 Definition of Done

A trusted client can request an agentic coding task, delegate appropriately, modify/test the repo, launch the app, validate it through Chrome CDP, inspect Git state and see the complete activity trail in Control Center while preserving the same R2/R3 approval guarantees.

## 32. Phase 3 — Full Windows Desktop Agent

Phase 3 extends execution adapters beyond coding/browser workflows.

### 32.1 Windows UI Automation

Add semantic Microsoft UI Automation first:

- enumerate/find windows/controls
- accessibility tree
- invoke buttons/menus
- get/set supported control values/text

Prefer semantic UIA actions over coordinate clicking.

### 32.2 Window management

Add:

- list
- activate
- move/resize
- minimize/maximize
- close

Closing apps with potential unsaved state must pass policy classification and may be R2/R3 depending on data-loss semantics.

### 32.3 Vision + input fallback

Fallback chain:

```text
UI Automation
↓ if unavailable
Vision
↓
Keyboard/Mouse input events
```

Coordinate automation is a fallback, not default.

### 32.4 Clipboard + file dialogs

Add bounded/audited clipboard and native Open/Save dialog automation.

### 32.5 Office

Add structured Word/Excel COM automation first. Do not expose arbitrary generic COM invocation as the initial design.

### 32.6 Screen capture/recording

Add window/monitor/region selection, duration/storage bounds and privacy-visible state.

### 32.7 Notifications/scheduler

Add Windows notifications and Scheduled Task management. Scheduled Task mutations default to sensitive approval and may escalate according to semantics.

### 32.8 `web_fetch`

Add bounded local-machine HTTP fetching with explicit protocol, timeout, size, credential and local-network policies.

### 32.9 Audio

Add microphone/audio playback only with explicit visible state and privacy-aware approval rules.

### Phase 3 Definition of Done

A cross-application E2E workflow must successfully:

```text
modify project
→ build/run application
→ interact with Windows UI
→ open Excel/Word
→ write verification result
→ save
→ capture evidence
→ preserve one complete audit timeline
```

All Phase 3 destructive actions continue to obey the R3 human-authorized approval guarantee.

## 33. Cross-phase architecture rules

These rules are permanent unless a later approved design explicitly changes them:

1. **Phase 1 Core is the platform.** Release 1.1 extends remote transport; Phase 2 extends agentic development capabilities; Phase 3 extends desktop execution adapters.
2. One `ToolDispatcher` lifecycle for all tools.
3. One `PolicyEngine`/`ApprovalManager` authority.
4. One process-management abstraction where applicable.
5. One audit/event model across phases.
6. Renderer remains UI; privileged backend stays outside renderer.
7. Do not implement future-phase capabilities during Phase 1 merely as speculative scaffolding.
8. Do not introduce cross-platform abstraction in Phase 1.
9. Never advertise raw shell as sandboxed.
10. Loopback binding and authentication remain distinct security controls.
11. Local HTTP authentication does not imply hostile-host security or malware resistance.
12. R3 requester self-approval is prohibited across all phases.
13. R3 final approval must be human-authorized across local and remote workflows.
14. R2 requester self-approval is permitted only under an explicitly enabled autonomous policy and must be visible in audit/UI.
15. Remote mode requires a dedicated security review; local session-token and same-user assumptions are not inherited automatically.
16. Future adapters must reuse Dispatcher/Policy/Approval/Audit rather than implement their own permission stack.

## 34. Implementation constraints for Codex

When implementation planning begins, Codex must treat the following as fixed requirements rather than optional suggestions:

- Windows-only Phase 1
- Electron Control Center owns privileged Agent lifecycle
- elevated Administrator runtime
- modular monolith AgentCore
- thin stdio bridge, not a second AgentCore
- HTTP loopback only
- per-session cryptographically random HTTP bearer token
- HTTP token rotated on every Agent session and never persisted/logged in plaintext
- current-user-scoped named-pipe ACL where practical
- multi-workspace registry
- strong structured-tool canonical path resolution
- raw unrestricted shell with best-effort inspection disclaimer
- risk levels R0–R3
- R0/R1 autonomous normal coding behavior
- external structured write is R2
- R2 requester self-approval only in explicit Autonomous / Unrestricted mode
- user/project deletion and destructive Git/data-loss operations are R3
- R3 requester self-approval is prohibited
- Phase 1 MCP approval API cannot final-allow R3
- R3 final allow requires explicit human-authorized Control Center action
- frozen approval request + request hash
- exactly-once approval execution
- SQLite operational source of truth
- application-level append-only audit events
- bounded outputs everywhere
- typed preload IPC and no renderer filesystem/shell/SQLite access
- first-class Doctor diagnostics
- Control Center IA includes Dashboard, Projects/Workspaces, Git, Activity, Processes, Approvals, Tunnel, Doctor, Settings
- Node/TypeScript project adapter first
- no active secure remote tunnel in Phase 1 Core
- Secure Remote Access is the immediate milestone after the Phase 1 Core release gate
- no Phase 2/3 capability implementation in Phase 1 Core
- release gate must pass on a packaged elevated Windows build

## 35. Decisions intentionally deferred to implementation planning

The design is behaviorally complete. The following are implementation-level selections, not unresolved product requirements:

- exact Electron/Node/package versions
- exact runtime schema validation library
- exact SQLite Node driver
- exact test framework and bundler
- exact internal repository/package-manager layout
- exact UI component library
- exact JSON/framing implementation used over the named pipe
- exact secure remote tunnel provider/protocol for Release 1.1
- exact remote identity/credential storage mechanism, subject to the Release 1.1 security requirements
- exact human-identity mechanism for any future remote human approval channel

Whichever choices are made in the implementation plan must preserve the contracts and boundaries in this design.

## 36. Final Phase 1 success statement

Phase 1 Core succeeds when LocalGPT Agent behaves as a local, elevated Windows coding execution platform that an authenticated MCP client can use autonomously for normal coding work, while sensitive R2 actions follow the selected Guarded/Autonomous policy and destructive R3 actions always require independent human-authorized approval.

Every important action must be reconstructable from the Control Center Activity/Audit timeline; Doctor must make core health and security misconfiguration diagnosable; the localhost HTTP endpoint must require a fresh per-session bearer token; and shutdown/restart must clean managed state and rotate the local HTTP credential.

The system must be powerful, observable and predictable on a trusted developer workstation. It must not misrepresent raw shell as sandboxed, HTTP authentication as host isolation, or operational audit as a tamper-proof forensic ledger.
