# Blitzy Project Guide — SFTPGo Command Injection Security Analysis

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive security analysis document for SFTPGo v0.9.5-dev (`github.com/drakkan/sftpgo`), a Go-based SFTP server. The analysis investigates a potential command injection vulnerability flagged by an inconsistent security scanner, tracing all four `os/exec` invocation pathways through the codebase, mapping their configuration-dependent reachability, and explaining why the scanner alternates between "vulnerable" and "not vulnerable." The sole deliverable is a 753-line markdown document (`blitzy/documentation/sftpgo_44634210287c.md`) containing evidence-based static analysis with source code references, a configuration-to-exploitability matrix, remediation recommendations, and logging behavior documentation. No source files in the repository were modified.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 82.4%
    "Completed (AI)" : 35
    "Remaining" : 7.5
```

| Metric | Value |
|---|---|
| **Total Project Hours** | 42.5 |
| **Completed Hours (AI)** | 35 |
| **Remaining Hours** | 7.5 |
| **Completion Percentage** | 82.4% |

**Calculation**: 35 completed hours / 42.5 total hours = 82.4% complete.

### 1.3 Key Accomplishments

- ✅ All 4 command execution pathways (`exec.Command` / `exec.CommandContext`) in the SFTPGo codebase fully traced from user-controlled entry point to OS-level execution
- ✅ Scanner inconsistency root cause identified: configuration-dependent reachability gates disable all exec paths under default settings
- ✅ 753-line security analysis document created with source code file paths and line numbers for every claim
- ✅ Configuration-to-exploitability matrix mapping 9 `sftpgo.json` keys to their attack surface effects
- ✅ 5 categories of remediation recommendations with specific actionable guidance
- ✅ Logging behavior documentation covering SSH commands, action hooks, authentication, and Fail2ban integration
- ✅ All code references verified against actual source (18 file:line references checked)
- ✅ Go build verification passed (`go build ./...` — exit code 0)
- ✅ No source files modified — single documentation artifact added to `blitzy/documentation/`
- ✅ 2 commits on feature branch: initial document creation + accuracy correction

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Document requires peer security review | Findings not yet validated by independent security expert | Human Developer | 2 hours |
| External program audit not performed | If action hooks or external auth scripts are in use, those scripts need independent review for shell quoting vulnerabilities | Human Developer | 2 hours |
| 9 pre-existing test failures in unchanged files | Environment-specific: root bypasses `os.Chmod` (3 tests), Go 1.13 SSH algo mismatch with modern OpenSSH (6 SCP tests) | Human Developer | 1 hour |

### 1.5 Access Issues

No access issues identified. The project deliverable is a documentation-only artifact that does not require service credentials, API keys, or external system access. All analysis was performed through static code reading of the repository source.

### 1.6 Recommended Next Steps

1. **[High]** Conduct peer security review of the analysis document (`blitzy/documentation/sftpgo_44634210287c.md`) by an independent security expert to validate all conclusions
2. **[High]** Audit production `sftpgo.json` configuration against the Configuration-to-Exploitability Matrix (Section 7 of the document) to confirm all exec paths are properly gated
3. **[Medium]** If external authentication or action hook programs are deployed, audit those scripts for shell quoting vulnerabilities as described in Remediation Recommendations
4. **[Medium]** Integrate the analysis findings into organizational security documentation and runbook procedures
5. **[Low]** Address pre-existing test environment issues (root permission bypass, OpenSSH algorithm compatibility) for CI/CD reliability

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| Repository Security Surface Discovery | 6 | Analysis of 54 Go source files across 12 packages; identification of 4 command execution pathways and 5 `os/exec` import sites; mapping of all user-controlled entry points to exec sinks |
| SSH System Commands Pathway Analysis | 4 | 11-step trace from SSH exec channel entry (`sftpd/server.go:327`) through command parsing, whitelist validation, routing, path resolution, permission enforcement, to `exec.Command` at `ssh_cmd.go:324` |
| SFTPD Action Hooks Pathway Analysis | 3 | Traced 5 trigger points (uploads, downloads, deletes, renames, SSH commands), 3 gate checks (`execute_on`, `command` length, `filepath.IsAbs`), env var construction, and risk assessment |
| External Authentication Pathway Analysis | 3 | Traced password and pubkey auth entries through `doExternalAuth`, documented credential passing via environment variables (`SFTPGO_AUTHD_PASSWORD`), identified as highest theoretical risk pathway |
| Provider Action Hooks Pathway Analysis | 3 | Traced REST API user CRUD operations through provider action hooks, documented unauthenticated API concern at `httpd/router.go`, argument/env var construction from `dataprovider/user.go` |
| Scanner Inconsistency Analysis | 2 | Determined root cause: configuration-dependent reachability gates on all 4 exec call sites; explained heuristic variation, configuration state dependency, and taint analysis limitations in scanners |
| Configuration-to-Exploitability Matrix | 2 | Mapped 9 `sftpgo.json` configuration keys to their security effects, documenting default (safe) and changed (risk) states for each |
| Remediation Recommendations | 2 | Developed 5 categories of actionable guidance: SSH command restriction, action hook script quoting, external auth credential handling, REST API access control, and principle of least privilege |
| Logging Behavior Documentation | 2 | Documented all log entry formats for SSH commands, action hooks, authentication (success/failure), Fail2ban integration patterns, and expected logs during attack attempts |
| Document Writing and Formatting | 4 | Authored 753-line markdown document with Mermaid data flow diagram, code blocks with syntax highlighting, configuration tables, and structured cross-references |
| Code Reference Verification | 2 | Verified every file:line reference in the document against actual source code (18 references checked); confirmed accuracy of all code snippets and configuration default values |
| Build and Test Validation | 1.5 | Executed `go build ./...` (passed), ran test suite across all packages, analyzed and documented 9 pre-existing environment-specific test failures in unchanged files |
| Document Accuracy Correction | 0.5 | Applied targeted fixes to code references after initial verification, committed as separate accuracy-fix commit |
| **Total** | **35** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Peer Security Review | 2 | High |
| Production Configuration Audit | 1.5 | High |
| External Program Script Audit | 2 | Medium |
| Security Documentation Integration | 1 | Medium |
| Test Environment Configuration | 1 | Low |
| **Total** | **7.5** | |

### 2.3 Hours Calculation

- **Completed Hours**: 35 (sum of Section 2.1 components)
- **Remaining Hours**: 7.5 (sum of Section 2.2 categories)
- **Total Project Hours**: 35 + 7.5 = 42.5
- **Completion Percentage**: 35 / 42.5 × 100 = **82.4%**

---

## 3. Test Results

All tests listed originate from Blitzy's autonomous validation execution during this project.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Unit — config package | `go test` | 5 | 5 | 0 | 100% | All configuration loading and validation tests pass |
| Unit/Integration — httpd package | `go test` | 2 (failed subset) | 0 | 2 | N/A | Pre-existing: `TestDumpdata`, `TestLoaddata` fail because `os.Chmod` restrictions are bypassed when running as root — tests expect HTTP 500 but get 200. Unchanged test files. |
| Unit/Integration — sftpd package | `go test` | 7 (failed subset) | 0 | 7 | N/A | Pre-existing: `TestOpenError` fails due to root bypassing filesystem permissions; 6 SCP tests fail due to Go 1.13 `crypto/ssh` algorithm mismatch with modern OpenSSH client. Unchanged test files. |
| Build Verification | `go build ./...` | 1 | 1 | 0 | N/A | Full project compilation succeeded (exit code 0). One non-fatal warning from vendored go-sqlite3 C code (pre-existing). |
| Document Reference Accuracy | Manual verification | 18 | 18 | 0 | 100% | All 18 file:line references in the security analysis document verified against actual source code |

**Key Observations**:
- The sole in-scope deliverable (`blitzy/documentation/sftpgo_44634210287c.md`) is a documentation artifact with no executable test coverage — all 18 code references were manually verified as an equivalent quality check.
- All 9 test failures are in **unchanged, out-of-scope** test files that the AAP explicitly prohibits modifying ("Don't modify any source files in the repository").
- Test failures are environment-specific: (a) running as root bypasses `os.Chmod` permission restrictions, (b) Go 1.13's `crypto/ssh` only offers `ssh-rsa` which modern OpenSSH clients reject by default.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ `go build ./...` — Full compilation of all 12 packages succeeded (exit code 0)
- ✅ Git working tree clean — no uncommitted in-scope changes
- ✅ Only 1 file added to repository (`blitzy/documentation/sftpgo_44634210287c.md`)
- ✅ No source files modified — confirmed via `git diff --name-status 44634210..HEAD`
- ⚠ Pre-existing non-fatal warning from vendored `go-sqlite3` C code during build (out-of-scope)

### Document Verification

- ✅ All 18 source code references (file paths and line numbers) verified accurate
- ✅ All `sftpgo.json` default values verified against actual configuration file
- ✅ Mermaid data flow diagram renders correctly (4 entry points → 4 validation gates → 4 execution sinks)
- ✅ Configuration-to-exploitability matrix covers all 9 security-relevant configuration keys
- ✅ All 4 command execution pathways fully traced with code evidence
- ✅ Document structure matches AAP specification (10 sections)

### API / Integration Verification

- ❌ Not applicable — This is a documentation-only deliverable. No server was started, no APIs were called, and no runtime behavior was tested. All analysis was performed through static code reading per AAP requirements.

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence | Notes |
|---|---|---|---|
| Security Code Analysis — SSH System Commands | ✅ Complete | Document lines 78–231: 11-step trace from `sftpd/server.go:327` to `ssh_cmd.go:324` | Covers parsing, whitelist, routing, path resolution, permissions, execution |
| Security Code Analysis — SFTPD Action Hooks | ✅ Complete | Document lines 232–311: 5 trigger points, gate checks, execution at `sftpd.go:421` | Covers all SFTP operations that trigger hooks |
| Security Code Analysis — External Authentication | ✅ Complete | Document lines 312–377: Credential env vars at `dataprovider.go:742–746` | Identified as highest theoretical risk pathway |
| Security Code Analysis — Provider Action Hooks | ✅ Complete | Document lines 378–474: REST API to `dataprovider.go:787` with unauthenticated API concern | Covers `httpd/router.go` no-auth middleware finding |
| Inconsistency Investigation | ✅ Complete | Document lines 41–73: Configuration-dependent reachability explanation | Directly answers "why scanner alternates" question |
| Vulnerability Condition Mapping | ✅ Complete | Document lines 475–494: 9-key configuration-to-exploitability matrix | Maps every `sftpgo.json` setting to attack surface effect |
| Deliverable Document Created | ✅ Complete | `blitzy/documentation/sftpgo_44634210287c.md` — 753 lines, committed | 2 commits: initial + accuracy fix |
| No Source Modification | ✅ Complete | `git diff --name-status 44634210..HEAD` shows only `A blitzy/documentation/sftpgo_44634210287c.md` | Zero source files touched |
| Evidence-Based Analysis | ✅ Complete | 18 file:line references, all verified against source | Per SWE-AtlasQnA-Repo: "base answers on the code as truth" |
| Rationale/Thinking Provided | ✅ Complete | Each pathway includes "Rationale" subsections explaining conclusions | Per SWE-AtlasQnA-Repo: "Provide thinking / rationale" |
| Active Exploitation Not Performed | ✅ Complete | Document lines 495–521: Explanation of why and equivalent diagnostic value provided | Ethical boundary maintained |
| Remediation Recommendations | ✅ Complete | Document lines 522–572: 5 categories of actionable guidance | SSH commands, hooks, external auth, API, least privilege |
| Logging Behavior Documentation | ✅ Complete | Document lines 573–692: All log formats documented | Includes Fail2ban integration and attack attempt log behavior |

**Autonomous Validation Fixes Applied**:
- Commit `ef7d56cf`: Corrected documentation accuracy in security analysis document after initial code reference verification revealed minor discrepancies

**Outstanding Compliance Items**:
- Peer security review required to validate analysis conclusions (human task)
- Production configuration should be audited against the configuration matrix (human task)

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Analysis conclusions not yet peer-reviewed by independent security expert | Technical | Medium | Medium | Schedule security peer review of the document before acting on findings | Open |
| External programs (action hook scripts, external auth programs) may contain shell quoting vulnerabilities | Security | High | Medium | Audit all external programs per Remediation Recommendation #2 and #3 in the document | Open — Requires human action |
| REST API at `httpd/router.go` has no authentication middleware | Security | High | Low (default bind is localhost) | Keep `httpd.bind_address` at `127.0.0.1`; if network access needed, add reverse proxy with auth | Documented in analysis |
| Pre-existing test failures may mask future regressions | Technical | Low | Medium | Fix test environment: run tests as non-root user, update OpenSSH client or configure SSH algorithms | Open |
| Go 1.13 is end-of-life; `crypto/ssh` may have known vulnerabilities | Security | Medium | Medium | Upgrade to supported Go version; update `golang.org/x/crypto` dependency | Open — Out of AAP scope |
| Configuration drift could enable exec paths unintentionally | Operational | Medium | Medium | Implement configuration management and monitoring for `sftpgo.json` security-relevant keys | Open |
| Document may become stale as SFTPGo codebase evolves | Operational | Low | High | Re-run analysis after major version upgrades; track changes to `os/exec` call sites | Open |

---

## 7. Visual Project Status

### Project Hours Breakdown

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 35
    "Remaining Work" : 7.5
```

