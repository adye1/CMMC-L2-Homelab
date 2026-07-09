# CMMC Level 2 Homelab — Build Runbook (Option A: all-virtual enclave, no managed switch)

Step-by-step instructions to stand up a Windows-based lab that simulates a
mid-tier defense contractor subject to **DFARS 252.204-7012** and **CMMC Level 2**
(NIST SP 800-171 Rev 2, 110 controls), using only the equipment on hand — **no
managed switch required.**

- **HP Z840** — 2× Xeon E5-2680 v3 (24C/48T), 128 GB RAM, 4× 1 TB SSD, Quadro M4000 → Hyper-V host
- **3× HP EliteDesk 800 G2** — 16 GB / 2 TB → out-of-scope physical roles (admin, backup, general)
- **Home LAN** — `192.168.1.0/24` (unchanged)

> **Regulatory context (mid-2026):** The 48 CFR acquisition rule is in effect.
> Phase 1 (self-assessment) runs through 9 Nov 2026; **Phase 2 (C3PAO certification)
> begins 10 Nov 2026**. In solicitations, **DFARS 252.204-7025** states your required
> level, **252.204-7021** requires you to maintain it, and **7012** carries the
> 800-171 + 72-hour incident-reporting baseline.

> **This is a lab.** No real CUI or FCI is ever stored here. Use your own or
> evaluation Windows licenses. Only scan hosts you own.

---

## 0. Networking model — read this first

Because there is no managed switch, all network segmentation is done **inside
Hyper-V** using virtual switches, with the pfSense VM as the only router between
segments:

- **One External vSwitch** is bound to the Z840's physical NIC. That physical wire
  is your existing `192.168.1.0/24` home LAN — the Z840 host, the three physical
  EliteDesks, and your home devices all live here. It is also pfSense's **WAN**.
- **Three Internal vSwitches** (`SEG-CUI`, `SEG-MGMT`, `SEG-INFRA`) are virtual-only
  networks with no physical port. Lab VMs attach here. pfSense has a leg on each
  and firewalls between them.

Two consequences to accept:

1. **The enclave is on a different range than `192.168.1.0/24`.** pfSense's WAN is
   already `192.168.1.x`, so the segments behind it use `10.10.x.0/24` to avoid an
   overlap. Your home LAN is untouched.
2. **Physical machines can't join the enclave at L2.** Internal vSwitches have no
   physical port, so the in-scope CUI endpoint is a **VM**. The physical EliteDesks
   sit on the home LAN in out-of-scope roles and reach lab services only through
   explicit pfSense allow rules.

This is a valid, defensible scoping design: the CUI boundary is the `SEG-CUI`
interface on pfSense, and nothing enters or leaves it except through the firewall.

---

## 1. Prerequisites

### 1.1 Software / downloads
- Windows Server 2022 (or 2025) ISO — evaluation is fine
- Windows 11 Pro ISO
- pfSense CE (or OPNsense) ISO
- Ubuntu Server 22.04+ LTS ISO (for Wazuh)
- Kali Linux ISO or Greenbone Community container (for OpenVAS/GVM)
- Microsoft Security Compliance Toolkit + **LGPO.exe**
- DISA STIG GPO package and SCAP tool — from `public.cyber.mil`
- Sysinternals Suite (Sysmon)
- Wazuh install script — from `wazuh.com` (use the current version)
- Veeam Community Edition **or** TrueNAS CORE (backup node)

### 1.2 Hardware prep
- **No managed switch needed.** Segmentation is virtual (Section 0).
- Confirm VT-x/VT-d enabled in Z840 BIOS; update BIOS/firmware.
- Confirm TPM 2.0 enabled on the EliteDesks (for BitLocker).

### 1.3 IP plan

**Home LAN — `192.168.1.0/24` (physical, unchanged):**

