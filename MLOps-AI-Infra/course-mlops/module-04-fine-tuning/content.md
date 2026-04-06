# Module 4: Fine-Tuning LLMs

> **Goal**: Master the theory, tooling, and production patterns for fine-tuning large language models — from deciding *whether* to fine-tune, through LoRA/QLoRA training, alignment (DPO/RLHF), to serving adapters at scale. Everything here is interview-grade.

---

## Screen 1: The Decision Framework — When (and When NOT) to Fine-Tune

Before you touch a single weight, ask yourself: **do I actually need fine-tuning?** Most interview panels will probe your judgment here before diving into mechanics.

The golden rule is an escalation ladder:

```
┌─────────────────────────────────────────────────────────┐
│            THE LLM CUSTOMIZATION LADDER                 │
│                                                         │
│  Cost / Complexity ▲                                    │
│                    │                                    │
│              ┌─────┴──────┐                             │
│         4.   │ Fine-Tune  │  Change behavior/style      │
│              │ + RAG      │  AND add knowledge          │
│              └─────┬──────┘                             │
│              ┌─────┴──────┐                             │
│         3.   │ Fine-Tune  │  Change behavior, style,    │
│              │   (LoRA)   │  format, or domain jargon   │
│              └─────┬──────┘                             │
│              ┌─────┴──────┐                             │
│         2.   │    RAG     │  Add external knowledge     │
│              │            │  or private data             │
│              └─────┬──────┘                             │
│              ┌─────┴──────┐                             │
│         1.   │  Prompt    │  Few-shot, CoT, system      │
│              │Engineering │  prompts — try this FIRST   │
│              └────────────┘                             │
│                                                         │
│  Start at the bottom. Only climb when you must.         │
└─────────────────────────────────────────────────────────┘
```

**When TO fine-tune:**

- You need a **specific output style** (e.g., legal briefs, medical notes, brand voice)
- You need the model to learn **domain jargon** it doesn't know (internal code names, proprietary terms)
- You need **consistent structured output** (always JSON, always a specific schema)
- You want to **distill** a larger model's behavior into a smaller, cheaper one
- You have **latency/cost constraints** and need a smaller model that punches above its weight

**When NOT to fine-tune:**

