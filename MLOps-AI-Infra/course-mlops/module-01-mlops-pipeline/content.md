# MLOps Pipeline

> **Module Goal**: Master the end-to-end ML lifecycle — from experiment tracking and feature stores to pipeline orchestration and CI/CD for models — so you can design production ML systems and answer MLOps interview questions with confidence.

---

## Screen 1: The ML Lifecycle & Why MLOps Exists

Machine learning in production is **not** a Jupyter notebook. The moment a model leaves a data scientist's laptop, it enters a world of data drift, dependency hell, reproducibility nightmares, and silent failures. MLOps is the discipline that tames this chaos.

### The Production ML Gap

```
┌─────────────────────────────────────────────────────────┐
│              THE PRODUCTION ML LIFECYCLE                │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │   Data    │──▶│  Model   │──▶│   Experiment     │    │
│  │ Ingestion │   │ Training │   │   Tracking       │    │
│  └──────────┘   └──────────┘   └──────────────────┘    │
│       │                               │                 │
│       ▼                               ▼                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │ Feature   │   │  Model   │◀──│   Evaluation &   │    │
│  │  Store    │   │ Registry │   │   Validation     │    │
│  └──────────┘   └──────────┘   └──────────────────┘    │
│                      │                                  │
│                      ▼                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │Monitoring│◀──│  Model   │◀──│   CI/CD for ML   │    │
│  │ & Drift  │   │ Serving  │   │   Pipeline       │    │
│  └──────────┘   └──────────┘   └──────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### MLOps Maturity Model (Levels 0–4)

Understanding where an organization sits on the maturity curve is a **classic interview question**. Each level builds on the previous one:

| Level | Name                    | Key Characteristics                  |
|-------|-------------------------|--------------------------------------|
| 0     | Manual / Ad-Hoc         | Notebooks, manual deploys, no CI/CD  |
| 1     | Pipeline Automation     | Automated training pipeline, basic   |
| 2     | CI/CD Automation        | Automated testing + deployment       |
| 3     | Auto Retrain + Monitor  | Drift detection triggers retraining  |
| 4     | Full Autonomous ML      | Self-healing, auto feature eng.      |

**Level 0 — Manual, Notebook-Driven**: A data scientist trains a model in a Jupyter notebook, pickles it, and hands it to an engineer who wraps it in a Flask app. There is no versioning, no reproducibility, and no monitoring. Most organizations start here.

**Level 1 — ML Pipeline Automation**: Training is codified into a reproducible pipeline (e.g., Kubeflow, Airflow). Data ingestion, preprocessing, training, and evaluation are automated steps. However, deployment is still manual.

**Level 2 — CI/CD Pipeline Automation**: The ML pipeline itself is tested and deployed via CI/CD. Code changes trigger automated model builds, validation gates check model quality, and passing models are deployed automatically.

**Level 3 — Automated Retraining + Monitoring**: Production models are continuously monitored for data drift and performance degradation. When drift is detected, retraining pipelines are triggered automatically. Feature stores ensure consistency.

**Level 4 — Full Autonomous ML System**: The system autonomously manages feature engineering, model selection, hyperparameter tuning, A/B testing, and rollback. Human oversight is limited to setting business constraints and reviewing anomalies.

### 💡 Interview Insight

> **Q: "What MLOps maturity level is your current/previous team at, and what would you do to move them up?"**
>
> This is a culture-fit question disguised as a technical one. Be honest about the level (most teams are 1–2), then articulate a concrete plan: *"We were at Level 1 — we had Airflow pipelines but manual deployments. I proposed adding model validation gates in our CI pipeline and integrating MLflow's model registry to automate staging-to-production promotion. That moved us toward Level 2."*

---

## Screen 2: Experiment Tracking with MLflow

MLflow is the **open-source standard** for experiment tracking. It answers the fundamental question: *"Which combination of data, code, and hyperparameters produced this model?"*

### Core MLflow Concepts

```
┌─────────────────────────────────────────────────┐
│                  MLflow Server                  │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │            Experiment                   │    │
│  │  "fraud-detection-v2"                   │    │
│  │                                         │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐   │    │
│  │  │ Run #1  │ │ Run #2  │ │ Run #3  │   │    │
│  │  │lr=0.01  │ │lr=0.001 │ │lr=0.1   │   │    │
│  │  │acc=0.89 │ │acc=0.94 │ │acc=0.78 │   │    │
│  │  │artifact:│ │artifact:│ │artifact:│   │    │
│  │  │model.pkl│ │model.pkl│ │model.pkl│   │    │
│  │  └─────────┘ └─────────┘ └─────────┘   │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │          Model Registry                 │    │
│  │  "FraudDetector"                        │    │
│  │  v1 [Archived] → v2 [Production]       │    │
│  │                   v3 [Staging]          │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

- **Experiments** group related runs (e.g., all runs for a fraud-detection project).
- **Runs** capture parameters, metrics, artifacts, and source code for a single training execution.
- **Autolog** automatically logs metrics, parameters, and models for supported frameworks (sklearn, PyTorch, TensorFlow, XGBoost).
- **Model Registry** provides centralized model versioning with stage transitions: `None → Staging → Production → Archived`.

### MLflow in Practice: Full Experiment Tracking Example

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import (
    accuracy_score,
    precision_score,
    recall_score,
    f1_score,
)
import pandas as pd

# ── Connect to tracking server ──────────────────────
mlflow.set_tracking_uri("http://mlflow.internal:5000")
mlflow.set_experiment("fraud-detection-v2")

# ── Load and split data ─────────────────────────────
df = pd.read_parquet("s3://data-lake/fraud/features.parquet")
X_train, X_test, y_train, y_test = train_test_split(
    df.drop("is_fraud", axis=1),
    df["is_fraud"],
    test_size=0.2,
    random_state=42,
    stratify=df["is_fraud"],
)

