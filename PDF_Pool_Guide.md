# PDF Pool Guide

A complete operational and developer guide for the **PDF Pool** automation framework — a modular system that extracts structured CSV data from agency remittance and invoice PDFs, coordinated by PowerShell scripts and Node.js parsers.

> Tested on Windows with Node.js and Zoho RPA orchestration. Designed for hands-off operation via monitored PDF folders.

---

## Overview

**PDF Pool** automates the end-to-end pipeline from incoming PDF to ready-to-import CSV.

### Core flow

```
File Monitor (RPA or Windows Watcher)
        ↓
run_pdf_parser_wrapper.ps1
        ↓
write_text_and_invoke.ps1
        ↓
Catalyst pdfPool_node function
        ↓
flexParser / index_4Rec / index_TXM / TXMParser
        ↓
CSV Paradise (OneDrive)
```

---

## File Structure

```
C:\binary-proxy
│
├─ run_pdf_parser_wrapper.ps1    # Entry point, handles locking, logging, and file lifecycle
├─ write_text_and_invoke.ps1     # Writes text file and invokes Catalyst function
├─ flexParser.js                 # Utility for decloaking and row extraction (4Rec)
├─ index_4Rec.js                 # 4 Recruitment entry point
├─ index_TXM.js                  # TXM entry point
├─ TXMParser.js                  # TXM parser logic
├─ smart_parser.js               # Detects agency and routes PDFs to correct parser
├─ file_monitor_service.ps1      # Reference watcher (RPA trigger)
└─ PDF_Pool_Guide.md             # This document
```

---

## Main Entry Point — `run_pdf_parser_wrapper.ps1`

This script is the **recommended entry point** for all automated PDF parsing.  
It manages the entire execution chain safely and ensures that only one PDF is processed at a time.

### Key responsibilities

1. **Global Lock Handling**  
   - Creates a global lock file (`$env:TEMP\global_pdf_parser.lock`) before any parsing begins.  
   - Prevents concurrent runs of the parser when multiple PDFs arrive simultaneously.  
   - Deletes the lock file automatically after completion, even if an error occurs.

2. **Per-File Locking**  
   - Creates a `.lock` file next to each active PDF (e.g. `MyFile.pdf.lock`).  
   - Ensures no duplicate processing for the same file.  
   - Automatically cleans up these locks when the file is archived.

3. **Invocation Flow**  
   - Detects a new PDF (triggered by RPA or `file_monitor_service.ps1`).  
   - Calls `write_text_and_invoke.ps1` with path and filename arguments.  
   - Waits for that process to complete before continuing.  
   - Moves the processed PDF to an **archive** folder once finished.

4. **Error Handling**  
   - Logs all actions and errors to a timestamped `.log` file.  
   - On failure, removes per-file locks and restores the global lock state.  
   - Displays a concise summary at the console for debugging.

---

## Node.js Components

Each agency parser implements format-specific logic. The `smart_parser.js` router auto-detects which parser to use based on header text.

| File | Description |
|------|--------------|
| **flexParser.js** | Common parser utility for 4Rec, includes decloak, float repair, and row validation. |
| **index_4Rec.js** | Extracts member data and VAT handling for 4 Recruitment Services Limited. |
| **index_TXM.js** | Parses TXM invoices, handles deductions, and writes to TXMRecruit.csv. |
| **TXMParser.js** | Helper for line-by-line extraction and numeric sanity checks. |
| **smart_parser.js** | Detects agency type and dispatches to the correct entry point. |

---

## Output

All parsers write CSVs to:

```
C:\Users\zoho-admin\OneDrive - Sapphire Accounting Ltd\CSV Paradise
```

Each CSV uses an **append-safe** mode. Headers are written automatically on first creation.

### CSV Schema

| Column | Description |
|---------|-------------|
| employeeid | Worker ID or contractor reference |
| firstname | Extracted first name |
| surname | Extracted surname |
| weekending | End-of-week date |
| paymentdate | Payment date from PDF |
| amount | Amount paid per line |
| rate | Hourly rate or per-unit rate |
| totalpaid | Total payment value |
| agencyname | Agency name from header |
| paymentmethod | Blank or `Unit` for expenses/deductions |
| description | Optional text for remarks |
| vatamount | Included in TXMRecruit.csv (fixed column) |

---

## Developer Breakdown

### 1. Lock Handling

- A **global lock file** prevents multiple PDFs from running simultaneously.  
- Each PDF also generates a **per-file lock**, allowing safe queued processing.  
- If the system halts, stale locks can be manually cleared using:  

```powershell
Remove-Item $env:TEMP\global_pdf_parser.lock -ErrorAction SilentlyContinue
```

### 2. Invocation Flow

```
run_pdf_parser_wrapper.ps1
   ├─ Create global lock
   ├─ Detect new PDF
   ├─ Create per-file lock
   ├─ Call write_text_and_invoke.ps1
   ├─ Await completion
   ├─ Move PDF to archive
   └─ Remove locks and exit
```

### 3. File Move / Cleanup

After successful processing:

- The PDF is moved to an **Archive** directory for audit purposes.  
- A `.log` entry records the file name, timestamp, and output CSV name.  
- Locks are removed cleanly.

### 4. Error Recovery

- If a failure occurs during Catalyst invocation or file move, the wrapper catches it.  
- Lock files are deleted to prevent deadlocks.  
- Logs record the full error chain for debugging.

---

## Example Execution (Manual Test)

```powershell
# From C:\binary-proxy
.
un_pdf_parser_wrapper.ps1
```

The script will automatically detect the newest PDF in the configured Drop Zone, process it, and write CSVs to CSV Paradise.

---

## File Monitor Reference — `file_monitor_service.ps1`

This script is **only for reference** and is not required when using the wrapper.  
It shows how an RPA workflow or Windows service might detect PDFs and call the wrapper.

```powershell
$watch = New-Object IO.FileSystemWatcher "C:\PDF Drop Zone", "*.pdf"
$watch.EnableRaisingEvents = $true
Register-ObjectEvent $watch Created -Action {
  Start-Process powershell -ArgumentList "-File 'C:\binary-proxy\run_pdf_parser_wrapper.ps1'"
}
```

---

## Troubleshooting

| Issue | Resolution |
|--------|-------------|
| Script hangs | Check and remove `$env:TEMP\global_pdf_parser.lock` |
| CSV not generated | Verify PDF text is selectable and supported by one of the parsers |
| Duplicate CSV entries | Ensure the wrapper is not triggered twice for the same file |
| “PDF path invalid” | Confirm correct path passed to `write_text_and_invoke.ps1` |

---

## Conventions

- British English spelling  
- No Oxford commas  
- Windows-first file paths

---

## Roadmap

- Add additional agency-specific parsers  
- Integrate PDF image OCR fallback  
- Implement retry queue for transient errors  
- Add optional email notification for failures

---

## License

Private internal project. Replace with a licence file before public distribution.

