# LocalGPT Agent — Windows Coding Agent Core Design

**Status:** Approved design for implementation planning  
**Date:** 2026-08-17  
**Working product name:** LocalGPT Agent  
**Primary target:** Windows developer workstation  

## 1. Purpose

สร้าง Local Desktop Agent บน Windows ที่เปิดผ่าน Electron Control Center แบบ elevated Administrator แล้วให้ MCP clients สั่งงาน coding workflow บนเครื่องจริงได้ เช่น อ่าน/ค้น/แก้ source code, inspect Git, รัน test/lint/typecheck/build, เปิด dev process, อ่าน stdout/stderr และรัน PowerShell/cmd แบบ raw shell โดยมี policy classification, destructive approval, process lifecycle และ audit trail กลางชุดเดียว

Phase 1 ต้องเป็น **Coding Agent Core** เท่านั้น ไม่ใช่ full desktop automation platform และไม่ใช่ hardened sandbox

## 2. Source baseline and design ownership

แนวทางจากเอกสาร/ภาพที่ผู้ใช้ให้มาเป็น baseline ของ capability families ดังนี้:

- `workspace_*` สำหรับหลายโปรเจกต์, structure และ snapshot
- `read_file`, `search_text`, `apply_patch`
- `git_status`, `git_diff`, `git_log`
- `project_dev`, `test`, `lint`, `typecheck`, `build`
- `process_*`, `shell`
- `codex_run`
- `dom_cdp`
- `accessibility`, `input_event`, `vision`, `window`
- `office`, `clipboard`, `file_dialog`
- `screen_record`, `audio`, `notification`, `scheduler`
- `web_fetch`, `system_info`, `health`
- secure outbound MCP tunnel เป็นแนวทาง remote connectivity ในระบบต้นฉบับ

รายละเอียด architecture, schemas, permission levels, approval state machine, SQLite model, phase decomposition และ UI/IPC ในเอกสารนี้เป็น design ของ LocalGPT Agent ที่ตกลงร่วมกัน ไม่ได้อ้างว่าเป็น implementation ภายในของระบบต้นฉบับ

## 3. Approved product decisions

Phase 1 ล็อก decision ต่อไปนี้แล้ว:

1. **Scope:** Coding Agent Core
2. **Stack:** Electron + React + TypeScript + Node.js
3. **OS:** Windows only
4. **Privilege:** Elevated Administrator / Unrestricted runtime
5. **Destructive policy:** Destructive Guard
6. **Path policy:** Full create/edit inside registered workspaces; read outside workspace; external writes require approval
7. **Shell:** Raw unrestricted PowerShell/cmd with best-effort destructive inspection
8. **MCP transports:** localhost HTTP + stdio bridge
9. **Lifecycle:** Elevated Desktop App owns Agent lifecycle; closing the app shuts down Agent Core
10. **Project intelligence:** Node.js/TypeScript first; generic shell/file/Git fallback for other stacks
11. **Approval channels:** Electron UI + MCP approval API
12. **Approval authority in Phase 1:** Any connected MCP client may approve, including self-approval
13. **Workspace model:** Multi-workspace registry keyed by `workspaceId`
14. **State store:** SQLite
15. **Control Center:** Operational Dashboard, not an IDE
16. **HTTP auth in Phase 1:** None; bind loopback only
17. **Architecture style:** Modular monolith inside the Electron host, with a thin stdio bridge

## 4. Goals

Phase 1 must prove this end-to-end chain on a packaged Windows app:

```text
MCP Client
   ↓
localhost HTTP or stdio bridge
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

A successful Phase 1 allows an MCP client to:

1. register a Node/TypeScript workspace
2. inspect workspace snapshot/tree
3. read and search files
4. patch one or multiple source files
5. run tests, lint, typecheck and build
6. inspect Git diff/status/log
7. launch a managed dev process
8. read incremental process output
9. request a destructive operation
10. complete approval flow
11. verify a complete activity/audit timeline

## 5. Non-goals for Phase 1

Phase 1 must **not** implement these capabilities:

- secure remote tunnel
- `codex_run`
- `dom_cdp`
- Windows UI Automation / accessibility control
- keyboard/mouse input fallback
- screen vision/OCR automation
- window automation
- Word/Excel COM automation
- clipboard automation
- native file-dialog automation
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

Interfaces may be designed so future adapters can plug in, but Phase 1 code must not implement Phase 2/3 features “เผื่อไว้”.

## 6. Security posture and threat model

### 6.1 Explicit posture

Phase 1 is designed for:

```text
Trusted developer workstation
+
trusted local users/processes
+
high-powered Administrator automation
+
audit + best-effort destructive guard
```

Phase 1 is **not** designed for:

```text
Hostile multi-user machine
Untrusted localhost processes
Malware resistance
Privilege isolation
Strong shell containment
```

The combination of:

```text
Administrator
+ Raw Shell
+ unauthenticated localhost HTTP
+ any-client approval
```

means a local process able to reach the MCP endpoint may be able to invoke privileged actions and self-approve approval-gated actions. This must be displayed as a visible product warning, not hidden in documentation only.

### 6.2 Important guarantee boundary

Structured tools enforce workspace/path policy deterministically before execution.

Raw shell cannot provide the same guarantee. Shell classification is **best effort** because PowerShell/cmd can call scripts, encoded commands, downloaded executables and child processes whose behavior cannot be statically determined reliably.

Therefore:

- Path policy is a strong application rule for structured tools.
- Path/destructive policy for raw shell is an inspection guard, **not a sandbox**.

## 7. Top-level architecture

```text
┌───────────────────────────────────────────────────────────────┐
│ Electron Desktop Control Center                               │
│                                                               │
│  ┌──────────────── React Renderer ──────────────────────────┐ │
│  │ Dashboard │ Workspaces │ Activity │ Processes           │ │
│  │ Approvals │ Settings                                  │ │
│  └───────────────────┬─────────────────────────────────────┘ │
│                      │ typed preload IPC                     │
│  ┌───────────────────▼─────────────────────────────────────┐ │
│  │ Agent Core                                              │ │
│  │                                                         │ │
│  │ McpGateway                                              │ │
│  │ ├─ localhost HTTP                                       │ │
│  │ └─ named-pipe RPC target for stdio bridge               │ │
│  │          │                                              │ │
│  │          ▼                                              │ │
│  │ ToolDispatcher → ToolRegistry                           │ │
│  │          │                                              │ │
│  │          ▼                                              │ │
│  │ PolicyEngine                                            │ │
│  │ ├─ PathPolicy                                           │ │
│  │ └─ ShellInspector                                       │ │
│  │          │                                              │ │
│  │          ├─ allow ───────────────► Execution            │ │
│  │          └─ approval ─► ApprovalManager ─► Execution    │ │
│  │                                     │                   │ │
│  │ Execution Layer                     │                   │ │
│  │ ├─ Workspace/File                    │                   │ │
│  │ ├─ Git                               │                   │ │
│  │ ├─ Project Adapter                   │                   │ │
│  │ ├─ Shell                             │                   │ │
│  │ └─ ProcessManager                    │                   │ │
│  │          │                           │                   │ │
│  │          └─────────────┬─────────────┘                   │ │
│  │                        ▼                                 │ │
│  │                    EventBus                              │ │
│  │                 ┌──────┴──────┐                         │ │
│  │                 ▼             ▼                         │ │
│  │              AuditWriter   UiBroadcaster                 │ │
│  │                 │                                       │ │
│  │                 ▼                                       │ │
│  │               SQLite                                    │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘

