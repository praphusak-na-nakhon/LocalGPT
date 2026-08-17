# LocalGPT Agent — Canonical Windows Agent Gateway Design

**Status:** Canonical approved design for implementation planning  
**Date:** 2026-08-17  
**Working product name:** LocalGPT Agent  
**Primary target:** Windows developer workstation  
**Deployment posture:** Personal-use / trusted developer workstation  

> This document is the single implementation source of truth for new LocalGPT Agent work.
>
> It consolidates and supersedes the implementation guidance previously spread across:
>
> - `2026-08-17-localgpt-agent-design.md`
> - `2026-08-17-localgpt-agent-approval-authority-amendment.md`
> - `2026-08-17-localgpt-agent-personal-use-ergonomics-amendment.md`
>
> The older files remain useful as design history, but Codex/implementers must not resolve policy from them independently when this canonical document is available.

---

## 1. Purpose

สร้าง Local Desktop Agent Gateway บน Windows ที่เปิดผ่าน Electron Control Center แบบ elevated Administrator แล้วให้ AI/MCP clients ลงมือทำงานจริงบนเครื่องได้อย่างเป็นระบบ โดย Phase 1 เน้น coding workflow ได้แก่:

- workspace/project access
- bounded file read/search/edit/copy
- Git inspection
- Node/TypeScript project commands
- raw PowerShell/cmd
- managed processes
- policy + approvals
- audit/activity
- local MCP transports
- Doctor diagnostics

ระบบต้องรักษา product intent สำคัญสองด้านพร้อมกัน:

1. **Normal coding work should be cheap.** การอ่านไฟล์ แก้ source code รัน test/lint/typecheck/build และเปิด dev process ภายในขอบเขตที่กำหนดควรทำได้โดยไม่ต้องกดอนุมัติทุกครั้ง
2. **Privilege boundaries must remain explicit.** การลบ/ทำลายข้อมูล, opaque execution, writes นอก workspace, system mutation และ privileged remote access ต้องผ่าน policy/approval ตามระดับความเสี่ยงที่กำหนด

Phase 1 คือ **Coding Agent Core** ไม่ใช่ full desktop automation platform, malware sandbox, IDE หรือระบบที่ต้องไล่จำนวน tools ให้เท่ากับ reference implementation ใด ๆ

---

## 2. Source baseline, reference materials and evidence boundary

Reference materials ที่ผู้ใช้ให้มาแสดง product direction ของ Windows AI Agent Gateway ที่มีลักษณะสำคัญดังนี้:

- AI client เชื่อมเข้ากับ local/remote agent gateway
- gateway จัดการ tools, permissions และ logs
- AI สามารถอ่าน/แก้ไฟล์ รันคำสั่ง ดู Git และควบคุม workflow บน Windows จริง
- มี Live Logs / work activity / Doctor-style diagnostics
- roadmap/capability breadth สามารถขยายไป browser, Windows UI, Office, vision, managed tasks, agent delegation, context intelligence และ external MCP
- source material ระบุ catalog ขนาดใหญ่ถึง `184 tools` และ capability groups จำนวนมาก

เอกสารนี้ใช้ข้อมูลดังกล่าวเป็น **capability/product-direction baseline** เท่านั้น

เอกสารนี้ **ไม่อ้าง** ว่าเราทราบ:

- internal architecture ของระบบอ้างอิง
- tunnel/authentication protocol ภายใน
- permission enforcement implementation ภายใน
- เหตุผลหรือกลไกของ claim เรื่อง quota/unlimited usage
- exact schemas ของ 184 tools นอกจากสิ่งที่ source แสดงให้เห็น

ดังนั้น LocalGPT รับแนวคิดที่มีประโยชน์ แต่เลือก architecture, security, phasing และ tool granularity ของตนเอง

### 2.1 Tool count is not a success metric

`184 tools` ไม่ใช่ Definition of Done ของ LocalGPT

สิ่งที่มีค่าจากระบบขนาดใหญ่คือ architecture ที่รองรับ catalog ที่โตขึ้น เช่น:

- on-demand tool/schema discovery
- context ranking/indexing
- code intelligence
- task orchestration
- batch/dependency execution
- handoff/checkpoint
- extension governance

LocalGPT จึง optimize สำหรับ:

```text
small coherent Phase 1 tool surface
+
strong policy/audit foundation
+
future discovery/composition architecture
```

แทนการเพิ่ม wrappers จำนวนมากเพื่อให้ตัวเลข tool สูง

---

## 3. Locked product decisions

Phase 1 ล็อก decision ต่อไปนี้:

1. **OS:** Windows only
2. **Stack:** Electron + React + TypeScript + Node.js
3. **Privilege:** Electron AgentCore runs elevated Administrator
4. **Architecture:** modular monolith AgentCore
5. **Core flow:** `McpGateway → ToolDispatcher → PolicyEngine → ApprovalManager → ResourceGovernor → Execution → Audit/EventBus`
6. **Workspace:** multi-workspace registry keyed by opaque `workspaceId`
7. **Structured paths:** canonical Windows path resolution; deterministic policy before mutation
8. **Shell:** raw PowerShell/cmd available; inspection is best effort, not sandboxing
9. **Local transports:** authenticated localhost HTTP + thin stdio bridge over current-user Windows Named Pipe
10. **Persistent local MCP recommendation:** stdio bridge is preferred for editor/desktop integrations that survive restarts
11. **HTTP credential:** cryptographically random per-Agent-session bearer token; memory-only; rotates every session
12. **Security mode:** every new Agent session starts `Guarded`
13. **Autonomous:** current-session explicit human opt-in only; supports bounded lease; never auto-restored after restart/crash/relaunch
14. **R3:** human-only final authorization in every mode
15. **Opaque indirect execution:** human-only final authorization in every mode even when semantic risk is only R2
16. **Direct unknown execution:** R2 by default; human in Guarded; Autonomous-eligible
17. **Repository scripts:** structured project scripts may remain R1 by explicit ProjectAdapter contract but are marked `repository_controlled`
18. **Content trust:** workspace/Git/process/browser content is data, not policy/approval authority
19. **Audit:** SQLite operational history + EventBus; not tamper-proof forensic logging
20. **Retention:** audit events 30 days by default; full process output 7 days by default
21. **Resource governance:** bounded executions/processes/approvals/transport admission
22. **Control Center:** operational dashboard, not IDE
23. **Remote:** Secure Remote Access is Release 1.1 immediately after Phase 1 Core
24. **Later capability growth:** developer intelligence before broad multi-agent/desktop expansion
25. **No blanket trusted workspace:** use narrow Workspace Command Rules instead

---

## 4. Goals

Phase 1 packaged Windows build must prove this chain:

```text
MCP Client
   ↓
McpGateway
   ↓
ToolDispatcher
   ↓
PolicyEngine
   ├─ PathPolicy
   ├─ ShellInspector
   ├─ SecurityModePolicy
   ├─ ExecutionTrustPolicy
   └─ WorkspaceCommandRulePolicy
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

A successful Phase 1 allows a client to:

1. register a workspace
2. inspect bounded tree/snapshot
3. read/search files
4. patch text files
5. copy files through structured path policy
6. run project test/lint/typecheck/build/dev
7. inspect Git status/diff/log
8. run PowerShell/cmd
9. manage Agent-owned processes
10. receive policy classification before risky execution
11. complete authorized R2/R3 approval flows
12. see Activity/Live Logs/Audit
13. inspect Doctor health
14. restart Agent and observe fresh HTTP credential + Guarded mode

---

## 5. Non-goals for Phase 1 Core

Phase 1 must not implement:

- active secure remote tunnel transport
- `codex_run`
- Browser Debug Adapter / `dom_cdp`
- workspace semantic index
- LSP/code intelligence
- smart task-specific context engine
- managed multi-step task orchestration
- `tool_batch`
- agent delegation/parallel workers
- plugin ecosystem
- external MCP auto-discovery/chaining
- Windows UI Automation
- keyboard/mouse automation
- vision fallback
- Word/Excel automation
- PDF visual comparison
- generic browser/network automation
- audio/screen recording
- notifications/scheduler
- Windows Service/tray persistence
- macOS/Linux abstraction
- IDE/editor functionality
- malware-resistant sandboxing

Phase 1 interfaces may remain extensible, but implementation must not add speculative future subsystems merely because they are on the roadmap.

---

# Part I — Security, policy and trust

## 6. Threat model

### 6.1 Designed for

```text
Personal/trusted developer workstation
+
Administrator-level local automation
+
authenticated local MCP clients
+
explicit human Control Center authority
+
strong structured-path enforcement
+
best-effort raw-shell inspection
+
audit/observability
```

### 6.2 Not designed for

```text
Hostile multi-user host
Malware resistance
Strong privilege isolation
Perfect shell containment
Automatic safety of arbitrary repository code
Tamper-proof forensic audit
```

### 6.3 Canonical security invariants

Security-sensitive implementation requirements use stable IDs.

**SEC-R3-001 — Human authorization**  
Successful R3 execution requires independent human-authorized final approval.

**SEC-R3-002 — No requester self-approval for R3**  
Requester MCP/client credentials cannot satisfy the human authorization requirement for their own R3 request.

**SEC-R3-003 — Autonomous cannot downgrade R3**  
Autonomous mode cannot convert R3 to a lower approval class.

**SEC-AUTHORITY-001 — Opaque execution is human-only**  
`opaque_indirect` execution cannot be final-approved by the requesting MCP client or Autonomous-mode self-approval. Human Control Center authorization is required.

**SEC-MODE-001 — Session reset**  
Every new Agent session starts Guarded. Active Autonomous state never survives restart/crash/relaunch.

**SEC-AUTH-001 — Loopback + authentication**  
Local HTTP MCP requires loopback-only binding and the current per-session bearer credential.

**SEC-CONTENT-001 — Content is data, not authority**  
Text/data read from workspace, Git, process output, browser/network sources or future adapters cannot itself change AgentCore policy, security mode, approval authority or credentials.

**SEC-EXEC-001 — Repository scripts are code execution**  
Known project commands do not imply safe side effects. Repository-controlled scripts/hooks/plugins execute arbitrary code under Agent privileges.

**SEC-AUDIT-001 — Operational audit boundary**  
Normal business execution does not rewrite/delete historical audit events. Only dedicated retention/archive maintenance may prune eligible audit/runtime records.

**SEC-RESOURCE-001 — Bounded admission**  
Transport requests, concurrent tool executions, managed processes, output and pending approvals are bounded.

### 6.4 Important boundary statements

- loopback ≠ authentication
- authentication ≠ sandbox
- workspace registration ≠ trusted instructions
- known executable name ≠ known behavior
- current cwd inside workspace ≠ blanket shell trust
- ProjectAdapter R1 ≠ side-effect-free execution
- R3 human approval ≠ complete protection from malicious R1 repository scripts
- audit history ≠ tamper-proof evidence against Administrator

---

## 7. Untrusted content and indirect instructions

LocalGPT expects the upstream AI client to consume content and decide future tool calls. Content may include malicious or deceptive instructions.

Potential sources:

- README/source comments
- generated files
- Git commit messages/history
- command stdout/stderr
- files outside workspace
- future browser/DOM/network output
- future external MCP output

AgentCore cannot reliably classify natural-language prompt injection. It instead enforces the authority boundary:

> **Content may inform the model, but content cannot grant Agent permission.**

Model-facing results should carry bounded provenance where practical:

```ts
type ContentProvenance = {
  source:
    | "workspace_file"
    | "external_file"
    | "git_metadata"
    | "process_output"
    | "browser_content"
    | "external_mcp"
    | "system_generated"
    | "agent_generated";
  trust: "untrusted_content" | "local_operational" | "human_authoritative";
  workspaceId?: string;
  path?: string;
};
```

Rules:

- workspace files default to `untrusted_content` for instruction authority
- Git text defaults to `untrusted_content`
- process output defaults to `untrusted_content`
- `human_authoritative` is reserved for explicit Control Center actions/approvals, not arbitrary text files
- provenance is not a sandbox and never replaces PolicyEngine

---

## 8. Risk model

### 8.1 Risk levels

| Level | Meaning | Default behavior |
|---|---|---|
| `R0 READ_ONLY` | no host state mutation | auto |
| `R1 NORMAL_WRITE_EXECUTE` | normal workspace/project coding mutation/execution | auto |
| `R2 SENSITIVE` | external/system-sensitive/uncertain operations | approval according to authority/mode |
| `R3 DESTRUCTIVE` | recognized deletion/data loss/destructive mutation | human-only |

### 8.2 Typical R2

- structured create/edit outside registered workspace
- external process termination
- registry/system configuration writes
- service/firewall/scheduled-task mutation
- system-wide install/installer
- ACL/ownership changes
- direct unknown raw-shell execution

### 8.3 Typical R3

- delete user/project files
- destructive truncation/overwrite with data loss
- `Remove-Item`, `del`, destructive `rmdir/rd`
- format/disk destructive operations
- `git reset --hard`
- `git clean -f/-fd`
- destructive restore/checkout when loss is recognized
- force operation with recognized data-loss/destructive semantics

---

## 9. Approval authority as a separate dimension

Risk and authority are independent.

```ts
type ApprovalAuthority =
  | "none"
  | "human_or_autonomous"
  | "human_only";
```

Examples:

| Request | Risk | Authority |
|---|---:|---|
| `git status` | R0 | none |
| normal workspace patch | R1 | none |
| external process stop | R2 | human_or_autonomous |
| direct unfamiliar CLI | R2 | human_or_autonomous |
| encoded/dynamic opaque shell | >=R2 | human_only |
| `git reset --hard` | R3 | human_only |

Canonical decision type:

```ts
type PolicyDecision = {
  decision: "allow" | "approval_required" | "deny";
  riskLevel: "R0" | "R1" | "R2" | "R3";
  approvalAuthority: ApprovalAuthority;
  reasons: string[];
  inspectorFlags: string[];
  workspaceBoundary: "inside" | "outside" | "not_applicable";
  destructive: boolean;
  executionTransparency?: ExecutionTransparency;
  executionTrust?: "direct" | "repository_controlled" | "opaque";
  unknownRisk?: boolean;
  contentProvenance?: ContentProvenance[];
};
```

---

## 10. Security modes and personal-use ergonomics

### 10.1 Guarded

- R0/R1 → auto according to normal tool policy
- R2 `human_or_autonomous` → human approval
- R2 `human_only` → human approval
- R3 → human approval

### 10.2 Autonomous / Unrestricted

Autonomous is explicit human opt-in for the current Agent session.

- R0/R1 unchanged
- R2 `human_or_autonomous` → authenticated MCP/client may final-allow, including requester self-approval where allowed
- R2 `human_only` → still human-only
- R3 → still human-only

Autonomous does not relabel unknown/opaque commands to R1.

### 10.3 Startup behavior

Every Agent startup:

```text
security.mode = guarded
```

No setting may automatically start new Agent sessions in Autonomous.

### 10.4 One-click resume

If previous session used Autonomous, Dashboard may show:

```text
Previous session used Autonomous / Unrestricted
[ Resume Autonomous for this session ]
```

This is a fresh explicit human action.

### 10.5 Autonomous lease

Supported convenience leases:

```text
1 hour
4 hours
Until Agent exits
```

Rules:

- explicit Control Center action required
- lease expiry returns to Guarded
- restart/crash/relaunch ends lease
- use monotonic deadline where practical so clock changes do not silently extend lease
- mode changes/expiry audited
- pending frozen `approvalAuthority` cannot be downgraded by toggling mode
- preferred lease duration may persist, active mode may not

Persistent preference may include:

```text
security.preferredAutonomousLease = 1h | 4h | until_exit
```

Forbidden in Phase 1:

```text
security.startInAutonomous = true
```

---

## 11. Raw-shell execution transparency

Raw shell is powerful and cannot be made equivalent to structured tools through lexical inspection alone.

```ts
type ExecutionTransparency =
  | "direct_known"
  | "direct_unknown"
  | "indirect_repository_controlled"
  | "opaque_indirect";
```

### 11.1 `direct_known`

Narrow command shape recognized by reviewed classifier rule.

Candidate examples:

```text
git status
git diff
git log
node --version
npm --version
```

`direct_known` does not override recognized R2/R3 semantics.

### 11.2 `direct_unknown`

Unfamiliar executable/shape, but structurally direct:

- executable identifiable
- arguments inspectable
- no encoded payload
- no dynamic evaluation
- no download-and-execute
- no meaningful hidden shell indirection

Default:

```text
risk = R2
approvalAuthority = human_or_autonomous
```

### 11.3 `indirect_repository_controlled`

Execution delegates through explicit ProjectAdapter/repository boundary, for example:

```text
npm test
npm run build
pnpm run lint
project_dev
Git hook caused by a modeled operation
```

Normal structured ProjectAdapter path may remain:

```text
risk = R1
executionTrust = repository_controlled
approvalAuthority = none
```

This is an explicit product tradeoff, not a safety claim.

### 11.4 `opaque_indirect`

Invocation materially hides/dynamically constructs meaningful behavior.

Indicators include:

- PowerShell `-EncodedCommand`
- dynamic `Invoke-Expression` / `iex`
- `irm ... | iex`
- downloaded content piped to interpreter
- dynamic/effectively opaque `node -e`, `python -c` payloads
- nested shell where meaningful payload cannot be extracted reliably
- complex evaluation/redirection/indirection hiding executed code

Default floor:

```text
risk >= R2
approvalAuthority = human_only
```

If destructive semantics are also recognized:

```text
risk = R3
approvalAuthority = human_only
```

Opacity alone does not mean R3; R3 remains semantic destructive/data-loss classification.

---

## 12. Approval workflow

Risky calls never hang an open MCP request waiting indefinitely for UI interaction.

```text
1. Client invokes tool.
2. Validate schema/context.
3. Policy decides approval is required.
4. Freeze exact tool name + arguments + policy metadata.
5. Compute canonical request hash.
6. Create approvalRequestId.
7. Original call returns APPROVAL_REQUIRED.
8. A decision arrives later through Control Center or MCP approval API.
9. ApprovalManager validates approvalAuthority + current mode + approver source.
10. Validate frozen request hash.
11. Atomically claim PENDING → EXECUTING.
12. Execute exact frozen request once.
13. Record result/audit.
```

### 12.1 State machine

```text
PENDING
   ├─ deny ───────────► DENIED
   ├─ expire ─────────► EXPIRED
   ├─ shutdown ───────► CANCELLED
   └─ authorized allow
          ▼
       EXECUTING
          ├─ success ─► COMPLETED
          └─ failure ─► FAILED
```

Terminal approvals cannot be revived.

### 12.2 Exactly-once

Approval execution must protect against:

- repeated approve calls
- concurrent UI/MCP decisions
- mutation after approval
- replay after completion
- execution after expiry/deny/cancel

Use atomic database transition equivalent to:

```sql
UPDATE approvals
SET status='EXECUTING'
WHERE id=? AND status='PENDING';
```

Only one caller wins.

### 12.3 Approval source rules

For `human_only`:

- MCP final `allow` is rejected
- request remains PENDING
- return `HUMAN_APPROVAL_REQUIRED`

For R2 `human_or_autonomous`:

- Guarded → human Control Center final allow
- Autonomous → authenticated MCP/client may final allow
- same requester may self-approve in Autonomous
- self-approval is visibly audited

Default approval expiry: **15 minutes**; configurable 5/15/30/60.

---

## 13. Path addressing and canonical path policy

This section resolves the previous ambiguity between “workspaceId is the main addressing model” and “outside-workspace reads are allowed.”

### 13.1 PathTarget

Structured file/path tools use exactly one addressing form:

```ts
type PathTarget =
  | {
      workspaceId: string;
      path: string; // relative to canonical workspace root
    }
  | {
      absolutePath: string; // explicit absolute Windows path
    };
```

Rules:

- `workspaceId + relative path` is the preferred normal form
- `absolutePath` is an explicit external/system path form
- callers must not provide both forms
- Git/project tools remain workspace-scoped and require `workspaceId`
- absolute path support does not bypass policy; it exists to make external-read/write semantics explicit and deterministic

### 13.2 Canonical resolution

`PathPolicy.resolve()` must:

1. validate requested addressing form
2. derive absolute candidate
3. normalize separators/dot segments/case representation
4. resolve symlink/junction/reparse target where applicable
5. obtain canonical target
6. compare canonical target against canonical workspace root where relevant
7. classify `inside/outside/system-sensitive`
8. apply operation policy

Must cover:

- `..` traversal
- sibling-prefix confusion (`C:\Project` vs `C:\ProjectFake`)
- case variants
- NTFS junction/reparse escape
- symlink escape
- alternate separators
- dot segments
- invalid/unresolvable paths

Conceptual API:

```ts
PathPolicy.resolve({
  target,
  operation: "read" | "write" | "delete"
})
```

### 13.3 Path matrix

| Location | Read | Create/Edit | Delete/Destructive overwrite |
|---|---|---|---|
| inside workspace | allow | allow | R3 human |
| outside workspace | allow | R2 | R3 human |
| system-sensitive path | OS ACL permitting | R2+ | R3/higher by semantics |

Structured path policy is stronger than shell inference because AgentCore knows the explicit target before execution.

---

## 14. Workspace model

```ts
type Workspace = {
  id: string;
  name: string;
  rootPath: string;
  canonicalRootPath: string;
  projectType: string;
  packageManager?: string;
  gitRoot?: string;
  createdAt: string;
  updatedAt: string;
  lastOpenedAt?: string;
  enabled: boolean;
};
```

Workspace registration grants project scope, not instruction trust.

`workspace_snapshot` is a logical summary, not backup.

Nested/duplicate roots should be detected and surfaced during registration rather than silently creating ambiguous policy scope.

---

## 15. Workspace Command Rules

Phase 1 does not implement blanket “Trusted Workspace = unrestricted shell.”

Instead, repeated `direct_unknown` friction may be reduced with narrow, human-reviewed workspace-specific rules.

```ts
type WorkspaceCommandRule = {
  id: string;
  workspaceId: string;
  shell: "powershell" | "cmd" | "any";
  commandShape: string;
  classification: "direct_known";
  reviewedRiskCeiling: "R0" | "R1" | "R2";
  enabled: boolean;
  createdAt: string;
  updatedAt: string;
  note?: string;
};
```

Rules may recognize narrow shapes such as:

```text
internal-cli status <bounded-args>
internal-cli inspect <bounded-args>
internal-cli --version
```

Rules must never:

- convert `opaque_indirect` to direct known
- bypass `SEC-AUTHORITY-001`
- lower recognized R3
- authorize encoded/dynamic/download-and-execute behavior
- grant trust to arbitrary repository scripts
- match all PowerShell/Node/scripts in a workspace
- auto-create from approval history

Runtime stronger-risk detection always wins.

Rule creation/edit/delete is an explicit audited Control Center action.

---

## 16. Guarded-friction dogfood

Personal-use safety fails if Guarded becomes so noisy that the user always enables Autonomous merely to work around classifier errors.

Phase 1 dogfood therefore measures friction locally.

Recommended 7-day metrics:

```text
direct_unknown_rate
direct_unknown_human_allow_rate
direct_unknown_self_approve_rate
opaque_indirect_rate
opaque_indirect_human_allow_rate
repeat_approved_direct_unknown_shape_count
autonomous_activation_count
autonomous_resume_count
autonomous_lease_expiry_count
http_session_credential_copy_count
workspace_rule_candidate_count
workspace_rule_match_count
```

Rules:

- metrics are local-only by default
- repeated approval is evidence for manual review, not proof of safety
- no auto-learning that turns “approved N times” into trusted R1
- tune `direct_unknown` first
- treat `opaque_indirect` primarily as a security signal
- use normalized/redacted command shapes instead of raw secret-bearing commands

Candidate `direct_known` promotion requires human review + regression test.

---

# Part II — Phase 1 architecture and tools

## 17. Top-level architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Electron Desktop Control Center                                      │
│                                                                      │
│ React Renderer                                                       │
│ Dashboard │ Projects │ Git │ Activity │ Processes                   │
│ Approvals │ Tunnel │ Doctor │ Settings                              │
│                 │                                                    │
│                 │ typed narrow preload IPC                           │
│                 ▼                                                    │
│ AgentCore                                                            │
│ ├─ McpGateway                                                        │
│ │  ├─ localhost HTTP → SessionTokenAuth                              │
│ │  └─ Named Pipe RPC target for stdio bridge                         │
│ ├─ ToolRegistry / ToolDispatcher                                     │
│ ├─ PolicyEngine                                                      │
│ │  ├─ PathPolicy                                                     │
│ │  ├─ ShellInspector                                                 │
│ │  ├─ SecurityModePolicy                                             │
│ │  ├─ ExecutionTrustPolicy                                           │
│ │  └─ WorkspaceCommandRulePolicy                                     │
│ ├─ ApprovalManager                                                   │
│ ├─ ResourceGovernor                                                  │
│ ├─ Execution                                                        │
│ │  ├─ Workspace/File                                                 │
│ │  ├─ Git                                                            │
│ │  ├─ ProjectAdapter                                                 │
│ │  ├─ Shell                                                          │
│ │  └─ ProcessManager                                                 │
│ ├─ DoctorService                                                     │
│ ├─ AuditRetentionService                                             │
│ ├─ EventBus                                                          │
│ │  ├─ AuditWriter → SQLite                                           │
│ │  └─ UiBroadcaster → Renderer                                       │
│ └─ repositories/persistence                                          │
└──────────────────────────────────────────────────────────────────────┘

MCP client
   │ stdio
   ▼
mcp-stdio-bridge
   │ framed local RPC
   ▼
current-user Windows Named Pipe
   │
   ▼
AgentCore
```

