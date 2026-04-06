# Module 1: LLM Fundamentals

> **Scenario**: You are the lead ML engineer at ShopAssist Inc., building an AI-powered customer support system for a large e-commerce platform. Throughout this module, every concept is grounded in real decisions you'll face — from choosing a model to configuring inference parameters for production.

---

## Screen 1: GPT Decoder-Only Architecture

Modern large language models — GPT-4, Claude, Llama 3 — share a common ancestor: the **decoder-only Transformer**. Understanding this architecture is non-negotiable for anyone building on top of these models.

### The Core Idea: Next-Token Prediction

A decoder-only model does exactly one thing during training: given a sequence of tokens, predict the next token. That's it. Every stunning capability — writing poetry, debugging code, answering customer queries — emerges from this deceptively simple objective.

```
Input:  "The customer wants to return their"
Output: " order"  (predicted next token)
```

### Causal Self-Attention

The key mechanism is **causal (masked) self-attention**. Unlike the encoder in the original Transformer (which sees the full sequence bidirectionally), a decoder masks future tokens so position `i` can only attend to positions `0..i`. This enforces the autoregressive property.

```
                    Causal Attention Mask
            ┌─────────────────────────────┐
            │  The  cust  wants  to  return│
  The       │   1     0     0    0     0   │
  cust      │   1     1     0    0     0   │
  wants     │   1     1     1    0     0   │
  to        │   1     1     1    1     0   │
  return    │   1     1     1    1     1   │
            └─────────────────────────────┘
              1 = can attend, 0 = masked
```

Each attention head computes Query, Key, Value projections: `Attention(Q, K, V) = softmax(QK^T / √d_k) · V`. Multi-head attention runs this in parallel across `h` heads, each learning different relationship patterns (syntactic, semantic, positional).

### Autoregressive Generation

At inference time, generation proceeds one token at a time:

```python
import openai

# For our customer support system, autoregressive generation means
# every token is conditioned on ALL previous tokens
def generate_support_response(client: openai.OpenAI, query: str) -> str:
    """Each token is generated sequentially — the model 'writes' left to right."""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a helpful e-commerce support agent."},
            {"role": "user", "content": query},
        ],
        stream=True,  # Streamhey're generated
    )
    full_response = ""
    for chunk in response:
        token = chunk.choices[0].delta.content or ""
        full_response += token
        print(token, end="", flush=True)  # Real-time streaming to the user
    return full_response
```

### Architecture Stack

```
┌─────────────────────────────────────┐
│         Output Probabilities        │  softmax over vocabulary (~100K tokens)
├─────────────────────────────────────┤
│      Linear Projection (LM Head)   │
├─────────────────────────────────────┤
│     Transformer Block × N Layers   │  GPT-4: ~120 layers (estimated)
│  ┌───────────────────────────────┐  │
│  │  Layer Norm                   │  │
│  │  Causal Multi-Head Attention  │  │  h=96 heads, d_model=12288
│  │  Residual Connection          │  │
│  │  Layer Norm                   │  │
│  │  Feed-Forward Network (MLP)   │  │  4× expansion: 12288→49152→12288
│  │  Residual Connection          │  │
│  └───────────────────────────────┘  │
├─────────────────────────────────────┤
│  Token Embedding + Positional Enc   │  RoPE (Rotary Position Embedding)
├─────────────────────────────────────┤
│         Input Token IDs             │
└─────────────────────────────────────┘
```

Modern models use **RoPE** (Rotary Position Embeddings) instead of learned positional embeddings, enabling better length generalization. They also use **SwiGLU** activation in the MLP and **RMSNorm** instead of LayerNorm.

### 💡 Interview Insight

