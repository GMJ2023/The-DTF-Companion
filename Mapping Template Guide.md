# 🧩 Mapping Template Guide  
### Managing Multiple CSV Mappings in the Data Transformation Framework (DTF)

---

## 📘 Overview  
Some agencies may provide more than one file format or header layout — for example, weekly vs monthly exports, or data from different branches using the same base name.  
The `_ik+` suffix allows you to support multiple mapping templates **for the same agency** without breaking the Data Transformation Framework (DTF) matching logic or CRM lookups.

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

## ✅ Benefits  

- Enables multiple mapping formats per agency  
- Prevents CRM confusion and duplicate account creation  
- Simplifies testing and parallel format transitions  
- Makes log inspection easier (`variantUsed` identifies which map was triggered)  

---

## 🖼️ Visual Example  
![template-mapping](https://github.com/GMJ2023/assets/blob/main/mapping-template-example.jpg)

If you’re viewing this guide on GitHub, you’ll see a sample screenshot of the `mapping-template.csv` file showing `_ik+` usage:

