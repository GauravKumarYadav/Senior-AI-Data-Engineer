# The Interview Gauntlet 🔥

This is where the rubber meets the road. Modules 1 and 2 gave you the blueprints and the vocabulary. This module tests whether you've internalized them. Treat each question like an interviewer just asked it — pause, structure your thoughts, and walk through your answer before reading the solution.

No peeking. Seriously.

---

## Screen 1: Design an AI System

Full system design problems. For each one, read the question, spend 5 minutes sketching your own answer, then compare against the solution walkthrough.

---

### Problem 1: "Design a RAG system for a company with 10M internal documents"

**Solution Walkthrough:**

Clarifying questions first: what document types (PDFs, wikis, Slack)? Are there access controls? What's the latency tolerance — internal tool (2-3s fine) or customer-facing (sub-second)?

The architecture splits into two pipelines. The **ingestion pipeline** pulls from connectors (Confluence API, SharePoint, S3), parses documents, chunks using semantic chunking that respects structure (headers, paragraphs), embeds via `e5-large-v2`, and writes to a vector database (Qdrant, self-hosted for data sovereignty). A metadata store tracks permissions, source URLs, and freshness.

The **query pipeline** runs auth check → query embedding → hybrid retrieval (dense vectors + BM25 via reciprocal rank fusion) → cross-encoder re-ranking (top 100 → top 5) → LLM generation with context → response with citations.

Key components worth calling out: the chunking strategy matters enormously at 10M documents — semantic chunking with ~512 token chunks and 50-token overlap gives the best retrieval quality. Document-level auth filtering happens *before* re-ranking so the LLM never sees unauthorized content. A semantic cache (cosine similarity > 0.95) catches the 30-40% of questions that are near-duplicates in enterprise settings, saving significant LLM costs.

Tradeoffs I'd discuss: self-hosted embedding model (control, cost) vs API-based (simpler ops). Cross-encoder re-ranking adds ~200ms but dramatically improves precision. Full re-indexing vs incremental updates — at 10M docs, incremental is mandatory with periodic full rebuilds.

At scale, the vector DB needs sharding with HNSW indices across multiple nodes. The LLM is the throughput bottleneck — request queuing, multiple instances, and aggressive caching are essential. I'd also add a feedback loop: thumbs-up/down signals feed into retrieval quality dashboards and periodic embedding model fine-tuning.

---

### Problem 2: "Design a real-time fraud detection pipeline processing 50K transactions/second"

**Solution Walkthrough:**

Clarifications: what's the latency budget (most processors require <100ms)? What's the fraud rate? What's the cost asymmetry between false negatives and false positives?

Three architectural paths. **Training:** historical transactions join with the feature store (point-in-time correct) → train XGBoost/LightGBM → register in MLflow → shadow deploy before promotion. **Serving:** transaction → Kafka → feature enrichment (online store for batch features + Flink for streaming features like `tx_count_last_5min`) → model server (Triton) → score → decision engine → approve/decline/review. **Feedback:** analyst labels → label store → periodic retraining.

At 50K TPS, the streaming layer is the most challenging component. Flink jobs partitioned by user ID maintain per-user state (running counts, velocity calculations) in RocksDB state backends. The online feature store (Redis Cluster) must handle 50K reads/second with sub-millisecond latency — sharding by user ID with read replicas.

The model choice is deliberately conservative: XGBoost gives sub-millisecond inference, handles tabular features well, and is interpretable enough for compliance audits. A deeper model might add 0.3% AUC but cost 10x the latency.

The decision engine is separate from the model — it applies configurable thresholds and business rules (never auto-decline transactions over $10K without human review). This separation lets you tune thresholds without redeploying the model.

Critical tradeoff: precision vs recall. I'd implement a three-way split — auto-approve (score < 0.1), auto-decline (score > 0.95), manual review (everything between). The review queue is prioritized by transaction amount × fraud probability. At 50K TPS, even a 1% review rate means 500 transactions/second needing human eyes, so the auto-decline threshold must be carefully tuned.

