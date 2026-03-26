# Technical Articles

A collection of in-depth technical articles on SAP Commerce and Cloud Native Architecture, written for experienced developers and architects.

🌐 **Published at**: [https://alextsvetkov.github.io/articles/](https://alextsvetkov.github.io/articles/)

---

## Articles Overview

### SAP Commerce

| # | Article | Topics |
|---|---------|--------|
| 1 | [SAP Commerce Cloud Architecture Demystified](SAP%20Commerce/1-SAP_Commerce_Cloud_Architecture_Demystified.md) | Multi-environment architecture, extensions, type system, service layer, design patterns for Java developers |
| 2 | [Mastering ImpEx in SAP Commerce](SAP%20Commerce/2-Mastering_ImpEx_in_SAP_Commerce.md) | ImpEx syntax, INSERT_UPDATE, bulk imports, scripted imports, performance tuning, data management |
| 3 | [FlexibleSearch Queries in SAP Commerce](SAP%20Commerce/3-FlexibleSearch_Queries_in_SAP_Commerce.md) | FlexibleSearch syntax, query optimization, joins, subqueries, performance at scale, DAO patterns |
| 4 | [Performance Optimization in SAP Commerce Cloud](SAP%20Commerce/4-Performance_Optimization_in_SAP_Commerce_Cloud.md) | Caching, query optimization, architecture patterns, configuration tuning, production performance |
| 5 | [Deploying SAP Commerce to CCv2](SAP%20Commerce/5-Deploying_SAP_Commerce_to_CCv2.md) | CCv2 platform, manifest.json, CI/CD pipeline, Kubernetes/Azure, deployment patterns, troubleshooting |
| 6 | [Building Custom Extensions in SAP Commerce](SAP%20Commerce/6-Building_Custom_Extensions_in_SAP_Commerce.md) | Extension architecture, layering, service layer patterns, testability, maintainability best practices |
| 7 | [SAP Commerce Catalog System Deep Dive](SAP%20Commerce/7-SAP_Commerce_Catalog_System_Deep_Dive.md) | Staged content, catalog synchronization, multi-catalog architecture, B2B/multi-brand scenarios |
| 8 | [Extending the OCC REST API in SAP Commerce](SAP%20Commerce/8-Extending_the_OCC_REST_API_in_SAP_Commerce.md) | Custom endpoints, DTOs, field-level mapping, security, Spartacus integration, API patterns |
| 9 | [SAP Commerce Composable Storefront Deep Dive](SAP%20Commerce/9-SAP_Commerce_Composable_Storefront_Deep_Dive.md) | Spartacus/Angular architecture, customization model, state management, CMS integration, production deployment |
| 10 | [Data Migration Strategies for SAP Commerce Projects](SAP%20Commerce/10-Data_Migration_Strategies_for_SAP_Commerce_Projects.md) | Migration planning, tooling, legacy platform migration, testing approaches, recovery strategies |
| 11 | [Caching Strategies in SAP Commerce Cloud](SAP%20Commerce/11-Caching_Strategies_in_SAP_Commerce_Cloud.md) | Type system cache, region cache, HTTP caching, CDN configuration, cache stampede prevention |
| 12 | [Testing Strategies for SAP Commerce Projects](SAP%20Commerce/12-Testing_Strategies_for_SAP_Commerce_Projects.md) | Unit testing with mocking, integration tests, Solr search testing, ImpEx validation, OCC API E2E tests |
| 13 | [Troubleshooting SAP Commerce in Production](SAP%20Commerce/13-Troubleshooting_SAP_Commerce_in_Production.md) | Memory issues, slow queries, thread deadlocks, cache problems, CronJob failures, deployment errors |
| 14 | [Upgrading SAP Commerce Cloud Versions](SAP%20Commerce/14-Upgrading_SAP_Commerce_Cloud_Versions.md) | Upgrade planning, dependency analysis, code migration, breaking changes, testing, deployment (1905–2211) |
| 15 | [The SAP Commerce Type System Deep Dive](SAP%20Commerce/15-The_SAP_Commerce_Type_System_Deep_Dive.md) | items.xml, metadata-driven modeling, generated Java classes, database schemas, runtime type metadata |

### Cloud Native Architecture *(coming soon)*

Article prompts are available in [`Cloud Native Architecture/prompts.md`](Cloud%20Native%20Architecture/prompts.md).

---

## How to Use This Repository

1. **Browse the table above** and click on any article to read it directly on GitHub or via the [published site](https://alextsvetkov.github.io/articles/).
2. **Each article includes:**
   - Practical code examples and configuration snippets
   - Architecture diagrams and comparison tables
   - Common pitfalls and production lessons
3. **Articles are platform-neutral Markdown** — they can be adapted for Dev.to, Medium, SAP Community, or LinkedIn using the prompts in [`prompt.md`](prompt.md).

## Workflow

1. **Generate a base article** using prompts from `<topic>/prompts.md`
2. **Adapt for each platform** using prompts from [`prompt.md`](prompt.md)
3. **Publish** to the target platform (Dev.to, Medium, SAP Community, LinkedIn)

## License

This material is provided for learning purposes.