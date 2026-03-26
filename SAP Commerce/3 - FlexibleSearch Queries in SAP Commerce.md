{% raw %}
# FlexibleSearch Queries in SAP Commerce: The Complete Developer's Guide

Every SAP Commerce developer writes FlexibleSearch queries daily. It's how you retrieve products, look up orders, debug data issues in HAC, and build the DAOs that power your services. Yet many developers treat it as "just SQL with curly braces" and miss the nuances that separate a query that works from one that performs well at scale.

This guide covers FlexibleSearch from first principles to production-hardened patterns. Every concept is illustrated with a working query, and we dig into the internals — how the engine translates your query, how caching works, and where the performance traps hide.

---

## What Is FlexibleSearch?

FlexibleSearch is SAP Commerce's query language for retrieving data from the type system. It looks like SQL but operates on the platform's type model rather than raw database tables. When you write:

```sql
SELECT {pk} FROM {Product}
```

You're not querying a `products` table directly. The FlexibleSearch engine (`de.hybris.platform.jalo.flexiblesearch.FlexibleSearch`) translates this into actual SQL by:

1. Resolving `{Product}` to the correct database table(s), including joined tables for subtypes (single-table inheritance or separate tables)
2. Resolving `{pk}` to the actual primary key column name
3. Applying implicit filters (e.g., catalog version restrictions from the session)
4. Generating database-specific SQL (HANA, MySQL, Oracle, etc.)
5. Executing the SQL via JDBC and wrapping results in `SearchResult`

The translation layer lives in `de.hybris.platform.persistence.flexiblesearch.TranslatedQuery`. You can see the generated SQL in HAC's FlexibleSearch console — after running a query, the actual SQL appears below the results.

### FlexibleSearch vs. Standard SQL

| Feature | FlexibleSearch | Standard SQL |
|---------|---------------|--------------|
| Target | Type system types | Database tables |
| Syntax | `{attribute}` curly braces | Column names directly |
| Localization | Built-in `{name[en]}` | Manual join to localization tables |
| Type hierarchy | Automatic subtype inclusion | Manual UNION or JOIN |
| Catalog filtering | Session-based automatic | Manual WHERE clauses |
| Parameterization | `?param` syntax | `?` or `:param` |
| Functions | Limited (DB-dependent) | Full SQL function support |
| INSERT/UPDATE/DELETE | Not supported | Supported |

**Key limitation**: FlexibleSearch is read-only. You cannot use it to modify data. All writes go through `ModelService`.

---

## Basic Query Syntax

### The Simplest Query

```sql
SELECT {pk} FROM {Product}
```

Returns the PKs of all `Product` items (including subtypes like `VariantProduct`, `ApparelProduct`, etc.). FlexibleSearch always requires at least `{pk}` in the SELECT clause.

### Selecting Multiple Attributes

```sql
SELECT {pk}, {code}, {name} FROM {Product}
```

Returns PKs, codes, and names. Note: when you use `FlexibleSearchService` in Java code, the result always returns Model objects hydrated via PK — the additional SELECT columns are mainly useful for HAC debugging or when using raw result mode.

### WHERE Clause

```sql
SELECT {pk} FROM {Product} WHERE {code} = 'PROD-001'
```

```sql
SELECT {pk} FROM {Product} WHERE {approvalStatus} = {{SELECT {pk} FROM {ArticleApprovalStatus} WHERE {code} = 'approved'}}
```

### Ordering

```sql
SELECT {pk} FROM {Product} WHERE {name} IS NOT NULL ORDER BY {name} ASC
```

### LIMIT and Pagination

FlexibleSearch doesn't have a `LIMIT` keyword. Pagination is controlled via the Java API:

```java
FlexibleSearchQuery query = new FlexibleSearchQuery("SELECT {pk} FROM {Product}");
query.setStart(0);     // offset
query.setCount(20);    // page size
SearchResult<ProductModel> result = flexibleSearchService.search(query);
```

### DISTINCT

```sql
SELECT DISTINCT {p.code} FROM {Product AS p}
```

### COUNT

```sql
SELECT COUNT({pk}) FROM {Product} WHERE {approvalStatus} = {{SELECT {pk} FROM {ArticleApprovalStatus} WHERE {code} = 'approved'}}
```

When using COUNT in Java:

```java
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT COUNT({pk}) FROM {Product}");
query.setResultClassList(Arrays.asList(Integer.class));
SearchResult<Integer> result = flexibleSearchService.search(query);
int totalProducts = result.getResult().get(0);
```

---

## Understanding the Curly Brace Syntax

The curly braces are what make FlexibleSearch different from SQL. They reference the type system, not database columns.

### `{attribute}` — Simple Attribute

```sql
{code}       -- resolves to the 'code' column of the item's table
{name}       -- resolves to the 'name' column (default language)
{pk}         -- the item's primary key
{creationtime}  -- platform-level audit attribute
{modifiedtime}  -- platform-level audit attribute
```

### `{name[en]}` — Localized Attribute

Localized attributes are stored in separate `*lp` tables. The `[lang]` syntax generates the necessary JOIN:

```sql
SELECT {pk} FROM {Product} WHERE {name[en]} LIKE '%widget%'
```

This translates to a JOIN with the localization table, filtering by the `en` language code.

You can query multiple languages:

```sql
SELECT {pk}, {name[en]}, {name[de]}, {name[fr]} FROM {Product}
```

### `{alias.attribute}` — Aliased References

When joining types, use aliases to disambiguate:

```sql
SELECT {p.pk}, {p.code}, {c.code}
FROM {Product AS p
    JOIN Category AS c ON {p.supercategories} = {c.pk}}
```

### `{Type}` — Type Reference in FROM Clause

```sql
FROM {Product}                    -- includes all subtypes
FROM {Product!}                   -- EXCLUDES subtypes (exact type only)
FROM {Product AS p}               -- with alias
FROM {Product*}                   -- same as {Product}, includes subtypes (explicit)
```

The `!` suffix is crucial. `{Product}` returns Products, VariantProducts, ApparelProducts, etc. `{Product!}` returns only items whose exact type is `Product`.

### `{{subquery}}` — Subqueries

Double curly braces denote subqueries:

```sql
SELECT {pk} FROM {Product}
WHERE {catalogVersion} IN (
    {{SELECT {pk} FROM {CatalogVersion}
      WHERE {catalog} IN (
          {{SELECT {pk} FROM {Catalog} WHERE {id} = 'myProductCatalog'}}
      ) AND {version} = 'Online'}}
)
```

---

## Joins and Subqueries

### JOIN Syntax

FlexibleSearch supports JOIN on type relations:

```sql
SELECT {p.pk}, {p.code}, {cv.version}
FROM {Product AS p
    JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}
    JOIN Catalog AS cat ON {cv.catalog} = {cat.pk}}
WHERE {cat.id} = 'myProductCatalog'
  AND {cv.version} = 'Online'
```

### LEFT JOIN

```sql
SELECT {p.pk}, {p.code}, {s.available}
FROM {Product AS p
    LEFT JOIN StockLevel AS s ON {s.productCode} = {p.code}}
WHERE {p.catalogVersion} = ?catalogVersion
```

### Many-to-Many Relations

Many-to-many relations in SAP Commerce create link tables. You reference them by the relation name:

```sql
-- Products in a specific category (many-to-many via CategoryProductRelation)
SELECT {p.pk}
FROM {Product AS p
    JOIN CategoryProductRelation AS rel ON {rel.target} = {p.pk}
    JOIN Category AS c ON {rel.source} = {c.pk}}
WHERE {c.code} = 'electronics'
  AND {p.catalogVersion} = ?catalogVersion
```

Alternatively, use the implicit relation traversal:

```sql
SELECT {p.pk}
FROM {Category AS c
    JOIN CategoryProductRelation AS cpr ON {cpr.source} = {c.pk}
    JOIN Product AS p ON {cpr.target} = {p.pk}}
WHERE {c.code} = 'electronics'
```

### Catalog Version Filtering

This is the most common join pattern in SAP Commerce:

```sql
SELECT {p.pk}
FROM {Product AS p
    JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}
    JOIN Catalog AS cat ON {cv.catalog} = {cat.pk}}
WHERE {cat.id} = 'electronicsProductCatalog'
  AND {cv.version} = 'Staged'
  AND {p.approvalStatus} = {{SELECT {pk} FROM {ArticleApprovalStatus} WHERE {code} = 'approved'}}
ORDER BY {p.code}
```

**Pro tip**: In most application code, you don't need explicit catalog version JOINs. The platform applies catalog version filtering from the session context automatically. But in HAC and unit tests, you must be explicit.

