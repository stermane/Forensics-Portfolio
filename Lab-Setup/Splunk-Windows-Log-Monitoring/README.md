# Windows Event Log Monitoring with Splunk

## Overview

This project demonstrates a basic SIEM log pipeline using Splunk Enterprise on Kali Linux and a Splunk Univerisal Forwarder on
a Windows 11 VM. Windows Security events are forwarded in real time, analyzed using targeted SPL queries, and visualized in a 
custom Splunk dashboard - mirroring entry-level SOC analyst workflows. 

All configuration and verification steps were performed through the Kali Linux CLI. 

---

## Lab Environment

| Component | Details |
| --- | --- |
| Host Machine | MSI Aegis R2 (Intel Core Ultra 9 E, 64GM DDR5) |
| SIEM | Splunk Enterprise - Kali Linux |
| Endpoint | Windows 11 VM (VMWare Workstation) |
| Log Forwarder | Splunk Universal Forwarder |
| Index | main |

---

## Objective

Simulate and detect common Windows Security Events using Splunk searches and a custom dashboard. EventIDs monitored:

| EventID | Descritpion | Why it Matters |
| --- | --- | --- | 
| 4624 | Successful Logon | Baseline user activity tracking | 
| 4625 | Failed Logon | Brute force / credential stuffing indicator | 
| 4688 | Process Creation | Detect suspicious process execution | 
| 4720 | New User Account Created | Insider threat / persistence indicator | 

---

## Step 1 - Enable Audit Policies on Windows 11 VM

Audit policies were enabled via an elevated Command Prompt on the Windows 11 VM. 

## Enable logon, process creation, and account management auditing: 

'''cmd
auditpol /set /subcategory:"Logon" /success: enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
'''

### Enable command-line logging via Group Police:

'''
gpedit.msc
Computer Configuration
Administrative Templates
System
Audit Process Creation
"Include command line in process creation events" - Enabled
'''

### Verify audit policy is applied:

'''cmd
auditpol /get /subcategory:"Process Creation"
auditpol /get /subcategory:"Logon"
'''

Both should return **Success and Failure** where applicable.

---

## Step 2 - Generate Windows Security Events

Events were generated manually on the Windows 11 VM to populate the Splunk index.

### Failed Logons (EventID 4625)

```powershell
$i=0; while ($i -lt 5) { net use \\localhost\IPC$ /user:fakeuser wrongpassword 2>$null; $i++ }
```

### Successful Logon (EventID 4624)

Screen was locked (`Win + L`) and the user logged back in normally.

### Process Creation (EventID 4688)

```cmd
cmd.exe /c whoami
powershell.exe -Command "Get-LocalUser"
net.exe user
```
### New User Account Created (EventID 4720)

```cmd
net user testuser Password123! /add
```

Events were confirmed in Windows Event Viewer under **Windows Logs → Security** before proceeding.

---

## Step 3 — Verify Log Flow in Splunk (Kali CLI)

All Splunk searches were run from the Kali Linux terminal using the Splunk CLI or via the web UI at `http://localhost:8000`.

### Confirm all four EventIDs are present:

```spl
index=main (EventCode=4624 OR EventCode=4625 OR EventCode=4688 OR EventCode=4720)
| stats count by EventCode
```

Expected output: all four EventIDs with counts > 0.

### Verify failed logon details:

```spl
index=main EventCode=4625
| table _time, host, Account_Name, Failure_Reason
| sort -_time
```

> **Note:** If `Account_Name` or `Failure_Reason` return empty, use raw `Message` field parsing — Windows Security logs may not auto-parse named fields without a TA
 installed. Use `rex` extractions as shown in the dashboard searches below.

### Verify process creation events:

```spl
index=main EventCode=4688
| table _time, host, Message
| sort -_time
```

---

## Step 4 — Splunk Dashboard

A 5-panel dashboard was built in Splunk titled **Windows Security Event Monitor**.

### Panel 1 — Failed Logons Over Time

```spl
index=main EventCode=4625
| timechart count span=5m
```

*Visualization: Line Chart*

---

### Panel 2 — Top Accounts with Failed Logons

```spl
index=main EventCode=4625
| rex field=Message "Account Name:\s+(?<Account_Name>\S+)"
| stats count by Account_Name
| sort -count
```

*Visualization: Bar Chart*

---

### Panel 3 — Process Creation Events

```spl
index=main EventCode=4688
| rex field=Message "New Process Name:\s+(?<Process_Name>[^\r\n]+)"
| table _time, host, Process_Name
| sort -_time
| head 20
```

*Visualization: Table*

---

### Panel 4 — New User Accounts Created

```spl
index=main EventCode=4720
| rex field=Message "Account Name:\s+(?<New_Account>[^\r\n]+)"
| table _time, host, New_Account
| sort -_time
```

*Visualization: Table*

---

### Panel 5 — Security Event Distribution

```spl
index=main (EventCode=4624 OR EventCode=4625 OR EventCode=4688 OR EventCode=4720)
| eval Event_Type=case(
    EventCode="4624", "Successful Logon",
    EventCode="4625", "Failed Logon",
    EventCode="4688", "Process Creation",
    EventCode="4720", "New User Created"
  )
| stats count by Event_Type
```

*Visualization: Pie Chart*

---

## Screenshots

| Screenshot | Description |
|------------|-------------|
| `screenshots/splunk-dashboard.png` | Full 5-panel dashboard view |
| `screenshots/splunk-search-eventcodes.png` | EventCode count verification search |
| `screenshots/splunk-search-4625.png` | Failed logon bar chart (Panel 2) |

---

## Key Takeaways

- Confirmed end-to-end log pipeline from Windows 11 endpoint to Splunk indexer on Kali Linux
- Practiced SPL queries including `timechart`, `stats`, `eval`, `rex`, and `table`
- Demonstrated how EventID 4625 clustering can indicate brute force patterns
- Used `rex` field extractions as a reliable alternative to TA-dependent field parsing
- Built a multi-panel Splunk dashboard simulating a basic SOC monitoring view

---

## Tools Used

- Splunk Enterprise (Kali Linux)
- Splunk Universal Forwarder (Windows 11)
- Windows Group Policy Editor (`gpedit.msc`)
- Windows Audit Policy CLI (`auditpol`)
- Windows Event Viewer (pre-Splunk verification)
- VMware Workstation

---

## Related Projects

- [Case 001 — Windows Suspicious User Investigation](../Case-001-Windows11-SuspiciousUser-001/)
- Case 002 — Insider Threat / Data Exfiltration *(in progress)*