> **Q: Why do modern LLMs use decoder-only architectures instead of encoder-decoder?**
>
> "Decoder-only architectures won out for three reasons. First, simplicity — you have one unified model for both understanding and generation, which makes scaling straightforward. Second, decoder-only models handle the full spectrum from classification to open-ended generation without architectural changes. Third, and most importantly, they're easier to scale via next-token prediction on massive unsupervised corpora. Encoder-decoder models like T5 require constructing explicit input-output pairs. Google's PaLM, Meta's Llama, and OpenAI's GPT series all converged on decoder-only because it provides the best capability-per-FLOP ratio at scale."

---

## Screen 2: Pre-training → SFT → RLHF/DPO Alignment Pipeline

Raw pre-training produces a model that completes text — not one that follows instructions helpfully and safely. The **alignment pipeline** transforms a base model into a useful assistant.

### The Three-Stage Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    LLM ALIGNMENT PIPELINE                            │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────┐  │
│  │  STAGE 1    │    │  STAGE 2    │    │  STAGE 3                 │  │
│  │             │    │             │    │                          │  │
│  │ Pre-train   │───▶│    SFT      │───▶│  RLHF  or  DPO          │  │
│  │             │    │             │    │                          │  │
│  │ Next-token  │    │ Instruction │    │ Preference               │  │
│  │ prediction  │    │ tuning on   │    │ optimization             │  │
│  │ on ~15T     │    │ ~100K high  │    │ from human               │  │
│  │ tokens      │    │ quality     │    │ rankings                 │  │
│  │             │    │ examples    │    │                          │  │
│  │ Cost: $5M+  │    │ Cost: $50K  │    │ Cost: $200K+             │  │
│  └─────────────┘    └─────────────┘    └──────────────────────────┘  │
│                                                                      │
│  Capability ────────────────────────────────────────────▶            │
│  Safety & Helpfulness ──────────────────────────────────▶            │
└──────────────────────────────────────────────────────────────────────┘
```

**Stage 1 — Pre-training**: The model learns language, world knowledge, and reasoning by predicting next tokens on trillions of tokens from the internet, books, code, and curated datasets. This is the most expensive stage (~$5–100M compute).

**Stage 2 — Supervised Fine-Tuning (SFT)**: The pre-trained model is fine-tuned on curated (instruction, response) pairs. These are high-quality demonstrations of the desired behavior. For our customer support system, this would be exemplary agent-customer conversations.

**Stage 3 — RLHF or DPO**: Human annotators rank model outputs from best to worst. In **RLHF**, a reward model is trained on these preferences, then Proximal Policy Optimization (PPO) optimizes the LLM against that reward model. **DPO** (Direct Preference Optimization) skips the reward model entirely and directly optimizes the LLM from preference pairs — it's simpler, cheaper, and increasingly preferred.

```python
# Conceptual DPO loss — directly optimize from preference pairs
# preferred response y_w vs dispreferred y_l for prompt x
def dpo_loss(policy, reference, preferred, dispreferred, beta=0.1):
    """
    DPO eliminates the reward model entirely.
    Loss = -log σ(β · (log π(y_w|x)/π_ref(y_w|x) - log π(y_l|x)/π_ref(y_l|x)))
    """
    log_ratio_w = policy.log_prob(preferred) - reference.log_prob(preferred)
    log_ratio_l = policy.log_prob(dispreferred) - reference.log_prob(dispreferred)
    return -torch.log(torch.sigmoid(beta * (log_ratio_w - log_ratio_l))).mean()