---

## Query Parameters

Always use parameterized queries to prevent SQL injection and enable query plan caching.

### `?parameter` Syntax

```sql
SELECT {pk} FROM {Product}
WHERE {code} = ?code
  AND {catalogVersion} = ?catalogVersion
```

In Java:

```java
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Product} WHERE {code} = ?code AND {catalogVersion} = ?catalogVersion");
query.addQueryParameter("code", "PROD-001");
query.addQueryParameter("catalogVersion", catalogVersionModel);
SearchResult<ProductModel> result = flexibleSearchService.search(query);
```

### Model Objects as Parameters

You can pass Model objects directly — the engine extracts the PK:

```java
query.addQueryParameter("catalogVersion", catalogVersionService.getCatalogVersion("myProductCatalog", "Staged"));
query.addQueryParameter("category", categoryModel);
query.addQueryParameter("user", userService.getCurrentUser());
```

### Collection Parameters

For IN clauses, pass a Collection:

```java
List<String> codes = Arrays.asList("PROD-001", "PROD-002", "PROD-003");
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Product} WHERE {code} IN (?codes)");
query.addQueryParameter("codes", codes);
```

### Enum Parameters

```java
query.addQueryParameter("status", ArticleApprovalStatus.APPROVED);
```

---

## Localized Attributes

Querying localized data has specific patterns in FlexibleSearch.

### Query by Current Session Language

```sql
SELECT {pk}, {name} FROM {Product} WHERE {name} LIKE '%widget%'
```

Without a language qualifier, `{name}` uses the current session language.

### Query by Specific Language

```sql
SELECT {pk}, {name[en]}, {name[de]} FROM {Product} WHERE {name[en]} IS NOT NULL
```

### Find Products Missing Translations

```sql
SELECT {pk}, {code}, {name[en]}, {name[de]}
FROM {Product}
WHERE {name[en]} IS NOT NULL
  AND {name[de]} IS NULL
  AND {catalogVersion} = ?catalogVersion
```

This is valuable for translation quality assurance.

### Query Across All Languages

To find a product regardless of which language the name was entered in:

```sql
SELECT {pk} FROM {Product}
WHERE {name[en]} LIKE '%search%'
   OR {name[de]} LIKE '%search%'
   OR {name[fr]} LIKE '%search%'
```

Not elegant, but necessary when you need cross-language search without Solr.

---

## Common Query Patterns

### Product Search with Catalog Version

```sql
SELECT {p.pk}
FROM {Product AS p}
WHERE {p.catalogVersion} = ?catalogVersion
  AND {p.approvalStatus} = ?approvalStatus
  AND {p.code} LIKE ?codePattern
ORDER BY {p.code} ASC
```

```java
query.addQueryParameter("catalogVersion", onlineCatalogVersion);
query.addQueryParameter("approvalStatus", ArticleApprovalStatus.APPROVED);
query.addQueryParameter("codePattern", "ELEC-%");
```

### Order Lookup by Customer

```sql
SELECT {o.pk}
FROM {Order AS o
    JOIN User AS u ON {o.user} = {u.pk}}
WHERE {u.uid} = ?userId
ORDER BY {o.creationtime} DESC
```

### Find Orders by Date Range

```sql
SELECT {pk}
FROM {Order}
WHERE {creationtime} >= ?startDate
  AND {creationtime} < ?endDate
  AND {status} = {{SELECT {pk} FROM {OrderStatus} WHERE {code} = 'COMPLETED'}}
ORDER BY {creationtime} DESC
```

### User Queries

```sql
-- Find customers by email domain
SELECT {pk}, {uid}, {name}
FROM {Customer}
WHERE {uid} LIKE '%@example.com'

-- Find customers who placed orders in the last 30 days
SELECT DISTINCT {u.pk}
FROM {Order AS o
    JOIN Customer AS u ON {o.user} = {u.pk}}
WHERE {o.creationtime} > ?thirtyDaysAgo
```

### CMS Component Queries

```sql
SELECT {pk}
FROM {CMSParagraphComponent}
WHERE {catalogVersion} = ?contentCatalogVersion
  AND {uid} = ?componentUid
```

### Price Row Queries

