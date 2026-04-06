# AI System Design Problems

This module covers nine foundational AI system design problems you'll encounter in real-world production environments. Each problem is presented with its architecture, key components, tradeoffs, and scale considerations. Think of these as the "greatest hits" — the patterns that show up again and again across companies, domains, and use cases.

The goal isn't to memorize blueprints. It's to internalize the *reasoning* behind design decisions so you can adapt when the requirements inevitably shift.

---

## 1. Enterprise RAG Q&A System

Retrieval-Augmented Generation (RAG) is the workhorse pattern for grounding LLMs in private, enterprise data. Instead of fine-tuning a model on your docs (expensive, stale), you retrieve relevant chunks at query time and feed them as context. The LLM generates answers *from* those chunks, with citations.

This matters because enterprises sit on mountains of unstructured knowledge — Confluence wikis, SharePoint sites, PDFs, Slack threads — and employees waste hours searching for answers that already exist somewhere.

### Architecture

The system splits into two major pipelines: **ingestion** and **query**.

**Ingestion pipeline:** Data sources (PDF, Confluence, SharePoint) → Document parser → Chunking engine → Embedding model → Vector database. A metadata store runs alongside, tracking document permissions, source URLs, and freshness timestamps.

**Query pipeline:** User question → Auth check → Query embedding → Hybrid retrieval (dense vector search + BM25 sparse search) → Re-ranker → LLM generation with context → Response with citations → Semantic cache write.

A feedback service sits off to the side, collecting thumbs-up/down signals and feeding them into evaluation dashboards.

### Key Components

- **Chunking engine**: This is deceptively critical. Naive fixed-size chunking (e.g., 512 tokens) breaks semantic boundaries. Better approaches use recursive character splitting with overlap, or — even better — semantic chunking that respects document structure (headers, paragraphs, code blocks). Chunk size directly impacts retrieval quality: too small and you lose context, too large and you dilute relevance.

- **Hybrid search**: Pure dense retrieval (vector similarity) struggles with exact keyword matches, acronyms, and proper nouns. BM25 handles these well. Combining both with reciprocal rank fusion gives you the best of both worlds:

```python
def hybrid_search(query: str, k: int = 20) -> list[Document]:
    dense_results = vector_db.search(embed(query), top_k=k)
    sparse_results = bm25_index.search(query, top_k=k)
    return reciprocal_rank_fusion(dense_results, sparse_results, k=60)
```

- **Re-ranker**: After retrieval pulls ~100 candidates, a cross-encoder re-ranker (e.g., Cohere Rerank, a fine-tuned BERT) scores each (query, chunk) pair jointly. This is more expensive than bi-encoder retrieval but dramatically improves precision for the final top-5 chunks sent to the LLM.

- **Document-level auth**: Every chunk inherits the permissions of its source document. At query time, the retrieval layer filters results by the requesting user's access groups *before* re-ranking. Never send unauthorized content to the LLM — even if you don't show the answer, the model "saw" it.

- **Semantic cache**: Hash the query embedding, check if a semantically similar question was asked recently (cosine similarity > 0.95). If so, return the cached answer. This saves LLM costs on repetitive questions like "What's the PTO policy?"

### Tradeoffs

| Decision | Option A | Option B |
|---|---|---|
| Chunking strategy | Fixed-size (simple, fast) | Semantic (better quality, complex) |
| Embedding model | OpenAI ada-002 (easy, hosted) | Self-hosted e5-large (control, cost) |
| Vector DB | Managed (Pinecone) — less ops | Self-hosted (Qdrant) — more control |
| Re-ranker | Cross-encoder (accurate, slow) | Skip re-ranking (fast, less precise) |

The biggest tradeoff is **retrieval quality vs. latency**. Every stage you add (hybrid search, re-ranking, query expansion) improves answer quality but adds latency. For an internal Q&A tool, 2-3 seconds is acceptable. For a customer-facing chat, you need to be under 1 second for retrieval.

