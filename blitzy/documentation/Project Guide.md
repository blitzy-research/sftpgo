# Blitzy Project Guide — SFTPGo Clean-Start Behavior Investigation (Q&A Deliverable)

> **Brand color legend** — Completed / AI Work: **Dark Blue `#5B39F3`** · Remaining / Not Completed: **White `#FFFFFF`** · Headings / Accents: **Violet-Black `#B23AF2`** · Highlight: **Mint `#A8FDD9`**

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded technical answer document that explains **what SFTPGo actually does at runtime during a "clean start"** for an operator preparing to run SFTPGo behind a TLS-terminating reverse proxy. It empirically resolves a debate between a teammate's belief that a temporary config directory "isolates everything" (so the working directory should not matter) and an operator's suspicion that state/defaults leak in from elsewhere and "shift between runs." The scope is strictly **read-only documentation**: the entire SFTPGo source tree (Go 1.13, module `github.com/drakkan/sftpgo`) is treated as reference material and left byte-for-byte unchanged, while exactly one markdown document is authored from observed, verbatim runtime output with exact `file:line` citations.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2'}}}%%
pie showData title Project Completion — 92.0% Complete (46 of 50 hours)
    "Completed Work (AI)" : 46
    "Remaining Work" : 4
```

| Metric | Value |
|---|---|
| **Total Hours** | **50** |
| **Completed Hours (AI + Manual)** | **46** (46 AI + 0 Manual) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **92.0%** |

> Completion is computed on AAP-scoped work only: `46 / (46 + 4) = 92.0%`. The 4 remaining hours are pure human path-to-production (review, merge, hand-off); the AAP deliverable itself is authored, validated, and committed.

### 1.3 Key Accomplishments

- ✅ **Single deliverable authored & committed** — `blitzy/documentation/sftpgo_44634210287c.md` (958 lines, 128 `file:line` citations) at HEAD `e20fbebf`.
- ✅ **All 13 named question items answered with verbatim evidence** — A, B, C, D1–D3, E1–E3, F1–F4 (13/13 coverage checklist items).
- ✅ **Investigated by running the code first** — SFTPGo built with its own toolchain (Go 1.13.15 + CGO) and executed across every scenario; the answer is written from captured output, not from reading alone.
- ✅ **Both mental models resolved** — teammate's "temp dir isolates everything" **refuted** for the config file (CWD is a Viper search path); operator's "state shifts between runs" **confirmed** (CWD config fallback + first-run `id_rsa` auto-generation).
- ✅ **First-start failure asymmetry documented** — D1 (missing/empty DB) and D2 (taken SFTP port) exit `0` gracefully; D3 (missing templates) panics via `template.Must` (exit `2`).
- ✅ **Proxy-header story pinned down** — leftmost `X-Forwarded-For` logged, `X-Real-IP` fallback, RFC 7239 `Forwarded` ignored, URI scheme from `r.TLS` only, `sftpgo_http` counters keyed on response status only.
- ✅ **Read-only constraint honored** — source tree byte-for-byte unchanged (`git diff` excluding `blitzy/` is empty); temporary scripts removed; `git status` clean.
- ✅ **Independently re-validated this session** — `go build ./...` exit 0, `go vet` clean, and F1/F2/B reproduced verbatim.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None_ | No issue blocks release or validation. The deliverable passed all five Blitzy production-readiness gates and an independent re-validation with zero required fixes. | — | — |

> **No critical unresolved issues identified.** Noteworthy SFTPGo *findings* (proxy-header spoofability, D3 panic) are documented as findings, not defects of this deliverable, and are explicitly out of scope to remediate.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | — | — | — |

> **No access issues identified.** The repository is fully accessible (all `git` operations succeed); the required toolchain — Go 1.13.15, gcc 15.2.0, sqlite3 3.46.1 — is present; and no external service credentials or third-party API access is required (SFTPGo runs locally; no external integrations are in scope).

### 1.6 Recommended Next Steps

1. **[High]** Have a subject-matter expert **review and accept the answer document**, spot-checking a sample of the 128 `file:line` citations against the pinned commit and confirming it resolves the operator↔teammate debate. _(2h)_
2. **[Medium]** **Review and merge the pull request**, verifying the single-file scope and that the SFTPGo source is byte-for-byte unchanged. _(1h)_
3. **[Low]** **Hand the document to the operator** and, optionally, have them re-run the Reproduction appendix on their own host to self-verify the findings. _(1h)_
4. **[Low]** *(Out of scope — future effort)* Consider follow-up product hardening tickets derived from the documented findings: proxy-header handling, graceful template-load failure, and README updates for the CWD config fallback and proxy-header logging.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Toolchain build & reproduction harness | 4 | Built SFTPGo with the project toolchain (Go 1.13.15, CGO for the SQLite driver); set up isolated scratch dirs; created a "usable" DB via the four ordered SQLite migrations (`20190828`→`20200116`); dual-stack SFTP port-holder; backgrounded servers; cleanup. Maps to the AAP "run first" mandate. |
| A/B/C — Clean-start, state-shift & working-directory investigation | 7 | Traced the Viper config search order incl. CWD `.` [config/config.go:L147-149]; host-key `id_rsa` auto-generation [sftpd/server.go]; CWD config fallback; two contrasting runs (port 8090 vs 8080); before/after `ls`/`stat` evidence. Answers A, B, C. |
| D1/D2/D3 — First-start failure comparison | 5.5 | Three SQLite states (missing/empty/usable) [dataprovider/sqlite.go:L28-37]; dual-stack SFTP port-taken path [sftpd/server.go:L180]; `template.Must` panic [httpd/web.go:L95]; exit codes `0`/`0`/`2`; comparison table. Answers D1, D2, D3. |
| E1/E2/E3 — Proxy-header behavior investigation | 5 | go-chi `RealIP` precedence (XFF > X-Real-IP), leftmost IP, comma-no-space nuance, scheme from `r.TLS` only, `Forwarded` ignored, CWD-irrelevance [logger/request_logger.go:L36-46]. Answers E1, E2, E3. |
| F1/F2/F3/F4 — Required evidence capture | 6 | Verbatim startup logs; HTTP response headers for 4 paths; one proxy-header access-log line; `sftpgo_http` counter movement at scale, with vs without forwarded headers [metrics/metrics.go:L131-146,L219-229]. Answers F1, F2, F3, F4. |
| External research | 2 | Confirmed go-chi/chi v4.0.2 `RealIP` semantics from module-cache source and RFC 7239 `Forwarded` grammar, grounding the "ignored `Forwarded`" and "leftmost IP" findings. |
| Document authoring | 10 | Wrote the 958-line answer with 128 `file:line` citations under strict one-claim/one-evidence discipline: TL;DR, per-question sections, comparison tables, reproduction appendix, and coverage checklist. |
| Code-review revision cycle | 4 | Major revision (commit `0a4a977f`, +437/−142) addressing code-review findings on the initial draft. |
| Final QA correction, coverage pass & read-only verification | 2.5 | Final QA correction of exact `sftpgo.json` citations (commit `e20fbebf`, +5/−5); coverage pass over all 13 items; `git status` clean verification; temporary-script cleanup. |
| **Total** | **46** | **Sums to Completed Hours in Section 1.2** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Technical review & acceptance of the answer document (SME verifies claims/citations, confirms it resolves the operator↔teammate debate) | 2 | High |
| Pull-request review & merge (single-file scope check, source byte-for-byte unchanged, merge to target) | 1 | Medium |
| Operator hand-off & self-verification via the Reproduction appendix | 1 | Low |
| **Total** | **4** | **Matches Remaining Hours in Sections 1.2 & 7** |

> **Out-of-scope optional follow-ups (0h against this project, not counted):** proxy-header hardening (finding S1), graceful template-load failure via `recover()`/pre-flight check (finding O1), and documenting the CWD config fallback + proxy-header logging in SFTPGo's README. These are product changes to SFTPGo, which this read-only task is forbidden from making.

---

## 3. Test Results

All results below originate from Blitzy's autonomous validation logs for this project; the build/vet and empirical reproductions were **independently re-executed this session** and matched.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Static Analysis & Build | Go 1.13.15 (`go build`/`go vet`/`go mod verify`) | 3 | 3 | 0 | n/a | `go build ./...` exit 0 (only a benign upstream `mattn/go-sqlite3` CGO warning); `go vet` clean; modules verified. Re-run this session. |
| Unit — `config` | `go test` | 5 | 5 | 0 | n/a | `ok` (0.010s). Config discovery / search-order tests. |
| Unit — `httpd` (full suite) | `go test` | 72 | 72 | 0 | n/a | `ok` (4.413s) as non-root user. `TestDumpdata`/`TestLoaddata` are root-only Linux-DAC-bypass artifacts that pass as non-root. |
| Unit — `sftpd` (internal white-box) | `go test` | 38 | 38 | 0 | n/a | `ok` (0.822s); e.g. `TestSCP*`, `TestSSHCommand*`, `TestUploadFiles`. |
| Empirical Reproduction Cross-Checks | Blitzy shell / curl / sqlite3 harness | 31 | 31 | 0 | 13/13 items | Deterministic literal checks across A/B/C, D1–D3, E1–E3, F1–F4. F1/F2/B independently re-reproduced verbatim this session. |
| **Totals** | | **149** | **149** | **0** | **13/13 named items** | Zero genuine failures. |

> **Note on coverage:** because this is a documentation deliverable that adds **no product code**, there is no meaningful line-coverage metric; the relevant coverage measure is the **13/13 named-item** answer coverage plus the **128 citations** that all resolve to the claimed code.
>
> **Note on `TestDumpdata`/`TestLoaddata`:** these two `httpd` tests `os.Chmod(backupsPath, 0001)` and then expect a write to fail (HTTP 500); as root, Linux DAC is bypassed so the write succeeds (200) and the assertion fails. Run as the non-root user, both pass. The test code is byte-identical to upstream (pre-existing, unrelated to this deliverable, out of scope to modify).

---

## 4. Runtime Validation & UI Verification

Runtime behavior was validated by building the binary (reports `SFTPGo version: 0.9.5-dev`) and executing each scenario; results below were reproduced verbatim in this session.

**HTTP endpoints (F2)** — headers captured with no redirect-follow so the 301s stay visible:
- ✅ `GET /` → **301 Moved Permanently**, `Location: /web/users`, `Content-Length: 45`
- ✅ `GET /web` → **301 Moved Permanently**, `Location: /web/users`, `Content-Length: 45`
- ✅ `GET /metrics` → **200 OK**, `Content-Type: text/plain; version=0.0.4; charset=utf-8`, chunked
- ✅ `GET /does-not-exist` → **404 Not Found**, `Content-Type: application/json; charset=utf-8`, `Content-Length: 48`

**SFTP service** — ✅ listener registered `address: [::]:2022` on a clean start.

**Startup & state (F1 / B)** — ✅ full clean-start log sequence emitted; ✅ first run auto-generates `id_rsa` (mode `0600`, 3243 bytes) into the `-c` dir; ✅ second run reuses it (only `Loading private key`, no `creating new private key`).

**Proxy-header access logging (E/F3)** — ✅ all-three-headers request logs `remote_addr":"203.0.113.7"` (leftmost XFF) with URI scheme `http`; ✅ `X-Real-IP`-only → `9.9.9.9`; ✅ `Forwarded`-only → raw socket address (ignored); ✅ `X-Forwarded-Proto: https` ignored (scheme stays `http`).

