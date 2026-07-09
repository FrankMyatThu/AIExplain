#### How I Built an Agentic Document Analyzer App
--------------------------------------------------

Imagine you need to analyze documents such as:

* Purchase Orders
* Supplier Invoices
* Shipping Documents
* Bank Statements
* Medical Reports

If those documents come from different suppliers, banks, or organizations across different countries, they rarely share the same format.

In reality:

* Different labels may represent the same business concept, such as **"Invoice No."**, **"Ref #"**, or **"Reference No."**
* Information may appear in one table, multiple tables, or even outside tables.
* Some values are written directly, while others use scaling notes such as **25 × 10⁻⁴** instead of **0.0025**.
* Important information may be located in headers, footers, barcodes, notes, or across multiple pages.

Although the business meaning is the same, the layout, terminology, and data presentation can be completely different.

However, the business systems cannot work directly with these inconsistent documents. They require the information to be converted into a single standardized format before it can be stored, searched, validated, or integrated with other systems.

### The Problem vs. The Goal/Solution

To make this concrete, imagine the **same business document** (a supplier invoice) arriving from two different suppliers. Real documents rarely put everything in one neat table — the same facts are scattered across the header, footer, legends, notes, and tables that are split across the page.

**Supplier A** puts the company in a title block, some facts in a footer, uses a legend to define a scaling factor, and splits the line items across two side-by-side tables:

```text
========================= PAGE HEADER =========================
ACME TRADING LTD
123 Harbour Road, Singapore

Invoice No. : INV-2026-001        Date : 08/07/2026   Ccy : USD

--- Table 1 (left) ---        --- Table 2 (right, same items) ---
Item Code | Qty                Item Code | Thickness¹
A100      | 25                 A100      | 25

Legend:  ¹ Thickness expressed in 10⁻⁴ m
========================= PAGE FOOTER =========================
Codes:  A01 = Net Amount   A02 = Tax
A01: 1,250.00      A02: 87,50
```

**Supplier B** carries the same concepts but with different labels, a compact one-line layout, direct values, and everything inline:

```text
GLOBAL METALS CO — Commercial Invoice

Ref # : 2026/JUL/077  |  Issued : 2026-07-08  |  Currency : US Dollar
Part No.: A100 ; Pieces: 25 ; Thick: 0.0025 m
Net: USD 1250 ; Tax: USD 87.50
```

Both documents mean exactly the same thing, but almost nothing lines up:

* The labels differ (`Invoice No.` vs `Ref #`, `Qty` vs `Pieces`, `A01` vs `Net`).
* Key facts hide in a **title block** (`Acme Trading Ltd`), a **footer** (net amount and tax), and a **legend** rather than in clean key-value pairs.
* The line items are **split across two tables** in Supplier A but written **inline** in Supplier B.
* The thickness uses a **scaling legend** (`25` with `¹ = 10⁻⁴ m` → `0.0025`) in Supplier A but a **direct value** (`0.0025`) in Supplier B.
* Even numbers differ: Supplier A writes tax as `87,50` (comma decimal), Supplier B as `87.50`.

**The goal** is to convert every one of these variations into one predictable, standardized record that business systems can trust. For example, a single JSON output:

```json
{
  "documentType": "SupplierInvoice",
  "referenceNumber": "INV-2026-001",
  "issueDate": "2026-07-08",
  "supplierName": "Acme Trading Ltd",
  "currency": "USD",
  "netAmount": 1250.00,
  "taxAmount": 87.50,
  "lineItems": [
    { "itemCode": "A100", "quantity": 25, "thicknessMeters": 0.0025 }
  ]
}
```

Which maps cleanly into standardized database tables:

```sql
CREATE TABLE invoice (
    id             BIGINT PRIMARY KEY,
    document_type  VARCHAR(50)   NOT NULL,
    reference_no   VARCHAR(100)  NOT NULL,
    issue_date     DATE          NOT NULL,
    supplier_name  VARCHAR(200)  NOT NULL,
    currency       CHAR(3)       NOT NULL,
    net_amount     DECIMAL(18,2),
    tax_amount     DECIMAL(18,2)
);

CREATE TABLE invoice_line_item (
    id                BIGINT PRIMARY KEY,
    invoice_id        BIGINT       NOT NULL REFERENCES invoice(id),
    item_code         VARCHAR(50)  NOT NULL,
    quantity          INT          NOT NULL,
    thickness_meters  DECIMAL(18,6)
);
```

So the core challenge is simple to state but hard to solve: **take many inconsistent input formats — with facts spread across headers, footers, legends, notes, and split tables — and reliably produce one clean, standardized output** that is always structured the same way, no matter which supplier the document came from.

