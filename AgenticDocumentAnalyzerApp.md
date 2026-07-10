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

### How Embeddings are Produced, Then Compared

Semantic matching has two different parts that are easy to mix up:

1. **SageMaker embedding inference** creates vectors from text.
2. **Fargate semantic matching/search** compares those vectors, ranks candidates,
   and applies threshold/gap rules.

The tokenizer, token embedding layer, transformer forward pass, pooling, and L2
normalization belong to embedding generation. The actual semantic search starts
after SageMaker returns vectors to the Fargate worker.

```text
raw text
  -> SageMaker embedding inference
      -> tokenizer
      -> trained transformer model token embedding layer
      -> trained transformer model forward pass
      -> pooling
      -> L2-normalized embedding vector
      -> JSON response: {"embeddings": [[...]], "count": N}
  -> Fargate semantic matching/search
      -> split returned vectors into canonical vectors + header vectors
      -> cosine similarity
      -> rank candidates
      -> threshold + confidence-gap decision
```

Top-to-bottom execution pipeline:

```text
Semantic Matching / Search (top to bottom = order of execution)
|
+-- A. Build embedding input text outside the model
|   +-- source header text: raw header + enriched meaning + sample data
|   +-- canonical field text: display name + description + synonyms
|
+-- B. SageMaker embedding inference
|   |
|   +-- 1. Tokenizer, before the neural model
|   |   +-- raw text -> token IDs + attention mask
|   |       e.g. "cool" -> [4658]
|   |
|   +-- 2. Token Embedding Layer, first layer inside the trained transformer model
|   |   +-- token IDs -> base per-token vectors
|   |       [4658] -> [0.02, -0.45, ...]
|   |
|   +-- 3. Transformer Layers / Forward Pass, inside the trained transformer model
|   |   +-- base per-token vectors -> context-aware per-token vectors
|   |       still one vector per token, now enriched by the whole sequence
|   |
|   +-- 4. Pooling, after the trained transformer model in inference.py
|   |   +-- context-aware per-token vectors -> one text vector
|   |       project supports: CLS pooling and mean pooling
|   |
|   +-- 5. L2 Normalization
|   |   +-- pooled vector -> final unit-length sentence/document embedding
|   |
|   +-- 6. Return JSON embeddings
|       +-- {"embeddings": [[0.012, -0.045, 0.083, "..."]], "count": N}
|
+-- C. Fargate semantic matching/search
    |
    +-- 7. Call SageMaker endpoint for canonical + header texts
    |
    +-- 8. Receive returned embedding vectors
    |   +-- canonical vectors: standardized field descriptions
    |   +-- header vectors: source header + enriched meaning + sample data
    |
    +-- 9. Compare canonical vectors vs header vectors with cosine similarity
    |
    +-- 10. Rank candidate headers
    |
    +-- 11. Apply similarity threshold + confidence gap
```

The diagram is the single source of truth for execution order. The important
boundary is this: SageMaker creates embeddings only; Fargate performs the
semantic search/ranking.

### Step 1. Tokenizer: Raw Text To Token IDs

The tokenizer is outside the neural network model. It is a text conversion
tool. Its job is to turn human-readable raw text into the numeric token IDs
that the model can accept.

Example:

```text
"cool" -> [4658]
```

The number `4658` is not the meaning of the word. It is an ID in the tokenizer
vocabulary. The ID tells the model which learned token row to look up in the
token embedding layer.

A token is a piece of text after tokenization. It may be:

- a full word
- part of a word
- punctuation
- a special model token

The tokenizer can add special tokens such as `[CLS]` and `[SEP]`, depending on
the model family.

Real project code:

`Backend/ML/millcert-header-model/inference/inference.py`

```python
enc = tokenizer(
    texts,
    padding=True,
    truncation=True,
    max_length=embedding_max_len,
    return_tensors="pt"
)
```

For uncommon words, the tokenizer can split one word into subword pieces:

```text
"embeddings" -> [embed, ##ding, s]
"embeddings are cool" -> [embed, ##ding, s, are, cool]
"certification" -> cert + ##ification
"ASTMXYZ9000" -> smaller known chunks, digits, or subword pieces
```

This is why the model can still process many words that were not trained as one
complete word. It can use known pieces. If the tokenizer cannot represent a
word well, some models may use an unknown token, which loses detail.

Fine-tuning does not teach one new weight per possible supplier word. It
teaches the model how to map header-like text into useful vector space using
the tokenizer pieces and learned transformer weights.

Padding adds placeholder tokens so every text in the same batch has the same
length. Neural network batches need rectangular tensor shapes.

An attention mask tells the model which token positions are real text and which
positions are padding:

```text
real token -> 1
padding    -> 0
```

Padding exists only to make batch shapes line up. It should not contribute
meaning to model attention, CLS pooling, or mean pooling.

Truncation cuts text longer than the configured maximum token length. The
project uses `embedding_max_len` to keep training and inference consistent. If
important content appears after the truncation limit, the model will not see
it. This is why embedding text should put the most important source evidence
early and avoid unnecessary noise.

### Step 2. Token Embedding Layer: Token IDs To Base Token Vectors

The token embedding layer is inside the model. It is the first model layer that
converts token IDs into vectors.

Conceptually, it behaves like a learned lookup table:

```text
token ID 4658 -> [0.02, -0.45, 0.18, ...]
```

At this point, the vector is not yet context-aware. It is the learned base
vector for that token ID. The vector knows something learned during pretraining
and fine-tuning, but it has not yet looked at the other tokens in the current
input sentence or header.

Example:

```text
"Heat No." -> token IDs -> token embedding vectors
```

The model still has one vector per token. It does not yet have one vector for
the whole header.

### Step 3. Forward Pass: Base Token Vectors To Context-Aware Token Vectors

A transformer model is a neural network architecture that reads a sequence of
tokens and builds context-aware vectors for them. In this project, the base
model is:

```text
sentence-transformers/all-MiniLM-L12-v2
```

The base model is the pretrained model. It already has general language
knowledge. Fine-tuning adjusts that model so it becomes better at this specific
MillCert header-matching domain.

Human analogy:

- The pretrained model is the starting brain.
- Fine-tuning adjusts that brain for the MillCert domain.
- The embedding output is the model's thought vector for the input text.

A forward pass is the step where inputs move through the model and the model
produces outputs:

```text
token IDs + attention mask -> transformer -> hidden states
```

During training, the forward pass records information needed for gradients.
During inference, the forward pass only creates embeddings.

Human analogy:

```text
forward pass = think
```

The forward pass sends token vectors through stacked transformer layers. These
layers contain attention and feed-forward network blocks. Attention lets each
token look at other tokens in the same input. That is what makes the output
context-aware.

Example:

```text
"Heat No."
```

The vector for `No.` can be influenced by `Heat`, so it can mean heat number
instead of invoice number or certificate number.

Another example:

```text
"Invoice No."
```

The same `No.` token is now surrounded by `Invoice`, so its contextual vector
becomes different.

For each training batch, the model thinks about anchor, positive, and negative
texts and produces embedding vectors for all three.

A hidden state is the model's internal vector representation for a token. After
the forward pass, one header text has many token vectors:

```text
token 1 -> vector
token 2 -> vector
token 3 -> vector
...
```

The important point:

```text
input to transformer = one vector per token
output from transformer = still one vector per token
```

The difference is that output vectors are enriched by context from the full
sequence. The transformer does not automatically collapse all tokens into one
sentence vector. That happens in pooling.

The tensor shape is conceptually:

```text
[batch_size, sequence_length, hidden_size]
```

For example, if 5 texts are padded to 256 tokens and the hidden size is 384:

```text
[5, 256, 384]
```

Real project code:

`Backend/ML/millcert-header-model/inference/inference.py`

