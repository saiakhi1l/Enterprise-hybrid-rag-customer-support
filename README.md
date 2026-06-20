# Hybrid Retrieval-Augmented Generation (RAG) System for Enterprise Support Analytics

## Overview

This project implements a production-oriented Hybrid Retrieval-Augmented Generation (RAG) system designed to answer complex support-related queries from large-scale enterprise ticket repositories.

The system combines lexical retrieval, semantic retrieval, metadata-aware filtering, rank fusion, reranking, and large language model generation to provide accurate and context-grounded responses.

Unlike traditional keyword search systems, this architecture leverages both BM25 and dense vector search to improve recall while maintaining precision through CrossEncoder reranking.

The solution was developed to address common retrieval challenges such as irrelevant semantic matches, keyword mismatch, duplicate retrievals, metadata filtering failures, and context quality degradation.



## Business Problem

Enterprise support teams manage thousands of incident reports, problem tickets, technical support requests, and maintenance records.

Traditional search systems often suffer from:

* Poor keyword matching
* Inability to understand semantic meaning
* Difficulty filtering by ticket attributes
* High retrieval noise
* Limited root-cause discovery

As ticket volumes increase, locating relevant historical incidents becomes time-consuming and operationally expensive.

This project addresses these challenges by enabling intelligent retrieval and summarization of support tickets using a hybrid retrieval architecture.



## Solution Architecture

### Data Ingestion

Support ticket datasets are extracted from compressed archives and loaded into memory using structured ingestion pipelines.

Each ticket contains:

* Subject
* Body
* Answer
* Priority
* Language
* Ticket Type
* Queue

The ingestion layer converts raw tabular records into searchable document representations.



### Document Construction

Each ticket is transformed into a unified text document.

Example:

Subject: Login Failure

Body: Users unable to authenticate after recent deployment.

Answer: Restart authentication services and validate configuration changes.

This transformation enables both lexical and semantic retrieval systems to operate on a common document representation.



### Metadata Extraction

Structured metadata is extracted and stored separately.

Example:

```python
{
    "priority": "high",
    "language": "en",
    "type": "Incident",
    "queue": "Technical Support"
}
```

Metadata enables retrieval filtering before semantic search.

This significantly improves retrieval precision and reduces search space.



### Embedding Generation

Dense vector representations are generated using:

BAAI/bge-small-en-v1.5

Embeddings are normalized to support cosine similarity search.

These embeddings capture semantic meaning beyond exact keyword matching.



### Vector Database Indexing

Embeddings and metadata are indexed in ChromaDB.

Each stored record contains:

* Document Text
* Embedding Vector
* Metadata
* Unique Identifier

This enables high-speed semantic retrieval with metadata-aware filtering.


### BM25 Lexical Retrieval

A BM25 index is built over all support documents.

BM25 provides strong keyword matching capabilities and handles exact terminology effectively.

Examples:

* login failure
* authentication issue
* server overload

BM25 often outperforms vector search when exact keywords are critical.



### Metadata-Aware Retrieval

User queries are parsed for structured constraints.

Examples:

* High Priority
* English
* Technical Support
* IT Support

These constraints are translated into ChromaDB filters.

Example:

```python
{
    "$and": [
        {"priority":"high"},
        {"language":"en"},
        {"queue":"Technical Support"}
    ]
}
```

This ensures retrieval is restricted to relevant ticket subsets.



### Hybrid Retrieval

The system performs:

1. BM25 Retrieval
2. Vector Retrieval

Both retrieval methods generate independent candidate sets.

This approach improves recall compared to relying on a single retrieval mechanism.



### Reciprocal Rank Fusion (RRF)

Initially, BM25 and vector results were concatenated directly.

This introduced ranking inconsistencies and duplicate candidates.

The system was upgraded to Reciprocal Rank Fusion (RRF).

RRF assigns scores based on ranking positions across both retrieval systems.

Documents that perform well in both BM25 and vector search naturally rise to the top.

Benefits:

* Improved retrieval stability
* Better recall
* Reduced ranking bias
* Higher-quality candidate pool



### CrossEncoder Reranking

A CrossEncoder reranker is applied after retrieval.

The reranker evaluates:

(Query, Document)

pairs and assigns relevance scores.

This stage significantly improves precision by reordering candidate documents according to contextual relevance.

Without reranking, retrieval quality degraded noticeably for complex support queries.



### Context Construction

Top-ranked documents are enriched with metadata before being sent to the language model.

Example:

```text
[Priority: high | Queue: Technical Support | Language: en]

Subject: Login Failure
...
```

This improves grounding and reduces missing-field responses.


### LLM-Based Answer Generation

The final context is passed to:

Llama 3.3 70B Versatile

through the Groq inference platform.

The model is instructed to:

* Use only retrieved context
* Avoid hallucinations
* Extract root causes
* Extract resolution steps
* Summarize support incidents


### Caching Layer

A query-level cache is implemented to reduce repeated inference calls.

Benefits:

* Lower latency
* Reduced inference cost
* Faster user experience


### Guardrails

Input Guardrails:

* Prompt injection prevention
* Secret detection
* Restricted instruction handling

Output Guardrails:

* API key detection
* Password detection
* Secret leakage prevention

These controls improve safety and reliability.



## Technical Challenges Encountered

### Challenge 1: Missing Login Tickets

Problem:

Queries related to login failures were returning unrelated synchronization and connectivity incidents.

Root Cause:

The reranker was receiving the entire user query including instruction words such as:

* show
* summarize
* technical support tickets

These terms diluted retrieval relevance.

Solution:

Introduced query cleaning and topic extraction before retrieval and reranking.

Result:

Login-related incidents consistently ranked higher.


# Challenge 2: Metadata Filters Returning Empty Results

Problem:

Metadata filters occasionally returned zero documents.

Root Cause:

Metadata values stored in ChromaDB did not exactly match query-generated filters.

Solution:

Standardized metadata values during ingestion.

Result:

Reliable metadata filtering.

---

### Challenge 3: Duplicate Documents

Problem:

The same ticket appeared multiple times in retrieval results.

Root Cause:

BM25 and vector retrieval returned overlapping documents.

Solution:

Implemented deduplication after fusion.

Result:

Cleaner candidate sets.

---

### Challenge 4: Poor Hybrid Ranking

Problem:

Simple concatenation of BM25 and vector results favored one retrieval source.

Root Cause:

Ranking information was lost.

Solution:

Implemented Reciprocal Rank Fusion.

Result:

Significantly improved retrieval quality.



### Challenge 5: LLM Reporting Missing Information

Problem:

The model frequently returned:

"Not specified"

even when information existed.

Root Cause:

Important ticket attributes were hidden within free-text content.

Solution:

Metadata-enriched context construction.

Result:

Improved extraction accuracy.

---

## Technology Stack

Programming Language

* Python

Retrieval

* BM25
* ChromaDB

Embeddings

* BAAI/bge-small-en-v1.5

Reranking

* CrossEncoder

LLM

* Llama 3.3 70B
* Groq API

Data Processing

* Pandas
* NumPy

Infrastructure

* Google Colab
* ChromaDB


## Key Learnings

* Hybrid retrieval consistently outperformed standalone retrieval approaches.
* Metadata filtering significantly improved retrieval precision.
* Retrieval quality matters more than model size in many RAG systems.
* Reranking is essential when handling large candidate pools.
* Rank fusion provides more robust retrieval than naive result concatenation.
* Proper debugging of retrieval pipelines requires inspecting each stage independently.
* Context quality directly impacts final answer quality.



## Future Enhancements

* Multi-format document ingestion (PDF, DOCX, PPTX, HTML, TXT, CSV, XLSX, JSON)
* Automatic metadata generation using LLMs
* Domain classification
* Query rewriting
* Semantic caching
* Multi-vector retrieval
* Agentic retrieval workflows
* Evaluation framework using Precision@K, Recall@K, MRR, and NDCG
* Distributed vector database deployment
* Production API deployment using FastAPI and Docker

## Outcome

The final system successfully combines lexical search, semantic search, metadata-aware retrieval, rank fusion, reranking, and LLM generation into a unified retrieval architecture capable of answering complex enterprise support queries with significantly improved relevance and accuracy compared to traditional search approaches.
