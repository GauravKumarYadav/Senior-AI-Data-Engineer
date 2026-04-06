# Module 3: RAG Architecture

> **Scenario**: ShopAssist Inc. has a working LLM-powered customer support agent, but it hallucinates product details, cites outdated return policies, and can't answer questions about your specific product catalog. RAG (Retrieval-Augmented Generation) fixes this by grounding the model in your actual data.

---

## Screen 1: Why RAG — The Case for Retrieval-Augmented Generation

### The Problem: LLMs Don't Know Your Business

Your GPT-4o-powered support agent is great at language, but it doesn't know:
- Your current return policy (updated last Tuesday)
- That order #7788 shipped via FedEx with tracking #12345
- Your product catalog of 50,000 SKUs with specifications
- Internal escalation procedures and SLAs

Fine-tuning could teach some of this, but it's expensive, slow to update, and still hallucinates. RAG gives the model access to current, authoritative information at query time.

### RAG vs Fine-Tuning vs Long Context

| Approach | Cost | Freshness | Accuracy | Best For |
|---|---|---|---|---|
| RAG | Low ($0.01/query) | Real-time | High (grounded) | Dynamic knowledge, large corpora |
| Fine-Tuning | High ($500-5K) | Stale until retrained | Moderate | Behavior/style changes, domain adaptation |
| Long Context | Medium (token cost) | Per-request | Good (if in context) | Small, focused docs (<100 pages) |
| RAG + Fine-Tuning | Highest | Real-time | Highest | Mission-critical production systems |

### The Full RAG Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          RAG PIPELINE                                    │
│                                                                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────────┐  │
│  │ Document  │──▶│ Chunking │──▶│Embedding │──▶│   Vector Store       │  │
│  │ Loading   │   │          │   │          │   │   (Indexing)         │  │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┬───────────┘  │
│                                                           │              │
│                     INGESTION PIPELINE (offline)          │              │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─  │
│                     QUERY PIPELINE (online)               │              │
│                                                           │              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐              │              │
│  │  User    │──▶│  Query   │──▶│Retrieval │◀─────────────┘              │
│  │  Query   │   │ Embedding│   │ (ANN)    │                             │
│  └──────────┘   └──────────┘   └────┬─────┘                             │
│                                      │                                   │
│                                      ▼                                   │
│                              ┌──────────────┐   ┌──────────────────┐    │
│                              │ Augmentation  │──▶│   Generation     │    │
│                              │ (prompt build)│   │   (LLM call)     │    │
│                              └──────────────┘   └──────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### ShopAssist Knowledge Sources

For our customer support system, we need to ingest:
- **Product catalog** (50K products with specs, pricing, images)
- **Return/refund policies** (updated quarterly, 40 pages)
- **FAQ knowledge base** (500+ articles)
- **Order data** (accessed via API, not embedded)
- **Previous support conversations** (for similar-case retrieval)

### 💡 Interview Insight

> **Q: When would you choose RAG over fine-tuning for a customer support application?**
>
> "RAG is my default for knowledge-grounded applications, and here's why. First, freshness — our return policy changes quarterly, product catalog updates daily. Fine-tuning would require retraining every time, but RAG picks up changes as soon as documents are re-indexed. Second, attribution — RAG can cite which document it used, which builds customer trust and enables auditing. Third, cost — fine-tuning GPT-4o costs thousands per run, while RAG adds about $0.01 per query for embeddings and retrieval. I'd consider fine-tuning only for changing the model's behavior or style — like teaching it our company's communication tone — and combine it with RAG for factual knowledge. The combination of a fine-tuned model with RAG gives you both domain-adapted behavior and grounded, current information."

---

## Screen 2: Document Loading — Getting Data In

Before you can retrieve anything, you need to load, parse, and clean your documents.

### PDF Loading

```python
import pymupdf  # PyMuPDF — fast, handles complex layouts
from pathlib import Path

def load_pdf(file_path: str) -> list[dict]:
    """Load PDF with PyMuPDF — handles text, tables, and metadata."""
    doc = pymupdf.open(file_path)
    pages = []
    for page_num, page in enumerate(doc):
        text = page.get_text("text")
        # Extract tables separately for structured data
        tables = page.find_tables()
        table_data = [table.extract() for table in tables]
        
        pages.append({
            "text": text,
            "tables": table_data,
            "page_number": page_num + 1,
            "source": Path(file_path).name,
            "metadata": {
                "title": doc.metadata.get("title", ""),
                "page": page_num + 1,
                "total_pages": len(doc),
            }
        })
    doc.close()
    return pages


# For complex documents with mixed layouts, use Unstructured
from unstructured.partition.pdf import partition_pdf

def load_complex_pdf(file_path: str) -> list[dict]:
    """Use Unstructured for complex PDFs with images, tables, headers."""
    elements = partition_pdf(
        filename=file_path,
        strategy="hi_res",             # OCR + layout detection
        infer_table_structure=True,     # Detect and parse tables
        extract_images_in_pdf=True,     # Extract embedded images
    )
    
    documents = []
    for element in elements:
        documents.append({
            "text": str(element),
            "type": element.category,    # "Title", "NarrativeText", "Table", etc.
            "metadata": element.metadata.to_dict(),
        })
    return documents
```

### Web Scraping for Knowledge Base

