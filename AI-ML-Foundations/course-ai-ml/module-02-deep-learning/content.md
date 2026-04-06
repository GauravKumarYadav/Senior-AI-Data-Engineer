# Module 2: Deep Learning Foundations

> **Duration:** ~4 hours | **Difficulty:** Intermediate-Advanced | **Prerequisites:** Module 1 (ML Fundamentals), basic calculus
> Understand the architectures powering modern AI — from neurons to Transformers.

---

## Screen 1: Neural Network Building Blocks

### The Artificial Neuron

A neuron computes a weighted sum of inputs, adds a bias, and passes the result through an activation function.

```
Inputs        Weights         Sum + Bias      Activation      Output
 x₁ ──── w₁ ──┐
                ├──→ Σ(wᵢxᵢ) + b ──→ f(z) ──→ a
 x₂ ──── w₂ ──┤
                │
 x₃ ──── w₃ ──┘

  z = w₁x₁ + w₂x₂ + w₃x₃ + b    (linear combination)
  a = f(z)                          (non-linear activation)
```

### Layers and Architecture

```
Input Layer      Hidden Layer 1     Hidden Layer 2     Output Layer
  (features)     (learned repr.)    (higher-level)     (prediction)

   ○ x₁ ─────┐    ○ h₁ ─────┐       ○ h₁' ─────┐      ○ ŷ₁
              ├────┤         ├───────┤           ├──────┤
   ○ x₂ ─────┤    ○ h₂ ─────┤       ○ h₂' ─────┤      ○ ŷ₂
              ├────┤         ├───────┤           │
   ○ x₃ ─────┘    ○ h₃ ─────┘       ○ h₃' ─────┘

   Each connection = a learnable weight
   Each node = weighted sum → activation function
```

### Activation Functions

| Function | Formula | Range | Use Case |
|---|---|---|---|
| ReLU | `max(0, x)` | [0, ∞) | Default for hidden layers |
| GELU | `x · Φ(x)` | (-0.17, ∞) | Transformers (smoother ReLU) |
| Sigmoid | `1 / (1 + e⁻ˣ)` | (0, 1) | Binary output, gates |
| Tanh | `(eˣ - e⁻ˣ)/(eˣ + e⁻ˣ)` | (-1, 1) | Centered output, RNNs |
| Softmax | `eˣⁱ / Σeˣʲ` | (0, 1), sums to 1 | Multi-class output layer |
| Swish | `x · sigmoid(x)` | (-0.28, ∞) | Deep networks, EfficientNet |

```python
import torch
import torch.nn as nn

# Common activations in PyTorch
relu = nn.ReLU()
gelu = nn.GELU()          # Used in BERT, GPT
sigmoid = nn.Sigmoid()     # Binary classification output
softmax = nn.Softmax(dim=-1)  # Multi-class output

# Why ReLU? Avoids vanishing gradient, computationally cheap
# Why GELU? Smooth approximation of ReLU, better for Transformers
```

### Loss Functions

```python
# Classification
cross_entropy = nn.CrossEntropyLoss()        # Multi-class (includes softmax)
bce_loss = nn.BCEWithLogitsLoss()            # Binary (includes sigmoid)

# Regression
mse_loss = nn.MSELoss()                       # Mean Squared Error
l1_loss = nn.L1Loss()                         # Mean Absolute Error
huber_loss = nn.HuberLoss(delta=1.0)          # Smooth L1 (robust to outliers)
```

### 💡 Interview Insight

> **"Why do we need activation functions?"**
> Without them, stacking layers is pointless — a composition of linear transformations is still linear (`W₂(W₁x + b₁) + b₂ = W'x + b'`). Activation functions introduce non-linearity, allowing the network to learn complex, non-linear decision boundaries. The universal approximation theorem states that a single hidden layer with non-linear activations can approximate any continuous function (given enough neurons).

---

## Screen 2: Backpropagation & Optimizers

### Backpropagation: How Networks Learn

Backpropagation computes gradients of the loss with respect to every weight using the **chain rule**, then updates weights to minimize the loss.

```
Forward Pass (compute predictions):
  Input → Layer 1 → Layer 2 → ... → Output → Loss

Backward Pass (compute gradients):
  ∂Loss/∂w = ∂Loss/∂output · ∂output/∂hidden · ∂hidden/∂w
  
  Loss ← ∂L/∂ŷ ← ∂ŷ/∂h₂ ← ∂h₂/∂h₁ ← ∂h₁/∂w

Update weights:
  w_new = w_old - learning_rate × ∂Loss/∂w
```