```python
outputs = model(**enc)
```

### Step 4. Pooling: Context-Aware Token Vectors To One Text Vector

The project needs one vector per header or canonical field so vectors can be
compared. This conversion from many token vectors to one text vector is called
pooling.

Pooling is post-processing applied to the transformer output. It is not a new
transformer layer in the stack.

Before pooling:

```text
token 1 -> vector
token 2 -> vector
token 3 -> vector
```

After pooling:

```text
whole text -> one vector
```

`[CLS]` is a special token placed at the beginning of the input for BERT-style
models. It is commonly used as a summary position. The model can learn to put
useful whole-text information into the `[CLS]` vector.

In tensor indexing, CLS pooling uses:

```python
outputs.last_hidden_state[:, 0, :]
```

That means:

- `:`: all texts in the batch
- `0`: token position 0, the `[CLS]` token
- `:`: all hidden dimensions

CLS pooling means:

```text
Use the vector for the first `[CLS]` token as the embedding for the whole text.
```

Why use CLS pooling:

- It is simple.
- It can preserve a strong signal from the beginning of a header.
- It often gives a wider score range in this project.

When CLS pooling can hurt:

- Very short text can sometimes score weaker.
- If the first part of the text is noisy, the summary can be less useful.

Mean pooling means:

```text
Average all real token vectors into one vector.
```

Padding tokens are excluded using the attention mask. In this project, mean
pooling is attention-masked. It averages only real tokens, not padding tokens.

Why use mean pooling:

- It can help short phrases.
- It uses information across the whole text.

When mean pooling can hurt:

- Long noisy text can dilute the important words.
- Score range can become compressed.

Last-token pooling uses the final token vector. It is common in some model
families, but this project does not currently implement it.

The project supports both `cls` and `mean`. The chosen mode is saved with the
model in:

```text
output_model/hf_triplet_model/pooling.json
```

Training, local evaluation, and SageMaker inference must use the same pooling
mode. Previous SageMaker inference comments described CLS-only pooling. That is
still valid when pooling mode is `cls`, but the current implementation supports
both `cls` and `mean` and loads the selected mode from the model artifact.

Real project code:

`Backend/ML/millcert-header-model/inference/inference.py`

```python
def pool_embeddings(last_hidden_state, attention_mask, mode):
    if mode == "mean":
        mask = attention_mask.unsqueeze(-1).to(last_hidden_state.dtype)
        summed = (last_hidden_state * mask).sum(dim=1)
        counts = mask.sum(dim=1).clamp(min=1e-9)
        return summed / counts
    return last_hidden_state[:, 0, :]

pooled = pool_embeddings(outputs.last_hidden_state, enc["attention_mask"], pooling)
```

### Step 5. Sentence / Document Embedding: The Result, Not A Layer

The sentence/document embedding is the result after pooling. It is data, not a
layer. Said another way: it is data, not a layer.

In this project, that result is a 384-number vector for MiniLM before or after
normalization, depending on the exact step being discussed.

Example idea:

```text
"Heat No." -> [0.12, -0.08, 0.33, ...]
```

The final sentence/document embedding is not produced by choosing one word. It
is created after the model has produced one contextual vector per token and
pooling converts those token vectors into one text vector.

### L2 Normalization

After pooling, the vector can have any length. The project normalizes it.

The L2 norm is the vector length:

```text
L2 norm = √(x₁² + x₂² + x₃² + ... + xₙ²)
```

L2 normalization divides a vector by its own length:

```text
normalized_vector = vector / L2_norm(vector)
```

After this, the vector length is `1`.

Normalization makes comparison focus on direction, not magnitude. For header
matching, direction is usually what matters:

- Similar meaning should point in a similar direction.
- Different meaning should point in a different direction.

Without normalization:

- large-magnitude vectors can dominate scores
- texts may look close because vector lengths are large, not because meanings
  are similar
