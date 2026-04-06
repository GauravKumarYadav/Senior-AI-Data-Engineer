# Model Serving & Deployment

> **Module Goal**: Master the full spectrum of model serving — from batch inference to real-time APIs to LLM serving — so you can design production deployment architectures, optimize for latency and cost, and confidently answer deployment interview questions.

---

## Screen 1: Serving Patterns — Choosing How Your Model Meets the World

The way you serve a model is as important as the model itself. A brilliant model behind the wrong serving pattern is like a Ferrari on a dirt road. Let's break down the four fundamental patterns.

### The Four Serving Patterns

```
┌──────────────────────────────────────────────────────────────┐
│                  MODEL SERVING PATTERNS                      │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   BATCH     │  │  REAL-TIME  │  │     STREAMING       │  │
│  │             │  │    API      │  │                     │  │
│  │ ┌───┐ ┌───┐│  │  Request ──▶│  │  Event ──▶ Model    │  │
│  │ │DB │→│Job││  │  ◀── Reply  │  │           ──▶ Sink   │  │
│  │ └───┘ └───┘│  │  (ms-level) │  │  (sub-second)       │  │
│  │ (hours)    │  └─────────────┘  └─────────────────────┘  │
│  └─────────────┘                                            │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              EMBEDDED / EDGE                          │  │
│  │   ┌────────┐     ┌──────────┐     ┌───────────┐      │  │
│  │   │ Mobile │     │  Browser │     │ IoT / Edge│      │  │
│  │   │  App   │     │  (WASM)  │     │  Device   │      │  │
│  │   └────────┘     └──────────┘     └───────────┘      │  │
│  │         Model runs ON the device — zero latency       │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Batch Inference

Batch inference processes large volumes of data on a schedule — hourly, daily, or weekly. Predictions are precomputed and stored for lookup.

**When to use**: Recommendation engines (precompute top-N for all users), credit scoring (nightly batch), churn prediction (weekly reports).

```python
# batch_inference.py — Spark-based batch scoring
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, udf
from pyspark.sql.types import FloatType
import mlflow.pyfunc

spark = SparkSession.builder \
    .appName("batch-inference") \
    .config("spark.executor.memory", "8g") \
    .getOrCreate()

# Load model from registry
model = mlflow.pyfunc.spark_udf(
    spark,
    model_uri="models:/churn-classifier/Production",
    result_type=FloatType(),
)

# Score entire customer table
customers = spark.read.parquet("s3://data-lake/customers/")
predictions = customers.withColumn(
    "churn_probability", model(*[col(f) for f in feature_columns])
)

# Write predictions for downstream consumption
predictions.write \
    .mode("overwrite") \
    .partitionBy("region") \
    .parquet("s3://data-lake/predictions/churn/")
```

### Real-Time API (REST / gRPC)

The model runs behind an HTTP or gRPC endpoint. Each request gets an immediate prediction.

**When to use**: Fraud detection (transaction-time scoring), search ranking, dynamic pricing, chatbots/LLMs.

**REST vs gRPC trade-offs**:

| Aspect               | REST (HTTP/JSON)                | gRPC (HTTP/2 + Protobuf)        |
|-----------------------|---------------------------------|---------------------------------|
| Latency               | Higher (text serialization)    | Lower (binary serialization)   |
| Browser support       | Native                         | Requires grpc-web proxy        |
| Streaming             | Limited (SSE, WebSockets)      | Native bidirectional           |
| Debugging             | Easy (curl, Postman)           | Harder (need grpcurl)          |
| Best for              | External APIs, simple cases    | Internal microservices, perf   |

### Streaming Inference

Process events as they arrive through a message bus (Kafka, Kinesis). The model sits in the stream processing pipeline.

```python
# streaming_inference.py — Spark Structured Streaming + ML
from pyspark.sql import SparkSession
from pyspark.ml import PipelineModel

spark = SparkSession.builder \
    .appName("streaming-fraud-detection") \
    .getOrCreate()

model = PipelineModel.load("s3://models/fraud-detector-v3/")

# Read from Kafka
transactions = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "transactions") \
    .load()

# Parse and score
parsed = parse_transaction_schema(transactions)
scored = model.transform(parsed)

# Write flagged transactions to alert topic
scored.filter("prediction = 1") \
    .writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("topic", "fraud-alerts") \
    .option("checkpointLocation", "/checkpoints/fraud/") \
    .start()
```

### Embedded / Edge

The model is compiled and shipped inside the client application — mobile, browser (ONNX.js, TF.js), or IoT device.

**When to use**: Offline-capable apps, ultra-low latency (<1ms), privacy-sensitive (data never leaves device), bandwidth-constrained environments.

### The Decision Framework

```
                    ┌─────────────────────┐
                    │  What latency does   │
                    │  the use case need?  │
                    └──────────┬──────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
          Hours/Days      Milliseconds     Sub-second
               │               │               │
               ▼               ▼               ▼
        ┌──────────┐    ┌──────────┐    ┌──────────────┐
        │  BATCH   │    │ REAL-TIME│    │  STREAMING   │
        │          │    │   API    │    │              │
        └──────────┘    └──────────┘    └──────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Network available? │
                    └──────────┬──────────┘
                           No │
                              ▼
                       ┌──────────┐
                       │ EMBEDDED │
                       └──────────┘
