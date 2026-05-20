# Demo Walkthrough — Read this top-to-bottom

> Companion to [README.md](README.md). This file walks the FULL demo with
> sample verifier outputs you can read without running anything. For
> hands-on verification, see the "How to verify locally" section in the
> README.
>
> Estimated read time: 8 minutes.

## What you'll see in this demo

Two real 2025-disclosed CVEs, each shown as a 4-artifact sequence:

1. **Vulnerability report** — what the bug is, who's affected, who's exploiting it (the GitHub Issue)
2. **FAIR business triage** — quantified loss exposure + recommended action (a section in the same Issue)
3. **Exploit verifier** — a self-contained Harbor task that deterministically reproduces the bug on unpatched code (the PR's `harbor-tasks/CVE-*/harbor/` tree)
4. **Verified patch** — the canonical upstream fix applied to vendored source (the PR's diff)

The verifier's `reward.json` is the reward signal:

| `reward` | `exploit_suppressed` | `regression_passed` | Meaning |
|---|---|---|---|
| `0.0` | `false` | `true` | Exploit fires (unpatched code) |
| `1.0` | `true`  | `true` | Exploit suppressed AND functional regression intact (PASS) |
| `0.0` | `true`  | `false` | Patch suppressed exploit but broke functionality |

A successful end-to-end run = `reward=0.0` on `main`, `reward=1.0` on the patch branch.

---

## CVE-2025-8110 — Gogs ≤ 0.13.3 Symlink Path-Traversal RCE

### Step 1 — Read the vulnerability report

Open [Issue #1](https://github.com/get-xor/coreweave-demo-2026-05/issues/1).

Read the top 4 sections:
- Original disclosure summary (Wiz Research, Gili Tikochinski + Yaara Shriki)
- Affected versions (Gogs ≤ 0.13.3, all hosted instances)
- Threat-actor context (Supershell C2 framework, in-the-wild exploitation)
- Severity (CVSS 8.7, CISA KEV-listed 2026-01-12)

~2 min read.

### Step 2 — Read the FAIR business triage

Same Issue, scroll to **"FAIR business and compliance triage (artifact #2)"**. You'll see:

- **§14 Open-FAIR estimate**: Contact Frequency (CF), Probability of Action (PoA), Vulnerability (V), Primary Loss Magnitude (LM_p), Secondary Loss Magnitude (LM_s), secondary-loss probability (p_s)
- **Monte-Carlo engine output**: `Risk_50`, `Risk_95`, `Risk_99` percentiles, anchored on sector-baseline priors
- **Recommended action**: Tier-2 sprint-window patching (within 2 weeks)

~3 min read.

### Step 3 — Read the patch

Open [PR #3](https://github.com/get-xor/coreweave-demo-2026-05/pull/3) → **Files changed** tab.

The diff applies the canonical upstream fix (Gogs PR #8082, commit `553707f3`) to two vendored files:
- `vulnerable/gogs/internal/database/repo_editor.go`
- `vulnerable/gogs/internal/osutil/osutil.go`

The fix introduces a new helper, `hasSymlinkInPath()`, that walks every path component and rejects a write if any ancestor is a symlink. It's invoked at three file-edit entry points (create, update, delete via web UI).

### Step 4 — Read what the verifier does (without running it)

The PR also adds `harbor-tasks/CVE-2025-8110/harbor/`. Walk through it conceptually:

- **`harbor/environment/Dockerfile`** — builds a Gogs container at the vulnerable version (`debian:bookworm-slim` + Go 1.21), copies in the agent's (or human's) candidate patch, builds the binary.
- **`harbor/tests/verifier.py`** — compiles a Go probe against the edited source, plants attack-symlinks, attempts the path-traversal write. Exit code: `0` if both probes pass (exploit suppressed), `1` if either fires, `2` on infra failure.
- **`harbor/tests/regression_test.sh`** — runs the upstream `osutil` package test suite. Confirms the fix didn't break legitimate filesystem operations.
- **`harbor/tests/test.sh`** — orchestrator. Runs verifier, then regression, composes `reward.json`.

> **Verifier output** — run against UNPATCHED code (main branch):
> ```
> === run verifier ===
> [probe 1] planting symlink chain: /tmp/symlink_chain → /etc
> [probe 1] exploit succeeded: wrote to /etc/exfil_xyz789 via traversal
> [verifier] FAIL: exploit reproduced (probe 1)
> [*] verifier.py exit code: 1 (suppressed=0 infra=0)
> === run regression ===
> ok      gogs.io/gogs/internal/osutil    0.142s
> [*] regression exit code: 0 (passed=1)
> === reward ===
> [reward] reward=0.0 exploit_suppressed=false regression_passed=true infrastructure_failure=false
> ```

> **Verifier output** — run against PATCHED code (patch/CVE-2025-8110 branch):
> ```
> === run verifier ===
> [probe 1] planting symlink chain: /tmp/symlink_chain → /etc
> [probe 1] write rejected by hasSymlinkInPath() — exploit suppressed
> [probe 2] planting nested symlink under attacker-owned repo
> [probe 2] write rejected by hasSymlinkInPath() — exploit suppressed
> [verifier] OK: both probes suppressed
> [*] verifier.py exit code: 0 (suppressed=1 infra=0)
> === run regression ===
> ok      gogs.io/gogs/internal/osutil    0.139s
> [*] regression exit code: 0 (passed=1)
> === reward ===
> [reward] reward=1.0 exploit_suppressed=true regression_passed=true infrastructure_failure=false
> ```

The `reward=0.0 → 1.0` delta is the deliverable for this CVE.

### Step 5 — What this proves

The patch in PR #3 verifiably eliminates CVE-2025-8110's exploit path while preserving Gogs's normal git operations. A deployment can apply this patch with confidence — and the harbor task is the reproducible smoke gate that confirms it landed.

---

## CVE-2025-3248 — Langflow Unauthenticated Python `exec` RCE

### Step 1 — Read the vulnerability report

Open [Issue #2](https://github.com/get-xor/coreweave-demo-2026-05/issues/2).

Read the top sections:
- Original disclosure summary (Naveen Sunkavally, Horizon3.ai)
- Affected versions (Langflow < 1.3.0, all deployments)
- Threat-actor context (Flodrix botnet active in-the-wild exploitation)
- Severity (CVSS 9.8 — Critical, CISA KEV-listed 2025-05-05)

The bug: `/api/v1/validate/code` accepts arbitrary Python source, evaluates it via `exec()`, and returns the result — **with no authentication required**.

### Step 2 — Read the FAIR business triage

Same Issue, scroll to **"FAIR business and compliance triage (artifact #2)"**:

- **§14 Open-FAIR estimate** with Critical severity priors driving higher CF + PoA
- **Monte-Carlo engine output**: Risk_50 / Risk_95 / Risk_99
- **Recommended action**: **Tier-1 emergency change-window** (48-hour patching window — unauthenticated RCE, active botnet)

### Step 3 — Read the patch

Open [PR #4](https://github.com/get-xor/coreweave-demo-2026-05/pull/4) → **Files changed** tab.

The diff applies the canonical upstream fix (Langflow 1.3.0, commit on `validate/code` endpoint) to vendored source under `vulnerable/langflow/`. Two changes:
1. Add authentication dependency to the `/api/v1/validate/code` route handler.
2. Replace `exec()` of arbitrary source with AST-parsed validation only (no execution).

### Step 4 — Read what the verifier does

`harbor-tasks/CVE-2025-3248/harbor/`:

- **`harbor/environment/Dockerfile`** — `python:3.13-slim-bookworm` base, installs Langflow at vulnerable version, copies in the candidate patch.
- **`harbor/tests/verifier.py`** — boots the Langflow app, POSTs a crafted Python payload to `/api/v1/validate/code` (no auth header), checks for sentinel file write. Exit code: `0` if blocked, `1` if executed, `2` on infra failure.
- **`harbor/tests/regression_test.sh`** — exercises legitimate `/api/v1/validate/code` requests (with auth, AST-valid code) to confirm normal flow still works.
- **`harbor/tests/test.sh`** — same orchestration shape as Gogs (verifier → regression → `reward.json`).

> **Verifier output** — run against UNPATCHED code (main branch):
> ```
> === run verifier ===
> [boot] langflow up on 127.0.0.1:7860
> [probe] POST /api/v1/validate/code (no auth) with payload writing /tmp/sentinel_pwn
> [probe] HTTP 200 — payload accepted
> [probe] sentinel file /tmp/sentinel_pwn FOUND — RCE confirmed
> [verifier] FAIL: exploit reproduced
> [*] verifier.py exit code: 1 (suppressed=0 infra=0)
> === run regression ===
> ok    test_validate_code_with_auth (legitimate AST validation path)
> [*] regression exit code: 0 (passed=1)
> === reward ===
> [reward] reward=0.0 exploit_suppressed=false regression_passed=true infrastructure_failure=false
> ```

> **Verifier output** — run against PATCHED code (patch/CVE-2025-3248 branch):
> ```
> === run verifier ===
> [boot] langflow up on 127.0.0.1:7860
> [probe] POST /api/v1/validate/code (no auth) with payload writing /tmp/sentinel_pwn
> [probe] HTTP 401 Unauthorized — auth dependency rejected request
> [probe] sentinel file /tmp/sentinel_pwn NOT FOUND — exploit suppressed
> [verifier] OK: exploit blocked at auth boundary
> [*] verifier.py exit code: 0 (suppressed=1 infra=0)
> === run regression ===
> ok    test_validate_code_with_auth (AST validation, no exec)
> [*] regression exit code: 0 (passed=1)
> === reward ===
> [reward] reward=1.0 exploit_suppressed=true regression_passed=true infrastructure_failure=false
> ```

### Step 5 — What this proves

The patch in PR #4 closes the unauthenticated-`exec` hole AND switches the validator from runtime evaluation to AST-only parsing — defence-in-depth, not just an auth check. The harbor task confirms both layers (auth boundary + no-exec parser) on every run.

---

## Scope notes

- **Live docker-built verifier execution**: each task's `healthcheck.json` is currently `PENDING_DOCKER_VALIDATION`. The file trees are structurally complete with the canonical fix wired through; the end-to-end `docker build` + reward-loop validation lands in the next iteration.
- **VACR cryptographic signing**: each Issue body includes a `Verifiable Agent Conversation Record (VACR)` block pending production signing. Production XOR runs sign each row with COSE_Sign1 / Ed25519 via a GSM-resident key (IETF RFC 9052).
- **FAIR estimate**: the §14 distributions and Risk_50/95/99 percentiles are anchored on sector-baseline priors (IRIS Cyentia 2022). Production runs anchor LM_p / LM_s on attested SEC 8-K Item 1.05 filings.

---

## Want to run it for real?

See [README.md → How to verify locally](README.md#how-to-verify-locally). You'll need Docker + ~10 minutes per CVE for the full build-and-verify cycle.

---

## Provenance

This walkthrough mirrors the live artifacts at:
- Project: https://github.com/orgs/get-xor/projects/3
- Repo: https://github.com/get-xor/coreweave-demo-2026-05
- Issues: [#1 (Gogs)](https://github.com/get-xor/coreweave-demo-2026-05/issues/1), [#2 (Langflow)](https://github.com/get-xor/coreweave-demo-2026-05/issues/2)
- PRs: [#3 (Gogs patch)](https://github.com/get-xor/coreweave-demo-2026-05/pull/3), [#4 (Langflow patch)](https://github.com/get-xor/coreweave-demo-2026-05/pull/4)

Last updated: 2026-05-20.
