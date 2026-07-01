# Blitzy Project Guide — SFTPGo Contention & Quota Investigation (Evidence-Grounded QnA)

> **Brand color key** — <span style="color:#5B39F3">**Completed / AI Work = Dark Blue `#5B39F3`**</span> · **Remaining / Not Completed = White `#FFFFFF`** · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded Markdown answer document that explains — from **observed runtime behavior**, not code reading alone — how **SFTPGo** (`drakkan/sftpgo`, a full-featured Go SFTP server; commit `44634210`, build `0.9.5-dev`) coordinates **connection handling, quota enforcement, atomic uploads, and the periodic idle checker** when a strict-quota user opens multiple sessions near `max_sessions` and launches concurrent uploads that collectively exceed both `quota_files` and `quota_size`. It is a **read-only investigative QnA** task: the server is exercised and its real output captured, then six sub-questions (Q1–Q6) are answered with verbatim evidence and exact `file:line` citations. The audience is engineers onboarding to SFTPGo's concurrency and quota internals.

### 1.2 Completion Status

The project is **93.3% complete** on an AAP-scoped basis. All 14 investigation/deliverable requirements are complete and validated; the remaining 4 hours are human path-to-production activities (SME review, reproduction, PR merge).

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#5B39F3','pieStrokeWidth':'2px','pieOuterStrokeColor':'#5B39F3','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieTitleTextColor':'#5B39F3','pieLegendTextColor':'#333333'}}}%%
pie showData title AAP-Scoped Completion — 93.3% Complete
    "Completed Work (AI)" : 56
    "Remaining Work" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **60** |
| **Completed Hours (AI + Manual)** | **56** (AI = 56, Manual = 0) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **93.3%** |

> Completion % = Completed ÷ Total = 56 ÷ 60 = **93.3%** (AAP-scoped work only, per PA1).

### 1.3 Key Accomplishments