### Remaining Work by Priority

| Priority | Hours | Categories |
|---|---|---|
| High | 3.5 | Peer Security Review (2h), Production Configuration Audit (1.5h) |
| Medium | 3 | External Program Script Audit (2h), Security Documentation Integration (1h) |
| Low | 1 | Test Environment Configuration (1h) |
| **Total** | **7.5** | |

---

## 8. Summary & Recommendations

### Achievements

The Blitzy autonomous agents successfully delivered a comprehensive 753-line security analysis document that fully addresses the user's request to investigate a command injection vulnerability in SFTPGo v0.9.5-dev. The project is **82.4% complete** (35 completed hours out of 42.5 total hours). All AAP-specified deliverables — the four command execution pathway analyses, scanner inconsistency explanation, configuration-to-exploitability matrix, remediation recommendations, and logging behavior documentation — were completed with source code evidence. Every code reference was verified for accuracy. No source files were modified, and the Go build passes cleanly.

### Key Finding Summary

The analysis determined that SFTPGo's default configuration is safe — all four `exec.Command`/`exec.CommandContext` call sites are gated behind disabled configuration settings. The scanner's inconsistency results from its inability to resolve configuration-dependent reachability deterministically. The primary risk is not in Go's exec mechanism (which bypasses shell interpretation) but in how external programs handle the arguments and environment variables they receive from SFTPGo.