| Host | IP | Role |
|---|---|---|
| Home router (gateway) | 192.168.1.1 | Internet gateway / pfSense WAN upstream |
| Z840 host (mgmt OS) | 192.168.1.10 | Hyper-V host |
| pfSense WAN | 192.168.1.20 | Firewall WAN leg |
| EliteDesk-ADMIN | 192.168.1.50 | Physical admin / jump box (out of scope) |
| EliteDesk-BACKUP | 192.168.1.51 | Backup & recovery node (out of scope) |
| EliteDesk-GEN | 192.168.1.52 | Domain-joined general workstation (out of scope) |

**Lab segments (virtual, behind pfSense):**

| Segment | vSwitch | Subnet | Gateway | Members |
|---|---|---|---|---|
| **CUI enclave** (in scope) | SEG-CUI | 10.10.10.0/24 | 10.10.10.1 | FS01 `.11`, CLIENT-CUI `.10` |
| Management | SEG-MGMT | 10.10.30.0/24 | 10.10.30.1 | PAW01 `.10`, GVM01 `.30` |
| Infrastructure | SEG-INFRA | 10.10.40.0/24 | 10.10.40.1 | DC01 `.10`, WAZUH01 `.20` |

---

## 2. Architecture at a glance

**Z840 (Hyper-V VMs):** pfSense · DC01 · FS01 · WAZUH01 · GVM01 · PAW01 · CLIENT-CUI
**EliteDesks (physical, home LAN):** admin jump box · backup node · general workstation
**Boundary:** pfSense routes between SEG-CUI / SEG-MGMT / SEG-INFRA and WAN, deny-by-default, IDS on, everything logged to Wazuh.

---

## Phase 1 — Hyper-V host + virtual switches

**Goal:** virtualization host with the four-switch network.

1. Install Windows 11 Pro (or Server) on the Z840; one SSD for the host OS.
2. Pool the other 3 SSDs for VM storage (Storage Spaces mirror for the CUI volume
   to demonstrate media-protection thinking).
3. Enable Hyper-V:
   - Win 11 Pro: `Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All`
   - Server: `Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart`
4. Create one External switch (home LAN / WAN) and three Internal switches:
   ```powershell
   New-VMSwitch -Name "WAN" -NetAdapterName "Ethernet" -AllowManagementOS $true
   New-VMSwitch -Name "SEG-CUI"   -SwitchType Internal
   New-VMSwitch -Name "SEG-MGMT"  -SwitchType Internal
   New-VMSwitch -Name "SEG-INFRA" -SwitchType Internal
   ```
5. Attach each VM to its segment (no VLAN tagging). pfSense gets a vNIC on every
   switch so it can route/firewall between them:
   ```powershell
   Add-VMNetworkAdapter -VMName pfSense -SwitchName "SEG-CUI"
   Add-VMNetworkAdapter -VMName pfSense -SwitchName "SEG-MGMT"
   Add-VMNetworkAdapter -VMName pfSense -SwitchName "SEG-INFRA"
   Connect-VMNetworkAdapter -VMName DC01 -SwitchName "SEG-INFRA"
   ```

**Validate:** `Get-VMSwitch` lists WAN + three SEG switches; VMs see their vNIC.
**Evidence (CM):** switch configuration screenshot → `evidence/CM/`.

---

## Phase 2 — pfSense boundary firewall

**Goal:** inter-segment routing, deny-by-default, IDS.

1. Create the pfSense VM (2 vCPU, 2 GB) with **four** vNICs: WAN (External switch)
   and one each on SEG-CUI, SEG-MGMT, SEG-INFRA.
2. Install pfSense. Assign WAN (`192.168.1.20`, gateway `192.168.1.1`), then assign
   the three internal interfaces and give each its gateway address (`10.10.10.1`,
   `10.10.30.1`, `10.10.40.1`).