- training, evaluation, and inference scores become harder to compare
- cosine-based runtime behavior may no longer match training behavior

Real project code:

`Backend/ML/millcert-header-model/inference/inference.py`

```python
embeddings = F.normalize(pooled, p=2, dim=1)
```

After this line, the SageMaker inference layer has finished its ML work. It has
converted text into L2-normalized vectors. It does not compare fields, rank
headers, apply thresholds, or decide mappings.

The SageMaker endpoint returns JSON embeddings:

```json
{
  "embeddings": [
    [0.012, -0.045, 0.083, 0.019, "..."]
  ],
  "count": 1
}
```

In the real matching flow, Fargate sends both canonical field texts and source
header texts to the SageMaker endpoint. SageMaker returns one vector per input
text. Fargate then splits the returned vectors back into two groups.

Example canonical vectors returned from SageMaker:

```json
{
  "canonical_vectors": [
    {
      "canonical_key": "referenceNumber",
      "embedding_input": "Reference Number | Commercial invoice identifier | invoice no ref document id",
      "vector": [0.018, -0.031, 0.076, 0.044, "..."]
    },
    {
      "canonical_key": "netAmount",
      "embedding_input": "Net Amount | Net amount before tax | subtotal amount excluding tax",
      "vector": [-0.022, 0.057, 0.011, -0.064, "..."]
    }
  ]
}
```

Example header vectors returned from SageMaker:

```json
{
  "header_vectors": [
    {
      "source_header": "Ref #",
      "enriched": "commercial invoice identifier",
      "sample_data": "2026/JUL/077",
      "embedding_input": "Ref # | commercial invoice identifier | 2026/JUL/077",
      "vector": [0.021, -0.028, 0.081, 0.039, "..."]
    },
    {
      "source_header": "Net",
      "enriched": "net amount before tax",
      "sample_data": "USD 1250",
      "embedding_input": "Net | net amount before tax | USD 1250",
      "vector": [-0.019, 0.061, 0.008, -0.069, "..."]
    }
  ]
}
```

Those vectors are the handoff point:

```text
SageMaker inference layer ends here:
text -> tokenizer -> trained transformer model -> pooling -> L2-normalized vector -> JSON response

Fargate matching layer starts here:
canonical vectors + header vectors -> cosine similarity -> ranking -> threshold/gap decision
```

### Why Fargate Combines Canonical Texts And Header Texts

Fargate does not call SageMaker separately for every canonical field and every
source header. It first combines the embedding input texts into one ordered list:

```text
[
  canonical text 1,
  canonical text 2,
  header text 1,
  header text 2
]
```

SageMaker returns embeddings in the same order:

```text
[
  vector for canonical text 1,
  vector for canonical text 2,
  vector for header text 1,
  vector for header text 2
]
```

Then Fargate splits the returned list back into two groups:

```text
canonical_vectors = first part
header_vectors = second part
```

This design is intentional:

- It reduces repeated network calls between Fargate and SageMaker.
- It keeps embedding generation batched, which is usually faster than many tiny
  requests.
- It makes retry behavior simpler because each chunk is handled by one retry
  path.
- It guarantees canonical and header vectors are generated by the same endpoint
  and model version during that matching pass.
- It keeps the comparison step simple: compare every canonical vector against
  every header vector.

There are two limits to understand:

1. **Token limit per text**: `embedding_max_len` applies to each individual text
   string. Long text is truncated before the model sees it.
2. **Batch/payload limit per request**: Fargate still sends the combined list in
   controlled chunks so one SageMaker request does not become too large.

Real SageMaker inference token limit:

`Backend/ML/millcert-header-model/inference/inference.py`

```python
enc = tokenizer(
    texts,
    padding=True,
    truncation=True,
    max_length=embedding_max_len,
    return_tensors="pt"
)
```

Real Fargate batch control:

`Backend/Fargate/app/pipeline/correction_mapping/embedding_client.py`

