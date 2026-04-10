# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive, code-level security architecture analysis document for SFTPGo (v0.9.5-dev), a Go-based SFTP server. The deliverable is a single new Markdown file (`blitzy/documentation/sftpgo_44634210287c.md`, 927 lines) that answers five interrelated security questions about SFTPGo's internal behavior — covering external utility invocation, client/server decision boundaries, input validation, permission enforcement ordering, and filesystem protection mechanisms. Every technical claim in the document is grounded in source code evidence with precise file:line citations across 12 analyzed source files. No existing source files were modified per the project constraint.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 88.2%
    "Completed (AI)" : 30
    "Remaining" : 4
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 34 |
| **Completed Hours (AI)** | 30 |
| **Remaining Hours** | 4 |
| **Completion Percentage** | 88.2% |

**Calculation:** 30 completed hours / (30 completed + 4 remaining) = 30 / 34 = 88.2%

### 1.3 Key Accomplishments

- ✅ Created comprehensive 927-line security analysis document at `blitzy/documentation/sftpgo_44634210287c.md`
- ✅ Analyzed 12 source files across `sftpd/`, `dataprovider/`, and `vfs/` packages
- ✅ Documented all 5 security topics with dedicated sections and subsections (23 subsections total)
- ✅ Included 40+ source code citations with file:line references — all verified against actual code
- ✅ Created 4 Mermaid diagrams (sequence diagram, flowcharts) — all syntactically verified
- ✅ Verified 26 function signature citations match actual code exactly
- ✅ Maintained zero source file modifications — strict read-only analysis
- ✅ Compilation verified: `go build ./...` passes with zero errors
- ✅ Config tests verified: 5/5 PASS
- ✅ Clean working tree — no temporary files or test artifacts remaining
- ✅ Identified and documented critical security findings including string-prefix containment limitation

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Security findings require expert human review | Documented security concerns (e.g., string-prefix path traversal) need validation by a security engineer before acting on recommendations | Human Developer / Security Team | 2 hours |
| Mermaid diagram rendering not verified across all target platforms | 4 Mermaid diagrams may render differently in non-GitHub Markdown viewers | Human Developer | 0.5 hours |

### 1.5 Access Issues

No access issues identified. This is a documentation-only project requiring read access to the source repository, which was available throughout analysis.

### 1.6 Recommended Next Steps

1. **[High]** Security expert review — Have a security engineer validate the accuracy of all security findings documented in the analysis, particularly the string-prefix containment limitation in `vfs/osfs.go:285` and the permission enforcement ordering inconsistency
2. **[High]** Stakeholder sign-off — Product/engineering leadership should review the document before it is used to inform security decisions
3. **[Medium]** Mermaid rendering verification — Verify all 4 Mermaid diagrams render correctly in the target documentation platform (GitHub, VS Code, Confluence, etc.)
4. **[Low]** Post-review refinements — Apply any corrections or additional context from the expert review
5. **[Low]** Consider follow-up actions — Evaluate whether the documented security concerns (string-prefix limitation, unquoted env vars) warrant code changes in the upstream SFTPGo repository

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source Code Analysis and Deep Reading | 6 | Read and analyzed 12 Go source files across sftpd/, dataprovider/, vfs/, and config/ packages to trace security-relevant code paths, identify function signatures, and map call chains |
| Section 1 — External Utility Invocation Security | 4 | Documented command allowlist mechanism (sftpd.go), SSH exec payload processing (ssh_cmd.go:45-81), system command preparation and execution (ssh_cmd.go:288-331, 151-286), UID/GID credential wrapping (cmd_unix.go), and rsync symlink hardening (ssh_cmd.go:306-321) |
| Section 2 — Client vs. Server Decision Boundary | 4 | Documented session routing in AcceptInboundConnection() (server.go:243-333), command parsing via parseCommandPayload() (ssh_cmd.go:423-429), allowlist filtering, path transformation chain through getDestPath() (ssh_cmd.go:356-371), and post-operation action hooks (sftpd.go:438-492) |
| Section 3 — Input Validation and Protocol Parsing | 3 | Documented SSH exec payload parsing, SCP protocol message parsing via readProtocolMessage() (scp.go:542-563) and parseUploadMessage() (scp.go:631-663), SFTP request path handling, and edge cases with parsing assumptions table |
| Section 4 — Permission Enforcement Ordering | 3 | Traced and documented the ordering of ResolvePath() vs HasPerm() calls across all SFTP handlers (handler.go:47-233), SSH command handlers (ssh_cmd.go:104-286), and SCP handlers (scp.go:226-285), with comparative analysis table |
| Section 5 — Filesystem Protection Mechanisms | 4 | Documented ResolvePath() chroot gate (osfs.go:200-223), symlink evaluation via isSubDir() (osfs.go:278-291), non-existent path handling via findFirstExistingDir() (osfs.go:250-276), coverage across all access paths, and string-prefix containment limitations |
| Summary and Key Findings | 1 | Created critical security mechanisms table, areas of concern table with severity ratings, and question-to-answer reference map |
| Mermaid Diagram Creation | 2 | Created 4 diagrams: SSH exec request lifecycle flowchart, system command execution sequence diagram, path resolution chain flowchart, and session routing diagram |
| Citation Verification and Accuracy Checking | 2 | Verified 46 source citations against actual files and 26 function signature citations for exact match accuracy |
| Build and Test Validation | 1 | Ran go build ./... (success), go test ./config/ (5/5 PASS), verified no source files modified, confirmed clean working tree |
| **Total** | **30** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Security expert review and validation of findings | 2 | High |
| Mermaid diagram cross-platform rendering verification | 0.5 | Medium |
| Post-review refinements and corrections | 1.5 | Low |
| **Total** | **4** | |