```

| Factor           | Batch          | Real-Time API   | Streaming       | Embedded        |
|------------------|----------------|-----------------|-----------------|-----------------|
| Latency          | Hours          | 10-500ms        | 100ms-seconds   | <1ms            |
| Throughput       | Millions/run   | 100s-1000s RPS  | 1000s events/s  | Device-limited  |
| Cost model       | Compute bursts | Always-on infra | Always-on infra | Device cost     |
| Model size limit | Unlimited      | ~Cluster RAM    | ~Executor RAM   | <500MB typical  |
| Freshness        | Stale          | Live            | Near-live       | Stale (updates) |

### 💡 Interview Insight

> **Q: "Your team has a recommendation system. When would you choose batch vs. real-time serving?"**
>
> **A**: Use **batch** for the homepage "top picks" — precompute nightly for all users, store in a fast KV store (Redis/DynamoDB), and serve with sub-ms lookup. Use **real-time** for "similar items" on the product page — the context (which product the user is viewing) isn't known until request time. A hybrid approach is common: batch for personalization features, real-time for contextual re-ranking.

---

## Screen 2: Model Serving Frameworks — The Runtime Layer

Once you've chosen a serving pattern, you need a runtime. This is where serving frameworks come in — they handle model loading, request batching, health checks, metrics, and multi-model management.

### Framework Landscape

```
┌─────────────────────────────────────────────────────────────┐
│              MODEL SERVING FRAMEWORKS                       │
│                                                             │
│  Traditional ML / DL:                                       │
│  ┌────────────┐ ┌──────────────┐ ┌───────────────────────┐ │
│  │ TorchServe │ │  TF Serving  │ │  Triton Inference     │ │
│  │ (PyTorch)  │ │ (TensorFlow) │ │  Server (NVIDIA)      │ │
│  └────────────┘ └──────────────┘ └───────────────────────┘ │
│                                                             │
│  LLM-Specific:                                              │
│  ┌────────────┐ ┌──────────────┐                           │
│  │   vLLM     │ │     TGI      │                           │
│  │ (fastest   │ │ (HuggingFace │                           │
│  │  OSS LLM)  │ │  optimized)  │                           │
│  └────────────┘ └──────────────┘                           │
│                                                             │
│  Higher-Level Orchestration:                                │
│  ┌────────────┐ ┌──────────────┐                           │
│  │  BentoML   │ │  Ray Serve   │                           │
│  │ (packaging)│ │ (distributed)│                           │
│  └────────────┘ └──────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### TorchServe (PyTorch Native)

TorchServe is the official serving solution for PyTorch models. It uses a **handler-based** architecture where you define how to preprocess, infer, and postprocess.

```python
# custom_handler.py — TorchServe custom handler
import torch
from ts.torch_handler.base_handler import BaseHandler
from torchvision import transforms


class ImageClassifierHandler(BaseHandler):
    """Custom handler for image classification."""

    def initialize(self, context):
        """Load model and set up transforms."""
        super().initialize(context)
        self.transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225],
            ),
        ])

    def preprocess(self, data):
        """Convert raw input to tensor batch."""
        images = []
        for row in data:
            image = Image.open(io.BytesIO(row["body"]))
            images.append(self.transform(image))
        return torch.stack(images)

    def inference(self, data):
        """Run forward pass with no_grad."""
        with torch.no_grad():
            return self.model(data)

    def postprocess(self, inference_output):
        """Convert logits to class probabilities."""
        probs = torch.softmax(inference_output, dim=1)
        top5 = torch.topk(probs, 5, dim=1)
        return [
            {
                "classes": top5.indices[i].tolist(),
                "probabilities": top5.values[i].tolist(),
            }
            for i in range(len(probs))
        ]
```

```bash
# Package and serve
torch-model-archiver \
    --model-name resnet50 \
    --version 1.0 \
    --serialized-file model.pt \
    --handler custom_handler.py \
    --export-path model_store/

torchserve --start \
    --model-store model_store \
    --models resnet50=resnet50.mar \
    --ts-config config.properties
```

**Key TorchServe features**: Dynamic batching, model versioning, metrics endpoint (Prometheus), A/B testing via model snapshots, TorchScript + eager mode support.

### NVIDIA Triton Inference Server

Triton is the **Swiss Army knife** of model serving. It supports multiple frameworks (PyTorch, TensorFlow, ONNX, TensorRT, Python) simultaneously and is heavily optimized for GPU utilization.

```
# Triton model repository structure
model_repository/
├── text_encoder/
│   ├── config.pbtxt
│   └── 1/
│       └── model.onnx
├── image_classifier/
│   ├── config.pbtxt
│   └── 1/
│       └── model.plan        # TensorRT optimized
└── ensemble_pipeline/
    └── config.pbtxt          # Chains models together
```

```protobuf
# config.pbtxt — Triton model configuration
name: "text_encoder"
platform: "onnxruntime_onnx"
max_batch_size: 64

input [
  {
    name: "input_ids"
    data_type: TYPE_INT64
    dims: [ -1 ]              # Dynamic sequence length
  },
  {
    name: "attention_mask"
    data_type: TYPE_INT64
    dims: [ -1 ]
  }
]

output [
  {
    name: "embeddings"
    data_type: TYPE_FP32
    dims: [ 768 ]
  }
]

# Dynamic batching — Triton's superpower
dynamic_batching {
  preferred_batch_size: [ 8, 16, 32 ]
  max_queue_delay_microseconds: 100
}

# Instance groups — how many model copies
instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [ 0 ]
  }
]
```

### vLLM — The LLM Serving King

vLLM introduced **PagedAttention**, which manages KV-cache memory like an OS manages virtual memory — in non-contiguous pages. This eliminates memory fragmentation and achieves 2-4x higher throughput than naive implementations.

```bash
# Serve Llama-3-8B with AWQ quantization
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-8B-Instruct \
    --quantization awq \
    --dtype half \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.90 \
    --tensor-parallel-size 2 \
    --enable-chunked-prefill \
    --max-num-seqs 256 \
    --port 8000

# Test with OpenAI-compatible API
curl http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Meta-Llama-3-8B-Instruct",
        "prompt": "Explain MLOps in one paragraph:",
        "max_tokens": 200,
        "temperature": 0.7
    }'
```

