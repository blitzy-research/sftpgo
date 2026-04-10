# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of runtime-behavior questions about the SFTPGo server by observing its actual live behavior rather than relying solely on static code analysis. The request falls into the category of **Create new documentation** and the documentation type is a hybrid **Technical Investigation Guide / Q&A Document**.

The user is onboarding into the SFTPGo repository and wants a single authoritative markdown document that answers the following observational questions grounded in real runtime evidence:

- **Startup without configuration**: Which ports open and which log messages appear when SFTPGo launches without a configuration file, revealing the defaults it falls back to?
- **Failed SFTP authentication**: What appears in the server logs when an SFTP client connects with a nonexistent username, and how does the SSH handshake surface the rejection?
- **Web admin first access**: What HTTP response does the root endpoint (`/`) return, and how does the browser end up at the admin interface?
- **Missing database on first startup**: What errors or warnings appear when the SQLite database file does not exist, and what steps does the server go through before it either reaches a running state or halts?
- **Default configuration disclosure through logs**: Which bind addresses, timeouts, SSH commands, provider settings, and other values are revealed in log output when no configuration file is found?

Implicit documentation needs surfaced from the requirements:

- The document must distinguish between **structured JSON log lines** (the primary logger) and **console-level messages** (the console logger), since SFTPGo emits to both channels during startup.
- The document must trace the **full startup lifecycle** from `main.go` → `cmd.Execute()` → `service.Start()` → config loading → data provider initialization → SFTP listener binding → HTTP listener binding.
- The runtime observations must be grounded in **evidence from actual execution** of the SFTPGo v0.9.5-dev binary, not hypothetical descriptions.
- The user explicitly requires that **no repository files are modified** and that any temporary scripts used for observation are cleaned up afterward.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL**: The implementation rule `SWE-AtlasQnA-Repo` mandates that the output is a new markdown document named `sftpgo_44634210287c.md` placed in the `blitzy/documentation` directory in the destination repository.
- **No existing files may be modified** in the source repository. All observations are read-only; any temporary artifacts (database files, key files, log captures) must be cleaned up.
- The document must provide **thinking and rationale** behind the answers.
- Answers must be **based on the code as the truth** — no assumptions permitted.
- **Style preference**: The user favors a narrative, exploratory tone ("watch which ports quietly open", "a bit of a black box", "another layer of mystery") suggesting the document should explain behavior as a guided walkthrough rather than a dry reference table.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the startup behavior without a config file**, we will trace the code path from `main.go` through `cmd/serve.go` → `service/service.go:Start()` → `config/config.go:LoadConfig()`, capturing the exact warning message emitted by Viper when no config file is found and the full default configuration struct that gets logged.
- To **document failed SFTP authentication**, we will trace `sftpd/server.go:validatePasswordCredentials()` → `dataprovider/dataprovider.go:CheckUserAndPass()` → `dataprovider/sqlcommon.go:sqlCommonValidateUserAndPass()`, capturing the `connection_failed` log entry with its `client_ip`, `username`, `login_type`, and `error` fields.
- To **document the web admin root redirect**, we will trace `httpd/router.go:initializeRouter()` where `router.Get("/", ...)` issues an `http.StatusMovedPermanently` (301) redirect to `/web/users`.
- To **document missing database behavior**, we will trace `dataprovider/sqlite.go:initializeSQLiteProvider()` which calls `os.Stat(dbPath)` and emits a warn-level log before returning the error to `service.Start()`, which logs the fatal error and exits.
- To **document default configuration values**, we will extract the complete `globalConfig` struct initialization from `config/config.go:init()` and correlate it with the log output.

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `logger` package has two distinct output channels (`logger` and `consoleLogger`) whose behavior varies by startup mode — the document should explain which channel produces which output.
- Based on structure: The startup sequence spans `cmd/`, `config/`, `service/`, `dataprovider/`, `sftpd/`, and `httpd/` — the document needs a consolidated lifecycle narrative.
- Based on dependencies: The relationship between `config.LoadConfig()` and Viper's search path (current directory, `$HOME/.config/sftpgo`, `/etc/sftpgo` on Linux) determines whether defaults are used — this must be documented.
- Based on user journey: A newcomer onboarding needs the document to connect observable log output back to specific source files and line ranges for further exploration.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal, informal documentation structure** with no dedicated documentation generator, no `docs/` directory, and no structured documentation framework. All existing documentation lives in scattered markdown files at the repository root and inside packaging subdirectories.

