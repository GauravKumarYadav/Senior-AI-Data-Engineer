# Module 4: The Interview Gauntlet 🔥

> **Duration:** ~3 hours | **Difficulty:** Advanced | **Prerequisites:** Modules 1-3
> Battle-tested questions that mirror real AI/ML interviews. No hand-holding — this is the real deal.

---

## Screen 1: ML Design Questions

These questions test your ability to think end-to-end: from problem framing to deployment.

---

### Q1: "Design a model to detect fraudulent credit card transactions."

**Strong Answer Framework:**

```
1. PROBLEM FRAMING
   - Binary classification: fraud (1) vs legitimate (0)
   - Extreme class imbalance: ~0.1-0.5% fraud rate
   - Real-time constraint: must score transactions in <100ms

2. DATA & FEATURES
   Transaction features:
   ├── Amount, merchant category, time of day, location
   ├── Velocity features: # transactions in last 1hr, 24hr
   ├── Deviation features: amount vs user's average, new merchant?
   ├── Geographic: distance from last transaction, unusual country
   └── Behavioral: time since last transaction, unusual category

3. MODEL SELECTION
   - XGBoost/LightGBM for tabular data (NOT deep learning)
   - Why not neural nets? Tabular data, need interpretability, fast inference
   - Ensemble: stack multiple models for robustness

4. HANDLING IMBALANCE
   - DON'T just oversample — use SMOTE or class_weight='balanced'
   - Optimize for AUC-PR, not accuracy
   - Set decision threshold based on business cost:
     cost_FN (missed fraud) >> cost_FP (blocked legit transaction)

5. EVALUATION
   - Metric: AUC-PR (primary), recall@precision=0.95
   - Business metric: $ fraud prevented vs $ legitimate blocked
   - Confusion matrix analysis with dollar amounts, not just counts

6. DEPLOYMENT
   - Real-time scoring via feature store + model server
   - A/B test against rule-based system
   - Monitor for concept drift (fraud patterns evolve!)
   - Human-in-the-loop for edge cases (uncertain predictions)
```

---

### Q2: "Build a recommendation system for an e-commerce platform."

**Key Design Decisions:**

```
Approach               Pros                        Cons
──────────────────────────────────────────────────────────────
Collaborative Filtering  Finds surprising items     Cold start problem
(user-user / item-item)  No content understanding   Popularity bias
                         needed                     

Content-Based            No cold start for items    Limited discovery
(item features)          Explainable                Filter bubble

Hybrid (production)      Best of both               More complex
Matrix Factorization     Scales well                Training complexity
+ Content features       

Two-Tower (modern)       Embedding-based            Needs lots of data
  User encoder → emb     Handles new items via      Training infrastructure
  Item encoder → emb     content tower
  Score = dot product     

LLM-based                Rich understanding         Expensive inference
(emerging)               Multi-modal                Latency concerns
```

**Feature Engineering for Recommendations:**

```python
# User features: purchase history embedding, browse categories, price sensitivity
# Item features: category, price range, description embedding, popularity
# Interaction features: time-weighted purchase history, clicks/impressions ratio
# Context features: time of day, device, season, recent searches

# Two-stage architecture:
# Stage 1: Candidate generation (retrieve ~1000 items from millions)
#   → Approximate Nearest Neighbors on embeddings (FAISS)
# Stage 2: Ranking (score ~1000 candidates, return top 20)
#   → LightGBM/neural ranker with rich features
```

---

### Q3: "Design an ML system to predict customer churn."

**Answer Highlights:**
- **Target definition matters:** When is a customer "churned"? No activity in 30 days? 90 days? This is a business decision that dramatically affects model performance.
- **Feature windows:** Use features from a "lookback window" (e.g., last 90 days) to predict churn in a "prediction window" (e.g., next 30 days). Never include data from the prediction window!
- **Key features:** Login frequency trend, support ticket frequency, feature usage decline, billing changes, contract renewal date proximity.
- **Model:** Survival analysis or gradient boosting (XGBoost) with calibrated probabilities.
- **Action:** Rank customers by P(churn) × customer_value. Target high-value, high-risk customers with retention offers.

---

### Q4: "Design an email spam classifier."