```python
import httpx
from bs4 import BeautifulSoup

async def scrape_knowledge_base(base_url: str, urls: list[str]) -> list[dict]:
    """Scrape ShopAssist knowledge base articles."""
    documents = []
    async with httpx.AsyncClient() as client:
        for url in urls:
            response = await client.get(url)
            soup = BeautifulSoup(response.text, "html.parser")
            
            # Extract main content, skip nav/footer
            article = soup.find("article") or soup.find("main")
            if not article:
                continue
            
            # Clean text — remove scripts, styles, excess whitespace
            for tag in article.find_all(["script", "style", "nav"]):
                tag.decompose()
            
            text = article.get_text(separator="\n", strip=True)
            title = soup.find("h1").get_text(strip=True) if soup.find("h1") else ""
            
            documents.append({
                "text": text,
                "title": title,
                "url": url,
                "source": "knowledge_base",
                "metadata": {
                    "title": title,
                    "url": url,
                    "scraped_at": "2024-01-25",
                }
            })
    return documents
```

### Multimodal Document Processing

For product images with text or specifications, use GPT-4o vision to extract structured content. Encode images as base64, send via the vision API, and store the extracted text as a searchable document alongside the image reference.

### 💡 Interview Insight

> **Q: How do you handle diverse document types in a production RAG pipeline?**
>
> "I build a document loader registry — a factory pattern that routes files to the right parser based on MIME type. PDFs go through PyMuPDF for simple layouts or Unstructured for complex ones with tables and images. HTML goes through BeautifulSoup with content extraction heuristics. For Office docs, I use python-docx and openpyxl. The critical step most teams skip is quality validation after loading: I sample 5% of loaded documents, compare the extracted text to the original visually, and measure extraction accuracy. Tables are the hardest — I parse them into Markdown format so the LLM can reason about rows and columns. For images with text, I use GPT-4o vision to extract structured content. Every loaded document gets metadata: source, page number, last modified date, document type. This metadata is essential for filtering during retrieval."

---

## Screen 3: Chunking Strategies — The Art of Splitting Documents

Chunking is where many RAG systems fail. Too large and retrieval is imprecise. Too small and you lose context.

### Fixed-Size Chunking

```python
def fixed_size_chunk(text: str, chunk_size: int = 512, overlap: int = 50) -> list[str]:
    """Simple fixed-size chunking with overlap."""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap  # Overlap to avoid cutting context
    return chunks
```

### Recursive Character Splitting (LangChain-style)

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""],  # Try larger splits first
    length_function=len,
)

# For our return policy document
return_policy = """
## 30-Day Return Policy

All items purchased from MegaShop can be returned within 30 days of delivery.

### Electronics
Electronics must be in original packaging. Opened items are subject to a 15% 
restocking fee. Defective items are exempt from restocking fees.

### Clothing
Clothing can be returned if unworn with tags attached. Final sale items 
cannot be returned.

### Furniture
Furniture returns must be scheduled with our logistics team. A pickup fee 
of $49.99 applies for items over 50 lbs.
"""

chunks = splitter.split_text(return_policy)
for i, chunk in enumerate(chunks):
    print(f"Chunk {i} ({len(chunk)} chars): {chunk[:80]}...")
```

### Semantic Chunking

Instead of splitting at arbitrary character boundaries, semantic chunking splits where the topic changes.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

semantic_splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=90,  # Split when similarity drops below 90th percentile
)

chunks = semantic_splitter.split_text(return_policy)
```

### Parent-Child Chunking (Best of Both Worlds)

```
┌─────────────────────────────────────────────────────┐
│  PARENT-CHILD CHUNKING STRATEGY                     │
│                                                      │
│  Parent Document (1024 tokens) ← Sent to LLM        │
│  ┌─────────────────────────────────────────────────┐ │
│  │  ## Electronics Return Policy                    │ │
│  │                                                  │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │ │
│  │  │ Child 1  │ │ Child 2  │ │ Child 3  │        │ │
│  │  │ 256 tok  │ │ 256 tok  │ │ 256 tok  │        │ │
│  │  │          │ │          │ │          │        │ │
│  │  │ Used for │ │ Used for │ │ Used for │        │ │
│  │  │ RETRIEVAL│ │ RETRIEVAL│ │ RETRIEVAL│        │ │
│  │  └──────────┘ └──────────┘ └──────────┘        │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  Small chunks → precise retrieval                    │
│  Parent doc   → full context for generation          │
└─────────────────────────────────────────────────────┘
```

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_chroma import Chroma

# Small chunks for retrieval precision
child_splitter = RecursiveCharacterTextSplitter(chunk_size=256, chunk_overlap=30)
# Large chunks for generation context
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1024, chunk_overlap=100)

