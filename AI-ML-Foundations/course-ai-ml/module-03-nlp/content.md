# Module 3: NLP for AI Engineers

> **Duration:** ~3.5 hours | **Difficulty:** Intermediate-Advanced | **Prerequisites:** Module 2 (Deep Learning Foundations)
> From raw text to production NLP systems — tokenization, embeddings, Transformers, and Hugging Face.

---

## Screen 1: Text Preprocessing — Tokenization, Stemming & Lemmatization

### Why Text Preprocessing?

Raw text is messy. Models need structured numerical input. Preprocessing bridges that gap.

```
Raw Text                          Cleaned                         Tokenized
"I can't BELIEVE it's            "i cannot believe it is         ["i", "cannot", "believe",
 only $9.99!!!"                   only 9.99"                      "it", "is", "only", "9.99"]
```

### Tokenization Strategies

```
Word-level tokenization:
  "I love machine learning" → ["I", "love", "machine", "learning"]
  ✅ Simple, interpretable
  ❌ Huge vocabulary, can't handle OOV words ("unforgettable" → UNK)

Character-level tokenization:
  "cat" → ["c", "a", "t"]
  ✅ Tiny vocabulary (26 letters + special)
  ❌ Sequences too long, loses word-level meaning

Subword tokenization (the sweet spot):
  "unforgettable" → ["un", "forget", "table"]  (BPE)
  "playing" → ["play", "##ing"]                 (WordPiece)
  ✅ Handles any word, reasonable vocabulary (30K-50K)
  ✅ Rare words decomposed, common words stay whole
```

### Subword Tokenization Deep Dive

| Method | Used By | How It Works |
|---|---|---|
| BPE (Byte-Pair Encoding) | GPT, Llama, RoBERTa | Merge most frequent byte pairs iteratively |
| WordPiece | BERT | Like BPE but maximizes likelihood |
| SentencePiece | T5, Llama | Language-agnostic, operates on raw text |
| tiktoken | GPT-3.5/4 | OpenAI's fast BPE implementation |

```python
# BPE: How it works conceptually
# Start: ['l', 'o', 'w', 'e', 'r'] frequency: 5
#        ['l', 'o', 'w', 'e', 's', 't'] frequency: 2
# Step 1: Most frequent pair ('l','o') → merge → 'lo'
# Step 2: Most frequent pair ('lo','w') → merge → 'low'
# Step 3: Most frequent pair ('e','r') → merge → 'er'
# ... until vocabulary reaches desired size

# Practical: using tiktoken (OpenAI's tokenizer)
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")
tokens = enc.encode("Hello, world!")
print(tokens)          # [9906, 11, 1917, 0]
print(len(tokens))     # 4 tokens
decoded = enc.decode(tokens)  # "Hello, world!"

# Using HuggingFace tokenizers
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
encoded = tokenizer("I love machine learning!", return_tensors="pt")
print(encoded['input_ids'])       # tensor([[ 101, 1045, 2293, 3698, 4083, 999,  102]])
print(encoded['attention_mask'])  # tensor([[1, 1, 1, 1, 1, 1, 1]])
# 101=[CLS], 102=[SEP] — special tokens added automatically
```

### Special Tokens

```
BERT tokens:
  [CLS] The cat sat on the mat [SEP]
  ↑                              ↑
  Classification token           Separator token
  (aggregate representation)     (end of segment)

GPT tokens:
  <|endoftext|> The cat sat on the mat
  ↑
  End of text / beginning marker

Llama tokens:
  <s> The cat sat on the mat </s>
  ↑ BOS (beginning)           ↑ EOS (end of sequence)
```

### Stemming vs Lemmatization

```python
import nltk
from nltk.stem import PorterStemmer, WordNetLemmatizer

stemmer = PorterStemmer()
lemmatizer = WordNetLemmatizer()

#              Stemming              Lemmatization
# "running"  → "run"                → "run"        (both work)
# "better"   → "better"             → "good"       (lemma wins!)
# "studies"  → "studi"              → "study"      (lemma is real word)
# "flies"    → "fli"                → "fly"        (lemma is real word)

# Stemming: crude, rule-based, fast
print(stemmer.stem("studies"))       # "studi" ← not a real word!

# Lemmatization: dictionary-based, produces real words
print(lemmatizer.lemmatize("studies", pos='v'))  # "study" ← correct!
```