**Answer Highlights:**
- **Features:** TF-IDF on email body, sender reputation, header analysis, URL analysis, attachment types, HTML/text ratio.
- **Model:** Start simple (Logistic Regression + TF-IDF), then gradient boosted trees.
- **Precision > Recall:** False positives (legitimate email in spam) are worse than false negatives (spam in inbox). Optimize precision at a recall threshold.
- **Deployment:** Real-time scoring at ingestion. User feedback loop (mark as spam/not spam) for continuous learning.
- **Adversarial:** Spammers adapt! Need regular retraining, concept drift monitoring.

---

### Q5: "You have tabular data with 500 features and 10K rows. Walk through your approach."

```
Step 1: EDA
  - Check class distribution, missing values, feature types
  - Correlation matrix → drop highly correlated features (>0.95)

Step 2: Feature Selection (500 → ~50-100)
  - Remove zero-variance features
  - Mutual information scores
  - L1 (Lasso) for automatic selection
  - Recursive Feature Elimination (RFE) with XGBoost

Step 3: Model Selection
  - Baseline: Logistic Regression (always!)
  - Primary: XGBoost/LightGBM (handles wide data well)
  - With 10K rows and 500 features, regularization is critical

Step 4: Validation
  - 5-fold stratified CV (10K rows = too few for hold-out)
  - Pipeline to prevent leakage in preprocessing

Step 5: Feature Importance
  - SHAP values for interpretability
  - Permutation importance for validation
```

---

### Q6: "Design a content moderation system for user-generated text."

**Answer Highlights:**
- **Multi-label classification:** Content can be toxic AND sexually explicit AND threatening simultaneously.
- **Hierarchical categories:** Safe → Borderline → Toxic → Severe. Different thresholds per category.
- **Model:** Fine-tuned RoBERTa for text. Multi-modal model if images/video are involved.
- **Challenges:** Context matters ("kill it!" in gaming vs threat), sarcasm, coded language, multilingual.
- **Human-in-the-loop:** Uncertain cases go to human reviewers. Active learning to improve on edge cases.
- **Latency:** Real-time for chat, batch for posts. Use distilled model for latency-sensitive paths.

---

### Q7: "How would you build a search ranking model?"

**Key Concepts:**
- **Learning to Rank (LTR):** Pointwise (predict relevance score), Pairwise (predict which doc is more relevant), Listwise (optimize entire ranking).
- **Features:** BM25 score, semantic similarity (bi-encoder), click-through rate, freshness, authority, query-document overlap.
- **Two-stage:** Retrieval (BM25 + semantic search) → Re-ranking (cross-encoder or LambdaMART).
- **Metrics:** NDCG@10, MRR, MAP — not accuracy or F1!
- **Online metrics:** Click-through rate, dwell time, query reformulation rate.

---

### Q8: "Design a demand forecasting model for a retail chain."

**Answer Highlights:**
- **Time series problem** but with rich features → gradient boosted trees often beat traditional time series models.
- **Features:** Day of week, month, holidays, promotions, weather, historical sales (lag features, rolling averages), store-level features, product hierarchy.
- **Hierarchy:** Forecast at SKU × Store × Day level, then reconcile up to category/region.
- **Model:** LightGBM with temporal features > ARIMA for high-dimensional retail forecasting.
- **Evaluation:** WMAPE (Weighted Mean Absolute Percentage Error), evaluated at multiple aggregation levels.
- **Deployment:** Retrain weekly, daily inference, alert on anomalous predictions.

---

## Screen 2: Evaluation & Metrics — Tricky Questions

### Q1: "Your model has 95% accuracy on a binary classification task. Is it good?"

**Answer:** It depends entirely on the class distribution.

```
Scenario A: 50/50 balance → 95% accuracy is excellent!
Scenario B: 95/5 balance → 95% accuracy = predicting majority class always
                            (the model learned NOTHING)

Always ask: What's the baseline? What's the class distribution?
Always look at: Confusion matrix, precision, recall, AUC-PR
```

---

### Q2: "When would you optimize for precision vs recall?"

```
Optimize PRECISION (minimize false positives):
  - Spam filtering: blocking legit email = bad UX
  - Content recommendation: irrelevant suggestions = user frustration
  - Drug approval screening: false approvals = patient risk

Optimize RECALL (minimize false negatives):
  - Cancer screening: missing cancer = patient harm
  - Fraud detection: missing fraud = financial loss
  - Security threat detection: missing threats = breach

The F1 score balances both. F-beta lets you weight:
  Fβ = (1 + β²) × (P × R) / (β² × P + R)
  β > 1: emphasize recall
  β < 1: emphasize precision
```

---

### Q3: "Your AUC-ROC is 0.99 but AUC-PR is 0.15. What's happening?"