For scale: add tiered scoring — a lightweight rule-based filter catches obvious cases (80% of volume) so only 20% hit the ML model, reducing serving costs 5x.

---

### Problem 3: "Design a search ranking system for an e-commerce platform"

**Solution Walkthrough:**

Clarifications: catalog size? Query volume? Optimizing for clicks, add-to-cart, or revenue?

Multi-stage funnel architecture. **Stage 1 — Candidate Retrieval (~10ms):** Bi-encoder embeds the query, HNSW retrieves top 1,000, BM25 runs in parallel, results merge via reciprocal rank fusion. Optimizes for recall. **Stage 2 — Re-ranking (~20ms):** LambdaMART scores each (query, candidate) pair with rich features: BM25 score, embedding similarity, CTR, conversion rate, freshness, seller rating. Narrows to top 50. **Stage 3 — Personalization & Business Rules (~5ms):** User features (purchase/browse history, price sensitivity) adjust scores. Business rules enforce diversity, boost promoted products, filter availability. Final top 10-20.

The offline pipeline processes click logs with position bias correction, generates training data, and retrains LambdaMART. A/B testing uses interleaving — 10-100x more sensitive than traditional splits.

Key tradeoff: NDCG improvements offline don't always translate to conversion gains online. Always validate with live experiments. BERT cross-encoders beat LambdaMART on accuracy but cost 100x the latency — reserve for re-ranking top 20 at most. At scale, head query caching is essential (top 1% of queries = 30%+ of traffic).

---

### Problem 4: "Design an AI customer support chatbot that handles 80% of tickets"

**Solution Walkthrough:**

The 80% target is the key constraint. This means the system needs to be good enough to resolve most queries autonomously, but smart enough to know when to escalate the remaining 20%.

Architecture: User message → Conversation manager → Intent classifier → LangGraph state machine routing to specialized nodes: `retrieve_knowledge` (RAG over help docs), `call_order_api`, `process_refund`, `generate_response`, `escalate_to_human`. Conditional edges route by intent.

To hit 80%: the knowledge base must be comprehensive and fresh (weekly sync from help center and policy docs). The agent needs tool-calling — read-only actions (order lookup) launch immediately; write actions (refunds) have guardrails (spending limits, confirmation). Confidence-based escalation kicks in when retrieved context has low relevance or the user expresses frustration twice.

Guardrails are non-negotiable: PII redaction, topic restrictions, output validation, and a hard "talk to a human" escape hatch. Every conversation is logged. Human agents label escalated responses as correct/incorrect, feeding weekly retraining. Track automation rate and CSAT per channel as primary metrics.

Tradeoff: start with read-only actions, expand to write actions as confidence grows. Streaming improves perceived latency but complicates guardrail checking — run async checks and retract if needed.

---

### Problem 5: "Design an LLM gateway that serves 500 internal teams"

**Solution Walkthrough:**

Without a gateway, 500 teams means 500 API keys, zero cost tracking, and no fallback when providers go down.

Architecture: Client request → API Gateway (auth, rate limiting) → Router (complexity analysis, model selection) → Cache check (exact + semantic) → LLM provider call → Response logging (model, tokens, cost) → Cache write → Response.

The router sends simple queries to GPT-4o-mini, complex reasoning to GPT-4o or Claude. Start rule-based (prompt length, explicit model parameter), evolve to ML-based as traffic data grows.

Rate limiting at three levels: per-user, per-team (budgets with alerts at 80%, hard caps at 100%), and global. Track both request count and token count. Fallback chain: primary model → secondary → cached response → error, with circuit breakers preventing cascading failures.

Cost tracking aggregates by team, project, and user — preventing runaway scripts from burning $50K overnight. Semantic caching (cosine > 0.95) provides 30-40% hit rates in enterprise settings, saving hundreds of thousands monthly at this scale.

Tradeoff: centralization adds latency and a single point of failure. Mitigate with multi-region deployment and circuit breakers. The visibility benefits far outweigh the complexity.