### 💡 Interview Insight

> **"Why did the industry move to subword tokenization?"**
> Word-level creates massive vocabularies (100K+) and can't handle unseen words. level makes sequences too long and loses semantic meaning. Subword tokenization (BPE, WordPiece) hits the sweet spot: reasonable vocabulary size (~30K-50K), handles any word (including rare/new ones), and preserves meaning for common words. It's the reason modern LLMs can handle any text, including code, URLs, and multilingual content.

---

## Screen 2: Word Embeddings — Word2Vec, GloVe & FastText

### From One-Hot to Dense Vectors

```
One-Hot (sparse, no semantics):        Dense Embedding (semantic):

"cat"  → [0, 0, 1, 0, 0, ..., 0]     "cat"  → [0.21, -0.35, 0.72, ...]
"dog"  → [0, 0, 0, 1, 0, ..., 0]     "dog"  → [0.23, -0.31, 0.69, ...]
"car"  → [0, 0, 0, 0, 1, ..., 0]     "car"  → [-0.45, 0.82, 0.11, ...]

dim = vocab_size (50,000)              dim = 100-300
cat·dog = 0 (no similarity!)          cat·dog = 0.95 (similar!)
                                        cat·car = 0.12 (dissimilar!)
```

### Word2Vec

Two architectures that learn word vectors from co-occurrence in large text corpora:

```
Skip-gram:                           CBOW:
  Given: center word                   Given: context words
  Predict: surrounding words           Predict: center word

  "The cat [sat] on the"              "The cat [???] on the"
   context ← sat → context             context → predict → sat

  Better for rare words                Better for frequent words
  More commonly used                   Faster training
```

```python
from gensim.models import Word2Vec

# Train Word2Vec on your corpus
sentences = [["the", "cat", "sat"], ["the", "dog", "ran"]]
model = Word2Vec(sentences, vector_size=100, window=5, min_count=1, sg=1)  # sg=1: Skip-gram

# Get word vector
cat_vec = model.wv['cat']  # 100-dimensional vector

# Semantic operations!
# king - man + woman ≈ queen
result = model.wv.most_similar(positive=['king', 'woman'], negative=['man'])
```

### GloVe (Global Vectors)

Unlike Word2Vec (which learns from local context windows), GloVe factorizes the **global co-occurrence matrix**.

```
Co-occurrence matrix (simplified):
         the   cat   sat   dog   ran
  the  [ 0     5     3     4     2  ]
  cat  [ 5     0     4     1     0  ]
  sat  [ 3     4     0     0     0  ]
  ...

GloVe objective: w_i · w_j + b_i + b_j = log(X_ij)
→ Learns vectors whose dot product = log probability of co-occurrence
```

```python
# Load pre-trained GloVe (download from nlp.stanford.edu)
import numpy as np

def load_glove(path, dim=100):
    embeddings = {}
    with open(path, 'r') as f:
        for line in f:
            values = line.split()
            word = values[0]
            vector = np.array(values[1:], dtype='float32')
            embeddings[word] = vector
    return embeddings

glove = load_glove('glove.6B.100d.txt')
similarity = np.dot(glove['cat'], glove['dog']) / (
    np.linalg.norm(glove['cat']) * np.linalg.norm(glove['dog'])
)
print(f"cat-dog similarity: {similarity:.4f}")  # ~0.80
```

### FastText: Subword Embeddings

FastText represents each word as a bag of character n-grams, then sums their embeddings.

```
"where" → <wh, whe, her, ere, re>, <where>

Advantage: Can generate vectors for OOV words!
"wherefrom" → uses shared n-grams with "where" and "from"
```

### Static vs Contextual Embeddings

```
Static (Word2Vec, GloVe):              Contextual (BERT, GPT):

"bank" → [0.2, -0.3, 0.5]             "river bank" → [0.1, -0.5, 0.8]
  Same vector regardless of context     "bank account" → [-0.3, 0.7, 0.2]
                                         Different vectors for different meanings!

One vector per word type                 One vector per word TOKEN in context
```