### Optimizers Compared

| Optimizer | Key Idea | When to Use |
|---|---|---|
| SGD | Basic gradient descent + momentum | CNNs, when you want fine control |
| Adam | Adaptive LR per parameter + momentum | Default choice, Transformers |
| AdamW | Adam with decoupled weight decay | Transformers, fine-tuning LLMs |
| SGD + Momentum | Accelerates in consistent directions | Image classification |

```python
import torch.optim as optim

# SGD with momentum
optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9, weight_decay=1e-4)

# Adam: adaptive learning rates (most popular)
optimizer = optim.Adam(model.parameters(), lr=1e-3, betas=(0.9, 0.999))

# AdamW: better weight decay (used for Transformers)
optimizer = optim.AdamW(model.parameters(), lr=5e-5, weight_decay=0.01)
```

### Learning Rate Schedules

```
Constant LR          Step Decay           Cosine Annealing      Warmup + Decay
  lr│───────          lr│──┐               lr│╲                  lr│  ╱──╲
    │                   │  └──┐              │  ╲                  │ ╱    ╲
    │                   │     └──┐           │   ╲                 │╱      ╲
    └──────→            └────────→           └────╲──→             └────────→
   epochs              epochs               epochs               epochs
```

```python
from torch.optim.lr_scheduler import CosineAnnealingLR, OneCycleLR

# Cosine annealing: smooth decay
scheduler = CosineAnnealingLR(optimizer, T_max=num_epochs, eta_min=1e-6)

# OneCycle: warmup → peak → decay (best for training from scratch)
scheduler = OneCycleLR(optimizer, max_lr=1e-3, total_steps=total_steps)

# In training loop:
for epoch in range(num_epochs):
    train_one_epoch(model, optimizer)
    scheduler.step()
```

### Batch Normalization vs Layer Normalization

```
Batch Norm (across batch):        Layer Norm (across features):

 Feature 1  Feature 2             Feature 1  Feature 2
 ┌───┐      ┌───┐                ┌───┐      ┌───┐
 │ a │      │ d │  Sample 1      │ a │ ←──→ │ d │  Normalize across features
 │ b │      │ e │  Sample 2      ├───┤      ├───┤
 │ c │      │ f │  Sample 3      │ b │ ←──→ │ e │  Each sample independently
 └───┘      └───┘                ├───┤      ├───┤
   ↕ normalize    ↕ normalize    │ c │ ←──→ │ f │
   across batch   across batch   └───┘      └───┘

BatchNorm: depends on batch       LayerNorm: independent of batch
→ Used in CNNs                    → Used in Transformers
→ Different behavior train/eval   → Same behavior train/eval
```

### 💡 Interview Insight

> **"Why is Adam the default optimizer for Transformers but SGD is still used for CNNs?"**
> Adam adapts the learning rate per parameter, which works well for Transformers where different attention heads and layers may need very different learning rates. SGD with momentum, while requiring careful LR tuning, often generalizes slightly better for CNNs and reaches flatter minima. AdamW (Adam with decoupled weight decay) is the gold standard for fine-tuning pre-trained Transformers.

---

## Screen 3: Dropout, Weight Init & Regularization in Deep Learning

### Dropout: The Elegant Regularizer

```
Training (dropout=0.5):              Inference (no dropout):

   ○ ──── ○ ──── ○                    ○ ──── ○ ──── ○
   ○ ──── ✗      ○    Randomly        ○ ──── ○ ──── ○
   ○      ○ ──── ✗    drop neurons    ○ ──── ○ ──── ○
   ○ ──── ✗ ──── ○    each forward    ○ ──── ○ ──── ○
                       pass
   Prevents co-adaptation             All neurons active
   Forces redundancy                  (weights scaled by 1-p)
```

```python
class MyModel(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, dropout=0.3):
        super().__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),        # Randomly zero 30% of neurons
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim, output_dim)
        )
    
    def forward(self, x):
        return self.network(x)

# CRITICAL: Switch modes for train vs eval
model.train()   # Dropout ACTIVE
model.eval()    # Dropout DISABLED (all neurons, scaled weights)
```

