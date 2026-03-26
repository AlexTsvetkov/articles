# Troubleshooting SAP Commerce in Production: A Practitioner's Guide

Production issues in SAP Commerce don't announce themselves politely. They arrive as vague alerts, customer complaints, or a sudden spike in error rates at the worst possible time. The difference between a 15-minute resolution and a 4-hour outage comes down to how quickly you can identify the root cause, and that requires knowing where to look and what tools to use.

This article is a field guide for diagnosing and resolving the most common production issues in SAP Commerce Cloud: memory problems, slow queries, thread deadlocks, cache issues, CronJob failures, and deployment errors.

---

## Diagnostic Toolkit

Before diving into specific problems, know your tools:

```
┌──────────────────────────────────┬────────────────────────────────────┐
│ Tool                              │ Use For                            │
├──────────────────────────────────┼────────────────────────────────────┤
│ HAC (Administration Console)      │ Cache stats, FlexibleSearch,       │
│                                   │ ImpEx, CronJobs, Spring config     │
├──────────────────────────────────┼────────────────────────────────────┤
│ Cloud Portal (CCv2)               │ Deployment logs, pod status,       │
│                                   │ scaling, environment config        │
├──────────────────────────────────┼────────────────────────────────────┤
│ Dynatrace / Application Logs      │ Request tracing, error rates,      │
│                                   │ JVM metrics, slow transactions     │
├──────────────────────────────────┼────────────────────────────────────┤
│ Thread Dumps                      │ Deadlocks, blocked threads,        │
│                                   │ thread pool exhaustion             │
├──────────────────────────────────┼────────────────────────────────────┤
│ Heap Dumps                        │ Memory leaks, large object         │
│                                   │ retention, GC pressure             │
├──────────────────────────────────┼────────────────────────────────────┤
│ FlexibleSearch in HAC             │ Data integrity checks, count       │
│                                   │ queries, orphan detection          │
├──────────────────────────────────┼────────────────────────────────────┤
│ Kibana / Log Aggregation          │ Log search, error correlation,     │
│                                   │ timeline reconstruction            │
└──────────────────────────────────┴────────────────────────────────────┘
```

---

## Problem 1: OutOfMemoryError

### Symptoms
- Application pods restarting repeatedly
- `java.lang.OutOfMemoryError: Java heap space` in logs
- Increasing response times before the crash

### Immediate Response

```bash
# 1. Check current memory usage in CCv2 Cloud Portal
#    Navigate to: Environments → [env] → Monitoring

# 2. If you have access, trigger a heap dump before restart
jmap -dump:format=b,file=/tmp/heapdump.hprof <pid>

# 3. Check recent deployments or configuration changes
#    Was anything deployed in the last 24 hours?
```

### Common Causes

**Large ImpEx imports consuming memory:**

```
# Check for running imports in HAC → ImpEx → Import
# Large imports without batching can consume gigabytes

# Fix: Split large imports into batches
# Before: Single 2GB import file
# After: Multiple files, 50,000 lines each
```

**Catalog synchronization on large catalogs:**

```properties
# Catalog sync loads products into memory
# For catalogs with 500k+ products, increase memory or optimize sync

# Reduce sync batch size
catalog.sync.workers=4
synchronization.itemcopycreator.batchSize=100
```

**Unbounded FlexibleSearch results:**

```java
// PROBLEM: Loading all products into memory
String query = "SELECT {pk} FROM {Product}";
List<ProductModel> allProducts = flexibleSearchService.search(query).getResult();
// With 500,000 products, this consumes massive heap

// FIX: Always paginate
FlexibleSearchQuery fsq = new FlexibleSearchQuery(query);
fsq.setStart(0);
fsq.setCount(100);  // Process in pages
```

**Session data accumulation:**

```properties
# Sessions storing too much data (cart calculations, comparison lists)
# Check session sizes in Dynatrace or via JMX

# Reduce session timeout for anonymous users
default.session.timeout=600

# Store large session data externally (Redis) rather than in-memory
```

### Analyzing Heap Dumps

```bash
# Download heap dump from pod
kubectl cp commerce-pod:/tmp/heapdump.hprof ./heapdump.hprof

# Open with Eclipse MAT (Memory Analyzer Tool)
# Key reports:
# 1. Leak Suspects Report — shows objects retaining the most memory
# 2. Dominator Tree — shows largest objects by retained size
# 3. Top Consumers — aggregate view by class

# Common findings:
# - Large HashMap instances (session data)
# - List<ProductModel> with millions of entries (unbounded queries)
# - byte[] arrays (media processing without streaming)
```

---

## Problem 2: Slow Response Times