### 💡 Interview Insight

> **"What's the fundamental limitation of Word2Vec/GloVe?"**
> They produce **static** embeddings — one vector per word regardless of context. "Bank" has the same vector in "river bank" and "bank account." Contextual embeddings (BERT, GPT) solve this by computing embeddings as a function of the surrounding context, producing different vectors for different word senses. However, static embeddings are still useful for fast similarity lookups, cold-start scenarios, and as initialization for neural networks.

---

## Screen 3: Sentence & Document Embeddings

### From Word to Sentence Embeddings

```
Strategy 1: Average word embeddings (simple but weak)
  "The cat sat" → mean(embed("the"), embed("cat"), embed("sat"))
  ❌ Loses word order, treats "dog bites man" = "man bites dog"

Strategy 2: [CLS] token from BERT (common but not ideal)
  BERT("The cat sat") → hidden state of [CLS] token
  ❌ [CLS] wasn't trained for sentence similarity

Strategy 3: Sentence-BERT (purpose-built, best)
  Trained specifically for sentence similarity
  ✅ Semantically meaningful distance in embedding space
```

### Sentence-BERT Architecture

```
Sentence A: "The cat sat on the mat"    Sentence B: "A kitten rested on a rug"
          ↓                                        ↓
     ┌─────────┐                           ┌─────────┐
     │  BERT    │                           │  BERT    │  (shared weights)
     └────┬────┘                           └────┬────┘
          ↓                                        ↓
     Mean Pooling                            Mean Pooling
          ↓                                        ↓
     Embedding A                             Embedding B
          ↓                                        ↓
     ┌─────────────────────────────────────────────┐
     │  Cosine Similarity = 0.92 (very similar!)   │
     └─────────────────────────────────────────────┘

Trained with: (sentence_a, sentence_b, similarity_label) triplets
```

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Load a pre-trained sentence embedding model
model = SentenceTransformer('all-MiniLM-L6-v2')  # Fast, good quality

# Encode sentences
sentences = [
    "The cat sat on the mat",
    "A kitten rested on a rug",
    "The stock market crashed today"
]
embeddings = model.encode(sentences)  # shape: (3, 384)

# Compute cosine similarity
from sklearn.metrics.pairwise import cosine_similarity
sim_matrix = cosine_similarity(embeddings)
print(sim_matrix)
# [[1.00, 0.72, 0.08],    cat/kitten: similar!
#  [0.72, 1.00, 0.05],    kitten/stock: different!
#  [0.08, 0.05, 1.00]]
```

### Bi-Encoders vs Cross-Encoders

```
Bi-Encoder (Sentence-BERT):         Cross-Encoder:

  Sentence A → Encoder → Emb_A       Sentence A + Sentence B
  Sentence B → Encoder → Emb_B                ↓
         ↓                             ┌──────────────┐
  cosine(Emb_A, Emb_B)                │    BERT       │
                                       └──────┬───────┘
                                              ↓
                                        Score: 0.92

  ✅ Fast: encode once, compare many   ✅ More accurate (sees both at once)
  ✅ Pre-compute & index embeddings    ❌ Slow: must run for EVERY pair
  ✅ Scalable (FAISS, ANN search)      ❌ Can't pre-compute
  
  Use for: retrieval (first stage)     Use for: re-ranking (second stage)
```

| Aspect | Bi-Encoder | Cross-Encoder |
|---|---|---|
| Speed (1M comparisons) | ~seconds | ~hours |
| Accuracy | Good | Best |
| Pre-compute embeddings? | ✅ Yes | ❌ No |
| Use case | Retrieval, search | Re-ranking top results |
| Common pattern | Retrieve top-100 → re-rank with cross-encoder |

### 💡 Interview Insight

> **"How would you build a semantic search system?"**
> Two-stage architecture: (1) **Bi-encoder** (Sentence-BERT) encodes all documents offline into embeddings, stored in a vector index (FAISS, Pinecone). At query time, encode the query → ANN search → retrieve top-K candidates. (2) **Cross-encoder** re-ranks the top-K candidates for higher accuracy. This gives you the speed of bi-encoders with the accuracy of cross-encoders. This is exactly how modern RAG systems work.

---

## Screen 4: Transformer-Based NLP — BERT vs GPT

### Encoder vs Decoder Models

```
ENCODER (BERT):                      DECODER (GPT):
Bidirectional — sees full context     Causal — sees only left context

