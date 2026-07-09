# pfSense + DC01 Build Log (Option A, no managed switch)

As-built checklist for the boundary firewall and the first domain controller,
including the gotchas hit during the build. Segments are Hyper-V internal
switches; pfSense routes between them. Home LAN stays `192.168.1.0/24`.

## Segment / IP reference

| Segment | vSwitch | pfSense IF | Subnet | Gateway |
|---|---|---|---|---|
| Management | SEG-MGMT | LAN | 10.10.30.0/24 | 10.10.30.1 |
| CUI enclave | SEG-CUI | OPT1 → "CUI" | 10.10.10.0/24 | 10.10.10.1 |
| Infrastructure | SEG-INFRA | OPT2 → "INFRA" | 10.10.40.0/24 | 10.10.40.1 |
| WAN | WAN (External) | WAN | 192.168.1.0/24 | 192.168.1.1 |

Z840 host has a vNIC on SEG-MGMT at `10.10.30.5` for web-UI access.

## pfSense build (Gen 1 VM)

- [ ] Gen 1 VM, 2 GB static RAM, 2 vCPU, 20 GB disk, first NIC = WAN
- [ ] Add 3 more NICs: SEG-CUI, SEG-MGMT, SEG-INFRA
- [ ] Record each vNIC MAC (`Get-VMNetworkAdapter -VMName pfSense`) before install
- [ ] Install pfSense → Auto (UFS) BIOS → reboot → eject ISO
- [ ] At console: VLANs? **n**
- [ ] Assign by MAC: WAN, LAN=SEG-MGMT, OPT1=SEG-CUI, OPT2=SEG-INFRA
- [ ] Set IPs (console option 2): WAN 192.168.1.20/24 gw .1; LAN 10.10.30.1/24;
      OPT1 10.10.10.1/24; OPT2 10.10.40.1/24 (no gateway on internal legs)
- [ ] Host: `New-NetIPAddress -InterfaceAlias "vEthernet (SEG-MGMT)" -IPAddress 10.10.30.5 -PrefixLength 24`
- [ ] Browse https://10.10.30.1, admin/pfsense, run wizard, set strong password
- [ ] WAN wizard: **uncheck** "Block RFC1918" (WAN is a private LAN)
- [ ] Rename OPT1→CUI, OPT2→INFRA (Interfaces → Assignments)
- [ ] DHCP per segment, pool .100–.150 (Services → DHCP Server)

## GOTCHA — INFRA "no connectivity" despite everything looking right

Symptom: DC01 (10.10.40.10) could not ping/reach gateway 10.10.40.1. Interface
"up", DC01 on correct switch, IP config correct, MACs matched (no swap).

Root cause: the **INFRA firewall pass rule was malformed / not applied**. The
tell was **Status → Interfaces → INFRA** showing a climbing **block** counter
(e.g. 898 blocked in) while pass stayed near zero — proof traffic arrived and was
dropped by the firewall, not lost on the network.

Fix: **Firewall → Rules → INFRA** → delete existing rule(s) → Add (↑) →
Action Pass, IPv4, **Protocol Any**, **Source Any**, Dest Any → Save →
**Apply Changes**. Ping replied immediately.

Debug order that works: (1) confirm switch/IP on both ends, (2) check
**Status → Interfaces** pass/block counters, (3) rebuild the rule Any/Any and
**Apply**. ICMP timeouts alone are misleading — Windows blocks inbound ping, so
use `Test-NetConnection <ip> -Port 443` against pfSense instead.

> ⚠️ **INFRA Any/Any is BUILD-TIME SCAFFOLDING — remove at hardening.** Replace
> with least-privilege rules (AD/DNS to enclave, SIEM to Wazuh, updates out).
> A wide-open segment rule left in place is an easy assessment finding.

## DC01 promotion (Gen 2 VM)

- [ ] Gen 2 VM, 6 GB, 2 vCPU, 80 GB, Server 2022 Desktop Experience, on SEG-INFRA
- [ ] Static IP 10.10.40.10/24, gw 10.10.40.1, DNS 10.10.40.1 (temporary)
- [ ] Rename to DC01, reboot
- [ ] `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools`
- [ ] `Install-ADDSForest -DomainName dclab.local -DomainNetbiosName DCLAB ...`
      → set DSRM password (record it — separate from Administrator), auto-reboot