**Metrics counters (F4)** — ✅ `sftpgo_http_*` counters move identically with and without forwarded headers (bucketed on response status only): observed progression `0/0/0/0 → 0/51/51/0 → 0/102/102/0 → 30/103/133/0 → 60/104/164/0` across the five scrapes.

**Failure modes (D)** — ✅ D1 missing/empty DB → clean **exit 0**; ✅ D2 SFTP port taken → clean **exit 0**; ✅ D3 missing templates → `template.Must` panic → **exit 2**.

**UI note:** No user-facing UI was built or modified by this task. The SFTPGo web UI (`/web/users`) exists in the product but is out of scope; only its HTTP response headers were captured as required evidence (F2). No UI screenshots apply.

---

## 5. Compliance & Quality Review

Cross-map of the governing `SWE-AtlasQnA-Repo` rule set (AAP §0.7) and key deliverable requirements to observed compliance.

| Requirement (AAP) | Benchmark | Status | Evidence / Notes |
|---|---|---|---|
| Single new markdown doc at fixed path & name | `blitzy/documentation/sftpgo_44634210287c.md` | ✅ Pass | 958-line file present; named for source branch `sftpgo_44634210287c`. |
| Investigate by RUNNING code first | Build → run → capture verbatim | ✅ Pass | Binary built (Go 1.13.15 + CGO); every scenario executed; output captured. Re-verified this session. |
| Quote observed output verbatim | Literal log lines, headers, exit codes | ✅ Pass | Verbatim startup logs, header slices, access-log lines, and `/metrics` scrapes embedded. |
| One claim, one piece of evidence | Evidence adjacent to each claim | ✅ Pass | 128 `file:line` citations; per-claim evidence pairing throughout. |
| Answer every named item + coverage pass | A, B, C, D1–D3, E1–E3, F1–F4 | ✅ Pass | 13/13 coverage checklist items marked `[x]`. |
| Be exact and grounded | Exact literals with `file:line` | ✅ Pass | Status codes (301/404/200), exit codes (0/2), counter names, and config keys quoted exactly. |
| Observe at representative magnitude (F4) | Sufficient scale to see deltas | ✅ Pass | Batches of 50×/50×/30×/60× requests; raw before/after counters reported. |
| Provide reasoning / rationale | "Why," not just "what" | ✅ Pass | Each section explains the mechanism behind the observation. |
| Read-only source scope | No source file modified | ✅ Pass | `git diff` excluding `blitzy/` is empty; `git status` clean; temp scripts removed. |
| Toolchain fidelity | Go 1.13.15 + CGO | ✅ Pass | Binary reports `0.9.5-dev`; matches cited code. |

