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

* Different labels may represent the same business concept, such as **"Invoice No."**, **"Inv #"**, or **"Reference No."**
* Information may appear in one table, multiple tables, or even outside tables.
* Some values are written directly, while others use scaling notes such as **25 × 10⁻⁴** instead of **0.0025**.
* Important information may be located in headers, footers, barcodes, notes, or across multiple pages.

Although the business meaning is the same, the layout, terminology, and data presentation can be completely different.

However, the business systems cannot work directly with these inconsistent documents. They require the information to be converted into a single standardized format before it can be stored, searched, validated, or integrated with other systems.

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
3. Preserve original field headers and table column headers exactly as they appear in the source.
4. Do not return final database mappings, confidence scores, or business decisions.
5. Produce a source-bound structured JSON document for later mapping.
6. Use enhanced OCR to confirm unclear small fonts, superscripts, subscripts, exponent values, or visually ambiguous text.
7. Use native PDF text when available to verify OCR results.
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
        }
      ],
      "tables": [
        {
          "title": "LineItems",
          "columns": [
            {
              "header": "Item Code",
              "enriched": "product or service item identifier"
            },
            {
              "header": "Qty",
              "enriched": "ordered quantity"
            }
          ],
          "rows": [
            {
              "Item Code": "A100",
              "Qty": "25"
            }
          ]
        }
      ]
    }
  ]
}
```

Without this normalization step, the remaining pipeline becomes much harder. The embedding model, mapping rules, and verifier would all need to understand every possible document layout. By converting all extracted content into a predictable intermediate JSON first, the later stages can focus on semantic meaning instead of layout confusion.
