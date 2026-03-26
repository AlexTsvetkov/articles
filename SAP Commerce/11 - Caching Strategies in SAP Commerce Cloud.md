# Caching Strategies in SAP Commerce Cloud: From Type System to CDN

Caching is the single most impactful performance lever in SAP Commerce. A well-configured caching strategy can reduce page load times by 10x, cut database load by 90%, and let a single cluster handle traffic spikes that would otherwise require emergency scaling. A poorly configured one can serve stale data, cause cache stampedes, and create debugging nightmares.

This article covers every caching layer in the SAP Commerce stack — from the type system cache and region cache through to HTTP caching and CDN configuration — with practical configuration examples and troubleshooting guidance.

---

## The Caching Layers

SAP Commerce has multiple caching layers, each serving a different purpose:

```
┌────────────────────────────────────────────────────┐
│  CDN (Cloudflare, Akamai, CloudFront)              │  TTL: minutes–hours
│  Static assets, anonymous page responses            │
├────────────────────────────────────────────────────┤
│  HTTP Reverse Proxy (Varnish / CDN edge)           │  TTL: seconds–minutes
│  OCC API responses, page fragments                  │
├────────────────────────────────────────────────────┤
│  Application Server — Spring Cache                  │  TTL: configurable
│  Method-level caching for services                  │
├────────────────────────────────────────────────────┤
│  SAP Commerce Region Cache                          │  TTL: per-region
│  Entity cache, query cache, search cache            │
├────────────────────────────────────────────────────┤
│  Type System Cache                                  │  Permanent (until restart)
│  Type definitions, attribute metadata               │
├────────────────────────────────────────────────────┤
│  Database Query Cache (MySQL/HANA)                  │  Managed by DB engine
│  SQL result caching                                 │
└────────────────────────────────────────────────────┘
```

---

## Type System Cache

The type system cache holds the metadata about all item types, attributes, relations, and enumerations. It's loaded at startup and stays in memory.

### How It Works

When SAP Commerce starts, it reads the entire type system definition from the database and builds an in-memory representation. Every `modelService.get()`, every FlexibleSearch query, every ImpEx import uses this cache to understand the data model.

### Configuration

```properties
# Type system cache is not configurable per se — it's always on.
# But you can control the startup behavior:

# Force type system rebuild on startup (use after items.xml changes)
typesystem.cache.validate=true

# Log type system cache statistics
log4j2.logger.typesystem.name = de.hybris.platform.persistence.type
log4j2.logger.typesystem.level = DEBUG
```

### When It Causes Problems

The type system cache only causes issues during development when you change `items.xml` and forget to run `ant updatesystem`. Symptoms include:

- `UnknownIdentifierException` for types you just added
- Missing attributes on existing types
- Stale enum values

**Fix**: Always run `ant updatesystem` after changing `items.xml`, then restart.

---

## Region Cache

The region cache is the primary application-level cache in SAP Commerce. It caches entities, queries, and other frequently accessed data in memory.

### Cache Regions

SAP Commerce defines several cache regions:

| Region | Purpose | Default Size | Typical Hit Rate |
|--------|---------|-------------|------------------|
| `entityCacheRegion` | Item model instances | 50,000 | 80-95% |
| `queryCacheRegion` | FlexibleSearch results | 10,000 | 60-85% |
| `typesystemCacheRegion` | Type definitions | 5,000 | ~100% |
| `entityEnumCacheRegion` | Enum value lookups | 5,000 | ~100% |
| `catalogVersionsCacheRegion` | Catalog version data | 1,000 | ~100% |

### Configuration

```properties
# local.properties or project.properties

# Entity cache — the most important cache
regioncache.entityCacheRegion.size=100000
regioncache.entityCacheRegion.evictionpolicy=LRU
regioncache.entityCacheRegion.statsEnabled=true

# Query cache — caches FlexibleSearch results
regioncache.queryCacheRegion.size=20000
regioncache.queryCacheRegion.evictionpolicy=LRU
regioncache.queryCacheRegion.statsEnabled=true

# Enable cache statistics (monitor via HAC)
cache.main.regioncache.stats=true
```

### Cache Invalidation

Cache invalidation happens automatically when items are modified through the SAP Commerce API:

```java
// This automatically invalidates the cache for this product
modelService.save(productModel);

// This invalidates the cache for the removed item
modelService.remove(productModel);

// ImpEx imports also trigger cache invalidation
```

### Cluster Cache Invalidation

In a multi-node cluster, cache invalidation must propagate across all nodes. SAP Commerce uses a cache invalidation topic:

```properties
# Cluster cache sync configuration
cluster.node.stale.timeout=60000
cluster.broadcast.method.jgroups=jgroups-tcp.xml

# CCv2 uses its own cluster discovery — don't override
# But ensure broadcast is working:
cluster.broadcast.methods=jgroups
```

When node A saves a product, it broadcasts an invalidation message. Nodes B, C, and D receive this message and evict the product from their local caches.

### Monitoring Cache Performance

Access cache statistics via HAC → Monitoring → Cache:

```
Cache Region Statistics:
┌───────────────────────┬───────────┬──────────┬───────────┬──────────┐
│ Region                 │ Size      │ Hit Rate │ Hits      │ Misses   │
├───────────────────────┼───────────┼──────────┼───────────┼──────────┤
│ entityCacheRegion      │ 87,432    │ 94.2%    │ 2,341,897 │ 142,103  │
│ queryCacheRegion       │ 15,678    │ 78.5%    │ 892,456   │ 244,332  │
│ typesystemCacheRegion  │ 3,245     │ 99.9%    │ 567,890   │ 45       │
│ enumCacheRegion        │ 1,234     │ 99.8%    │ 234,567   │ 123      │
└───────────────────────┴───────────┴──────────┴───────────┴──────────┘
```

**Target hit rates**: Entity cache >90%, query cache >70%. If entity cache hit rate drops below 80%, increase the cache size or investigate which items are being loaded repeatedly.

---

## FlexibleSearch Query Cache

FlexibleSearch queries are cached at two levels: the query plan cache and the result cache.

### Query Plan Cache

Compiled query plans are cached to avoid re-parsing SQL on every execution:

```properties
# Query plan cache (parsed SQL → compiled query)
flexiblesearch.cache.size=10000
flexiblesearch.cache.enabled=true
```

### Result Cache

Query results can be cached based on the query string and parameters:

```java
// This query's results are cached
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Product} WHERE {code} = ?code");
query.addQueryParameter("code", "CAM-001");
query.setCacheable(true);  // Enable result caching for this query

SearchResult<ProductModel> result = flexibleSearchService.search(query);
```

### When Query Cache Hurts

The query cache stores complete result sets. For queries that return large result sets or queries that are rarely repeated with the same parameters, caching wastes memory:

```java
// BAD: Don't cache queries with unique parameters
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Order} WHERE {code} = ?code");
query.addQueryParameter("code", uniqueOrderCode);
query.setCacheable(true);  // Wastes cache space — each order code is unique

// GOOD: Cache queries with reusable parameters
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Product} WHERE {approvalStatus} = ?status AND {catalogVersion} = ?cv");
query.addQueryParameter("status", ApprovalStatus.APPROVED);
query.addQueryParameter("cv", onlineCatalogVersion);
query.setCacheable(true);  // Good — this query is reused frequently
```

---

## Solr Cache Configuration

If you use Solr for product search, its caches significantly impact search performance.

### Solr Cache Types

```xml
<!-- solrconfig.xml -->
<config>
  <!-- Filter cache: caches filter queries (category, facet, stock status) -->
  <filterCache class="solr.CaffeineCache"
               size="4096"
               initialSize="1024"
               autowarmCount="512"/>

  <!-- Query result cache: caches ordered result sets -->
  <queryResultCache class="solr.CaffeineCache"
                    size="2048"
                    initialSize="512"
                    autowarmCount="256"/>

  <!-- Document cache: caches stored fields for documents -->
  <documentCache class="solr.CaffeineCache"
                 size="8192"
                 initialSize="2048"/>
</config>
```

### Solr Cache Warming

After indexing, Solr needs to warm its caches for the new searcher:

```xml
<listener event="newSearcher" class="solr.QuerySenderListener">
  <arr name="queries">
    <!-- Pre-warm with common category queries -->
    <lst>
      <str name="q">*:*</str>
      <str name="fq">catalogVersion:Online</str>
      <str name="sort">score desc</str>
      <str name="rows">10</str>
    </lst>
  </arr>
</listener>
```

### Solr Cache Monitoring

```bash
# Check Solr cache statistics via admin API
curl "http://localhost:8983/solr/electronics_Product/admin/mbeans?cat=CACHE&stats=true&wt=json" | jq '.["solr-mbeans"]'
```

Key metrics to watch:
- **Filter cache hit rate**: Should be >85% for production traffic
- **Query result cache hit rate**: Should be >60%
- **Evictions**: High eviction rate means cache is too small

---

## HTTP-Level Caching

### OCC API Response Caching

For the headless storefront (Spartacus), cache OCC API responses at the HTTP layer:

```java
@GetMapping("/products/{productCode}")
public ResponseEntity<ProductWsDTO> getProduct(
        @PathVariable String productCode,
        @RequestParam(defaultValue = DEFAULT_FIELD_SET) String fields) {
    
    ProductData data = productFacade.getProductForCode(productCode);
    ProductWsDTO dto = dataMapper.map(data, ProductWsDTO.class, fields);
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl
            .maxAge(5, TimeUnit.MINUTES)
            .staleWhileRevalidate(30, TimeUnit.SECONDS))
        .eTag(generateETag(dto))
        .body(dto);
}
```

### Cache-Control Headers by Resource Type

| Resource | Cache-Control | Rationale |
|----------|--------------|-----------|
| Product detail | `max-age=300, stale-while-revalidate=30` | Changes infrequently |
| Product list/search | `max-age=60` | Moderate change frequency |
| Cart | `no-store` | User-specific, never cache |
| Checkout | `no-store` | User-specific, never cache |
| Static assets (JS, CSS) | `max-age=31536000, immutable` | Content-hashed filenames |
| Product images | `max-age=86400` | Rarely change |
| CMS content | `max-age=600, stale-while-revalidate=60` | Updated by business users |

### Conditional Requests (ETags)

ETags enable efficient cache revalidation:

```java
@GetMapping("/categories/{categoryCode}/products")
public ResponseEntity<ProductSearchPageWsDTO> searchProducts(
        @PathVariable String categoryCode,
        HttpServletRequest request) {
    
    // Generate ETag from content hash
    ProductSearchPageData results = searchFacade.categorySearch(categoryCode);
    String etag = DigestUtils.md5Hex(results.hashCode() + "");
    
    // Check If-None-Match header
    if (etag.equals(request.getHeader("If-None-Match"))) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();
    }
    
    ProductSearchPageWsDTO dto = dataMapper.map(results, ProductSearchPageWsDTO.class);
    return ResponseEntity.ok()
        .eTag(etag)
        .cacheControl(CacheControl.maxAge(1, TimeUnit.MINUTES))
        .body(dto);
}
```

---

## CDN Configuration

For production deployments, a CDN layer caches responses close to users geographically.

### CDN Caching Rules

```
CDN Configuration (e.g., Cloudflare, Akamai):

┌─────────────────────────────┬────────────┬────────────────────┐
│ URL Pattern                  │ CDN TTL    │ Strategy           │
├─────────────────────────────┼────────────┼────────────────────┤
│ /occ/v2/*/products/*        │ 5 min      │ Cache, honor origin│
│ /occ/v2/*/categories        │ 10 min     │ Cache, honor origin│
│ /occ/v2/*/cms/pages/*       │ 10 min     │ Cache, honor origin│
│ /occ/v2/*/users/*           │ 0 (bypass) │ Never cache        │
│ /occ/v2/*/cart*             │ 0 (bypass) │ Never cache        │
│ /occ/v2/*/orders*           │ 0 (bypass) │ Never cache        │
│ /medias/*                   │ 24 hours   │ Cache aggressively │
│ /*.js, /*.css               │ 1 year     │ Immutable          │
│ /*.html (SSR pages)         │ 1 min      │ Stale-while-reval  │
└─────────────────────────────┴────────────┴────────────────────┘
```

### Cache Key Design

The CDN cache key determines what makes two requests "the same." Include:

- **URL path**: Always included
- **Query parameters**: Include `fields`, `lang`, `curr`. Exclude tracking parameters.
- **Headers**: Include `Accept-Language` for multi-language sites
- **Cookies**: Exclude from cache key for anonymous requests

```
# Cloudflare page rule example
Match: /occ/v2/*/products/*
Cache Level: Cache Everything
Edge Cache TTL: 300
Browser Cache TTL: 60
Cache Key: URL + query string (filtered) + Accept-Language header
Bypass: If Cookie contains "access_token"
```

### Cache Purging

When product data changes, purge the CDN cache:

```java
@Component
public class CDNCachePurger implements AfterSaveListener {
    
    @Override
    public void afterSave(Collection<AfterSaveEvent> events) {
        for (AfterSaveEvent event : events) {
            if (isProductRelated(event)) {
                String productCode = getProductCode(event);
                purgeProductFromCDN(productCode);
            }
        }
    }
    
    private void purgeProductFromCDN(String productCode) {
        // Purge specific product URL patterns from CDN
        List<String> urlsToPurge = List.of(
            "/occ/v2/*/products/" + productCode + "*",
            "/medias/*" + productCode + "*"
        );
        cdnClient.purge(urlsToPurge);
    }
}
```