# ── Train with full experiment tracking ─────────────
with mlflow.start_run(run_name="rf-baseline-v2") as run:
    # Log parameters
    params = {
        "n_estimators": 200,
        "max_depth": 15,
        "min_samples_split": 5,
        "class_weight": "balanced",
    }
    mlflow.log_params(params)

    # Log dataset metadata (not the data itself!)
    mlflow.log_param("train_samples", len(X_train))
    mlflow.log_param("test_samples", len(X_test))
    mlflow.log_param("fraud_rate", y_train.mean().round(4))

    # Train model
    model = RandomForestClassifier(**params, random_state=42)
    model.fit(X_train, y_train)

    # Evaluate and log metrics
    y_pred = model.predict(X_test)
    metrics = {
        "accuracy": accuracy_score(y_test, y_pred),
        "precision": precision_score(y_test, y_pred),
        "recall": recall_score(y_test, y_pred),
        "f1": f1_score(y_test, y_pred),
    }
    mlflow.log_metrics(metrics)

    # Log model with signature for serving
    from mlflow.models.signature import infer_signature

    signature = infer_signature(X_train, y_pred)
    mlflow.sklearn.log_model(
        model,
        "model",
        signature=signature,
        registered_model_name="FraudDetector",
    )

    # Log artifacts (plots, reports, etc.)
    mlflow.log_artifact("confusion_matrix.png")

    print(f"Run ID: {run.info.run_id}")
    print(f"Metrics: {metrics}")
```

### Using Autolog (The Lazy-But-Smart Way)

```python
import mlflow

# One line to log everything for sklearn
mlflow.sklearn.autolog(
    log_input_examples=True,
    log_model_signatures=True,
    log_models=True,
)

# Now just train normally — MLflow captures it all
with mlflow.start_run():
    model = RandomForestClassifier(n_estimators=100)
    model.fit(X_train, y_train)
    # Parameters, metrics, model — all logged automatically
```

### Model Registry: Promoting to Production

```python
from mlflow import MlflowClient

client = MlflowClient()

# Transition the best model to production
client.transition_model_version_stage(
    name="FraudDetector",
    version=3,
    stage="Production",
    archive_existing_versions=True,  # Auto-archive old prod
)

# Load the production model for serving
model = mlflow.pyfunc.load_model("models:/FraudDetector/Production")
predictions = model.predict(new_data)
```

### Model Serving

```bash
# Serve a model as a REST API (one command!)
mlflow models serve \
    -m "models:/FraudDetector/Production" \
    --port 5001 \
    --no-conda

# Query the endpoint
curl -X POST http://localhost:5001/invocations \
    -H "Content-Type: application/json" \
    -d '{"inputs": [{"amount": 500, "hour": 3, "country": "US"}]}'
```

### MLflow Projects: Reproducible Runs

An `MLproject` file makes any experiment reproducible:

```yaml
# MLproject
name: fraud-detection

conda_env: conda.yaml

entry_points:
  main:
    parameters:
      n_estimators: {type: int, default: 200}
      max_depth: {type: int, default: 15}
      learning_rate: {type: float, default: 0.01}
    command: >
      python train.py
        --n-estimators {n_estimators}
        --max-depth {max_depth}
        --learning-rate {learning_rate}
```

```bash
# Run from anywhere — reproducibly
mlflow run git@github.com:team/fraud-model.git \
    -P n_estimators=300 \
    -P max_depth=20
```

### 💡 Interview Insight

> **Q: "Compare MLflow with Weights & Biases."**
>
> | Dimension              | MLflow                        | Weights & Biases (W&B)        |
> |------------------------|-------------------------------|-------------------------------|
> | Deployment             | Self-hosted (open-source)     | Cloud SaaS (hosted)           |
> | Cost                   | Free (infra costs)            | Free tier, then paid          |
> | UX / Visualization     | Functional, basic             | Polished, interactive         |
> | Experiment Comparison  | Table-based                   | Rich parallel coords, sweeps |
> | Model Registry         | Built-in, mature              | Built-in, newer               |
> | Data Privacy           | Full control (on-prem)        | Data leaves your network      |
> | Community              | Large, Apache project         | Growing, strong in research   |
> | Hyperparameter Tuning  | External (Optuna, etc.)       | Built-in Sweeps               |
>
> **Bottom line**: MLflow wins for enterprises needing data sovereignty and open-source flexibility. W&B wins for research teams wanting best-in-class visualization and collaboration UX. Many teams use both — W&B for experiment exploration, MLflow for model registry and deployment.

---

## Screen 3: Feature Stores with Feast

A feature store is the **data layer of production ML**. It solves three critical problems: training-serving skew, feature reuse, and point-in-time correctness.

### Why Feature Stores?

Without a feature store, every ML team reinvents feature engineering. The fraud team computes `avg_transaction_amount_7d` one way for training (SQL on the warehouse) and another way for serving (Python in the API). The values diverge. The model performs differently in production. Nobody knows why.

```
┌─────────── WITHOUT Feature Store ──────────────┐
│                                                 │
│  Training:  SELECT AVG(amount) ...  → 142.50    │
│  Serving:   python rolling_mean()   → 143.12    │
│                                                 │
│  Result:  Training/Serving Skew = Silent Bugs   │
└─────────────────────────────────────────────────┘

