# Splunk Saved Searches — Windows Security Event Monitor

All searches target `index=main` and are designed for Windows Security logs
forwarded via Splunk Universal Forwarder. `rex` extractions are used in place
of TA-dependent field parsing for portability.

---

## Verification Searches

### All Monitored EventIDs — Count by Type
```spl
index=main (EventCode=4624 OR EventCode=4625 OR EventCode=4688 OR EventCode=4720)
| stats count by EventCode
```
*Use this first to confirm all four EventIDs are present in the index.*

---

### Most Recent Events — Any Monitored EventID
```spl
index=main (EventCode=4624 OR EventCode=4625 OR EventCode=4688 OR EventCode=4720)
| table _time, EventCode, host, Message
| sort -_time
| head 50
```

---

## EventID 4624 — Successful Logon

### All Successful Logons
```spl
index=main EventCode=4624
| table _time, host, Account_Name, Logon_Type
| sort -_time
```

### Successful Logons by Account
```spl
index=main EventCode=4624
| stats count by Account_Name, host
| sort -count
```

### Successful Logons Over Time
```spl
index=main EventCode=4624
| timechart count span=5m
```

---

## EventID 4625 — Failed Logon

### All Failed Logons
```spl
index=main EventCode=4625
| table _time, host, Account_Name, Failure_Reason
| sort -_time
```

### Failed Logons by Account (rex fallback)
```spl
index=main EventCode=4625
| rex field=Message "Account Name:\s+(?<Account_Name>\S+)"
| stats count by Account_Name
| sort -count
```

### Failed Logons Over Time
```spl
index=main EventCode=4625
| timechart count span=5m
```

### Brute Force Detection — Accounts with 3+ Failures
```spl
index=main EventCode=4625
| rex field=Message "Account Name:\s+(?<Account_Name>\S+)"
| stats count by Account_Name, host
| where count >= 3
| sort -count
```

---

## EventID 4688 — Process Creation

### All Process Creation Events
```spl
index=main EventCode=4688
| rex field=Message "New Process Name:\s+(?<Process_Name>[^\r\n]+)"
| table _time, host, Process_Name
| sort -_time
```

### Process Creation — Top Processes
```spl
index=main EventCode=4688
| rex field=Message "New Process Name:\s+(?<Process_Name>[^\r\n]+)"
| stats count by Process_Name
| sort -count
```

### Suspicious Process Hunting — Common LOLBins
```spl
index=main EventCode=4688
| rex field=Message "New Process Name:\s+(?<Process_Name>[^\r\n]+)"
| search Process_Name IN ("*cmd.exe*", "*powershell.exe*", "*wscript.exe*", "*cscript.exe*", "*mshta.exe*", "*certutil.exe*", "*net.exe*")
| table _time, host, Process_Name
| sort -_time
```

### Process Creation with Command-Line Arguments
```spl
index=main EventCode=4688
| rex field=Message "Process Command Line:\s+(?<CommandLine>[^\r\n]+)"
| table _time, host, CommandLine
| sort -_time
```

---

## EventID 4720 — New User Account Created

### All New Accounts Created
```spl
index=main EventCode=4720
| rex field=Message "Account Name:\s+(?<New_Account>[^\r\n]+)"
| table _time, host, New_Account
| sort -_time
```

### New Account Creation Count by Host
```spl
index=main EventCode=4720
| stats count by host
| sort -count
```

---

## Dashboard Searches

### Panel 1 — Failed Logons Over Time
```spl
index=main EventCode=4625
| timechart count span=5m
```

### Panel 2 — Top Accounts with Failed Logons
```spl
index=main EventCode=4625
| rex field=Message "Account Name:\s+(?<Account_Name>\S+)"
| stats count by Account_Name
| sort -count
```

### Panel 3 — Process Creation Events
```spl
index=main EventCode=4688
| rex field=Message "New Process Name:\s+(?<Process_Name>[^\r\n]+)"
| table _time, host, Process_Name
| sort -_time
| head 20
```

### Panel 4 — New User Accounts Created
```spl
index=main EventCode=4720
| rex field=Message "Account Name:\s+(?<New_Account>[^\r\n]+)"
| table _time, host, New_Account
| sort -_time
```

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

---

## Notes

- If named fields (`Account_Name`, `Failure_Reason`, etc.) return empty, use
  the `rex` extractions above — Windows Security logs may not auto-parse
  without a Windows TA installed.
- Time range: set to **Last 24 hours** or **Last 7 days** depending on how
  recently events were generated.
- All searches assume `index=main` — update if your forwarder targets a
  different index.
