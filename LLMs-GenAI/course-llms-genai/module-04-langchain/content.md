# Module 4: LangChain

> **Scenario**: ShopAssist Inc. has built RAG pipelines with raw OpenAI calls and manual orchestration. It works, but the codebase is a tangled mess of prompt strings, retrieval logic, and parsing code. LangChain provides composable abstractions that let you build, swap, and scale LLM workflows without rewriting everything when you change a model or retrieval strategy.

---

## Screen 1: Core Abstractions — The Building Blocks

### Why LangChain?

You've been writing code like this:

```python
# ❌ Raw OpenAI — works, but tightly coupled
import openai

def answer_question(question: str, context: str) -> str:
    prompt = f"You are a ShopAssist agent.\n\nContext: {context}\n\nQuestion: {question}"
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content
```

Problems: changing to Anthropic means rewriting everything. Parsing is manual. No streaming. No batching. No tracing. LangChain wraps all of this behind a **uniform Runnable interface**.

### LLMs vs ChatModels

```
┌──────────────────────────────────────────────────────────┐
│                  MODEL ABSTRACTIONS                       │
│                                                           │
│   BaseLLM (legacy)              BaseChatModel (modern)    │
│   ┌─────────────────┐           ┌─────────────────────┐  │
│   │ Input: str      │           │ Input: List[Message] │  │
│   │ Output: str     │           │ Output: AIMessage    │  │
│   │                 │           │                       │  │
│   │ .invoke("hi")   │          │ .invoke([HumanMsg])   │  │
│   │ .batch(["a","b"])│          │ .batch([[msgs]])      │  │
│   │ .stream("hi")   │           │ .stream([msgs])      │  │
│   └─────────────────┘           └─────────────────────┘  │
│   Examples:                     Examples:                  │
│   - OpenAI (legacy)             - ChatOpenAI              │
│                                 - ChatAnthropic            │
│                                 - ChatOllama               │
│                                 - ChatVertexAI             │
│                                                           │
│   ⚠️ Use ChatModels — LLMs class is deprecated            │
└──────────────────────────────────────────────────────────┘
```

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, SystemMessage

# Both implement the same Runnablrface
gpt = ChatOpenAI(model="gpt-4o", temperature=0)
claude = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)

# Identical API — swap models without changing code
messages = [
    SystemMessage(content="You are ShopAssist, a helpful e-commerce support agent."),
    HumanMessage(content="What's your return policy?"),
]

response_gpt = gpt.invoke(messages)      # AIMessage
response_claude = claude.invoke(messages) # AIMessage

# Both return AIMessage with .content, .response_metadata, .usage_metadata
print(response_gpt.content)
print(response_gpt.usage_metadata)  # {'input_tokens': 25, 'output_tokens': 142}
```

### PromptTemplate vs ChatPromptTemplate

```python
from langchain_core.prompts import PromptTemplate, ChatPromptTemplate

# PromptTemplate — single string with variables (legacy, for LLMs)
simple = PromptTemplate.from_template(
    "Answer the customer's question about {topic}: {question}"
)
print(simple.invoke({"topic": "returns", "question": "Can I return shoes?"}))
# StringPromptValue: "Answer the customer's question about returns: Can I return shoes?"

# ChatPromptTemplate — structured messages (modern, for ChatModels)
chat_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are ShopAssist, an e-commerce support agent for {company}."),
    ("human", "Order ID: {order_id}\nQuestion: {question}"),
])

# .invoke() returns ChatPromptValue with .to_messages()
prompt_value = chat_prompt.invoke({
    "company": "ShopAssist",
    "order_id": "ORD-7788",
    "question": "Where is my package?",
})
print(prompt_value.to_messages())
# [SystemMessage("You are ShopAssist..."), HumanMessage("Order ID: ORD-7788\n...")]

# MessagesPlaceholder for dynamic message lists (conversation history)
from langchain_core.prompts import MessagesPlaceholder

chat_with_history = ChatPromptTemplate.from_messages([
    ("system", "You are ShopAssist."),
    MessagesPlaceholder("chat_history"),  # Inserts a list of messages here
    ("human", "{question}"),
])
```

### Output Parsers

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

# StrOutputParser — extracts .content from AIMessage
str_parser = StrOutputParser()
# AIMessage(content="Your order ships tomorrow") → "Your order ships tomorrow"

# JsonOutputParser — parses JSON from LLM output
json_parser = JsonOutputParser()

# PydanticOutputParser — validates against a schema
class TicketClassification(BaseModel):
    category: str = Field(description="One of: billing, shipping, product, account")
    priority: str = Field(description="One of: low, medium, high, urgent")
    summary: str = Field(description="One-sentence summary of the issue")

pydantic_parser = PydanticOutputParser(pydantic_object=TicketClassification)

# The parser provides format instructions to inject into prompts
print(pydantic_parser.get_format_instructions())
# "The output should be formatted as a JSON instance that conforms to the
#  JSON schema below..."

# Full chain: prompt → model → parser
classify_chain = (
    ChatPromptTemplate.from_messages([
        ("system", "Classify this support ticket.\n{format_instructions}"),
        ("human", "{ticket_text}"),
    ]).partial(format_instructions=pydantic_parser.get_format_instructions())
    | gpt
    | pydantic_parser
)
result = classify_chain.invoke({"ticket_text": "I was charged twice for order ORD-1234!"})
# TicketClassification(category='billing', priority='high', summary='Customer charged twice...')
```

### The Document Class

```python
from langchain_core.documents import Document

# Document = page_content + metadata — the universal unit of text in LangChain
doc = Document(
    page_content="Returns are accepted within 30 days of purchase with original receipt.",
    metadata={
        "source": "return_policy.pdf",
        "page": 3,
        "last_updated": "2026-01-15",
        "category": "policy",
    },
)

# Everything in LangChain that deals with text uses Document:
# - Document loaders produce List[Document]
# - Text splitters consume and produce List[Document]
# - Vector stores index Document objects
# - Retrievers return List[Document]
```