**Fixes applied during autonomous validation:** code-review findings addressed in commit `0a4a977f` (+437/−142); exact `sftpgo.json` citation corrections in commit `e20fbebf` (+5/−5). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Run-specific evidence variances (timestamps, request IDs, ephemeral ports, keygen timing, resp_size, stack-trace path prefix) misread as errors by a re-runner | Technical | Low | Medium | Document explicitly characterizes each as non-behavioral / run-specific | Mitigated |
| Toolchain drift — behavior grounded in Go 1.13.15 may differ under a newer Go | Technical | Low | Low | Ground-truth section pins commit `44634210287c` + Go 1.13.15 + CGO | Mitigated |
| Point-in-time snapshot — answer may go stale if SFTPGo is upgraded past the pinned commit | Technical | Low | Medium | Exact commit and version pinned in the document | Documented |
| go-chi `RealIP` trusts client-supplied `X-Forwarded-For`/`X-Real-IP` (leftmost, spoofable); ignores RFC 7239 `Forwarded`; deprecated/CVE-flagged in chi v5 | Security | Medium | Medium | Reported as a finding; remediation explicitly out of scope (read-only task) | Documented (OOS) |
| Logged URI scheme always `http` behind a TLS-terminating proxy — could mislead a log auditor | Security | Low | Low | Explained in E1 (scheme derives from `r.TLS`, never headers) | Documented (OOS) |
| D3 missing-templates → unrecovered `template.Must` panic (exit 2) crashes the whole process | Operational | Medium | Low | Reported as a finding with exit-code evidence; remediation out of scope | Documented (OOS) |
| E2 comma-no-space `X-Forwarded-For` trap — go-chi splits only on exact `", "` | Operational | Low | Medium | Documented with a verbatim proof line and operator warning | Documented (OOS) |
| Single-file deliverable must land at the correct destination path/name | Integration | Low | Low | Correctly placed, named, and committed at HEAD `e20fbebf` | Resolved |
| Reproduction appendix depends on curl/python3/sqlite3/gcc/Go 1.13.15 for reader self-verification | Integration | Low | Low | Section 9 dev guide lists all prerequisites | Mitigated |