"The [MASK] sat on the mat"          "The cat" → predicts "sat"
  ←──────────────────────→            ←──────→
  Looks left AND right                Only looks left (autoregressive)

Pre-training: Masked Language Model   Pre-training: Next Token Prediction
Output: contextual embeddings         Output: probability of next token

Good for:                             Good for:
✅ Classification                     ✅ Text generation
✅ NER, token-level tasks             ✅ Chatbots, code generation
✅ Sentence embeddings                ✅ Summarization, translation
✅ Extractive QA                      ✅ Few-shot/zero-shot tasks
```

### BERT Architecture

```
Input: [CLS] The cat sat [SEP]

     Token Embeddings:     E_[CLS]  E_The  E_cat  E_sat  E_[SEP]
   + Position Embeddings:  P_0      P_1    P_2    P_3    P_4
   + Segment Embeddings:   S_A      S_A    S_A    S_A    S_A
   ─────────────────────────────────────────────────────────
   = Input Embeddings      → 12 Transformer Encoder Layers → Output

BERT-base: 12 layers, 768 hidden, 12 heads, 110M params
BERT-large: 24 layers, 1024 hidden, 16 heads, 340M params
```

```python
from transformers import AutoTokenizer, AutoModel
import torch

# Load BERT
tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
model = AutoModel.from_pretrained('bert-base-uncased')

# Tokenize
text = "Machine learning is fascinating"
inputs = tokenizer(text, return_tensors='pt', padding=True, truncation=True)

# Forward pass
with torch.no_grad():
    outputs = model(**inputs)

# outputs.last_hidden_state: (batch, seq_len, 768) — token embeddings
# outputs.pooler_output: (batch, 768) — [CLS] representation
cls_embedding = outputs.last_hidden_state[:, 0, :]  # [CLS] token
```

### GPT Architecture

```
Input: "The cat sat"

