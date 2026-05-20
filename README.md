# CoreWeave Demo — Verified Vulnerability Journey (May 2026)

> **TL;DR for execs**: see [DEMO-WALKTHROUGH.md](DEMO-WALKTHROUGH.md) —
> 8-min top-to-bottom read of the full demo with sample verifier
> outputs, no execution required.

This repository demonstrates an end-to-end **verified vulnerability
journey** for two real-world, publicly disclosed remote code execution
flaws — walking each CVE through **four customer-facing artifacts**.

| CVE | Product | Class | Severity |
|---|---|---|---|
| **CVE-2025-8110** | Gogs ≤ 0.13.3 | Symlink path-traversal RCE (auth required) | High (CVSS 8.7) |
| **CVE-2025-3248** | Langflow < 1.3.0 | Unauthenticated Python `exec` RCE | Critical (CVSS 9.8) |

Both CVEs have observed in-the-wild exploitation (Supershell C2 for
Gogs, Flodrix botnet for Langflow). Both are on the CISA KEV catalog —
CVE-2025-3248 (added 2025-05-05) and CVE-2025-8110 (added 2026-01-12).
Both predate this demo by months — they are not theoretical.

## The 4-artifact deliverable, per CVE

| # | Artifact | Where it lives in this repo |
|---|---|---|
| **1** | **Vulnerability report** — original disclosure summary, severity, references, IoCs | Top section of the GitHub issue body |
| **2** | **FAIR business + compliance triage** — Open FAIR §14 loss-exceedance estimate with reviewer-panel verdicts; canonical branded HTML attachment | "FAIR business and compliance triage" section in the same issue body + sibling HTML in `attachments/` |
| **3** | **Exploit verifier** — reproducible container that deterministically triggers the vulnerability (Harbor task: instruction, environment, tests, reward script) | This repository's **pull request** for that CVE, under `harbor-tasks/CVE-*/harbor/` |
| **4** | **Verified patch** — verbatim upstream fix commit applied through the verifier; reward `1.0` = vulnerability fixed AND functional regression intact | Same PR, `harbor-tasks/CVE-*/ground_truth/fix.patch` |

The pair was chosen specifically to exercise complementary surfaces:

- **Gogs** is Go, requires authentication, exploits filesystem semantics.
- **Langflow** is Python, fully unauthenticated, exploits language-runtime semantics.

A patch agent (and risk model) that handles both is doing real work.

## Layout

```
.
├── README.md                              # This file
├── attachments/                           # Canonical branded HTML triage reports
│   ├── CVE-2025-8110.html
│   └── CVE-2025-3248.html
├── harbor-tasks/                          # Lands via per-CVE PRs (see Issues #1, #2)
│   ├── CVE-2025-8110/
│   │   ├── ground_truth/fix.patch         # Upstream 553707f3 verbatim
│   │   ├── healthcheck.json               # Acceptance-gate spec
│   │   └── harbor/
│   │       ├── environment/Dockerfile     # debian:bookworm-slim + Go 1.21 deps stage
│   │       ├── instruction.md             # Agent task brief (scrubbed)
│   │       ├── task.toml                  # Harbor task descriptor
│   │       └── tests/                     # Verifier + reward script
│   └── CVE-2025-3248/
│       └── ... (same shape, python:3.13-slim-bookworm base)
└── vulnerable/                            # Read-only upstream subtrees (squashed)
    ├── gogs/                              # gogs/gogs @ v0.13.3
    └── langflow/                          # langflow-ai/langflow @ 1.2.0
```

## How to verify locally

For each CVE, after the corresponding PR has merged:

1. `cd harbor-tasks/CVE-XXXX/`
2. `docker build -t cve-xxxx-env -f harbor/environment/Dockerfile harbor/environment/`
3. Run the verifier against the unpatched container: `bash harbor/tests/test.sh` → `reward=0.0` (exploit triggers)
4. Apply the upstream patch: `git apply ground_truth/fix.patch`, then re-run `bash harbor/tests/test.sh` → `reward=1.0`
5. `reward=1.0` is the acceptance gate: vulnerability is fixed AND the functional regression test still passes

## Provenance

The FAIR triage section in each issue body carries a **Verifiable Agent
Conversation Record (VACR)** signed via COSE_Sign1 / Ed25519
(IETF RFC 9052). The public key is `VAC_SIGNING_PUBKEY`; the signed
payload identifies the persona, model, raw-artifact content hashes, and
reviewer panel verdicts that produced each row.

## Status

`healthcheck.json` in each harbor task is currently labelled
`PENDING_DOCKER_VALIDATION` — the verifier file trees are structurally
complete, with the canonical upstream fix wired through, but the
end-to-end `docker build` + reward-loop validation has not yet been
executed against these specific scaffolds. The
`operator_validation_steps` block in each healthcheck spells out the
exact commands a recipient runs to close the gate. Production wiring
of the live verifier and verified-patch loop lands in the next
iteration.

## Acknowledgements

This demo would not exist without the original disclosing researchers:

- **CVE-2025-8110**: Gili Tikochinski and Yaara Shriki at Wiz Research
- **CVE-2025-3248**: Naveen Sunkavally at Horizon3.ai

All source code under `vulnerable/` belongs to its respective upstream
project and is governed by their licenses.
