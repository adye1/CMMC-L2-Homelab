# Plan of Action & Milestones (POA&M)
## CMMC Level 2 Homelab — CUI Enclave

**Version:** 1.0 · **Date:** July 2026 · **Owner:** Adam Dye, CISO

A POA&M records identified weaknesses, remediation plans, and status. This document
has two parts: **(A)** gaps discovered *during* the build via active control testing
(all remediated), and **(B)** controls designed but not yet fully implemented.

---

## Part A — Findings discovered and remediated during build

These were surfaced by **testing controls rather than assuming them** — the core of a
real self-assessment. Each was found, root-caused, and fixed within the build.

| ID | Weakness | Control | Discovery | Remediation | Status |
|---|---|---|---|---|---|
| F-01 | Servers tier GPO missing "Deny log on through Terminal Services" — a Tier 0/2 admin could still RDP into servers | AC.3.1.1 | Tier logon test (console blocked but RDP path unverified) | Added RDP deny (Tier0+Tier2) to Servers GPO; re-verified via GPO report | **Closed** |
| F-02 | Share-level denials (Event 5145) not logged — bsmith's denial was unrecorded | AU.3.3.1 | jdoe/bsmith access test | Enabled File Share + Detailed File Share audit subcategories | **Closed** |
| F-03 | BitLocker recovery-escrow GPO empty — CLIENT-CUI key would not escrow to AD | MP.3.8.9 / SC.3.13.16 | CLIENT-CUI encryption | Configured OS + Fixed Data drive escrow settings; escrowed key; verified msFVE object in AD | **Closed** |
| F-04 | Temporary "INFRA → any/any" firewall rule left from troubleshooting | SC.3.13.1 | Build scaffolding review | Replaced with least-privilege egress (53/80/443/123/ICMP) + logged deny-all; verified port-22 now blocked | **Closed** |
| F-05 | Tier admins had no local-admin rights on their tier's machines (t2-admin couldn't administer workstations) | AC.3.1.4 / 3.1.5 | t2-admin elevation attempt | GPP local-group GPOs granting Tier1→servers, Tier2→workstations; verified membership + elevation | **Closed** |

**Cross-cutting lesson (documented for the SSP):** "the GPO applies" and "the GPO
does what was intended" are different checks. F-05 in particular applied cleanly per
`gpresult` but targeted the wrong local group ("Authenticated Users" instead of
"Administrators") — caught only by verifying actual group membership. Verification is
done against *effect*, not *application status*.

---

## Part B — Planned controls (designed, not yet implemented)

Honestly scoped remaining work. Milestone dates are illustrative for the lab.

| ID | Control area | Planned action | Target milestone |
|---|---|---|---|
| P-01 | RA.3.11.2 | Deploy OpenVAS/Greenbone; credentialed scans of all in-scope hosts; findings → POA&M | Next build phase |
| P-02 | IA.3.5.3 | Enforce MFA for privileged and enclave logon (cert/virtual smart card via CA01) | Next build phase |
| P-03 | MP.3.8.9 / CP | Stand up backup node (EliteDesk-BACKUP); perform and document a tested restore | Next build phase |
| P-04 | CM.3.4.1 | Complete STIG/CIS baseline documentation across all hosts; capture SCA scores as evidence | Documentation phase |
| P-05 | IR.3.6.1 | Incident Response plan incl. 72-hr DIBNet reporting workflow; tabletop exercise | Documentation phase |
| P-06 | AT.3.2.1 | Security awareness training records | Documentation phase |
| P-07 | MA / PS / PE | Maintenance logging, personnel-security and physical-protection statements | Documentation phase |
| P-08 | CA.3.12.4 | Compute official SPRS score using DoD Assessment Methodology weighting | After P-01–P-04 |

---

## Notes

- All Part A items are **closed and verified**; see `evidence/` and `POAM-STATUS.md`.
- Part B represents intentional remaining scope, not undiscovered gaps.
- This lab does not assert a compliant SPRS score; see the SSP §6 honesty statement.