### 2.3 Hour Calculation Verification

- **Section 2.1 Total (Completed):** 6 + 4 + 4 + 3 + 3 + 4 + 1 + 2 + 2 + 1 = **30 hours**
- **Section 2.2 Total (Remaining):** 2 + 0.5 + 1.5 = **4 hours**
- **Sum:** 30 + 4 = **34 hours** = Total Project Hours in Section 1.2 ✓
- **Completion:** 30 / 34 = **88.2%** ✓

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|--------------|-----------|-------------|--------|--------|-----------|-------|
| Unit — Config Package | `go test` | 5 | 5 | 0 | N/A | TestLoadConfigTest, TestEmptyBanner, TestInvalidUploadMode, TestInvalidExternalAuthScope, TestSetGetConfig |
| Compilation Verification | `go build` | 1 (all packages) | 1 | 0 | N/A | `go build ./...` succeeded; only warning from upstream go-sqlite3 C binding (out-of-scope dependency) |
| Citation Verification | Manual + scripted | 46 | 46 | 0 | 100% | All source file:line citations verified against actual code |
| Function Signature Verification | Manual + scripted | 26 | 26 | 0 | 100% | All function names and signatures match actual source code |
| Mermaid Diagram Syntax | Manual | 4 | 4 | 0 | 100% | sequenceDiagram and graph TD types verified syntactically correct |
| Source File Integrity | `git diff` | 1 | 1 | 0 | N/A | Confirmed zero existing source files modified via `git diff origin/sftpgo_44634210287c --name-status` |

**Note:** The `sftpd` and `httpd` packages contain integration tests that require a running SFTP server (port 2022), HTTP server (port 8080), and SQLite database. This is a **pre-existing infrastructure requirement** confirmed by testing the base commit (`44634210`) — the same failure occurs without any changes from this branch. These tests are unrelated to the documentation-only deliverable.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Go compilation** — `go build ./...` completes successfully with exit code 0
- ✅ **Config package tests** — 5/5 unit tests pass
- ✅ **Working tree** — Clean, no uncommitted changes or temporary files
- ✅ **Git status** — Branch `blitzy-48cdbeaa-1f7d-4b7e-8a6e-11d487862b96` is up to date with remote

### Documentation Verification

- ✅ **Document exists** — `blitzy/documentation/sftpgo_44634210287c.md` (927 lines)
- ✅ **All 5 security topics covered** — Sections 1–5 with 23 subsections total
- ✅ **40+ source citations** — All verified against actual source files
- ✅ **4 Mermaid diagrams** — Syntactically correct
- ✅ **Summary and Key Findings** — Critical mechanisms table, concerns table, Q&A map
- ✅ **No source modifications** — Confirmed via `git diff`
- ⚠ **Mermaid rendering** — Not yet verified across all target platforms (GitHub, VS Code, etc.)

### API / Integration Verification

