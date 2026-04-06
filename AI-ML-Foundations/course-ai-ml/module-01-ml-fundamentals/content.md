# Module 1: ML Fundamentals

> **Duration:** ~4 hours | **Difficulty:** Intermediate | **Prerequisites:** Python basics, NumPy, Pandas
> Master the building blocks that every ML interview question builds upon.

---

## Screen 1: The ML Landscape — Supervised, Unsupervised & Self-Supervised

### Supervised Learning

Supervised learning is the workhorse of production ML. You have labeled data — inputs paired with known outputs — and the model learns the mapping function `f(X) → y`.

**Classification vs Regression:**

| Aspect | Classification | Regression |
|---|---|---|
| Output | Discrete categories | Continuous values |
| Example | Spam / not spam | House price prediction |
| Loss function | Cross-entropy | MSE / MAE |
| Metrics | Accuracy, F1, AUC-ROC | RMSE, R², MAPE |
| Decision boundary | Separating hyperplane | Best-fit curve |

```python
from sklearn.linear_model import LogisticRegression, LinearRegression

# Classification: predict category
clf = LogisticRegression()
clf.fit(X_train, y_train)           # y_train: [0, 1, 1, 0, ...]
predictions = clf.predict(X_test)   # outputs: [0, 1, ...]

# Regression: predict continuous value
reg = LinearRegression()
reg.fit(X_train, y_train)           # y_train: [250000, 340000, ...]
predictions = reg.predict(X_test)   # outputs: [275340.50, ...]
```

### Unsupervised Learning

No labels — the model discovers hidden structure in data.

```
Unsupervised Learning
├── Clustering         → Group similar items (K-Means, DBSCAN)
├── Dimensionality     → Reduce features while preserving info (PCA, t-SNE, UMAP)
│   Reduction
└── Anomaly Detection  → Find outliers (Isolation Forest)
```

**K-Means** partitions data into K clusters by iteratively assigning points to the nearest centroid and recomputing centroids. Use the **elbow method** (plot inertia vs K) to pick the right K.

**PCA** (Principal Component Analysis) projects data onto orthogonal axes that capture maximum variance. Critical for visualization and denoising high-dimensional data.

```python
from sklearn.cluster import KMeans
from sklearn.decomposition import PCA

# Clustering
kmeans = KMeans(n_clusters=3, random_state=42, n_init=10)
clusters = kmeans.fit_predict(X)  # assigns each row a cluster ID

# Dimensionality reduction
pca = PCA(n_components=2)
X_reduced = pca.fit_transform(X)
print(f"Variance explained: {pca.explained_variance_ratio_}")
```

### Self-Supervised Learning

The model creates its own labels from the data itself — no human annotation required. This is the paradigm behind modern foundation models.

- **Masked Language Modeling (BERT):** Mask 15% of tokens, predict them.
- **Next Token Prediction (GPT):** Given a sequence, predict the next token.
- **Contrastive Learning (SimCLR, CLIP):** Pull similar pairs closer, push dissimilar pairs apart in embedding space.

### 💡 Interview Insight

> **"When would you choose unsupervised over supervised?"**
> When you lack labels (exploration phase), need to discover latent groups (customer segmentation), or want to reduce dimensionality before supervised learning. In practice, you often combine them: cluster first, then build a classifier per cluster.

---

## Screen 2: Bias-Variance Tradeoff & Data Splitting

### The Fundamental Tradeoff

Every ML model fights a two-front war:

```
Error = Bias² + Variance + Irreducible Noise

  High Bias                          High Variance
  (Underfitting)                     (Overfitting)
  ┌─────────┐                        ┌─────────┐
  │  ·   ·  │   Model too simple     │ ·~·~·~· │   Model memorizes noise
  │ · · · · │   Misses the pattern   │·       ·│   Fails on new data
  │  ·   ·  │                        │ ·~·~·~· │
  └─────────┘                        └─────────┘
       ↕                                  ↕
  Training error: HIGH               Training error: LOW
  Test error:     HIGH               Test error:     HIGH
```

**Sweet Spot:** A model complex enough to capture the true pattern but simple enough to generalize. Regularization, cross-validation, and early stopping help you find it.

### Data Splitting Strategies

**Hold-Out Split (most common):**