### Remaining Gaps

7.5 hours of human-required work remain, primarily:
- **Peer security review** (2h) — An independent security expert should validate the analysis conclusions before they inform production decisions
- **Production configuration audit** (1.5h) — The deployed `sftpgo.json` should be checked against the configuration-to-exploitability matrix
- **External program audit** (2h) — If action hooks or external auth scripts are in use, they need independent review for shell quoting vulnerabilities

### Critical Path to Production

1. Complete peer security review of the document
2. Audit production configuration against the analysis matrix
3. If external programs are deployed, audit for shell quoting issues
4. Integrate findings into organizational security documentation

### Production Readiness Assessment

The security analysis document is **production-ready for review**. It provides the complete diagnostic information needed to resolve the scanner inconsistency and make informed decisions about SFTPGo's security posture. The document itself requires no further technical work — remaining hours are for human validation, organizational integration, and follow-up actions on the findings.

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|---|---|---|
| Go | 1.13+ | Build and test the SFTPGo project |
| Git | 2.x+ | Repository operations |
| SQLite3 | 3.x+ | Default database backend; required for test database initialization |
| GCC/C compiler | Any recent | Required for `go-sqlite3` CGO compilation |
| OpenSSH client | 7.x+ | Required for sftpd integration tests (SCP tests) |

