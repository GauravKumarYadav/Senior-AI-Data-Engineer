---
tags: [ai, ml, fundamentals, phase-4]
phase: 4
status: not-started
priority: high
---

# 🤖 ML Fundamentals

> **Phase:** 4 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[03 - Python AI-ML]], [[02 - Deep Learning Foundations]], [[01 - MLOps Pipeline]], [[01 - System Design AI Problems]]

---

## Checklist

### Core ML Concepts (Refresh)
- [ ] Supervised: classification (labels) vs regression (continuous)
- [ ] Unsupervised: clustering (K-means, DBSCAN), dimensionality reduction (PCA, t-SNE, UMAP)
- [ ] Self-supervised: pretext tasks, contrastive learning (foundation for LLMs)
- [ ] Bias-variance tradeoff: underfitting vs overfitting, model complexity sweet spot
- [ ] Train/validation/test splits: hold-out, K-fold cross-validation, stratified splits
- [ ] Regularization: L1 (Lasso, sparsity), L2 (Ridge, shrinkage), ElasticNet
- [ ] Ensemble methods: bagging (Random Forest), boosting (XGBoost), stacking

### Feature Engineering
- [ ] Encoding categoricals: one-hot, label, target encoding, frequency encoding
- [ ] Scaling: StandardScaler, MinMaxScaler, RobustScaler — when to use each
- [ ] Feature hashing: for high-cardinality categoricals
- [ ] Binning: continuous → categorical, equal-width vs equal-frequency
- [ ] Missing values: imputation (mean, median, KNN), indicator columns
- [ ] Feature interactions: polynomial features, domain-specific combinations
- [ ] Text features: TF-IDF, count vectors (bridge to [[03 - NLP for AI Engineers]])
- [ ] Temporal features: day of week, lag features, rolling statistics
- [ ] Feature selection: correlation, mutual information, feature importance

### Evaluation Metrics
- [ ] Classification: accuracy, precision, recall, F1-score, macro/micro/weighted
- [ ] AUC-ROC: threshold-independent evaluation, TPR vs FPR curve
- [ ] AUC-PR: precision-recall curve, better for imbalanced datasets
- [ ] Log-loss: probabilistic predictions, calibration
- [ ] Regression: RMSE, MAE, MAPE, R² — when to use each
- [ ] Ranking: NDCG, MAP, MRR (for search/recommendation systems)
- [ ] Confusion matrix: true positives, false positives, Type I vs Type II errors
- [ ] Business metrics: revenue impact, conversion rate — connecting ML to business

### Tree-Based Models
- [ ] Decision Trees: splitting criteria (Gini, entropy), pruning
- [ ] Random Forest: bagging, feature randomness, out-of-bag error
- [ ] XGBoost: gradient boosting, regularization, `learning_rate`, `max_depth`, `n_estimators`
- [ ] LightGBM: leaf-wise growth (vs level-wise), faster training, `num_leaves`
- [ ] CatBoost: native categorical handling, ordered boosting
- [ ] Feature importance: gain-based, permutation importance, SHAP values
- [ ] Hyperparameter tuning: GridSearch, RandomSearch, Optuna (Bayesian)

### Scikit-Learn Pipelines
- [ ] `Pipeline`: chain preprocessing + model, prevent data leakage
- [ ] `ColumnTransformer`: different transforms for different column types
- [ ] Custom transformers: `BaseEstimator`, `TransformerMixin`
- [ ] `make_pipeline` shorthand
- [ ] Cross-validation with pipelines: `cross_val_score`, `GridSearchCV`
- [ ] Saving pipelines: `joblib.dump`, model serialization

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Hands-On Machine Learning" by Aurélien Géron (Chapters 1-7)
- [ ] Andrew Ng: Machine Learning Specialization (Coursera)
- [ ] Scikit-learn docs: User Guide
- [ ] Kaggle: practice competitions

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Walk through building an ML model from data to evaluation
- How do you handle an imbalanced dataset?
- Explain XGBoost vs LightGBM — key differences
- How do you prevent data leakage in feature engineering?
