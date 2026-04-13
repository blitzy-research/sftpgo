# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the requirement is to produce a comprehensive investigative Q&A document—`sftpgo_44634210287c.md` placed in `blitzy/documentation/`—that explains SFTPGo's observable runtime behavior across a matrix of first-start conditions, HTTP endpoint responses, proxy-header processing, and Prometheus metric accounting. No source files in the repository are to be modified; the sole deliverable is the new markdown document.

The feature requirements, restated with enhanced clarity, are:

- **Startup-state matrix**: Document what happens when SFTPGo is started with each of three SQLite database conditions—file missing, file present but zero bytes, and file present and properly initialized—and explain the resulting log output, exit status, and whether the SFTP and HTTP servers come up.
- **Port-conflict scenario**: Describe the behavior when SFTP port 2022 is already occupied by another process, including the error message text, whether the HTTP server still starts, and the process exit code.
- **Missing web-UI assets**: Explain the difference between missing `templates/` (which triggers a `template.Must` panic during initialization) and missing `static/` (which only causes per-request 404s at runtime), including the resulting exit codes and user-visible effects.
- **HTTP endpoint responses**: For `GET /`, `GET /web`, `GET /metrics`, and two arbitrary missing paths, document the HTTP status codes, response headers, and body formats (JSON vs. HTML vs. Prometheus exposition).
- **Proxy-header handling**: Clarify exactly which headers the chi v4.0.2 `middleware.RealIP` inspects (`X-Forwarded-For`, `X-Real-IP`), which it ignores (`Forwarded` per RFC 7239, `True-Client-IP`), what priority order applies, how multi-IP `X-Forwarded-For` chains are parsed (leftmost IP wins, comma-split), and what address and scheme the structured request logger ultimately records.
- **Working-directory sensitivity**: Explain why starting from the repo root versus an empty temp directory can change behavior even with an explicit `--config-dir`, because Viper adds CWD (`.`) as a fallback config search path and because the log file path is resolved relative to CWD, not `--config-dir`.
- **Metric counter accounting**: Show how `sftpgo_http_req_total`, `sftpgo_http_req_ok_total`, `sftpgo_http_client_errors_total`, and `sftpgo_http_server_errors_total` increment for 301 redirects, 200 responses, and 404 responses, noting that 3xx statuses fall through all classification buckets and only increment the total.
- **Representative snippets**: Provide short, redacted examples of startup log lines, HTTP response headers, a structured access-log entry with proxy-style headers, and `/metrics` output showing counter progression.

Implicit requirements detected:

- The document must include reasoning or rationale behind each answer, per the project implementation rule `SWE-AtlasQnA-Repo`.
- All claims must be grounded in specific source code evidence (file paths, line references, function names) rather than assumptions.
- The interaction between `config.LoadConfig` silently returning an error (whose return value is discarded by `service.Start()`) and the use of hardcoded defaults is central to explaining "ghost state" between runs and must be explicitly addressed.
- The fact that the scheme field in the HTTP access log is derived from `r.TLS` (always `nil` behind a TLS-terminating proxy) rather than from `X-Forwarded-Proto` must be called out, as this directly contradicts a common operator expectation.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability**: The implementation rule `SWE-AtlasQnA-Repo` mandates: "Do not modify any existing files in the source repository. Do not add any other code in the source repository (besides the above requested document)." The sole artifact is the markdown file.
- **Document naming**: The file must be named `sftpgo_44634210287c.md` (the source branch name) and placed in `blitzy/documentation/`.
- **Evidence-based reasoning**: Every answer must cite specific source code locations. No assumptions are permitted.
- **Snippet redaction**: Volatile fields (timestamps, request IDs, PIDs) in example log lines and HTTP headers should be replaced with placeholders such as `<TIMESTAMP>` or `<REQID>`.
- **Temporary scripts**: The user authorizes temporary observation scripts for validation, but the implementation rule prohibits adding code to the repo. All answers will be derived from static source analysis; temporary scripts are unnecessary because the code paths are fully deterministic and traceable.

### 0.1.3 Technical Interpretation

These requirements translate to the following technical implementation strategy:

- To document the SQLite startup matrix, we will trace the code path through `dataprovider/sqlite.go → initializeSQLiteProvider()` which calls `os.Stat()` and checks `fi.Size() == 0`, then follow the error propagation through `dataprovider/dataprovider.go → Initialize()` back to `service/service.go → Start()`, noting that `Start()` returns the error which causes the cobra `Run` function to skip `service.Wait()` and exit with code 0.
- To document port-conflict behavior, we will trace `sftpd/server.go → Initialize()` where `net.Listen("tcp", addr)` fails, the error is logged, the goroutine sends to `s.Shutdown`, and `service.Wait()` unblocks, leading to exit code 0.
- To document missing-asset behavior, we will trace `httpd/httpd.go → Initialize()` calling `httpd/web.go → loadTemplates()` which uses `template.Must()` (panics on error, exit code 2), versus the static file server `http.FileServer(http.Dir(path))` which defers filesystem access to per-request time.
- To document HTTP endpoint behavior, we will trace the chi router setup in `httpd/router.go` for each registered route and the `NotFound` handler.
- To document proxy-header behavior, we will analyze chi v4.0.2's `middleware.RealIP` (checks `X-Forwarded-For` then `X-Real-IP`, ignores `Forwarded` and `True-Client-IP`), then trace how `logger/request_logger.go → NewLogEntry()` reads the already-modified `r.RemoteAddr` and derives `scheme` from `r.TLS`.
- To document metric accounting, we will analyze `metrics/metrics.go → HTTPRequestServed(status)` and its classification ranges: 200–299 → OK, 400–499 → client errors, 500+ → server errors, with 3xx falling through all conditional branches.