### Environment Setup

```bash
# Clone the repository
git clone <repository-url>
cd sftpgo

# Ensure Go modules are enabled
export GO111MODULE=on

# Initialize the SQLite test database (required for tests)
sqlite3 sftpgo.db 'CREATE TABLE "users" ("id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "username" varchar(255) NOT NULL UNIQUE, "password" varchar(255) NULL, "public_keys" text NULL, "home_dir" varchar(255) NOT NULL, "uid" integer NOT NULL, "gid" integer NOT NULL, "max_sessions" integer NOT NULL, "quota_size" bigint NOT NULL, "quota_files" integer NOT NULL, "permissions" text NOT NULL, "used_quota_size" bigint NOT NULL, "used_quota_files" integer NOT NULL, "last_quota_update" bigint NOT NULL, "upload_bandwidth" integer NOT NULL, "download_bandwidth" integer NOT NULL, "expiration_date" bigint NOT NULL, "last_login" bigint NOT NULL, "status" integer NOT NULL, "filters" TEXT NULL, "filesystem" text NULL);'
```

### Dependency Installation

```bash
# Download all Go module dependencies
go get -v -t ./...
```

Expected output: Module downloads for ~20 direct dependencies including `golang.org/x/crypto`, `github.com/pkg/sftp`, `github.com/go-chi/chi`, etc.