---

### Problem 6: "Design a content moderation system for a social media platform"

**Solution Walkthrough:**

Clarifications: what modalities? What volume? Can moderation be async (post-upload) or must it block publishing?

Tiered pipeline is the core insight. **Stage 1 — Fast classifier (<50ms):** Lightweight models (distilled BERT, MobileNet) handle 85-90%. Safe → approve. Violating → block. **Stage 2 — LLM (~500ms):** Nuanced cases — sarcasm, context-dependent content. 8-12%. **Stage 3 — Human review:** Ambiguous cases, 1-3%. Queue prioritized by severity × reach × confidence gap.

Multi-modal: text classifiers for hate speech/spam/self-harm, vision models for NSFW/violence, video via keyframe sampling + audio transcription. Content hashing (perceptual hashes) catches re-uploads in O(1).

Category taxonomy needs severity levels, not binary labels. Different categories get different thresholds — child safety demands near-100% recall; humor/satire detection prioritizes precision. Human decisions retrain Stage 1 weekly. User appeals enter the review queue.

---

### Problem 7: "Design an ML feature platform serving 200 ML models"

**Solution Walkthrough:**

At 200 models, ad-hoc feature computation is chaos. The feature platform centralizes everything.

Four layers. **Computation:** Batch (Spark, daily/hourly) + streaming (Flink, real-time). **Storage:** Offline store (BigQuery/Snowflake) for training; online store (Redis Cluster) for sub-millisecond serving. **Serving:** Feature server API returns feature vectors by entity ID. **Registry:** Feature catalog with owners, definitions, freshness SLAs, and lineage.

Point-in-time correctness is the most critical requirement — training data must use feature values as they existed at event time, not current values. Getting this wrong causes feature leakage. Offline-online consistency is the hardest problem — the same definition must produce identical values in batch and real-time.

At 200 models, the registry prevents feature sprawl (15 teams independently computing `user_avg_spend_30d`). Materialization pipelines handle backfills and incremental updates.

Tradeoff: Feast (open-source, operational investment) vs Tecton (managed, vendor dependency). Freshness vs cost: only compute in real-time what genuinely changes the decision at serving time. Scale: Redis Cluster with read replicas, federated team ownership over shared infrastructure.

---

## Screen 2: Component Deep Dives

Technical questions about specific AI/ML infrastructure components.

---

### "Explain how a feature store achieves point-in-time correctness"

Point-in-time correctness ensures that when building training data, each example uses feature values as they existed at the moment of that event — not current values or future values. The feature store achieves this by maintaining timestamped feature values in the offline store. When you request historical features, you pass an entity DataFrame where each row has an entity ID and an event timestamp. The store performs an asof-join: for each row, it finds the most recent feature value with a timestamp *before* the event timestamp. This prevents feature leakage — for example, if training a fraud model on a January transaction, you get the user's spending patterns as of January, not April. Without this, features computed from future data leak information about the label, causing inflated offline metrics that don't transfer to production. Feast implements this via `get_historical_features()`, which accepts an entity DataFrame with timestamps and performs the temporal join against the offline store. The key implementation detail is that feature values must be stored with their computation timestamps, and the join logic must handle late-arriving data and out-of-order events correctly.

---

### "How would you design a model registry for a team of 50 ML engineers?"

The registry needs four core capabilities: versioning, metadata, lineage, and promotion workflows. Every model gets an immutable version with its artifact (serialized weights), hyperparameters, training data version, evaluation metrics, and the git commit of the training code. Lineage traces from a production prediction back through the model version, training run, dataset version, and feature definitions. Promotion follows a stage gate: `Development → Staging → Production → Archived`. Promotion to staging requires passing automated tests (inference latency, output schema validation, metric thresholds). Promotion to production requires a shadow deployment showing no regression, plus peer approval. For 50 engineers, governance matters — ownership per model, audit logs for who promoted what and when, and automated alerts when a model in staging hasn't been promoted or archived in 30 days. I'd use MLflow as the foundation with custom CI/CD hooks: a PR merging a model promotion triggers automated validation, shadow deployment, and a Slack notification for approval. Access controls prevent anyone from directly pushing to production without the pipeline.