vectorstore = Chroma(embedding_function=embeddings, collection_name="support_docs")
docstore = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
```

### Chunk Size Tradeoffs

| Chunk Size | Retrieval Precision | Context Quality | Token Cost | Best For |
|---|---|---|---|---|
| 128-256 | ★★★★★ | ★★☆☆☆ | Low | FAQ, short answers |
| 512 | ★★★★☆ | ★★★★☆ | Medium | General purpose (default) |
| 1024 | ★★★☆☆ | ★★★★★ | High | Complex policies, procedures |
| 2048+ | ★★☆☆☆ | ★★★★★ | Very high | Full document retrieval |

### 💡 Interview Insight

> **Q: How do you choose chunking strategy and chunk size for a RAG system?**
>
> "I start with recursive character splitting at 512 tokens with 50-token overlap — it's the reliable baseline. Then I evaluate retrieval quality on a test set. If retrieval precision is low — the right chunk isn't in the top-3 results — I try smaller chunks (256 tokens) or parent-child chunking. If the LLM generates incomplete answers despite finding the right chunk, I increase chunk size or switch to parent-child, where small chunks drive retrieval but the full parent section is sent to the LLM. Semantic chunking sounds appealing but adds latency and cost since it requires embedding every sentence during splitting. For our support system, I'd use parent-child: 256-token children for retrieval, 1024-token parents for context. The return policy has logical sections that map perfectly to parent documents."

---

## Screen 4: Embedding Models — Turning Text into Vectors

Embeddings are the bridge between text and math. Your choice of embedding model directly determines retrieval quality.

### Generating Embeddings

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

def embed_texts(texts: list[str], model: str = "text-embedding-3-small") -> list[list[float]]:
    """Generate embeddings for a batch of texts."""
    response = client.embeddings.create(
        model=model,
        input=texts,
        dimensions=512,  # text-embedding-3 supports dimension reduction (Matryoshka)
    )
    return [item.embedding for item in response.data]

# Embed our support knowledge base chunks
chunks = [
    "Electronics can be returned within 30 days. Opened items have a 15% restocking fee.",
    "Clothing returns require unworn items with tags attached. No final sale returns.",
    "Furniture pickup can be scheduled through our logistics portal. Fee: $49.99 for items over 50 lbs.",
]

embeddings = embed_texts(chunks)
print(f"Embedding dim: {len(embeddings[0])}")  # 512
print(f"Batch size: {len(embeddings)}")         # 3
```

### Matryoshka Embeddings

OpenAI's text-embedding-3 models support **Matryoshka representation learning** — you can truncate embeddings to smaller dimensions with graceful quality degradation.

```python
# Full 3072 dimensions — maximum quality
emb_full = embed_texts(["return policy for electronics"], model="text-embedding-3-large")
# 3072 dims → ~12KB per vector

# Truncated to 256 dimensions — 12× smaller, ~95% quality
emb_small = embed_texts(
    ["return policy for electronics"],
    model="text-embedding-3-large",
)
# Request 256 dims: dimensions=256 → ~1KB per vector

# Trade-off: 50K documents × 3072 dims × 4 bytes = 614 MB
# Trade-off: 50K documents × 256 dims  × 4 bytes = 51 MB   ← 12× savings!
```

### Embedding Model Comparison

| Model | Provider | Dims | MTEB Score | Cost (1M tokens) | Open? |
|---|---|---|---|---|---|
| text-embedding-3-large | OpenAI | 3072 | 64.6 | $0.13 | ❌ |
| text-embedding-3-small | OpenAI | 1536 | 62.3 | $0.02 | ❌ |
| embed-v3 (english) | Cohere | 1024 | 64.5 | $0.10 | ❌ |
| BGE-large-en-v1.5 | BAAI | 1024 | 63.6 | Free | ✅ |
| E5-mistral-7b | Microsoft | 4096 | 66.6 | Free | ✅ |
| GTE-Qwen2-7B | Alibaba | 3584 | 67.2 | Free | ✅ |
| NV-Embed-v2 | NVIDIA | 4096 | 69.3 | Free | ✅ |

**Key insight**: Open-source embedding models have caught up to and surpassed proprietary ones. For cost-sensitive production, self-host BGE or GTE. For simplicity, use OpenAI text-embedding-3-small.

### 💡 Interview Insight

> **Q: How do you select an embedding model for a production RAG system?**
>
> "I evaluate on three axes: quality, cost, and operational complexity. For quality, I create a test set of 100 query-document pairs from our actual support data and measure recall@5 — does the correct document appear in the top 5 results? I test 3-4 models head-to-head. For cost, I calculate total spend at our volume: 50K queries/day × embedding cost. text-embedding-3-small costs about $1/day at that volume — trivial. For operational complexity, API-based models mean zero infrastructure but add latency and a dependency. Self-hosted models like BGE eliminate the dependency but need GPU infrastructure. For ShopAssist, I'd start with text-embedding-3-small for speed-to-market, then evaluate self-hosted GTE-Qwen2 once we hit scale where the API latency or cost becomes a concern."

---

## Screen 5: Vector Databases — Storing and Searching Embeddings

Vector databases are purpose-built for storing high-dimensional vectors and performing fast similarity search.

### ChromaDB — Getting Started Quickly

```python
import chromadb
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction

# Initialize ChromaDB (local, perfect for development)
client = chromadb.PersistentClient(path="./shopassist_vectordb")

embedding_fn = OpenAIEmbeddingFunction(
    model_name="text-embedding-3-small",
    api_key="your-api-key",
)

# Create collection for support documents
collection = client.get_or_create_collection(
    name="support_knowledge",
    embedding_function=embedding_fn,
    metadata={"hnsw:space": "cosine"},  # Cosine similarity
)

# Add documents with metadata
collection.add(
    documents=[
        "Electronics can be returned within 30 days. Opened items: 15% restocking fee.",
        "Clothing returns require unworn items with original tags attached.",
        "Free shipping on orders over $50. Express shipping: $12.99.",
    ],
    metadatas=[
        {"category": "returns", "department": "electronics", "updated": "2024-01-15"},
        {"category": "returns", "department": "clothing", "updated": "2024-01-15"},
        {"category": "shipping", "department": "general", "updated": "2024-02-01"},
    ],
    ids=["ret-elec-001", "ret-cloth-001", "ship-gen-001"],
)

# Query with metadata filtering
results = collection.query(
    query_texts=["Can I return my opened laptop?"],
    n_results=3,
    where={"category": "returns"},  # Filter to returns category only
)

print(results["documents"][0])
# ["Electronics can be returned within 30 days. Opened items: 15% restocking fee."]
```