```
┌──────────────────────────────────────────────────────┐
│   Training (60-70%)   │  Val (15-20%)  │ Test (15-20%) │
└──────────────────────────────────────────────────────┘
  Learn weights           Tune hyperparams   Final eval
                          (model selection)   (NEVER peek!)
```

```python
from sklearn.model_selection import train_test_split

# Step 1: Separate test set FIRST
X_temp, X_test, y_temp, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)
# Step 2: Split remaining into train/val
X_train, X_val, y_train, y_val = train_test_split(
    X_temp, y_temp, test_size=0.25, random_state=42, stratify=y_temp
)
# Result: 60% train, 20% val, 20% test
```

**K-Fold Cross-Validation:**

When data is limited, K-fold gives you K estimates of model performance by rotating the validation fold.

```
Fold 1: [VAL] [Train] [Train] [Train] [Train]
Fold 2: [Train] [VAL] [Train] [Train] [Train]
Fold 3: [Train] [Train] [VAL] [Train] [Train]
Fold 4: [Train] [Train] [Train] [VAL] [Train]
Fold 5: [Train] [Train] [Train] [Train] [VAL]

Final score = mean(fold_scores) ± std(fold_scores)
```

```python
from sklearn.model_selection import cross_val_score, StratifiedKFold

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X_train, y_train, cv=cv, scoring='f1_macro')
print(f"F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

### 💡 Interview Insight

> **"Why can't you use the test set for hyperparameter tuning?"**
> If you tune on the test set, you leak information about it into your model selection process. The test set becomes another validation set, and you no longer have a true out-of-sample estimate of generalization. Your reported metrics will be optimistically biased.

---

## Screen 3: Regularization — L1, L2 & ElasticNet

Regularization adds a penalty term to the loss function to discourage overly complex models.

### The Three Musketeers

| Method | Penalty Term | Effect | Use When |
|---|---|---|---|
| L1 (Lasso) | `λ Σ|wᵢ|` | Drives weights to zero | Feature selection needed |
| L2 (Ridge) | `λ Σwᵢ²` | Shrinks weights uniformly | All features matter |
| ElasticNet | `λ₁ Σ|wᵢ| + λ₂ Σwᵢ²` | Combines both | Correlated features |

```
Loss = Original Loss + Regularization Penalty

L1 (Lasso):     Loss_total = MSE + λ * Σ|wᵢ|
                 → Sparse weights (some become exactly 0)
                 → Built-in feature selection

L2 (Ridge):     Loss_total = MSE + λ * Σwᵢ²
                 → Small but non-zero weights
                 → Handles multicollinearity well

ElasticNet:     Loss_total = MSE + λ₁ * Σ|wᵢ| + λ₂ * Σwᵢ²
                 → Best of both worlds
                 → Good when features are correlated groups
```

```python
from sklearn.linear_model import Lasso, Ridge, ElasticNet

# L1: drives irrelevant feature weights to exactly 0
lasso = Lasso(alpha=0.1)
lasso.fit(X_train, y_train)
print(f"Non-zero features: {(lasso.coef_ != 0).sum()} / {len(lasso.coef_)}")

# L2: shrinks all weights proportionally
ridge = Ridge(alpha=1.0)
ridge.fit(X_train, y_train)

# ElasticNet: mix of L1 and L2
enet = ElasticNet(alpha=0.1, l1_ratio=0.5)  # l1_ratio: 0=Ridge, 1=Lasso
enet.fit(X_train, y_train)
```

**In tree-based models**, regularization takes different forms:

```
Tree Regularization Arsenal:
├── max_depth          → limits tree depth (most impactful)
├── min_samples_leaf   → minimum samples required at a leaf node
├── min_samples_split  → minimum samples to attempt a split
├── max_features       → random subset of features per split (Random Forest)
├── n_estimators       → more trees = less variance (with diminishing returns)
├── learning_rate      → shrinks each tree's contribution (boosting)
├── subsample          → random subset of rows per tree (stochastic gradient boosting)
├── colsample_bytree   → random subset of features per tree (XGBoost)
├── reg_alpha (L1)     → L1 penalty on leaf weights (XGBoost)
└── reg_lambda (L2)    → L2 penalty on leaf weights (XGBoost)
```

**Early stopping** is another powerful regularization technique for boosted trees — monitor validation loss and stop adding trees when it stops improving:

```python
import xgboost as xgb