MCP client ──stdio──► mcp-stdio-bridge ──Windows named pipe──► Agent Core
```

### 7.1 Core rules

- MCP transports contain no tool business logic.
- Tool implementations do not make their own permission decisions.
- React renderer never touches filesystem, shell, SQLite or MCP server internals directly.
- Every tool invocation follows one lifecycle owned by `ToolDispatcher`.
- Every privileged action is auditable through the same event model.

## 8. Standard tool execution lifecycle

Every MCP tool call follows this exact conceptual path:

```text
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

Tool implementations must never send MCP responses directly.

## 9. Permission model

### 9.1 Risk levels

| Level | Meaning | Default behavior |
|---|---|---|
| `R0 READ_ONLY` | No state mutation | Auto allow |
| `R1 NORMAL_WRITE_EXECUTE` | Normal coding write/execute inside workspace | Auto allow |
| `R2 SENSITIVE` | External/system/high-impact mutation | Approval required |
| `R3 DESTRUCTIVE` | Likely data/state loss | Approval required |

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

`R3`:

- deleting a user/project file or directory
- deleting a file through `apply_patch`
- truncating/clearing content in a way classified as data loss
- `git reset --hard`
- `git clean -fd`
- destructive `git restore` / `git checkout -- <path>` patterns
- force push
- destructive disk/filesystem commands

### 9.3 Path policy matrix

| Target | Read | Create/Edit | Delete |
|---|---:|---:|---:|
| Registered workspace | Allow | Allow | Approval |
| Outside workspace | Allow | Approval | Approval |
| Windows/system-sensitive path | Allow if OS ACL allows | Approval | Approval |

Agent-owned ephemeral runtime artifacts under the Agent data directory are a special operational class. Bounded rotation/pruning of Agent-generated temporary process logs may occur according to configured retention; this exception must never apply to user workspace/source files.

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

Known risky patterns become `R2` or `R3`. Unknown commands remain executable under the selected unrestricted mode, with audit flags such as `unknownRisk=true` where appropriate.

The UI and documentation must never claim shell inspection is containment.

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

## 12. Approval model

### 12.1 Approval semantics

The original tool request does not stay blocked on an open MCP connection.

Flow:

```text
1. Client invokes risky tool.
2. Policy returns APPROVAL_REQUIRED.
3. Agent freezes exact tool name + arguments and creates approvalRequestId.
4. Original call returns APPROVAL_REQUIRED + approvalRequestId.
5. Electron UI or any MCP client calls approval_decide.
6. On ALLOW, Agent validates the frozen request hash and executes that exact request once.
7. approval_decide returns the execution outcome.
```

### 12.2 State machine

```text
PENDING
   ├─ deny ─────────────► DENIED
   ├─ expire ───────────► EXPIRED
   └─ allow
          ▼
       EXECUTING
          ├─ success ───► COMPLETED
          └─ failure ───► FAILED
```

Terminal approval states are not revivable. A new execution attempt requires a new tool call and approval request.

### 12.3 Exactly-once protection

The Agent must prevent:

- double approval causing double execution
- replay of completed approvals
- mutation of command/arguments after approval
- execution of expired/denied approvals

Before enqueueing approval, canonicalize and hash the frozen tool request. Before execution, the hash must still match.

Approval decision must use an atomic database transition, so concurrent UI/MCP decisions cannot both execute the request.

### 12.4 Self-approval

Phase 1 deliberately allows:

```text
requesterClientId === approverClientId
```

When this happens, set `selfApproved=true` and display a visible `SELF-APPROVED` indicator in history.

This makes the Phase 1 approval API a workflow/audit guard, not a separation-of-duties security control.

### 12.5 Expiration

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

- Policy: `R2`, approval required

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
- File deletion: `R3`

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

No dedicated `git_commit`, `git_push` or `git_reset` tool is required in Phase 1. Those remain available through raw shell and therefore pass ShellInspector.

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

Policy: classified dynamically by `ShellInspector`.

### 14.6 Process tools

#### `process_list`
Lists Agent-managed processes by default.

#### `process_get`
Returns process metadata/state.

