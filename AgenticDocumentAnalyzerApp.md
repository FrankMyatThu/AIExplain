#### How I Built an Agentic Document Analyzer App
--------------------------------------------------

Let's say you want to read/analyze 
    - Purchase Orders or
    - Supplier Invoices or
    - Shipping Documents or
    - Bank Statements or 
    - Medical Reports

But if they are coming from different suppliers or different banks from different countries,
then those documents are 90% inconsistent format/layout. 
But you don't want to manually read them, you want to let AI system to analyze them.
Then you may think you will use most popular LLM like ChatGPT, Gemini or Claude or whatever.

But the problems are 

1. They are not trained AI for your specific business requirement.
    Meaning they cannot know 100% about your industrial knowledge.
    So if you let them analyze those documents, the result cannot be accurate no matter how good/perfect your prompt it is.
    Because from 1 supplier to another supplier may have different layout/ different technical terms, different labels , different data presentation layout.
    
2. 




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

