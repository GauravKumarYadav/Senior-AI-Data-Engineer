# Containers & Deployment

Welcome to Module 3 — the module that bridges the gap between "it works on my machine" and "it works in production." If you're preparing for backend engineering interviews, containers and deployment are non-negotiable topics. Almost every modern team ships code inside containers, orchestrates them with Kubernetes, and automates the whole pipeline with CI/CD. This guide covers all of it, from your first Dockerfile to Helm charts and monitoring stacks.

Let's dive in.

---

## Screen 1 — Docker Fundamentals

### What Are Containers?

A container is a lightweight, standalone, executable package that includes everything needed to run a piece of software: the code, the runtime, system libraries, and settings. Containers run on top of a shared OS kernel, which makes them dramatically faster and lighter than virtual machines.

Think of it this way: a **virtual machine** virtualizes the *hardware*. Each VM runs its own full operating system — its own kernel, its own memory management, its own everything. A **container** virtualizes the *operating system*. Containers share the host's kernel and isolate the application processes using Linux namespaces and cgroups.

| Feature | Containers | Virtual Machines |
|---|---|---|
| Boot time | Milliseconds | Minutes |
| Size | Megabytes | Gigabytes |
| Isolation | Process-level (shared kernel) | Full OS isolation |
| Overhead | Minimal | Significant |
| Density | Hundreds per host | Tens per host |
| Portability | Excellent (OCI standard) | Good (hypervisor-dependent) |

> 💡 **Interview Gold**: When asked "containers vs VMs," don't just list differences. Say: *"Containers share the host kernel, so they're faster to start and use fewer resources. VMs provide stronger isolation because each has its own kernel. In practice, most teams use containers for application workloads and VMs for the underlying infrastructure — your Kubernetes nodes are usually VMs running containers."*

### The Dockerfile

A Dockerfile is a declarative recipe for building a container image. Each instruction creates a **layer** in the image filesystem. Understanding layers is crucial for writing efficient Dockerfiles.

#### Layer Caching Strategy — Order Matters!

Docker caches each layer. When you rebuild an image, Docker reuses cached layers until it hits a layer where something changed. After that, every subsequent layer is rebuilt. This means you should order your Dockerfile instructions from **least frequently changed** to **most frequently changed**.

```dockerfile
# ❌ BAD — copying all files before installing dependencies
# Every code change invalidates the pip install cache
COPY . /app
RUN pip install -r requirements.txt

# ✅ GOOD — copy requirements first, then install, then copy code
# Dependencies are cached unless requirements.txt changes
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
```

> 🎯 **Aha!** The reason this matters so much is that `pip install` can take minutes for large projects. By copying `requirements.txt` separately, you only re-run the install step when your dependencies actually change — not every time you tweak a line of application code.

#### Multi-Stage Builds for Smaller Python Images

Multi-stage builds let you use one image for building (with compilers, dev tools, etc.) and a separate, minimal image for running. The final image only contains what you need at runtime.

Here's a complete, production-ready multi-stage Dockerfile for a FastAPI application:

```dockerfile
# =============================================================
# Stage 1: Builder — install dependencies in a full Python image
# =============================================================
FROM python:3.12-slim AS builder

# Prevent Python from writing .pyc files and enable unbuffered output
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies needed for building Python packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies into a virtual env
COPY requirements.txt .
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# =============================================================
# Stage 2: Runtime — slim image with only the venv and app code
# =============================================================
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install only runtime system libraries (no compiler!)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Copy the virtual environment from the builder stage
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code last (changes most often)
COPY . .

# Create a non-root user for security
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8000

# Health check so orchestrators know we're alive
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why this works well:**

- **Stage 1** has `gcc` and build tools — needed to compile C extensions (like `psycopg2`). These are *not* in the final image.
- **Stage 2** only has the compiled virtual environment, runtime libs, and app code. The final image can be 60–80% smaller.
- We run as a **non-root user** — a critical security best practice.
- The `HEALTHCHECK` instruction lets Docker (and orchestrators) verify the container is actually serving traffic.

#### Optimizing Base Images: slim vs alpine

| Base Image | Size | Pros | Cons |
|---|---|---|---|
| `python:3.12` | ~900 MB | Everything included | Huge |
| `python:3.12-slim` | ~150 MB | Good balance | Missing some libs |
| `python:3.12-alpine` | ~50 MB | Tiny | Uses musl libc, breaks many wheels |

> 💡 **Interview Gold**: *"I prefer `slim` over `alpine` for Python. Alpine uses musl libc instead of glibc, which means many Python packages can't use pre-built wheels and must compile from source — making builds slower and sometimes introducing subtle bugs. The size savings of alpine rarely justify the headaches."*

#### .dockerignore — What to Exclude

Just like `.gitignore`, a `.dockerignore` file prevents unnecessary files from being sent to the Docker daemon during builds. This speeds up builds and reduces image size.

```
# .dockerignore
__pycache__
*.pyc
*.pyo
.git
.github
.gitignore
.env
.env.*
*.md
docs/
tests/
.pytest_cache
.mypy_cache
.ruff_cache
.venv
venv
*.egg-info
dist/
build/
docker-compose*.yml
Dockerfile
.dockerignore
*.log
.coverage
htmlcov/
```

> 🎯 **Aha!** Forgetting `.dockerignore` is a classic mistake. Without it, Docker sends your entire build context (including `.git`, `node_modules`, test data, etc.) to the daemon. For large repos, this can add minutes to your build time — and potentially leak secrets into your image.

---

## Screen 2 — Docker Compose & Local Development

### Why Docker Compose?

In the real world, your application doesn't run in isolation. It talks to a database, a cache, a message queue, maybe a search engine. Docker Compose lets you define and run **multi-container applications** with a single YAML file and a single command: `docker compose up`.

### Complete docker-compose.yml Example

Here's a production-like local development setup with a FastAPI app, PostgreSQL, and Redis:

```yaml
# docker-compose.yml
version: "3.9"

services:
  # ── FastAPI Application ──────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime          # use the runtime stage from multi-stage build
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app/app         # bind mount for hot-reload during dev
    environment:
      - DATABASE_URL=postgresql+asyncpg://myuser:mypass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=debug
    env_file:
      - .env                   # additional secrets from .env file
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped
    networks:
      - backend

  # ── PostgreSQL Database ──────────────────────────────────
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data        # named volume
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # seed data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

  # ── Redis Cache ──────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    command: redis-server --appendonly yes
    restart: unless-stopped
    networks:
      - backend

volumes:
  postgres_data:               # named volume — survives container restarts
  redis_data:

networks:
  backend:
    driver: bridge
```

### Volume Mounts: Bind Mounts vs Named Volumes

This is a frequent interview question. Know the difference cold.

**Bind mounts** map a host directory directly into the container:

```yaml
volumes:
  - ./app:/app/app    # host path : container path
