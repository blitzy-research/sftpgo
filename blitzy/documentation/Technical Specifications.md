# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers investigative questions about SFTPGo's quota enforcement behavior during SFTP file uploads, based entirely on source code analysis of the repository.

**Documentation Type:** Technical investigation / Q&A documentation (code-driven analysis document)

**Category:** Create new documentation

The user's requirements translate to a structured investigation document that must answer these specific questions through code-level evidence:

- **Quota Limit Behavior**: What exactly happens when an SFTP upload would exceed a user's configured file count or size quota? Does the transfer complete, get rejected mid-stream, or fail immediately before starting?
- **SFTP Client Error Messages**: What is the exact SFTP protocol error code/message returned to the client when a quota rejection occurs?
- **Server-Side Logging**: What gets logged in the SFTPGo server logs during a quota rejection, including the log level, sender field, and structured log fields?
- **Quota Accuracy**: After each upload, does the quota usage that SFTPGo tracks in its database match the actual bytes stored on disk in the user's home directory? Are there conditions under which discrepancies arise?
- **Quota Check Timing**: Based on code analysis, is the quota check performed before data transfer begins, during the upload, or after the upload completes?

**Implicit Documentation Needs Surfaced:**
- The `track_quota` configuration modes (0, 1, 2) and their impact on enforcement behavior must be documented since they directly affect whether quota checks function at all
- The difference between SFTP uploads (via `handler.go`) and SCP uploads (via `scp.go`) in quota handling must be addressed since both code paths call `hasSpace()` but return different error types
- The atomic upload mode's interaction with quota (how temporary files affect size tracking) is relevant context
- The `copyFromReaderToWriter` quota enforcement in `ssh_cmd.go` for system commands like `rsync` adds a third quota enforcement pattern that behaves differently (mid-transfer check)
- Concurrent upload race conditions between the pre-transfer quota check and post-transfer quota update are an observable behavior the document should address

### 0.1.2 Special Instructions and Constraints

- **Temporary Database Requirement**: The user explicitly requires creating a temporary `sftpgo.db` file for investigation and removing it after completing the analysis. The user also specifies removing any temporary files and leaving the codebase in its original unchanged state.
- **Implementation Rule — SWE-AtlasQnA-Repo**: The output must be a new markdown document named `sftpgo_44634210287c.md` placed in the `blitzy/documentation` directory. It must provide thinking/rationale behind answers, base answers on code as truth, and not modify any existing files in the source repository.
- **No Source Modifications**: The document answers questions about observed behavior from code analysis — no functional code changes are in scope.
- **Code as Truth**: All answers must be grounded in specific source code references (file paths, line numbers, function names), not assumptions or external documentation.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document quota enforcement behavior, we will **create** `blitzy/documentation/sftpgo_44634210287c.md` by analyzing the quota check logic in `sftpd/handler.go` (the `hasSpace()` method at line 515 and the `handleSFTPUploadToNewFile`/`handleSFTPUploadToExistingFile` handlers), the transfer completion logic in `sftpd/transfer.go` (the `Close()` method at line 121), and the data provider quota functions in `dataprovider/dataprovider.go` (`UpdateUserQuota` at line 312, `GetUsedQuota` at line 326).
- To document the exact SFTP error returned, we will trace the return path from `hasSpace()` returning `false` through the handler returning `sftp.ErrSSHFxFailure` back to the SFTP client via the `pkg/sftp` library.
- To document server log entries, we will catalog every `c.Log()` and `logger.*` call in the quota rejection path across `sftpd/handler.go`, `sftpd/scp.go`, `sftpd/transfer.go`, and `sftpd/ssh_cmd.go`.
- To document quota accuracy and timing, we will map the complete lifecycle from the pre-upload `hasSpace()` check through `Transfer.WriteAt()` to `Transfer.Close()` where `dataprovider.UpdateUserQuota` is invoked, identifying the asynchronous gap.

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `dataprovider` package's `TrackQuota` configuration (modes 0, 1, 2 defined in `dataprovider/dataprovider.go` lines 130–135) gates whether quota tracking is even active — this context is essential to the investigation.
- Based on structure: Quota enforcement spans multiple files (`sftpd/handler.go`, `sftpd/scp.go`, `sftpd/ssh_cmd.go`, `sftpd/transfer.go`, `dataprovider/dataprovider.go`, `dataprovider/sqlcommon.go`, `dataprovider/sqlqueries.go`) requiring a consolidated explanation.
- Based on dependencies: The `pkg/sftp` library's error constants (`sftp.ErrSSHFxFailure`) map to SSH protocol-level status codes that determine what the SFTP client actually sees.
- Based on the investigation flow: The SQL `UPDATE` query in `dataprovider/sqlqueries.go` (function `getUpdateQuotaQuery`) uses incremental arithmetic (`used_quota_size = used_quota_size + ?`) rather than absolute assignment when `reset=false`, which has implications for quota accuracy under concurrent uploads.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal, README-centric documentation structure** with no dedicated documentation generator or framework in use.

**Documentation files discovered:**

| File | Purpose | Quota Coverage |
|------|---------|----------------|
| `README.md` | Main project onboarding document; covers features, installation, configuration, authentication, storage, logging, metrics, deployment | One line: "Quota support: accounts can have individual quota expressed as max total size and/or max number of files" — no behavioral details |
| `docker/README.md` | Docker deployment guide | None |
| `docker/sftpgo/alpine/README.md` | Alpine Docker variant | None |
| `docker/sftpgo/debian/README.md` | Debian Docker variant | None |
| `scripts/README.md` | REST API CLI and migration scripts | None |

**Documentation framework:** None detected. No `mkdocs.yml`, `docusaurus.config.js`, `sphinx.conf.py`, or similar configuration files exist in the repository. Documentation is pure Markdown without a static site generator.

**API documentation:** An OpenAPI 3.0.1 schema exists at `httpd/schema/openapi.yaml` documenting the REST API surface. This file defines quota-related endpoints (`/api/v1/quota_scan`) but does not document quota enforcement behavior during SFTP transfers.

**Diagram tools:** No Mermaid, PlantUML, or other diagramming tool configuration detected. The `README.md` does not contain any diagrams.

**Logging format:** Structured JSON logs via `zerolog` (configured in `logger/logger.go`) with fields including `sender`, `connection_id`, `timestamp`, and `message`. Transfer logs include `elapsed_ms`, `size_bytes`, `username`, `file_path`, `protocol`.

**Destination directory:** The `blitzy/documentation/` directory does not currently exist and must be created.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify quota-related code:

- **Quota enforcement entry points**: `grep -rn "hasSpace\|quota\|QuotaFiles\|QuotaSize" --include="*.go"` across the `sftpd/` and `dataprovider/` packages
- **Error path analysis**: `grep -rn "ErrSSHFxFailure\|denying.*write.*space\|errQuotaExceeded"` to trace the client-facing error
- **Log emission points**: `grep -rn "c.Log\|logger\." sftpd/handler.go sftpd/scp.go sftpd/transfer.go sftpd/ssh_cmd.go` for all log messages in quota paths
- **Database quota operations**: Analysis of `dataprovider/sqlqueries.go` (`getUpdateQuotaQuery`, `getQuotaQuery`) and `dataprovider/sqlcommon.go` (`sqlCommonUpdateQuota`, `sqlCommonGetUsedQuota`)

**Key directories examined:**

| Directory | Relevance |
|-----------|-----------|
| `sftpd/` | Core quota enforcement logic: `handler.go` (SFTP upload handlers, `hasSpace()`), `transfer.go` (post-transfer quota update), `scp.go` (SCP quota check), `ssh_cmd.go` (SSH command quota), `sftpd.go` (quota scan tracking) |
| `dataprovider/` | Quota persistence: `dataprovider.go` (UpdateUserQuota, GetUsedQuota, TrackQuota config), `user.go` (User struct with QuotaFiles/QuotaSize/UsedQuotaFiles/UsedQuotaSize), `sqlcommon.go` (SQL execution), `sqlqueries.go` (SQL query definitions) |
| `vfs/` | Filesystem abstraction: `osfs.go` (ScanRootDirContents for quota rescan), `vfs.go` (Fs interface) |
| `config/` | Configuration: `config.go` (default TrackQuota setting, configuration loading) |
| `logger/` | Logging: `logger.go` (structured log format, TransferLog, severity levels) |
| `sql/sqlite/` | Schema: `20190828.sql` (baseline users table with quota columns) |
| `httpd/` | REST API: `api_quota.go` (quota scan endpoints) |

**Related existing documentation:** The `README.md` mentions quota support in the features list and briefly describes the `track_quota` configuration option values (0, 1, 2) in the data provider configuration section but provides no detail about enforcement mechanics, error responses, or logging behavior.

### 0.2.3 Web Search Research Conducted

No external web search is required for this documentation task. The user explicitly instructs to base all answers on the code as truth. The investigation is entirely source-code-driven, analyzing the Go source files to determine runtime behavior from the implementation rather than external documentation or third-party sources.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation analysis:**

- **Module: `sftpd/handler.go`**
  - Public APIs relevant: `Filewrite()` (line 98), `handleSFTPUploadToNewFile()` (line 412), `handleSFTPUploadToExistingFile()` (line 450), `hasSpace()` (line 515)
  - Current documentation: Code comments exist but no external documentation of behavior
  - Documentation needed: Detailed analysis of the pre-upload quota check, the `>=` comparison operator, the `sftp.ErrSSHFxFailure` error return, and the log message at `LevelInfo`

- **Module: `sftpd/transfer.go`**
  - Public APIs relevant: `Transfer.Close()` (line 121), `Transfer.WriteAt()` (line 91), `Transfer.copyFromReaderToWriter()` (line 215)
  - Current documentation: Code comment on `Close()` states it "updates the user quota (for uploads)"
  - Documentation needed: Analysis of the post-transfer `dataprovider.UpdateUserQuota()` call and its parameters (`numFiles`, `bytesReceived`, `reset=false`)