model = xgb.XGBClassifier(n_estimators=1000, learning_rate=0.05, random_state=42)
model.fit(
    X_train, y_train,
    eval_set=[(X_val, y_val)],
    verbose=False
)
# early_stopping_rounds is set via set_params or callbacks in modern XGBoost
print(f"Best iteration: {model.best_iteration}")
```

### 💡 Interview Insight

> **"How does L1 achieve sparsity but L2 doesn't?"**
> Geometrically, L1's constraint region is a diamond (with corners on the axes), so the optimal solution is more likely to land on a corner where some weights are exactly zero. L2's constraint region is a circle — the optimal solution typically sits on the smooth surface where all weights are non-zero but small.

---

## Screen 4: Feature Engineering

Feature engineering is where domain knowledge meets ML. It's often the difference between a mediocre model and a great one.

### Encoding Categorical Variables

```python
import pandas as pd
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder, LabelEncoder

# One-Hot: for nominal categories (no order)
# color: [red, blue, green] → [1,0,0], [0,1,0], [0,0,1]
ohe = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
X_encoded = ohe.fit_transform(df[['color']])

# Ordinal: for ordered categories
# size: [S, M, L, XL] → [0, 1, 2, 3]
oe = OrdinalEncoder(categories=[['S', 'M', 'L', 'XL']])
df['size_encoded'] = oe.fit_transform(df[['size']])

# Target Encoding: replace category with mean of target
# city → mean(price) per city — BEWARE of data leakage!
# Use sklearn.preprocessing.TargetEncoder (sklearn 1.3+)
from sklearn.preprocessing import TargetEncoder
te = TargetEncoder(smooth="auto")
df['city_encoded'] = te.fit_transform(df[['city']], y_train)
```

### Scaling & Normalization

| Scaler | Formula | Use When |
|---|---|---|
| StandardScaler | `(x - μ) / σ` | Gaussian-ish data, SVM, LR |
| MinMaxScaler | `(x - min) / (max - min)` | Neural nets, bounded features |
| RobustScaler | `(x - median) / IQR` | Data has outliers |

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler

# StandardScaler: zero mean, unit variance
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)  # fit on TRAIN only!
X_test_scaled = scaler.transform(X_test)  # transform test with train stats
```

### Handling Missing Values

```python
from sklearn.impute import SimpleImputer, KNNImputer

# Strategy 1: Statistical imputation
imputer = SimpleImputer(strategy='median')  # or 'mean', 'most_frequent'
X_imputed = imputer.fit_transform(X_train)

# Strategy 2: KNN imputation (uses similar samples)
knn_imp = KNNImputer(n_neighbors=5)
X_imputed = knn_imp.fit_transform(X_train)

# Strategy 3: Add a missing indicator column
from sklearn.impute import MissingIndicator
indicator = MissingIndicator()
missing_flags = indicator.fit_transform(X_train)
```

### Temporal Features

```python
# Extract rich features from datetime columns
df['hour'] = df['timestamp'].dt.hour
df['day_of_week'] = df['timestamp'].dt.dayofweek
df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)
df['month'] = df['timestamp'].dt.month
df['quarter'] = df['timestamp'].dt.quarter

# Lag features for time series
df['sales_lag_1'] = df['sales'].shift(1)
df['sales_lag_7'] = df['sales'].shift(7)
df['sales_rolling_7'] = df['sales'].rolling(window=7).mean()
df['sales_rolling_30'] = df['sales'].rolling(window=30).mean()
df['sales_expanding_mean'] = df['sales'].expanding().mean()
```

### Feature Selection Methods

```python
from sklearn.feature_selection import mutual_info_classif, SelectKBest
from sklearn.inspection import permutation_importance

# Method 1: Correlation — drop features correlated > 0.95
corr_matrix = X_train.corr().abs()
upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
to_drop = [col for col in upper.columns if any(upper[col] > 0.95)]

# Method 2: Mutual Information — non-linear feature relevance
mi_scores = mutual_info_classif(X_train, y_train, random_state=42)
mi_ranking = pd.Series(mi_scores, index=X_train.columns).sort_values(ascending=False)
print(mi_ranking.head(10))  # Top 10 most informative features

# Method 3: Permutation Importance — model-agnostic
result = permutation_importance(model, X_val, y_val, n_repeats=10, random_state=42)
importances = pd.Series(result.importances_mean, index=X_val.columns)
print(importances.sort_values(ascending=False).head(10))

# Method 4: SelectKBest — automated selection
selector = SelectKBest(mutual_info_classif, k=20)
X_selected = selector.fit_transform(X_train, y_train)
```

