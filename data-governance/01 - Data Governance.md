---
tags: [data-governance, supplementary]
phase: supplementary
status: not-started
priority: medium
---

# 🔒 Data Governance & Security

> **Phase:** Supplementary | **Priority:** Medium
> **Related:** [[01 - Data Modeling]], [[01 - Cloud AWS]], [[01 - System Design Data Platform]]

---

## Checklist

### Data Catalog & Discovery
- [ ] DataHub (LinkedIn): discovery, lineage, governance, metadata management
- [ ] Amundsen (Lyft): search-first data discovery
- [ ] Unity Catalog (Databricks): unified governance for lakehouse
- [ ] OpenMetadata: open-source, comprehensive metadata platform
- [ ] Features: search, column descriptions, owners, tags, usage stats

### Data Lineage
- [ ] Column-level lineage: track data flow from source column to target
- [ ] Impact analysis: what downstream tables/reports are affected by a change?
- [ ] OpenLineage: open standard for lineage event collection
- [ ] Integration: Airflow, Spark, dbt → lineage events → DataHub/Marquez

### PII Handling
- [ ] Detection: regex patterns (SSN, email), NER models, commercial tools
- [ ] Masking: replace with fake data, preserve format
- [ ] Tokenization: replace with random token, maintain lookup table
- [ ] Anonymization: irreversible — k-anonymity, differential privacy
- [ ] Encryption: column-level encryption for sensitive fields

### Access Control
- [ ] RBAC (Role-Based): roles → permissions, simpler to manage
- [ ] ABAC (Attribute-Based): policies based on attributes (department, project)
- [ ] Column-level security: restrict access to sensitive columns
- [ ] Row-level security: filter rows based on user attributes
- [ ] Dynamic data masking: mask values at query time based on user role

### Compliance
- [ ] GDPR: right to deletion (right to be forgotten), data portability
- [ ] CCPA: California consumer privacy, opt-out rights
- [ ] SOC 2: security, availability, processing integrity, confidentiality
- [ ] Implications for data pipelines: deletion requests, audit trails, encryption

### Data Contracts
- [ ] Schema validation: enforce structure between producer and consumer
- [ ] SLAs: freshness, completeness, volume guarantees
- [ ] Quality rules: accepted ranges, uniqueness, referential integrity
- [ ] Ownership: who is responsible for each dataset
- [ ] Breaking change process: notification, migration period

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] DataHub docs: https://datahubproject.io/
- [ ] "Data Mesh" by Zhamak Dehghani (governance chapters)
- [ ] OpenLineage: https://openlineage.io/
