# Scheduled Tasks Guide

This guide documents the automated monitoring tasks used within the RPA and PDF processing ecosystem. It combines information from:

- **Monitor Output Log.xml**
- **MonitorBrowser.xml**
- **monitor_output_log.ps1**
- **monitor_browser.ps1**

These tasks ensure continuous, self-healing operation of automation components.

---

## 1. Monitor Output Log Task (Task Scheduler)

This scheduled task monitors the JSON log file `output_log_.log` for anomalies or low row counts and triggers webhook-based alerts.

### Source file
`Monitor Output Log.xml`

### Behaviour summary

- Trigger: **BootTrigger** with a repetition interval of **5 minutes**
- Runs as: `rpa\zoho-admin` with **HighestAvailable** privileges
- Action:
  ```text
  Program:          powershell.exe
  Arguments:        -NoProfile -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_output_log.ps1"
  WorkingDirectory: C:\binary-proxy
  ```
- Multiple instances: `IgnoreNew` — if a run is still executing when the next trigger fires, Task Scheduler skips the new one
- Integrated with the `monitor_output_log.ps1` script, which does the heavy lifting (see section 3)

---

## 2. Monitor Browser Task (Task Scheduler)

This task ensures Chrome remains active for **My Digital Accounts** automation and that the Zoho RPA agent and file monitor service stay healthy.

### Source file
`MonitorBrowser.xml`

### Behaviour summary

- Trigger: **BootTrigger** with a repetition interval of **5 minutes**
- Runs as: `rpa\zoho-admin` with **HighestAvailable** privileges
- Hidden execution:
  ```text
  Program:          powershell.exe
  Arguments:        -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "C:\binary-proxy\monitor_browser.ps1"
  WorkingDirectory: C:\binary-proxy
  ```
- Multiple instances: `IgnoreNew`
- Main responsibilities (via `monitor_browser.ps1`):
  - Check Chrome + DevTools remote debugging
  - Check the `browser_state.json` WebSocket endpoint
  - Check My Digital Accounts login state
  - Check File Monitor Service heartbeat
  - Check Zoho RPA Desktop Agent

---

## 3. Script: monitor_output_log.ps1

This script does **two big jobs**:

1. Watches `output_log_.log` for **low row count** events and alerts immediately  
2. Monitors a chain of **OneDrive pipeline folders** for **stuck files** and sends a *single friendly alert per file* with targeted guidance

### 3.1 Configuration

Key configuration values at the top of the script:

- `\$LogFilePath` – path to the JSON log file written by the Catalyst / RPA pipeline  
- `\$LogDir` – folder where this script writes its own log (e.g. `output_log_monitor_YYYYMMDD.txt`)  
- `\$WebhookUrl` – Zoho Flow webhook endpoint used for alerts  
- Timing/tuning:
  - `\$DebounceSeconds` – suppress repeated log change events within a few seconds
  - `\$FolderCheckSeconds` – how often folder activity is scanned (default 10 seconds)
  - `\$HeartbeatInterval` – heartbeat logging interval (seconds)
  - `\$StatusIntervalSeconds` – how often to self-check and self-heal the log watcher
  - `\$InactivityThresholdSec` – how long a file can sit unchanged before being considered **stuck** (default 900 seconds = 15 minutes)

### 3.2 Friendly hints per folder

The script defines a `\$FolderHints` hashtable, mapping **key OneDrive folders** to targeted help text. Each entry includes placeholders:

- `{IDLE_MIN}` – populated with the number of minutes the file has been idle
- `{NOW}` – populated with the current timestamp

Folders covered and their logical “flow” numbers:

1. `Timesheets` – Flow #1 (RPA Transform Files)  
2. `PDF Drop Zone` – Flow #2 (file manager service feeding the PDF parser)  
2.1. `CSV Paradise` – Flow #2.1 (CSV Paradise feeding CSV Drop Zone)  
3. `CSV Drop Zone` – Flow #3  
4. `Holding Zone` – Flow #4  
5. `AssembleXlsx` – Flow #5  
6. `Formatted` – Flow #6 (final handoff to My Digital Accounts)

Each hint tells the user which **RPA flow**, **service**, or **Task Scheduler job** to check, plus a concrete “move the file out and back in” suggestion to re-trigger the pipeline when appropriate.

### 3.3 State files and tracking

- `\$TempDir` – `C:\temp`
- `\$ActivityStateFile` – `C:\temp\folder_activity.json`