Manually processing these documents process flow is very slow and error-prone. General-purpose AI models such as ChatGPT, Gemini, and Claude are powerful, but they are trained on broad knowledge rather than the specific document structures and terminology used in your business.

This is where an **Agentic Document Analyzer** becomes valuable. By combining specialized AI components, it can consistently extract, normalize, map, and verify information across highly inconsistent document formats.

In this article, I'll share how I built this system and the ideas behind its architecture.


## 1. Data Extraction from PDF Files

Most documents in my project come from different suppliers as PDF files. Some contain selectable text, while others are scanned images. Since AI accuracy depends heavily on the quality of the extracted data, the first stage focuses on obtaining the most reliable document content.

**One important lesson I learned is that no single extraction method is perfect.** Each technique has its own strengths and weaknesses, so my system combines multiple extraction methods and later verifies the results against one another.

### 1.1 AWS Textract

I use AWS Textract because the system needs more than plain OCR text. In addition to extracting text, Textract recognizes document structure, including tables, key-value pairs, page layout, and the relationships between labels and values. This structured information provides a strong foundation for the later AI stages.

### 1.2 Enhanced OCR Images

Small fonts, subscripts, superscripts, and exponent values can sometimes be difficult for OCR to recognize accurately. To improve extraction quality, I preprocess the document images using Python to produce clearer, sharper, and higher-contrast images before sending them to Textract.

Although image enhancement often improves OCR accuracy, it can occasionally introduce artifacts such as distortion or extra spacing. Instead of replacing the original OCR result, I keep both versions for later verification.

### 1.3 Native PDF Text

When a PDF contains selectable text, I also extract the native PDF text. Because it comes directly from the PDF instead of OCR, it often preserves characters and multilingual text more accurately. It serves as another source for verifying the OCR output.

By combining these extraction methods instead of relying on just one, the system produces more reliable input for the downstream AI pipeline.

## 2. Claude LLM Structure Normalization

After PDF/Data extraction, the data is still not ready for reliable mapping. Different suppliers may place the same information in completely different areas: headers, tables, footer notes, key-value fields, or across multiple pages.

So before semantic mapping, I use an LLM for **Structure Normalization**.

The purpose of this step is not to make the final business decision. It does not map fields to the database yet. Its job is to reorganize extracted PDF, OCR, and native PDF text into one consistent JSON structure for the later pipeline.

Example prompt idea:

```text
You are a deterministic document structure extraction engine.

Rules:
1. Do not invent values, labels, rows, columns, translations, or units.
2. Use only PDF text, OCR text, enhanced OCR text, native PDF text, and detected tables from the input.
3. Keep each original source label exactly as written, and add an "enriched" plain-language meaning for every field and column.
4. Do NOT decide the final database field name, and do NOT return database mappings, confidence scores, or business decisions. That field matching is done later by the ML model.
5. Reconstruct logical structure: merge tables that were split across the page, lift key facts out of title blocks and footers, and apply footer legends that define coded columns.
6. Resolve explicit scaling only when the source states it (a column-header scale or a legend), and record the scale you used so it can be audited. If the scale is unknown, keep the raw value.
7. Use enhanced OCR to confirm unclear small fonts, superscripts, subscripts, and exponent values, and use native PDF text to verify OCR.
8. Return valid JSON only.
```

Target output example:

```json
{
  "pages": [
    {
      "pageNumber": 1,
      "fields": [
        {
          "header": "Invoice No.",
          "enriched": "commercial invoice identifier",
          "value": "INV-2026-001"
        },
        {
          "header": "Date",
          "enriched": "document issue date",
          "value": "08/07/2026"
        },
        {
          "header": "Supplier name",
          "enriched": "supplier or vendor legal name",
          "value": "Acme Trading Ltd"
        },
        {
          "header": "Ccy",
          "enriched": "monetary currency code",
          "value": "USD"
        },
        {
          "header": "A01",
          "enriched": "net amount before tax",
          "value": "1250.00"
        },
        {
          "header": "A02",
          "enriched": "tax amount",
          "value": "87.50"
        }
      ],
      "tables": [
        {
          "title": "LineItems",
          "scale_context": {
            "mode": "LEGEND_REFERENCE_ROW",
            "legend": { "1": "10^-4" }
          },
          "columns": [
            {
              "header": "Item Code",
              "enriched": "product item identifier"
            },
            {
              "header": "Qty",
              "enriched": "ordered quantity"
            },
            {
              "header": "Thickness",
              "enriched": "item thickness measurement in meters",
              "metadata": {
                "value_mode": "SCALED_BY_LEGEND",
                "scale_ref": "1",
                "scale": "10^-4",
                "unit": "m"
              }
            }
          ],
          "rows": [
            {
              "Item Code": "A100",
              "Qty": "25",
              "Thickness": "0.0025"
            }
          ]
        }
      ]
    }
  ]
}
```