### Vector Database Comparison

| Database | Type | Filtering | Scale | Best For |
|---|---|---|---|---|
| ChromaDB | Embedded/local | Basic | 100K vectors | Prototyping, small apps |
| pgvector | Postgres extension | Full SQL | 1M vectors | Existing Postgres stack |
| Pinecone | Managed cloud | Rich metadata | 1B+ vectors | Zero-ops production |
| Weaviate | Self-hosted/cloud | GraphQL + hybrid | 100M+ vectors | Hybrid search |
| Qdrant | Self-hosted/cloud | Rich payload | 100M+ vectors | Performance-critical |
| Milvus | Self-hosted/cloud | Complex filters | 1B+ vectors | Massive scale |

### pgvector — Production with PostgreSQL

```python
import asyncpg

async def setup_pgvector():
    """Set up pgvector — use your existing Postgres infrastructure."""
    conn = await asyncpg.connect("postgresql://localhost/shopassist")
    await conn.execute("CREATE EXTENSION IF NOT EXISTS vector")
    await conn.execute("""
        CREATE TABLE IF NOT EXISTS support_docs (
            id SERIAL PRIMARY KEY,
            content TEXT NOT NULL,
            embedding vector(512),
            category VARCHAR(50),
            department VARCHAR(50),
            updated_at TIMESTAMP DEFAULT NOW()
        )
    """)
    # HNSW index for fast approximate search
    await conn.execute("""
        CREATE INDEX IF NOT EXISTS support_docs_embedding_idx 
        ON support_docs USING hnsw (embedding vector_cosine_ops)
        WITH (m = 16, ef_co 200)
    """)
    return conn
```

### 💡 Interview Insight

> **Q: How do you choose between vector database options for production?**
>
> "My decision tree: If you already run Postgres, start with pgvector — zero new infrastructure, SQL filtering, and it handles up to a million vectors well. If you need managed, zero-ops hosting at scale, Pinecone is the safe choice. If you need hybrid search (vector + keyword) natively, Weaviate or Qdrant. For our ShopAssist system with 50K documents and 500K chunks, pgvector is the pragmatic choice: we already have Postgres for order data, and at this scale, pgvector with HNSW indexing gives us sub-50ms queries. I'd only move to a dedicated vector DB if we scale beyond 10M vectors or need specialized features like multi-tenancy or real-time index updates at high throughput."

---

## Screen 6: Similarity Search — How Retrieval Actually Works

Understanding how vectors are compared and indexed is essential for tuning retrieval quality.

### Distance Metrics

```python
import numpy as np

def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    """Most common for normalized embeddings. Range: -1 to 1."""
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def dot_product(a: np.ndarray, b: np.ndarray) -> float:
    """Equivalent to cosine similarity when vectors are normalized."""
    return np.dot(a, b)

def l2_distance(a: np.ndarray, b: np.ndarray) -> float:
    """Euclidean distance. Lower = more similar."""
    return np.linalg.norm(a - b)

# Example: Query vs documents
query_emb = np.array(embed_texts(["How do I return electronics?"])[0])
doc1_emb = np.array(embed_texts(["Electronics return policy: 30 days, 15% restocking fee"])[0])
doc2_emb = np.array(embed_texts(["Our company was founded in 2010 in Seattle"])[0])

print(f"Query ↔ Returns doc:  {cosine_similarity(query_emb, doc1_emb):.4f}")  # ~0.85
print(f"Query ↔ About us doc: {cosine_similarity(query_emb, doc2_emb):.4f}")  # ~0.31
```

### ANN Algorithms: HNSW vs IVF-PQ

Exact nearest-neighbor search is O(n) — too slow at scale. Approximate Nearest Neighbor (ANN) algorithms trade a tiny accuracy loss for massive speed gains.

```
HNSW (Hierarchical Navigable Small World)
┌────────────────────────────────────────────────────┐
│  Layer 2 (few nodes):    A ─────────── B           │
│                          │                          │
│  Layer 1 (more nodes):   A ── C ── D ── B          │
│                          │    │    │    │           │
│  Layer 0 (all nodes):    A  E C  F D G  B H        │
│                                                     │
│  Search: Start at top layer, greedily descend       │
│  Recall@10: ~99%  |  Latency: <5ms for 1M vectors  │
│  Memory: High (stores graph + vectors)              │
└────────────────────────────────────────────────────┘

IVF-PQ (Inverted File + Product Quantization)
┌────────────────────────────────────────────────────┐
│  Step 1: Cluster vectors into N partitions (IVF)   │
│  Step 2: Compress vectors with PQ (quantization)   │
│                                                     │
│  Partition 1: [v1, v5, v9]    ← only search        │
│  Partition 2: [v2, v6, v10]      relevant           │
│  Partition 3: [v3, v7, v11]      partitions         │
│  Partition 4: [v4, v8, v12]                         │
│                                                     │
│  Recall@10: ~95%  |  Latency: <10ms for 1B vectors │
│  Memory: Low (compressed vectors)                   │
└────────────────────────────────────────────────────┘
```