- ✅ **Single deliverable created and committed** — `blitzy/documentation/sftpgo_44634210287c.md` (841 lines, 71,477 bytes), the only repository change.
- ✅ **All six sub-questions (Q1–Q6) answered explicitly**, each under a dedicated heading, plus a final coverage pass listing every asked-for value as an exact literal.
- ✅ **Investigated by running first** — SFTPGo built (Go 1.13.15 + CGO) and run live; concurrent `pkg/sftp` uploads driven, connections force-closed mid-stream, quota inspected in SQLite.
- ✅ **Verbatim evidence captured** — JSON logs, HTTP responses, `sqlite3` quota readings, filesystem listings of `.sftpgo-upload.<xid>.<name>` temp files, and a `-race` run (0 data races).
- ✅ **124 exact `file:line` citations** across 14 source files; independently spot-checked (6/6 matched) and fully audited by autonomous validation (zero discrepancies).
- ✅ **Read-only mandate honored** — `git diff 44634210..HEAD` adds exactly one file; no source/test/config edits; `git status --porcelain` empty; all runtime artifacts confined to `/tmp` and removed.
- ✅ **Build independently re-verified** in this assessment — `SFTPGo version: 0.9.5-dev`, **31,882,120 bytes** (byte-for-byte match to the document).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None blocking._ Deliverable is complete, validated, and committed; repository is clean. | No release blocker | — | — |
| Human SME technical sign-off pending (routine acceptance gate, not a defect) | Formal acceptance of the answer document | SME / Reviewer | ≤ 2h |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| — | — | **No access issues identified.** Build toolchain (Go 1.13.15 + gcc), full module cache (offline builds), and repository access were all available; the REST control plane and SFTP subsystem responded during autonomous validation. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform SME technical review of `sftpgo_44634210287c.md` — verify Q1–Q6 claims (session gate, quota TOCTOU + SQLite serialization, atomic-abort accounting) and spot-check a sample of the 124 citations (~2h).
2. **[Medium]** Independently reproduce 2–3 headline experiments using the Development Guide (Section 9) to confirm the verbatim evidence (~1.5h).
3. **[Medium]** Approve and merge the PR to the target branch (~0.5h).
4. **[Low]** _(Out of scope, informational)_ If the documented `hasSpace` TOCTOU quota-overshoot is deemed a product defect, schedule a **separate** remediation initiative — it is explicitly out of scope for this read-only QnA task.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work maps to AAP requirements R1–R14 (investigation methodology + the answer document). **Total = 56 hours** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|-----------|------:|-------------|
| Codebase investigation & code-path mapping | 8 | Read `sftpd`, `dataprovider`, `vfs`, `httpd`, `logger`, `metrics`; located exact `file:line` evidence for Q1–Q6 (session gate, `hasSpace`, `Transfer.Close()`, idle ticker, SQLite serialization). |
| Build & runtime foundation | 4 | CGO build with Go 1.13.15 → `SFTPGo version: 0.9.5-dev`; isolated `/tmp` workspace (config/home/db/logs); created SQLite `users` schema. |
| Observation harness | 8 | Concurrent `pkg/sftp` client + shell glue: staggered/concurrent sessions, quota-filling uploads, mid-stream force-close via REST, 30-way burst. |
| Q1/Q4 evidence capture | 4 | Session-limit rejection (`2/2`) and per-step quota fill/denial (`0|0 → 1|2048 → 2|4096 → denied`). |
| Q2/Q6 evidence capture | 6 | TOCTOU overshoot (`0|0 → 4|8192`), 30-way no-lost-updates (`30|122880`), `-race` build/run, login-before-register window, genuine ~5-min idle close. |
| Q3/Q5 evidence capture | 5 | Atomic mode-1 mid-stream abort (temp deleted, quota unchanged), clean completion (`31|126976`), mode-2 resume (temp kept), mode-0 standard (partial lingers). |
| Answer document authoring | 12 | Q1–Q6 prose, coordination Mermaid diagram, 5 tables, evidence appendix (4.1–4.12), final coverage pass, and 124 `file:line` citations. |
| Web-search corroboration research | 2 | Go race-detector best practices, concurrent `pkg/sftp` usage, SFTPGo config semantics (confirmatory; code is source of truth). |
| Review & fix cycles | 6 | 3 documentation commits (initial + 3 review findings + 2 verbatim-literal fixes) plus the final autonomous validation reproduction pass. |
| Cleanup & read-only verification | 1 | Tore down `/tmp` workspace; confirmed `git status --porcelain` empty and no artifacts leaked into the repository. |
| **Total** | **56** | **Matches §1.2 Completed Hours.** |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production (AAP requirements R15–R16). **Total = 4 hours** (matches Remaining Hours in §1.2 and the Section 7 pie chart).

| Category | Hours | Priority |
|----------|------:|----------|
| SME technical review of the answer document (verify Q1–Q6 claims, spot-check citations, confirm coverage) | 2.0 | High |
| Independent evidence reproduction (build SFTPGo, run server, reproduce 2–3 headline experiments per Section 9) | 1.5 | Medium |
| PR approval & merge branch to target | 0.5 | Medium |
| **Total** | **4.0** | — |

### 2.3 Reconciliation

- Section 2.1 (Completed) = **56h** · Section 2.2 (Remaining) = **4h** · **Sum = 60h** = Total Project Hours (§1.2). ✔
- Remaining hours are identical across §1.2, §2.2, and the §7 pie chart (**4h**). ✔
- No AAP requirement is left unmapped; there are **no partially-completed items** (each requirement is either fully complete or a not-yet-started human gate).

---

## 3. Test Results