```

- You see changes instantly (great for development with hot-reload).
- The host filesystem "wins" — if the host directory is empty, the container sees empty.
- Performance can be slow on macOS due to filesystem translation.

**Named volumes** are managed by Docker:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

- Docker manages the storage location — you don't need to know where.
- Data persists even when containers are removed.
- Better performance than bind mounts (especially on macOS).
- Ideal for database storage and other persistent data.

> 💡 **Interview Gold**: *"I use bind mounts for application code during development — so I get hot-reload. For databases and caches, I use named volumes because they're managed by Docker and persist across container restarts. In production, I'd use neither — I'd use managed services like RDS or cloud-native volumes."*

### Networking in Compose

Docker Compose automatically creates a network for your services. Each service can reach other services **by container name** as the hostname. In the example above, the app connects to `db:5432` and `redis:6379` — Docker's internal DNS resolves those names.

```python
# In your FastAPI app, the database URL uses the service name:
DATABASE_URL = "postgresql+asyncpg://myuser:mypass@db:5432/mydb"
#                                                    ^^
#                                        service name, not localhost!
```

### Environment Variables and .env Files

There are three ways to pass environment variables in Compose:

1. **Inline in `docker-compose.yml`** — good for non-sensitive defaults.
2. **`env_file` directive** — loads from a `.env` file. Never commit this file!
3. **Shell environment** — variables from your shell override compose file values.

```bash
# .env (NEVER commit this file to git)
SECRET_KEY=super-secret-key-change-in-production
JWT_ALGORITHM=HS256
SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
```

### Health Checks in Compose

Health checks tell Docker whether a container is actually *working*, not just *running*. The `depends_on` with `condition: service_healthy` ensures your app doesn't start until the database is actually accepting connections — not just when the container exists.

Without health checks, your app might crash on startup because it tries to connect to a database that hasn't finished initializing yet. Race conditions like this are the most common source of "works sometimes" bugs in containerized environments.

> 🎯 **Aha!** `depends_on` without a health check condition only waits for the container to *start*, not for the service inside it to be *ready*. This subtle difference causes real-world outages. Always pair `depends_on` with health checks.

---

## Screen 3 — Kubernetes Basics

### Why Kubernetes?

Docker Compose is great for local development and small deployments. But what happens when you need to run hundreds of containers across dozens of machines, auto-scale based on traffic, do zero-downtime deployments, and self-heal when containers crash? That's Kubernetes (K8s).

### K8s Architecture

Kubernetes follows a **control plane + worker node** architecture:

**Control Plane** (the brain):
- **API Server (`kube-apiserver`)** — the front door. Every interaction with K8s (kubectl, dashboards, CI/CD) goes through the API server.
- **etcd** — a distributed key-value store that holds *all* cluster state. Think of it as the cluster's database.
- **Scheduler (`kube-scheduler`)** — decides which node should run a new Pod based on resource requirements, affinity rules, and constraints.
- **Controller Manager** — runs control loops that watch the cluster state and work to make actual state match desired state (e.g., "I want 3 replicas" → ensure 3 are running).

**Worker Nodes** (the muscle):
- **kubelet** — an agent on each node that ensures containers described in PodSpecs are running and healthy.
- **kube-proxy** — handles network rules so Pods can communicate with each other and the outside world.
- **Container Runtime** — actually runs the containers (containerd, CRI-O).

> 💡 **Interview Gold**: *"Kubernetes is a declarative system. You tell it WHAT you want (3 replicas of my app), and the controllers figure out HOW to make it happen. If a Pod dies, the controller notices the actual state doesn't match the desired state and creates a new Pod. This reconciliation loop is the heart of K8s."*

### Core Objects

#### Pods

A Pod is the smallest deployable unit in K8s — a wrapper around one or more containers that share networking and storage. In practice, most Pods contain a single application container. Multi-container Pods are used for sidecar patterns (logging agents, service mesh proxies).

#### Deployments

A Deployment manages the lifecycle of Pods. It handles:
- **Replica count** — how many instances to run.
- **Rolling updates** — gradually replacing old Pods with new ones (zero downtime).
- **Rollbacks** — reverting to a previous version if something goes wrong.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-app
  namespace: production
  labels:
    app: fastapi-app
    version: v1.2.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # allow 1 extra Pod during rollout
      maxUnavailable: 0      # never have fewer than 3 running
  template:
    metadata:
      labels:
        app: fastapi-app
        version: v1.2.0
    spec:
      containers:
        - name: fastapi-app
          image: my-registry.example.com/fastapi-app:v1.2.0
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "100m"       # guaranteed minimum
              memory: "128Mi"
            limits:
              cpu: "500m"       # maximum allowed
              memory: "512Mi"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: log-level
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 20
```

#### Services

A Service provides a stable network endpoint for a set of Pods. Pods come and go (they get new IPs each time), but a Service gives you a consistent DNS name and IP.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
  namespace: production
spec:
  type: ClusterIP              # internal only (default)
  selector:
    app: fastapi-app           # routes to Pods with this label
  ports:
    - port: 80                 # port the Service listens on
      targetPort: 8000         # port on the Pod
      protocol: TCP