```

### 💡 Interview Insight

> **Q: When would you use DPO over RLHF for aligning a customer support model?**
>
> "DPO is my default choice for most alignment tasks, including customer support. It eliminates the reward model training step and PPO's notorious instability — you just need pairs of 'better' and 'worse' responses. For our support system, we'd collect agent ratings on model outputs, create preference pairs, and run DPO directly. RLHF with PPO is still relevant when you need fine-grained reward shaping — like penalizing specific failure modes with different weights — or when you have a high-quality reward model you want to iterate against. But for most production teams, DPO gets you 90% of the benefit at 30% of the complexity."

---

## Screen 3: KV-Cache — Inference Speed & Memory Tradeoffs

When your customer support system handles thousands of concurrent conversations, inference speed becomes a production concern. The **KV-cache** is the single most important optimization to understand.

### The Problem Without KV-Cache

In autoregressive generation, to produce token `t_n`, the model computes attention over all previous tokens `t_1...t_{n-1}`. Without caching, generating the next token requires recomputing Q, K, V for **every** previous token — O(n²) per token, O(n³) for the full sequence.

### How KV-Cache Works

The insight: when generating token `t_n`, the Key and Value vectors for tokens `t_1...t_{n-1}` haven't changed. We cache them and only compute K, V for the new token.

```
Without KV-Cache (generating token 5):
  Compute Q,K,V for tokens 1,2,3,4,5 → attend → output token 5
  
  Generating token 6:
  Compute Q,K,V for tokens 1,2,3,4,5,6 → attend → output token 6  ← REDUNDANT!

With KV-Cache:
  Token 5: Compute Q5, K5, V5. Cache K5,V5. Attend Q5 to [K1..K5] → output
  Token 6: Compute Q6, K6, V6. Cache K6,V6. Attend Q6 to [K1..K6] → output
                                                         ↑ cached!
```

### Memory Cost

KV-cache memory scales as: `2 × n_layers × n_heads × d_head × seq_len × batch_size × bytes_per_param`

For a 70B model (80 layers, 64 heads, d_head=128) with FP16 at 4K context:
```
2 × 80 × 64 × 128 × 4096 × 2 bytes = ~10.7 GB per request
```

This is why long-context models (128K+ tokens) are so memory-hungry. Techniques like **Multi-Query Attention (MQA)** and **Grouped-Query Attention (GQA)** share K,V heads across query heads, reducing KV-cache size by 4–8×.

```
Standard MHA:  64 Q heads, 64 K heads, 64 V heads  → full KV-cache
GQA (Llama 3): 64 Q heads,  8 K heads,  8 V heads  → 8× smaller KV-cache
MQA:           64 Q heads,  1 K head,   1 V head   → 64× smaller KV-cache
```

For our customer support system handling 100 concurrent conversations at 4K context, the KV-cache alone could require **1 TB of GPU memory** with standard MHA on a 70B model — this is why GQA and model selection matter enormously.

### 💡 Interview Insight

> **Q: How would you optimize inference costs for a high-throughput customer support system?**
>
> "I'd focus on three layers. First, model selection — use a GQA model like Llama 3 that reduces KV-cache by 8× compared to standard multi-head attention. Second, batching strategy — continuous batching (as in vLLM) packs requests dynamically so GPU utilization stays above 80%, versus naive batching where short requests wait for long ones. Third, KV-cache management — techniques like PagedAttention in vLLM allocate KV-cache memory in non-contiguous blocks, eliminating fragmentation waste. Together these can reduce per-query cost by 3-5× compared to naive serving."

---

## Screen 4: Chinchilla Scaling Laws & Compute-Optimal Training

When deciding which model to use — or whether to train your own — understanding **scaling laws** tells you how to allocate your compute budget.

### The Chinchilla Insight

DeepMind's Chinchilla paper (2022) showed that most LLMs were **over-parameterized and under-trained**. The compute-optimal ratio is roughly:

```
Optimal tokens ≈ 20 × parameters
```

| Model | Parameters | Training Tokens | Chinchilla Optimal? |
|---|---|---|---|
| GPT-3 | 175B | 300B | ❌ Under-trained (needs ~3.5T) |
| Chinchilla | 70B | 1.4T | ✅ Optimal |
| Llama 2 70B | 70B | 2T | ✅ Over-trained (better inference) |
| Llama 3 8B | 8B | 15T | ✅ Deliberately over-trained |

### The Post-Chinchilla Shift

In practice, teams now deliberately **over-train smaller models** beyond the Chinchilla optimum. Why? Training compute is a one-time cost, but inference compute is ongoing. A smaller model trained on more data gives better performance-per-FLOP at inference time.

This is exactly what Meta did with Llama 3 8B (15T tokens for an 8B model — nearly 1900× the parameters). For our customer support system, this means a well-trained 8B model can match a 70B model's quality while being 8× cheaper to serve.

### Practical Implications

```
Decision Framework for ShopAssist:
                                                          
  Budget: $10K/month inference    ──▶  Use Llama 3 8B / Mistral 7B
  Budget: $50K/month inference    ──▶  Use GPT-4o-mini / Claude 3.5 Haiku
  Budget: $200K/month inference   ──▶  Use GPT-4o / Claude 3.5 Sonnet
  Need custom domain knowledge    ──▶  Fine-tune a small model on domain data
  Need maximum quality, any cost  ──▶  GPT-4o / Claude 3.5 Opus