The script maintains:

- `PreviousState` – last known file counts per folder
- `LastChangeTimes` – when each folder last changed
- `FileCountAtLastChange` – snapshot of counts at last change
- `FileTimers` – when each specific file started being tracked as “present and idle”
- `LastAlertTimes` – per-file record of when an alert was sent (so you do **not** spam alerts)

This state is persisted to `folder_activity.json` so that restarts do not lose history.

### 3.4 Logging helper

```powershell
function Write-Log {
    param([string]$Message)
    $ts = Get-Date -Format "yyyy-MM-ddTHH:mm:ss.fffZ"
    Write-Host "$ts - $Message"
    $logName = "output_log_monitor_$(Get-Date -Format 'yyyyMMdd').txt"
    $logPath = Join-Path $LogDir $logName
    "$ts - $Message" | Out-File -FilePath $logPath -Append -Encoding UTF8
}
```

- Writes to **console** and to a daily log file under `service log`  
- Ensures the logging directory exists before use

### 3.5 Webhook helper

```powershell
function Send-ZohoWebhook {
    param(
        [string]$Title,
        [string]$Body,
        [int]$Line = 4
    )
    ...
}
```

- Wraps `Invoke-RestMethod` and logs success or failure
- `line` is passed through for routing / display on the Zoho Flow side

### 3.6 Log file watcher – low row count detection

The script creates a `System.IO.FileSystemWatcher` on the **folder** `C:\binary-proxy` with filter `output_log_.log`:

- Watches **Changed** events only
- Uses `\$DebounceSeconds` to avoid double-firing when the file updates multiple times quickly
- Waits an extra **2 seconds** before reading, to allow the writer to finish

Event action (`$logAction`):

1. Reads `output_log_.log` as JSON
2. Checks `processedRowCount`  
3. If `processedRowCount -eq 0`:
   - Builds an alert with:
     - Proposed file name
     - Catalyst run ID
     - Timestamp
   - Sends webhook with title `"<proposedFileName> - Low Row Count!"`
4. Otherwise logs that the row count is OK

There is also a **self-healing** block in the main loop that:

- Recreates the watcher if it becomes `\$null` or loses its `EnableRaisingEvents` property
- Re-attaches the event and re-enables raising events

### 3.7 Folder activity monitor – stuck file detection

`CheckFolderActivity` is the heart of the **stuck file** logic:

1. Ensures global tracking tables are initialised
2. Defines the `FlowOrder` map (folder → flow number)
3. Loops through each folder in `\$FolderList`:
   - Skips if the folder path does not exist
   - Collects all files recursively
   - Records the total file count as `CurrentState[folder]`
   - Compares current count to `PreviousState[folder]`

#### When the file count changes

- Updates `LastChangeTimes[folder]` and `FileCountAtLastChange[folder]`
- For any **new** file, starts tracking in `FileTimers` and logs `NEW FILE TRACKING`
- Removes entries from `FileTimers` where the file no longer exists (file moved or processed) and logs `FILE REMOVED` / `DE-LATCHED`
- Skips idle checks for this cycle (because activity has just occurred)

#### When the file count stays the same

For each file in the folder:

1. Ensures there is an entry in `FileTimers` with the first time we saw the file  
2. Calculates how long the file has been tracked (`idleSec`)  
3. If `idleSec >= InactivityThresholdSec` **and** no alert has been sent for this file:
   - Determines `flow = FlowOrder[folder]`
   - Converts idle time into minutes (`idleMin`)
   - Builds a **folder-specific hint** from `FolderHints[folder]`, replacing `{IDLE_MIN}` and `{NOW}`
   - Creates a detailed `emailBody`, including:
     - Which folder and flow have stalled
     - Which file is stuck
     - The idle duration
     - The hint text
   - Sends a Zoho webhook with title:  
     `DTF Stalled - <FolderName> (Flow #<flow>): <FileName>`  
   - Logs that a **CRITICAL** alert has been sent
   - Records `LastAlertTimes[filePath] = now` so we do not alert again for the same file

After scanning all folders, the script dumps `CurrentState` back into `folder_activity.json` and updates `PreviousState` in memory.

### 3.8 Main loop and heartbeat

The main loop runs every **second** and coordinates:

- Folder scanning (`CheckFolderActivity`) every `\$FolderCheckSeconds`
- Heartbeat logging every `\$HeartbeatInterval` seconds
- Log watcher self-check and “System healthy” log every `\$StatusIntervalSeconds`