```

**Service types:**
- **ClusterIP** (default) — internal only. Other services in the cluster can reach it.
- **NodePort** — exposes the service on a static port on every node. Rarely used in production.
- **LoadBalancer** — provisions a cloud load balancer (AWS ELB, GCP LB). The standard way to expose services externally.

#### Ingress

An Ingress is a layer 7 (HTTP) routing rule that maps external URLs to internal Services. You need an **Ingress Controller** (like NGINX Ingress or Traefik) running in the cluster to make Ingress resources work.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fastapi-service
                port:
                  number: 80
```

### ConfigMaps and Secrets

**ConfigMaps** store non-sensitive configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  log-level: "info"
  max-connections: "100"
  feature-flag-dark-mode: "true"
```

**Secrets** store sensitive data (base64-encoded, *not* encrypted by default):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYjo1NDMyL215ZGI=
  api-key: c3VwZXItc2VjcmV0LWtleQ==
```

> 🎯 **Aha!** Kubernetes Secrets are base64-encoded, **not encrypted**. Anyone with access to the namespace can decode them. For real security, use a secrets manager like HashiCorp Vault, AWS Secrets Manager, or enable K8s encryption at rest with a KMS provider. Interviewers love to test whether you know this distinction.

### Namespaces

Namespaces provide logical isolation within a cluster. Common patterns:

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   30d
# kube-system       Active   30d    ← K8s internal components
# production        Active   25d
# staging           Active   25d
# development       Active   20d
# monitoring        Active   20d    ← Prometheus, Grafana
```

Namespaces let you apply resource quotas, network policies, and RBAC rules per environment.

---

## Screen 4 — K8s Advanced & Helm

### Health Checks: Readiness vs Liveness Probes

This is one of the most commonly asked Kubernetes interview questions. Get this right.

**Liveness Probe** — "Is this container alive?"
- K8s checks periodically. If the probe **fails**, the container is **killed and restarted**.
- Purpose: recover from deadlocks, infinite loops, or corrupted state.
- Failure action: **restart the container**.

**Readiness Probe** — "Is this container ready to receive traffic?"
- K8s checks periodically. If the probe **fails**, the Pod is **removed from the Service's endpoints** (no traffic routed to it).
- The container is NOT restarted — it stays running.
- Purpose: handle warm-up time, temporary overload, or dependency unavailability.
- Failure action: **stop sending traffic**.

```yaml
# In a Pod spec:
readinessProbe:
  httpGet:
    path: /health/ready       # check DB connection, cache, etc.
    port: 8000
  initialDelaySeconds: 5      # wait 5s before first check
  periodSeconds: 10           # check every 10s
  failureThreshold: 3         # 3 failures = not ready

livenessProbe:
  httpGet:
    path: /health/alive        # lightweight — is the process running?
    port: 8000
  initialDelaySeconds: 15     # give the app time to start
  periodSeconds: 20           # check every 20s
  failureThreshold: 3         # 3 failures = restart container
```

There's also a **Startup Probe** — used for slow-starting containers. It disables liveness/readiness checks until the startup probe succeeds. This prevents K8s from killing a container that's still initializing.

> 💡 **Interview Gold**: *"The key insight is what happens on failure. Liveness failure restarts the container — use it for unrecoverable states. Readiness failure removes the Pod from traffic — use it for temporary issues like database reconnection. A common mistake is using an aggressive liveness probe that kills healthy containers during temporary load spikes."*

### Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of Pod replicas based on observed metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # wait 60s before scaling up again
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5min before scaling down (avoid flapping)
```

HPA requires the **Metrics Server** to be installed in the cluster. For custom metrics (requests per second, queue depth), you'd use a custom metrics adapter like Prometheus Adapter.

> 🎯 **Aha!** HPA scales based on the *ratio* of current metric value to target. If you set target CPU at 70% and current average is 140%, K8s roughly doubles the replicas. The `stabilizationWindowSeconds` on scale-down prevents "flapping" — where the autoscaler keeps scaling up and down rapidly.

### Jobs and CronJobs