**Feature engineering workflow:**

```
Raw Data → Clean → Encode Categoricals → Scale Numerics → Engineer New Features
                                                                    ↓
                                              Select Top Features (MI, SHAP)
                                                                    ↓
                                                     Final Feature Set → Model
```

### 💡 Interview Insight

> **"How do you prevent data leakage in feature engineering?"**
> Always fit transformers (scalers, imputers, encoders) on training data only, then transform val/test. Use `sklearn.pipeline.Pipeline` to automate this. Target encoding is especially dangerous — always use cross-validated target encoding or a library that handles it (e.g., `TargetEncoder` with built-in CV).

---

## Screen 5: Evaluation Metrics Deep Dive

### Classification Metrics

```
                    Predicted
                  Pos    Neg
Actual  Pos  │  TP   │  FN  │   ← Recall = TP / (TP + FN)
        Neg  │  FP   │  TN  │
                  ↑
          Precision = TP / (TP + FP)

Accuracy  = (TP + TN) / (TP + TN + FP + FN)  ← misleading if imbalanced!
F1-Score  = 2 * (Precision * Recall) / (Precision + Recall)
```

**When to use which metric:**

| Scenario | Preferred Metric | Why |
|---|---|---|
| Balanced classes | Accuracy or F1 | All metrics agree |
| Imbalanced (fraud, cancer) | AUC-PR, Recall, F1 | Accuracy is misleading |
| Cost of FP is high (spam) | Precision | Don't annoy users |
| Cost of FN is high (cancer) | Recall | Don't miss cases |
| Need threshold-free eval | AUC-ROC or AUC-PR | Compare models fairly |
| Ranking quality | NDCG, MRR | Search/recommendations |

```python
from sklearn.metrics import (
    classification_report, confusion_matrix,
    roc_auc_score, average_precision_score, f1_score
)

# Full classification report
print(classification_report(y_test, y_pred, digits=4))

# AUC-ROC (use predict_proba, not predict!)
y_proba = model.predict_proba(X_test)[:, 1]
auc_roc = roc_auc_score(y_test, y_proba)

# AUC-PR (better for imbalanced data)
auc_pr = average_precision_score(y_test, y_proba)
print(f"AUC-ROC: {auc_roc:.4f} | AUC-PR: {auc_pr:.4f}")
```

### AUC-ROC vs AUC-PR

```
AUC-ROC Curve                    AUC-PR Curve
TPR (Recall)                     Precision
  1 ┤     ╭────────             1 ┤──╮
    │   ╭─╯                       │   ╲
    │  ╭╯                         │    ╲──╮
    │ ╭╯    ← Good model          │       ╲── ← Good model
    │╭╯                            │          ╲
  0 ┤─────────────              0 ┤────────────
    0    FPR     1                 0  Recall   1

Use AUC-ROC: balanced classes    Use AUC-PR: imbalanced classes
(shows overall discrimination)   (focuses on positive class perf)
```

### Regression Metrics

| Metric | Formula | Interpretation | Use When |
|---|---|---|---|
| RMSE | `√(mean((y - ŷ)²))` | Same units as target | Penalize large errors |
| MAE | `mean(|y - ŷ|)` | Same units, robust | Outlier-robust evaluation |
| R² | `1 - SS_res/SS_tot` | % variance explained | Compare to baseline |
| MAPE | `mean(|y - ŷ|/|y|)` | Percentage error | Scale-independent |

```python
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
import numpy as np

rmse = np.sqrt(mean_squared_error(y_test, y_pred))   # Same units as target
mae = mean_absolute_error(y_test, y_pred)              # Robust to outliers
r2 = r2_score(y_test, y_pred)                          # % variance explained
mape = np.mean(np.abs((y_test - y_pred) / y_test))    # Percentage error

print(f"RMSE: ${rmse:,.2f} | MAE: ${mae:,.2f} | R²: {r2:.4f} | MAPE: {mape:.2%}")
# RMSE: $24,500.00 | MAE: $18,200.00 | R²: 0.8934 | MAPE: 12.45%
# RMSE > MAE → indicates some large prediction errors (outliers)
```