**Answer:**

```
This happens with extreme class imbalance (e.g., 99.9% negative).

AUC-ROC = 0.99: The model ranks positives above negatives well.
  BUT it operates on TPR vs FPR, and FPR = FP / (FP + TN).
  When TN is massive, even many false positives barely move FPR.
  → AUC-ROC is inflated and misleading!

AUC-PR = 0.15: When you actually look at precision vs recall,
  the model has terrible precision (lots of false positives relative
  to the small number of true positives).

Lesson: For imbalanced data, ALWAYS use AUC-PR as your primary metric.
```

---

### Q4: "How do you evaluate a ranking model (search, recommendations)?"

```
NDCG@K (Normalized Discounted Cumulative Gain):
  - Accounts for position: top results matter more
  - Accounts for graded relevance (not just binary)
  - DCG = Σ (2^relevance - 1) / log₂(position + 1)
  - NDCG = DCG / ideal_DCG

MRR (Mean Reciprocal Rank):
  - How high is the FIRST relevant result?
  - MRR = average of 1/rank_of_first_relevant across queries

MAP (Mean Average Precision):
  - Average precision at each relevant position
  - Good for binary relevance judgments

Online metrics (A/B test):
  - Click-through rate (CTR)
  - Conversion rate
  - Session length / dwell time
  - Return rate / query reformulation
```

---

### Q5: "How do you know if your model is well-calibrated?"

**Answer:** A well-calibrated model's predicted probabilities match actual frequencies. If the model says "80% probability of positive," ~80% of those cases should actually be positive.

```python
from sklearn.calibration import calibration_curve, CalibratedClassifierCV
import matplotlib.pyplot as plt

# Check calibration
prob_true, prob_pred = calibration_curve(y_test, y_proba, n_bins=10)

# Perfect calibration = diagonal line
# Below diagonal = overconfident
# Above diagonal = underconfident

# Fix calibration with Platt scaling or isotonic regression
calibrated_model = CalibratedClassifierCV(model, method='isotonic', cv=5)
calibrated_model.fit(X_train, y_train)
```

---

### Q6: "Your model performs great on validation but poorly in production. What happened?"

```
Root Causes (ranked by likelihood):
1. Data drift: production data distribution differs from training
2. Data leakage: accidentally used future data or target info in features
3. Preprocessing mismatch: different scaling/encoding in training vs serving
4. Selection bias: training data not representative of production traffic
5. Concept drift: the relationship between features and target changed
6. Feedback loops: model predictions influence future data (self-fulfilling)

Debugging steps:
1. Compare feature distributions: training vs production
2. Check for feature leakage: mutual information with target
3. Verify preprocessing pipeline: exact same transforms?
4. Evaluate on recent data: does performance degrade over time?
5. Shadow mode: run new model alongside old, compare outputs
```

---

### Q7: "How do you set a classification threshold?"

```
Default threshold = 0.5, but this is almost never optimal.

Better approaches:
1. Cost-based: threshold = C_FP / (C_FP + C_FN)
   If missing fraud costs 100x more than blocking legit → threshold ≈ 0.01

2. Precision-at-recall: "I need 95% recall, what precision can I get?"
   Plot PR curve → find threshold where recall ≥ 0.95

3. F1-optimal: threshold that maximizes F1 on validation set

4. Business constraint: "No more than 1% false positive rate"
   → Find threshold on ROC curve where FPR ≤ 0.01
```

---

### Q8: "Explain Type I vs Type II errors with a real example."

```
Context: Medical test for a disease

Type I Error (False Positive):         Type II Error (False Negative):
  Test says: POSITIVE                    Test says: NEGATIVE
  Reality:   HEALTHY                     Reality:   SICK

  Impact: Unnecessary treatment,         Impact: Missed diagnosis,
  anxiety, cost                          disease progression, death

  Controlled by: PRECISION               Controlled by: RECALL
  (or specificity)                       (or sensitivity)

  Which is worse? Depends on context!
  Cancer: Type II is worse (miss cancer)
  Spam:   Type I is worse (block real email)
```

---

## Screen 3: Deep Learning & NLP Questions

### Q1: "Explain the attention mechanism. Why is it better than RNNs?"