### 17.1 Permanent architecture rules

- transports contain no tool business logic
- tool implementations do not decide their own permissions
- renderer is never policy authority
- renderer never opens SQLite/filesystem/shell directly
- every future transport/adaptor reuses dispatcher/policy/approval/resource/audit path
- ResourceGovernor is admission control, not permission authority
- Doctor observes/diagnoses; it is not a second execution stack

---

## 18. Standard request lifecycle

```text
HTTP only: validate bearer token
↓
transport admission/backpressure
↓
receive request
↓
assign toolCallId
↓
runtime schema validation
↓
resolve client/session/workspace/path context
↓
attach content/execution provenance
↓
PolicyEngine classification
↓
audit policy decision
↓
ALLOW → ResourceGovernor → Execution
or
APPROVAL_REQUIRED → freeze/hash → ApprovalManager → return approvalRequestId
or
DENY → stable error
↓
record result/failure
↓
MCP response
```

Authentication failures do not create executable tool calls and never echo submitted token.

---

## 19. Phase 1 MCP tool catalog

Phase 1 intentionally keeps the catalog small and coherent.

### 19.1 Workspace

#### `workspace_list`
R0. Lists registered workspaces.

#### `workspace_get`
R0. Gets one workspace.

#### `workspace_register`
R1/CONTROL. Canonicalizes root, detects project metadata, checks duplicate/nested roots.

#### `workspace_update`
R1/CONTROL.

#### `workspace_unregister`
R2 by current mode. Removes registry metadata only; never project files.

#### `workspace_tree`
R0. Bounded tree with depth/result limits.

#### `workspace_snapshot`
R0. Bounded project/tree/Git summary.

### 19.2 File/search

#### `read_file`

- R0
- accepts `PathTarget`
- bounded by line/byte range
- binary returns metadata/unsupported rather than unbounded decode
- returned model-facing text carries `ContentProvenance`

#### `search_text`

- R0
- preferred workspace form; explicit absolute root allowed for external read
- bounded results
- include/exclude glob, case and max result controls
- implementation may use ripgrep internally

#### `apply_patch`

Primary text create/edit tool.

- workspace text create/edit → R1
- external structured write → R2
- file deletion/destructive truncation/recognized destructive overwrite → R3 human-only
- validate all targets before modification
- compute edits in memory where practical
- temp + atomic replace per file where practical
- multi-file atomicity not guaranteed; partial results explicit

#### `copy_file`

Structured file copy added to Phase 1 so agents do not need raw shell for ordinary copy operations.

Conceptual input:

```ts
{
  source: PathTarget;
  destination: PathTarget;
  overwrite?: boolean; // default false
}
```

Rules:

- source uses read policy
- destination inside workspace, new file → R1
- destination outside workspace, new file → R2
- `overwrite=false` and destination exists → stable conflict error
- `overwrite=true` when an existing destination would be replaced → R3 human-only because of recognized data-loss semantics
- directory recursive copy is not required Phase 1
- binary files are supported without reading their content into model context
- canonicalize/validate both source and destination before copy
- audit source/destination boundaries without leaking file content

No generic unrestricted `write_file` is required Phase 1.

### 19.3 Git inspection

#### `git_status`
R0.

#### `git_diff`
R0; bounded; returned text is untrusted `git_metadata`.

#### `git_log`
R0; bounded; commit messages are untrusted content.

Phase 1 does not add generic `git_run(subcommand)` solely to increase tool count. Git mutation may use raw shell and therefore passes ShellInspector/PolicyEngine.

Dedicated safe Git mutations may be added later only when they improve semantics/UX enough to justify a structured tool.

### 19.4 Node/TypeScript ProjectAdapter

#### `project_info`
R0. Detects package manager/scripts/monorepo hints/Node/Git root/capabilities.

#### `project_dev`
R1 by explicit ProjectAdapter contract. Starts managed background process.

#### `test`
#### `lint`
#### `typecheck`
#### `build`

- R1 by ProjectAdapter contract
- execute only detected configured scripts
- absent script → `PROJECT_SCRIPT_NOT_FOUND`
- do not invent a command silently
- mark `executionTrust="repository_controlled"`
- bounded stdout/stderr, exit code, duration, actual command

### 19.5 Shell

#### `shell`

```ts
{
  command: string;
  workspaceId?: string;
  cwd?: string;
  shell?: "powershell" | "cmd";
  env?: Record<string, string>;
  timeoutMs?: number;
  mode?: "foreground" | "background";
}
```

- raw Administrator PowerShell/cmd
- no command allowlist sandbox
- best-effort semantic inspection
- `direct_unknown` → R2
- `opaque_indirect` → >=R2 + human-only
- known destructive semantics → R3
- absolute external cwd allowed but audited as external
- env audit records keys only; values matching secret heuristics are redacted

Foreground returns bounded stdout/stderr/exit/duration.

Background returns Agent process ID + Windows PID + command/cwd/status.

### 19.6 Processes

#### `process_list`
Agent-managed processes by default.

#### `process_get`
Metadata/state.

#### `process_output`
Bounded incremental stdout/stderr with cursor/offset semantics.

#### `process_stop`
- managed process → CONTROL/auto
- external process → normally R2; destructive semantics may elevate

#### `process_restart`
Managed process only using frozen launch metadata.

### 19.7 Approvals

#### `approval_list`
Pending/recent approvals.

#### `approval_get`
Frozen request + risk/authority/mode/expiry/requester.

#### `approval_decide`

```ts
{
  approvalRequestId: string;
  decision: "allow" | "deny";
  note?: string;
}
```

MCP allow is accepted only when approval authority/mode permits it.

### 19.8 Operations

#### `health`
Bounded machine-readable Agent/SQLite/transport/process/resource summary.

#### `system_info`
Bounded coding-relevant Windows/CPU/RAM/disk/Node/Git info.

Doctor remains richer Control Center service rather than broad privileged MCP diagnostic surface in Phase 1.

---

## 20. Internal ToolDescriptor contract

Phase 1 does not need MCP-visible tool discovery yet, but ToolRegistry should own a consistent internal descriptor for every tool because the same metadata is already required for validation/policy/UI.

Conceptually:

```ts
type ToolDescriptor = {
  name: string;
  group: string;
  description: string;
  inputSchemaId: string;
  outputSchemaId: string;
  maturity: "phase1" | "future";
  defaultRisk: "R0" | "R1" | "dynamic";
  supportsApproval: boolean;
  modelFacingContent: boolean;
};
```

This is not a speculative Phase 2 discovery service. It is the canonical metadata owned by `ToolRegistry`.

When catalog size grows later, Phase 2A exposes on-demand discovery from this registry instead of duplicating schemas.

---

## 21. Result/error contracts

```ts
type ToolResult<T> =
  | { ok: true; toolCallId: string; data: T }
  | {
      ok: false;
      toolCallId: string;
      error: {
        code: string;
        message: string;
        details?: unknown;
      };
    };
```