### Ranking Metrics (Search & Recommendations)

```
NDCG (Normalized Discounted Cumulative Gain):
  - Measures ranking quality with graded relevance
  - Position matters: top results weighted more heavily
  - DCG@K = Σᵢ (2^relevanceᵢ - 1) / log₂(i + 1)
  - NDCG@K = DCG@K / ideal_DCG@K (normalized to [0, 1])

MRR (Mean Reciprocal Rank):
  - Average of 1/rank_of_first_relevant_result across queries
  - Query 1: first relevant at rank 3 → 1/3
  - Query 2: first relevant at rank 1 → 1/1
  - MRR = (1/3 + 1) / 2 = 0.667
```

### 💡 Interview Insight

> **"You have a fraud detection model with 99.5% accuracy. Is it good?"**
> If only 0.5% of transactions are fraudulent, a model that predicts "not fraud" for everything achieves 99.5% accuracy. This is useless! Use **AUC-PR**, **recall** (catch fraudulent transactions), and **precision** (don't block legitimate users). Always check the class distribution first.

---

## Screen 6: Tree-Based Models — From Decision Trees to XGBoost

### Decision Trees

```
                   Is income > 50K?
                  /               \
               Yes                 No
              /                      \
      Credit score > 700?        Deny Loan
      /              \
   Yes                No
   /                    \
Approve Loan       Check employment
                   /            \
                 Yes             No
                Approve         Deny
```

- **Splitting criteria:** Gini impurity (default in sklearn), Entropy (information gain)
- **Prone to overfitting:** A deep tree memorizes training data
- **Pruning:** Limit `max_depth`, `min_samples_leaf`, `min_samples_split`

### Ensemble Methods

```
Bagging (Random Forest)              Boosting (XGBoost/LightGBM)
┌────┐ ┌────┐ ┌────┐                 Tree₁ → residuals →
│Tree│ │Tree│ │Tree│                  Tree₂ → residuals →
│ 1  │ │ 2  │ │ 3  │                  Tree₃ → residuals →
└──┬─┘ └──┬─┘ └──┬─┘                  Tree₄ → ...
   │      │      │
   └──────┼──────┘                 Each tree corrects the
     VOTE / AVERAGE                errors of the previous one
  (parallel, reduce variance)      (sequential, reduce bias)
```

### XGBoost vs LightGBM vs CatBoost

| Feature | XGBoost | LightGBM | CatBoost |
|---|---|---|---|
| Tree growth | Level-wise | Leaf-wise (faster) | Symmetric trees |
| Speed | Fast | Fastest | Moderate |
| Categoricals | Manual encoding | Built-in (limited) | Native (best) |
| Missing values | Built-in handling | Built-in handling | Built-in handling |
| GPU support | Yes | Yes | Yes |
| Key params | `max_depth`, `eta` | `num_leaves` | `depth`, `l2_leaf_reg` |

```python
import xgboost as xgb
from lightgbm import LGBMClassifier
from catboost import CatBoostClassifier

# XGBoost
xgb_clf = xgb.XGBClassifier(
    n_estimators=500, max_depth=6, learning_rate=0.1,
    subsample=0.8, colsample_bytree=0.8,
    reg_alpha=0.1, reg_lambda=1.0,  # L1, L2 regularization
    eval_metric='logloss', early_stopping_rounds=50,
    random_state=42
)
xgb_clf.fit(X_train, y_train, eval_set=[(X_val, y_val)], verbose=False)

# LightGBM — leaf-wise growth, often faster
lgbm_clf = LGBMClassifier(
    n_estimators=500, num_leaves=31, learning_rate=0.1,
    subsample=0.8, colsample_bytree=0.8, verbose=-1
)

# CatBoost — native categorical support
cat_clf = CatBoostClassifier(
    iterations=500, depth=6, learning_rate=0.1,
    cat_features=['city', 'category'],  # pass column names!
    verbose=0
)
```

### Feature Importance & SHAP

```python
import shap

# SHAP: the gold standard for model interpretability
explainer = shap.TreeExplainer(xgb_clf)
shap_values = explainer.shap_values(X_test)

# Summary plot: which features matter most and how
shap.summary_plot(shap_values, X_test)

# Single prediction explanation
shap.force_plot(explainer.expected_value, shap_values[0], X_test.iloc[0])
```

### 💡 Interview Insight

> **"XGBoost vs LightGBM — when would you pick one over the other?"**
> LightGBM is typically faster due to leaf-wise growth and histogram-based splitting. Use it for large datasets and rapid iteration. XGBoost is more mature with slightly better default regularization. CatBoost shines when you have many categorical features — it handles them natively with ordered target statistics, avoiding leakage. For most Kaggle-style problems, all three perform similarly; pick based on your data characteristics.

---

## Screen 7: Hyperparameter Tuning — GridSearch & Optuna

### GridSearch: The Brute Force Approach

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'n_estimators': [100, 300, 500],
    'max_depth': [4, 6, 8],
    'learning_rate': [0.01, 0.05, 0.1],
    'subsample': [0.8, 1.0]
}