## 0.2 Repository Scope Discovery


### 0.2.1 Comprehensive File Analysis

The following source files were examined to derive answers for every scenario in the user's prompt. Each file is listed with its role in the analysis.

**Configuration loading and path resolution**

| File | Role in Analysis |
|---|---|
| `config/config.go` | Viper-based config loading; `init()` seeds all defaults; `LoadConfig()` adds configDir, platform paths, then CWD to Viper search; `ReadInConfig()` failure is non-fatal (logged as warning, defaults used); return value is **discarded** by caller `service.Start()` |
| `config/config_linux.go` | Adds `$HOME/.config/sftpgo` and `/etc/sftpgo` as additional Viper search paths on Linux |
| `config/config_nolinux.go` | No-op on non-Linux platforms; no additional paths added |
| `sftpgo.json` | Default configuration: SFTP port 2022 on `0.0.0.0`, HTTP port 8080 on `127.0.0.1`, SQLite driver with `sftpgo.db`, relative paths `templates`, `static`, `backups` |

**Service lifecycle and startup orchestration**

| File | Role in Analysis |
|---|---|
| `service/service.go` | `Start()`: initializes logger → loads config (ignores error) → initializes data provider (fatal on error) → launches SFTP goroutine → launches HTTP goroutine if `BindPort > 0`; `Wait()`: blocks on Shutdown channel |
| `cmd/serve.go` | Cobra `Run` handler: calls `service.Start()`; if error → function returns (no `Wait()`); if nil → blocks on `Wait()`. Uses `Run` not `RunE`, so errors are not propagated to Cobra's exit handling |
| `cmd/root.go` | Defines CLI flags: `--config-dir` (default `.`), `--config-file` (default `sftpgo`), `--log-file-path` (default `sftpgo.log`); `Execute()` calls `os.Exit(1)` only on Cobra-level errors, not on `Run` function failures |
| `main.go` | Thin entry point: imports MySQL/PostgreSQL/SQLite drivers for side effects, delegates to `cmd.Execute()` |

**Data provider and SQLite initialization**

| File | Role in Analysis |
|---|---|
| `dataprovider/dataprovider.go` | `Initialize(cnf, basePath)`: dispatches to provider-specific init; on success starts 30-second availability timer updating `sftpgo_dataprovider_availability` metric |
| `dataprovider/sqlite.go` | `initializeSQLiteProvider(basePath)`: resolves `config.Name` relative to basePath if not absolute; `os.Stat(dbPath)` → file missing returns error with "does not exists" message; file exists but `fi.Size() == 0` returns error with "is invalid" message; file valid → opens with `file:<path>?cache=shared`, MaxOpenConns=1 |
| `sql/sqlite/20190828.sql` | Initial schema: `CREATE TABLE users(...)` with core fields |
| `sql/sqlite/20191112.sql` | Adds `expiration_date` column |
| `sql/sqlite/20191230.sql` | Adds `filters` and `filesystem` columns |
| `sql/sqlite/20200116.sql` | Adds `status`, `last_login` columns |

**HTTP server, routing, and web UI**

| File | Role in Analysis |
|---|---|
| `httpd/httpd.go` | `Conf` struct with BindPort, BindAddress, TemplatesPath, StaticFilesPath, BackupsPath; `Initialize(configDir)` resolves relative paths against configDir via `filepath.Join`, calls `loadTemplates`, `initializeRouter`, then `ListenAndServe` |
| `httpd/router.go` | Chi router setup: middleware chain `RequestID → RealIP → StructuredLogger → Recoverer`; route map: `GET /` → 301 to `/web/users`, `GET /web` → 301 to `/web/users`, `GET /metrics` → `promhttp.Handler()`, `NotFound` → JSON `{"error":"","message":"Not Found","status":404}` |
| `httpd/web.go` | `loadTemplates(path)`: uses `template.Must(template.ParseFiles(...))` which **panics** if any template file is absent; templates: `base.html`, `users.html`, `user.html`, `connections.html`, `message.html` |
| `httpd/api_utils.go` | `sendAPIResponse()`: writes JSON `{"error":"...","message":"...","status":N}` with `Content-Type: application/json; charset=utf-8` for non-200 responses |
| `templates/` | Directory containing HTML templates: `base.html`, `users.html`, `user.html`, `connections.html`, `message.html` |
| `static/` | Directory containing CSS/JS assets served by `http.FileServer` at `/static` route with gzip compression |

**SFTP server startup**

| File | Role in Analysis |
|---|---|
| `sftpd/server.go` | `Initialize(configDir)`: configures SSH, generates host keys if needed (`id_rsa` 4096-bit RSA in configDir), opens TCP listener; `net.Listen` failure → error logged as "error starting listener on address %s:%d: %v" → goroutine sends to Shutdown |
| `sftpd/sftpd.go` | Connection tracking, transfer management, idle timeout checker (5-minute interval), supported SSH commands list |

**Logging and request instrumentation**

| File | Role in Analysis |
|---|---|
| `logger/logger.go` | `InitLogger(logFilePath, ...)`: if path non-empty → lumberjack file writer; if empty → stdout; timestamp format `2006-01-02T15:04:05.000`; log file path resolved relative to CWD (not configDir) |
| `logger/request_logger.go` | `NewLogEntry(r)`: records `remote_addr` from `r.RemoteAddr` (post-RealIP-middleware), `scheme` from `r.TLS != nil` (always `http` behind TLS proxy), `uri` as `scheme://host/requestURI`; `Write(status, bytes, elapsed)`: calls `metrics.HTTPRequestServed(status)`, logs info-level structured JSON |

