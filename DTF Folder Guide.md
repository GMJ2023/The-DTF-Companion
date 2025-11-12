# 🗂️ DTF Folder Guide

---

Welcome!  
This guide explains how the **Payroll Automation folders** work — what goes where, what happens automatically, and where to find your processed files.  

It’s written as a simple, practical guide to help you see what happens behind the scenes — no technical detail needed.  
From the moment you drop a file into **Timesheets**, the **Zoho RPA Flow called “Transform Files”** takes over — sorting, converting, and preparing everything automatically.  

Each folder plays a specific part in this journey, quietly working in the background to keep your data organised, consistent, and always ready for upload to **My Digital Accounts**.

---

## 📁 Folder Layout

```text
C:\Users\zoho-admin\OneDrive - Sapphire Accounting Ltd\
│
├── Timesheets
├── XLSX Drop Zone
├── PDF Drop Zone
├── CSV Paradise
├── CSV Drop Zone
├── Holding Zone
├── AssembleXLSX
├── Formatted
├── Processed
└── Rejected
```

---

## `Timesheets` – Your Starting Point

This is **where everything begins**.  
Drop any timesheet here — whether it’s a **CSV**, **PDF**, **XLSX**, or **XLS** file — and the system will take care of the rest.

The **Zoho RPA Flow** called **Transform Files** monitors this folder and automatically sorts each file by type.

---

### 🧩 What the “Transform Files” Flow Does

| File Type      | What Happens                                                 |
| -------------- | ------------------------------------------------------------ |
| **PDF**        | Moved to the **`PDF Drop Zone`** for text extraction and conversion |
| **XLSX / XLS** | Moved to the **`XLSX Drop Zone`** where it’s converted to `.csv`, then returned to `Timesheets` |
| **CSV**        | Left untouched — ready for the next stage                    |

> The **Transform Files** flow ensures every timesheet ends up in the right place and in the correct format.

---

### 💻 What Happens Next

Once a **CSV file** is present in the **Timesheets** folder, it’s automatically picked up by the **DTF process** for uploading and transformation.  
DTF only accepts **CSV** files, so Excel and PDF files are always converted first by the **Transform Files** flow.

---

### What You Do

1. Save your timesheet into:  
   `C:\Users\zoho-admin\OneDrive - Sapphire Accounting Ltd\Timesheets`  
2. Wait a few seconds — the system will detect it and move it automatically.

---

### Behind the Scenes

| File Type      | Handled By                   | Action                                                       |
| -------------- | ---------------------------- | ------------------------------------------------------------ |
| **PDF**        | Zoho RPA – *Transform Files* | → Moved to **`PDF Drop Zone`** for OCR parsing               |
| **XLSX / XLS** | Zoho RPA – *Transform Files* | → Moved to **`XLSX Drop Zone`** → Converted to `.csv` → Returned to **`Timesheets`** |
| **CSV**        | DTF Process                  | → Picked up for upload and transformation                    |

---

### Example Flow

```text
You drop: MorganKing_April2025.xlsx
       ↓
Zoho RPA ("Transform Files") detects it
       ↓
Moved to XLSX Drop Zone → converted → MorganKing_April2025.csv
       ↓
CSV returned to Timesheets
       ↓
DTF picks it up → processed automatically
```

---

## 🧾 PDF Drop Zone – Smart PDF Parsing

After sorting, any **PDF timesheets** are sent straight here for conversion.  
The **PDF Drop Zone** is where the system reads, extracts, and converts PDF timesheets into clean, structured **CSV files** — ready to join the main data flow.

---

### ✨ How It Works

The system automatically:

1. **Reads the PDF** and extracts the text (even from scanned or password-protected files, when credentials are stored in Zoho CRM).  
2. **Parses the data** into a standard CSV layout.  
3. **Groups files by agency**, combining all PDFs from the same agency into **one CSV file** — instead of creating multiple separate files.  
4. The resulting combined CSV file is first created and stored safely in the **`CSV Paradise`** folder — the system’s working area for new CSVs.

This means that if you have **10 PDFs from the same agency**, you’ll end up with **one CSV file** containing 10 rows — one for each worker or timesheet — rather than 10 individual CSVs.

---

### 🔁 What Happens Automatically

| Step | Action                                                       |
| ---- | ------------------------------------------------------------ |
| 1    | PDF file dropped into `Timesheets`                           |
| 2    | Moved to `PDF Drop Zone` by the Transform Files flow         |
| 3    | System extracts text and parses it into CSV format           |
| 4    | CSVs from the same agency are **combined into one file** and created in `CSV Paradise` |
| 5    | Every 10 minutes, the scheduler checks for CSVs in `CSV Paradise` |
| 6    | When ready, the CSV is **moved to the `CSV Drop Zone`** to join the main processing flow |

---

### Example Flow