### Scale Considerations

At **10M documents**, your vector DB needs to handle billions of chunks. You'll need sharding and approximate nearest neighbor (ANN) indices like HNSW. Ingestion becomes a pipeline engineering problem — you need incremental updates, not full re-indexes.

At **1000 concurrent users**, the LLM becomes your bottleneck. You'll need request queuing, multiple LLM instances, and aggressive caching. The semantic cache becomes critical — in enterprise settings, 30-40% of questions are near-duplicates.

At **100x scale**, consider tiered retrieval: a cheap, fast first-pass filter (BM25 or lightweight embeddings) followed by expensive re-ranking only on the top candidates.

---

## 2. Real-Time Fraud Detection

Fraud detection is the canonical ML systems problem because it touches everything: feature engineering, real-time serving, class imbalance, feedback loops, and the tension between precision and recall (block a legitimate transaction and you lose a customer; miss fraud and you lose money).

### Architecture

**Training path:** Historical transactions → Feature engineering (Spark batch) → Feature store (offline) → Model training → Model registry → Deploy to model server.

**Serving path:** Incoming transaction → Feature enrichment (online store lookup + real-time computation) → Model server (Triton/TF Serving) → Score → Decision engine (approve/decline/review) → Response (<50ms total).

**Feedback path:** Fraud analyst labels → Label store → Periodic retraining trigger → Model registry → Shadow deploy → Champion/challenger evaluation → Promote.

A streaming layer (Kafka + Flink) continuously computes real-time features and writes to the online store.

### Key Components

- **Feature store (Feast)**: The dual-store pattern is essential. The **offline store** (data warehouse) holds historical feature values for training — you need point-in-time correct joins to avoid label leakage. The **online store** (Redis/DynamoDB) serves the latest feature values at inference time with single-digit ms latency.

```python
# Point-in-time correct feature retrieval for training
training_df = feast_store.get_historical_features(
    entity_df=transactions_with_timestamps,
    features=[
        "user_features:avg_spend_30d",
        "user_features:tx_count_7d",
        "merchant_features:risk_score",
    ],
).to_df()
```

- **Batch vs. streaming features**: Batch features (daily Spark jobs) capture slow-moving patterns — average monthly spend, merchant risk scores, account age. Streaming features (Flink) capture fast-moving signals — number of transactions in the last 5 minutes, geographic velocity (two transactions 1000 miles apart in 10 minutes). The interplay between these two feature types is where most of the model's power comes from.

- **Model choice**: XGBoost or LightGBM dominate here. They're fast at inference (sub-millisecond), handle tabular features well, and are interpretable enough for compliance. Deep learning models might squeeze out an extra 0.5% AUC but at 10x the latency. Some teams run an ensemble: a fast GBM for the hot path, a deeper model for the review queue.

- **Decision engine**: The model outputs a probability score, but the *business logic* decides the action. This is a separate component with configurable thresholds, rules, and override logic. Separating the model from the decision logic lets you tune thresholds without redeploying the model.

### Tradeoffs

Precision vs. recall is the existential tradeoff. A false positive (blocking a legit transaction) annoys a customer. A false negative (approving fraud) costs real money. Most systems optimize for high recall with a manual review queue for borderline cases — the "approve / decline / review" trichotomy.

Model freshness vs. stability: retraining too often risks instability; too rarely risks drift. Most teams retrain weekly to biweekly, with drift detection triggering ad-hoc retrains.

### Scale Considerations

At **10x transaction volume**, the streaming feature computation layer becomes the bottleneck. Flink/Spark Streaming jobs need careful partitioning by user ID to maintain state efficiently. The online feature store needs to handle 10x read throughput — this is where Redis cluster sharding earns its keep.

At **100x**, you start needing tiered scoring: a lightweight rule-based filter catches obvious fraud/legit transactions (80% of volume), and only borderline cases hit the ML model. This drastically reduces model serving costs.