#### `process_output`
Returns incremental bounded stdout/stderr using cursor/offset semantics.

#### `process_stop`
- Agent-managed process: auto allow
- External process: `R2` approval

#### `process_restart`
Restarts an Agent-managed process using frozen launch metadata.

Agent `processId` is the primary identity; Windows PID is metadata only.

### 14.7 Approval tools

#### `approval_list`
Lists pending/recent approvals.

#### `approval_get`
Returns frozen request, risk reasons, requester, expiry and decision state.

#### `approval_decide`
Input:

```ts
{
  approvalRequestId: string
  decision: "allow" | "deny"
  note?: string
}
```

On allow, executes the exact frozen request once.

### 14.8 Operational tools

#### `health`
Returns Agent version/uptime plus health of SQLite, transports and process manager.

#### `system_info`
Returns bounded coding-relevant Windows/system info such as OS version, architecture, CPU/RAM summary, disk summary and Node/Git availability.

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

**Validation**

- `INVALID_ARGUMENT`
- `UNKNOWN_TOOL`
- `INVALID_WORKSPACE`
- `INVALID_PATH`

**Policy/approval**

- `APPROVAL_REQUIRED`
- `POLICY_DENIED`
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
- `AGENT_NOT_RUNNING`

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
shutdown_reason TEXT NULL
status TEXT
```

Statuses:

- `STARTING`
- `RUNNING`
- `SHUTTING_DOWN`
- `STOPPED`
- `CRASHED`

### 16.5 `clients`

```text
id TEXT PK
session_id TEXT
name TEXT
transport TEXT
connected_at DATETIME
last_seen_at DATETIME
disconnected_at DATETIME NULL
metadata_json TEXT
```

Transport values include:

- `stdio`
- `http`
- `internal_ui`

Client identity is operational/audit metadata only; without HTTP authentication it is not a cryptographically trusted principal.

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

### 16.7 `approvals`

```text
id TEXT PK
tool_call_id TEXT UNIQUE
requester_client_id TEXT
approver_client_id TEXT NULL
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
self_approved INTEGER
```

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
- `TOOL_REQUEST_RECEIVED`
- `POLICY_DECIDED`
- `APPROVAL_CREATED`
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
- `AGENT_SHUTDOWN_STARTED`
- `AGENT_SHUTDOWN_COMPLETED`

`tool_calls` represents current/final invocation state; `audit_events` reconstructs the timeline.

The audit log is operational and inspectable, not a tamper-proof forensic ledger. An Administrator can still modify the local database outside the application.

### 16.11 Settings

Logical `settings` values include:

- `mcp.http.enabled`
- `mcp.http.port`
- `mcp.stdio.enabled`
- `shell.default`
- `shell.defaultTimeoutMs`
- `approval.expirationMinutes`
- `process.shutdownGraceMs`
- process-output limits/retention
- audit display/retention policy

Default shutdown grace: **5 seconds** before best-effort force termination of Agent-managed processes.

## 17. Event bus and live UI

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

The UI must not poll SQLite directly.

## 18. Session and process lifecycle

### 18.1 One app launch = one Agent session

Startup creates one session record. Closing the app closes that session.

If a previous session remains `RUNNING` without a clean shutdown marker, mark it `CRASHED` on next startup.

### 18.2 Managed process lifecycle

States include:

- `STARTING`
- `RUNNING`
- `EXITED`
- `START_FAILED`
- `ORPHANED`

Agent `processId` is a UUID/opaque ID. Never use PID as the stable external identity because Windows may reuse PIDs.

Record PID + process start timestamp + command for identity checks.

### 18.3 Shutdown

When Electron exits:

```text
Agent → SHUTTING_DOWN
↓
reject new executions
↓
cancel pending approvals
↓
allow current foreground work a bounded grace period
↓
stop Agent-managed background/dev processes
↓
leave external processes untouched
↓
flush audit/state
↓
stop MCP transports and named-pipe RPC
↓
close SQLite
↓
STOPPED
```

### 18.4 Crash recovery

Do not automatically adopt old process records after restart. If a previous managed process cannot be safely re-identified, mark the old record `ORPHANED`.

## 19. MCP transports

### 19.1 Localhost HTTP

Endpoint concept:

```text
http://127.0.0.1:<configured-port>/mcp
```

Requirements:

- bind loopback only
- no `0.0.0.0`
- no LAN binding setting in Phase 1
- no authentication in Phase 1
- do not enable permissive browser CORS as a convenience feature
- reject/flag invalid non-loopback binding configuration

If HTTP port startup fails, Agent Core may remain running with transport status `ERROR: PORT_IN_USE`; the UI must expose the partial failure.

### 19.2 stdio bridge

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

If the Desktop App is not running, return stable `AGENT_NOT_RUNNING` behavior.

Named pipe ACL should be scoped to the current user where practical. This is defense-in-depth, not a replacement for the trusted-local-machine posture.

## 20. Electron Control Center

The Electron application is a control/observability/approval plane, not an IDE.

### 20.1 Electron renderer security

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

### 20.2 Pages

#### Dashboard

Must visibly show:

- Agent status and uptime
- Administrator/elevation state
- raw-shell status
- HTTP endpoint/status
- `HTTP AUTH: DISABLED`
- stdio bridge status
- connected clients
- active processes
- pending approvals
- recent activity

#### Workspaces

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

#### Live Activity

Filter by:

- time
- workspace
- tool
- client
- risk
- status

Tool detail view must show timeline, policy decision, exact/audit-safe arguments, output and event history.

#### Processes

Show Agent-managed process cards with:

- Agent process ID
- PID
- workspace
- command/cwd
- uptime/state
- recent output
- restart/stop actions

Process output viewer is bounded streaming text, not a full terminal emulator.

#### Approvals

Pending approvals must display:

- workspace
- client
- tool
- frozen command/arguments
- risk level/reasons
- expiry
- allow/deny
- self-approval indicator where relevant

#### Settings

At minimum:

- default shell
- default command timeout
- shutdown grace
- HTTP enabled/port
- stdio enabled/status
- approval expiry
- process-output limits
- audit/log retention controls
- read-only security posture summary

Security summary must include:

```text
Elevation: Administrator
Shell: Unrestricted
External writes: Approval required
Destructive actions: Approval required
HTTP authentication: None
Approval clients: Any MCP client
```

## 21. Typed IPC boundary

Preload exposes narrow named APIs conceptually like:

```ts
window.agent.dashboard.getStatus()
window.agent.workspaces.list()
window.agent.workspaces.register(...)
window.agent.workspaces.update(...)
window.agent.activity.list(...)
window.agent.activity.subscribe(...)
window.agent.processes.list()
window.agent.processes.stop(...)
window.agent.approvals.list()
window.agent.approvals.decide(...)
window.agent.settings.get()
window.agent.settings.update(...)
```

Renderer UX restrictions are not security decisions. AgentCore still validates, classifies and audits every mutation reached through IPC.

## 22. Internal module boundaries

Logical structure:

```text
AgentCore
│
├─ McpGateway
│  ├─ HttpTransport
│  └─ NamedPipeRpcServer
│
├─ ToolDispatcher
├─ ToolRegistry
│
├─ PolicyEngine
│  ├─ PathPolicy
│  └─ ShellInspector
│
├─ ApprovalManager
├─ WorkspaceManager
├─ ProjectDetector / ProjectAdapter
├─ ProcessManager
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

