# Windows Audit Policy

Baseline auditing plus the two subcategories the default policy misses — both of which caused real gaps during testing (see [`06-detection-coverage.md`](06-detection-coverage.md)).

## Baseline (auditpol / GPO)

Enable via Local Group Policy → `Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration`, or `auditpol.exe`:

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable
auditpol /set /subcategory:"User Account Management" /success:enable
auditpol /set /subcategory:"Security Group Management" /success:enable
auditpol /set /subcategory:"Process Creation" /success:enable
```

Enable **PowerShell Script Block Logging** via GPO: `Administrative Templates → Windows Components → Windows PowerShell → Turn on PowerShell Script Block Logging`.

## Scheduled Task Creation (Event 4698)

The default `Other System Events` subcategory does **not** cover scheduled task creation — it lands under Object Access instead. This was missed on the first pass and had to be corrected:

```powershell
auditpol /set /subcategory:"Other Object Access Events" /success:enable
```

Re-verify after applying: create a test scheduled task and confirm 4698 appears in the Security log before trusting the pipeline.

## DCSync (Event 4662)

Audit policy alone is not sufficient here — `Directory Service Access` auditing has to be enabled *and* there has to be an explicit SACL on the AD replication objects, or nothing logs at all:

```powershell
auditpol /set /subcategory:"Directory Service Access" /success:enable
```

Then, on the Domain Controller, add a SACL for `DS-Replication-Get-Changes` / `DS-Replication-Get-Changes-All` on the domain's distinguished name (via `AddAuditRule` in PowerShell or through ADUC → Properties → Security → Advanced → Auditing on the domain object). Without this SACL, DCSync activity is invisible to Wazuh regardless of audit policy state.

## Verification Pattern

For both of the above, don't trust the config — trigger the action and confirm the event actually lands before calling it "detected":

1. Apply the policy change.
2. Trigger the action from an attacker or test context.
3. Check the Security log locally (`Get-WinEvent -LogName Security -MaxEvents 20`) before checking the Wazuh Dashboard, to isolate whether a miss is an audit-policy problem or an ingestion problem.
