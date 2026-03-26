# Performance Optimization in SAP Commerce Cloud: A Battle-Tested Guide

Performance issues in SAP Commerce are inevitable at scale. A storefront that responds in 200ms with 100 products and 10 concurrent users becomes a sluggish 5-second experience when the catalog grows to 500,000 SKUs and traffic hits 1,000 requests per second. The difference between a fast and slow SAP Commerce deployment rarely comes down to hardware — it's architecture, configuration, and code patterns.

This guide covers the performance optimization techniques that matter most in real-world SAP Commerce projects. Every recommendation is drawn from production experience, not theoretical benchmarks.

---

## Where SAP Commerce Spends Time

Before optimizing anything, understand where the platform spends its time during a typical storefront request:

```
Incoming HTTP Request (OCC API)
    │
    ├── [5-15ms]  Spring Controller + Security Filter Chain
    ├── [10-50ms] Facade Layer (converter/populator chain)
    ├── [20-200ms] Service Layer (business logic + FlexibleSearch queries)
    ├── [5-100ms] Database Queries (via JDBC)
    ├── [0-50ms]  Cache Lookups/Misses
    ├── [0-500ms] Solr Queries (product search pages)
    └── [5-20ms]  Response Serialization (JSON)
    
Total: 50-900ms typical range
```

The biggest offenders, in order of frequency:

1. **Database queries** — poorly written FlexibleSearch, missing indexes, N+1 patterns
2. **Cache misses** — undersized caches, wrong eviction policies, no caching strategy
3. **Populator chains** — too many populators, each triggering lazy loads
4. **Solr queries** — complex facet configurations, large result sets
5. **Catalog synchronization** — blocking operations, full syncs instead of incremental

---

## Database and Query Optimization

### Indexing Strategy

The most impactful optimization is often the simplest: add the right indexes. SAP Commerce generates tables from `items.xml`, but the default indexes are minimal.

**Identify missing indexes** by enabling slow query logging:

```properties
# Log queries taking longer than 500ms
db.log.sql.slow=true
db.log.sql.slow.threshold=500
```

Then define indexes in your `items.xml`:

```xml
<itemtype code="Order" autocreate="false" generate="false">
    <indexes>
        <index name="orderUserDateIdx">
            <key attribute="user"/>
            <key attribute="creationtime"/>
        </index>
        <index name="orderStatusIdx">
            <key attribute="status"/>
        </index>
    </indexes>
</itemtype>
```

**Rules of thumb for SAP Commerce indexes:**
- Index every foreign key attribute used in WHERE clauses
- Create composite indexes for common multi-column queries
- Index `creationtime` and `modifiedtime` for any type you query by date
- Don't over-index — each index slows down writes

### FlexibleSearch Optimization

**Always SELECT {pk} only.** The Model layer handles attribute loading via cache:

```java
// BAD — fetches all columns from DB
"SELECT * FROM {Product} WHERE {catalogVersion} = ?cv"

// GOOD — fetches PKs, attributes loaded from cache on access
"SELECT {pk} FROM {Product} WHERE {catalogVersion} = ?cv"
```

**Eliminate N+1 queries.** This is the single most common performance issue in SAP Commerce DAOs:

```java
// BAD: 1 query for products + N queries for prices
List<ProductModel> products = productDao.findAll(cv);
for (ProductModel p : products) {
    priceService.getPrice(p); // triggers another query
}

// GOOD: batch fetch in one query
"SELECT {p.pk}, {pr.pk} FROM {Product AS p JOIN PriceRow AS pr ON {pr.product} = {p.pk}} WHERE {p.catalogVersion} = ?cv"
```

**Use query parameters, never string concatenation.** Parameterized queries enable database query plan caching:

```java
// Enables plan caching
query.addQueryParameter("code", productCode);
// vs. plan cache miss every time
"WHERE {code} = '" + productCode + "'"
```

**Avoid LIKE with leading wildcards.** `{name} LIKE '%widget%'` always triggers a full table scan. Use Solr for full-text search.

### Connection Pool Tuning

The database connection pool is a frequent bottleneck under load:

```properties
# Connection pool size — set based on (number of web threads * 2)
db.pool.maxActive=100
db.pool.maxIdle=50
db.pool.minIdle=10

# Connection validation
db.pool.testOnBorrow=true
db.pool.validationQuery=SELECT 1

# Connection timeout
db.pool.maxWait=10000

# Statement cache (significant for repeated queries)
db.pool.maxOpenPreparedStatements=200
```

**Sizing formula**: `maxActive` should be at least `tomcat.maxThreads * 1.5` to avoid thread starvation. If you have 200 web threads, set `maxActive=300`.