Not applicable — this is a documentation-only project with no runtime API components or UI.

---

## 5. Compliance & Quality Review

| Compliance Criterion | Status | Evidence |
|---------------------|--------|----------|
| **No source file modifications** | ✅ Pass | `git diff origin/sftpgo_44634210287c --name-status` shows only `A blitzy/documentation/sftpgo_44634210287c.md` |
| **Evidence-based answers only** | ✅ Pass | 40+ `Source: file.go:LineNumber` citations; 26 verified function signatures |
| **All 5 security questions answered** | ✅ Pass | Sections 1–5 each address one question with dedicated subsections |
| **Code-level citations** | ✅ Pass | Every section contains multiple citations in `Source: path/to/file.go:LineNumber` format |
| **Mermaid diagrams included** | ✅ Pass | 4 diagrams: SSH exec lifecycle, system command sequence, path resolution chain, session routing |
| **Test script cleanup** | ✅ Pass | Working tree clean; no temporary artifacts |
| **No theoretical explanations** | ✅ Pass | Document follows pattern: summary answer → code evidence with line references → analysis |
| **Document placed correctly** | ✅ Pass | File at `blitzy/documentation/sftpgo_44634210287c.md` per AAP specification |
| **Compilation integrity** | ✅ Pass | `go build ./...` succeeds; no Go compilation errors |
| **Inferred documentation needs covered** | ✅ Pass | Rsync symlink hardening (§1.5), SCP experimental disclaimer (§3.2), external auth warning (§2.5), command argument stripping (§2.2), post-operation hooks (§2.5) |

### Quality Metrics

| Metric | Value |
|--------|-------|
| Document length | 927 lines |
| Major sections | 5 + Summary |
| Subsections | 23 |
| Source citations | 40+ |
| Function signature citations | 26 |
| Mermaid diagrams | 4 |
| Tables | 7+ |
| Source files analyzed | 12 |
| Go packages analyzed | 4 (sftpd, dataprovider, vfs, config) |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Security findings accuracy — the string-prefix containment limitation and other findings could be incorrect or incomplete without expert validation | Technical | Medium | Low | Human security expert review of all findings before acting on recommendations | Open |
| Mermaid diagram rendering — 4 diagrams may not render in all Markdown platforms | Technical | Low | Medium | Verify rendering in target platform (GitHub, VS Code, Confluence); provide text-based alternatives if needed | Open |
| Line number drift — source citations reference specific line numbers that may change with future code updates | Operational | Low | High | Document states version (0.9.5-dev); line numbers are accurate for the analyzed commit (44634210) | Accepted |
| SCP experimental status — document flags SCP as experimental; reader may misinterpret as a newly discovered issue | Operational | Low | Low | Document clearly cites the upstream developer's own comment at server.go:81-88 | Mitigated |
| No automated citation verification pipeline — citations were verified manually during this project but have no ongoing automated checks | Operational | Low | Medium | Consider adding a CI step that validates file:line citations if the document is maintained long-term | Open |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 30
    "Remaining Work" : 4