```
Attention lets each token directly attend to every other token,
regardless of distance. No more sequential bottleneck.

RNN problems that attention solves:
1. Sequential processing: RNN must process token by token → can't parallelize
   Attention: all tokens processed simultaneously → GPU parallelism

2. Vanishing gradients: Long-range dependencies lost after many steps
   Attention: direct connection between any two tokens → O(1) path length

3. Information bottleneck: RNN compresses entire input into fixed hidden state
   Attention: every token has access to every other token's representation

Cost: O(n²) memory and compute for sequence length n
→ Why context windows are limited (quadratic scaling)
→ Solutions: Flash Attention, sparse attention, linear attention
```

---

### Q2: "What's the difference between self-attention and cross-attention?"

```
Self-attention: Q, K, V all come from the SAME input
  "The cat sat on the mat"
  Each word attends to every other word in the same sequence
  Used in: BERT encoder, GPT decoder (masked)

Cross-attention: Q from one source, K/V from another
  Decoder Q → attends to → Encoder K, V
  "How much should each output word focus on each input word?"
  Used in: T5/BART decoder layers, image captioning
```

---

### Q3: "Why do we need positional encoding in Transformers?"

**Answer:** Self-attention is permutation-equivariant — it treats the input as a set, not a sequence. Without positional encoding, "dog bites man" and "man bites dog" produce identical representations. Positional encoding injects order information.

```
Original (sinusoidal): Fixed, deterministic, generalizes to longer sequences
Learned (BERT): Trainable, limited to max_position_embeddings
RoPE (Llama, modern): Encodes RELATIVE position through rotation, 
                       supports extrapolation to longer sequences
ALiBi: Adds position-dependent bias to attention scores
```

---

### Q4: "Explain how BERT is pre-trained."

```
Two pre-training objectives:

1. Masked Language Modeling (MLM):
   - Randomly mask 15% of tokens
   - Of those: 80% replaced with [MASK], 10% random token, 10% unchanged
   - Model predicts original tokens
   - Forces bidirectional understanding

   Input:  "The [MASK] sat on the [MASK]"
   Target: "The  cat   sat on the  mat"

2. Next Sentence Prediction (NSP):
   - Given two sentences: are they consecutive?
   - [CLS] A [SEP] B [SEP] → binary classification
   - 50% real pairs, 50% random pairs
   - (Later work showed NSP is not very useful → RoBERTa dropped it)

Pre-training data: BooksCorpus + English Wikipedia (~16GB text)
Pre-training cost: 4 days on 16 TPUs
```

---

### Q5: "What is the vanishing gradient problem and how do modern architectures solve it?"

```
Problem: In deep networks, gradients shrink exponentially as they
propagate backward through many layers.

∂Loss/∂w₁ = ∂Loss/∂hₙ · ∂hₙ/∂hₙ₋₁ · ... · ∂h₂/∂h₁ · ∂h₁/∂w₁
             └─────────────── Many terms < 1 ───────────────┘
             Product → 0 (gradients vanish)

Solutions:
1. ReLU activation: gradient is 1 for positive inputs (no shrinkage)
2. Residual connections: gradient highway bypasses layers
3. Layer normalization: stabilizes activations at each layer
4. LSTM gates: explicit memory cell with gating (historical)
5. Careful initialization: He/Kaiming for ReLU networks
6. Gradient clipping: prevents exploding gradients
```

---

### Q6: "How does a causal attention mask work in GPT?"

```
For sequence "A B C D":

Attention mask:
       A    B    C    D
  A [  1    0    0    0  ]   A can only see A
  B [  1    1    0    0  ]   B can see A, B
  C [  1    1    1    0  ]   C can see A, B, C
  D [  1    1    1    1  ]   D can see everything

Implementation: set masked positions to -infinity BEFORE softmax
  softmax(-∞) = 0 → effectively zero attention to future tokens

This ensures the model can only use past context when predicting
the next token — matching the autoregressive generation process.
```

---

### Q7: "What is catastrophic forgetting and how do you mitigate it?"

**Answer:** When fine-tuning a pre-trained model on a new task, it can "forget" its pre-trained knowledge, degrading performance on the original capabilities.

```
Mitigation strategies:
1. Small learning rate (2e-5): Gentle updates preserve pre-trained weights
2. Warmup schedule: Gradually increase LR, don't shock the weights
3. Gradual unfreezing: Start with head only, slowly unfreeze deeper layers
4. Elastic Weight Consolidation (EWC): Penalize changes to important weights
5. LoRA/QLoRA: Only train small adapter layers, freeze original weights
6. Multi-task learning: Train on new + original tasks simultaneously
```

---