3. Enable a **DHCP** scope per internal segment (or use statics per the IP plan).
4. **Firewall rules — remove any allow-all, then add explicit allows.** Representative
   minimum set (deny-by-default, log denies to Wazuh):

   | From | To | Allow | Purpose |
   |---|---|---|---|
   | SEG-CUI | SEG-INFRA (DC01) | 53, 88, 135, 389, 445, 636 | AD/DNS auth (shared service) |
   | SEG-CUI | SEG-INFRA (WAZUH01) | 1514, 1515 | SIEM agent |
   | SEG-CUI | WAN | (none, or updates only) | Stricter enclave egress |
   | SEG-MGMT | any | 3389, 443, 5985, 5986 | Admin plane |
   | SEG-INFRA | SEG-CUI | established/related | Return traffic only |
   | Home `192.168.1.50` | SEG-MGMT | 3389, 443 | Admin jump box in |
   | Home `192.168.1.52` | SEG-INFRA (DC01) | AD/DNS ports | General workstation domain-join |
   | Home `192.168.1.5x` | SEG-INFRA (WAZUH01) | 1514, 1515 | Agent on physical hosts |
   | SEG-INFRA/CUI | Home `192.168.1.51` | backup ports | Backups to EliteDesk-BACKUP |
   | any | SEG-CUI | **deny (default)** | No path into the enclave |

5. Install **Suricata** (or Snort) on WAN and the inter-segment interfaces.
6. Point pfSense syslog at WAZUH01.

**Validate:** the general workstation **cannot** reach the CUI enclave; the admin
box can reach SEG-MGMT; VMs reach the internet via WAN.
**Evidence (SC / AC):** rule export + a blocked-connection log line → `evidence/SC/`.

> **Scoping note:** AD is a shared service reaching into the enclave, so DC01 is
> in scope for the assessment even though it lives on SEG-INFRA. Document this in
> the SSP. A stricter design places a dedicated DC inside SEG-CUI.

---

## Phase 3 — DC01 (Active Directory, DNS, PKI)

**Goal:** identity foundation with certificate services.

1. Create DC01 (Server 2022, 2 vCPU, 6 GB) on SEG-INFRA, static `10.10.40.10`,
   gateway `10.10.40.1`, DNS = self.
2. Promote to a domain controller (choose your own domain; `.local` is fine for a lab):
   ```powershell
   Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
   Install-ADDSForest -DomainName "dclab.local" -InstallDns
   ```
3. Create OUs: `Servers`, `Workstations`, `Enclave`, `Admins`, `Users`.
4. Create **tiered** admin accounts (separate day-to-day vs. Domain Admin).
5. Install the enterprise CA (cert-based auth, FIPS crypto):
   ```powershell
   Install-WindowsFeature AD-Certificate -IncludeManagementTools
   Install-AdcsCertificationAuthority -CAType EnterpriseRootCA
   ```
6. Configure NTP against a reliable source (audit timestamps depend on it).

**Validate:** `dcdiag` clean; a client joins the domain; CA issues a test cert.
**Evidence (IA / AC):** OU/tiering, CA config → `evidence/IA/`, `evidence/AC/`.

---

## Phase 4 — Hardening baselines (GPO / STIG / FIPS / audit)

**Goal:** enforce configuration baselines and the central audit trail.

1. Import the **DISA STIG GPO** package (`public.cyber.mil`) or Microsoft Security
   Baselines via `LGPO.exe`; link to the relevant OUs.
2. Enforce an **advanced audit policy** GPO (logon/logoff, object access, privilege
   use, policy change, account management) — feeds AU.
3. Enable **FIPS mode**: *System cryptography: Use FIPS compliant algorithms* (SC).
4. Configure **BitLocker** via GPO (TPM + startup PIN) for workstations (MP/SC).
5. Set account lockout, password policy, session lock/timeout (AC/IA).
6. Restrict **removable media** on enclave machines (MP).
7. Deploy **Sysmon** by GPO for high-fidelity telemetry.