### Weight Initialization

Bad initialization → vanishing or exploding gradients → training fails.

| Method | Formula | Best With |
|---|---|---|
| Xavier/Glorot | `Var(w) = 2 / (fan_in + fan_out)` | Sigmoid, Tanh |
| He/Kaiming | `Var(w) = 2 / fan_in` | ReLU, LeakyReLU |
| Default PyTorch | Kaiming uniform for Linear | Most cases |

```python
# Custom initialization
def init_weights(module):
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, mode='fan_in', nonlinearity='relu')
        if module.bias is not None:
            nn.init.zeros_(module.bias)

model.apply(init_weights)
```

### 💡 Interview Insight

> **"Dropout is 0.1 in BERT but 0.5 in older networks. Why?"**
> BERT is pre-trained on massive data, so it's less prone to overfitting — light dropout (0.1) is sufficient. Older, smaller networks trained on less data need aggressive dropout (0.5) to regularize. During fine-tuning, you might increase dropout if your downstream dataset is small. The right dropout rate depends on model size, data size, and task complexity.

---

## Screen 4: CNN Basics — Convolutions, Pooling & ResNet

### How Convolutions Work

```
Input Image (5×5)         Filter/Kernel (3×3)        Output Feature Map (3×3)

┌───┬───┬───┬───┬───┐    ┌───┬───┬───┐
│ 1 │ 0 │ 1 │ 0 │ 1 │    │ 1 │ 0 │ 1 │              ┌───┬───┬───┐
├───┼───┼───┼───┼───┤    ├───┼───┼───┤              │ 4 │ 3 │ 4 │
│ 0 │ 1 │ 0 │ 1 │ 0 │ ★  │ 0 │ 1 │ 0 │  = Slide →  ├───┼───┼───┤
├───┼───┼───┼───┼───┤    ├───┼───┼───┤              │ 2 │ 4 │ 3 │
│ 1 │ 0 │ 1 │ 0 │ 1 │    │ 1 │ 0 │ 1 │              ├───┼───┼───┤
├───┼───┼───┼───┼───┤    └───┴───┴───┘              │ 4 │ 3 │ 4 │
│ 0 │ 1 │ 0 │ 1 │ 0 │                                └───┴───┴───┘
├───┼───┼───┼───┼───┤
│ 1 │ 0 │ 1 │ 0 │ 1 │    Element-wise multiply
└───┴───┴───┴───┴───┘    then sum → one output value

Key params: kernel_size, stride, padding, in_channels, out_channels
```

**Why CNNs?**
- **Parameter sharing:** Same filter applied across entire image (far fewer params than fully connected)
- **Translation invariance:** Detects patterns regardless of position
- **Spatial hierarchy:** Early layers detect edges, later layers detect objects

### Pooling

```
Max Pooling (2×2, stride 2):

┌───┬───┬───┬───┐         ┌───┬───┐
│ 1 │ 3 │ 2 │ 1 │         │ 4 │ 6 │   Takes the MAX
├───┼───┼───┼───┤   →     ├───┼───┤   from each 2×2 region
│ 4 │ 2 │ 6 │ 1 │         │ 8 │ 4 │   Reduces spatial dims by 2x
├───┼───┼───┼───┤         └───┴───┘
│ 1 │ 8 │ 3 │ 4 │
├───┼───┼───┼───┤
│ 5 │ 2 │ 1 │ 2 │
└───┴───┴───┴───┘
```

### ResNet: Skip Connections

The key innovation enabling very deep networks (50, 101, 152 layers):

```
Standard Block:                ResNet Block (Skip Connection):

  Input x                        Input x ─────────────────┐
    ↓                              ↓                       │
  Conv + BN + ReLU               Conv + BN + ReLU          │
    ↓                              ↓                       │
  Conv + BN                      Conv + BN                 │
    ↓                              ↓                       │
  ReLU                           + ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
    ↓                              ↓
  Output                         ReLU
                                   ↓
  Problem: gradients             Output = F(x) + x
  vanish in deep nets            
                                 "Learn the RESIDUAL F(x)"
                                 If F(x)=0 is optimal, just
                                 pass x through unchanged
```

```python
import torch.nn as nn
import torchvision.models as models

# Load pretrained ResNet
resnet = models.resnet50(weights='IMAGENET1K_V2')

# Modify for custom task (e.g., 10 classes)
resnet.fc = nn.Linear(resnet.fc.in_features, 10)
```