### HANA-Specific Optimizations

If running on SAP HANA (standard for CCv2):

```properties
# Enable HANA-specific optimizations
db.hana.column.store=true

# Use HANA connection pooling features
db.pool.removeAbandoned=true
db.pool.removeAbandonedTimeout=300

# HANA statement routing for scale-out
db.hana.statementRouting=true
```

---

## Caching Strategy

SAP Commerce uses a multi-layer caching architecture. Understanding and tuning each layer is critical.

### Cache Architecture

```
┌─────────────────────────────────────┐
│   Application Code                   │
├─────────────────────────────────────┤
│   L1: Model Cache (per-session)      │  ← Fastest, smallest
├─────────────────────────────────────┤
│   L2: Region Cache (CacheRegion)     │  ← Shared across threads
├─────────────────────────────────────┤
│   L3: FlexibleSearch Query Cache     │  ← Query result caching
├─────────────────────────────────────┤
│   L4: Database Query Cache           │  ← DB-level caching
└─────────────────────────────────────┘
```

### Region Cache Configuration

Region caches are the primary caching mechanism. Configure them in `local.properties`:

```properties
# Main cache region (type system data, models)
cache.main=500000
cache.main.eviction=LRU
cache.main.ttl=3600

# Entity cache (individual item caching)
regioncache.entityregion.size=500000
regioncache.entityregion.eviction=LRU

# Query result cache
regioncache.queriesregion.size=100000
regioncache.queriesregion.eviction=LRU

# Type system cache
regioncache.typesystemregion.size=200000
regioncache.typesystemregion.eviction=LFU
```

### Cache Monitoring

Monitor cache hit rates via HAC (`Platform → Cache`) or JMX:

```
MBean: de.hybris.platform:type=Cache,name=RegionCacheAdapter
Attributes: HitCount, MissCount, HitRate, Size, Evictions
```

**Target hit rates:**
- Type system cache: 99%+
- Entity cache: 90%+
- Query cache: 70%+

If hit rates are below these thresholds, increase cache sizes or review your access patterns.

### Spring Cache for Custom Code

Use Spring's `@Cacheable` for your own service methods:

```java
@Cacheable(value = "productMetadataCache", key = "#productCode + '_' + #catalogVersion.pk")
public ProductMetadata getProductMetadata(String productCode, CatalogVersionModel catalogVersion) {
    // Expensive computation or external call
    return computeMetadata(productCode, catalogVersion);
}
```

Configure the cache in Spring:

```xml
<bean id="productMetadataCacheManager" class="org.springframework.cache.concurrent.ConcurrentMapCacheManager">
    <constructor-arg>
        <list>
            <value>productMetadataCache</value>
        </list>
    </constructor-arg>
</bean>
```

For distributed caching across cluster nodes, consider using the platform's `CacheRegion` API or an external cache like Redis.

### Cache Invalidation Strategies

The hardest problem in caching. SAP Commerce handles invalidation via:

1. **Automatic invalidation**: `ModelService.save()` and `remove()` invalidate the entity cache for the affected item
2. **Cluster-aware invalidation**: Cache invalidation events are broadcast to all cluster nodes via the cluster communication layer
3. **Time-based expiration**: TTL settings on cache regions
4. **Manual invalidation**: Call `cacheable.invalidateCache()` or clear specific cache regions

**Common pitfall**: Direct SQL updates (via `jdbcTemplate` or raw SQL) bypass the cache invalidation mechanism. Always use `ModelService` for writes unless you manually handle cache invalidation.

---

## Tomcat and Thread Pool Tuning

SAP Commerce runs on embedded Tomcat. Thread pool configuration directly impacts concurrent request handling.

### Thread Pool Configuration

```properties
# Maximum number of concurrent request processing threads
tomcat.maxthreads=200

# Minimum number of threads always kept alive
tomcat.minsparethreads=25

# Maximum queue length for incoming connections
tomcat.acceptcount=100

# Connection timeout in milliseconds
tomcat.connectiontimeout=60000

# Max number of connections
tomcat.maxconnections=10000
```

**Sizing guideline**: For a typical storefront, `tomcat.maxthreads` should be 200-400 for production. Set too low and requests queue up. Set too high and you overwhelm the database connection pool and cause memory pressure.

### Async Processing

Offload long-running operations to background threads:

```java
@Resource
private TaskService taskService;

public void processLargeOrderAsync(OrderModel order) {
    TaskModel task = modelService.create(TaskModel.class);
    task.setRunnerBean("orderProcessingTaskRunner");
    task.setContext(order.getPk().toString());
    task.setExecutionDate(new Date());
    modelService.save(task);
    // Returns immediately — task executes in background
}
```

