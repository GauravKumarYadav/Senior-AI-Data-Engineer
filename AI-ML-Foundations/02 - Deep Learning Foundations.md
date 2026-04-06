---
tags: [ai, deep-learning, phase-4]
phase: 4
status: not-started
priority: high
---

# 🧠 Deep Learning Foundations

> **Phase:** 4 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[01 - ML Fundamentals]], [[03 - NLP for AI Engineers]], [[01 - LLM Fundamentals]], [[04 - Fine-Tuning]]

---

## Checklist

### Neural Network Building Blocks
- [ ] Linear layers (fully connected): `y = Wx + b`, weight matrices
- [ ] Activation functions: ReLU, GELU, Sigmoid, Tanh, Swish — when to use each
- [ ] Loss functions: CrossEntropy (classification), MSE (regression), BCE (binary)
- [ ] Backpropagation: chain rule, computing gradients through the network
- [ ] Gradient descent: SGD, Adam, AdamW — optimizer comparison
- [ ] Learning rate: schedules (cosine, warmup+decay), learning rate finder
- [ ] Batch normalization, Layer normalization — stabilizing training
- [ ] Dropout: regularization, training vs inference mode
- [ ] Weight initialization: Xavier/Glorot, He, importance for deep networks

### PyTorch Essentials
- [ ] Tensors: creation, operations, GPU transfer (`.to(device)`)
- [ ] Autograd: `requires_grad`, `backward()`, `torch.no_grad()`
- [ ] `nn.Module`: defining models, `forward()`, `parameters()`
- [ ] `DataLoader`: batching, shuffling, `num_workers`, custom `Dataset`
- [ ] Training loop: forward → loss → backward → optimizer.step() → zero_grad
- [ ] Saving/loading: `state_dict`, `torch.save`, `torch.load`
- [ ] Mixed precision: `torch.cuda.amp`, `autocast`, `GradScaler`
- [ ] Distributed training basics: `DataParallel`, `DistributedDataParallel`

### CNN Basics (Awareness)
- [ ] Convolution operation: filters, feature maps, stride, padding
- [ ] Pooling: max pooling, average pooling — dimensionality reduction
- [ ] Architecture evolution: LeNet → AlexNet → VGG → ResNet (skip connections)
- [ ] Why CNNs: spatial hierarchy, parameter sharing, translation invariance
- [ ] Use in vision: image classification, object detection, segmentation

### RNN/LSTM Basics (Historical Context)
- [ ] RNN: sequential processing, hidden state, vanishing gradient problem
- [ ] LSTM: gates (forget, input, output), long-range dependencies
- [ ] GRU: simplified LSTM, fewer parameters
- [ ] Why they were replaced: attention mechanism → Transformers
- [ ] Still used: some time-series, speech recognition edge cases

### Attention Mechanism (Critical!)
- [ ] Intuition: "which parts of input to focus on for each output"
- [ ] Self-attention: Query, Key, Value matrices from same input
- [ ] Attention score: $\text{Attention}(Q,K,V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$
- [ ] Multi-head attention: multiple attention heads, concatenate, project
- [ ] Why $\sqrt{d_k}$ scaling: prevent softmax saturation for large dimensions
- [ ] Cross-attention: Q from decoder, K/V from encoder (used in encoder-decoder)

### Transformer Architecture
- [ ] Original paper: "Attention Is All You Need" (2017)
- [ ] Encoder: self-attention + feed-forward, residual connections, layer norm
- [ ] Decoder: masked self-attention + cross-attention + feed-forward
- [ ] Positional encoding: sinusoidal (original), learned (BERT), RoPE (modern)
- [ ] Encoder-only: BERT (bidirectional, good for classification, embeddings)
- [ ] Decoder-only: GPT (causal/autoregressive, good for generation)
- [ ] Encoder-decoder: T5, BART (good for translation, summarization)
- [ ] Feed-forward network: two linear layers with GELU activation
- [ ] Residual connections + LayerNorm: enable deep stacking

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Attention Is All You Need" paper (annotated version: harvardnlp.github.io)
- [ ] Andrej Karpathy: "Neural Networks: Zero to Hero" (YouTube series)
- [ ] 3Blue1Brown: Neural Networks series (visual intuition)
- [ ] "Deep Learning" by Goodfellow, Bengio, Courville (Chapters 6-10)
- [ ] Jay Alammar: "The Illustrated Transformer" (blog)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain the self-attention mechanism in your own words
- Why do Transformers outperform RNNs for sequence tasks?
- Walk through a single Transformer encoder layer
- What is the purpose of multi-head attention?