### Q8: "Explain the difference between fine-tuning and LoRA."

```
Full Fine-Tuning:                    LoRA (Low-Rank Adaptation):
┌────────────────────┐               ┌────────────────────┐
│  All weights       │               │  Original weights  │ ← FROZEN
│  updated           │               │  W (d × d)         │
│  W_new = W + ΔW    │               ├────────────────────┤
│                    │               │  Small adapters     │ ← TRAINED
│  Parameters: ALL   │               │  ΔW = A × B        │
│  (e.g., 7B)        │               │  A (d × r), B (r × d) │
└────────────────────┘               │  r << d (e.g., 8)  │
                                     │  Parameters: ~0.1% │
Memory: ~28GB (7B × 4 bytes)         └────────────────────┘
                                     Memory: ~4GB (QLoRA with 4-bit)

LoRA intuition: weight changes during fine-tuning are LOW RANK.
We can approximate ΔW ≈ A × B where A and B are tiny matrices.
Rank r=8 or 16 captures most of the task-specific information.
```

---

## Screen 4: Practical Coding Scenarios

### Scenario 1: "Write a scikit-learn pipeline for a classification task with mixed feature types."

```python
import pandas as pd
import numpy as np
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score, StratifiedKFold

# Identify column types
numeric_features = ['age', 'income', 'credit_score', 'num_transactions']
categorical_features = ['state', 'product_type', 'channel']

# Build preprocessor
preprocessor = ColumnTransformer([
    ('num', Pipeline([
        ('imputer', SimpleImputer(strategy='median')),
        ('scaler', StandardScaler()),
    ]), numeric_features),
    ('cat', Pipeline([
        ('imputer', SimpleImputer(strategy='most_frequent')),
        ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False)),
    ]), categorical_features),
])

# Full pipeline
pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', GradientBoostingClassifier(
        n_estimators=200, max_depth=5, learning_rate=0.1, random_state=42
    ))
])

# Evaluate with cross-validation
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(pipeline, X, y, cv=cv, scoring='f1_macro', n_jobs=-1)
print(f"F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

---

### Scenario 2: "Implement a PyTorch training loop with early stopping."

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader
import copy

class EarlyStopping:
    def __init__(self, patience=5, min_delta=1e-4):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss = float('inf')
        self.best_model = None
    
    def __call__(self, val_loss, model):
        if val_loss < self.best_loss - self.min_delta:
            self.best_loss = val_loss
            self.best_model = copy.deepcopy(model.state_dict())
            self.counter = 0
            return False  # Don't stop
        self.counter += 1
        return self.counter >= self.patience  # Stop if patience exceeded

def train(model, train_loader, val_loader, epochs=50, lr=1e-3):
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = model.to(device)
    optimizer = torch.optim.AdamW(model.parameters(), lr=lr, weight_decay=0.01)
    criterion = nn.CrossEntropyLoss()
    scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
    early_stopping = EarlyStopping(patience=5)
    
    for epoch in range(epochs):
        # Train
        model.train()
        train_loss = 0
        for X_batch, y_batch in train_loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            optimizer.zero_grad()
            logits = model(X_batch)
            loss = criterion(logits, y_batch)
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()
            train_loss += loss.item()
        
        # Validate
        model.eval()
        val_loss = 0
        with torch.no_grad():
            for X_batch, y_batch in val_loader:
                X_batch, y_batch = X_batch.to(device), y_batch.to(device)
                logits = model(X_batch)
                val_loss += criterion(logits, y_batch).item()
        
        train_loss /= len(train_loader)
        val_loss /= len(val_loader)
        scheduler.step()
        
        print(f"Epoch {epoch+1}: Train={train_loss:.4f}, Val={val_loss:.4f}")
        
        if early_stopping(val_loss, model):
            print(f"Early stopping at epoch {epoch+1}")
            model.load_state_dict(early_stopping.best_model)
            break
    
    return model
```

---

### Scenario 3: "Fine-tune a Hugging Face model for sentiment analysis."