- [ ] Log in as **DCLAB\Administrator**
- [ ] `Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 127.0.0.1`
- [ ] `Add-DnsServerForwarder -IPAddress 8.8.8.8` (and 1.1.1.1)
- [ ] Verify: `Get-ADDomain`, `Resolve-DnsName microsoft.com`,
      `dcdiag /test:Services /test:Replications /test:Advertising /test:FSMO`

### Note on dcdiag /q noise (safe to ignore on a fresh DC)
- **DFSREvent** — SYSVOL initialization events from during promotion; clears once
  they age out of the 24h window. Confirm with `Get-SmbShare SYSVOL,NETLOGON`.
- **SystemLog** — unrelated noise: `netprofm` blip during the promotion reboot,
  and the "Printer Extensions marked interactive" boilerplate. Neither is AD.
- Trust the targeted tests (Services/Replications/Advertising/FSMO) over `/q`.

## Status: COMPLETE & AD-published (certutil -ping successful)
pfSense boundary up; DC01 = healthy DC for dclab.local with working internal +
external DNS. Next: OU structure, tiered admin accounts, enterprise CA (AD CS).

---

# CA01 Build — Enterprise Root CA (AD CS)

Tier 0 asset. Dedicated VM on SEG-INFRA, separate from DC01 to keep the Tier 0
trust boundary clean. Single-tier Enterprise Root CA (lab choice; production
would use an offline root + online subordinate — note this in the SSP).

## Addressing
- CA01 — SEG-INFRA, static **10.10.40.12**, gw 10.10.40.1, **DNS 10.10.40.10 (DC01)**

## Build checklist
- [ ] Gen 2 VM, 4 GB, 2 vCPU, 80 GB, Server 2022 Desktop Experience, on SEG-INFRA
- [ ] Static IP 10.10.40.12/24, gw 10.10.40.1
- [ ] **DNS = 10.10.40.10 (DC01), NOT pfSense** — required to find the domain
- [ ] Verify before join: `Resolve-DnsName dclab.local` returns 10.10.40.10
- [ ] Rename + domain-join with `DCLAB\t0-admin` into OU=Servers
- [ ] Install role: `Install-WindowsFeature AD-Certificate -IncludeManagementTools`
- [ ] Configure CA (see command below)
- [ ] Verify: CertSvc Running, `certutil -CAInfo` = Enterprise Root CA, CRL Valid
- [ ] Confirm AD publication (Enrollment Services container, on DC01)

## CA configuration (FIPS-appropriate: SHA-256, 4096-bit)
```powershell
Install-AdcsCertificationAuthority `
  -CAType EnterpriseRootCA -CACommonName "DCLAB-CA01-CA" `
  -KeyLength 4096 -HashAlgorithmName SHA256 `
  -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
  -ValidityPeriod Years -ValidityPeriodUnits 10 -Force
```

## GOTCHA 1 — DNS must point at DC01 before domain join
Fresh member server may still have pfSense (10.10.40.1) or blank DNS, causing
`dclab.local : DNS record does not exist`. Fix:
`Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 10.10.40.10`
then `Clear-DnsClientCache`. Confirm with `Test-NetConnection 10.10.40.10 -Port 53`
and `Resolve-DnsName dclab.local`.

## GOTCHA 2 — Rename + join in one shot can skip the rename
`Rename-Computer` + `Add-Computer -Restart` together sometimes joins but leaves
the default `WIN-xxxxx` name. **Critical for a CA** — the name is baked into every
issued cert and `DNS Name` in `certutil -CAInfo`; renaming after CA install is
very painful. Fix BEFORE installing AD CS:
`Rename-Computer -NewName CA01 -DomainCredential (Get-Credential DCLAB\t0-admin) -Restart`
Verify `Get-CimInstance Win32_ComputerSystem` shows Name = CA01 first.

Note: AD cmdlets (Get-ADComputer etc.) don't exist on member servers by default —
run them on DC01, or `Install-WindowsFeature RSAT-AD-PowerShell` if needed.

## Evidence produced
- IA (3.5.3 cert-based auth foundation): CA config, `certutil -CAInfo` output
- SC (FIPS crypto): SHA-256 / 4096-bit settings → evidence/SC/, evidence/IA/

## Status: COMPLETE & AD-published (certutil -ping successful)
Tier 0 core done: DC01 (AD DS + DNS) + CA01 (Enterprise Root CA), tier-separated
on SEG-INFRA. Next: FS01 (CUI file server) → tiering GPO proof-test.