### Building the Project

```bash
# Build all packages
go build ./...
```

Expected output: Clean compilation with exit code 0. One non-fatal CGO warning from `go-sqlite3` is expected and harmless.

### Running Tests

```bash
# Run all tests (note: some may fail if running as root or with modern OpenSSH)
go test -v ./... -coverprofile=coverage.txt -covermode=atomic

# Run only config package tests (most reliable)
go test -v ./config/

# Run with specific timeout
go test -v -timeout 300s ./...
```

**Known Test Environment Issues**:
- Running as root bypasses `os.Chmod` permission restrictions, causing `TestDumpdata`, `TestLoaddata`, and `TestOpenError` to fail
- Go 1.13's `crypto/ssh` only offers `ssh-rsa`, which modern OpenSSH clients reject — 6 SCP tests fail
- **Recommendation**: Run tests as a non-root user for accurate results

### Viewing the Security Analysis Document

```bash
# The deliverable document is located at:
cat blitzy/documentation/sftpgo_44634210287c.md

# Or view specific sections:
grep -n "^##" blitzy/documentation/sftpgo_44634210287c.md
```

### Running SFTPGo (for manual testing)

```bash
# Build the binary
go build -o sftpgo main.go

# Start with default configuration
./sftpgo serve

# Start in portable mode (single user, no database)
./sftpgo portable --username testuser --password testpass --directory /tmp/sftp_home
```

**Default ports**: SFTP on 2022, HTTP API on 8080 (localhost only).

### Troubleshooting

| Issue | Cause | Resolution |
|---|---|---|
| `cgo: C compiler not found` | Missing GCC for go-sqlite3 | Install `gcc` or `build-essential` |
| `cannot find package "github.com/..."` | Module cache empty | Run `go get -v -t ./...` |
| SCP tests fail with algorithm error | Go 1.13 crypto/ssh incompatibility | Update Go version or add `PubkeyAcceptedAlgorithms +ssh-rsa` to SSH config |
| Permission tests fail | Running as root | Run tests as non-root user |
| `sftpgo.db: no such table: users` | Test database not initialized | Run the `sqlite3 sftpgo.db 'CREATE TABLE...'` command from Environment Setup |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose | Context |
|---|---|---|
| `go build ./...` | Compile all packages | Verify build integrity |
| `go test -v ./...` | Run all tests with verbose output | Quality verification |
| `go test -v ./config/` | Run config package tests only | Reliable subset for quick validation |
| `sqlite3 sftpgo.db '<SQL>'` | Initialize/query test database | Test environment setup |
| `git diff --name-status 44634210..HEAD` | View files changed by Blitzy agents | Verify no source modifications |
| `git log --oneline 44634210..HEAD` | View Blitzy commits | Audit commit history |

### B. Port Reference

| Port | Service | Default Bind Address | Notes |
|---|---|---|---|
| 2022 | SFTP (SSH) server | `0.0.0.0` (all interfaces) | Configurable via `sftpd.bind_port` and `sftpd.bind_address` |
| 8080 | HTTP REST API and Web UI | `127.0.0.1` (localhost only) | Configurable via `httpd.bind_port` and `httpd.bind_address`. **Security-critical**: No authentication middleware on API routes |

### C. Key File Locations

| File | Purpose |
|---|---|
| `blitzy/documentation/sftpgo_44634210287c.md` | **Deliverable** — Security analysis document (753 lines) |
| `sftpgo.json` | Default runtime configuration — controls all security-relevant settings |
| `sftpd/ssh_cmd.go` | SSH command parsing and execution — `exec.Command` at line 324 |
| `sftpd/sftpd.go` | SFTPD action hook execution — `exec.CommandContext` at line 421 |
| `dataprovider/dataprovider.go` | External auth and provider hooks — `exec.CommandContext` at lines 742, 787 |
| `vfs/osfs.go` | Chroot path resolution — `ResolvePath()` at line 200 |
| `httpd/router.go` | HTTP API routing — no authentication middleware (lines 21–80) |
| `logger/logger.go` | Structured JSON logging — `CommandLog` at line 153, `ConnectionFailedLog` at line 175 |
| `fail2ban/filters/sftpgo.conf` | Fail2ban filter for authentication failures |