```

**Integrity Verification:**
- Completed Work: **30 hours** (matches Section 1.2 and Section 2.1 total)
- Remaining Work: **4 hours** (matches Section 1.2 and Section 2.2 total)
- Total: **34 hours** (30 + 4 = 34 ✓)
- Completion: **88.2%** (30 / 34 ✓)

### Remaining Work by Priority

| Priority | Hours | Percentage of Remaining |
|----------|-------|------------------------|
| High — Security expert review | 2 | 50% |
| Medium — Rendering verification | 0.5 | 12.5% |
| Low — Post-review refinements | 1.5 | 37.5% |
| **Total** | **4** | **100%** |

---

## 8. Summary & Recommendations

### Achievement Summary

This project successfully delivered a comprehensive, 927-line security architecture analysis document for SFTPGo. The document provides evidence-based answers to five critical security questions, grounded exclusively in source code analysis with 40+ precise file:line citations across 12 analyzed source files. The project is **88.2% complete** (30 of 34 total hours), with the remaining 4 hours consisting of human expert review, rendering verification, and post-review refinements.

### Key Findings Documented

The security analysis uncovered several significant findings that warrant attention:

1. **String-prefix containment limitation** (`vfs/osfs.go:285`) — The `strings.HasPrefix(sub, parent)` check can incorrectly allow access when directory names share prefixes (e.g., `/home/user` vs `/home/user2`).
2. **Permission enforcement ordering inconsistency** — `Fileread()` checks permissions before path resolution, while all other handlers resolve the path first. This creates a minor information-leak differential.
3. **SCP experimental status** — The SCP subsystem is explicitly labeled experimental by the developers with acknowledged edge cases.
4. **Unbounded SCP message length** — `readProtocolMessage()` has no maximum length check, creating a potential memory exhaustion vector.
5. **Unquoted environment variables** — Action hooks and external auth pass user-controlled data in unquoted environment variables.

### Critical Path to Production

The remaining 4 hours focus exclusively on human validation:
1. **Security expert review** (2h) — The most critical remaining task. A security engineer should validate all findings before they inform decisions.
2. **Rendering verification** (0.5h) — Ensure Mermaid diagrams display correctly in the target platform.
3. **Post-review corrections** (1.5h) — Apply any adjustments from the expert review.

### Production Readiness Assessment

The documentation deliverable is **complete and ready for review**. All AAP requirements have been fulfilled:
- All 5 security questions comprehensively answered with code evidence
- All diagrams and citations verified
- Zero source file modifications
- Clean repository state

The document should not be used to make security decisions until a human security expert has validated the findings.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Go | 1.13+ (repo target; 1.22.2 available in build environment) | Build and test the SFTPGo project |
| Git | 2.x+ | Version control, branch management |
| GCC/C compiler | Any recent version | Required for CGo / go-sqlite3 dependency |
| Markdown viewer | GitHub, VS Code, or compatible | View the documentation deliverable |

### Environment Setup

```bash
# Clone the repository and switch to the feature branch
git clone <repository-url>
cd sftpgo
git checkout blitzy-48cdbeaa-1f7d-4b7e-8a6e-11d487862b96
```

### Dependency Installation

```bash
# Download Go module dependencies
go mod download

# Verify dependencies
go mod verify
```

### Build Verification

```bash
# Compile all packages (expected: success with only upstream sqlite3 warning)
go build ./...

# Run config package tests (expected: 5/5 PASS)
go test ./config/ -v -count=1
```

**Expected output for tests:**
```
=== RUN   TestLoadConfigTest
--- PASS: TestLoadConfigTest (0.00s)
=== RUN   TestEmptyBanner
--- PASS: TestEmptyBanner (0.00s)
=== RUN   TestInvalidUploadMode
--- PASS: TestInvalidUploadMode (0.00s)
=== RUN   TestInvalidExternalAuthScope
--- PASS: TestInvalidExternalAuthScope (0.00s)
=== RUN   TestSetGetConfig
--- PASS: TestSetGetConfig (0.00s)
PASS
ok      github.com/drakkan/sftpgo/config    0.008s
```

### Viewing the Documentation

```bash
# View the documentation file
cat blitzy/documentation/sftpgo_44634210287c.md

# Or open in your preferred Markdown editor/viewer
# The file contains Mermaid diagrams that render in GitHub, VS Code (with extension), etc.
```

### Verification Steps

```bash
# 1. Verify the documentation file exists and has expected line count
wc -l blitzy/documentation/sftpgo_44634210287c.md
# Expected: 927 lines

# 2. Verify no source files were modified
git diff origin/sftpgo_44634210287c --name-status
# Expected: A    blitzy/documentation/sftpgo_44634210287c.md (only addition)

# 3. Verify compilation passes
go build ./...
# Expected: success (with only upstream sqlite3 C binding warning)

# 4. Verify working tree is clean
git status
# Expected: "nothing to commit, working tree clean"

