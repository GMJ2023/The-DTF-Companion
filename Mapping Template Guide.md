# 🧩 Mapping Template Guide  
### Managing Multiple CSV Mappings in the Data Transformation Framework (DTF)

---

## 📘 Overview  
Some agencies may provide more than one file format or header layout — for example, weekly vs monthly exports, or data from different branches using the same base name.  
The `_ik+` suffix allows you to support multiple mapping templates **for the same agency** without breaking the Data Transformation Framework (DTF) matching logic or CRM lookups.  

Each mapping also defines a **TransformationRule**, which tells DTF how to interpret and reshape the data from the input file — for example, splitting names, applying defaults, or generating dynamic columns.  
Together, the `_ik+` suffix and the `TransformationRule` system give you complete control over how data is recognised and normalised before upload.

---

## 🧠 Syntax  
Append `_ik+<number>` to the **TemplateName** column in your `mapping-template.csv`.  

| Example        | Meaning                                  |
| -------------- | ---------------------------------------- |
| `AgencyA`      | Default mapping template for this agency |
| `AgencyA_ik+1` | Secondary mapping for the same agency    |
| `AgencyA_ik+2` | Tertiary mapping for the same agency     |

---

## 🧾 Important Note  

There is **no requirement** to include agencies that are handled via **PDF processing** in this mapping file.  
The `mapping-template.csv` only applies to agencies submitting **CSV or Excel-based data**.  
Agencies whose data arrives in PDF form are processed separately through the **PDF Parser** component of DTF, which automatically extracts and formats their information using pre-defined parsing logic — no mapping rows are needed.

---

## ⚙️ How It Works  

1. During transformation, DTF detects templates such as `AgencyName_ik+1` or `AgencyName_ik+2`.  
2. It uses that variant mapping file **for column transformation only**.  
3. When passing the file to CRM for fuzzy name resolution, the `_ik+<number>` part is automatically **removed**, ensuring clean matching (e.g. `AgencyA_ik+1 → AgencyA`).  
4. The `resolveAgencyName_node` logs both values — the “clean” name for CRM, and the original variant for internal tracking.  

---

## ⚙️ Column Structure  

Each row in the `mapping-template.csv` defines how data is converted from the source file into the DTF master format.  

| Column | Description |
|---------|--------------|
| **TemplateName** | The agency name or variant (e.g. `AgencyA`, `AgencyA_ik+1`). |
| **InputColumn** | The name of the column from the agency’s CSV file. |
| **OutputColumn** | The target column in the DTF master CSV. |
| **TransformationRule** | The logic applied to modify or split values before writing to the output. |
| **headerRowNumber** | The row number where the **column headers** are found in the input CSV. |
| **firstRowNumber** | The row number where the **actual data** begins in the input CSV. |

---

## ⚙️ Transformation Rules  

Each row in the mapping template includes a **TransformationRule** that tells DTF how to handle the data from the input file before it is written to the master format.  
These rules are interpreted by the transformation logic within `datatransformation_node`.

---

### 🧩 Supported Rules

| Rule | Description | Example |
|------|--------------|----------|
| **Direct** | Passes the value through exactly as found in the input CSV. | `Pay rate → rate` |
| **SplitR:**_Token_ | Splits a field into parts and takes the **right-hand** element. Common for names written as `Jones, Geoffrey` or fields with multiple bits of data in one cell. | See examples below |
| **Split:**_Token_ | Splits a field into parts and takes the **left-hand** element. Common for names written in natural order like `Geoffrey Jones`. | See examples below |
| **Default:**_Value_ | Inserts a fixed value into the output cell. Use `Default:` followed by the value you want, or leave blank (`Default:`) to produce an empty cell. | `Default:value` or `Default:` |
| **Dynamic** | Used when a sheet is laid out horizontally (with multiple rate types or headings running left to right). The column header itself becomes the variable — e.g. `OT1`, `OT2`, etc. | `Dynamic → OT1` |

---

### 🧩 **Split Examples**

#### **Example 1 — SplitR (comma-delimited input)**  
If the input cell contains:  
```
Jones, Geoffrey
```
You can map it using:  
```
Worker SplitR:surname
Worker SplitR:firstname
```
DTF will output:  
| Field | Value |
|--------|--------|
| firstname | Geoffrey |
| surname | Jones |

---

#### **Example 2 — Split (space-delimited input)**  
If the input cell contains:  
```
Geoffrey Jones
```
You can map it using:  
```
Worker Split:Firstname
Worker Split:Surname
```
DTF will output:  
| Field | Value |
|--------|--------|
| firstname | Geoffrey |
| surname | Jones |

---

#### **Example 3 — Three-part mixed cell**  
If the input cell contains:  
```
A1234B Jones, Geoff
```
You can map it using:  
```
Worker SplitR EmployeeID
Worker SplitR:surname
Worker SplitR:firstname
```
DTF will output:  
| Field | Value |
|--------|--------|
| employeeid | A1234B |
| firstname | Geoff |
| surname | Jones |

This flexible split logic allows DTF to extract structured data even from inconsistent or combined input fields.

---

### 🧠 How DTF Applies Rules

1. For each row in the mapping template, DTF reads the **InputColumn** value.  
2. If a `TransformationRule` exists, the rule is applied to modify or format the data.  
3. If no rule is defined, the transformation defaults to **Direct**.  
4. Rules can be combined across multiple rows to produce fully structured output, even when input data is unclean.

---

💡 *If your agency sheet looks like chaos, DTF’s transformation rules will still bring order to it — one cell at a time.*

---

## 🖼️ Visual Example  
![template-mapping](https://github.com/GMJ2023/assets/blob/main/mapping-template-example.jpg)

If you’re viewing this guide on GitHub, you’ll see a sample screenshot of the `mapping-template.csv` file showing `_ik+` usage.

---

**Authored by Geoffrey Jones**  
*Part of The DTF Companion — a growing collection of guides and documentation for the Data Transformation Framework.*

---
**Copyright © 2025 Geoffrey Jones**

**All rights reserved.**

This codebase and all associated materials are proprietary to Geoffrey Jones.
 No part of this work may be copied reproduced modified distributed or used for any purpose other than the specific engagement for which access was granted.
 Any other use requires prior written consent from the copyright holder.