```

### 💡 Interview Insight

> **Q: How do scaling laws influence your model selection for production?**
>
> "Scaling laws tell us that for inference-heavy workloads like customer support, we should prefer smaller models trained on more data. The Chinchilla insight shifted the industry: instead of throwing more parameters at the problem, train smaller models longer. For our support system handling millions of queries daily, a Llama 3 8B model trained on 15 trillion tokens gives us 80-90% of GPT-4's quality at 1/50th the inference cost. I'd start with a small over-trained model, measure quality gaps on our specific use cases, and only upgrade to a larger model for the tasks where the small model fails."

---

## Screen 5: Emergent Abilities & the Model Landscape

### Emergent Abilities

As models scale, they develop capabilities that weren't explicitly trained:

**In-Context Learning (ICL)**: The model learns new tasks from examples in the prompt — no gradient updates needed. For our support system, we can teach the model new return policies just by including examples in the prompt.

**Chain-of-Thought (CoT)**: Models above ~60B parameters can "reason" step-by-step when prompted with "Let's think step by step." This dramatically improves performance on complex support scenarios like multi-item returns with partial refunds.

**Instruction Following**: After alignment, models follow natural language instructions reliably. "Respond in JSON with fields: resolution, refund_amount, reason" — and they do it.

### Model Landscape 2024–2026

| Model | Provider | Params | Context | Cost (1M in/out) | Open? | Best For |
|---|---|---|---|---|---|---|
| GPT-4o | OpenAI | ~200B (MoE) | 128K | $2.50 / $10.00 | ❌ | General excellence |
| GPT-4o-mini | OpenAI | ~25B (est.) | 128K | $0.15 / $0.60 | ❌ | Cost-effective production |
| Claude 3.5 Sonnet | Anthropic | ~70B (est.) | 200K | $3.00 / $15.00 | ❌ | Long-context, coding |
| Claude 3.5 Haiku | Anthropic | ~20B (est.) | 200K | $0.25 / $1.25 | ❌ | Fast, cheap, capable |
| Gemini 1.5 Pro | Google | ~MoE | 1M–2M | $1.25 / $5.00 | ❌ | Massive context windows |
| Llama 3.1 405B | Meta | 405B | 128K | Self-hosted | ✅ | Open-source frontier |
| Llama 3.1 70B | Meta | 70B | 128K | Self-hosted | ✅ | Best open mid-range |
| Llama 3.1 8B | Meta | 8B | 128K | Self-hosted | ✅ | Edge / cost-sensitive |
| Mistral Large 2 | Mistral | ~120B (MoE) | 128K | $2.00 / $6.00 | ❌ | European compliance |
| Mixtral 8x22B | Mistral | 141B (MoE) | 64K | Self-hosted | ✅ | Open MoE efficiency |
| Qwen 2.5 72B | Alibaba | 72B | 128K | Self-hosted | ✅ | Multilingual, math |
| DeepSeek-V3 | DeepSeek | 671B (MoE) | 128K | $0.27 / $1.10 | ✅ | Cost breakthrough |

*Note: Costs and context windows evolve rapidly. DeepSeek-V3 and Llama 4 represent the 2025-2026 trend of open models closing the gap.*

### 💡 Interview Insight

> **Q: How would you choose between open and closed models for an enterprise customer support system?**
>
> "It depends on three factors: data sensitivity, cost structure, and quality requirements. For our customer support system, customer conversations contain PII — names, order numbers, addresses. If regulatory compliance requires that data never leaves our infrastructure, open models like Llama 3.1 70B deployed on our own GPUs are the way to go. If we can use API-based models with a BAA or DPA in place, GPT-4o-mini or Claude 3.5 Haiku give us excellent quality at low cost without operational burden. My recommendation is typically a hybrid: route simple queries to an open 8B model self-hosted, and escalate complex ones to a frontier API model. This balances cost, quality, and data governance."

---

## Screen 6: Inference Parameters — Controlling Generation

Every customer support response your system generates is shaped by inference parameters. Misconfigure them and you get either robotic repetition or chaotic hallucination.

### Temperature

Temperature scales the logits before softmax. Low temperature → deterministic (greedy); high temperature → creative (random).

```python
import openai

