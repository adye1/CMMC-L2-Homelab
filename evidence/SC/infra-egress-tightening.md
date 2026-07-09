# INFRA Segment Egress — Least-Privilege Tightening

Control: SC 3.13.1 (boundary protection), SC 3.13.6 (deny-by-default)
POA&M closure: the temporary "INFRA net -> any" build-time rule (added during DC01
connectivity troubleshooting) is REMOVED and replaced with least-privilege egress.

## Final INFRA ruleset (pfSense, Firewall > Rules > INFRA)
- Pass  INFRA subnets -> any :53   (DNS out)
- Pass  INFRA subnets -> any :443  (HTTPS - updates/activation/feeds)
- Pass  INFRA subnets -> any :80   (HTTP - updates)
- Pass  INFRA subnets -> any :123  (NTP - time sync)
- Pass  INFRA subnets -> any ICMP  (diagnostics - allowed on infra/mgmt, DENIED in CUI enclave)
- Block INFRA subnets -> any (LOGGED) - "Deny-all INFRA egress - SC 3.13.1"

## Verification
- DNS (Resolve-DnsName), HTTPS (curl/Test-NetConnection :443): SUCCESS
- Non-allowed port (Test-NetConnection 8.8.8.8 :22): BLOCKED (times out) + logged to Wazuh
- Same-subnet server-to-server traffic (DC01/CA01/FS01/WAZUH01) unaffected -
  intra-10.10.40.0/24 does not traverse pfSense.

## SSP note
ICMP egress permitted from INFRA (operational diagnostics) but denied from the CUI
enclave - documented, intentional differential (tighter controls where CUI lives).

## Status: COMPLETE - INFRA scaffolding rule closed, least-privilege egress verified.