**Deliverable-level risk posture:** No risk blocks release. All *Medium*-severity items are **findings about SFTPGo** that the task is required to document rather than fix.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 50h)
    "Completed Work" : 46
    "Remaining Work" : 4
```

**Remaining work by task (hours)** — from Section 2.2 (sums to 4h):

```mermaid
xychart-beta
    title "Remaining Hours by Task"
    x-axis ["Tech Review [H]", "PR Merge [M]", "Operator Hand-off [L]"]
    y-axis "Hours" 0 --> 3
    bar [2, 1, 1]
```

> **Integrity check:** "Remaining Work" = **4h** matches Section 1.2 (Remaining Hours = 4) and the Section 2.2 total (2 + 1 + 1 = 4). "Completed Work" = **46h** matches Section 1.2 and the Section 2.1 total. `46 + 4 = 50` = Total Project Hours.

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.0% complete** (46 of 50 hours). The AAP deliverable — a single, evidence-grounded answer document — is authored, code-reviewed, QA-corrected, committed (HEAD `e20fbebf`), and independently re-validated. It answers all 13 named question items (A, B, C, D1–D3, E1–E3, F1–F4) with 128 `file:line` citations and verbatim runtime output, and it decisively resolves the operator↔teammate debate: **the temp `-c` directory does not isolate the config file** (the CWD is a Viper search path), **but it does isolate the DB, templates, and static assets**; and **state genuinely shifts between runs** via the CWD config fallback and first-run `id_rsa` host-key generation.

**Remaining gaps (4 hours, all human).** The only outstanding work is path-to-production for a documentation artifact: SME technical review and acceptance (2h), pull-request review and merge (1h), and operator hand-off / self-verification (1h). There is **no remaining engineering or code work** within the AAP scope.

**Critical path to production.** Review → merge → hand-off. Because the source tree is unchanged and the deliverable passed all validation gates, the path is short and low-risk.

**Success metrics.**

| Metric | Target | Actual |
|---|---|---|
| Named items answered | 13/13 | ✅ 13/13 |
| `file:line` citations resolving | 100% | ✅ 128/128 |
| Source files modified | 0 | ✅ 0 (`git diff` empty) |
| Validation gates passed | 5/5 | ✅ 5/5 (+ independent re-run) |
| Genuine test failures | 0 | ✅ 0 |

**Production readiness.** The deliverable is **ready for human review and merge**. It is accurate, complete, evidence-grounded, and non-intrusive. Recommended follow-ups (proxy-header hardening, graceful template-load failure, README updates) are optional, out-of-scope product improvements to be tracked separately.

---

## 9. Development Guide

This guide explains how to build SFTPGo at the pinned commit, view the deliverable, and re-run the investigation to self-verify the answer. **Every command below was executed successfully during validation.**

### 9.1 System Prerequisites

- **OS:** Linux x86_64 (validated on Ubuntu 25.10).
- **Go:** **1.13.15** (the highest explicitly documented supported toolchain; `go 1.13` in `go.mod`). Available at `/usr/local/go`.
- **C compiler:** **gcc** (required by the CGO-based `mattn/go-sqlite3` driver). Validated with gcc 15.2.0.
- **SQLite CLI:** **sqlite3** (used to hand-initialize a valid DB schema for the "usable DB" scenario). Validated with 3.46.1.
- **Utilities:** `git`, `curl`, `python3` (for the dual-stack SFTP port-holder in D2).

### 9.2 Environment Setup

```bash
# Put Go 1.13.15 on PATH and enable CGO (needed for the SQLite driver)
export PATH=$PATH:/usr/local/go/bin
export CGO_ENABLED=1