┌─────────── WITH Feature Store ─────────────────┐
│                                                 │
│  Training:  feast.get_historical()  → 142.50    │
│  Serving:   feast.get_online()      → 142.50    │
│                                                 │
│  Result:  Consistency = Reliable Predictions    │
└─────────────────────────────────────────────────┘
```

### Feast Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   FEAST ARCHITECTURE                    │
│                                                         │
│  ┌──────────────┐    materialize     ┌──────────────┐   │
│  │ Offline Store │ ─────────────────▶│ Online Store  │   │
│  │ (BigQuery /   │   (batch sync)    │ (Redis /      │   │
│  │  Snowflake /  │                   │  DynamoDB)    │   │
│  │  S3 Parquet)  │                   │               │   │
│  └──────┬───────┘                   └──────┬───────┘   │
│         │                                  │            │
│         │ get_historical_features()        │ get_       │
│         │ (training)                       │ online_    │
│         ▼                                  │ features() │
│  ┌──────────────┐                         │ (serving)  │
│  │   Training   │                          ▼            │
│  │   Pipeline   │                   ┌──────────────┐   │
│  └──────────────┘                   │  Prediction  │   │
│                                     │  Service     │   │
│  ┌──────────────┐                   └──────────────┘   │
│  │   Feature    │──────────────────────────────────▶   │
│  │   Pipeline   │   compute → store → serve            │
│  │ (Spark/Flink)│                                      │
│  └──────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

**Offline Store**: Holds historical feature values for training. Backed by a data warehouse (BigQuery, Snowflake) or file storage (S3 Parquet). Supports **point-in-time joins** — this is the killer feature.

**Online Store**: Holds the *latest* feature values for low-latency serving. Backed by Redis or DynamoDB. Typically sub-10ms reads.

**Materialization**: The batch process that syncs features from the offline store to the online store. Run on a schedule (e.g., hourly, daily).

### Feast Feature Definition (Python)

```python
from datetime import timedelta
from feast import (
    Entity,
    Feature,
    FeatureView,
    FileSource,
    ValueType,
    Field,
)
from feast.types import Float32, Int64, String

# ── Data source ──────────────────────────────────
transactions_source = FileSource(
    path="s3://feature-store/transactions.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_at",
)

# ── Entity (the "join key") ──────────────────────
customer = Entity(
    name="customer_id",
    value_type=ValueType.INT64,
    description="Unique customer identifier",
)

# ── Feature View ─────────────────────────────────
customer_transaction_features = FeatureView(
    name="customer_transaction_features",
    entities=[customer],
    ttl=timedelta(days=90),  # Features expire after 90 days
    schema=[
        Field(name="avg_transaction_amount_7d", dtype=Float32),
        Field(name="transaction_count_7d", dtype=Int64),
        Field(name="max_transaction_amount_30d", dtype=Float32),
        Field(name="unique_merchants_30d", dtype=Int64),
        Field(name="avg_time_between_txns", dtype=Float32),
    ],
    source=transactions_source,
    online=True,  # Materialize to online store
)

# ── Feature Service (bundle for a use case) ──────
from feast import FeatureService

fraud_detection_service = FeatureService(
    name="fraud_detection",
    features=[
        customer_transaction_features,
        # Can include features from other FeatureViews
    ],
)
```

### Using Features for Training and Serving

```python
from feast import FeatureStore
import pandas as pd

store = FeatureStore(repo_path="feature_repo/")

# ── Training: get historical features ────────────
# Entity DataFrame defines WHO + WHEN
entity_df = pd.DataFrame({
    "customer_id": [1001, 1002, 1003, 1001],
    "event_timestamp": pd.to_datetime([
        "2025-01-15 10:00:00",
        "2025-01-15 11:30:00",
        "2025-01-16 09:00:00",
        "2025-01-17 14:00:00",  # Same customer, different time
    ]),
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_transaction_features:avg_transaction_amount_7d",
        "customer_transaction_features:transaction_count_7d",
        "customer_transaction_features:max_transaction_amount_30d",
    ],
).to_df()

# ── Serving: get latest features (online) ────────
feature_vector = store.get_online_features(
    features=[
        "customer_transaction_features:avg_transaction_amount_7d",
        "customer_transaction_features:transaction_count_7d",
    ],
    entity_rows=[{"customer_id": 1001}],
).to_dict()
# Returns: {"avg_transaction_amount_7d": [142.50], ...}
```

### Point-in-Time Joins: Preventing Data Leakage

This is the most important concept in feature stores. When you train a model using data from January 15th, you need the feature values **as they existed on January 15th** — not the current values.

```
Timeline:
─────────────────────────────────────────────────▶ time
     Jan 10         Jan 15         Jan 20

     avg_txn=120    avg_txn=142    avg_txn=165
                       ▲
                       │
              Training event here.
              Feature value = 142 ✓
              NOT 165 (future leak) ✗
```

A naive join would use the latest value (165), leaking future information into your training data. Feast's `get_historical_features()` performs **temporal joins** that respect event timestamps, preventing this subtle but devastating bug.

### Materialization

```bash
# Sync features from offline → online store
feast materialize 2025-01-01T00:00:00 2025-06-01T00:00:00

# Incremental materialization (only new data)
feast materialize-incremental $(date +%Y-%m-%dT%H:%M:%S)
```

### 💡 Interview Insight

> **Q: "Explain the difference between batch and streaming feature pipelines."**
>
> | Aspect             | Batch Pipeline                    | Streaming Pipeline               |
> |--------------------|-----------------------------------|----------------------------------|
> | Latency            | Minutes to hours                  | Seconds to milliseconds          |
> | Compute Engine     | Spark, SQL, dbt                   | Flink, Kafka Streams, Spark SS   |
> | Use Case           | Daily aggregates, historical      | Real-time counts, session data   |
> | Complexity         | Lower                             | Higher (exactly-once, ordering)  |
> | Example Feature    | avg_purchase_amount_30d           | transactions_last_5_minutes      |
> | Materialization    | Scheduled (cron)                  | Continuous                       |
>
> **Managed alternatives to Feast**: Databricks Feature Store (tight Unity Catalog integration), Vertex AI Feature Store (GCP-native, auto-scales), SageMaker Feature Store (AWS-native), and Tecton (enterprise, streaming-first). Feast wins on portability and avoiding vendor lock-in.

---

## Screen 4: Data Versioning with DVC

You version your code with Git. Why not your data? **DVC (Data Version Control)** extends Git to handle large files, datasets, and ML pipelines — without bloating your repository.

### How DVC Works

```
┌─────────────────────────────────────────────────┐
│                DVC Architecture                 │
│                                                 │
│  Git Repository          Remote Storage         │
│  ┌──────────────┐        ┌──────────────┐       │
│  │ train.csv.dvc │──────▶│  S3 / GCS /  │       │
│  │  (44 bytes)   │ push  │  Azure Blob  │       │
│  │  md5: a3f2... │       │              │       │
│  │              │◀──────│ train.csv    │       │
│  │              │ pull   │ (2.3 GB)     │       │
│  └──────────────┘        └──────────────┘       │
│                                                 │
│  .dvc file = metadata pointer (hash + size)     │
│  Actual data lives in remote storage            │
│  Git tracks the pointer, DVC tracks the data    │
└─────────────────────────────────────────────────┘
```

The `.dvc` file is tiny — it's just a YAML pointer containing an MD5 hash, file size, and remote path. Git tracks these pointers while DVC manages the actual large files in remote storage (S3, GCS, Azure Blob, or even a local directory).

### DVC Commands in Practice

```bash
# Initialize DVC in a git repo
dvc init