---

## Cache Anti-Patterns

### 1. Caching User-Specific Data

```java
// WRONG: This caches per-user data with a shared key
@Cacheable("priceCache")
public PriceData getPrice(String productCode) {
    // Returns user-specific price (based on contract, user group, etc.)
    return priceFacade.getPrice(productCode);
}

// RIGHT: Include user-identifying information in cache key
@Cacheable(value = "priceCache", key = "#productCode + '-' + #userGroup")
public PriceData getPrice(String productCode, String userGroup) {
    return priceFacade.getPrice(productCode);
}
```

### 2. Unbounded Cache Growth

```properties
# WRONG: No size limit
regioncache.entityCacheRegion.size=0

# RIGHT: Set appropriate limits based on available memory
# Rule of thumb: entity cache size ≈ unique items accessed in a typical hour
regioncache.entityCacheRegion.size=100000
```

### 3. Too-Long TTL on Dynamic Content

```java
// WRONG: Caching stock levels for an hour
return ResponseEntity.ok()
    .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS))
    .body(stockData);

// RIGHT: Short TTL or no cache for volatile data
return ResponseEntity.ok()
    .cacheControl(CacheControl.maxAge(30, TimeUnit.SECONDS))
    .body(stockData);
```

### 4. Cache Stampede

When a cached item expires and many requests arrive simultaneously, they all miss the cache and hit the database at once.

```java
// Solution: stale-while-revalidate
return ResponseEntity.ok()
    .cacheControl(CacheControl
        .maxAge(5, TimeUnit.MINUTES)
        .staleWhileRevalidate(60, TimeUnit.SECONDS))  // Serve stale while refreshing
    .body(dto);

// Solution: Cache warming
@Scheduled(fixedRate = 240000)  // Every 4 minutes (before 5-min TTL expires)
public void warmProductCache() {
    List<String> topProductCodes = analyticsService.getTopProductCodes(100);
    for (String code : topProductCodes) {
        productFacade.getProductForCode(code);  // Refreshes cache
    }
}
```

---

## Monitoring and Tuning

### Key Metrics to Watch

```
Cache Health Dashboard:
┌──────────────────────────────────────────────────┐
│ Entity Cache                                      │
│   Hit Rate: 94.2% ████████████████████░░ (Target: >90%)
│   Size: 87,432 / 100,000 (87% full)              │
│   Evictions/min: 45                               │
├──────────────────────────────────────────────────┤
│ Query Cache                                       │
│   Hit Rate: 72.1% ██████████████░░░░░░░ (Target: >70%)
│   Size: 15,678 / 20,000 (78% full)               │
│   Evictions/min: 120                              │
├──────────────────────────────────────────────────┤
│ CDN Cache                                         │
│   Hit Rate: 88.5% █████████████████░░░░ (Target: >85%)
│   Bandwidth saved: 4.2 TB/day                     │
│   Origin requests: 1.2M/day (down from 10.8M)    │
├──────────────────────────────────────────────────┤
│ Solr Filter Cache                                 │
│   Hit Rate: 91.3% ██████████████████░░░ (Target: >85%)
│   Evictions: 234 (since last commit)              │
└──────────────────────────────────────────────────┘
```

### Tuning Process

1. **Enable statistics** on all cache regions
2. **Baseline** the current hit rates and sizes
3. **Identify low-hit-rate caches** — increase size if evictions are high, or disable caching if content is too unique
4. **Identify high-eviction caches** — increase size or switch eviction policy
5. **Monitor after changes** — give it at least a full traffic cycle (24 hours) before evaluating
6. **Profile cache contents** — use HAC to inspect what's actually in the cache

---

## Summary

Effective caching in SAP Commerce requires understanding and tuning every layer:

1. **Type system cache** — automatic, loaded at startup, rarely needs attention
2. **Region cache** — the workhorse cache for entities and queries; size it based on your data volume and monitor hit rates
3. **FlexibleSearch cache** — enable for reusable queries, disable for unique-parameter queries
4. **Solr cache** — tune filter and query result caches based on search traffic patterns
5. **HTTP caching** — set appropriate Cache-Control headers per resource type; never cache user-specific data
6. **CDN caching** — cache static assets aggressively, API responses conservatively, and never cache authenticated content
7. **Monitor continuously** — cache performance degrades as data grows and traffic patterns change

The goal isn't maximum caching — it's the right caching. Cache the data that's expensive to compute and frequently requested, with TTLs that balance freshness against performance. And always have a way to invalidate when the source data changes.