```python
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    TrainingArguments, Trainer
)
from datasets import load_dataset
import numpy as np
from sklearn.metrics import accuracy_score, f1_score

# Load data and model
dataset = load_dataset("imdb")
model_name = "distilbert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Tokenize
def tokenize(batch):
    return tokenizer(batch['text'], truncation=True, padding='max_length', max_length=256)

tokenized = dataset.map(tokenize, batched=True)
tokenized = tokenized.rename_column("label", "labels")
tokenized.set_format("torch", columns=["input_ids", "attention_mask", "labels"])

# Model + metrics + train
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)

def compute_metrics(eval_pred):
    preds = np.argmax(eval_pred.predictions, axis=-1)
    return {
        'accuracy': accuracy_score(eval_pred.label_ids, preds),
        'f1': f1_score(eval_pred.label_ids, preds, average='macro'),
    }

trainer = Trainer(
    model=model,
    args=TrainingArguments(
        output_dir="./results", num_train_epochs=3, learning_rate=2e-5,
        per_device_train_batch_size=16, evaluation_strategy="epoch",
        save_strategy="epoch", load_best_model_at_end=True,
        metric_for_best_model="f1", fp16=True, warmup_ratio=0.1,
    ),
    train_dataset=tokenized["train"].select(range(5000)),  # Subset for demo
    eval_dataset=tokenized["test"].select(range(1000)),
    compute_metrics=compute_metrics,
)
trainer.train()
```

---

### Scenario 4: "Write a custom sklearn transformer that creates interaction features."

```python
from sklearn.base import BaseEstimator, TransformerMixin
import numpy as np
import pandas as pd
from itertools import combinations

class InteractionFeatures(BaseEstimator, TransformerMixin):
    """Creates pairwise multiplication features for specified columns."""
    
    def __init__(self, columns=None, max_interactions=10):
        self.columns = columns
        self.max_interactions = max_interactions
    
    def fit(self, X, y=None):
        if self.columns is None:
            self.columns = list(range(X.shape[1]))
        # Select top interactions by correlation with target (if y provided)
        self.pairs_ = list(combinations(self.columns, 2))[:self.max_interactions]
        return self
    
    def transform(self, X):
        X = np.array(X)
        interactions = []
        for i, j in self.pairs_:
            interactions.append(X[:, i] * X[:, j])
        return np.column_stack([X] + interactions) if interactions else X
    
    def get_feature_names_out(self, input_features=None):
        base = input_features or [f"x{i}" for i in range(len(self.columns))]
        interaction_names = [f"{base[i]}_x_{base[j]}" for i, j in self.pairs_]
        return list(base) + interaction_names
```

---

