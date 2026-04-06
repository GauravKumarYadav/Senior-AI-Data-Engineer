---
tags: [mlops, phase-6]
phase: 6
status: not-started
priority: high
---

# ⚙️ MLOps Pipeline

> **Phase:** 6 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[01 - ML Fundamentals]], [[02 - Model Serving]], [[03 - LLMOps]], [[01 - System Design AI Problems]]

---

## Checklist

### ML Experiment Tracking
- [ ] MLflow:
  - [ ] Experiments: group related runs
  - [ ] Runs: log parameters, metrics, artifacts
  - [ ] `mlflow.autolog()` — automatic logging for sklearn, PyTorch, etc.
  - [ ] Model Registry: versioning, staging → production promotion
  - [ ] Model serving: `mlflow models serve`, REST API
  - [ ] MLflow Projects: reproducible runs with `MLproject` file
- [ ] Weights & Biases (W&B): experiment tracking, hyperparameter sweeps, model registry
- [ ] Comparison: MLflow (open-source, self-hosted) vs W&B (cloud, better UX)

### Feature Stores
- [ ] Why: consistent features between training and serving, reuse, point-in-time correctness
- [ ] Feast:
  - [ ] Offline store: historical features from data warehouse/lake (training)
  - [ ] Online store: low-latency features for real-time serving (Redis, DynamoDB)
  - [ ] Feature services: group features for a model
  - [ ] Materialization: batch sync from offline → online store
- [ ] Point-in-time joins: prevent data leakage with temporal feature joins
- [ ] Feature pipelines: compute → store → serve (batch vs streaming features)
- [ ] Databricks Feature Store / Vertex AI Feature Store (managed alternatives)

### Data Versioning
- [ ] DVC (Data Version Control): git for data, track large files
  - [ ] `.dvc` files: metadata pointers to remote storage
  - [ ] Pipelines: `dvc.yaml` → reproducible ML pipelines
  - [ ] Remote storage: S3, GCS, Azure Blob
- [ ] lakeFS: git-like branching for data lakes (branch, commit, merge)
- [ ] Delta Lake time travel: version data natively (see [[02 - Data Storage Formats]])

### ML Pipeline Frameworks
- [ ] Kubeflow Pipelines: Kubernetes-native, DSL for pipeline definition
- [ ] SageMaker Pipelines: AWS-managed, integrated with SageMaker ecosystem
- [ ] Vertex AI Pipelines: GCP, based on Kubeflow
- [ ] ZenML: framework-agnostic, modular stacks (experiment tracker, artifact store, etc.)
- [ ] Components: data prep → training → evaluation → registration → deployment

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] MLflow docs: https://mlflow.org/
- [ ] Feast docs: https://feast.dev/
- [ ] "Designing Machine Learning Systems" by Chip Huyen (Chapters 9-11)
- [ ] DVC docs: https://dvc.org/

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design an ML pipeline from training to production
- What is a feature store and why is it important?
- How do you handle model versioning and reproducibility?