### Session Management

Large HTTP sessions consume memory. Minimize session data:

```properties
# Session timeout (seconds)
default.session.timeout=3600

# Restrict session size
spring.session.store-type=none
```

---

## Solr Search Optimization

Solr powers product search and navigation. It's often the slowest part of product listing pages.

### Index Configuration

Reduce index size by only indexing what you need:

```xml
<!-- solr.impex -->
INSERT_UPDATE SolrIndexedProperty;solrIndexedType(identifier)[unique=true];name[unique=true];type(code);sortableType(code);fieldValueProvider;facet;facetType(code);multiValue
;$solrIndexedType;name;text;;;true;MultiSelectOr;false
;$solrIndexedType;code;string;;;false;;false
;$solrIndexedType;price;double;double;productPriceValueProvider;true;MultiSelectOr;false
;$solrIndexedType;category;string;;;true;Refine;true
;$solrIndexedType;inStockFlag;boolean;;;true;MultiSelectOr;false
```

**Don't index attributes you don't search or facet on.** Each indexed property increases index size, rebuild time, and query complexity.

### Indexing Performance

```properties
# Batch size for indexing
solrserver.default.indexer.batch.size=200

# Number of indexer threads
solrserver.default.indexer.thread.count=4

# Commit interval during indexing
solrserver.default.indexer.autocommit.maxtime=30000
```

### Query Performance

Enable Solr query caching:

```xml
<!-- solrconfig.xml customization -->
<queryResultCache class="solr.LRUCache" size="512" initialSize="256" autowarmCount="128"/>
<documentCache class="solr.LRUCache" size="4096" initialSize="1024" autowarmCount="512"/>
<filterCache class="solr.LRUCache" size="512" initialSize="256" autowarmCount="128"/>
```

**Limit facet calculations.** Each facet adds processing time. On mobile, consider showing fewer facets:

```java
searchQuery.setFacets(Arrays.asList("category", "brand", "priceRange")); // Only essential facets
// Instead of 15+ facets that desktop might show
```

### Incremental Indexing

Full reindexing is expensive. Use incremental indexing for real-time updates:

```java
// Instead of full index, update only changed products
solrIndexerService.performIndexOperation(indexedType, IndexerOperationValues.UPDATE, productPks);
```

Configure a CronJob for periodic incremental indexing:

```impex
INSERT_UPDATE CronJob;code[unique=true];job(code);sessionLanguage(isocode)
;solrIncrementalIndexCronJob;solrIncrementalIndexJob;en
```

---

## Catalog Synchronization Optimization

Catalog sync (Staged → Online) is one of the most resource-intensive operations in SAP Commerce.

### Incremental Sync

Never run full synchronization in production if incremental sync is possible:

```java
SyncConfig syncConfig = new SyncConfig();
syncConfig.setSynchronizationType(SyncConfig.INCREMENTAL);
syncConfig.setCreateSavedValues(false); // Skip audit trail for speed
syncConfig.setLogToDatabase(false);     // Skip DB logging
syncConfig.setLogToFile(false);         // Skip file logging
syncConfig.setForceUpdate(false);       // Only sync changed items

catalogSynchronizationService.synchronize(syncItemJob, syncConfig);
```

### Sync Performance Tuning

```properties
# Number of sync worker threads
catalog.sync.workers=4

# Batch size for sync operations
catalog.sync.batch.size=100

# Disable unnecessary hooks during sync
catalog.sync.enable.interceptors=false
```

### Scheduling Sync

Run full syncs during off-peak hours:

```impex
INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression
;productCatalogSyncCronJob;0 0 3 * * ?
```

---

## JVM Tuning

SAP Commerce is a memory-intensive application. JVM configuration significantly impacts performance.

### Memory Configuration

```properties
# In CCv2 manifest.json or local startup scripts
# Heap size — typically 4-8GB for production
-Xms4g
-Xmx8g

# Metaspace (class metadata) — SAP Commerce loads many classes
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=1g

# Young generation sizing
-XX:NewRatio=3
```

### Garbage Collection

For SAP Commerce workloads, G1GC is the recommended collector:

```properties
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45

# GC logging (for tuning)
-Xlog:gc*:file=/var/log/hybris/gc.log:time,uptime:filecount=10,filesize=100m
```

**Avoid** `-XX:+UseConcMarkSweepGC` (CMS) — it's deprecated since Java 9 and removed in Java 14.

### Key JVM Flags for SAP Commerce