# Add remote storage
dvc remote add -d myremote s3://my-bucket/dvc-store

# Track a large dataset
dvc add data/train.csv
# Creates: data/train.csv.dvc (tracked by git)
# Creates: .gitignore entry for data/train.csv

# Push data to remote
git add data/train.csv.dvc data/.gitignore
git commit -m "Add training dataset v1"
dvc push

# Later, on another machine or after a git checkout
dvc pull  # Downloads data matching the current .dvc pointers
```

### DVC Pipelines: Reproducible ML Workflows

DVC pipelines define your entire ML workflow as a DAG with explicit inputs and outputs. If a dependency hasn't changed, DVC **skips that stage** — saving hours of compute.

```yaml
# dvc.yaml — Reproducible ML Pipeline
stages:
  prepare:
    cmd: python src/prepare.py
    deps:
      - src/prepare.py
      - data/raw/transactions.csv
    params:
      - prepare.test_size
      - prepare.random_seed
    outs:
      - data/processed/train.parquet
      - data/processed/test.parquet

  featurize:
    cmd: python src/featurize.py
    deps:
      - src/featurize.py
      - data/processed/train.parquet
      - data/processed/test.parquet
    params:
      - featurize.window_sizes
      - featurize.aggregations
    outs:
      - data/features/train_features.parquet
      - data/features/test_features.parquet

  train:
    cmd: python src/train.py
    deps:
      - src/train.py
      - data/features/train_features.parquet
    params:
      - train.n_estimators
      - train.max_depth
      - train.learning_rate
    outs:
      - models/model.pkl
    metrics:
      - metrics/train_metrics.json:
          cache: false
    plots:
      - plots/confusion_matrix.csv:
          x: predicted
          y: actual

  evaluate:
    cmd: python src/evaluate.py
    deps:
      - src/evaluate.py
      - models/model.pkl
      - data/features/test_features.parquet
    metrics:
      - metrics/eval_metrics.json:
          cache: false
    plots:
      - plots/roc_curve.csv:
          x: fpr
          y: tpr
```

```yaml
# params.yaml — Hyperparameters (tracked by git)
prepare:
  test_size: 0.2
  random_seed: 42

featurize:
  window_sizes: [7, 14, 30]
  aggregations: ["mean", "max", "count"]

train:
  n_estimators: 200
  max_depth: 15
  learning_rate: 0.01
```

```bash
# Run the full pipeline (skips unchanged stages)
dvc repro

# Compare metrics across git branches/commits
dvc metrics diff

# Visualize the pipeline DAG
dvc dag
#  prepare → featurize → train → evaluate
```

### Beyond DVC: Data Lake Versioning

| Tool           | Approach                           | Best For                          |
|----------------|------------------------------------|-----------------------------------|
| DVC            | Git-like pointers to remote files  | ML projects, small-medium data    |
| lakeFS         | Git-like branching for data lakes  | Data lake governance, S3/GCS      |
| Delta Lake     | Time travel via transaction log    | Spark ecosystems, ACID on lakes   |
| Iceberg        | Hidden partitioning, time travel   | Large-scale analytics, Trino      |

**lakeFS** gives you `git branch`, `git commit`, and `git merge` semantics for your entire data lake — you can create a branch of your S3 bucket, experiment with data transformations, and merge back without duplicating data. **Delta Lake** achieves versioning through a transaction log, enabling `SELECT * FROM table TIMESTAMP AS OF '2025-01-15'` for time travel queries.

### 💡 Interview Insight

> **Q: "How do you ensure reproducibility in ML experiments?"**
>
> Reproducibility requires versioning **four things**: (1) **Code** — Git, obviously. (2) **Data** — DVC, lakeFS, or Delta Lake snapshots. (3) **Environment** — Docker images, conda.yaml, or MLflow Projects. (4) **Configuration** — params.yaml tracked in Git, MLflow parameters logged per run. DVC pipelines tie all four together: each stage has explicit deps (code + data), params (config), and outs (artifacts), creating a fully reproducible DAG.

---

## Screen 5: ML Pipeline Orchestration

Individual scripts are not a pipeline. An ML pipeline is a **directed acyclic graph (DAG)** of containerized steps with dependency management, retry logic, caching, and metadata tracking.

### The Standard ML Pipeline

```
┌──────────────────────────────────────────────────────┐
│            STANDARD ML PIPELINE (DAG)                │
│                                                      │
│  ┌─────────┐   ┌───────────┐   ┌─────────────────┐  │
│  │  Data    │──▶│ Feature   │──▶│    Training      │  │
│  │  Prep    │   │ Engineer  │   │  (distributed)   │  │
│  └─────────┘   └───────────┘   └────────┬────────┘  │
│                                          │           │
│                                          ▼           │
│  ┌─────────┐   ┌───────────┐   ┌─────────────────┐  │
│  │  Model   │◀──│   Model   │◀──│   Evaluation    │  │
│  │  Deploy  │   │  Registry │   │ (validation     │  │
│  │         │   │           │   │  gates)          │  │
│  └─────────┘   └───────────┘   └─────────────────┘  │
│                                                      │
│  Each box = containerized step (Docker image)        │
│  Each arrow = data dependency + metadata             │
└──────────────────────────────────────────────────────┘
```

### Kubeflow Pipelines: Kubernetes-Native ML

Kubeflow Pipelines is the **most widely adopted** open-source ML orchestrator for Kubernetes environments. Each step runs as a container on K8s with full resource isolation.

```python
from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Model, Metrics

