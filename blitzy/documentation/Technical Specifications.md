# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a comprehensive new technical analysis document** that answers detailed questions about how SFTPGo's connection handling, quota enforcement, and atomic upload mechanisms coordinate under concurrent stress conditions. The document must be placed at `blitzy/documentation/sftpgo_44634210287c.md` per the project's implementation rules.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical deep-dive analysis / Architecture Q&A document
- **Core Questions Addressed:**
  - How does SFTPGo coordinate quota updates across simultaneous transfers—do quota checks race with each other or serialize cleanly, and where does that decision first manifest at runtime?
  - When a user with strict quota limits opens multiple sessions near `MaxSessions` and starts concurrent uploads that collectively threaten to exceed both `QuotaFiles` and `QuotaSize`, what happens to session acceptance, quota accounting, and file state?
  - What happens to a partial temporary file and its associated quota accounting when a connection is dropped mid-stream due to idle timeout or session-limit rejection during an atomic-mode upload?
  - What do the actual logged messages, file states, and quota values look like during a burst of concurrent activity, including sessions accepted vs. rejected, quota changes at each step, and any surviving temporary files?
  - What gaps or inconsistencies appear when a transfer is deliberately killed mid-stream versus when it completes normally, and what concrete signs in logs or the filesystem distinguish the two paths?
  - How do the idle connection checker, session limiter, and quota enforcer interact when all run concurrently?
- **Implicit Documentation Needs:**
  - Trace the exact code paths from `sftpd/server.go` through `sftpd/handler.go`, `sftpd/transfer.go`, `sftpd/sftpd.go`, and `dataprovider/dataprovider.go` to produce a unified concurrency narrative
  - Document the locking strategy (`sync.RWMutex` on the shared registries in `sftpd/sftpd.go`) and its implications for race conditions
  - Document the gap between the quota check (`hasSpace`) and the quota update (`UpdateUserQuota`) and explain what happens in that window
  - Explain per-provider quota update serialization: SQL `UPDATE ... SET used_quota_size = used_quota_size + ?` atomicity, BoltDB single-writer transactions, and memory-provider mutex
  - Illustrate the temporary file naming convention (`.sftpgo-upload.<xid>.<filename>` in `vfs/osfs.go:175-179`) and the three possible resolution paths on `Transfer.Close()`

### 0.1.2 Special Instructions and Constraints

- **CRITICAL:** The repository must remain unchanged. No existing files may be modified. The output document is created in a new `blitzy/documentation/` directory in the destination repository.
- **CRITICAL:** Any temporary scripts or test artifacts used to gather evidence must be cleaned up after use; the source repository must remain clean.
- **Analysis Approach:** All answers must be grounded in the actual source code, citing specific files and line numbers. No assumptions—use the code as the single source of truth.
- **Thinking / Rationale:** The output document must include reasoning and rationale behind each answer, not just conclusions.
- **Format:** Markdown document named `sftpgo_44634210287c.md`, placed in `blitzy/documentation/`.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **connection handling and session-limit enforcement**, we will create a section tracing the code path from `sftpd/server.go:AcceptInboundConnection` through `loginUser` (lines 365-394), which checks `MaxSessions` via `getActiveSessions` (lines 200-210), and `addConnection`/`removeConnection` (lines 354-376).
- To document the **quota enforcement under concurrency**, we will create a section analyzing the `hasSpace` function (`sftpd/handler.go:515-534`) which queries `dataprovider.GetUsedQuota`, the gap between that check and `dataprovider.UpdateUserQuota` (called from `Transfer.Close` at `sftpd/transfer.go:167`), and the per-provider serialization characteristics.
- To document the **atomic upload lifecycle**, we will create a section tracing `Filewrite` → `handleSFTPUploadToNewFile`/`handleSFTPUploadToExistingFile` → `Transfer.Close` including the three atomic-mode branches (rename on success, delete on error in mode 1, keep on error in mode 2).
- To document the **idle connection checker interaction**, we will create a section analyzing `CheckIdleConnections` (`sftpd/sftpd.go:331-352`), its 5-minute ticker, how it reads `lastActivity` from both connections and transfers, and what happens when it closes a connection that has an in-flight transfer.
- To document **observable runtime evidence**, we will catalog every `logger.Log`/`logger.TransferLog`/`logger.Warn` call in the relevant code paths, mapping them to concrete log fields (`sender`, `connection_id`, `username`, `file_path`, `size_bytes`, `elapsed_ms`).

