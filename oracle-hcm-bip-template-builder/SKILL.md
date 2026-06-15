---
name: oracle-hcm-bip-template-builder
description: "Build BI Publisher Excel templates (.xlsx) for Oracle HCM extracts. Use this skill whenever the user wants to: build a BIP Excel template, create a Config Matrix or BR100 template, align a template's columns with the fields present in an HCM extract, configure XDO_METADATA bindings, set up a repeating-row layout, or produce a BIP-ready workbook from an HCM extract definition. Also triggers on: 'align Config Matrix with extract', 'XDO_METADATA', 'XDO_GROUP', 'BIP Excel template', 'template alignment', 'tag-binding row', 'cell-to-xpath'. This skill covers the .xlsx TEMPLATE layout and binding — not BIP SQL authoring, HCM Extract definitions, or Oracle AI Agent JSON generation."
---

# Oracle HCM BIP Excel Template Builder

You are building `.xlsx` workbooks that BI Publisher renders against an HCM extract XML to produce populated Config Matrix / BR100 output files. This skill encodes layout rules, XDO_METADATA conventions, alignment workflows, and visual-standards guidance learned through verified BIP deployments on Oracle Fusion Cloud.

This is **not** about writing BIP SQL, defining HCM Extract structures, or building Oracle AI Agent JSON. It is specifically about the template file that BIP binds to an extract and renders.

---

## Portability & Fallbacks

This skill is **fully self-contained**. It requires:

- No other skill, no "superpowers" framework, no Skill tool invocations.
- Standard Python libraries only: `openpyxl` (or `xlsxwriter`) for `.xlsx`; `python-docx` for `.docx` — but see **[Render path](#render-path--design-vs-server-driving-template-critical)**: a template that must actually drive server-side BIP rendering requires the Oracle Java API, not openpyxl/xlsxwriter.
- No special Oracle SDK or BIP client library (except when producing a server-driving template — see Render path section below).

**Optional project files** — if these exist at the project root, use them as the source of truth; if they do not exist, fall back to the structural rules in this document:

| File | Role | Fallback |
|---|---|---|
| `EXTRACT_TAG_MAPPING.json` | Maps Config Matrix columns → XML tags, ID lookups, Phase 2 flags | Derive bindings from extract XML structure directly |
| Reference template `.xlsx` | Canonical shape example | Build from the layout rules below |

This skill works identically in Claude Code, Codex, Copilot, Gemini, or a plain LLM context.

---

## BI Publisher Excel Template — Build Recipe

When the user asks for a "BIP Excel template for an HCM extract" (Workforce Structures, Payroll, Benefits, or any other module), they want a single `.xlsx` that BI Publisher renders against the extract XML to produce a populated Config Matrix / BR100 output. This is **not** the Config Matrix sample data file — that is the *output*; this recipe builds the *template*.

### Required workbook shape

```
Cover                  ← BR100-style cover sheet (visible)
<Block 1>              ← one visible sheet per extract block (e.g. Jobs)
<Block 2>
...
XDO_METADATA           ← hidden sheet — BIP control plane
```

### Per-data-sheet row layout (consistent across all sheets)

```
Row 1  Title bar (brand primary-dark fill, merged across columns)
Row 2  Meta line: "Export Date: <?EXPORT_DATE?>" + "Total Records: <?count(.)?>" + "Source: …"
Row 3  Column headers (brand primary-accent fill, white bold, centered, wrap-text)
Row 4  Tag-binding documentation row — INFORMATIONAL only, NOT BIP-bound (neutral-fill grey)
Row 5  *** BIP repeating data row *** — every cell is `<?XmlTag?>`. XDO_GROUP_<sheet> targets this row.
Row 6+ Footer notes (XPath group path, ID-resolution caveats, Phase 2 fields)
```

The repeating row is **row 5**, not row 2 like a legacy raw-tag dump template. Row 4 stays so the file is self-documenting; BIP renders it once at the top of the output.

### XDO_METADATA layout

```
Row 1   Version
Row 2   ARU-dbdrv
Row 3   Extractor Version
Row 4   Template Code            <YOUR_TEMPLATE_CODE>
Row 5   Template Type            TYPE_EXCEL_TEMPLATE
Row 6   Preprocess XSLT File
Row 7   Last Modified Date
Row 8   Last Modified By
Row 9   <blank>
Row 10  Data Constraints
Row 11+ Per sheet:
        XDO_GROUP_<Sheet>           <?for-each:Root/<Block>/<Row>?>
        '<Sheet>'!A5                <?XmlTag1?>
        '<Sheet>'!B5                <?XmlTag2?>
        ...
```

Each sheet gets a `XDO_GROUP_<Sheet>` row immediately followed by one cell-binding row per column. Cell references use the **single-quoted sheet name** form `'Jobs'!A5` (required by BIP — unquoted breaks if the sheet name contains a space).

### Cell-to-xpath rules

- **Direct field:** `'Jobs'!A5` → `<?JobCode?>`
- **ID column** (PositionJobID, BULegalEntityID, etc.): emit the raw ID → `<?PositionJobID?>`. Resolution to a human-readable name happens downstream (in a mapping agent or post-processing step), not in BIP. Document the lookup target in row 4 (e.g. `PositionJobID → Jobs.JobName`).
- **Phase 2 placeholder:** `<?'— Phase 2 —'?>` (literal string). Keep the column so downstream consumers can rely on column ordering.
- **Aggregate / count fields:** use BIP XPath functions (`<?count(.)?>`, `<?sum(...)?>`) — not Excel formulas.

### Cover sheet (BR100 cover-page parity)

| Field        | Value                          |
|--------------|--------------------------------|
| Client       | `<?CLIENT_NAME?>`              |
| Module       | (set per customer / module)    |
| Object       | (set per extract scope)        |
| Version      | 1.0                            |
| Date         | `<?EXPORT_DATE?>`              |
| Prepared By  | (set per customer)             |
| Status       | Draft                          |
| Source       | HCM Extract (BIP) + REST API supplements |

Include a sheet-index table summarizing block paths and Phase 2 inventory.

### Common mistakes to avoid

- **Wrong row for repeating data.** If you put `<?xpath?>` cells in row 2 with no spacer, BIP treats the meta-line cells as part of the loop. Put the repeating row at row 5.
- **Using `/latest/` XPath roots.** BIP XPaths against the extract use the actual block names (e.g. `Work_Structures/Job_Details/JobDetails`), never a version segment.
- **Missing single quotes on sheet names with spaces.** `'Business Units'!A5`, not `Business Units!A5`.
- **Trying to do ID→Name resolution inside BIP.** Using `<?xdoxslt:set_variable/get_variable?>` for cross-block lookups is fragile and breaks for circular references. Keep BIP dumb: emit IDs, resolve downstream.
- **Hidden sheet not actually hidden.** Set `ws.sheet_state = 'hidden'` explicitly on `XDO_METADATA` — `xlsxwriter`/`openpyxl` defaults vary.
- **Forgetting `Template Type: TYPE_EXCEL_TEMPLATE`.** Without it, BIP rejects the upload.

### Verification before declaring done

- Open the `.xlsx` in Excel; XDO_METADATA should be hidden but accessible via right-click → Unhide.
- Every `XDO_GROUP_<Sheet>` row in XDO_METADATA must have at least one corresponding cell-binding row immediately below it.
- Every cell reference in XDO_METADATA must point to row 5 (or the repeating row) of the named sheet.
- Every cell on the data sheet's row 5 must contain a `<?...?>` token (or a literal placeholder for Phase 2).
- Reference: https://docs.oracle.com/middleware/12211/bip/BIPRD/GUID-0A62F78E-3B19-4BA6-8C9F-2757D2104782.htm

### Reference implementation

If a reference template `.xlsx` exists in the project (e.g. `BIP_Template_Work_Structure_Extract.xlsx` at the project root), use it as the canonical example for shape, binding count, and Phase 1 vs Phase 2 column split. If no reference template exists, build from the layout rules above — they are sufficient to produce a valid BIP-importable workbook.

---

## Render path — design vs server-driving template (critical)

There are **two distinct goals** when building a BIP Excel template. Confusing them causes hours of silent-failure debugging.

### (a) Design / preview — openpyxl or xlsxwriter is fine

Use Python (`openpyxl`, `xlsxwriter`) to design the layout, validate column bindings, check XDO_METADATA structure, and preview the workbook visually. This is the correct tool for the work described in the Build Recipe section above.

### (b) A template that actually drives server-side BIP rendering — requires the Oracle Java API

If the `.xls` file must be **uploaded to Fusion Cloud BIP and rendered by the BIP server**, it must be produced via Oracle's `oracle.xdo.excel.user.*` Java API (`xdo-core.jar` + a JDK). **Do not use openpyxl + LibreOffice/soffice for this step.**

**Why:** openpyxl+soffice BIFF8 output fails silently on the Fusion Cloud BIP server — the server returns an HTML error page (body ≈ 1746 bytes, `mXSLFile null`) with no useful diagnostic, even when the file is byte-structurally identical to a known-good Oracle sample. Switching the same template to Oracle's `oracle.xdo.excel.user.Workbook` writer produces a BIP-compatible BIFF8 file that the server accepts on first attempt. The local BIP engine (`xdo-core.jar` standalone) is more permissive than the server — a successful local render is **not** proof of server compatibility.

### Defined Names are the mechanism — without them the template is visual-only

BIP locates each repeating region via workbook-global Defined Names (`XDO_?<GroupTag>_<FieldTag>?` + `XDO_GROUP_?<GroupTag>?`). A workbook produced by openpyxl/xlsxwriter that omits or mis-formats these names looks visually correct but **cannot drive rendering** — it is a visual-only prototype. Defined Names must be written by the Oracle Java API writer to survive the BIP server's XSL preprocessing stage.

**Forking rule:** When building a new version of a server-driving template, copy the working Java builder source file — never hand-edit the rendered `.xls` chassis or rebuild the chassis with Python. Preserve the Defined Name pattern bit-for-bit. Any change that shifts the `DATA_ROW` offset must move every Defined Name range to match.

### Verification — "rendered without error" + "rows returned" is not proof

A render that returns a file with no ORA errors and a non-zero row count can still be functionally blank. Placeholder-string rows (e.g. `SELECT 'sheet: no data' FROM DUAL`) pass row-count checks but render literal strings instead of data. Before declaring a template done:

1. Confirm the BIP server returns `application/vnd.ms-excel` with magic bytes `D0 CF 11 E0 A1 B1 1A E1` and a file size well above the error-page threshold (≥ 10 KB for a trivial template; typically several MB for real data).
2. Open the rendered output and **sample 2–3 actual data rows from every sheet** — not just headers. Flag any sheet where the data band contains only uppercase-underscore column-tag strings (that is a placeholder row, not real data).
3. Treat `MISSING_TABLE` warnings in any pre-flight audit as blocking — do not dismiss them as "must exist on the pod"; verify before uploading.

### Advanced last-resort: byte-level surgery on an existing working `.xls`

If the Oracle Java build environment (`xdo-core.jar`, chassis `.xls`, bindings TSV) is unavailable but a minor structural edit is needed on an already-working server-renderable `.xls`, in-place binary surgery is viable:

- Open the `.xls` via `olefile.OleFileIO`, read the `Workbook` stream into a `bytearray`.
- Walk BIFF8 records (2-byte type + 2-byte length + payload) and make **same-size edits only** — adding or removing bytes shifts `BOUNDSHEET.lbPlyPos` and internal `INDEX`/`DBCELL` offsets, breaking the file.
- Safe pattern: replace a `LABELSST` cell record (14 bytes) with a `BLANK` record (10 bytes) + a 4-byte no-op filler (type `0xFFEE`, length 0).
- Write back with `OleFileIO(out, write_mode=True).write_stream(['Workbook'], bytes(modified))` — stream size must equal original.
- After editing, verify the Defined Name count is unchanged (e.g. via `xlrd.open_workbook`). If it drops, the write failed silently.
- **Avoid `xlutils.copy` + `xlwt.save`** — this toolchain strips all Defined Names (verified: 1196 → 0), making the file useless for BIP.
- This is a last-resort patch path only. The correct fix is to rebuild the Java environment and regenerate via the Oracle API.

---

## Template Alignment (Config Matrix ↔ Extract)

Every BIP project has the same loop: a Config Matrix (or BR100, or BIP-driven Excel) gets defined optimistically, then someone discovers half the columns are not in the extract. The fix is mechanical, not creative.

### The 5-step alignment workflow

1. **Load the canonical mapping.** If `EXTRACT_TAG_MAPPING.json` exists at the project root, it is the source of truth for which Config Matrix columns map to which XML tags, which are ID lookups, and which are Phase 2. If it does not exist, derive bindings directly from the extract XML structure (inspect actual extract output or schema documentation).
2. **Inspect the current template.** For `.xlsx` use `openpyxl`; for `.docx` use `python-docx`. Dump every sheet/section's header row to compare against the mapping.
3. **Rewrite each sheet/section** so:
   - Header row (row 3) = the column names from `mapping[].matrix_col`.
   - Add a **second header row immediately below** (the "tag-binding row", row 4) that prints the `xml_tag` (or REST endpoint, or `— Phase 2 (...)` placeholder). This row is what makes the template self-documenting — anyone reading the file sees the binding without opening JSON.
   - Repeating data row(s) (row 5) with `<?xpath?>` tokens.
   - Footer note giving the XML root path.
4. **Mark unsupported fields with `— Phase 2 (reason)`** in the tag-binding row and leave the data cell empty. Do not delete the column; downstream consumers may rely on column ordering.
5. **Refresh the Index sheet** (workbook only) with: source (`Extract` vs `REST`), block name + tag count, and any Phase 2 fields per sheet.

### Why row 4 (tag-binding) matters

Without it, the template silently falls out of sync with the extract every time Oracle adds or renames a tag. A downstream mapping agent (or any consumer) can read row 4 as its XPath dictionary — making the template both human-readable AND machine-consumable. **Always include it.**

### Example: a Workforce Structures workbook

The table below is a concrete worked example for a Workforce Structures extract. Use it as a reference for sheet count, binding depth, and which sheets require REST joins vs direct extract tags. It is not the only supported configuration — the same patterns apply to Payroll, Benefits, Talent, or any other HCM module extract.

| Sheet | Block / Endpoint | Status |
|---|---|---|
| Jobs | `Job_Details` (34 tags) | Direct + JobFamilyID lookup |
| Job Families | `Job_Family_Details` or REST `/jobFamilies` | Derived |
| Positions | `Position_Details` (49 tags) | Direct + 4 ID lookups (Job, Dept, Grade, Location) |
| Grades | `Grade_Details` (27 tags) | Direct |
| Grade Rates | REST `/hcmRestApi/.../gradeRates` | Not in extract — REST join on GradeCode |
| Departments | `Department_Details` (123 tags) | Direct + DeptLocationID lookup; Manager = Phase 2 |
| Locations | `Location_Details` (69 tags) | Direct |
| Business Units | `Business_Unit_Details` (123 tags) | Direct + BULegalEntityID lookup; Primary Ledger / Default Currency = Phase 2 (ERP) |
| Legal Entities | `Legal_Entity_Details` (121 tags) | Direct; Registration Number = Phase 2 |

### Quick verification

After alignment:
- Every column in row 3 has a non-empty cell in row 4 (either an XML tag, a REST path, or a Phase 2 placeholder).
- No column references a field that is absent from `EXTRACT_TAG_MAPPING.json` (or the extract schema, if the file does not exist).
- Phase 2 columns have empty data cells — do not invent values.
- Index sheet's "Sheet directory" matches the actual workbook sheet order.

---

## Visual Standards — brand palette (replace per customer)

> **These tokens are an EXAMPLE palette (NTT DATA).** Replace the hex values with the customer's brand palette. The **structure** — which row gets which fill, which role each colour serves — is the portable part. The specific hex values are not a mandate; they are a worked example.
>
> If no brand palette is supplied, use a conservative default: dark navy title bar (`#1A2744`), mid-blue column headers (`#0055A0`), light-grey tag-binding rows (`#E8E8E8`), white page background (`#FFFFFF`), black body text (`#000000`).
>
> Do not mix in unrelated brand palettes (e.g. a different customer's colours, or a vendor's internal brand). Confirm the exact customer palette before applying any non-default colours.

### Palette roles and their structural application

| Role | Function | Applied to |
|---|---|---|
| `primary-dark` | Title bars, strongest brand anchor | Row 1 fill |
| `primary-accent` | Column headers, XPath token colour | Row 3 fill; `<?xpath?>` cell font colour |
| `primary-accent-light` | Hyperlinks, light accents | Hyperlinks, secondary UI elements |
| `secondary-accent` | Optional highlight, secondary brand | Supplementary callouts (use sparingly) |
| `warning` | Attention, phase-2 highlights | Phase 2 tag-binding cells (select cases) |
| `warning-light` | Soft warning fills | Phase 2 placeholder cells |
| `strong-attention` | Strong alerts (use sparingly) | Critical error cells only |
| `neutral-border` | Borders, muted UI | Cell borders throughout |
| `neutral-fill` | Tag-binding rows, background fills | Row 4 fill |
| `secondary-text` | Footnotes, meta lines | Row 2 text, footer notes |
| `body-text` | Primary body text | Data rows |
| `page-bg` | Cell background | All non-filled cells |

### Structural application rules (palette-role language)

- **Row 1 title bar:** `primary-dark` fill, `page-bg` Arial 14pt bold, merged across columns.
- **Row 2 meta:** Arial 9pt italic `secondary-text`. XPath placeholders (`<?EXPORT_DATE?>`) in `primary-accent` colour.
- **Row 3 column headers:** `primary-accent` fill, `page-bg` Arial 11pt bold, centered, wrap-text, 28pt height.
- **Row 4 tag-binding:** `neutral-fill` fill (or `warning-light` for Phase 2 placeholders), Arial 9pt italic `secondary-text`.
- **Data rows / xpath tokens:** Arial 10pt — body text in `body-text`, `<?xpath?>` cells in `primary-accent`. `neutral-border` cell borders.

### NTT DATA example palette (worked example — replace per customer)

NTT **DATA** branding (blue-based). Verified against the NTT DATA Visual Identity Guidelines palette (Background colors / Foreground accent Colors / More accents / Text colors).

| Token | Role | Hex | Use |
|---|---|---|---|
| Smart Navy | `primary-dark` | `#070F38` | Title bars |
| Future Blue | `primary-accent` | `#0072BD` | Column headers, xpath tokens |
| Future Blue 50 | `primary-accent-light` | `#19A3FC` | Hyperlinks |
| Turquoise | `secondary-accent` | `#00DFED` | Secondary callouts |
| Yellow | `warning` | `#FFC400` | Attention |
| Yellow 50 | `warning-light` | `#FFEDB2` | Phase 2 highlights |
| Orange 100 | `strong-attention` | `#E42600` | Critical alerts only |
| Grey 100 | `neutral-border` | `#949494` | Borders |
| Grey 50 | `neutral-fill` | `#E6E8E8` | Tag-binding rows |
| Text Grey | `secondary-text` | `#2E404D` | Footnotes, meta |
| Black | `body-text` | `#000000` | Body text |
| White | `page-bg` | `#FFFFFF` | Background |

> **DO NOT use** for NTT DATA projects: Veklo orange `#F47C26`, Veklo navy `#0A0F2C`, NTT Corporation red `#E60028` (parent-company red — wrong brand entity for NTT DATA).

Typography: **Arial** for body and headers (Georgia is approved for headlines but Excel renders Arial cleanly). Any customer wordmark must be the official logo image — never recreate it in text.