- **Module: `sftpd/scp.go`**
  - Public APIs relevant: `handleUploadFile()` (line 184), `handleUpload()` (line 226)
  - Current documentation: None regarding quota behavior
  - Documentation needed: SCP-specific quota check path analysis (calls `hasSpace(true)` same as SFTP but sends error string directly via channel)

- **Module: `sftpd/ssh_cmd.go`**
  - Public APIs relevant: `executeSystemCommand()` (line 151), `rescanHomeDir()` (line 333)
  - Current documentation: None regarding quota behavior
  - Documentation needed: Analysis of the different quota enforcement approach for system commands (uses `UsedQuotaFiles > QuotaFiles` and `remainingQuotaSize` with `copyFromReaderToWriter`)

- **Module: `dataprovider/dataprovider.go`**
  - Public APIs relevant: `UpdateUserQuota()` (line 312), `GetUsedQuota()` (line 326), `GetQuotaTracking()` (line 215)
  - Current documentation: GoDoc comments exist
  - Documentation needed: Analysis of TrackQuota modes and how `MethodDisabledError` affects quota enforcement fallback

- **Module: `dataprovider/user.go`**
  - Public APIs relevant: `User` struct (line 64), `HasQuotaRestrictions()` (line 258), `GetQuotaSummary()` (line 263)
  - Current documentation: Struct field comments exist
  - Documentation needed: Analysis of `QuotaFiles`, `QuotaSize`, `UsedQuotaFiles`, `UsedQuotaSize` fields and their relationship

- **Module: `dataprovider/sqlcommon.go`**
  - Public APIs relevant: `sqlCommonUpdateQuota()` (line 74), `sqlCommonGetUsedQuota()` (line 109)
  - Current documentation: None
  - Documentation needed: Analysis of incremental vs reset quota update SQL mechanics

- **Module: `dataprovider/sqlqueries.go`**
  - Public APIs relevant: `getUpdateQuotaQuery()` (line 43), `getQuotaQuery()` (line 56)
  - Current documentation: None
  - Documentation needed: The SQL arithmetic (`used_quota_size = used_quota_size + ?`) and its implications

- **Module: `logger/logger.go`**
  - Public APIs relevant: `Log()` (line 84), `TransferLog()` (line 139), structured field definitions
  - Current documentation: Package comments
  - Documentation needed: Analysis of the structured log format and which fields appear during quota rejection events

**Configuration requiring documentation:**

| Config File | Option | Documented | Missing |
|-------------|--------|------------|---------|
| `sftpgo.json` | `data_provider.track_quota` | Values 0/1/2 in README | Behavioral impact on quota enforcement |
| `sftpgo.json` | `data_provider.manage_users` | Mentioned in README | Impact on quota update operations |
| `sftpgo.json` | `sftpd.upload_mode` | Mentioned in README | Interaction with atomic uploads and quota |

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No existing documentation** explains what happens at the SFTP protocol level when a quota limit is reached
- **No existing documentation** describes the exact error code (`SSH_FX_FAILURE`) returned to SFTP clients on quota rejection
- **No existing documentation** catalogs the server-side log entries produced during quota enforcement
- **No existing documentation** analyzes the timing relationship between quota checks and data transfer
- **No existing documentation** addresses potential discrepancies between database-tracked quota and actual disk usage
- **No existing documentation** covers the behavioral differences in quota enforcement across SFTP, SCP, and SSH command (rsync/git) protocols
- **No existing documentation** describes the race condition window between the pre-transfer `hasSpace()` check and the post-transfer `UpdateUserQuota()` call

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the implementation rule `SWE-AtlasQnA-Repo` and must be a single Markdown file answering the user's questions with code-based evidence. The planned structure:

```
blitzy/
└── documentation/
    └── sftpgo_44634210287c.md
        ├── Overview (context and approach)
        ├── Quota Configuration and Setup
        │   ├── User quota fields and their meaning
        │   ├── TrackQuota configuration modes
        │   └── Test setup methodology
        ├── Quota Enforcement Behavior on Upload
        │   ├── When is the quota checked? (timing analysis)
        │   ├── What happens when quota is exceeded?
        │   ├── SFTP protocol error returned to client
        │   └── Behavioral differences: SFTP vs SCP vs SSH commands
        ├── Server-Side Logging Analysis
        │   ├── Log entries during quota rejection
        │   ├── Structured log fields and format
        │   └── Log level and sender details
        ├── Quota Accuracy: Database vs Disk
        │   ├── How quota counters are updated
        │   ├── Incremental update mechanics
        │   ├── Potential discrepancy scenarios
        │   └── Quota scan reconciliation
        ├── Timing Analysis: Quota Check Lifecycle
        │   ├── Pre-transfer check (hasSpace)
        │   ├── During-transfer behavior
        │   ├── Post-transfer quota update
        │   └── Concurrent upload race window
        └── Summary and Key Findings
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract quota enforcement flow from `sftpd/handler.go` by tracing the `Filewrite()` → `handleSFTPUploadToNewFile()`/`handleSFTPUploadToExistingFile()` → `hasSpace()` call chain (Source: `sftpd/handler.go:98-534`)
- Extract SFTP error constants from `handler.go` return statements: `sftp.ErrSSHFxFailure` at lines 415 and 455 (Source: `sftpd/handler.go`)
- Extract transfer completion quota update from `Transfer.Close()` at line 167: `dataprovider.UpdateUserQuota(dataProvider, t.user, numFiles, t.bytesReceived, false)` (Source: `sftpd/transfer.go:121-169`)
- Extract SQL quota operations from `dataprovider/sqlqueries.go` functions `getUpdateQuotaQuery()` and `getQuotaQuery()` (Source: `dataprovider/sqlqueries.go:43-58`)
- Extract log message patterns from `c.Log()` calls throughout `sftpd/handler.go` and `logger.TransferLog()` in `sftpd/transfer.go` (Source: `logger/logger.go:139-149`)
- Generate examples by analyzing the test functions `TestBasicSFTPHandling`, `TestQuotaFileReplace`, and `TestQuotaDisabledError` in `sftpd/sftpd_test.go` (Source: `sftpd/sftpd_test.go:226-1621`)

**Documentation Standards:**

- Markdown formatting with headers (`#`, `##`, `###`) for hierarchical organization
- Code snippets using fenced blocks with Go syntax highlighting for source references
- Tables for structured comparisons (e.g., quota check behavior across protocols)
- Source citations as inline references in the format `Source: /path/to/file.go:LineNumber`
- All answers grounded in specific code evidence, never assumption

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create within the output document:**