### Symptoms
- Page load times exceeding SLA thresholds
- Timeout errors on storefront
- Database CPU running high

### Diagnosis Flow

```
Slow response reported
        │
        ▼
Is it all pages or specific pages?
        │
   ┌────┴────┐
   │ All     │ Specific
   │         │
   ▼         ▼
Check DB    Check page-specific
load,       queries and logic
cache hit   (product detail,
rates,      search, category)
JVM GC
```

### Database Slow Queries

```sql
-- Find slow queries (MySQL)
SELECT query_time, sql_text 
FROM mysql.slow_log 
WHERE start_time > NOW() - INTERVAL 1 HOUR
ORDER BY query_time DESC 
LIMIT 20;

-- Check for missing indexes
EXPLAIN SELECT * FROM products 
WHERE p_code = 'CAM-001' 
AND TypePkString = 8796093382738;
-- Look for: type=ALL (full table scan) → needs index
```

**Common slow query patterns:**

```java
// PROBLEM: N+1 queries — loading products, then loading prices one by one
for (ProductModel product : products) {
    PriceInformation price = priceService.getWebPriceForProduct(product);
    // This executes a separate query for EACH product
}

// FIX: Batch load or use a single query with JOIN
String query = "SELECT {p.pk}, {pr.price} FROM {Product AS p "
    + "JOIN PriceRow AS pr ON {pr.product} = {p.pk}} "
    + "WHERE {p.catalogVersion} = ?cv";
```

### Cache Miss Investigation

```
In HAC → Monitoring → Cache:

If entity cache hit rate < 80%:
  → Cache is too small for the working set
  → Increase regioncache.entityCacheRegion.size

If query cache hit rate < 50%:
  → Queries use unique parameters (session IDs, timestamps)
  → Review FlexibleSearch queries for cacheability

If cache size is at maximum with high eviction rate:
  → Working set is larger than cache
  → Increase cache size or reduce loaded items per request
```

### Solr Search Slowness

```bash
# Check Solr query performance
curl "http://solr-host:8983/solr/admin/metrics?group=core&prefix=QUERY" | jq .

# Common issues:
# 1. Too many facets computed per query
# 2. Large result sets with highlighting enabled
# 3. Complex filter queries without caching
# 4. Index too large for allocated memory
```

---

## Problem 3: Thread Deadlocks and Pool Exhaustion

### Symptoms
- Requests hanging indefinitely
- Thread pool metrics showing 100% utilization
- `RejectedExecutionException` in logs

### Taking Thread Dumps

```bash
# Via jstack (if you have pod access)
jstack <pid> > thread_dump_$(date +%s).txt

# Take 3 dumps, 10 seconds apart, to see thread progression
for i in 1 2 3; do
  jstack <pid> > thread_dump_${i}.txt
  sleep 10
done

# Via HAC → Monitoring → Thread Dump
# Downloads a formatted thread dump
```

### Reading Thread Dumps

```
# Look for BLOCKED threads
"http-nio-9002-exec-42" #142 daemon prio=5
   java.lang.Thread.State: BLOCKED (on object monitor)
    at de.hybris.platform.persistence.GenericBMPBean.ejbLoad(GenericBMPBean.java:456)
    - waiting to lock <0x00000007f8e45678> (a java.lang.Object)
    - locked by "http-nio-9002-exec-17" #117

# This thread is blocked waiting for a lock held by thread exec-17
# Check what exec-17 is doing — likely stuck in a long database operation

# Look for deadlocks (two threads each waiting for the other's lock)
"Found one Java-level deadlock:"
  Thread 1 waiting for lock held by Thread 2
  Thread 2 waiting for lock held by Thread 1
```

### Common Thread Issues

**Database connection pool exhaustion:**

```properties
# Symptom: Threads waiting for database connections
# Check: All DB connections are in use, new requests queue up

# Increase pool size (default is often 10-20)
db.pool.maxActive=50
db.pool.maxIdle=20
db.pool.minIdle=5

# Add connection validation to reclaim leaked connections
db.pool.testOnBorrow=true
db.pool.validationQuery=SELECT 1

# Set max wait time to fail fast rather than queue indefinitely
db.pool.maxWait=5000
```

**Long-running transactions blocking other threads:**

```java
// PROBLEM: Transaction held open during external API call
@Transactional
public void processOrder(OrderModel order) {
    orderService.updateStatus(order, OrderStatus.PROCESSING);
    
    // This external call takes 30 seconds during peak load
    // The transaction stays open, holding database locks
    paymentGateway.charge(order.getPaymentInfo());
    
    orderService.updateStatus(order, OrderStatus.PAID);
}

// FIX: Separate the external call from the transaction
public void processOrder(OrderModel order) {
    updateOrderStatus(order, OrderStatus.PROCESSING);
    
    PaymentResult result = paymentGateway.charge(order.getPaymentInfo());
    
    if (result.isSuccess()) {
        updateOrderStatus(order, OrderStatus.PAID);
    }
}

@Transactional
private void updateOrderStatus(OrderModel order, OrderStatus status) {
    orderService.updateStatus(order, status);
}
```

