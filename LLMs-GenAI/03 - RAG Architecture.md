---
tags: [ai, rag, phase-5]
phase: 5
status: not-started
priority: high
---

# 🔍 RAG Architecture

> **Phase:** 5 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[01 - LLM Fundamentals]], [[02 - Prompt Engineering]], [[04 - LangChain]], [[01 - System Design AI Problems]]

---

## Checklist

### RAG Pipeline Overview
- [ ] Why RAG: reduce hallucinations, use private data, no fine-tuning needed
- [ ] Architecture: Indexing → Retrieval → Augmentation → Generation
- [ ] Offline pipeline: document loading → chunking → embedding → vector store
- [ ] Online pipeline: query → embed → retrieve → augment prompt → LLM → response
- [ ] RAG vs fine-tuning vs long context — decision framework

### Document Loading
- [ ] PDF: `PyPDF`, `pdfplumber`, `Unstructured`, `LlamaParse`
- [ ] Web: `BeautifulSoup`, `Playwright`, `Firecrawl`
- [ ] Office: `python-docx`, `openpyxl`, `Unstructured`
- [ ] Structured data: SQL databases, APIs, CSV
- [ ] Multi-modal: tables (extract as markdown), images (OCR, vision models)
- [ ] Metadata extraction: source, page number, section headers

### Chunking Strategies
- [ ] Fixed-size: character/token count, simple but can break context
- [ ] Recursive character splitting: split by paragraphs → sentences → words
- [ ] Semantic chunking: use embeddings to find natural topic boundaries
- [ ] Document-structure-aware: respect headers, sections, code blocks
- [ ] Chunk size tradeoffs: smaller = more precise retrieval, larger = more context
- [ ] Chunk overlap: 10-20% overlap prevents losing context at boundaries
- [ ] Parent-child chunking: retrieve small chunk, return parent for context
- [ ] Typical sizes: 256-1024 tokens per chunk

### Embedding Models
- [ ] OpenAI: `text-embedding-3-small` (1536d), `text-embedding-3-large` (3072d)
- [ ] Cohere: `embed-v3`, multilingual support
- [ ] Open-source: `bge-large`, `e5-mistral`, `nomic-embed` — run locally
- [ ] Matryoshka embeddings: variable-dimension, truncate for speed/cost tradeoff
- [ ] Embedding quality: MTEB benchmark for comparison
- [ ] Embedding dimensions: higher = more info, but more storage/compute
- [ ] Fine-tuning embeddings: for domain-specific retrieval improvement

### Vector Databases
- [ ] Pinecone: fully managed, serverless, metadata filtering
- [ ] Weaviate: open-source, hybrid search, multi-tenancy
- [ ] Qdrant: open-source, filtering, payload storage, Rust-based
- [ ] Milvus/Zilliz: open-source, GPU-accelerated, enterprise scale
- [ ] ChromaDB: lightweight, local development, Python-native
- [ ] pgvector: PostgreSQL extension, HNSW/IVFFlat indexes
- [ ] Choosing: scale, hosting preference, filtering needs, existing infra

### Similarity Search
- [ ] Cosine similarity: angle between vectors, most common
- [ ] Dot product: faster, assumes normalized vectors
- [ ] Euclidean (L2): distance-based, less common for text
- [ ] Approximate Nearest Neighbor (ANN): HNSW (graph-based), IVF-PQ (quantized)
- [ ] HNSW: build graph of neighbors, log(N) search time, memory-intensive
- [ ] IVF-PQ: cluster centroids + product quantization, less memory
- [ ] Exact vs approximate: accuracy vs latency tradeoff

### Advanced RAG Techniques
- [ ] Hybrid search: dense (embedding) + sparse (BM25/TF-IDF), combine scores
- [ ] Re-ranking: cross-encoder models (Cohere Reranker, BGE-reranker)
  - [ ] Bi-encoder (embedding) for recall → cross-encoder for precision
- [ ] Multi-query: generate multiple query variations → retrieve → deduplicate
- [ ] HyDE: Hypothetical Document Embedding — generate answer first, embed it, search
- [ ] Self-query: LLM extracts metadata filters from natural language query
- [ ] Contextual compression: summarize retrieved chunks before passing to LLM
- [ ] Sentence-window retrieval: retrieve sentence, expand to surrounding window
- [ ] Recursive retrieval: summarize documents → retrieve summary → retrieve full doc

### RAG Evaluation
- [ ] Context relevance: are retrieved chunks relevant to the query?
- [ ] Answer faithfulness: is the answer grounded in the context (no hallucination)?
- [ ] Answer relevance: does the answer address the original question?
- [ ] RAGAS framework: automated evaluation metrics for RAG
- [ ] LLM-as-judge: score retrieval quality and answer quality
- [ ] Evaluation dataset: curate question-answer pairs with source attribution

### Knowledge Graphs + RAG (GraphRAG)
- [ ] Entity extraction from documents → build knowledge graph
- [ ] Graph traversal for related context retrieval
- [ ] Combining vector search + graph traversal
- [ ] Microsoft GraphRAG: community detection, summarization, global queries
- [ ] Neo4j + LangChain integration
- [ ] When GraphRAG > vector RAG: complex relationships, multi-hop reasoning

### Multi-Modal RAG
- [ ] Images in documents: extract with vision models, embed separately
- [ ] Table extraction: convert to markdown/CSV, embed as text
- [ ] Cross-modal retrieval: text query → retrieve images/tables
- [ ] Colpali/ColQwen: document-level visual embeddings

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] LangChain RAG tutorial: https://python.langchain.com/docs/tutorials/rag/
- [ ] "Building RAG Applications" (O'Reilly)
- [ ] RAGAS docs: https://docs.ragas.io/
- [ ] Jerry Liu talks on Advanced RAG (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design a RAG system for a company's internal documentation
- How do you evaluate RAG quality? What metrics matter?
- Compare naive RAG vs advanced RAG — what improvements can you make?
- When would you use GraphRAG vs vector-only RAG?