Repository/project source layout may be selected in the implementation plan, but boundaries above are requirements. The implementation must preserve one-way dependencies so renderer/preload do not become backend execution layers.

## 23. Runtime schemas

MCP inputs, IPC inputs and domain data crossing module boundaries require runtime validation, not TypeScript compile-time types only.

Use one shared schema source per contract and derive TypeScript types from it where practical. The exact schema library and SQLite driver are implementation choices, but they must be isolated so contracts do not depend on a specific transport or persistence driver.

## 24. Startup sequence

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
initialize repositories/EventBus
↓
initialize PolicyEngine
↓
initialize ToolRegistry/Dispatcher
↓
start Named Pipe RPC
↓
start localhost HTTP MCP
↓
renderer ready
↓
Agent RUNNING
```

Critical failures in SQLite, PolicyEngine or ToolRegistry prevent `RUNNING` state.

Non-critical transport failure may yield partial-running status surfaced in the Dashboard.

Use a single-instance lock so a second launch foregrounds the existing Control Center instead of spawning a second privileged AgentCore/SQLite/process manager.

## 25. Failure handling requirements

### 25.1 `apply_patch`

- parse/validate the full patch before modifying files
- resolve all targets before write
- compute edits in memory where practical
- use per-file temp + atomic replace where practical
- multi-file atomicity is not guaranteed
- partial failure must list which files changed and which failed
- audit partial failures explicitly

### 25.2 Foreground shell timeout

On timeout:

1. mark timeout state
2. best-effort terminate Agent-owned process tree
3. capture final bounded output
4. audit timeout
5. return `COMMAND_TIMEOUT`

Do not claim child-process containment is perfect.

### 25.3 Output bounding

No MCP response may dump unbounded stdout/stderr, file content, tree data or search results. Every potentially large tool requires explicit bounds/truncation metadata.

## 26. Testing strategy

Phase 1 requires five test layers:

```text
Static / Typecheck
Unit
Contract
Integration
E2E / packaged Windows smoke
```

### 26.1 Unit tests

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
taskkill               → R2/R3
reg add                → R2
unknown-tool foo       → allowed unrestricted path + audit flag
```