**Validate:** run the DoD **SCAP** tool / a STIG checklist against a client and
capture the baseline compliance percentage.
**Evidence (CM / AU / MP / SC):** SCAP/STIG report, GPO backups → per family.

---

## Phase 5 — FS01 (CUI file server, in the enclave)

**Goal:** a controlled CUI repository inside SEG-CUI.

1. Create FS01 (Server 2022, 2 vCPU, 4 GB) on **SEG-CUI**, static `10.10.10.11`,
   gateway `10.10.10.1`, DNS = `10.10.40.10`; domain-join.
2. Create a `CUI` share on the Storage Spaces mirror; NTFS ACLs to a single
   `CUI-Users` group (deny broad access).
3. Enable **Dynamic Access Control** / file classification; tag files `CUI` with a
   simulated marking.
4. Turn on **object-access auditing** on the share (AU).
5. Confirm pfSense permits only SEG-CUI hosts to reach the share.

**Validate:** the enclave client reads the share; a general-zone host is denied at
both the network and NTFS layers.
**Evidence (AC / MP / AU):** ACLs, classification, access-denied audit event.

---

## Phase 6 — CLIENT-CUI (in-scope enclave endpoint, virtual)

**Goal:** the in-scope user workstation — a VM, since no physical box can join the enclave.

1. Create CLIENT-CUI (Win 11 Pro, 2 vCPU, 4 GB) on **SEG-CUI**, static `10.10.10.10`;
   domain-join.
2. Apply the enclave baseline: BitLocker on, removable media blocked, session lock,
   Wazuh agent, no split path to the home LAN.

**Validate:** reaches the CUI share; cannot reach the home LAN except allowed return
traffic; appears in Wazuh.
**Evidence (CM / SI):** baseline status, agent enrollment.

---

## Phase 7 — PAW (Privileged Access Workstation)

**Goal:** a hardened admin plane separate from user workstations.

1. Create PAW01 (Win 11 Pro, 2 vCPU, 4 GB) on **SEG-MGMT**, static `10.10.30.10`;
   domain-join.
2. Apply the strictest baseline; block internet and email.
3. Administer DCs/servers **only** from here (pfSense limits admin protocols to
   SEG-MGMT sources).
4. Require MFA for admin logon.

**Validate:** admin protocols to servers succeed from PAW01 / the admin jump box and
are blocked elsewhere.
**Evidence (AC / IA):** rule + admin session logs.

---

## Phase 8 — Wazuh SIEM

**Goal:** centralized audit, monitoring, FIM, compliance dashboard.

1. Create WAZUH01 (Ubuntu Server, 4 vCPU, 12 GB) on **SEG-INFRA**, static `10.10.40.20`.
2. Install the single-node stack using the current official installer from `wazuh.com`.
3. Deploy the **Wazuh agent** to DC01, FS01, PAW01, CLIENT-CUI, and the physical
   EliteDesks; point them at `10.10.40.20` (physical hosts use the home→INFRA allow).
4. Enable **FIM** on the CUI share and system dirs (SI/AU), **SCA** (CM), and
   **vulnerability detection** (RA).
5. Open the built-in **NIST 800-53** compliance dashboard.
6. **Extend toward 800-171:** add custom rules tagged with the control ID in the
   rule's `<group>`, so your detections appear in the compliance view. Keep these in
   `scripts/siem/`.
7. Forward pfSense syslog into Wazuh so boundary events correlate.

**Validate:** a test failed-logon on DC01 appears tagged to a control; FIM fires on
a CUI-share edit.
**Evidence (AU / SI / CA):** dashboard export, tagged alerts → per family.

---

## Phase 9 — Vulnerability scanner (OpenVAS / Greenbone CE)

**Goal:** recurring authenticated vulnerability scanning (RA).