### 💡 Interview Insight

> **Q: Why use LangChain instead of calling LLM APIs directly?**
>
> "LangChain's value isn't the wrappers — it's the **composability and the Runnable interface**. Every component (prompts, models, parsers, retrievers) implements the same `.invoke()`, `.batch()`, `.stream()` interface. This means I can swap GPT-4o for Claude without touching my chain logic, add streaming by changing one method call, or batch 100 requests with automatic concurrency. The real killer feature is LCEL — LangChain Expression Language — which lets me compose these components with the `|` pipe operator, giving me automatic streaming, batching, and tracing for free. That said, for simple single-LLM-call apps, raw API calls are perfectly fine. LangChain shines when you have multi-step chains, multiple models, and need observability via LangSmith."

---

## Screen 2: LCEL — LangChain Expression Language

### The Runnable Interface

Every LangChain component implements the `Runnable` protocol:

```python
class Runnable:
    def invoke(self, input, config=None) -> output:     # Single input → single output
        ...
    def batch(self, inputs, config=None) -> list:        # List of inputs → list of outputs
        ...
    def stream(self, input, config=None) -> Iterator:    # Single input → streamed output
        ...
    async def ainvoke(self, input, config=None) -> output:  # Async versions of all three
        ...
    async def abatch(self, inputs, config=None) -> list:
        ...
    async def astream(self, input, config=None) -> AsyncIterator:
        ...
```

### Piping with `|` — Composing Chains

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# The | operator creates a RunnableSequence
chain = (
    ChatPromptTemplate.from_messages([
        ("system", "You are ShopAssist. Answer concisely."),
        ("human", "{question}"),
    ])
    | llm
    | StrOutputParser()
)

# Under the hood: prompt.invoke() → messages → llm.invoke() → AIMessage → parser.invoke() → str
result = chain.invoke({"question": "How do I track my order?"})
# "You can track your order by visiting shopassist.com/orders and entering your order ID."

# Streaming works end-to-end automatically!
for chunk in chain.stream({"question": "How do I track my order?"}):
    print(chunk, end="", flush=True)

# Batching with concurrency
questions = [
    {"question": "Return policy?"},
    {"question": "Shipping time?"},
    {"question": "Payment methods?"},
]
results = chain.batch(questions, config={"max_concurrency": 3})
```

### RunnablePassthrough — Forwarding Input

```python
from langchain_core.runnables import RunnablePassthrough

# RunnablePassthrough passes input through unchanged
# Useful when a chain needs the original input alongside computed values

retriever = vector_store.as_retriever()

rag_chain = (
    {
        "context": retriever,                    # question → retrieved docs
        "question": RunnablePassthrough(),        # question → question (unchanged)
    }
    | ChatPromptTemplate.from_messages([
        ("system", "Answer based on context:\n{context}"),
        ("human", "{question}"),
    ])
    | llm
    | StrOutputParser()
)

# Input "What's the return policy?" flows to BOTH retriever and passthrough
result = rag_chain.invoke("What's the return policy?")
```

### RunnableParallel — Fan Out

```python
from langchain_core.runnables import RunnableParallel

# Run multiple chains in parallel, collect results as dict
parallel = RunnableParallel(
    sentiment=sentiment_chain,
    category=category_chain,
    summary=summary_chain,
)

# All three chains run concurrently on the same input
result = parallel.invoke({"ticket": "I was charged twice and I'm furious!"})
# {
#   "sentiment": "angry",
#   "category": "billing",
#   "summary": "Double charge on recent order"
# }

# Shorthand: passing a dict to | also creates RunnableParallel
chain = (
    {"sentiment": sentiment_chain, "category": category_chain}
    | combine_results_chain
)
```

### RunnableLambda — Custom Functions

```python
from langchain_core.runnables import RunnableLambda

# Wrap any function as a Runnable
def format_docs(docs: list[Document]) -> str:
    return "\n\n".join(doc.page_content for doc in docs)

format_runnable = RunnableLambda(format_docs)

# Now it has .invoke(), .batch(), .stream() for free
rag_chain = (
    {
        "context": retriever | format_runnable,
        "question": RunnablePassthrough(),
    }
    | prompt
    | llm
    | StrOutputParser()
)

# You can also use the @chain decorator for more complex functions
from langchain_core.runnables import chain as chain_decorator

@chain_decorator
def classify_and_route(ticket: dict) -> str:
    classification = classify_chain.invoke(ticket)
    if classification.priority == "urgent":
        return escalation_chain.invoke(ticket)
    return standard_chain.invoke(ticket)
```

### RunnableBranch — Conditional Routing

```python
from langchain_core.runnables import RunnableBranch

# Route to different chains based on conditions
branch = RunnableBranch(
    (lambda x: x["category"] == "billing", billing_chain),
    (lambda x: x["category"] == "shipping", shipping_chain),
    (lambda x: x["category"] == "product", product_chain),
    general_chain,  # Default fallback (last positional arg)
)

result = branch.invoke({"category": "billing", "question": "Why was I charged twice?"})
```

### `.assign()` — Adding Keys to the State

```python
# .assign() adds new keys while keeping existing ones
chain = (
    RunnablePassthrough()
    .assign(category=classify_chain)       # Adds "category" key
    .assign(sentiment=sentiment_chain)     # Adds "sentiment" key
    .assign(response=response_chain)       # Can use category + sentiment
)