**ApprovalManager**

- allow
- deny
- expiry
- self-approval
- concurrent double approval
- replay
- hash mismatch
- exactly-once execution

### 26.2 MCP tool contract tests

Every tool must test:

- schema validation
- policy classification
- success shape
- known failure shape
- audit event creation

### 26.3 Persistence integration tests

Use a fresh temporary SQLite DB per test group to cover:

- migrations
- foreign keys
- WAL/busy behavior
- session lifecycle
- tool-call lifecycle
- approval atomic transition
- audit append
- process records
- crash recovery

### 26.4 Real filesystem/Git integration tests

Use Windows temporary fixture repositories, never the developer's real repository.

Test:

- read/search/patch
- canonical paths
- Git inspection
- project detection
- package-manager/script detection

### 26.5 Shell/process integration tests

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

### 26.6 Security regression suite

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

Every discovered permission bug requires a regression test before the fix is considered complete.

### 26.7 Electron UI tests

Cover:

- dashboard states
- activity filters/detail
- process output/stop/restart
- approval allow/deny/self-approved display
- settings validation
- transport errors
- Agent degraded/error states

### 26.8 Critical E2E flows

**Flow A — Coding**

```text
launch
→ register fixture workspace
→ workspace_snapshot
→ read_file
→ search_text
→ apply_patch
→ git_diff
→ verify Activity UI
```

**Flow B — Project execution**

```text
project_info
→ test
→ project_dev
→ process_output
→ process_stop
```

**Flow C — Destructive approval**

```text
destructive shell request
→ APPROVAL_REQUIRED
→ request visible in UI
→ allow
→ exact frozen request executes once
→ audit history matches
```

**Flow D — MCP self-approval**

```text
client creates approval
→ same client approves
→ execution
→ selfApproved=true
```

### 26.9 Elevation/release smoke tests

Core logic should be testable without repeatedly invoking UAC. A packaged Windows release smoke test must separately verify:

- actual elevation
- actual PowerShell/cmd
- actual Git
- actual HTTP MCP
- actual stdio bridge/named pipe
- app shutdown cleanup

## 27. Phase 1 implementation roadmap

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