# Verify the toolchain
go version          # -> go version go1.13.15 linux/amd64
gcc --version       # -> gcc (Ubuntu 15.2.0-...) 15.2.0
sqlite3 --version   # -> 3.46.1 ...
```

### 9.3 Dependency Installation & Build

Dependencies are pinned in `go.mod`/`go.sum`; the module cache resolves them on first build.

```bash
# From the repository root
cd /tmp/blitzy/sftpgo/blitzy-a09432f2-d916-478f-84d7-c3baf9198075_10a8a6

# Build the binary (README-style ldflags embed commit + date)
go build -ldflags "-s -w \
  -X github.com/drakkan/sftpgo/utils.commit=$(git describe --always --dirty) \
  -X github.com/drakkan/sftpgo/utils.date=$(date -u +%FT%TZ)" \
  -o /tmp/sftpgo .
# Exit 0. A single benign upstream 'mattn/go-sqlite3' CGO warning is expected.

# Simpler form (no ldflags):
CGO_ENABLED=1 go build -o /tmp/sftpgo .

# Confirm the version
/tmp/sftpgo --version   # -> SFTPGo version: 0.9.5-dev-<commit>-<date>
```

### 9.4 View the Deliverable

```bash
# The single answer document (958 lines)
sed -n '1,60p' blitzy/documentation/sftpgo_44634210287c.md
wc -l           blitzy/documentation/sftpgo_44634210287c.md   # -> 958
```

### 9.5 Application Startup & Verification (self-verify the answer)

```bash
# 1) Build a "usable" SQLite DB via the four ordered migrations
SRC=/tmp/blitzy/sftpgo/blitzy-a09432f2-d916-478f-84d7-c3baf9198075_10a8a6
mkdir -p /tmp/run/confdir /tmp/run/neutral
for m in 20190828 20191112 20191230 20200116; do
  sqlite3 /tmp/run/confdir/sftpgo.db < "$SRC/sql/sqlite/$m.sql"