---

## Problem 4: CronJob Failures

### Symptoms
- Stale data (products not synced, search index outdated)
- `CronJob stuck in RUNNING state` in HAC
- Missing order exports or inventory updates

### Diagnosing CronJob Issues

```
HAC → System → CronJobs:

Check:
1. Last execution time — is it running at all?
2. Status — RUNNING for too long = stuck
3. Result — FAILURE with error message
4. Logs — node-specific log files
```

### Stuck CronJob Recovery

```impex
# Option 1: Abort via HAC → CronJobs → select job → Abort

# Option 2: Reset via ImpEx if HAC abort doesn't work
UPDATE CronJob;code[unique=true];status(code);requestAbort
;solrIndexerJob;ABORTED;true

# Option 3: For truly stuck jobs, update the trigger
UPDATE Trigger;cronJob(code)[unique=true];active
;solrIndexerJob;false

# Then re-enable after fixing the root cause
UPDATE Trigger;cronJob(code)[unique=true];active
;solrIndexerJob;true
```

### Common CronJob Problems

**Catalog sync timeout:**

```properties
# Large catalog sync exceeds default timeouts
# Increase timeout and reduce batch size

catalog.sync.workers=2
synchronization.itemcopycreator.batchSize=50

# For CCv2: Ensure the CronJob runs on a background processing node
# not on a node serving customer traffic
```

**Solr indexing failures:**

```
Typical errors:
1. "Connection refused" → Solr pod is down or restarting
   Fix: Check Solr pod status, restart if needed

2. "OutOfMemoryError during indexing" → Too many products indexed at once
   Fix: Reduce batch size in Solr indexer configuration

3. "Document contains invalid characters" → Bad product data
   Fix: Find and fix the problematic product data
   
   -- Find products with problematic characters
   SELECT {code}, {name} FROM {Product} 
   WHERE {name} LIKE '%\x00%' OR {description} LIKE '%\x00%'
```

---

## Problem 5: Deployment Failures on CCv2

### Symptoms
- Build succeeds but deployment fails
- Pod crashes immediately after startup
- Health checks failing

### Debugging Deployment Failures

```
CCv2 Cloud Portal → Builds:

1. Check build logs for compilation errors
2. Check deployment logs for startup errors
3. Check pod logs for runtime exceptions

Common failure patterns:
┌─────────────────────────┬──────────────────────────────────────┐
│ Error                    │ Cause                                │
├─────────────────────────┼──────────────────────────────────────┤
│ Build timeout            │ Too many extensions, slow network    │
│ OOM during build         │ Increase build memory in manifest    │
│ Type system update fail  │ Incompatible items.xml changes       │
│ Bean definition error    │ Missing Spring dependency            │
│ Health check timeout     │ Slow startup, increase timeout       │
│ License validation fail  │ License expired or misconfigured     │
└─────────────────────────┴──────────────────────────────────────┘
```

### Manifest Configuration Issues

```json
// manifest.json — common misconfigurations

{
  "commerceSuiteVersion": "2211.25",
  "extensions": [
    // PROBLEM: Extension listed but not in repository
    "myextension",
    // PROBLEM: Dependency extension missing
    // myextension requires basecommerce but it's not listed
  ],
  "aspects": [
    {
      "name": "backoffice",
      "properties": [
        {
          "key": "db.pool.maxActive",
          "value": "50"
          // PROBLEM: Value too high for pod memory allocation
          // 50 connections × ~5MB each = 250MB just for DB connections
        }
      ]
    }
  ]
}
```

### Rolling Back a Failed Deployment

```
CCv2 Cloud Portal → Deployments:

1. Identify the last working build number
2. Create new deployment using the previous build
3. Deploy to the affected environment

Important: If the failed deployment included database changes
(new types, removed attributes), the rollback build may need
to handle the modified schema. Test rollback scenarios before
production deployments.
```

---

## Problem 6: Data Inconsistency

### Symptoms
- Products visible in Backoffice but not on storefront
- Prices showing incorrectly
- Categories missing products

### Diagnosis Checklist