**Metrics**

| File | Role in Analysis |
|---|---|
| `metrics/metrics.go` | Prometheus counters: `sftpgo_http_req_total`, `sftpgo_http_req_ok_total` (200–299), `sftpgo_http_client_errors_total` (400–499), `sftpgo_http_server_errors_total` (500+); `HTTPRequestServed(status)`: 3xx statuses increment only total (fall through all classification branches) |

**Version**

| File | Role in Analysis |
|---|---|
| `utils/version.go` | `version = "0.9.5-dev"`; `VersionInfo` struct with Version, BuildDate, CommitHash |

### 0.2.2 Integration Point Discovery

The following cross-module integration points are central to the behavior being documented:

- **Config → Service**: `service.Start()` calls `config.LoadConfig()` but discards the error return value (line ~72 of `service/service.go`). This means a missing config file is silently tolerated and defaults take effect without any fatal signal.
- **Service → DataProvider → SQLite**: `service.Start()` calls `dataprovider.Initialize()` which dispatches to `initializeSQLiteProvider()`. This IS a fatal path—errors here cause `Start()` to return an error and prevent the SFTP/HTTP goroutines from launching.
- **Service → SFTP (goroutine)**: SFTP server errors in the goroutine send to `s.Shutdown` channel, unblocking `Wait()`.
- **Service → HTTP (goroutine)**: HTTP server initialization panics (missing templates) crash the entire process. Non-panic errors (port conflict) send to `s.Shutdown`.
- **Chi middleware → Logger**: `middleware.RealIP` modifies `r.RemoteAddr` before `logger.NewStructuredLogger` creates the log entry, so the logger always sees the proxy-extracted IP.
- **Logger → Metrics**: `StructuredLoggerEntry.Write()` calls `metrics.HTTPRequestServed(status)` on every request completion, coupling the request log middleware directly to Prometheus counters.

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/sftpgo_44634210287c.md` — The sole deliverable. A comprehensive markdown Q&A document with the following sections:
  - Startup-state matrix (SQLite missing / empty / valid)
  - Port-conflict behavior
  - Missing web-UI assets (templates vs. static)
  - HTTP endpoint responses (`/`, `/web`, `/metrics`, missing paths)
  - Configuration search and working-directory effects
  - Proxy-header handling (X-Forwarded-For, X-Real-IP, Forwarded, scheme logging)
  - Prometheus metric counter accounting
  - Representative redacted snippets (startup logs, HTTP headers, access log lines, `/metrics` output)

No other files are created or modified.


## 0.3 Dependency Inventory


### 0.3.1 Key Packages Relevant to This Analysis

All versions are taken directly from `go.mod` in the repository root. No dependency changes are required—this task produces only a documentation file.

| Registry | Package | Version | Purpose in This Analysis |
|---|---|---|---|
| Go modules | `github.com/go-chi/chi` | v4.0.2 | HTTP router and middleware stack; `middleware.RealIP` drives proxy-header behavior; `middleware.RequestID` generates per-request IDs; `middleware.Recoverer` handles handler panics (not init panics) |
| Go modules | `github.com/rs/zerolog` | v1.17.2 | Structured JSON logger; provides the `zerolog.Logger` used by `logger/request_logger.go` to produce HTTP access log entries with `remote_addr`, `scheme`, `resp_status` fields |
| Go modules | `github.com/spf13/viper` | v1.6.1 | Configuration management; `AddConfigPath`, `ReadInConfig`, `SetConfigName`, `Unmarshal` drive the config file search order and fallback behavior |
| Go modules | `github.com/spf13/cobra` | v0.0.5 | CLI framework; `Run` (not `RunE`) handler means service startup errors do not propagate to Cobra's exit-code machinery |
| Go modules | `github.com/prometheus/client_golang` | v1.3.0 | Prometheus client library; `promauto.NewCounter` registers HTTP counters; `promhttp.Handler()` serves the `/metrics` endpoint |
| Go modules | `github.com/mattn/go-sqlite3` | v2.0.2 | CGo-based SQLite3 driver; imported for side effects in `main.go`; `initializeSQLiteProvider` opens the database file via `sql.Open("sqlite3", ...)` |
| Go modules | `go.etcd.io/bbolt` | v1.3.3 | BoltDB key-value store; alternative data provider (not directly exercised in the SQLite scenarios but part of the provider dispatch) |
| Go modules | `gopkg.in/natefinch/lumberjack.v2` | v2.0.0 | Log file rotation; `logger.InitLogger` passes the log file path to lumberjack, which resolves relative paths against CWD |
| Go modules | `golang.org/x/crypto` | v0.0.0-20200109152110-61a87790db17 | SSH server implementation used by `sftpd/server.go` for host key management and authentication callbacks |
| Go modules | `github.com/go-chi/render` | v1.0.1 | JSON response rendering for API endpoints; used by `sendAPIResponse` and route handlers |
| Go stdlib | `net/http` | (Go 1.13) | HTTP server (`http.Server`), `http.Redirect` for 301 responses, `http.FileServer` for static assets, `http.Dir` for filesystem rooting |
| Go stdlib | `html/template` | (Go 1.13) | `template.Must` and `template.ParseFiles` for web UI template loading; `Must` panics on parse failure |

### 0.3.2 Dependency Updates

Not applicable. This task creates a single markdown documentation file and requires no dependency additions, removals, or version changes. The analysis is purely source-code-driven.

### 0.3.3 Runtime Version

| Component | Documented Version | Source |
|---|---|---|
| Go | 1.13 | `go.mod` line 3 (`go 1.13`) |
| SFTPGo | 0.9.5-dev | `utils/version.go` line 3 (`const version = "0.9.5-dev"`) |


## 0.4 Integration Analysis


### 0.4.1 Existing Code Touchpoints

No source files are modified. The touchpoints below describe the code paths that must be **traced and documented** in the deliverable markdown file. Each touchpoint maps a user question to the specific functions and control-flow transitions that determine the answer.

**Touchpoint 1 — Configuration search chain (answers "why does CWD matter")**

```
cmd/root.go: configDir default = "."
  → service/service.go: Start() calls config.LoadConfig(s.ConfigDir, s.ConfigFile)
    → config/config.go: LoadConfig() — Viper search order:
        1. viper.AddConfigPath(configDir)
        2. setViperAdditionalConfigPaths()  [Linux: $HOME/.config/sftpgo, /etc/sftpgo]
        3. viper.AddConfigPath(".")          [CWD — always added]
    → config.LoadConfig returns error → service.Start() DISCARDS it
    → defaults from config.init() take effect silently