Heartbeat messages look like:

```text
Heartbeat: alive for X minutes
```

This gives an easy way to confirm the monitor script is still running without digging into Task Scheduler.

---

## 4. Script: monitor_browser.ps1

This script is the **watchdog** for:

- The Chrome browser running My Digital Accounts via a **remote debugging port**
- The DevTools / WebSocket endpoint saved in `browser_state.json`
- The My Digital Accounts **login state** (is the login screen visible?)
- The **File Monitor Service** (via heartbeat)
- The **Zoho RPA Desktop Agent** (process check)

It sends readable, friendly emails via a Zoho Flow webhook whenever something is wrong.

### 4.1 Configuration

Key settings near the top:

- `\$LogDir` – log folder: `C:\binary-proxy\service log`
- `\$WebhookUrl` – Zoho Flow webhook for all alerts
- `\$BrowserStatePath` – JSON file containing `wsEndpoint` (WebSocket URL)
- `\$CheckIntervalSeconds` – main loop sleep interval (default 60 seconds)
- `\$MdaLoginUrl` – pattern used to detect the MDA login page (`https://www2.mydigitalaccounts.com/login*`)
- `\$DevToolsUrl` – DevTools HTTP endpoint (`http://localhost:9222/json`)

File Monitor Service monitoring:

- `\$FileManagerScriptName` – for reference (`file_manager_service.ps1`)
- `Test-FileManagerService()` – checks heartbeat file:  
  `C:\binary-proxy\running\file_monitor_service.heartbeat`
- Considers the File Monitor healthy if the heartbeat is **less than 180 seconds old**
- Uses `\$FileManagerAlertCooldown` (300 seconds) to avoid repeated alerts

Zoho RPA Agent monitoring:

- `\$ZohoAgentProcessName` – set to `ZohoRPAAgent`
- `Test-ZohoAgent()` – uses `Get-Process` to see if the agent is running
- Uses `\$ZohoAlertCooldown` (300 seconds) to space alerts

### 4.2 Logging helper

`Write-Log` writes a timestamped message both to console and to:

```text
C:inary-proxy\service log\monitor_browser_YYYYMMDD.txt
```

This gives you a rolling daily log of all checks, detected states, and webhook attempts.

### 4.3 Webhook helper – human‑friendly alerts

`Send-WebhookNotification` takes two parameters: `Reason` and `Title`.

- Builds a customised **emailBody** based on the `Title`
- Titles currently handled include:
  - `"My Digital Accounts Needs a Quick Login"`
  - `"My Digital Accounts Browser Needs a Quick Restart"`
  - `"CRITICAL: File Monitor Service Heartbeat Lost"`
  - `"CRITICAL: ZOHO Agent Down on RPA VM"`
- Each body contains friendly, actionable steps, such as:
  - How to restart the browser (`Start_browser_new.bat`)
  - How to restart the File Monitor Service in Task Scheduler
  - How to bring the Zoho RPA Agent back up

All payloads are sent as JSON to Zoho Flow.

### 4.4 DevTools port check

```powershell
function Test-DevToolsPort {
    try {
        $tcp = New-Object System.Net.Sockets.TcpClient
        $tcp.Connect('localhost', 9222)
        $tcp.Close()
        Write-Log "DevTools port 9222 is accessible"
        return $true
    } catch {
        Write-Log "DevTools port 9222 is not accessible: $($_.Exception.Message)"
        return $false
    }
}
```

This confirms that Chrome is actually listening on port `9222` before trying to query the DevTools API.

### 4.5 Chrome + browser_state.json check

In each loop, the script:

1. Queries all `chrome.exe` processes via WMI (`Get-CimInstance Win32_Process`)
2. Filters to the instance that has `--remote-debugging-port=9222` in the command line
3. Reads `browser_state.json`:
   - Parses it as JSON
   - Confirms `wsEndpoint` starts with `ws://localhost:9222/`
4. Calls `Test-DevToolsPort` to ensure the socket is reachable

If any of these fail (no Chrome with the right flag, missing/invalid `wsEndpoint`, port not listening), the script treats the **browser state as failed**.

### 4.6 Login screen detection

If Chrome is running and `wsEndpoint` is valid and port 9222 responds:

1. The script calls `GET http://localhost:9222/json`
2. This returns a list of open tabs with their URLs
3. It looks for any tab where `url -like $MdaLoginUrl`