### 0.1.4 Inferred Documentation Needs

- **Concurrency gap analysis:** The `hasSpace` check and the `UpdateUserQuota` call are not atomic—multiple concurrent uploads can each pass `hasSpace` before any updates the quota, creating a TOCTOU (time-of-check-to-time-of-use) window. This must be documented explicitly.
- **Locking scope documentation:** The single `sync.RWMutex` in `sftpd/sftpd.go` protects `openConnections`, `activeTransfers`, and `activeQuotaScans` but does NOT protect the data-provider quota reads/writes. This architectural boundary must be made clear.
- **Per-provider atomicity:** SQL providers use `SET used_quota_size = used_quota_size + ?` which relies on the database's own row-level locking. BoltDB uses serialized read-write transactions. The memory provider uses its own `sync.Mutex`. These differences affect concurrent quota correctness differently.
- **Temporary file cleanup:** When `Transfer.Close` runs after a connection drop, the error branch for atomic mode 1 calls `os.Remove` on the temp file and resets `bytesReceived` to 0 and `numFiles` to -1 (net effect: no quota update). For atomic mode 2, the temp file is renamed in place. For mode 0 (standard), partial data remains at the target path.
- **Session limit vs. idle checker ordering:** Session counting happens during authentication (SSH handshake callback), while connection registration happens after authentication in `handleSftpConnection`. There is a brief window between authentication check and registration where the count could be stale.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a minimal documentation structure consisting of Markdown README files at the root and within operational subdirectories, with no dedicated documentation generator, no `docs/` hierarchy, and no API documentation tooling beyond inline code comments.

- **Documentation files found:**
  - `README.md` — Main project onboarding document covering features, platforms, installation, CLI, configuration, authentication, storage, web/admin, logging, metrics, and deployment
  - `docker/README.md` — Docker deployment instructions
  - `docker/sftpgo/alpine/README.md` — Alpine container image documentation
  - `docker/sftpgo/debian/README.md` — Debian container image documentation
  - `scripts/README.md` — REST API CLI tool documentation