# ── Component 1: Data Preparation ────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "pyarrow", "scikit-learn"],
)
def prepare_data(
    input_path: str,
    test_size: float,
    train_data: Output[Dataset],
    test_data: Output[Dataset],
) -> None:
    """Load raw data, clean, and split."""
    import pandas as pd
    from sklearn.model_selection import train_test_split

    df = pd.read_parquet(input_path)
    df = df.dropna(subset=["amount", "is_fraud"])

    train_df, test_df = train_test_split(
        df, test_size=test_size, stratify=df["is_fraud"],
        random_state=42,
    )

    train_df.to_parquet(train_data.path)
    test_df.to_parquet(test_data.path)


# ── Component 2: Training ────────────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=[
        "pandas", "scikit-learn", "mlflow", "pyarrow",
    ],
)
def train_model(
    train_data: Input[Dataset],
    n_estimators: int,
    max_depth: int,
    model_artifact: Output[Model],
    metrics: Output[Metrics],
) -> None:
    """Train a fraud detection model."""
    import pandas as pd
    import pickle
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.metrics import f1_score

    df = pd.read_parquet(train_data.path)
    X = df.drop("is_fraud", axis=1)
    y = df["is_fraud"]

    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth,
        class_weight="balanced",
        random_state=42,
    )
    model.fit(X, y)

    f1 = f1_score(y, model.predict(X))
    metrics.log_metric("train_f1", f1)

    with open(model_artifact.path, "wb") as f:
        pickle.dump(model, f)


# ── Component 3: Evaluation Gate ─────────────────
@dsl.component(
    base_image="python:3.11-slim",
    packages_to_install=["pandas", "scikit-learn", "pyarrow"],
)
def evaluate_model(
    model_artifact: Input[Model],
    test_data: Input[Dataset],
    f1_threshold: float,
    metrics: Output[Metrics],
) -> bool:
    """Evaluate model. Return True if it passes the quality gate."""
    import pandas as pd
    import pickle
    from sklearn.metrics import f1_score, precision_score, recall_score

    with open(model_artifact.path, "rb") as f:
        model = pickle.load(f)

    df = pd.read_parquet(test_data.path)
    X_test = df.drop("is_fraud", axis=1)
    y_test = df["is_fraud"]

    y_pred = model.predict(X_test)
    f1 = f1_score(y_test, y_pred)
    precision = precision_score(y_test, y_pred)
    recall = recall_score(y_test, y_pred)

    metrics.log_metric("test_f1", f1)
    metrics.log_metric("test_precision", precision)
    metrics.log_metric("test_recall", recall)

    return f1 >= f1_threshold


# ── Pipeline Definition ──────────────────────────
@dsl.pipeline(
    name="fraud-detection-pipeline",
    description="End-to-end fraud detection training pipeline",
)
def fraud_pipeline(
    input_path: str = "gs://data/transactions.parquet",
    test_size: float = 0.2,
    n_estimators: int = 200,
    max_depth: int = 15,
    f1_threshold: float = 0.85,
):
    # Step 1: Prepare data
    prep_task = prepare_data(
        input_path=input_path,
        test_size=test_size,
    )

    # Step 2: Train model (depends on step 1)
    train_task = train_model(
        train_data=prep_task.outputs["train_data"],
        n_estimators=n_estimators,
        max_depth=max_depth,
    )

    # Step 3: Evaluate (depends on steps 1 & 2)
    eval_task = evaluate_model(
        model_artifact=train_task.outputs["model_artifact"],
        test_data=prep_task.outputs["test_data"],
        f1_threshold=f1_threshold,
    )