client = openai.OpenAI()

# Temperature 0.0 — Deterministic, best for factual support responses
response_factual = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "What is your return policy?"}],
    temperature=0.0,  # Always picks the highest-probability token
)

# Temperature 1.2 — Creative, useful for marketing copy generation
response_creative = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Write a fun apology for a late delivery"}],
    temperature=1.2,  # Flattens distribution, more variety
)
```

### Top-p (Nucleus Sampling) and Top-k

```
Vocabulary probabilities (sorted):
  "order"   → 0.35
  "item"    → 0.25
  "product" → 0.15
  "package" → 0.10
  "thing"   → 0.05
  "widget"  → 0.03
  ...

Top-k = 3:  Sample from {"order", "item", "product"}
Top-p = 0.75: Sample from {"order", "item", "product"} (cumulative = 0.75)
Top-p = 0.95: Sample from {"order", "item", "product", "package", "thing"}
```

**Rule of thumb**: Use `temperature` OR `top_p`, not both. For customer support: `temperature=0.1, top_p=1.0` (near-deterministic but not completely greedy).

### Frequency & Presence Penalty

```python
# Without penalties — model may repeat itself
# "Your order is being processed. Your order will arrive soon. Your order..."

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Tell me about my order status"}],
    frequency_penalty=0.5,  # Penalizes tokens proportional to their count
    presence_penalty=0.3,   # Penalizes any token that appeared at all
    temperature=0.3,
)
```

| Parameter | Range | Effect | Support System Recommendation |
|---|---|---|---|
| temperature | 0.0–2.0 | Randomness of sampling | 0.0–0.3 for factual responses |
| top_p | 0.0–1.0 | Nucleus sampling cutoff | 0.9–1.0 |
| top_k | 1–100 | Hard cutoff on candidates | Usually not needed w/ top_p |
| frequency_penalty | -2.0–2.0 | Reduces repetition | 0.3–0.5 |
| presence_penalty | -2.0–2.0 | Encourages topic diversity | 0.0–0.3 |
| max_tokens | 1–model max | Output length limit | 500 for support, 2000 for reports |

### 💡 Interview Insight

> **Q: What inference parameters would you set for a production customer support chatbot?**
>
> "For customer support, consistency and accuracy trump creativity. I'd set temperature to 0.1 — not 0.0 because perfectly greedy decoding can sometimes get stuck in degenerate loops, but low enough for reliable, factual responses. Top-p at 0.95 as a safety net. Frequency penalty at 0.3 to prevent the model from repeating phrases like 'I apologize for the inconvenience' five times. Max tokens at 500 — support responses should be concise. I'd also implement response-level caching: identical queries within a time window get the same response, which improves consistency and reduces costs by 20-30%."

---

## Screen 7: Context Windows & Tokenization

### Tokenization: How Models See Text

LLMs don't see characters or words — they see **tokens**. Most modern models use **Byte-Pair Encoding (BPE)**, learned during training.

```python
import tiktoken

