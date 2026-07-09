# Wazuh SIEM — Centralized Audit & Monitoring Evidence

Manager: WAZUH01 (Ubuntu, SEG-INFRA 10.10.40.20), Wazuh 4.9.x single-node
(indexer + server + dashboard). 12 GB / 4 vCPU / 120 GB.

Control mapping:
- AU 3.3.1 / 3.3.2 — centralized audit records, traceable to users
- AU 3.3.8 — protect/retain audit info (central collection off the endpoints)
- SI 3.14.1 / 3.14.6 — monitor system for changes and events
- CA 3.12.3 — continuous monitoring
- SC 3.13.1 — boundary events captured (via pfSense syslog)

## Agent coverage (4 endpoints reporting Active)
- DC01 (domain controller, SEG-INFRA)
- CA01 (enterprise CA, SEG-INFRA)
- FS01 (CUI file server, SEG-INFRA)
- CLIENT-CUI (enclave workstation, SEG-CUI — agent traffic crosses boundary via
  CUI->WAZUH01 firewall rule on TCP 1514/1515)

Note: pfSense is not an agent (FreeBSD) — integrated via syslog instead (below).
WAZUH01 is the manager, not a counted agent. EliteDesks not yet built.

## File Integrity Monitoring (SI)
FS01 ossec.conf syscheck: realtime FIM on E:\CUI
`<directories check_all="yes" report_changes="yes" realtime="yes">E:\CUI</directories>`
Verified: create/modify events on E:\CUI appear in dashboard with content diffs.
Gotcha logged: realtime FIM records a baseline on (re)start; only post-baseline
changes alert. Confirm "Monitoring path: 'e:\cui' ... realtime" in ossec.log with
a current timestamp before testing.

## Security Configuration Assessment (CM)
SCA enabled by default; agents scored against CIS benchmarks (e.g. cis_win2022.yml)
— continuous config-compliance, viewable per agent.

## NIST compliance mapping
Wazuh tags alerts to NIST 800-53 controls out of the box (observed tags on live
alerts: nist_800_53_AU.14, AC.6, AC.7). Regulatory Compliance > NIST 800-53
dashboard provides framework-organized view (800-53 overlaps 800-171).

## pfSense syslog integration (SC / AU boundary logging)
- WAZUH01 ossec.conf: <remote><connection>syslog</connection><port>514</port>
  <protocol>udp</protocol><allowed-ips>10.10.40.1</allowed-ips></remote>
- pfSense: Status > System Logs > Settings > Remote Logging, Source=INFRA,
  target 10.10.40.20:514, Firewall Events (or Everything).
- VERIFIED end-to-end: a denied connection from CLIENT-CUI (enclave deny-all rule)
  logged by pfSense -> shipped to Wazuh -> surfaced in dashboard. Boundary and
  hosts now monitored in one pane.

## Debug notes (for runbook)
- Confirm listener: `sudo ss -uln | grep 514` -> 0.0.0.0:514
- Confirm packets on wire: `sudo tcpdump -n -i any udp port 514` (source must match
  allowed-ips = 10.10.40.1)
- pfSense pass/block filterlog events decode at low severity; may appear in Discover
  (indexed) without hitting alerts.log (level 3+). Set logall/logall_json=yes to see
  raw archives if needed.

## Screenshots to capture
- [ ] Endpoints page: 4 agents Active
- [ ] FIM event on E:\CUI (with diff)
- [ ] NIST 800-53 compliance dashboard
- [ ] pfSense CLIENT-CUI deny event in Wazuh

## Status: WAZUH SIEM COMPLETE — 4 agents + FIM on CUI + pfSense boundary syslog,
## NIST-tagged. Hosts and perimeter centralized. Optional polish: tune pfSense
## firewall-event severity for louder alerting.