done
cp -r "$SRC/templates" /tmp/run/confdir/templates
cp -r "$SRC/static"    /tmp/run/confdir/static
sqlite3 /tmp/run/confdir/sftpgo.db '.tables'   # -> users

# 2) Start from a NEUTRAL working directory; log to stdout via -l ""
cd /tmp/run/neutral
/tmp/sftpgo serve -c /tmp/run/confdir -l "" > /tmp/run/srv.log 2>&1 &
SRV_PID=$!
until grep -q "server listener registered" /tmp/run/srv.log; do sleep 0.3; done

# 3) Verify HTTP endpoints (F2)
for p in / /web /metrics /does-not-exist; do
  curl -s -o /dev/null -w "GET $p -> %{http_code}\n" "http://127.0.0.1:8080$p"
done
# Expected: / -> 301, /web -> 301, /metrics -> 200, /does-not-exist -> 404

# 4) Verify proxy-header logging (E/F3)
curl -s -o /dev/null \
  -H "X-Forwarded-For: 203.0.113.7, 70.41.3.18" \
  -H "X-Real-IP: 9.9.9.9" \
  "http://127.0.0.1:8080/metrics?probe=demo"
grep 'probe=demo' /tmp/run/srv.log
# Expected: "remote_addr":"203.0.113.7" (leftmost XFF), uri scheme "http" (from r.TLS)

# 5) Stop and clean up
kill "$SRV_PID"; wait "$SRV_PID" 2>/dev/null
rm -rf /tmp/run /tmp/sftpgo
```

### 9.6 Example Usage — reproducing the failure modes (D)

```bash
# D1 missing DB: empty -c dir -> descriptive log, clean exit 0
mkdir -p /tmp/d1_missing
/tmp/sftpgo serve -c /tmp/d1_missing -l ""; echo "exit=$?"   # exit 0

# D1 empty DB: 0-byte file -> "database file is invalid", exit 0
mkdir -p /tmp/d1_empty; : > /tmp/d1_empty/sftpgo.db
/tmp/sftpgo serve -c /tmp/d1_empty -l ""; echo "exit=$?"     # exit 0