Required stable errors include:

**Authentication/transport**

- `AUTH_REQUIRED`
- `AUTH_INVALID`
- `HTTP_BIND_NOT_LOOPBACK`
- `PORT_IN_USE`
- `AGENT_NOT_RUNNING`
- `RATE_LIMITED`
- `TOO_MANY_CONCURRENT_REQUESTS`

**Validation/path**

- `INVALID_ARGUMENT`
- `UNKNOWN_TOOL`
- `INVALID_WORKSPACE`
- `INVALID_PATH`
- `PATH_TARGET_CONFLICT`
- `DESTINATION_EXISTS`

**Policy/approval**

- `APPROVAL_REQUIRED`
- `POLICY_DENIED`
- `HUMAN_APPROVAL_REQUIRED`
- `SELF_APPROVAL_NOT_ALLOWED`
- `AUTONOMOUS_MODE_REQUIRED`
- `APPROVAL_EXPIRED`
- `APPROVAL_ALREADY_DECIDED`
- `REQUEST_HASH_MISMATCH`
- `DIRECT_UNKNOWN_REQUIRES_R2`
- `OPAQUE_INDIRECT_REQUIRES_HUMAN`

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
- `FILE_COPY_FAILED`
- `PATCH_FAILED`
- `GIT_FAILED`
- `PROJECT_SCRIPT_NOT_FOUND`

**System**

- `DATABASE_ERROR`
- `INTERNAL_ERROR`
- `AGENT_SHUTTING_DOWN`

---

## 22. SQLite persistence

Use per-user app data such as:

```text
%LOCALAPPDATA%\LocalGPT Agent\
├─ agent.db
├─ logs\
├─ exports\
└─ runtime\
```

Exact location derives from Electron/Windows app-data APIs.

Requirements:

- SQLite WAL
- foreign keys
- bounded busy timeout
- migrations from first release
- renderer never opens DB
- repository layer owns persistence

### 22.1 Core logical tables

#### `workspaces`

```text
id
name
root_path
canonical_root_path
project_type
package_manager
git_root
created_at
updated_at
last_opened_at
is_enabled
```

#### `sessions`

```text
id
started_at
ended_at
app_version
windows_user
windows_session_id
elevated
http_auth_method
http_auth_enabled
security_mode
mode_lease_kind
mode_lease_started_at
mode_lease_expires_at
shutdown_reason
status
```

New session always initializes `security_mode='guarded'`.

#### `clients`

```text
id
session_id
name
transport
authenticated
auth_method
connected_at
last_seen_at
disconnected_at
metadata_json
```

#### `tool_calls`

```text
id
session_id
client_id
workspace_id
tool_name
arguments_json
arguments_hash
risk_level
approval_authority
policy_decision
policy_reasons_json
inspector_flags_json
execution_transparency
execution_trust
content_provenance_json
classification_shape
inspector_rule_version
status
requested_at
execution_started_at
execution_finished_at
duration_ms
result_summary_json
error_code
error_message
```

#### `approvals`

```text
id
tool_call_id UNIQUE
requester_client_id
approver_client_id
approver_kind
approval_source
risk_level
approval_authority
reasons_json
frozen_arguments_json
request_hash
status
created_at
expires_at
decided_at
execution_started_at
completed_at
decision_note
human_authorized
self_approved
autonomous_mode
```

Human-only successful execution requires human-authorized source. MCP allow cannot convert `human_only` request to executable state.

#### `processes`

```text
id
session_id
workspace_id
pid
parent_pid
process_start_timestamp
owner_type
command
cwd
shell_type
execution_trust
status
started_at
exited_at
exit_code
restart_count
created_by_tool_call_id
```

Agent process ID is primary identity; PID is metadata.

#### `workspace_command_rules`

```text
id
workspace_id
shell_type
command_shape
rule_json
reviewed_risk_ceiling
enabled
created_at
updated_at
note
```

Malformed/unsupported rules fail closed and Doctor reports them.

#### `audit_events`

```text
id
session_id
tool_call_id
client_id
workspace_id
event_type
severity
payload_json
created_at
```

#### `settings`

Persistent settings may include:

```text
mcp.http.enabled
mcp.http.port
mcp.http.authMode = session_bearer
mcp.stdio.enabled
mcp.preferredLocalTransport = stdio
shell.default
shell.defaultTimeoutMs
approval.expirationMinutes
process.shutdownGraceMs
security.preferredAutonomousLease
resource-governance limits
retention controls
```

Plaintext HTTP token and active `security.mode` are not persistent restoration settings.

---

## 23. Output storage and bounds

Process output:

```text
runtime\processes\<processId>\stdout.log
runtime\processes\<processId>\stderr.log
```

SQLite stores chunk/index metadata, not one unbounded row.

Foreground MCP response cap default:

```text
stdout 256 KB
stderr 256 KB
```

Return truncation metadata when larger.

No tree/search/file/process response may be unbounded.

---

## 24. Audit and retention

Minimum event vocabulary includes:

```text
SESSION_STARTED
CLIENT_CONNECTED
HTTP_AUTH_REJECTED
SECURITY_MODE_CHANGED
AUTONOMOUS_LEASE_STARTED
AUTONOMOUS_LEASE_EXPIRED
AUTONOMOUS_LEASE_CANCELLED
TOOL_REQUEST_RECEIVED
POLICY_DECIDED
DIRECT_UNKNOWN_CLASSIFIED
OPAQUE_INDIRECT_CLASSIFIED
APPROVAL_CREATED
APPROVAL_ALLOW_REJECTED
APPROVAL_APPROVED
APPROVAL_DENIED
APPROVAL_EXPIRED
APPROVAL_CANCELLED
EXECUTION_STARTED
EXECUTION_SUCCEEDED
EXECUTION_FAILED
PROCESS_STARTED
PROCESS_EXITED
PROCESS_STOP_REQUESTED
WORKSPACE_REGISTERED
WORKSPACE_UPDATED
WORKSPACE_UNREGISTERED
WORKSPACE_COMMAND_RULE_CREATED
WORKSPACE_COMMAND_RULE_UPDATED
WORKSPACE_COMMAND_RULE_DISABLED
WORKSPACE_COMMAND_RULE_DELETED
WORKSPACE_COMMAND_RULE_MATCHED
DOCTOR_RUN_COMPLETED
RESOURCE_ADMISSION_REJECTED
AUDIT_RETENTION_RUN
AGENT_SHUTDOWN_STARTED
AGENT_SHUTDOWN_COMPLETED
```

Secret/token values never enter audit payloads.

### 24.1 Retention defaults

```text
Active audit events: 30 days
Full process output: 7 days
Manual audit export: JSONL
```

`AuditRetentionService` is the only application path allowed to prune eligible historical operational data.

It never deletes workspace/source/user files.

---

## 25. Resource governance

Default Phase 1 limits:

```text
Global concurrent tool executions: 8
Concurrent executions per MCP client: 4
Agent-managed process limit: 32
Pending approval limit: 100
Authenticated HTTP admission: 120 req/min per client session, burst 30
```

Requirements:

- bounded queues
- deterministic stable error on saturation
- output bounds independent of request-rate limits
- failed auth throttled/backed off without logging token
- process start fails closed when cap reached
- risky approval creation fails closed when queue full
- Doctor/UI show pressure/limits

---

## 26. MCP transports

### 26.1 Preferred local integration

```text
Persistent editor/desktop MCP integration
→ stdio bridge
→ current-user Named Pipe
→ AgentCore
```

The bridge contains no filesystem/shell/policy/SQLite logic.

If AgentCore is not running: `AGENT_NOT_RUNNING`.

### 26.2 Localhost HTTP

Endpoint concept:

```text
http://127.0.0.1:<port>/mcp
```

Requirements:

- loopback only
- no `0.0.0.0`
- bearer auth every request
- random >=256-bit-equivalent session token
- memory-only plaintext secret
- token rotates every Agent session
- old token invalid after restart
- no permissive browser CORS convenience mode
- bounded request admission

### 26.3 Static token non-goal

Phase 1 does not add one static global bearer token.

If a future important local client cannot use stdio or refresh session credentials, design **paired-client credentials** separately with:

- per-client identity
- explicit pairing
- revocation
- rotation
- Windows-appropriate secure storage
- audit-safe identity

This is deferred.

---

## 27. Electron Control Center

Renderer security:

- `nodeIntegration=false`
- `contextIsolation=true`
- renderer sandbox where compatible
- narrow typed preload APIs
- no generic arbitrary IPC invoke

Navigation:

```text
Dashboard
Projects / Workspaces
Git
Activity / Live Logs
Processes
Approvals
Tunnel
Doctor
Settings
```

### 27.1 Dashboard

Show:

- Agent state/uptime
- elevation/Admin
- Guarded vs Autonomous
- Autonomous lease/countdown
- one-click resume when contextually useful
- Return to Guarded
- raw-shell status
- HTTP auth health without token
- stdio bridge health
- client/process/approval counts
- resource pressure
- recent activity

### 27.2 Projects / Workspaces

Show workspace metadata and security notice:

> Project access does not make repository text/scripts trusted instructions or sandboxed code.

Include Workspace Command Rules panel and candidate shapes from local dogfood metrics.

### 27.3 Git

Operational view only:

- branch
- status
- bounded diff
- recent log
- changed files
- link/filter into Activity

No IDE editor.

### 27.4 Activity / Live Logs

Filters include:

- time/workspace/client/tool
- risk/status
- transparency/trust
- approval authority
- workspace command-rule match

Detail shows policy/audit-safe args/provenance/result timeline.

### 27.5 Processes

Managed-process metadata + bounded output + stop/restart.

### 27.6 Approvals

Show:

- frozen request
- requester
- workspace
- risk
- approval authority
- reasons
- transparency/trust
- current security mode
- expiry
- self-approval eligibility
- human-only requirement

For `opaque_indirect`:

```text
OPAQUE / INDIRECT EXECUTION
HUMAN APPROVAL REQUIRED
Autonomous eligibility: No
```

For R3:

```text
DESTRUCTIVE ACTION
HUMAN APPROVAL REQUIRED
```

### 27.7 Tunnel

Phase 1 shows unavailable/not configured status only.

Release 1.1 activates remote transport/identity/health.

### 27.8 Settings

Expose validated operational settings, preferred stdio transport guidance and preferred Autonomous lease.

Do not expose:

- start-in-Autonomous
- static global HTTP token
- blanket trust entire workspace shell

---

## 28. Doctor

Doctor returns `PASS`, `WARN`, `FAIL` with actionable explanations and no secrets.

Checks include:

- AgentCore health
- elevation
- session security mode
- Autonomous lease state
- SQLite/migrations/WAL
- audit retention
- HTTP auth/loopback/port
- named pipe
- stdio bridge
- workspace accessibility
- malformed/broad Workspace Command Rules
- Git/Node/npm/pnpm/yarn
- PowerShell/cmd
- ProcessManager
- ResourceGovernor
- writable app/runtime dirs
- stale/crashed session
- missing dependencies

Doctor may show Guarded Friction metrics as diagnostics, not as automatic policy changes.

---

## 29. Startup/shutdown lifecycle

Startup:

```text
Electron starts elevated
↓
single-instance lock
↓
app directories
↓
SQLite + migrations
↓
recover previous crashed session
↓
create new Agent session with Guarded
↓
generate fresh HTTP token
↓
repositories/EventBus
↓
PolicyEngine
↓
ResourceGovernor
↓
ToolRegistry/Dispatcher
↓
Named Pipe RPC
↓
authenticated HTTP MCP
↓
Doctor state
↓
renderer ready
↓
RUNNING
```

Shutdown:

```text
SHUTTING_DOWN
↓
reject new executions
↓
cancel pending approvals according to policy
↓
bounded foreground grace
↓
stop Agent-managed background processes
↓
leave external processes untouched
↓
flush audit/state
↓
stop transports
↓
invalidate HTTP token
↓
discard Autonomous lease/state
↓
close SQLite
↓
STOPPED
```

Default shutdown grace: **5 seconds**.

Crash recovery does not auto-adopt orphaned processes whose identity cannot be safely re-established.

---

# Part III — Verification and Phase 1 delivery

## 30. Testing strategy

Required pyramid:

```text
Static / Typecheck
Unit
Contract
Integration
Security regression
Electron E2E
Packaged Windows smoke
```

### 30.1 PathPolicy tests

- inside/outside
- PathTarget mutually exclusive forms
- traversal
- case variants
- sibling prefix
- junction/symlink/reparse escape
- invalid/unresolvable path
- external read
- external write classification

### 30.2 Shell/ExecutionTrust tests

```text
git status                         → direct_known
unknown direct CLI                 → direct_unknown + R2 + human_or_autonomous
EncodedCommand                     → opaque_indirect + >=R2 + human_only
Invoke-Expression dynamic          → opaque_indirect + human_only
download | interpreter             → opaque_indirect + human_only
git reset --hard                   → R3 + human_only
npm test via ProjectAdapter        → R1 + indirect_repository_controlled
```

### 30.3 ApprovalManager tests

- R2 Guarded human allow
- R2 Guarded MCP/self allow reject
- R2 Autonomous eligible self allow
- opaque Guarded MCP allow reject
- opaque Autonomous MCP allow reject
- opaque human allow
- R3 MCP allow reject
- R3 human allow
- hash mismatch
- replay
- concurrent double approval
- expiry
- shutdown cancellation
- exactly once

### 30.4 SecurityMode tests

- every new session Guarded
- explicit UI action required for Autonomous
- 1h/4h lease expiry returns Guarded
- until-exit ends on shutdown
- crash/restart never restores active Autonomous
- preferred lease only preselects UI

### 30.5 File/copy tests

- read/search/patch
- `copy_file` inside workspace
- binary copy
- external source read + workspace destination
- external destination → R2
- existing destination + overwrite false → conflict
- overwrite true → R3
- canonical validation before copy

### 30.6 Workspace Command Rule tests

- narrow direct-unknown rule becomes direct-known only in same workspace
- same CLI elsewhere remains unknown
- encoded/opaque indicators override rule
- R3 indicators override rule
- malformed/broad rule fails closed
- approval history cannot auto-create rule
- CRUD audited

### 30.7 HTTP/transport tests

- no token reject
- wrong token reject
- current token success
- restart rotates token
- old token invalid
- token absent SQLite/audit/log/Doctor/normal renderer state
- explicit copy only
- non-loopback bind reject
- stdio bridge reconnect works across Agent restart without editing HTTP token

### 30.8 Persistence tests

- migrations/WAL/FK
- Guarded session initialization
- approval authority persisted frozen
- Autonomous state not restored
- token absent persisted data
- retention safe boundaries
- workspace rules fail closed if malformed

### 30.9 Resource tests

- per-client/global concurrency
- process cap
- approval cap
- rate/burst
- stable errors
- no policy bypass under saturation

### 30.10 Security regressions

Every permission/auth/trust-boundary bug receives regression coverage.

Dedicated suite covers:

- path escape
- destructive Git
- shell quoting/aliases/indirection
- external writes
- approval replay/mutation
- human-only authority
- token leakage
- content provenance
- repository-controlled execution
- resource exhaustion
- workspace-rule bypass attempts

---

## 31. Phase 1 roadmap

### Phase 1.1 — Foundation

Deliver:

- Electron/React/TS shell
- elevation + single instance
- SQLite/migrations
- session lifecycle
- Guarded initialization
- EventBus/logger
- typed preload foundation
- basic Dashboard

### Phase 1.2 — Workspace/File Core

Deliver:

- WorkspaceRegistry
- canonical `PathTarget` + PathPolicy
- ProjectDetector
- ContentProvenance
- workspace tools
- `read_file`
- `search_text`
- `apply_patch`
- **`copy_file`**

Acceptance includes external read semantics and copy overwrite/R2/R3 behavior.

### Phase 1.3 — Git + Node ProjectAdapter

Deliver:

- Git inspection
- project metadata
- test/lint/typecheck/build/dev
- package-manager detection
- repository-controlled execution metadata

### Phase 1.4 — Shell + Processes

Deliver:

- raw shell
- four-class ExecutionTransparency
- ShellInspector
- ProcessManager
- bounded output
- timeout/restart/shutdown

### Phase 1.5 — Policy + Approvals + Personal Ergonomics