```sql
SELECT {pr.pk}, {pr.price}, {pr.currency}, {pr.net}
FROM {PriceRow AS pr
    JOIN Product AS p ON {pr.product} = {p.pk}}
WHERE {p.code} = ?productCode
  AND {pr.catalogVersion} = ?catalogVersion
  AND ({pr.startTime} IS NULL OR {pr.startTime} <= ?now)
  AND ({pr.endTime} IS NULL OR {pr.endTime} > ?now)
```

### Stock Level Queries

```sql
SELECT {sl.pk}, {sl.available}, {sl.warehouse}
FROM {StockLevel AS sl}
WHERE {sl.productCode} = ?productCode
  AND {sl.warehouse} IN (?warehouses)
```

---

## Performance Optimization

### Avoid SELECT *

Never use `SELECT *` in FlexibleSearch. Always select specific attributes, or at minimum just `{pk}`:

```sql
-- BAD: triggers full row hydration for every column
SELECT * FROM {Product}

-- GOOD: returns PKs, models are lazy-loaded on access
SELECT {pk} FROM {Product}
```

When `FlexibleSearchService` returns `SearchResult<ProductModel>`, the Model objects are populated lazily — attributes are fetched from cache or database only when accessed. Selecting `{pk}` is sufficient.

### Use Indexes

SAP Commerce creates indexes from `items.xml` definitions:

```xml
<itemtype code="LoyaltyTransaction" ...>
    <indexes>
        <index name="codeIdx" unique="true">
            <key attribute="code"/>
        </index>
        <index name="customerDateIdx">
            <key attribute="customer"/>
            <key attribute="transactionDate"/>
        </index>
    </indexes>
</itemtype>
```

Design indexes based on your query patterns. If you frequently query `WHERE {customer} = ? AND {transactionDate} > ?`, create a composite index on both attributes.

### Query Result Caching

FlexibleSearch supports result caching. Enable it per query:

```java
FlexibleSearchQuery query = new FlexibleSearchQuery("SELECT {pk} FROM {Currency}");
query.setCacheable(true);
```

Or enable it globally for specific type queries:

```properties
# Cache all FlexibleSearch results for Currency type for 300 seconds
flexiblesearch.cache.enabled=true
```

**Caution**: Caching mutable data (products, orders) requires careful invalidation. Best used for slowly-changing reference data (currencies, languages, units, countries).

### Analyzing Query Plans

In HAC's FlexibleSearch console, enable "Show SQL" to see the generated SQL. Copy that SQL and run `EXPLAIN` in your database console:

```sql
-- For HANA
EXPLAIN PLAN FOR SELECT ... FROM products p0 WHERE ...

-- For MySQL
EXPLAIN SELECT ... FROM products p0 WHERE ...
```

Look for:
- **Full table scans** → Add an index
- **Nested loop joins on large tables** → Consider restructuring the query
- **Sort operations on unindexed columns** → Add an index for ORDER BY columns

### Avoid N+1 Queries

A common anti-pattern in DAOs:

```java
// BAD: N+1 queries
List<ProductModel> products = getProducts(catalogVersion);
for (ProductModel product : products) {
    List<PriceRowModel> prices = getPricesForProduct(product); // Another query per product!
}
```

Better approach — use a single JOIN query:

```java
// GOOD: Single query with JOIN
String queryStr = "SELECT {p.pk} FROM {Product AS p " +
    "JOIN PriceRow AS pr ON {pr.product} = {p.pk}} " +
    "WHERE {p.catalogVersion} = ?cv AND {pr.price} < ?maxPrice";
```

Or prefetch prices in batch:

```java
String queryStr = "SELECT {pk} FROM {PriceRow} WHERE {product} IN (?products)";
query.addQueryParameter("products", productModels);
```

---

## FlexibleSearch vs. Solr

Both are query mechanisms, but they serve different purposes.

| Aspect | FlexibleSearch | Solr |
|--------|---------------|------|
| Source | Database (live data) | Solr index (snapshot) |
| Speed | Slower for full-text | Faster for full-text |
| Data freshness | Real-time | Eventual (after indexing) |
| Full-text search | Basic (LIKE) | Advanced (stemming, synonyms, fuzzy) |
| Faceting | Not built-in | Built-in |
| Use case | Service layer DAOs, admin, reports | Storefront product search |
| Scalability | Limited by DB | Horizontally scalable |

**Use FlexibleSearch when:**
- You need real-time data accuracy (order processing, cart operations)
- You're building DAOs for service-layer operations
- You're querying in HAC for debugging
- You need data that isn't in the Solr index