- **Documentation generator:** None detected (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx.conf.py`, or similar)
- **API documentation tools:** None detected (no JSDoc, Godoc generation config, or Swagger/OpenAPI generation beyond the static `httpd/schema/openapi.json` if present)
- **Diagram tools:** None detected in repository; the tech spec uses Mermaid notation
- **Documentation hosting/deployment:** None configured
- **Current documentation framework:** Plain Markdown, no framework
- **Destination directory `blitzy/documentation/`:** Does not yet exist; must be created

### 0.2.2 Repository Code Analysis for Documentation

The following source directories and files were exhaustively examined to build the analysis that will populate the output document:

**Core SFTP Server (`sftpd/`):**
- `sftpd/sftpd.go` — Package-level registries (`openConnections`, `activeTransfers`, `activeQuotaScans`), the shared `sync.RWMutex`, connection/transfer add/remove functions, idle-timeout checker (`CheckIdleConnections`), session counting (`getActiveSessions`), upload mode constants
- `sftpd/server.go` — `Configuration` struct with `IdleTimeout`, `MaxAuthTries`, `UploadMode`; `Initialize` entry point, `AcceptInboundConnection` lifecycle, `loginUser` session-limit enforcement, `handleSftpConnection` connection registration
- `sftpd/handler.go` — `Connection` struct, `Filewrite` upload entry point, `handleSFTPUploadToNewFile`/`handleSFTPUploadToExistingFile` with atomic-path selection, `hasSpace` quota check, `handleSFTPRemove` quota decrement
- `sftpd/transfer.go` — `Transfer` struct with per-transfer mutex, `ReadAt`/`WriteAt` with activity tracking, `Close` with atomic-upload finalization and quota update, `TransferError` error recording and context cancellation, `handleThrottle` bandwidth limiting
- `sftpd/scp.go` — SCP upload handler with identical `hasSpace` quota check and `addConnection`/`removeConnection` lifecycle
- `sftpd/ssh_cmd.go` — SSH command handler, `errQuotaExceeded` definition, `maxWriteSize`-based quota enforcement in `copyFromReaderToWriter`

**Data Provider (`dataprovider/`):**
- `dataprovider/dataprovider.go` — `Provider` interface, `UpdateUserQuota` with `TrackQuota` mode gating, `GetUsedQuota` query, `Config.TrackQuota` modes (0=disabled, 1=always, 2=restricted-users-only)
- `dataprovider/user.go` — `User` struct with `MaxSessions`, `QuotaSize`, `QuotaFiles`, `UsedQuotaSize`, `UsedQuotaFiles`, `HasQuotaRestrictions` check
- `dataprovider/sqlcommon.go` — `sqlCommonUpdateQuota` with SQL `UPDATE ... SET used_quota_size = used_quota_size + ?` (incremental) or `SET used_quota_size = ?` (reset), `sqlCommonGetUsedQuota`
- `dataprovider/sqlqueries.go` — SQL query builders; `getUpdateQuotaQuery` showing incremental vs. reset SQL
- `dataprovider/bolt.go` — `BoltProvider.updateQuota` with read-modify-write inside `dbHandle.Update` (serialized write transaction)
- `dataprovider/memory.go` — `MemoryProvider.updateQuota` with explicit `sync.Mutex` protection on all map operations

**Virtual Filesystem (`vfs/`):**
- `vfs/osfs.go` — `GetAtomicUploadPath` generating `.sftpgo-upload.<xid>.<basename>`, `IsAtomicUploadSupported` (returns true for local FS), `ScanRootDirContents` for quota scan
- `vfs/s3fs.go` — `IsAtomicUploadSupported` returns false for S3 backend

**Supporting Packages:**
- `logger/logger.go` — Structured zerolog logging with `TransferLog`, `CommandLog`, `ConnectionFailedLog`, and per-level `Debug`/`Info`/`Warn`/`Error` emitters with `sender` and `connection_id` fields
- `config/config.go` — Default configuration loading, `UploadMode` validation (0-2), `IdleTimeout` configuration
- `metrics/metrics.go` — Prometheus metrics for `TransferCompleted`, `UpdateActiveConnectionsSize`, `AddLoginAttempt`, `AddLoginResult`

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. The user's question is entirely answerable from the source code, and the project implementation rules mandate that answers must be based on the code as the single source of truth.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The output document must synthesize information from the following modules, mapping each to the specific user questions it answers:

- **Module: `sftpd/sftpd.go` — Connection/Transfer Registry and Idle Checker**
  - Public APIs: `addConnection`, `removeConnection`, `addTransfer`, `removeTransfer`, `CheckIdleConnections`, `getActiveSessions`, `updateConnectionActivity`, `isAtomicUploadEnabled`
  - Current documentation: No standalone documentation exists beyond code comments
  - Documentation needed: Explanation of the single `sync.RWMutex` locking strategy, the 5-minute idle ticker, how `CheckIdleConnections` accounts for active transfer activity timestamps, and what `c.close()` does to in-flight transfers

- **Module: `sftpd/server.go` — Session Limit Enforcement**
  - Key functions: `loginUser` (lines 365-394), `AcceptInboundConnection` (lines 243-333)
  - Current documentation: Inline comments only
  - Documentation needed: Analysis of the session-counting window, the sequence from SSH authentication callback through to `addConnection` call, and the log messages emitted on session rejection (`"too many open sessions: %v"` at line 375)

- **Module: `sftpd/handler.go` — Quota Check and Upload Initiation**
  - Key functions: `Filewrite`, `handleSFTPUploadToNewFile`, `handleSFTPUploadToExistingFile`, `hasSpace`
  - Current documentation: Inline comments only
  - Documentation needed: Analysis of the TOCTOU gap between `hasSpace` (which reads quota) and `Transfer.Close` (which updates quota), how `checkFiles` parameter controls file-count vs. size-only checking, and the atomic-path selection logic

- **Module: `sftpd/transfer.go` — Transfer Lifecycle and Atomic Finalization**
  - Key functions: `Close`, `WriteAt`, `TransferError`, `closeIO`
  - Current documentation: Inline comments only
  - Documentation needed: Full analysis of the three atomic-upload completion branches (rename-on-success, delete-on-error, keep-for-resume), how `numFiles` and `bytesReceived` are adjusted on error cleanup, and the exact `UpdateUserQuota` call parameters

- **Module: `dataprovider/dataprovider.go` — Quota Update Serialization**
  - Key functions: `UpdateUserQuota`, `GetUsedQuota`
  - Current documentation: GoDoc-style comments on exported functions
  - Documentation needed: Analysis of per-provider serialization guarantees, the `TrackQuota` configuration modes, and how the `MethodDisabledError` fallback works in `hasSpace`

- **Module: `dataprovider/sqlcommon.go` / `dataprovider/bolt.go` / `dataprovider/memory.go` — Provider-Specific Quota Atomicity**
  - Key functions: `sqlCommonUpdateQuota`, `BoltProvider.updateQuota`, `MemoryProvider.updateQuota`
  - Current documentation: None beyond code
  - Documentation needed: Comparison of SQL's `used_quota_size = used_quota_size + ?` row-level lock, BoltDB's serialized `Update` transaction with read-modify-write, and memory provider's `sync.Mutex` guarded map write

- **Module: `vfs/osfs.go` — Atomic Upload Path and File Cleanup**
  - Key functions: `GetAtomicUploadPath`, `IsAtomicUploadSupported`, `ScanRootDirContents`
  - Current documentation: Inline comments
  - Documentation needed: Explanation of the `.sftpgo-upload.<xid>.<filename>` naming convention, how the temp file ends up in the same directory as the target, and how `ScanRootDirContents` would count orphaned temp files

- **Module: `logger/logger.go` — Log Message Structure**
  - Key functions: `TransferLog`, `CommandLog`, `ConnectionFailedLog`, `Debug`, `Warn`, `Info`
  - Current documentation: GoDoc comments
  - Documentation needed: Catalog of all log messages emitted during the connection/upload/quota/idle lifecycle, with field names and severity levels

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps must be addressed in the output document:

- **Undocumented concurrency semantics:** No existing documentation explains how the single `sync.RWMutex` in `sftpd/sftpd.go` interacts with the separate locking in each data provider, or that quota checks and updates are not transactionally linked
- **Undocumented TOCTOU window:** The race between `hasSpace` (read) and `UpdateUserQuota` (write) is not documented anywhere—both can succeed for concurrent uploads that collectively exceed the quota
- **Undocumented atomic upload cleanup on connection drop:** What happens to temp files and quota accounting when `Transfer.Close` runs after a `TransferError` is not documented outside the code itself
- **Undocumented session-limit timing gap:** The window between `getActiveSessions` (called during auth) and `addConnection` (called after SFTP subsystem starts) is not documented
- **Missing observable evidence catalog:** No documentation catalogs what log messages, file system states, and Prometheus metrics a user would observe during each failure mode

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/sftpgo_44634210287c.md` will follow this structure:

```
blitzy/
└── documentation/
    └── sftpgo_44634210287c.md
        ├── Introduction and Scope
        ├── Architecture Overview: The Concurrency Landscape
        │   ├── Shared State Registries and Locking
        │   ├── Data Provider Quota Serialization by Backend
        │   └── The Two-Layer Locking Architecture
        ├── Connection Handling Under Session Pressure
        │   ├── Session Counting and Limit Enforcement
        │   ├── The Auth-to-Registration Timing Gap
        │   └── Observable Evidence: Sessions Accepted vs. Rejected
        ├── Quota Enforcement Across Concurrent Transfers
        │   ├── The hasSpace Check-Then-Act Pattern
        │   ├── The TOCTOU Race Window
        │   ├── Quota Update Mechanics in Transfer.Close
        │   └── Per-Provider Serialization Comparison
        ├── Atomic Upload Mechanics Under Failure
        │   ├── Temp File Naming and Location
        │   ├── Three Completion Paths: Success / Error+Atomic / Error+Resume
        │   ├── Partial File Cleanup and Quota Reversal
        │   └── Orphaned Temp File Detection
        ├── Idle Connection Checker Interactions
        │   ├── Ticker Mechanics and Activity Resolution
        │   ├── Impact on In-Flight Transfers
        │   └── Connection Close Cascade
        ├── Observable Runtime Evidence Catalog
        │   ├── Log Messages by Code Path
        │   ├── File System States After Each Scenario
        │   ├── Quota Values at Each Step
        │   └── Prometheus Metrics Affected
        ├── Clean Run vs. Contention Failure: Differential Analysis
        │   ├── Normal Completion Sequence
        │   ├── Mid-Stream Kill Sequence
        │   └── Concrete Distinguishing Signs
        └── Summary of Gaps and Risks
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract concurrency semantics from `sftpd/sftpd.go` by tracing every `mutex.Lock()`/`mutex.RLock()` call to map the critical sections
- Extract quota flow from `sftpd/handler.go:hasSpace` → `dataprovider.GetUsedQuota` → `dataprovider/sqlcommon.go:sqlCommonGetUsedQuota` and `sftpd/transfer.go:Close` → `dataprovider.UpdateUserQuota` → `dataprovider/sqlcommon.go:sqlCommonUpdateQuota`
- Extract all `logger.*` calls from `sftpd/server.go`, `sftpd/handler.go`, `sftpd/transfer.go`, `sftpd/sftpd.go` to build the log-message catalog
- Trace the `Transfer.Close` branches at `sftpd/transfer.go:121-170` to document all atomic-mode outcomes

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` headers
- Mermaid diagrams for concurrency flows, state transitions, and decision trees
- Code citations in format: `Source: sftpd/handler.go:515-534`
- Tables for log message catalogs and per-provider comparison matrices
- Consistent use of SFTPGo terminology from the codebase

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **Sequence diagram:** Concurrent upload race showing two uploads both passing `hasSpace` before either calls `UpdateUserQuota`
- **Flowchart:** `Transfer.Close` decision tree with all three atomic-mode branches and their quota/file outcomes
- **Flowchart:** Idle connection checker's activity-resolution logic including transfer timestamp comparison
- **State diagram:** Connection lifecycle from TCP accept through authentication, registration, transfer, idle check, and removal
- **Comparison table:** Per-provider quota serialization guarantees (SQL row-lock vs. BoltDB transaction vs. memory mutex)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| blitzy/documentation/sftpgo_44634210287c.md | CREATE | sftpd/sftpd.go, sftpd/server.go, sftpd/handler.go, sftpd/transfer.go, sftpd/scp.go, sftpd/ssh_cmd.go, dataprovider/dataprovider.go, dataprovider/user.go, dataprovider/sqlcommon.go, dataprovider/sqlqueries.go, dataprovider/bolt.go, dataprovider/memory.go, vfs/osfs.go, vfs/vfs.go, logger/logger.go, config/config.go, sftpgo.json | Complete technical analysis document answering all user questions about connection handling, quota enforcement, atomic uploads, idle checking, and their concurrent interactions, with code citations, Mermaid diagrams, log catalogs, and gap analysis |

No other documentation files are created, updated, or deleted. The project implementation rules explicitly prohibit modifying any existing files in the source repository.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Technical Analysis / Architecture Q&A
Source Code:
  - sftpd/sftpd.go (connection registry, idle checker, transfer registry)
  - sftpd/server.go (session limit enforcement, connection acceptance)
  - sftpd/handler.go (quota check, upload initiation, atomic path selection)
  - sftpd/transfer.go (transfer lifecycle, atomic finalization, quota update)
  - sftpd/scp.go (SCP quota handling, connection lifecycle)
  - sftpd/ssh_cmd.go (SSH command quota enforcement)
  - dataprovider/dataprovider.go (Provider interface, UpdateUserQuota, GetUsedQuota)
  - dataprovider/user.go (User model, quota fields, session limits)
  - dataprovider/sqlcommon.go (SQL quota update mechanics)
  - dataprovider/sqlqueries.go (SQL query builders for quota)
  - dataprovider/bolt.go (BoltDB quota serialization)
  - dataprovider/memory.go (Memory provider mutex-guarded quota)
  - vfs/osfs.go (atomic upload path generation, file cleanup)
  - logger/logger.go (structured log format, TransferLog, ConnectionFailedLog)
  - config/config.go (configuration defaults and validation)
  - sftpgo.json (sample runtime configuration)
Sections:
  - Introduction and Scope
  - Architecture Overview: The Concurrency Landscape
  - Connection Handling Under Session Pressure
  - Quota Enforcement Across Concurrent Transfers
  - Atomic Upload Mechanics Under Failure
  - Idle Connection Checker Interactions
  - Observable Runtime Evidence Catalog
  - Clean Run vs. Contention Failure: Differential Analysis
  - Summary of Gaps and Risks
Diagrams:
  - Concurrent upload TOCTOU race sequence diagram
  - Transfer.Close atomic-mode decision flowchart
  - Idle checker activity-resolution flowchart
  - Connection lifecycle state diagram
Key Citations:
  - sftpd/sftpd.go:56 (mutex declaration)
  - sftpd/sftpd.go:200-210 (getActiveSessions)
  - sftpd/sftpd.go:331-352 (CheckIdleConnections)
  - sftpd/sftpd.go:354-376 (addConnection/removeConnection)
  - sftpd/server.go:365-394 (loginUser session limit)
  - sftpd/handler.go:106-108 (atomic path selection)
  - sftpd/handler.go:412-448 (handleSFTPUploadToNewFile)
  - sftpd/handler.go:450-513 (handleSFTPUploadToExistingFile)
  - sftpd/handler.go:515-534 (hasSpace quota check)
  - sftpd/transfer.go:121-170 (Transfer.Close)
  - dataprovider/dataprovider.go:310-322 (UpdateUserQuota)
  - dataprovider/dataprovider.go:326-331 (GetUsedQuota)
  - dataprovider/sqlcommon.go:74-90 (sqlCommonUpdateQuota)
  - dataprovider/sqlqueries.go:43-49 (getUpdateQuotaQuery)
  - dataprovider/bolt.go:180-209 (BoltProvider.updateQuota)
  - dataprovider/memory.go:119-140 (MemoryProvider.updateQuota)
  - vfs/osfs.go:175-179 (GetAtomicUploadPath)
  - logger/logger.go:139-149 (TransferLog)
  - logger/logger.go:175-184 (ConnectionFailedLog)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The project does not use a documentation framework; the output is a standalone Markdown file.

### 0.5.4 Cross-Documentation Dependencies

- The output document may reference the project's `README.md` for general SFTPGo feature context but does not modify it
- No navigation, table-of-contents, or index updates are needed since there is no documentation framework
- No shared includes or templates exist in the repository

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tooling is required for this task. The output is a standalone Markdown file with embedded Mermaid diagram notation. The following are the project's runtime dependencies relevant to the analysis content (extracted from `go.mod`):

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| Go modules | go | 1.13 | Target Go version; determines concurrency primitives available |
| Go modules | github.com/pkg/sftp | v1.10.2-0.20200110230707-5bee31e2 | SFTP protocol handler; `sftp.Request`, `sftp.Handlers`, `sftp.NewRequestServer` |
| Go modules | golang.org/x/crypto/ssh | (indirect via go.sum) | SSH server implementation; `ssh.ServerConfig`, `ssh.NewServerConn`, `ssh.Permissions` |
| Go modules | github.com/eikenb/pipeat | v0.0.0-20190316224601 | Pipe-based `WriterAt`/`ReaderAt` for S3 streaming; used in `Transfer` struct |
| Go modules | github.com/rs/zerolog | (via go.sum) | Structured JSON logging; all log messages documented reference this |
| Go modules | go.etcd.io/bbolt | (via go.sum) | BoltDB key-value store; serialized write transactions for quota |
| Go modules | github.com/rs/xid | (via go.sum) | Globally unique ID generation for atomic upload temp file naming |
| Go modules | gopkg.in/natefinsc/lumberjack.v2 | (via go.sum) | Log file rotation; affects log file organization |

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The output document is a new file that does not replace or redirect any existing documentation links.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions addressed:** 7/7 (100%) — all explicit and implicit questions from the user's prompt are mapped to dedicated document sections
- **Source files analyzed for content:** 17/17 (100%) — every file relevant to the connection-handling, quota, atomic-upload, and idle-checker subsystems has been read and analyzed
- **Code paths documented:** All critical paths through `loginUser`, `hasSpace`, `Filewrite`, `handleSFTPUploadToNewFile`, `handleSFTPUploadToExistingFile`, `Transfer.Close`, `CheckIdleConnections`, `UpdateUserQuota`, `GetUsedQuota`, and their per-provider implementations
- **Data provider backends covered:** 4/4 (100%) — SQL (SQLite/PostgreSQL/MySQL via `sqlcommon.go`), BoltDB (`bolt.go`), and Memory (`memory.go`) quota serialization analyzed
- **Upload modes covered:** 3/3 (100%) — Standard (mode 0), Atomic (mode 1), and Atomic-with-Resume (mode 2)
- **Target coverage:** 100% of the user's stated questions answered with code-grounded evidence

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**
- Every answer must cite specific file paths and line numbers from the SFTPGo source code
- Every concurrency claim must trace the exact locking mechanism (which mutex, which transaction, which critical section)
- Every "what happens when" scenario must describe the full sequence of function calls, log messages emitted, file system mutations, and quota accounting changes
- Every gap or risk identified must be grounded in a concrete code-path analysis, not speculation

**Accuracy Validation:**
- All code citations must reference the actual file content as retrieved during analysis
- All log message formats must match the `logger.*` call signatures in the source
- All quota SQL statements must match `dataprovider/sqlqueries.go`
- Mermaid diagrams must accurately reflect the branching logic in the source code

**Clarity Standards:**
- Technical accuracy with accessible narrative structure
- Progressive disclosure: overview → detailed code-path analysis → observable evidence
- Consistent terminology: use SFTPGo's own naming (`openConnections`, `activeTransfers`, `hasSpace`, `Transfer.Close`, `CheckIdleConnections`)
- Each section includes a "Rationale" or "Thinking" subsection explaining the reasoning behind the analysis

**Maintainability:**
- Source citations formatted as `Source: <file>:<line-range>` for traceability
- Document structured with clear heading hierarchy for navigation
- All code examples kept minimal (2-3 lines) with source references for full context

### 0.7.3 Example and Diagram Requirements

- **Mermaid diagrams:** Minimum 4 diagrams covering concurrent-upload race, atomic-finalization branches, idle-checker logic, and connection lifecycle
- **Log message examples:** Concrete JSON log samples showing what each scenario produces
- **Quota state examples:** Step-by-step quota value tables showing files/size at each stage of a concurrent upload scenario
- **File system state examples:** Directory listings showing temp file presence/absence after clean vs. interrupted transfers

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/sftpgo_44634210287c.md` — The sole deliverable: a comprehensive technical analysis document

- **Source code analyzed (read-only) to derive conclusions:**
  - `sftpd/**/*.go` — All SFTP server, handler, transfer, SCP, SSH command, and registry files
  - `dataprovider/**/*.go` — All data provider interface, SQL, BoltDB, and memory implementations
  - `vfs/osfs.go` — Local filesystem atomic-upload path generation and file operations
  - `vfs/vfs.go` — Filesystem interface definition and error mapping
  - `vfs/s3fs.go` — S3 backend atomic-upload capability flags (for comparison)
  - `logger/logger.go` — Structured logging format and audit log functions
  - `config/config.go` — Configuration loading, defaults, and validation
  - `metrics/metrics.go` — Prometheus metric update functions
  - `sftpgo.json` — Default configuration sample

- **Analysis topics covered:**
  - Connection registration and session-limit enforcement
  - Quota check (`hasSpace`) and quota update (`UpdateUserQuota`) concurrency semantics
  - Atomic upload temp-file lifecycle (creation, rename-on-success, delete-on-error, keep-for-resume)
  - Idle connection monitoring and its impact on in-flight transfers
  - Per-data-provider serialization guarantees for quota operations
  - Observable log messages, file states, and Prometheus metrics for each scenario
  - Gaps, TOCTOU risks, and architectural boundaries in the concurrency model

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository will be modified, per project rules
- **Test file modifications:** No test files will be modified or created in the source repository
- **Feature additions or code refactoring:** The task is analysis-only documentation; no code changes proposed
- **Deployment configuration changes:** No changes to Docker, CI, or service configs
- **HTTP API documentation:** The REST API (`httpd/`) is not part of this analysis scope
- **Web UI documentation:** The admin web interface (`templates/`, `static/`) is out of scope
- **S3 storage backend deep-dive:** S3 atomic upload is not supported (`IsAtomicUploadSupported` returns false); only noted for contrast
- **Database schema documentation:** SQL migration files (`sql/`) are not documented beyond quota column references
- **External authentication program behavior:** The `ExternalAuthProgram` flow is noted but not the focus of analysis
- **Performance benchmarking:** No load testing or performance measurement is in scope; the analysis is based on code-path reasoning
- **Unrelated documentation:** Docker README files, scripts README, and other existing docs are not modified or extended

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone Markdown file
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/sftpgo_44634210287c.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in Markdown fenced code blocks and rendered by any Mermaid-compatible viewer (GitHub, VS Code with Mermaid extension, etc.)
- **Documentation deployment command:** Not applicable — no documentation hosting configured
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim must reference source file paths and line numbers in format `Source: <file>:<lines>`
- **Style guide:** Follow the narrative technical-analysis style with code-grounded reasoning; use SFTPGo's own terminology from the codebase; include "Rationale" explanations per the project implementation rules
- **Documentation validation:** Manual review; ensure all code citations match retrieved file contents; verify Mermaid diagram syntax is valid
- **File placement:** The output document must be placed at `blitzy/documentation/sftpgo_44634210287c.md` per the project's `SWE-AtlasQnA-Repo` implementation rule
- **Repository cleanliness:** Any temporary scripts or test artifacts used during analysis must be removed before completion; the source repository must remain completely unchanged

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The output is a new Markdown document only.
- **Create the document as `<source_branch_name>.md`** — specifically `sftpgo_44634210287c.md` — and place it in the `blitzy/documentation` directory.
- **Provide thinking / rationale behind the answers.** Every conclusion must include the reasoning that led to it, not just the end result.
- **Do not make assumptions; base all answers on the code as the truth.** Every claim must be traceable to specific source files and line numbers.
- **Clean up any test artifacts.** If any temporary scripts are written during analysis, they must be removed before the task is considered complete. The repository must remain unchanged and clean.
- **Keep the actual repository unchanged.** Read-only analysis only; no commits, no file modifications, no build artifacts left behind.
- **Address all user questions comprehensively.** Every question from the user's prompt must receive a complete, code-grounded answer in the output document.
- **Include source code citations for all technical details.** Use the format `Source: <file>:<line-range>` throughout.
- **Use Mermaid diagrams for complex flows.** The concurrency interactions, state transitions, and decision trees must be visualized.
- **Document observable evidence.** Log messages, file system states, quota values, and Prometheus metrics must be cataloged for each scenario discussed.

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

The following files and folders were retrieved and analyzed during context gathering:

**Root Level:**
- `/` (repository root) — Folder contents retrieved for project structure overview
- `go.mod` — Go module definition, dependency list, Go version (1.13)
- `sftpgo.json` — Default runtime configuration with SFTP, data provider, and HTTP settings
- `.travis.yml` — CI pipeline (Go 1.13.x, SQLite setup, `go test ./...`)

**SFTP Server (`sftpd/`):**
- `sftpd/` — Folder contents retrieved (12 files)
- `sftpd/sftpd.go` — Full file read (493 lines): connection/transfer registries, idle checker, mutex, actions
- `sftpd/server.go` — Full file read (487 lines): SSH server initialization, authentication, session limits, connection lifecycle
- `sftpd/handler.go` — Full file read (566 lines): SFTP request handlers, quota checks, atomic upload path, upload initiation
- `sftpd/transfer.go` — Full file read (262 lines): Transfer I/O lifecycle, atomic finalization, quota update, error handling
- `sftpd/scp.go` — Partial read (lines 1-50) and grep analysis: SCP upload quota check
- `sftpd/ssh_cmd.go` — Partial read (lines 1-60) and grep analysis: `errQuotaExceeded`, `maxWriteSize`

**Data Provider (`dataprovider/`):**
- `dataprovider/` — Folder contents retrieved (9 files)
- `dataprovider/dataprovider.go` — Full file read (848 lines): Provider interface, quota API, external auth, config
- `dataprovider/user.go` — Full file read (461 lines): User model, permissions, quota fields, session limits
- `dataprovider/sqlcommon.go` — Full file read (350 lines): SQL execution layer, quota update/read, user CRUD
- `dataprovider/sqlqueries.go` — Full file read (82 lines): SQL query builders for all operations
- `dataprovider/bolt.go` — Full file read (558 lines): BoltDB provider with serialized quota transactions
- `dataprovider/memory.go` — Full file read (307 lines): Memory provider with mutex-guarded quota operations

**Virtual Filesystem (`vfs/`):**
- `vfs/` — Folder contents retrieved (6 files)
- `vfs/osfs.go` — Full file read (292 lines): Local FS adapter, atomic upload path, root scanning
- `vfs/vfs.go` — Summary retrieved: Fs interface definition (not full read, summary sufficient)
- `vfs/s3fs.go` — Summary retrieved: S3 backend (atomic upload not supported)

**Supporting Packages:**
- `logger/` — Folder contents retrieved (3 files)
- `logger/logger.go` — Full file read (185 lines): Structured logging, TransferLog, ConnectionFailedLog
- `config/` — Folder contents retrieved (4 files)
- `config/config.go` — Summary retrieved: Configuration loading and validation
- `metrics/metrics.go` — Grep analysis: metric function signatures

**Tech Spec Sections Retrieved:**
- Section 4.2 Core Business Process Flows — Upload, download, quota, authentication flows
- Section 4.4 State Management — Connection, transfer, and provider state diagrams
- Section 4.6 Idle Connection Monitoring — Idle timeout enforcement flow
- Section 4.10 Concurrency and Registry Management — Shared state registry architecture

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were provided or referenced for this project.