### D. Technology Versions

| Technology | Version | Source |
|---|---|---|
| Go | 1.13 (target) | `go.mod` line 3 |
| SFTPGo | 0.9.5-dev | `utils/version.go` line 3 |
| golang.org/x/crypto | v0.0.0-20200109152110 | `go.mod` |
| github.com/pkg/sftp | v1.11.0 | `go.mod` |
| github.com/go-chi/chi | v4.0.2+incompatible | `go.mod` |
| github.com/spf13/viper | v1.6.1 | `go.mod` |
| github.com/rs/zerolog | v1.17.2 | `go.mod` |
| github.com/mattn/go-sqlite3 | v2.0.2+incompatible | `go.mod` |
| github.com/aws/aws-sdk-go | v1.28.3 | `go.mod` |

### E. Environment Variable Reference

| Variable | Purpose | Set By |
|---|---|---|
| `GO111MODULE=on` | Enable Go modules | Developer (build-time) |
| `SFTPGO_ACTION` | Action hook operation type | SFTPGo → external command (`sftpd/sftpd.go:423`) |
| `SFTPGO_ACTION_USERNAME` | Action hook username | SFTPGo → external command (`sftpd/sftpd.go:424`) |
| `SFTPGO_ACTION_PATH` | Action hook file path | SFTPGo → external command (`sftpd/sftpd.go:425`) |
| `SFTPGO_ACTION_TARGET` | Action hook target path (renames) | SFTPGo → external command (`sftpd/sftpd.go:426`) |
| `SFTPGO_ACTION_SSH_CMD` | Action hook SSH command string | SFTPGo → external command (`sftpd/sftpd.go:427`) |
| `SFTPGO_ACTION_FILE_SIZE` | Action hook file size | SFTPGo → external command (`sftpd/sftpd.go:428`) |
| `SFTPGO_AUTHD_USERNAME` | External auth login username | SFTPGo → external auth program (`dataprovider/dataprovider.go:743`) |
| `SFTPGO_AUTHD_PASSWORD` | External auth login password (**sensitive**) | SFTPGo → external auth program (`dataprovider/dataprovider.go:744`) |
| `SFTPGO_AUTHD_PUBLIC_KEY` | External auth login public key | SFTPGo → external auth program (`dataprovider/dataprovider.go:745`) |
| `SFTPGO_USER_ACTION` | Provider hook action type | SFTPGo → provider hook (`dataprovider/user.go:452`) |
| `SFTPGO_USER_USERNAME` | Provider hook username | SFTPGo → provider hook (`dataprovider/user.go:453`) |
| `SFTPGO_USER_HOME_DIR` | Provider hook home directory | SFTPGo → provider hook (`dataprovider/user.go:457`) |

### G. Glossary

| Term | Definition |
|---|---|
| `exec.Command` | Go standard library function (`os/exec`) that creates a new process. Passes arguments as `argv` array directly to `execve()` — does NOT invoke a shell. |
| `exec.CommandContext` | Same as `exec.Command` but with a context for timeout/cancellation. |
| `execve()` | POSIX system call that replaces the current process with a new program. Arguments are passed as an array, not interpreted by a shell. |
| Chroot | A mechanism to restrict a process's view of the filesystem to a specific directory. SFTPGo implements virtual chroot via `ResolvePath()` in `vfs/osfs.go`. |
| Action Hook | An external command executed by SFTPGo after certain operations (upload, download, delete, rename, SSH command). Configured via `sftpd.actions` in `sftpgo.json`. |
| External Auth Program | An external program invoked by SFTPGo to authenticate users. Receives credentials via environment variables. Configured via `data_provider.external_auth_program`. |
| Shell Metacharacters | Characters with special meaning in shell interpreters: `;`, `|`, `&`, `$()`, `` ` ``, `{}`, `>`, `<`, `*`, `?`. These are NOT interpreted by Go's `exec.Command`. |
| Taint Analysis | A static analysis technique that tracks user-controlled data ("tainted" input) through code to determine if it reaches security-sensitive functions ("sinks"). |
| Configuration Gate | A runtime check in the code that only allows execution to proceed if a specific configuration setting is enabled. All 4 SFTPGo exec paths are gated. |