grid_search = GridSearchCV(
    xgb.XGBClassifier(random_state=42),
    param_grid,
    cv=5,
    scoring='f1_macro',
    n_jobs=-1,
    verbose=1
)
grid_search.fit(X_train, y_train)
print(f"Best params: {grid_search.best_params_}")
print(f"Best F1: {grid_search.best_score_:.4f}")
```

**Problem:** 3 × 3 × 3 × 2 = 54 combinations × 5 folds = 270 model fits. It explodes combinatorially.

### Optuna: Bayesian Optimization (The Smart Way)

```python
import optuna

def objective(trial):
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 100, 1000),
        'max_depth': trial.suggest_int('max_depth', 3, 10),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 0.3, log=True),
        'subsample': trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.6, 1.0),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
    }
    model = xgb.XGBClassifier(**params, random_state=42)
    
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    scores = cross_val_score(model, X_train, y_train, cv=cv, scoring='f1_macro')
    return scores.mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=100, show_progress_bar=True)

print(f"Best F1: {study.best_value:.4f}")
print(f"Best params: {study.best_params}")
```

**Why Optuna wins:** It uses Tree-Structured Parzen Estimators (TPE) to model the search space. Each trial informs the next, focusing on promising regions. It also supports pruning — killing bad trials early.

### 💡 Interview Insight

> **"How do you decide which hyperparameters to tune?"**
> Focus on the ones with the most impact: `learning_rate` and `n_estimators` (coupled — lower LR needs more estimators), `max_depth` / `num_leaves` (model complexity), and `subsample` / `colsample_bytree` (regularization via randomness). Always tune `learning_rate` on a log scale. Use early stopping instead of tuning `n_estimators` directly.

---

## Screen 8: Scikit-Learn Pipelines — Production-Ready ML

### Why Pipelines?

1. **Prevent data leakage:** Transformers are fit only on training data
2. **Reproducibility:** One object encapsulates the entire workflow
3. **Deployment:** Serialize and deploy as a single artifact

### Building a Complete Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import RandomForestClassifier

# Define column groups
numeric_features = ['age', 'income', 'credit_score']
categorical_features = ['city', 'employment_type']

# Numeric preprocessing: impute → scale
numeric_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

# Categorical preprocessing: impute → one-hot encode
categorical_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('onehot', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

# Combine with ColumnTransformer
preprocessor = ColumnTransformer([
    ('num', numeric_transformer, numeric_features),
    ('cat', categorical_transformer, categorical_features)
])

# Full pipeline: preprocessing → model
pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', RandomForestClassifier(n_estimators=200, random_state=42))
])

# Fit and predict — clean and safe
pipeline.fit(X_train, y_train)
y_pred = pipeline.predict(X_test)
score = pipeline.score(X_test, y_test)
```

### Custom Transformers

```python
from sklearn.base import BaseEstimator, TransformerMixin
import numpy as np

class TemporalFeatureExtractor(BaseEstimator, TransformerMixin):
    """Extract hour, day_of_week, is_weekend from datetime columns."""
    
    def __init__(self, datetime_col: str = 'timestamp'):
        self.datetime_col = datetime_col
    
    def fit(self, X, y=None):
        return self  # Nothing to learn
    
    def transform(self, X):
        X = X.copy()
        dt = pd.to_datetime(X[self.datetime_col])
        X['hour'] = dt.dt.hour
        X['day_of_week'] = dt.dt.dayofweek
        X['is_weekend'] = (dt.dt.dayofweek >= 5).astype(int)
        X = X.drop(columns=[self.datetime_col])
        return X
```

