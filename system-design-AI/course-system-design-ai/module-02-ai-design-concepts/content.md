# AI System Design Concepts

Every ML system you'll ever design—whether it's a recommendation engine, a fraud detector, or an LLM-powered assistant—shares the same set of recurring building blocks. This module is a deep-dive into those building blocks. Master these concepts and you'll be able to reason about *any* AI system design problem thrown at you, in interviews or in production.

---

## 1. ML System Design Framework

The single most important tool in your AI system design toolkit is a repeatable framework. Without one, you'll ramble. With one, you'll be structured, thorough, and confident. Here's the six-step framework that works for virtually every ML system design question.

### Step 1: Clarify Requirements

Before you touch a single model, **ask questions**. What is the business objective? Who are the users? What are the latency constraints? What's the scale (requests per second, data volume)? Are there fairness or compliance requirements?

Common pitfalls: jumping straight to modeling, assuming requirements that were never stated, optimizing for ML metrics nobody cares about.

> **Interview Tip:** Spend 3–5 minutes clarifying. Interviewers *want* you to ask questions. It shows maturity. A senior engineer asks "what does success look like?" before writing a line of code.

### Step 2: Data

Where does the data come from? What's the labeling strategy (human labels, implicit signals, weak supervision)? How much data exists today, and how fast does it grow? What does the schema look like?

Key decisions: batch vs streaming ingestion, label quality, handling class imbalance, privacy constraints (PII masking, data retention policies), data versioning.

### Step 3: Model

Choose the right model for the right job. Don't reach for a transformer when logistic regression will do. Consider the tradeoff triangle: **accuracy vs latency vs cost**. Define your offline evaluation metrics (precision, recall, NDCG, BLEU—whatever fits).

Architecture decisions: single model vs multi-stage pipeline (candidate generation → ranking → re-ranking), ensemble vs single model, pretrained foundation model vs train-from-scratch.

### Step 4: Serving

How does the model get predictions to users? Real-time inference (model behind an API), batch prediction (precompute and store), or near-real-time (streaming inference)?

Key concerns: latency SLAs, throughput, model serialization format (ONNX, TorchScript, SavedModel), hardware (CPU vs GPU vs specialized accelerators), autoscaling, load balancing.

### Step 5: Monitoring

Models rot. Data drifts. The world changes. You need to monitor: prediction distributions, feature distributions, latency, error rates, and—critically—business metrics. Set up alerts for data drift (PSI, KL divergence) and model performance degradation.

### Step 6: Iteration

ML is never "done." Build feedback loops so the system improves over time. Retrain on fresh data. Run A/B tests. Iterate on features. This step connects back to Step 2 (new data) and Step 3 (better models), creating a virtuous cycle.

> **Interview Tip:** State this framework up front: "I'll walk through clarification, data, modeling, serving, monitoring, and iteration." It gives the interviewer a roadmap and shows you think in systems, not just models.

---

## 2. Feature Stores

A feature store is a centralized system for managing, storing, and serving ML features. Without one, every team computes features independently—leading to duplicated work, inconsistent definitions, and training-serving skew.

### Why They Exist

Imagine three teams all computing "user's average order value over the last 30 days." Without a feature store, each team writes its own SQL, its own pipeline, and inevitably each gets slightly different numbers. A feature store gives you **one definition, one pipeline, one source of truth**.

### Offline vs Online Stores

- **Offline store**: A data warehouse (BigQuery, S3 + Parquet) holding historical feature values. Used for training. You query it with a timestamp: "give me user X's features *as they were on March 15th*."
- **Online store**: A low-latency key-value store (Redis, DynamoDB) holding the *latest* feature values. Used for real-time inference. Sub-millisecond reads.

### Point-in-Time Correctness