```

**Touchpoint 2 — SQLite initialization (answers "startup-state matrix")**

```
service/service.go: Start()
  → dataprovider/dataprovider.go: Initialize(providerConf, s.ConfigDir)
    → dataprovider/sqlite.go: initializeSQLiteProvider(basePath)
      → filepath.Join(basePath, config.Name)  [resolves "sftpgo.db" relative to configDir]
      → os.Stat(dbPath):
          - ENOENT  → error "sqlite database file does not exists..."
          - exists, Size()==0 → error "sqlite database file is invalid..."
          - exists, Size()>0  → sql.Open("sqlite3", "file:<path>?cache=shared") → success
    → error returns to Start() → Start() returns error to serve.Run
      → serve.Run skips Wait() → process exits code 0
```

**Touchpoint 3 — SFTP port binding (answers "port already taken")**

```
service/service.go: Start() launches goroutine
  → sftpd/server.go: Initialize(configDir)
    → net.Listen("tcp", bindAddress:bindPort)
      - EADDRINUSE → error returned
    → goroutine logs "could not start SFTP server: %v"
    → s.Shutdown <- true  →  Wait() unblocks → exit code 0
```

**Touchpoint 4 — HTTP initialization and asset loading (answers "missing templates vs. static")**

```
service/service.go: Start() launches goroutine
  → httpd/httpd.go: Initialize(configDir)
    → filepath.Join(configDir, "templates")  [relative path resolution]
    → filepath.Join(configDir, "static")
    → httpd/web.go: loadTemplates(templatesPath)
      → template.Must(template.ParseFiles(...))
        - files missing → ParseFiles returns error → Must panics
        - UNRECOVERED panic in goroutine → runtime terminates process → exit code 2
    → httpd/router.go: initializeRouter(staticFilesPath)
      → http.FileServer(http.Dir(staticFilesPath))  [deferred: no check at init time]
      → requests to /static/* with missing dir → per-request 404
```

**Touchpoint 5 — HTTP endpoint routing (answers "what do /, /web, /metrics, missing paths return")**

```
httpd/router.go: initializeRouter()
  GET "/"        → http.Redirect(w, r, "/web/users", 301)
  GET "/web"     → http.Redirect(w, r, "/web/users", 301)
  GET "/metrics" → promhttp.Handler().ServeHTTP(w, r)  [200, text/plain Prometheus format]
  <not found>    → sendAPIResponse(w, r, nil, "Not Found", 404)
                    → JSON {"error":"","message":"Not Found","status":404}
```

**Touchpoint 6 — Proxy-header pipeline (answers "which IP and scheme get logged")**

```
httpd/router.go: middleware chain order:
  1. middleware.RequestID   → adds X-Request-Id header and context value
  2. middleware.RealIP      → checks X-Forwarded-For (leftmost IP, comma-split)
                             → falls back to X-Real-IP
                             → ignores Forwarded (RFC 7239)
                             → ignores True-Client-IP (chi v5+ only)
                             → overwrites r.RemoteAddr with extracted IP (no port)
  3. logger.NewStructuredLogger → NewLogEntry reads:
       r.RemoteAddr  [already modified by RealIP]
       r.TLS         [nil for plain HTTP → scheme = "http" always]
  4. middleware.Recoverer  → catches handler panics (not init panics)
```

**Touchpoint 7 — Metric counter classification (answers "how do counters move")**

```
logger/request_logger.go: Write(status, bytes, elapsed)
  → metrics/metrics.go: HTTPRequestServed(status)
    → totalHTTPRequests.Inc()                   [always]
    → if status >= 200 && status < 300 → totalHTTPOK.Inc()
    → else if status >= 400 && status < 500 → totalHTTPClientErrors.Inc()
    → else if status >= 500 → totalHTTPServerErrors.Inc()
    → 3xx (e.g. 301): increments ONLY totalHTTPRequests — falls through all else-if branches
```

### 0.4.2 Cross-Cutting Observations

- **Log file vs. configDir**: The log file (`sftpgo.log` by default) is passed to lumberjack as a relative path, resolved against CWD by the OS, NOT against `--config-dir`. This means changing CWD changes where logs appear, even when `--config-dir` is explicitly set.
- **Host key generation side-effect**: `sftpd/server.go` auto-generates `id_rsa` (4096-bit RSA) inside configDir if no host keys are configured and the file does not exist. This creates a persistent artifact in a supposedly "temporary" config directory, which changes behavior on subsequent starts (key already present → no generation).
- **Availability timer**: On successful data provider init, a 30-second ticker starts in `dataprovider/dataprovider.go` that probes provider health and updates the `sftpgo_dataprovider_availability` gauge. This metric appears in `/metrics` output even on the very first scrape.


## 0.5 Technical Implementation


### 0.5.1 File-by-File Execution Plan

There is exactly one file to create. No files are modified.

- **CREATE**: `blitzy/documentation/sftpgo_44634210287c.md`
  - Purpose: Comprehensive Q&A document answering all questions about SFTPGo runtime behavior
  - Format: Markdown with headings, tables, fenced code blocks for log/header/metric snippets
  - Approximate structure: 8–10 top-level sections covering each scenario class, with sub-sections for individual questions, a rationale paragraph per answer, and redacted representative snippets

### 0.5.2 Implementation Approach — Document Content Blueprint

The markdown document will be organized into the following sections, each grounded in specific source file evidence.

**Section A — Startup-State Matrix: SQLite Scenarios**

Three sub-sections covering:

- *SQLite file missing*: Traced through `dataprovider/sqlite.go` `initializeSQLiteProvider()` → `os.Stat()` returns `ENOENT` → error message "sqlite database file does not exists, please be sure to create and initialize a database before starting sftpgo" → `dataprovider.Initialize()` returns error → `service.Start()` returns error → `cmd/serve.go` `Run` function skips `Wait()` → process exits with code 0. Neither SFTP nor HTTP goroutines are launched. A representative startup log snippet will show the error-level message from `service.Start()` ("error initializing data provider: ...") and the console echo.
- *SQLite file empty (0 bytes)*: Same code path; `os.Stat()` succeeds but `fi.Size() == 0` → error message "sqlite database file is invalid, please be sure to create and initialize a database before starting sftpgo" → same fatal exit via `Start()`. Behavior is identical to the missing-file case except for the error message text.
- *SQLite file valid*: `sql.Open("sqlite3", "file:<configDir>/sftpgo.db?cache=shared")` succeeds → `MaxOpenConns` set to 1 → availability timer starts → `Start()` returns nil → SFTP and HTTP goroutines launch.

**Section B — SFTP Port Conflict**

Traced through `sftpd/server.go` `Initialize()` → `net.Listen("tcp", "0.0.0.0:2022")` returns `bind: address already in use` → error logged as "could not start SFTP server: error starting listener on address 0.0.0.0:2022: listen tcp 0.0.0.0:2022: bind: address already in use" → goroutine sends to `s.Shutdown` → `Wait()` unblocks. The HTTP server goroutine may or may not have finished starting; the two goroutines race. Process exit code: 0. A log snippet will show the error-level SFTP message followed by the service shutdown debug message.

**Section C — Missing Web UI Assets**

Two sub-cases:

- *Templates directory missing*: `httpd/httpd.go` `Initialize()` → `filepath.Join(configDir, "templates")` → `httpd/web.go` `loadTemplates()` → `template.ParseFiles()` fails → `template.Must()` panics → unrecovered goroutine panic → Go runtime prints stack trace to stderr → `exit(2)`. The HTTP server never reaches `ListenAndServe`. The SFTP goroutine is racing and may or may not have bound its port before the panic kills the process.
- *Static files directory missing*: `httpd/router.go` `initializeRouter()` creates `http.FileServer(http.Dir(staticFilesPath))`. `http.Dir()` is a type cast (no I/O). `http.FileServer` performs no directory validation at construction time. The HTTP server starts normally. When a client requests `/static/css/sb-admin-2.css`, `FileServer` calls `http.Dir.Open()` which returns `os.ErrNotExist` → response is a plain-text 404. The web UI HTML loads (served from in-memory parsed templates) but references broken CSS/JS resources, rendering a visually broken page.

**Section D — HTTP Endpoint Responses**

| Endpoint | Status | Content-Type | Body Summary | Code Path |
|---|---|---|---|---|
| `GET /` | 301 | `text/html; charset=utf-8` | HTML redirect body with link to `/web/users` | `httpd/router.go` → `http.Redirect(w, r, "/web/users", 301)` |
| `GET /web` | 301 | `text/html; charset=utf-8` | HTML redirect body with link to `/web/users` | `httpd/router.go` → `http.Redirect(w, r, webUsersPath, 301)` |
| `GET /metrics` | 200 | `text/plain; version=0.0.4; charset=utf-8` | Prometheus exposition format with all registered metrics | `httpd/router.go` → `promhttp.Handler()` |
| `GET /nonexistent` | 404 | `application/json; charset=utf-8` | `{"error":"","message":"Not Found","status":404}` | `httpd/router.go` NotFound handler → `sendAPIResponse()` |
| `GET /also/missing` | 404 | `application/json; charset=utf-8` | `{"error":"","message":"Not Found","status":404}` | Same NotFound handler |

Response headers for all endpoints include `X-Request-Id` (UUID from `middleware.RequestID`) and standard Go `net/http` headers (`Date`, `Content-Length` or `Transfer-Encoding`). Redirect responses include a `Location: /web/users` header.

**Section E — Configuration Search and Working-Directory Sensitivity**

The document will explain the Viper config search order from `config/config.go` `LoadConfig()`:
1. `configDir` (from `--config-dir` flag, default `.`)
2. `$HOME/.config/sftpgo` (Linux only, from `config_linux.go`)
3. `/etc/sftpgo` (Linux only)
4. `.` (CWD, always appended)

Key insights to document:
- When `--config-dir=/tmp/testcfg`, CWD is still searched as a fallback. If CWD (or `$HOME/.config/sftpgo` or `/etc/sftpgo`) contains a `sftpgo.json`, it will be loaded despite the explicit configDir.
- `LoadConfig()` returns an error on failure, but `service.Start()` discards it. Defaults are silently used.
- Relative paths (`templates`, `static`, `sftpgo.db`) resolve against `configDir`, not against the directory where the config file was found.
- The log file path (`sftpgo.log`) resolves against CWD via lumberjack, not against configDir.
- Host key `id_rsa` is auto-generated in `configDir` and persists across runs, creating hidden state in a "temporary" directory.

**Section F — Proxy Header Handling**

The document will explain the chi v4.0.2 `middleware.RealIP` behavior:

- Header priority: `X-Forwarded-For` (checked first) → `X-Real-IP` (fallback). Both `Forwarded` (RFC 7239) and `True-Client-IP` are completely ignored.
- `X-Forwarded-For` parsing: split by comma, take leftmost entry, trim whitespace. For `"10.0.0.1, 20.0.0.2, 30.0.0.3"` → `r.RemoteAddr` becomes `"10.0.0.1"`.
- When both `X-Forwarded-For` and `X-Real-IP` are present, `X-Forwarded-For` wins.
- When only `Forwarded: for=1.2.3.4;proto=https` is sent, it is ignored; `r.RemoteAddr` retains the original TCP peer address.
- The middleware sets `r.RemoteAddr` to the bare IP (no port), replacing the original `ip:port` format.
- Scheme logging: `logger/request_logger.go` `NewLogEntry()` derives scheme from `r.TLS != nil`. Behind a TLS-terminating proxy, `r.TLS` is always `nil`, so scheme is always `"http"` regardless of any `X-Forwarded-Proto` header. SFTPGo does not read `X-Forwarded-Proto`.
- Working directory has **zero effect** on proxy header handling. The RealIP middleware is a pure HTTP-layer concern with no filesystem interaction.

**Section G — Prometheus Metric Counter Accounting**

The document will trace `metrics/metrics.go` `HTTPRequestServed(status)`:

| Request | HTTP Status | `sftpgo_http_req_total` | `sftpgo_http_req_ok_total` | `sftpgo_http_client_errors_total` | `sftpgo_http_server_errors_total` |
|---|---|---|---|---|---|
| `GET /` | 301 | +1 | — | — | — |
| `GET /metrics` | 200 | +1 | +1 | — | — |
| `GET /nonexistent` | 404 | +1 | — | +1 | — |

The 301 redirect increments **only** `sftpgo_http_req_total` because 301 is outside all three classification ranges (200–299, 400–499, 500+). This is true whether or not proxy headers are present, because metric accounting is based solely on HTTP status code—the presence or absence of forwarded headers changes only the `remote_addr` logged, not the status code or metric classification.

**Section H — Representative Snippets**

The document will include redacted examples:
- Startup log lines for each SQLite scenario (info + error level messages from `service.Start()` and `dataprovider.Initialize()`)
- HTTP response headers for `GET /` (showing `Location`, `X-Request-Id`, `Content-Type`)
- HTTP response headers for `GET /metrics` (showing Prometheus content type)
- HTTP response headers for `GET /nonexistent` (showing JSON content type and body)
- A structured access-log JSON line for a request with `X-Forwarded-For: 10.0.0.1, 20.0.0.2` showing `remote_addr` as `10.0.0.1` and scheme as `http`
- `/metrics` output showing counter values after a sequence of: one `GET /` (301), one `GET /metrics` (200), one `GET /missing` (404), then a repeat with `X-Forwarded-For` header (same counters, different `remote_addr` in logs but identical metric increments)

### 0.5.3 Implementation Approach Per File

- Establish the document structure with clear H2 sections matching the user's questions
- For each section, lead with the direct answer, follow with the code-path rationale citing specific files and functions, then provide a redacted snippet
- Cross-reference related sections (e.g., the metric section references the endpoint section for status codes)
- Ensure every claim cites a specific source file path


## 0.6 Scope Boundaries


### 0.6.1 Exhaustively In Scope

**Deliverable artifact**

- `blitzy/documentation/sftpgo_44634210287c.md` — the sole file created

**Source files traced for evidence (read-only analysis)**

- `config/config.go` — config loading, defaults, Viper search order
- `config/config_linux.go` — Linux-specific Viper search paths
- `config/config_nolinux.go` — non-Linux no-op
- `sftpgo.json` — default configuration values
- `service/service.go` — startup orchestration, goroutine launch, shutdown channel
- `cmd/serve.go` — cobra `Run` handler, error handling (or lack thereof)
- `cmd/root.go` — CLI flag definitions, default values
- `main.go` — entry point, driver imports
- `dataprovider/dataprovider.go` — provider initialization dispatch, availability timer
- `dataprovider/sqlite.go` — SQLite file existence/size validation, connection string
- `httpd/httpd.go` — HTTP server init, relative path resolution
- `httpd/router.go` — chi router setup, middleware chain, route definitions, NotFound handler
- `httpd/web.go` — template loading with `template.Must`
- `httpd/api_utils.go` — `sendAPIResponse` JSON response format, `apiResponse` struct
- `sftpd/server.go` — TCP listener binding, host key generation, error handling
- `sftpd/sftpd.go` — connection tracking, idle timeout, SSH command list
- `logger/logger.go` — log initialization, file vs. stdout, timestamp format
- `logger/request_logger.go` — HTTP access log fields, scheme derivation, metrics call
- `metrics/metrics.go` — all Prometheus counter/gauge definitions, `HTTPRequestServed` classification
- `utils/version.go` — version constant
- `sql/sqlite/*.sql` — migration scripts (20190828, 20191112, 20191230, 20200116)
- `templates/` — HTML template files (existence verification)
- `static/` — static asset directory (existence verification)
- `go.mod` — dependency versions

**External research conducted**

- chi v4.0.2 `middleware.RealIP` behavior: confirmed via GitHub source, test files, and issue discussions (#708, #711) that v4 checks X-Forwarded-For first (leftmost IP, comma-split), then X-Real-IP; does NOT check Forwarded or True-Client-IP

**Topics explicitly covered in the document**

- SQLite missing → fatal startup error, exit code 0
- SQLite empty (0 bytes) → fatal startup error, exit code 0
- SQLite valid → successful startup
- SFTP port conflict → SFTP goroutine error, HTTP may or may not start, exit code 0
- Templates missing → `template.Must` panic, exit code 2
- Static files missing → server starts, per-request 404s
- `GET /` → 301 redirect to `/web/users`
- `GET /web` → 301 redirect to `/web/users`
- `GET /metrics` → 200 Prometheus exposition
- Missing paths → 404 JSON response
- Config search order and CWD fallback
- Log file path resolved against CWD (not configDir)
- Host key auto-generation side effect in configDir
- `X-Forwarded-For` handling (leftmost IP, comma-split)
- `X-Real-IP` handling (fallback when no X-Forwarded-For)
- `Forwarded` header ignored (RFC 7239, not supported in chi v4)
- Multi-IP forwarding chains
- Scheme always "http" behind TLS proxy (derived from `r.TLS`, not from headers)
- Working directory irrelevance for proxy header handling
- Metric counter classification for 301, 200, 404 responses
- 3xx status codes increment only `total`, not OK/client/server buckets
- Proxy headers do not affect metric counters
- Representative startup log snippets
- Representative HTTP response headers
- Representative access log line with proxy headers
- Representative `/metrics` counter output

### 0.6.2 Explicitly Out of Scope

- **Modifying any existing repository file** — prohibited by implementation rule `SWE-AtlasQnA-Repo`
- **Adding executable code to the repository** — prohibited; no test scripts, Go files, or shell scripts committed
- **Runtime experimentation** — all answers derived from static source code analysis; the code paths are fully deterministic
- **Non-SQLite data providers** — the user's scenario involves the default SQLite driver; MySQL, PostgreSQL, BoltDB, and in-memory provider behaviors are not documented
- **S3 filesystem backend behavior** — not relevant to the user's HTTP/startup/proxy questions
- **SFTP protocol-level behavior** — file transfer operations, SCP, SSH commands are not in scope
- **Windows Service or macOS launchd behavior** — the user's context is Linux with a TLS-terminating reverse proxy
- **Portable mode** — the user describes standard `serve` mode, not portable mode
- **REST API CRUD operations** — user management, quota scanning, backup/restore are not exercised
- **External authentication** — not relevant to the startup and HTTP scenarios
- **Performance tuning or optimization** — out of scope
- **Refactoring recommendations** — the document describes observed behavior, not prescriptive changes


## 0.7 Rules for Feature Addition


### 0.7.1 Implementation Rule: SWE-AtlasQnA-Repo

The project-level implementation rule `SWE-AtlasQnA-Repo` governs this task with the following directives:

- **Create a new markdown document** named `sftpgo_44634210287c.md` (the source branch name) that comprehensively answers the questions posed in the prompt.
- **Provide thinking and rationale** behind every answer. Each answer must explain the "why" by citing specific source code paths, function names, and control-flow logic.
- **Do not make assumptions** — base all answers on the code as the single source of truth. Where behavior depends on runtime conditions (e.g., race between goroutines), explicitly state the nondeterminism rather than guessing an outcome.
- **Do not modify any existing files** in the source repository.
- **Do not add any other code** in the source repository besides the requested document.
- **Place the document** in the `blitzy/documentation/` directory in the destination repository.

### 0.7.2 Documentation Quality Requirements

- Every factual claim must cite at least one source file path (e.g., `dataprovider/sqlite.go`)
- Code snippets showing log lines, HTTP headers, and metrics output must use redacted placeholders for volatile fields (`<TIMESTAMP>`, `<REQID>`, `<PID>`)
- Answers must distinguish between deterministic behavior (e.g., "exit code is always 0 for this path") and nondeterministic behavior (e.g., "the SFTP and HTTP goroutines race; the HTTP server may or may not have bound its port before the SFTP error triggers shutdown")
- The document must be self-contained and readable without requiring access to the source code, while still providing precise file references for verification

### 0.7.3 Repository Integrity Constraints

- The `blitzy/documentation/` directory must be created if it does not exist
- No files outside `blitzy/documentation/sftpgo_44634210287c.md` may be created
- No git-tracked files may be modified, deleted, or renamed
- No temporary scripts, test files, or build artifacts may be committed


## 0.8 References


### 0.8.1 Repository Files and Folders Searched

The following files and directories were systematically explored to derive all conclusions in this Agent Action Plan. Files are grouped by the subsystem they inform.

**Root-level files**

| File | Purpose |
|---|---|
| `main.go` | Entry point; driver imports for side effects; delegates to `cmd.Execute()` |
| `sftpgo.json` | Default configuration file; SFTP port 2022, HTTP port 8080 on 127.0.0.1, SQLite driver, relative paths |
| `go.mod` | Module declaration (`github.com/drakkan/sftpgo`), Go 1.13, all dependency versions |
| `go.sum` | Dependency integrity checksums |

**`config/` — Configuration subsystem**

| File | Purpose |
|---|---|
| `config/config.go` | Viper-based config init, defaults in `init()`, `LoadConfig()` with multi-path search, `Unmarshal` into `globalConf` |
| `config/config_linux.go` | Adds `$HOME/.config/sftpgo` and `/etc/sftpgo` to Viper search on Linux |
| `config/config_nolinux.go` | No-op for non-Linux platforms |

**`cmd/` — CLI subsystem**

| File | Purpose |
|---|---|
| `cmd/root.go` | CLI flag definitions (`--config-dir`, `--config-file`, `--log-file-path`), defaults (`.`, `sftpgo`, `sftpgo.log`), Viper env bindings |
| `cmd/serve.go` | `serve` command: creates `service.Service`, calls `Start()`, blocks on `Wait()`; uses `Run` not `RunE` |

**`service/` — Service lifecycle**

| File | Purpose |
|---|---|
| `service/service.go` | `Start()`: logging init → config load (error discarded) → data provider init (fatal) → SFTP goroutine → HTTP goroutine; `Wait()` blocks on Shutdown channel |

**`dataprovider/` — Data provider subsystem**

| File | Purpose |
|---|---|
| `dataprovider/dataprovider.go` | Provider initialization dispatch, 30s availability timer, provider interface |
| `dataprovider/sqlite.go` | SQLite-specific init: file existence check (`os.Stat`), size validation (`fi.Size() == 0`), `sql.Open` with `cache=shared` |

**`httpd/` — HTTP server subsystem**

| File | Purpose |
|---|---|
| `httpd/httpd.go` | `Conf` struct, `Initialize(configDir)` with relative path resolution via `filepath.Join`, template loading, router init, `ListenAndServe` |
| `httpd/router.go` | Chi router: middleware chain (`RequestID → RealIP → StructuredLogger → Recoverer`), route map, NotFound/MethodNotAllowed handlers, static file server, Prometheus metrics endpoint |
| `httpd/web.go` | `loadTemplates()` with `template.Must` (panics on failure), web UI handlers |
| `httpd/api_utils.go` | `sendAPIResponse()` JSON format, `apiResponse` struct definition, `getRespStatus()` error-to-status mapping |
| `httpd/internal_test.go` | Test cases confirming error-to-status mapping and response checking |

**`sftpd/` — SFTP server subsystem**

| File | Purpose |
|---|---|
| `sftpd/server.go` | SSH server config, host key generation (`id_rsa` 4096-bit RSA), TCP listener binding, accept loop |
| `sftpd/sftpd.go` | Connection tracking map, active transfer list, idle timeout (5 min), supported SSH commands |

**`logger/` — Logging subsystem**

| File | Purpose |
|---|---|
| `logger/logger.go` | `InitLogger()`: lumberjack file writer (path relative to CWD) or stdout, zerolog configuration, timestamp format |
| `logger/request_logger.go` | `NewLogEntry()`: extracts `remote_addr` (from `r.RemoteAddr`, post-RealIP), `scheme` (from `r.TLS`), `uri`, `method`, `user_agent`, `request_id`; `Write()`: calls `metrics.HTTPRequestServed(status)`, logs info-level JSON |

**`metrics/` — Prometheus metrics**

| File | Purpose |
|---|---|
| `metrics/metrics.go` | Counter/gauge definitions via `promauto`; `HTTPRequestServed(status)` classification: 200–299→OK, 400–499→client, 500+→server, 3xx→total only |

**`utils/` — Utilities**

| File | Purpose |
|---|---|
| `utils/version.go` | `version = "0.9.5-dev"`, `VersionInfo` struct, `GetVersionAsString()` |

**`sql/sqlite/` — Database migration scripts**

| File | Purpose |
|---|---|
| `sql/sqlite/20190828.sql` | Initial `users` table schema |
| `sql/sqlite/20191112.sql` | Adds `expiration_date` column |
| `sql/sqlite/20191230.sql` | Adds `filters`, `filesystem` columns |
| `sql/sqlite/20200116.sql` | Adds `status`, `last_login` columns |

**`templates/` and `static/` directories**

| Directory | Purpose |
|---|---|
| `templates/` | HTML templates (`base.html`, `users.html`, `user.html`, `connections.html`, `message.html`) loaded by `loadTemplates()` |
| `static/` | CSS/JS assets served by `http.FileServer` at `/static` route |

### 0.8.2 External Research Conducted

| Topic | Source | Key Finding |
|---|---|---|
| chi v4.0.2 RealIP middleware | GitHub `go-chi/chi` master and tagged sources, issues #708 and #711 | v4 priority: X-Forwarded-For (leftmost IP, comma-split) → X-Real-IP; Forwarded (RFC 7239) and True-Client-IP not supported; True-Client-IP added in v5 |
| chi RealIP X-Forwarded-For parsing | GitHub `go-chi/chi` test file `realip_test.go` | Tests confirm both `"100.100.100.100, 200.200.200.200"` and `"100.100.100.100,200.200.200.200"` resolve to `"100.100.100.100"` (leftmost, with whitespace trimming) |
| chi RealIP header precedence | GitHub issue #708 | v5 reversed order to True-Client-IP → X-Real-IP → X-Forwarded-For; v4 uses X-Forwarded-For → X-Real-IP; confirms no RFC 7239 Forwarded support in any version |

### 0.8.3 Attachments

No attachments were provided for this project. No Figma URLs or external design assets are referenced.