**Jobs** run a task to completion (not a long-running server):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3             # retry up to 3 times on failure
  activeDeadlineSeconds: 600  # kill if running longer than 10 minutes
  template:
    spec:
      containers:
        - name: migrate
          image: my-registry.example.com/fastapi-app:v1.2.0
          command: ["alembic", "upgrade", "head"]
      restartPolicy: Never
```

**CronJobs** run Jobs on a schedule:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"       # 2:00 AM every day
  concurrencyPolicy: Forbid   # don't start a new run if previous is still going
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report-generator
              image: my-registry.example.com/report-gen:latest
              command: ["python", "generate_report.py"]
          restartPolicy: OnFailure
```

### Helm Charts

Helm is the **package manager for Kubernetes**. Instead of managing dozens of raw YAML files, you create a **chart** — a templated, versioned, reusable package.

**Chart structure:**

```
my-fastapi-chart/
├── Chart.yaml              # metadata: name, version, description
├── values.yaml             # default configuration values
├── charts/                 # sub-chart dependencies
└── templates/
    ├── deployment.yaml     # templated K8s manifest
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    ├── _helpers.tpl        # reusable template snippets
    └── NOTES.txt           # post-install instructions
```

**values.yaml example:**

```yaml
# values.yaml — override these per environment
replicaCount: 3

image:
  repository: my-registry.example.com/fastapi-app
  tag: "v1.2.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8000

ingress:
  enabled: true
  host: api.example.com
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  LOG_LEVEL: info
  WORKERS: "4"

secrets:
  databaseUrl: ""            # set via --set or separate values file
  apiKey: ""

probes:
  readiness:
    path: /health/ready
    initialDelaySeconds: 5
  liveness:
    path: /health/alive
    initialDelaySeconds: 15
```

**Using Helm:**

```bash
# Install a chart
helm install my-app ./my-fastapi-chart -f values-production.yaml

# Upgrade with new values
helm upgrade my-app ./my-fastapi-chart --set image.tag=v1.3.0

# Rollback to a previous release
helm rollback my-app 1

# See release history
helm history my-app
```

> 💡 **Interview Gold**: *"Helm solves the problem of managing Kubernetes manifests at scale. Instead of copy-pasting YAML for each environment, you template it once and override values. The real power is in the release management — every `helm upgrade` creates a versioned release, and you can `helm rollback` instantly if something breaks."*

---

## Screen 5 — CI/CD, Monitoring & 12-Factor

### CI/CD Pipelines

CI/CD (Continuous Integration / Continuous Deployment) automates the journey from code commit to production deployment. A typical pipeline:

1. **Build** — install dependencies, compile if needed
2. **Test** — run unit tests, integration tests, linting
3. **Build Image** — create a Docker image
4. **Push** — push the image to a container registry
5. **Deploy** — update the Kubernetes deployment with the new image

#### GitHub Actions Workflow Example

```yaml
# .github/workflows/deploy.yml
name: Build, Test & Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Step 1: Test ──────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U testuser"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with ruff
        run: ruff check .

      - name: Type check with mypy
        run: mypy . --strict

      - name: Run tests
        run: pytest --cov=app --cov-report=xml -v
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.xml

  # ── Step 2: Build & Push Docker Image ─────────────────────
  build-and-push:
    needs: test                 # only runs if tests pass
    if: github.ref == 'refs/heads/main'   # only on main branch
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Step 3: Deploy to Kubernetes ──────────────────────────
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production     # requires manual approval

    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app ./helm/my-fastapi-chart \
            --set image.tag=${{ github.sha }} \
            --namespace production \
            --wait --timeout 300s
```

> 🎯 **Aha!** Notice the `--wait` flag on `helm upgrade`. This tells Helm to wait until all Pods are ready before marking the deployment as successful. Without it, the CI pipeline might report "deployed" before the new Pods are actually healthy — giving you a false green check.

### Structured Logging and the ELK Stack

In a containerized world, you can't SSH into a server and `tail -f` a log file. You need **structured logging** that feeds into a centralized system.