Causal attention mask (can't look ahead):
          The  cat  sat
  The   [  1    0    0  ]   ← "The" only sees itself
  cat   [  1    1    0  ]   ← "cat" sees "The" and itself
  sat   [  1    1    1  ]   ← "sat" sees everything before it

Next token prediction:
  "The"         → predict "cat"
  "The cat"     → predict "sat"
  "The cat sat" → predict "on"
```

### Encoder-Decoder: T5 and BART

```
Input: "Summarize: The cat sat on the mat. The mat was blue."
         ↓
    ┌──────────┐
    │ ENCODER   │ → Understands the full input (bidirectional)
    └────┬─────┘
         ↓ cross-attention
    ┌──────────┐
    │ DECODER   │ → Generates output autoregressively
    └────┬─────┘
         ↓
Output: "A cat sat on a blue mat."

T5: Treats EVERY task as text-to-text
  "translate English to French: Hello" → "Bonjour"
  "classify: I love this movie" → "positive"
  "summarize: ..." → "..."
```

### 💡 Interview Insight

> **"When would you use BERT vs GPT?"**
> **BERT** (encoder) when you need to understand text: classification, NER, extractive QA, sentence embeddings for search/retrieval. It sees the full context bidirectionally, making it ideal for comprehension tasks. **GPT** (decoder) when you need to generate text: chatbots, content creation, code generation, summarization. It generates tokens autoregressively. In practice, modern GPT-scale models (GPT-4, Claude) are so powerful they handle understanding tasks too via prompting, but BERT-family models are far more efficient for embedding and classification at scale.

---

## Screen 5: NLP Tasks — Classification, NER, Summarization & QA

### Text Classification

```python
from transformers import pipeline

# Zero-shot classification (no training needed!)
classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")
result = classifier(
    "The new iPhone has an incredible camera",
    candidate_labels=["technology", "sports", "politics", "food"]
)
print(result['labels'][0], result['scores'][0])  # technology, 0.97

# Sentiment analysis (pre-trained)
sentiment = pipeline("sentiment-analysis")
result = sentiment("I absolutely love this product!")
print(result)  # [{'label': 'POSITIVE', 'score': 0.9998}]
```

### Named Entity Recognition (NER)

```
Input: "Apple CEO Tim Cook announced the new iPhone in San Francisco."

Output:
  "Apple"          → ORG   (Organization)
  "Tim Cook"       → PER   (Person)
  "iPhone"         → MISC  (Miscellaneous)
  "San Francisco"  → LOC   (Location)
```

```python
ner = pipeline("ner", model="dslim/bert-base-NER", aggregation_strategy="simple")
results = ner("Apple CEO Tim Cook announced the new iPhone in San Francisco.")
for entity in results:
    print(f"{entity['word']:20s} → {entity['entity_group']} ({entity['score']:.2f})")
# Apple                → ORG (0.99)
# Tim Cook             → PER (0.99)
# San Francisco        → LOC (0.99)
```

### Summarization

```python
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")
article = """
Machine learning is a subset of artificial intelligence that focuses on building 
systems that learn from data. Unlike traditional programming where rules are 
explicitly coded, ML algorithms discover patterns in data and make predictions 
or decisions without being explicitly programmed for each specific task.
"""
summary = summarizer(article, max_length=50, min_length=20, do_sample=False)
print(summary[0]['summary_text'])
```

### Question Answering

```
Extractive QA (BERT):                 Generative QA (GPT/T5):
  Context: "Paris is the capital       Input: "What is the capital of France?"
            of France."                 
  Question: "What is the capital       Output: "The capital of France is Paris."
             of France?"               
  Answer: "Paris"  ← span from text    Answer: generated from model knowledge
  
  ✅ Grounded in context               ✅ Flexible, can synthesize
  ❌ Answer must be in text             ❌ May hallucinate
```

```python
qa = pipeline("question-answering", model="deepset/roberta-base-squad2")
result = qa(
    question="What is the capital of France?",
    context="France is a country in Europe. Paris is the capital of France."
)
print(f"Answer: {result['answer']} (confidence: {result['score']:.4f})")
# Answer: Paris (confidence: 0.9876)
```

### 💡 Interview Insight

> **"How do you choose between extractive and generative QA?"**
> **Extractive QA** (BERT/RoBERTa): when answers must be grounded in a provided document — no hallucination risk, easy to verify, great for RAG systems. **Generative QA** (T5/GPT): when you need synthesized answers, multi-hop reasoning, or answers that aren't verbatim in the text. For production systems, extractive QA + RAG is safer. For user-facing chatbots, generative QA with citation is the modern approach.

---

## Screen 6: Hugging Face Ecosystem

### The Hub: Models, Datasets & Spaces

```
┌──────────────────────────────────────────────────────────┐
│                  Hugging Face Ecosystem                    │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐      │
│  │  Model Hub   │  │  Datasets   │  │   Spaces     │      │
│  │  300K+ models│  │  70K+ sets  │  │  ML demos    │      │
│  │  Any task    │  │  Streaming  │  │  Gradio/      │     │
│  │  Any lang    │  │  Map/filter │  │  Streamlit   │      │
│  └─────────────┘  └─────────────┘  └──────────────┘      │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐      │
│  │ Transformers │  │  Tokenizers │  │  Accelerate  │      │
│  │  Main lib    │  │  Fast Rust  │  │  Multi-GPU   │      │
│  │  Auto*       │  │  tokenizers │  │  Mixed prec  │      │
│  │  pipeline()  │  │             │  │              │      │
│  └─────────────┘  └─────────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────┘
```

### The pipeline() API — Fastest Path to Inference

```python
from transformers import pipeline

# Text Classification
clf = pipeline("text-classification", model="distilbert-base-uncased-finetuned-sst-2-english")

# Summarization
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

# Translation
translator = pipeline("translation_en_to_fr", model="Helsinki-NLP/opus-mt-en-fr")

# Text Generation
generator = pipeline("text-generation", model="gpt2")

# Fill-Mask (BERT-style)
filler = pipeline("fill-mask", model="bert-base-uncased")
filler("Machine learning is [MASK].")  # → "fun", "great", "important"...

# Feature Extraction (embeddings)
extractor = pipeline("feature-extraction", model="bert-base-uncased")
```

### AutoModel Pattern

```python
from transformers import (
    AutoTokenizer, 
    AutoModel,
    AutoModelForSequenceClassification,
    AutoModelForTokenClassification,
    AutoModelForQuestionAnswering,
    AutoModelForCausalLM,
)

# One interface for ANY model on the Hub
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased", num_labels=3
)