# 5. Verify commit history
git log --oneline origin/sftpgo_44634210287c..HEAD
# Expected: 5bd0b0ff Add comprehensive SFTPGo security architecture analysis document
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| `go build` fails with missing GCC | Install `build-essential` (Debian/Ubuntu) or `gcc` (RHEL/CentOS) for CGo sqlite3 support |
| Mermaid diagrams don't render | Install VS Code "Markdown Preview Mermaid Support" extension, or view on GitHub which supports Mermaid natively |
| `go mod download` fails | Ensure Go 1.13+ is installed and `GOPATH`/`GOMODCACHE` are writable |
| sftpd/httpd tests fail | These are integration tests requiring a running SFTP+HTTP server environment — this is a pre-existing condition unrelated to documentation changes |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose | Expected Result |
|---------|---------|----------------|
| `go build ./...` | Compile all packages | Success (exit 0), only upstream sqlite3 warning |
| `go test ./config/ -v -count=1` | Run config unit tests | 5/5 PASS |
| `git diff origin/sftpgo_44634210287c --name-status` | Verify changes | Only `A blitzy/documentation/sftpgo_44634210287c.md` |
| `git log --oneline origin/sftpgo_44634210287c..HEAD` | View commit history | 1 commit: `5bd0b0ff` |
| `wc -l blitzy/documentation/sftpgo_44634210287c.md` | Verify document size | 927 lines |

### B. Port Reference

Not applicable — this is a documentation-only project. SFTPGo's default ports for reference:

| Service | Default Port | Configuration Key |
|---------|-------------|-------------------|
| SFTP Server | 2022 | `sftpd.bind_port` |
| HTTP API/Web UI | 8080 | `httpd.bind_port` |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/sftpgo_44634210287c.md` | **Primary deliverable** — Security architecture analysis document |
| `sftpd/ssh_cmd.go` | SSH command processing, system command execution (most-cited source) |
| `sftpd/server.go` | Session routing, authentication callbacks |
| `sftpd/handler.go` | SFTP request handlers, permission enforcement |
| `sftpd/scp.go` | SCP protocol parsing and handlers |
| `sftpd/sftpd.go` | Command registries, action hooks |
| `sftpd/cmd_unix.go` | UID/GID credential wrapping |
| `dataprovider/user.go` | Permission model, HasPerm(), GetPermissionsForPath() |
| `dataprovider/dataprovider.go` | Authentication flow, external auth |
| `vfs/osfs.go` | ResolvePath(), isSubDir(), chroot enforcement |
| `config/config.go` | Configuration loading and validation |
| `sftpgo.json` | Default runtime configuration |

### D. Technology Versions

| Technology | Version | Notes |
|------------|---------|-------|
| Go | 1.13 (module target) / 1.22.2 (build env) | Module declares Go 1.13; built with 1.22.2 |
| SFTPGo | 0.9.5-dev | Development version under analysis |
| pkg/sftp | v1.11.0 | SFTP protocol library |
| golang.org/x/crypto | v0.0.0-20200109 | SSH library |
| spf13/viper | v1.6.1 | Configuration management |
| spf13/cobra | v0.0.5 | CLI framework |
| go-sqlite3 | (from go.mod) | SQLite database driver |

### E. Environment Variable Reference

Not applicable for the documentation deliverable itself. SFTPGo environment variables documented in the analysis include:

| Variable | Context | Source |
|----------|---------|--------|
| `SFTPGO_ACTION_PATH` | Set by action hooks with user-controlled file path | `sftpd/sftpd.go:422-429` |
| `SFTPGO_ACTION_USERNAME` | Set by action hooks with username | `sftpd/sftpd.go:422-429` |
| `SFTPGO_ACTION_TARGET` | Set by action hooks with target path | `sftpd/sftpd.go:422-429` |
| `SFTPGO_AUTHD_*` | Set by external auth with user-controlled data (warning: unquoted) | `dataprovider/dataprovider.go:148-157` |

### G. Glossary

| Term | Definition |
|------|-----------|
| **Allowlist** | A list of explicitly permitted commands; only commands on this list can execute |
| **ResolvePath** | The chroot enforcement function in `vfs/osfs.go` that confines file access to a user's home directory |
| **HasPerm** | The permission check function in `dataprovider/user.go` that validates per-directory operation permissions |
| **isSubDir** | The string-prefix-based containment check in `vfs/osfs.go` that verifies a path is within the root directory |
| **wrapCmd** | The credential wrapping function in `sftpd/cmd_unix.go` that applies UID/GID to spawned processes |
| **SSH exec** | An SSH channel request type that sends a command for remote execution |
| **SCP** | Secure Copy Protocol — a file transfer protocol that runs over SSH |
| **SFTP** | SSH File Transfer Protocol — a subsystem of SSH for file management |
| **Chroot** | A security mechanism that restricts file system access to a designated root directory |
| **EvalSymlinks** | Go's `filepath.EvalSymlinks()` function that resolves all symbolic links in a path to their real targets |