> Tenable changed the free tier — Nessus Essentials is now a 30-day / 5-IP
> evaluation with no data retention, so it doesn't fit a persistent multi-host lab.
> Use **Greenbone Community Edition (OpenVAS)**. Neither free scanner includes STIG
> *compliance-audit* templates; that config compliance comes from Phase 4 + Wazuh
> SCA. STIG audit templates are a Nessus Professional paid feature if wanted later.

1. Create GVM01 (Kali or Ubuntu, 2 vCPU, 6 GB) on **SEG-MGMT**, static `10.10.30.30`.
2. Install GVM (Kali): `sudo apt update && sudo apt install gvm && sudo gvm-setup`,
   then `sudo gvm-check-setup`; wait for the feed sync.
3. Create a **credentialed** scan using a dedicated least-privilege AD service
   account (unauthenticated scans miss most Windows findings).
4. pfSense already allows SEG-MGMT → other segments; scan the enclave, infra, and
   the physical hosts.
5. Schedule weekly scans; export reports to `evidence/RA/`; turn findings into POA&M rows.

**Validate:** a scan of DC01/FS01 returns a findings report with severities.
**Evidence (RA / SI):** scan report + a POA&M entry.

---

## Phase 10 — Physical EliteDesk roles (home LAN, out of scope)

**Goal:** put the physical fleet to work without a managed switch.

1. **EliteDesk-ADMIN (`192.168.1.50`)** — physical admin / jump box. RDP into PAW01
   and consoles through the home→SEG-MGMT allow rule. Where you operate the lab.
2. **EliteDesk-GEN (`192.168.1.52`)** — Win 11 Pro, domain-joined general workstation
   (out of scope). Reaches DC01 via the home→SEG-INFRA AD allow. Demonstrates
   managing physical hardware by GPO and Wazuh agent.
3. **EliteDesk-BACKUP (`192.168.1.51`)** — backup/recovery node (Phase 11).

**Validate:** admin box reaches SEG-MGMT; general workstation domain-joins and pulls
GPOs; neither can reach SEG-CUI.
**Evidence (CM / AC):** agent enrollment, baseline status, blocked-to-enclave log.

---

## Phase 11 — Backup & recovery

**Goal:** demonstrable backup and a tested restore (MP 3.8.9 / contingency).

1. On **EliteDesk-BACKUP**, install Veeam Community Edition (or TrueNAS as an
   SMB/iSCSI target).
2. Allow SEG-INFRA/SEG-CUI → `192.168.1.51` backup ports in pfSense.
3. Schedule backups of DC01, FS01, and the CUI volume.
4. **Test a restore** — recover a file and a full VM; document the result.

**Validate:** restored file opens; restored VM boots.
**Evidence (MP / recovery):** job config + restore-test writeup.

---

## Phase 12 — GRC dashboard

**Goal:** an at-a-glance compliance view for the portfolio.

1. Deploy `docs/cmmc-grc-dashboard.html`; update its `DATA` object as controls move
   to met/partial; serve via **GitHub Pages** from `docs/`.
2. Use Wazuh's compliance dashboard as the **live monitoring** evidence layer.
3. *(Optional)* Stand up **Eramba Community Edition** as a VM on SEG-INFRA for a full
   GRC platform (risk register, policy lifecycle, control catalog).

**Validate:** dashboard reflects current numbers; Pages URL loads.
**Evidence (CA):** dashboard URL/screenshot → `evidence/sprs/`.

---

## Phase 13 — Self-assessment, SSP, POA&M, SPRS

**Goal:** turn the lab into assessment-grade documentation.

1. Run `scripts/audit/` checks and the STIG/SCAP reports; complete
   `assessment/800-171-matrix.md` per control (met / partial / planned + evidence link).
2. Write the **System Security Plan** (`docs/ssp/`) — one section per control,
   describing implementation and pointing to evidence. Include the AD shared-service
   scoping note from Phase 2.