- **Sequence diagram**: The complete lifecycle of an SFTP upload with quota check — from `Filewrite()` through `hasSpace()`, file creation, `Transfer.WriteAt()`, `Transfer.Close()`, and `UpdateUserQuota()`
- **Flowchart**: The `hasSpace()` decision logic showing the branching between file count check and size check, and how `TrackQuota` modes gate enforcement
- **Sequence diagram**: The difference between quota enforcement for SFTP (pre-transfer check returning `ErrSSHFxFailure`) vs SCP (pre-transfer check sending error string) vs SSH system commands (mid-transfer check via `copyFromReaderToWriter`)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/sftpgo_44634210287c.md` | CREATE | `sftpd/handler.go`, `sftpd/transfer.go`, `sftpd/scp.go`, `sftpd/ssh_cmd.go`, `sftpd/sftpd.go`, `dataprovider/dataprovider.go`, `dataprovider/user.go`, `dataprovider/sqlcommon.go`, `dataprovider/sqlqueries.go`, `logger/logger.go`, `config/config.go`, `sftpgo.json`, `sql/sqlite/20190828.sql` | Complete Q&A investigation document answering all user questions about SFTPGo quota enforcement behavior, grounded in code analysis with source citations, Mermaid diagrams, and structured findings |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Technical Investigation / Q&A Document
Source Code:
  - sftpd/handler.go (quota check logic, SFTP upload handlers, hasSpace method)
  - sftpd/transfer.go (post-transfer quota update, Transfer.Close)
  - sftpd/scp.go (SCP upload quota enforcement)
  - sftpd/ssh_cmd.go (SSH command quota, errQuotaExceeded, rescanHomeDir)
  - sftpd/sftpd.go (quota scan tracking, connection management)
  - dataprovider/dataprovider.go (UpdateUserQuota, GetUsedQuota, TrackQuota config)
  - dataprovider/user.go (User struct: QuotaFiles, QuotaSize, UsedQuotaFiles, UsedQuotaSize)
  - dataprovider/sqlcommon.go (sqlCommonUpdateQuota, sqlCommonGetUsedQuota)
  - dataprovider/sqlqueries.go (getUpdateQuotaQuery, getQuotaQuery SQL definitions)
  - logger/logger.go (structured log format, TransferLog, severity levels)
  - config/config.go (default TrackQuota, configuration loading)
  - sftpgo.json (sample configuration with track_quota: 2)
  - sql/sqlite/20190828.sql (baseline schema with quota columns)
Sections:
  - Overview (purpose, approach, and investigation scope)
  - Quota Configuration and Setup
    - User model quota fields (from dataprovider/user.go:64-110)
    - TrackQuota modes 0/1/2 (from dataprovider/dataprovider.go:130-135)
    - Database schema quota columns (from sql/sqlite/20190828.sql)
  - What Happens When Quota Is Reached
    - Pre-transfer hasSpace() check (from sftpd/handler.go:515-534)
    - SFTP client receives sftp.ErrSSHFxFailure (SSH_FX_FAILURE)
    - Transfer is rejected before data transfer begins
    - File is never created on disk when quota is already full
  - Exact SFTP Client Error Message
    - sftp.ErrSSHFxFailure maps to SSH_FX_FAILURE (status code 4)
    - SCP path sends "denying file write due to space limit" string
  - Server Log Analysis
    - "denying file write due to space limit" at LevelInfo (sftpd/handler.go:414,454)
    - "quota exceed for user" at LevelDebug with detailed fields (sftpd/handler.go:528-529)
    - Structured JSON fields: sender, connection_id, timestamp, message
  - Quota Database vs Disk Comparison
    - Incremental SQL update: used_quota_size = used_quota_size + ? (sqlqueries.go:48)
    - Quota updated after transfer.Close() (transfer.go:167)
    - Quota scan rescanHomeDir() resets to absolute values (ssh_cmd.go:333-353)
    - Potential discrepancy during concurrent uploads
  - Quota Check Timing Analysis
    - CHECK: Before transfer starts (hasSpace in Filewrite)
    - TRANSFER: Data written to disk (WriteAt in Transfer)
    - UPDATE: After transfer completes (Close in Transfer)
    - Race window between check and update
  - Summary of Key Findings
Diagrams:
  - Sequence diagram: SFTP upload quota lifecycle
  - Flowchart: hasSpace() decision logic
  - Comparison diagram: SFTP vs SCP vs SSH quota enforcement
Key Citations:
  - sftpd/handler.go:412-534
  - sftpd/transfer.go:121-169
  - sftpd/scp.go:184-190
  - sftpd/ssh_cmd.go:29,151-157,333-353
  - dataprovider/dataprovider.go:130-135,312-322,326-331
  - dataprovider/user.go:64-110,258-259
  - dataprovider/sqlcommon.go:74-90,109-126
  - dataprovider/sqlqueries.go:43-58
  - logger/logger.go:84-116,139-149
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository does not use a documentation generator framework. The output is a standalone Markdown file placed in `blitzy/documentation/`.

### 0.5.4 Cross-Documentation Dependencies

- The new document references the `README.md` section on quota support for context on the user-facing feature description
- The new document references `sftpgo.json` as the sample configuration showing default `track_quota: 2`
- The new document references `httpd/schema/openapi.yaml` for the REST API quota scan endpoint definition
- No navigation links, table of contents updates, or index/glossary changes are required since the output is a self-contained standalone document

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following runtime and build dependencies from the project are directly relevant to the quota investigation documentation. All versions are sourced from `go.mod`.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go modules | `go` (language) | 1.13 | Build-time dependency; the Go runtime version targeted by the project |
| Go modules | `github.com/pkg/sftp` | v1.11.0 | SFTP protocol library defining `sftp.ErrSSHFxFailure` and SFTP request/response handling |
| Go modules | `golang.org/x/crypto` | v0.0.0-20200109152110 | SSH transport layer; defines `ssh.Channel` used for SCP error message delivery |
| Go modules | `github.com/mattn/go-sqlite3` | v2.0.2+incompatible | SQLite3 driver for the `sftpgo.db` database used in quota storage/retrieval |
| Go modules | `github.com/rs/zerolog` | v1.17.2 | Structured JSON logging library; all quota-related log entries use zerolog |
| Go modules | `gopkg.in/natefinch/lumberjack.v2` | v2.0.0 | Log file rotation for the zerolog-based log output |
| Go modules | `github.com/spf13/viper` | v1.6.1 | Configuration management; reads `track_quota` and other settings from sftpgo.json |
| Go modules | `github.com/spf13/cobra` | v0.0.5 | CLI framework; `serve` and `portable` commands bootstrap the server with quota config |
| Go modules | `go.etcd.io/bbolt` | v1.3.3 | BoltDB alternative data provider with its own quota update implementation |
| Go modules | `github.com/eikenb/pipeat` (replaced) | v0.0.0-20200114135659 | Pipe-based I/O for S3 streaming uploads; `PipeWriterAt`/`PipeReaderAt` used in Transfer |

No additional documentation-specific tooling (mkdocs, Sphinx, Docusaurus) is required. The output is a self-contained Markdown file that does not need a build step.

### 0.6.2 Documentation Reference Updates

Not applicable. This is a new standalone document creation with no existing links to update. The document is placed in a new `blitzy/documentation/` directory that does not currently exist in the repository, and no other files reference this directory.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of quota behavior documentation:**

| Documentation Area | Current State | Target |
|-------------------|---------------|--------|
| Quota enforcement timing (when check occurs) | 0% — Not documented anywhere | 100% — Full lifecycle analysis with code citations |
| SFTP client error message on quota rejection | 0% — Not documented anywhere | 100% — Exact error code and message documented |
| Server log entries during quota events | 0% — Not documented anywhere | 100% — All log emission points cataloged with fields |
| Database vs disk quota accuracy | 0% — Not documented anywhere | 100% — Incremental update mechanics explained |
| Quota behavior differences across protocols | 0% — Not documented anywhere | 100% — SFTP, SCP, SSH commands compared |
| TrackQuota configuration impact | ~10% — Brief mention in README | 100% — All three modes documented with behavioral impact |
| Concurrent upload quota race conditions | 0% — Not documented anywhere | 100% — Race window analysis with code evidence |

**Target coverage:** 100% of all user questions answered with direct code citations.

**Coverage gaps to address:**
- The `hasSpace()` method's `>=` operator (line 526-527) means "at limit" equals "rejected" — this must be explicitly stated
- The asymmetry between new file uploads (`checkFiles=true`) and existing file overwrites (`checkFiles=false`) in the `hasSpace()` parameter
- The `MethodDisabledError` fallback in `hasSpace()` (line 519-521) that permits uploads when quota tracking is disabled
- The SCP error path sends a human-readable string whereas SFTP sends a protocol-level status code

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must cite specific source files, function names, and line numbers
- All quota enforcement code paths (SFTP, SCP, SSH commands) must be analyzed
- The complete lifecycle from upload request through quota check, data transfer, transfer close, and quota update must be traced
- Both the file count quota (`QuotaFiles`) and size quota (`QuotaSize`) enforcement branches must be covered

**Accuracy validation:**
- All code references must match the actual source as retrieved from the repository
- The SFTP error constant (`sftp.ErrSSHFxFailure`) must be correctly identified and its SSH protocol mapping (`SSH_FX_FAILURE`, status code 4) must be verified
- SQL query analysis must accurately reflect the actual queries generated by `getUpdateQuotaQuery()` and `getQuotaQuery()`
- Log format analysis must match the zerolog structured output format configured in `logger/logger.go`

**Clarity standards:**
- Technical findings presented with progressive disclosure: summary answer first, then detailed code evidence
- Each question from the user's prompt addressed in a clearly identifiable section
- Mermaid diagrams to visualize the quota check lifecycle and decision logic
- Consistent terminology: "quota check" for the `hasSpace()` call, "quota update" for the `UpdateUserQuota()` call, "quota scan" for the `rescanHomeDir()` full-directory traversal

**Maintainability:**
- Source citations include file paths and line numbers for traceability
- Findings are organized by question/topic for easy navigation

### 0.7.3 Example and Diagram Requirements

- Minimum code snippet examples: at least one per answer showing the critical code path
- Diagram types required: sequence diagram (upload lifecycle), flowchart (hasSpace decision logic)
- Code example accuracy: all snippets are direct extracts from the analyzed source files, not fabricated examples

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/sftpgo_44634210287c.md` — The complete investigation document