```text
10 PDFs dropped into Timesheets (same agency)
       ↓
Moved to PDF Drop Zone
       ↓
Parsed → Data extracted → Combined into 1 CSV file
       ↓
CSV created in CSV Paradise
       ↓
Scheduler checks every 10 mins
       ↓
CSV moved to CSV Drop Zone
       ↓
Joins normal data transformation process
```

---

## 🔧 Data Transformation & Agency Resolution

Once your CSV file has been created and placed back into the **Timesheets** folder, the system automatically begins the **data transformation process**.

Two **Zoho Catalyst Functions** work together to prepare your data for final upload:

| Function                | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| **Data Transformation** | Converts each agency’s timesheet data into a single, consistent master format. |
| **Resolve Agency Name** | Identifies and matches the correct agency name so that every file is standardised before upload. |

---

### What Happens Automatically

1. The initial `.csv` file from **Timesheets** is passed through both Catalyst functions.  
2. Once the file has been transformed and the agency name confirmed, it’s **saved to the `CSV Drop Zone`**.  
3. Inside `CSV Drop Zone`, the file is converted to **`.xlsx` format** and stored in **`AssembleXLSX`**.  

If a file with the **same name** already exists in `AssembleXLSX`, the new file is **diverted to the `Holding Zone`** until it’s safe to move.  
A scheduled background task checks the `Holding Zone` every **10 minutes**, automatically transferring files once the folder is clear.

The same logic also applies to files that are still in `.csv` format:  

- If the `CSV Drop Zone` already contains a file with the same name, the new CSV is **temporarily stored in `CSV Paradise`**.  
- The scheduler checks every **10 minutes** and moves it back to `CSV Drop Zone` when it’s safe to do so.

---

### Example Flow

```text
DTF finishes data transformation
       ↓
Catalyst – Data Transformation → Master CSV created
       ↓
Catalyst – Resolve Agency Name → Agency standardised
       ↓
File saved to CSV Drop Zone
       ↓
Converted to Excel (.xlsx) → Saved to AssembleXLSX
       ↓
If duplicate found → Diverted to Holding Zone (or CSV Paradise)
       ↓
Scheduler checks every 10 mins → Moves back when safe
```

---

## 🚦 Releasing Files to My Digital Accounts (MDA)

Once all transformations are complete and the `.xlsx` files are ready, the **Zoho Flow called “Release File to MDA”** takes over.  
This flow controls when files are allowed to move forward for upload to **My Digital Accounts**.

---

### 🕹️ Using the Zoho Flow “Release File to MDA” as a Virtual Pause Button

Sometimes it’s useful to temporarily pause automatic uploads — for example, while you review files or wait for confirmations.

The **Virtual Pause Button** in Zoho Flow allows this:  

- When **disabled**, the process pauses — files will stay safely in the **`AssembleXLSX`** folder.  
- When **enabled**, the flow resumes — files move automatically into the **`Formatted`** folder for upload.  

The system will safely queue files until you’re ready to continue.

---

### 📤 When the Flow Is Enabled

When the **“Release File to MDA”** flow is enabled:

1. Files in **`AssembleXLSX`** are automatically moved to the **`Formatted`** folder.  
2. From there, they are processed and uploaded through the **My Digital Accounts** website.  
3. Once upload and processing are complete, the files are transferred to the **`Processed`** folder automatically.

---

### Example Flow

```text
Release File to MDA – Enabled
       ↓
AssembleXLSX → Formatted
       ↓
Uploaded through My Digital Accounts
       ↓
Processed → Moved to final archive
```

---

## ✅ `Processed` – Completed & Archived

Once a file has been **successfully uploaded and processed** through the **My Digital Accounts** website, it reaches its final destination — the **`Processed`** folder.

This folder acts as a secure archive of all completed payroll files.  
Every file here has been fully transformed, validated, and uploaded — meaning the process for that timesheet is now officially **closed**.

---

### What Happens Automatically

| Stage                                     | Action                                          |
| ----------------------------------------- | ----------------------------------------------- |
| **File processed in My Digital Accounts** | Upload confirmed successfully                   |
| **Zoho Flow completes run**               | File moved automatically to `Processed`         |
| **Result**                                | File archived safely — no further action needed |

---

### What You Do

Nothing — once a file lands here, it’s all done ✅  
You can open it for reference, but there’s no need to move or rename it.  
The next payroll cycle will begin automatically when new timesheets are dropped into the **`Timesheets`** folder.

---

### Example Flow

```text
File uploaded through My Digital Accounts
       ↓
Processing confirmed
       ↓
File moved automatically to Processed
       ↓
Process complete – ready for the next cycle
```

---

### 🎯 In Summary

From start to finish, every folder has its role.  
The system quietly handles conversions, checks, and uploads —  
everything happens automatically, with no manual steps or missed files. ✅
---

Copyright © 2025 Geoffrey Jones
All rights reserved.

This repository and its contents are proprietary and confidential.
No part may be copied, modified, or distributed without prior written consent.