This is the subtle killer. When you train a model, you need features as they existed *at the time of each training example*. If you accidentally use future data (e.g., using a user's lifetime value to predict their first purchase), that's **feature leakage**, and your offline metrics will look amazing while your production model falls flat.

A proper feature store handles point-in-time joins automatically, ensuring your training data is temporally consistent.

### Materialization Pipelines

Features don't appear by magic. **Batch features** are computed periodically (hourly, daily) via Spark or SQL jobs and written to the offline/online store. **Streaming features** are computed in real-time via Flink or Spark Streaming—think "number of transactions in the last 5 minutes" for fraud detection.

```
[Raw Events] → [Flink / Spark Streaming] → [Online Store (Redis)]
[Data Warehouse] → [Spark Batch Job] → [Offline Store (S3/BQ)]
```

### Feature Registries

A catalog of all available features with metadata: owner, description, data type, freshness SLA, lineage (what raw data it came from). Tools like **Feast** (open-source) and **Tecton** (managed) provide this out of the box.

> **Interview Tip:** Mentioning feature stores signals you understand production ML. Say: "I'd use a feature store to avoid training-serving skew and ensure point-in-time correctness."

---

## 3. Model Registries

A model registry is a versioned catalog of trained models along with their metadata—hyperparameters, training data version, evaluation metrics, lineage, and deployment status.

### Why You Need One

Without a registry, "which model is in production?" becomes an archaeology exercise. Someone SSHs into a server, finds a `model_v3_final_FINAL.pkl`, and hopes for the best. A model registry gives you:

- **Versioning**: Every model gets a version. You can diff v7 vs v8.
- **Metadata**: What data was it trained on? What were the hyperparameters? What's the AUC?
- **Lineage**: Trace from a production prediction back to the exact training run, dataset, and code commit.
- **Promotion workflows**: Models move through stages—`None → Staging → Production → Archived`. Promotion requires approval, passing tests, and meeting metric thresholds.

### MLflow Model Registry

**MLflow** is the most common open-source example. You log a model with `mlflow.log_model()`, register it, then promote it through stages. It integrates with most serving frameworks and CI/CD pipelines.

```python
import mlflow

with mlflow.start_run():
    mlflow.log_params({"learning_rate": 0.01, "epochs": 50})
    mlflow.log_metrics({"auc": 0.92, "f1": 0.87})
    mlflow.sklearn.log_model(model, "model", registered_model_name="fraud-detector")
```

> **Interview Tip:** Governance and reproducibility are first-class concerns in production ML. Model registries are how you achieve them. Mentioning this shows you've operated ML systems at scale.

---

## 4. A/B Testing for ML

A/B testing ML systems is harder than A/B testing a button color. Here's why—and how to do it right.

### Why It's Harder

ML changes are **diffuse**. A new ranking model affects every search result, every user session, in complex ways. Effects are often small, noisy, and take time to materialize. You might improve click-through rate but hurt long-term retention—how do you catch that?

### Metrics Selection

Define metrics at two levels:

- **Business metrics** (the ones leadership cares about): revenue, engagement, retention, conversion rate.
- **ML metrics** (proxies you can measure faster): NDCG, precision@k, prediction accuracy.

The danger: optimizing for ML metrics that don't correlate with business metrics. Always validate the connection.

### Statistical Rigor

Decide on **statistical significance** thresholds (typically p < 0.05), **minimum detectable effect** (how small an improvement do you care about?), and **sample size** (which determines how long you need to run the test). Running underpowered tests leads to false negatives; peeking at results early inflates false positives.

### Advanced Techniques

- **Interleaving** (for ranking): Show results from both models interleaved in a single list. Users implicitly "vote" by clicking. Much more sensitive than split A/B tests—needs ~10x fewer samples.
- **Multi-arm bandits**: Instead of fixed 50/50 splits, dynamically allocate more traffic to the winning variant. **Thompson Sampling** uses Bayesian posteriors; **epsilon-greedy** explores with probability ε. Bandits are great when you want to minimize regret during the test.

### Guardrail Metrics

Even if your primary metric improves, you need **guardrails**—metrics that must not degrade. Examples: latency p99, crash rate, content safety violations. If a guardrail trips, the experiment stops.

### Novelty Effects and Long-Term Measurement

Users engage more with anything new. A shiny new recommendation algorithm might show a lift in week one that vanishes by week three. Run tests long enough to see past the novelty. For long-term effects (retention, LTV), consider **holdback groups**—a small percentage of users permanently on the old model.

> **Interview Tip:** When proposing an A/B test, always specify: primary metric, guardrail metrics, sample size calculation, and test duration. This is the difference between a junior and senior answer.

---

## 5. Online vs Offline Evaluation

### Offline Evaluation

You evaluate against a static dataset *before* deploying. Common metrics:

| Task | Metrics |
|---|---|
| Classification | Precision, Recall, F1, AUC-ROC |
| Ranking | NDCG, MAP, MRR |
| Regression | MSE, MAE, R² |
| Generation (LLMs) | BLEU, ROUGE, BERTScore |

Techniques: hold-out test sets, k-fold cross-validation, stratified splits for imbalanced data. Always use a **temporal split** for time-series data (train on past, test on future).

### Online Evaluation

You evaluate in production with real users. Methods:

- **Shadow mode**: New model runs alongside production, receives real traffic, but its predictions are *not* shown to users. You compare outputs offline.
- **Canary deployment**: Roll out to 1–5% of traffic. Monitor for regressions. Gradually increase if healthy.
- **A/B test**: Full statistical comparison (covered in Section 4).
- **Interleaving**: Merge results from two rankers into one list (covered in Section 4).

### The Gap

A model can improve offline metrics by 5% and show zero lift online. Why?

1. **Distribution mismatch**: Your test set doesn't reflect real traffic patterns.
2. **Feedback effects**: The model changes user behavior, which changes the data distribution.
3. **Latency**: A more accurate but slower model may cause users to abandon.
4. **Business context**: A 2% AUC improvement might be irrelevant if the bottleneck is inventory, not prediction quality.

> **Interview Tip:** Always say: "I'd validate offline first, then run a shadow deployment, then a canary, and finally a full A/B test." This progressive rollout strategy is production-grade thinking.

---

## 6. The Data Flywheel

The data flywheel is the most powerful competitive moat in AI: **more users → more data → better models → better product → more users**. It's a compounding loop.

### Examples

- **Google Search**: Billions of queries generate click data that improves ranking models, which makes search better, which attracts more users.
- **Netflix**: Every play, pause, skip, and rewatch trains recommendation models that surface better content, increasing engagement.
- **Tesla**: Every mile driven feeds back camera and sensor data that improves self-driving models, which makes the product more compelling.

### Bootstrapping (Cold Start for the Flywheel)

The flywheel needs a push to start spinning:

- Use **heuristics or rules** as v1 (popularity-based recommendations).
- License or **purchase seed data**.
- Use **transfer learning** from pretrained models.
- Build a **great v1 product** that attracts users even without great ML (Instagram had filters before it had recommendations).

### When the Flywheel Stalls

More data doesn't always mean better models. You hit diminishing returns when: the model is limited by architecture (not data), the data is noisy or biased, or you're optimizing for the wrong metric. Recognizing when to invest in model architecture vs more data is a critical skill.

> **Interview Tip:** When designing a system, explicitly name the flywheel: "User interactions generate implicit labels that feed back into retraining, creating a data flywheel." This shows you think about long-term system dynamics.

---

## 7. Feedback Loops

Feedback loops are how ML systems learn from the real world after deployment. Get them wrong and your model either stagnates or spirals into dysfunction.

### Implicit vs Explicit Feedback

**Implicit feedback** is inferred from behavior—it's abundant but noisy:
- Clicks (positive signal, but click-bait inflates it)
- Dwell time (longer = more engaged, usually)
- Scroll depth, video watch percentage
- Conversions, purchases, add-to-cart
- Skip, dismiss, back-button (negative signals)

**Explicit feedback** is volunteered by users—it's clean but sparse:
- Thumbs up/down, star ratings
- Text reviews, survey responses
- "Not relevant" or "report" buttons

### Logging Requirements

You must log **everything** needed to reconstruct the decision: the model version, the features used, the candidates considered, the scores, the final ranking, and the user's response. Without this, you can't do counterfactual analysis or offline replay.

### Label Quality

If you're using human annotators, measure **inter-annotator agreement** (Cohen's kappa, Fleiss' kappa). Low agreement means ambiguous guidelines—fix the guidelines before scaling up annotation. Consider using **weak supervision** (Snorkel-style labeling functions) to programmatically generate labels at scale.

### Dangerous Feedback Loops

- **Popularity bias**: The model recommends popular items → popular items get more clicks → the model recommends them even more. Unpopular items never get a chance.
- **Filter bubbles**: The model shows users content similar to what they've engaged with → their interests appear to narrow → the model doubles down. Diversity drops.
- **Runaway loops**: A fraud model flags a legitimate user → the user behaves oddly (retries, calls support) → the model sees the odd behavior as more evidence of fraud.

Mitigations: exploration (show some random/diverse items), debiasing techniques, and monitoring diversity metrics.

> **Interview Tip:** Always address feedback loops in your design. Say: "I'd log all impressions and interactions to create a continuous learning pipeline, while monitoring for popularity bias and filter bubbles."

---

## 8. Cold Start Problem

The cold start problem is what happens when your ML system has no data to work with—for a new user, a new item, or an entirely new system.

### New Users

No interaction history means collaborative filtering is useless. Solutions:

- **Demographic priors**: Use age, location, device type to map to user segments with known preferences.
- **Onboarding flows**: Ask users to select interests, rate sample items, or connect social accounts.
- **Content-based features**: Recommend based on item attributes rather than user history.
- **Popularity-based defaults**: Show globally popular items as a baseline.

### New Items

No users have interacted with the new item yet. Solutions:

- **Content-based features**: Use item metadata (title, description, category, image embeddings) to find similar items and borrow their engagement signals.
- **Exploration strategies**: Inject new items into recommendations with boosted exposure, then let real engagement data take over.
- **Active learning**: Prioritize showing new items to users whose feedback is most informative.

### New Systems

You have no data at all. This is the "cold start for the flywheel" problem from Section 6. Solutions: transfer learning from pretrained models, synthetic data generation, seed data from public datasets, and rule-based v1 systems.

> **Interview Tip:** When designing a recommendation system, the interviewer *will* ask about cold start. Have a crisp answer for each scenario: new user, new item, new system.

---

## 9. Embedding Infrastructure

Embeddings are dense vector representations that encode semantic meaning. They're the lingua franca of modern ML—used in search, recommendations, clustering, classification, and retrieval-augmented generation.

### Types of Embeddings

- **Text embeddings**: Sentence-transformers (open-source, self-hosted), OpenAI's `text-embedding-3-small/large` (API-based). Map text to 384–3072 dimensional vectors.
- **Image embeddings**: CLIP (maps images and text into the same space), ResNet feature extractors.
- **Multi-modal**: CLIP, ImageBind—encode different modalities into a shared vector space so you can search images with text queries.

### Generation Pipelines

Embeddings don't just appear—you need a pipeline:

```
[Raw Data (text/images)] → [Embedding Model] → [Vectors] → [Vector Store]
```

For batch: run a Spark/Dataflow job over your corpus, generate embeddings, write to storage.
For real-time: embed queries on-the-fly at inference time via an embedding service.

### Storage and Versioning

Embeddings are large (millions of items × 768 dimensions = gigabytes). Store them in:
- Vector databases (Pinecone, Weaviate, Milvus)
- Columnar formats (Parquet) for offline use
- Redis with vector search extensions for online use

**Versioning** matters: when you update your embedding model, all vectors become incompatible. You need to re-embed your entire corpus and swap atomically. Maintain a mapping of `embedding_model_version → vector_index_version`.

### Refresh Strategies

When should you re-embed? Options:
- **Full re-embed on model update** (expensive but clean)
- **Incremental** for new items, full re-embed on a schedule
- **Never** if the embedding model is frozen (e.g., using a stable API version)

> **Interview Tip:** Embeddings come up in almost every modern ML design question. Be ready to discuss: which model, dimensionality, storage, and how you handle model updates.

---

## 10. Vector Search at Scale

Once you have embeddings, you need to search them efficiently. Exact nearest-neighbor search is O(n)—fine for 10k vectors, useless for 100M. Enter **Approximate Nearest Neighbor (ANN)** search.

### ANN Index Types

**HNSW (Hierarchical Navigable Small World)**:
- Graph-based, in-memory
- Excellent recall (>95%) at high speed
- Tradeoff: high memory usage (stores full vectors + graph)
- Best for: datasets that fit in RAM, latency-critical applications

**IVF-PQ (Inverted File Index + Product Quantization)**:
- Partition vectors into clusters (IVF), compress via product quantization (PQ)
- Much lower memory footprint
- Slightly lower recall than HNSW
- Best for: billion-scale datasets, disk-friendly deployments

### Sharding

When your index outgrows a single machine, shard across nodes. Strategies:
- **Hash-based**: Distribute by item ID. Query goes to all shards, results merge.
- **Partition-based**: Cluster vectors, assign clusters to shards. Query routes to relevant shards only.

### Metadata Filtering

Often you need "find similar items *in category X*" or "find nearest neighbors *created after 2024*." Two approaches:
- **Pre-filter**: Apply metadata filter first, then search within the filtered set. Precise but can be slow if the filter is very selective (small candidate set → poor ANN performance).
- **Post-filter**: Run ANN search first, then discard results that don't match filters. Fast but wasteful if many results are filtered out—you may return fewer results than requested.

### Hybrid Search

Combine dense vectors (semantic similarity) with sparse vectors (BM25 / keyword matching). This handles queries where exact keywords matter (product SKUs, proper nouns) alongside semantic understanding.

```
Final Score = α × dense_score + (1 - α) × sparse_score
```

### Databases

| Database | Type | Notes |
|---|---|---|
| Pinecone | Managed | Easy to use, scales well |
| Weaviate | Open-source | Hybrid search built-in |
| Milvus | Open-source | Billion-scale, GPU support |
| pgvector | Postgres ext. | Good for small-medium scale |

> **Interview Tip:** When designing a retrieval system, specify: index type (HNSW vs IVF-PQ), whether you need metadata filtering (and pre vs post), and whether hybrid search is warranted. This shows depth.

---

## 11. Caching for LLMs

LLM inference is expensive—$0.01–$0.10 per call at scale adds up fast. Caching is your first line of defense.

### Exact Match Cache

Hash the prompt → look up in Redis/Memcached. If hit, return cached response. Simple, fast, effective for repetitive queries (FAQ bots, common search queries).

```python
import hashlib, redis

def get_or_generate(prompt: str) -> str:
    cache_key = hashlib.sha256(prompt.encode()).hexdigest()
    cached = redis_client.get(cache_key)
    if cached:
        return cached.decode()
    response = llm.generate(prompt)
    redis_client.setex(cache_key, ttl=3600, value=response)
    return response
```

**Limitation**: "What's the weather?" and "Tell me the weather" are different hashes. Zero fuzzy matching.

### Semantic Cache

Embed the query → search a vector index of previously cached queries → if similarity > threshold, return the cached response. This catches paraphrases and near-duplicates.

```
[User Query] → [Embed] → [Vector Search cached queries]
    → if similarity > 0.95 → return cached response
    → else → call LLM → cache query embedding + response
```

Tradeoff: adds ~10–50ms of latency for the embedding + vector search, but saves ~500–2000ms of LLM inference when it hits.

### Cache Invalidation

- **TTL (Time-to-Live)**: Expire after N seconds. Simple but crude.
- **Version-based**: Tag cache entries with a version. When underlying data changes (e.g., product catalog update), bump the version and invalidate.
- **Query normalization**: Lowercase, strip whitespace, remove filler words *before* hashing to increase hit rate.

### Cost Savings Math

If your cache hit rate is 40% and each LLM call costs $0.02, then for 1M daily queries:
- Without cache: 1M × $0.02 = **$20,000/day**
- With cache: 600K × $0.02 = **$12,000/day**
- Savings: **$8,000/day** = **$240K/month**

> **Interview Tip:** Always mention caching when designing LLM systems. The cost savings are too significant to ignore. Specify exact vs semantic caching based on query diversity.

---

## 12. Prompt Routing & Model Selection

Not every query needs GPT-4. Smart routing sends simple queries to cheap, fast models and reserves expensive models for complex ones.

### Routing Strategies

**Complexity-based routing**: A lightweight classifier (logistic regression, small BERT) estimates query complexity. Simple factual questions → small model (GPT-3.5 / Llama-7B). Complex reasoning → large model (GPT-4 / Llama-70B).

**Topic-based routing**: Route to specialized models. Medical questions → fine-tuned medical model. Code questions → code-specialized model. General chat → general model.

**Cost-based routing**: Set a budget. Route to the cheapest model that meets a quality threshold for each query type.

### Cascading

Try the cheapest model first. If the response confidence is low (measured by log-probabilities, self-consistency, or a quality classifier), escalate to a more expensive model.

```
User Query → Small Model (fast, cheap)
    → if confidence > threshold → return response
    → else → Large Model (slow, expensive) → return response
```

This can reduce costs by 50–70% with minimal quality degradation, because most queries are simple.

### Implementation

The router itself must be fast (< 10ms) and cheap. Options:
- Rule-based (keyword matching, query length)
- Lightweight classifier (TF-IDF + logistic regression)
- Small embedding model + nearest-centroid classification

> **Interview Tip:** Prompt routing shows you think about cost optimization at the architecture level. Mention it whenever you're designing a multi-model LLM system.

---

## 13. Guardrails Architecture

LLMs are powerful but unpredictable. Guardrails are the safety net between the model and the user.

### Input Validation

Before the prompt reaches the model:
- **Prompt injection detection**: Classify whether the user is trying to override system instructions ("ignore previous instructions and..."). Use a fine-tuned classifier or pattern matching.
- **Off-topic detection**: Is this query within the system's intended scope? A customer service bot shouldn't answer questions about nuclear physics.
- **Harmful content filtering**: Block requests for illegal content, self-harm instructions, etc.

### Output Validation

After the model responds, before the user sees it:
- **PII detection**: Scan for social security numbers, credit card numbers, email addresses, phone numbers. Redact or block.
- **Factuality checking**: For RAG systems, verify the response is grounded in the retrieved documents (not hallucinated).
- **Format compliance**: Does the response match the expected schema? If you asked for JSON, did you get valid JSON?
- **Toxicity classification**: Run through a toxicity model (Perspective API, custom classifier).

### Architecture Pattern

```
[User Input]
    → [Input Guardrails] → block / pass
    → [LLM / RAG Pipeline]
    → [Output Guardrails] → block / redact / pass
    → [User Response]
```

### Topic Boundaries and Rate Limiting

Define what the system can and cannot discuss. Implement as a combination of system prompts and classifier-based enforcement. Add rate limiting per user to prevent abuse and cost explosions.

### Frameworks

- **NeMo Guardrails** (NVIDIA): Define conversational rails in a declarative format (Colang). Good for dialogue flow control.
- **Guardrails AI**: Schema-based output validation. Define expected structure, validators auto-fix or reject.

> **Interview Tip:** In any LLM system design, dedicate a section to guardrails. Mention both input and output validation. Interviewers want to see you think about safety and reliability, not just the happy path.

---

## 14. Human-in-the-Loop Design

No ML system is perfect. The best systems know when to admit uncertainty and escalate to a human.

### When to Escalate

- **Low confidence**: Model prediction probability below a threshold (e.g., < 0.7 for classification, high perplexity for generation).
- **Sensitive topics**: Medical advice, legal questions, financial transactions above a threshold, content involving minors.
- **User request**: "Let me talk to a human" should always work.
- **Guardrail triggers**: If input or output guardrails flag an issue that can't be auto-resolved.

### Queue Management

Escalated requests need a queue system:
- **Priority**: Urgent issues (safety, high-value customers) jump the queue.
- **SLA**: Define response time targets (e.g., 95% of escalations handled within 5 minutes).
- **Assignment**: Route to agents with the right expertise (topic-based routing, but for humans).
- **Load balancing**: Distribute evenly across available agents, accounting for shift schedules and capacity.

### The Feedback Loop

Human decisions are *gold labels*. Every time a human resolves an escalated case, that decision should feed back into the training pipeline:

```
[AI handles query] → [Low confidence] → [Escalate to human]
    → [Human resolves] → [Log decision as training label]
    → [Retrain model] → [AI handles similar queries next time]
```

Over time, the escalation rate should decrease as the model learns from human decisions. Track this rate as a key system health metric.

### UX for Seamless Handoff

The user experience during handoff matters enormously:
- **Preserve context**: The human agent must see the full conversation history, the AI's attempted response, and the reason for escalation.
- **Warm handoff**: "I'm connecting you with a specialist who can help with this" is better than a cold transfer.
- **No dead ends**: If no human is available, provide a fallback (email, callback, ticket) rather than just failing.

> **Interview Tip:** Human-in-the-loop isn't a failure of the AI system—it's a feature. Frame it as: "The system knows what it doesn't know, and it uses human expertise to both serve the user and improve itself over time."

---

## Tying It All Together

These 14 concepts don't exist in isolation. In a real system, they're deeply interconnected:

- Your **feature store** feeds features to models tracked in your **model registry**, which are evaluated **offline** then promoted via **A/B tests** (online evaluation).
- User interactions create **feedback loops** that spin the **data flywheel**, while the **cold start problem** determines your bootstrapping strategy.
- **Embeddings** power **vector search**, which is central to retrieval-augmented generation systems that also need **caching**, **prompt routing**, **guardrails**, and **human-in-the-loop** escalation.
- **Monitoring** (from the framework) ties everything together—watching for drift, degradation, and dangerous feedback loops.

The ML System Design Framework from Section 1 is your skeleton. The remaining 13 concepts are the organs. Learn them individually, but always think about how they connect.

```
┌─────────────────────────────────────────────────────────┐
│                  ML System Design                       │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │ Feature  │──▶│  Model   │──▶│    Serving        │    │
│  │  Store   │   │          │   │  (+ Caching,      │    │
│  └──────────┘   └──────────┘   │   Routing,        │    │
│       ▲              │         │   Guardrails)      │    │
│       │              ▼         └────────┬───────────┘    │
│       │         ┌──────────┐            │               │
│       │         │  Model   │            ▼               │
│       │         │ Registry │     ┌─────────────┐        │
│       │         └──────────┘     │  A/B Test /  │        │
│       │                          │  Monitoring  │        │
│       │                          └──────┬──────┘        │
│       │                                 │               │
│       │         ┌──────────────┐        │               │
│       └─────────│  Feedback    │◀───────┘               │
│                 │  Loops       │                         │
│                 │  (Flywheel)  │                         │
│                 └──────────────┘                         │
└─────────────────────────────────────────────────────────┘
```

When you walk into a system design interview, you're not just designing a model—you're designing this entire ecosystem. These building blocks are your vocabulary. Speak it fluently.
