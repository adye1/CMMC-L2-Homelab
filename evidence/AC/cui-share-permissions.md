# CUI Share — Least-Privilege Access Control Evidence

Host: FS01 (Tier 1, SEG-INFRA 10.10.40.11) | Share: \\FS01\CUI | Volume: E:\CUI (CUI-Data, NTFS, 200GB)

Control mapping:
- AC 3.1.1 — Limit system access to authorized users
- AC 3.1.5 — Least privilege
- MP 3.8.x — Protect CUI on media/at rest (dedicated volume; BitLocker to follow)

## Access model (defense-in-depth: effective access = most restrictive of the two layers)
Authorized: DCLAB\CUI-Users (member: jdoe). NOT authorized: bsmith (deliberately excluded, deny-test).

## NTFS ACL (filesystem layer) — `icacls E:\CUI`
Inheritance disabled (/inheritance:r); no Everyone / BUILTIN\Users entries.
```
E:\CUI BUILTIN\Administrators:(F)
       NT AUTHORITY\SYSTEM:(OI)(CI)(F)
       DCLAB\Domain Admins:(OI)(CI)(F)
       DCLAB\CUI-Users:(OI)(CI)(M)
```

## Share ACL (network layer) — `Get-SmbShareAccess -Name CUI`
No Everyone entry (confirmed via Revoke no-op).
```
CUI  DCLAB\Domain Admins  Allow  Full
CUI  DCLAB\CUI-Users      Allow  Change
```

## Screenshots to capture
- [ ] icacls E:\CUI output
- [ ] Get-SmbShareAccess -Name CUI output
- [ ] Folder Properties > Security tab and Sharing tab (GUI view)

## Pending proof-test (stage 5)
From a domain-joined client: jdoe reads \\FS01\CUI (ALLOW), bsmith denied (DENY),
plus the resulting Event 4656/4663 access-denied audit entry (AU evidence).

## Status: share built and locked. Auditing + live deny-test pending CLIENT-CUI.

---

## AU — Object-access auditing (added)

Control mapping: AU 3.3.1 (create/retain audit records), AU 3.3.2 (trace actions to users)

### Audit policy — `auditpol /get /subcategory:"File System"`
```
Object Access
  File System    Success and Failure
```

### Folder SACL — `(Get-Acl E:\CUI -Audit).Audit`
```
IdentityReference : Everyone
AuditFlags        : Success, Failure
FileSystemRights  : ReadData, CreateFiles, Delete, ReadPermissions
InheritanceFlags  : ContainerInherit, ObjectInherit
```

Both layers required for audit events to fire (policy + SACL) — both confirmed present.
Expected events during the deny-test: 4656 (handle requested) / 4663 (object access
attempt), with 4656 failure entries for bsmith's denied access.

Note: per-machine auditpol here for the lab; production pattern = GPO-driven
advanced audit policy pushed fleet-wide (hardening baseline). Note in SSP.

## Status: share locked + audited. Live jdoe/bsmith deny-test pending CLIENT-CUI.

---

## LIVE ACCESS TEST — completed from CLIENT-CUI (enclave endpoint, SEG-CUI)

Control mapping: AC 3.1.1 (authorized access), AC 3.1.5 (least privilege),
AU 3.3.1 (audit records), AU 3.3.2 (trace to individual users)

Test matrix (results matched predicted outcomes stated before testing):

| Account | In CUI-Users | Result at \\FS01\CUI | Audit event |
|---|---|---|---|
| jdoe   | Yes | ACCESS GRANTED (read test-document.txt) | 4663 SUCCESS on E:\CUI |
| bsmith | No  | ACCESS DENIED | **5145 FAILURE  bsmith  \\*\CUI** |

Key evidence line (FS01 Security log):
```
7/8/2026 7:59:xx AM  5145  FAILURE  bsmith  \\*\CUI
```
A named unauthorized user's denial to CUI, captured as an audit-failure record —
links AC enforcement to AU accountability in a single event.

### Finding during testing (POA&M)
Share-level denials log as 5145 (File Share subcategory), which is SEPARATE from
the File System (4663) auditing enabled earlier. File Share + Detailed File Share
auditing had to be enabled via auditpol to capture bsmith's denial:
`auditpol /set /subcategory:"File Share" /success:enable /failure:enable`
`auditpol /set /subcategory:"Detailed File Share" /success:enable /failure:enable`
Lesson: NTFS SACL auditing alone does not capture share-layer denials; both
subcategories are needed for complete coverage. Production = push via GPO.

Screenshots to capture:
- [ ] jdoe: open \\FS01\CUI folder + test file
- [ ] bsmith: access-denied error dialog
- [ ] FS01 Security log 5145 FAILURE for bsmith (Event Viewer detail)

## Status: CUI SHARE FULLY DEMONSTRATED — permissions enforce + denials audited,
## tied to named users. AC + AU evidence complete.