result = chain.invoke({"ticket": "Double charge on ORD-1234"})
# {
#   "ticket": "Double charge on ORD-1234",     ← original
#   "category": "billing",                      ← added by classify
#   "sentiment": "frustrated",                  ← added by sentiment
#   "response": "I apologize for the..."        ← added by response
# }
```

### LCEL Composition Summary

```
┌──────────────────────────────────────────────────────────────┐
│                    LCEL BUILDING BLOCKS                       │
│                                                              │
│  RunnableSequence      A | B | C         Sequential pipeline │
│  RunnableParallel      {"a": A, "b": B}  Fan-out, merge     │
│  RunnablePassthrough   Pass input as-is  Identity function   │
│  RunnableLambda        lambda/function   Custom logic        │
│  RunnableBranch        if/else routing   Conditional paths   │
│  .assign()             Add keys          Accumulate state    │
│  .bind()               Fix parameters    Partial application │
│  .with_config()        Set metadata      Tracing/tags        │
│  .with_retry()         Auto-retry        Error resilience    │
│  .with_fallbacks()     Backup chains     Graceful degradation│
└──────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: Explain LCEL and why it matters for production LLM applications.**
>
> "LCEL is LangChain's declarative composition syntax. The key insight is that every component — prompts, models, parsers, retrievers, custom functions — implements the same `Runnable` interface with `.invoke()`, `.batch()`, and `.stream()`. The `|` pipe operator chains them into a `RunnableSequence` where the output of one becomes the input of the next. Why this matters for production: First, **streaming is automatic** — if I stream the final chain, tokens flow through every step without extra code. Second, **batching with concurrency** — `.batch()` runs inputs in parallel with configurable concurrency. Third, **tracing** — every step is automatically logged to LangSmith with inputs, outputs, latency, and token counts. Fourth, **fallbacks** — `.with_fallbacks([backup_model])` adds automatic model failover. You can't get all of this with raw function calls without building significant infrastructure."

---

## Screen 3: Document Loaders & Text Splitters

### Document Loaders — Getting Data Into LangChain

```python
# PDF Loading
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("shopassist_return_policy.pdf")
docs = loader.load()  # List[Document] — one per page
# Each doc has .page_content (text) and .metadata (source, page number)
print(docs[0].metadata)  # {'source': 'shopassist_return_policy.pdf', 'page': 0}

# Web Page Loading
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://shopassist.com/faq")
docs = loader.load()  # Extracts text from HTML

# CSV Loading — each row becomes a Document
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader("products.csv", source_column="product_id")
docs = loader.load()
# Document(page_content="name: Wireless Mouse\nprice: $29.99\n...", metadata={"source": "P-1001"})

# Directory Loading — bulk ingest entire folders
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader(
    "shopassist_docs/",
    glob="**/*.pdf",               # File pattern
    loader_cls=PyPDFLoader,        # Loader for each file
    show_progress=True,
    use_multithreading=True,       # Parallel loading
)
docs = loader.load()  # All PDFs in directory → flat list of Documents
```

### Loader Comparison

| Loader | Input | Output | Best For |
|---|---|---|---|
| PyPDFLoader | PDF file | 1 doc per page | Standard PDFs |
| WebBaseLoader | URL | 1 doc per page | Web scraping |
| CSVLoader | CSV file | 1 doc per row | Tabular data |
| DirectoryLoader | Folder path | All files in dir | Bulk ingestion |
| UnstructuredLoader | Any format | Parsed elements | Complex docs |
| JSONLoader | JSON file | Extracted fields | API responses |
| Docx2txtLoader | Word doc | Full text | Office documents |

### Text Splitters — Chunking Documents

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    CharacterTextSplitter,
    TokenTextSplitter,
)

policy_text = """# Return Policy

## General Returns
Items can be returned within 30 days of purchase. Original receipt required.
Items must be in original packaging and unused condition.

## Electronics
Electronics have a 15-day return window. Opened software cannot be returned.
Defective electronics may be exchanged within the manufacturer warranty period.

## Clothing
Clothing can be returned within 45 days. Tags must be attached.
Worn or washed items cannot be returned. Undergarments are final sale.

## Refund Process
Refunds are processed within 5-7 business days to the original payment method.
Store credit is issued immediately at the time of return."""

# RecursiveCharacterTextSplitter — the default choice
# Tries to split on: "\n\n" → "\n" → " " → "" (in that order)
recursive_splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=30,
    separators=["\n\n", "\n", ". ", " ", ""],  # default hierarchy
)
chunks = recursive_splitter.split_text(policy_text)
# Splits at paragraph boundaries first, preserving semantic coherence

# CharacterTextSplitter — splits on a single separator only
char_splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=200,
    chunk_overlap=30,
)
# Less flexible — if no separator found, chunk may exceed chunk_size

# TokenTextSplitter — splits by token count (not characters)
token_splitter = TokenTextSplitter(
    chunk_size=100,       # 100 tokens per chunk
    chunk_overlap=20,     # 20 token overlap
    encoding_name="cl100k_base",  # GPT-4 tokenizer
)
# Use when you need precise token budgets for LLM context windows

# Splitting Documents (preserves metadata)
docs = recursive_splitter.split_documents(loaded_docs)
# Each chunk inherits metadata from parent doc + gets start_index
```

### Splitter Comparison

```
┌────────────────────────────────────────────────────────────────────┐
│              TEXT SPLITTER DECISION TREE                            │
│                                                                    │
│  Need precise token counts? ──Yes──▶ TokenTextSplitter            │
│         │                                                          │
│         No                                                         │
│         │                                                          │
│  Structured text (markdown/code)? ──Yes──▶ RecursiveCharacter     │
│         │                              (with language-specific     │
│         │                               separators)                │
│         No                                                         │
│         │                                                          │
│  General prose? ──▶ RecursiveCharacterTextSplitter                │
│                     (default: the safe choice 95% of the time)     │
│                                                                    │
│  Semantic boundaries matter a lot? ──▶ SemanticChunker            │
│                     (uses embeddings — slower, better quality)     │
└────────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: How do you decide chunk size and overlap for a RAG system?**
>
> "I start with `RecursiveCharacterTextSplitter` at 500-1000 characters with 10-20% overlap, then tune empirically. The key tradeoff: smaller chunks (200-500 chars) give more precise retrieval but may lack context for the LLM to generate complete answers. Larger chunks (1000-2000 chars) provide more context but dilute retrieval precision — the embedding represents a bigger, more diffuse semantic space. For ShopAssist, I'd use 500 chars for FAQ articles (short, focused answers) and 1000 chars for policy documents (need surrounding context for nuance). Overlap prevents information loss at chunk boundaries — if a key fact spans two chunks, overlap ensures at least one chunk captures it completely. I always validate chunk quality by inspecting 20-30 random chunks manually before building the full index."