```
Product not visible on storefront:

□ Is the product in the ONLINE catalog version?
  SELECT {cv.version} FROM {Product AS p 
  JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}}
  WHERE {p.code} = 'PROD-001'

□ Is the approval status "approved"?
  SELECT {approvalStatus} FROM {Product} WHERE {code} = 'PROD-001'

□ Does the product have a price?
  SELECT {price} FROM {PriceRow} WHERE {product} = (
    SELECT {pk} FROM {Product} WHERE {code} = 'PROD-001')

□ Is the product indexed in Solr?
  curl "http://solr:8983/solr/products/select?q=code_string:PROD-001"

□ Has the catalog sync run recently?
  Check last sync time in HAC → System → CronJobs

□ Is the product assigned to a category in the site's catalog?
  SELECT {c.code} FROM {Category AS c 
  JOIN CategoryProductRelation AS r ON {r.source} = {c.pk}}
  WHERE {r.target} = (SELECT {pk} FROM {Product} WHERE {code} = 'PROD-001')
```

### Fixing Catalog Sync Issues

```impex
# Force re-sync of specific products
# 1. Touch the product in Staged catalog (triggers sync)
UPDATE Product;code[unique=true];catalogVersion(catalog(id),version)[unique=true];description
;PROD-001;myProductCatalog:Staged;Updated description (triggers sync)

# 2. Run catalog sync
# HAC → System → CronJobs → Start sync CronJob

# 3. Verify product in Online catalog
# HAC → FlexibleSearch:
SELECT {p.code}, {p.name}, {cv.version} FROM {Product AS p 
JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}}
WHERE {p.code} = 'PROD-001'
```

---

## Problem 7: Cluster Synchronization Issues

### Symptoms
- Changes made on one node not visible on others
- Inconsistent behavior between requests (load balancer routing to different nodes)
- Stale cache data despite modifications

### Diagnosis

```properties
# Check cluster node status in HAC → Platform → Cluster

# Verify cluster communication
# All nodes should show as "ALIVE" with recent heartbeat

# If nodes show as "STALE" or "DEAD":
# 1. Check network connectivity between pods
# 2. Verify JGroups configuration
# 3. Check if cluster broadcast is enabled

cluster.broadcast.methods=jgroups
cluster.broadcast.method.jgroups.channel.name=hybris-broadcast
```

### Cache Invalidation Verification

```java
// Test: modify an item and check if other nodes see the change
// On Node 1:
ProductModel product = productService.getProductForCode("TEST-001");
product.setName("Updated Name");
modelService.save(product);

// On Node 2 (via different request):
ProductModel product = productService.getProductForCode("TEST-001");
// If name is still old → cache invalidation is not working
// If name is updated → cluster sync is working
```

---

## Runbook Template

For each known issue, maintain a runbook:

```markdown
## Issue: [Name]

### Severity: P1/P2/P3

### Symptoms
- What does the user/monitoring see?

### Diagnosis Steps
1. Check [specific metric/log/dashboard]
2. Run [specific query/command]
3. Look for [specific pattern]

### Resolution
1. Immediate fix: [steps]
2. Permanent fix: [steps]

### Escalation
- If not resolved in [X] minutes, escalate to [team/person]

### Prevention
- What monitoring should catch this earlier?
- What code/config change prevents recurrence?
```

---

## Best Practices

1. **Monitor proactively** — set alerts for memory usage >80%, cache hit rates <70%, error rates above baseline, and CronJob failures.

2. **Keep thread dumps and heap dumps accessible** — configure the JVM to dump on OOM (`-XX:+HeapDumpOnOutOfMemoryError`) and know how to take thread dumps quickly.

3. **Maintain runbooks** — document every production issue and its resolution. The next incident at 3 AM shouldn't require the same diagnosis from scratch.

4. **Test at production scale** — issues that appear with 500,000 products and 10,000 concurrent users don't show up in dev environments.

5. **Deploy during low-traffic windows** — CCv2 supports rolling deployments, but having fewer users during deployment reduces the blast radius of issues.

6. **Know the rollback process** — practice rolling back deployments before you need to do it under pressure.

7. **Separate background processing from customer-facing traffic** — CronJobs, catalog sync, and indexing should run on dedicated nodes, not on nodes serving API requests.

---

## Summary

Production troubleshooting in SAP Commerce follows a consistent pattern:

1. **Identify the symptom** — error logs, monitoring alerts, user reports
2. **Isolate the layer** — is it database, cache, application, or infrastructure?
3. **Gather evidence** — thread dumps, heap dumps, slow query logs, cache stats
4. **Apply the fix** — targeted resolution based on root cause
5. **Verify** — confirm the fix resolves the symptom
6. **Document** — update the runbook for next time

The best teams don't just fix issues — they build monitoring, automation, and documentation so that the same issue is either prevented or resolved faster the next time.