# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers five interrelated security questions about SFTPGo's internal behavior, grounded exclusively in code-level evidence from the source repository.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical Q&A analysis document (security-focused code analysis)
- **Output Artifact:** A single Markdown file named `sftpgo_44634210287c.md` placed in `blitzy/documentation/` within the destination repository

The user's requirements decompose into the following specific documentation objectives:

- **Objective 1 — External Utility Invocation Security:** Document how SFTPGo's security model governs the invocation of external system commands (rsync, git-receive-pack, git-upload-pack, git-upload-archive) when initiated through SSH exec requests. The user requires evidence of what filtering, validation, and allowlisting occurs between client request and server-side execution. Key source: `sftpd/ssh_cmd.go`, `sftpd/sftpd.go`, `sftpd/server.go`.

- **Objective 2 — Client vs. Server Decision Boundary:** Document the precise boundary between what the connecting client controls and what the server determines. The user needs a clear picture of the decision points in the request processing chain — specifically, how an SSH exec payload is parsed, validated against an allowlist, and either dispatched or rejected. Key source: `sftpd/ssh_cmd.go:processSSHCommand()`, `sftpd/server.go:AcceptInboundConnection()`.

- **Objective 3 — Input Validation for Protocol-Level Data:** Document how the system handles inputs with unusual delimiters, unexpected formatting, or non-standard structures. The user is particularly interested in protocol-level parsing assumptions (e.g., the SCP protocol's `C0644 size name` message format, command payload space-splitting) and where those assumptions might fail. Key source: `sftpd/scp.go:parseUploadMessage()`, `sftpd/ssh_cmd.go:parseCommandPayload()`.

- **Objective 4 — Permission Enforcement Ordering:** Document where in the processing chain directory-level permission restrictions are enforced. The user is concerned about whether any processing or path transformation occurs before the permission check, which could affect the security guarantee. Key source: `sftpd/handler.go`, `sftpd/ssh_cmd.go`, `sftpd/scp.go`, `dataprovider/user.go`.

- **Objective 5 — Filesystem Protection Mechanisms:** Document the path traversal and chroot protections — specifically when they engage, how symlinks are evaluated, and whether coverage is comprehensive across all access paths. Key source: `vfs/osfs.go:ResolvePath()`, `vfs/osfs.go:isSubDir()`, `vfs/osfs.go:findFirstExistingDir()`.

### 0.1.2 Special Instructions and Constraints

The following directives from the user must be strictly observed:

- **No source code modification:** "Don't modify any source files in the repository." — This is an absolute constraint. No existing `.go`, `.json`, `.yml`, or any other repository file may be altered.
- **Evidence-based answers only:** "I don't want theoretical explanations of what the code should do. I want to see actual evidence of how the system behaves in practice." — Every claim in the output document must cite specific source file paths and line numbers.
- **Test script allowance with cleanup:** "You can create test scripts to observe the actual behavior, but clean them up when finished." — Any temporary scripts created for behavioral analysis must be deleted before completion.
- **Implementation rule (SWE-AtlasQnA-Repo):** The output must be a new Markdown document named `sftpgo_44634210287c.md` placed in `blitzy/documentation/`. It must provide thinking and rationale behind answers, base all answers on the code as truth, and not modify any existing repository files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **external utility invocation security model**, we will create a detailed walkthrough of `sftpd/ssh_cmd.go:processSSHCommand()` (lines 45–81), `sftpd/ssh_cmd.go:getSystemCommand()` (lines 288–331), and `sftpd/ssh_cmd.go:executeSystemCommand()` (lines 151–286), tracing the exact code path from SSH exec request receipt through allowlist validation to OS-level process execution with credential wrapping via `sftpd/cmd_unix.go:wrapCmd()`.

- To document the **client vs. server decision boundary**, we will create a flow analysis of `sftpd/server.go:AcceptInboundConnection()` (lines 243–333) showing how the server routes session channels, identifying the `"exec"` request type dispatch (line 327), and mapping the command validation chain in `processSSHCommand()` — specifically the `utils.IsStringInSlice(name, enabledSSHCommands)` check (line 51) against the `supportedSSHCommands` list defined in `sftpd/sftpd.go` (lines 66–68).

- To document **input validation for protocol-level data**, we will analyze `sftpd/ssh_cmd.go:parseCommandPayload()` (lines 423–429) and `sftpd/scp.go:parseUploadMessage()` (lines 631–663) and `sftpd/scp.go:readProtocolMessage()` (lines 542–563) to identify the parsing assumptions (space-delimited splitting, newline-terminated messages, expected prefix characters) and edge cases.

- To document **permission enforcement ordering**, we will trace the execution order in handlers like `sftpd/handler.go:Fileread()` (lines 47–95), `sftpd/handler.go:Filewrite()` (lines 98–134), `sftpd/ssh_cmd.go:executeSystemCommand()` (lines 151–286), and `sftpd/scp.go:handleUpload()` (lines 226–285) to show whether `ResolvePath()` or `HasPerm()` is called first.

- To document **filesystem protections**, we will perform a deep analysis of `vfs/osfs.go:ResolvePath()` (lines 200–223), `vfs/osfs.go:isSubDir()` (lines 278–291), and `vfs/osfs.go:findFirstExistingDir()` (lines 250–276) to explain the symlink evaluation, string-prefix-based containment checking, and the handling of non-existent path components.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation needs are inferred:

- **Rsync symlink hardening:** `sftpd/ssh_cmd.go:getSystemCommand()` (lines 306–321) injects `--safe-links` or `--munge-links` into rsync arguments based on symlink permissions. This server-side argument injection is a critical security behavior that the user's questions about "filtering or validation" directly implicate.

- **SCP protocol implementation disclaimer:** The SCP subsystem is explicitly labeled as experimental in `sftpd/server.go` (lines 81–91). The comment states: "We may not handle some borderline cases or have sneaky bugs." This directly answers the user's concern about whether protocol assumptions hold up under unusual inputs.

- **External auth environment variable injection risk:** `dataprovider/dataprovider.go` (lines 148–157) contains an explicit security warning that environment variables passed to external auth programs are "not quoted" and "may contain special characters" under the control of "a possibly malicious remote user." This is relevant to the user's question about what happens between client initiation and server action.

- **Command argument stripping behavior:** In `sftpd/ssh_cmd.go:getDestPath()` (lines 356–371), the last argument is stripped of single and double quotes and normalized via `path.Clean()`. This implicit input transformation occurs before `ResolvePath()` is called, which is relevant to the user's questions about input formatting and processing order.

- **Post-operation action hook execution:** `sftpd/sftpd.go:executeAction()` (lines 438–492) and `sftpd/sftpd.go:executeNotificationCommand()` (lines 418–435) run external commands and HTTP notifications after file operations. User-controlled paths and filenames flow into these command arguments and environment variables, which is pertinent to the security boundary questions.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a minimal documentation infrastructure with no dedicated documentation generator, framework, or build tooling. All existing documentation is in plain Markdown files co-located with the source code.

- **Documentation generator:** None detected. No `mkdocs.yml`, `docusaurus.config.js`, `sphinx.conf.py`, or similar configuration files exist in the repository.
- **API documentation tools:** An OpenAPI 3.0.1 schema exists at `httpd/schema/openapi.yaml` for the REST API, but no auto-generated API documentation tooling (JSDoc, Godoc, etc.) is configured.
- **Diagram tools:** None configured in the repository. Mermaid diagrams will be introduced by the output documentation.
- **Documentation hosting/deployment:** None configured. Documentation is consumed directly from the repository.

**Existing Markdown documentation files discovered:**

| File Path | Purpose | Relevance to Task |
|-----------|---------|-------------------|
| `README.md` | Main project onboarding: features, installation, configuration, SSH commands, authentication | HIGH — contains SSH command documentation the user references |
| `docker/README.md` | Docker deployment overview | LOW — not related to security model |
| `docker/sftpgo/alpine/README.md` | Alpine Docker image instructions | LOW |
| `docker/sftpgo/debian/README.md` | Debian Docker image instructions | LOW |
| `scripts/README.md` | REST API CLI and migration scripts | LOW |

**Critical finding:** There is no existing dedicated security architecture documentation, no SSH command flow documentation, and no document that explains the internal processing chain from client request to server action. The user's questions target an area with zero existing documentation coverage.

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns were used to identify the code modules requiring documentation for this task:

- **SSH command handling:** `sftpd/ssh_cmd.go` — Contains `processSSHCommand()`, `getSystemCommand()`, `executeSystemCommand()`, `handleHashCommands()`, `parseCommandPayload()`, `getDestPath()`
- **SCP protocol handler:** `sftpd/scp.go` — Contains `scpCommand.handle()`, `handleRecursiveUpload()`, `handleUpload()`, `handleDownload()`, `parseUploadMessage()`, `readProtocolMessage()`
- **SFTP request handler:** `sftpd/handler.go` — Contains `Fileread()`, `Filewrite()`, `Filecmd()`, `Filelist()`, `handleSFTPSetstat()`, `handleSFTPRename()`, etc.
- **Server and session routing:** `sftpd/server.go` — Contains `AcceptInboundConnection()`, `checkSSHCommands()`, `loginUser()`, `validatePasswordCredentials()`, `validatePublicKeyCredentials()`
- **Runtime state management:** `sftpd/sftpd.go` — Contains `supportedSSHCommands`, `systemCommands`, `sshHashCommands`, `executeAction()`, `executeNotificationCommand()`
- **Transfer engine:** `sftpd/transfer.go` — Contains `Transfer` struct, `ReadAt()`, `WriteAt()`, `Close()`, `copyFromReaderToWriter()`
- **User model and permissions:** `dataprovider/user.go` — Contains `User` struct, `HasPerm()`, `HasPerms()`, `GetPermissionsForPath()`, `IsLoginAllowed()`
- **Authentication and user management:** `dataprovider/dataprovider.go` — Contains `CheckUserAndPass()`, `CheckUserAndPubKey()`, `doExternalAuth()`, `validateUser()`, `checkLoginConditions()`
- **Filesystem abstraction and path resolution:** `vfs/osfs.go` — Contains `ResolvePath()`, `isSubDir()`, `findFirstExistingDir()`, `findNonexistentDirs()`
- **Process credential wrapping:** `sftpd/cmd_unix.go` — Contains `wrapCmd()` for UID/GID credential assignment
- **Configuration:** `sftpgo.json` — Default configuration including `enabled_ssh_commands`, `config/config.go` — Configuration loading and validation

**Key directories examined:**

| Directory | Files Analyzed | Relevance |
|-----------|---------------|-----------|
| `sftpd/` | 12 files (all) | PRIMARY — contains all SSH/SFTP/SCP server logic |
| `dataprovider/` | 9 files (all) | PRIMARY — user model, permissions, authentication |
| `vfs/` | 6 files (all) | PRIMARY — filesystem isolation and path resolution |
| `config/` | 4 files (all) | SECONDARY — configuration validation |
| `utils/` | 4 files (all) | SECONDARY — helper functions, encryption |

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. The user explicitly stated: "I don't want theoretical explanations of what the code should do. I want to see actual evidence of how the system behaves in practice." All documentation content is derived exclusively from code analysis of the repository, which aligns with the SWE-AtlasQnA-Repo implementation rule: "Do not make assumptions, base your answers on the code as the truth."

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation to comprehensively answer the user's five security questions:

- **Module: `sftpd/ssh_cmd.go`**
  - Public APIs: `processSSHCommand()`, `sshCommand.handle()`, `sshCommand.getSystemCommand()`, `sshCommand.executeSystemCommand()`, `sshCommand.handleHashCommands()`, `sshCommand.getDestPath()`, `parseCommandPayload()`
  - Current documentation: No dedicated documentation exists. Inline code comments are sparse.
  - Documentation needed: Complete walkthrough of command allowlisting, argument transformation, path resolution sequencing, credential wrapping, and system command execution pipeline.

- **Module: `sftpd/server.go`**
  - Key APIs: `Configuration.AcceptInboundConnection()`, `Configuration.checkSSHCommands()`, `loginUser()`, `processSSHCommand()` dispatch
  - Current documentation: Configuration struct has GoDoc comments. No flow documentation.
  - Documentation needed: Session routing logic, channel type filtering (`"session"` only), request type dispatch (`"subsystem"` vs `"exec"`), and the point where SSH exec payloads are handed to command processing.

- **Module: `sftpd/scp.go`**
  - Key APIs: `scpCommand.handle()`, `parseUploadMessage()`, `readProtocolMessage()`, `getNextUploadProtocolMessage()`, `handleRecursiveUpload()`, `handleUpload()`, `handleDownload()`
  - Current documentation: Comments note SCP is experimental. No protocol parsing documentation.
  - Documentation needed: Analysis of the SCP protocol message parsing assumptions (space-delimited, newline-terminated, single-character prefix), edge cases, and where validation occurs or is absent.

- **Module: `sftpd/handler.go`**
  - Key APIs: `Fileread()`, `Filewrite()`, `Filecmd()`, `Filelist()`, `handleSFTPSetstat()`, `handleSFTPRename()`, `handleSFTPRmdir()`, `handleSFTPRemove()`, `handleSFTPSymlink()`, `handleSFTPMkdir()`
  - Current documentation: GoDoc comments on exported methods. No enforcement-order documentation.
  - Documentation needed: Precise ordering of `ResolvePath()` vs `HasPerm()` calls in each handler, showing where path resolution and permission checks occur relative to each other.

- **Module: `sftpd/sftpd.go`**
  - Key APIs: `executeAction()`, `executeNotificationCommand()`, `supportedSSHCommands`, `systemCommands`, `sshHashCommands`
  - Current documentation: Minimal GoDoc. No security boundary documentation.
  - Documentation needed: Action hook execution pipeline, how user-controlled data flows into external commands and HTTP notifications.

- **Module: `dataprovider/user.go`**
  - Key APIs: `HasPerm()`, `HasPerms()`, `GetPermissionsForPath()`, `IsLoginAllowed()`
  - Current documentation: GoDoc on exported methods. No algorithm walkthrough.
  - Documentation needed: Permission resolution algorithm (most-specific-match), IP filter evaluation order (deny-first).

- **Module: `vfs/osfs.go`**
  - Key APIs: `ResolvePath()`, `isSubDir()`, `findFirstExistingDir()`, `findNonexistentDirs()`, `GetRelativePath()`
  - Current documentation: GoDoc comments. No security analysis documentation.
  - Documentation needed: Path resolution chain, symlink evaluation via `filepath.EvalSymlinks`, string-prefix-based containment checking, and handling of non-existent intermediate directories.

- **Module: `sftpd/cmd_unix.go`**
  - Key APIs: `wrapCmd()`
  - Current documentation: None.
  - Documentation needed: How UID/GID credentials are applied to spawned system commands.

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented security flows:** There is no documentation anywhere in the repository that traces the complete lifecycle of an SSH exec request from receipt through allowlist validation, path resolution, permission checking, to command execution. This is the primary gap.
- **No protocol parsing analysis:** The SCP protocol message format (`C0644 size name`, `D0755 0 dirname`) and its space-split parsing assumptions are not documented. The inline comment at `sftpd/scp.go:626–629` merely shows the format without analyzing edge cases.
- **No enforcement ordering documentation:** The relative ordering of `ResolvePath()` and `HasPerm()` calls varies between SFTP handlers, SSH command handlers, and SCP handlers. This variance is not documented.
- **No input transformation documentation:** The quote-stripping and `path.Clean()` normalization in `getDestPath()` (lines 356–371), the `filepath.ToSlash()` normalization in `GetPermissionsForPath()` (line 132), and the `filepath.Clean(filepath.Join(rootDir, sftpPath))` in `ResolvePath()` (line 204) represent a chain of transformations that collectively determine what the server "decides" from client input. This chain is entirely undocumented.
- **No rsync hardening documentation:** The server-side injection of `--safe-links` or `--munge-links` flags into rsync commands (lines 306–321 in `ssh_cmd.go`) is a significant security mechanism with no dedicated documentation.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/sftpgo_44634210287c.md` will follow this internal structure:

```
blitzy/documentation/sftpgo_44634210287c.md
├── Title and Preamble
├── 1. External Utility Invocation Security
│   ├── 1.1 Command Allowlist Mechanism
│   ├── 1.2 SSH Exec Payload Processing
│   ├── 1.3 System Command Preparation and Execution
│   ├── 1.4 Credential Wrapping (UID/GID)
│   └── 1.5 Rsync Symlink Hardening
├── 2. Client vs. Server Decision Boundary
│   ├── 2.1 Session Routing: Channel Types and Request Types
│   ├── 2.2 Command Parsing: What the Client Controls
│   ├── 2.3 Allowlist Filtering: What the Server Decides
│   ├── 2.4 Path Transformation Chain
│   └── 2.5 Post-Operation Action Hooks
├── 3. Input Validation and Protocol Parsing
│   ├── 3.1 SSH Exec Payload Parsing
│   ├── 3.2 SCP Protocol Message Parsing
│   ├── 3.3 SFTP Request Path Handling
│   └── 3.4 Edge Cases and Parsing Assumptions
├── 4. Permission Enforcement Ordering
│   ├── 4.1 SFTP Handler Enforcement Order
│   ├── 4.2 SSH Command Enforcement Order
│   ├── 4.3 SCP Handler Enforcement Order
│   └── 4.4 Comparative Analysis Across Protocols
├── 5. Filesystem Protection Mechanisms
│   ├── 5.1 ResolvePath() — The Chroot Gate
│   ├── 5.2 Symlink Evaluation and Containment
│   ├── 5.3 Non-Existent Path Handling
│   ├── 5.4 Coverage Across Access Paths
│   └── 5.5 String-Prefix Containment: Limitations
└── Summary and Key Findings
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract function signatures, control flow logic, and security-relevant code paths directly from `sftpd/ssh_cmd.go`, `sftpd/handler.go`, `sftpd/scp.go`, `sftpd/server.go`, `sftpd/sftpd.go`, `dataprovider/user.go`, `vfs/osfs.go`
  - Identify the exact line numbers where each security check occurs to provide precise citations
  - Map the call chains between modules to show the complete request processing pipeline

- **Documentation Standards:**
  - Markdown formatting with proper headers (`# ## ### ####`)
  - Mermaid diagrams for complex flow analysis (request routing, permission enforcement ordering)
  - Code-reference citations in the format `Source: /path/to/file.go:LineNumber`
  - Tables for comparative analysis (enforcement ordering across protocols)
  - Every claim backed by a specific file path and line reference

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **SSH Exec Request Lifecycle Flowchart:** Traces an SSH exec request from `AcceptInboundConnection()` through `processSSHCommand()` to either `sshCommand.handle()` or rejection. Source: `sftpd/server.go`, `sftpd/ssh_cmd.go`.

- **System Command Execution Sequence Diagram:** Shows the call sequence from `getSystemCommand()` through `ResolvePath()`, permission checking via `HasPerms()`, `wrapCmd()` credential assignment, to `cmd.Start()`. Source: `sftpd/ssh_cmd.go`, `sftpd/cmd_unix.go`.

- **Permission Enforcement Order Comparison Table:** Side-by-side comparison of the order of operations (ResolvePath, HasPerm, Stat, file operation) across SFTP handlers, SSH command handlers, and SCP handlers.

- **Path Resolution Chain Flowchart:** Shows the complete transformation from client-provided SFTP path through `getDestPath()` quote stripping, `path.Clean()`, `filepath.Join()`, `filepath.EvalSymlinks()`, to `isSubDir()` containment check. Source: `sftpd/ssh_cmd.go`, `vfs/osfs.go`.

- **SCP Protocol Parsing Diagram:** Illustrates the structure of SCP protocol messages and the space-split parsing logic in `parseUploadMessage()`. Source: `sftpd/scp.go`.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/sftpgo_44634210287c.md` | CREATE | `sftpd/ssh_cmd.go`, `sftpd/server.go`, `sftpd/handler.go`, `sftpd/scp.go`, `sftpd/sftpd.go`, `sftpd/cmd_unix.go`, `sftpd/transfer.go`, `dataprovider/user.go`, `dataprovider/dataprovider.go`, `vfs/osfs.go`, `config/config.go`, `sftpgo.json` | Comprehensive security analysis document answering all five user questions with code-level evidence, Mermaid diagrams, and line-number citations |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Technical Q&A analysis (security-focused)
Source Code:
    - sftpd/ssh_cmd.go (external utility invocation, command parsing, system command preparation)
    - sftpd/server.go (session routing, channel dispatch, authentication callbacks)
    - sftpd/handler.go (SFTP request handlers, permission enforcement)
    - sftpd/scp.go (SCP protocol parsing, upload/download handlers)
    - sftpd/sftpd.go (command registries, action hooks, runtime state)
    - sftpd/cmd_unix.go (UID/GID credential wrapping for child processes)
    - sftpd/transfer.go (transfer lifecycle, quota enforcement during I/O)
    - dataprovider/user.go (permission model, per-directory resolution, IP filtering)
    - dataprovider/dataprovider.go (authentication, external auth, user validation)
    - vfs/osfs.go (path resolution, chroot enforcement, symlink evaluation)
    - config/config.go (configuration loading, SSH command validation)
    - sftpgo.json (default enabled_ssh_commands, security defaults)
Sections:
    - Title and Preamble (methodology statement)
    - External Utility Invocation Security (command allowlist, exec payload processing, system command execution, credential wrapping, rsync hardening)
    - Client vs. Server Decision Boundary (session routing, command parsing, allowlist filtering, path transformation, action hooks)
    - Input Validation and Protocol Parsing (SSH exec parsing, SCP message parsing, SFTP path handling, edge cases)
    - Permission Enforcement Ordering (SFTP handler order, SSH command order, SCP handler order, comparative analysis)
    - Filesystem Protection Mechanisms (ResolvePath, symlink evaluation, non-existent path handling, coverage analysis, string-prefix limitations)
    - Summary and Key Findings
Diagrams:
    - SSH exec request lifecycle flowchart (Mermaid)
    - System command execution sequence diagram (Mermaid)
    - Path resolution chain flowchart (Mermaid)
    - Permission enforcement ordering comparison (table)
Key Citations:
    - sftpd/ssh_cmd.go:45-81 (processSSHCommand)
    - sftpd/ssh_cmd.go:288-331 (getSystemCommand)
    - sftpd/ssh_cmd.go:151-286 (executeSystemCommand)
    - sftpd/ssh_cmd.go:356-371 (getDestPath)
    - sftpd/ssh_cmd.go:423-429 (parseCommandPayload)
    - sftpd/server.go:243-333 (AcceptInboundConnection)
    - sftpd/server.go:396-414 (checkSSHCommands)
    - sftpd/sftpd.go:66-70 (supportedSSHCommands, systemCommands, sshHashCommands)
    - sftpd/sftpd.go:418-492 (executeNotificationCommand, executeAction)
    - sftpd/scp.go:631-663 (parseUploadMessage)
    - sftpd/scp.go:542-563 (readProtocolMessage)
    - sftpd/handler.go:47-95 (Fileread)
    - sftpd/handler.go:98-134 (Filewrite)
    - sftpd/handler.go:138-191 (Filecmd)
    - sftpd/handler.go:195-233 (Filelist)
    - dataprovider/user.go:120-156 (GetPermissionsForPath)
    - dataprovider/user.go:159-165 (HasPerm)
    - vfs/osfs.go:200-223 (ResolvePath)
    - vfs/osfs.go:278-291 (isSubDir)
    - vfs/osfs.go:250-276 (findFirstExistingDir)
    - sftpd/cmd_unix.go:10-16 (wrapCmd)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository has no documentation generator, build system, or navigation configuration. The output is a standalone Markdown file in a new `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The output document is self-contained with no dependencies on other documentation files.
- **No navigation links:** No existing documentation navigation to update.
- **No table of contents updates:** No existing ToC exists.
- **No index/glossary updates:** No existing index or glossary.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tools or packages are required for this task. The output is a single Markdown file with embedded Mermaid diagram syntax. The repository does not use any documentation generation framework.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `github.com/drakkan/sftpgo` | 0.9.5-dev | Primary source repository under analysis |
| Go module | `github.com/pkg/sftp` | v1.11.0 | SFTP library powering the server — referenced in documentation for request handler interfaces |
| Go module | `golang.org/x/crypto` | v0.0.0-20200109 | SSH library — referenced in documentation for `ssh.Unmarshal`, `ssh.Permissions`, channel handling |
| Go module | `github.com/spf13/viper` | v1.6.1 | Configuration library — referenced in documentation for `enabled_ssh_commands` loading |
| Go stdlib | `os/exec` | Go 1.13 | System command execution — referenced in documentation for `exec.Command` usage in `getSystemCommand()` |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates. The output document is a new standalone file with no inbound or outbound links to existing repository documentation.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **Current coverage analysis:**
  - Security flow documentation for SSH command handling: 0% — no existing documentation
  - Permission enforcement ordering documentation: 0% — no existing documentation
  - SCP protocol parsing analysis: 0% — no existing documentation
  - Path resolution/chroot mechanism documentation: 0% — no existing documentation
  - External utility invocation security documentation: 0% — no existing documentation

- **Target coverage:** 100% of the user's five stated questions, with every claim supported by code-level evidence (file path + line number).

- **Coverage gaps to address:**

| Topic | Current Coverage | Target | Focus Areas |
|-------|-----------------|--------|-------------|
| External utility invocation | 0% | 100% | Allowlist mechanism, command preparation, credential wrapping, rsync hardening |
| Client vs. server boundary | 0% | 100% | Session routing, exec dispatch, allowlist filtering, path transformation chain |
| Input validation / parsing | 0% | 100% | `parseCommandPayload()`, `parseUploadMessage()`, `readProtocolMessage()`, `getDestPath()` |
| Permission enforcement ordering | 0% | 100% | Ordering of ResolvePath vs HasPerm across SFTP, SSH, and SCP handlers |
| Filesystem protections | 0% | 100% | `ResolvePath()`, `isSubDir()`, symlink evaluation, non-existent path handling |

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every user question receives a dedicated section with a direct, evidence-based answer
  - All public APIs in the security-relevant call chains are documented with their file path, line number, and behavior
  - All Mermaid diagrams are syntactically correct and accurately represent the code flow

- **Accuracy validation:**
  - Every code citation must reference the correct file path and line numbers verified through `read_file` tool calls
  - All function signatures and variable names must match the actual source code exactly
  - The permission enforcement ordering must be verified by reading each handler's source and documenting the precise line numbers of `ResolvePath()` and `HasPerm()` calls

- **Clarity standards:**
  - Technical accuracy with accessible language — the user is evaluating SFTPGo, not writing its code
  - Progressive disclosure: start each section with a summary answer, then provide the detailed evidence
  - Consistent terminology: use "allowlist" not "whitelist", "SSH exec request" not "command request"

- **Maintainability:**
  - Source citations as inline references with file:line format throughout
  - Self-contained document requiring no external context to understand

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per security flow:** At least one complete code path trace per question
- **Diagram types required:** Mermaid flowcharts for request routing and path resolution; tables for comparative enforcement ordering
- **Code example testing:** Not applicable — no executable examples; all references are to existing source code
- **Visual content freshness:** Diagrams are derived from the current codebase state and are accurate as of this analysis

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation files:**
  - `blitzy/documentation/sftpgo_44634210287c.md` — the primary deliverable

- **Source code analysis targets (read-only, for evidence gathering):**
  - `sftpd/ssh_cmd.go` — SSH command processing, system command preparation, argument transformation
  - `sftpd/server.go` — Session routing, authentication callbacks, SSH command allowlist validation
  - `sftpd/handler.go` — SFTP request handlers, per-operation permission enforcement
  - `sftpd/scp.go` — SCP protocol parsing, upload/download handlers
  - `sftpd/sftpd.go` — Command registries (`supportedSSHCommands`, `systemCommands`, `sshHashCommands`), action hooks
  - `sftpd/cmd_unix.go` — UID/GID credential wrapping via `wrapCmd()`
  - `sftpd/transfer.go` — Transfer lifecycle, quota enforcement during I/O
  - `dataprovider/user.go` — Permission model, `HasPerm()`, `GetPermissionsForPath()`, IP filtering
  - `dataprovider/dataprovider.go` — Authentication flow, external auth, user validation, `doExternalAuth()`
  - `vfs/osfs.go` — `ResolvePath()`, `isSubDir()`, `findFirstExistingDir()`, `findNonexistentDirs()`
  - `vfs/vfs.go` — `Fs` interface definition, `IsLocalOsFs()`, `GetSFTPError()`
  - `config/config.go` — Configuration loading, SSH command validation
  - `sftpgo.json` — Default configuration values
  - `README.md` — Existing documentation for cross-reference

- **Security topics to cover:**
  - External utility invocation lifecycle (allowlisting, validation, execution)
  - Client vs. server decision boundary (what clients control, what servers decide)
  - Protocol-level input validation (SSH exec parsing, SCP message parsing, SFTP path handling)
  - Permission enforcement ordering across SFTP, SSH, and SCP code paths
  - Filesystem chroot/path traversal protections (ResolvePath, symlink evaluation, containment checks)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** Per user directive: "Don't modify any source files in the repository." No `.go`, `.json`, `.yml`, or any other existing repository file will be modified.
- **Test file modifications:** No test files (`sftpd/sftpd_test.go`, `sftpd/internal_test.go`, etc.) will be modified.
- **Feature additions or code refactoring:** No source code changes of any kind.
- **HTTP API security analysis:** While the HTTP server's security posture is noted in the tech spec (Section 6.4.3.5), the user's questions focus on the SSH/SFTP protocol layer, not the REST API.
- **Database backend security:** Not addressed — the user's questions concern the SSH processing chain, not data provider security.
- **S3 filesystem backend:** While `vfs/s3fs.go` exists, the user's questions about filesystem protections are specific to local filesystem path traversal, which is implemented in `vfs/osfs.go`.
- **Docker/deployment documentation:** Not related to the user's security model questions.
- **Migration scripts:** Not related to the user's questions.
- **Performance or benchmark documentation:** Not requested.
- **Theoretical security recommendations:** The user explicitly rejected theoretical explanations.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — no documentation generator is used. The output is a standalone Markdown file.
- **Documentation preview command:** Not applicable — Markdown can be previewed by any Markdown renderer.
- **Diagram generation command:** Not applicable — Mermaid diagrams are embedded inline within the Markdown and rendered by compatible Markdown viewers (GitHub, VS Code, etc.).
- **Documentation deployment command:** Not applicable — no documentation deployment pipeline exists.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every section must reference source files with the format `Source: path/to/file.go:LineNumber` or `(file.go:NN)`.
- **Style guide:** Evidence-based technical analysis. Each section follows the pattern: (1) summary answer, (2) code evidence with citations, (3) analysis and implications.
- **Documentation validation:** Manual review of Markdown syntax and Mermaid diagram correctness. No automated linting or link-checking tools are configured in this repository.

### 0.9.2 Output File Placement

The final documentation artifact must be placed at:

```
blitzy/documentation/sftpgo_44634210287c.md
```

This path is derived from the implementation rule `SWE-AtlasQnA-Repo`:
- Directory: `blitzy/documentation/` (in the destination repository)
- Filename: `sftpgo_44634210287c.md` (source branch name + `.md` extension)

The `blitzy/documentation/` directory must be created if it does not exist.

## 0.10 Rules for Documentation

The following rules govern the documentation output, derived from the user's explicit directives and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The user stated: "Don't modify any source files in the repository." This is an absolute, non-negotiable constraint.

- **Base all answers on the code as the truth.** Per the SWE-AtlasQnA-Repo rule: "Do not make assumptions, base your answers on the code as the truth." Every claim must be traceable to a specific source file and line number.

- **Provide thinking and rationale behind the answers.** Per the SWE-AtlasQnA-Repo rule, the document must explain the reasoning process, not just state conclusions.

- **No theoretical explanations.** The user explicitly stated: "I don't want theoretical explanations of what the code should do. I want to see actual evidence of how the system behaves in practice."

- **Test scripts must be cleaned up.** If any temporary test scripts are created to observe behavior, they must be deleted before task completion.

- **Place the output document at `blitzy/documentation/sftpgo_44634210287c.md`.** Per the SWE-AtlasQnA-Repo rule, the filename matches the source branch name.

- **Include source code citations throughout.** Every technical claim must cite the specific file path and line numbers from which the evidence is drawn.

- **Use Mermaid diagrams for complex flow visualization.** The request processing chain, permission enforcement ordering, and path resolution chain should be visualized with diagrams.

- **Document all five security topics comprehensively.** No question may be left partially answered. Each of the user's five areas of confusion must receive a dedicated, thorough section.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Primary Source Files (read in full):**

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `sftpd/ssh_cmd.go` | SSH command processing: `processSSHCommand()`, `getSystemCommand()`, `executeSystemCommand()`, `handleHashCommands()`, `getDestPath()`, `parseCommandPayload()` |
| `sftpd/server.go` | Server startup, authentication callbacks, session routing via `AcceptInboundConnection()`, SSH command allowlist via `checkSSHCommands()`, `loginUser()` pre-checks |
| `sftpd/handler.go` | SFTP request handlers with per-operation permission enforcement: `Fileread()`, `Filewrite()`, `Filecmd()`, `Filelist()`, and all `handleSFTP*` methods |
| `sftpd/scp.go` | SCP protocol implementation: `scpCommand.handle()`, `handleRecursiveUpload()`, `handleUpload()`, `handleDownload()`, `parseUploadMessage()`, `readProtocolMessage()` |
| `sftpd/sftpd.go` | Runtime state, command registries (`supportedSSHCommands`, `systemCommands`, `sshHashCommands`), `executeAction()`, `executeNotificationCommand()` |
| `sftpd/transfer.go` | Transfer lifecycle: `ReadAt()`, `WriteAt()`, `Close()`, `copyFromReaderToWriter()`, bandwidth throttling |
| `sftpd/cmd_unix.go` | UID/GID credential wrapping via `wrapCmd()` for spawned system commands |
| `dataprovider/user.go` | User model, permission constants, `HasPerm()`, `HasPerms()`, `GetPermissionsForPath()`, `IsLoginAllowed()`, `UserFilters` |
| `dataprovider/dataprovider.go` | Authentication: `CheckUserAndPass()`, `CheckUserAndPubKey()`, `doExternalAuth()`, `validateUser()`, `checkLoginConditions()`, external auth environment variable warning |
| `vfs/osfs.go` | Chroot enforcement: `ResolvePath()`, `isSubDir()`, `findFirstExistingDir()`, `findNonexistentDirs()`, `GetRelativePath()`, `CheckRootPath()` |
| `config/config.go` | Configuration loading and validation: `LoadConfig()`, default SSH commands, upload mode validation, external auth scope validation |
| `sftpgo.json` | Default runtime configuration: `enabled_ssh_commands`, `enable_scp`, `upload_mode`, `setstat_mode`, all security-relevant defaults |
| `go.mod` | Go module definition: Go 1.13, dependency versions including `pkg/sftp v1.11.0`, `golang.org/x/crypto` |
| `README.md` | Existing project documentation: features, SSH commands, configuration, authentication modes |
| `main.go` | Application entry point structure |

**Folders Explored (via `get_source_folder_contents`):**

| Folder Path | Files/Children Discovered | Relevance |
|-------------|--------------------------|-----------|
| `` (root) | 7 files, 14 folders | Repository structure discovery |
| `sftpd/` | 12 files | PRIMARY — all SSH/SFTP/SCP server logic |
| `dataprovider/` | 9 files | PRIMARY — user model, permissions, authentication |
| `vfs/` | 6 files | PRIMARY — filesystem abstraction, path resolution |
| `config/` | 4 files | SECONDARY — configuration validation |
| `utils/` | 4 files | SECONDARY — helper functions, encryption |

**Tech Spec Sections Retrieved:**

| Section Heading | Purpose |
|----------------|---------|
| 1.1 Executive Summary | Background context on SFTPGo's purpose and architecture |
| 6.4 Security Architecture | Comprehensive security model documentation for cross-reference |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma URLs or external design documents are associated with this task.

### 0.11.3 External References

No external web searches were performed. All conclusions are derived exclusively from the repository source code, consistent with the user's directive for code-based evidence and the SWE-AtlasQnA-Repo implementation rule.