# Tokenize → Model → Logits → Predictions
inputs = tokenizer("I love this movie", return_tensors="pt", truncation=True)
outputs = model(**inputs)
logits = outputs.logits                          # (1, num_labels)
predicted_class = logits.argmax(dim=-1).item()   # 0, 1, or 2
probabilities = torch.softmax(logits, dim=-1)    # [0.05, 0.90, 0.05]
```

### Datasets Library

```python
from datasets import load_dataset

# Load a dataset from the Hub
dataset = load_dataset("imdb")
print(dataset)
# DatasetDict({
#     train: Dataset({features: ['text', 'label'], num_rows: 25000}),
#     test:  Dataset({features: ['text', 'label'], num_rows: 25000})
# })

# Streaming (don't download the whole thing)
dataset = load_dataset("wikipedia", "20220301.en", streaming=True)

# Map / Filter / Select
def tokenize_function(examples):
    return tokenizer(examples['text'], truncation=True, padding='max_length', max_length=128)

tokenized = dataset['train'].map(tokenize_function, batched=True, num_proc=4)
filtered = dataset['train'].filter(lambda x: len(x['text']) > 100)
```

### 💡 Interview Insight

> **"How does the Hugging Face tokenizer handle variable-length inputs?"**
> Tokenizers handle this with padding and truncation: `padding=True` pads shorter sequences with `[PAD]` tokens to match the longest in the batch, `truncation=True` cuts sequences exceeding `max_length`. An `attention_mask` tensor marks real tokens (1) vs padding (0) so the model ignores padding. For efficiency, use `padding='longest'` (pad to batch max) instead of `padding='max_length'` (pad to model max) to save computation.

---

## Screen 7: End-to-End Text Classification Pipeline

### Full Fine-Tuning Workflow

```python
import torch
from datasets import load_dataset
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    TrainingArguments, Trainer
)
from sklearn.metrics import accuracy_score, f1_score
import numpy as np

# 1. Load dataset
dataset = load_dataset("imdb")

# 2. Load tokenizer and tokenize
model_name = "distilbert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)

def preprocess(examples):
    return tokenizer(
        examples['text'], 
        truncation=True, 
        padding='max_length', 
        max_length=256
    )

tokenized = dataset.map(preprocess, batched=True, num_proc=4)
tokenized = tokenized.rename_column("label", "labels")
tokenized.set_format("torch", columns=["input_ids", "attention_mask", "labels"])

# 3. Load model
model = AutoModelForSequenceClassification.from_pretrained(
    model_name, num_labels=2
)

# 4. Define metrics
def compute_metrics(eval_pred):
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)
    return {
        'accuracy': accuracy_score(labels, preds),
        'f1': f1_score(labels, preds, average='macro')
    }

# 5. Training arguments
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=32,
    learning_rate=2e-5,
    weight_decay=0.01,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    metric_for_best_model="f1",
    logging_steps=100,
    fp16=True,                       # Mixed precision
    warmup_ratio=0.1,               # Warmup first 10% of steps
    report_to="none",               # or "wandb" for experiment tracking
)

# 6. Train!
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized["train"],
    eval_dataset=tokenized["test"],
    compute_metrics=compute_metrics,
)
trainer.train()

# 7. Evaluate
results = trainer.evaluate()
print(f"Test Accuracy: {results['eval_accuracy']:.4f}")
print(f"Test F1: {results['eval_f1']:.4f}")

