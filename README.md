# CMMC Level 2 Homelab — CUI Enclave

A Windows-based lab that designs, builds, and **verifiably tests** a Controlled
Unclassified Information (CUI) enclave against **NIST SP 800-171 Rev 2** — the basis
of **CMMC Level 2** — in the context of **DFARS 252.204-7012**.

Built end-to-end on a single Hyper-V host: a segmented network, Active Directory, an
enterprise PKI, a tiered administrative model, an encrypted-and-audited CUI file
share, and a centralized SIEM — every control backed by a **tested** evidence
artifact, not just a checkbox.

> **This is a lab / portfolio project.** No real CUI or FCI is processed. It is an
> on-prem simulation of control *concepts*; a production CUI environment typically
> pairs an enclave with Microsoft 365 GCC High. See the
> [SSP](docs/ssp/system-security-plan.md) for the full honesty statement.

---

## What I Wanted to Do

- **Controls are tested, not assumed.** Every implemented control has a verification
  artifact showing it actually enforces (e.g., an *authorized* user reading the CUI
  share and an *unauthorized* user being denied — both captured as audit records).
- **The self-assessment found and fixed its own gaps.** Testing surfaced **five
  real misconfigurations** — a GPO that applied but targeted the wrong group, an
  audit subcategory that silently wasn't logging denials, an empty escrow policy —
  all root-caused and remediated. See [POA&M](docs/poam/poam.md). *A build with zero
  findings usually means nobody tested.*
- **It's honest about scope.** It implements a subset of the 110 controls *deeply*
  rather than claiming all 110 superficially, and documents every lab simplification.

---

## What's built

| Domain | Implementation |
|---|---|
| **Network** | pfSense boundary, deny-by-default segmentation (CUI / mgmt / infra), least-privilege egress, all denies logged |
| **Identity** | Active Directory + DNS; enterprise Root CA (SHA-256, 4096-bit) for cert-based auth |
| **Access control** | Tier 0/1/2 admin model — cross-tier logon denied *and* machine-admin rights scoped per tier, verified both directions |
| **CUI data** | Dedicated volume; least-privilege NTFS + SMB ACLs; access allow/deny tested with named users |
| **Encryption** | BitLocker XTS-AES-256 at rest (keys escrowed to AD) + SMB encryption in transit |
| **Audit & monitoring** | Wazuh SIEM (4 agents), real-time File Integrity Monitoring on CUI, pfSense boundary syslog, NIST-tagged alerts |

---

## Architecture

```
                    Internet
                       │
                 [ Home Router ] 192.168.1.0/24  ── physical hosts (out of scope)
                       │
                   [ pfSense ]  deny-by-default boundary
        ┌──────────────┼───────────────┐
   SEG-CUI          SEG-MGMT         SEG-INFRA
 10.10.10.0/24    10.10.30.0/24    10.10.40.0/24
   ┌──────┐         ┌──────┐      ┌──────┬──────┬────────┐
   │FS01  │         │PAW01 │      │DC01  │CA01  │WAZUH01 │
   │CLIENT│         └──────┘      │(AD)  │(PKI) │(SIEM)  │
   │-CUI  │                       └──────┴──────┴────────┘
   └──────┘
  IN SCOPE                          security-relevant shared services
```

---

## Repo map

| Path | What it is |
|---|---|
| [`docs/ssp/system-security-plan.md`](docs/ssp/system-security-plan.md) | System Security Plan — the control-by-control story |
| [`docs/poam/poam.md`](docs/poam/poam.md) | Plan of Action & Milestones (findings + planned work) |
| [`POAM-STATUS.md`](POAM-STATUS.md) | Quick view of the 5 findings found & fixed during the build |
| [`assessment/800-171-matrix.md`](assessment/800-171-matrix.md) | Control-by-family implementation matrix |
| [`evidence/`](evidence/) | Per-family verification artifacts (AC, AU, IA, SC) |
| [`BUILD.md`](BUILD.md) | Full 14-phase build runbook |
| [`docs/pfsense-and-dc01-setup.md`](docs/pfsense-and-dc01-setup.md) | As-built network/AD/PKI notes + gotchas |
| [`docs/cmmc-grc-dashboard.html`](docs/cmmc-grc-dashboard.html) | Interactive GRC readiness dashboard (GitHub Pages–ready) |

---

## Hardware

- **HP Z840** — 2× Xeon E5-2680 v3 (24C/48T), 128 GB RAM — Hyper-V host running the enclave
- **3× HP EliteDesk 800 G2** — out-of-scope physical roles (planned)

---

## Regulatory context

The 48 CFR CMMC acquisition rule is in effect; **Phase 2 (C3PAO certification for
Level 2) began 10 Nov 2026**. DFARS **252.204-7025** states a solicitation's required
level, **-7021** requires maintaining it, and **-7012** carries the underlying
800-171 + 72-hour incident-reporting baseline.

---

## Disclaimer

Educational project. Not legal or compliance advice. No real CUI/FCI. Confirm any
real-world scoping or assessment questions with a CMMC Registered Practitioner or
qualified federal-contracts counsel.