| Algorithm | Speed | Memory | Recall | Best For |
|---|---|---|---|---|
| HNSW | ★★★★★ | ★★☆☆☆ (high) | ★★★★★ | <10M vectors, quality-critical |
| IVF-PQ | ★★★★☆ | ★★★★★ (low) | ★★★☆☆ | >10M vectors, memory-constrained |
| IVF-Flat | ★★★☆☆ | ★★★☆☆ | ★★★★☆ | Medium scale, high recall needed |
| Flat (brute) | ★☆☆☆☆ | ★★★☆☆ | ★★★★★ | <100K vectors, development |

### 💡 Interview Insight

> **Q: How do you tune retrieval quality in a vector search system?**
>
> "I focus on three levers. First, the embedding model — this has the biggest impact. I benchmark 3-4 models on our actual queries and measure recall@5 against a labeled test set. Second, the index parameters — for HNSW, I tune ef_construction (build-time quality, higher = better but slower to build) and ef_search (query-time quality, higher = better recall but slower). I start with ef_construction=200 and ef_search=100, then adjust based on latency requirements. Third, metadata filtering — pre-filtering by category or department before vector search dramatically improves relevance. For ShopAssist, filtering by department ('electronics' vs 'clothing') before vector search eliminates irrelevant results that might be semantically similar but topically wrong."

---

## Screen 7: Advanced RAG — Beyond Naive Retrieve-and-Generate

Naive RAG (embed query → retrieve top-k → stuff into prompt) works for 70% of cases. Advanced RAG techniques handle the other 30%.

### Hybrid Search (Dense + Sparse)

Combining vector similarity (semantic) with BM25 keyword search (lexical) catches what either misses alone.

```python
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    """Combine dense vector search with sparse BM25 keyword search."""
    
    def __init__(self, documents: list[str], embeddings: list[list[float]]):
        self.documents = documents
        self.embeddings = np.array(embeddings)
        
        # BM25 index for keyword matching
        tokenized = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)
    
    def search(self, query: str, query_embedding: list[float],
               top_k: int = 5, alpha: float = 0.7) -> list[dict]:
        """
        Hybrid search with score fusion.
        alpha: weight for dense search (1.0 = pure vector, 0.0 = pure BM25)
        """
        # Dense scores (cosine similarity)
        q_emb = np.array(query_embedding)
        dense_scores = np.dot(self.embeddings, q_emb) / (
            np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(q_emb)
        )
        
        # Sparse scores (BM25)
        sparse_scores = self.bm25.get_scores(query.lower().split())
        
        # Normalize both to [0, 1]
        dense_norm = (dense_scores - dense_scores.min()) / (dense_scores.max() - dense_scores.min() + 1e-8)
        sparse_norm = (sparse_scores - sparse_scores.min()) / (sparse_scores.max() - sparse_scores.min() + 1e-8)
        
        # Weighted fusion
        combined = alpha * dense_norm + (1 - alpha) * sparse_norm
        
        # Return top-k
        top_indices = np.argsort(combined)[::-1][:top_k]
        return [
            {"document": self.documents[i], "score": combined[i], "dense": dense_scores[i], "sparse": sparse_scores[i]}
            for i in top_indices
        ]

# Example: "order #A1234 return" — BM25 catches the order number, dense catches "return policy"
```

### Re-Ranking with Cross-Encoders

First-stage retrieval is fast but approximate. Re-ranking uses a more expensive cross-encoder to reorder the top candidates.

```python
from sentence_transformers import CrossEncoder

# Cross-encoder scores query-document pairs jointly (more accurate than bi-encoder)
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")

def retrieve_and_rerank(query: str, retriever, top_k: int = 5, rerank_k: int = 20):
    """Two-stage retrieval: fast retrieval → accurate re-ranking."""
    # Stage 1: Fast retrieval of top-20 candidates
    candidates = retriever.search(query, top_k=rerank_k)
    
    # Stage 2: Re-rank with cross-encoder
    pairs = [(query, doc["document"]) for doc in candidates]
    rerank_scores = reranker.predict(pairs)
    
    # Sort by re-rank score and return top-k
    for i, doc in enumerate(candidates):
        doc["rerank_score"] = float(rerank_scores[i])
    
    candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
    return candidates[:top_k]
```

### HyDE (Hypothetical Document Embeddings)

Instead of embedding the query directly, generate a hypothetical answer and embed that — it's closer in embedding space to real documents.

```python
def hyde_retrieval(client, query: str, retriever) -> list[dict]:
    """HyDE: Generate hypothetical answer, embed it, search with it."""
    # Step 1: Generate a hypothetical answer
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer the question as if you were a customer support knowledge base article. Be specific and detailed."},
            {"role": "user", "content": query},
        ],
        temperature=0.0,
    )
    hypothetical_doc = response.choices[0].message.content
    
    # Step 2: Embed the hypothetical document (not the query)
    hyde_embedding = embed_texts([hypothetical_doc])[0]
    
    # Step 3: Search with the hypothetical document embedding
    return retriever.search_by_embedding(hyde_embedding, top_k=5)

# Query: "laptop restocking fee" → 
# HyDE generates: "Our restocking fee policy for electronics states that opened laptops 
#   are subject to a 15% restocking fee when returned within 30 days..."
# This hypothetical doc embeds closer to the actual policy document
```

