# System Security Plan (SSP)
## CMMC Level 2 Homelab — CUI Enclave

**Document version:** 1.0
**Date:** July 2026
**Prepared by:** Adam Dye, CISO
**Classification:** Lab / demonstration — no real CUI or FCI is processed, stored, or transmitted.

---

## 1. Purpose and honest scope statement

This System Security Plan documents a **lab environment** built to demonstrate the
design, implementation, and *testing* of security controls drawn from **NIST SP
800-171 Rev 2** (the basis of **CMMC Level 2**), in the context of **DFARS
252.204-7012**.

This is a portfolio and skills-demonstration artifact. It is deliberately honest
about what it is:

- It implements a **meaningful subset** of the 110 Level 2 controls **deeply and
  verifiably**, rather than superficially claiming all 110. Each control marked
  *Implemented* is backed by a testable evidence artifact.
- It is an **on-premises simulation** of control concepts. A production CUI
  environment for a defense contractor typically pairs an on-prem enclave with
  **Microsoft 365 GCC High** for email/collaboration. That delta is stated where
  relevant rather than hidden.
- Where a lab simplification was made (e.g., single-tier CA, ICMP allowed on
  management segments), it is documented as an intentional decision.

The value demonstrated here is not "a perfectly compliant system" but the
**engineering discipline**: designing controls, building them, and — critically —
*testing that they actually enforce*, then remediating the gaps testing revealed
(see the POA&M, which records five such findings).

---

## 2. System identification

| Field | Value |
|---|---|
| System name | CMMC L2 Homelab — CUI Enclave |
| System type | On-premises virtualized enclave |
| Categorization | CUI (simulated) |
| Hypervisor host | HP Z840 (2× Xeon E5-2680 v3, 128 GB, Windows 11 Pro + Hyper-V) |
| CMMC target | Level 2 (NIST SP 800-171 Rev 2) |
| Governing clauses | DFARS 252.204-7012, -7019/-7020, -7021, -7025 |

---

## 3. Authorization boundary and scope

### 3.1 In-scope (CUI enclave)
The CUI boundary is the **`SEG-CUI` network segment (10.10.10.0/24)**, enforced by
the pfSense firewall. Systems in scope:

| System | Role | Address |
|---|---|---|
| FS01 | CUI file server (CUI stored here) | 10.10.10.11 |
| CLIENT-CUI | Enclave workstation (CUI accessed here) | 10.10.10.10 |

### 3.2 Security-relevant shared services (in scope by dependency)
| System | Role | Address | Segment |
|---|---|---|---|
| DC01 | Active Directory, DNS | 10.10.40.10 | SEG-INFRA |
| CA01 | Enterprise Root CA (PKI) | 10.10.40.12 | SEG-INFRA |
| WAZUH01 | SIEM (audit/monitoring) | 10.10.40.20 | SEG-INFRA |
| pfSense | Boundary firewall | gateways | all segments |

> **Scoping note:** DC01 provides authentication into the enclave, so it is a
> security-relevant shared service and in scope. A stricter design would place a
> dedicated DC inside SEG-CUI; this is documented as a lab simplification.

### 3.3 Out of scope
- Home LAN (192.168.1.0/24) and the three physical HP EliteDesk hosts (planned
  out-of-scope roles: admin jump box, general workstation, backup node).
- SEG-MGMT and SEG-INFRA general infrastructure beyond the shared services above.

### 3.4 Network segmentation
| Segment | Subnet | Purpose | Scope |
|---|---|---|---|
| CUI enclave | 10.10.10.0/24 | CUI storage/processing | **In scope** |
| Management | 10.10.30.0/24 | Admin plane (PAW) | Out |
| Infrastructure | 10.10.40.0/24 | Shared services | Security-relevant |
| Home/WAN | 192.168.1.0/24 | Uplink, physical hosts | Out |

All inter-segment traffic is mediated by pfSense under a **deny-by-default** posture
with explicit least-privilege allow rules and logged denies.

---

## 4. Control implementation summary

Status legend: **Implemented** (built + evidence) · **Partial** (built, not fully
tested/scaled) · **Planned** (designed, not yet built).

### AC — Access Control
**Implemented.** Tiered administrative model (Tier 0/1/2) with separate admin
accounts and enforced separation of duties (AC.3.1.1, 3.1.4, 3.1.5). Logon-restriction
GPOs deny cross-tier logon (verified in both directions: Tier 1 admin blocked on
workstations, Tier 2 admin blocked on servers). Machine-admin rights scoped per tier
via GPP. CUI file share restricted to an authorized group (`CUI-Users`) at both the
NTFS and SMB layers; unauthorized access denied and logged.
*Evidence:* `evidence/AC/cui-share-permissions.md`, `evidence/AC/tier-logon-enforcement.md`.