**Source code analyzed (read-only) for documentation content:**
- `sftpd/handler.go` — SFTP upload handlers, `hasSpace()` quota check, `Filewrite()` entry point
- `sftpd/transfer.go` — `Transfer.Close()` post-transfer quota update, `Transfer.WriteAt()`, `Transfer.copyFromReaderToWriter()`
- `sftpd/scp.go` — SCP upload path quota check via `handleUploadFile()` and `handleUpload()`
- `sftpd/ssh_cmd.go` — SSH system command quota enforcement via `executeSystemCommand()` and `rescanHomeDir()`
- `sftpd/sftpd.go` — Quota scan tracking (`AddQuotaScan`, `RemoveQuotaScan`, `GetQuotaScans`), connection/transfer registries
- `sftpd/server.go` — Server configuration including `uploadMode` and `setstatMode` initialization
- `sftpd/sftpd_test.go` — Test cases validating quota behavior (`TestBasicSFTPHandling`, `TestQuotaFileReplace`, `TestQuotaDisabledError`, `TestQuotaScan`)
- `dataprovider/dataprovider.go` — `UpdateUserQuota()`, `GetUsedQuota()`, `TrackQuota` config, `MethodDisabledError`
- `dataprovider/user.go` — `User` struct definition with quota fields, `HasQuotaRestrictions()`, `GetQuotaSummary()`
- `dataprovider/sqlcommon.go` — `sqlCommonUpdateQuota()`, `sqlCommonGetUsedQuota()`, `getUserFromDbRow()`
- `dataprovider/sqlqueries.go` — `getUpdateQuotaQuery()`, `getQuotaQuery()`, `selectUserFields`
- `dataprovider/sqlite.go` — SQLite provider initialization
- `logger/logger.go` — Structured log format, `TransferLog()`, `Log()`, severity levels
- `config/config.go` — Default configuration including `TrackQuota: 1`
- `sftpgo.json` — Sample runtime configuration with `track_quota: 2`
- `sql/sqlite/20190828.sql` — Baseline SQLite schema with quota columns
- `README.md` — Existing quota feature description for context
- `go.mod` — Dependency versions (Go 1.13, pkg/sftp v1.11.0, zerolog v1.17.2, etc.)
- `httpd/api_quota.go` — REST API quota scan handlers for context
- `vfs/osfs.go` — `ScanRootDirContents()` used in quota rescan