For this investigative QnA task, the "tests" are the **evidence-reproduction experiments and build gates executed by Blitzy's autonomous validation systems** (Final Validator, Gate 1 & Gate 3). Each was reproduced against a live SFTPGo server; the stable, asked-for literals matched the document exactly. The offline build and its identity/size were **independently re-verified during this assessment**.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|:-----------:|:------:|:------:|:----------:|-------|
| Concurrency / Session limit (Q1, Q6) | Custom `pkg/sftp` harness | 2 | 2 | 0 | 100%* | 4 sessions → **2 accepted / 2 rejected** at `2/2`; login-before-register window reproduced. |
| Quota enforcement (Q2, Q4) | `pkg/sftp` harness + `sqlite3` | 3 | 3 | 0 | 100%* | Per-step fill `0\|0 → 1\|2048 → 2\|4096 → denied` (`SSH_FX_FAILURE`); TOCTOU overshoot `0\|0 → 4\|8192`; 30-way burst → exact `30\|122880` (no lost updates). |
| Atomic upload lifecycle (Q3, Q5) | Harness + REST `DELETE` + FS inspection | 4 | 4 | 0 | 100%* | Mode-1 abort (temp deleted, quota unchanged); clean completion (`31\|126976`); mode-2 resume (temp kept); mode-0 standard (partial lingers). |
| Idle checker (Q1, Q6) | Live server observation | 1 | 1 | 0 | 100%* | Genuine idle close at the hard-coded ~300s ticker cadence. |
| Race detection | Go `-race` toolchain | 1 | 1 | 0 | N/A | `DATA RACE` count = **0** on the `RWMutex`-guarded registries. |
| Build & static gates | `go build` / `go vet` | 2 | 2 | 0 | N/A | Offline build clean (`0.9.5-dev`, 31,882,120 bytes); `go vet` passes. |
| **Total** | — | **13** | **13** | **0** | — | 0 failures. |

> `*` Coverage for the QnA rows denotes **sub-question coverage** — all six sub-questions (Q1–Q6) are answered with reproduced evidence (6/6 = 100%). Traditional line coverage is not applicable to a read-only documentation task.
>
> **Integrity note:** every test above originates from Blitzy's autonomous validation logs for this project (evidence reproduction + build/vet/`-race` gates). The build identity/size was additionally re-verified independently in this assessment.

---

## 4. Runtime Validation & UI Verification

Runtime health observed during autonomous validation (and corroborated by an independent offline build in this assessment):

- ✅ **Operational** — SFTPGo binary builds offline (`GOPROXY=off`, CGO) and reports `SFTPGo version: 0.9.5-dev` (31,882,120 bytes; `-race` variant 39,685,616 bytes).
- ✅ **Operational** — SFTP subsystem on port **2022**: concurrent `pkg/sftp` uploads accepted/rejected exactly at the `max_sessions` boundary (`2/2`).
- ✅ **Operational** — REST control plane on **127.0.0.1:8080**: `POST /api/v1/user`, `GET /api/v1/version`, `DELETE /api/v1/connection/{id}`, and `/api/v1/quota_scan` all responded with documented status/bodies.
- ✅ **Operational** — Quota persistence (SQLite): `used_quota_files`/`used_quota_size` progressed and reconciled exactly (`30|122880` after a 30-way burst; incremented to `31|126976` after a clean upload).
- ✅ **Operational** — Idle checker: genuine idle close observed at the hard-coded ~5-minute ticker cadence; deterministic mid-stream drops via REST `DELETE`.
- ⚠ **Not applicable (by design)** — **UI Verification:** this task delivers a Markdown document with **no UI changes in scope**. SFTPGo's optional web UI (`/web/*`) was not exercised because it is outside the AAP scope; there is nothing to visually verify for this deliverable.

---

## 5. Compliance & Quality Review

Cross-mapping AAP deliverables and the governing rule (`SWE-AtlasQnA-Repo`) to quality benchmarks:

| Benchmark / Requirement | Status | Progress | Evidence |
|-------------------------|:------:|:--------:|----------|
| Single answer document at mandated path/name (`blitzy/documentation/sftpgo_44634210287c.md`) | ✅ Pass | 100% | File present; branch-derived name; only repo addition. |
| Investigate-by-running-first | ✅ Pass | 100% | Live server build/run; all Q1–Q6 reproduced (Gate 1/2). |
| Verbatim evidence with producing command | ✅ Pass | 100% | Evidence appendix §4.1–4.12 (logs, HTTP, `sqlite3`, FS listings, `-race`). |
| Exactness & grounding — exact literals + `file:line` | ✅ Pass | 100% | 124 citations; 6/6 independent spot-checks matched; full audit = zero discrepancies. |
| Answer every sub-question + final coverage pass | ✅ Pass | 100% | Dedicated Q1–Q6 headings + coverage pass with exact-literals list. |
| Web-search corroboration | ✅ Pass | 100% | Section 5 of the document + 4 cited sources (confirmatory only). |
| Read-only source scope | ✅ Pass | 100% | `git diff` adds one file; zero source/test/config edits. |
| Cleanup / clean working tree | ✅ Pass | 100% | `/tmp` workspace removed; `git status --porcelain` empty. |
| Build / static quality (`go build`, `go vet`, `-race`) | ✅ Pass | 100% | Clean offline build; `go vet` passes; 0 data races. |
| Markdown validity (fences, tables, diagram) | ✅ Pass | 100% | 54 balanced fences, 5 tables, 1 Mermaid diagram; arithmetic consistency verified. |
| Human SME technical sign-off | ⏳ Pending | 0% | Scheduled — Section 2.2 / task HT-1 (path-to-production). |

**Fixes applied during autonomous validation:** none were required this session — the document was proven flawless (zero edits). Earlier review cycles (prior commits) resolved 3 code-review findings and corrected 2 verbatim-fidelity literals.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|------------|--------|
| Residual technical-accuracy risk pending human SME sign-off | Technical | Low | Low | 124 citations audited + 6/6 independent spot-checks matched + all Q1–Q6 reproduced live; SME review scheduled | Open (human gate) |
| Citation version-drift if the document is reused against a different commit | Technical | Low | Low | Document pins commit `44634210` / build `0.9.5-dev`; keep citations tied to that commit | Mitigated |
| Run-specific evidence values vary per run (interruption byte counts, idle fractional seconds, total log-line counts) | Technical | Low | Low | Methodology stated in the document; factual claims use stable literals; run-specific values labeled as such | Documented |
| Documented `hasSpace` TOCTOU quota-overshoot is an **observed real behavior**, not fixed (out of scope) | Technical (informational) | Low | Medium | Flagged as out-of-scope observed behavior; no code change mandated by the read-only AAP | Documented |
| Ephemeral test credentials + auto-generated host key were in `/tmp` only | Security | Low | Low | Read-only task; `git status` empty; no secrets/binaries/DB committed (verified) | Mitigated |
| Evidence reproduction needs Go 1.13.15 + CGO(gcc) + module cache/network | Operational | Low | Medium | Development Guide documents toolchain + offline (`GOPROXY=off`) and online paths; module cache present in this environment | Mitigated |
| Genuine idle-close observation is slow (~5-min hard-coded ticker) | Operational | Low | Low | Deterministic `DELETE /api/v1/connection/{id}` lever used for mid-stream drops; idle path documented separately | Documented |
| PR merge to target branch | Integration | Low | Low | Single new file in a new path (`blitzy/documentation/`), no source overlap → negligible conflict risk | Open (human gate) |

**Risk profile:** uniformly **Low severity** (no High/Medium), consistent with a fully-validated, read-only, documentation-only deliverable whose only outstanding work is human acceptance. No security risk was introduced (no code changed); no runtime-integration risk (standalone Markdown, no external services, API keys, or deployment).

---

## 7. Visual Project Status