- You need **factual knowledge** → use RAG instead (fine-tuning memorizes, doesn't retrieve)
- You have **< 500 quality examples** → prompt engineering will outperform
- Your task changes frequently → retraining is expensive
- A ** model + prompting** already solves it at acceptable cost

**The best production systems combine both**: fine-tune for style/format + RAG for knowledge.

### 💡 Interview Insight
> *"Walk me through how you'd decide between RAG and fine-tuning."*
> Lead with the escalation ladder. Say: "I always start with prompt engineering. If the gap is **knowledge**, I add RAG. If the gap is **behavior or style**, I fine-tune. For many production systems, I combine both — fine-tune a base model for domain tone and structured output, then wire in RAG for up-to-date factual grounding."

**Cost-benefit analysis** interviewers love:

| Factor                        | Fine-Tuning              | API (GPT-4 etc.)          |
|-------------------------------|--------------------------|---------------------------|
| Upfront cost                  | GPU hours ($50–$500+)    | $0                        |
| Per-request cost              | Self-hosted (~$0.001)    | $0.01–$0.06 per 1K tok    |
| Data privacy                  | Full control             | Data leaves your infra    |
| Latency                       | Optimizable (vLLM)       | Network-bound             |
| Customization depth           | Deep behavioral change   | Prompt-level only         |
| Maintenance                   | You own the infra        | Provider handles it       |

At scale (millions of requests/month), fine-tuning a smaller model almost always wins on cost.

---

## Screen 2: LoRA — Low-Rank Adaptation

LoRA is the technique that made fine-tuning accessible. Instead of updating all model weights (billions of parameters), you inject tiny trainable matrices into specific layers.

### The Core Math

For a pretrained weight matrix **W** of dimension **(d × d)**:

```
Standard fine-tuning:    W' = W + ΔW        (ΔW is d × d — huge!)

LoRA:                    W' = W + B·A        where B is (d × r), A is (r × d)
                                              r << d  (rank, typically 8–64)
```

The key insight: **ΔW is low-rank**. Most task-specific adaptations live in a low-dimensional subspace. Instead of storing a full **(d × d)** update, you store two small matrices whose product approximates it.

**Parameter savings**: If d = 4096 and r = 16:
- Full ΔW: 4096 × 4096 = **16.7M** parameters
- LoRA (B + A): (4096 × 16) + (16 × 4096) = **131K** parameters → **~128× fewer**

```
┌──────────────────────────────────────────────────────┐
│          LoRA Injection into a Transformer Layer      │
│                                                      │
│   Input x                                            │
│     │                                                │
│     ├──────────────────────┐                         │
│     │                      │                         │
│     ▼                      ▼                         │
│  ┌──────────┐        ┌──────────┐                    │
│  │ Frozen W │        │  LoRA    │                    │
│  │ (d × d)  │        │  Branch  │                    │
│  │ ■■■■■■■■ │        │          │                    │
│  │ (no grad)│        │ ┌──────┐ │                    │
│  └────┬─────┘        │ │ A    │ │  A: (r × d)       │
│       │              │ │(r×d) │ │  ← trainable      │
│       │              │ └──┬───┘ │                    │
│       │              │    │     │                    │
│       │              │ ┌──▼───┐ │                    │
│       │              │ │ B    │ │  B: (d × r)       │
│       │              │ │(d×r) │ │  ← trainable      │
│       │              │ └──┬───┘ │                    │
│       │              │    │     │                    │
│       │              │  × α/r  │  ← scaling factor  │
│       │              └────┬─────┘                    │
│       │                   │                          │
│       └───────┬───────────┘                          │
│               │ (add)                                │
│               ▼                                      │
│          Output: Wx + (α/r)·BAx                      │
│                                                      │
│  Total trainable params: ~0.1–1% of model            │
└──────────────────────────────────────────────────────┘
```

### Key Hyperparameters

| Hyperparameter     | Typical Range      | Effect                          |
|--------------------|--------------------|---------------------------------|
| `r` (rank)         | 8–64               | Higher = more capacity          |
| `lora_alpha`       | 16–128 (often 2×r) | Scaling; higher = stronger      |
| `target_modules`   | q_proj, v_proj, …  | Which layers get LoRA           |
| `lora_dropout`     | 0.0–0.1            | Regularization                  |

**Targeting more modules** (q, k, v, o, gate, up, down projections) generally improves quality at the cost of more trainable parameters. A common starting point is `["q_proj", "v_proj"]`; for stronger adaptation, target all linear layers.

The **scaling factor** is `α/r`. When you double `r`, double `α` to keep the effective learning rate stable.

### 💡 Interview Insight
> *"Why does LoRA work? Isn't throwing away most of the update matrix lossy?"*
> The hypothesis from the original paper (Hu et al., 2021) is that pretrained LLMs have a **low intrinsic dimensionality** — task-specific adaptations live in a small subspace. Empirically, rank 8–16 recovers 95%+ of full fine-tuning quality on most tasks. The remaining capacity is redundant for the adaptation.

---

## Screen 3: QLoRA and DoRA — Memory-Efficient Variants

### QLoRA: Quantize Then Adapt

QLoRA (Dettmers et al., 2023) combines **4-bit quantization** of the frozen base model with LoRA adapters on top. This is the technique that lets you fine-tune a **70B model on a single 48GB GPU**.

```
┌─────────────────────────────────────────────────────┐
│                QLoRA Memory Breakdown                │
│                                                     │
│   Technique          │ 7B Model  │ 70B Model        │
│   ───────────────────┼───────────┼────────────────   │
│   Full FP16          │ ~14 GB    │ ~140 GB           │
│   LoRA (FP16 base)   │ ~14 GB    │ ~140 GB           │
│   QLoRA (4-bit base) │ ~4 GB     │ ~38 GB    ✅     │
│   + LoRA adapters    │ +~50 MB   │ +~200 MB          │
│   + optimizer states │ +~100 MB  │ +~400 MB          │
│                                                     │
│   QLoRA total 70B:   ~38.6 GB → fits on 1× A100!   │
└─────────────────────────────────────────────────────┘
```

**Key innovations in QLoRA:**

1. **NF4 (4-bit NormalFloat)** — quantization format optimized for normally distributed weights
2. **Double quantization** — quantize the quantization constants themselves (saves ~0.4 bits/param)
3. **Paged optimizers** — offload optimizer states to CPU when GPU memory spikes

```python
from transformers import BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",            # NormalFloat4
    bnb_4bit_compute_dtype=torch.bfloat16, # Compute in bf16
    bnb_4bit_use_double_quant=True,        # Double quantization
)
```

### DoRA: Weight-Decomposed LoRA

DoRA (Liu et al., 2024) decomposes the weight update into **magnitude** and **direction** components:

```
Standard LoRA:   W' = W + BA

DoRA:            W' = m · (W + BA) / ‖W + BA‖
                 ↑ magnitude    ↑ direction (unit vector)
```

DoRA consistently beats LoRA at the **same rank** across benchmarks, with only ~1-5% extra compute. It's increasingly becoming the default choice.

### 💡 Interview Insight
> *"When would you choose QLoRA over full-precision LoRA?"*
> QLoRA is almost always the right choice for models ≥ 13B, or when GPU budget is limited. The quality gap vs. FP16 LoRA is minimal (~0.5% on most benchmarks). For 7B models where you have abundant GPU memory, FP16 LoRA trains faster due to less quantization overhead. For production fine-tuning of 70B+ models, QLoRA is the only practical single-GPU option.

---

## Screen 4: Dataset Preparation — Garbage In, Garbage Out

Dataset quality is the single biggest lever for fine-tuning success. 1,000 excellent examples beat 100,000 mediocre ones.

### Common Formats

**Alpaca format** (instruction-following):
```python
dataset = [
    {
        "instruction": "Summarize the key findings.",
        "input": "The study found that LoRA achieves 95% of ...",
        "output": "The study demonstrates that LoRA ..."
    },
    {
        "instruction": "Convert to SQL.",
        "input": "Find all users who signed up in 2024.",
        "output": "SELECT * FROM users WHERE YEAR(signup_date) = 2024;"
    }
]
```

**Chat/ShareGPT format** (multi-turn):
```python
dataset = [
    {
        "conversations": [
            {"role": "system", "content": "You are a helpful coding assistant."},
            {"role": "user", "content": "How do I read a CSV in pandas?"},
            {"role": "assistant", "content": "Use `pd.read_csv('file.csv')` ..."},
            {"role": "user", "content": "What if it's tab-separated?"},
            {"role": "assistant", "content": "Pass `sep='\\t'` ..."}
        ]
    }
]
```

**ChatML format** (used by many models natively):
```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Explain LoRA in one sentence.<|im_end|>
<|im_start|>assistant
LoRA injects small trainable matrices into frozen model layers.<|im_end|>
```

### Dataset Preparation Pipeline

```python
from datasets import load_dataset, Dataset
import json

def prepare_alpaca_dataset(file_path: str) -> Dataset:
    """Load and format dataset in Alpaca style for SFTTrainer."""

    with open(file_path) as f:
        raw_data = json.load(f)

    def format_prompt(example: dict) -> dict:
        """Convert to the prompt template the model expects."""
        if example.get("input"):
            text = (
                f"### Instruction:\n{example['instruction']}\n\n"
                f"### Input:\n{example['input']}\n\n"
                f"### Response:\n{example['output']}"
            )
        else:
            text = (
                f"### Instruction:\n{example['instruction']}\n\n"
                f"### Response:\n{example['output']}"
            )
        return {"text": text}

    dataset = Dataset.from_list(raw_data)
    dataset = dataset.map(format_prompt)

    # Quality filters
    dataset = dataset.filter(
        lambda x: 50 < len(x["text"]) < 4096  # Length bounds
    )

    # Train/eval split
    split = dataset.train_test_split(test_size=0.1, seed=42)
    return split["train"], split["test"]

train_ds, eval_ds = prepare_alpaca_dataset("my_data.json")
print(f"Train: {len(train_ds)}, Eval: {len(eval_ds)}")
```

### Dataset Size Guidelines

| Dataset Size     | Use Case                          | Notes                          |
|------------------|-----------------------------------|--------------------------------|
| 100–500          | Style transfer, format change     | Works if quality is high       |
| 1K–10K           | Domain adaptation                 | Sweet spot for most tasks      |
| 10K–100K         | Complex multi-task fine-tuning    | Diminishing returns past 50K   |
| 100K+            | Pre-training continuation         | Rarely needed for fine-tuning  |

### Data Quality Checklist

- **Deduplication**: remove exact and near-duplicates (MinHash)
- **Safety filtering**: strip PII, toxic content, copyrighted material
- **Consistency**: same format and tone across all examples
- **Augmentation**: use a stronger model (e.g., GPT-4) to generate/refine examples
- **Balanced**: don't overrepresent one category — the model will overfit to it

### 💡 Interview Insight
> *"How do you ensure dataset quality for fine-tuning?"*
> Talk about the pipeline: "I start with deduplication using MinHash, filter for length and quality, then manually review a random 5-10% sample. I use stratified splitting for train/eval. If the dataset is small, I augment with a stronger model — generating variations or using the strong model to score and filter weak examples. Quality over quantity is the mantra — I've seen 2K excellent examples outperform 50K scraped ones."

---

## Screen 5: Full QLoRA Training Script

This is the script you should be able to write (or explain) in an interview. Every line matters.

```python
"""
Full QLoRA fine-tuning script using HuggingFace ecosystem.
Fine-tunes Llama-3-8B on a custom instruction dataset.
"""
import torch
from datasets import load_dataset
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    BitsAndBytesConfig,
)
from peft import (
    LoraConfig,
    get_peft_model,
    prepare_model_for_kbit_training,
    TaskType,
)
from trl import SFTTrainer, SFTConfig

# ── 1. Quantization Config ───────────────────────────────
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

# ── 2. Load Model + Tokenizer ────────────────────────────
model_name = "meta-llama/Meta-Llama-3-8B"

tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token
tokenizer.padding_side = "right"  # Required for SFTTrainer

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto",
    attn_implementation="flash_attention_2",  # Speed + memory
    torch_dtype=torch.bfloat16,
)

# Prepare model for k-bit training (freeze base, enable grad for adapters)
model = prepare_model_for_kbit_training(model)

# ── 3. LoRA Config ───────────────────────────────────────
lora_config = LoraConfig(
    r=16,                          # Rank — start here, go to 32/64 if needed
    lora_alpha=32,                 # Scaling factor (2× rank is a good default)
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",  # Attention
        "gate_proj", "up_proj", "down_proj",      # MLP (FFN)
    ],
    lora_dropout=0.05,             # Light regularization
    bias="none",                   # Don't train biases
    task_type=TaskType.CAUSAL_LM,
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 13,631,488 || all params: 8,043,724,800
#         || trainable%: 0.1695%

# ── 4. Load Dataset ──────────────────────────────────────
dataset = load_dataset("json", data_files="train_data.json", split="train")

def format_instruction(example):
    """Format into the prompt template."""
    return {
        "text": (
            f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n"
            f"{example['instruction']}\n"
            f"{example.get('input', '')}<|eot_id|>"
            f"<|start_header_id|>assistant<|end_header_id|>\n\n"
            f"{example['output']}<|eot_id|>"
        )
    }

dataset = dataset.map(format_instruction)

# ── 5. Training Config ───────────────────────────────────
training_args = SFTConfig(
    output_dir="./qlora-llama3-8b",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,       # Effective batch = 4 × 4 = 16
    learning_rate=2e-4,                  # LoRA can handle higher LR
    lr_scheduler_type="cosine",
    warmup_ratio=0.05,
    weight_decay=0.01,
    bf16=True,
    logging_steps=10,
    eval_strategy="steps",
    eval_steps=100,
    save_strategy="steps",
    save_steps=100,
    save_total_limit=3,
    max_seq_length=2048,
    dataset_text_field="text",           # Column name in dataset
    packing=True,                        # Pack short examples together
    gradient_checkpointing=True,         # Trade compute for memory
    gradient_checkpointing_kwargs={"use_reentrant": False},
    report_to="wandb",                   # Or "mlflow"
    run_name="qlora-llama3-8b-v1",
)

# ── 6. Train ─────────────────────────────────────────────
trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=dataset,
    eval_dataset=eval_dataset,           # Your eval split
    tokenizer=tokenizer,
)

trainer.train()

# ── 7. Save LoRA Adapter ─────────────────────────────────
trainer.save_model("./qlora-llama3-8b/final")
tokenizer.save_pretrained("./qlora-llama3-8b/final")
# This saves ONLY the adapter weights (~30MB), not the full model
```

### Key Training Hyperparameters Explained

| Parameter                     | Value    | Why                               |
|-------------------------------|----------|-----------------------------------|
| `learning_rate`               | 2e-4     | LoRA tolerates higher LR          |
| `num_train_epochs`            | 1–3      | More epochs → risk overfitting    |
| `gradient_accumulation_steps` | 4–8      | Simulate larger batch size        |
| `warmup_ratio`                | 0.03–0.1 | Gentle start prevents spikes      |
| `packing`                     | True     | Pack short seqs → faster training |
| `gradient_checkpointing`      | True     | ~30% less VRAM, ~20% slower       |
| `max_seq_length`              | 2048     | Match your data; longer = more RAM|

### 💡 Interview Insight
> *"Walk me through this training script. What would you change for a production run?"*
> Key things to mention: (1) I'd add **early stopping** based on eval loss to prevent overfitting. (2) I'd use **packing=True** to avoid wasting compute on padding. (3) I'd integrate **Weights & Biases** for experiment tracking. (4) For larger models, I'd enable **DeepSpeed ZeRO Stage 2** via the `deepspeed` argument. (5) I'd checkpoint frequently and run eval every N steps to catch divergence early.

---

## Screen 6: Alignment — RLHF, DPO, and Beyond

Fine-tuning teaches a model *what* to say. Alignment teaches it *how* to behave — helpful, harmless, and honest.

### The Alignment Landscape

```
┌──────────────────────────────────────────────────────────┐
│              ALIGNMENT TECHNIQUES EVOLUTION               │
│                                                          │
│   2022          2023           2024          2024+        │
│                                                          │
│   RLHF ──────► DPO ────────► ORPO ───────► SimPO        │
│   (complex)    (simpler)     (unified)     (simplest)    │
│                                                          │
│   3 models     2 models      1 model       1 model       │
│   unstable     stable        stable        stable        │
│   PPO needed   no RL         no RL         no RL         │
│   ▼            ▼             ▼             ▼             │
│   reward +     preferred/    SFT +         reference-    │
│   policy +     rejected      preference    free DPO      │
│   reference    pairs         in one step                 │
└──────────────────────────────────────────────────────────┘
```

### Comparison Table

| Aspect             | RLHF              | DPO               | ORPO                | SimPO              |
|--------------------|--------------------|--------------------|---------------------|--------------------| 
| Complexity         | Very High          | Medium             | Low                 | Low                |
| Models in memory   | 3 (reward+pol+ref) | 2 (policy+ref)     | 1                   | 1                  |
| Training stability | Fragile (PPO)      | Stable             | Stable              | Stable             |
| Data requirement   | Ranked preferences | (chosen, rejected) | (chosen, rejected)  | (chosen, rejected) |
| Quality ceiling    | Highest (tunable)  | Very good          | Good                | Good               |
| Industry adoption  | OpenAI, Anthropic  | Most common now    | Growing             | Emerging           |

### RLHF (Reinforcement Learning from Human Feedback)

The original alignment method. Three-phase process:
1. **SFT**: supervised fine-tune the base model
2. **Reward Model**: train a model to score responses (good vs. bad)
3. **PPO**: optimize the SFT model against the reward model using proximal policy optimization

Complexity and instability make this impractical for most teams.

### DPO (Direct Preference Optimization)

DPO eliminates the reward model entirely. Instead of training a reward model + RL, you **directly optimize** the policy using preference pairs.

The loss function implicitly defines the reward:

```
L_DPO = -log σ(β · (log π(y_w|x)/π_ref(y_w|x) - log π(y_l|x)/π_ref(y_l|x)))

where:
  y_w = preferred (winning) response
  y_l = rejected (losing) response
  π    = current policy
  π_ref = reference (frozen SFT) model
  β    = temperature (controls deviation from reference)
```

### DPO Training Code

```python
"""DPO training with TRL — preference alignment."""
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer
from trl import DPOTrainer, DPOConfig
from peft import LoraConfig

# Load your SFT-tuned model (the one you want to align)
model_name = "./qlora-llama3-8b/merged"  # Merged LoRA from SFT step

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",
    device_map="auto",
)
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# DPO dataset: each example has prompt + chosen + rejected
# Example structure:
# {
#   "prompt": "Explain quantum computing.",
#   "chosen": "Quantum computing uses qubits that can exist in ...",
#   "rejected": "Quantum computing is when computers go really fast ..."
# }
dpo_dataset = load_dataset("json", data_files="dpo_pairs.json", split="train")

# LoRA config for DPO (train new adapters on top of SFT model)
peft_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

# DPO training config
training_args = DPOConfig(
    output_dir="./dpo-llama3-8b",
    num_train_epochs=1,                  # DPO usually needs only 1 epoch
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    learning_rate=5e-5,                  # Lower LR for alignment
    beta=0.1,                            # DPO temperature
    max_length=1024,
    max_prompt_length=512,
    bf16=True,
    logging_steps=10,
    save_strategy="epoch",
    gradient_checkpointing=True,
    report_to="wandb",
)

# Train
dpo_trainer = DPOTrainer(
    model=model,
    args=training_args,
    train_dataset=dpo_dataset,
    tokenizer=tokenizer,
    peft_config=peft_config,
)

dpo_trainer.train()
dpo_trainer.save_model("./dpo-llama3-8b/final")
```

### ORPO: One-Step Alignment

ORPO (Odds Ratio Preference Optimization) combines SFT and alignment into a single training step — no need for a separate SFT phase. It adds a preference penalty to the standard cross-entropy loss. This makes it simpler and cheaper but gives you slightly less control.

### 💡 Interview Insight
> *"Compare RLHF vs DPO. When would you still choose RLHF?"*
> "DPO is my default for alignment — it's simpler, more stable, and requires fewer resources. I'd only choose RLHF when I need **fine-grained reward shaping** (e.g., separate scores for helpfulness, safety, and conciseness) or when the preference distribution is complex and multi-modal. OpenAI and Anthropic use RLHF because they operate at a scale where the extra complexity pays off. For most teams, DPO with 5K–20K preference pairs delivers excellent alignment."

---

## Screen 7: Evaluation of Fine-Tuned Models

Training is half the battle. Rigorous evaluation determines if your fine-tuned model is production-ready.

### Evaluation Hierarchy

```
┌────────────────────────────────────────────────┐
│         EVALUATION PYRAMID                      │
│                                                │
│              ┌───────┐                         │
│              │ Human │   Gold standard         │
│              │ Eval  │   (expensive, slow)      │
│              └───┬───┘                         │
│            ┌─────┴─────┐                       │
│            │ LLM-as-   │   GPT-4 judges        │
│            │ Judge     │   (scalable, good)     │
│            └─────┬─────┘                       │
│          ┌───────┴───────┐                     │
│          │ Task-Specific │   F1, ROUGE, Acc     │
│          │ Metrics       │   (automated)        │
│          └───────┬───────┘                     │
│       ┌──────────┴──────────┐                  │
│       │ Perplexity / Loss   │   Quick sanity   │
│       │ (on held-out data)  │   check          │
│       └─────────────────────┘                  │
└────────────────────────────────────────────────┘
```

### Metrics Breakdown

| Metric             | What It Measures                   | When to Use                    |
|--------------------|------------------------------------|--------------------------------|
| Eval loss          | Prediction confidence              | Always (sanity check)          |
| Perplexity         | How surprised model is by text     | Language modeling tasks         |
| ROUGE-L            | Overlap with reference text        | Summarization                  |
| F1 / Accuracy      | Classification correctness         | Classification tasks           |
| BLEU               | N-gram precision vs reference      | Translation                    |
| HumanEval          | Code generation correctness        | Coding tasks                   |
| MMLU               | General knowledge breadth          | Regression testing             |
| LLM-as-Judge       | Holistic quality rating            | Open-ended generation          |

### Regression Testing — Don't Break General Capabilities

This is critical and often overlooked: fine-tuning on a narrow domain can cause **catastrophic forgetting** of general capabilities.

```python
# Evaluate on standard benchmarks BEFORE and AFTER fine-tuning
benchmarks = {
    "mmlu": "General knowledge (should stay stable)",
    "hellaswag": "Common sense (should stay stable)",
    "humaneval": "Code generation (monitor for regression)",
}

# Compare: base model vs fine-tuned on each benchmark
# If MMLU drops > 2-3 points, you may be overfitting or
# training too many epochs.
```

### LLM-as-Judge Pattern

```python
judge_prompt = """
Rate the following response on a scale of 1-5 for:
1. Accuracy - Is the information correct?
2. Helpfulness - Does it address the user's need?
3. Style - Does it match the expected format/tone?

User Query: {query}
Model Response: {response}

Provide scores and brief justification for each.
"""

# Run on 100+ test examples, compare base vs fine-tuned
# Average scores across dimensions
```

### 💡 Interview Insight
> *"How do you evaluate a fine-tuned model before deploying it?"*
> "I use a four-layer approach: (1) **Eval loss** — must be decreasing and lower than baseline. (2) **Task-specific metrics** — F1, ROUGE, or whatever applies to the domain. (3) **LLM-as-Judge** — GPT-4 compares my fine-tuned model vs. the base on 200+ test prompts for quality, safety, and format compliance. (4) **Regression tests** — I run MMLU and HellaSwag to verify I haven't broken general capabilities. Only after all four layers pass do I promote to staging."

---

## Screen 8: Serving Fine-Tuned Models in Production

Training a great model means nothing if you can't serve it efficiently.

### Step 1: Merge LoRA Weights

```python
"""Merge LoRA adapter into base model and export."""
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

# Load base model (full precision for merging)
base_model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Meta-Llama-3-8B",
    torch_dtype="auto",
    device_map="auto",
)

# Load LoRA adapter on top
model = PeftModel.from_pretrained(base_model, "./qlora-llama3-8b/final")

# Merge: folds LoRA weights into the base weights
# After this, the model is a standard model — no PEFT needed at inference
model = model.merge_and_unload()

# Save the merged model
model.save_pretrained("./llama3-8b-merged", safe_serialization=True)

tokenizer = AutoTokenizer.from_pretrained("./qlora-llama3-8b/final")
tokenizer.save_pretrained("./llama3-8b-merged")
```

### Step 2: Serve with vLLM

**Option A: Serve the merged model** (simpler):
```bash
# Serve merged model directly
python -m vllm.entrypoints.openai.api_server \
    --model ./llama3-8b-merged \
    --dtype bfloat16 \
    --max-model-len 4096 \
    --port 8000
```

**Option B: Serve base + swappable LoRA adapters** (powerful for multi-tenant):
```bash
# Serve base model with dynamically loadable LoRA adapters
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-8B \
    --enable-lora \
    --lora-modules \
        legal-lora=./adapters/legal \
        medical-lora=./adapters/medical \
        code-lora=./adapters/code \
    --max-loras 3 \
    --max-lora-rank 64 \
    --port 8000
```

```python
"""Call different LoRA adapters via the OpenAI-compatible API."""
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

# Use the legal adapter
response = client.chat.completions.create(
    model="legal-lora",  # Adapter name from --lora-modules
    messages=[
        {"role": "user", "content": "Draft a liability clause."}
    ],
)

# Use the medical adapter — same base model, different behavior
response = client.chat.completions.create(
    model="medical-lora",
    messages=[
        {"role": "user", "content": "Summarize this patient note."}
    ],
)

# Use base model (no adapter)
response = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3-8B",
    messages=[
        {"role": "user", "content": "Hello!"}
    ],
)
```

This is incredibly powerful: **one base model in GPU memory, multiple specialized behaviors** via tiny adapter swaps. Each LoRA adapter is ~30MB vs. 16GB for the base model.

### Step 3: Export to GGUF for Ollama / llama.cpp

```bash
# Convert merged model to GGUF format (requires llama.cpp)
python llama.cpp/convert_hf_to_gguf.py \
    ./llama3-8b-merged \
    --outfile llama3-8b-finetuned.gguf \
    --outtype q4_K_M  # 4-bit quantization

# Create Ollama Modelfile
cat > Modelfile <<'EOF'
FROM ./llama3-8b-finetuned.gguf

PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 4096
PARAMETER stop "<|eot_id|>"

SYSTEM """You are a specialized assistant fine-tuned for
legal document analysis. Provide precise, well-structured
responses citing relevant legal principles."""

TEMPLATE """<|begin_of_text|><|start_header_id|>system<|end_header_id|>

{{ .System }}<|eot_id|><|start_header_id|>user<|end_header_id|>

{{ .Prompt }}<|eot_id|><|start_header_id|>assistant<|end_header_id|>

"""
EOF

# Build and run
ollama create my-legal-llm -f Modelfile
ollama run my-legal-llm "Summarize this NDA clause..."
```

### Serving Architecture Decision

| Deployment Target     | Best For                     | Latency       | Throughput   |
|-----------------------|------------------------------|---------------|--------------|
| vLLM (merged)         | High-throughput APIs         | Low (~50ms)   | Very High    |
| vLLM (multi-LoRA)     | Multi-tenant / multi-domain  | Low (~55ms)   | High         |
| TGI                   | HuggingFace ecosystem users  | Low           | High         |
| Ollama                | Local dev, edge deployment   | Medium        | Single-user  |
| GGUF + llama.cpp      | CPU/edge, no GPU available   | Higher        | Low          |

### 💡 Interview Insight
> *"How would you serve multiple fine-tuned variants of the same base model?"*
> "I'd use vLLM's multi-LoRA serving. You load the base model once into GPU memory and register multiple LoRA adapters. At request time, the client specifies which adapter to use via the `model` parameter. This is dramatically more memory-efficient than running separate model instances — you're sharing ~16GB of base weights and swapping ~30MB adapters. For production, I'd put this behind an API gateway that routes to the correct adapter based on tenant or use-case."

---

## Screen 9: Quiz — Test Your Fine-Tuning Knowledge

### Question 1
**What is the primary advantage of LoRA over full fine-tuning?**

- A) LoRA achieves higher quality than full fine-tuning
- B) LoRA trains only a small number of injected low-rank matrices, drastically reducing memory and storage ✅
- C) LoRA doesn't require a GPU
- D) LoRA works only with encoder models