- **Documentation framework**: None — no `mkdocs.yml`, `docusaurus.config.js`, `sphinx.conf.py`, `.readthedocs.yml`, or equivalent configuration file exists.
- **Documentation generator**: None detected.
- **API documentation tools**: The OpenAPI 3.0.1 schema at `httpd/schema/openapi.yaml` serves as the formal REST API contract, but no JSDoc, GoDoc site generation, or Swagger UI deployment is configured.
- **Diagram tools**: No Mermaid, PlantUML, or other diagram tooling is configured in the repository.
- **Documentation hosting**: No documentation deployment pipeline exists. The `README.md` (686 lines) serves as the single primary onboarding document.

Existing documentation files discovered:

| File | Size | Purpose |
|------|------|---------|
| `README.md` | 686 lines | Primary project documentation: features, installation, CLI, configuration, authentication, storage, REST API, web admin, metrics, logging |
| `docker/README.md` | ~15 lines | Docker overview pointing to Alpine and Debian variants |
| `docker/sftpgo/alpine/README.md` | ~80 lines | Alpine Docker image build and run instructions |
| `docker/sftpgo/debian/README.md` | ~40 lines | Debian Docker image build and run instructions |
| `scripts/README.md` | ~50 lines | REST API CLI Python script documentation |
| `LICENSE` | GPLv3 license text | License declaration |

No `blitzy/documentation/` directory exists yet — it must be created to house the new document per the `SWE-AtlasQnA-Repo` implementation rule.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code relevant to the documentation questions:

- **Startup lifecycle**: `main.go`, `cmd/root.go`, `cmd/serve.go`, `service/service.go` — the full chain from binary entry to service readiness
- **Configuration defaults and loading**: `config/config.go`, `config/config_linux.go` — Viper-based config initialization, search paths, and default values
- **SFTP server initialization and authentication**: `sftpd/server.go`, `sftpd/sftpd.go` — listener binding, host key generation, password/pubkey callbacks, connection acceptance
- **HTTP server and routing**: `httpd/httpd.go`, `httpd/router.go` — HTTP listener, Chi router, root redirect, API paths
- **Data provider initialization**: `dataprovider/dataprovider.go`, `dataprovider/sqlite.go`, `dataprovider/sqlcommon.go` — provider selection, SQLite file validation, availability timer
- **Logging system**: `logger/logger.go`, `logger/request_logger.go` — zerolog initialization, dual-channel logging, structured log format, `ConnectionFailedLog` function
- **Version and utilities**: `utils/version.go` (version `0.9.5-dev`), `utils/utils.go` (umask, encryption helpers)
- **Metrics**: `metrics/metrics.go` — Prometheus counters for login attempts, connections, transfers
- **Sample configuration**: `sftpgo.json` — default configuration file with all three subsystem blocks
- **Database migrations**: `sql/sqlite/20190828.sql` through `sql/sqlite/20200116.sql` — schema evolution for the users table

Key directories examined: `cmd/`, `config/`, `service/`, `sftpd/`, `httpd/`, `dataprovider/`, `logger/`, `metrics/`, `utils/`, `sql/sqlite/`

### 0.2.3 Runtime Observation Research Conducted

Live runtime observations were performed by building the SFTPGo binary from source using Go 1.13.15 (the version specified in `go.mod`) and executing it under controlled conditions:

- **Scenario A — No config file, no database**: Launched SFTPGo with `--log-file-path ""` and `--config-dir` pointing to an empty directory. Captured the warn-level config loading failure and the error-level data provider initialization failure.
- **Scenario B — No config file, database present**: Created a SQLite database using all four migration scripts (`sql/sqlite/20190828.sql` through `sql/sqlite/20200116.sql`), placed template and static assets in the config directory, and launched again. Captured full startup including config fallback, SQLite handle creation, host key generation, SFTP listener binding, and HTTP server initialization.
- **Scenario C — Failed SFTP authentication**: Used `sshpass` and `ssh` with `HostKeyAlgorithms=+ssh-rsa` to attempt password authentication as a nonexistent user (`ghost_user`). Captured the `connection_failed` structured log entry and the warn-level SQLite authentication error.
- **Scenario D — HTTP root endpoint access**: Used `curl` to hit `http://127.0.0.1:8080/` and captured the `301 Moved Permanently` redirect to `/web/users`. Also verified `/api/v1/version`, `/api/v1/providerstatus`, and `/metrics` endpoints.
- **Scenario E — Cleanup**: All temporary files (SQLite database, generated `id_rsa` key, log files, observation directory) were removed after capturing results.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules contain the runtime behaviors that must be documented in the new Q&A document:

- **Module: `main.go`**
  - Public APIs: `main()` — thin launcher calling `cmd.Execute()`
  - Current documentation: Inline comment only
  - Documentation needed: Explain side-effect imports of SQL drivers (`_ "github.com/go-sql-driver/mysql"`, `_ "github.com/lib/pq"`, `_ "github.com/mattn/go-sqlite3"`)

- **Module: `cmd/root.go` + `cmd/serve.go`**
  - Public APIs: `Execute()`, `addServeFlags()`, `serveCmd.Run`
  - Current documentation: CLI help strings
  - Documentation needed: Default flag values (`configDir="."`, `logFilePath="sftpgo.log"`, `logVerbose=true`, `logMaxSize=10`, `logMaxBackups=5`, `logMaxAge=28`, `logCompress=false`), Viper environment variable bindings

- **Module: `config/config.go`**
  - Public APIs: `LoadConfig()`, `GetSFTPDConfig()`, `GetHTTPDConfig()`, `GetProviderConf()`
  - Current documentation: None beyond inline comments
  - Documentation needed: Full default config struct with every field value, Viper search path behavior, config file not-found fallback, redacted config logging

- **Module: `service/service.go`**
  - Public APIs: `Service.Start()`, `Service.Wait()`, `Service.Stop()`
  - Current documentation: None
  - Documentation needed: Startup sequence (logging init → version log → config load → provider init → SFTP goroutine → HTTP goroutine), shutdown channel behavior

- **Module: `sftpd/server.go`**
  - Public APIs: `Configuration.Initialize()`, `AcceptInboundConnection()`, `validatePasswordCredentials()`, `validatePublicKeyCredentials()`, `checkHostKeys()`, `generatePrivateKey()`
  - Current documentation: Struct field comments
  - Documentation needed: Host key auto-generation behavior, SSH banner construction (`SSH-2.0-SFTPGo_0.9.5-dev`), listener address format, authentication failure flow and log output

- **Module: `httpd/httpd.go` + `httpd/router.go`**
  - Public APIs: `Conf.Initialize()`, `initializeRouter()`
  - Current documentation: Package comment and path constants
  - Documentation needed: Root endpoint 301 redirect, web admin paths, API paths, middleware chain (RequestID, RealIP, StructuredLogger, Recoverer), HTTP server timeouts (ReadTimeout=300s, WriteTimeout=300s, MaxHeaderBytes=64KB)

- **Module: `dataprovider/dataprovider.go` + `dataprovider/sqlite.go`**
  - Public APIs: `Initialize()`, `CheckUserAndPass()`, `GetProviderStatus()`
  - Current documentation: Struct field comments
  - Documentation needed: SQLite file existence check behavior, missing DB error message, availability timer (30-second tick), `RecordNotFoundError` for nonexistent users

- **Module: `logger/logger.go`**
  - Public APIs: `InitLogger()`, `ConnectionFailedLog()`, `Info()`, `Warn()`, `Error()`
  - Current documentation: Package comment
  - Documentation needed: Structured JSON log format (`level`, `time`, `sender`, `connection_id`, `message`), `ConnectionFailedLog` field schema (`sender=connection_failed`, `client_ip`, `username`, `login_type`, `error`)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented runtime startup behavior**: No existing document describes the observable log output sequence during a fresh SFTPGo startup. The `README.md` covers configuration syntax but not the startup lifecycle narrative.
- **Undocumented authentication failure output**: No document describes what a failed login looks like from the server's perspective in log output. The `fail2ban/` directory contains filter patterns but no narrative explanation.
- **Undocumented HTTP redirect behavior**: The 301 redirect from `/` to `/web/users` is visible only in `httpd/router.go` source code — no documentation explains the web admin entry path.
- **Undocumented database initialization failure behavior**: The error sequence when SQLite is missing is only documented in code comments within `dataprovider/sqlite.go` — no operational guide covers first-startup database preparation.
- **Undocumented default configuration surface**: The complete set of defaults in `config/config.go:init()` has never been extracted into a documentation-friendly format showing every applied value.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document. The document hierarchy follows the structure mandated by `SWE-AtlasQnA-Repo`:

```
blitzy/
└── documentation/
    └── sftpgo_44634210287c.md
```

The internal structure of `sftpgo_44634210287c.md` will be organized as a guided Q&A walkthrough with the following sections:

```
sftpgo_44634210287c.md
├── Header (title, context, version)
├── 1. Startup Without a Configuration File
│   ├── What Viper searches for and where
│   ├── The warning log message
│   ├── Default configuration values applied
│   └── How the default banner is constructed
├── 2. Ports That Open and Readiness Signals
│   ├── SFTP listener (port 2022, all interfaces)
│   ├── HTTP listener (port 8080, localhost only)
│   ├── Host key generation on first run
│   └── Log messages signaling readiness
├── 3. Failed SFTP Authentication for a Nonexistent User
│   ├── SSH handshake and banner exchange
│   ├── Password callback triggering provider lookup
│   ├── The connection_failed log entry
│   └── How the server responds to the client
├── 4. Web Admin Root Endpoint Behavior
│   ├── The 301 redirect from / to /web/users
│   ├── Chi router middleware chain
│   └── API and web path constants
├── 5. Missing Database on First Startup
│   ├── SQLite file existence check
│   ├── The warning and error log messages
│   ├── Service startup abort sequence
│   └── Steps to bootstrap the database
├── 6. Default Configuration Revealed in Logs
│   ├── SFTPD defaults table
│   ├── Data provider defaults table
│   ├── HTTPD defaults table
│   └── Password redaction in log output
└── Source Code References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract default configuration values from `config/config.go:init()` (lines 41-101) and correlate with the `sftpgo.json` sample file
- Extract startup log sequence from the actual runtime captures performed during observation
- Extract authentication failure flow from `sftpd/server.go:validatePasswordCredentials()` → `dataprovider/sqlcommon.go:sqlCommonValidateUserAndPass()` → `logger/logger.go:ConnectionFailedLog()`
- Extract HTTP routing from `httpd/router.go:initializeRouter()` where the root handler issues `http.Redirect(w, r, webUsersPath, http.StatusMovedPermanently)`
- Extract database initialization failure from `dataprovider/sqlite.go:initializeSQLiteProvider()` where `os.Stat(dbPath)` failure triggers the warning message

**Documentation Standards Applied:**

- Markdown formatting with `#`, `##`, `###` headers
- JSON log examples displayed in fenced code blocks with `json` syntax highlighting
- Source citations as inline references: `Source: /path/to/file.go:LineNumber`
- Tables for default configuration values, HTTP endpoints, and log field schemas
- Mermaid diagrams for the startup lifecycle and authentication flow
- Every answer must trace back to specific source code evidence and, where possible, to actual captured log output

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the document:

- **Startup Lifecycle Flowchart**: `main.go` → `cmd.Execute()` → `serveCmd.Run` → `service.Start()` → config load → provider init → SFTP + HTTP goroutines. Shows the decision points (config found vs. not found, DB exists vs. missing).
- **Authentication Failure Sequence Diagram**: Client → SSH handshake → Password callback → `dataprovider.CheckUserAndPass()` → SQLite lookup → `RecordNotFoundError` → `ConnectionFailedLog` → SSH rejection back to client.