**Structured logging** means emitting logs as JSON instead of plain text:

```python
import structlog
import logging

# Configure structured logging
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

# Usage
logger.info(
    "order_processed",
    order_id="ORD-12345",
    user_id="USR-67890",
    amount=99.99,
    duration_ms=42,
)
# Output: {"event": "order_processed", "order_id": "ORD-12345",
#           "user_id": "USR-67890", "amount": 99.99,
#           "duration_ms": 42, "level": "info",
#           "timestamp": "2026-04-06T12:00:00Z"}
```

**The ELK Stack:**
- **Elasticsearch** — stores and indexes log data for fast searching.
- **Logstash** (or Fluentd/Fluent Bit) — collects, transforms, and ships logs.
- **Kibana** — web UI for searching, visualizing, and creating dashboards from log data.

In K8s, you typically run Fluent Bit as a **DaemonSet** (one per node) that collects container logs from `/var/log/containers/` and forwards them to Elasticsearch.

### Monitoring with Prometheus + Grafana

**Prometheus** is a pull-based monitoring system. It scrapes metrics from your applications at regular intervals.

**Your app exposes metrics:**

```python
from prometheus_client import Counter, Histogram, generate_latest
from fastapi import FastAPI, Response

app = FastAPI()

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "endpoint"],
)

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type="text/plain",
    )
```

**Grafana** connects to Prometheus as a data source and lets you build dashboards and set up alerts (e.g., "alert me if p99 latency exceeds 500ms for 5 minutes").

### The 12-Factor App

The [12-Factor App](https://12factor.net) is a methodology for building modern, cloud-native applications. Interviewers love this topic because it tests whether you think about operational concerns, not just code.

Here are all 12 factors, with the **top 5 interview favorites** marked with ⭐:

| # | Factor | Summary |
|---|---|---|
| 1 | **Codebase** | One codebase in version control, many deploys |
| 2 | **Dependencies** | Explicitly declare and isolate deps |
| 3 | ⭐ **Config** | Store config in environment variables |
| 4 | **Backing Services** | Treat databases, caches as attached resources |
| 5 | ⭐ **Build, Release, Run** | Strictly separate build and run stages |
| 6 | ⭐ **Processes** | Execute as stateless processes |
| 7 | **Port Binding** | Export services via port binding |
| 8 | ⭐ **Concurrency** | Scale out via the process model |
| 9 | ⭐ **Disposability** | Fast startup, graceful shutdown |
| 10 | **Dev/Prod Parity** | Keep environments as similar as possible |
| 11 | **Logs** | Treat logs as event streams |
| 12 | **Admin Processes** | Run admin tasks as one-off processes |

**Deep dive on the top 5:**

**⭐ Factor 3 — Config:** Never hardcode configuration. Use environment variables. This is why we use ConfigMaps and Secrets in K8s, `.env` files in Compose, and environment variables in CI/CD. Config that changes between deploys (database URLs, API keys, feature flags) must be external to the code.

**⭐ Factor 5 — Build, Release, Run:** The build stage creates an artifact (Docker image). The release stage combines the artifact with config. The run stage executes it. These should be strictly separated — you should never modify code in a running container.

**⭐ Factor 6 — Processes:** Your app should be stateless. Don't store session data in memory — use Redis or a database. If a container dies and gets replaced, users shouldn't notice. This is what makes horizontal scaling possible.

**⭐ Factor 8 — Concurrency:** Scale by adding more processes (Pods), not by making one process bigger. This is exactly what HPA does — it adds more replicas. Design your app to work behind a load balancer with multiple instances.

**⭐ Factor 9 — Disposability:** Containers should start fast and shut down gracefully. Handle SIGTERM by finishing in-flight requests before exiting. This enables smooth rolling deployments and fast autoscaling.

> 💡 **Interview Gold**: *"The 12-Factor methodology is really about building apps that are cloud-native by design. The three I emphasize most are: stateless processes (Factor 6) so you can scale horizontally, config in environment variables (Factor 3) so the same image runs in any environment, and disposability (Factor 9) so containers can be created and destroyed freely. These three factors are why Kubernetes works — if your app violates them, K8s can't manage it effectively."*

---

## Screen 6 — Quiz Time

Test your knowledge with these interview-style questions. Try answering before looking at the solution!

---

### Q1: Why should you copy `requirements.txt` before copying your application code in a Dockerfile?

A) Python requires dependencies to be installed before code is present  
B) It ensures the `pip install` layer is cached and only re-runs when dependencies change  
C) Docker cannot process COPY instructions after RUN instructions  
D) It prevents requirements.txt from being overwritten by the application code  