### Question 2
**In QLoRA, what quantization format is used for the frozen base model?**

- A) INT8
- B) FP8
- C) NF4 (4-bit NormalFloat) ✅
- D) BF16

### Question 3
**What does DPO eliminate compared to RLHF?**

- A) The need for training data
- B) The reward model and PPO optimization loop ✅
- C) The need for a base model
- D) The SFT pre-training step

### Question 4
**When using vLLM's multi-LoRA serving, what is shared across all adapters?**

- A) The LoRA weights
- B) The optimizer states
- C) The base model weights in GPU memory ✅
- D) The training dataset

### Question 5
**Your fine-tuned model scores well on your domain task but MMLU scores dropped 5 points. What's the most likely cause?**

- A) The base model was too small
- B) The learning rate was too low
- C) Catastrophic forgetting from overfitting on domain data ✅
- D) The LoRA rank was too high

---

## Key Takeaways

```
┌──────────────────────────────────────────────────────────┐
│               FINE-TUNING CHEAT SHEET                    │
│                                                          │
│  1. DECIDE WISELY                                        │
│     Prompt eng → RAG → Fine-tune → Fine-tune + RAG      │
│     Fine-tune for STYLE/BEHAVIOR, RAG for KNOWLEDGE      │
│                                                          │
│  2. USE QLoRA                                            │
│     4-bit base + LoRA adapters = fine-tune 70B on 1 GPU  │
│     Start: r=16, alpha=32, target all linear layers      │
│                                                          │
│  3. DATA IS KING                                         │
│     1K excellent examples > 100K mediocre ones           │
│     Dedup, filter, manually review, augment with GPT-4   │
│                                                          │
│  4. ALIGN WITH DPO                                       │
│     Simpler than RLHF, no reward model needed            │
│     5K–20K preference pairs is a solid starting point    │
│                                                          │
│  5. EVALUATE RIGOROUSLY                                  │
│     Eval loss + task metrics + LLM-as-judge + regression │
│     Always check MMLU/HellaSwag for catastrophic forget  │
│                                                          │
│  6. SERVE EFFICIENTLY                                    │
│     vLLM multi-LoRA: 1 base model, N adapters            │
│     merge_and_unload() for simpler single-model deploy   │
│     GGUF + Ollama for local/edge deployment              │
│                                                          │
│  7. TRACK EVERYTHING                                     │
│     W&B or MLflow for experiments                        │
│     Version adapters like code — they're only ~30MB      │
│     Git-tag every adapter with training config + metrics │
└──────────────────────────────────────────────────────────┘
```

