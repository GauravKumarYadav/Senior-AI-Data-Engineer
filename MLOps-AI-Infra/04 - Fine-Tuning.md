---
tags: [ai, fine-tuning, phase-6]
phase: 6
status: not-started
priority: medium
---

# 🎛️ Fine-Tuning LLMs

> **Phase:** 6 | **Duration:** ~3 days | **Priority:** Medium
> **Related:** [[01 - LLM Fundamentals]], [[02 - Deep Learning Foundations]], [[02 - Model Serving]], [[01 - MLOps Pipeline]]

---

## Checklist

### When to Fine-Tune (Decision Framework)
- [ ] Prompt engineering first → RAG next → fine-tuning last
- [ ] Fine-tune when: specific output style, domain jargon, consistent format
- [ ] Don't fine-tune when: need factual knowledge (use RAG), small dataset
- [ ] Combine: fine-tune base model + RAG for retrieval (best of both)
- [ ] Cost-benefit: training cost + ongoing serving vs API costs

### Parameter-Efficient Fine-Tuning (PEFT)
- [ ] LoRA (Low-Rank Adaptation):
  - [ ] Concept: inject low-rank matrices into attention layers
  - [ ] Train only LoRA weights (~0.1-1% of model parameters)
  - [ ] `r` (rank): 8-64, higher = more capacity but more compute
  - [ ] `alpha`: scaling factor, typically 2× rank
  - [ ] `target_modules`: which layers to adapt (q_proj, v_proj, etc.)
- [ ] QLoRA: quantize base model to 4-bit + LoRA on top
  - [ ] NF4 quantization + double quantization
  - [ ] Fine-tune 70B model on single GPU (A100 48GB)
- [ ] DoRA: weight-decomposed LoRA (magnitude + direction)
- [ ] Full fine-tuning: update all parameters (only for small models or huge budgets)

### Dataset Preparation
- [ ] Instruction tuning format: `{"instruction": ..., "input": ..., "output": ...}`
- [ ] Chat format: `[{"role": "user", "content": ...}, {"role": "assistant", "content": ...}]`
- [ ] Alpaca format, ShareGPT format, ChatML format
- [ ] Dataset size: 1K-100K examples typically sufficient for LoRA
- [ ] Quality > quantity: well-curated examples matter more
- [ ] Data augmentation: generate synthetic training data with stronger model
- [ ] Filtering: remove duplicates, low-quality, harmful examples
- [ ] Train/eval split: hold out evaluation set for quality assessment

### Training with HuggingFace
- [ ] `transformers`: model loading, `Trainer` class
- [ ] `peft`: LoRA configuration, `get_peft_model()`
- [ ] `trl` (Transformer Reinforcement Learning):
  - [ ] `SFTTrainer`: supervised fine-tuning
  - [ ] `DPOTrainer`: Direct Preference Optimization (simpler than RLHF)
- [ ] `bitsandbytes`: 4-bit/8-bit quantization for QLoRA
- [ ] Training config: learning rate (1e-4 to 2e-5), epochs (1-3), batch size
- [ ] `wandb` / `mlflow` integration for experiment tracking
- [ ] DeepSpeed / FSDP: distributed training for larger models

### Alignment (RLHF & DPO)
- [ ] RLHF: reward model + PPO optimization (complex, unstable)
- [ ] DPO (Direct Preference Optimization): simpler, no reward model needed
  - [ ] Input: pairs of (preferred, rejected) responses
  - [ ] Directly optimizes model to prefer good outputs
- [ ] ORPO, SimPO: newer alignment methods, even simpler
- [ ] When to align: safety, helpfulness, specific tone/style

### Evaluation of Fine-Tuned Models
- [ ] Perplexity: how well model predicts held-out text (lower = better)
- [ ] Task-specific metrics: accuracy, F1, ROUGE for specific downstream tasks
- [ ] LLM-as-judge: compare fine-tuned vs base model on test prompts
- [ ] Human evaluation: blind comparison, preference ranking
- [ ] Regression testing: ensure fine-tuning didn't hurt general capabilities
- [ ] Benchmarks: MMLU, HellaSwag, HumanEval (coding) — check for degradation

### Serving Fine-Tuned Models
- [ ] Merge LoRA weights into base model: `merge_and_unload()`
- [ ] Export formats: safetensors, GGUF (for llama.cpp)
- [ ] Serve with vLLM: load merged model or serve with LoRA adapters
- [ ] Serve with TGI: similar to vLLM, Hugging Face ecosystem
- [ ] Ollama: create custom Modelfile from fine-tuned weights
- [ ] Multiple LoRA adapters: serve one base model with swappable LoRAs

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] Hugging Face PEFT docs: https://huggingface.co/docs/peft
- [ ] TRL docs: https://huggingface.co/docs/trl
- [ ] Sebastian Raschka: "LLM Fine-Tuning" (blog series)
- [ ] Philipp Schmid: fine-tuning tutorials (Hugging Face blog)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- When would you fine-tune vs use RAG vs prompt engineering?
- Explain LoRA — how does it work and why is it efficient?
- How do you prepare a dataset for instruction tuning?
- What is DPO and how does it compare to RLHF?