**Why vLLM wins**: PagedAttention (near-zero KV-cache waste), continuous batching (new requests don't wait for old ones), speculative decoding, OpenAI-compatible API, prefix caching for shared system prompts.

### TGI (Text Generation Inference by HuggingFace)

TGI is HuggingFace's optimized inference server for transformer models. It supports Flash Attention, tensor parallelism, and token streaming out of the box.

```bash
# Serve with TGI using Docker
docker run --gpus all \
    -p 8080:80 \
    -v /models:/data \
    ghcr.io/huggingface/text-generation-inference:latest \
    --model-id meta-llama/Meta-Llama-3-8B-Instruct \
    --quantize gptq \
    --max-input-length 4096 \
    --max-total-tokens 8192 \
    --max-batch-prefill-tokens 4096
```

### Framework Comparison

| Feature              | TorchServe       | Triton           | vLLM             | TGI              |
|----------------------|------------------|------------------|-------------------|-------------------|
| Primary use          | PyTorch models   | Multi-framework  | LLM serving       | HF transformers   |
| GPU optimization     | Good             | Excellent        | Excellent         | Very good          |
| Dynamic batching     | Yes              | Yes (advanced)   | Continuous        | Continuous         |
| Multi-model          | Yes              | Yes (native)     | No                | No                 |
| Streaming output     | Limited          | Yes              | Yes (SSE)         | Yes (SSE)          |
| Quantization         | Via model        | TensorRT, ONNX   | AWQ, GPTQ, FP8   | GPTQ, AWQ, EETQ   |
| Ease of setup        | Medium           | Complex          | Simple            | Simple             |

### 💡 Interview Insight

> **Q: "You need to serve a 70B parameter LLM with low latency. What framework and hardware do you choose?"**
>
> **A**: I'd use **vLLM** with AWQ INT4 quantization. A 70B model in FP16 needs ~140GB VRAM (70B × 2 bytes). With INT4 quantization, that drops to ~35GB + overhead ≈ 40GB. I'd use **2× A100 80GB** GPUs with tensor parallelism (`--tensor-parallel-size 2`). vLLM's PagedAttention and continuous batching will maximize throughput. For production, I'd front it with KServe on Kubernetes for autoscaling and canary deployments.

---

## Screen 3: Containerization & Kubernetes for ML

Models don't ship as Python scripts in production. They ship as **containers** running on **Kubernetes**. This screen covers the packaging and orchestration layer.

### Docker for Model Serving

Two strategies for including the model in the container:

**Strategy 1: Model Baked In** — Model weights are copied into the image at build time.
- ✅ Self-contained, reproducible, no runtime downloads
- ❌ Huge images (multi-GB for LLMs), slow CI/CD, rebuild per model version

**Strategy 2: Model Fetched at Startup** — Container downloads the model from a registry (S3, GCS, MLflow) at startup.
- ✅ Smaller images, same image for multiple model versions
- ❌ Cold start penalty, requires network access, potential for version mismatch

```dockerfile
# Dockerfile — Multi-stage build for model serving
# Stage 1: Build dependencies
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install \
    -r requirements.txt

# Stage 2: Runtime image
FROM python:3.11-slim AS runtime

# Security: non-root user
RUN groupadd -r mlserve && \
    useradd -r -g mlserve -d /app -s /sbin/nologin mlserve

WORKDIR /app
COPY --from=builder /install /usr/local
COPY src/ ./src/
COPY configs/ ./configs/

# Option A: Bake model in (for smaller models)
# COPY models/classifier_v3.onnx ./models/

# Option B: Fetch at startup (for larger models)
ENV MODEL_URI="s3://model-registry/classifier/v3/"
ENV MODEL_LOCAL_PATH="/app/models/"

# Health check for K8s liveness probe
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Expose metrics and serving ports
EXPOSE 8080 8081

USER mlserve

ENTRYPOINT ["python", "-m", "src.server"]
CMD ["--config", "configs/production.yaml"]
```

```python
# src/server.py — FastAPI model server
from contextlib import asynccontextmanager
from typing import Any

import numpy as np
import onnxruntime as ort
from fastapi import FastAPI
from pydantic import BaseModel


class PredictRequest(BaseModel):
    features: list[float]

class PredictResponse(BaseModel):
    prediction: float
    confidence: float
    model_version: str


model_session: ort.InferenceSession | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Load model on startup, cleanup on shutdown."""
    global model_session
    model_session = ort.InferenceSession(
        "models/classifier_v3.onnx",
        providers=["CUDAExecutionProvider", "CPUExecutionProvider"],
    )
    # Warmup inference to avoid cold-start latency
    dummy = np.zeros((1, 128), dtype=np.float32)
    model_session.run(None, {"input": dummy})
    yield
    model_session = None

app = FastAPI(lifespan=lifespan)

@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "healthy"}

@app.post("/predict", response_model=PredictResponse)
async def predict(req: PredictRequest) -> PredictResponse:
    inputs = np.array([req.features], dtype=np.float32)
    outputs = model_session.run(None, {"input": inputs})
    prob = float(outputs[0][0][1])
    return PredictResponse(
        prediction=round(prob),
        confidence=prob,
        model_version="v3.1.0",
    )
```

### KServe — Kubernetes-Native Model Serving

KServe (formerly KFServing) provides a **Kubernetes CRD** called `InferenceService` that handles model deployment, autoscaling, canary rollouts, and request routing.

```
┌─────────────────────────────────────────────────────┐
│                 KServe Architecture                  │
│                                                     │
│   Ingress (Istio / Knative)                         │
│        │                                            │
│        ▼                                            │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│   │  Router  │───▶│Predictor │    │Transform-│     │
│   │ (traffic │    │ (model   │◀──▶│  er      │     │
│   │  split)  │    │  runtime)│    │ (pre/    │     │
│   └──────────┘    └──────────┘    │  post)   │     │
│        │                          └──────────┘     │
│        │                                            │
│        └───▶ ┌──────────┐    ┌──────────┐          │
│              │Predictor │    │Explainer │          │
│              │ (canary) │    │ (SHAP /  │          │
│              └──────────┘    │  LIME)   │          │
│                              └──────────┘          │
└─────────────────────────────────────────────────────┘
```

```yaml
# kserve-inference-service.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: fraud-detector
  namespace: ml-serving
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  predictor:
    # Autoscaling configuration
    minReplicas: 2              # Avoid cold starts
    maxReplicas: 20
    scaleTarget: 10             # Target concurrency per pod
    scaleMetric: concurrency

    # Container spec
    containers:
      - name: kserve-container
        image: registry.example.com/fraud-detector:v3.1
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
            nvidia.com/gpu: "1"
          limits:
            cpu: "4"
            memory: "8Gi"
            nvidia.com/gpu: "1"
        env:
          - name: MODEL_NAME
            value: fraud-detector
          - name: STORAGE_URI
            value: s3://models/fraud-detector/v3.1/
        ports:
          - containerPort: 8080
            protocol: TCP
        readinessProbe:
          httpGet:
            path: /v2/health/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

    # Canary traffic split
    canaryTrafficPercent: 10

  # Transformer for feature engineering
  transformer:
    containers:
      - name: feature-transformer
        image: registry.example.com/fraud-features:v2
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
```

### Seldon Core vs KServe

| Aspect                | KServe                        | Seldon Core                   |
|-----------------------|-------------------------------|-------------------------------|
| K8s integration       | Knative + Istio native        | Custom operators              |
| Autoscaling           | Knative (scale-to-zero)       | HPA-based                     |
| Canary/A-B            | Built-in traffic split        | Built-in, more flexible       |
| Explainability        | Native explainer component    | Alibi integration             |
| Multi-model           | ModelMesh (shared memory)     | Multi-armed bandit routing    |
| Learning curve        | Moderate                      | Steeper                       |

### Cold Start Mitigation

Cold starts kill latency SLOs. Here's how to fight them:

1. **`minReplicas: 2`** — Always keep warm pods running
2. **Model warmup** — Run dummy inference during container startup
3. **Model caching** — Use init containers to pre-download models to shared volumes
4. **Smaller models** — Quantize and distill to reduce load time
5. **Readiness probes** — Don't route traffic until the model is loaded and warm

### 💡 Interview Insight

> **Q: "How would you handle GPU autoscaling for a model serving endpoint?"**
>
> **A**: Standard HPA scales on CPU/memory, which doesn't reflect GPU load. I'd use **custom metrics** from NVIDIA DCGM (Data Center GPU Manager) exported to Prometheus — specifically `DCGM_FI_DEV_GPU_UTIL` (GPU utilization) and `DCGM_FI_DEV_FB_USED` (GPU memory used). Configure HPA to scale when GPU utilization exceeds 70%. Combine with KServe's concurrency-based scaling (`scaleTarget: 10`) so pods scale based on in-flight request count, which correlates better with GPU load than raw utilization. Always set `minReplicas ≥ 1` for latency-sensitive services to avoid cold starts.

---

## Screen 4: A/B Testing, Canary, and Safe Rollouts

Deploying a model is not a "ship it and pray" activity. Production ML systems need **progressive delivery** — the ability to test new models on real traffic without risking the entire user base.

### Deployment Strategies Compared

```
┌────────────────────────────────────────────────────────────┐
│              DEPLOYMENT STRATEGIES                          │
│                                                            │
│  BLUE-GREEN                  CANARY                        │
│  ┌──────┐  ┌──────┐         ┌──────────────────────────┐  │
│  │Blue  │  │Green │         │  1% ──▶ 5% ──▶ 25% ──▶  │  │
│  │(v1)  │  │(v2)  │         │      100% rollout        │  │
│  └──┬───┘  └──┬───┘         └──────────────────────────┘  │
│     │         │                                            │
│  100% ──switch──▶ 100%      Gradual traffic shift          │
│  Instant cutover                                           │
│                                                            │
│  SHADOW                      A/B TEST                      │
│  ┌──────────────────┐       ┌──────────────────────────┐  │
│  │ Request ──▶ v1 ──▶ User │ │ Traffic ──▶ v1 (50%)    │  │
│  │    └────▶ v2 ──▶ /dev/  │ │        └──▶ v2 (50%)    │  │
│  │           null   │       │ │ Measure business metric │  │
│  └──────────────────┘       └──────────────────────────┘  │
│  v2 gets real traffic                                      │
│  but results are discarded   Statistical significance      │
└────────────────────────────────────────────────────────────┘
```

### Strategy Deep Dive

| Strategy     | Risk    | Traffic to new | Rollback speed | Use case                      |
|--------------|---------|----------------|----------------|-------------------------------|
| Blue-Green   | Medium  | 100% (switch)  | Instant         | Simple models, low risk       |
| Canary       | Low     | 1% → 100%     | Fast            | Most ML deployments           |
| Shadow       | Minimal | 100% (silent)  | N/A             | Validate before any exposure  |
| A/B Test     | Medium  | 50/50 split    | Fast            | Measure business impact       |

### Canary Release in Practice

```yaml
# Istio VirtualService for canary routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fraud-detector-vs
spec:
  hosts:
    - fraud-detector.ml-serving.svc.cluster.local
  http:
    - route:
        # Stable version: 95% traffic
        - destination:
            host: fraud-detector-v2
            port:
              number: 8080
          weight: 95
        # Canary version: 5% traffic
        - destination:
            host: fraud-detector-v3
            port:
              number: 8080
          weight: 5
      # Automatic rollback on error spike
      retries:
        attempts: 3
        retryOn: 5xx
```

### Shadow Deployment Pattern

Shadow deployments are the **safest** way to test a new model in production. The new model receives real traffic but its responses are discarded — only logged for offline comparison.

```python
# shadow_proxy.py — Shadow deployment router
import asyncio
import time
from typing import Any

import httpx
from fastapi import FastAPI, Request

app = FastAPI()

PRIMARY_URL = "http://model-v2:8080/predict"
SHADOW_URL = "http://model-v3:8080/predict"


@app.post("/predict")
async def predict(request: Request) -> dict[str, Any]:
    body = await request.json()

    async with httpx.AsyncClient(timeout=5.0) as client:
        # Fire both requests concurrently
        primary_task = client.post(PRIMARY_URL, json=body)
        shadow_task = client.post(SHADOW_URL, json=body)

        primary_resp, shadow_resp = await asyncio.gather(
            primary_task,
            shadow_task,
            return_exceptions=True,
        )

    # Log shadow results for offline analysis
    if not isinstance(shadow_resp, Exception):
        log_shadow_comparison(
            request_id=body.get("request_id"),
            primary=primary_resp.json(),
            shadow=shadow_resp.json(),
        )

    # ONLY return primary model's response
    return primary_resp.json()
```

### Rollback Strategies

1. **Version pinning** — KServe InferenceService points to a specific model version; rollback = update the version tag
2. **Traffic shifting** — Shift canary weight back to 0%
3. **Automated rollback** — Monitor error rate / latency P99; if threshold breached, auto-revert via Argo Rollouts or Flagger
4. **Model registry status** — Mark model as "Archived" in MLflow registry, which triggers serving layer to reload previous "Production" model

### 💡 Interview Insight

> **Q: "How do you decide between A/B testing and canary deployment for ML models?"**
>
> **A**: They serve different purposes. **Canary** is a *risk mitigation* strategy — you're rolling out a model you believe is better and validating it won't break things (latency, errors, crashes). **A/B testing** is a *measurement* strategy — you're running a controlled experiment to measure whether the new model improves a business metric (CTR, conversion, revenue). In practice, I use canary first (validate safety at 5% traffic), then A/B test (50/50 split for statistical power), then full rollout. Shadow deployment comes before both — to catch obvious regressions offline.

---

## Screen 5: Model Optimization for Production

Training a model and serving a model are two different disciplines. A model that takes 500ms per inference in research is useless in a 50ms latency budget. This screen covers the optimization toolkit.

### GPU Memory Planning — The Math You Must Know

Understanding GPU memory is **non-negotiable** for ML infrastructure roles.

```python
# gpu_memory_calculator.py — Know your numbers
from dataclasses import dataclass
from enum import Enum


class DType(Enum):
    FP32 = 4       # 4 bytes per parameter
    FP16 = 2       # 2 bytes (half precision)
    BF16 = 2       # 2 bytes (bfloat16)
    INT8 = 1       # 1 byte
    INT4 = 0.5     # 0.5 bytes


@dataclass
class GPUMemoryEstimate:
    model_name: str
    params_billions: float
    dtype: DType
    context_length: int = 4096
    batch_size: int = 1
    num_layers: int = 32
    hidden_dim: int = 4096
    num_kv_heads: int = 32

    @property
    def model_memory_gb(self) -> float:
        """Model weights memory."""
        return (
            self.params_billions * 1e9 * self.dtype.value
        ) / (1024 ** 3)

    @property
    def kv_cache_per_token_bytes(self) -> float:
        """KV-cache memory per token per layer.

        Each layer stores K and V vectors:
        2 (K+V) × num_kv_heads × head_dim × dtype_bytes
        """
        head_dim = self.hidden_dim // self.num_kv_heads
        return (
            2 * self.num_kv_heads * head_dim * self.dtype.value
        )

    @property
    def kv_cache_memory_gb(self) -> float:
        """Total KV-cache memory for full context."""
        total_bytes = (
            self.kv_cache_per_token_bytes
            * self.context_length
            * self.num_layers
            * self.batch_size
        )
        return total_bytes / (1024 ** 3)

    @property
    def total_memory_gb(self) -> float:
        """Total = model + KV-cache + ~10% overhead."""
        base = self.model_memory_gb + self.kv_cache_memory_gb
        return base * 1.1  # 10% CUDA overhead

    def recommend_gpus(self) -> str:
        """Recommend GPU configuration."""
        total = self.total_memory_gb
        if total <= 24:
            return "1× RTX 4090 (24GB) or 1× A10G"
        elif total <= 40:
            return "1× A100 40GB"
        elif total <= 80:
            return "1× A100 80GB or 1× H100 80GB"
        elif total <= 160:
            return "2× A100 80GB (tensor parallel)"
        else:
            n_gpus = int(total // 80) + 1
            return f"{n_gpus}× A100 80GB (tensor parallel)"


# Example calculations for interview
models = [
    GPUMemoryEstimate("Llama-3-8B",  8,  DType.FP16),
    GPUMemoryEstimate("Llama-3-8B",  8,  DType.INT4),
    GPUMemoryEstimate("Llama-3-70B", 70, DType.FP16),
    GPUMemoryEstimate("Llama-3-70B", 70, DType.INT4),
    GPUMemoryEstimate("GPT-4 (est)", 1800, DType.FP16),
]

for m in models:
    print(
        f"{m.model_name:20s} | {m.dtype.name:4s} | "
        f"Model: {m.model_memory_gb:7.1f}GB | "
        f"KV$: {m.kv_cache_memory_gb:5.1f}GB | "
        f"Total: {m.total_memory_gb:7.1f}GB | "
        f"→ {m.recommend_gpus()}"
    )
```

**Output (approximate)**:
```
Llama-3-8B           | FP16 | Model:    14.9GB | KV$:   2.0GB | Total:    18.6GB | → 1× RTX 4090 (24GB) or 1× A10G
Llama-3-8B           | INT4 | Model:     3.7GB | KV$:   1.0GB | Total:     5.2GB | → 1× RTX 4090 (24GB) or 1× A10G
Llama-3-70B          | FP16 | Model:   130.4GB | KV$:   2.0GB | Total:   145.6GB | → 2× A100 80GB (tensor parallel)
Llama-3-70B          | INT4 | Model:    32.6GB | KV$:   1.0GB | Total:    37.0GB | → 1× A100 40GB
GPT-4 (est)          | FP16 | Model:  3355.0GB | KV$:   2.0GB | Total:  3692.7GB | → 47× A100 80GB (tensor parallel)
```

### Quantization Deep Dive

Quantization reduces model weights from high-precision floats to lower-precision integers:

```
FP32 (32 bits) ──▶ FP16/BF16 (16 bits) ──▶ INT8 (8 bits) ──▶ INT4 (4 bits)
   Full              Half precision          4× smaller         8× smaller
   precision         ~0% quality loss        ~1% quality loss   ~2-5% loss
```

| Technique | Precision | Size Reduction | Quality Impact | Best For             |
|-----------|-----------|----------------|----------------|----------------------|
| FP16/BF16 | 16-bit   | 2×             | Negligible     | Default training     |
| GPTQ      | INT4/INT8 | 4-8×           | Small          | LLM inference        |
| AWQ       | INT4      | 8×             | Very small     | LLM inference (best) |
| GGUF      | INT4-INT8 | 4-8×           | Small          | CPU / edge / llama.cpp |
| FP8       | 8-bit     | 4×             | Minimal        | H100 native support  |

### Other Optimization Techniques

**Pruning** — Remove weights close to zero. Unstructured pruning removes individual weights (sparse matrices). Structured pruning removes entire neurons/heads (smaller dense model).

**Knowledge Distillation** — Train a small "student" model to mimic a large "teacher" model. The student learns from the teacher's soft probability distributions, which contain more information than hard labels.

```
Teacher (70B)                Student (7B)
    │                            ▲
    ▼                            │
Soft predictions ───────────────┘
   (logits)         Loss = KL-divergence
                    between teacher & student
                    distributions
```

**ONNX (Open Neural Network Exchange)** — Convert models from any framework (PyTorch, TensorFlow, JAX) to a framework-agnostic format. ONNX Runtime provides optimized inference across CPUs, GPUs, and edge devices.

```python
# Export PyTorch model to ONNX
import torch

model = load_trained_model()
dummy_input = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    input_names=["image"],
    output_names=["prediction"],
    dynamic_axes={
        "image": {0: "batch_size"},
        "prediction": {0: "batch_size"},
    },
    opset_version=17,
)
```

### Multi-GPU Strategies

```
TENSOR PARALLELISM                PIPELINE PARALLELISM
(split layers across GPUs)       (split model stages across GPUs)

  GPU 0        GPU 1               GPU 0      GPU 1      GPU 2
┌────────┐  ┌────────┐          ┌────────┐ ┌────────┐ ┌────────┐
│Layer 1 │  │Layer 1 │          │Layers  │ │Layers  │ │Layers  │
│(left   │  │(right  │          │ 1-10   │ │ 11-20  │ │ 21-32  │
│ half)  │  │ half)  │          │        │ │        │ │        │
│Layer 2 │  │Layer 2 │          └───┬────┘ └───┬────┘ └───┬────┘
│(left)  │  │(right) │              │          │          │
│  ...   │  │  ...   │              ▼          ▼          ▼
└────────┘  └────────┘          Micro-batches flow through pipeline

Best for: latency              Best for: throughput
(all layers computed in         (pipeline keeps all GPUs busy
 parallel per token)             with different micro-batches)
```

### 💡 Interview Insight

> **Q: "A 7B parameter model in FP16 — how much GPU memory does it need? Can you run it on a single RTX 4090?"**
>
> **A**: Model weights alone: 7B × 2 bytes (FP16) = 14GB. Add KV-cache for inference (~1-2GB at 4K context), CUDA overhead (~10%), and you're at roughly **17-18GB**. An RTX 4090 has 24GB VRAM, so yes — it fits comfortably. For batch inference with larger batches, KV-cache grows linearly with batch size, so you may hit limits at batch size 8+. For tighter fit, INT4 quantization (AWQ/GPTQ) drops weights to ~3.5GB, leaving 20GB for KV-cache and large batches.

---

## Screen 6: Scaling & Production Monitoring

A model in production without monitoring is a car without a dashboard — you're driving blind. This screen covers how to scale serving infrastructure and keep it healthy.

### Scaling Strategies

```
┌──────────────────────────────────────────────────────┐
│             SCALING MODEL SERVING                    │
│                                                      │
│  HORIZONTAL SCALING          VERTICAL SCALING        │
│  ┌─────┐┌─────┐┌─────┐     ┌────────────────┐      │
│  │Pod 1││Pod 2││Pod 3│     │  Bigger GPU    │      │
│  │(GPU)││(GPU)││(GPU)│     │  A10G → A100   │      │
│  └─────┘└─────┘└─────┘     │  → H100        │      │
│  Load balancer distributes  └────────────────┘      │
│                                                      │
│  REQUEST BATCHING            GPU SHARING             │
│  ┌─────────────────┐        ┌────────────────┐      │
│  │ req1 ┐          │        │ Model A  50%   │      │
│  │ req2 ├─▶ batch  │        │ Model B  30%   │      │
│  │ req3 ┘   infer  │        │ Model C  20%   │      │
│  │ (amortize GPU   │        │ (MIG / MPS /   │      │
│  │  kernel launch) │        │  time-sharing) │      │
│  └─────────────────┘        └────────────────┘      │
└──────────────────────────────────────────────────────┘
```

### Key Monitoring Metrics

Every ML serving system must track these metrics — interviewers will ask about them:

**Latency Metrics**:
- **P50 (median)**: Typical user experience
- **P95**: What 95% of users experience
- **P99**: Worst-case tail latency (critical for SLOs)
- **TTFT** (Time to First Token): LLM-specific, measures perceived responsiveness

**Throughput Metrics**:
- **RPS / QPS**: Requests per second
- **Tokens/second**: LLM throughput metric
- **GPU utilization**: Should be 60-80% (not 100% — need headroom for spikes)

**Reliability Metrics**:
- **Error rate**: 5xx responses / total requests
- **Availability**: Uptime percentage (99.9% = 8.7 hours downtime/year)
- **Request queue depth**: Signals under-provisioning

```python
# monitoring.py — Prometheus metrics for model serving
from prometheus_client import (
    Counter,
    Histogram,
    Gauge,
    start_http_server,
)

# Latency histogram with custom buckets for ML
INFERENCE_LATENCY = Histogram(
    "model_inference_duration_seconds",
    "Time spent on model inference",
    labelnames=["model_name", "model_version"],
    buckets=[
        0.005, 0.01, 0.025, 0.05, 0.075,
        0.1, 0.25, 0.5, 0.75, 1.0, 2.5,
    ],
)

PREDICTION_COUNTER = Counter(
    "model_predictions_total",
    "Total predictions made",
    labelnames=["model_name", "model_version", "status"],
)

GPU_MEMORY_USED = Gauge(
    "gpu_memory_used_bytes",
    "GPU memory currently in use",
    labelnames=["gpu_id"],
)

PREDICTION_VALUE = Histogram(
    "model_prediction_value",
    "Distribution of prediction values (drift detection)",
    labelnames=["model_name"],
    buckets=[0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
)


# Usage in serving code
import time

def serve_prediction(model, request):
    start = time.perf_counter()
    try:
        result = model.predict(request.features)
        PREDICTION_COUNTER.labels(
            model_name="fraud-v3",
            model_version="3.1.0",
            status="success",
        ).inc()
        PREDICTION_VALUE.labels(
            model_name="fraud-v3"
        ).observe(result.score)
        return result
    except Exception as e:
        PREDICTION_COUNTER.labels(
            model_name="fraud-v3",
            model_version="3.1.0",
            status="error",
        ).inc()
        raise
    finally:
        duration = time.perf_counter() - start
        INFERENCE_LATENCY.labels(
            model_name="fraud-v3",
            model_version="3.1.0",
        ).observe(duration)
```

### Model Drift Detection

Models degrade silently. The data they were trained on stops reflecting reality.

```
┌──────────────────────────────────────────────────────┐
│               TYPES OF DRIFT                         │
│                                                      │
│  DATA DRIFT              CONCEPT DRIFT               │
│  Input distribution      Relationship between        │
│  changes                 input → output changes      │
│                                                      │
│  Example: User age       Example: COVID changed      │
│  distribution shifted    buying patterns, but age    │
│  (new market segment)    distributions stayed same   │
│                                                      │
│  Detection: Compare      Detection: Monitor model    │
│  feature distributions   performance on labeled      │
│  (KS test, PSI, JS      samples (accuracy, F1       │
│  divergence)             over time)                  │
│                                                      │
│  PREDICTION DRIFT                                    │
│  Model output            Proxy when you don't have   │
│  distribution changes    ground truth labels yet     │
│                                                      │
│  Detection: Track        Often the first signal      │
│  prediction histograms   that something is wrong     │
│  over time windows                                   │
└──────────────────────────────────────────────────────┘
```

### SLOs for ML Services

| SLO Metric        | Target (Typical)    | Alert Threshold          |
|--------------------|---------------------|--------------------------|
| Availability       | 99.9%               | <99.5% over 5 min       |
| P50 latency        | <50ms               | >100ms over 5 min       |
| P99 latency        | <200ms              | >500ms over 1 min       |
| Error rate          | <0.1%               | >1% over 5 min          |
| Prediction drift    | PSI < 0.1           | PSI > 0.2               |
| GPU utilization     | 60-80%              | >90% sustained 10 min   |

### 💡 Interview Insight

> **Q: "Your model's accuracy dropped 5% over the past month, but no code changes were made. What's your debugging process?"**
>
> **A**: I follow a systematic approach:
> 1. **Check data drift first** — Compare recent input feature distributions against the training set using PSI (Population Stability Index). If PSI > 0.2 for key features, data drift is the likely cause.
> 2. **Inspect prediction distribution** — Plot prediction histograms for recent vs. historical. A shift suggests the model is responding to changed inputs.
> 3. **Check for upstream data issues** — Did a data pipeline break? Are features arriving as nulls? Feature store staleness?
> 4. **Evaluate on recent labeled data** — If you have ground truth, compute metrics on a recent holdout. Compare per-segment.
> 5. **Remediation** — If data drift: retrain on recent data. If concept drift: may need model architecture changes or new features. Set up automated drift monitoring to catch this earlier next time.

---

## Screen 7: End-to-End Architecture & Interview Scenarios

Let's tie everything together. In system design interviews, you need to draw the full serving architecture and defend your choices.

### Production Model Serving Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                 PRODUCTION ML SERVING SYSTEM                        │
│                                                                     │
│  ┌─────────┐    ┌──────────┐    ┌────────────────────────────────┐ │
│  │ Client  │───▶│ API      │───▶│      Kubernetes Cluster       │ │
│  │ (App /  │    │ Gateway  │    │                                │ │
│  │  Web)   │    │ (Kong /  │    │  ┌──────────┐  ┌───────────┐  │ │
│  └─────────┘    │ Envoy)   │    │  │ KServe   │  │ Feature   │  │ │
│                 └──────────┘    │  │ Router   │  │ Store     │  │ │
│                      │         │  │ (Istio)  │  │ (Feast)   │  │ │
│                      │         │  └────┬─────┘  └─────┬─────┘  │ │
│                      │         │       │              │         │ │
│                      │         │  ┌────▼──────────────▼──────┐  │ │
│                      │         │  │   Transformer Pod        │  │ │
│                      │         │  │   (feature enrichment)   │  │ │
│                      │         │  └────────────┬─────────────┘  │ │
│                      │         │               │                │ │
│                      │         │    ┌──────────▼──────────┐     │ │
│                      │         │    │  Predictor Pods      │     │ │
│                      │         │    │  ┌──────┐ ┌──────┐  │     │ │
│                      │         │    │  │ v3   │ │ v4   │  │     │ │
│                      │         │    │  │(95%) │ │(5%)  │  │     │ │
│                      │         │    │  │      │ │canary│  │     │ │
│                      │         │    │  └──────┘ └──────┘  │     │ │
│                      │         │    └─────────────────────┘     │ │
│                      │         │                                │ │
│                      │         └────────────────────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    OBSERVABILITY STACK                       │   │
│  │  ┌──────────┐  ┌──────────────┐  ┌───────────────────────┐ │   │
│  │  │Prometheus│  │   Grafana    │  │  Drift Detector       │ │   │
│  │  │ (metrics)│─▶│ (dashboards) │  │  (Evidently /         │ │   │
│  │  └──────────┘  └──────────────┘  │   custom PSI checks)  │ │   │
│  │                                   └───────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    MODEL LIFECYCLE                           │   │
│  │  ┌──────────┐  ┌──────────────┐  ┌───────────────────────┐ │   │
│  │  │ MLflow   │  │ CI/CD        │  │  Model Registry       │ │   │
│  │  │ (track)  │─▶│ (validate +  │─▶│  (version, promote,   │ │   │
│  │  └──────────┘  │  deploy)     │  │   rollback)           │ │   │
│  │                └──────────────┘  └───────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Interview Scenario: Design a Fraud Detection System

> **Q: "Design a real-time fraud detection system that scores credit card transactions with <100ms latency at 10,000 TPS."**

**Solution walkthrough**:

**1. Serving Pattern**: Real-time API. Cannot be batch (need instant decision). Cannot be embedded (model updates frequently, needs centralized features).

**2. Architecture**:
- **API Gateway** → rate limiting, auth, request validation
- **Feature enrichment** → join real-time transaction data with precomputed features from Feature Store (user spending patterns, merchant risk scores)
- **Model serving** → ONNX Runtime on GPU for low latency; alternatively Triton with dynamic batching
- **Response** → fraud score + decision (approve/decline/review)

**3. Scaling**:
- 10K TPS with dynamic batching (batch of 32 per GPU call) = ~312 inference calls/sec
- P99 latency budget: 30ms for feature enrichment, 50ms for inference, 20ms overhead
- Need ~4 GPU pods (each handling ~100 batched calls/sec with headroom)
- HPA on request queue depth, minReplicas=4

**4. Deployment**:
- Canary release (5% → 25% → 100%)
- Shadow deployment first to compare false positive rates
- Automated rollback if error rate > 0.5% or P99 > 200ms

**5. Monitoring**:
- Prediction distribution (are we flagging more/fewer transactions?)
- Feature drift (spending patterns shifting?)
- False positive rate from manual review feedback loop
- Latency P50/P95/P99 with SLO alerting

### Interview Scenario: Serve an LLM Chat Application

> **Q: "Design the serving infrastructure for an internal chatbot powered by a 70B parameter LLM."**

**Solution walkthrough**:

**1. Model optimization**: 70B in FP16 = 140GB → INT4 AWQ quantization → ~35GB. Use 2× A100 80GB with tensor parallelism for headroom and fast generation.

**2. Framework**: vLLM for continuous batching and PagedAttention. OpenAI-compatible API makes client integration simple.

```bash
# Production vLLM deployment
python -m vllm.entrypoints.openai.api_server \
    --model /models/llama-3-70b-awq \
    --quantization awq \
    --tensor-parallel-size 2 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.85 \
    --max-num-seqs 64 \
    --enable-prefix-caching \
    --disable-log-requests
```

**3. Scaling**: Token generation is sequential per request. Throughput scales horizontally (more pod replicas). Each pod handles ~20-30 concurrent requests. For 200 concurrent users → 7-10 pods.

**4. Caching**: Enable prefix caching for shared system prompts. Add a Redis semantic cache layer — if a semantically similar question was asked recently, return the cached response.

**5. Guardrails**: Input validation (PII detection, prompt injection filtering), output filtering (toxicity, hallucination detection), rate limiting per user.

### 💡 Interview Insight

> **Q: "What's the difference between tensor parallelism and pipeline parallelism? When do you use each?"**
>
> **A**: **Tensor parallelism** splits individual layers across GPUs — each GPU computes a slice of every layer. This reduces per-token latency because all GPUs work on the same token simultaneously. Best for latency-sensitive serving. **Pipeline parallelism** assigns different layers to different GPUs and processes micro-batches in a pipelined fashion. This maximizes throughput but adds latency (tokens must traverse all GPUs sequentially). For LLM serving, I prefer tensor parallelism (latency matters). For training, pipeline parallelism shines with large batches.

---

## Screen 8: Quiz & Key Takeaways

### Quiz: Model Serving & Deployment

**Question 1**: A model needs to score 50 million customer records nightly for a recommendation email. Which serving pattern is most appropriate?

- A) Real-time API with auto-scaling
- B) Streaming inference via Kafka
- C) Batch inference with Spark ✅
- D) Embedded model in the email client