**Documentation assets:**
- Mermaid diagrams embedded within the output Markdown document

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No Go source files will be modified. The implementation rule explicitly requires: "Do not modify any existing files in the source repository."
- **Test file modifications**: No test files will be created or modified.
- **Feature additions or code refactoring**: No new functionality, bug fixes, or code changes.
- **Deployment configuration changes**: No changes to Docker, systemd, launchd, or CI configuration.
- **Database modifications**: A temporary `sftpgo.db` may be created for investigative purposes but must be removed after the investigation, leaving the codebase in its original state.
- **README.md updates**: The existing `README.md` will not be modified; all findings go into the new `blitzy/documentation/` file.
- **Non-quota documentation**: Documentation for other SFTPGo features (authentication, permissions, S3 storage, bandwidth throttling, etc.) is not in scope.
- **External SFTP client behavior**: The document analyzes server-side code; variations in how different SFTP clients display the `SSH_FX_FAILURE` error are not in scope.
- **Performance testing or benchmarking**: Quota behavior is analyzed from code, not through load testing.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file that does not require compilation or static site generation.
- **Documentation preview command**: Standard Markdown viewer or `cat blitzy/documentation/sftpgo_44634210287c.md`
- **Diagram generation command**: Mermaid diagrams are embedded inline in the Markdown using fenced code blocks with the `mermaid` language tag. They render natively in GitHub, GitLab, and most modern Markdown renderers.
- **Default format**: Markdown with Mermaid diagrams and Go code snippets
- **Citation requirement**: Every technical claim must reference a specific source file, function name, and line number from the repository
- **Style guide**: Follows the `SWE-AtlasQnA-Repo` implementation rule:
  - Provide thinking and rationale behind each answer
  - Base all answers on the code as the single source of truth
  - Do not make assumptions — every conclusion must be traceable to specific source code
  - Do not modify any existing files in the source repository