# 8. Save & Load
trainer.save_model("./my-sentiment-model")
# Later:
from transformers import pipeline
clf = pipeline("text-classification", model="./my-sentiment-model")
clf("This movie was absolutely brilliant!")
```

### Training Visualization

```
Epoch  Train Loss   Val Loss   Val F1    LR
─────  ──────────   ────────   ──────    ──────────
  1      0.3421      0.2187    0.9012    2.0e-5 → 1.8e-5
  2      0.1854      0.1923    0.9234    1.8e-5 → 1.0e-5
  3      0.0912      0.2104    0.9198    1.0e-5 → 0.0
                          ↑
               Val loss increasing = early stopping point
               Best model: epoch 2 (load_best_model_at_end=True)
```

### Key Hyperparameters for Fine-Tuning

| Parameter | Typical Range | Notes |
|---|---|---|
| Learning rate | 1e-5 to 5e-5 | Most critical parameter |
| Batch size | 8, 16, 32 | Larger = smoother gradients |
| Epochs | 2-5 | More = overfitting risk |
| Weight decay | 0.01 | AdamW regularization |
| Warmup ratio | 0.05-0.1 | Gradual LR increase at start |
| Max sequence length | 128-512 | Task-dependent, affects speed |

### 💡 Interview Insight

> **"Walk me through fine-tuning a pre-trained model for a new classification task."**
> (1) Choose a pre-trained model appropriate for your language/domain (DistilBERT for speed, RoBERTa for accuracy). (2) Add a classification head (linear layer) on top. (3) Tokenize data with the model's tokenizer — same vocab and special tokens. (4) Use a small learning rate (2e-5) with warmup to avoid catastrophic forgetting. (5) Train for 2-5 epochs with early stopping on validation F1. (6) Use mixed precision (fp16) to double speed. (7) Evaluate on held-out test set. The Hugging Face `Trainer` API handles most of this boilerplate.

---

## Screen 8: Advanced Topics — Embedding Models for RAG & Practical Considerations

### Choosing Embedding Models

```
Model                    Dims    Speed     Quality   Use Case
────────────────────────────────────────────────────────────────
all-MiniLM-L6-v2         384     ★★★★★    ★★★       Fast prototyping
bge-base-en-v1.5         768     ★★★★     ★★★★      Production RAG
e5-large-v2              1024    ★★★      ★★★★★     High-quality retrieval
text-embedding-3-small   1536    ★★★★     ★★★★      OpenAI API
text-embedding-3-large   3072    ★★★      ★★★★★     Best quality (API)
gte-Qwen2-7B             4096    ★★       ★★★★★★    State-of-the-art
```

### Embedding Dimensions Tradeoff

```
Smaller dimensions (384):              Larger dimensions (3072):
  ✅ Faster similarity search           ✅ More expressive
  ✅ Less storage                        ✅ Captures nuanced semantics
  ✅ Good enough for most tasks          ❌ Slower retrieval
  ❌ May miss subtle differences         ❌ More memory needed
  
  Use for: chatbots, FAQ matching        Use for: legal docs, medical, code
```

### Practical Considerations

```python
# Batch encoding for efficiency
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

# DON'T: Encode one at a time
# for doc in documents:
#     embedding = model.encode(doc)  # Slow!

# DO: Batch encode
embeddings = model.encode(
    documents,
    batch_size=64,        # Process 64 at a time
    show_progress_bar=True,
    normalize_embeddings=True,  # Unit vectors for cosine similarity
)

# Similarity with normalized embeddings = just dot product!
similarity = embeddings[0] @ embeddings[1]  # Same as cosine_similarity
```

### Token Counting Matters

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4") -> int:
    """Count tokens for cost/context window estimation."""
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

# Rule of thumb: 1 token ≈ 4 characters ≈ 0.75 words
text = "Machine learning is a subset of artificial intelligence."
tokens = count_tokens(text)
print(f"Characters: {len(text)}, Words: {len(text.split())}, Tokens: {tokens}")
# Characters: 56, Words: 8, Tokens: 9

# Context window limits:
# GPT-4: 128K tokens      BERT: 512 tokens
# Claude: 200K tokens     Llama 3: 8K-128K tokens
```

### 💡 Interview Insight

