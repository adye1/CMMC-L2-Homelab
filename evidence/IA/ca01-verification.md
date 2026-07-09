# CA01 — Enterprise Root CA Verification Evidence

Control mapping: IA (3.5.3 cert-based auth foundation), SC (3.13.11 FIPS crypto)

## CA configuration (as-built)
- Type: Enterprise Root CA (`CA type 0`)
- Common name: DCLAB-CA01-CA
- DNS name: CA01.dclab.local
- Key length: 4096-bit RSA
- Hash algorithm: SHA-256
- Validity: 10 years
- Provider: RSA#Microsoft Software Key Storage Provider

## Verification results (certutil)
- `Get-Service CertSvc` -> Running / Automatic
- `certutil -CAInfo` -> CA type 0 (Enterprise Root CA); CA cert Valid; CRL Valid
  (CPF_BASE + CPF_COMPLETE); DNS Name CA01.dclab.local
- `certutil -config "CA01.dclab.local\DCLAB-CA01-CA" -ping` -> successful
  (ICertRequest2 interface alive) = CA reachable and AD-published

## Screenshots to capture
- [ ] certutil -CAInfo full output
- [ ] Certification Authority console (certsrv.msc) showing CA healthy
- [ ] AD Sites/Services or Enrollment Services container listing DCLAB-CA01-CA

## Production delta (note for SSP)
Lab uses a single-tier online Enterprise Root CA. Production CUI environments
use a two-tier PKI: offline root CA + online subordinate issuing CA. Documented
as a known, intentional lab simplification.

---

# BitLocker AD Escrow — POA&M closure (separate from CA, filed here for MP/SC)

Gap: "BitLocker - AD Recovery Escrow" GPO was empty (Computer Config blank), so
CLIENT-CUI's C: recovery key could not escrow to AD; worked around locally at first.
Fix: configured the GPO properly (OS Drives + Fixed Data Drives -> Save recovery to
AD DS -> Store recovery passwords and key packages), verified settings persisted in
Get-GPOReport, pushed to CLIENT-CUI, and escrowed the existing key.
Verified: msFVE-RecoveryInformation object present under CN=CLIENT-CUI in AD.
Result: ALL encrypted volumes (FS01 C:/E:, CLIENT-CUI C:) now escrow to AD.