---

## Screen 4: Vector Stores & Retrievers

### Vector Store Integration

```python
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_community.vectorstores import FAISS
from langchain_postgres import PGVector

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Chroma — great for development, persistent local storage
chroma_store = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="shopassist_docs",
    persist_directory="./chroma_db",
)

# FAISS — Facebook's similarity search, blazing fast, in-memory
faiss_store = FAISS.from_documents(chunks, embeddings)
faiss_store.save_local("faiss_index")                    # Save to disk
faiss_store = FAISS.load_local("faiss_index", embeddings) # Load back

# PGVector — production-grade, uses existing PostgreSQL
pgvector_store = PGVector.from_documents(
    documents=chunks,
    embedding=embeddings,
    connection="postgresql://user:pass@localhost:5432/shopassist",
    collection_name="support_docs",
)
```

### Vector Store Comparison

| Store | Speed | Persistence | Scalability | Best For |
|---|---|---|---|---|
| Chroma | Fast | Local disk | ~1M vectors | Dev/prototyping |
| FAISS | Fastest | Manual save | ~100M vectors | Read-heavy, in-memory |
| PGVector | Good | PostgreSQL | ~10M vectors | Existing Postgres infra |
| Pinecone | Fast | Cloud-managed | Billions | Fully managed SaaS |
| Weaviate | Fast | Self/cloud | Billions | Hybrid search built-in |

### Retriever Types

```python
# Basic VectorStoreRetriever
retriever = chroma_store.as_retriever(
    search_type="similarity",     # or "mmr" or "similarity_score_threshold"
    search_kwargs={"k": 5},       # Return top 5
)
docs = retriever.invoke("What's the return policy for electronics?")

# MMR (Maximal Marginal Relevance) — diversity in results
mmr_retriever = chroma_store.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 5,                    # Return 5 docs
        "fetch_k": 20,             # Fetch 20 candidates first
        "lambda_mult": 0.7,        # 0=max diversity, 1=max relevance
    },
)

# MultiQueryRetriever — generates multiple query variants with LLM
from langchain.retrievers import MultiQueryRetriever

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=chroma_store.as_retriever(),
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0.3),
)
# "return policy electronics" generates:
# - "What is the electronics return window?"
# - "Can I return opened electronics?"
# - "Electronics refund policy details"
# Retrieves docs for ALL variants, deduplicates

# SelfQueryRetriever — extracts metadata filters from natural language
from langchain.retrievers import SelfQueryRetriever

metadata_field_info = [
    {"name": "category", "type": "string", "description": "Document category: policy, faq, product"},
    {"name": "last_updated", "type": "string", "description": "Last update date in YYYY-MM-DD"},
]

self_query_retriever = SelfQueryRetriever.from_llm(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    vectorstore=chroma_store,
    document_contents="ShopAssist customer support documents",
    metadata_field_info=metadata_field_info,
)
# Query: "return policy updated after January 2026"
# → Semantic search: "return policy" + Filter: last_updated > "2026-01-01"

# EnsembleRetriever — combines dense (vector) + sparse (BM25) retrieval
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25 = BM25Retriever.from_documents(chunks, k=5)
vector_retriever = chroma_store.as_retriever(search_kwargs={"k": 5})

ensemble = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.4, 0.6],  # 40% BM25 + 60% vector
)
# Catches keyword matches BM25 finds AND semantic matches vectors find

# ParentDocumentRetriever — small chunks for search, big chunks for context
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

parent_retriever = ParentDocumentRetriever(
    vectorstore=chroma_store,
    docstore=InMemoryStore(),
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
parent_retriever.add_documents(docs)
# Search matches small 400-char chunks → returns parent 2000-char chunks
```

### Similarity Search vs MMR

```
┌───────────────────────────────────────────────────────────┐
│         SIMILARITY SEARCH vs MMR                          │
│                                                           │
│  Query: "return policy"                                   │
│                                                           │
│  Similarity Search (top 3):          MMR (top 3):         │
│  ┌──────────────────────┐           ┌──────────────────┐  │
│  │ 1. General return     │          │ 1. General return │  │
│  │    policy (0.95)      │          │    policy (0.95)  │  │
│  │ 2. Return policy      │          │ 2. Electronics    │  │
│  │    FAQ (0.93)         │          │    returns (0.88) │  │
│  │ 3. Return policy      │          │ 3. Refund process │  │
│  │    update (0.91)      │          │    (0.82)         │  │
│  └──────────────────────┘           └──────────────────┘  │
│                                                           │
│  Problem: Results 1-3 all      MMR balances relevance     │
│  say basically the same thing  with diversity — covers    │
│                                 more ground                │
└───────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: How would you design a retrieval strategy for a support system with diverse document types?**
>
> "I'd use an `EnsembleRetriever` combining BM25 and vector search as the foundation — BM25 catches exact product IDs and order numbers that semantic search misses, while vectors handle paraphrased questions. I'd wrap this with `MultiQueryRetriever` to generate query variants, increasing recall for ambiguous questions. For metadata-rich documents like our product catalog, I'd add `SelfQueryRetriever` so 'laptops under $500 updated this month' automatically becomes a filtered search. Finally, `ParentDocumentRetriever` with 400-char child chunks and 2000-char parent chunks gives us precise matching with rich context. The weights and parameters get tuned against a golden test set of 100+ question-answer pairs evaluated with RAGAS metrics."

---

## Screen 5: Memory — Conversational Context

### The Memory Problem

```
Without memory:                     With memory:
┌──────────────────────────┐       ┌──────────────────────────┐
│ User: Where's my order?  │       │ User: Where's my order?  │
│ Bot: I need your order # │       │ Bot: I need your order # │
│                          │       │                          │
│ User: It's ORD-7788     │       │ User: It's ORD-7788      │
│ Bot: I need your order # │       │ Bot: ORD-7788 shipped    │
│      (forgot everything!)│       │      via FedEx, arrives  │
│                          │       │      Thursday!            │
└──────────────────────────┘       └──────────────────────────┘
```

### Legacy Memory Classes (Still Common in Interviews)

```python
from langchain.memory import (
    ConversationBufferMemory,
    ConversationSummaryMemory,
    ConversationBufferWindowMemory,
    ConversationEntityMemory,
)