Deliver:

- risk classifier
- first-class ApprovalAuthority
- SecurityModePolicy
- Autonomous lease/resume
- frozen hash/exactly-once
- Workspace Command Rules
- dogfood classifier metrics

### Phase 1.6 — Transports + ResourceGovernor

Deliver:

- HTTP per-session auth
- stdio bridge/Named Pipe
- preferred local transport guidance
- rate/concurrency/process/approval limits

### Phase 1.7 — Operational Control Center + Doctor

Deliver all Control Center pages and structured diagnostics.

### Phase 1.8 — Hardening/release gate

Deliver:

- all test layers
- security regressions
- retention/export
- Electron E2E
- packaged elevated Windows smoke
- dogfood friction baseline

---

## 32. Phase 1 Core Definition of Done

Packaged Windows release gate:

```text
1. Launch elevated Control Center.
2. Verify new session Guarded.
3. Verify Doctor baseline.
4. Connect persistent local client through stdio bridge.
5. Restart Agent and verify stdio integration needs no HTTP token editing.
6. Verify HTTP token rotated and old token fails.
7. Register fixture workspace.
8. snapshot/read/search.
9. Verify content provenance.
10. apply_patch.
11. copy_file new destination.
12. verify overwrite conflict and R3 overwrite path.
13. test/lint/typecheck/build.
14. verify repository_controlled execution metadata.
15. Git status/diff/log.
16. project_dev + process_output.
17. direct_unknown shell in Guarded → R2 human approval.
18. requester MCP allow rejected in Guarded.
19. human allow executes exact frozen request once.
20. explicitly activate bounded Autonomous lease.
21. eligible direct_unknown/ordinary R2 self-approval works and is audited.
22. opaque_indirect in Autonomous still rejects MCP final allow.
23. human opaque approval succeeds exactly once.
24. R3 request rejects requester MCP allow.
25. human R3 approval succeeds exactly once.
26. replay/double approval cannot re-execute.
27. create narrow workspace command rule from reviewed direct_unknown shape.
28. verify rule is workspace-scoped.
29. verify opaque/R3 indicators override rule.
30. exercise resource limits without unauthorized execution.
31. verify Activity/Live Logs/Audit timeline.
32. verify token absent persisted/logged/Doctor state.
33. verify retention defaults and safe cleanup.
34. close app and verify managed process cleanup.
35. restart and verify Guarded + new token.
```

Release 1.1 remote work starts only after this gate is stable.

---

# Part IV — Post-Core roadmap informed by the larger reference capability set

## 33. Release 1.1 — Secure Remote Access

Remote access is immediate post-Core milestone because it is central to the reference product direction but materially changes the trust boundary.