```properties
# String deduplication (saves memory with many duplicate strings)
-XX:+UseStringDeduplication

# Optimize for large heaps
-XX:+AlwaysPreTouch

# Thread stack size (reduce if you have many threads)
-Xss512k

# Direct memory (for NIO operations)
-XX:MaxDirectMemorySize=512m
```

---

## Populator and Converter Optimization

The Populator/Converter pattern is elegant but can cause performance issues when chains are long and trigger lazy loading.

### The Problem: Death by a Thousand Populators

A single `productConverter.convert(productModel)` call might invoke 15+ populators, each accessing multiple model attributes, each potentially triggering a database query (lazy load):

```
productConverter.convert(product)
  ├── BasicProductPopulator         → accesses code, name, description
  ├── PricePopulator                → triggers price query
  ├── StockPopulator                → triggers stock query  
  ├── CategoryPopulator             → triggers category tree traversal
  ├── ImagePopulator                → triggers media queries
  ├── ReviewPopulator               → triggers review query
  ├── PromotionPopulator            → triggers promotion calculation
  ├── ClassificationPopulator       → triggers classification query
  └── ... (custom populators)
```

Each populator that accesses a lazy-loaded relation generates a database query. For a product listing page showing 20 products, this can mean 300+ database queries.

### Solutions

**1. Use `fieldSetLevelHelper` to conditionally skip populators:**

```java
public class PricePopulator implements Populator<ProductModel, ProductData> {
    @Resource
    private FieldSetLevelHelper fieldSetLevelHelper;
    
    @Override
    public void populate(ProductModel source, ProductData target) {
        if (fieldSetLevelHelper.isFieldIncluded("prices")) {
            // Only execute expensive price logic when explicitly requested
            target.setPrice(computePrice(source));
        }
    }
}
```

OCC API clients control which fields are populated via the `fields` query parameter:
```
GET /occ/v2/electronics/products/PROD-001?fields=code,name,price
```

**2. Batch-prefetch data before conversion:**

```java
public List<ProductData> convertProducts(List<ProductModel> products) {
    // Pre-fetch all prices in one query
    Map<String, PriceInformation> priceMap = priceService.getPricesForProducts(products);
    
    // Pre-fetch all stock levels in one query
    Map<String, StockData> stockMap = stockService.getStockForProducts(products);
    
    // Now convert — populators read from maps, not DB
    ThreadLocalContext.setPriceMap(priceMap);
    ThreadLocalContext.setStockMap(stockMap);
    
    return productConverter.convertAll(products);
}
```

**3. Create list-specific vs. detail-specific converters:**

```xml
<!-- Lightweight converter for listing pages -->
<bean id="productListConverter" parent="abstractPopulatingConverter">
    <property name="targetClass" value="com.mycompany.data.ProductData"/>
    <property name="populators">
        <list>
            <ref bean="basicProductPopulator"/>
            <ref bean="pricePopulator"/>
            <ref bean="imagePopulator"/>
        </list>
    </property>
</bean>

<!-- Full converter for PDP (product detail page) -->
<bean id="productDetailConverter" parent="abstractPopulatingConverter">
    <property name="targetClass" value="com.mycompany.data.ProductData"/>
    <property name="populators">
        <list>
            <ref bean="basicProductPopulator"/>
            <ref bean="pricePopulator"/>
            <ref bean="stockPopulator"/>
            <ref bean="imagePopulator"/>
            <ref bean="reviewPopulator"/>
            <ref bean="classificationPopulator"/>
            <ref bean="promotionPopulator"/>
        </list>
    </property>
</bean>
```

---

## Monitoring and Profiling

You can't optimize what you can't measure.

### Platform-Level Monitoring

**HAC Performance Monitoring:**
- `Platform → Cache` — Cache hit rates and sizes
- `Monitoring → Performance` — Request timing breakdown
- `Monitoring → Database` — Query statistics

**Key properties for monitoring:**

```properties
# Enable performance monitoring
hac.monitoring.enabled=true

# Log slow requests (>2 seconds)
monitoring.slowrequest.threshold=2000
monitoring.slowrequest.enabled=true

# Log slow queries (>500ms)
db.log.sql.slow=true
db.log.sql.slow.threshold=500
```

### JMX Monitoring

Expose key metrics via JMX for monitoring tools (Dynatrace, Datadog, Prometheus):

```properties
# Enable JMX
tomcat.jmx.port=9999
tomcat.jmx.enabled=true
```

Key MBeans to monitor:
- `de.hybris.platform:type=Cache` — cache statistics
- `java.lang:type=Memory` — heap usage
- `java.lang:type=GarbageCollector` — GC pause times
- `de.hybris.platform:type=Cluster` — cluster node health