**Use Solr when:**
- You're building product search and navigation for the storefront
- You need faceted search (filter by category, price range, brand)
- You need full-text search with relevance scoring
- You need high-throughput read operations

In the standard architecture, `CommerceSearchService` (which delegates to Solr) handles storefront product search, while `FlexibleSearchService` handles everything else.

---

## Using FlexibleSearch in Code

### FlexibleSearchService

The primary Java API:

```java
@Resource
private FlexibleSearchService flexibleSearchService;

public List<ProductModel> findProductsByCategory(CategoryModel category, CatalogVersionModel cv) {
    String queryStr = "SELECT {p.pk} FROM {Product AS p " +
        "JOIN CategoryProductRelation AS cpr ON {cpr.target} = {p.pk} " +
        "JOIN Category AS c ON {cpr.source} = {c.pk}} " +
        "WHERE {c.pk} = ?category AND {p.catalogVersion} = ?cv " +
        "ORDER BY {p.name} ASC";
    
    FlexibleSearchQuery query = new FlexibleSearchQuery(queryStr);
    query.addQueryParameter("category", category);
    query.addQueryParameter("cv", cv);
    query.setCount(100); // max results
    
    SearchResult<ProductModel> result = flexibleSearchService.search(query);
    return result.getResult();
}
```

### SearchResult API

```java
SearchResult<ProductModel> result = flexibleSearchService.search(query);

List<ProductModel> items = result.getResult();      // The actual results
int totalCount = result.getTotalCount();              // Total matching items (for pagination)
int requestedCount = result.getRequestedCount();      // The count you requested
int requestedStart = result.getRequestedStart();      // The offset you requested
```

### Pagination Pattern

```java
public SearchPageData<ProductModel> findProducts(int page, int pageSize) {
    FlexibleSearchQuery query = new FlexibleSearchQuery("SELECT {pk} FROM {Product}");
    query.setStart(page * pageSize);
    query.setCount(pageSize);
    query.setNeedTotal(true); // Required for total count
    
    SearchResult<ProductModel> result = flexibleSearchService.search(query);
    
    SearchPageData<ProductModel> pageData = new SearchPageData<>();
    pageData.setResults(result.getResult());
    
    PaginationData pagination = new PaginationData();
    pagination.setTotalNumberOfResults(result.getTotalCount());
    pagination.setCurrentPage(page);
    pagination.setPageSize(pageSize);
    pagination.setNumberOfPages((int) Math.ceil((double) result.getTotalCount() / pageSize));
    pageData.setPagination(pagination);
    
    return pageData;
}
```

### Raw Result Mode

When you need non-Model results (aggregations, projections):

```java
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {approvalStatus}, COUNT({pk}) FROM {Product} GROUP BY {approvalStatus}");
query.setResultClassList(Arrays.asList(Object.class, Integer.class));

SearchResult<List<Object>> result = flexibleSearchService.search(query);
for (List<Object> row : result.getResult()) {
    Object status = row.get(0);
    Integer count = (Integer) row.get(1);
    // process...
}
```

---

## Using FlexibleSearch in HAC

The Hybris Administration Console (HAC) is your primary tool for ad-hoc FlexibleSearch queries.

### Accessing the Console

Navigate to `https://localhost:9002/hac/console/flexsearch` (or your HAC URL). You'll see:

- **Query input**: Where you type your FlexibleSearch query
- **Max count**: Limits returned rows
- **Commit mode**: For queries that might affect DB state (rarely needed)
- **Output**: Results table, generated SQL, execution time

### Debugging Tips

**Check if an item exists:**

```sql
SELECT {pk}, {code}, {name[en]}, {catalogVersion} FROM {Product} WHERE {code} = 'PROD-001'
```

**Inspect all attributes of an item:**

```sql
SELECT * FROM {Product} WHERE {code} = 'PROD-001'
```

**Check catalog versions:**

```sql
SELECT {cv.pk}, {cv.version}, {cat.id}
FROM {CatalogVersion AS cv JOIN Catalog AS cat ON {cv.catalog} = {cat.pk}}
```

**Find orphaned items:**

```sql
-- Products without categories
SELECT {p.pk}, {p.code}
FROM {Product AS p}
WHERE {p.catalogVersion} = ?cv
  AND NOT EXISTS (
    {{SELECT 1 FROM {CategoryProductRelation AS cpr} WHERE {cpr.target} = {p.pk}}}
  )
```

