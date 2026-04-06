---
tags: [llmops, ai, phase-6]
phase: 6
status: not-started
priority: high
---

# 🔭 LLMOps

> **Phase:** 6 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[01 - MLOps Pipeline]], [[02 - Model Serving]], [[04 - LangChain]], [[05 - LangGraph]]

---

## Checklist

### LLM Observability
- [ ] LangSmith: traces, evaluations, datasets, playground (LangChain ecosystem)
- [ ] Langfuse: open-source alternative, traces, scores, cost tracking
- [ ] Phoenix (Arize): traces, embeddings visualization, evals
- [ ] Tracing: track each LLM call, retrieval step, tool call — latency, tokens, cost
- [ ] What to log: inputs, outputs, model, temperature, token counts, latency, cost
- [ ] Dashboards: request volume, error rates, latency percentiles, cost trends

### Cost Management
- [ ] Token usage monitoring: input vs output tokens per request
- [ ] Cost per query: aggregate across model calls + embeddings + vector search
- [ ] Semantic caching: cache similar queries → reuse answers (GPTCache)
- [ ] Prompt compression: shorten prompts while preserving meaning (LLMLingua)
- [ ] Model routing: use cheaper model (GPT-4-mini) for simple queries, expensive for complex
- [ ] Batching: group requests for better throughput
- [ ] Budget alerts: set spending limits, notifications

### Guardrails
- [ ] NeMo Guardrails (NVIDIA): define rails for input/output, topical control
- [ ] Guardrails AI: output validation, structured output enforcement
- [ ] Input guardrails: block harmful, off-topic, injection attempts
- [ ] Output guardrails: PII detection, hallucination check, format validation
- [ ] Content moderation: OpenAI Moderation API, custom classifiers
- [ ] Topic filtering: restrict to allowed topics, domain boundaries

### Evaluation Pipelines
- [ ] Offline evaluation: run model on test datasets before deployment
- [ ] Online evaluation: sample live traffic, score responses
- [ ] LLM-as-Judge: automated scoring with criteria rubrics
- [ ] Human evaluation: labeling interface, inter-annotator agreement
- [ ] Regression testing: ensure prompt changes don't degrade quality
- [ ] Evaluation metrics: relevance, faithfulness, helpfulness, safety
- [ ] Custom evaluators: domain-specific criteria

### CI/CD for AI Applications
- [ ] Prompt versioning: store prompts in version control, track changes
- [ ] Evaluation gates: automated eval → pass/fail before deployment
- [ ] Config management: model, temperature, system prompt as config (not code)
- [ ] Feature flags: roll out prompt changes gradually
- [ ] Rollback: quick revert to previous prompt/model version
- [ ] Testing: unit tests for chains, integration tests for full pipeline
- [ ] Staging environment: test against representative queries

### Monitoring in Production
- [ ] Drift detection: distribution shift in inputs or outputs
- [ ] Quality degradation: declining evaluation scores over time
- [ ] Latency monitoring: P50, P95, P99 response times
- [ ] Error tracking: rate of failures, types of errors
- [ ] User feedback: thumbs up/down, explicit ratings
- [ ] Alerting: PagerDuty/Slack for quality drops, errors, cost spikes

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] LangSmith docs: https://docs.smith.langchain.com/
- [ ] Langfuse docs: https://langfuse.com/docs
- [ ] NeMo Guardrails: https://github.com/NVIDIA/NeMo-Guardrails
- [ ] Hamel Husain: "Your AI Product Needs Evals" (blog)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design an observability system for an LLM application
- How do you manage costs for a high-traffic LLM API?
- What guardrails would you implement for a customer-facing chatbot?