### 💡 Interview Insight

> **"Why are skip connections so important?"**
> They solve the degradation problem: deeper networks should perform at least as well as shallow ones (worst case, extra layers learn identity), but in practice gradients vanish and deeper nets train worse. Skip connections provide a gradient highway — gradients flow directly through the shortcut, enabling training of 100+ layer networks. This idea inspired residual connections in Transformers.

---

## Screen 5: The Transformer Architecture

### The Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSFORMER                               │
│                                                              │
│   ┌─────────────┐              ┌──────────────┐             │
│   │   ENCODER    │              │   DECODER     │             │
│   │             │              │              │             │
│   │ ┌─────────┐ │              │ ┌──────────┐ │             │
│   │ │Self-Attn│ │     K,V      │ │Masked     │ │             │
│   │ │         │ │──────────→   │ │Self-Attn  │ │             │
│   │ └────┬────┘ │              │ └─────┬────┘ │             │
│   │      ↓      │              │       ↓      │             │
│   │ ┌─────────┐ │              │ ┌──────────┐ │             │
│   │ │Feed-Fwd │ │              │ │Cross-Attn│ │             │
│   │ │Network  │ │              │ │(Q=dec,   │ │             │
│   │ └─────────┘ │              │ │ K,V=enc) │ │             │
│   │   × N layers│              │ └─────┬────┘ │             │
│   └─────────────┘              │       ↓      │             │
│                                │ ┌──────────┐ │             │
│   Encoder-only: BERT           │ │Feed-Fwd  │ │             │
│   (classification, embeddings) │ │Network   │ │             │
│                                │ └──────────┘ │             │
│   Decoder-only: GPT            │   × N layers│             │
│   (text generation)            └──────────────┘             │
│                                                              │
│   Encoder-Decoder: T5, BART                                  │
│   (translation, summarization)                               │
└─────────────────────────────────────────────────────────────┘
```

### Self-Attention: The Core Mechanism

**Intuition:** For each word, compute how much attention it should pay to every other word.

```
Input: "The cat sat on the mat"

Step 1: Create Q, K, V matrices from input embeddings
        Q = X · W_Q    (What am I looking for?)
        K = X · W_K    (What do I contain?)
        V = X · W_V    (What information do I provide?)

Step 2: Compute attention scores
        Scores = Q · K^T / √d_k     (dot product similarity)
        
Step 3: Apply softmax → attention weights
        Weights = softmax(Scores)

Step 4: Weighted sum of values
        Output = Weights · V

Formula: Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

**Why scale by √d_k?** Without scaling, dot products grow large for high dimensions, pushing softmax into regions with near-zero gradients (saturation). Dividing by √d_k keeps the variance of dot products at ~1.

### Multi-Head Attention

```
Instead of one attention function:

Head 1: Attention(Q·W₁ᵠ, K·W₁ᵏ, V·W₁ᵛ)  → "Focus on syntax"
Head 2: Attention(Q·W₂ᵠ, K·W₂ᵏ, V·W₂ᵛ)  → "Focus on semantics"
Head 3: Attention(Q·W₃ᵠ, K·W₃ᵏ, V·W₃ᵛ)  → "Focus on position"
...
Head h: Attention(Q·Wₕᵠ, K·Wₕᵏ, V·Wₕᵛ)  → "Focus on coreference"

Concatenate all heads → Linear projection → Output

d_model=768, h=12 heads → d_k = 768/12 = 64 per head
```

### Positional Encoding

Transformers process all tokens in parallel — they have no inherent notion of order. Positional encoding adds position information.

```
Original (sinusoidal):    PE(pos, 2i)   = sin(pos / 10000^(2i/d))
                          PE(pos, 2i+1) = cos(pos / 10000^(2i/d))

Learned (BERT):           Trainable embedding table, one vector per position

RoPE (modern, Llama):     Rotary Position Embedding — encodes relative position
                          through rotation in the embedding space
```

### 💡 Interview Insight

> **"Walk through a single Transformer encoder layer."**
> Input embeddings + positional encoding → (1) Multi-head self-attention: each token attends to all others, producing context-aware representations → Add & LayerNorm (residual connection) → (2) Feed-forward network: two linear layers with GELU activation, applied independently to each token → Add & LayerNorm → Output. The residual connections enable gradient flow in deep stacks. Each sub-layer has the pattern: `output = LayerNorm(x + Sublayer(x))`.