### Scenario 5: "Implement cosine similarity search over a corpus using sentence embeddings."

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticSearch:
    def __init__(self, model_name='all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
        self.corpus_embeddings = None
        self.corpus = None
    
    def index(self, documents: list[str], batch_size=64):
        """Encode and store document embeddings."""
        self.corpus = documents
        self.corpus_embeddings = self.model.encode(
            documents, batch_size=batch_size,
            normalize_embeddings=True, show_progress_bar=True
        )
    
    def search(self, query: str, top_k: int = 5) -> list[dict]:
        """Find top-k most similar documents to query."""
        query_embedding = self.model.encode(
            [query], normalize_embeddings=True
        )
        # Cosine similarity = dot product (since normalized)
        scores = query_embedding @ self.corpus_embeddings.T
        scores = scores[0]
        
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [
            {"document": self.corpus[i], "score": float(scores[i])}
            for i in top_indices
        ]

# Usage
searcher = SemanticSearch()
searcher.index([
    "Python is a programming language",
    "Machine learning uses data to learn patterns",
    "The weather is sunny today",
    "Neural networks are inspired by the brain",
])
results = searcher.search("deep learning algorithms")
for r in results:
    print(f"{r['score']:.4f}: {r['document']}")
```

---

### Scenario 6: "Write a function to compute and visualize a confusion matrix."

```python
import numpy as np
from sklearn.metrics import confusion_matrix, classification_report

def analyze_predictions(y_true, y_pred, class_names=None):
    """Full prediction analysis with confusion matrix and metrics."""
    cm = confusion_matrix(y_true, y_pred)
    n_classes = cm.shape[0]
    class_names = class_names or [f"Class {i}" for i in range(n_classes)]
    
    # Print confusion matrix as ASCII
    max_width = max(len(name) for name in class_names) + 2
    header = " " * (max_width + 2) + "  ".join(f"{name:>{max_width}}" for name in class_names)
    print(f"\n{'Confusion Matrix':^{len(header)}}")
    print(f"{'(rows=actual, cols=predicted)':^{len(header)}}")
    print(header)
    print("-" * len(header))
    for i, row in enumerate(cm):
        cells = "  ".join(f"{val:>{max_width}}" for val in row)
        print(f"{class_names[i]:>{max_width}} | {cells}")
    
    # Classification report
    print(f"\n{classification_report(y_true, y_pred, target_names=class_names, digits=4)}")
    
    # Per-class insights
    for i, name in enumerate(class_names):
        tp = cm[i, i]
        fn = cm[i, :].sum() - tp
        fp = cm[:, i].sum() - tp
        if fn > 0:
            worst_confused = class_names[np.argmax(np.delete(cm[i, :], i))]
            print(f"  ⚠ {name}: most confused with {worst_confused} ({fn} false negatives)")

# Usage
analyze_predictions(y_test, y_pred, class_names=["Negative", "Neutral", "Positive"])
```

---

## Screen 5: Rapid Fire — 20 Q&A Pairs

Answer each in 2-3 sentences max. These test breadth and precision.

---

**1. What is the bias-variance tradeoff?**
Bias is error from underfitting (model too simple); variance is error from overfitting (model too complex). Total error = bias² + variance + irreducible noise. The goal is finding the model complexity sweet spot.

**2. Why is cross-validation better than a single train/test split?**
It uses all data for both training and validation across K folds, giving K estimates of performance. The mean ± std tells you both performance and stability. Especially important with small datasets where a single split may be unrepresentative.

**3. What does L1 regularization do that L2 doesn't?**
L1 drives some weights to exactly zero, performing automatic feature selection. L2 only shrinks weights toward zero but never makes them exactly zero. Use L1 when you suspect many features are irrelevant.

**4. How does XGBoost differ from Random Forest?**
Random Forest uses bagging (parallel trees, majority vote) to reduce variance. XGBoost uses boosting (sequential trees, each corrects previous errors) to reduce bias. XGBoost also includes built-in regularization and handles missing values natively.

**5. What is data leakage?**
When information from outside the training set (test data, future data, or the target variable) leaks into the model during training. Common causes: fitting scalers on full data, using features derived from the target, not respecting time order. Pipelines prevent most preprocessing leakage.

**6. What activation function do Transformers use?**
GELU (Gaussian Error Linear Unit) in the feed-forward layers. It's a smooth approximation of ReLU that allows small negative values to pass through, improving gradient flow. Some newer models use SwiGLU (Swish-Gated Linear Unit).

**7. What is the purpose of the attention mask in Transformers?**
It tells the model which tokens to attend to and which to ignore. Padding tokens get mask=0 so they don't influence the output. In causal (GPT) models, the mask also prevents attending to future tokens.

**8. Why do we use `AdamW` instead of `Adam` for Transformers?**
AdamW decouples weight decay from the gradient update, applying it directly to weights rather than through the gradient. This leads to better generalization. Standard Adam's weight decay interacts with the adaptive learning rate in unintended ways.

**9. What is the difference between `model.train()` and `model.eval()` in PyTorch?**
`train()` enables dropout (randomly zeros neurons) and uses batch statistics for BatchNorm. `eval()` disables dropout and uses stored running statistics for BatchNorm. Always switch to eval before inference.

**10. Explain BPE tokenization in one sentence.**
BPE starts with individual characters and iteratively merges the most frequent adjacent pair into a new token until reaching the target vocabulary size.

**11. What's the difference between BERT and GPT?**
BERT is an encoder that sees the full context bidirectionally (great for understanding/classification). GPT is a decoder that only sees left context autoregressively (great for generation). BERT uses masked language modeling; GPT uses next token prediction.

**12. What is a learning rate warmup?**
Gradually increasing the learning rate from near-zero to the target value over the first few hundred/thousand steps. This prevents large, destructive weight updates at the start when gradients are noisy and the model hasn't stabilized.

**13. When would you use AUC-PR over AUC-ROC?**
When classes are imbalanced. AUC-ROC can be misleadingly high because the huge number of true negatives makes the false positive rate look small. AUC-PR focuses on the positive class, giving a more honest picture of performance.

**14. What is gradient clipping and why is it used?**
Limiting the magnitude of gradients during backpropagation (typically max_norm=1.0). It prevents exploding gradients that cause training instability, especially in deep networks and recurrent models. It doesn't change gradient direction, only scales if too large.

**15. What is the cold start problem in recommender systems?**
New users have no interaction history, and new items have no ratings — the system can't make personalized recommendations. Solutions: content-based features (item descriptions), popularity-based fallback, asking users for preferences during onboarding.

**16. What is SHAP and why is it useful?**
SHAP (SHapley Additive exPlanations) provides consistent, theoretically grounded feature attributions for individual predictions. It tells you exactly how much each feature pushed the prediction up or down. Essential for model interpretability and debugging.

**17. What is mixed precision training?**
Using float16 for forward/backward passes (faster, less memory) and float32 for weight updates (numerical stability). Achieves ~2x speedup and ~50% memory reduction with minimal accuracy impact. Enabled via `torch.cuda.amp`.

**18. What is the difference between a bi-encoder and a cross-encoder?**
Bi-encoder encodes texts independently (fast, scalable, pre-computable). Cross-encoder processes both texts jointly (more accurate, but O(n²) comparisons). Use bi-encoder for retrieval, cross-encoder for re-ranking.

**19. How does dropout work at inference time?**
It doesn't — dropout is disabled during inference. All neurons are active, but their outputs are scaled by (1 - dropout_rate) to maintain the same expected output magnitude as during training. PyTorch handles this automatically when you call `model.eval()`.

**20. What is LoRA and why does it matter?**
Low-Rank Adaptation freezes the original model weights and trains small rank-decomposed matrices (ΔW = A × B where rank r ≪ d). It reduces trainable parameters by ~99%, enabling fine-tuning of large models on consumer GPUs. QLoRA adds 4-bit quantization for even more savings.

---

## Screen 6: Key Takeaways — Your Interview Cheat Sheet

### The 10 Commandments of ML Interviews

```
 1. ALWAYS start with the problem, not the model
    → "What are we optimizing for? What's the business metric?"

 2. ALWAYS ask about the data
    → "How much labeled data? Class balance? Feature types?"

 3. NEVER say "accuracy" for imbalanced problems
    → AUC-PR, F1, recall — show you know WHY

 4. ALWAYS mention data leakage prevention
    → Pipelines, proper splits, temporal awareness

 5. KNOW the bias-variance tradeoff cold
    → Every regularization technique maps to this concept

 6. TREE MODELS dominate tabular data
    → XGBoost/LightGBM + Optuna is the winning formula

 7. TRANSFORMERS dominate text and language
    → BERT for understanding, GPT for generation

 8. ALWAYS discuss evaluation BEFORE modeling
    → Metric selection drives every decision downstream

 9. MENTION deployment considerations
    → Latency, throughput, monitoring, drift detection

10. BE HONEST about tradeoffs
    → "This approach is simpler but sacrifices X"
```

### Quick Reference: Model Selection

```
Data Type        Small Data (<1K)     Medium (1K-100K)      Large (100K+)
──────────────────────────────────────────────────────────────────────────
Tabular          Logistic Reg,        XGBoost/LightGBM      XGBoost/LightGBM
                 Random Forest                               + feature eng.

Text             Zero-shot (LLM),     Fine-tune DistilBERT  Fine-tune RoBERTa
                 Few-shot             or BERT                or domain LLM

Image            Transfer learning    Fine-tune ResNet/      Train from scratch
                 (frozen backbone)    EfficientNet           or fine-tune ViT

Sequence         Statistical models   LSTM / Transformer     Transformer
(time series)    (ARIMA, Prophet)     hybrid                 + rich features
```

### Quick Reference: Metric Selection

```
Task                      Primary Metric        Secondary
──────────────────────────────────────────────────────────
Balanced classification   F1-macro              Accuracy
Imbalanced classification AUC-PR                Recall, Precision
Regression                RMSE                  MAE, R²
Ranking / Search          NDCG@K                MRR, MAP
Probabilistic             Log-loss              Calibration curve
Recommendation            Hit Rate@K            NDCG, Coverage
```

### The Interview Flow

```
1. CLARIFY: "What exactly are we trying to predict/optimize?"
2. DATA:    "What data do we have? Volume, features, quality?"
3. METRIC:  "How do we measure success? What's the business cost of errors?"
4. BASELINE: "What's the simplest model that could work?"
5. MODEL:   "Here's my approach and why..." (tradeoffs!)
6. EVAL:    "Here's how I'd validate and prevent leakage..."
7. DEPLOY:  "Here's how I'd productionize and monitor..."
8. ITERATE: "Here's how I'd improve over time..."
```

### Final Words

```
The best ML engineers don't just know algorithms —
they know WHEN to use them, WHY to use them,
and HOW to evaluate them honestly.

Every model is wrong. Some models are useful.
Your job is to find the useful ones and
explain WHY they're useful to non-technical stakeholders.

Now go crush that interview. 🔥
```