```text
Remote Client
↓
Secure Remote Transport
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

- no parallel remote policy stack
- real remote identity/auth design
- local HTTP session token is not remote identity
- remote human approval identity is explicit
- R3 and opaque human-only rules remain
- remote-specific rate/concurrency abuse controls
- Tunnel UI + Doctor diagnostics
- threat-model delta before shipping

Exact provider/protocol is deferred.

---

## 34. Phase 2A — Developer Intelligence Foundation

This phase adopts the most valuable architecture ideas from the larger reference catalog **before** adding broad agent orchestration.

### 34.1 Tool Catalog and on-demand discovery

Problem when tool count grows:

```text
large catalog
→ schema/context overhead
→ model sees irrelevant tools
→ discovery cost and duplicate wrappers
```

Add a discovery surface backed by canonical `ToolRegistry` descriptors, conceptually:

```text
capabilities
tool_search
tool_describe
tool_group_list
```

Goals:

- do not require loading every full schema up front
- return compact capability summaries first
- fetch exact schema on demand
- primitive/base tools remain available according to client integration contract
- discovery output is bounded/paged
- discovery never changes permission; discovered tools still pass normal policy pipeline

### 34.2 Large-file paging

Add explicit cursor/continuation behavior for very large files/results:

```text
read_page
continue_cursor
```

or equivalent extension to `read_file`.

Must preserve bounds and path policy.

### 34.3 Workspace Index + watcher

Add persistent/project-local index metadata for faster context retrieval.

Responsibilities:

- file inventory
- normalized symbols metadata where available
- incremental file-change watch
- index freshness/status
- invalidate/rebuild safely

The index is derived data, never policy authority.

### 34.4 Context Engine

Add ranked/paged project context retrieval so the AI does not need repeated ad-hoc file reads.

Conceptual outputs:

```text
top relevant files
relevant snippets
related symbols
Git/project metadata
reason/rank signals
continuation cursor
```

Content remains untrusted under `SEC-CONTENT-001`.

### 34.5 Code Intelligence

Use language-aware services/LSP where practical for:

- definition
- references
- implementations
- symbols
- call hierarchy
- type/symbol search
- dependency graph

Do not pretend text grep equals semantic code intelligence.

### 34.6 Smart task-specific context

Add purpose-specific context entry points for:

- debug
- code review
- test change
- frontend
- backend
- architecture/repository overview

These should compose Index + Code Intelligence + Git context rather than become dozens of unrelated implementations.

### 34.7 Git Review Intelligence

Add higher-level analysis inputs:

- blame/history
- changed symbol mapping
- affected modules
- related commits
- review context

Mutation still uses the same policy pipeline.

### 34.8 Testing Intelligence

Add:

- test-suite discovery
- changed-file → likely impacted tests mapping
- run targeted tests
- result/coverage context where supported

### 34.9 Context Economy

Treat “context economy” as an engineering objective, not a quota guarantee.

Use:

- on-demand tool schemas
- bounded/paged results
- deduplication
- index-aware ranking
- cache of derived metadata
- stable cursors

Do not claim provider quota is unlimited.

### 34.10 Phase 2A DoD

A client can ask for relevant project context and semantic code relationships without manually reading most of the repository, while all returned content remains bounded, attributable and untrusted for authority.

---

## 35. Phase 2B — Agentic Development

### 35.1 `codex_run`

Delegate to local Codex CLI through existing workspace/process/policy/audit/resource infrastructure.

No second process manager.

Delegated output is untrusted process/agent content and cannot self-elevate permissions.

### 35.2 Browser Debug Adapter

Go beyond a single primitive `dom_cdp` tool family by providing a coherent browser debugging adapter over CDP.

Capabilities may include:

- navigate
- DOM query/inspection
- click/type/forms
- console capture
- network request/response metadata
- JS evaluation
- screenshots
- correlated DOM/console/network/screenshot debug context

Browser/page content is untrusted.

### 35.3 Additional ProjectAdapters

Add Python/.NET/Rust according to actual usage priority.

Each adapter declares:

- detected commands
- execution trust
- bounded output
- capability availability

### 35.4 Phase 2B security gate

Any new delegation/browser/code-execution surface requires threat-model delta + regression updates before shipping.

---

## 36. Phase 2C — Workflow, Managed Tasks and Delegation

This phase introduces composition only after single-tool lifecycle and developer intelligence are stable.

### 36.1 Managed Tasks

A managed task is durable orchestration state, not merely a long MCP request.

Conceptually:

```ts
type ManagedTask = {
  id: string;
  workspaceId?: string;
  objective: string;
  status: "queued" | "running" | "blocked" | "completed" | "failed" | "cancelled";
  createdAt: string;
  updatedAt: string;
};
```

### 36.2 `tool_batch`

Support composition with:

- ordered/dependent steps
- DAG dependencies
- bounded parallelism
- timeout
- cancellation
- partial results

Critical invariant:

> **Every child tool invocation still enters ToolDispatcher → PolicyEngine → ApprovalManager → ResourceGovernor.**

Batch approval never grants blanket permission to all children.

### 36.3 Agent Delegation

Add worker/agent delegation only through Managed Tasks.

Requirements:

- explicit worker identity/audit
- bounded concurrency
- shared policy authority
- no worker bypass of R2/R3/opaque authority
- task cancellation propagation
- partial result tracking

### 36.4 Persistent Handoff

Do not overload runtime `sessions` table for AI work continuation.

Introduce separate handoff/checkpoint concept:

```ts
type TaskHandoff = {
  id: string;
  workspaceId: string;
  objective: string;
  checkpointSummary: string;
  completedSteps: string[];
  pendingSteps: string[];
  relevantPaths: string[];
  gitStateRef?: string;
  processRefs?: string[];
  createdAt: string;
};
```

Handoff text is context, not policy authority.

### 36.5 Recipes / preview

Reusable workflow recipes may be introduced with preview of planned child tool calls before execution where useful.

Recipes describe intended orchestration but do not bypass child policy classification.

### 36.6 Hooks

Hooks/events may trigger Managed Tasks after explicit human configuration.

Avoid hidden arbitrary background execution. Hook definitions must have clear owner, trigger, enabled state, bounds and audit.

---

## 37. Extension ecosystem — deferred, security-reviewed

Reference materials include Skills, Plugin System and External MCP Discovery. LocalGPT intentionally does **not** prioritize these before core policy/orchestration is mature.

Future extension model must define at least:

- namespace/origin
- declared capabilities
- tool manifest/schema
- identity/version
- permission mapping
- content provenance
- ResourceGovernor limits
- audit
- enable/disable/revocation

External MCP chaining must not mean “remote tool discovered → trusted automatically.”

A connected external MCP capability is a new trust/content/execution boundary requiring policy mapping.

Skills/plugin/external-MCP work is optional after Phase 2C and requires its own threat-model delta.

---

## 38. Phase 3 — Full Windows Desktop Agent

Phase 3 expands beyond coding/browser workflows.

### 38.1 UI Automation

Prefer Microsoft UI Automation semantic controls first.

### 38.2 Window management

list/activate/move/resize/minimize/maximize/close, with unsaved/destructive semantics classified.

### 38.3 Vision + input fallback

```text
UI Automation
↓ if unavailable
Vision
↓
Keyboard/Mouse
```

Coordinate clicking is fallback, not default.

### 38.4 Clipboard + file dialogs

Bounded/audited automation.

### 38.5 Office

Structured Word/Excel COM adapters first; avoid unrestricted generic COM invocation initially.

### 38.6 Visual / Excel / PDF validation

Add inspection-oriented features where useful:

- screenshot/layout comparison
- Excel preview/layout validation
- rendered document comparison
- PDF page render/compare

These are validation capabilities, not an excuse to duplicate specialized document tooling into the core AgentCore.

### 38.7 Screen capture/recording

Bound duration/storage and show visible privacy state.

### 38.8 Windows diagnostics

Expand Doctor/diagnostics selectively into:

- service/process tree
- ports
- PATH/runtime
- registry inspection
- Event Log
- startup configuration

Mutation remains policy-gated and should not be conflated with read diagnostics.

### 38.9 Notifications/scheduler

Scheduled-task mutation is at least R2 and may be higher by semantics.

### 38.10 `web_fetch`

Bound protocol/timeout/size/credential/local-network policy. Fetched content is untrusted.

### 38.11 Audio

Explicit visible privacy state and approval rules.

### 38.12 Phase 3 security gate

Every new OS/UI/Office/vision/network/audio capability requires threat-model delta + regression coverage.

---

# Part V — Capability comparison and intentional differences

## 39. Capability maturity map against the large reference catalog

The reference source describes a mature broad capability set. LocalGPT maps it intentionally by maturity rather than tool count.

### 39.1 Already in Phase 1 Core

- Workspace & Files
- File Search
- Git inspection
- Project commands
- Process management
- Raw shell
- Permission/Governance
- Logs/Activity
- Doctor/Windows coding diagnostics
- local MCP transports
- bounded output

### 39.2 Immediate post-Core

- Secure Remote Access / Tunnel

### 39.3 Phase 2A

- Tool Schema & Discovery
- Large-file paging
- Workspace Index/watch
- Context Engine
- Code Intelligence
- Smart Context
- Git Review Intelligence
- Testing Intelligence
- Context Economy/cache of derived context metadata
- Planning/Repository Intelligence

### 39.4 Phase 2B

- Codex delegation
- advanced Browser/UI debugging via CDP
- more project adapters

### 39.5 Phase 2C

- Managed Tasks
- Multi-tool workflow / `tool_batch`
- dependency/parallel execution
- Agent Delegation
- Persistent Handoff
- Recipes
- Hooks

### 39.6 Deferred extension ecosystem

- Skills
- Skill intelligence
- Plugin System
- External MCP Discovery/chaining

### 39.7 Phase 3

- Windows UI/vision/input
- Clipboard/file dialogs
- Office
- visual/screenshot/Excel/PDF validation
- deeper Windows diagnostics
- scheduler/notification
- web fetch
- audio/recording

---

## 40. Intentional differences where LocalGPT design is preferred

### 40.1 Fewer, stronger Phase 1 tools

LocalGPT does not create a generic tool for every Git subcommand or command family when raw shell already provides capability and PolicyEngine can classify it.

Structured tools are added when they materially improve:

- deterministic target knowledge
- policy correctness
- result schema
- auditability
- common UX

`copy_file` satisfies this test, so it is added.

A generic `git_run(any)` does not currently justify itself.

### 40.2 Workspace-scoped structured addressing

Normal structured project operations use `workspaceId`, while explicit absolute `PathTarget` handles external file reads/writes under deterministic policy.

This is more policy/audit-friendly than treating arbitrary absolute paths as equivalent to workspace paths.

### 40.3 Human-only R3 and opaque execution

LocalGPT keeps human authorization for recognized destructive actions and truly opaque execution even in Autonomous mode.

This deliberately favors predictable privileged execution over maximum autonomy.

### 40.4 Guarded reset with cheap explicit re-entry

LocalGPT does not persist active Autonomous across restart, but makes re-entry inexpensive through one-click resume/leases.

### 40.5 Per-session HTTP token + stdio default

LocalGPT keeps short-lived HTTP secrets and solves persistent-client friction by preferring stdio, instead of using one global static bearer credential.

### 40.6 Security-reviewed remote/plugin growth

Remote transport, plugin systems and external MCP chaining expand attack surface significantly. They are phased after core invariants and require threat-model deltas rather than being treated as ordinary tool additions.

---

## 41. Cross-phase invariants

Permanent unless explicitly redesigned:

1. one ToolDispatcher lifecycle
2. one PolicyEngine/ApprovalManager authority
3. one ResourceGovernor model
4. one audit/event model
5. renderer remains non-privileged UI
6. structured paths are canonicalized before mutation
7. raw shell is not advertised as sandboxed
8. project scripts are not advertised as safe merely because command name is known
9. content does not become authority
10. R3 remains human-only
11. opaque_indirect remains human-only
12. active Autonomous never restores automatically across local Agent sessions
13. future transports/adapters do not create parallel policy stacks
14. every new attack-surface milestone requires a threat-model delta + regressions
15. tool discovery/composition never grants permission by discovery/batching alone
16. child calls in workflows/delegation remain independently policy-classified
17. audit retention authority never grants deletion rights over workspace/user data

---

## 42. Decisions deferred to implementation planning

Behavioral/security requirements above are fixed. Implementation choices remain open where they do not change those contracts:

- Electron/Node versions
- schema validation library
- SQLite driver
- test framework
- bundler/package layout
- UI component library
- Named Pipe framing format
- exact ShellInspector parser/rule representation
- exact Workspace Command Rule matcher representation
- exact Phase 2A indexing backend
- exact LSP/language-service integration
- exact cache/index storage format
- exact remote tunnel provider/protocol
- exact paired-client credential mechanism if later needed
- exact future plugin signing/origin mechanism
- exact browser CDP library

---

## 43. Final success statement

LocalGPT Agent succeeds when it behaves as a local, elevated Windows Agent Gateway that an AI client can use for real coding work while remaining:

- **powerful** — normal coding work is low-friction
- **observable** — every important action has Activity/Audit visibility
- **predictable** — structured tools have deterministic path/policy semantics
- **bounded** — output, concurrency, processes and approvals have limits
- **honest about trust** — raw shell/project scripts/content are not described as sandboxed
- **human-controlled where it matters** — R3 and opaque indirect execution remain human-only
- **extensible without architectural sprawl** — later context, browser, tasks, delegation and desktop capabilities reuse the same core pipeline

The long-term objective is not “184 tools.”

The objective is a Windows-first Agent Gateway where a growing capability catalog remains discoverable, context-efficient, governable, auditable and composable without weakening the core security model.