---

## Screen 6: Pre-Training Paradigms & Transfer Learning

### Pre-Training Objectives

```
Masked Language Model (BERT):
  Input:  "The [MASK] sat on the [MASK]"
  Target: "The  cat  sat on the  mat"
  → Bidirectional: sees context on BOTH sides
  → Good for: understanding, classification, embeddings

Causal Language Model (GPT):
  Input:  "The cat sat"
  Target: "cat sat on"
  → Autoregressive: only sees LEFT context
  → Good for: text generation, completion, chat

Span Corruption (T5):
  Input:  "The <X> on the mat"
  Target: "<X> cat sat <Y>"
  → Encoder-decoder: flexible input/output
  → Good for: translation, summarization, QA
```

### Transfer Learning Strategies

```
Strategy 1: Feature Extraction              Strategy 2: Fine-Tuning
┌──────────────────────┐                    ┌──────────────────────┐
│  Pre-trained Model   │ ← FROZEN           │  Pre-trained Model   │ ← UNFROZEN
│  (BERT, ResNet)      │   (no gradients)   │  (BERT, ResNet)      │   (small LR)
├──────────────────────┤                    ├──────────────────────┤
│  New Classification  │ ← TRAINED          │  New Classification  │ ← TRAINED
│  Head                │                    │  Head                │
└──────────────────────┘                    └──────────────────────┘

Use when:                                   Use when:
- Very small dataset                        - Moderate+ dataset
- Task similar to pre-training              - Task differs from pre-training
- Compute-limited                           - Need best performance

Strategy 3: Gradual Unfreezing
Epoch 1: Train only new head
Epoch 2: Unfreeze last 2 layers
Epoch 3: Unfreeze last 4 layers
...until all unfrozen (with tiny LR for early layers)
```

```python
from transformers import AutoModelForSequenceClassification

# Fine-tuning BERT for classification
model = AutoModelForSequenceClassification.from_pretrained(
    'bert-base-uncased', num_labels=3
)

# Feature extraction: freeze everything except the classifier
for param in model.bert.parameters():
    param.requires_grad = False
# Only the classification head trains

# Fine-tuning: unfreeze all (use smaller LR for pre-trained layers)
for param in model.bert.parameters():
    param.requires_grad = True

# Differential learning rates
optimizer = optim.AdamW([
    {'params': model.bert.parameters(), 'lr': 2e-5},       # Pre-trained: tiny LR
    {'params': model.classifier.parameters(), 'lr': 1e-3}, # New head: normal LR
])
```

### 💡 Interview Insight

> **"When would you fine-tune vs use zero-shot/few-shot?"**
> Fine-tune when: you have labeled data (hundreds+), need high accuracy, task is specialized. Zero/few-shot when: no labeled data, rapid prototyping, task is general (sentiment, summarization). Fine-tuning a small model often beats few-shot with a large model and is cheaper at inference time. The trend is moving toward efficient fine-tuning (LoRA, QLoRA) that gets fine-tuning quality with few-shot cost.

---

## Screen 7: PyTorch Essentials

### Tensors & GPU

```python
import torch

# Create tensors
x = torch.tensor([1.0, 2.0, 3.0])
X = torch.randn(32, 784)           # Random normal (batch of 32, 784 features)
zeros = torch.zeros(3, 3)
ones = torch.ones(5, dtype=torch.float32)

# GPU transfer
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
x = x.to(device)
# Or: x = x.cuda()

# Operations (just like NumPy)
y = x @ W.T + b                    # Matrix multiply
z = torch.softmax(y, dim=-1)       # Along last dimension
```

### Autograd: Automatic Differentiation

```python
# Autograd tracks operations for gradient computation
x = torch.tensor([2.0, 3.0], requires_grad=True)
y = x ** 2 + 3 * x                 # y = x² + 3x
loss = y.sum()
loss.backward()                     # Compute gradients
print(x.grad)                       # dy/dx = 2x + 3 → [7.0, 9.0]

# Disable gradient tracking (inference, validation)
with torch.no_grad():
    predictions = model(test_data)
```

### Building Models with nn.Module