```python
for index in range(0, len(cleaned_texts), self._batch_size):
    chunk = cleaned_texts[index : index + self._batch_size]
    response = self._invoke_with_retry(chunk)
```

Real Fargate default batch size:

`Backend/Fargate/app/pipeline/correction_mapping/config.py`

```python
embedding_batch_size: int = 5
```

Real Fargate combine-then-split logic:

`Backend/Fargate/app/pipeline/correction_mapping/proposals.py`

```python
texts = [item["embedding_input"] for item in canonical_candidates]
texts.extend(item["embedding_text"] for item in header_candidates)
embeddings = embedding_client.embed_texts(texts)

canonical_vectors = embeddings[: len(canonical_candidates)]
header_vectors = embeddings[len(canonical_candidates) :]
```

For the live model-and-matching path, keep three steps separate:

1. Training normalizes anchor, positive, and negative embeddings before cosine
   triplet loss compares them.
2. SageMaker inference normalizes embeddings and ends when it returns vectors.
3. Downstream Fargate semantic matching/search consumes those vectors and
   compares direction/meaning with cosine similarity.

Real Fargate endpoint call:

`Backend/Fargate/app/pipeline/correction_mapping/embedding_client.py`

```python
return self._runtime_client.invoke_endpoint(
    EndpointName=self._endpoint_name,
    ContentType="application/json",
    Body=json.dumps({"inputs": chunk}).encode("utf-8"),
)
```

Real Fargate cosine comparison:

`Backend/Fargate/app/pipeline/correction_mapping/proposals.py`

```python
def cosine_similarity(left: list[float] | None, right: list[float] | None) -> float:
    if not left or not right or len(left) != len(right):
        return -1.0

    dot = 0.0
    left_norm = 0.0
    right_norm = 0.0
    for left_value, right_value in zip(left, right):
        dot += left_value * right_value
        left_norm += left_value * left_value
        right_norm += right_value * right_value

    if left_norm == 0 or right_norm == 0:
        return -1.0
    return dot / (math.sqrt(left_norm) * math.sqrt(right_norm))
```

Real Fargate embedding split:

`Backend/Fargate/app/pipeline/correction_mapping/proposals.py`

```python
texts = [item["embedding_input"] for item in canonical_candidates]
texts.extend(item["embedding_text"] for item in header_candidates)
embeddings = embedding_client.embed_texts(texts)

canonical_vectors = embeddings[: len(canonical_candidates)]
header_vectors = embeddings[len(canonical_candidates) :]
```

Real Fargate threshold/gap decision:

`Backend/Fargate/app/pipeline/correction_mapping/proposals.py`

```python
if top1_score is None or top1_score < config.similarity_threshold:
    return "NO_MATCH", "Below similarity threshold"

if top2_score is not None and (top1_score - top2_score) < config.confidence_gap:
    return "AMBIGUOUS", "Top1 too close to Top2"

return "STRONG", "Passed threshold and confidence gap"
```

```text
Training path
triplets
  -> tokenizer
  -> transformer model
  -> pooling
  -> L2 normalize pooled embeddings
  -> cosine triplet loss compares anchor / positive / negative
  -> optimizer updates model weights

SageMaker inference path
canonical text + source header text
  -> SageMaker tokenizer
  -> trained transformer model
  -> pooling
  -> L2 normalize returned embeddings
  -> embedding vectors returned

Downstream semantic matching/search path
L2-normalized canonical vectors + L2-normalized header vectors
  -> Fargate cosine similarity compares direction/meaning
  -> rank candidate headers
  -> threshold and confidence-gap rules decide outcome
```

Cosine similarity is:

```text
cosine_similarity = (Σᵢ aᵢbᵢ) / (√(Σᵢ aᵢ²) × √(Σᵢ bᵢ²))
```

If both vectors are normalized, then:

```text
norm(a) = 1
norm(b) = 1
```

So:

```text
cosine_similarity = dot(a, b)
```

This makes scoring simpler and stable.