# ConversationBufferMemory — stores ALL messages verbatim
buffer_memory = ConversationBufferMemory(return_messages=True)
buffer_memory.save_context(
    {"input": "Where's my order ORD-7788?"},
    {"output": "Let me look that up. ORD-7788 shipped via FedEx."},
)
buffer_memory.load_memory_variables({})
# {'history': [HumanMessage("Where's my order ORD-7788?"),
#              AIMessage("Let me look that up...")]}
# ⚠️ Problem: grows unbounded — 200 messages = huge token cost

# ConversationBufferWindowMemory — keeps last K exchanges
window_memory = ConversationBufferWindowMemory(k=10, return_messages=True)
# Only stores last 10 human-AI pairs — old messages are dropped
# Good for: cost control. Bad for: loses early context.

# ConversationSummaryMemory — LLM summarizes older messages
summary_memory = ConversationSummaryMemory(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    return_messages=True,
)
# After each exchange, summarizes conversation so far
# "Customer asked about order ORD-7788 which shipped via FedEx..."
# Good for: long conversations. Bad for: summary cost, detail loss.

# ConversationEntityMemory — tracks entities mentioned in conversation
entity_memory = ConversationEntityMemory(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    return_messages=True,
)
# Extracts and updates entities: {"ORD-7788": "FedEx shipment, shipped March 1"}
# Good for: support agents tracking customer/order details
```

### Memory Comparison

| Memory Type | Token Growth | Detail Retention | Cost | Best For |
|---|---|---|---|---|
| Buffer | O(n) — linear | 100% — perfect | High | Short conversations |
| BufferWindow(k) | O(k) — bounded | Recent only | Low | Cost-sensitive apps |
| Summary | O(1) — constant | Compressed | Medium (LLM calls) | Long conversations |
| Entity | O(entities) | Entity-focused | Medium | Support/CRM contexts |

### RunnableWithMessageHistory (The Modern Way)

```python
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import BaseChatMessageHistory, InMemoryChatMessageHistory

# Step 1: Define your base chain (no memory baked in)
chain = (
    ChatPromptTemplate.from_messages([
        ("system", "You are ShopAssist, an e-commerce support agent."),
        MessagesPlaceholder("chat_history"),
        ("human", "{question}"),
    ])
    | ChatOpenAI(model="gpt-4o")
    | StrOutputParser()
)

# Step 2: Create a session store (in production: Redis, PostgreSQL, etc.)
session_store: dict[str, InMemoryChatMessageHistory] = {}

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in session_store:
        session_store[session_id] = InMemoryChatMessageHistory()
    return session_store[session_id]

# Step 3: Wrap the chain with message history
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="question",
    history_messages_key="chat_history",
)

# Step 4: Invoke with session_id in config
config = {"configurable": {"session_id": "customer-abc-123"}}

r1 = chain_with_history.invoke(
    {"question": "Where's my order ORD-7788?"},
    config=config,
)
# "Let me look into order ORD-7788 for you..."

r2 = chain_with_history.invoke(
    {"question": "When will it arrive?"},  # "it" = ORD-7788 from context
    config=config,
)
# "Based on the tracking information for ORD-7788, it should arrive Thursday."
# ✅ The chain remembers the conversation!
```

### 💡 Interview Insight

> **Q: How do you manage conversation memory in a production chatbot?**
>
> "I use `RunnableWithMessageHistory` — it's LangChain's modern approach that separates the chain logic from history management. The key architectural decision is the history backend: `InMemoryChatMessageHistory` for development, Redis for high-throughput production (sub-millisecond reads), or PostgreSQL for auditability. For long conversations, I implement a hybrid strategy: keep the last 10 messages verbatim in the prompt, plus a rolling summary of older messages generated by a cheap model like GPT-4o-mini. This caps token costs while preserving important context. The `session_id` maps to the user's support ticket ID, so if they return later, the agent has full context. I also set a maximum history window to prevent prompt injection via extremely long conversations."

---

## Screen 6: Tools & Tool Calling

### Why Tools?

An LLM can *talk* about tracking orders, but it can't *actually* track orders. Tools bridge this gap — they let the model call real functions.

```
┌──────────────────────────────────────────────────────────┐
│              TOOL CALLING FLOW                            │
│                                                           │
│  User: "Where's my order ORD-7788?"                      │
│         │                                                 │
│         ▼                                                 │
│  ┌──────────────┐                                        │
│  │   LLM        │  "I need to call track_order()"        │
│  │  (decides)   │──────────────────────┐                 │
│  └──────────────┘                      │                 │
│         │                              ▼                 │
│         │                    ┌──────────────────┐        │
│         │                    │  track_order()   │        │
│         │                    │  order_id="7788" │        │
│         │                    └────────┬─────────┘        │
│         │                             │                  │
│         │    result: {"status":       │                  │
│         │     "shipped", "eta":       │                  │
│         ▼     "Thursday"}             ▼                  │
│  ┌──────────────┐◀────────────────────┘                  │
│  │   LLM        │                                        │
│  │ (synthesizes)│  "ORD-7788 shipped, arrives Thursday!" │
│  └──────────────┘                                        │
└──────────────────────────────────────────────────────────┘
```

### Defining Tools

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field

# Method 1: @tool decorator (simplest)
@tool
def track_order(order_id: str) -> dict:
    """Look up the current status and ETA for a customer order.
    
    Args:
        order_id: The order ID (e.g., 'ORD-7788')
    """
    # In production: call your order management API
    return {
        "order_id": order_id,
        "status": "shipped",
        "carrier": "FedEx",
        "tracking": "FX123456789",
        "eta": "2026-04-08",
    }

@tool
def search_products(query: str, max_price: float | None = None) -> list[dict]:
    """Search the ShopAssist product catalog.
    
    Args:
        query: Product search query
        max_price: Optional maximum price filter
    """
    # In production: query your product database
    return [{"name": "Wireless Mouse", "price": 29.99, "sku": "P-1001"}]

@tool
def process_return(order_id: str, reason: str) -> dict:
    """Initiate a return for a customer order.
    
    Args:
        order_id: The order ID to return
        reason: Customer's reason for the return
    """
    return {"return_id": "RET-001", "status": "initiated", "label_url": "..."}

# Method 2: Pydantic schema for complex inputs
class RefundInput(BaseModel):
    order_id: str = Field(description="Order ID to refund")
    amount: float = Field(description="Refund amount in USD")
    reason: str = Field(description="Reason for refund")

@tool(args_schema=RefundInput)
def process_refund(order_id: str, amount: float, reason: str) -> dict:
    """Process a refund for a customer order."""
    return {"refund_id": "REF-001", "status": "processed", "amount": amount}
```