```python
import torch.nn as nn

class TextClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, num_classes):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.encoder = nn.Sequential(
            nn.Linear(embed_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.3),
        )
        self.classifier = nn.Linear(hidden_dim, num_classes)
    
    def forward(self, input_ids):
        # input_ids: (batch_size, seq_len)
        embeds = self.embedding(input_ids)       # (batch, seq, embed_dim)
        pooled = embeds.mean(dim=1)              # (batch, embed_dim)
        hidden = self.encoder(pooled)            # (batch, hidden_dim)
        logits = self.classifier(hidden)         # (batch, num_classes)
        return logits

model = TextClassifier(vocab_size=30000, embed_dim=128, hidden_dim=256, num_classes=5)
print(f"Parameters: {sum(p.numel() for p in model.parameters()):,}")
```

### The Training Loop

```python
from torch.utils.data import DataLoader, TensorDataset

# Prepare data
dataset = TensorDataset(X_train_tensor, y_train_tensor)
dataloader = DataLoader(dataset, batch_size=32, shuffle=True, num_workers=4)

# Training loop
model = TextClassifier(30000, 128, 256, 5).to(device)
optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
criterion = nn.CrossEntropyLoss()
scheduler = CosineAnnealingLR(optimizer, T_max=10)

for epoch in range(10):
    model.train()
    total_loss = 0
    
    for batch_X, batch_y in dataloader:
        batch_X, batch_y = batch_X.to(device), batch_y.to(device)
        
        # Forward pass
        logits = model(batch_X)
        loss = criterion(logits, batch_y)
        
        # Backward pass
        optimizer.zero_grad()     # Clear previous gradients
        loss.backward()           # Compute gradients
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()          # Update weights
        
        total_loss += loss.item()
    
    scheduler.step()
    avg_loss = total_loss / len(dataloader)
    print(f"Epoch {epoch+1}: Loss = {avg_loss:.4f}, LR = {scheduler.get_last_lr()[0]:.6f}")
```

### Saving & Loading

```python
# Save (only state_dict, not entire model)
torch.save(model.state_dict(), 'model.pth')

# Load
model = TextClassifier(30000, 128, 256, 5)
model.load_state_dict(torch.load('model.pth', map_location=device))
model.eval()
```

### 💡 Interview Insight

> **"Why `optimizer.zero_grad()` before `loss.backward()`?"**
> PyTorch accumulates gradients by default (useful for gradient accumulation with large effective batch sizes). If you don't zero them, gradients from the previous batch add to the current ones, leading to incorrect updates. The pattern is always: zero_grad → forward → loss → backward → step. Some prefer `optimizer.zero_grad(set_to_none=True)` for a small performance gain.

---

## Screen 8: Mixed Precision, Distributed Training & Practical Tips

### Mixed Precision Training

Uses float16 for forward/backward (faster, less memory) and float32 for weight updates (numerical stability).

```python
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch_X, batch_y in dataloader:
    optimizer.zero_grad()
    
    # Forward pass in mixed precision
    with autocast():
        logits = model(batch_X)
        loss = criterion(logits, batch_y)
    
    # Backward pass with gradient scaling
    scaler.scale(loss).backward()
    scaler.unscale_(optimizer)
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
    scaler.step(optimizer)
    scaler.update()
```

**Benefits:** ~2x faster training, ~50% less GPU memory → bigger batch sizes or bigger models.

### Distributed Training (Awareness)

```
DataParallel (DP):               DistributedDataParallel (DDP):
┌────────────┐                   ┌──────┐  ┌──────┐  ┌──────┐
│  GPU 0     │                   │GPU 0 │  │GPU 1 │  │GPU 2 │
│ (master)   │                   │Model │  │Model │  │Model │
│ ┌────────┐ │                   │Copy  │  │Copy  │  │Copy  │
│ │Gather  │ │                   └──┬───┘  └──┬───┘  └──┬───┘
│ │gradients│ │                      │         │         │
│ └────────┘ │                      └────┬────┘─────────┘
│  GPU 1,2,3 │                        AllReduce (NCCL)
│  replicas  │                     Average gradients across all
└────────────┘                   
Single-process, GIL bottleneck   Multi-process, scales linearly
→ DON'T USE                      → USE THIS
```

```python
# DDP setup (simplified)
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

dist.init_process_group(backend='nccl')
model = DDP(model.to(local_rank), device_ids=[local_rank])
```