> **"How do you handle documents longer than the model's context window?"**
> Several strategies: (1) **Chunking**: Split into overlapping chunks (e.g., 512 tokens with 50-token overlap), embed each chunk, retrieve relevant chunks. (2) **Hierarchical**: Summarize sections, embed summaries. (3) **Long-context models**: Use models with larger windows (128K+). (4) **Map-reduce**: Process chunks independently, then aggregate. For RAG, chunking with overlap is the standard approach — chunk size is a critical hyperparameter (typical: 256-1024 tokens).

---

## Screen 9: Quiz & Key Takeaways

### 🧪 Quiz

**Q1:** What is BPE tokenization and why is it preferred over word-level tokenization?

<details><summary>Answer</summary>Byte-Pair Encoding starts with individual characters and iteratively merges the most frequent adjacent pairs until reaching a target vocabulary size. It's preferred because: (1) handles any word including OOV words by decomposing into known subwords, (2) keeps common words as single tokens for efficiency, (3) produces a manageable vocabulary size (30K-50K vs 100K+ for word-level), (4) works across languages without language-specific rules.</details>

**Q2:** Explain the difference between a bi-encoder and a cross-encoder. When would you use each?

<details><summary>Answer</summary>A bi-encoder encodes each text independently into a fixed vector, then compares vectors (e.g., cosine similarity). A cross-encoder takes both texts as a single input and outputs a similarity score directly. Bi-encoders are fast (pre-compute embeddings, use ANN search) but less accurate. Cross-encoders are more accurate (see both texts jointly) but can't pre-compute — must run for every pair. Use bi-encoders for retrieval (first stage), cross-encoders for re-ranking (second stage).</details>

**Q3:** You need to classify customer support tickets into 50 categories. You have 10,000 labeled examples. What approach do you take?

<details><summary>Answer</summary>Fine-tune a pre-trained model (DistilBERT or RoBERTa) with a 50-class classification head. With 10K examples (~200 per class on average), fine-tuning will significantly outperform zero-shot. Use stratified splits, learning rate 2e-5, 3-5 epochs with early stopping, and evaluate with macro F1 (since classes may be imbalanced). Consider data augmentation for underrepresented classes. If some classes have very few examples, supplement with few-shot prompting for those specific classes.</details>

**Q4:** Why does BERT use both `[CLS]` and `[SEP]` special tokens?

<details><summary>Answer</summary>`[CLS]` (classification) is prepended to every input and its final hidden state is used as an aggregate sequence representation for classification tasks. `[SEP]` (separator) marks the boundary between two segments (e.g., sentence A and sentence B) for tasks like sentence pair classification, next sentence prediction, and QA where the model needs to distinguish two inputs. During pre-training, BERT uses both for the next sentence prediction (NSP) objective.</details>

**Q5:** What's the difference between a model's context window and its embedding dimension?

<details><summary>Answer</summary>Context window = maximum number of tokens the model can process at once (e.g., 512 for BERT, 128K for GPT-4). It determines how much text you can input. Embedding dimension = the size of the vector representing each token or sentence (e.g., 768 for BERT-base, 1536 for text-embedding-3-small). It determines the richness of the representation. They are independent: a model can have a small context window but large embeddings, or vice versa.</details>

### 🎯 Key Takeaways

1. **Subword tokenization (BPE)** is the industry standard — it handles any word while keeping vocabulary manageable
2. **Static embeddings** (Word2Vec, GloVe) are fast but context-blind; **contextual embeddings** (BERT, GPT) adapt to context
3. **Sentence-BERT** produces purpose-built sentence embeddings — don't just average word vectors
4. **Bi-encoder + Cross-encoder** is the two-stage pattern for production search/retrieval systems
5. **BERT = understanding**, **GPT = generation** — choose the right architecture for your task
6. **Hugging Face pipeline()** gets you from zero to working NLP in 3 lines of code
7. **Fine-tuning** with small LR (2e-5) + warmup + early stopping is the recipe for adapting pre-trained models
8. **Token counting matters** — it determines cost, latency, and whether your text fits the context window
9. **Chunking with overlap** is the standard approach for handling long documents in RAG systems