# D3 missing templates: usable DB + static/, but NO templates/ -> panic, exit 2
/tmp/sftpgo serve -c /tmp/d3_notemplates -l ""; echo "exit=$?"   # exit 2 (template.Must panic)
```

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `go: command not found` | Go not on PATH | `export PATH=$PATH:/usr/local/go/bin` |
| CGO / `gcc` build error on `go-sqlite3` | CGO disabled or gcc missing | `export CGO_ENABLED=1`; ensure `gcc` is installed |
| Server logs "Config File \"sftpgo\" Not Found" then uses defaults | No `sftpgo.json` in `-c` dir or CWD (expected in a neutral CWD) | Intentional in the clean-start scenario; provide a `sftpgo.json` if custom settings are desired |
| Process exits immediately with a descriptive error | Missing/empty SQLite DB (D1) or taken SFTP port :2022 (D2) | Initialize the DB via migrations; free port 2022 |
| Process crashes with `panic: open .../templates/base.html` (exit 2) | `templates/` directory absent (D3) | Copy `templates/` into the `-c` directory |
| `sqlite database file does not exists` | DB not initialized | Run the four ordered migrations from `sql/sqlite/` |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Verify Go toolchain | `go version` |
| Build binary | `CGO_ENABLED=1 go build -o /tmp/sftpgo .` |
| Show version | `/tmp/sftpgo --version` |
| Compile all packages | `go build ./...` |
| Static vet (cited pkgs) | `go vet ./config/ ./logger/ ./metrics/ ./httpd/` |
| Verify modules | `go mod verify` |
| Initialize usable DB | `for m in 20190828 20191112 20191230 20200116; do sqlite3 CONF/sftpgo.db < sql/sqlite/$m.sql; done` |
| Start server (stdout logs) | `/tmp/sftpgo serve -c CONF -l ""` |
| Header slice for a path | `curl -s -D - -o /dev/null http://127.0.0.1:8080/PATH` |
| Scrape HTTP metrics | `curl -s http://127.0.0.1:8080/metrics \| grep '^sftpgo_http_'` |
| Confirm source pristine | `git status --porcelain` (empty = clean) |

### B. Port Reference