### Practical Training Tips

| Problem | Solution |
|---|---|
| Loss not decreasing | Lower LR, check data loading, verify labels |
| Loss spikes | Gradient clipping, lower LR, check data |
| NaN loss | Check for log(0), div by 0, reduce LR |
| Overfitting | Dropout, weight decay, more data, early stop |
| Underfitting | Bigger model, lower regularization, more epochs |
| Slow training | Mixed precision, larger batch, profile bottlenecks |
| OOM (Out of Memory) | Smaller batch, gradient accumulation, mixed precision |

### 💡 Interview Insight

> **"How would you train a model that doesn't fit in GPU memory?"**
> (1) Gradient accumulation: simulate large batch by accumulating gradients over N small batches before stepping. (2) Mixed precision: cut memory ~50%. (3) Gradient checkpointing: trade compute for memory by recomputing activations during backward pass. (4) Model parallelism: split model across GPUs. (5) DeepSpeed / FSDP: automated model sharding for very large models. Start with the simplest solution first.

---

## Screen 9: Quiz & Key Takeaways

### 🧪 Quiz

**Q1:** Why does GELU replace ReLU in Transformers?

<details><summary>Answer</summary>GELU (Gaussian Error Linear Unit) provides a smooth, differentiable approximation of ReLU. Unlike ReLU's hard zero cutoff, GELU allows small negative values to pass through with small probability, which helps with gradient flow. It empirically performs better in attention-based architectures. The smoothness also aids optimization.</details>

**Q2:** In the attention formula `softmax(QK^T / √d_k) · V`, what happens if we remove the `√d_k` scaling?

<details><summary>Answer</summary>Without scaling, dot products grow proportionally with dimension d_k. For large d_k, some softmax inputs become very large, pushing outputs toward one-hot vectors (saturation). Gradients become near-zero in these saturated regions, making training extremely slow or unstable. Dividing by √d_k normalizes the variance of dot products to ~1.</details>

**Q3:** You're fine-tuning BERT on a dataset with only 500 labeled examples. What precautions do you take?

<details><summary>Answer</summary>(1) Use a small learning rate (2e-5 to 5e-5) to avoid catastrophic forgetting. (2) Keep dropout or even increase it. (3) Use early stopping based on validation loss. (4) Consider freezing most layers and only fine-tuning the last 2-3 layers + classification head. (5) Use data augmentation if applicable. (6) Consider few-shot approaches or gradual unfreezing.</details>

**Q4:** What's the difference between `model.train()` and `model.eval()` in PyTorch?

<details><summary>Answer</summary>`model.train()` enables training-specific behaviors: Dropout randomly zeros neurons, BatchNorm uses batch statistics. `model.eval()` disables these: Dropout is off (all neurons active, weights scaled), BatchNorm uses running mean/variance from training. Always call `.eval()` before inference and wrap in `torch.no_grad()` to save memory.</details>

**Q5:** Explain the purpose of residual connections in Transformers.

<details><summary>Answer</summary>Residual connections (output = x + Sublayer(x)) serve two purposes: (1) Enable gradient flow through deep networks by providing a shortcut path for gradients during backpropagation, preventing vanishing gradients. (2) Allow layers to learn refinements (residuals) rather than complete transformations, making optimization easier. Combined with LayerNorm, they enable stacking 12-96+ identical layers.</details>

### 🎯 Key Takeaways

1. **Activation functions are non-negotiable** — they give networks the power to learn non-linear patterns
2. **Backpropagation + chain rule** is how every neural network learns — understand it deeply
3. **Adam/AdamW is the default optimizer** for most modern architectures; SGD still has its place for CNNs
4. **Dropout, weight decay, and early stopping** are your regularization toolkit for deep learning
5. **ResNet's skip connections** revolutionized deep networks and directly inspired Transformer design
6. **Self-attention** is the core of Transformers — understand Q, K, V and the scaling factor
7. **Multi-head attention** lets the model attend to different types of relationships simultaneously
8. **Pre-training + fine-tuning** is the dominant paradigm — massive unsupervised pretraining, then task-specific adaptation
9. **PyTorch training loop**: zero_grad → forward → loss → backward → clip → step — memorize this pattern
10. **Mixed precision + DDP** are essential for efficient training at scale