These diagrams will use standard Mermaid `flowchart` and `sequenceDiagram` syntax embedded in fenced code blocks within the markdown document.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/sftpgo_44634210287c.md` | CREATE | `main.go`, `cmd/root.go`, `cmd/serve.go`, `config/config.go`, `config/config_linux.go`, `service/service.go`, `sftpd/server.go`, `sftpd/sftpd.go`, `httpd/httpd.go`, `httpd/router.go`, `dataprovider/dataprovider.go`, `dataprovider/sqlite.go`, `dataprovider/sqlcommon.go`, `logger/logger.go`, `utils/version.go`, `sftpgo.json`, `sql/sqlite/*.sql` | Complete Q&A document answering all five runtime-behavior questions with log output evidence, source citations, Mermaid diagrams, and configuration default tables |

This is the **only** file transformation required. No existing files in the repository are modified, updated, or deleted per the explicit user constraint and the `SWE-AtlasQnA-Repo` rule.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Technical Investigation / Q&A Document
Source Code: All packages listed above
Sections:
    - Title and Context (SFTPGo version, branch, purpose)
    - Startup Without a Configuration File
        Source: config/config.go:140-155, service/service.go:60-66
    - Ports and Readiness Signals
        Source: sftpd/server.go:178-191, httpd/httpd.go:91-98
    - Failed SFTP Authentication
        Source: sftpd/server.go:448-461, dataprovider/sqlcommon.go:28-39,
                logger/logger.go:175-184
    - Web Admin Root Endpoint
        Source: httpd/router.go:36-38, httpd/httpd.go:20-35
    - Missing Database on First Startup
        Source: dataprovider/sqlite.go:18-50, service/service.go:69-73
    - Default Configuration in Logs
        Source: config/config.go:41-94, sftpgo.json
    - Source Code References (consolidated file list)
Diagrams:
    - Startup lifecycle flowchart (Mermaid flowchart)
    - Authentication failure sequence diagram (Mermaid sequenceDiagram)
Key Citations: config/config.go, service/service.go, sftpd/server.go,
               httpd/router.go, dataprovider/sqlite.go, logger/logger.go
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository has no documentation generator (`mkdocs.yml`, `docusaurus.config.js`, etc.) and no navigation or sidebar to update. The new file is a standalone markdown document placed in a new `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared includes or partials**: The new document is self-contained.
- **No navigation links**: There is no documentation site to update.
- **No table of contents**: The `README.md` does not reference a `blitzy/documentation/` directory, and per the rule not to modify existing files, no link to the new document will be added.
- **No index or glossary**: None exists in the repository.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation task requires building and running the SFTPGo binary from source. The following dependencies are required for the observation-based documentation workflow:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| golang.org | Go | 1.13.15 | Build runtime — the highest patch release of Go 1.13 specified by `go.mod` |
| go module | `github.com/drakkan/sftpgo` | 0.9.5-dev | The SFTPGo server being documented (version from `utils/version.go`) |
| go module | `github.com/spf13/viper` | 1.6.1 | Configuration loading — drives config file search and env var binding |
| go module | `github.com/spf13/cobra` | 0.0.5 | CLI framework — `serve` command and flag definitions |
| go module | `github.com/rs/zerolog` | 1.17.2 | Structured JSON logging — produces all observed log output |
| go module | `gopkg.in/natefinish/lumberjack.v2` | 2.0.0 | Log file rotation (used when `--log-file-path` is set) |
| go module | `github.com/go-chi/chi` | 4.0.2 | HTTP router — defines web admin and API routes including root redirect |
| go module | `github.com/mattn/go-sqlite3` | 2.0.2 | SQLite driver — default data provider backend |
| go module | `github.com/prometheus/client_golang` | 1.3.0 | Prometheus metrics endpoint at `/metrics` |
| go module | `golang.org/x/crypto` | 0.0.0-20200109152110 | SSH server implementation and key generation |
| apt | `sqlite3` | system | CLI tool for initializing the SQLite database from migration scripts |
| apt | `openssh-client` | system | SSH client for testing SFTP authentication failure |
| apt | `sshpass` | system | Non-interactive SSH password provider for observation scripts |
| apt | `curl` | system | HTTP client for testing web admin endpoints |
| apt | `gcc` | system | C compiler required by `go-sqlite3` CGO bindings |

All Go module versions above are the **exact** versions pinned in `go.mod` and verified via `go.sum`.

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files contain links that require updating, and no new cross-links are being created. The new document is entirely self-contained within `blitzy/documentation/sftpgo_44634210287c.md`.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

Current coverage analysis against the user's five stated questions:

| Question | Source Files Analyzed | Runtime Evidence Captured | Coverage Target |
|----------|----------------------|--------------------------|-----------------|
| Startup without config file | `config/config.go`, `service/service.go`, `cmd/root.go` | Full warn-level JSON log captured | 100% — must reproduce exact log message and explain each field |
| Ports and readiness signals | `sftpd/server.go`, `httpd/httpd.go` | Listener registration log and HTTP response captured | 100% — must list both ports with bind addresses |
| Failed SFTP authentication | `sftpd/server.go`, `dataprovider/sqlcommon.go`, `logger/logger.go` | `connection_failed` structured log entry captured for user `ghost_user` | 100% — must show exact log format and explain each field |
| Web admin root endpoint | `httpd/router.go` | HTTP 301 response with `Location: /web/users` captured via `curl` | 100% — must show HTTP status, headers, and redirect target |
| Missing database on startup | `dataprovider/sqlite.go`, `service/service.go` | Error-level service log and warn-level SQLite log captured | 100% — must show both messages and explain recovery steps |
| Default config in logs | `config/config.go:init()`, `sftpgo.json` | Full redacted config struct captured in debug-level log | 100% — must tabulate every default value for all three subsystems |

Overall target: **100% coverage** of all five questions with both code-level explanation and runtime log evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must include the actual captured log output (JSON) as a code block
- Every answer must cite the specific source file and line range where the behavior originates
- Every answer must include a rationale explaining *why* the behavior occurs (e.g., "Viper searched these paths and found no matching file")
- Default configuration tables must list every field, its default value, and a brief description

**Accuracy validation:**
- All JSON log examples must be real output captured from running SFTPGo v0.9.5-dev built with Go 1.13.15
- All source code citations must reference actual line numbers verified by `read_file`
- HTTP responses must be verified via actual `curl` output, not inferred from code
- Authentication failure logs must be captured from an actual SSH connection attempt

**Clarity standards:**
- The document should follow a narrative walkthrough style, matching the exploratory tone of the user's request
- Technical terms (Viper, zerolog, Chi, SSH handshake) should be briefly contextualized for an onboarding reader
- Each section should open with the question being answered, then provide the answer with evidence

**Maintainability:**
- Source file references use the format `Source: path/to/file.go:LineRange` for grep-ability
- The document header records the SFTPGo version (`0.9.5-dev`), Go version (`1.13.15`), and observation date for freshness tracking

### 0.7.3 Example and Diagram Requirements

- Minimum examples per question: 1 real log output block + 1 source code citation
- Diagram types required: 1 startup lifecycle flowchart (Mermaid), 1 authentication failure sequence diagram (Mermaid)
- Code example testing: All log examples are real captured output from observation runs
- Default configuration tables: 3 tables (SFTPD, DataProvider, HTTPD) with exhaustive field-by-field coverage

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/sftpgo_44634210287c.md` — the sole deliverable

**Source files analyzed for documentation content (read-only, no modifications):**
- `main.go` — entry point and driver imports
- `cmd/root.go` — CLI root, flag defaults, Viper bindings
- `cmd/serve.go` — serve command wiring
- `config/config.go` — default config struct, `LoadConfig()`, config search paths
- `config/config_linux.go` — Linux-specific Viper search paths
- `service/service.go` — `Service.Start()` lifecycle orchestration
- `sftpd/server.go` — SFTP listener initialization, host key management, auth callbacks
- `sftpd/sftpd.go` — connection registry, idle timer, SSH command defaults
- `httpd/httpd.go` — HTTP server initialization, `Conf` struct
- `httpd/router.go` — Chi router setup, root redirect, all route registrations
- `dataprovider/dataprovider.go` — `Initialize()`, `CheckUserAndPass()`, availability timer
- `dataprovider/sqlite.go` — SQLite provider initialization, file existence check
- `dataprovider/sqlcommon.go` — `sqlCommonValidateUserAndPass()`, user lookup
- `logger/logger.go` — `InitLogger()`, log format, `ConnectionFailedLog()`
- `logger/request_logger.go` — HTTP request structured logging middleware
- `utils/version.go` — version constant `0.9.5-dev`
- `metrics/metrics.go` — Prometheus metric definitions
- `sftpgo.json` — sample configuration file
- `sql/sqlite/*.sql` — four migration scripts for database initialization
- `go.mod` — Go 1.13 requirement and dependency versions

**Temporary observation artifacts (created and destroyed during documentation):**
- Temporary SQLite database for startup observation
- Temporary `id_rsa` host key generated by SFTPGo during observation
- Temporary log capture files
- Temporary observation directory (`/tmp/observation_run/`)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No `.go` files, configuration files, or any existing files in the repository will be changed, per the user's explicit directive and the `SWE-AtlasQnA-Repo` rule.
- **Test file modifications**: No test files will be created or modified.
- **Feature additions or refactoring**: No new features, bug fixes, or code restructuring.
- **Deployment or infrastructure changes**: No Dockerfiles, CI configs, or deployment scripts will be touched.
- **Documentation for unrelated areas**: The document focuses exclusively on the five runtime-behavior questions asked. Other areas (S3 backend, portable mode, Windows service, backup/restore, quota scanning) are not in scope.
- **README.md updates**: The existing `README.md` will not be modified to link to the new document.
- **Documentation generator setup**: No `mkdocs`, `docusaurus`, or other documentation tooling will be added to the repository.
- **Persistent state**: All temporary observation artifacts (database files, key files, log files) will be cleaned up after documentation generation.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command**: `cd <repo_root> && CGO_ENABLED=1 go build -o /tmp/sftpgo .` (requires Go 1.13.x and `gcc` for SQLite CGO bindings)
- **Database initialization**: `cat sql/sqlite/20190828.sql sql/sqlite/20191112.sql sql/sqlite/20191230.sql sql/sqlite/20200116.sql | sqlite3 <config_dir>/sftpgo.db`
- **Server launch for observation**: `/tmp/sftpgo serve --log-file-path "" --config-dir <config_dir>` (log to stdout, specify a clean config directory)
- **HTTP endpoint verification**: `curl -s -i http://127.0.0.1:8080/` to observe the 301 redirect
- **SFTP auth failure trigger**: `sshpass -p "anypass" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o HostKeyAlgorithms=+ssh-rsa -p 2022 nonexistent_user@127.0.0.1`
- **Default format**: Markdown with Mermaid diagrams and JSON code blocks
- **Citation requirement**: Every section must reference the source file(s) and line ranges that produce the observed behavior
- **Style guide**: Narrative walkthrough matching the exploratory onboarding tone requested by the user
- **Documentation validation**: Manual review of all JSON log examples against actual captured output; manual verification that all source citations point to valid file paths and line numbers in the repository

### 0.9.2 Output File Placement

Per the `SWE-AtlasQnA-Repo` implementation rule:
- **File name**: `sftpgo_44634210287c.md` (derived from the source branch name `sftpgo_44634210287c`)
- **Directory**: `blitzy/documentation/` in the destination repository
- **Full path**: `blitzy/documentation/sftpgo_44634210287c.md`

The `blitzy/documentation/` directory must be created since it does not yet exist.

## 0.10 Rules for Documentation

The following rules are explicitly derived from the user's instructions and the `SWE-AtlasQnA-Repo` implementation rule:

- **Do not modify any existing files in the source repository.** The new document is the only artifact created.
- **Provide thinking and rationale behind the answers.** Every answer must explain *why* the behavior occurs, not just *what* happens.
- **Do not make assumptions — base answers on the code as the truth.** All claims must be traceable to specific source files and line numbers, or to actual captured runtime output.
- **Place the generated document in the `blitzy/documentation` directory** with the file name `sftpgo_44634210287c.md`.
- **Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.** All observation artifacts (SQLite database, generated host keys, log captures, temporary directories) must be removed after documentation is complete.
- **Answers should describe what really shows up** in the logs and HTTP responses — actual log lines, actual HTTP status codes, actual error messages — not theoretical descriptions.
- **The document should serve an onboarding reader** who wants to understand SFTPGo's runtime behavior through direct observation, using a narrative walkthrough style that connects observable output to source code origins.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| Path | Type | Relevance |
|------|------|-----------|
| `main.go` | File | Entry point — imports SQL drivers, calls `cmd.Execute()` |
| `cmd/root.go` | File | CLI root command, flag defaults, Viper env bindings |
| `cmd/serve.go` | File | Serve command wiring — creates `service.Service` and calls `Start()` |
| `config/config.go` | File | Default config struct, `LoadConfig()`, Viper search paths, redacted logging |
| `config/config_linux.go` | File | Linux-specific Viper config paths (`$HOME/.config/sftpgo`, `/etc/sftpgo`) |
| `service/service.go` | File | `Service.Start()` lifecycle — config load, provider init, SFTP/HTTP goroutines |
| `sftpd/server.go` | File | SFTP `Configuration.Initialize()`, host key generation, auth callbacks, listener binding |
| `sftpd/sftpd.go` | File | Connection registry, idle timer, default/supported SSH commands, action execution |
| `httpd/httpd.go` | File | HTTP `Conf.Initialize()`, path constants, HTTP server timeout settings |
| `httpd/router.go` | File | Chi router setup, root 301 redirect, all API/web route registrations, middleware chain |
| `dataprovider/dataprovider.go` | File | `Initialize()`, `CheckUserAndPass()`, provider interface, availability timer, error types |
| `dataprovider/sqlite.go` | File | SQLite provider init, file existence check, warn message for missing DB |
| `dataprovider/sqlcommon.go` | File | `sqlCommonValidateUserAndPass()`, user lookup by username, error logging |
| `dataprovider/bolt.go` | File | BoltDB provider init and bucket structure (context for provider comparison) |
| `dataprovider/user.go` | Summary | User model, permissions, filesystem config, login validation |
| `logger/logger.go` | File | `InitLogger()`, dual-channel logging, `ConnectionFailedLog()`, structured JSON format |
| `logger/request_logger.go` | File | HTTP request structured logging via Chi middleware |
| `utils/version.go` | File | Version constant `0.9.5-dev`, `VersionInfo` struct |
| `metrics/metrics.go` | Summary | Prometheus counters and gauges for login, transfer, and HTTP events |
| `sftpgo.json` | File | Sample config with SFTPD, data provider, and HTTPD sections |
| `go.mod` | File | Go 1.13 requirement, all dependency versions |
| `sql/sqlite/20190828.sql` | File | Initial schema migration — creates `users` table |
| `sql/sqlite/20191112.sql` | File | Adds `expiration_date`, `last_login`, `status` columns |
| `sql/sqlite/20191230.sql` | File | Adds `filters` column |
| `sql/sqlite/20200116.sql` | File | Adds `filesystem` column |
| `README.md` | Summary | Primary project documentation (686 lines) |
| `docker/README.md` | File | Docker overview |
| `scripts/README.md` | Summary | REST API CLI documentation |
| `.travis.yml` | Summary | CI pipeline for Go 1.13 on Linux/macOS |
| `cmd/` | Folder | CLI command layer — serve, portable, Windows service commands |
| `config/` | Folder | Configuration subsystem — Viper loading, defaults, platform paths |
| `service/` | Folder | Service lifecycle — Start, Wait, Stop, portable mode |
| `sftpd/` | Folder | SSH/SFTP/SCP server implementation |
| `httpd/` | Folder | HTTP server, REST API, web admin |
| `dataprovider/` | Folder | Data provider layer — SQL, Bolt, memory backends |
| `logger/` | Folder | Logging subsystem — zerolog wrapper, HTTP middleware |
| `utils/` | Folder | Utilities — version, umask, encryption, formatting |
| `metrics/` | Folder | Prometheus instrumentation |
| `sql/sqlite/` | Folder | SQLite migration scripts |

### 0.11.2 Tech Spec Sections Referenced

- **1.1 Executive Summary** — Project overview, version identification, stakeholders
- **1.3 Scope** — In-scope features, system boundaries, platform coverage, database compatibility

### 0.11.3 Attachments

No attachments were provided by the user. No Figma URLs or external design assets are associated with this task.

### 0.11.4 Runtime Observations Performed

The following observation scenarios were executed during context gathering (all temporary artifacts cleaned up afterward):

| Scenario | Command | Key Finding |
|----------|---------|-------------|
| No config, no DB | `sftpgo serve --log-file-path "" --config-dir <empty_dir>` | Viper warns about missing config; SQLite provider fails with `stat ... no such file or directory`; service exits with error |
| No config, DB present | Same command with initialized `sftpgo.db` | Config falls back to `sftpgo.json` found elsewhere in Viper search path; SQLite handle created; SFTP listener registered on `[::]:2022`; HTTP server initialized on `127.0.0.1:8080` |
| Failed SFTP auth | `sshpass -p "wrongpass" ssh ... ghost_user@127.0.0.1 -p 2022` | Server logs `connection_failed` with `sender=connection_failed`, `login_type=password`, `error="Not found: sql: no rows in result set"` |
| HTTP root access | `curl -s -i http://127.0.0.1:8080/` | Returns `301 Moved Permanently` with `Location: /web/users` |
| API version | `curl -s http://127.0.0.1:8080/api/v1/version` | Returns `{"version":"0.9.5-dev","build_date":"","commit_hash":""}` |
| Provider status | `curl -s http://127.0.0.1:8080/api/v1/providerstatus` | Returns `{"error":"","message":"Alive","status":200}` |