3. Log every gap in the **POA&M** (`docs/poam/`) with owner, milestone, date.
4. Compute your **SPRS score** with the DoD Assessment Methodology (start 110,
   subtract 1/3/5 points per open control per the official weighting). Record it in
   `evidence/sprs/`. Confirm weights against the official scoring guide before
   publishing a number — the dashboard's figure is an unweighted floor.

**Validate:** every "met" control has a matching evidence artifact.
**Evidence (CA):** SSP, POA&M, scored matrix.

---

## Phase 14 — GitHub publishing

**Goal:** a clean, sanitized portfolio repo.

1. Initialize and push:
   ```bash
   git init && git add . && git commit -m "CMMC L2 homelab" && git branch -M main
   git remote add origin <your-repo-url> && git push -u origin main
   ```
2. **Sanitize first:** no credentials or secrets; the `10.10.x` and `192.168.1.x`
   lab addresses are fine to publish. Add a `.gitignore` for keys, `.pfx`, and any
   config containing passwords.
3. Enable **GitHub Pages** on `docs/` to publish the GRC dashboard.
4. Keep `README.md` as the front door; link BUILD.md, the matrix, and the dashboard.

**Validate:** repo builds, Pages renders, no secrets in history.

---

## Appendix A — Control-family evidence map

| Family | Built in phase | Primary evidence |
|---|---|---|
| AC – Access Control | 3,5,7 | OU tiering, share ACLs, PAW rules |
| AT – Awareness & Training | 13 | Training register |
| AU – Audit & Accountability | 4,8 | Audit GPO, Wazuh alerts, time sync |
| CM – Configuration Management | 4,8,10 | STIG/SCAP report, SCA, GPO backups |
| IA – Identification & Auth | 3,4,7 | MFA, CA/PKI, password policy |
| IR – Incident Response | 13 | IR plan, tabletop, 72-hr reporting flow |
| MA – Maintenance | 11,13 | Maintenance log |
| MP – Media Protection | 4,5,11 | BitLocker, media control, restore test |
| PS – Personnel Security | 13 | On/offboarding SOP |
| PE – Physical Protection | 10,13 | Physical control statement |
| RA – Risk Assessment | 9 | OpenVAS reports, risk register |
| CA – Security Assessment | 12,13 | SSP, POA&M, dashboard |
| SC – System & Comms Protection | 1,2,4 | Segments, firewall rules, FIPS, IDS |
| SI – System & Info Integrity | 8,9 | FIM alerts, patch/vuln reports |

## Appendix B — Validation checklist

- [ ] Four vSwitches present; VMs on correct segments
- [ ] pfSense enforces CUI isolation (general→enclave denied at network + NTFS)
- [ ] `dcdiag` clean; clients domain-joined; CA issues certs
- [ ] STIG/SCAP baseline captured for a server and a workstation
- [ ] FIPS mode on; BitLocker enforced on workstations
- [ ] Wazuh receiving from all agents + pfSense; FIM and a tagged alert verified
- [ ] OpenVAS credentialed scan produces a findings report
- [ ] Backup restore of a file **and** a full VM tested and documented
- [ ] SPRS score computed with official weights and recorded
- [ ] SSP + POA&M complete; every "met" control has evidence
- [ ] Repo sanitized; Pages dashboard live

## Appendix C — Suggested build order & time

1. Phases 1–2 (host, vSwitches, firewall) — weekend 1
2. Phases 3–6 (AD/PKI, hardening, file server, enclave client) — weekend 2
3. Phases 7–11 (PAW, SIEM, scanner, physical roles, backup) — weekend 3
4. Phases 12–14 (dashboard, docs, publish) — weekend 4

---

## Disclaimer

Educational lab. On-prem simulation of control concepts; a production CUI
environment typically uses Microsoft 365 GCC High plus a compliant enclave. Not
legal or compliance advice — confirm scope and assessment specifics with a CMMC
Registered Practitioner or qualified federal-contracts counsel.

