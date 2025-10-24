# 🧩 Mapping Template Guide

### Managing Multiple CSV Mappings in the Data Transformation Framework (DTF)

------

## 📘 Overview

Some agencies may provide more than one file format or header layout — for example, weekly vs monthly exports, or data from different branches using the same base name.
 The `_ik+` suffix allows you to support multiple mapping templates **for the same agency** without breaking the Data Transformation Framework (DTF) matching logic or CRM lookups.

------

## 🧠 Syntax

Append `_ik+<number>` to the **TemplateName** column in your `mapping-template.csv`.

| Example        | Meaning                                  |
| -------------- | ---------------------------------------- |
| `AgencyA`      | Default mapping template for this agency |
| `AgencyA_ik+1` | Secondary mapping for the same agency    |
| `AgencyA_ik+2` | Tertiary mapping for the same agency     |

------

## 🧾 Important Note

There is **no requirement** to include agencies that are handled via **PDF processing** in this mapping file.
 The `mapping-template.csv` only applies to agencies submitting **CSV or Excel-based data**.
 Agencies whose data arrives in PDF form are processed separately through the **PDF Parser** component of DTF, which automatically extracts and formats their information using pre-defined parsing logic — no mapping rows are needed.

------

## ⚙️ How It Works

1. During transformation, DTF detects templates such as `AgencyName_ik+1` or `AgencyName_ik+2`.
2. It uses that variant mapping file **for column transformation only**.
3. When passing the file to CRM for fuzzy name resolution, the `_ik+<number>` part is automatically **removed**, ensuring clean matching (e.g. `AgencyA_ik+1 → AgencyA`).
4. The `resolveAgencyName_node` logs both values — the “clean” name for CRM, and the original variant for internal tracking.

------

## ⚙️ Transformation Rules

Each row in the mapping template includes a **TransformationRule** that tells DTF how to handle the data from the input file before it is written to the master format.
 These rules are interpreted by the transformation logic within `datatransformation_node`.

### 🧩 Supported Rules

| Rule                | Description                                                  | Example                   |
| ------------------- | ------------------------------------------------------------ | ------------------------- |
| **Direct**          | Passes the value through exactly as found in the input CSV.  | `Pay rate → rate`         |
| **SplitR:***Token*  | Splits a field into parts and takes the **right-hand** element. Common for names written as `Jones, Geoffrey` or fields with multiple bits of data in one cell. | See examples below        |
| **SplitL:***Token*  | Splits a field into parts and takes the **left-hand** element. Common for names written in natural order like `Geoffrey Jones`. | See examples below        |
| **Default:***Value* | Inserts a fixed value into the output cell. Use `Default:` followed by the value you want, or leave blank (`Default:`) to produce an empty cell. | `Default:1` or `Default:` |
| **Dynamic**         | Used when a sheet is laid out horizontally (with multiple rate types or headings running left to right). The column header itself becomes the variable — e.g. `OT1`, `OT2`, etc. | `Dynamic → OT1`           |

------

### 🧩 **Split Examples**

#### **Example 1 — SplitR (comma-delimited input)**

If the input cell contains:

```
Jones, Geoff
```

You can map it using:

```
Worker SplitR:surname
Worker SplitR:firstname
```

DTF will output:

| Field     | Value |
| --------- | ----- |
| firstname | Geoff |
| surname   | Jones |

------

#### **Example 2 — SplitL (space-delimited input)**

If the input cell contains:

```
Geoff Jones
```

You can map it using:

```
Worker SplitL:Firstname
Worker SplitL:Surname
```

DTF will output:

| Field     | Value |
| --------- | ----- |
| firstname | Geoff |
| surname   | Jones |

------

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

| Field      | Value  |
| ---------- | ------ |
| employeeid | A1234B |
| firstname  | Geoff  |
| surname    | Jones  |

This flexible split logic allows DTF to extract structured data even from inconsistent or combined input fields.

------

### 🧠 How DTF Applies Rules

1. For each row in the mapping template, DTF reads the **InputColumn** value.
2. If a `TransformationRule` exists, the rule is applied to modify or format the data.
3. If no rule is defined, the transformation defaults to **Direct**.
4. Rules can be combined across multiple rows to produce fully structured output, even when input data is unclean.

------

💡 *If your agency sheet looks like chaos, DTF’s transformation rules will still bring order to it — one cell at a time.*