### Phase 1.3 — Git and Node project adapter

Deliver:

- `git_status`, `git_diff`, `git_log`
- `project_info`, `project_dev`, `test`, `lint`, `typecheck`, `build`
- npm/pnpm/yarn script detection

Acceptance:

- fixture project can run all configured scripts
- absent script returns `PROJECT_SCRIPT_NOT_FOUND`
- Git output is bounded and audited

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

### Phase 1.5 — Policy and approvals

Deliver:

- Risk classifier
- ShellInspector
- external path classification
- ApprovalManager
- approval tools
- immutable request/hash
- exactly-once transition

Acceptance:

- normal test/build auto-run
- delete/destructive Git require approval
- external structured write requires approval
- self-approval succeeds and is visibly marked
- replay/double-approval cannot execute twice

### Phase 1.6 — MCP transports

Deliver:

- localhost HTTP MCP
- thin stdio bridge
- named-pipe RPC
- client lifecycle metadata

Acceptance:

- same tool behavior through HTTP and stdio
- same policy/audit behavior for both transports
- bridge returns `AGENT_NOT_RUNNING` when app is closed
- HTTP never binds outside loopback

### Phase 1.7 — Operational Dashboard

Deliver full pages:

- Dashboard
- Workspaces
- Activity
- Processes
- Approvals
- Settings

Acceptance:

A user can determine from the UI, without backend terminal logs:

- what Agent is doing
- which client requested it
- which workspace is affected
- current command/process state
- why approval was required
- whether the action succeeded/failed

### Phase 1.8 — Hardening and release gate

Deliver:

- unit/contract/integration/security suites
- Electron E2E
- packaged elevated Windows smoke test
- audit export support if needed for release diagnosis

Phase 1 is complete only when the full release-gate workflow passes on a packaged Windows build.

## 28. Phase 1 Definition of Done

The following exact workflow is the release gate:

```text
1. Launch packaged elevated Control Center
2. Register Node/TypeScript fixture repo
3. workspace_snapshot
4. read_file / search_text
5. apply_patch
6. test
7. lint
8. typecheck
9. build
10. git_diff
11. project_dev
12. process_output
13. process_stop/restart validation
14. send destructive shell/Git command
15. receive APPROVAL_REQUIRED
16. approve exact frozen request
17. verify exactly-once execution
18. verify complete Activity/Audit timeline
19. close app
20. verify managed-process cleanup and clean session shutdown
```

Phase 2 must not begin while this flow is flaky, non-deterministic, or has unresolved permission bugs.

## 29. Phase 2 — Remote + Agentic Development

Phase 2 extends the Phase 1 platform; it must not create a second policy/audit/process stack.

### 29.1 Secure remote transport

Add an outbound secure tunnel adapter so remote ChatGPT/MCP clients can reach the same `McpGateway → ToolDispatcher → Policy → Audit` path.

Remote security/identity must be redesigned explicitly. Phase 1 assumptions (`no HTTP auth`, `any MCP client can approve`) must **not** automatically carry into remote mode.

Acceptance target:

```text
remote client
→ workspace_snapshot
→ read_file
→ apply_patch
→ test
→ activity visible in Control Center
```

### 29.2 `codex_run`

Add delegation to Local Codex CLI using the existing workspace, process, policy and audit infrastructure.

Do not create a separate process manager for Codex.

Exact tool schema is intentionally designed during Phase 2 after Phase 1 behavior is validated.

### 29.3 Browser CDP

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

### 29.4 Additional project adapters

Add Python/.NET/Rust adapters only according to actual usage priority. All implement the same `ProjectAdapter` boundary.

### 29.5 Remote approval UX

Design authenticated remote approval/identity separately. Revisit self-approval and approver authority before remote release.

### Phase 2 Definition of Done

From another device, a trusted remote client can request a coding task, modify/test the repo, launch the app, validate it through Chrome CDP, inspect Git diff and see the complete activity trail in Control Center.

