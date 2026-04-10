# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative Q&A document** that explains the runtime behavior of SFTPGo (v0.9.5-dev) across several specific "first start" scenarios, with a particular focus on conditions that produce non-obvious or shifting results between runs.

- **Category**: Create new documentation
- **Documentation type**: Technical investigation / runtime behavior Q&A
- **Target file**: `blitzy/documentation/sftpgo_44634210287c.md`

The user's requirements break down into six distinct investigation areas, each requiring precise, code-grounded answers:

- **Clean-start isolation**: Whether a temporary config directory truly isolates SFTPGo state, or whether the process discovers defaults, configuration files, or generated state from other locations (including the current working directory)
- **SQLite data-provider states**: What happens on startup when the SQLite database file is missing, empty (zero bytes), or already valid — and how each case affects startup success, exit status, and subsequent HTTP endpoint behavior
- **SFTP port conflicts**: What startup behavior and exit semantics result when the default SFTP port (2022) is already occupied by another process
- **Missing web UI assets**: What happens when template files or static assets are not located at the paths the HTTP server expects
- **Proxy header resolution**: Exactly which IP address and scheme SFTPGo logs when requests arrive with `Forwarded`, `X-Forwarded-For`, and `X-Real-IP` headers — including multi-IP forwarding chains — and whether the working directory has any influence
- **Metrics counter behavior**: How the `sftpgo_http_*` Prometheus counters move for identical requests sent with and without proxy-style headers

### 0.1.2 Special Instructions and Constraints

- **Repository must remain unchanged**: The user explicitly requires that "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." Therefore, the output document must not propose any modifications to existing source files.
- **Implementation rule**: The project rule `SWE-AtlasQnA-Repo` mandates: create a new markdown document named `<source_branch_name>.md` (i.e., `sftpgo_44634210287c.md`) in the `blitzy/documentation` directory. The document must comprehensively answer the questions posed, provide thinking/rationale, base answers on the code as truth, and not modify any existing files.
- **Evidence standard**: All answers must be grounded in the actual source code; no assumptions are permitted. Each claim must reference the specific file and logic that proves it.
- **Snippet requirements**: The user requests short, illustrative snippets — representative startup log lines, HTTP response headers, access log lines (with volatile fields redacted), and specific `/metrics` counter lines — that "make the behavior undeniable."
- **Cleanup**: Any temporary scripts or test artifacts discussed in the document should include cleanup guidance.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document clean-start isolation behavior, we will trace the Viper config search path in `config/config.go` (`LoadConfig` adds configDir → platform paths → `.`), the SQLite path resolution in `dataprovider/sqlite.go`, the host key generation in `sftpd/server.go` (`checkHostKeys`), and template/static path resolution in `httpd/httpd.go` (`Initialize`).
- To document SQLite state scenarios, we will analyze `dataprovider/sqlite.go` (`initializeSQLiteProvider`) which uses `os.Stat` to check file existence and size, and `service/service.go` (`Start`) which aborts on data provider initialization failure.
- To document SFTP port conflicts, we will trace `sftpd/server.go` (`Initialize`) where `net.Listen("tcp", addr)` returns an error on bind failure, which propagates through the goroutine in `service/service.go` to the `Shutdown` channel.
- To document missing web UI assets, we will analyze `httpd/web.go` (`loadTemplates`) which uses `template.Must(template.ParseFiles(...))` — a call that panics on missing files — called from `httpd/httpd.go` (`Initialize`) before the HTTP server starts accepting connections.
- To document proxy header behavior, we will trace `httpd/router.go` (middleware chain: `RequestID` → `RealIP` → `StructuredLogger` → `Recoverer`), the chi v4.0.2 `middleware.RealIP` implementation, and `logger/request_logger.go` (`NewLogEntry`) which reads the already-modified `r.RemoteAddr`.
- To document metrics counter movement, we will analyze `metrics/metrics.go` (`HTTPRequestServed`) and `logger/request_logger.go` (`Write`) which calls `HTTPRequestServed(status)` for every completed request.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, several implicit documentation needs emerge:

- **Config search path and CWD leakage**: The Viper config search path in `config/config.go` includes `.` (the current working directory) as the final fallback. This means starting SFTPGo from the repo root (which contains `sftpgo.json`) versus an empty `/tmp` directory can silently load different configuration even when an explicit configDir is provided — but only if configDir lacks a config file. This directly explains the user's observation that "the process is quietly finding state or defaults from somewhere else."
- **Log file path is CWD-relative**: The default log file `sftpgo.log` (from `cmd/root.go`) is passed to `lumberjack.Logger` as a relative path, which resolves against the CWD, not configDir. This means the log file location shifts with working directory.
- **Host key generation as side effect**: `sftpd/server.go` (`checkHostKeys`) auto-generates an RSA host key (`id_rsa`) in configDir if none exists. This side effect occurs on every "first start" and changes the SSH server's fingerprint between runs with different temp directories.
- **Redirect metrics gap**: 3xx HTTP responses (e.g., the 301 redirects from `/` and `/web`) only increment `totalHTTPRequests` in `metrics/metrics.go`, not any sub-category counter (`totalHTTPOK`, `totalHTTPClientErrors`, `totalHTTPServerErrors`). This may confuse someone comparing total requests to the sum of subcategories.
- **Forwarded header blind spot**: chi v4.0.2's `middleware.RealIP` does NOT parse the RFC 7239 `Forwarded` header — only `X-Forwarded-For` and `X-Real-IP`. This is a critical gap for the user's TLS-terminating reverse proxy scenario since `Forwarded: for=...; proto=https` would be silently ignored.
- **Scheme detection limitation**: The logger in `logger/request_logger.go` determines scheme from `r.TLS != nil`, not from any forwarded header. Behind a TLS-terminating proxy, the logged scheme will always be `http` regardless of `X-Forwarded-Proto` or `Forwarded: proto=https` headers.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal documentation structure** with no dedicated documentation generator, no `docs/` directory, and no documentation tooling configuration (no mkdocs.yml, sphinx conf, or similar). Existing documentation consists of:

- **README.md**: Not present in the repository root (no README found)
- **No `docs/` folder**: The repository lacks any structured documentation directory
- **Inline code documentation**: Go source files contain package-level doc comments (e.g., `// Package httpd implements REST API and Web interface for SFTPGo.`) and function-level doc comments, but no generated godoc output
- **OpenAPI schema reference**: `httpd/httpd.go` line 5 references an OpenAPI 3 schema at `https://github.com/drakkan/sftpgo/tree/master/api/schema/openapi.yaml` but no local `api/schema/` directory exists in this checkout
- **Config example**: `sftpgo.json` at the repository root serves as a reference configuration file with all default values

Current documentation framework: None (no documentation generator installed)
Documentation generator configuration: N/A
API documentation tools: None — only Go doc comments in source files
Diagram tools: None detected in repository
Documentation hosting/deployment: None configured

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used and key directories examined:

- **Startup orchestration**: `cmd/root.go`, `cmd/serve.go`, `cmd/portable.go` — Cobra command structure, flag definitions, default config directory (`.`), default config name (`sftpgo`), default log file (`sftpgo.log`)
- **Service lifecycle**: `service/service.go` — `Start()` method orchestrating logger init → config load → data provider init → SFTP goroutine → HTTP goroutine. `StartPortableMode()` for ephemeral operation
- **Configuration system**: `config/config.go`, `config/config_linux.go` — Viper-based config with multi-path search (configDir → platform paths → `.`), default values seeded in `init()`
- **Data provider / SQLite**: `dataprovider/dataprovider.go`, `dataprovider/sqlite.go` — Provider initialization with file existence/size checks; 30-second availability timer
- **HTTP server**: `httpd/httpd.go`, `httpd/router.go`, `httpd/web.go`, `httpd/api_utils.go` — chi router with middleware chain, path constants, template loading via `template.Must`, static file serving
- **SFTP server**: `sftpd/server.go` — SSH server setup, host key auto-generation, `net.Listen` for port binding
- **Logging**: `logger/logger.go`, `logger/request_logger.go` — zerolog-based logger with lumberjack rotation; HTTP request logger implementing chi's `LogFormatter`/`LogEntry` interfaces
- **Metrics**: `metrics/metrics.go` — Prometheus counters registered via `promauto`; `HTTPRequestServed(status)` for HTTP status classification
- **Versioning**: `utils/version.go` — Version `0.9.5-dev` with build date and commit hash
- **Templates and static**: `templates/` directory (base.html, users.html, user.html, connections.html, message.html), `static/` directory (css, js, vendor, favicon.ico)

Related documentation found: No existing documentation about runtime behavior, startup scenarios, or proxy header handling exists in the repository.

### 0.2.3 Web Search Research Conducted

- **chi v4.0.2 `middleware.RealIP` behavior**: Confirmed via the official pkg.go.dev documentation for `github.com/go-chi/chi/middleware` and GitHub issue #708. In v4.0.2, the middleware parses `X-Forwarded-For` first, then `X-Real-IP` as fallback. For multi-IP `X-Forwarded-For` values (e.g., `"10.0.0.1, 192.168.1.1"`), only the first IP is extracted. The `Forwarded` header (RFC 7239) is not supported. The middleware overwrites `r.RemoteAddr` in place, so downstream handlers (including the logger) see the proxied IP.
- **chi v4.0.2 header precedence edge cases**: GitHub issue #708 documents that when multiple instances of the same header exist, chi accepts the first header value; proxied services with different logic (e.g., last-value-wins) may disagree, creating spoofing risk.
- **X-Forwarded-Proto handling**: Not supported by chi's `RealIP` middleware in v4. The middleware only modifies `RemoteAddr`, not request scheme or TLS state.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation for the runtime behavior investigation:

- **Module: `config/config.go`**
  - Public APIs: `LoadConfig(configDir, configName)`, `GetSFTPDConfig()`, `GetHTTPDConfig()`, `GetProviderConf()`
  - Current documentation: Inline Go doc comments only; no external docs
  - Documentation needed: Explanation of the Viper config search path (configDir → platform paths → `.`), what happens when no config file is found (defaults used, warning logged, not fatal), and how this creates CWD sensitivity

- **Module: `dataprovider/sqlite.go`**
  - Public APIs: `initializeSQLiteProvider(basePath)` (internal)
  - Current documentation: None
  - Documentation needed: SQLite file existence check (`os.Stat`), empty-file rejection (size == 0), connection string defaults (`file:<path>?cache=shared`), and how `basePath` (configDir) determines DB location

- **Module: `service/service.go`**
  - Public APIs: `Service.Start()`, `Service.Wait()`, `Service.Stop()`
  - Current documentation: Package-level comment only
  - Documentation needed: Full startup sequence, error propagation from data provider init, SFTP/HTTP goroutine lifecycle, and shutdown channel semantics

- **Module: `httpd/router.go`**
  - Public APIs: `initializeRouter(staticFilesPath)` (internal)
  - Current documentation: None
  - Documentation needed: Complete middleware chain, route table (`/` → 301, `/web` → 301, `/metrics` → Prometheus, 404 handler), and how `middleware.RealIP` placement affects logged IPs

- **Module: `httpd/web.go`**
  - Public APIs: `loadTemplates(templatesPath)` (internal)
  - Current documentation: None
  - Documentation needed: Template file requirements (base.html, users.html, user.html, connections.html, message.html), `template.Must` panic behavior on missing files

- **Module: `sftpd/server.go`**
  - Public APIs: `Configuration.Initialize(configDir)`
  - Current documentation: None
  - Documentation needed: Port binding via `net.Listen`, host key auto-generation (`checkHostKeys`), error propagation on bind failure

- **Module: `logger/request_logger.go`**
  - Public APIs: `NewStructuredLogger(logger)`, `StructuredLoggerEntry.Write(status, bytes, elapsed)`
  - Current documentation: Inline comments
  - Documentation needed: How `r.RemoteAddr` is captured (post-RealIP modification), scheme detection (`r.TLS != nil`), log field structure, and the call to `metrics.HTTPRequestServed(status)`