**Project hours (AAP-scoped).** "Remaining Work" = **4h**, equal to §1.2 Remaining Hours and the sum of the §2.2 Hours column.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#5B39F3','pieStrokeWidth':'2px','pieOuterStrokeColor':'#5B39F3','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieTitleTextColor':'#5B39F3','pieLegendTextColor':'#333333'}}}%%
pie showData title Project Hours Breakdown (Total 60h)
    "Completed Work" : 56
    "Remaining Work" : 4
```

**Remaining work by task (sums to 4h).**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#5B39F3','pieStrokeWidth':'1px','pieOuterStrokeColor':'#5B39F3','pieSectionTextColor':'#111111','pieTitleTextColor':'#5B39F3','pieLegendTextColor':'#333333'}}}%%
pie showData title Remaining Work by Task (4h)
    "SME review (High)" : 2.0
    "Evidence reproduction (Medium)" : 1.5
    "PR approval & merge (Medium)" : 0.5
```

| Remaining Category (from §2.2) | Hours | Priority |
|---|---:|---|
| SME technical review | 2.0 | High |
| Independent evidence reproduction | 1.5 | Medium |
| PR approval & merge | 0.5 | Medium |
| **Total** | **4.0** | — |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-grounded answer to a demanding concurrency/quota/atomic-upload investigation. All six sub-questions (Q1–Q6) are answered explicitly from **observed runtime behavior**: session admission enforced at `loginUser` (rejection at the exact `2/2` boundary); the quota **check racing** in `hasSpace` (reading committed usage with no reservation) while the **write serializes** at `SetMaxOpenConns(1)` via an incremental additive `UPDATE` (30-way burst reconciles to exactly `30|122880`); and atomic-mode aborts that delete the `.sftpgo-upload.<xid>.<name>` temp file and reverse quota accounting, contrasted against clean completion (`31|126976`) and modes 0/2. Every claim carries an exact `file:line` citation; a `-race` run shows zero data races.

**Remaining gaps.** None are engineering defects. The outstanding **4 hours** are human path-to-production: SME technical sign-off, an optional independent reproduction of headline experiments, and PR merge.

**Critical path to production.** SME review → (optional) reproduction → PR merge. There are no blocking issues and no access issues.

**Production readiness.** The deliverable is **93.3% complete** (56h of 60h AAP-scoped). The document is complete, validated (5/5 autonomous gates passed, zero edits required), correctly named/located, and committed atop a clean, read-only repository. It is ready for human acceptance; per policy, completion is held below 100% pending that human review.

| Success Metric | Result |
|----------------|--------|
| Sub-questions answered (Q1–Q6) | 6 / 6 (100%) |
| `file:line` citations (audited) | 124 (zero discrepancies) |
| Autonomous validation gates passed | 5 / 5 |
| Evidence-reproduction checks | 13 / 13 passed |
| Data races (`-race`) | 0 |
| Repository files modified (source/test/config) | 0 (read-only honored) |
| AAP-scoped completion | 93.3% |

---

## 9. Development Guide

This guide explains how to build and run SFTPGo and reproduce the document's evidence. Commands marked **(tested)** were executed during this assessment. All runtime artifacts should live **outside** the repository (e.g., under `/tmp`) to preserve the read-only mandate.

### 9.1 System Prerequisites

- **OS:** Linux/amd64 (validated on Ubuntu).
- **Go:** 1.13.x toolchain — **(tested)** `go version` → `go version go1.13.15 linux/amd64`.
- **C toolchain:** `gcc` + `libc6-dev` (CGO is required by the default `go-sqlite3` provider) — **(tested)** `gcc --version` → `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`.
- **Utilities:** `sqlite3` CLI (inspect quota) and `curl` (drive the REST API).

### 9.2 Environment Setup

```bash
# Put the Go toolchain on PATH (this environment ships a profile script)
source /etc/profile.d/go.sh
go version            # expect: go version go1.13.15 linux/amd64

# CGO must be enabled for the default SQLite provider
export CGO_ENABLED=1
```