- **Output file path**: `blitzy/documentation/sftpgo_44634210287c.md`
- **Temporary artifacts**:
  - A temporary `sftpgo.db` SQLite database file may be created during investigation
  - All temporary files (including `sftpgo.db`) must be removed after completing the investigation
  - The codebase must be left in its original unchanged state upon completion

### 0.9.2 Environment Configuration

- **Go version**: 1.13 (as specified in `go.mod`)
- **SQLite support**: Required for database analysis; the `mattn/go-sqlite3` CGO package requires a C compiler
- **Database file**: `sftpgo.db` — the default SQLite database name as defined in `sftpgo.json` (line 26) and `config/config.go` (line 67: `Name: "sftpgo.db"`)
- **TrackQuota default**: The code default in `config/config.go` is `TrackQuota: 1` (always track), while the sample `sftpgo.json` overrides this to `track_quota: 2` (track only for users with restrictions)

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit directives and the `SWE-AtlasQnA-Repo` implementation rule:

- **Create a new markdown document named `sftpgo_44634210287c.md`** — The file name must match the source branch name exactly.
- **Place the generated document in the `blitzy/documentation` directory** — This directory must be created if it does not exist.
- **Provide thinking / rationale behind the answers** — Each answer must include the reasoning chain showing how the conclusion was derived from the code, not just the conclusion itself.
- **Do not make assumptions, base your answers on the code as the truth** — Every statement about SFTPGo's quota behavior must be directly traceable to a specific function, line, or code construct in the analyzed source files.
- **Do not modify any existing files in the source repository** — The only filesystem change permitted is creating the new `blitzy/documentation/sftpgo_44634210287c.md` file and the `blitzy/documentation/` directory.
- **Create a temporary `sftpgo.db` file for the investigation** — The database file may be created during analysis but must be removed upon completion.
- **Remove any temporary files created during investigation** — All artifacts created during the analysis process must be cleaned up.
- **Leave the codebase in its original unchanged state** — After the documentation task is complete, the only addition to the repository should be the `blitzy/documentation/` directory and its Markdown file.

## 0.11 References

### 0.11.1 Files and Folders Searched

The following files and folders were comprehensively searched and analyzed across the codebase to derive conclusions for this Agent Action Plan:

**Root-level files:**

| File | Purpose | Key Findings |
|------|---------|--------------|
| `go.mod` | Go module dependencies | Go 1.13; `pkg/sftp` v1.11.0 (defines `ErrSSHFxFailure`); `mattn/go-sqlite3` v2.0.2; `zerolog` v1.17.2 |
| `go.sum` | Dependency checksums | Validates reproducible builds |
| `main.go` | Application entry point | Imports SQLite driver for side effects, delegates to `cmd.Execute()` |
| `sftpgo.json` | Sample runtime configuration | Default `track_quota: 2`, `driver: sqlite`, `name: sftpgo.db`, `manage_users: 1` |
| `README.md` | Project documentation | Single line on quota: "accounts can have individual quota expressed as max total size and/or max number of files" |
| `LICENSE` | GPLv3 license | N/A for quota analysis |
| `.travis.yml` | CI configuration | Go 1.13.x, Linux/macOS, SQLite bootstrapping |

**`sftpd/` package (SFTP/SCP/SSH server implementation):**

| File | Purpose | Key Findings |
|------|---------|--------------|
| `sftpd/handler.go` | SFTP request handlers | `hasSpace()` at line 515: pre-transfer quota check using `>=` comparison; returns `sftp.ErrSSHFxFailure` on quota exceeded; logs "denying file write due to space limit" at LevelInfo and "quota exceed for user" at LevelDebug |
| `sftpd/transfer.go` | Transfer abstraction | `Close()` at line 121: post-transfer `UpdateUserQuota()` call with incremental values; `copyFromReaderToWriter()` at line 215: mid-transfer quota check for SSH commands |
| `sftpd/scp.go` | SCP protocol handler | `handleUploadFile()` at line 184: calls `hasSpace(true)`, sends string error "denying file write due to space limit" via channel |
| `sftpd/ssh_cmd.go` | SSH command handler | `errQuotaExceeded` at line 29; `executeSystemCommand()` at line 151: checks `UsedQuotaFiles > QuotaFiles`; `rescanHomeDir()` at line 333: full directory scan with absolute quota reset |
| `sftpd/sftpd.go` | Runtime bookkeeping | `ActiveQuotaScan` type; `AddQuotaScan`/`RemoveQuotaScan` for single-flight quota scans; upload mode constants; `SetDataProvider()` |
| `sftpd/server.go` | Server configuration | `Configuration` struct; `uploadMode` and `setstatMode` globals; SSH authentication callbacks |
| `sftpd/sftpd_test.go` | Integration tests | `TestBasicSFTPHandling` (quota tracking after upload/delete), `TestQuotaFileReplace` (overwrite quota behavior, quota exceeded on size restriction), `TestQuotaDisabledError` (TrackQuota=0 behavior), `TestQuotaScan` (quota scan verification) |
| `sftpd/internal_test.go` | Unit tests | `errQuotaExceeded` assertion tests for `copyFromReaderToWriter` |
| `sftpd/lister.go` | Directory listing adapter | Not relevant to quota |
| `sftpd/cmd_unix.go` | Unix process credentials | Not relevant to quota |
| `sftpd/cmd_windows.go` | Windows process stub | Not relevant to quota |