---

### "Describe the architecture of a semantic cache for an LLM application"

A semantic cache stores query-response pairs indexed by the query's embedding vector rather than its exact text. When a new query arrives, it's embedded using the same model, and a vector similarity search runs against the cache index. If the nearest neighbor exceeds a similarity threshold (typically cosine > 0.95), the cached response is returned without calling the LLM. The architecture has three components: an embedding service (lightweight, ~5ms per query), a vector index (HNSW in Redis or a dedicated vector store, ~5-10ms search), and a response store (Redis or similar, keyed by cache entry ID). TTL-based expiration prevents stale responses. The similarity threshold is the critical tuning knob — too low (0.85) produces false matches ("How do I return an item?" matching "How do I return a library book?"); too high (0.99) rarely hits. Version tagging invalidates cache entries when underlying data changes. The total overhead is ~10-50ms per query, but cache hits save 500-2000ms of LLM inference. At enterprise scale with repetitive queries, 30-40% hit rates translate to massive cost savings.

---

### "How does HNSW indexing work and when would you choose it over IVF-PQ?"

HNSW (Hierarchical Navigable Small World) builds a multi-layer graph where each node is a vector. The bottom layer contains all vectors; upper layers contain progressively fewer, acting as "express lanes." Search starts at the top layer, greedily navigating to the nearest node, then drops down a layer and repeats until reaching the bottom layer where it performs a local graph search. Construction parameters `M` (connections per node) and `efConstruction` (search width during building) control the accuracy-memory tradeoff. HNSW excels when the dataset fits in RAM — it delivers >95% recall with microsecond query times. Choose HNSW for latency-critical applications under ~100M vectors. IVF-PQ (Inverted File Index with Product Quantization) partitions vectors into clusters (IVF) and compresses each vector via product quantization, reducing memory 10-50x. It's disk-friendly and handles billion-scale datasets, but recall is lower (85-93%) and query latency is higher. Choose IVF-PQ when your dataset exceeds available RAM or cost constraints demand compression. In practice, many systems use HNSW for the hot tier and IVF-PQ for the cold tier.

---

### "Design a model monitoring system that detects concept drift"

Concept drift means the relationship between inputs and outputs has changed — the model's learned patterns no longer hold. Detection requires comparing model performance over time against a reference baseline. The system has three layers. **Data collection:** log every prediction with its input features, model output, and — when available — ground truth labels. **Metrics computation:** on a scheduled basis (hourly/daily), compute statistical tests comparing the current window against a reference window. For data drift, use PSI or KS tests on input feature distributions. For prediction drift, compare output score distributions. For concept drift specifically, you need delayed ground truth — compute performance metrics (accuracy, AUC, precision, recall) on the labeled subset and track degradation over time. **Alerting:** three severity levels — INFO for minor shifts, WARNING for moderate drift triggering investigation, CRITICAL for severe degradation triggering automated rollback. The hardest part is that ground truth labels arrive with a delay (fraud labels take weeks), so you must rely on proxy signals — prediction distribution shifts, feature drift on high-importance features, and business metric anomalies — as early warning signals while waiting for labels.

---

### "How would you build an embedding refresh pipeline?"

When you update your embedding model, all existing vectors become incompatible with new query embeddings — you must re-embed the entire corpus. The pipeline has four phases. **Trigger:** a new embedding model is registered in the model registry (scheduled quarterly or triggered by quality regression). **Batch re-embedding:** a distributed Spark/Ray job reads all documents from the corpus store, chunks them (reusing existing chunk boundaries unless the chunking strategy also changed), and generates new embeddings. For 10M documents with ~50M chunks, this takes hours on GPU clusters. **Atomic swap:** write new embeddings to a fresh vector index (not the live one). Run validation — compare retrieval quality on a golden test set between old and new indices. If quality improves, swap the alias so the query pipeline now points to the new index. **Cleanup:** keep the old index for 48 hours as a rollback target, then decommission. Incremental strategies (only re-embed new/modified documents) save compute but create a mixed-version index where old and new embeddings coexist — this degrades search quality because the vector spaces aren't aligned. Full re-embedding is cleaner and worth the cost at reasonable corpus sizes.