### 9.3 Dependency Installation

```bash
# The module cache in this environment is fully populated (96 modules),
# so offline builds work. To fetch on a fresh machine instead:
#   go mod download
# No dependency changes are made by this task (read-only investigation).
ls /root/go/pkg/mod/github.com/pkg/sftp@v1.11.0          # cached (tested)
ls /root/go/pkg/mod/github.com/mattn/go-sqlite3@v2.0.2+incompatible   # cached (tested)
```

### 9.4 Build

```bash
cd <repo-root>
# Build OUTSIDE the repo tree to keep the working tree clean:
mkdir -p /tmp/sftpgo-out
CGO_ENABLED=1 GOPROXY=off go build -o /tmp/sftpgo-out/sftpgo .   # (tested) exit 0, ~3s warm cache
/tmp/sftpgo-out/sftpgo --version                                 # (tested) -> SFTPGo version: 0.9.5-dev
```

**Expected:** identity `SFTPGo version: 0.9.5-dev`; binary size **31,882,120 bytes**. A benign CGO warning `sqlite3-binding.c:125801:10: warning: ...` is expected and is **not** an error.

### 9.5 Runtime Provisioning (isolated `/tmp` workspace)

```bash
WS=/tmp/sftpgo_obs
mkdir -p "$WS"/{config,home,db,logs}

# The SQLite provider does NOT auto-create the schema — create the users table first.
# (Full DDL is in .travis.yml before_script; key quota/session columns shown.)
sqlite3 "$WS/db/sftpgo.db" 'CREATE TABLE "users" (
  "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT, "username" varchar(255) NOT NULL UNIQUE,
  "password" varchar(255) NULL, "public_keys" text NULL, "home_dir" varchar(255) NOT NULL,
  "uid" integer NOT NULL, "gid" integer NOT NULL, "max_sessions" integer NOT NULL,
  "quota_size" bigint NOT NULL, "quota_files" integer NOT NULL, "permissions" text NOT NULL,
  "used_quota_size" bigint NOT NULL, "used_quota_files" integer NOT NULL, "last_quota_update" bigint NOT NULL,
  "upload_bandwidth" integer NOT NULL, "download_bandwidth" integer NOT NULL, "expiration_date" bigint NOT NULL,
  "last_login" bigint NOT NULL, "status" integer NOT NULL, "filters" TEXT NULL, "filesystem" text NULL);'

# Write a config into the workspace overriding upload_mode (1 = atomic; 2 = atomic-with-resume),
# a short idle_timeout, and the data_provider paths. Then start the server with verbose JSON logs.
/tmp/sftpgo-out/sftpgo serve -c "$WS/config" -l "$WS/logs/sftpgo.log" -v &
echo $! > "$WS/sftpgo.pid"     # track the PID so ONLY this process is killed later
```

**Ports:** SFTP on **2022**; REST/web on **127.0.0.1:8080**.

### 9.6 Verification & Reproducing the Evidence

```bash
# Server identity
curl -s http://127.0.0.1:8080/api/v1/version

# Create a strict user (small quota_files / quota_size / max_sessions)
curl -s -X POST http://127.0.0.1:8080/api/v1/user -H 'Content-Type: application/json' -d '{
  "username":"strict","password":"strictpass","home_dir":"'"$WS"'/home/strict",
  "uid":0,"gid":0,"max_sessions":2,"quota_size":4096,"quota_files":2,
  "permissions":{"/":["*"]},"status":1
}'

# Drive concurrent pkg/sftp uploads with a small Go harness (built under /tmp):
#   - 4 staggered sessions -> 2 ACCEPTED / 2 REJECTED ("too many open sessions: 2/2")
#   - per-step 2048-byte uploads -> quota 0|0 -> 1|2048 -> 2|4096 -> denied (SSH_FX_FAILURE)

# Force a mid-stream drop deterministically (atomic-abort path):
CID=$(curl -s http://127.0.0.1:8080/api/v1/connection | python3 -c 'import sys,json;print(json.load(sys.stdin)[0]["connection_id"])')
curl -s -X DELETE "http://127.0.0.1:8080/api/v1/connection/$CID"

# Inspect quota (unchanged after an atomic abort; incremented after a clean upload)
sqlite3 "$WS/db/sftpgo.db" 'SELECT used_quota_files, used_quota_size FROM users WHERE username="strict";'
```

