---
tags: [system-design, ai, phase-7]
phase: 7
status: not-started
priority: high
---

# 🏛️ System Design — AI Problems

> **Phase:** 7 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[02 - System Design AI Concepts]], [[03 - RAG Architecture]], [[02 - Model Serving]]

---

## Checklist

### Practice Problems (1 per day)

#### Problem 1: Enterprise RAG Q&A System
- [ ] Document ingestion: PDF/Confluence/Sharepoint → chunking → embedding → vector DB
- [ ] Retrieval: hybrid search (dense + BM25), re-ranking
- [ ] Generation: LLM with retrieved context, citation/attribution
- [ ] Auth: document-level permissions, user can only retrieve authorized docs
- [ ] Caching: semantic cache for repeated questions
- [ ] Evaluation: automated quality checks, user feedback loop
- [ ] Scale: 10M documents, 1000 concurrent users

#### Problem 2: Real-Time Fraud Detection
- [ ] Features: transaction amount, frequency, location, device, time patterns
- [ ] Batch features: historical spend patterns, merchant risk scores (daily)
- [ ] Real-time features: last 5 transactions, velocity checks (streaming)
- [ ] Feature store: Feast — offline for training, online for serving
- [ ] Model: XGBoost/LightGBM for low-latency, ensemble for accuracy
- [ ] Serving: <50ms latency requirement, model server (Triton)
- [ ] Monitoring: drift detection, false positive/negative tracking
- [ ] Feedback loop: fraud team labels → retrain periodically

#### Problem 3: Search Ranking System
- [ ] Stage 1: candidate retrieval — embedding search (ANN) for top-1000
- [ ] Stage 2: re-ranking — cross-encoder or learned-to-rank for top-50
- [ ] Stage 3: personalization — user features + item features → final ranking
- [ ] Features: query-document relevance, click-through rate, freshness
- [ ] Training: learning-to-rank (LambdaMART, or fine-tuned BERT for re-ranking)
- [ ] Online metrics: NDCG, click-through rate, dwell time
- [ ] A/B testing: interleaving experiments for ranking changes

#### Problem 4: AI Customer Support Agent
- [ ] Architecture: LangGraph multi-step agent with tools
- [ ] Tools: knowledge base (RAG), order lookup API, refund processing, escalation
- [ ] Multi-turn: conversation memory, context management
- [ ] Guardrails: topic restriction, PII handling, escalation triggers
- [ ] Human-in-the-loop: escalate complex/sensitive cases to human agents
- [ ] Evaluation: resolution rate, CSAT, containment rate, safety
- [ ] Streaming: token-by-token response for UX

#### Problem 5: LLM Gateway / Router
- [ ] API gateway: single endpoint, route to multiple LLM providers
- [ ] Rate limiting: per-user, per-model, global limits
- [ ] Fallback: primary model fails → fallback to secondary
- [ ] Model routing: complexity-based (simple → GPT-4-mini, complex → GPT-4)
- [ ] Caching: exact match + semantic caching
- [ ] Cost optimization: track spending per team, budget alerts
- [ ] Observability: logging, tracing, latency tracking

#### Problem 6: Content Moderation System
- [ ] Multi-modal: text + image + video analysis
- [ ] Pipeline: fast classifier (filter obvious) → LLM (nuanced cases) → human review
- [ ] Categories: hate speech, violence, spam, misinformation, NSFW
- [ ] Low latency: pre-upload screening for real-time platforms
- [ ] Feedback loop: human decisions train better classifiers
- [ ] Scale: millions of posts per day, prioritization queue

#### Problem 7: ML Feature Platform
- [ ] Feature computation: batch (Spark) + streaming (Flink/Spark Streaming)
- [ ] Offline store: data warehouse/lake — historical features for training
- [ ] Online store: Redis/DynamoDB — low-latency for serving
- [ ] Point-in-time correctness: prevent feature leakage in training
- [ ] Feature registry: discoverability, documentation, ownership
- [ ] Materialization: schedule batch → online sync
- [ ] Monitoring: feature freshness, drift, serving latency

#### Problem 8: Model Monitoring & Drift Detection
- [ ] Data drift: input distribution changes (KL divergence, PSI)
- [ ] Concept drift: relationship between input and output changes
- [ ] Prediction drift: output distribution changes
- [ ] Detection: statistical tests, reference windows, alerts
- [ ] Response: retrain trigger, human review, rollback
- [ ] Dashboard: feature distributions, model performance over time

#### Problem 9: Multi-Agent Document Processing
- [ ] Agent 1: document classification (type, urgency, department)
- [ ] Agent 2: information extraction (key fields, entities)
- [ ] Agent 3: validation and cross-referencing
- [ ] Agent 4: routing and action determination
- [ ] Orchestration: LangGraph supervisor pattern
- [ ] Error handling: human review queue for low-confidence outputs

---

## 📝 ML System Design Framework
1. **Clarify:** business problem, success metrics, scale
2. **Data:** sources, labels, features, quality
3. **Model:** architecture, training approach, offline metrics
4. **Serving:** latency, throughput, deployment strategy
5. **Monitoring:** drift, quality, feedback loops
6. **Iteration:** retraining, A/B testing, continuous improvement

---

## 🔗 Resources
- [ ] "Designing Machine Learning Systems" by Chip Huyen
- [ ] "Machine Learning System Design Interview" by Ali Aminian
- [ ] Stanford CS 329S: Machine Learning Systems Design
