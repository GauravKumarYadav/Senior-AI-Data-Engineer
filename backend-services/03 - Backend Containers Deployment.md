---
tags: [backend, containers, devops, supplementary]
phase: supplementary
status: not-started
priority: low
---

# 🐳 Backend — Containers & Deployment

> **Phase:** Supplementary (Good-to-Have) | **Priority:** Low
> **Related:** [[01 - Backend FastAPI]], [[05 - Spark in Production]], [[01 - Cloud AWS]]

---

## Checklist

### Docker
- [ ] Dockerfile: multi-stage builds for smaller Python images
- [ ] Docker Compose: multi-container local dev (app + DB + Redis)
- [ ] Optimizing Python images: use slim/alpine, layer caching, .dockerignore
- [ ] Volume mounts for development, bind mounts vs named volumes

### Kubernetes Basics
- [ ] Pods, Deployments, Services, Ingress
- [ ] ConfigMaps and Secrets: configuration management
- [ ] Health checks: readiness and liveness probes
- [ ] Horizontal Pod Autoscaler (HPA): auto-scaling based on metrics
- [ ] Jobs and CronJobs: batch workloads (relevant to data pipelines)
- [ ] Helm charts: templated K8s manifests, package management

### Infrastructure as Code
- [ ] Terraform basics: providers, resources, state, modules
- [ ] Terraform for data infra: S3 buckets, Glue jobs, Redshift clusters, IAM roles
- [ ] State management: remote state (S3 backend), state locking

### CI/CD
- [ ] GitHub Actions: workflows for data pipelines
- [ ] dbt CI: test models on PR, deploy on merge
- [ ] Airflow DAG testing: validate DAG syntax, unit test tasks
- [ ] SQLFluff: SQL linting in CI
- [ ] Docker image builds in CI: build, test, push to registry

---

## 📝 Notes

_Start writing notes here as you study..._
