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

Manually processing these documents is slow and error-prone. General-purpose AI models such as ChatGPT, Gemini, and Claude are powerful, but they are trained on broad knowledge rather than the specific document structures and terminology used in your business.

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


