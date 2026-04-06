---
tags: [ai, llm, phase-4]
phase: 4
status: not-started
priority: high
---

# 🧠 LLM Fundamentals

> **Phase:** 4 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[02 - Deep Learning Foundations]], [[03 - NLP for AI Engineers]], [[02 - Prompt Engineering]], [[03 - RAG Architecture]]

---

## Checklist

### GPT Architecture
- [ ] Decoder-only Transformer: causal (autoregressive) self-attention
- [ ] Causal mask: each token can only attend to previous tokens
- [ ] Next-token prediction: P(token_n | token_1, ..., token_{n-1})
- [ ] Training: massive text corpus, self-supervised, cross-entropy loss
- [ ] Pre-training → SFT (Supervised Fine-Tuning) → RLHF/DPO (alignment)
- [ ] KV-cache: store computed key/value pairs, avoid recomputation during generation

### Scaling Laws
- [ ] Chinchilla scaling: optimal balance of model size vs training data
- [ ] Emergent abilities: capabilities that appear at scale (chain-of-thought, etc.)
- [ ] Compute-optimal training: N parameters ≈ 20N training tokens
- [ ] Diminishing returns: 10x compute → ~0.5x perplexity improvement
- [ ] Over-training smaller models: Llama approach (more tokens, smaller model)

### Model Landscape (2024-2026)
- [ ] OpenAI: GPT-4, GPT-4o, o1/o3 (reasoning), GPT-4 Turbo
- [ ] Anthropic: Claude 3 (Haiku, Sonnet, Opus), Claude 3.5
- [ ] Meta: Llama 2, Llama 3 (8B, 70B, 405B) — open weights
- [ ] Mistral: Mistral 7B, Mixtral 8x7B (MoE), Mistral Large
- [ ] Google: Gemini (Nano, Pro, Ultra), Gemma (open)
- [ ] Comparison: quality vs speed vs cost vs context window vs open/closed

### Inference Parameters
- [ ] Temperature: controls randomness (0 = deterministic, 1 = creative)
- [ ] Top-p (nucleus sampling): cumulative probability threshold
- [ ] Top-k: consider only k most likely tokens
- [ ] Beam search: explore multiple sequences, select best (more deterministic)
- [ ] Max tokens: limit output length
- [ ] Stop sequences: terminate generation at specific strings
- [ ] Frequency/presence penalty: reduce repetition
- [ ] Seed: reproducible outputs (when supported)

### Context Windows & Tokenization
- [ ] Context window: maximum total tokens (input + output)
- [ ] GPT-4 Turbo: 128K context, Claude: 200K, Gemini: 1M+
- [ ] Long context challenges: "lost in the middle" — models attend to beginning/end
- [ ] Tokenization cost: 1 token ≈ 4 characters (English), varies by language
- [ ] Pricing: input tokens vs output tokens (output typically 3-4x more expensive)
- [ ] Token counting: `tiktoken` for OpenAI, model-specific tokenizers
- [ ] Context window management: summarization, rolling context, RAG

### LLM APIs
- [ ] OpenAI API: `ChatCompletion`, messages format (system, user, assistant)
- [ ] Function calling / Tool use: structured output, JSON schema
- [ ] Streaming: `stream=True`, SSE (Server-Sent Events), token-by-token output
- [ ] Batching: Batch API for async, cheaper processing
- [ ] Rate limits: TPM (tokens per minute), RPM (requests per minute)
- [ ] Error handling: retries, exponential backoff, fallback models

### Multi-Modal LLMs
- [ ] Vision: GPT-4V, Claude Vision — image understanding
- [ ] Audio: Whisper (transcription), GPT-4o (native audio)
- [ ] Use cases: document understanding, chart analysis, visual QA
- [ ] Sending images: base64 encoding, URL reference, token cost

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] OpenAI docs: https://platform.openai.com/docs
- [ ] Anthropic docs: https://docs.anthropic.com/
- [ ] Andrej Karpathy: "State of GPT" (talk)
- [ ] "Attention Is All You Need" + GPT papers
- [ ] Lilian Weng blog: "Large Language Model" series

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain how GPT generates text (step by step)
- What are scaling laws and why do they matter?
- Compare temperature, top-p, and top-k — when to use each
- How do you choose between GPT-4, Claude, and an open-source model?