---

## 3. Search Ranking System

Search ranking is a multi-stage funnel: you can't run an expensive model on millions of documents, so you progressively narrow the candidate set while increasing model complexity at each stage.

### Architecture

**Query flow:** User query → Query understanding (spell check, expansion, intent classification) → Stage 1: Candidate retrieval (ANN search, top-1000) → Stage 2: Re-ranking (learned-to-rank model, top-50) → Stage 3: Personalization & business rules (final top-10) → Results page.

**Offline pipeline:** Click logs + impressions → Feature extraction → Training data generation (with position bias correction) → Model training → A/B test deployment.

### Key Components

- **Stage 1 — Candidate retrieval**: Embedding-based ANN (approximate nearest neighbor) search using HNSW indices. The query and documents are encoded by a bi-encoder into the same vector space. This stage prioritizes *recall* — you want to make sure good results are in the candidate set, even if some bad ones sneak in. Latency budget: ~10ms for top-1000.

- **Stage 2 — Re-ranking**: A cross-encoder or LambdaMART model scores each (query, document) pair with richer features: BM25 score, click-through rate, document freshness, quality signals, query-document token overlap. LambdaMART (gradient-boosted trees optimized for ranking loss) is the industry workhorse. Fine-tuned BERT cross-encoders are more accurate but 10-100x slower.

```python
# LambdaMART feature vector per (query, doc) pair
features = {
    "bm25_score": 12.4,
    "embedding_similarity": 0.82,
    "doc_click_rate": 0.15,
    "doc_freshness_days": 3,
    "query_doc_token_overlap": 0.4,
    "doc_authority_score": 0.91,
}
```

