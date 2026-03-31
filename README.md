# Senior AI Data Engineer — Interview Prep Hub

Interactive HTML courses covering core data engineering topics for interview preparation. No setup required — runs entirely in your browser.

**[Open the Live Site →](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/)**

## Courses

| Course | Modules | Topics |
|--------|---------|--------|
| [Apache Spark](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/spark/course-spark-interview/) | 7 | RDDs, DataFrames, architecture, performance, streaming |
| [Scala](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/scala-interview/course-scala-interview/) | 8 | Case classes, pattern matching, collections, concurrency, implicits |
| [Data Modeling](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-modeling/course-data-modeling/) | 6 | Star schemas, SCDs, grain, bridge tables, fact tables |
| [Data Quality](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-quality/course-data-quality/) | 6 | Quality dimensions, data contracts, schema drift, testing |
| [Data Governance](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/data-governance/course-data-governance/) | 6 | RBAC/ABAC, PII, GDPR, catalogs, retention policies |
| [Cost Optimization](https://gauravkumaryadav.github.io/Senior-AI-Data-Engineer/cost-optmization/course-cost-optimization/) | 6 | Partitioning, shuffle, FinOps, materialization, cluster mgmt |

## Features

- Interactive quizzes with instant feedback
- Visual diagrams and flow charts
- Glossary tooltips on technical terms
- Scenario-based interview questions
- Keyboard navigation and mobile support
- Each module is a self-contained HTML file — no build step

## Adding a New Course

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

3. Push to `main` — the landing page picks it up automatically. No HTML changes needed.

## Local Development

Open `index.html` via any static server (the landing page uses `fetch` which requires HTTP, not `file://`):

```bash
npx serve .
# or
python3 -m http.server
```

Individual course modules can be opened directly as files in any browser.
