# Architecture

**Windows 11 Host • WSL2 • Docker • VMware NAT Network**

## Overview

```
Physical Host (Windows 11)
        │
        ▼
WSL2 – Ubuntu 24.04
        │
        ├── Docker
        │     └── Wazuh SIEM Stack (single-node)
        │           ├── Wazuh Manager    — 1514 (TCP/UDP), 55000 (API)
        │           ├── Wazuh Indexer    — 9200 (HTTP)
        │           └── Wazuh Dashboard  — 5601 → mapped to 443 on host
        │
        ▼
VMware NAT Network (VMnet8)
        │
        ├── Windows Server 2025 (Domain Controller)
        │     AD DS, DNS, DHCP (optional), Wazuh Agent, Sysmon
        │
        ├── Windows 11 Enterprise #1 (domain-joined)
        │     Wazuh Agent, Sysmon
        │
        ├── Windows 11 Enterprise #2 (domain-joined)
        │     Wazuh Agent, Sysmon
        │
        └── Kali Linux (unmonitored)
              AD attack tooling, red team activity
```

## Communication Flow

| Flow | Purpose |
|---|---|
| VMs → Wazuh Manager | Agents ship logs/events over port 1514 |
| Wazuh Manager → Indexer | Store and query alert data |
| Wazuh Dashboard → Indexer | Visualization and search |
| All VMs | Communicate over VMnet8 (NAT) |

## Port Summary (VMs → Wazuh Stack)

| Port | Protocol | Purpose |
|---|---|---|
| 1514 | TCP/UDP | Wazuh Agent traffic |
| 1515 | TCP | Agent enrollment |
| 55000 | TCP | Wazuh API |
| 9200 | TCP | Wazuh Indexer |
| 443 (host) → 5601 (container) | TCP | Wazuh Dashboard |

## Key Design Points

- Wazuh SIEM runs inside WSL2 using Docker containers on the physical host — not as a separate VM.
- All lab VMs sit on a single VMware NAT network (VMnet8), isolated from the rest of the host's LAN/internet by default.
- Every Windows host ships Security log, Sysmon, PowerShell Script Block, FIM, and Windows Firewall (WFP) telemetry to the manager.
- Kali is deliberately left unmonitored to preserve a clean attacker/defender boundary.

## Full IP Scheme

| Host | Role | IP |
|---|---|---|
| Domain Controller | AD DS / DNS | 192.168.221.133 |
| Windows 11 Client #1 | Domain-joined workstation | 192.168.221.135 |
| Windows 11 Client #2 | Domain-joined workstation | 192.168.221.136 |
| Kali | Attacker | 192.168.221.128 |
| Physical host (VMnet8 gateway) | Wazuh Manager endpoint | 192.168.221.1 |

See [`Active-Directory-Attack-Lab/LAB-TOPOLOGY.md`](https://github.com/Zabi-01/Active-Directory-Attack-Lab) for the attacker-side view of the same topology.