- **Stage 3 — Personalization**: User-level features (search history, department, role) are combined with the re-ranked results. Business rules apply here too — boost promoted content, enforce diversity (don't show 10 results from the same source), apply recency bias for news-like content.

- **A/B testing with interleaving**: Traditional A/B testing (50/50 split) requires large sample sizes for ranking changes. Interleaving experiments merge results from two rankers into a single list and measure which ranker's results get clicked more. This is 10-100x more sensitive and requires fewer users.

### Tradeoffs

Model complexity at each stage is a latency-quality knob. The fanciest re-ranker in the world is useless if it adds 500ms. Most production systems use GBM-based rankers (LambdaMART) because they're fast and good enough. BERT-based re-rankers are reserved for the top-20 candidates at most.

Online vs. offline metrics often diverge. A model can improve NDCG on held-out data but hurt click-through rate in production because it doesn't account for position bias or presentation effects. Always validate with online experiments.

### Scale Considerations

At **10x corpus size**, ANN index build times grow, and you'll need distributed HNSW indices (e.g., sharded across multiple nodes). Index refresh frequency becomes a design decision — real-time indexing vs. periodic batch rebuilds.

At **100x query volume**, caching becomes essential. Popular queries (head queries) follow a power law — the top 1% of queries account for 30%+ of traffic. Cache the full ranking for these. For tail queries, cache embedding computations at minimum.

---

## 4. AI Customer Support Agent

This is the new frontier: LLM-powered agents that can actually *do things* — look up orders, process refunds, answer questions from a knowledge base — not just chat. The key challenge is reliability and safety. An agent that confidently gives wrong answers or processes unauthorized refunds is worse than no agent at all.

### Architecture

**Runtime flow:** User message → Conversation manager (maintains session state) → LangGraph agent (plans next action) → Tool selection → Tool execution (RAG / API call / escalation) → Response generation → Guardrail check → Stream response to user.

**Tool ecosystem:** Knowledge base (RAG pipeline), Order lookup API, Refund processing API, Escalation to human agent, FAQ cache.

**Monitoring:** Every agent turn is logged with the tool calls made, the context retrieved, and the response generated. A separate evaluation pipeline samples conversations and grades them.

### Key Components

- **LangGraph multi-step agent**: The agent operates as a state machine. Each node represents a step (retrieve context, call API, generate response), and edges represent transitions based on the LLM's tool-calling decisions. This is superior to a simple ReAct loop because you can enforce structure — e.g., always check the knowledge base before calling the refund API.

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("classify_intent", classify_intent)
graph.add_node("retrieve_knowledge", retrieve_from_kb)
graph.add_node("call_order_api", lookup_order)
graph.add_node("process_refund", handle_refund)
graph.add_node("generate_response", generate_with_context)
graph.add_node("escalate", escalate_to_human)

graph.add_conditional_edges("classify_intent", route_by_intent, {
    "question": "retrieve_knowledge",
    "order_status": "call_order_api",
    "refund": "process_refund",
    "complex": "escalate",
})
```

- **Conversation memory**: Short-term memory is the conversation history (last N turns). Long-term memory can include user preferences and past interactions, stored in a database and retrieved per session. The trick is keeping the context window manageable — summarize older turns rather than stuffing the full history.

- **Guardrails**: These are non-negotiable. Topic restriction (the agent should not discuss politics or competitors). PII handling (detect and redact SSNs, credit card numbers before they reach the LLM). Escalation triggers (if the user expresses frustration more than twice, escalate). Output validation (ensure the agent isn't hallucinating order numbers).

- **Human-in-the-loop**: Not every case can be automated. The system needs a seamless handoff path: when the agent's confidence is low or the case is sensitive (large refund, legal threat), it packages the conversation summary and routes it to a human agent's queue with full context.

### Tradeoffs

**Autonomy vs. safety**: The more actions you let the agent take autonomously (e.g., processing refunds), the more value it delivers — but the higher the risk. Most teams start with read-only actions (lookups, knowledge retrieval) and gradually expand to write actions with guardrails and spending limits.

**Streaming vs. batch response**: Token-by-token streaming improves perceived latency dramatically. But it makes guardrail checking harder — you can't validate the full response before the user sees it. One approach: stream the response but run async guardrail checks, and retract/edit if a violation is detected.

### Scale Considerations

At **10x concurrent conversations**, the LLM becomes the bottleneck. You'll need request queuing, load balancing across multiple LLM instances, and aggressive caching for common intents (FAQ-style questions). Consider using a smaller, faster model for simple intents and routing complex cases to a larger model.

At **100x**, you need a full conversation orchestration platform with session affinity, distributed state management, and multi-region deployment for latency.

---

## 5. LLM Gateway / Router

As organizations adopt LLMs, you quickly end up with a mess: different teams calling different providers with different API keys, no visibility into costs, no fallback when a provider goes down. The LLM Gateway is your centralized control plane.

### Architecture

**Request flow:** Client request → API Gateway (auth, rate limiting) → Router (complexity analysis, model selection) → Cache check (exact + semantic) → LLM provider call → Response processing (logging, cost tracking) → Cache write → Response to client.

**Control plane:** Admin dashboard for rate limits, budget alerts, model routing rules, usage analytics per team/user.

### Key Components

- **Model router**: Analyzes the incoming request and routes to the optimal model. Simple classification queries go to GPT-4-mini (cheap, fast). Complex reasoning or code generation goes to GPT-4 or Claude. The router can be rule-based (keyword matching, prompt length) or ML-based (a small classifier trained on prompt complexity).

```python
def route_request(prompt: str, metadata: dict) -> str:
    if metadata.get("force_model"):
        return metadata["force_model"]
    complexity = estimate_complexity(prompt)
    if complexity < 0.3:
        return "gpt-4o-mini"
    elif complexity < 0.7:
        return "gpt-4o"
    else:
        return "claude-3-5-sonnet"
```

- **Rate limiting**: Three tiers — per-user (prevent one user from hogging resources), per-team (enforce budgets), and global (prevent provider rate limit errors). Use token bucket or sliding window algorithms. Track both request count and token count, since a single long prompt can cost more than 100 short ones.

- **Fallback chain**: If the primary model returns a 5xx or times out, automatically retry with a fallback model. The chain might be: Claude → GPT-4 → GPT-4-mini → cached response → error. Each step has its own timeout. This gives you resilience without manual intervention.

- **Cost tracking**: Every request logs the model used, input/output token counts, and computed cost. Aggregate by team, project, and user. Set budget alerts at 80% and hard caps at 100%. This is how you prevent a runaway script from burning $50K overnight.

- **Semantic caching**: Beyond exact-match caching (same prompt = same response), embed the prompt and check for semantically similar recent queries. If "What's the capital of France?" was asked 5 minutes ago, "What is France's capital city?" should hit the cache. Use a similarity threshold (cosine > 0.95) to avoid false matches.

### Tradeoffs

**Centralization vs. autonomy**: A gateway gives you control and visibility but adds a hop of latency and becomes a single point of failure. Mitigate with multi-region deployment and circuit breakers.

**Smart routing vs. simplicity**: ML-based routing is elegant but adds complexity and its own failure modes. Many teams start with simple rules (prompt length, explicit model parameter) and only add ML routing when they have enough traffic data to train on.

### Scale Considerations

At **10x traffic**, caching becomes your best friend. A 30-40% cache hit rate means 30-40% fewer LLM calls. At **100x**, you need the gateway itself to be horizontally scalable — stateless request processing with shared cache (Redis) and shared rate limit state. Consider edge deployment to reduce latency for global teams.

---

## 6. Content Moderation System

Every platform that accepts user-generated content needs moderation. The challenge is doing it at scale, across modalities (text, images, video), with acceptable latency, and without drowning your human review team.

### Architecture

**Pipeline:** Content upload → Stage 1: Fast classifier (lightweight CNN/transformer, <50ms) → If clearly safe, approve. If clearly violating, block. → Stage 2: LLM analysis (nuanced cases, context-aware, ~500ms) → If confident, auto-action. → Stage 3: Human review queue (ambiguous cases, prioritized by severity and reach).

**Feedback loop:** Human decisions are logged and used to retrain Stage 1 classifiers weekly. False positive/negative reports from users feed into the review queue.

### Key Components

- **Multi-modal analysis**: Text moderation uses fine-tuned classifiers (BERT-based) for hate speech, spam, and toxicity. Image moderation uses vision models for NSFW content, violence, and banned symbols. Video moderation is the hardest — sample key frames and apply image classification, plus audio transcription for spoken content.

- **Tiered pipeline**: This is the core design insight. A single sophisticated model can't handle millions of posts per day at acceptable latency. Instead, the fast classifier handles 85-90% of content (obvious spam, clearly safe posts). The LLM handles 8-12% (nuanced sarcasm, borderline content, context-dependent cases). Humans handle 1-3% (genuinely ambiguous cases, appeals).

- **Prioritization queue**: Not all content awaiting human review is equal. A potentially violent post on a public page with millions of followers should be reviewed before a borderline joke in a private group chat. The queue scores items by: severity of potential violation × reach of the content × confidence gap of the automated classifiers.

- **Category taxonomy**: Categories aren't binary. You need a taxonomy with severity levels. "Violence" ranges from a boxing match (allowed) to graphic gore (blocked). The classifiers output multi-label predictions with confidence scores per category and severity level.

### Tradeoffs

**Precision vs. recall by category**: For child safety content, you want near-100% recall (catch everything) even at the cost of high false positives — over-moderation is acceptable. For satire/humor misclassified as hate speech, you want higher precision — over-moderation kills the user experience. Different categories get different thresholds.

**Pre-upload vs. post-upload screening**: Pre-upload screening adds latency to the upload flow but prevents violating content from ever being visible. Post-upload screening is non-blocking but creates a window where bad content is live. Most platforms do pre-upload for images/video and post-upload for text (because text classification is fast enough to run nearly synchronously).

### Scale Considerations

At **millions of posts/day**, the fast classifier must run on GPU-optimized infrastructure with batched inference. The LLM tier needs careful rate management to avoid costs exploding. At **10x scale**, you'll need content hashing (perceptual hashes for images, fingerprinting for videos) to instantly catch re-uploads of previously blocked content without running classifiers again.

---

## 7. ML Feature Platform

The feature platform is the infrastructure that makes feature engineering a first-class, reusable concern rather than ad-hoc code scattered across notebooks. If the model is the brain, the feature platform is the nervous system — it determines what signals reach the model and how fast.

### Architecture

**Computation layer:** Batch features (Spark jobs, daily/hourly) + Streaming features (Flink/Spark Streaming, real-time) → Materialization service.

**Storage layer:** Offline store (data warehouse — BigQuery, Snowflake, Hive) for historical features + Online store (Redis, DynamoDB) for serving-time lookups.

**Serving layer:** Feature server exposes a low-latency API. Given an entity ID (user, transaction, item), return the latest feature vector.

**Registry layer:** Feature definitions, metadata, ownership, documentation, lineage tracking.

### Key Components

- **Point-in-time correctness**: This is the single most important concept. When building training data, you must join features as they existed *at the time of the event*, not as they exist today. If you're training a fraud model on a transaction from January, you need the user's spending patterns as of January — not April. Getting this wrong causes feature leakage and silently inflates offline metrics.

```python
# WRONG: uses current feature values for historical events
features = feature_store.get_online_features(entity_ids)

# RIGHT: uses feature values as of each event's timestamp
features = feature_store.get_historical_features(
    entity_df=events_with_timestamps,  # each row has an event_timestamp
    features=["user:avg_spend_30d", "user:tx_count_7d"],
)
```

- **Offline-online consistency**: The same feature definition must produce identical values whether computed in batch (for training) or in real-time (for serving). This is one of the hardest problems. Feast and similar platforms solve it by using a single feature definition that gets materialized to both stores.

- **Feature registry**: A catalog of all features across the organization. Each feature has a name, description, owner, data type, computation logic, freshness SLA, and downstream consumers. This prevents duplication (three teams independently computing "user_avg_spend_30d") and enables discovery.

- **Materialization pipeline**: The process of computing features and writing them to the online store. Batch materialization runs on a schedule (e.g., daily at 2 AM). Streaming materialization writes to the online store in near-real-time. The platform must handle backfills (recompute historical features when logic changes).

### Tradeoffs

**Build vs. buy**: Feast is open-source and flexible but requires operational investment. Managed platforms (Tecton, Databricks Feature Store) reduce ops burden but add vendor dependency and cost. The choice depends on team size and ML maturity.

**Freshness vs. cost**: Streaming features (updated in seconds) are expensive to compute and maintain. Batch features (updated daily) are cheap and reliable. Most features don't need real-time freshness. Only compute in real-time what genuinely changes the model's decision at serving time.

### Scale Considerations

At **10x features**, the registry becomes critical — without it, you get feature sprawl. At **10x models**, the online store read throughput becomes the bottleneck. Redis cluster with read replicas handles this well. At **100x**, you need a federated architecture where different teams own their feature domains but share a common serving infrastructure.

---

## 8. Model Monitoring & Drift Detection

Deploying a model isn't the finish line — it's the starting line. Models degrade silently. The data distribution shifts, user behavior changes, upstream data pipelines break. Without monitoring, you won't know your model is broken until a business metric craters weeks later.

### Architecture

**Data flow:** Production predictions + inputs → Logging service → Metrics computation (batch, hourly/daily) → Statistical tests (comparison against reference window) → Alert engine → Dashboard + notification.

**Reference management:** A reference dataset (typically the validation set or a recent "known good" window) is stored and used as the baseline for comparison.

**Response pipeline:** Alert → Triage (automated severity classification) → Response action (retrain / rollback / human review).

### Key Components

- **Data drift**: The input feature distributions change. Measured via KL divergence (for continuous features), Population Stability Index (PSI), or Kolmogorov-Smirnov tests. Example: a fraud model trained on US transactions starts receiving 40% international transactions — the feature distributions are different even if the model logic is still valid.

- **Concept drift**: The relationship between inputs and outputs changes. The model's learned patterns no longer hold. Example: during COVID, consumer spending patterns shifted dramatically — features that predicted fraud pre-COVID became unreliable. This is harder to detect because you need ground truth labels, which arrive with a delay.

- **Prediction drift**: The model's output distribution shifts. If your fraud model suddenly flags 20% of transactions instead of the usual 2%, something is wrong — even if you don't yet know whether it's data drift, concept drift, or a pipeline bug.

```python
from scipy.stats import ks_2samp

def detect_drift(reference: np.ndarray, current: np.ndarray,
                 threshold: float = 0.05) -> bool:
    """KS test for distribution drift on a single feature."""
    statistic, p_value = ks_2samp(reference, current)
    return p_value < threshold  # True = drift detected
```

- **Alerting strategy**: Not every statistical blip is actionable. Use severity levels: INFO (minor distribution shift, log it), WARNING (moderate drift, trigger investigation), CRITICAL (severe drift or prediction distribution collapse, trigger automated rollback). Windowed comparisons (last 24h vs. last 30d) smooth out noise.

- **Dashboard**: Feature-level distribution plots (histograms, quantile comparisons), model performance metrics over time (accuracy, precision, recall, AUC), prediction volume and latency, and a drift score heatmap across all features.

### Tradeoffs

**Sensitivity vs. noise**: Aggressive drift detection catches real problems early but generates false alarms that erode trust. Conservative detection is quiet but risks missing gradual drift until it's severe. Most teams start conservative and tighten thresholds as they learn their system's normal variation.

**Automated response vs. human review**: Automated retraining on drift detection sounds appealing but can be dangerous — if the drift is caused by a data pipeline bug, retraining on bad data makes things worse. A safer pattern: automated *detection* and *alerting*, but human-approved *response* (retrain, rollback, or investigate).

### Scale Considerations

At **10x models**, you need a centralized monitoring platform — per-model custom dashboards don't scale. Standardize on a common metrics schema and alerting framework. At **100x features per model**, computing drift statistics for every feature becomes expensive. Use feature importance to prioritize — monitor the top-20 most important features intensively, the rest on a sampling basis.

---

## 9. Multi-Agent Document Processing

When documents require complex processing — classification, extraction, validation, routing — a single monolithic model struggles. Multi-agent architectures decompose the problem: each agent specializes in one task, and an orchestrator coordinates the workflow.

This pattern shows up in insurance claims processing, legal document review, invoice handling, and healthcare intake forms.

### Architecture

**Pipeline flow:** Document upload → OCR/parsing (if needed) → Agent 1: Classification (type, urgency, department) → Agent 2: Information extraction (key fields, entities, amounts) → Agent 3: Validation & cross-referencing (check extracted data against business rules and external systems) → Agent 4: Routing & action determination (approve, reject, escalate, request more info) → Output action + human review queue for low-confidence items.

**Orchestration:** A LangGraph supervisor agent manages the pipeline. It decides which agents to invoke, handles retries, manages state between agents, and routes to human review when confidence is low.

### Key Components

- **Document classification agent**: Determines document type (invoice, contract, claim, correspondence) and metadata (urgency, department, language). Uses a fine-tuned classifier or an LLM with few-shot prompting. Classification accuracy here is critical — a misclassified document enters the wrong downstream pipeline.

- **Information extraction agent**: Extracts structured fields from unstructured text. For invoices: vendor name, amount, line items, due date. For contracts: parties, effective date, term, key clauses. This agent typically uses an LLM with a structured output schema (JSON mode or function calling) and validates against expected field types.

```python
extraction_prompt = """
Extract the following fields from this invoice:
- vendor_name: str
- invoice_number: str
- total_amount: float
- due_date: str (YYYY-MM-DD)
- line_items: list[{description: str, quantity: int, unit_price: float}]

Return as JSON. If a field is not found, use null.
"""
```

- **Validation agent**: Cross-references extracted data against business rules and external systems. Does the vendor exist in the vendor database? Is the amount within the expected range for this vendor? Does the invoice number match any existing records (duplicate check)? This agent catches extraction errors and fraud.

- **Routing agent**: Based on classification, extracted data, and validation results, determines the next action: auto-approve (high confidence, within policy), escalate to manager (above threshold amount), reject (failed validation), or request additional information from the submitter.

- **Supervisor orchestrator**: The LangGraph supervisor pattern coordinates the agents. It maintains a shared state object that accumulates results from each agent. If an agent fails or returns low confidence, the supervisor can retry with different parameters, invoke a fallback agent, or route to human review.

```python
class DocumentState(TypedDict):
    raw_text: str
    doc_type: str | None
    confidence: float
    extracted_fields: dict
    validation_results: list[ValidationResult]
    action: str | None
    needs_human_review: bool
```

### Tradeoffs

**Agents vs. single pipeline**: A single LLM call with a mega-prompt ("classify this, extract these fields, validate, and route") is simpler but fragile — one task's failure corrupts the whole output. Multi-agent is more robust and debuggable but adds latency (multiple LLM calls) and complexity (state management, error handling between agents).

**LLM vs. specialized models**: Each agent can be an LLM (flexible, handles novel document types) or a specialized fine-tuned model (faster, cheaper, more accurate on known types). The pragmatic approach: use specialized models for high-volume, well-defined document types, and LLMs for the long tail of rare or novel documents.

**Confidence thresholds**: Setting the human review threshold is a business decision. Too low and humans review everything (defeats the purpose). Too high and errors slip through. Start conservative (route 30-40% to humans), measure error rates, and tighten as the system proves itself.

### Scale Considerations

At **10x document volume**, parallel processing becomes essential. Documents are independent, so you can process them concurrently. The bottleneck shifts to the LLM provider's rate limits — batch requests and use multiple provider endpoints.

At **100x**, you need a job queue (Celery, Temporal) with priority levels. Urgent documents (insurance claims, time-sensitive contracts) get processed first. Low-priority documents (archival, bulk processing) get batched during off-peak hours. You'll also want to invest in fine-tuned specialized models for your top-5 document types to reduce LLM costs and latency.

---

## Wrapping Up: Cross-Cutting Patterns

After studying all nine problems, several patterns emerge:

1. **Tiered processing**: Almost every system uses a cheap-fast tier for easy cases and an expensive-slow tier for hard cases. Fraud detection, content moderation, search ranking, and document processing all follow this pattern.

2. **Feature stores are everywhere**: Fraud detection and search ranking both rely on the same offline/online feature store pattern. If you're building multiple ML systems, invest in the feature platform early.

3. **Human-in-the-loop is not optional**: Every production AI system needs an escape hatch to humans. The question is where to draw the line — and that line should start conservative and move as you build trust.

4. **Monitoring is a first-class concern**: Drift detection isn't a nice-to-have. Models degrade silently. Build monitoring from day one, not as an afterthought.

5. **Caching is an architectural pattern, not an optimization**: Semantic caching in RAG and the LLM Gateway, result caching in search ranking, content hash caching in moderation — caching appears at every layer and often provides the biggest bang for your infrastructure buck.

These nine problems give you the vocabulary and structural intuition to tackle most AI system design challenges. The specific technologies will change — the architectural patterns won't.