**Check recent modifications:**

```sql
SELECT {pk}, {code}, {modifiedtime}
FROM {Product}
WHERE {modifiedtime} > '2025-03-01 00:00:00'
ORDER BY {modifiedtime} DESC
```

### Setting Catalog Version in HAC

HAC doesn't automatically set a catalog version session context. For queries that rely on it, use explicit WHERE clauses:

```sql
-- Explicit catalog version
SELECT {pk} FROM {Product}
WHERE {catalogVersion} IN (
    {{SELECT {pk} FROM {CatalogVersion}
      WHERE {version} = 'Online'
        AND {catalog} IN ({{SELECT {pk} FROM {Catalog} WHERE {id} = 'myProductCatalog'}})}})
```

---

## Real-World Complex Query Examples

### Find Products with Missing Prices

```sql
SELECT {p.pk}, {p.code}, {p.name[en]}
FROM {Product AS p}
WHERE {p.catalogVersion} = ?catalogVersion
  AND {p.approvalStatus} = {{SELECT {pk} FROM {ArticleApprovalStatus} WHERE {code} = 'approved'}}
  AND NOT EXISTS (
    {{SELECT {pr.pk} FROM {PriceRow AS pr}
      WHERE {pr.product} = {p.pk}
        AND {pr.currency} = {{SELECT {pk} FROM {Currency} WHERE {isocode} = 'USD'}}
    }}
  )
ORDER BY {p.code}
```

### Customers with Most Orders (Last 90 Days)

```sql
SELECT {u.uid}, {u.name}, COUNT({o.pk}) AS orderCount
FROM {Order AS o
    JOIN Customer AS u ON {o.user} = {u.pk}}
WHERE {o.creationtime} > ?ninetyDaysAgo
GROUP BY {u.uid}, {u.name}
ORDER BY orderCount DESC
```

### Products with Low Stock Across All Warehouses

```sql
SELECT {p.code}, {p.name[en]}, SUM({sl.available}) AS totalStock
FROM {StockLevel AS sl
    JOIN Product AS p ON {sl.productCode} = {p.code}}
WHERE {p.catalogVersion} = ?catalogVersion
GROUP BY {p.code}, {p.name[en]}
HAVING SUM({sl.available}) < 10
ORDER BY totalStock ASC
```

### Unsynced Products (In Staged but Not in Online)

```sql
SELECT {staged.pk}, {staged.code}
FROM {Product AS staged}
WHERE {staged.catalogVersion} = ?stagedCV
  AND NOT EXISTS (
    {{SELECT {online.pk} FROM {Product AS online}
      WHERE {online.code} = {staged.code}
        AND {online.catalogVersion} = ?onlineCV}}
  )
```

### CMS Pages with Empty Slots

```sql
SELECT {page.pk}, {page.uid}, {page.name}
FROM {ContentPage AS page}
WHERE {page.catalogVersion} = ?contentCatalogVersion
  AND EXISTS (
    {{SELECT {rel.pk}
      FROM {ContentSlotForPage AS rel}
      WHERE {rel.page} = {page.pk}
        AND NOT EXISTS (
          {{SELECT {crel.pk}
            FROM {ElementsForSlot AS crel}
            WHERE {crel.source} IN (
              {{SELECT {slot.pk} FROM {ContentSlot AS slot}
                WHERE {slot.pk} = {rel.contentSlot}}}
            )}}
        )}}
  )
```

### Order Revenue by Month

```sql
SELECT MONTH({o.creationtime}), YEAR({o.creationtime}), 
       SUM({o.totalPrice}), {o.currency}
FROM {Order AS o}
WHERE {o.creationtime} >= ?startDate
  AND {o.status} = {{SELECT {pk} FROM {OrderStatus} WHERE {code} = 'COMPLETED'}}
GROUP BY MONTH({o.creationtime}), YEAR({o.creationtime}), {o.currency}
ORDER BY YEAR({o.creationtime}), MONTH({o.creationtime})
```

**Note**: Date functions like `MONTH()` and `YEAR()` are database-specific. These work on HANA and MySQL but may need adjustment for other databases.

---

## Top 10 FlexibleSearch Mistakes and How to Fix Them

### 1. Forgetting Catalog Version Filtering

**Problem**: Query returns zero results even though the data exists.

