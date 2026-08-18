# Detection Coverage

Results against the attack chain run in [Active-Directory-Attack-Lab](https://github.com/Zabi-01/Active-Directory-Attack-Lab). Raw verification output for the two remediated detections lives in [`testing-logs/`](../testing-logs).

## Attack → Detection Mapping

| Attack | Event ID(s) | Status | Notes |
|---|---|---|---|
| Port scanning (nmap) | 5157 (WFP) | ✅ Detected | ~600 events; requires firewall logging enabled explicitly |
| Failed logon / brute force (Hydra) | 4625 | ✅ Detected | 81 events |
| Successful logon | 4624 | ✅ Detected | 1,047 events — noisy at scale, scope by time window when citing |
| Kerberoasting | 4769 | ✅ Detected | 48 events |
| Pass-the-Hash | Rule 92652 | ✅ Detected | 128 events, custom Wazuh rule |
| Privileged logon | 4672 | ✅ Detected | 114 events |
| User creation | 4720 | ✅ Detected | |
| Domain group modification | 4728 / 4732 | ✅ Detected | |
| Service creation | 7045 | ✅ Detected | 17 events |
| Persistence (malware-path file drop) | Rule 92213 | ✅ Detected | 9 events, level 15 |
| Scheduled task creation | 4698 | ⚠️ Initially missed | Wrong audit subcategory (`Other System Events` instead of `Object Access → Other Object Access Events`) — fixed, see [`04-windows-audit-policy.md`](04-windows-audit-policy.md) |
| DCSync | 4662 | ⚠️ Initially missed | Audit policy alone isn't sufficient — requires an explicit SACL on the AD replication objects (`DS-Replication-Get-Changes[-All]`). Not logged until the SACL is set |
| LLMNR / NBT-NS poisoning | — | ❌ Architecturally blind | Host-based agents can't see L2/broadcast spoofing; would need a network sensor (Zeek/Suricata) |
| IPv6 / mitm6 | — | ❌ Architecturally blind | Same limitation — wire-level, invisible to host agents |

## Alert Severity Breakdown (Dashboard)

| Severity | Level | Count |
|---|---|---|
| Critical | 15+ | 18 |
| High | 12–14 | 120 |
| Medium | 7–11 | 5,181 |
| Low | 0–6 | 11,641 |

## MITRE ATT&CK Coverage

| Technique | Tactic(s) | Count |
|---|---|---|
| Valid Accounts | Defense Evasion, Initial Access, Persistence, Privilege Escalation | 709 |
| Domain Accounts | Defense Evasion, Initial Access, Lateral Movement, Persistence, Privilege Escalation | 258 |
| Pass the Hash | Defense Evasion, Initial Access, Lateral Movement, Persistence, Privilege Escalation | 258 |
| Domain Policy Modification | Defense Evasion, Privilege Escalation | 206 |
| Ingress Tool Transfer | Command and Control | 180 |

## Known Gaps & Fixes Applied

1. **Scheduled Task (4698)** — corrected audit subcategory to `Object Access → Other Object Access Events`; re-tested and confirmed.
2. **DCSync (4662)** — requires an AD-level SACL grant, not just `auditpol`. Applied via `AddAuditRule` against `DS-Replication-Get-Changes` / `-All` on the domain DN.
3. **Duplicate agent entries** — repeated agent re-enrollment during setup left stale IDs registered under the same hostnames; cleaned via `manage_agents -r` before trusting aggregate counts.
4. **LLMNR / mitm6** — accepted as a permanent blind spot for a host-agent-only architecture, not something fixable at this layer.

## Attribution Query Gotcha

Windows logs an attacker's source IP under different field names depending on event type — a single `sourceAddress` filter silently undercounts. Query all three when attributing activity to the attacker:

```bash
grep -E '"(sourceAddress|ipAddress|sourceIp)":"<attacker_ip>"' alerts.json
```