```

### Orchestrator Comparison

| Feature               | Kubeflow Pipelines       | SageMaker Pipelines       | Vertex AI Pipelines       |
|-----------------------|--------------------------|---------------------------|---------------------------|
| Infrastructure        | Any Kubernetes cluster   | AWS managed               | GCP managed               |
| Language              | Python DSL               | Python SDK                | Python (KFP-based)        |
| Container Support     | Any Docker image         | Built-in + custom         | Any Docker image          |
| Experiment Tracking   | Via metadata store       | Built-in                  | Built-in                  |
| Caching               | Step-level               | Step-level                | Step-level                |
| Cost Model            | K8s cluster costs        | Per-step pricing          | Per-step pricing          |
| Vendor Lock-in        | Low (portable)           | High (AWS)                | Medium (KFP compatible)   |
| Learning Curve        | Steep (K8s knowledge)    | Moderate                  | Moderate                  |

**ZenML** deserves a special mention — it's an abstraction layer that lets you write pipelines once and deploy them to any orchestrator (Kubeflow, Airflow, Vertex, SageMaker). Think of it as the "write once, run anywhere" of ML pipelines.

### 💡 Interview Insight

> **Q: "How do you decide between Kubeflow and a managed pipeline service?"**
>
> Use **Kubeflow** when you need portability across clouds, already run Kubernetes, want full control over infrastructure, or have strict data residency requirements. Use **managed services** (SageMaker/Vertex) when you want to minimize operational overhead, are single-cloud, and prefer paying for convenience over managing K8s clusters. The decision often comes down to: *Do you have a platform team that can operate Kubernetes?* If yes, Kubeflow. If no, go managed.

---

## Screen 6: CI/CD for Machine Learning

CI/CD for ML is fundamentally different from traditional software CI/CD because you're testing **three things**: code, data, and model quality. A code change that passes unit tests can still produce a terrible model.

### The ML CI/CD Pipeline

```
┌──────────────────────────────────────────────────────────┐
│                ML CI/CD PIPELINE                         │
│                                                          │
│  Code Push                                               │
│     │                                                    │
│     ▼                                                    │
│  ┌──────────────────────────────────────────────┐        │
│  │  CI: Code Quality                            │        │
│  │  ├── Linting (ruff, mypy)                    │        │
│  │  ├── Unit tests (pytest)                     │        │
│  │  └── Security scan (bandit)                  │        │
│  └──────────────────────┬───────────────────────┘        │
│                         ▼                                │
│  ┌──────────────────────────────────────────────┐        │
│  │  CI: Data Validation                         │        │
│  │  ├── Schema validation (Great Expectations)  │        │
│  │  ├── Distribution drift check                │        │
│  │  └── Data quality assertions                 │        │
│  └──────────────────────┬───────────────────────┘        │
│                         ▼                                │
│  ┌──────────────────────────────────────────────┐        │
│  │  CI: Model Training & Validation             │        │
│  │  ├── Train on sample data (fast smoke test)  │        │
│  │  ├── Model quality gates (F1 > 0.85)         │        │
│  │  ├── Performance regression check            │        │
│  │  └── Bias and fairness audit                 │        │
│  └──────────────────────┬───────────────────────┘        │
│                         ▼                                │
│  ┌──────────────────────────────────────────────┐        │
│  │  CD: Deployment                              │        │
│  │  ├── Register model in registry              │        │
│  │  ├── Deploy to staging (shadow mode)         │        │
│  │  ├── Integration tests against staging       │        │
│  │  ├── Canary / Blue-Green deploy to prod      │        │
│  │  └── Monitor rollout metrics                 │        │
│  └──────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
```

### Model Validation Gates

Model validation gates are automated checks that must pass before a model can be promoted. They are the **most important part of ML CI/CD**.

```python
# tests/test_model_quality.py
import pytest
import mlflow
import pandas as pd
from sklearn.metrics import f1_score, recall_score


@pytest.fixture
def model_and_data():
    """Load the candidate model and test data."""
    model = mlflow.pyfunc.load_model("models:/FraudDetector/Staging")
    test_df = pd.read_parquet("data/test/holdout.parquet")
    return model, test_df


class TestModelQualityGates:
    """Quality gates that must pass before production deployment."""

    def test_f1_above_threshold(self, model_and_data):
        model, df = model_and_data
        y_pred = model.predict(df.drop("is_fraud", axis=1))
        f1 = f1_score(df["is_fraud"], y_pred)
        assert f1 >= 0.85, f"F1 {f1:.3f} below threshold 0.85"

    def test_recall_above_threshold(self, model_and_data):
        """For fraud detection, recall is critical — missing fraud is costly."""
        model, df = model_and_data
        y_pred = model.predict(df.drop("is_fraud", axis=1))
        recall = recall_score(df["is_fraud"], y_pred)
        assert recall >= 0.90, f"Recall {recall:.3f} below threshold 0.90"

    def test_no_performance_regression(self, model_and_data):
        """New model must be at least as good as current production."""
        candidate, df = model_and_data
        production = mlflow.pyfunc.load_model(
            "models:/FraudDetector/Production"
        )

        X = df.drop("is_fraud", axis=1)
        y = df["is_fraud"]

        candidate_f1 = f1_score(y, candidate.predict(X))
        production_f1 = f1_score(y, production.predict(X))

        assert candidate_f1 >= production_f1 - 0.01, (
            f"Candidate F1 ({candidate_f1:.3f}) regressed vs "
            f"production ({production_f1:.3f})"
        )

    def test_prediction_latency(self, model_and_data):
        """Model must respond within SLA for real-time serving."""
        import time
        model, df = model_and_data
        single_row = df.drop("is_fraud", axis=1).iloc[[0]]

        start = time.perf_counter()
        for _ in range(100):
            model.predict(single_row)
        elapsed = (time.perf_counter() - start) / 100

        assert elapsed < 0.050, (
            f"Avg latency {elapsed*1000:.1f}ms exceeds 50ms SLA"
        )

    def test_model_size(self, model_and_data):
        """Ensure model isn't too large for deployment target."""
        import os
        model_path = mlflow.artifacts.download_artifacts(
            "models:/FraudDetector/Staging"
        )
        size_mb = os.path.getsize(
            os.path.join(model_path, "model.pkl")
        ) / (1024 * 1024)
        assert size_mb < 500, f"Model size {size_mb:.1f}MB exceeds 500MB limit"