- **Module: `metrics/metrics.go`**
  - Public APIs: `HTTPRequestServed(status)`
  - Current documentation: None
  - Documentation needed: Counter names, status-code classification logic (2xx→OK, 4xx→client error, 5xx→server error, 3xx→total only), `promauto` registration

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented startup behavior**: No existing document explains the config search path, default values, or how different starting conditions (missing DB, port conflict, missing templates) affect startup success
- **Undocumented CWD sensitivity**: The fact that Viper searches `.` (current directory) for config — creating behavior that varies by working directory — is not documented anywhere
- **Undocumented proxy header handling**: No documentation explains which headers chi's `middleware.RealIP` honors, which it ignores, or how this affects logged IP addresses
- **Undocumented metrics classification**: The gap where 3xx responses only increment the total counter but no subcategory is not documented
- **Undocumented host key side effect**: The auto-generation of SSH host keys in configDir on first start is mentioned only in code comments
- **Undocumented scheme detection limitation**: The logger's reliance on `r.TLS` (rather than forwarded headers) for scheme detection behind a TLS-terminating proxy is not documented

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/sftpgo_44634210287c.md` will follow a question-and-answer structure organized by investigation area, with each section providing code-grounded analysis and illustrative snippets:

```
blitzy/
└── documentation/
    └── sftpgo_44634210287c.md
        ├── Introduction (scope, version, methodology)
        ├── 1. Configuration Search Path and Working Directory Effects
        │   ├── How Viper discovers config files
        │   ├── Why CWD matters
        │   └── Illustrative startup log snippets
        ├── 2. SQLite Data Provider: Three First-Start States
        │   ├── Missing DB file
        │   ├── Empty (zero-byte) DB file
        │   ├── Valid DB file
        │   └── Effect on HTTP endpoints after each state
        ├── 3. SFTP Port Conflict Behavior
        │   ├── What net.Listen returns on bind failure
        │   ├── Shutdown channel propagation
        │   └── Exit status and log output
        ├── 4. Missing Web UI Templates and Static Assets
        │   ├── template.Must panic behavior
        │   ├── Comparison: missing templates vs missing static
        │   └── What the HTTP server returns (or doesn't)
        ├── 5. HTTP Endpoint Response Mapping
        │   ├── GET / → 301 redirect
        │   ├── GET /web → 301 redirect
        │   ├── GET /metrics → Prometheus output
        │   ├── Missing paths → JSON 404
        │   └── Response header snippets
        ├── 6. Proxy Header Resolution
        │   ├── chi v4.0.2 middleware.RealIP behavior
        │   ├── X-Forwarded-For vs X-Real-IP priority
        │   ├── Multi-IP forwarding chains
        │   ├── Forwarded header (RFC 7239) — not supported
        │   ├── Scheme detection limitation
        │   └── Representative access log line
        ├── 7. Metrics Counter Behavior
        │   ├── Counter names and classification logic
        │   ├── The 3xx gap
        │   ├── With vs without proxy headers
        │   └── /metrics output snippets
        └── 8. Summary Table
            └── Scenario × behavior matrix
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract startup sequence logic from `service/service.go` lines 47–117 to map the complete init chain
- Extract config search path from `config/config.go` `LoadConfig` function and `config/config_linux.go` platform paths
- Extract SQLite checks from `dataprovider/sqlite.go` `initializeSQLiteProvider` — `os.Stat` call, size check, connection string construction
- Extract route table from `httpd/router.go` `initializeRouter` — all `r.Get`, `r.Mount`, `router.NotFound` handlers
- Extract template loading from `httpd/web.go` `loadTemplates` — the `template.Must(template.ParseFiles(...))` pattern
- Extract SFTP binding from `sftpd/server.go` `Initialize` — `net.Listen("tcp", bindAddr)` call
- Extract middleware chain from `httpd/router.go` — `middleware.RequestID`, `middleware.RealIP`, logger, `middleware.Recoverer` order
- Extract log entry fields from `logger/request_logger.go` `NewLogEntry` — `remote_addr`, `proto`, `method`, `user_agent`, `uri`, `request_id`
- Extract metrics classification from `metrics/metrics.go` `HTTPRequestServed` — status range checks

**Snippet Strategy:**

All code snippets in the output document will be short (2-5 lines) and focused on making behavior "undeniable" per the user's request:
- Startup log lines: zerolog JSON output with redacted timestamps showing `sender`, `message`, and relevant fields
- HTTP response headers: `HTTP/1.1 301 Moved Permanently` with `Location` header
- Access log line: Single zerolog JSON line with `remote_addr` showing proxied IP, volatile fields redacted
- Metrics lines: Specific `sftpgo_http_*` counter lines from `/metrics` output with integer values

### 0.4.3 Diagram and Visual Strategy

- **Mermaid flowchart**: Startup sequence showing the decision tree from `Service.Start()` through config loading, data provider init, SFTP goroutine, and HTTP goroutine — including failure paths
- **Mermaid sequence diagram**: HTTP request lifecycle through the middleware chain (RequestID → RealIP → Logger → Router → Handler → Logger.Write → Metrics)
- **Summary table**: Scenario × outcome matrix covering all six investigation areas with columns for startup result, exit status, HTTP behavior, and logged IP

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/sftpgo_44634210287c.md` | CREATE | `config/config.go`, `config/config_linux.go`, `dataprovider/sqlite.go`, `dataprovider/dataprovider.go`, `service/service.go`, `httpd/httpd.go`, `httpd/router.go`, `httpd/web.go`, `logger/request_logger.go`, `logger/logger.go`, `metrics/metrics.go`, `sftpd/server.go`, `cmd/root.go`, `cmd/serve.go`, `utils/version.go`, `sftpgo.json` | Complete Q&A document covering all six runtime behavior investigation areas with code-grounded answers, illustrative snippets, mermaid diagrams, and summary matrix |

Only one documentation file is being created. No existing files are updated or deleted.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/sftpgo_44634210287c.md
Type: Technical Investigation / Runtime Behavior Q&A
Source Code: All 16 source files listed above
Sections:
    - Introduction (version 0.9.5-dev, scope, methodology)
    - Section 1: Configuration Search Path and Working Directory Effects
        - Viper search order from config/config.go LoadConfig
        - Platform-specific paths from config/config_linux.go
        - Default values from config/config.go init()
        - CWD sensitivity explanation with sftpgo.json in repo root
        - Log file path CWD-relative behavior from logger/logger.go
        - Illustrative startup log snippets
    - Section 2: SQLite Data Provider First-Start States
        - Missing file: os.Stat error path from dataprovider/sqlite.go
        - Empty file: fi.Size()==0 rejection from dataprovider/sqlite.go
        - Valid file: successful open with cache=shared from dataprovider/sqlite.go
        - Startup abort on provider init failure from service/service.go
        - HTTP endpoint availability in each state
    - Section 3: SFTP Port Conflict Behavior
        - net.Listen failure from sftpd/server.go Initialize
        - Goroutine error propagation from service/service.go
        - Shutdown channel semantics
        - Exit status implications
    - Section 4: Missing Web UI Templates and Static Assets
        - template.Must panic from httpd/web.go loadTemplates
        - Distinction between missing templates (panic at init) vs missing static files (404 at runtime)
        - Recovery behavior (middleware.Recoverer does not help — panic is pre-server)
    - Section 5: HTTP Endpoint Response Mapping
        - GET / → 301 to /web/users from httpd/router.go
        - GET /web → 301 to /web/users from httpd/router.go
        - GET /metrics → Prometheus handler from httpd/router.go
        - Unknown paths → JSON 404 from httpd/router.go router.NotFound
        - Response header snippets for each endpoint
    - Section 6: Proxy Header Resolution
        - chi v4.0.2 middleware.RealIP: XFF first, XRI fallback
        - Multi-IP XFF parsing (first IP only)
        - Forwarded header (RFC 7239) not supported
        - Scheme detection via r.TLS (not forwarded headers)
        - Representative access log line
        - Working directory has no effect on header processing
    - Section 7: Metrics Counter Behavior
        - Counter names from metrics/metrics.go
        - HTTPRequestServed classification logic
        - The 3xx redirect gap (total only, no subcategory)
        - Proxy headers do not affect counter values
        - /metrics output snippets
    - Section 8: Summary Table (scenario × outcome matrix)
Diagrams:
    - Startup sequence flowchart (mermaid)
    - HTTP request middleware pipeline (mermaid sequence)
Key Citations: All 16 source files with specific function and line references
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be created or updated. The repository does not use any documentation generator. The output is a standalone Markdown file placed in the `blitzy/documentation/` directory per the project rule `SWE-AtlasQnA-Repo`.

### 0.5.4 Cross-Documentation Dependencies

- No shared content or includes — the document is self-contained
- No navigation links to other documents — none exist in the repository
- No table of contents updates — no TOC infrastructure exists
- No index or glossary updates — none exist

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No documentation tools or packages are required for this task. The output is a standalone Markdown file authored directly. No documentation generator (mkdocs, sphinx, docusaurus, godoc) is involved.

The following table lists the SFTPGo runtime dependencies that are directly relevant to the investigation and will be referenced in the output document. Versions are taken from `go.mod`:

| Registry | Package Name | Version | Relevance to Documentation |
|---|---|---|---|
| Go modules | `github.com/go-chi/chi` | v4.0.2+incompatible | HTTP router and middleware; `middleware.RealIP` is central to proxy header analysis |
| Go modules | `github.com/go-chi/render` | v1.0.1 | JSON response rendering for API endpoints and 404 handler |
| Go modules | `github.com/spf13/viper` | v1.6.1 | Configuration management; config file search path behavior is a primary investigation area |
| Go modules | `github.com/spf13/cobra` | v0.0.5 | CLI command structure; defines default flag values for configDir, configFile, logFilePath |
| Go modules | `github.com/rs/zerolog` | v1.17.2 | Structured JSON logging; defines the log entry format documented in access log snippets |
| Go modules | `github.com/prometheus/client_golang` | v1.3.0 | Prometheus metrics; `promauto` registration and `promhttp.Handler()` for /metrics endpoint |
| Go modules | `github.com/mattn/go-sqlite3` | v2.0.2+incompatible | SQLite driver; governs database file handling behavior |
| Go modules | `gopkg.in/natefinsh/lumberjack.v2` | v2.0.0 | Log file rotation; receives the (possibly CWD-relative) log file path |
| Go modules | `golang.org/x/crypto` | v0.0.0-20200109152110 | SSH server implementation; host key generation uses `rsa.GenerateKey` |
| Go (stdlib) | `html/template` | Go 1.13 | Template parsing; `template.Must` panic behavior on missing files |
| Go (stdlib) | `net` | Go 1.13 | `net.Listen("tcp", addr)` for SFTP port binding; `net.ParseIP` in RealIP validation |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation links need updating since no documentation files currently exist in the repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's requirements define six distinct investigation areas. Coverage targets for each:

| Investigation Area | Source Files Analyzed | Key Functions Traced | Coverage Target |
|---|---|---|---|
| Clean-start / CWD isolation | `config/config.go`, `config/config_linux.go`, `cmd/root.go` | `LoadConfig`, `setViperAdditionalConfigPaths`, `addServeFlags` | 100% — all config search paths traced |
| SQLite data provider states | `dataprovider/sqlite.go`, `dataprovider/dataprovider.go`, `service/service.go` | `initializeSQLiteProvider`, `Initialize`, `Start` | 100% — all three states (missing, empty, valid) documented |
| SFTP port conflict | `sftpd/server.go`, `service/service.go` | `Initialize`, `Start` | 100% — bind failure path fully traced |
| Missing web UI assets | `httpd/web.go`, `httpd/httpd.go`, `httpd/router.go` | `loadTemplates`, `Initialize`, `initializeRouter` | 100% — template panic and static file fallback both documented |
| Proxy header resolution | `httpd/router.go`, `logger/request_logger.go`, chi v4.0.2 `middleware.RealIP` | `initializeRouter`, `NewLogEntry`, `RealIP` | 100% — all three headers (Forwarded, XFF, XRI) addressed; multi-IP chains covered |
| Metrics counter behavior | `metrics/metrics.go`, `logger/request_logger.go` | `HTTPRequestServed`, `Write` | 100% — all status ranges and the 3xx gap documented |

Overall target: **100% coverage of all six investigation areas** with code-grounded evidence for every claim.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question posed in the user's prompt must receive a direct, unambiguous answer
- Each answer must cite the specific source file and function that proves it
- All requested snippet types must be present: startup log lines, HTTP response headers, access log line (with volatile fields redacted), and `/metrics` counter lines
- Both "with proxy headers" and "without proxy headers" scenarios must be contrasted

**Accuracy validation:**
- All code paths described must be traceable to actual source code in this checkout (branch `sftpgo_44634210287c`)
- No assumptions are permitted — the project rule states "base your answers on the code as truth"
- HTTP status codes, redirect targets, and JSON response bodies must exactly match the route definitions in `httpd/router.go`
- Metric counter names must exactly match the Prometheus registrations in `metrics/metrics.go`
- Chi middleware behavior must align with v4.0.2 documentation, not v5 or master

**Clarity standards:**
- Each section opens with a direct answer to the question before providing supporting evidence
- Code citations use the format `Source: filename.go:FunctionName` or `Source: filename.go:LineNumber`
- Log and response snippets are realistic (following zerolog JSON format) with only volatile fields (timestamps, request IDs) redacted
- A summary matrix at the end provides a quick-reference view across all scenarios

**Maintainability:**
- Source citations are anchored to function names (not just line numbers) for resilience against minor code changes
- The document declares the exact SFTPGo version (0.9.5-dev) and chi version (v4.0.2) it applies to

### 0.7.3 Example and Diagram Requirements

- **Minimum snippets per investigation area**: At least one illustrative snippet per area; proxy header section requires multiple (with/without headers, multi-IP chain)
- **Diagram types required**: Startup sequence flowchart (mermaid), HTTP middleware pipeline sequence diagram (mermaid)
- **Snippet format**: Fenced code blocks with language annotation (`json` for log lines, `http` for response headers, `text` for metrics output)
- **Snippet authenticity**: All snippets must be derivable from the actual code — log formats from zerolog field definitions, HTTP responses from chi router handlers, metrics from Prometheus client output format

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `blitzy/documentation/sftpgo_44634210287c.md` — the sole deliverable

**Source files analyzed for documentation content (read-only, no modifications):**
- `config/config.go` — Viper config loading, search paths, defaults
- `config/config_linux.go` — Linux-specific config search paths
- `cmd/root.go` — CLI flag definitions, default values
- `cmd/serve.go` — `serve` command wiring
- `service/service.go` — Service lifecycle, startup orchestration
- `dataprovider/dataprovider.go` — Provider interface, initialization
- `dataprovider/sqlite.go` — SQLite file checks, connection setup
- `httpd/httpd.go` — HTTP server configuration, path resolution
- `httpd/router.go` — Route table, middleware chain
- `httpd/web.go` — Template loading, `template.Must` usage
- `httpd/api_utils.go` — API response format, utility functions
- `logger/logger.go` — Logger initialization, log levels
- `logger/request_logger.go` — HTTP access log format, metrics call
- `metrics/metrics.go` — Prometheus counter definitions, status classification
- `sftpd/server.go` — SFTP server initialization, port binding, host key generation
- `utils/version.go` — Version info (0.9.5-dev)
- `utils/utils.go` — `GetIPFromRemoteAddress` helper
- `sftpgo.json` — Default configuration reference
- `go.mod` — Dependency versions

**External references analyzed:**
- chi v4.0.2 `middleware.RealIP` documentation and behavior (pkg.go.dev, GitHub)

**Investigation areas covered:**
- Config search path and CWD isolation effects
- SQLite data provider: missing, empty, and valid DB file states
- SFTP port conflict (bind failure)
- Missing web UI templates and static assets
- HTTP endpoint responses for `/`, `/web`, `/metrics`, and unknown paths
- Proxy header resolution (`Forwarded`, `X-Forwarded-For`, `X-Real-IP`)
- Multi-IP forwarding chain handling
- Scheme detection behind TLS-terminating proxy
- Prometheus metrics counter classification and the 3xx gap
- Metrics behavior with and without proxy headers

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No Go source files will be created, modified, or deleted. The project rule explicitly prohibits modifying existing files.
- **Test file modifications**: No test files will be changed.
- **Feature additions or code refactoring**: No runtime behavior changes.
- **Deployment or infrastructure changes**: No Dockerfiles, CI configs, or deployment scripts.
- **Non-SQLite data providers**: The analysis focuses on SQLite as the default. PostgreSQL (`dataprovider/pgsql.go`), MySQL (`dataprovider/mysql.go`), and BoltDB (`dataprovider/bolt.go`) startup behaviors are not investigated.
- **Portable mode**: `StartPortableMode` in `service/service.go` is out of scope — the user's scenario is the standard `serve` command.
- **SFTP protocol behavior**: SFTP command handling, file transfer, quota enforcement, and SSH authentication flows are not investigated.
- **Web UI rendering**: The content and behavior of the web UI pages (user management, connection list) are not investigated — only the template loading mechanism.
- **API CRUD operations**: REST API endpoints for user management (`/api/v1/user`), quota scans, connections, backup/restore are not investigated.
- **S3/GCS virtual filesystem**: The `vfs` package and cloud storage integration are out of scope.
- **Windows-specific behavior**: Only Linux config paths (`$HOME/.config/sftpgo`, `/etc/sftpgo`) are analyzed.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: N/A — standalone Markdown file, no build step required
- **Documentation preview command**: Any Markdown renderer (e.g., `grip sftpgo_44634210287c.md`, VS Code Markdown preview, or GitHub rendering)
- **Diagram generation command**: Mermaid diagrams are embedded inline in fenced code blocks; rendering is handled by any Mermaid-compatible Markdown viewer
- **Documentation deployment command**: N/A — file is committed directly to `blitzy/documentation/`
- **Default format**: GitHub-Flavored Markdown with Mermaid diagram blocks
- **Citation requirement**: Every section must reference source files using the format `Source: <filename>:<FunctionName>` or `Source: <filename>:L<LineNumber>`
- **Style guide**: The document follows the Q&A investigation style mandated by the user — direct answers first, then supporting evidence, then illustrative snippets
- **Documentation validation**: Manual review against source code; no automated linting configured

### 0.9.2 Output File Placement

The output document must be placed at:

```
<repo_root>/blitzy/documentation/sftpgo_44634210287c.md
```

This follows the project rule `SWE-AtlasQnA-Repo` which specifies:
- Directory: `blitzy/documentation`
- Filename: `<source_branch_name>.md` → `sftpgo_44634210287c.md`
- The `blitzy/documentation/` directory does not currently exist and must be created

## 0.10 Rules for Documentation

The following rules apply to this documentation task, derived from user instructions and the project implementation rules:

- **Do not modify any existing files in the source repository.** The repository must remain unchanged. The only permitted file operation is creating the new document at `blitzy/documentation/sftpgo_44634210287c.md`.
- **Do not make assumptions; base all answers on the code as the truth.** Every claim in the document must be traceable to a specific source file, function, or code path in this checkout.
- **Provide thinking and rationale behind the answers.** The document must explain *why* a behavior occurs, not just *what* happens — citing the specific code logic that produces the observed outcome.
- **Temporary scripts may be used for observation, but should be cleaned up afterward.** The document may describe observation scripts for the reader's use, but must include cleanup instructions. The document creation itself must not leave temporary artifacts in the repository.
- **Snippet requirements are non-negotiable.** The user explicitly requests: startup log lines, HTTP response headers, an access log line with volatile fields redacted, and `/metrics` counter lines showing how counters move with and without proxy headers. All of these must appear in the document.
- **Redact volatile fields.** Timestamps, request IDs, and other run-specific values in log and response snippets must be replaced with placeholder markers (e.g., `<TIMESTAMP>`, `<REQUEST_ID>`) to focus on the deterministic behavior.
- **Cover all named scenarios.** The user specifically lists: SQLite missing vs empty vs usable, SFTP port already taken, web UI assets not found, requests to `/`, `/web`, `/metrics`, and missing paths, proxy headers `Forwarded` / `X-Forwarded-For` / `X-Real-IP` with multi-IP chains. Every scenario must be addressed.
- **Address working directory effects explicitly.** The user asks whether starting from the repo root vs another folder matters. The document must give a definitive, code-grounded answer for each behavior area (config loading, DB path, template path, log file path, proxy header processing).

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| File Path | Purpose in Analysis |
|---|---|
| `config/config.go` | Viper config loading, search path construction, default values (`init()`), `LoadConfig()` function |
| `config/config_linux.go` | Linux platform-specific config paths: `$HOME/.config/sftpgo`, `/etc/sftpgo` |
| `cmd/root.go` | CLI flag definitions, default configDir (`.`), default configFile (`sftpgo`), default logFilePath (`sftpgo.log`), env var bindings |
| `cmd/serve.go` | `serve` command wiring — creates `service.Service` and calls `Start()` then `Wait()` |
| `service/service.go` | Service lifecycle: `Start()` (logger init → config load → provider init → SFTP goroutine → HTTP goroutine), `Wait()`, `Stop()`, `StartPortableMode()` |
| `dataprovider/dataprovider.go` | Provider interface, `Initialize(cnf, basePath)`, availability timer (30s tick) |
| `dataprovider/sqlite.go` | `initializeSQLiteProvider(basePath)`: `os.Stat` for existence, `fi.Size() == 0` rejection, `file:<path>?cache=shared`, `MaxOpenConns(1)` |
| `httpd/httpd.go` | `Conf` struct, `Initialize(configDir)`: path resolution for backups/static/templates, `http.Server` creation with `ReadTimeout: 300s`, `ListenAndServe()` |
| `httpd/router.go` | `initializeRouter(staticFilesPath)`: middleware chain (RequestID → RealIP → StructuredLogger → Recoverer), route table, NotFound/MethodNotAllowed handlers |
| `httpd/web.go` | `loadTemplates(templatesPath)`: `template.Must(template.ParseFiles(...))` with 5 required template files |
| `httpd/api_utils.go` | `apiResponse` struct, `sendAPIResponse()`, HTTP client utilities |
| `logger/logger.go` | `InitLogger()`: lumberjack config, console logger setup, `zerolog.TimeFieldFormat` |
| `logger/request_logger.go` | `NewLogEntry()`: captures `r.RemoteAddr`, scheme from `r.TLS`, URI; `Write()`: calls `metrics.HTTPRequestServed(status)`, logs with `resp_status`, `resp_size`, `elapsed_ms` |
| `metrics/metrics.go` | `promauto` counter registrations, `HTTPRequestServed(status)`: 200-299→OK, 400-499→client error, 500+→server error; 3xx increments total only |
| `sftpd/server.go` | `Initialize(configDir)`: `checkHostKeys` (auto-generates `id_rsa`), `net.Listen("tcp", addr)`, SSH server configuration |
| `utils/version.go` | `version = "0.9.5-dev"`, `VersionInfo` struct |
| `utils/utils.go` | `GetIPFromRemoteAddress()`: `net.SplitHostPort` helper |
| `sftpgo.json` | Default config: SFTP port 2022, SQLite driver with `sftpgo.db`, HTTPD port 8080 on `127.0.0.1` |
| `go.mod` | Dependency versions: chi v4.0.2, viper v1.6.1, zerolog v1.17.2, prometheus v1.3.0, sqlite3 v2.0.2, etc. |
| `main.go` | Entry point: imports SQLite/MySQL/PgSQL drivers, calls `cmd.Execute()` |

Folders explored via `get_source_folder_contents`:
- Root (`""`)
- `config/`
- `httpd/`
- `service/`
- `cmd/`
- `metrics/`
- `logger/`
- `dataprovider/`
- `sftpd/`

### 0.11.2 External References

| Source | URL / Identifier | Purpose |
|---|---|---|
| chi v4.0.2 middleware.RealIP documentation | pkg.go.dev/github.com/go-chi/chi/middleware | Confirmed header priority (XFF first, XRI fallback), `Forwarded` not supported |
| chi GitHub Issue #708 | github.com/go-chi/chi/issues/708 | Documented RealIP limitations: spoofing risk, multi-header handling, no X-Forwarded-Proto support |
| chi middleware/realip.go source (multiple versions) | github.com/go-chi/chi/blob/v0.9.0/middleware/realip.go | Confirmed implementation is ported from Goji, uses only XFF and XRI variables |

### 0.11.3 Attachments

No attachments were provided for this project. No Figma screens or design files are referenced.