# GPT-4o uses the o200k_base tokenizer (~200K vocab)
enc = tiktoken.encoding_for_model("gpt-4o")

text = "Customer wants to return order #A1234-XZ for a refund"
tokens = enc.encode(text)
print(f"Text: {text}")
print(f"Tokens: {tokens}")
print(f"Token count: {len(tokens)}")
# Token count: 13

# Decode individual tokens to see the splits
for t in tokens:
    print(f"  {t} → '{enc.decode([t])}'")
# Common words = single token, rare words = multiple tokens
# "return" → 1 token, "#A1234-XZ" → multiple tokens

# Cost estimation for our support system
avg_input_tokens = 150   # typical customer query with context
avg_output_tokens = 200  # typical support response
queries_per_day = 50_000

daily_cost_gpt4o_mini = (
    (avg_input_tokens * queries_per_day / 1_000_000) * 0.15 +
    (avg_output_tokens * queries_per_day / 1_000_000) * 0.60
)
print(f"Daily cost (GPT-4o-mini): ${daily_cost_gpt4o_mini:.2f}")
# Daily cost: $7.13/day — very affordable!
```

### The Lost-in-the-Middle Problem

Research from Stanford (2023) showed that LLMs perform worse when relevant information is placed in the **middle** of long contexts. Performance is best when key information is at the **beginning** or **end**.

```
┌─────────────────────────────────────────────────────┐
│                 Performance vs Position              │
│                                                      │
│  High ████                                    ████   │
│       ████                                    ████   │
│       ████                                    ████   │
│       ████   ████                       ████  ████   │
│  Low  ████   ████  ████  ████  ████     ████  ████   │
│       ████   ████  ████  ████  ████     ████  ████   │
│       ─────────────────────────────────────────────  │
│       Start        Middle                    End     │
│                Position of relevant info              │
└─────────────────────────────────────────────────────┘
```

**Implication for our support system**: When stuffing retrieved context into the prompt, put the most relevant documents at the beginning and end. Don't bury critical return policy details in the middle of 10 retrieved chunks.

### 💡 Interview Insight

> **Q: How does tokenization impact cost and quality in production?**
>
> "Tokenization affects both cost and capability. First, cost: every token costs money. For our support system at 50K queries/day, the difference between a 100-token and 200-token system prompt is $0.75/day on GPT-4o-mini, but $25/day on GPT-4o — that compounds. Second, quality: non-English languages and code tokenize less efficiently, using 2-3× more tokens for the same content, which means the effective context window shrinks. Third, the lost-in-the-middle problem means how we arrange information in the context window matters as much as what we include. I always place the most relevant retrieved context at the top of the prompt, followed by instructions, with the user query at the end."

---

## Screen 8: LLM APIs — ChatCompletion, Function Calling, Streaming

### The ChatCompletion Format

All major LLM APIs have converged on the chat message format:

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

# The standard chat format — system, user, assistant roles
messages = [
    {"role": "system", "content": "You are ShopAssist, an e-commerce support agent. "
                                   "Be concise, empathetic, and action-oriented."},
    {"role": "user", "content": "I received a damaged item. Order #9912."},
    {"role": "assistant", "content": "I'm sorry to hear that! Let me look up order #9912 for you."},
    {"role": "user", "content": "It's a laptop with a cracked screen."},
]

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    temperature=0.1,
)
```

### Function Calling / Tool Use

Function calling lets the LLM invoke structured actions — essential for a support system that needs to look up orders, process refunds, and check inventory.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "Look up order details by order ID",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "The order ID, e.g., #9912"},
                },
                "required": ["order_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "initiate_refund",
            "description": "Start a refund process for an order",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string"},
                    "reason": {"type": "string", "enum": ["damaged", "wrong_item", "not_received", "changed_mind"]},
                    "refund_type": {"type": "string", "enum": ["full", "partial"]},
                },
                "required": ["order_id", "reason", "refund_type"],
            },
        },
    },
]

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    tools=tools,
    tool_choice="auto",  # Model decides when to call tools
)