```sql
-- Returns nothing because no catalog version context is set
SELECT {pk} FROM {Product} WHERE {code} = 'PROD-001'
```

**Fix**: Always include catalog version in the WHERE clause or set it in the session:

```sql
SELECT {pk} FROM {Product} WHERE {code} = 'PROD-001' AND {catalogVersion} = ?cv
```

### 2. Using SELECT * in Production Code

**Problem**: Selects all columns, including LOB fields, causing unnecessary data transfer.

**Fix**: Always `SELECT {pk}` and let the Model layer handle lazy loading.

### 3. Not Parameterizing Queries

**Problem**: String concatenation opens SQL injection vectors and prevents query plan caching.

```java
// BAD
String query = "SELECT {pk} FROM {Product} WHERE {code} = '" + userInput + "'";

// GOOD
FlexibleSearchQuery query = new FlexibleSearchQuery(
    "SELECT {pk} FROM {Product} WHERE {code} = ?code");
query.addQueryParameter("code", userInput);
```

### 4. Missing the `!` Suffix for Exact Type Matching

**Problem**: You want only base Products but also get VariantProducts, ApparelProducts, etc.

```sql
-- Returns ALL product subtypes
SELECT {pk} FROM {Product}

-- Returns ONLY exact Product type
SELECT {pk} FROM {Product!}
```

### 5. Forgetting `setNeedTotal(true)` for Pagination

**Problem**: `getTotalCount()` returns -1.

```java
query.setNeedTotal(true); // Must be set BEFORE search
SearchResult result = flexibleSearchService.search(query);
result.getTotalCount(); // Now returns the actual count
```

### 6. N+1 Query Pattern in Loops

**Problem**: Executing a query per item in a loop.

**Fix**: Batch your queries. Use IN clauses with collections or JOINs.

### 7. LIKE with Leading Wildcard

**Problem**: `LIKE '%searchterm%'` cannot use indexes and triggers full table scans.

```sql
-- Slow: leading wildcard prevents index usage
WHERE {name[en]} LIKE '%widget%'

-- Faster: trailing wildcard only
WHERE {name[en]} LIKE 'widget%'
```

For full-text search, use Solr instead.

### 8. Querying Localized Attributes Without Language

**Problem**: Inconsistent results depending on session language.

**Fix**: Always specify the language explicitly when the query is language-specific:

```sql
WHERE {name[en]} LIKE '%widget%'   -- explicit
-- vs
WHERE {name} LIKE '%widget%'       -- depends on session language
```

### 9. Not Handling Empty Results

```java
SearchResult<ProductModel> result = flexibleSearchService.search(query);
List<ProductModel> products = result.getResult();
if (products.isEmpty()) {
    // Handle no results — don't just .get(0)
}
```

### 10. Subquery Performance

**Problem**: Deeply nested subqueries can be very slow.

```sql
-- Slow: 3 levels of subquery nesting
WHERE {catalogVersion} IN ({{SELECT ... WHERE {catalog} IN ({{SELECT ... WHERE ...}})}})
```

**Fix**: Use JOINs instead of subqueries when possible:

```sql
-- Faster: JOIN-based approach
FROM {Product AS p
    JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}
    JOIN Catalog AS cat ON {cv.catalog} = {cat.pk}}
WHERE {cat.id} = 'myProductCatalog' AND {cv.version} = 'Online'
```

---

## Summary

FlexibleSearch is the primary data query mechanism in SAP Commerce. The key principles:

1. **It operates on the type system, not database tables** — curly braces reference types and attributes, not columns
2. **Always parameterize** — use `?param` syntax for safety and performance
3. **SELECT {pk} is usually sufficient** — the Model layer handles attribute loading
4. **Catalog version context matters** — forgetting it is the #1 cause of "no results" bugs
5. **Use JOINs over subqueries** — they're more readable and generally faster
6. **Know when to use Solr instead** — full-text search, faceting, and high-throughput reads belong in Solr
7. **Profile your queries** — check the generated SQL, run EXPLAIN, and add indexes for frequently used WHERE/ORDER BY columns
8. **Cache carefully** — enable query caching for stable reference data, not for frequently-changing entities

FlexibleSearch is a tool you'll use every day on SAP Commerce projects. Invest in understanding it deeply, and it will pay dividends in debugging speed, query performance, and code quality.
{% endraw %}