## 30. Phase 3 — Full Windows Desktop Agent

Phase 3 extends execution adapters beyond coding/browser workflows.

### 30.1 Windows UI Automation

Add semantic Microsoft UI Automation first:

- enumerate/find windows/controls
- accessibility tree
- invoke buttons/menus
- get/set supported control values/text

Prefer semantic UIA actions over coordinate clicking.

### 30.2 Window management

Add:

- list
- activate
- move/resize
- minimize/maximize
- close

Closing apps with potential unsaved state must pass policy classification.

### 30.3 Vision + input fallback

Fallback chain:

```text
UI Automation
↓ if unavailable
Vision
↓
Keyboard/Mouse input events
```

Coordinate automation is a fallback, not default.

### 30.4 Clipboard + file dialogs

Add bounded/audited clipboard and native Open/Save dialog automation.

### 30.5 Office

Add structured Word/Excel COM automation first. Do not expose arbitrary generic COM invocation as the initial design.

### 30.6 Screen capture/recording

Add window/monitor/region selection, duration/storage bounds and privacy-visible state.

### 30.7 Notifications/scheduler

Add Windows notifications and Scheduled Task management. Scheduled Task mutations default to sensitive approval.

### 30.8 `web_fetch`

Add bounded local-machine HTTP fetching with explicit protocol, timeout, size, credential and local-network policies.

### 30.9 Audio

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

## 31. Cross-phase architecture rules

These rules are permanent unless a later approved design explicitly changes them:

1. **Phase 1 is the platform.** Phase 2 extends transports/dev capabilities; Phase 3 extends execution adapters.
2. One `ToolDispatcher` lifecycle for all tools.
3. One `PolicyEngine`/`ApprovalManager` authority.
4. One process-management abstraction where applicable.
5. One audit/event model across phases.
6. Renderer remains UI; privileged backend stays outside renderer.
7. Do not implement future-phase features during Phase 1 merely as speculative scaffolding.
8. Do not introduce cross-platform abstraction in Phase 1.
9. Never advertise raw shell as sandboxed.
10. Remote mode requires a new security review; localhost trust assumptions are not inherited automatically.

## 32. Implementation constraints for Codex

When implementation planning begins, Codex must treat the following as fixed requirements rather than optional suggestions:

- Windows-only Phase 1
- Electron Control Center owns privileged Agent lifecycle
- elevated Administrator runtime
- modular monolith AgentCore
- thin stdio bridge, not a second AgentCore
- HTTP loopback only and unauthenticated in Phase 1
- multi-workspace registry
- strong structured-tool path resolution
- raw unrestricted shell with best-effort inspection disclaimer
- risk levels R0–R3
- external structured write requires approval
- user/project deletion requires approval
- any MCP client may self-approve in Phase 1, with visible audit flag
- SQLite operational source of truth
- application-level append-only audit events
- bounded outputs everywhere
- Node/TypeScript project adapter first
- no Phase 2/3 features in Phase 1
- release gate must pass on a packaged elevated Windows build

## 33. Decisions intentionally deferred to implementation planning

The design is behaviorally complete. The following are implementation-level selections, not unresolved product requirements:

- exact Electron/Node/package versions
- exact runtime schema validation library
- exact SQLite Node driver
- exact test framework and bundler
- exact internal repository/package-manager layout
- exact UI component library
- exact JSON/framing implementation used over the named pipe

Whichever choices are made in the implementation plan must preserve the contracts and boundaries in this design.

## 34. Final Phase 1 success statement

Phase 1 succeeds when LocalGPT Agent behaves as a local, elevated Windows coding execution platform that an MCP client can use autonomously for normal coding work, while destructive/sensitive operations are surfaced through a consistent approval workflow and every important action can be reconstructed from the Control Center audit timeline.

It must be powerful, observable and predictable on a trusted developer workstation; it must not misrepresent itself as a hardened security sandbox.