```

### Automated Retraining Triggers

Retraining can be triggered by multiple signals:

```
┌─────────── Retraining Triggers ──────────────────┐
│                                                   │
│  1. SCHEDULED     — Weekly/monthly cron job       │
│  2. DATA DRIFT    — Feature distributions shift   │
│  3. PERF DECAY    — Online metrics degrade        │
│  4. NEW DATA      — Threshold of new labels       │
│  5. CODE CHANGE   — Model code updated via CI/CD  │
│                                                   │
│  Trigger → Run Pipeline → Validate → Deploy/Fail  │
└───────────────────────────────────────────────────┘
```

### Deployment Strategies for Models

| Strategy         | Description                        | Risk     | Rollback Speed |
|------------------|------------------------------------|----------|----------------|
| Shadow / Dark    | New model runs alongside old,      | None     | N/A            |
|                  | predictions logged but not served  |          |                |
| Canary           | 5% traffic → new model, monitor    | Low      | Fast           |
| Blue-Green       | Full switch, old version standby   | Medium   | Instant        |
| A/B Test         | Statistical comparison of models   | Low      | Fast           |
| Rolling          | Gradual instance-by-instance swap  | Low      | Medium         |

### 💡 Interview Insight

> **Q: "What's different about CI/CD for ML vs traditional software?"**
>
> Three key differences: (1) **You test data, not just code.** A schema change in upstream data can break your model silently. Use Great Expectations or Pandera for data validation. (2) **You have model quality gates.** Unit tests passing doesn't mean the model is good — you need metrics thresholds, regression tests against the production model, and bias checks. (3) **Deployment is riskier.** A bad model can silently serve wrong predictions for days before anyone notices. Shadow deployments and canary releases are essential. The core principle: **never deploy a model you haven't validated against the current production model on a held-out dataset.**

---

## Screen 7: Putting It All Together — End-to-End MLOps Architecture

Now let's connect every piece into a cohesive system. In interviews, you'll be asked to **design an end-to-end ML platform**. Here's the reference architecture:

### Complete MLOps Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    END-TO-END MLOPS PLATFORM                     │
│                                                                  │
│  DATA LAYER                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │
│  │Data Lake │  │DVC/lakeFS│  │  Feature  │  │ Great        │     │
│  │(S3/GCS)  │  │Versioning│  │  Store    │  │ Expectations │     │
│  │          │  │          │  │ (Feast)   │  │ (Validation) │     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘     │
│       │              │             │               │              │
│  ─────┴──────────────┴─────────────┴───────────────┴──────────   │
│                                                                  │
│  ORCHESTRATION LAYER                                             │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Kubeflow Pipelines / Vertex AI / SageMaker Pipelines    │    │
│  │  prep → featurize → train → evaluate → register → deploy│    │
│  └──────────────────────────────┬───────────────────────────┘    │
│                                 │                                │
│  EXPERIMENT & MODEL LAYER       │                                │
│  ┌──────────────┐  ┌───────────┴────┐  ┌──────────────┐         │
│  │   MLflow     │  │    Model       │  │  Container   │         │
│  │  Tracking    │  │   Registry     │  │  Registry    │         │
│  │  (Params,    │  │ (Stage mgmt)   │  │  (Docker)    │         │
│  │   Metrics)   │  │               │  │              │         │
│  └──────────────┘  └───────┬───────┘  └──────┬───────┘         │
│                            │                  │                  │
│  CI/CD LAYER               │                  │                  │
│  ┌─────────────────────────┴──────────────────┴──────────┐      │
│  │  GitHub Actions / GitLab CI / Jenkins                  │      │
│  │  lint → test → train → validate → build → deploy      │      │
│  └───────────────────────────────────┬───────────────────┘      │
│                                      │                          │
│  SERVING LAYER                       │                          │
│  ┌──────────────┐  ┌───────────┐  ┌──┴───────────┐             │
│  │  Model       │  │   API     │  │   Canary     │             │
│  │  Server      │  │  Gateway  │  │  Controller  │             │
│  │(TF Serving / │  │(Kong/     │  │  (Istio/     │             │
│  │ Triton /     │  │ Envoy)    │  │   Flagger)   │             │
│  │ MLflow)      │  │           │  │              │             │
│  └──────────────┘  └───────────┘  └──────────────┘             │
│                                                                  │
│  MONITORING LAYER                                                │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────┐             │
│  │  Prometheus  │  │ Evidently │  │  Alerting    │             │
│  │  + Grafana   │  │  AI       │  │  (PagerDuty/ │             │
│  │ (Infra)      │  │ (Drift)   │  │   Slack)     │             │
│  └──────────────┘  └───────────┘  └──────────────┘             │
└──────────────────────────────────────────────────────────────────┘
```

### Designing for an Interview: The Checklist

When asked to design an ML platform, walk through these layers systematically:

1. **Data Layer**: Where does data live? How is it versioned? How do you ensure quality?
2. **Feature Layer**: How do you compute, store, and serve features consistently?
3. **Training Layer**: How are experiments tracked? How are pipelines orchestrated?
4. **Evaluation Layer**: What quality gates exist? How do you prevent regression?
5. **Deployment Layer**: How do models get to production? What's the rollback strategy?
6. **Monitoring Layer**: How do you detect drift? How do you trigger retraining?

### Tool Selection Guide

| Need                    | Open-Source Choice          | Managed Alternative          |
|-------------------------|-----------------------------|------------------------------|
| Experiment Tracking     | MLflow                      | W&B, Comet                   |
| Feature Store           | Feast                       | Databricks, Vertex, Tecton   |
| Data Versioning         | DVC, lakeFS                 | Delta Lake, Iceberg          |
| Pipeline Orchestration  | Kubeflow, Airflow           | SageMaker, Vertex Pipelines  |
| Model Serving           | Triton, TF Serving, BentoML | SageMaker Endpoints, Vertex  |
| Data Validation         | Great Expectations, Pandera | Built-in (SageMaker/Vertex)  |
| Drift Detection         | Evidently, NannyML          | SageMaker Model Monitor      |
| ML Abstraction          | ZenML, MetaFlow             | Databricks MLflow managed    |

### 💡 Interview Insight

> **Q: "Design an MLOps platform for a team of 20 data scientists."**
>
> Start with the **highest-leverage, lowest-effort** components: (1) **MLflow** for experiment tracking and model registry — it's the foundation everything else builds on. (2) **DVC + Git** for data and code versioning — reproducibility is non-negotiable. (3) **A simple orchestrator** — start with Airflow if you don't have K8s, or Kubeflow if you do. (4) **Feature store only if needed** — don't introduce Feast until you have feature reuse problems across multiple models. (5) **CI/CD with model quality gates** — this is where most teams get the biggest ROI. The key insight: **don't build a Level 4 platform when your team is at Level 0. Iterate incrementally.** Start with experiment tracking, then add pipeline automation, then CI/CD, then monitoring.