### Binding Tools & Tool Calling

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, ToolMessage

llm = ChatOpenAI(model="gpt-4o")
tools = [track_order, search_products, process_return, process_refund]

# bind_tools tells the LLM what tools are available
llm_with_tools = llm.bind_tools(tools)

# The LLM decides whether to call a tool
response = llm_with_tools.invoke([
    HumanMessage(content="Where is order ORD-7788?")
])

# response.tool_calls contains the LLM's tool invocation request
print(response.tool_calls)
# [{'name': 'track_order', 'args': {'order_id': 'ORD-7788'}, 'id': 'call_abc123'}]

# Execute the tool and send result back
tool_result = track_order.invoke(response.tool_calls[0]["args"])
tool_message = ToolMessage(
    content=str(tool_result),
    tool_call_id=response.tool_calls[0]["id"],
)

# LLM synthesizes the final answer
final = llm_with_tools.invoke([
    HumanMessage(content="Where is order ORD-7788?"),
    response,           # AIMessage with tool_calls
    tool_message,       # ToolMessage with result
])
print(final.content)
# "Your order ORD-7788 has been shipped via FedEx (tracking: FX123456789)
#  and is expected to arrive on April 8, 2026."
```

### AgentExecutor — The Tool Loop

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", """You are ShopAssist, a customer support agent.
    Use the provided tools to look up order information, search products,
    and process returns/refunds. Always verify order IDs before taking action."""),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),  # Where tool call history goes
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,          # Print reasoning steps
    max_iterations=5,      # Safety: prevent infinite loops
    handle_parsing_errors=True,
)

# The executor handles the full loop: LLM → tool call → result → LLM → ...
result = agent_executor.invoke({
    "input": "I want to return order ORD-7788 because the item was defective",
    "chat_history": [],
})
# Agent will:
# 1. Call track_order("ORD-7788") to verify the order exists
# 2. Call process_return("ORD-7788", "item was defective") to initiate return
# 3. Synthesize a response with the return details
```

### 💡 Interview Insight

> **Q: How does tool calling work with LLMs, and what are the failure modes?**
>
> "Tool calling works through a structured protocol: the LLM receives tool schemas (name, description, parameter types) and decides when to call them. Modern models like GPT-4o and Claude 3.5 natively support tool calling — they output structured JSON with the function name and arguments rather than free text. The LLM doesn't execute the tool itself; our code executes it and sends the result back as a `ToolMessage`. Common failure modes: (1) the LLM calls the wrong tool — good tool descriptions and names are critical; (2) it hallucinates parameter values — I validate all tool inputs with Pydantic before execution; (3) infinite loops — `max_iterations` caps the agent loop; (4) the LLM tries to call tools that don't exist — `bind_tools()` constrains available tools. For ShopAssist, I also add confirmation steps before destructive actions like refunds — the agent must ask the user to confirm before calling `process_refund()`."

---

## Screen 7: LangSmith — Observability & Evaluation

### Why LangSmith?

```
┌──────────────────────────────────────────────────────────────┐
│            WITHOUT LANGSMITH           WITH LANGSMITH        │
│                                                              │
│  ❌ "The agent gave a wrong answer"   ✅ Full trace:         │
│     ...but why?                          Prompt → LLM call   │
│                                          → Tool call → result│
│  ❌ "It's slow sometimes"              ✅ Latency per step:  │
│     ...which step?                       Retrieval: 200ms    │
│                                          LLM: 1.2s          │
│                                          Tool: 50ms          │
│                                                              │
│  ❌ "Did the new prompt help?"         ✅ A/B comparison:    │
│     ...no way to compare                 v1: 72% correct     │
│                                          v2: 89% correct     │
│                                                              │
│  ❌ "Is it production-ready?"          ✅ Eval dataset:      │
│     ...vibes-based assessment            100 test cases,     │
│                                          scored automatically │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up Tracing

```python
import os

# Enable LangSmith tracing (set these environment variables)
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__..."  # Your LangSmith API key
os.environ["LANGCHAIN_PROJECT"] = "shopassist-production"

# That's it! All LangChain calls are now traced automatically.
# Every chain.invoke() logs:
# - Input/output at each step
# - Latency per step
# - Token counts and cost
# - Tool calls and results
# - Error traces with full stack
```

### Evaluation Datasets

```python
from langsmith import Client

client = Client()

# Create a golden evaluation dataset
dataset = client.create_dataset("shopassist-qa-golden")

