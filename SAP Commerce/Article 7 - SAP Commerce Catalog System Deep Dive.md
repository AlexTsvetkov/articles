# SAP Commerce Catalog System Deep Dive: Staged Content, Synchronization, and Multi-Catalog Architecture

The catalog system is one of SAP Commerce's most powerful — and most misunderstood — features. It provides a staging mechanism that lets business users prepare product data, pricing, and content changes in isolation, then publish everything to the live storefront in a single atomic operation. When used correctly, it eliminates the "edit live data and hope nothing breaks" anti-pattern. When used incorrectly, it becomes a source of synchronization headaches, data inconsistency, and performance bottlenecks.

This article breaks down how the catalog system actually works, how synchronization moves data from staged to online, and how to architect multi-catalog setups for complex B2B and multi-brand scenarios.

---

## The Two-Version Model

Every catalog in SAP Commerce has at least two catalog versions:

```
┌─────────────────────────────────────────┐
│  Product Catalog: "electronicsProductCatalog"  │
│                                         │
│  ┌───────────────┐    ┌──────────────┐ │
│  │   Staged       │───>│   Online     │ │
│  │   Version      │sync│   Version    │ │
│  │               │───>│              │ │
│  │ • Edit here   │    │ • Read-only  │ │
│  │ • Preview     │    │ • Live data  │ │
│  │ • No customer │    │ • Storefront │ │
│  │   impact      │    │   serves this│ │
│  └───────────────┘    └──────────────┘ │
└─────────────────────────────────────────┘
```

**Staged**: The working version. Business users, product managers, and content editors make all their changes here. Products are created, prices are updated, categories are reorganized — all without affecting what customers see.

**Online**: The published version. The storefront (OCC API, Accelerator, Spartacus) reads data from this version. It's effectively read-only from a business perspective — changes should only come through synchronization from Staged.

### Why This Matters

Without the staged/online model, every product edit would immediately be visible to customers. Imagine changing a product price from $99 to $79 for a Black Friday promotion — but the promotion doesn't start until Friday at midnight. With direct editing, you'd have to set the new price at exactly midnight. With staged/online, you set the price in Staged days in advance and synchronize at midnight.

### Setting Up a Catalog

Catalogs are created via ImpEx:

```impex
# Create the catalog
INSERT_UPDATE Catalog;id[unique=true];name[lang=en];defaultCatalog
;myProductCatalog;My Product Catalog;true

# Create catalog versions
INSERT_UPDATE CatalogVersion;catalog(id)[unique=true];version[unique=true];active;languages(isocode)
;myProductCatalog;Staged;false;en,de,fr
;myProductCatalog;Online;true;en,de,fr
```

The `active=true` flag marks the Online version as the one served to customers.

---

## How Synchronization Works

Synchronization is the process of copying data from one catalog version to another — typically from Staged to Online.

### The Sync Job

A `CatalogVersionSyncJob` defines what to synchronize:

```impex
INSERT_UPDATE CatalogVersionSyncJob;code[unique=true];sourceVersion(catalog(id),version)[unique=true];targetVersion(catalog(id),version)[unique=true];createNewItems;removeMissingItems
;myProductCatalogSyncJob;myProductCatalog:Staged;myProductCatalog:Online;true;true
```

| Parameter | Meaning |
|-----------|---------|
| `createNewItems` | If `true`, new items in Staged are created in Online |
| `removeMissingItems` | If `true`, items deleted from Staged are deleted from Online |

### What Gets Synchronized

The sync job synchronizes all **catalog-aware** item types. An item type is catalog-aware if it has a `catalogVersion` attribute. The standard catalog-aware types include:

- `Product` (and subtypes like `VariantProduct`)
- `Category`
- `Media`
- `Keyword`
- `CatalogVersionSyncScheduleMedia`

Custom types become catalog-aware when you add a `catalogVersion` attribute:

```xml
<itemtype code="CustomContentBlock" extends="GenericItem" autocreate="true" generate="true">
    <deployment table="CustomContentBlocks" typecode="25050"/>
    <attributes>
        <attribute qualifier="code" type="java.lang.String">
            <persistence type="property"/>
            <modifiers unique="true"/>
        </attribute>
        <attribute qualifier="catalogVersion" type="CatalogVersion">
            <persistence type="property"/>
            <modifiers read="true" write="true"/>
        </attribute>
        <!-- other attributes -->
    </attributes>
</itemtype>
```

### Sync Internals: What Actually Happens

When a sync job runs, the platform:

1. **Identifies changed items** — compares timestamps between source and target catalog versions
2. **Copies item data** — creates or updates items in the target version with data from the source
3. **Resolves references** — translates references from source catalog version to target (e.g., a product's category reference in Staged maps to the corresponding category in Online)
4. **Handles media** — copies media items and their binary data
5. **Maintains link consistency** — ensures all relations (product-to-category, product-to-media) are consistent in the target version

```
Sync Process Flow:
┌─────────────────────────────────────────────────┐
│ 1. Collect modified items from source (Staged)   │
│ 2. For each item:                                │
│    a. Find/create counterpart in target (Online) │
│    b. Copy all synchronized attributes           │
│    c. Resolve catalog-version-specific references│
│    d. Update timestamps                          │
│ 3. Handle deletions (if removeMissingItems=true) │
│ 4. Update sync status and counters               │
└─────────────────────────────────────────────────┘
```

### Counterpart Resolution

During synchronization, the platform needs to find the "counterpart" of a Staged item in Online. It uses unique keys for this:

- For `Product`: `code` + `catalogVersion`
- For `Category`: `code` + `catalogVersion`
- For `Media`: `code` + `catalogVersion`

A product with code `PROD-001` in `myProductCatalog:Staged` maps to `PROD-001` in `myProductCatalog:Online`. If the Online counterpart doesn't exist and `createNewItems=true`, it's created.

---

## Synchronization Performance

Catalog sync is one of the most resource-intensive operations in SAP Commerce. Understanding and optimizing it is critical for production environments.

### Full vs. Incremental Sync

**Full sync**: Compares every item in source and target. Safe but slow. Use only during initial setup or when data consistency is questionable.

**Incremental sync**: Only processes items modified since the last sync. Much faster. Use for regular production synchronization.

```java
// Programmatic incremental sync
SyncConfig config = new SyncConfig();
config.setSynchronizationType(SyncConfig.INCREMENTAL);
config.setCreateSavedValues(false);
config.setLogToDatabase(false);
config.setLogToFile(false);

catalogSynchronizationService.synchronize(syncJob, config);
```

### Performance Tuning

```properties
# Number of parallel sync workers
catalog.sync.workers=4

# Batch size — number of items processed per transaction
catalog.sync.batch.size=100

# Disable interceptors during sync (significant speedup)
catalog.sync.enable.interceptors=false

# Skip saved values (audit trail) for speed
catalog.sync.create.saved.values=false
```

### Timing and Scheduling

Schedule syncs during low-traffic periods:

```impex
INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression
;myProductCatalogSyncJob;0 0 3 * * ?
```

For time-critical syncs (flash sales, price changes), trigger programmatically:

```java
@Resource
private CatalogSynchronizationService catalogSynchronizationService;

public void publishCatalogChanges() {
    CatalogVersionSyncJobModel syncJob = // lookup sync job
    SyncConfig config = new SyncConfig();
    config.setSynchronizationType(SyncConfig.INCREMENTAL);
    
    CatalogVersionSyncCronJobModel cronJob = catalogSynchronizationService.synchronize(syncJob, config);
    LOG.info("Sync started: {}", cronJob.getCode());
}
```

---

## Content Catalogs

In addition to product catalogs, SAP Commerce uses **content catalogs** for CMS content (pages, components, slots). The same staged/online model applies:

```impex
INSERT_UPDATE ContentCatalog;id[unique=true];name[lang=en]
;myContentCatalog;My Content Catalog

INSERT_UPDATE CatalogVersion;catalog(id)[unique=true];version[unique=true];active
;myContentCatalog;Staged;false
;myContentCatalog;Online;true
```

### CMS Content Structure

```
Content Catalog (Staged)
├── PageTemplate: ProductDetailPageTemplate
├── ContentPage: homepage
│   ├── ContentSlot: Section1
│   │   ├── BannerComponent: heroBanner
│   │   └── ProductCarouselComponent: newArrivals
│   ├── ContentSlot: Section2
│   │   └── CategoryFeatureComponent: topCategories
│   └── ContentSlot: Footer
│       └── FooterNavigationComponent: mainFooter
├── ContentPage: faqPage
└── ContentPage: contactPage
```

Content catalog sync works identically to product catalog sync — changes made to CMS pages in Staged are published to Online through synchronization.

### SmartEdit Integration

SmartEdit (SAP Commerce's visual CMS editor) works exclusively with the Staged content catalog. Editors drag and drop components, rearrange content slots, and preview pages — all in Staged. When they click "Sync," the content catalog synchronization job runs and publishes changes to Online.

---

## Multi-Catalog Architecture

Real-world projects often need multiple catalogs. Here are the common patterns.

### Pattern 1: Shared Master Catalog

Multiple storefronts share a master product catalog but have separate content catalogs:

```
┌─────────────────────────────────┐
│     Master Product Catalog       │
│  (Staged → Online)              │
├────────────┬────────────────────┤
│            │                    │
│  ┌─────────▼──────┐ ┌─────────▼──────┐
│  │ US Content     │ │ EU Content     │
│  │ Catalog        │ │ Catalog        │
│  │ (Staged→Online)│ │ (Staged→Online)│
│  └────────────────┘ └────────────────┘
```

```impex
# Shared product catalog
INSERT_UPDATE Catalog;id[unique=true];name[lang=en]
;globalProductCatalog;Global Product Catalog

# Region-specific content catalogs
INSERT_UPDATE ContentCatalog;id[unique=true];name[lang=en]
;usContentCatalog;US Content Catalog
;euContentCatalog;EU Content Catalog
```

### Pattern 2: Multi-Country Product Catalogs

Each country has its own product catalog with country-specific pricing, availability, and product assortment:

```
┌──────────────────┐
│ Global Product    │
│ Catalog (Master)  │
│ Staged → Online   │
└────────┬─────────┘
         │ sync
    ┌────┴────┐
    │         │
┌───▼────┐ ┌─▼──────┐
│ US     │ │ DE     │
│ Product│ │ Product│
│ Catalog│ │ Catalog│
│ S → O  │ │ S → O  │
└────────┘ └────────┘
```

```impex
# Global master catalog
INSERT_UPDATE Catalog;id[unique=true];name[lang=en]
;globalProductCatalog;Global Product Catalog

# Country catalogs
INSERT_UPDATE Catalog;id[unique=true];name[lang=en]
;usProductCatalog;US Product Catalog
;deProductCatalog;DE Product Catalog

# Cross-catalog sync: Global Staged → US Staged
INSERT_UPDATE CatalogVersionSyncJob;code[unique=true];sourceVersion(catalog(id),version);targetVersion(catalog(id),version)
;globalToUsSyncJob;globalProductCatalog:Staged;usProductCatalog:Staged
;globalToDeSyncJob;globalProductCatalog:Staged;deProductCatalog:Staged

# Then US Staged → US Online
INSERT_UPDATE CatalogVersionSyncJob;code[unique=true];sourceVersion(catalog(id),version);targetVersion(catalog(id),version)
;usProductCatalogSyncJob;usProductCatalog:Staged;usProductCatalog:Online
;deProductCatalogSyncJob;deProductCatalog:Staged;deProductCatalog:Online
```

### Pattern 3: B2B Multi-Catalog with Visibility Rules

B2B scenarios often require different product visibility per customer group:

```impex
# Base catalog visible to all customers
INSERT_UPDATE Catalog;id[unique=true];name[lang=en]
;baseProductCatalog;Base Product Catalog

# Premium catalog with additional products
INSERT_UPDATE Catalog;id[unique=true];name[lang=en]
;premiumProductCatalog;Premium Product Catalog

# Map catalogs to customer groups via CMSSite
INSERT_UPDATE CMSSite;uid[unique=true];contentCatalogs(id);productCatalogs(id)
;standardSite;standardContentCatalog;baseProductCatalog
;premiumSite;premiumContentCatalog;baseProductCatalog,premiumProductCatalog
```

---

## Catalog-Aware Queries

When querying data, you must always specify the catalog version. This is one of the most common sources of bugs — forgetting the catalog version filter returns data from all versions (Staged + Online), causing duplicates.

### FlexibleSearch with Catalog Version

```java
// CORRECT: Always include catalogVersion in queries
String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code AND {catalogVersion} = ?cv";
FlexibleSearchQuery fsq = new FlexibleSearchQuery(query);
fsq.addQueryParameter("code", "PROD-001");
fsq.addQueryParameter("cv", catalogVersionService.getCatalogVersion("myProductCatalog", "Online"));
```

```java
// WRONG: Missing catalogVersion — returns products from ALL versions
String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
// This returns PROD-001 from both Staged AND Online — likely a bug
```

### Session Catalog Versions

SAP Commerce maintains a set of "session catalog versions" — the catalog versions the current user should see. The storefront sets Online versions; Backoffice sets Staged versions.

```java
// Get the session catalog version
Collection<CatalogVersionModel> sessionVersions = catalogVersionService.getSessionCatalogVersions();

// Set session catalog versions (typically done by the storefront filter chain)
catalogVersionService.setSessionCatalogVersions(
    Collections.singletonList(
        catalogVersionService.getCatalogVersion("myProductCatalog", "Online")
    )
);
```

Services like `ProductService.getProductForCode()` automatically filter by session catalog versions, so you don't need to pass the catalog version explicitly when using the service layer.

---

## Category Hierarchies

Categories are catalog-version-aware and form a tree structure within each catalog version.

### Defining Categories

```impex
$catalogVersion = catalogVersion(catalog(id[default='myProductCatalog']),version[default='Staged'])

INSERT_UPDATE Category;code[unique=true];$catalogVersion;name[lang=en];supercategories(code,$catalogVersion)
;electronics;;Electronics;
;cameras;;Cameras;electronics
;dslr;;DSLR Cameras;cameras
;mirrorless;;Mirrorless Cameras;cameras
;smartphones;;Smartphones;electronics
;accessories;;Accessories;electronics
;cases;;Phone Cases;accessories
;chargers;;Chargers;accessories
```

This creates a hierarchy:

```
Electronics
├── Cameras
│   ├── DSLR Cameras
│   └── Mirrorless Cameras
├── Smartphones
└── Accessories
    ├── Phone Cases
    └── Chargers
```

### Assigning Products to Categories

```impex
$catalogVersion = catalogVersion(catalog(id[default='myProductCatalog']),version[default='Staged'])

INSERT_UPDATE CategoryProductRelation;source(code,$catalogVersion)[unique=true];target(code,$catalogVersion)[unique=true]
;dslr;CANON-EOS-R5
;dslr;NIKON-Z8
;mirrorless;SONY-A7IV
;smartphones;IPHONE-15
;cases;CASE-IPHONE-15
```

A product can belong to multiple categories. During synchronization, these relationships are copied from Staged to Online.

---

## Handling Sync Conflicts and Edge Cases

### Partial Sync

Sometimes you need to sync only specific items rather than the entire catalog:

```java
// Sync specific products only
List<ItemModel> itemsToSync = Arrays.asList(product1, product2, product3);
SyncConfig config = new SyncConfig();
config.setSynchronizationType(SyncConfig.PARTIAL);

for (ItemModel item : itemsToSync) {
    catalogSynchronizationService.synchronizeItem(item, syncJob, config);
}
```

### Sync Excludes

Exclude specific attributes from synchronization:

```impex
# Don't sync the 'internalNotes' attribute — keep it Staged-only
INSERT_UPDATE CatalogVersionSyncJob;code[unique=true];&syncJobRef
;myProductCatalogSyncJob;syncJobRef

INSERT_UPDATE SyncAttributeDescriptorConfig;syncJob(&syncJobRef)[unique=true];attributeDescriptor(qualifier,enclosingType(code))[unique=true];includedInSync
;syncJobRef;internalNotes:Product;false
```

### Avoiding Common Sync Problems

**1. Missing references**: If a product references a category that hasn't been synced yet, the sync fails. Solution: sync categories before products, or sync both in the same job.

**2. Orphaned items**: If `removeMissingItems=true` and someone accidentally deletes items from Staged, they'll be deleted from Online too. Consider setting `removeMissingItems=false` for safety and handling deletions manually.

**3. Large media files**: Syncing catalogs with thousands of high-resolution images is slow. Consider using external media (CDN URLs) instead of storing media in the catalog.

**4. Concurrent modifications**: If two users modify the same product in Staged while a sync is running, the sync might pick up an inconsistent state. Best practice: lock the catalog (or communicate a sync window) during synchronization.

---

## Monitoring Sync Status

### Via HAC

Navigate to `Platform → System → Catalog Version Syncing` to see:
- Last sync date
- Number of items synced
- Sync status (SUCCESS, ERROR, RUNNING)
- Error details

### Programmatically

```java
CatalogVersionSyncCronJobModel syncCronJob = // lookup
LOG.info("Sync status: {}", syncCronJob.getStatus());
LOG.info("Sync result: {}", syncCronJob.getResult());
LOG.info("Items processed: {}", syncCronJob.getProcessedItemsCount());
LOG.info("Items synced: {}", syncCronJob.getSyncedItemsCount());
LOG.info("Errors: {}", syncCronJob.getFailedItemsCount());
```

### Sync Audit

Enable saved values to track what changed during each sync:

```properties
catalog.sync.create.saved.values=true
```

This creates `SavedValues` items that record the before/after state of each synced attribute. Useful for debugging but adds significant overhead — disable in production unless needed.

---

## Best Practices

1. **Always work in Staged** — never modify Online directly. All changes should flow through sync.
2. **Use incremental sync** — full sync is only for initial setup or data repair.
3. **Schedule syncs during off-peak hours** — sync is CPU and memory intensive.
4. **Separate product and content syncs** — they have different schedules and different business owners.
5. **Test sync with production-like data volumes** — sync that works with 100 products may fail with 500,000.
6. **Monitor sync duration trends** — increasing sync times indicate growing data complexity or missing indexes.
7. **Always include catalogVersion in FlexibleSearch queries** — the most common source of data bugs.
8. **Plan your multi-catalog architecture early** — changing catalog structure after go-live requires data migration.

---

## Summary

The SAP Commerce catalog system is a content staging and publishing framework. Its two-version model provides safety for business operations, while synchronization gives you atomic publishing of changes. The key principles:

1. **Staged is for editing, Online is for serving** — never break this boundary
2. **Synchronization copies data from Staged to Online** — it resolves references, handles media, and maintains consistency
3. **Incremental sync is your production workhorse** — full sync is for exceptional circumstances
4. **Multi-catalog architectures serve multi-country and B2B scenarios** — plan the catalog hierarchy carefully
5. **Always filter by catalog version in queries** — missing this filter is the most common catalog-related bug
6. **Monitor and tune sync performance** — it's one of the most resource-intensive platform operations

Understanding the catalog system deeply is what allows you to build commerce solutions that scale — both technically and operationally.