### Application Performance Management (APM)

For production environments, integrate an APM tool:

```properties
# Dynatrace OneAgent (CCv2)
# Configured via CCv2 portal, not properties

# Generic Java agent
CATALINA_OPTS=-javaagent:/path/to/agent.jar
```

APM tools provide:
- Transaction tracing (see the full call stack per request)
- Database query breakdown
- Memory leak detection
- Exception tracking
- Response time percentiles (P50, P95, P99)

---

## CCv2-Specific Optimizations

When running on Commerce Cloud v2, additional considerations apply.

### Aspect Sizing

CCv2 uses "aspects" — predefined server configurations:

```json
{
  "aspects": [
    {
      "name": "backoffice",
      "properties": [
        { "key": "tomcat.maxthreads", "value": "100" },
        { "key": "db.pool.maxActive", "value": "50" }
      ]
    },
    {
      "name": "accstorefront",
      "properties": [
        { "key": "tomcat.maxthreads", "value": "400" },
        { "key": "db.pool.maxActive", "value": "200" }
      ]
    },
    {
      "name": "backgroundProcessing",
      "properties": [
        { "key": "tomcat.maxthreads", "value": "50" },
        { "key": "cronjob.maxthreads", "value": "8" }
      ]
    }
  ]
}
```

Separate background processing (CronJobs, imports, sync) from storefront traffic to prevent resource contention.

### Horizontal Scaling

CCv2 supports multiple instances per aspect. Request distribution is handled by the load balancer:

```json
{
  "aspects": [
    {
      "name": "accstorefront",
      "properties": [],
      "webapps": [
        { "name": "ycommercewebservices", "contextPath": "/occ" }
      ]
    }
  ]
}
```

Scale storefront aspects horizontally for traffic peaks. Scale backgroundProcessing aspects for batch operations.

### CDN and Static Resource Optimization

```properties
# Enable static resource versioning
storefront.resourceBundle.enabled=true

# Set aggressive caching for static resources
media.default.cache.control=public, max-age=31536000
```

Configure the CDN (Azure Front Door in CCv2) to cache:
- Product images
- Static JavaScript/CSS
- Media files

---

## Performance Checklist for Production Readiness

Use this checklist before go-live:

### Database
- [ ] All FlexibleSearch queries in DAOs use `SELECT {pk}` only
- [ ] No N+1 query patterns (verified via SQL logging)
- [ ] Indexes created for all custom types' commonly queried attributes
- [ ] Connection pool sized correctly (`maxActive ≥ tomcat.maxthreads * 1.5`)
- [ ] Slow query logging enabled (`db.log.sql.slow.threshold=500`)

### Caching
- [ ] Cache regions sized appropriately (monitor hit rates > 90%)
- [ ] Region cache sizes documented and justified
- [ ] Custom service methods use `@Cacheable` where appropriate
- [ ] Cache invalidation strategy defined for all cached data

### Application
- [ ] Populator chains optimized (list vs. detail converters)
- [ ] `fields` parameter used in OCC API calls to limit data
- [ ] Long-running operations offloaded to background tasks
- [ ] Session sizes minimized

### Solr
- [ ] Only necessary attributes indexed
- [ ] Incremental indexing configured for real-time updates
- [ ] Facet count limited on listing pages
- [ ] Solr cache configuration reviewed

### JVM
- [ ] Heap size set appropriately (4-8GB typical)
- [ ] G1GC configured with appropriate pause time target
- [ ] GC logging enabled for production debugging

### Infrastructure
- [ ] Storefront and background processing on separate aspects
- [ ] CDN configured for static resources and media
- [ ] Monitoring/APM tool integrated
- [ ] Load testing completed with production-like data volume

---

## Summary

Performance optimization in SAP Commerce is not a one-time activity — it's a continuous practice. The key principles:

1. **Measure first** — profile before optimizing. Use HAC, SQL logging, and APM tools to identify actual bottlenecks
2. **Database queries dominate** — fix N+1 patterns, add indexes, use `SELECT {pk}` only
3. **Caching is your best friend** — configure region caches, use Spring `@Cacheable`, monitor hit rates
4. **Populator chains are hidden costs** — use field sets, batch-prefetch data, create context-specific converters
5. **Separate concerns** — keep storefront traffic isolated from background processing
6. **JVM tuning matters** — right-size heap, use G1GC, monitor GC pauses
7. **Solr needs attention** — limit indexed properties, use incremental indexing, tune caches

The best-performing SAP Commerce deployments aren't the ones with the most hardware — they're the ones where developers understand the platform's internals and make informed architectural decisions from day one.