In short, this stage does three things and deliberately avoids a fourth:

- **Keeps every original label** exactly as written — `Invoice No.` and even cryptic footer codes like `A01`/`A02` — and attaches a plain-language `enriched` meaning to each (for example `commercial invoice identifier`, `net amount before tax`).
- **Fixes the messy layout**: split tables are merged into one `LineItems` table, the supplier name is lifted out of the title block, footer codes are resolved through the page legend, and comma-decimals like `87,50` become `87.50`.
- **Resolves explicit scaling only**: because Supplier A's legend defines `¹ = 10⁻⁴ m`, the raw thickness `25` becomes `0.0025`, and the applied scale is recorded in `scale_context` / column `metadata` so it can be audited later.
- **Does *not* choose the database field.** It never claims `Invoice No.` *is* the `referenceNumber` column — that matching happens in the next (ML) stage.

By turning every supplier's layout into one predictable JSON first, the later embedding, mapping, and verifier stages can focus on meaning instead of fighting layout differences.

> **You may be wondering: why did I let Claude add the `enriched` field at all?**
>
> Because the next stage matches on *meaning*, not spelling. A raw label like `A01` or `Ref #` carries almost no signal on its own, and it shares no words with a standardized field name like `netAmount` or `referenceNumber`. The `enriched` hint (`net amount before tax`, `commercial invoice identifier`) gives the embedding model real semantic text to compare against the standardized fields — so one model can handle any supplier without a hard-coded synonym list for every possible label. Just as important, I keep the raw label **and** the hint side by side: the hint drives matching, while the original label stays for audit and traceability.

## 3. Matching/Searching Layer

We now have clean JSON, but the field names are still supplier-specific. One document says "Invoice No.", another says "Ref #", and a third says "Document ID". We still don't know which standardized field each belongs to. 

### Keyword Search

The most obvious idea to solve this issue is keyword search: take the standardized field name we want (for example `referenceNumber`) and look for it in the document labels. In practice this fails almost immediately.

Suppose our standardized schema field is:

```text
referenceNumber  ->  commercial invoice / document reference identifier
```

The same business fact arrives from three suppliers with three different labels:

```text
Supplier A:  "Invoice No."   = INV-2026-001
Supplier B:  "Ref #"         = 2026/JUL/077
Supplier C:  "Document ID"   = DOC-8891
```

If we search for the keyword `referenceNumber` — or even loosen it to fragments like `reference` or `number` — none of the real labels match:

| Search term       | `Invoice No.` | `Ref #` | `Document ID` |
| ----------------- | ------------- | ------- | ------------- |
| `referenceNumber` | miss          | miss    | miss          |
| `reference`       | miss          | miss    | miss          |
| `invoice`         | hit           | miss    | miss          |
| `number` / `No.`  | hit           | miss    | miss          |

This exposes two problems at the same time:

1. **Same meaning, no shared words.** `Ref #` and `Document ID` mean exactly the same thing as `referenceNumber`, but they share almost no characters with it. Keyword search cannot bridge that gap unless we hand-maintain a giant synonym list for every supplier — which defeats the goal of handling documents we have never seen before.
2. **Shared words, different meaning.** Labels like `Invoice No.`, `DO No.`, and `Contract No.` all contain `No.`. A keyword rule based on `No.` or `number` will happily score the *wrong* field high, even though the business meaning is different.

So keyword search is brittle in both directions: it **misses correct matches** when the wording differs, and it **over-matches wrong fields** when common tokens like `No.` overlap. It only works when everyone uses the exact same words — which is precisely the assumption that breaks with real supplier documents.

### Semantic Search

Instead of comparing spelling, this layer compares **meaning**.

Recall that **Claude LLM Structure Normalization layer** already gave every source field two useful pieces of text:

* the original label (`header`), for example `"Ref #"`
* a plain-language hint (`enriched`), for example `"commercial invoice identifier"`

The Semantic Matching Layer converts both the source text and each standardized field description into embedding vectors — lists of numbers where similar meanings point in similar directions — and then ranks which standardized field is closest in meaning. That is how `"Invoice No."`, `"Ref #"`, and `"Document ID"` can all map to `referenceNumber` without their words ever matching, while still keeping `Invoice No.` and `DO No.` apart even though both contain `No.`.

The following sections explain how those embeddings are produced and compared.

