---
tags: [ai, nlp, phase-4]
phase: 4
status: not-started
priority: high
---

# 📝 NLP for AI Engineers

> **Phase:** 4 | **Duration:** ~2 days | **Priority:** High
> **Related:** [[02 - Deep Learning Foundations]], [[01 - LLM Fundamentals]], [[03 - RAG Architecture]]

---

## Checklist

### Tokenization
- [ ] Word-level: simple, huge vocabulary, OOV (out-of-vocabulary) problem
- [ ] Subword: BPE (Byte-Pair Encoding) — used by GPT, most common
- [ ] WordPiece: used by BERT, similar to BPE but probability-based
- [ ] SentencePiece: language-agnostic, used by T5, Llama
- [ ] `tiktoken`: OpenAI's fast BPE tokenizer, used by GPT models
- [ ] Token count matters: context window limits, cost per token
- [ ] Special tokens: `[CLS]`, `[SEP]`, `<|endoftext|>`, `<s>`, `</s>`

### Embeddings
- [ ] Word2Vec: Skip-gram, CBOW — dense word representations
- [ ] GloVe: co-occurrence matrix factorization
- [ ] Limitation of static embeddings: one vector per word regardless of context
- [ ] Contextual embeddings: BERT/GPT produce different vectors for same word in different contexts
- [ ] Sentence embeddings: aggregate token embeddings, sentence-transformers
- [ ] Embedding dimensions: 768 (BERT-base), 1024, 1536, 3072 — tradeoffs
- [ ] Embedding models for RAG: see [[03 - RAG Architecture]]

### Pre-training Objectives
- [ ] Masked Language Modeling (MLM): BERT — predict masked tokens
- [ ] Next-token prediction (Causal LM): GPT — predict next token autoregressively
- [ ] Span corruption: T5 — mask spans, generate them
- [ ] Contrastive learning: CLIP — align image and text embeddings
- [ ] Why pre-training works: learn language structure, world knowledge, reasoning

### Transfer Learning
- [ ] Pre-train on large corpus → fine-tune on specific task
- [ ] Feature extraction: freeze model, use embeddings as features
- [ ] Fine-tuning: unfreeze model (or top layers), train on downstream task
- [ ] Few-shot learning: in-context examples without gradient updates (GPT)
- [ ] Zero-shot: no examples, just task description

### Hugging Face Transformers
- [ ] `AutoModel`, `AutoTokenizer` — load any model with one interface
- [ ] `pipeline()` — high-level API for common tasks (classification, QA, generation)
- [ ] Model Hub: browse 300K+ models, filter by task/language/size
- [ ] Tokenizer workflow: `tokenizer(text, return_tensors="pt")` → input_ids, attention_mask
- [ ] Inference: `model.generate()`, `model(**inputs)`, logits → predictions
- [ ] Datasets library: `load_dataset()`, streaming, map/filter
- [ ] Model card: documentation, intended use, limitations, biases

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] Hugging Face NLP Course: https://huggingface.co/learn/nlp-course
- [ ] Jay Alammar: "The Illustrated BERT" (blog)
- [ ] Stanford CS224N: NLP with Deep Learning (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain BPE tokenization — how does it work?
- What's the difference between BERT and GPT architectures?
- When would you fine-tune vs use zero-shot/few-shot?
