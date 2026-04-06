---
tags: [mlops, serving, phase-6]
phase: 6
status: not-started
priority: high
---

# 🚀 Model Serving & Deployment

> **Phase:** 6 | **Duration:** ~2 days | **Priority:** High
> **Related:** [[01 - MLOps Pipeline]], [[03 - LLMOps]], [[04 - Fine-Tuning]], [[01 - Backend FastAPI]]

---

## Checklist

### Serving Patterns
- [ ] Batch inference: run model on large datasets periodically (Spark, scheduled jobs)
- [ ] Real-time API: REST/gRPC endpoint, low-latency predictions
- [ ] Streaming inference: process events as they arrive (Spark Streaming + model)
- [ ] Embedded: model runs in client app (mobile, edge)
- [ ] Choosing: latency requirements, throughput, cost

### Model Serving Frameworks
- [ ] TorchServe: PyTorch-native, handler-based, metrics, batching
- [ ] Triton Inference Server (NVIDIA): multi-framework, GPU optimization, dynamic batching
- [ ] TensorFlow Serving: TF models, gRPC/REST, model versioning
- [ ] BentoML: Python-first, easy packaging, adaptive batching
- [ ] Ray Serve: distributed serving, scaling, composition

### LLM Serving
- [ ] vLLM: PagedAttention, continuous batching, fastest open-source LLM serving
- [ ] TGI (Text Generation Inference): Hugging Face, optimized for transformers
- [ ] Ollama: local LLM running, pull-and-run simplicity
- [ ] llama.cpp: C++ inference, CPU-optimized, GGUF format
- [ ] SGLang: structured generation, compiler-based optimization

### Quantization (Reduce Model Size)
- [ ] INT8 quantization: 4x memory reduction, minimal quality loss
- [ ] INT4 quantization: 8x reduction, more quality impact
- [ ] GPTQ: post-training quantization, GPU inference
- [ ] AWQ: activation-aware quantization, better quality than GPTQ
- [ ] GGUF: format for CPU inference (llama.cpp), variable quantization levels
- [ ] Quantization tradeoffs: memory savings vs accuracy vs speed

### GPU Memory Planning
- [ ] Model memory: parameters × dtype bytes (7B × 2 bytes FP16 = 14GB)
- [ ] Inference memory: model + KV-cache + activations
- [ ] KV-cache growth: proportional to batch_size × seq_length × num_layers
- [ ] Multi-GPU: tensor parallelism (split model), pipeline parallelism (split layers)
- [ ] GPU selection: A100 (80GB), H100 (80GB), L4 (24GB), T4 (16GB)

### Deployment Strategies
- [ ] A/B testing: route traffic between model versions, measure metrics
- [ ] Shadow deployment: new model runs alongside, results not served
- [ ] Canary release: gradual traffic shift (1% → 5% → 25% → 100%)
- [ ] Blue-green: instant switch between deployments
- [ ] Auto-scaling: HPA (Kubernetes), based on request volume/latency
- [ ] Cold start mitigation: min replicas, model warmup

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] vLLM docs: https://vllm.readthedocs.io/
- [ ] Triton Inference Server docs (NVIDIA)
- [ ] "Designing Machine Learning Systems" by Chip Huyen (Chapter 9)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- How would you deploy a 70B parameter LLM? What hardware do you need?
- Compare batch vs real-time inference — when to use each?
- Explain quantization and its tradeoffs