If found, it logs that the **MDA login screen is visible** and sends a one-off webhook with the title:

> `My Digital Accounts Needs a Quick Login`

This tells a human to log back in, keeping the automation alive.

### 4.7 Browser state tracking

Two internal state variables avoid spam:

- `\$lastBrowserState` – `"Running"` or `"Failed"`
- `\$lastLoginState` – `"LoggedIn"` or `"LoginRequired"`

Logic:

- If Chrome or `browser_state.json` is broken and `lastBrowserState -ne 'Failed'`:
  - Logs the problem
  - Sends: **"My Digital Accounts Browser Needs a Quick Restart"**
  - Sets state to `'Failed'`

- When the browser comes back and is healthy and `lastBrowserState -ne 'Running'`:
  - Logs the new PID and endpoint
  - Sets state back to `'Running'`

- When login is required and state changes from `'LoggedIn'` to `'LoginRequired'`, an alert is sent once
- When login screen disappears, state flips back to `'LoggedIn'`

### 4.8 File Monitor Service + Zoho Agent checks

On each full pass, if the configured cooldown has expired:

- `Test-FileManagerService`:
  - If heartbeat missing or too old:
    - Logs a **critical** message
    - Sends `"CRITICAL: File Monitor Service Heartbeat Lost"`
    - Explains how to restart the FileMonitorService task

- `Test-ZohoAgent`:
  - If the process is not running:
    - Logs a **critical** message
    - Sends `"CRITICAL: ZOHO Agent Down on RPA VM"`
    - Explains how to start the Zoho RPA Desktop Agent or run `Start_ZohoAgent.bat`

### 4.9 Grace period and main loop

When `monitor_browser.ps1` starts, it:

1. Logs a start message
2. Applies a **60‑second grace period** before the first check (to avoid alerting while services are still starting)
3. Enters an infinite loop that:
   - Performs all the checks described above
   - Logs results
   - Sleeps for `\$CheckIntervalSeconds` (default 60 seconds)

Any unhandled exception in the inner loop triggers an error log and, if state changed from running, a **“My Digital Accounts Browser Check Failed”** webhook.

A fatal error at the outer level logs and exits with code `1` so Task Scheduler can surface the failure.

---

## 5. Operational flow summary

```text
Windows Task Scheduler
    ├── Monitor Output Log
    │     └── monitor_output_log.ps1
    │            ├── Watches output_log_.log for low row counts
    │            └── Watches OneDrive flow folders for stuck files
    │
    └── MonitorBrowser
          └── monitor_browser.ps1
                 ├── Watches Chrome + browser_state.json + DevTools
                 ├── Detects My Digital Accounts login requirements
                 ├── Watches File Monitor Service heartbeat
                 └── Watches Zoho RPA Desktop Agent
```

Both monitor scripts write daily log files under:

```text
C:inary-proxy\service log
```

and use the same Zoho Flow webhook for human-readable notifications.

---

## 6. Troubleshooting

| Problem | Likely cause | Fix |
|--------|--------------|-----|
| No emails / alerts at all | Webhook URL invalid or outbound HTTPS blocked | Test `Invoke-RestMethod` manually with a small payload |
| “DTF Stalled” alerts but pipeline looks fine | `InactivityThresholdSec` too low or flow occasionally idle | Increase threshold or tune per-folder hints |
| Constant “Browser Needs a Quick Restart” | Chrome not started with `--remote-debugging-port=9222` or `browser_state.json` outdated | Restart the automation browser with the correct shortcut and regenerate `browser_state.json` |
| Constant “Needs a Quick Login” | MDA session expiry is frequent and no one logs in | Extend MDA session where possible or schedule regular manual logins |
| No stuck‑file alerts ever | `InactivityThresholdSec` too high or folders empty | Lower threshold and confirm files are being tracked in logs |
| Task Scheduler shows “running” but no logs | Permissions or working directory mismatch | Confirm the **Start in** folder is `C:\binary-proxy` and ExecutionPolicy is set to Bypass |

---

## 7. Conventions

- British English spelling  
- No Oxford commas  
- All scripts stored under `C:\binary-proxy`  
- Monitoring logs under `C:\binary-proxy\service log`

---

## 8. Ownership

© 2025 Geoffrey Jones. All rights reserved.  
Unauthorised copying or distribution of this material, in whole or in part, is strictly prohibited.