### AU — Audit and Accountability
**Implemented.** Advanced audit policy on servers; object-access (4663) and file-share
(5140/5145) auditing on the CUI share. Centralized collection via Wazuh SIEM (4 agents),
with events tied to named users and mapped to NIST controls. Access allow (jdoe) and
deny (bsmith) both captured as audit records (AU.3.3.1, 3.3.2).
*Evidence:* `evidence/AU/wazuh-siem.md`, `evidence/AC/cui-share-permissions.md`.

### CM — Configuration Management
**Partial.** DISA STIG / Microsoft security baselines applied via GPO; Wazuh Security
Configuration Assessment (SCA) scores agents against CIS benchmarks continuously
(CM.3.4.1, 3.4.2). Full baseline documentation across all hosts is in progress.
*Evidence:* `evidence/AU/wazuh-siem.md` (SCA), `docs/pfsense-and-dc01-setup.md`.

### IA — Identification and Authentication
**Partial → Implemented (PKI foundation).** Enterprise Root CA (SHA-256, 4096-bit)
provides the certificate foundation for cert-based/MFA authentication (IA.3.5.3).
Tiered unique accounts; domain password policy enforcing complexity (IA.3.5.7).
MFA enforcement is designed and pending full rollout.
*Evidence:* `evidence/IA/ca01-verification.md`.

### MP — Media Protection
**Implemented.** CUI stored on a dedicated volume; **BitLocker XTS-AES-256** at rest
on FS01 (C: and the E:\CUI data volume) and CLIENT-CUI (C:), with recovery keys
escrowed to Active Directory (MP.3.8.9 backup/recovery of keys). Removable-media
restriction on enclave hosts designed; session controls applied.
*Evidence:* `evidence/IA/ca01-verification.md` (escrow closure), BUILD.md.

### SC — System and Communications Protection
**Implemented.** pfSense boundary with deny-by-default segmentation (SC.3.13.1,
3.13.2, 3.13.5); enclave egress restricted and logged. **FIPS-mode** cryptography;
**SMB encryption in transit** on the CUI share (SC.3.13.8, 3.13.11). Least-privilege
egress verified on both the enclave and infrastructure segments.
*Evidence:* `evidence/SC/infra-egress-tightening.md`, `evidence/AC/cui-share-permissions.md`.

### SI — System and Information Integrity
**Implemented (core).** Real-time File Integrity Monitoring on the CUI share
(create/modify/delete with content diffs) via Wazuh (SI.3.14.x). Malware protection
and patch management via baseline; vulnerability scanning (OpenVAS) planned.
*Evidence:* `evidence/AU/wazuh-siem.md` (FIM section).

### CA — Security Assessment
**Implemented (as practice).** This SSP, the control matrix, and the POA&M constitute
the assessment artifacts (CA.3.12.1–3.12.4). Continuous monitoring is provided by
Wazuh. Notably, the assessment process itself surfaced and remediated **five control
gaps** through active testing (see POA&M) — evidence of a functioning security
assessment practice rather than assumed compliance.
*Evidence:* `POAM-STATUS.md`, `docs/poam/poam.md`, `assessment/800-171-matrix.md`.

### RA — Risk Assessment
**Planned.** Vulnerability scanning (OpenVAS/Greenbone) and a risk register are
designed and scheduled (RA.3.11.1–3.11.3).

### AT, IR, MA, PS, PE
**Planned / documented.** Policy-level artifacts and procedures (awareness training
records, incident response plan with 72-hour DIBNet reporting workflow, maintenance
logging, personnel/physical statements) are scoped for the documentation phase.

---

## 5. Roles and responsibilities

| Role | Responsibility |
|---|---|
| CISO / builder | System design, control implementation, self-assessment |
| Tier 0 admin (`t0-admin`) | Domain controllers, AD, CA administration only |
| Tier 1 admin (`t1-admin`) | Member server administration |
| Tier 2 admin (`t2-admin`) | Workstation administration |
| Standard user (`jdoe`) | Authorized CUI access |

---

## 6. Assessment posture and honesty statement

This lab does **not** claim a compliant SPRS score. Control implementation is scored
in `assessment/800-171-matrix.md`; the GRC dashboard (`docs/cmmc-grc-dashboard.html`)
presents an **unweighted implementation view** and explicitly notes that a real SPRS
score requires the official DoD weighting (1/3/5-point deductions). Any published
score would be computed against the official methodology.

The intent of this artifact is to demonstrate the **ability to build, operate, and
honestly self-assess** a CUI enclave — including finding and fixing its own gaps —
which is the core competency behind CMMC readiness.

---

## 7. Referenced artifacts

- `BUILD.md` — full build runbook
- `docs/pfsense-and-dc01-setup.md` — as-built network/AD/PKI notes
- `assessment/800-171-matrix.md` — control-by-family implementation matrix
- `docs/poam/poam.md` — Plan of Action & Milestones
- `POAM-STATUS.md` — findings discovered and remediated during the build
- `evidence/` — per-family verification artifacts (AC, AU, IA, SC)
- `docs/cmmc-grc-dashboard.html` — GRC readiness dashboard


