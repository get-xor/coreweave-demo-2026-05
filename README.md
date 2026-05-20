# CoreWeave Demo — Verified Vulnerability Journey (May 2026)

This repository demonstrates an end-to-end **verified vulnerability journey** for
two real-world, publicly disclosed remote code execution flaws:

| CVE | Product | Class | Severity |
|---|---|---|---|
| **CVE-2025-8110** | Gogs ≤ 0.13.3 | Symlink path-traversal RCE (auth required) | High (CVSS 8.7) |
| **CVE-2025-3248** | Langflow < 1.3.0 | Unauthenticated Python `exec` RCE | Critical (CVSS 9.8) |

Both CVEs are listed in CISA's **Known Exploited Vulnerabilities (KEV)** catalog,
have observed in-the-wild exploitation (Supershell C2 / Flodrix botnet), and
predate this demo by months — they are not theoretical.

## What the demo shows

Each CVE walks through three stages:

| Stage | Artifact | Where it lands |
|---|---|---|
| **1. Risk triage** | Calibrated Open FAIR loss exceedance (probability × magnitude) | Comment on the issue + linked HTML report |
| **2. Reachability + exploitability verification** | Reproducible container that triggers the vulnerability deterministically | Coming next |
| **3. Verified patch** | Multi-agent patch generation, scored by the Stage-2 verifier | Coming next |

The pair was chosen specifically because they exercise complementary surfaces:

- **Gogs** is Go, requires authentication, exploits filesystem semantics.
- **Langflow** is Python, fully unauthenticated, exploits language-runtime semantics.

A patch agent (and risk model) that handles both is doing real work.

## Layout

```
.
├── README.md
├── vulnerable/
│   ├── gogs/        # gogs/gogs @ v0.13.3 (last vulnerable commit before v0.13.4 fix)
│   └── langflow/    # langflow-ai/langflow @ 1.2.0 (last vulnerable line before 1.3.0 fix)
```

Both source trees are vendored as squashed git subtrees at their last vulnerable
upstream commits. Upstream license terms are preserved inside each subdirectory.

## Issues

- **#1** — CVE-2025-8110 — Gogs Symlink Path-Traversal RCE
- **#2** — CVE-2025-3248 — Langflow Unauthenticated Python `exec` RCE

Each issue contains the **verbatim original disclosure** from the discovering
researcher (Wiz Research for Gogs; Horizon3.ai for Langflow), followed by
authoritative references (NVD, GHSA, CISA KEV, vendor advisory).

## Acknowledgements

This demo would not exist without the original disclosing researchers:

- **CVE-2025-8110**: Gili Tikochinski and Yaara Shriki at Wiz Research
- **CVE-2025-3248**: Naveen Sunkavally at Horizon3.ai

All source code under `vulnerable/` belongs to its respective upstream project
and is governed by their licenses.