### Cross-Validation with Pipelines & Serialization

```python
from sklearn.model_selection import cross_val_score, GridSearchCV
import joblib

# Cross-validation — pipeline prevents leakage automatically
scores = cross_val_score(pipeline, X_train, y_train, cv=5, scoring='f1_macro')
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")

# GridSearch with pipeline parameters (use __ for nested access)
param_grid = {
    'classifier__n_estimators': [100, 300],
    'classifier__max_depth': [5, 10, None],
    'preprocessor__num__imputer__strategy': ['mean', 'median']
}

grid = GridSearchCV(pipeline, param_grid, cv=5, scoring='f1_macro', n_jobs=-1)
grid.fit(X_train, y_train)

# Save the entire pipeline (preprocessing + model)
joblib.dump(grid.best_estimator_, 'model_pipeline.joblib')

# Load and predict on new data — zero preprocessing code needed
loaded_pipeline = joblib.load('model_pipeline.joblib')
predictions = loaded_pipeline.predict(new_data)
```

### 💡 Interview Insight

> **"How do you deploy an ML model?"**
> Serialize the entire pipeline (not just the model) with `joblib`. This ensures the exact same preprocessing is applied at inference time. Wrap it in a FastAPI endpoint, containerize with Docker, and serve behind a load balancer. The pipeline pattern eliminates the #1 cause of training-serving skew: inconsistent preprocessing.

---

## Screen 9: Quiz & Key Takeaways

### 🧪 Quiz

**Q1:** You're building a model to detect fraudulent transactions (0.1% fraud rate). Which metric is most appropriate?

- A) Accuracy
- B) F1-Score
- C) AUC-PR
- D) R²

<details><summary>Answer</summary>C) AUC-PR — it focuses on the positive class performance and is threshold-independent. F1 is also acceptable but requires choosing a threshold. Accuracy is meaningless at 99.9% class imbalance.</details>

**Q2:** You notice your model has low training error but high validation error. What's happening and how do you fix it?

<details><summary>Answer</summary>The model is overfitting (high variance). Fixes: increase regularization (L1/L2, dropout), reduce model complexity (lower max_depth, fewer features), collect more training data, use early stopping, or apply data augmentation.</details>

**Q3:** Why should you fit a scaler on training data only and then transform test data with the same scaler?

<details><summary>Answer</summary>If you fit the scaler on all data (including test), test set statistics leak into preprocessing. The model indirectly "sees" test data characteristics during training, leading to optimistically biased metrics. This is data leakage.</details>

**Q4:** What's the key difference between LightGBM's leaf-wise and XGBoost's level-wise tree growth?

<details><summary>Answer</summary>Level-wise grows all nodes at the same depth before going deeper (balanced trees). Leaf-wise chooses the leaf with the highest loss reduction to split next, creating asymmetric trees that can be more accurate with fewer splits but are more prone to overfitting on small datasets.</details>

**Q5:** When would you use ElasticNet over Lasso or Ridge alone?

<details><summary>Answer</summary>When features are correlated in groups. Lasso tends to arbitrarily select one feature from a correlated group and zero out the rest. ElasticNet's L2 component keeps correlated features together while L1 still provides sparsity across groups.</details>

### 🎯 Key Takeaways

1. **Know your problem type** — classification vs regression determines your loss function, metrics, and model choices
2. **Bias-variance tradeoff** is the central theme — every technique either reduces bias or variance
3. **Data leakage** is the silent killer — use pipelines and fit only on training data
4. **Metric selection matters** — accuracy is misleading for imbalanced data; match metrics to business goals
5. **Tree-based models dominate** tabular data — XGBoost/LightGBM/CatBoost are your go-to
6. **Optuna > GridSearch** — Bayesian optimization is smarter and faster
7. **Pipelines are non-negotiable** — they prevent leakage, enable reproducibility, and simplify deployment
8. **Feature engineering > model tuning** — great features beat fancy models every time