### Multi-Query Retrieval

Generate multiple query perspectives, retrieve for each, deduplicate. The LLM rephrases the query from different angles — "laptop return fee" becomes "restocking charge electronics", "cost to return opened laptop", etc. — catching documents that match any phrasing.

### 💡 Interview Insight

> **Q: Walk me through how you'd improve a RAG system that's returning irrelevant results.**
>
> "I follow a systematic debugging process. First, I categorize failures: is the relevant document not in the top-20 results (retrieval miss), or is it retrieved but ranked too low (ranking issue)? For retrieval misses, I check if the document was chunked poorly — maybe the answer spans two chunks. I'd try smaller chunks or parent-child chunking. I also test hybrid search: sometimes the query contains specific terms like order numbers or product codes that BM25 catches but vector search misses. For ranking issues, I add a cross-encoder re-ranker — retrieve top-20 with fast vector search, then re-rank with a cross-encoder. This typically improves precision@5 by 15-25%. I also try HyDE for queries that are very different stylistically from the documents — customer questions vs formal policy language. Each change is evaluated on a test set of 50+ query-answer pairs before deploying."

---

## Screen 8: RAG Evaluation — Measuring What Matters

You can't improve what you can't measure. RAG evaluation has unique challenges because failures can occur at multiple points.

### RAGAS Framework

```
┌─────────────────────────────────────────────────────────┐
│                  RAGAS EVALUATION METRICS                 │
│                                                           │
│  ┌─────────────────┐                                     │
│  │ Context          │  Are the retrieved documents        │
│  │ Relevance        │  relevant to the question?          │
│  │                  │  (Retrieval quality)                 │
│  └────────┬────────┘                                     │
│           │                                               │
│  ┌────────▼────────┐                                     │
│  │ Faithfulness     │  Is the answer grounded in          │
│  │                  │  the retrieved context?              │
│  │                  │  (No hallucination)                  │
│  └────────┬────────┘                                     │
│           │                                               │
│  ┌────────▼────────┐                                     │
│  │ Answer           │  Does the answer actually           │
│  │ Relevance        │  address the question?              │
│  │                  │  (End-to-end quality)                │
│  └─────────────────┘                                     │
└─────────────────────────────────────────────────────────┘
```

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from datasets import Dataset

# Prepare evaluation dataset
eval_data = {
    "question": [
        "What is the restocking fee for opened electronics?",
        "How do I schedule a furniture pickup return?",
        "What's the shipping cost for express delivery?",
    ],
    "answer": [
        "The restocking fee for opened electronics is 15%.",
        "You can schedule a furniture pickup through our logistics portal.",
        "Express shipping costs $12.99 per order.",
    ],
    "contexts": [
        ["Electronics must be in original packaging. Opened items are subject to a 15% restocking fee."],
        ["Furniture returns must be scheduled with our logistics team. A pickup fee of $49.99 applies."],
        ["Free shipping on orders over $50. Express shipping: $12.99."],
    ],
    "ground_truth": [
        "Opened electronics have a 15% restocking fee when returned within 30 days.",
        "Furniture pickup returns are scheduled through the logistics portal with a $49.99 fee for items over 50 lbs.",
        "Express shipping costs $12.99.",
    ],
}

dataset = Dataset.from_dict(eval_data)

