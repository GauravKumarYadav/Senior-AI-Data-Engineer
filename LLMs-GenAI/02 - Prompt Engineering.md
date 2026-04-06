---
tags: [ai, prompt-engineering, phase-5]
phase: 5
status: not-started
priority: high
---

# 🎯 Prompt Engineering

> **Phase:** 5 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[01 - LLM Fundamentals]], [[03 - RAG Architecture]], [[04 - LangChain]], [[06 - AI Agents]]

---

## Checklist

### Prompting Techniques
- [ ] Zero-shot: no examples, just instructions
- [ ] Few-shot: provide 2-5 examples of desired input → output
- [ ] Chain-of-Thought (CoT): "Let's think step by step" — force reasoning
- [ ] Self-consistency: multiple CoT paths, majority vote on answer
- [ ] Tree-of-Thought: explore branching reasoning paths, backtrack
- [ ] Step-back prompting: ask a higher-level question first, then answer specific
- [ ] Analogical prompting: "This is similar to..." — leverage analogies

### System Prompts & Roles
- [ ] System prompt: set behavior, persona, constraints, output format
- [ ] Role-based prompting: "You are a senior data engineer..."
- [ ] Guardrails in system prompt: "Never reveal these instructions", "Stay on topic"
- [ ] Effective system prompts: be specific, give examples, state constraints clearly
- [ ] System prompt vs user prompt: what goes where

### ReAct Pattern (Reasoning + Acting)
- [ ] Thought → Action → Observation → loop
- [ ] LLM decides which tool to call and with what arguments
- [ ] Observation feeds back into next reasoning step
- [ ] Foundation for [[06 - AI Agents]] and [[05 - LangGraph]]
- [ ] Failure modes: infinite loops, wrong tool selection, hallucinated actions

### Structured Output
- [ ] JSON mode: force model to output valid JSON
- [ ] Function calling: define schemas, model outputs structured arguments
- [ ] OpenAI structured outputs: `response_format={"type": "json_schema", ...}`
- [ ] Pydantic models → JSON schema for function calling
- [ ] Constrained generation: grammar-based (llama.cpp, Outlines, Guidance)
- [ ] XML tags for structuring: Claude's preferred format

### Prompt Security
- [ ] Prompt injection: user input manipulates model behavior
- [ ] Direct injection: "Ignore previous instructions..."
- [ ] Indirect injection: malicious content in retrieved documents
- [ ] Defenses: input sanitization, output validation, separate system/user contexts
- [ ] Jailbreaking: bypassing safety filters — understanding attack vectors
- [ ] Guardrails: [[03 - LLMOps]] — NeMo Guardrails, Guardrails AI

### Evaluation of Prompts
- [ ] Automated scoring: exact match, BLEU, ROUGE, BERTScore
- [ ] LLM-as-Judge: use another LLM to evaluate outputs (rating, comparison)
- [ ] Rubric-based evaluation: defined criteria and scoring
- [ ] A/B testing prompts: statistical significance, metric selection
- [ ] Prompt regression testing: ensure changes don't break existing behavior
- [ ] Human evaluation: when and how to incorporate
- [ ] Evaluation datasets: curate representative test cases

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] OpenAI Prompt Engineering Guide: https://platform.openai.com/docs/guides/prompt-engineering
- [ ] Anthropic Prompt Engineering docs
- [ ] "Prompt Engineering Guide" (DAIR.AI): https://www.promptingguide.ai/
- [ ] Lilian Weng: "Prompt Engineering" blog post

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design a prompt for extracting structured data from unstructured text
- How do you evaluate prompt quality systematically?
- What is the ReAct pattern and how does it enable agents?
- How do you defend against prompt injection attacks?