---

## Screen 8: Quiz — Test Your MLOps Knowledge

Test your understanding with these interview-caliber questions.

### Question 1

**What is the primary purpose of a feature store in ML systems?**

A) To store trained model artifacts and their versions  
B) To provide a centralized, consistent feature computation layer for both training and serving ✅  
C) To orchestrate the steps in an ML training pipeline  
D) To track hyperparameters and metrics across experiments  

**Explanation**: A feature store eliminates training-serving skew by ensuring the same feature values are used in both contexts. It is *not* a model registry (A), an orchestrator (C), or an experiment tracker (D).

---

### Question 2

**In Feast, what does "materialization" refer to?**

A) Converting raw data into feature definitions  
B) Syncing feature values from the offline store to the online store for low-latency serving ✅  
C) Training a model on features from the feature store  
D) Validating feature schema against expected types  

**Explanation**: Materialization is the batch process that copies computed feature values from the offline store (warehouse) to the online store (Redis/DynamoDB) so they can be served in real-time. This is typically run on a schedule (e.g., hourly or daily).

---

### Question 3

**What problem do point-in-time joins solve?**

A) Joining features from multiple tables efficiently  
B) Preventing data leakage by using feature values as they existed at the time of each training event ✅  
C) Joining streaming and batch data sources together  
D) Improving query performance on large datasets  

**Explanation**: Without point-in-time correctness, your training data might include feature values from *after* the event you're predicting — that's future information leakage. Point-in-time joins ensure temporal correctness by matching each training example to the feature values that existed at that specific timestamp.

---

### Question 4

**Which MLOps maturity level is characterized by automated retraining triggered by drift detection?**

A) Level 1 — ML Pipeline Automation  
B) Level 2 — CI/CD Pipeline Automation  
C) Level 3 — Automated Retraining + Monitoring ✅  
D) Level 4 — Full Autonomous ML System  

**Explanation**: Level 3 adds continuous monitoring and automated retraining triggers. Level 1 automates the training pipeline but runs it manually. Level 2 adds CI/CD for the pipeline code. Level 4 goes further with autonomous feature engineering and model selection.

---

### Question 5

**In a DVC pipeline (`dvc.yaml`), what happens when you run `dvc repro` and no dependencies have changed for a stage?**

A) The stage runs again to ensure consistency  
B) DVC raises an error because nothing changed  
C) The stage is skipped because its inputs haven't changed ✅  
D) DVC deletes the cached output and reruns  

**Explanation**: DVC tracks the hash of all dependencies (`deps`), parameters (`params`), and outputs (`outs`) for each stage. If none of the inputs have changed since the last run, the stage is skipped entirely. This saves significant compute time — you only rerun what's necessary, similar to how `make` works for compiled code.

---

## Key Takeaways

### The MLOps Essentials Cheat Sheet

```
┌─────────────────────────────────────────────────────┐
│              MLOPS INTERVIEW CHEAT SHEET            │
│                                                     │
│  EXPERIMENT TRACKING (MLflow)                       │
│  • Experiments → Runs → Params + Metrics + Artifacts│
│  • Model Registry: Staging → Production workflow    │
│  • autolog() for quick wins, manual log for control │
│  • MLflow Projects for reproducible runs            │
│                                                     │
│  FEATURE STORES (Feast)                             │
│  • Offline store (training) + Online store (serving)│
│  • Point-in-time joins prevent data leakage         │
│  • Materialization syncs offline → online           │
│  • Feature Services bundle features per use case    │
│                                                     │
│  DATA VERSIONING (DVC)                              │
│  • .dvc files = Git-tracked pointers to large data  │
│  • dvc.yaml defines reproducible pipeline DAGs      │
│  • dvc repro skips unchanged stages (like make)     │
│  • Remote storage: S3, GCS, Azure Blob              │
│                                                     │
│  PIPELINE ORCHESTRATION (Kubeflow)                  │
│  • Each step = containerized component on K8s       │
│  • DSL for defining pipeline DAGs in Python         │
│  • Alternatives: SageMaker, Vertex AI, ZenML        │
│                                                     │
│  CI/CD FOR ML                                       │
│  • Test code + data + model quality                 │
│  • Model validation gates (metrics thresholds)      │
│  • Deployment: shadow → canary → blue-green         │
│  • Retraining triggers: schedule, drift, new data   │
│                                                     │
│  MATURITY LEVELS                                    │
│  • 0: Notebooks  → 1: Pipelines → 2: CI/CD         │
│  • 3: Auto-retrain → 4: Autonomous                 │
│  • Most teams are 1-2. Iterate, don't leap.         │
└─────────────────────────────────────────────────────┘
```

### The Top 5 Things Interviewers Want to Hear

1. **You understand training-serving skew** and know feature stores solve it through consistent feature computation.

2. **You version everything** — code (Git), data (DVC/lakeFS), models (MLflow Registry), and config (params.yaml).

3. **You don't deploy without validation gates** — metrics thresholds, regression checks against production, latency SLAs, and bias audits.

4. **You know when to use what** — Kubeflow for K8s shops, managed services for small teams, Feast when you have feature reuse problems.

5. **You think incrementally** — you don't propose a Level 4 platform on day one. You start with experiment tracking, add pipeline automation, then CI/CD, then monitoring. Each step delivers value.

### Further Reading

- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [Feast Documentation](https://docs.feast.dev/)
- [DVC Documentation](https://dvc.org/doc)
- [Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/)
- [Google's MLOps Whitepaper (Practitioners Guide)](https://cloud.google.com/resources/mlops-whitepaper)
- [Made With ML — MLOps Course](https://madewithml.com/)

---

*Next Module: Model Serving & Inference Optimization →*