# Add test cases
examples = [
    {
        "input": {"question": "What's the return window for electronics?"},
        "expected_output": "15 days",
    },
    {
        "input": {"question": "Can I return worn clothing?"},
        "expected_output": "No, worn or washed items cannot be returned",
    },
    {
        "input": {"question": "How long do refunds take?"},
        "expected_output": "5-7 business days",
    },
]

for ex in examples:
    client.create_example(
        inputs=ex["input"],
        outputs={"answer": ex["expected_output"]},
        dataset_id=dataset.id,
    )
```

### Running Evaluations

```python
from langsmith.evaluation import evaluate, LangChainStringEvaluator

# Define evaluators
correctness = LangChainStringEvaluator("qa")      # Is answer correct vs reference?
helpfulness = LangChainStringEvaluator("helpfulness")  # Is it helpful?

# Run evaluation against the dataset
def predict(inputs: dict) -> dict:
    result = rag_chain.invoke(inputs["question"])
    return {"answer": result}

results = evaluate(
    predict,
    data="shopassist-qa-golden",
    evaluators=[correctness, helpfulness],
    experiment_prefix="rag-v2-gpt4o",
)

# View results in LangSmith UI:
# - Aggregate scores across all test cases
# - Per-example pass/fail with explanations
# - Comparison with previous experiment runs
# - Regression detection between versions
```

### Few-Shot Example Selection

```python
from langchain_core.example_selectors import SemanticSimilarityExampleSelector

# Store examples in a vector store, retrieve the most relevant ones per query
example_selector = SemanticSimilarityExampleSelector.from_examples(
    [
        {"input": "Where's my order?", "output": "I'd be happy to help track your order. Could you provide your order ID?"},
        {"input": "I want a refund", "output": "I understand you'd like a refund. Let me pull up your order details."},
        {"input": "This product is broken", "output": "I'm sorry to hear that. Let's get this resolved — I can help with a return or exchange."},
    ],
    OpenAIEmbeddings(),
    Chroma,
    k=2,  # Select 2 most similar examples per query
)

# Use in prompt
from langchain_core.prompts import FewShotChatMessagePromptTemplate

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_selector=example_selector,
    example_prompt=ChatPromptTemplate.from_messages([
        ("human", "{input}"),
        ("ai", "{output}"),
    ]),
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are ShopAssist. Follow the style of these examples:"),
    few_shot_prompt,
    ("human", "{question}"),
])
```

### 💡 Interview Insight

> **Q: How do you evaluate and monitor LLM applications in production?**
>
> "I build a three-layer evaluation strategy. First, **offline evaluation** with a golden dataset of 100+ question-answer pairs in LangSmith, scored by LLM judges for correctness, helpfulness, and faithfulness. I run this on every prompt or retrieval change to catch regressions. Second, **online monitoring** via LangSmith tracing on 100% of production traffic — I track latency distributions, token costs, error rates, and tool call success rates with alerts on anomalies. Third, **user feedback loops** — thumbs up/down buttons in the support chat that feed back into LangSmith as annotations, letting me find the worst-performing queries and add them to the golden set. The key insight is that LLM evaluation is probabilistic — a single test case means nothing, but trends across hundreds of examples reveal real quality changes."

---

## Screen 8: Putting It All Together — ShopAssist Full Chain

### Complete ShopAssist Support Agent

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.tools import tool
from langchain_chroma import Chroma
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.chat_history import InMemoryChatMessageHistory

# --- 1. Models ---
llm = ChatOpenAI(model="gpt-4o", temperature=0)
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# --- 2. Vector Store (pre-loaded with ShopAssist docs) ---
vectorstore = Chroma(
    persist_directory="./shopassist_chroma",
    embedding_function=embeddings,
    collection_name="support_docs",
)
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 5, "fetch_k": 20},
)

# --- 3. Tools ---
@tool
def track_order(order_id: str) -> dict:
    """Look up current status and tracking info for a customer order."""
    # Production: call order management API
    return {"order_id": order_id, "status": "shipped", "eta": "2026-04-08"}

@tool
def search_knowledge_base(query: str) -> str:
    """Search ShopAssist support documents for policy and FAQ information."""
    docs = retriever.invoke(query)
    return "\n\n".join(doc.page_content for doc in docs)

@tool
def process_return(order_id: str, reason: str) -> dict:
    """Initiate a return for a customer order. Requires confirmation."""
    return {"return_id": "RET-001", "status": "initiated"}

@tool
def escalate_to_human(reason: str, priority: str = "medium") -> dict:
    """Escalate the conversation to a human agent when the issue is complex."""
    return {"ticket_id": "ESC-001", "status": "queued", "wait_time": "~5 minutes"}

tools = [track_order, search_knowledge_base, process_return, escalate_to_human]

# --- 4. Agent Prompt ---
system_prompt = """You are ShopAssist, an AI customer support agent for an e-commerce platform.

Guidelines:
1. Always greet the customer warmly and professionally.
2. Use search_knowledge_base for policy questions (returns, shipping, warranties).
3. Use track_order when customers ask about order status.
4. ALWAYS confirm with the customer before calling process_return or process_refund.
5. Escalate to a human agent for: billing disputes over $100, legal threats, or frustrated customers.
6. Never make up information — if you don't know, say so and offer to escalate.
7. Keep responses concise but helpful — no walls of text."""

prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    MessagesPlaceholder("chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

# --- 5. Agent + Executor ---
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, max_iterations=5, verbose=True)

# --- 6. Memory ---
sessions: dict[str, InMemoryChatMessageHistory] = {}

def get_history(session_id: str) -> InMemoryChatMessageHistory:
    if session_id not in sessions:
        sessions[session_id] = InMemoryChatMessageHistory()
    return sessions[session_id]

agent_with_memory = RunnableWithMessageHistory(
    executor,
    get_history,
    input_messages_key="input",
    history_messages_key="chat_history",
)

# --- 7. Usage ---
config = {"configurable": {"session_id": "cust-alice-001"}}

response = agent_with_memory.invoke(
    {"input": "Hi! I ordered a laptop last week, order ORD-7788. Where is it?"},
    config=config,
)
print(response["output"])
# "Hello! Let me look up order ORD-7788 for you. Your laptop has been shipped
#  and is expected to arrive on April 8, 2026. Is there anything else I can help with?"

response = agent_with_memory.invoke(
    {"input": "Actually, I want to return it when it arrives. It's the wrong model."},
    config=config,
)
print(response["output"])
# "I understand — receiving the wrong model is frustrating. I can help you initiate
#  a return for ORD-7788. Just to confirm: you'd like to return the laptop because
#  it's the wrong model. Shall I proceed?"
```

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                  SHOPASSIST LANGCHAIN AGENT                      │
│                                                                  │
│  Customer Message                                                │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────┐                                           │
│  │ RunnableWith     │◀── Session History (Redis/Memory)         │
│  │ MessageHistory   │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │ AgentExecutor    │                                           │
│  │  ┌────────────┐  │    ┌──────────────────────────────┐       │
│  │  │ LLM (GPT-4o│──┼───▶│ Tools:                      │       │
│  │  │ + tools)   │  │    │  • track_order()             │       │
│  │  └──────┬─────┘  │    │  • search_knowledge_base()   │       │
│  │         │        │    │  • process_return()           │       │
│  │    Loop until    │    │  • escalate_to_human()        │       │
│  │    done or max   │    └──────────────┬───────────────┘       │
│  │    iterations    │                   │                        │
│  └────────┬─────────┘                   │                        │
│           │                    ┌────────▼──────────┐             │
│           │                    │ Chroma VectorStore│             │
│           │                    │ (policies, FAQs)  │             │
│           │                    └───────────────────┘             │
│           ▼                                                      │
│  Final Response → Customer                                       │
│                                                                  │
│  [All steps traced in LangSmith]                                 │
└──────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: Walk me through how you'd build a production support agent with LangChain.**
>
> "I'd architect it in layers. **Model layer**: `ChatOpenAI` with `gpt-4o` for reasoning, `gpt-4o-mini` for classification and summarization — swap via the Runnable interface without changing chain logic. **Retrieval layer**: Chroma or PGVector with `EnsembleRetriever` (BM25 + vector), wrapped as a tool so the agent decides when to search. **Tool layer**: four core tools — `track_order`, `search_knowledge_base`, `process_return`, `escalate_to_human` — each with Pydantic-validated inputs and idempotent execution. **Memory layer**: `RunnableWithMessageHistory` backed by Redis, keyed by session ID. **Safety layer**: `max_iterations=5` on `AgentExecutor`, confirmation prompts before destructive actions, input validation on all tool parameters, and an escalation tool for edge cases. **Observability**: LangSmith tracing on 100% of traffic with alerts on error rates and latency. The entire thing runs behind a FastAPI endpoint with WebSocket support for streaming responses."