---

### "Explain the architecture of guardrails for an LLM-powered application"

Guardrails operate as a two-stage filter — input validation before the LLM sees the prompt, and output validation before the user sees the response. **Input guardrails** include: prompt injection detection (a fine-tuned classifier or pattern matching catches "ignore previous instructions" attacks), topic boundary enforcement (an intent classifier blocks off-topic queries — a customer service bot shouldn't discuss politics), PII detection in user input (redact before it reaches the LLM), and rate limiting per user. **Output guardrails** include: PII scanning of the response (SSNs, credit cards, emails), factuality grounding checks for RAG (verify the response cites retrieved documents, not hallucinated content), toxicity classification, and format validation (if you asked for JSON, is it valid JSON?). The architecture is a pipeline: `User Input → Input Guardrails → LLM → Output Guardrails → User Response`. Each guardrail runs independently and returns pass/block/redact. Blocked requests return a safe fallback message. The system logs all guardrail triggers for auditing. Frameworks like NeMo Guardrails (declarative dialogue flow control) and Guardrails AI (schema-based output validation) provide building blocks, but most production systems customize heavily for their domain.

---

## Screen 3: Trade-offs & Decisions

For each scenario, both options have merit. The right answer depends on context — and saying so is the point.

---

### "Real-time inference vs batch prediction for a recommendation system?"

It depends on how dynamic your recommendations need to be. **Batch prediction** precomputes recommendations for all users periodically (hourly/daily), stores them in a fast lookup table, and serves them with sub-millisecond latency. It's simpler, cheaper, and perfect when recommendations don't need to reflect the user's last 5 minutes of behavior. Most e-commerce homepages use this. **Real-time inference** computes recommendations on-the-fly using the user's latest activity. It captures in-session behavior ("you just looked at running shoes, here are related items") but requires a model server, online feature store, and tighter latency budgets. The pragmatic answer: use batch for the homepage and category pages (where freshness doesn't matter much), and real-time for in-session contexts like "similar items" or "you may also like" on product detail pages. Many production systems use a hybrid — batch-generated candidate sets with real-time re-ranking.

---

### "Fine-tuning vs RAG for a domain-specific Q&A system?"

**RAG** retrieves relevant documents at query time and passes them as context. It's better when: the knowledge base changes frequently (RAG reflects updates immediately without retraining), you need citations and source attribution, and you want to stay on a general-purpose model without the cost of fine-tuning. **Fine-tuning** bakes domain knowledge into model weights. It's better when: you need the model to adopt a specific tone, style, or reasoning pattern, the domain has specialized terminology the base model handles poorly, and latency matters (no retrieval step). In practice, they're complementary, not competing. Fine-tune for style and domain fluency; use RAG for factual grounding. A fine-tuned model that also uses RAG gives you the best of both — domain-native communication style with up-to-date, citable facts.

---

### "Single large model vs ensemble of specialized models for fraud detection?"

A **single large model** is simpler to operate — one training pipeline, one serving endpoint, one monitoring dashboard. It sees all features and can learn cross-pattern interactions. But it becomes a monolith that's hard to iterate on. An **ensemble of specialists** (one model for card-not-present fraud, one for account takeover, one for merchant collusion) lets teams iterate independently and tune precision/recall per fraud type. The tradeoff is operational complexity — more pipelines, more monitoring, and you need a meta-model or rules to combine scores. For most teams, start with a single model. Graduate to specialists when specific fraud types need different feature sets, retraining cadences, or threshold tuning. The decision engine downstream can always apply fraud-type-specific rules regardless of whether the model is monolithic or ensemble.

---

### "Pre-filter vs post-filter for metadata filtering in vector search?"

**Pre-filter** applies the metadata filter first, then runs ANN search on the filtered subset. It's precise — you only search relevant documents. But if the filter is very selective (e.g., "documents from last week" in a 10M corpus), the candidate set may be too small for ANN to work well, and index partitioning gets complex. **Post-filter** runs ANN on the full index, then discards results that don't match the metadata filter. It's simpler and leverages the full index, but wastes work — if 90% of results are filtered out, you're returning far fewer results than the user requested. The practical choice: use pre-filter when the filtered subset is large (>10% of corpus) and you have partition-aligned indices. Use post-filter when filters are loose or variable. Some vector databases (Weaviate, Qdrant) support hybrid approaches that combine both strategies adaptively.

---

### "Synchronous vs asynchronous model serving?"

**Synchronous** serving means the client blocks until the prediction returns. It's simple, predictable, and necessary when the response is part of a user-facing flow (search ranking, fraud scoring at checkout). **Asynchronous** serving accepts the request, returns immediately with a job ID, and delivers results later via callback or polling. It's better for batch-like workloads (document processing, bulk scoring) and when inference is slow (LLM generation, video analysis). The key question: does the caller need the result *right now* to proceed? If yes, synchronous. If the result can be consumed later, async decouples the caller from model latency, improves throughput, and enables request batching for GPU efficiency. Many systems use both — sync for the hot path, async for background enrichment.

---

### "Feature store vs ad-hoc feature computation?"

**Ad-hoc computation** (each model team writes its own feature pipelines) is fine for one or two models. Beyond that, you get duplicated work, inconsistent definitions, and training-serving skew. A **feature store** centralizes feature definitions, ensures offline-online consistency, and provides point-in-time correct joins for training. The cost is operational complexity — you're running shared infrastructure that every ML team depends on. The tipping point is around 5-10 models or 3+ teams. Below that, the overhead isn't justified. Above that, the consistency and reuse benefits compound rapidly. At 200 models, not having a feature store is organizational malpractice.

---

### "Shadow deployment vs canary deployment for a new ML model?"

**Shadow deployment** runs the new model on production traffic but doesn't serve its predictions to users — you compare outputs offline. It's zero-risk to users but you can't measure user-facing metrics (click-through, conversion). **Canary deployment** serves the new model to a small percentage of real users (1-5%) and measures actual impact. It carries real risk but gives you real signal. Use shadow first to catch catastrophic failures (crashes, latency spikes, wildly different output distributions), then canary to measure business impact. They're sequential, not competing — shadow is your safety net, canary is your experiment.

---

### "Exact match cache vs semantic cache for an LLM gateway?"

**Exact match** (hash the prompt, look up in Redis) is simple, fast (<1ms), and has zero false matches. But "What's the PTO policy?" and "How much PTO do I get?" are cache misses. **Semantic cache** (embed the prompt, vector search against cached queries) catches paraphrases but adds 10-50ms of overhead and risks false matches if the similarity threshold is too low. Use exact match as the first layer (it's free and fast), semantic cache as the second layer. The threshold is critical — 0.95 cosine similarity is a good starting point. In enterprise settings where 500 teams ask similar questions, semantic caching provides 30-40% hit rates that translate to massive cost savings.

---

## Screen 4: Rapid Fire

Short questions, short answers. If you can't answer these in one breath, review Modules 1 and 2.

---

**What is HNSW?**
Hierarchical Navigable Small World — a graph-based approximate nearest neighbor index that builds multi-layer skip-list-like graphs for fast vector search. High recall, high memory usage, best when the dataset fits in RAM.

**What is PSI (Population Stability Index)?**
A metric that measures how much a variable's distribution has shifted between two time periods. Commonly used in model monitoring to detect data drift. PSI < 0.1 means stable, 0.1-0.25 means moderate shift, > 0.25 means significant drift.

**Difference between data drift and concept drift?**
Data drift is when input feature distributions change. Concept drift is when the relationship between inputs and outputs changes. Data drift doesn't necessarily break the model; concept drift always does.

**What is a cold start problem?**
When a system has insufficient data to make good predictions — for new users (no interaction history), new items (no engagement data), or new systems (no data at all). Solved with content-based features, popularity defaults, or onboarding flows.

**What is interleaving in A/B testing?**
A technique for ranking experiments where results from two models are merged into a single list shown to users. Users implicitly vote by clicking. It's 10-100x more sensitive than traditional split A/B tests, requiring far fewer samples.

**What is a data flywheel?**
A compounding loop: more users → more data → better models → better product → more users. The strongest competitive moat in AI. Examples: Google Search, Netflix recommendations, Tesla autopilot.

**What is point-in-time correctness?**
Ensuring training data uses feature values as they existed at the time of each event, not current values. Prevents feature leakage. Implemented via temporal asof-joins in feature stores.

**What is product quantization?**
A vector compression technique that splits high-dimensional vectors into subvectors and quantizes each independently using a learned codebook. Reduces memory 10-50x with moderate recall loss. Used in IVF-PQ indices.

**What does a model registry store?**
Model artifacts, hyperparameters, training data version, evaluation metrics, lineage (code commit, dataset, feature versions), deployment stage (dev/staging/production), and ownership metadata.

**What is a feature store's online store?**
A low-latency key-value store (Redis, DynamoDB) holding the latest feature values for real-time model inference. Sub-millisecond reads. Updated by materialization pipelines from batch or streaming computation.

**What is Thompson Sampling?**
A Bayesian bandit algorithm that maintains probability distributions over each option's reward rate and samples from them to decide which option to try. Balances exploration and exploitation naturally. Used in A/B testing and recommendation exploration.

**What is a guardrail metric?**
A metric that must not degrade during an experiment, even if the primary metric improves. Examples: latency p99, crash rate, content safety violations. If a guardrail trips, the experiment stops regardless of primary metric gains.

**What is shadow mode deployment?**
Running a new model on production traffic without serving its predictions to users. Predictions are logged and compared against the production model offline. Zero user risk, used to validate before canary deployment.

**What is semantic caching?**
Caching LLM responses indexed by query embedding rather than exact text. New queries are embedded and compared via vector similarity — if a similar query was recently answered (cosine > 0.95), the cached response is returned. Catches paraphrases that exact-match caching misses.

**What is BM25?**
A sparse retrieval algorithm based on term frequency and inverse document frequency. Excels at exact keyword matching, acronyms, and proper nouns. Often combined with dense vector search in hybrid retrieval.

**What is NDCG?**
Normalized Discounted Cumulative Gain — a ranking metric that measures how well a ranked list places relevant results near the top. Accounts for position (higher-ranked results matter more) and graded relevance. Standard metric for search quality.

**What is a cross-encoder?**
A model that takes a (query, document) pair as joint input and scores their relevance together, unlike bi-encoders which encode them independently. More accurate but O(n) — can't be used for retrieval, only for re-ranking small candidate sets.

**What is model lineage?**
The full provenance chain from a production prediction back to the model version, training run, dataset version, feature definitions, and code commit. Essential for debugging, compliance, and reproducibility.

**What is feature leakage?**
When training data accidentally includes information from the future or from the label itself. Causes inflated offline metrics that don't transfer to production. Common cause: using current feature values instead of point-in-time correct values.

**What is the cascading pattern in LLM routing?**
Try the cheapest/fastest model first. If response confidence is low (measured by log-probabilities or a quality classifier), escalate to a more expensive model. Reduces costs 50-70% because most queries are simple enough for a small model.

---

## Screen 5: Key Takeaways

Everything you've learned across all three modules, distilled.

---

### The 6-Step ML System Design Framework

Every problem follows this structure. State it at the start of your interview answer to give the interviewer a roadmap.

1. **Clarify Requirements** — Ask about business objective, users, latency, scale, compliance. Spend 3-5 minutes here. It shows maturity.
2. **Data** — Sources, labeling strategy, volume, schema, privacy constraints. Batch vs streaming ingestion.
3. **Model** — Right model for the job. Accuracy vs latency vs cost tradeoff triangle. Offline evaluation metrics.
4. **Serving** — Real-time, batch, or streaming inference. Latency SLAs, autoscaling, hardware decisions.
5. **Monitoring** — Data drift, concept drift, prediction drift, business metrics. Alerting strategy with severity levels.
6. **Iteration** — Feedback loops, retraining cadence, A/B testing, the data flywheel. This connects back to Steps 2 and 3.

---

### Top 5 Patterns That Appear Across All Problems

1. **Tiered processing.** Cheap-fast tier for easy cases, expensive-slow tier for hard cases. Fraud detection, content moderation, search ranking, LLM routing — they all use this. It's the single most common architectural pattern in production ML.

2. **Offline-online duality.** Feature stores have offline and online stores. Evaluation has offline metrics and online experiments. Models have training and serving. Every system lives in two worlds — the batch world where you learn, and the real-time world where you serve.

3. **Human-in-the-loop is a feature, not a failure.** Every production AI system needs an escape hatch to humans. The system should know what it doesn't know. Start conservative, expand autonomy as trust builds.

4. **Caching is an architectural pattern, not an optimization.** Semantic caching in RAG, result caching in search, content hash caching in moderation, exact + semantic caching in LLM gateways. It appears at every layer and often provides the single biggest ROI.

5. **Feedback loops power the flywheel.** User interactions generate implicit labels. Human escalations generate gold labels. Both feed back into retraining. The system that learns fastest wins. But watch for dangerous loops — popularity bias, filter bubbles, runaway feedback.

---

### Top 5 Mistakes Candidates Make

1. **Jumping straight to the model.** The model is 10% of the system. Spend more time on data pipelines, feature engineering, serving infrastructure, and monitoring. Interviewers want to see systems thinking, not just "I'd use a transformer."

2. **Ignoring scale until asked.** Don't wait for "but what about scale?" Proactively discuss what breaks at 10x and 100x. Name specific bottlenecks — the online store, the LLM throughput, the ANN index rebuild time.

3. **Forgetting monitoring and iteration.** A system without monitoring is a system waiting to silently fail. Always include drift detection, alerting, and a retraining strategy. This is what separates a prototype from a production system.

4. **Binary tradeoffs.** Real systems use hybrid approaches. It's not "real-time OR batch" — it's batch candidates with real-time re-ranking. It's not "fine-tuning OR RAG" — it's both. Show nuance.

5. **Skipping the feedback loop.** If your design doesn't explain how the system gets *better over time*, it's incomplete. Name the data flywheel. Name the signals (clicks, thumbs up/down, human labels). Name the retraining trigger.

---

### Final Interview Tips

**Structure beats brilliance.** A well-organized answer that covers all six framework steps will outscore a brilliant but rambling deep-dive into one component. State your framework up front.

**Tradeoffs are the point.** Every design decision has a tradeoff. Name it. "I'd choose HNSW over IVF-PQ here because latency is critical and the dataset fits in RAM, but if we grow to 1B vectors, we'd need to revisit." This is senior thinking.

**Numbers matter.** "The system needs to be fast" is junior. "The system needs p99 latency under 50ms to fit within the payment processor's timeout window" is senior. Use concrete numbers for latency, throughput, data volume, and cost.

**Draw the diagram.** Even in a text-based interview, describe the architecture as a flow: "Request comes in, hits the API gateway, then the router, then the cache, then the model server." The interviewer should be able to draw the system from your description.

**Know when to stop.** You won't cover everything in 45 minutes. Demonstrate breadth by touching all six framework steps, then offer to go deep on any component the interviewer cares about. "I can dive deeper into the feature store design, the re-ranking model, or the monitoring strategy — what's most interesting to you?"

You've got the patterns. You've got the vocabulary. Now go crush it. 🔥