**`dataprovider/` package (user storage and quota persistence):**

| File | Purpose | Key Findings |
|------|---------|--------------|
| `dataprovider/dataprovider.go` | Provider orchestration | `UpdateUserQuota()` at line 312: TrackQuota gating logic (0=disabled, 1=always, 2=restricted users only); `GetUsedQuota()` at line 326: reads from provider; `MethodDisabledError` returned when TrackQuota=0 |
| `dataprovider/user.go` | User model | `User` struct: `QuotaSize` (int64), `QuotaFiles` (int), `UsedQuotaSize` (int64), `UsedQuotaFiles` (int), `LastQuotaUpdate` (int64); `HasQuotaRestrictions()` returns true if either limit > 0 |
| `dataprovider/sqlcommon.go` | Shared SQL execution | `sqlCommonUpdateQuota()` at line 74: executes update query; `sqlCommonGetUsedQuota()` at line 109: scans `used_quota_size, used_quota_files` from database |
| `dataprovider/sqlqueries.go` | SQL query definitions | `getUpdateQuotaQuery()` at line 43: `used_quota_size = used_quota_size + ?` (incremental) or `used_quota_size = ?` (reset); `getQuotaQuery()` at line 56: `SELECT used_quota_size, used_quota_files` |
| `dataprovider/sqlite.go` | SQLite provider | `initializeSQLiteProvider()`: resolves database path, opens with shared cache, limits to 1 connection |
| `dataprovider/bolt.go` | BoltDB provider | Alternative provider with own quota update implementation in Bolt buckets |
| `dataprovider/memory.go` | In-memory provider | Alternative provider with mutex-protected in-memory quota tracking |
| `dataprovider/mysql.go` | MySQL provider | Uses shared SQL layer from `sqlcommon.go` |
| `dataprovider/pgsql.go` | PostgreSQL provider | Uses shared SQL layer from `sqlcommon.go` |

**Other packages analyzed:**

| File | Purpose | Key Findings |
|------|---------|--------------|
| `logger/logger.go` | Structured logging | zerolog-based; JSON format with `sender`, `connection_id`, `timestamp`, `message` fields; `TransferLog()` adds `elapsed_ms`, `size_bytes`, `username`, `file_path`, `protocol` |
| `config/config.go` | Configuration management | Default `TrackQuota: 1` in code; Viper-based config loading from `sftpgo.json`/YAML/TOML/HCL |
| `vfs/osfs.go` | Local filesystem | `ScanRootDirContents()` used by `rescanHomeDir()` for full-directory quota recalculation |
| `vfs/vfs.go` | Filesystem interface | `Fs` interface; `GetSFTPError()` maps filesystem errors to SFTP errors |
| `sql/sqlite/20190828.sql` | Baseline SQLite schema | Creates `users` table with `quota_size`, `quota_files`, `used_quota_size`, `used_quota_files`, `last_quota_update` columns |
| `httpd/api_quota.go` | REST API quota scan | Quota scan endpoint handler; triggers full-directory scan and absolute quota reset |

**Folders examined:**

| Folder | Depth | Relevance |
|--------|-------|-----------|
| `/` (root) | Level 0 | Project structure, module metadata, sample config |
| `sftpd/` | Level 1 | Primary quota enforcement code — all files examined |
| `dataprovider/` | Level 1 | Quota persistence layer — all files examined |
| `vfs/` | Level 1 | Filesystem abstraction — relevant files examined |
| `logger/` | Level 1 | Log format — all files examined |
| `config/` | Level 1 | Configuration defaults — all files examined |
| `sql/` | Level 1 | Database schema root |
| `sql/sqlite/` | Level 2 | SQLite migration scripts — all files examined |
| `httpd/` | Level 1 | REST API layer — summary examined, `api_quota.go` relevant |
| `docker/` | Level 1 | Deployment — not relevant to quota behavior |
| `scripts/` | Level 1 | CLI tools — not relevant to quota behavior |

### 0.11.2 Attachments

No attachments were provided by the user for this project. The investigation is entirely based on the source code present in the `sftpgo_44634210287c` branch of the repository.

