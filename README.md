# Senior AI Data Engineer — Interview Prep Hub 🎯

Interactive HTML courses covering every core topic for Senior AI/Data Engineer interviews. No setup required — runs entirely in your browser.

**[🚀 Open the Live Site →](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/)**

---

## 📊 At a Glance

| | |
|---|---|
| **Courses** | 18 |
| **Modules** | 111+ |
| **Questions** | 1,200+ |
| **Prerequisites** | 0 |

---

## 📚 All Courses

### Core Data Engineering

| Course | Modules | Topics |
|--------|---------|--------|
| [Apache Spark](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/spark/course-spark-interview/) | 12 | RDDs, DataFrames, architecture, tuning, Spark UI, streaming, lakehouse |
| [SQL & Databases](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/databases-SQL/course-sql-databases/) | 6 | Window functions, CTEs, optimization, PostgreSQL, NoSQL |
| [Data Modeling](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-modeling/course-data-modeling/) | 9 | Star schemas, SCDs, Data Vault, lakehouse modeling |
| [Real-Time Streaming](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/realtime-streaming/course-streaming/) | 4 | Kafka, exactly-once, Schema Registry, Flink, watermarks |
| [dbt Mastery](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-infra-DevOps/course-dbt-mastery/) | 8 | Models, materializations, incremental, testing, Jinja, CI/CD |
| [Scala](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/scala-interview/course-scala-interview/) | 8 | Case classes, pattern matching, collections, concurrency, implicits |

### AI & Machine Learning

| Course | Modules | Topics |
|--------|---------|--------|
| [AI & ML Foundations](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/AI-ML-Foundations/course-ai-ml/) | 4 | ML fundamentals, deep learning, NLP, transformers, PyTorch |
| [LLMs & GenAI](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/LLMs-GenAI/course-llms-genai/) | 7 | LLM fundamentals, prompts, RAG, LangChain, LangGraph, agents |
| [MLOps & AI Infra](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/MLOps-AI-Infra/course-mlops/) | 5 | Pipelines, model serving, LLMOps, fine-tuning, monitoring |

### System Design & Architecture

| Course | Modules | Topics |
|--------|---------|--------|
| [System Design — DE](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/system-design-DE/course-system-design-de/) | 4 | Data platforms, real-time CDC, batch systems, tradeoffs |
| [System Design — AI/ML](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/system-design-AI/course-system-design-ai/) | 3 | RAG systems, fraud detection, search ranking, ML platforms |
| [Cloud AWS](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/cloud-platform/course-cloud-aws/) | 6 | S3, Glue, Athena, EMR, Kinesis, Redshift, Lake Formation |
| [Backend Services](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/backend-services/course-backend/) | 4 | FastAPI, Pydantic, REST, microservices, Docker, K8s, CI/CD |

### Quality, Governance & Operations

| Course | Modules | Topics |
|--------|---------|--------|
| [Data Quality](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-quality/course-data-quality/) | 6 | Quality dimensions, data contracts, schema drift, testing |
| [Data Governance](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-governance/course-data-governance/) | 6 | RBAC/ABAC, PII, GDPR, catalogs, retention policies |
| [Cost Optimization](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/cost-optmization/course-cost-optimization/) | 6 | Partitioning, shuffle, FinOps, materialization, cluster mgmt |

### Coding & Behavioral

| Course | Modules | Topics |
|--------|---------|--------|
| [DSA & Coding](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/DSA-coding/course-dsa/) | 8 | Arrays, hashmaps, trees, DP, sorting, Pandas/Spark coding, SQL |
| [Behavioral & Leadership](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/behavioral-leadership/course-behavioral/) | 5 | STAR method, communication, project mgmt, incidents |

---

## ✨ Features

- 🧩 Interactive quizzes with instant feedback
- 📐 Visual diagrams and flow charts
- 📖 Glossary tooltips on technical terms
- 🎭 Scenario-based interview questions
- ⌨️ Keyboard navigation and mobile support
- 📄 Each module is a self-contained HTML file — no build step

---

## ➕ Adding a New Course

1. Create a folder with your course content (e.g. `new-topic/course-new-topic/index.html` + modules)
2. Add one entry to `courses.json`:

```json
{
  "title": "New Topic Interview Mastery",
  "icon": "🔥",
  "color": "#E06B56",
  "href": "./new-topic/course-new-topic/",
  "desc": "Description of the course.",
  "tags": ["6 Modules", "Interactive", "Scenario Quizzes"]
}
```

3. Push to `main` — the landing page picks it up automatically via GitHub Actions. No HTML changes needed.

---

## 🛠️ Local Development

Open `index.html` via any static server (the landing page uses `fetch` which requires HTTP, not `file://`):

```bash
npx serve .
# or
python3 -m http.server
```

Individual course modules can be opened directly as files in any browser.

---

## 📦 Deployment

This site is deployed automatically to GitHub Pages via GitHub Actions on every push to `main`. The workflow is defined in `.github/workflows/deploy.yml`.

---

Built with ❤️ for interview prep. No frameworks, no build steps — just open and learn.