---

## Module 4 Quiz

**1. What is the main advantage of LCEL's pipe (`|`) operator over manually chaining function calls?**

- A) It makes the code run faster
- B) It automatically provides streaming, batching, async, and tracing across the entire chain ✅
- C) It reduces the number of API calls to the LLM
- D) It eliminates the need for prompt templates
- E) It makes the code shorter but has no functional benefit

**2. Which retriever combines BM25 (keyword) and vector (semantic) search for hybrid retrieval?**

- A) MultiQueryRetriever
- B) SelfQueryRetriever
- C) EnsembleRetriever ✅
- D) ParentDocumentRetriever
- E) ContextualCompressionRetriever

**3. In LangChain's tool calling flow, what happens AFTER the LLM outputs a `tool_calls` response?**

- A) The LLM executes the tool internally
- B) LangChain sends the tool schema to the API provider for execution
- C) Your application code executes the tool and sends the result back as a ToolMessage ✅
- D) The tool call is logged but not executed
- E) The AgentExecutor automatically generates a mock response

**4. What problem does `RunnableWithMessageHistory` solve compared to legacy memory classes?**

- A) It makes conversations faster
- B) It separates chain logic from history management, supporting multiple sessions and swappable backends ✅
- C) It reduces token usage by compressing messages
- D) It eliminates the need for a system prompt
- E) It automatically summarizes long conversations

**5. A ShopAssist customer asks "laptops under $500 updated this month." Which retriever type automatically extracts metadata filters from this natural language query?**

- A) MultiQueryRetriever
- B) SelfQueryRetriever ✅
- C) EnsembleRetriever
- D) ParentDocumentRetriever
- E) VectorStoreRetriever with MMR

---

## Key Takeaways

1. **LangChain's power is the Runnable interface** — every component (prompts, models, parsers, retrievers, tools) implements `.invoke()`, `.batch()`, `.stream()`. This uniformity enables composability, model swapping, and automatic observability.

2. **LCEL lets you compose chains declaratively** with the `|` pipe operator. `RunnableParallel` for fan-out, `RunnablePassthrough` for forwarding, `RunnableLambda` for custom logic, `RunnableBranch` for routing, and `.assign()` for accumulating state.

3. **Document loaders and text splitters** are your ingestion pipeline. `RecursiveCharacterTextSplitter` is the default choice for 95% of use cases. Always validate chunk quality manually.

4. **Choose your retriever strategy based on failure modes**: `EnsembleRetriever` for hybrid search, `MultiQueryRetriever` for query expansion, `SelfQueryRetriever` for metadata filtering, `ParentDocumentRetriever` for precision + context.

5. **Memory has evolved** — legacy classes (`ConversationBufferMemory`, etc.) still appear in interviews, but `RunnableWithMessageHistory` is the modern approach. Separate chain logic from history storage.

6. **Tools bridge the gap** between language and action. The `@tool` decorator + `bind_tools()` + `AgentExecutor` loop lets the LLM decide when to call functions, but your code executes them with proper validation and safety checks.

7. **LangSmith is non-negotiable for production** — tracing, evaluation datasets, regression testing, and cost monitoring. Without observability, you're flying blind with probabilistic systems.
