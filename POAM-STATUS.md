# POA&M Thread Status

Gaps found via self-testing during the build, and their resolution.

| # | Finding (control) | Found during | Status |
|---|---|---|---|
| 1 | Servers GPO missing "Deny log on through Terminal Services" (AC) | tier logon test | CLOSED - RDP deny added, verified |
| 2 | File Share auditing (5145) not enabled - share denials unlogged (AU) | jdoe/bsmith test | CLOSED - File Share + Detailed File Share auditing enabled |
| 3 | BitLocker escrow GPO empty - keys not escrowing to AD (MP/SC) | CLIENT-CUI encrypt | CLOSED - GPO configured, CLIENT-CUI key escrowed, verified in AD |
| 4 | Temporary INFRA "Any/Any" egress rule (SC 3.13.1) | build scaffolding | CLOSED - least-privilege egress + logged deny, verified |
| 5 | Tier admins had no machine-admin rights on their tier (AC 3.1.4/3.1.5) | t2-admin elevation | CLOSED - GPP local-admin GPOs, both tiers verified |

Note: every gap was surfaced by ACTUALLY TESTING controls rather than assuming
implementation - the self-assessment process finding and remediating its own gaps.
This is itself evidence of a functioning CA (security assessment) practice.

## Remaining (not gaps - unbuilt scope, by choice)
- EliteDesks (3x physical) - backup node, general workstation, admin jump box
- pfSense firewall-event severity tuning (optional polish)
- Evidence screenshots (batch at end)
- SSP + POA&M formal writeup

