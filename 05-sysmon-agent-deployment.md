# Sysmon + Wazuh Agent Deployment

Run per Windows host (Domain Controller and both clients).

## 1. Install the Wazuh agent

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER="192.168.221.1" WAZUH_AGENT_NAME="rootwin-x" WAZUH_REGISTRATION_SERVER="192.168.221.1"
NET START WazuhSvc
```

Swap `WAZUH_AGENT_NAME` per host. Confirm reachability to the Manager's enrollment port first — see [`03-network-configuration.md`](03-network-configuration.md).

## 2. Install Sysmon

Using the SwiftOnSecurity baseline config rather than a bare default — it covers process creation, network connections, and image loads without the noise of a fully unfiltered config:

```powershell
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile $env:tmp\Sysmon.zip
Expand-Archive -Path $env:tmp\Sysmon.zip -DestinationPath $env:tmp\Sysmon
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile $env:tmp\sysmonconfig.xml
& "$env:tmp\Sysmon\Sysmon64.exe" -accepteula -i "$env:tmp\sysmonconfig.xml"
```

## 3. Point the Wazuh agent at the Sysmon channel

Open `C:\Program Files (x86)\ossec-agent\ossec.conf` and add a new `<localfile>` block right after the existing one:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Save as **All Files** (not `.txt` — Notepad will silently append `.txt` otherwise) and restart the service:

```powershell
NET STOP WazuhSvc
Start-Sleep -Seconds 5
NET START WazuhSvc
Start-Sleep -Seconds 15
```

## 4. Verify ingestion

```powershell
Select-String -Path "C:\Program Files (x86)\ossec-agent\ossec.log" -Pattern "Analyzing event log" | Select-Object -Last 5
```

Sysmon (and PowerShell, once its channel is added per [`04-windows-audit-policy.md`](04-windows-audit-policy.md)) should appear in the list. If it doesn't, re-check the `ossec.conf` edit saved correctly and that the service actually restarted.

## Post-reboot behavior

Both the Wazuh agent and Sysmon run as Windows services and auto-start with the OS — no reinstall needed after a VM reboot. Only the agent's connection status needs re-confirming (Dashboard → Agents page → Active) before running a scenario.