| Port | Service | Source | Notes |
|---|---|---|---|
| **2022** | SFTP server | `sftpgo.json` `sftpd.bind_port` | Binds `[::]:2022` (dual-stack); D2 tests the "already taken" path |
| **8080** | HTTP/REST + `/web` + `/metrics` | `sftpgo.json` `httpd.bind_port` (default) | `bind_address` `127.0.0.1`; used for F2/F4 evidence |
| **8090** | HTTP (CWD-config demo) | CWD `sftpgo.json` in the C-scenario | Demonstrates the CWD config fallback (port 8090 vs default 8080) |
| 5432 | (config field only) | `sftpgo.json` `data_provider.port` | Default PostgreSQL port; unused with the SQLite driver |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/sftpgo_44634210287c.md` | **The deliverable** (958 lines, 128 citations) |
| `config/config.go` | Config search order incl. CWD `.` [L147-149]; not-found warning [L152-154]; "config file used" log [L180] |
| `config/config_linux.go` | Platform search paths `$HOME/.config/sftpgo`, `/etc/sftpgo` [L9-10] |
| `cmd/root.go`, `cmd/serve.go` | Flags (`-c`/`-l`/`-f`/`-v`), env bindings; process exit path |
| `service/service.go` | Startup sequence & goroutine orchestration |
| `dataprovider/sqlite.go` | D1 SQLite state checks [L28-37] |
| `sftpd/server.go` | SFTP listener error [L180]; host-key auto-gen |
| `httpd/httpd.go`, `httpd/router.go`, `httpd/web.go` | Asset paths; middleware & routes; `template.Must` panic [L95] |
| `logger/request_logger.go` | Access-log fields, scheme from `r.TLS` [L36-46] |
| `metrics/metrics.go` | `sftpgo_http` counters [L131-146]; status bucketing [L219-229] |
| `sftpgo.json` | Repo-root config (SFTP 2022, HTTP 8080, SQLite `sftpgo.db`) |
| `sql/sqlite/*.sql` | Ordered migrations to build a usable DB |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Go toolchain | 1.13.15 | `go.mod` (`go 1.13`); CI `1.13.x` |
| SFTPGo | 0.9.5-dev | Binary `--version`; commit `44634210287c` base |
| gcc | 15.2.0 | Host toolchain (CGO for SQLite) |
| sqlite3 CLI | 3.46.1 | Host toolchain |
| github.com/go-chi/chi | v4.0.2+incompatible | `go.mod` — `RealIP` middleware |
| github.com/spf13/viper | v1.6.1 | `go.mod` — config search incl. CWD |
| github.com/spf13/cobra | v0.0.5 | `go.mod` — `serve` lifecycle |
| github.com/rs/zerolog | v1.17.2 | `go.mod` — structured JSON logging |
| github.com/prometheus/client_golang | v1.3.0 | `go.mod` — `/metrics` |
| github.com/mattn/go-sqlite3 | v2.0.2+incompatible | `go.mod` — CGO SQLite driver |

### E. Environment Variable Reference

| Variable | Purpose | Default / Example |
|---|---|---|
| `CGO_ENABLED` | Enable CGO for the SQLite driver (required to build with SQLite) | `1` |
| `PATH` | Must include the Go 1.13.15 bin dir | `$PATH:/usr/local/go/bin` |
| `SFTPGO_CONFIG_DIR` | Overrides the `-c`/`--config-dir` flag | `.` (default; flag `-c`) |
| `SFTPGO_LOG_FILE_PATH` | Overrides the `-l`/`--log-file-path` flag; empty = stdout | `""` |
| `SFTPGO_*` (prefix, `__` separator) | Viper env binding for any config key | e.g. `SFTPGO_HTTPD__BIND_PORT=8090` |

> CLI flags: `-c/--config-dir` (default `.`), `-l/--log-file-path` (empty ⇒ stdout), `-f/--config-file` (default base name `sftpgo`), `-v/--version`.

### F. Developer Tools Guide

| Tool | Use in this project |
|---|---|
| `go build` / `go vet` / `go mod verify` | Compile & statically check the pinned commit (Gate 1) |
| `go test ./config/ ./httpd/ ./sftpd/` | Run cited-package unit tests (run as non-root) |
| `curl -D -` | Capture HTTP response headers for F2 |
| `curl ... -H "X-Forwarded-For: ..."` | Drive proxy-header access-log probes for E/F3 |
| `sqlite3` | Initialize a usable DB (D1) from `sql/sqlite/*.sql` |
| `python3` (socket) | Dual-stack `[::]:2022` port-holder for D2 |
| `git diff` / `git status` | Prove the source tree is byte-for-byte unchanged |

### G. Glossary

| Term | Definition |
|---|---|
| **AAP** | Agent Action Plan — the primary directive defining project scope (here: a read-only Q&A investigation). |
| **`-c` / config dir** | Directory passed via `--config-dir`; relative state (DB, templates, static) resolves under it. |
| **CWD fallback** | Viper's last config search path is `.` (the current working directory), so a `sftpgo.json` there is loaded when `-c` has none. |
| **`RealIP`** | go-chi middleware that overwrites `r.RemoteAddr` from `X-Forwarded-For` (preferred) or `X-Real-IP`, leftmost value; ignores RFC 7239 `Forwarded`. |
| **`template.Must`** | Go helper that panics if template parsing fails — the cause of the D3 exit-2 crash when `templates/` is absent. |
| **`sftpgo_http_*`** | Prometheus counters at `/metrics`, bucketed by response status (total / ok / client_errors / server_errors). |
| **Clean start** | Launching SFTPGo with a fresh/temporary config directory to observe first-run behavior. |
| **Path-to-production** | Standard human activities (review, merge, hand-off) to move the validated deliverable into use. |

---

*Prepared by the Blitzy Platform. Completion (92.0%) reflects AAP-scoped work only. Brand colors applied: Completed `#5B39F3`, Remaining `#FFFFFF`.*