### Production Fine-Tuning Workflow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Curate  │───►│  Train   │───►│ Evaluate │───►│  Serve   │
│  Dataset │    │  QLoRA   │    │  4-Layer │    │  vLLM    │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
  Dedup,          r=16,          Eval loss,       Merged or
  filter,        alpha=32,      Task metrics,    Multi-LoRA,
  augment,       3 epochs,      LLM judge,       GGUF for
  format         track w/ W&B   regression       edge
```

### Interview Power Phrases

- *"I follow the escalation ladder — prompting first, RAG for knowledge, fine-tuning for behavior."*
- *"LoRA exploits the low intrinsic dimensionality of task-specific adaptations."*
- *"QLoRA lets me fine-tune 70B models on a single A100 by quantizing the base to NF4."*
- *"DPO is my default alignment technique — simpler and more stable than RLHF for most use cases."*
- *"I always run regression tests on MMLU and HellaSwag to catch catastrophic forgetting."*
- *"For multi-tenant serving, vLLM's multi-LoRA is the architecture — one base model, swappable 30MB adapters."*
- *"Quality over quantity in training data. I've seen 2K curated examples beat 50K scraped ones."*

---

*Next module: [Module 5 — Inference Optimization: Quantization, KV-Cache, Speculative Decoding →]*
