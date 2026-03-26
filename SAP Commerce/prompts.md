# Prompts for generating articles about SAP Commerce

## Article 1: SAP Commerce Cloud Architecture Demystified: A Practical Guide for Developers
### Prompt:
Write a detailed, practical article titled "SAP Commerce Cloud Architecture Demystified: A Practical Guide for Developers." The article should cover:
- What SAP Commerce Cloud (formerly Hybris) is and where it fits in the SAP CX ecosystem
- High-level architecture overview: Client Layer, OCC REST API Layer, Application Layer, Platform Layer, Persistence Layer
- The Extension System: platform extensions, commerce extensions, custom extensions, and AddOns — how they compose together
- The Type System: items.xml, dynamic data modeling, relations, enums, and how it differs from traditional ORM
- The Service Layer pattern: Controllers → Facades → Services → DAOs → Models and why this layering matters
- Configuration hierarchy: platform defaults → extension properties → local.properties → environment variables → system properties
- Multi-tenancy model: master vs. slave tenants
- Key design patterns used in SAP Commerce: Strategy pattern, Populator/Converter pattern, Interceptor chain
- Common architectural mistakes new developers make and how to avoid them
- When to customize vs. when to extend vs. when to use out-of-the-box features

Tone: Professional but accessible, written from hands-on experience. Go deep into technical details — explain internal mechanisms, include class names, interface hierarchies, and platform internals where relevant. Use ASCII architecture diagrams where appropriate and include substantial code/configuration examples. Target audience: Java developers new to SAP Commerce or mid-level developers wanting deeper understanding. No word limit — cover each topic thoroughly.

