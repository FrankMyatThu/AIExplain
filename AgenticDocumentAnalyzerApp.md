#### How I Built an Agentic Document Analyzer App
--------------------------------------------------

Let's say you want to read/analyze:

* Purchase Orders
* Supplier Invoices
* Shipping Documents
* Bank Statements
* Medical Reports

But if they are coming from different suppliers or different banks in different countries,
then those documents will have 90% inconsistent formats/layouts.

In real documents, the whole page can be different:

* Some suppliers use different labels,
  such as "Invoice No.", "Inv #", or "Reference No."
  or "PO Number", "PO No.", "Order Ref", "Order ID", or "Purchase Order".
* Some suppliers use one clean table.
* Some split the same information into multiple tables.
* Some use short column names with footer legends.
* Some use direct decimal values, while others use integer values with a header legend that defines the exponent.
* Some put important values in the header, footer, barcode, or notes area.
* Some continue the table across multiple pages.
* Some mix structured tables with free-text instructions.

So even when the business meaning is the same, the document layout, terminology, and data presentation can be very different.

That is why a normal prompt is usually not enough for production use.
We need a document AI pipeline that can extract, normalize, map, and verify the information consistently.

But you don't want to manually read them; you want to let an AI system analyze them.
Then you may think you can use the most popular LLMs, such as ChatGPT, Gemini, Claude, or whatever.

But the problems are:

1. They are not trained specifically for your business requirements.
   This means they cannot know 100% of your industry-specific knowledge.
   So if you let them analyze those documents, the results cannot be fully accurate, no matter how good or perfect your prompt is.
   Because from one supplier to another, the document layout, technical terms, labels, and data presentation can all be different.





## Tree View

- [1. OCR And Vector Extraction](#1-ocr-and-vector-extraction)
- [2. Claude LLM Structure Normalization](#2-claude-llm-structure-normalization)
- [3. Mapping](#3-mapping)
  - [3.1 How The Application Knows Which Supplier Label Means What](#31-how-the-application-knows-which-supplier-label-means-what)
  - [3.1.A Why Keyword Search Is Not Enough](#31a-why-keyword-search-is-not-enough)
  - [3.1.B Why We Use Semantic Search](#31b-why-we-use-semantic-search)
    - [3.1.B.1 How Semantic Search Works](#31b1-how-semantic-search-works)
    - [3.1.B.2 Why A Ready-Made Embedding Model Is Not Enough](#31b2-why-a-ready-made-embedding-model-is-not-enough)
- [4. Training Layer](#4-training-layer)
- [5. Inference Layer](#5-inference-layer)
- [6. Fargate / Backend Runtime Layer](#6-fargate--backend-runtime-layer)
- [7. Claude Mapping Verifier](#7-claude-mapping-verifier)
- [8. Alignment Rules And Full Recap](#8-alignment-rules-and-full-recap)
- [9. Golden Knowledge Preservation Map](#9-golden-knowledge-preservation-map)

