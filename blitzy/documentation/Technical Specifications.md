# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a comprehensive, code-grounded security audit analysis of command injection attack surface within the SFTPGo SFTP server codebase.

- **Request Category:** Create new documentation
- **Documentation Type:** Security audit analysis document (Q&A research format)
- **Target Output:** A markdown document named `sftpgo_44634210287c.md` placed in the `blitzy/documentation/` directory

The user has described a security audit scenario involving:
- A vulnerability scanner producing inconsistent results for command injection in SFTPGo
- A need to understand which specific component is vulnerable and under what configuration conditions
- A request to analyze exploitability by tracing code paths, identifying injection vectors, and documenting what controls block or permit exploitation
- A requirement to document exact payloads, expected server behavior, log entries, and error messages

**Documentation Requirements with Enhanced Clarity:**

- **R1 — Vulnerability Identification:** Identify and document the exact code components in SFTPGo that handle user-controlled input and pass it to command execution functions (`exec.Command`, `exec.CommandContext`), with file paths and line numbers
- **R2 — Configuration-Dependent Exploitability:** Explain why the scanner produces inconsistent results by mapping each command injection vector to the specific configuration settings that enable or disable it (e.g., `enabled_ssh_commands`, `actions.command`, `external_auth_program`)
- **R3 — Exploitation Analysis:** Document the theoretical attack surface for command injection — including the payload format, what the server would do with it, and what protective mechanisms exist (Go's `exec.Command` non-shell behavior, allow-list validation, `ResolvePath` chroot enforcement)
- **R4 — Attack Payload Documentation:** Provide the exact payload strings that a security auditor would use when testing, with explanation of why they succeed or fail against each code path
- **R5 — Server Response Documentation:** Document what the complete server response looks like during each test case — including SSH exit status, channel output, and error messages
- **R6 — Log Entry Documentation:** Document what entries appear in SFTPGo's structured JSON logs when SSH commands execute, including the logger functions involved (`logger.CommandLog`, `logger.ConnectionFailedLog`, `logger.Debug`)
- **R7 — Blocking Mechanism Documentation:** When exploitation attempts fail, document the exact error messages, the Go source function that produces them, and the architectural reason the attack is blocked

**Implicit Documentation Needs Surfaced:**

- The document must cover the full SSH command lifecycle: parsing (`parseCommandPayload`), validation (allow-list check), path resolution (`getDestPath`, `ResolvePath`), execution (`exec.Command`), and post-execution actions (`executeAction`, `executeNotificationCommand`)
- The document must distinguish between direct command injection (via SSH exec channel) and indirect injection (via action hooks and external auth programs)
- The document must address the `rsync` argument injection vector separately from shell metacharacter injection, as rsync accepts powerful options that could be abused without shell involvement

### 0.1.2 Special Instructions and Constraints

**Critical Directives Captured:**

- **Do not modify any source files in the repository.** The analysis is read-only; no existing Go files, configuration files, or test files shall be altered
- **Temporary test scripts or helper tools are permitted** if needed for analysis, but must be cleaned up — however, the implementation rules further state: "Do not modify any existing files in the source repository," meaning no file in the SFTPGo repository tree may be changed
- **The analysis document must be placed in `blitzy/documentation/sftpgo_44634210287c.md`** per the project implementation rule ("SWE-AtlasQnA-Repo")
- **Provide thinking and rationale behind all answers** — the document must not merely state conclusions but show the code-level reasoning that supports them
- **Do not make assumptions; base answers on the code as the truth** — every claim about vulnerability or safety must reference a specific file, function, and line number in the SFTPGo source

**Template Requirements:**
The document follows the project rule template: a comprehensive markdown Q&A document named after the source branch, placed in `blitzy/documentation/`.

**Style Preferences:**
- Code-referenced analysis with `Source: /path/to/file.go:LineNumber` citations
- Clear section structure separating each attack vector
- Distinction between theoretical vulnerability and practical exploitability
- All payloads and server responses documented precisely

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the command injection surface**, we will **create** `blitzy/documentation/sftpgo_44634210287c.md` by analyzing SSH command handling in `sftpd/ssh_cmd.go`, action hook execution in `sftpd/sftpd.go`, external auth delegation in `dataprovider/dataprovider.go`, and path resolution in `vfs/osfs.go`
- To **explain inconsistent scanner results**, we will **trace** configuration-dependent code paths in `sftpd/server.go` (`checkSSHCommands`, `EnabledSSHCommands`), `sftpgo.json` (`enabled_ssh_commands`, `actions`, `external_auth_program`), and `config/config.go` (validation and defaults)
- To **document payload behavior**, we will **analyze** the `parseCommandPayload` function (ssh_cmd.go:423-429), the `processSSHCommand` entry point (ssh_cmd.go:45-81), and the `getSystemCommand` function (ssh_cmd.go:288-331) to trace exactly how user input flows through execution
- To **document server responses and log entries**, we will **analyze** `sendExitStatus` (ssh_cmd.go:380-407), `sendErrorResponse` (ssh_cmd.go:373-378), and the logging functions in `logger/logger.go` (`CommandLog`, `ConnectionFailedLog`, `Debug`)

### 0.1.4 Inferred Documentation Needs

Based on thorough code analysis, additional documentation needs have been identified:

- **Go's `exec.Command` security model:** The document must explain that Go's `exec.Command` does NOT invoke a shell, which is the single most important architectural defense against traditional command injection. This is the likely root cause of inconsistent scanner findings — scanners assuming shell invocation where none exists
- **Argument injection vs. shell injection distinction:** The `rsync` and `git-*` system commands accept user-supplied arguments directly. While shell metacharacters won't be interpreted, `rsync`-specific flags (e.g., `--rsync-path`) constitute a separate class of argument injection risk
- **Action hook injection surface:** When `actions.command` is configured as a shell script, user-controlled values (username, path, SSH command string) are passed as command-line arguments to `exec.CommandContext`. If the shell script doesn't properly quote `"$@"`, secondary injection becomes possible
- **External auth environment variable injection:** The `doExternalAuth` function passes username and password as environment variables. If the external auth program is a shell script that uses `$SFTPGO_AUTHD_PASSWORD` without quoting, an attacker could inject shell commands through the password field

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal documentation structure** with coverage limited to README files and inline code comments. No dedicated documentation framework (MkDocs, Sphinx, Docusaurus) is configured.

**Documentation files discovered:**

| File | Type | Coverage Status |
|------|------|-----------------|
| `README.md` | Project overview and operator guide | Comprehensive feature list, installation, configuration, but no security audit documentation |
| `docker/README.md` | Docker deployment overview | Container-specific setup |
| `docker/sftpgo/alpine/README.md` | Alpine Docker variant | Image-specific instructions |
| `docker/sftpgo/debian/README.md` | Debian Docker variant | Image-specific instructions |
| `scripts/README.md` | REST API CLI usage guide | CLI tool documentation with examples |

**Documentation infrastructure findings:**

- **Documentation framework:** None configured — no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` found
- **Documentation generator:** None — no JSDoc, Godoc, or API doc generation tooling
- **API documentation:** OpenAPI 3.0.1 schema exists at `httpd/schema/openapi.yaml` — the authoritative REST API contract
- **Diagram tools:** None configured — no Mermaid CLI or PlantUML; the tech spec uses Mermaid diagrams inline
- **Documentation hosting:** None configured — no deployment pipeline for documentation
- **Security documentation:** None found — no dedicated security audit documents, vulnerability reports, or threat model documentation exists in the repository

**Key observation:** The `blitzy/documentation/` directory does not yet exist and must be created as part of this task.

### 0.2.2 Repository Code Analysis for Documentation

Extensive code analysis was performed targeting command execution and security-critical paths:

**Search patterns used for security-relevant code:**
- SSH command handling: `sftpd/ssh_cmd.go` — `parseCommandPayload`, `processSSHCommand`, `getSystemCommand`, `executeSystemCommand`, `handle`
- Action hook execution: `sftpd/sftpd.go` — `executeAction`, `executeNotificationCommand`, `Actions` struct
- External auth: `dataprovider/dataprovider.go` — `doExternalAuth`, `ExternalAuthProgram`, `ExternalAuthScope`
- Path resolution: `vfs/osfs.go` — `ResolvePath`, `isSubDir`, `findFirstExistingDir`
- Server configuration: `sftpd/server.go` — `checkSSHCommands`, `Initialize`, `AcceptInboundConnection`, `processSSHCommand`
- Configuration loading: `config/config.go` — `LoadConfig`, defaults initialization, validation

**Key directories examined:**

| Directory | Relevance | Files Analyzed |
|-----------|-----------|----------------|
| `sftpd/` | Primary attack surface — SSH/SFTP/SCP server implementation | `ssh_cmd.go`, `sftpd.go`, `server.go`, `handler.go`, `scp.go`, `internal_test.go` |
| `dataprovider/` | Authentication and external auth program execution | `dataprovider.go`, `user.go` |
| `vfs/` | Filesystem abstraction and chroot enforcement | `osfs.go`, `vfs.go` |
| `config/` | Configuration loading, validation, and defaults | `config.go`, `config_test.go` |
| `httpd/` | REST API and management interface | `router.go`, `httpd.go`, `schema/openapi.yaml` |
| `logger/` | Structured logging used in audit trails | `logger.go` |
| `cmd/` | CLI command definitions and server startup | `root.go`, `serve.go`, `portable.go` |

**Related documentation found:**
- `README.md` documents the `enabled_ssh_commands` configuration including the `*` wildcard for enabling all commands
- `README.md` documents custom commands and HTTP notification hooks on file operations and SSH commands
- `sftpgo.json` provides the default configuration showing `enabled_ssh_commands: ["md5sum", "sha1sum", "cd", "pwd"]`
- `httpd/schema/openapi.yaml` documents the REST API surface but contains no security audit information

### 0.2.3 Web Search Research Conducted

No external web searches were required for this task. The analysis is entirely code-grounded per the user's directive: "Do not make assumptions, base your answers on the code as the truth." All findings derive from direct source code inspection of the SFTPGo repository at the `sftpgo_44634210287c` branch.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The security audit document must cover every code module involved in processing user-controlled input through to command execution. The following mapping identifies each module, its public attack surface, current documentation status, and what the new document must cover.

**Module: `sftpd/ssh_cmd.go` — SSH Command Processing**
- Public APIs / Entry Points:
  - `processSSHCommand(payload, connection, channel, enabledSSHCommands)` — line 45
  - `parseCommandPayload(command string)` — line 423
  - `sshCommand.handle()` — line 83
  - `sshCommand.getSystemCommand()` — line 288
  - `sshCommand.executeSystemCommand(command)` — line 151
  - `sshCommand.getDestPath()` — line 356
  - `sshCommand.handleHashCommands()` — line 105
  - `sshCommand.sendExitStatus(err)` — line 380
  - `sshCommand.sendErrorResponse(err)` — line 373
- Current documentation: Inline comments only; no security analysis documentation exists
- Documentation needed: Complete attack surface analysis — how user SSH exec payloads flow through parsing, validation, path resolution, and command execution; why `exec.Command` prevents shell injection; what error/exit behaviors look like

**Module: `sftpd/sftpd.go` — Action Hook Execution**
- Public APIs / Entry Points:
  - `executeAction(operation, username, path, target, sshCmd, fileSize)` — line 438
  - `executeNotificationCommand(operation, username, path, target, sshCmd, fileSize)` — line 418
  - `Actions` struct (ExecuteOn, Command, HTTPNotificationURL) — line 91
- Current documentation: Minimal inline comments; no security analysis
- Documentation needed: How user-controlled data (username, file paths, SSH command strings) flows into external command execution and HTTP notification URLs; conditions under which this code path is active

**Module: `dataprovider/dataprovider.go` — External Authentication**
- Public APIs / Entry Points:
  - `doExternalAuth(username, password, pubKey)` — line 730
  - `executeNotificationCommand(operation, user)` — line 783
  - `Config.ExternalAuthProgram` — line 169
  - `Config.ExternalAuthScope` — line 175
- Current documentation: Configuration comments in README.md; no security analysis
- Documentation needed: How attacker-controlled credentials (username, password) are passed to the external auth program via environment variables; injection risks when the external program is a shell script

**Module: `vfs/osfs.go` — Path Resolution and Chroot Enforcement**
- Public APIs / Entry Points:
  - `OsFs.ResolvePath(sftpPath)` — line 200
  - `OsFs.isSubDir(sub, rootPath)` — line 278
  - `OsFs.findFirstExistingDir(path, rootPath)` — line 250
- Current documentation: Inline comments; documented in tech spec section 6.4
- Documentation needed: How path resolution blocks directory traversal attacks; interaction with SSH command argument processing

**Module: `sftpd/server.go` — SSH Server Configuration and Command Validation**
- Public APIs / Entry Points:
  - `Configuration.checkSSHCommands()` — line 396
  - `Configuration.Initialize(configDir)` — line 117
  - `Configuration.AcceptInboundConnection(conn, config)` — line 243
  - `Configuration.EnabledSSHCommands` — line 99
- Current documentation: Extensive inline comments; README.md coverage
- Documentation needed: How the allow-list validation controls which commands are executable; the `*` wildcard behavior; how SCP commands are routed separately

**Module: `sftpgo.json` — Default Configuration**
- Key settings: `enabled_ssh_commands`, `actions.execute_on`, `actions.command`, `external_auth_program`, `external_auth_scope`
- Current documentation: Documented in README.md
- Documentation needed: Security implications of each setting; which configurations create injection risk

### 0.3.2 Configuration Options Requiring Documentation

| Config Setting | File | Default Value | Security Implication |
|----------------|------|---------------|----------------------|
| `sftpd.enabled_ssh_commands` | `sftpgo.json` | `["md5sum","sha1sum","cd","pwd"]` | Controls which SSH commands can be executed; adding `rsync` or `git-*` enables system command execution |
| `sftpd.actions.execute_on` | `sftpgo.json` | `[]` (empty) | Determines which operations trigger external command execution |
| `sftpd.actions.command` | `sftpgo.json` | `""` (empty) | Path to external command; if a shell script, creates injection surface |
| `sftpd.actions.http_notification_url` | `sftpgo.json` | `""` (empty) | URL receiving user-controlled data in query parameters |
| `data_provider.external_auth_program` | `sftpgo.json` | `""` (empty) | Path to external auth program receiving credentials as env vars |
| `data_provider.external_auth_scope` | `sftpgo.json` | `0` | Scope 0 = all methods; scope 1 = password only; scope 2 = pubkey only |

### 0.3.3 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps exist:

- **No security audit documentation exists** — The repository contains zero dedicated security analysis, threat modeling, or vulnerability assessment documents
- **No command injection analysis** — Despite multiple `exec.Command` and `exec.CommandContext` call sites, there is no documentation analyzing injection risk
- **No configuration security guide** — The README documents features but does not document the security implications of configuration choices
- **No test coverage for injection payloads** — The `internal_test.go` tests cover path traversal but not shell metacharacter handling in SSH command payloads
- **No action hook security documentation** — The notification command execution path lacks any documentation about the injection risk when shell scripts are used as action handlers

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/sftpgo_44634210287c.md` will follow this structure:

```
blitzy/
└── documentation/
    └── sftpgo_44634210287c.md
        ├── Introduction (Audit Context and Methodology)
        ├── Why the Scanner Produces Inconsistent Results
        │   ├── Configuration-Dependent Attack Surface
        │   ├── Default Configuration Analysis
        │   └── High-Risk Configuration Combinations
        ├── Command Injection Vector Analysis
        │   ├── Vector 1: SSH Exec Channel — Direct Command Injection
        │   │   ├── parseCommandPayload Analysis
        │   │   ├── Allow-List Validation
        │   │   ├── exec.Command (No Shell) Protection
        │   │   └── Exploitation Assessment
        │   ├── Vector 2: System Commands — Argument Injection via rsync/git
        │   │   ├── getSystemCommand Analysis
        │   │   ├── rsync Argument Injection Risk
        │   │   └── Exploitation Assessment
        │   ├── Vector 3: Action Hook Commands — Indirect Injection
        │   │   ├── executeNotificationCommand Analysis
        │   │   ├── Shell Script Argument Handling Risk
        │   │   └── Exploitation Assessment
        │   ├── Vector 4: External Auth Program — Environment Variable Injection
        │   │   ├── doExternalAuth Analysis
        │   │   ├── Shell Script Environment Variable Risk
        │   │   └── Exploitation Assessment
        │   └── Vector 5: HTTP Notification URL — Query Parameter Injection
        │       ├── executeAction HTTP Path Analysis
        │       └── Exploitation Assessment
        ├── Exploitation Attempt Documentation
        │   ├── Test Payloads and Expected Behavior
        │   ├── Server Response Documentation
        │   ├── Log Entry Documentation
        │   └── Error Messages and Blocking Mechanisms
        ├── Protective Mechanisms Summary
        │   ├── exec.Command Non-Shell Architecture
        │   ├── Allow-List Command Validation
        │   ├── ResolvePath Chroot Enforcement
        │   └── Permission System Enforcement
        └── Conclusions and Recommendations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract SSH command processing flow from `sftpd/ssh_cmd.go` by tracing the call chain: `processSSHCommand` → `parseCommandPayload` → allow-list check → `sshCommand.handle` → `getSystemCommand` → `exec.Command`
- Extract action hook execution flow from `sftpd/sftpd.go` by tracing: `executeAction` → `executeNotificationCommand` → `exec.CommandContext(ctx, actions.Command, operation, username, path, target, sshCmd)`
- Extract external auth flow from `dataprovider/dataprovider.go` by tracing: `doExternalAuth` → `exec.CommandContext(ctx, config.ExternalAuthProgram)` with env vars `SFTPGO_AUTHD_USERNAME`, `SFTPGO_AUTHD_PASSWORD`, `SFTPGO_AUTHD_PUBLIC_KEY`
- Generate exploitation test payloads by analyzing `parseCommandPayload`'s naive `strings.Split(command, " ")` behavior and how it interacts with `exec.Command`'s argument passing
- Extract server response format from `sendExitStatus` (ssh_cmd.go:380-407) and `sendErrorResponse` (ssh_cmd.go:373-378)
- Extract log entry format from `logger/logger.go` (`CommandLog`, `Debug`) and the zerolog structured JSON output

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Code examples using fenced code blocks with Go syntax highlighting
- Source citations in format: `Source: sftpd/ssh_cmd.go:423-429`
- Tables for parameter descriptions, configuration mappings, and test case results
- Consistent terminology: "command injection," "argument injection," "shell metacharacter," "allow-list," "chroot enforcement"

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create within the document:**

- **SSH Command Processing Flow:** A flowchart tracing user SSH exec payload through `parseCommandPayload` → allow-list check → command routing (hash commands / system commands / cd / pwd) → `exec.Command` execution → exit status return
- **Attack Vector Decision Tree:** A diagram showing which configuration conditions activate each injection vector and what protective mechanisms block each path
- **Action Hook Injection Flow:** A sequence diagram showing how user-initiated SSH commands trigger `executeAction` → `executeNotificationCommand` and how user-controlled data flows into the external command arguments
- **Configuration-to-Risk Matrix:** A visual mapping of configuration settings to their security implications

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/sftpgo_44634210287c.md` | CREATE | `sftpd/ssh_cmd.go`, `sftpd/sftpd.go`, `sftpd/server.go`, `sftpd/handler.go`, `dataprovider/dataprovider.go`, `vfs/osfs.go`, `config/config.go`, `sftpgo.json`, `logger/logger.go` | Complete security audit analysis document covering command injection vectors, exploitation analysis, server responses, log entries, and protective mechanisms |

**Note:** This task involves creation of exactly one new documentation file. No existing files in the repository are modified, updated, or deleted per the user's explicit instruction and the project implementation rules.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Security Audit Analysis (Q&A Research Document)
Source Code References:
  - sftpd/ssh_cmd.go (primary — SSH command parsing, system command execution)
  - sftpd/sftpd.go (action hook execution, notification commands)
  - sftpd/server.go (SSH command validation, connection handling)
  - sftpd/handler.go (SFTP request handling, permission enforcement)
  - dataprovider/dataprovider.go (external auth program execution)
  - dataprovider/user.go (user model, permissions, filters)
  - vfs/osfs.go (path resolution, chroot enforcement)
  - config/config.go (configuration loading, validation, defaults)
  - sftpgo.json (default configuration values)
  - logger/logger.go (structured JSON logging functions)
  - sftpd/internal_test.go (existing test coverage for SSH commands)
Sections:
  - Introduction: Audit context, methodology, and scope definition
  - Scanner Inconsistency Explanation: Configuration-dependent attack surface analysis
  - Vector 1 — Direct SSH Command Injection: parseCommandPayload + exec.Command analysis
  - Vector 2 — System Command Argument Injection: rsync/git argument handling
  - Vector 3 — Action Hook Indirect Injection: executeNotificationCommand analysis
  - Vector 4 — External Auth Environment Injection: doExternalAuth analysis
  - Vector 5 — HTTP Notification URL Injection: query parameter construction
  - Exploitation Attempt Documentation: Test payloads with exact format, expected behavior,
    server responses, log entries, and error messages for each vector
  - Protective Mechanisms: exec.Command non-shell model, allow-list, ResolvePath chroot,
    permission system
  - Conclusions: Risk assessment per configuration profile and remediation recommendations
Diagrams:
  - SSH Command Processing Flow (Mermaid flowchart)
  - Attack Vector Decision Tree (Mermaid flowchart)
  - Action Hook Injection Flow (Mermaid sequence diagram)
Key Citations:
  - sftpd/ssh_cmd.go:423-429 (parseCommandPayload — naive space splitting)
  - sftpd/ssh_cmd.go:324 (exec.Command invocation — no shell)
  - sftpd/ssh_cmd.go:45-51 (processSSHCommand — allow-list check)
  - sftpd/ssh_cmd.go:288-331 (getSystemCommand — argument construction)
  - sftpd/ssh_cmd.go:356-371 (getDestPath — quote stripping, path normalization)
  - sftpd/ssh_cmd.go:380-407 (sendExitStatus — exit code and logging)
  - sftpd/sftpd.go:418-435 (executeNotificationCommand — external command with user data)
  - sftpd/sftpd.go:438-492 (executeAction — action dispatch and HTTP notification)
  - sftpd/sftpd.go:66-70 (supportedSSHCommands and systemCommands lists)
  - dataprovider/dataprovider.go:730-777 (doExternalAuth — env var credential passing)
  - dataprovider/dataprovider.go:783-793 (executeNotificationCommand — user data as args)
  - vfs/osfs.go:200-223 (ResolvePath — chroot enforcement)
  - vfs/osfs.go:278-291 (isSubDir — path boundary validation)
  - sftpd/server.go:396-414 (checkSSHCommands — allow-list validation)
  - sftpd/server.go:326-327 (AcceptInboundConnection — exec request routing)
  - config/config.go (configuration defaults and validation)
  - sftpgo.json:22 (enabled_ssh_commands default: md5sum, sha1sum, cd, pwd)
  - sftpgo.json:10-14 (actions default: empty execute_on, empty command)
  - sftpgo.json:43-44 (external_auth_program default: empty)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The project does not use a documentation framework (no `mkdocs.yml`, `docusaurus.config.js`, or similar). The new file is a standalone markdown document placed in the `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The new document is self-contained
- **No navigation links required:** No documentation site to update
- **No table of contents updates:** No documentation index exists
- **No index or glossary updates:** None exist in the repository
- **Reference to existing documentation:** The new document should cross-reference `README.md` sections on `enabled_ssh_commands` and custom commands/HTTP notifications for context, using relative links

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tools or packages are required for this task. The output is a single standalone markdown file created directly in the repository. The document uses standard Markdown syntax with Mermaid diagram notation embedded inline (rendered by GitHub and other Markdown viewers).

**Relevant project dependencies referenced in the analysis (from `go.mod`):**

| Registry | Package Name | Version | Relevance to Security Audit |
|----------|--------------|---------|----------------------------|
| Go modules | `golang.org/x/crypto` | v0.0.0-20200109152110 | SSH library providing `ssh.Channel`, `ssh.Permissions`, `exec` channel handling — the transport layer through which SSH command payloads arrive |
| Go modules | `github.com/pkg/sftp` | v1.11.0 | SFTP protocol library; SFTP subsystem request handling — distinct from the SSH exec channel where command injection occurs |
| Go modules | `github.com/spf13/cobra` | v0.0.5 | CLI framework; defines `serve` and `portable` commands that start the server with specific `EnabledSSHCommands` |
| Go modules | `github.com/spf13/viper` | v1.6.1 | Configuration framework; loads `sftpgo.json` including `enabled_ssh_commands`, `actions`, and `external_auth_program` |
| Go modules | `github.com/rs/zerolog` | v1.17.2 | Structured JSON logging library; produces the log entries documented in the audit (CommandLog, ConnectionFailedLog) |
| Go modules | `github.com/go-chi/chi` | v4.0.2+incompatible | HTTP router; relevant to HTTP notification URL handling in `executeAction` |
| Go stdlib | `os/exec` | (Go 1.13) | Standard library `exec.Command` and `exec.CommandContext` — the critical functions that execute system commands WITHOUT a shell |

**Runtime version:**

| Component | Version | Source |
|-----------|---------|--------|
| Go | 1.13 | `go.mod` line 3 |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. This task creates a new standalone document without modifying any existing files or links in the repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**

| Category | Documented | Total | Coverage |
|----------|-----------|-------|----------|
| `exec.Command` / `exec.CommandContext` call sites analyzed | 0 | 4 | 0% |
| SSH command processing functions documented | 0 | 9 | 0% |
| Configuration settings with security implications documented | 0 | 6 | 0% |
| Action hook execution paths documented | 0 | 2 | 0% |
| External auth injection risks documented | 0 | 1 | 0% |

**Target coverage: 100%** — Every command execution call site, every SSH command processing function, every security-relevant configuration setting, and every injection vector must be fully documented in the new audit document.

**Coverage gaps to address:**

| Component | Current | Target | Focus Areas |
|-----------|---------|--------|-------------|
| `sftpd/ssh_cmd.go` — command execution | 0% | 100% | `parseCommandPayload`, `getSystemCommand`, `executeSystemCommand`, `processSSHCommand`, `handleHashCommands`, `getDestPath`, `sendExitStatus`, `sendErrorResponse`, `handle` |
| `sftpd/sftpd.go` — action hooks | 0% | 100% | `executeAction`, `executeNotificationCommand`, HTTP notification URL construction |
| `dataprovider/dataprovider.go` — external auth | 0% | 100% | `doExternalAuth`, `executeNotificationCommand` (dataprovider variant), env var injection |
| `vfs/osfs.go` — path enforcement | 0% | 100% | `ResolvePath`, `isSubDir`, chroot boundary validation |
| `sftpd/server.go` — command validation | 0% | 100% | `checkSSHCommands`, `EnabledSSHCommands`, wildcard behavior |
| `sftpgo.json` — security configuration | 0% | 100% | All 6 security-relevant settings with default values and risk implications |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every `exec.Command` and `exec.CommandContext` call site must include: the file path and line number, what user-controlled data enters the call, what protective mechanisms exist, and the exploitability assessment
- Every SSH command processing function must include: its role in the command lifecycle, what inputs it receives, what validation it performs, and what it passes downstream
- Every configuration setting must include: the default value, the security implication of changing it, and under what values the attack surface expands
- Every injection vector must include: the attack payload format, the code path it traverses, the expected server response (success or failure), and the log entries produced
- Every blocking mechanism must include: the function name and file, the error message returned, and the architectural reason the attack fails

**Accuracy validation:**
- All code references must cite exact file names and line numbers verified against the repository at branch `sftpgo_44634210287c`
- All payload examples must be syntactically valid SSH exec requests
- All server response descriptions must be derived from actual `sendExitStatus` and `sendErrorResponse` logic in the code
- All log entry formats must match the `zerolog` structured JSON format used by `logger/logger.go`

**Clarity standards:**
- Technical accuracy with precise Go and SSH terminology
- Progressive disclosure: summary finding → code evidence → detailed analysis → exploitation assessment
- Clear separation between theoretical vulnerability (code pattern suggests risk) and practical exploitability (default configuration blocks it)
- Consistent terminology: "shell injection" vs. "argument injection" vs. "environment variable injection" used precisely

**Maintainability:**
- Source code citations at every technical claim for traceability
- Section structure that allows individual vectors to be updated independently
- Configuration-referenced analysis that remains valid as long as the cited code paths exist

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per injection vector:** 2 (one showing the payload that would be needed, one showing the blocking mechanism or error response)
- **Diagram types required:** Mermaid flowcharts for SSH command processing and attack vector decision tree; Mermaid sequence diagram for action hook injection flow
- **Code example testing:** All payload examples are derived from code analysis (tracing through `parseCommandPayload` → allow-list check → `exec.Command` arguments) rather than live execution, per the read-only analysis constraint
- **Visual content freshness:** Diagrams reference specific file names and line numbers for traceability

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/sftpgo_44634210287c.md` — The complete security audit analysis document

**Source code files analyzed for documentation (read-only — no modifications):**
- `sftpd/ssh_cmd.go` — SSH command parsing, system command execution, hash command handling, path resolution, exit status handling
- `sftpd/sftpd.go` — Action hook execution, notification command invocation, HTTP notification URL construction, supported command lists, connection state management
- `sftpd/server.go` — SSH server initialization, command allow-list validation (`checkSSHCommands`), connection acceptance and exec request routing, authentication callbacks
- `sftpd/handler.go` — SFTP request handlers, permission enforcement, chroot enforcement via VFS, quota checking
- `sftpd/scp.go` — SCP protocol handling (related attack surface)
- `sftpd/cmd_unix.go` — Unix-specific process credential wrapping (`wrapCmd`)
- `sftpd/internal_test.go` — Existing test coverage analysis for SSH commands and path handling
- `dataprovider/dataprovider.go` — External auth program execution, data provider notification command execution, configuration validation
- `dataprovider/user.go` — User model, permission constants, IP filtering, login validation
- `vfs/osfs.go` — `ResolvePath`, `isSubDir`, `findFirstExistingDir` — chroot enforcement
- `vfs/vfs.go` — Filesystem interface definition, SFTP error translation
- `config/config.go` — Configuration loading, defaults initialization, validation of SSH commands and auth scope
- `sftpgo.json` — Default configuration with all security-relevant settings
- `logger/logger.go` — Structured logging functions (`CommandLog`, `ConnectionFailedLog`, `Debug`, `TransferLog`)
- `go.mod` — Dependency versions (Go 1.13, x/crypto, pkg/sftp, os/exec)
- `README.md` — Existing documentation for cross-reference

**Documentation assets:**
- Mermaid diagrams embedded within the markdown document (SSH command flow, attack vector decision tree, action hook injection flow)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No Go source files, configuration files, test files, or any other existing repository files will be modified, per the user's explicit instruction and the project implementation rule
- **Active exploitation or penetration testing:** The analysis is code-grounded and read-only; no server instances will be started, no SSH connections will be made, no payloads will be executed against a running SFTPGo instance
- **Test file modifications:** No changes to `sftpd/internal_test.go`, `sftpd/sftpd_test.go`, `config/config_test.go`, or `httpd/httpd_test.go`
- **Feature additions or code refactoring:** No code changes of any kind
- **Deployment configuration changes:** No modifications to Docker files, systemd configs, Travis CI configs, or Fail2ban configs
- **REST API / HTTP API security analysis:** The HTTP API is localhost-only and unauthenticated by design — this is documented in the tech spec but is not the focus of this command injection audit
- **S3 backend analysis:** The S3 filesystem backend (`vfs/s3fs.go`) does not support system command execution (`IsLocalOsFs` check blocks it) and is out of scope
- **Password hash algorithm analysis:** The multi-algorithm password validation in `dataprovider/dataprovider.go` is authentication security, not command injection, and is out of scope
- **Unrelated documentation:** README updates, Docker documentation, scripts documentation, and OpenAPI schema changes are not in scope

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — No documentation build system; the output is a standalone markdown file
- **Documentation preview command:** N/A — Standard markdown rendering (e.g., GitHub, VS Code)
- **Diagram generation command:** N/A — Mermaid diagrams are embedded inline in the markdown and rendered by supporting viewers
- **Documentation deployment command:** N/A — No documentation hosting configured
- **Default format:** Markdown with Mermaid diagrams, fenced code blocks with Go syntax highlighting
- **Citation requirement:** Every technical claim must reference a specific source file and line number in the format `Source: path/to/file.go:LineNumber` or `Source: path/to/file.go:StartLine-EndLine`
- **Style guide:** Follow the project implementation rule template — comprehensive Q&A markdown document with thinking/rationale behind answers, grounded in code evidence
- **Documentation validation:** Manual review for completeness against the 5 injection vectors, 6 configuration settings, and 9 SSH command processing functions identified in the scope analysis

### 0.9.2 Analysis Methodology Parameters

The security audit document must follow this methodology:

- **For each command execution call site (`exec.Command` / `exec.CommandContext`):**
  - Identify the function, file, and line number
  - Trace the data flow from user input to command arguments
  - Identify what sanitization or validation occurs on the path
  - Determine exploitability given the default configuration
  - Determine exploitability given the most permissive configuration
  - Document the exact payload format, expected server behavior, and log output

- **For each configuration setting:**
  - Document the default value from `sftpgo.json`
  - Document the validation logic from `config/config.go`
  - Document the security implication of the default value
  - Document the security implication of the most permissive value

- **For the overall inconsistency explanation:**
  - Map each scanner configuration to the specific code paths it exercises
  - Explain why the same scanner can report "vulnerable" with one config and "not vulnerable" with another
  - Provide a clear configuration-to-risk matrix

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project implementation rules:

- **Do not modify any existing files in the source repository.** The analysis document is created in `blitzy/documentation/` only. No Go source files, test files, configuration files, or existing markdown files are altered.
- **Do not make assumptions; base all answers on the code as the truth.** Every vulnerability claim, exploitability assessment, and protective mechanism description must cite specific source code with file path and line number.
- **Provide thinking and rationale behind all answers.** The document must not merely state "vulnerable" or "not vulnerable" but show the complete reasoning chain: what the code does → what an attacker would need to do → what blocks or permits the attack → what the observable result would be.
- **Create the document as `blitzy/documentation/sftpgo_44634210287c.md`.** The filename matches the source branch name per the "SWE-AtlasQnA-Repo" implementation rule.
- **Place the document in the `blitzy/documentation` directory.** This directory must be created if it does not exist.
- **Temporary test scripts or helper tools may be created if needed, but must be cleaned up.** The codebase must be left unchanged when the task is complete. In practice, this task requires no temporary scripts — the analysis is code-reading only.
- **The document must comprehensively answer all questions posed in the prompt.** This includes:
  - What conditions make the vulnerability exploitable
  - Which component is the problem
  - Exact payload strings for testing
  - Complete server response during attack attempts
  - Exact filename and content for any created files (or explanation of why file creation is blocked)
  - Server log entries during command execution
  - Exact error messages when exploitation fails and what is blocking it
- **Add source code citations for all technical details.** Every function, variable, configuration setting, and behavior cited must include the file path and line number reference.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were inspected to derive the conclusions in this Agent Action Plan:

**Primary source files analyzed (full content retrieved):**

| File | Lines | Purpose in Analysis |
|------|-------|---------------------|
| `sftpd/ssh_cmd.go` | 1–430 | SSH command parsing (`parseCommandPayload`), system command construction (`getSystemCommand`), execution (`executeSystemCommand`), hash command handling, path resolution (`getDestPath`), error and exit status handling |
| `sftpd/sftpd.go` | 1–493 | Action hook execution (`executeAction`, `executeNotificationCommand`), HTTP notification URL construction, supported/default/system command lists, connection state management, `Actions` struct definition |
| `sftpd/server.go` | 1–487 | SSH server configuration (`Configuration` struct), `checkSSHCommands` allow-list validation, `Initialize` server startup, `AcceptInboundConnection` with exec request routing to `processSSHCommand`, authentication callbacks |
| `sftpd/handler.go` | 1–566 | SFTP request handlers (`Fileread`, `Filewrite`, `Filecmd`, `Filelist`), permission enforcement, `ResolvePath` usage, `hasSpace` quota enforcement, root directory protection |
| `sftpd/scp.go` | 1–80 | SCP command handling entry point and recursive upload/download structure |
| `sftpd/internal_test.go` | 517–900 (selected) | Existing test coverage for SSH command paths, system command errors, `getDestPath` path traversal tests |
| `dataprovider/dataprovider.go` | 1–100, 725–800 | External auth program execution (`doExternalAuth`), data provider notification command (`executeNotificationCommand`), `Config` struct with `ExternalAuthProgram` and `ExternalAuthScope`, configuration validation |
| `vfs/osfs.go` | 1–291 | `ResolvePath` chroot enforcement, `isSubDir` boundary validation, `findFirstExistingDir` path walking, `OsFs` struct and all filesystem operation implementations |
| `config/config.go` | (via folder summary) | Configuration loading, Viper integration, default SSH commands, upload mode validation, external auth scope validation |
| `sftpgo.json` | 1–53 | Default configuration: `enabled_ssh_commands`, `actions` (execute_on, command, http_notification_url), `external_auth_program`, `external_auth_scope` |
| `go.mod` | 1–30 | Go 1.13 version, dependency versions for x/crypto, pkg/sftp, cobra, viper, zerolog, chi |
| `main.go` | (via folder summary) | Application entry point — imports SQL drivers and delegates to `cmd.Execute()` |
| `README.md` | 1–60 (partial) | Existing feature documentation, installation instructions |

**Folders explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| `/` (root) | Level 0 | Repository structure discovery |
| `sftpd/` | Level 1 | Primary attack surface — all 12 Go files inspected |
| `dataprovider/` | Level 1 | Authentication and external auth — key files inspected |
| `vfs/` | Level 1 | Filesystem abstraction — all 6 files assessed |
| `config/` | Level 1 | Configuration — all 4 files assessed |
| `httpd/` | Level 1 | REST API — structure and schema inspected |
| `httpd/schema/` | Level 2 | OpenAPI specification assessed |
| `cmd/` | Level 1 | CLI commands — structure assessed |
| `logger/` | Level 1 | Structured logging — all 3 files assessed |
| `scripts/` | Level 1 | REST API CLI — structure assessed |
| `docker/` | Level 1 | Container configs — structure assessed |

**Tech spec sections retrieved:**
- Section 1.1 Executive Summary — Project context, architecture overview, stakeholder analysis
- Section 6.4 Security Architecture — Authentication framework, authorization system, data protection, chroot enforcement, fail2ban integration, security control matrix

### 0.11.2 Attachments

No attachments were provided for this project. No Figma URLs, design files, or external documents were supplied.

### 0.11.3 External Resources

No external web searches were conducted. All analysis is derived from direct repository source code inspection at branch `sftpgo_44634210287c`.