**Answer: B**

Docker caches each layer. By copying `requirements.txt` and running `pip install` *before* copying the rest of your code, the install layer stays cached across builds where only application code changes. This can save minutes per build. If you copy all files first, any code change invalidates the pip install cache.

---

### Q2: A Kubernetes Pod is running but failing its readiness probe. What happens?

A) The Pod is immediately deleted and a new one is created  
B) The Pod is restarted by the kubelet  
C) The Pod is removed from the Service's endpoints so it receives no traffic  
D) The Deployment scales up to compensate for the unhealthy Pod  

**Answer: C**

Readiness probe failure removes the Pod from the Service's load balancer — it stops receiving traffic. The Pod is *not* restarted (that's what liveness probes do) and *not* deleted. It keeps running so it can potentially recover and become ready again. This is the critical distinction between readiness and liveness probes.

---

### Q3: In Docker Compose, what does `depends_on` with `condition: service_healthy` guarantee?

A) The dependent service will automatically restart if the dependency fails  
B) The dependent service won't start until the dependency's health check passes  
C) Network traffic between the services is encrypted  
D) The dependent service shares the same container as the dependency  

**Answer: B**

`depends_on` with `condition: service_healthy` ensures the dependent container doesn't start until the dependency's health check reports healthy. Without the condition, `depends_on` only waits for the container to *start*, not for the application inside to be *ready*. This prevents race conditions where your app crashes because the database isn't accepting connections yet.

---

### Q4: Which 12-Factor principle is violated when you store user session data in application memory?

A) Factor 3 — Config  
B) Factor 5 — Build, Release, Run  
C) Factor 6 — Processes  
D) Factor 11 — Logs  

**Answer: C**

Factor 6 (Processes) states that applications should be stateless. Storing session data in memory makes the process stateful — if the container is restarted or a request is routed to a different Pod, the session is lost. Instead, store sessions in an external backing service like Redis or a database. This is a prerequisite for horizontal scaling.

---

### Q5: What is the primary benefit of using Helm charts over raw Kubernetes YAML manifests?

A) Helm charts run faster than raw YAML  
B) Helm provides templating, versioning, and release management for K8s resources  
C) Helm eliminates the need for a container registry  
D) Helm automatically scales applications based on traffic  

**Answer: B**

Helm's core value is templating (reuse manifests across environments with different values), versioning (each deployment is a numbered release), and release management (easy rollback with `helm rollback`). It doesn't affect runtime performance, doesn't replace container registries, and doesn't handle autoscaling — that's HPA's job.

---

## Wrapping Up

You've now covered the full journey from writing a Dockerfile to deploying, scaling, and monitoring applications in production. Here's your cheat sheet for interviews:

1. **Docker** — multi-stage builds, layer caching, non-root users, slim base images
2. **Docker Compose** — multi-service local dev, health checks, bind mounts vs named volumes
3. **Kubernetes basics** — Pods, Deployments, Services, ConfigMaps, Secrets
4. **K8s advanced** — readiness vs liveness probes, HPA, Helm for templating
5. **CI/CD** — build → test → push → deploy pipelines, GitHub Actions
6. **Operations** — structured logging (ELK), metrics (Prometheus + Grafana), 12-Factor principles

The golden thread connecting all of this: **treat your infrastructure as code, your containers as disposable, and your deployments as automated.** If you can articulate that philosophy and back it up with specifics from this module, you'll ace the containers and deployment portion of any backend interview.

Good luck out there! 🚀