**Headline expected values:** session boundary `2/2`; per-step quota `0|0 → 1|2048 → 2|4096`; concurrent overshoot `0|0 → 4|8192`; no-lost-update total `30|122880`; clean-completion increment `31|126976`; atomic temp-file name `.sftpgo-upload.<xid>.<name>`; `-race` `DATA RACE` count `0`.

### 9.7 Teardown (mandatory — keep the repo clean)

```bash
kill "$(cat /tmp/sftpgo_obs/sftpgo.pid)" 2>/dev/null   # kill ONLY the tracked PID (never broad pkill)
rm -rf /tmp/sftpgo_obs /tmp/sftpgo-out
cd <repo-root> && git status --porcelain               # expect: empty (clean tree)
```

### 9.8 Troubleshooting

| Symptom | Cause | Resolution |
|---------|-------|------------|
| `exec: "gcc": executable file not found in $PATH` | CGO needs a C compiler | Install: `apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y gcc libc6-dev`. |
| `Package 'gcc' has no installation candidate` | apt index not refreshed | Run `apt-get update` **first**, then install `gcc`. |
| Build fails offline with proxy errors | Module cache incomplete | Populate cache (`go mod download` online) or set `GOPROXY=off` only when the cache is present. |
| Provider error about a missing `users` table | SQLite provider does not auto-create schema | Create the `users` table (DDL in §9.5) **before** starting the server. |
| `sqlite3-binding.c:125801:10: warning:` during build | Benign CGO compiler warning | Expected — not an error; the build still succeeds. |
| Cannot force a genuine idle close quickly | Idle ticker is hard-coded to 5 minutes | Use `DELETE /api/v1/connection/{id}` for deterministic mid-stream drops; document the idle path separately. |

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Toolchain check | `source /etc/profile.d/go.sh && go version` |
| Offline build (repo-clean) | `CGO_ENABLED=1 GOPROXY=off go build -o /tmp/sftpgo-out/sftpgo .` |
| Binary identity | `/tmp/sftpgo-out/sftpgo --version` |
| Static check | `go vet ./...` |
| Race build | `CGO_ENABLED=1 go build -race -o /tmp/sftpgo-out/sftpgo-race .` |
| Start server | `./sftpgo serve -c <config> -l <log> -v &` |
| REST: version | `curl -s http://127.0.0.1:8080/api/v1/version` |
| REST: create user | `curl -s -X POST http://127.0.0.1:8080/api/v1/user -d '{...}'` |
| REST: list connections | `curl -s http://127.0.0.1:8080/api/v1/connection` |
| REST: force-close (mid-stream) | `curl -s -X DELETE http://127.0.0.1:8080/api/v1/connection/{connectionID}` |
| Quota inspect | `sqlite3 sftpgo.db 'SELECT used_quota_files, used_quota_size FROM users WHERE username="strict";'` |
| Repo cleanliness | `git status --porcelain` |

### Appendix B — Port Reference

| Service | Address / Port | Config Key |
|---------|----------------|------------|
| SFTP server | `:2022` | `sftpd.bind_port` |
| REST API / Web | `127.0.0.1:8080` | `httpd.bind_port` / `httpd.bind_address` |

### Appendix C — Key File Locations

