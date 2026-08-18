# Network Configuration — VMware VMnet8 (NAT)

## Why NAT / VMnet8

The lab uses VMware's default NAT network (VMnet8) rather than a fully isolated Host-only network. This keeps the physical host's Wazuh Manager (running in WSL2/Docker) reachable from every lab VM without additional routing, since Docker Desktop publishes container ports across all host interfaces — including VMnet8 — by default.

| Trade-off | Detail |
|---|---|
| Pro | Manager reachable at the VMnet8 gateway IP with zero extra routing |
| Pro | Simple to reason about — one flat subnet |
| Con | VMs technically have outbound internet access via NAT, unlike a fully air-gapped Host-only design |

## Finding the Manager's Reachable IP

The Windows host exposes a virtual adapter for VMnet8. From PowerShell:

```powershell
ipconfig
```

Look for **VMware Network Adapter VMnet8** — its IPv4 address is the address every lab VM uses to reach the Wazuh Manager.

## Verifying Reachability from a Lab VM

```powershell
Test-NetConnection -ComputerName <VMnet8_gateway_IP> -Port 1515
```

`TcpTestSucceeded : True` confirms the VM can reach the Manager's enrollment port before installing any agent.

## IP Scheme

| Host | IP |
|---|---|
| VMnet8 gateway (Wazuh Manager) | 192.168.221.1 |
| Domain Controller | 192.168.221.133 |
| Windows 11 Client #1 | 192.168.221.135 |
| Windows 11 Client #2 | 192.168.221.136 |
| Kali (attacker) | 192.168.221.128 |

## Firewall Considerations

Windows Defender Firewall on the host can silently block inbound traffic arriving on the VMnet8 adapter before it reaches Docker's port proxy. If agent enrollment fails despite a healthy Manager, confirm the relevant ports are explicitly allowed:

```powershell
New-NetFirewallRule -DisplayName "Wazuh Manager Ports" -Direction Inbound -Protocol TCP -LocalPort 1514,1515,443,9200,55000 -Action Allow
```