# The model returns a tool call instead of text
tool_call = response.choices[0].message.tool_calls[0]
print(f"Function: {tool_call.function.name}")
print(f"Args: {tool_call.function.arguments}")
# Function: lookup_order
# Args: {"order_id": "#9912"}
```

### Streaming & Batching

```python
# Streaming — essential for UX, reduces perceived latency
async def stream_response(client, messages):
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        stream=True,
    )
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

# Batching — for bulk operations (e.g., categorizing 1000 support tickets)
# OpenAI Batch API: 50% cost reduction, 24-hour SLA
batch_input = [
    {"custom_id": f"ticket-{i}", "method": "POST", "url": "/v1/chat/completions",
     "body": {"model": "gpt-4o-mini", "messages": [{"role": "user", "content": ticket}]}}
    for i, ticket in enumerate(tickets)
]
```

### 💡 Interview Insight

> **Q: How do you handle rate limits and reliability in a production LLM API integration?**
>
> "I implement a three-layer resilience strategy. First, client-side rate limiting with token bucket algorithms — I track tokens-per-minute and requests-per-minute locally and queue excess requests. Second, exponential backoff with jitter on 429 and 5xx errors, using tenacity or a custom retry decorator. Third, model fallback — if GPT-4o returns errors, automatically fall back to GPT-4o-mini or Claude 3.5 Haiku. I also implement request hedging for latency-critical paths: fire the same request to two providers, take whichever responds first, cancel the other. For our support system, I'd add a circuit breaker that serves cached responses from a similar-query index when all LLM providers are down."

---

## Screen 9: Multi-Modal LLMs — Vision, Audio, and Beyond

Modern LLMs process more than text. For our customer support system, multi-modal capabilities unlock powerful use cases.

### Vision: Processing Customer Images

Customers often send photos of damaged items, wrong products, or unclear labels. GPT-4V, Claude 3.5, and Gemini can all process images.

```python
import base64
from openai import OpenAI

client = OpenAI()

def assess_damage_from_photo(image_path: str) -> dict:
    """Use vision model to assess product damage from customer photo."""
    with open(image_path, "rb") as f:
        image_b64 = base64.b64encode(f.read()).decode()

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a product damage assessor. Analyze the image and return "
                    "a JSON object with: damage_severity (minor/moderate/severe), "
                    "damage_description, recommended_action (replace/refund/repair)."
                ),
            },
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "Customer sent this photo of their damaged laptop:"},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_b64}"}},
                ],
            },
        ],
        temperature=0.1,
        response_format={"type": "json_object"},
    )
    return response.choices[0].message.content
```

### Audio: Voice-Based Support with Whisper

```python
from openai import OpenAI

client = OpenAI()

def transcribe_support_call(audio_path: str) -> str:
    """Transcribe customer support call for analysis and record-keeping."""
    with open(audio_path, "rb") as audio_file:
        transcript = client.audio.transcriptions.create(
            model="whisper-1",
            file=audio_file,
            response_format="verbose_json",
            timestamp_granularities=["segment"],
        )
    return transcript

