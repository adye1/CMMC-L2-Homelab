# Tiered Admin — Logon Restriction Enforcement Evidence

Control mapping: AC 3.1.1 (authorized users), AC 3.1.4 (separation of duties),
AC 3.1.5 (least privilege)

## Tier model (Microsoft Enterprise Access model)
Higher-tier credentials never log on to lower-tier machines.

| Machine type | Tier | Deny logon to | Admins who belong |
|---|---|---|---|
| DC / CA (DC01, CA01) | Tier 0 | (none) | t0-admin |
| Member servers (FS01) | Tier 1 | Tier 0, Tier 2 | t1-admin |
| Workstations (CLIENT-CUI) | Tier 2 | Tier 0, Tier 1 | t2-admin |

## GPOs
- "Tier Logon Restrictions - Workstations" -> linked to OU=Workstations and OU=Enclave
  Deny log on locally + Deny log on through Terminal Services = Tier0-Admins, Tier1-Admins
- "Tier Logon Restrictions - Servers" -> linked to OU=Servers
  Deny = Tier0-Admins, Tier2-Admins  (VERIFY - see POA&M note)

## Test results on CLIENT-CUI (Tier 2 workstation, OU=Enclave)
- DCLAB\t2-admin  -> LOGON ALLOWED  (correct: Tier 2 admin belongs on workstations)
- DCLAB\t1-admin  -> LOGON BLOCKED  (correct: Tier 1 admin denied on workstation)
  Message: "account is configured to prevent you from using this computer"
- Additional: t2-admin could log on but could NOT elevate PowerShell (no local
  admin rights granted yet) — privilege separation partially demonstrated.

Screenshots to capture:
- [ ] t1-admin blocked logon screen
- [ ] t2-admin successful logon
- [ ] gpresult /scope computer showing the tier GPO applied

## Findings / POA&M items discovered during testing
1. Workstations GPO had a DUPLICATE Tier0-Admins entry in "Deny log on locally"
   (cosmetic; clean up).
2. VERIFY the Servers GPO denies Tier0 + Tier2 (not swapped). Generate
   Get-GPOReport for "Tier Logon Restrictions - Servers" and confirm before FS01
   is used as evidence.

## Note (SSP)
Testing-based validation: logon attempts were actually performed, not assumed.
Group membership changes require re-logon (token rebuild) to take effect.

## Server GPO verification (completed)
Finding: "Deny log on through Terminal Services" was MISSING on the Servers GPO
(only console logon deny had saved) — a Tier0/Tier2 admin could still RDP into
FS01. Remediated by adding the RDP deny (Tier0-Admins, Tier2-Admins) and pushing
gpupdate to FS01. Duplicate Tier0-Admins entries also cleaned on both GPOs.

Final verified state (Get-GPOReport):
- Servers GPO: Deny log on locally = Tier0, Tier2 ; Deny log on through TS = Tier0, Tier2
- Workstations GPO: Deny log on locally = Tier0, Tier1 ; Deny log on through TS = Tier0, Tier1 (proven live)

POA&M value: partially-implemented control (RDP deny) discovered via self-testing
and remediated — demonstrates the assessment process actually finds gaps.

## Status: TIER MODEL FULLY VERIFIED across servers (FS01) and workstations (CLIENT-CUI).

---

## Tier machine-admin rights (POA&M closure)

Control: AC 3.1.4 (separation of duties), AC 3.1.5 (least privilege)
Gap closed: tier admins could log on (per tier) but had no local admin rights on
their tier's machines (t2-admin could log onto CLIENT-CUI but not elevate).

GPOs (Computer Config > Preferences > Control Panel Settings > Local Users and Groups,
group "Administrators (built-in)", action Update, member ADD):
- "Local Admins - Tier1 on Servers"      -> Tier1-Admins, linked OU=Servers
- "Local Admins - Tier2 on Workstations" -> Tier2-Admins, linked OU=Workstations + OU=Enclave

Verified:
- FS01 local Administrators now contains DCLAB\Tier1-Admins (+ Domain Admins, local Admin)
- CLIENT-CUI: DCLAB\Tier2-Admins in local Administrators; t2-admin can elevate (fixed)

Complete tier isolation now enforced BOTH directions:
- Tier1 admin: local admin on servers, DENIED logon on workstations
- Tier2 admin: local admin on workstations, DENIED logon on servers
Power scoped to tier; locked out across tiers. Textbook separation-of-duties.

Gotcha: GPP local-group setting must use "Administrators (built-in)" from the
dropdown (SID S-1-5-32-544), not typed "Administrators"; requires gpupdate to pull.