> **Explanation**: 50M records on a schedule is textbook batch inference. Spark distributes the workload across a cluster, writes predictions to storage, and the email service reads them. No need for real-time latency.

---

**Question 2**: A 13B parameter model in INT4 quantization requires approximately how much GPU memory for weights alone?

- A) ~3.25 GB
- B) ~6.5 GB ✅
- C) ~13 GB
- D) ~26 GB

> **Explanation**: 13B parameters × 0.5 bytes (INT4) = 6.5 GB. This fits on a single RTX 4090 (24GB) with plenty of room for KV-cache and overhead.

---

**Question 3**: Which deployment strategy runs the new model on production traffic but discards its responses?

- A) Canary deployment
- B) Blue-green deployment
- C) Shadow deployment ✅
- D) A/B testing

> **Explanation**: Shadow deployment mirrors real traffic to the new model for comparison purposes, but only the existing model's responses are served to users. This is the safest way to validate a model in production.

---

**Question 4**: What is the primary advantage of vLLM's PagedAttention over traditional KV-cache management?

- A) It uses less GPU compute for attention
- B) It eliminates the need for KV-cache entirely
- C) It reduces memory fragmentation/waste ✅
- D) It enables larger model sizes on the same GPU

> **Explanation**: Traditional KV-cache allocates contiguous memory blocks per sequence, leading to fragmentation and waste (especially with variable-length sequences). PagedAttention manages KV-cache in non-contiguous pages (like OS virtual memory), reducing waste from ~60-80% to near zero and enabling 2-4× higher throughput.