# Run RAGAS evaluation
results = evaluate(
    dataset=dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(results)
# {'faithfulness': 0.92, 'answer_relevancy': 0.88, 'context_precision': 0.85}
```

### Failure Modes

```
┌──────────────────────────────────────────────────────────┐
│                  RAG FAILURE TAXONOMY                     │
│                                                           │
│  1. RETRIEVAL MISS                                        │
│     Question: "What about international returns?"         │
│     Retrieved: docs about domestic returns                │
│     Cause: No relevant document exists OR poor embedding  │
│     Fix: Add missing docs, improve chunking               │
│                                                           │
│  2. CONTEXT STUFFING                                      │
│     Question: "Return policy for headphones?"             │
│     Retrieved: 10 loosely related chunks filling context   │
│     Cause: Too many results, low relevance threshold      │
│     Fix: Reduce top_k, add re-ranking, raise threshold    │
│                                                           │
│  3. HALLUCINATION DESPITE CONTEXT                         │
│     Question: "What's the restocking fee?"                │
│     Context: "15% restocking fee for electronics"         │
│     Answer: "The restocking fee is 20%"                   │
│     Cause: Model ignores or misreads context              │
│     Fix: Better prompt engineering, faithfulness checks    │
│                                                           │
│  4. WRONG CONTEXT SELECTED                                │
│     Question: "Laptop return after 45 days"               │
│     Retrieved: General return policy (not electronics)     │
│     Cause: Metadata filtering missing, ambiguous query     │
│     Fix: Add metadata filters, multi-query retrieval       │
└──────────────────────────────────────────────────────────┘
```

### Building an Evaluation Pipeline

```python
from dataclasses import dataclass

@dataclass
class RAGEvalResult:
    query: str
    retrieved_docs: list[str]
    answer: str
    ground_truth: str
    retrieval_hit: bool       # Was the correct doc in top-k?
    faithfulness_score: float  # Is answer grounded in context?
    answer_score: float        # Does answer match ground truth?
    failure_mode: str | None   # Categorized failure

def evaluate_rag_pipeline(
    pipeline, 
    test_cases: list[dict],
    judge_client,
) -> list[RAGEvalResult]:
    """End-to-end RAG evaluation with failure categorization."""
    results = []
    for case in test_cases:
        # Run the RAG pipeline
        retrieved = pipeline.retrieve(case["query"])
        answer = pipeline.generate(case["query"], retrieved)
        
        # Check if correct document was retrieved
        retrieval_hit = any(
            case["expected_doc_id"] in doc.metadata.get("id", "")
            for doc in retrieved
        )
        
        # LLM-as-Judge for faithfulness
        faithfulness = judge_faithfulness(
            judge_client, answer, [d.text for d in retrieved]
        )
        
        # Categorize failure mode
        failure = None
        if not retrieval_hit:
            failure = "retrieval_miss"
        elif faithfulness < 0.5:
            failure = "hallucination_despite_context"
        elif answer_score < 0.5:
            failure = "wrong_answer"
        
        results.append(RAGEvalResult(
            query=case["query"],
            retrieved_docs=[d.text for d in retrieved],
            answer=answer,
            ground_truth=case["ground_truth"],
            retrieval_hit=retrieval_hit,
            faithfulness_score=faithfulness,
            answer_score=answer_score,
            failure_mode=failure,
        ))
    
    return results
```

### 💡 Interview Insight

> **Q: How do you evaluate and monitor a RAG system in production?**
>
> "I evaluate RAG at three levels. First, retrieval quality: I maintain a test set of 200 query-document pairs and measure recall@5 weekly — does the correct document appear in the top 5? This catches embedding model degradation and index issues. Second, generation quality: I use RAGAS metrics — faithfulness (is the answer grounded in context?), answer relevance (does it address the question?), and context precision (are retrieved docs relevant?). Third, production monitoring: I sample 50 conversations daily, run them through LLM-as-Judge, and track scores over time. I also instrument failure detection — if the model says 'I don't have information about that,' I log it as a potential retrieval miss and add the topic to our knowledge base gap list. The failure taxonomy is critical: distinguishing retrieval misses from hallucinations guides very different interventions."

---

## Screen 9: GraphRAG — When Vectors Aren't Enough

Some questions require understanding **relationships** between entities — something vector search alone can't handle.

### The Limitation of Vector RAG

```
Question: "Which products from the same manufacturer as my laptop 
           are currently on sale?"

Vector RAG retrieves:
  - Laptop product page (relevant but doesn't answer the question)
  - Sale page (relevant but doesn't know which manufacturer)
  - Random manufacturer page (not the right one)

The model can't connect: laptop → manufacturer → other products → sale status
```

### Knowledge Graph Approach

```
┌──────────────────────────────────────────────────────┐
│              KNOWLEDGE GRAPH FOR SHOPASSIST            │
│                                                        │
│  [Customer: John]──ORDERED──▶[Order: #5567]            │
│                                │                       │
│                          CONTAINS                      │
│                                │                       │
│                    ┌───────────▼──────────┐             │
│                    │                      │             │
│           [Product: Laptop]     [Product: Mouse]        │
│              │         │            │                   │
│         MADE_BY    CATEGORY    MADE_BY                  │
│              │         │            │                   │
│    [Brand: Dell]  [Cat: Elec]  [Brand: Logitech]       │
│         │                          │                    │
│     ALSO_MAKES                ALSO_MAKES                │
│         │                          │                    │
│    [Monitor]  [Keyboard]     [Webcam]  [Headset]        │
│         │                          │                    │
│     ON_SALE                    ON_SALE                   │
│         │                          │                    │
│     [Sale: Summer2024]        [Sale: Summer2024]        │
└──────────────────────────────────────────────────────┘

Query path: Customer → Order → Product → Brand → Other Products → Sales
```

### Microsoft GraphRAG Approach

Microsoft's GraphRAG (2024) extracts entities and relationships from documents, builds a knowledge graph, and creates community summaries for global queries.

```python
# Simplified GraphRAG entity extraction
def extract_entities(client, text: str) -> dict:
    """Extract entities and relationships from text using LLM."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": (
                "Extract entities and relationships from the text. "
                "Return JSON with: entities (list of {name, type, description}) "
                "and relationships (list of {source, target, relation, description})."
            )},
            {"role": "user", "content": text},
        ],
        response_format={"type": "json_object"},
        temperature=0.0,
    )
    return json.loads(response.choices[0].message.content)

# Example extraction from a support document:
text = """
The Dell XPS 15 laptop comes with a 2-year manufacturer warranty from Dell Inc. 
Extended warranties can be purchased through our partner, SquareTrade. 
The XPS 15 is compatible with the Dell WD19 docking station and Dell P2722H monitor.
"""

entities = extract_entities(client, text)
# entities: [
#   {"name": "Dell XPS 15", "type": "Product", "description": "Laptop"},
#   {"name": "Dell Inc.", "type": "Manufacturer", "description": "Makes XPS 15"},
#   {"name": "SquareTrade", "type": "Partner", "description": "Extended warranty provider"},
#   {"name": "Dell WD19", "type": "Product", "description": "Docking station"},
#   {"name": "Dell P2722H", "type": "Product", "description": "Monitor"},
# ]
# relationships: [
#   {"source": "Dell XPS 15", "target": "Dell Inc.", "relation": "MANUFACTURED_BY"},
#   {"source": "Dell XPS 15", "target": "Dell WD19", "relation": "COMPATIBLE_WITH"},
#   {"source": "Dell XPS 15", "target": "Dell P2722H", "relation": "COMPATIBLE_WITH"},
#   {"source": "SquareTrade", "target": "Dell XPS 15", "relation": "PROVIDES_WARRANTY"},
# ]
```

### When to Use Graph vs Vector

| Question Type | Vector RAG | GraphRAG | Best Approach |
|---|---|---|---|
| "What's the return policy?" | ★★★★★ | ★★☆☆☆ | Vector |
| "Which accessories work with my laptop?" | ★★☆☆☆ | ★★★★★ | Graph |
| "Summarize all issues with Product X" | ★★★☆☆ | ★★★★★ | Graph (community summaries) |
| "How do I reset my password?" | ★★★★★ | ★☆☆☆☆ | Vector |
| "Which customers bought X also bought Y?" | ★☆☆☆☆ | ★★★★★ | Graph |
| "Compare warranty options for electronics" | ★★★★☆ | ★★★★☆ | Hybrid |

### Hybrid Vector + Graph Architecture

```
┌──────────────────────────────────────────────────────┐
│           HYBRID RAG ARCHITECTURE                     │
│                                                       │
│  User Query                                           │
│       │                                               │
│       ▼                                               │
│  ┌─────────────┐                                      │
│  │ Query Router │──── Factual lookup ───▶ Vector RAG   │
│  │             │                                      │
│  │             │──── Relationship Q ────▶ Graph RAG    │
│  │             │                                      │
│  │             │──── Complex/Global ───▶ Both + Merge  │
│  └─────────────┘                                      │
│                                                       │
│       Results merged and re-ranked                     │
│              │                                         │
│              ▼                                         │
│       ┌──────────┐                                     │
│       │   LLM    │──▶ Final Answer                     │
│       └──────────┘                                     │
└──────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: When would you add a knowledge graph to a RAG system?**
>
> "I add GraphRAG when I see two signals. First, users ask relationship questions that vector search can't answer — 'What other products are compatible with my laptop?', 'Which customers had the same issue?', 'Show me all products from this manufacturer.' Vector search finds similar text, but it can't traverse relationships. Second, when I need global summarization across many documents — 'What are the top 5 issues with our electronics department this quarter?' Microsoft's GraphRAG creates community summaries that answer these aggregate questions well. For ShopAssist, I'd implement a hybrid: vector RAG for direct knowledge lookups (return policies, FAQs) and graph RAG for product relationships, compatibility queries, and cross-customer pattern analysis. The query router examines the question type and directs to the appropriate retrieval strategy."

---

## Module 3 Quiz

**1. What is the primary advantage of RAG over fine-tuning for keeping an LLM updated with current information?**

- A) RAG is always cheaper than fine-tuning
- B) RAG updates are instant — just re-index documents, no retraining needed ✅
- C) RAG eliminates all hallucinations
- D) RAG makes the model run faster
- E) RAG doesn't require any data preparation

**2. In parent-child chunking, what is the purpose of using small chunks for retrieval but large chunks for generation?**

- A) It reduces the total number of embeddings needed
- B) Small chunks provide precise retrieval, while large parent chunks give the LLM enough context to generate complete answers ✅
- C) It makes the vector database smaller
- D) Large chunks are faster to embed
- E) It eliminates the need for re-ranking

**3. A RAG system retrieves the correct document but the LLM's answer contradicts the retrieved context. Which RAGAS metric would detect this?**

- A) Context Relevance
- B) Answer Relevance
- C) Faithfulness ✅
- D) Context Recall
- E) Precision

**4. Which advanced RAG technique is most effective when the user's query uses very different language from the documents?**

- A) Increasing chunk size
- B) HyDE (Hypothetical Document Embeddings) ✅
- C) Adding more metadata filters
- D) Using a smaller embedding model
- E) Reducing the number of retrieved documents

**5. When should you consider adding GraphRAG to a support system?**

- A) When documents are too large to embed
- B) When the vector database is too slow
- C) When users ask relationship and cross-entity questions that vector similarity can't answer ✅
- D) When you need to reduce API costs
- E) When the knowledge base has fewer than 100 documents

---

## Key Takeaways

1. **RAG grounds LLMs in your data** — reducing hallucinations, enabling real-time knowledge updates, and providing citation/attribution. It's the default architecture for knowledge-grounded applications.

2. **Document loading quality is foundational**. Bad extraction → bad chunks → bad retrieval → bad answers. Validate extraction quality on a sample before scaling.

3. **Chunking strategy matters more than most people think**. Parent-child chunking (small chunks for retrieval, large for context) is the sweet spot for most production systems.

4. **Embedding model selection** should be data-driven: benchmark 3-4 models on your actual queries. Open-source models (GTE, BGE) now rival proprietary ones.

5. **Start with pgvector** if you already run Postgres. Move to a dedicated vector DB only when you outgrow it (>10M vectors or need specialized features).

6. **Advanced RAG techniques** — hybrid search, re-ranking, HyDE, multi-query — each address specific failure modes. Apply them based on diagnosed problems, not as defaults.

7. **RAGAS evaluation** with faithfulness, answer relevance, and context precision gives you a quantitative handle on RAG quality. Monitor these in production.

8. **GraphRAG** complements vector RAG for relationship queries and global summarization. Use a query router to direct questions to the appropriate retrieval strategy.