## Article 2: Mastering ImpEx in SAP Commerce: From Basics to Advanced Patterns
### Prompt:
Write a comprehensive article titled "Mastering ImpEx in SAP Commerce: From Basics to Advanced Patterns." The article should include:
- What ImpEx is and why it's the backbone of data management in SAP Commerce
- Basic syntax: INSERT, INSERT_UPDATE, UPDATE, REMOVE operations with clear examples
- Header modifiers deep dive: [unique=true], [lang=en], [default=value], [mode=append], [translator=class], [allownull=true], [forceWrite=true]
- Referencing related items: attribute paths, nested references, and PK references
- Macros and variables: how to use $-macros for clean, maintainable ImpEx files
- Advanced patterns: collections, maps, media import (from JAR and URL), conditional operations (#% if/endif)
- BeanShell and Groovy scripting within ImpEx: beforeEach, afterEach hooks
- Performance optimization: batch mode, worker threads, disabling interceptors for bulk imports
- Organizing ImpEx files: essentialdata, projectdata, sampledata structure
- Export ImpEx: how to export data for migration or backup
- Common pitfalls and troubleshooting: "Could not resolve item", duplicate unique keys, encoding issues
- Real-world tips from production experience

Tone: Tutorial-style with progressive complexity. Include many practical ImpEx code examples that readers can copy and adapt. Go deep into technical details — explain internal mechanisms, include class names, interface hierarchies, and platform internals where relevant. Target audience: SAP Commerce developers at all levels. No word limit — cover each topic thoroughly.

## Article 3: FlexibleSearch Queries in SAP Commerce: The Complete Developer's Guide
### Prompt:
Write a thorough article titled "FlexibleSearch Queries in SAP Commerce: The Complete Developer's Guide." Cover:
- What FlexibleSearch is and how it differs from standard SQL
- Basic query syntax: SELECT, FROM, WHERE, JOIN with type system integration
- Understanding the curly brace syntax: {pk}, {code}, {name[en]}, and how they map to the type system
- Joins and subqueries: relating types, many-to-many relations, catalog version filtering
- Query parameters: using ?parameter syntax for safe, parameterized queries
- Localized attributes: querying across multiple languages
- Common query patterns: product search, order lookups, user queries, catalog version-aware queries
- Performance optimization: avoiding SELECT *, using indexes, query result caching, analyzing query plans
- FlexibleSearch vs. Solr: when to use each and why
- Using FlexibleSearch in code: FlexibleSearchService, SearchResult, pagination
- Using FlexibleSearch in HAC (Hybris Administration Console) for debugging
- Real-world complex query examples from production systems
- Top 10 FlexibleSearch mistakes and how to fix them

Tone: Practical and example-driven. Every concept should be illustrated with a working query example. Go deep into technical details — explain internal mechanisms, query execution internals, and platform behavior where relevant. Target audience: SAP Commerce developers who want to master data querying. No word limit — cover each topic thoroughly.

## Article 4: Performance Optimization in SAP Commerce Cloud: Lessons from Production
### Prompt:
Write an in-depth article titled "Performance Optimization in SAP Commerce Cloud: Lessons from Production." Include:
- Why performance matters in e-commerce and the cost of slow page loads
- SAP Commerce caching system deep dive: platform cache regions, query result caching, cache invalidation strategies, distributed caching
- FlexibleSearch query optimization: common anti-patterns, index usage, query plan analysis
- Solr search performance: indexing strategies, batch sizes, commit intervals, partial vs. full indexing
- JVM tuning: heap sizing, garbage collector selection (G1GC, ZGC), GC log analysis for SAP Commerce workloads
- Database optimization: connection pool sizing, query tuning, SAP HANA vs. MySQL/PostgreSQL considerations
- ServiceLayer performance patterns: lazy loading, DTO optimization, avoiding N+1 queries in populators
- CDN configuration for static assets and media files
- Async processing: task engine, cron jobs, background processing aspect
- Load testing strategies: tools (Gatling, JMeter), realistic test scenarios, baseline establishment
- Monitoring and profiling: HAC monitoring tools, JMX metrics, thread dump analysis
- Real-world performance war stories: what went wrong, how we diagnosed it, and the fix
- Performance checklist for production readiness

Tone: Experience-driven and actionable. Include configuration examples, property settings, and before/after metrics where possible. Go deep into technical details — explain internal mechanisms, JVM internals, database behavior, and platform architecture where relevant. Target audience: SAP Commerce developers and architects responsible for production systems. No word limit — cover each topic thoroughly.

## Article 5: Deploying SAP Commerce to CCv2: A Complete Guide to Cloud Deployment
### Prompt:
Write a detailed article titled "Deploying SAP Commerce to CCv2: A Complete Guide to Cloud Deployment." Cover:
- What CCv2 (Commerce Cloud v2) is and how it changes the deployment model
- CCv2 architecture: Cloud Portal, environments (Dev/Staging/Production), managed services (Solr, ZooKeeper, HANA)
- The manifest.json file: complete breakdown of structure — commerceSuiteVersion, extensions, aspects, properties, storefrontAddons, tests
- Aspects explained: accstorefront, api, backoffice, backgroundProcessing — what each does and how to configure them
- Repository structure: core-customize/ and js-storefront/ layout
- Build process: creating builds from Git branches, build lifecycle, build optimization tips
- Deployment strategies: Rolling Update vs. Blue-Green, database update modes (INIT, UPDATE, NONE)
- Environment configuration: aspect-specific properties, environment-specific settings, secrets management
- Scaling: horizontal scaling per aspect, auto-scaling configuration, sizing guidelines
- Monitoring and logging: Cloud Portal dashboard, centralized logs, health endpoints
- Security: WAF, CORS configuration, SSL/TLS, IP whitelisting
- CI/CD pipeline integration: automating builds and deployments via CCv2 API
- Migration from on-premise to CCv2: key considerations and gotchas
- Best practices and common deployment mistakes to avoid

Tone: Step-by-step practical guide with real configuration examples. Include manifest.json snippets, API curl commands, and property configurations. Go deep into technical details — explain internal mechanisms, deployment internals, and infrastructure architecture where relevant. Target audience: SAP Commerce developers and DevOps engineers deploying to CCv2. No word limit — cover each topic thoroughly.

## Article 6: Building Custom Extensions in SAP Commerce: Best Practices and Patterns
### Prompt:
Write a practical article titled "Building Custom Extensions in SAP Commerce: Best Practices and Patterns." Include:
- Why custom extensions are the primary customization mechanism in SAP Commerce
- Extension anatomy: extensioninfo.xml, items.xml, -spring.xml, -web-spring.xml, resources, web directory
- Creating a new extension: using the extgen ant target, template selection, naming conventions
- Defining custom types in items.xml: Item Types, Relations, Enums, Collection Types, Map Types
- Implementing the Service Layer: Services, DAOs, Facades, Populators, Converters — with code examples
- Interceptors and Validators: PrepareInterceptor, ValidateInterceptor, InitDefaultsInterceptor, RemoveInterceptor — lifecycle hooks explained
- Spring configuration: bean definitions, dependency injection patterns, profiles
- Event handling: the event system, custom events, event listeners
- Dynamic attribute handlers: computed attributes without database columns
- AddOn development: when to use AddOns vs. regular extensions, AddOn installation and structure
- Extension dependencies: requires-extension, managing the dependency graph
- Testing custom extensions: unit tests, integration tests, test annotations
- Code organization patterns for large projects: core/facades/storefront/backoffice extension split
- Common mistakes: over-customization, breaking upgrade paths, ignoring the extension lifecycle

Tone: Hands-on tutorial with progressive complexity. Include complete code examples (Java, Spring XML, items.xml). Go deep into technical details — explain internal mechanisms, class hierarchies, Spring context behavior, and platform lifecycle where relevant. Target audience: SAP Commerce developers building custom functionality. No word limit — cover each topic thoroughly.

## Article 7: SAP Commerce Catalog System and Synchronization: A Deep Dive
### Prompt:
Write a comprehensive article titled "SAP Commerce Catalog System and Synchronization: A Deep Dive." Cover:
- The catalog concept in SAP Commerce: why catalogs exist and the business problems they solve
- Catalog structure: Catalog → CatalogVersion → Products/Categories/Media hierarchy
- Staged vs. Online concept: the dual-catalog model and content approval workflow
- Product catalogs vs. Content catalogs vs. Classification catalogs — differences and use cases
- Catalog version filtering: how the session catalog version context works and why it matters
- Synchronization mechanism: how sync jobs work internally, SyncJob configuration, sync attributes
- Configuring catalog synchronization: CatalogVersionSyncJob, CatalogVersionSyncCronJob, partial sync
- Sync performance optimization: max threads, batch size, selective attribute sync, excluding types
- Content catalog sync and CMS components: page synchronization, slot synchronization, component versioning
- Multi-catalog strategies: multi-country setups, B2B/B2C catalog splits, shared vs. local catalogs
- Classification system: classification catalogs, classification classes, features and feature values
- Common synchronization issues: missing items, stale references, sync conflicts — and how to resolve them
- Best practices for catalog design in large-scale projects
- Real-world patterns from enterprise implementations

Tone: Architectural and practical, explaining both the "why" and the "how." Include ImpEx examples for catalog setup and sync configuration. Go deep into technical details — explain internal synchronization mechanisms, catalog version resolution internals, and platform behavior where relevant. Target audience: SAP Commerce developers and solution architects. No word limit — cover each topic thoroughly.

## Article 8: Extending the OCC REST API in SAP Commerce: Building Modern Commerce APIs
### Prompt:
Write a detailed article titled "Extending the OCC REST API in SAP Commerce: Building Modern Commerce APIs." Include:
- What OCC (Omni Commerce Connect) is: the RESTful API layer for headless commerce
- OCC architecture: Controllers, DTOs (WsDTO), Validators, Mappers, and how they connect to the Service Layer
- Out-of-the-box API endpoints: products, cart, checkout, users, orders — overview of available functionality
- Extending existing endpoints: adding fields to existing DTOs, customizing response data
- Creating new custom endpoints: step-by-step guide with @Controller, @RequestMapping, @ResponseBody
- DTO design: WsDTO classes, field-level mapping, Orika mappers, field set levels (BASIC, DEFAULT, FULL)
- Authentication and authorization: OAuth2 configuration, client credentials, password grant, trusted client
- API versioning and backward compatibility strategies
- Error handling: exception mapping, error response format, validation error responses
- Swagger/OpenAPI documentation: automatic generation, custom annotations
- CORS configuration for headless storefront integration
- API performance: caching headers, pagination, partial response with field sets
- Security best practices: rate limiting, input validation, token management
- Testing OCC APIs: integration testing, Postman collections, automated API tests
- Real-world tips for building APIs that power Spartacus/Composable Storefront

Tone: Tutorial-style with complete code examples. Include Java code, Spring configuration, and curl command examples for testing. Go deep into technical details — explain internal request processing, DTO mapping internals, and security architecture where relevant. Target audience: SAP Commerce developers building or extending REST APIs. No word limit — cover each topic thoroughly.

## Article 9: SAP Commerce Composable Storefront (Spartacus): Architecture and Customization Guide
### Prompt:
Write a thorough article titled "SAP Commerce Composable Storefront (Spartacus): Architecture and Customization Guide." Cover:
- What Spartacus/Composable Storefront is: Angular-based SPA, headless architecture, relationship with OCC API
- Architecture overview: feature modules, lazy loading, state management (NgRx), CMS-driven layout
- Setup and project structure: creating a new Spartacus project, library modules, configuration
- CMS-driven page structure: how pages, slots, and components map from SAP Commerce CMS to Angular components
- Customizing components: component replacement, outlet-based customization, custom CMS component mapping
- Extending services: overriding OCC adapters, custom normalizers/serializers, service injection
- Theming and styling: SCSS structure, CSS custom properties, responsive design patterns
- Routing and navigation: configurable routes, SEO-friendly URLs, route guards
- State management patterns: NgRx store, effects, selectors — when and how to customize
- SmartEdit integration: how in-context editing works with Spartacus
- Server-Side Rendering (SSR): setup, benefits, common issues and fixes
- Performance optimization: lazy loading strategies, preloading, bundle size optimization
- B2B Commerce features: organization management, approval workflows, purchase orders
- Upgrading Spartacus: version migration strategies, breaking changes, schematics
- Common customization patterns from real projects

Tone: Practical and architecture-focused. Include TypeScript/Angular code examples and configuration snippets. Go deep into technical details — explain Angular internals, NgRx state management patterns, CMS rendering pipeline, and SSR architecture where relevant. Target audience: frontend developers working with SAP Commerce Composable Storefront. No word limit — cover each topic thoroughly.

## Article 10: Data Migration Strategies for SAP Commerce Projects
### Prompt:
Write a practical article titled "Data Migration Strategies for SAP Commerce Projects." Include:
- Why data migration is one of the most challenging aspects of SAP Commerce projects
- Types of data migration: initial load, incremental sync, live cutover, parallel run
- ImpEx-based migration: using ImpEx for bulk data loading, performance tuning, error handling
- Hot Folders: automated file-based import, configuration, monitoring, error recovery
- Data Hub and Integration API: when to use for complex transformations
- Migration planning: source data analysis, mapping documents, transformation rules
- Product data migration: catalogs, categories, products, variants, prices, stock — the dependency order
- Customer data migration: accounts, addresses, B2B organizations, consent and GDPR considerations
- Order history migration: historical orders, order statuses, payment records
- Content migration: CMS pages, components, media assets
- Handling large datasets: chunking strategies, parallel processing, memory management
- Data validation and reconciliation: automated checks, record counts, sampling verification
- Rollback strategies: what to do when migration goes wrong
- Testing data migration: dry runs, staging environment validation, performance testing
- Timeline and cutover planning: minimizing downtime, go-live checklist
- Real-world migration war stories and lessons learned

Tone: Project-focused and experience-driven. Include ImpEx examples, scripts, and checklists. Go deep into technical details — explain data loading internals, Hot Folder processing mechanisms, and platform import architecture where relevant. Target audience: SAP Commerce developers and project managers involved in implementation projects. No word limit — cover each topic thoroughly.

## Article 11: Caching Strategies in SAP Commerce Cloud: A Production-Ready Guide
### Prompt:
Write an in-depth article titled "Caching Strategies in SAP Commerce Cloud: A Production-Ready Guide." Cover:
- Why caching is critical for SAP Commerce performance at scale
- SAP Commerce cache architecture: region cache, EHCache under the hood, cache regions configuration
- Platform cache regions: entityRegion, queryCacheRegion, typesystemRegion — what each caches and how to tune them
- Cache configuration: maxElementsInMemory, timeToLiveSeconds, timeToIdleSeconds, eviction policies
- Query result caching: enabling FlexibleSearch result caching, cache hints, TTL configuration
- Cache invalidation: automatic invalidation on model save, manual invalidation, cluster-wide invalidation
- Distributed caching in clustered environments: cache synchronization, invalidation broadcasting
- HTTP caching: Cache-Control headers, ETags, CDN integration for OCC API responses
- Storefront caching: CDN for static assets, page caching strategies, Spartacus service worker
- CMS component caching: component-level caching, personalization impact on cacheability
- Solr cache: filterCache, queryResultCache, documentCache — Solr-specific tuning
- Cache monitoring: hit rates, miss rates, eviction counts, using HAC and JMX
- Common caching pitfalls: stale data, memory pressure, cache stampede, over-caching
- Cache warm-up strategies after deployment
- Configuration examples for different deployment sizes (small, medium, large)
- Production checklist for cache configuration

Tone: Technical and production-focused. Include property configurations, monitoring queries, and sizing recommendations. Go deep into technical details — explain cache internals, EHCache region behavior, invalidation propagation, and cluster cache synchronization where relevant. Target audience: SAP Commerce developers and operations engineers. No word limit — cover each topic thoroughly.

## Article 12: Testing in SAP Commerce: Unit Tests, Integration Tests, and Quality Assurance
### Prompt:
Write a practical article titled "Testing in SAP Commerce: Unit Tests, Integration Tests, and Quality Assurance." Include:
- Why testing in SAP Commerce is different from standard Java application testing
- Testing pyramid for SAP Commerce: unit tests, integration tests, API tests, E2E tests
- Unit testing: JUnit 5, Mockito, testing services/facades/populators without the platform
- Integration testing: @IntegrationTest annotation, ServicelayerTransactionalTest, running with the platform context
- Test annotations in SAP Commerce: @UnitTest, @IntegrationTest, @DemoTest — and how they're used in CCv2 builds
- Testing ImpEx data loading: creating test fixtures, test data isolation
- Testing FlexibleSearch queries: verifying query results, testing with sample data
- Testing custom interceptors and validators: lifecycle hook testing patterns
- Testing OCC API endpoints: MockMvc, REST Assured, end-to-end API testing
- Testing Spartacus customizations: Jasmine/Karma, component testing, service testing
- Performance testing: Gatling scripts for SAP Commerce, realistic load scenarios
- Test data management: creating reusable test data, ImpEx test fixtures, test data cleanup
- CI/CD integration: running tests in CCv2 builds, test configuration in manifest.json
- Code coverage: tools and realistic coverage targets for SAP Commerce projects
- Common testing challenges and how to overcome them
- Best practices for building a testing culture on SAP Commerce projects

Tone: Hands-on with code examples for each testing pattern. Include Java test code, configuration snippets, and CI/CD pipeline examples. Go deep into technical details — explain test framework internals, platform test context lifecycle, and CCv2 test execution mechanisms where relevant. Target audience: SAP Commerce developers who want to improve code quality. No word limit — cover each topic thoroughly.

## Article 13: Troubleshooting SAP Commerce in Production: A Developer's Survival Guide
### Prompt:
Write a practical, experience-driven article titled "Troubleshooting SAP Commerce in Production: A Developer's Survival Guide." Cover:
- The mindset for production troubleshooting: systematic approach, don't panic, gather data first
- Essential diagnostic tools: HAC (Hybris Administration Console), what each tab offers and when to use it
- Log analysis: log4j2 configuration, log levels, how to enable debug logging for specific packages safely in production
- Thread dump analysis: how to capture thread dumps, identifying blocked threads, deadlocks, high-CPU threads
- Memory analysis: heap dumps, identifying memory leaks, common memory leak patterns in SAP Commerce (type system cache, session bloat, large collections)
- FlexibleSearch debugging: slow query identification, query plan analysis, the FlexibleSearch monitoring in HAC
- ImpEx import troubleshooting: import error logs, unresolved references, line-by-line debugging
- Catalog synchronization issues: sync stuck, missing items, stale references — diagnostic steps
- Solr indexing problems: partial indexing failures, schema mismatches, ZooKeeper connectivity
- Cluster issues: node communication problems, cache invalidation failures, session replication
- CCv2-specific troubleshooting: build failures, deployment issues, log access, support ticket best practices
- CronJob debugging: monitoring running jobs, identifying stuck jobs, CronJob lifecycle
- Performance degradation: identifying the bottleneck layer (DB, JVM, Solr, network)
- Common production issues and their solutions (top 10 list from experience)
- Building a runbook: what to document for operations teams
- Post-mortem practices: learning from incidents

Tone: War-story style with practical, actionable advice. Include command-line examples, HAC screenshots described, log patterns to search for, and step-by-step diagnostic procedures. Go deep into technical details — explain JVM diagnostic internals, thread lifecycle, memory management, and platform monitoring architecture where relevant. Target audience: SAP Commerce developers and support engineers. No word limit — cover each topic thoroughly.

## Article 14: Migrating to a New Version of SAP Commerce Cloud: A Practical Upgrade Guide
### Prompt:
Write a comprehensive, experience-driven article titled "Migrating to a New Version of SAP Commerce Cloud: A Practical Upgrade Guide." Cover:
- Why keeping SAP Commerce up to date matters: security patches, new features, end-of-maintenance timelines, CCv2 compatibility
- Understanding SAP Commerce versioning: major releases (1905, 2005, 2011, 2105, 2205, 2211), patch releases, and the yearly release cadence
- Pre-upgrade assessment: analyzing your current customizations, deprecated API usage, extension compatibility matrix
- Reading the release notes effectively: what to look for in "What's New", breaking changes, deprecated features, and removed APIs
- Step-by-step upgrade process:
  1. Set up a fresh installation of the target version
  2. Compare and merge configuration files (local.properties, localextensions.xml, manifest.json)
  3. Update items.xml definitions and handle type system changes
  4. Fix compilation errors from deprecated/removed APIs
  5. Run system update and verify database schema changes
  6. Update Solr schema and re-index
  7. Verify catalog synchronization
  8. Run full regression test suite
- Handling deprecated APIs and classes: finding replacements, using SAP's migration guides, search strategies for deprecated usage
- Type system migration: handling changed or removed types, attribute changes, enum modifications
- Spring framework upgrades: when SAP Commerce bumps Spring versions, common breaking changes
- AddOn to Composable Storefront migration: moving from legacy JSP storefront to Spartacus
- Spartacus/Composable Storefront version upgrades: Angular version bumps, library API changes, schematics usage
- manifest.json updates for CCv2: new commerceSuiteVersion, extension changes, property adjustments
- Database considerations: schema migration scripts, data compatibility, HANA-specific changes
- Testing strategy for upgrades: what to test, regression test prioritization, smoke test checklist
- Managing the upgrade in a team: branching strategy, parallel development during upgrade, feature freeze considerations
- Common upgrade pitfalls and how to avoid them:
  - Not reading the full release notes
  - Skipping versions (jumping multiple major versions at once)
  - Ignoring deprecation warnings in the current version
  - Not testing catalog sync after upgrade
  - Forgetting to update third-party integrations
- Rollback planning: what to prepare if the upgrade goes wrong
- Real-world tips and lessons learned from multiple SAP Commerce version upgrades
- Upgrade checklist: a comprehensive pre-upgrade, during-upgrade, and post-upgrade checklist

Tone: Practical and advisory, written from the perspective of someone who has performed multiple version upgrades. Include configuration diff examples, command sequences, and a printable checklist. Go deep into technical details — explain type system migration internals, Spring context changes, database schema evolution, and platform bootstrap mechanisms where relevant. Target audience: SAP Commerce developers, tech leads, and architects planning version upgrades. No word limit — cover each topic thoroughly.

## Article 15: The SAP Commerce Type System Deep Dive: From items.xml to Database and Beyond
### Prompt:
Write a comprehensive, in-depth article titled "The SAP Commerce Type System Deep Dive: From items.xml to Database and Beyond." Cover:
- What the type system is and why SAP Commerce chose a metadata-driven approach over traditional ORM (JPA/Hibernate)
- The type hierarchy explained: Item → GenericItem → your custom types, ComposedType, AtomicType, CollectionType, MapType, EnumType, RelationType
- items.xml anatomy in full detail: every element and attribute explained — `<itemtype>`, `<enumtype>`, `<maptypes>`, `<collectiontypes>`, `<relations>`, `<deployment>`, `<indexes>`, `<custom-properties>`
- Attribute persistence modes: `property` (database column), `dynamic` (DynamicAttributeHandler), `jalo` (legacy), `cmp` — when to use each
- Attribute modifiers deep dive: `unique`, `optional`, `read`, `write`, `initial`, `encrypted`, `dontOptimize`, `partof`, `isSelectionOf`
- Type deployment and typecodes: how the platform maps types to database tables, typecode assignment rules, the `ydeployments` table, single-table vs. dedicated-table inheritance, and the `props` table for overflow attributes
- Generated model classes: how the build generates Java code from items.xml, the Generated* superclass pattern, PersistenceContext, dirty tracking, lazy loading internals
- Localized attributes: how `localized:java.lang.String` works, the `*lp` tables, session language resolution, querying across languages
- Relations in depth: one-to-many, many-to-many, ordered collections, `partof` semantics, link table generation, `RelationDescriptor`, and relation end attributes
- Enums: static enums vs. dynamic enums, when to use each, adding values at runtime, HybrisEnumValue internals
- Collection types and map types: defining and using them, database storage format, performance implications
- Dynamic attributes: implementing `DynamicAttributeHandler`, Spring bean registration, use cases (computed fields, virtual attributes, external lookups)
- The runtime type system: `TypeManager`, `ComposedType`, `AttributeDescriptor` — inspecting the type system programmatically
- Type system and FlexibleSearch: how curly brace `{attribute}` syntax resolves to SQL, type-aware queries, polymorphic queries
- Type system evolution: adding attributes, removing attributes, changing types — what `ant updatesystem` does behind the scenes, schema migration internals
- Indexes: defining indexes in items.xml, unique indexes, composite indexes, database-specific index behavior
- Type system and the Backoffice: how the type system drives the Backoffice UI, automatic CRUD generation, editor area configuration
- Type system best practices: naming conventions, typecode ranges, when to create new types vs. extend existing ones, avoiding common pitfalls (orphaned types, typecode collisions, attribute name conflicts)
- Performance considerations: attribute count impact, table width, property table overflow, lazy loading behavior
- Real-world modeling patterns: product hierarchies, customer extensions, custom order attributes, multi-tenant type design
- Common mistakes and how to avoid them: modifying OOTB types incorrectly, breaking the deployment table, ignoring partof semantics, misusing dynamic attributes

Tone: Authoritative and deeply technical. This should be the definitive reference on the SAP Commerce type system. Include extensive items.xml examples, generated Java code snippets, database schema diagrams (ASCII), and FlexibleSearch queries that demonstrate type system concepts. Explain internal platform classes and mechanisms. Target audience: SAP Commerce developers who want to master the type system — from juniors learning the fundamentals to seniors who need to understand the internals. No word limit — cover each topic thoroughly.
