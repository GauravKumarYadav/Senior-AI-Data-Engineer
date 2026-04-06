---
tags: [system-design, ai, concepts, phase-7]
phase: 7
status: not-started
priority: high
---

# 🏛️ System Design — AI Concepts

> **Phase:** 7 | **Duration:** Review daily during Phase 7 | **Priority:** High
> **Related:** [[01 - System Design AI Problems]], [[03 - RAG Architecture]], [[02 - Model Serving]]

---

## Checklist

### Embedding Search at Scale
- [ ] ANN indexes: HNSW (in-memory, fast), IVF-PQ (disk-friendly, compact)
- [ ] Sharding: partition vectors across multiple nodes (hash on ID or metadata)
- [ ] Replication: replicate for availability and read throughput
- [ ] Filtering: metadata filters + vector search — pre-filter vs post-filter
- [ ] Hybrid: combine dense (embedding) + sparse (BM25) retrieval
- [ ] Refresh: how to update embeddings when documents change
- [ ] Scale: billion-scale search — product quantization, disk-based indexes

### Caching for LLMs
- [ ] Exact match cache: hash(prompt) → cached response (Redis)
- [ ] Semantic cache: embed query → find similar cached queries (vector similarity)
- [ ] Cache invalidation: TTL, manual purge, versioning
- [ ] Cache hit rate optimization: normalize queries before caching
- [ ] Cost savings: avoid LLM call for repeated/similar queries

### Prompt Routing & Model Selection
- [ ] Complexity-based: simple queries → small/fast model, complex → large model
- [ ] Topic-based: different models for different domains
- [ ] Cost-based: budget constraints → route to cheapest adequate model
- [ ] Classifier: lightweight classifier to determine routing
- [ ] Cascading: try cheap model first → escalate if confidence low

### Guardrails Architecture
- [ ] Input validation: check for injection, off-topic, harmful content
- [ ] Output validation: factuality check, PII detection, format compliance
- [ ] PII detection: regex patterns, NER models, redaction before LLM
- [ ] Content filtering: toxicity classifier on output
- [ ] Topic boundaries: classifier to ensure relevance
- [ ] Rate limiting: per-user, per-session limits

### Human-in-the-Loop Design
- [ ] When to escalate: low confidence, sensitive topics, user request
- [ ] Queue management: priority, SLA, assignment
- [ ] Feedback loop: human decisions improve model over time
- [ ] UX: seamless handoff from AI to human agent

### Feedback Loops
- [ ] Implicit feedback: clicks, dwell time, conversions, scroll depth
- [ ] Explicit feedback: thumbs up/down, star ratings, text feedback
- [ ] Logging: store all interactions for analysis and retraining
- [ ] Label quality: inter-annotator agreement, annotation guidelines
- [ ] Continuous learning: retrain models with new labeled data

### A/B Testing for ML Models
- [ ] Metrics selection: business metrics (revenue) vs ML metrics (accuracy)
- [ ] Statistical significance: sample size calculation, p-values, confidence intervals
- [ ] Interleaving: for ranking models — serve mixed results
- [ ] Multi-arm bandits: adaptive allocation, Thompson Sampling
- [ ] Guardrail metrics: metrics that must not degrade (latency, error rate)
- [ ] Analysis: segment analysis, novelty effects, long-term effects

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Designing Machine Learning Systems" by Chip Huyen (Chapters 6-11)
- [ ] Eugene Yan blog: ML system design
- [ ] ByteByteGo: ML system design videos