---

**Question 5**: Your model's P99 latency suddenly spikes from 100ms to 800ms, but P50 remains at 40ms. What is the most likely cause?

- A) The model weights are corrupted
- B) GPU thermal throttling under load
- C) A few requests hitting cold cache or GC pauses ✅
- D) The model has experienced concept drift

> **Explanation**: When P50 is stable but P99 spikes, it indicates tail latency issues — not systemic degradation. Common causes: garbage collection pauses in the Python/Java serving layer, cold KV-cache on first requests after idle, request batching causing some requests to wait in queue, or intermittent GPU memory contention. Concept drift would affect prediction quality, not latency.

---

### Key Takeaways

```
┌──────────────────────────────────────────────────────────┐
│          MODEL SERVING — WHAT YOU MUST REMEMBER          │
│                                                          │
│  1. PATTERN SELECTION is architecture — batch vs         │
│     real-time vs streaming vs embedded. Wrong choice     │
│     = wrong system. Most production systems are hybrid.  │
│                                                          │
│  2. MEMORY MATH is non-negotiable:                       │
│     GPU memory ≈ params × dtype_bytes + KV-cache         │
│     7B × 2 bytes = 14GB FP16. Know this cold.           │
│                                                          │
│  3. vLLM + PagedAttention is the current standard for   │
│     LLM serving. Continuous batching > static batching.  │
│                                                          │
│  4. QUANTIZATION is your best friend:                    │
│     AWQ INT4 gives 8× size reduction with minimal       │
│     quality loss. Always quantize for inference.         │
│                                                          │
│  5. KUBERNETES + KSERVE is the production deployment     │
│     stack. InferenceService CRD, autoscaling, canary     │
│     traffic splits, transformer components.              │
│                                                          │
│  6. CANARY BEFORE A/B — Validate safety (canary at 5%), │
│     then measure impact (A/B at 50/50), then roll out.   │
│                                                          │
│  7. MONITOR EVERYTHING — P50/P95/P99 latency,           │
│     throughput, error rates, prediction distributions,   │
│     feature drift. Set SLOs and alert on breaches.       │
│                                                          │
│  8. COLD STARTS KILL LATENCY — minReplicas ≥ 1,         │
│     model warmup at startup, readiness probes.           │
│                                                          │
│  9. MULTI-GPU STRATEGY matters:                          │
│     Tensor parallel = lower latency (serving)            │
│     Pipeline parallel = higher throughput (training)     │
│                                                          │
│  10. SHADOW → CANARY → A/B → FULL ROLLOUT               │
│      is the safest deployment progression.               │
└──────────────────────────────────────────────────────────┘
```

### Quick Reference Card

| Topic                 | Key Numbers / Facts            |
|-----------------------|--------------------------------|
| FP16 memory           | params × 2 bytes               |
| INT4 memory           | params × 0.5 bytes             |
| A100 VRAM             | 40GB or 80GB                   |
| H100 VRAM             | 80GB                           |
| 99.9% availability    | ~8.7 hours downtime/year       |
| 99.99% availability   | ~52 minutes downtime/year      |
| Canary start          | 1-5% traffic                   |
| PSI drift threshold   | >0.2 = significant drift       |
| vLLM advantage        | PagedAttention, cont. batching |
| Triton advantage      | Multi-framework, GPU-optimized |

### 💡 Interview Insight (Final)

> **Parting wisdom**: In ML system design interviews, **don't jump to the model**. Start with requirements (latency, throughput, availability), choose the serving pattern, design the infrastructure, then discuss the model. Interviewers care more about your ability to design a reliable, scalable serving system than your knowledge of transformer architectures. Always mention monitoring, rollback strategies, and cost trade-offs — these separate senior candidates from juniors.