# Pipeline: Voice call → Whisper transcription → LLM analysis → Action
```

### Multi-Modal Architecture

```
┌────────────────────────────────────────────────────┐
│           Multi-Modal Customer Support              │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  Text     │  │  Image   │  │  Audio   │         │
│  │  Input    │  │  Input   │  │  Input   │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │              │              │                │
│       │         ┌────▼─────┐  ┌────▼─────┐         │
│       │         │  Vision  │  │  Whisper  │         │
│       │         │  Encoder │  │  ASR      │         │
│       │         └────┬─────┘  └────┬─────┘         │
│       │              │              │                │
│       ▼              ▼              ▼                │
│  ┌─────────────────────────────────────────┐        │
│  │     Unified LLM Processing (GPT-4o)     │        │
│  │   Text + visual tokens + transcription  │        │
│  └──────────────────┬──────────────────────┘        │
│                     │                                │
│            ┌────────▼────────┐                       │
│            │  Tool Calling   │                       │
│            │  lookup_order() │                       │
│            │  initiate_refund│                       │
│            │  escalate()     │                       │
│            └────────┬────────┘                       │
│                     │                                │
│            ┌────────▼────────┐                       │
│            │  Response       │                       │
│            │  (text/voice)   │                       │
│            └─────────────────┘                       │
└────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: How would you integrate multi-modal capabilities into a customer support system?**
>
> "I'd build multi-modal support as an extension of the existing text pipeline, not a separate system. When a customer uploads a photo of a damaged product, I use GPT-4o's vision to assess the damage and extract details — severity, product identification, even reading serial numbers from the image. This feeds into the same tool-calling pipeline that handles text queries. For voice support, Whisper transcribes the call in real-time, and the LLM processes the transcript plus any images shared. The key architectural decision is using a unified model like GPT-4o that handles text and images natively, rather than stitching together separate models, which reduces latency and improves cross-modal reasoning."

---

## Module 1 Quiz

**1. In a decoder-only Transformer, what does the causal attention mask ensure?**

- A) All tokens can attend to all other tokens bidirectionally
- B) Each token can only attend to itself
- C) Each token can only attend to previous tokens (and itself) ✅
- D) Only the last token attends to all previous tokens
- E) Attention is applied only between adjacent tokens

**2. What is the primary advantage of DPO over RLHF?**

- A) DPO produces higher quality models
- B) DPO eliminates the need for human preference data
- C) DPO eliminates the separate reward model, simplifying the pipeline ✅
- D) DPO requires less training data
- E) DPO only works with encoder-decoder models

**3. For a 70B parameter model with GQA (8 KV heads instead of 64), how much is the KV-cache reduced compared to standard MHA?**

- A) 2×
- B) 4×
- C) 8× ✅
- D) 16×
- E) 64×

**4. According to the Chinchilla scaling laws, approximately how many tokens should a 10B parameter model be trained on for compute-optimal training?**

- A) 10 billion tokens
- B) 50 billion tokens
- C) 200 billion tokens ✅
- D) 1 trillion tokens
- E) 10 trillion tokens

**5. A customer support system receives a complex multi-step query. Which inference parameter configuration is most appropriate?**

- A) temperature=1.5, top_p=1.0, frequency_penalty=0.0
- B) temperature=0.0, top_p=0.5, frequency_penalty=2.0
- C) temperature=0.1, top_p=0.95, frequency_penalty=0.3 ✅
- D) temperature=2.0, top_p=0.1, frequency_penalty=0.0
- E) temperature=0.5, top_p=0.5, frequency_penalty=-1.0

---

## Key Takeaways

1. **Decoder-only Transformers** use causal self-attention and next-token prediction — the simplicity of this objective is what enables scaling to emergent capabilities.

2. **The alignment pipeline** (Pre-train → SFT → RLHF/DPO) transforms a text completer into a useful assistant. DPO is increasingly preferred for its simplicity.

3. **KV-cache** is the critical inference optimization. GQA reduces its memory footprint by 8×, enabling practical long-context serving.

4. **Chinchilla scaling laws** show that smaller models trained on more data are more inference-efficient — prefer over-trained small models for production.

5. **Inference parameters** (temperature, top-p, penalties) dramatically affect output quality. For customer support: low temperature, moderate frequency penalty.

6. **Tokenization** affects cost and quality. Understand token counts, BPE mechanics, and the lost-in-the-middle problem for prompt design.

7. **Function calling** is the bridge between LLM reasoning and real-world actions — essential for any production support system.

8. **Multi-modal models** (vision + audio + text) enable rich customer interactions — damage assessment from photos, voice transcription, and unified processing.