| Item | Path |
|------|------|
| **Deliverable (only repo change)** | `blitzy/documentation/sftpgo_44634210287c.md` |
| Connection registry, idle ticker | `sftpd/sftpd.go` (RWMutex `:56-61`, 5-min ticker `:132`, idle close `:348`) |
| Atomic completion & quota mutation | `sftpd/transfer.go` (rename/delete `:134-148`, quota guard `:166-168`) |
| Pre-upload quota check & write routing | `sftpd/handler.go` (`hasSpace` `:515-534`, deny log `:414`) |
| Session limit & connection lifecycle | `sftpd/server.go` (`loginUser` `:371-376`, idle timeout minutes `:39`) |
| Quota provider / SQL / SQLite serialization | `dataprovider/dataprovider.go`, `dataprovider/sqlqueries.go`, `dataprovider/sqlite.go:44` |
| Atomic temp-file naming | `vfs/osfs.go:175-178` (`.sftpgo-upload.<xid>.<name>`) |
| REST control plane | `httpd/httpd.go:22-28`, `httpd/router.go` |
| Default configuration | `sftpgo.json` |
| Users-table DDL | `.travis.yml` (before_script) |

### Appendix D — Technology Versions

| Component | Version |
|-----------|---------|
| Go toolchain | 1.13.15 (`go.mod` declares `go 1.13`) |
| gcc (CGO) | 15.2.0 |
| SFTPGo build identity | `0.9.5-dev` (commit `44634210`) |
| `github.com/pkg/sftp` | v1.11.0 |
| `github.com/mattn/go-sqlite3` | v2.0.2+incompatible (CGO) |
| `github.com/rs/xid` | v1.2.1 |
| `github.com/rs/zerolog` | v1.17.2 |
| `github.com/go-chi/chi` | v4.0.2+incompatible |
| `github.com/prometheus/client_golang` | v1.3.0 |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `CGO_ENABLED` | `1` | Required for the default `go-sqlite3` provider. |
| `GOPROXY` | `off` (optional) | Force offline builds when the module cache is populated. |
| `DEBIAN_FRONTEND` | `noninteractive` | Non-interactive `apt-get` when installing `gcc`/`libc6-dev`. |
| `CI` | `true` (recommended) | Non-interactive Go tooling. |

### Appendix F — Developer Tools Guide

- **Race detector** — build/run with `-race` to characterize the `RWMutex`-guarded registries; the document reports a `DATA RACE` count of **0** (the quota overshoot is a *logical* TOCTOU race, not an unsynchronized memory access).
- **`sqlite3` CLI** — read `used_quota_files` / `used_quota_size` to observe per-step quota progression and no-lost-update reconciliation.
- **`curl`** — drive the REST control plane; the `DELETE /api/v1/connection/{id}` endpoint is the deterministic lever for mid-stream drops.
- **`git status --porcelain`** — the read-only invariant check; must be empty aside from the single new document.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the authoritative scope for this task. |
| **QnA** | Investigative question-and-answer documentation deliverable. |
| **TOCTOU** | Time-of-check/time-of-use — the `hasSpace` window where concurrent uploads can both pass the quota check and collectively overshoot. |
| **Atomic upload** | `upload_mode:1` — data written to a temp file, then renamed to the target on success; deleted on error (accounting reversed). |
| **Atomic-with-resume** | `upload_mode:2` — like atomic, but the temp file is kept/renamed on error to allow resume. |
| **Standard upload** | `upload_mode:0` (default) — written directly to the target; a partial file lingers on failure. |
| **Idle checker** | Background goroutine that closes idle connections; driven by a hard-coded 5-minute ticker against the minute-granularity `idle_timeout`. |
| **`hasSpace`** | Pre-upload quota check reading committed usage without reserving capacity. |
| **`SetMaxOpenConns(1)`** | SQLite provider setting that serializes all writes — the runtime serialization point for quota updates. |

---

*Generated by the Blitzy Platform. Completion is AAP-scoped (PA1 methodology): 56h completed of 60h total = 93.3%. Remaining 4h are human path-to-production activities.*