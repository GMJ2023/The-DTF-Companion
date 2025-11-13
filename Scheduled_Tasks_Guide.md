# Scheduled Tasks Guide

This guide documents the automated monitoring tasks used within the RPA and PDF processing ecosystem. It combines information from:

- **Monitor Output Log.xml**
- **MonitorBrowser.xml**
- **monitor_output_log.ps1**
- **monitor_browser.ps1**

These tasks ensure continuous, self-healing operation of automation components.

---

## 1. Monitor Output Log Task

This scheduled task monitors `output_log_.log` for anomalies or low row counts and triggers webhook-based alerts.

### Source File
`Monitor Output Log.xml` fileciteturn51file0

### Behaviour Summary
- Runs **every 5 minutes** after system boot.
- Executes as the **zoho-admin** account with *HighestAvailable* privileges.
- Launches:
  ```
  C:inary-proxy\monitor_output_log.ps1
  ```  
- Ensures only one instance runs at a time using `IgnoreNew`.
- Designed to detect:
  - Low row counts  
  - Missing log updates  
  - Error signatures within the log  
  - Conditions requiring manual intervention  

---

## 2. Monitor Browser Task

This task ensures Chrome remains active for **timesheet_automation.js** sessions.

### Source File
`MonitorBrowser.xml` fileciteturn52file0

### Behaviour Summary
- Runs **every 5 minutes** after boot.
- Hidden execution window (`-WindowStyle Hidden`).
- Calls:
  ```
  C:inary-proxy\monitor_browser.ps1
  ```
- Uses `IgnoreNew` to prevent task pile‑up.
- Ensures:
  - Chrome is running  
  - Automation JS is loaded  
  - Browser sessions recover if stalled  

---

## 3. monitor_output_log.ps1

This script provides real-time monitoring of the automation log.

### Key Functions
- Opens and reads `output_log_.log`
- Checks for:
  - Zero-row output
  - Failed recognitions  
  - Stale log timestamps  
  - Unexpected terminations  
- Posts alerts to the configured webhook endpoint
- Writes internal diagnostics for audit trails

(Contents validated from your uploaded file.) fileciteturn53file0

---

## 4. monitor_browser.ps1

This script ensures Chrome automations cannot silently fail.

### Behaviour
- Checks if Chrome.exe is running
- Checks if `timesheet_automation.js` remains attached to the session
- Restarts Chrome if needed
- Writes status output for debugging

(This is the script triggered by MonitorBrowser.xml.) fileciteturn50file2

---

## Operational Flow

```
Windows Task Scheduler
    ├── Monitor Output Log  → monitor_output_log.ps1 → Webhook + Diagnostics
    └── MonitorBrowser      → monitor_browser.ps1     → Chrome watcher
```

Both tasks use:
- 5‑minute repetition interval  
- BootTrigger to start monitoring immediately  
- Highest privileges  
- IgnoreNew to prevent overlapping runs  

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|--------|--------------|------|
| Task not running | XML not imported correctly | Re-import task into Task Scheduler |
| Alerts firing constantly | Log file malformed or stuck | Check `output_log_.log` for last update timestamp |
| Browser not restarting | Chrome path changed or permissions issue | Verify installation and rerun task manually |
| Task shows “running” endlessly | Stuck PowerShell instance | End task in Task Manager and restart service |

---

## Best Practices

- Keep all monitor scripts in **C:\binary-proxy**
- Ensure webhook endpoints remain valid  
- Review logs weekly to confirm detection rules behave correctly  
- Use British English and avoid Oxford commas for consistency with all project documents  

---

## Ownership

© 2025 Geoffrey Jones. All rights reserved.  
Unauthorised copying or distribution of this material, in whole or in part, is strictly prohibited.
