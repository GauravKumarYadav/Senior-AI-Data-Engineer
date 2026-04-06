---
tags: [ai, langchain, phase-5]
phase: 5
status: not-started
priority: high
---

# 🦜 LangChain

> **Phase:** 5 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[01 - LLM Fundamentals]], [[03 - RAG Architecture]], [[05 - LangGraph]], [[06 - AI Agents]]

---

## Checklist

### Core Abstractions
- [ ] LLMs vs ChatModels: `BaseLLM` (text → text) vs `BaseChatModel` (messages → message)
- [ ] Prompt templates: `ChatPromptTemplate`, `SystemMessage`, `HumanMessage`
- [ ] Output parsers: `StrOutputParser`, `JsonOutputParser`, `PydanticOutputParser`
- [ ] Document: `page_content` + `metadata` — the universal document format
- [ ] Callbacks: hooks for logging, tracing, streaming

### LCEL (LangChain Expression Language)
- [ ] `Runnable` interface: `invoke`, `batch`, `stream`, `ainvoke`
- [ ] Piping: `chain = prompt | model | parser` — compose components
- [ ] `RunnablePassthrough`: pass input through, useful for combining
- [ ] `RunnableParallel`: run multiple chains simultaneously
- [ ] `RunnableLambda`: wrap any function as a Runnable
- [ ] `RunnableBranch` / `RunnableRouter`: conditional routing
- [ ] `assign()`: add values to the running dict
- [ ] Why LCEL: streaming, batching, async — all for free via Runnable protocol

### Document Loaders & Splitters
- [ ] Loaders: `PyPDFLoader`, `WebBaseLoader`, `CSVLoader`, `DirectoryLoader`
- [ ] Text splitters: `RecursiveCharacterTextSplitter`, `TokenTextSplitter`
- [ ] `MarkdownHeaderTextSplitter`: split by headers, preserve hierarchy
- [ ] `HTMLSectionSplitter`: extract sections from HTML
- [ ] Custom splitters: extend `TextSplitter` base class

### Vector Stores & Retrievers
- [ ] Vector store interface: `add_documents`, `similarity_search`, `as_retriever()`
- [ ] Supported stores: Chroma, Pinecone, Weaviate, Qdrant, pgvector, FAISS
- [ ] Retriever types:
  - [ ] `VectorStoreRetriever`: basic similarity search
  - [ ] `MultiQueryRetriever`: generate query variations
  - [ ] `SelfQueryRetriever`: LLM extracts metadata filters
  - [ ] `ContextualCompressionRetriever`: compress retrieved docs
  - [ ] `EnsembleRetriever`: combine multiple retrievers (hybrid search)
  - [ ] `ParentDocumentRetriever`: retrieve small chunks, return parent

### Memory
- [ ] `ConversationBufferMemory`: store full conversation (simple, grows unbounded)
- [ ] `ConversationSummaryMemory`: LLM summarizes conversation (reduces tokens)
- [ ] `ConversationBufferWindowMemory`: keep last K exchanges
- [ ] `EntityMemory`: track entities mentioned in conversation
- [ ] Memory with LCEL: `RunnableWithMessageHistory`
- [ ] Chat message history: `ChatMessageHistory`, Redis/Postgres persistence

### Tools & Tool Calling
- [ ] `@tool` decorator: define tools with descriptions
- [ ] `BaseTool`: class-based tools with `_run` and `_arun`
- [ ] Tool calling: model decides which tool to call, with arguments
- [ ] Built-in tools: search (Tavily), Wikipedia, Python REPL
- [ ] `ToolMessage`: return tool results to the model
- [ ] `bind_tools()`: attach tools to a chat model
- [ ] Structured tool calling: multiple tools, parallel execution

### Tracing & Observability
- [ ] LangSmith: tracing, evaluation, datasets, playground
- [ ] `LANGCHAIN_TRACING_V2=true` — enable tracing
- [ ] Trace anatomy: runs, child runs, inputs, outputs, token counts
- [ ] Evaluators: custom scoring functions, LLM-as-judge
- [ ] Datasets: create test datasets, run evaluations

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] LangChain docs: https://python.langchain.com/
- [ ] LangChain tutorials: https://python.langchain.com/docs/tutorials/
- [ ] LangSmith docs: https://docs.smith.langchain.com/
- [ ] Harrison Chase talks (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Build a RAG chain using LangChain with LCEL
- When would you use LangChain vs building from scratch?
- How do you evaluate a LangChain application?
- Explain LCEL and why it's useful
