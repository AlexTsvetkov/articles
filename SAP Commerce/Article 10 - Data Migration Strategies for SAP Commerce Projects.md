# Data Migration Strategies for SAP Commerce Projects: Planning, Execution, and Recovery

Every SAP Commerce project involves data migration — whether you're migrating from a legacy platform, consolidating multiple systems, upgrading from an older Hybris version, or integrating with external data sources. The difference between a smooth go-live and a catastrophic one often comes down to how well the data migration was planned and tested.

This article covers the migration strategies, tooling, common pitfalls, and testing approaches that make the difference between a migration that works in theory and one that works in production.

---

## Migration Landscape

```
┌─────────────────────┐     ┌──────────────────────┐
│  Source Systems       │     │  SAP Commerce Cloud   │
│                       │     │                       │
│  Legacy eCommerce ──────────> Products & Categories │
│  ERP (SAP, Oracle)──────────> Pricing & Stock       │
│  PIM (Akeneo, etc)──────────> Product Content       │
│  CRM ──────────────────────> Customer Accounts      │
│  OMS ──────────────────────> Order History          │
│  CMS (WordPress) ──────────> Content Pages          │
│  CSV/Excel Files  ──────────> Reference Data        │
└─────────────────────┘     └──────────────────────┘
```

### Types of Migration

| Type | Description | Typical Timing |
|------|-------------|----------------|
| **Initial Load** | Full data population for go-live | During cutover window |
| **Delta Migration** | Incremental changes since last load | Between initial load and go-live |
| **Ongoing Sync** | Continuous integration with source systems | Post go-live |
| **Platform Upgrade** | Data schema changes between versions | During upgrade window |

---

## Planning the Migration

### Step 1: Data Inventory

Before writing any migration code, catalog what data needs to migrate:

```
Data Inventory Worksheet:
┌──────────────────┬──────────┬───────────┬────────────┬──────────────┐
│ Data Entity       │ Source   │ Volume    │ Priority   │ Dependencies │
├──────────────────┼──────────┼───────────┼────────────┼──────────────┤
│ Products          │ PIM      │ 50,000    │ Critical   │ Categories   │
│ Categories        │ PIM      │ 2,000     │ Critical   │ None         │
│ Prices            │ ERP      │ 200,000   │ Critical   │ Products     │
│ Stock Levels      │ ERP      │ 50,000    │ Critical   │ Products,    │
│                   │          │           │            │ Warehouses   │
│ Customers         │ CRM      │ 500,000   │ High       │ None         │
│ Addresses         │ CRM      │ 800,000   │ High       │ Customers    │
│ Order History     │ OMS      │ 2,000,000 │ Medium     │ Customers,   │
│                   │          │           │            │ Products     │
│ CMS Pages         │ CMS      │ 500       │ High       │ Media        │
│ Media/Images      │ CDN/PIM  │ 150,000   │ High       │ None         │
│ Promotions        │ Legacy   │ 200       │ Medium     │ Products,    │
│                   │          │           │            │ Categories   │
│ Classifications   │ PIM      │ 10,000    │ High       │ Products     │
└──────────────────┴──────────┴───────────┴────────────┴──────────────┘
```

### Step 2: Dependency Graph

Data must be loaded in dependency order. Loading products before categories fails because products reference categories.

```
Load Order (dependency-based):
1. Reference Data     → Countries, Currencies, Languages, Units
2. Warehouses         → Warehouse definitions, PointOfService
3. Categories         → Category hierarchy (parent before child)
4. Classification     → ClassificationSystem, ClassAttributeAssignment
5. Products           → Base products, then variants
6. Prices             → Price rows linked to products
7. Stock              → Stock levels linked to products + warehouses
8. Media              → Product images, category images
9. Customers          → Customer accounts
10. Addresses         → Customer addresses
11. Orders            → Historical orders (optional)
12. CMS Content       → Pages, components, media
13. Promotions        → Promotion rules
```

### Step 3: Mapping Specification

Document how source fields map to SAP Commerce fields:

```
Product Mapping:
┌──────────────────┬──────────────────────┬──────────────────┐
│ Source Field      │ SAP Commerce Field   │ Transformation   │
├──────────────────┼──────────────────────┼──────────────────┤
│ sku               │ Product.code         │ Direct           │
│ title             │ Product.name[en]     │ Direct           │
│ titel             │ Product.name[de]     │ Direct           │
│ description_html  │ Product.description  │ HTML sanitize    │
│ price_usd         │ PriceRow.price       │ Decimal (2dp)    │
│ weight_lbs        │ Product.weight       │ Convert to kg    │
│ category_path     │ CategoryProductRel   │ Split on '>'     │
│ main_image_url    │ Media.URL            │ Download + store │
│ brand_name        │ Product.manufacturer │ Lookup by name   │
│ active            │ Product.approvalStatus│ true→approved   │
│ created_at        │ Product.creationtime │ ISO date parse   │
└──────────────────┴──────────────────────┴──────────────────┘
```

---

## Migration Tooling: ImpEx

ImpEx is the primary migration tool in SAP Commerce. It handles bulk data operations with dependency resolution.

### Basic Product Migration ImpEx

```impex
# Macro definitions for reuse
$catalogVersion = catalogVersion(catalog(id[default='myProductCatalog']),version[default='Staged'])
$supercategories = supercategories(code, $catalogVersion)
$approved = approvalStatus(code)[default='approved']

# Categories first (dependencies)
INSERT_UPDATE Category;code[unique=true];name[lang=en];name[lang=de];$catalogVersion;$supercategories
;electronics;Electronics;Elektronik;;
;cameras;Cameras;Kameras;;electronics
;laptops;Laptops;Laptops;;electronics

# Products (depend on categories)
INSERT_UPDATE Product;code[unique=true];name[lang=en];name[lang=de];description[lang=en];$catalogVersion;$supercategories;$approved;unit(code)[default='pieces'];ean
;CAM-001;Professional DSLR;Professionelle DSLR;Professional-grade camera body;cameras
;CAM-002;Mirrorless Camera;Spiegellose Kamera;Compact mirrorless system;cameras
;LAP-001;Business Laptop;Business-Laptop;14-inch business laptop;laptops

# Prices (depend on products)
INSERT_UPDATE PriceRow;product(code,$catalogVersion)[unique=true];price;currency(isocode)[unique=true];unit(code)[default='pieces'];net
;CAM-001;2499.99;USD;;false
;CAM-001;2299.99;EUR;;false
;CAM-002;1799.99;USD;;false
;LAP-001;1299.99;USD;;false

# Stock levels (depend on products and warehouses)
INSERT_UPDATE StockLevel;productCode[unique=true];warehouse(code)[unique=true];available;inStockStatus(code)
;CAM-001;warehouse_us;150;forceInStock
;CAM-002;warehouse_us;200;forceInStock
;LAP-001;warehouse_us;500;forceInStock
```

### ImpEx for Customer Migration

```impex
# Customer groups
INSERT_UPDATE UserGroup;uid[unique=true];name;groups(uid)
;premiumCustomers;Premium Customers;customergroup

# Customers
INSERT_UPDATE Customer;uid[unique=true];name;customerID;groups(uid);sessionLanguage(isocode);sessionCurrency(isocode)
;john.doe@example.com;John Doe;CUST-10001;customergroup;en;USD
;jane.smith@example.com;Jane Smith;CUST-10002;premiumCustomers;en;USD
;hans.mueller@example.com;Hans Müller;CUST-10003;customergroup;de;EUR

# Addresses
INSERT_UPDATE Address;owner(Customer.uid)[unique=true];streetname;streetnumber;postalcode;town;country(isocode);billingAddress;shippingAddress;firstname;lastname;&addressID
;john.doe@example.com;Main Street;123;10001;New York;US;true;true;John;Doe;addr_10001
;jane.smith@example.com;Oak Avenue;456;90210;Beverly Hills;US;true;true;Jane;Smith;addr_10002
;hans.mueller@example.com;Hauptstraße;78;80331;München;DE;true;true;Hans;Müller;addr_10003

# Set default addresses
UPDATE Customer;uid[unique=true];defaultPaymentAddress(&addressID);defaultShipmentAddress(&addressID)
;john.doe@example.com;addr_10001;addr_10001
;jane.smith@example.com;addr_10002;addr_10002
;hans.mueller@example.com;addr_10003;addr_10003
```

---

## Automated Migration Pipeline

For large volumes, build an automated pipeline rather than running ImpEx manually.

### Architecture

```
┌─────────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│ Source       │    │ Extract &    │    │ ImpEx         │    │ SAP Commerce │
│ Systems     │───>│ Transform    │───>│ Generator     │───>│ Import       │
│ (DB, API,   │    │ (Python/Java)│    │               │    │ (HAC/API)    │
│  CSV, etc)  │    │              │    │               │    │              │
└─────────────┘    └──────────────┘    └───────────────┘    └──────────────┘
                          │                    │                     │
                          ▼                    ▼                     ▼
                    Validation           Generated            Import Logs
                    Reports              .impex files         & Error Reports
```

### Python ETL Script Example

```python
import csv
import re
from decimal import Decimal

class ProductMigrator:
    """Transforms source product data to ImpEx format."""
    
    IMPEX_HEADER = """
$catalogVersion = catalogVersion(catalog(id[default='myProductCatalog']),version[default='Staged'])
$supercategories = supercategories(code, $catalogVersion)
$approved = approvalStatus(code)[default='approved']

INSERT_UPDATE Product;code[unique=true];name[lang=en];description[lang=en];$catalogVersion;$supercategories;$approved;unit(code)[default='pieces'];ean;manufacturerName
"""
    
    def __init__(self, source_file, output_file, batch_size=5000):
        self.source_file = source_file
        self.output_file = output_file
        self.batch_size = batch_size
        self.errors = []
        self.processed = 0
        self.skipped = 0
    
    def transform_product(self, row):
        """Transform a single source row to ImpEx fields."""
        code = self.sanitize_code(row['sku'])
        if not code:
            self.errors.append(f"Invalid SKU: {row.get('sku', 'EMPTY')}")
            return None
        
        name = self.sanitize_text(row.get('title', ''))
        if not name:
            self.errors.append(f"Missing name for SKU: {code}")
            return None
        
        description = self.sanitize_html(row.get('description_html', ''))
        category = self.map_category(row.get('category_path', ''))
        ean = row.get('ean', '')
        manufacturer = self.sanitize_text(row.get('brand_name', ''))
        
        return f";{code};{name};{description};;{category};;;{ean};{manufacturer}"
    
    def sanitize_code(self, code):
        """Ensure product code is valid."""
        if not code:
            return None
        # Remove special characters, keep alphanumeric and hyphens
        return re.sub(r'[^a-zA-Z0-9\-_]', '', str(code).strip())
    
    def sanitize_text(self, text):
        """Clean text for ImpEx (escape semicolons)."""
        if not text:
            return ''
        text = str(text).strip()
        # ImpEx uses semicolons as delimiters — escape them
        text = text.replace(';', '\\;')
        # Remove newlines
        text = text.replace('\n', ' ').replace('\r', '')
        return text
    
    def sanitize_html(self, html):
        """Clean HTML content for product descriptions."""
        if not html:
            return ''
        # Remove script tags
        html = re.sub(r'<script[^>]*>.*?</script>', '', html, flags=re.DOTALL)
        # Escape semicolons
        html = html.replace(';', '\\;')
        return html.strip()
    
    def map_category(self, category_path):
        """Map source category path to SAP Commerce category code."""
        # Source: "Electronics > Cameras > DSLR"
        # Target: "dslr" (leaf category code)
        if not category_path:
            return ''
        parts = [p.strip().lower().replace(' ', '-') for p in category_path.split('>')]
        return parts[-1] if parts else ''
    
    def run(self):
        """Execute the migration."""
        with open(self.source_file, 'r', encoding='utf-8') as infile, \
             open(self.output_file, 'w', encoding='utf-8') as outfile:
            
            reader = csv.DictReader(infile)
            outfile.write(self.IMPEX_HEADER)
            
            for row in reader:
                line = self.transform_product(row)
                if line:
                    outfile.write(line + '\n')
                    self.processed += 1
                else:
                    self.skipped += 1
                
                # Write batch separator for large imports
                if self.processed % self.batch_size == 0:
                    outfile.write(f"\n# --- Batch {self.processed // self.batch_size} ---\n")
        
        self.write_report()
    
    def write_report(self):
        """Generate migration report."""
        report = f"""
Migration Report
================
Source: {self.source_file}
Output: {self.output_file}
Processed: {self.processed}
Skipped: {self.skipped}
Errors: {len(self.errors)}

Error Details:
"""
        for error in self.errors[:100]:  # First 100 errors
            report += f"  - {error}\n"
        
        with open(self.output_file.replace('.impex', '_report.txt'), 'w') as f:
            f.write(report)
        
        print(report)

# Usage
migrator = ProductMigrator('source_products.csv', 'products_import.impex')
migrator.run()
```

---

## Handling Large Data Volumes

### Batch Processing

For millions of records, split imports into manageable batches:

```bash
# Split a large ImpEx file into 10,000-line chunks
split -l 10000 products_import.impex products_batch_

# Generate batch import script
for file in products_batch_*; do
  echo "Importing $file..."
  curl -X POST "https://commerce-host:9002/hac/console/impex/import" \
    -H "Content-Type: multipart/form-data" \
    -F "scriptContent=@$file" \
    -F "encoding=UTF-8" \
    -F "maxThreads=4"
done
```

### Performance-Optimized Import Configuration

```properties
# project.properties — Import performance tuning

# Increase import batch size
impex.import.workers=8

# Disable interceptors during bulk import
impex.import.disable.interceptors=true

# Skip validation for trusted data
import.strict.mode=false

# JVM settings for large imports
# -Xmx8g -XX:MaxMetaspaceSize=512m
```

### Direct Database Loading

For extreme volumes (tens of millions), consider direct database operations alongside ImpEx:

```sql
-- Bulk insert stock levels directly (bypasses type system)
-- Only for reference data where interceptors aren't needed
LOAD DATA LOCAL INFILE '/tmp/stock_levels.csv'
INTO TABLE stocklevels
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(productCodePOS, warehousePOS, available, p_instockstatus);

-- IMPORTANT: Update PK sequences after direct inserts
-- And run catalog sync afterward
```

**Warning**: Direct database loading skips the SAP Commerce type system, interceptors, and caching. Use only when ImpEx performance is insufficient and you fully understand the data model.

---

## Data Validation

### Pre-Import Validation

Validate data before importing to catch issues early:

```python
class DataValidator:
    """Validate migration data before import."""
    
    def __init__(self):
        self.errors = []
        self.warnings = []
    
    def validate_products(self, products):
        seen_codes = set()
        
        for i, product in enumerate(products):
            row = i + 1
            
            # Required fields
            if not product.get('code'):
                self.errors.append(f"Row {row}: Missing product code")
                continue
            
            # Duplicates
            if product['code'] in seen_codes:
                self.errors.append(f"Row {row}: Duplicate code '{product['code']}'")
            seen_codes.add(product['code'])
            
            # Code format
            if not re.match(r'^[A-Za-z0-9\-_]+$', product['code']):
                self.errors.append(
                    f"Row {row}: Invalid code format '{product['code']}' "
                    f"(only alphanumeric, hyphens, underscores)")
            
            # Name length
            name = product.get('name', '')
            if len(name) > 255:
                self.warnings.append(
                    f"Row {row}: Name exceeds 255 chars, will be truncated")
            
            # Price validation
            price = product.get('price')
            if price:
                try:
                    p = Decimal(str(price))
                    if p < 0:
                        self.errors.append(f"Row {row}: Negative price {price}")
                    if p > 999999.99:
                        self.warnings.append(f"Row {row}: Very high price {price}")
                except:
                    self.errors.append(f"Row {row}: Invalid price format '{price}'")
            
            # Category reference
            if not product.get('category'):
                self.warnings.append(f"Row {row}: No category for '{product['code']}'")
        
        return len(self.errors) == 0
```

### Post-Import Verification

After import, verify data integrity:

```impex
# Verification queries via FlexibleSearch

# Check product count
# Expected: 50,000
SELECT COUNT(*) FROM {Product} WHERE {catalogVersion} = (
  SELECT {pk} FROM {CatalogVersion} WHERE {version} = 'Staged'
  AND {catalog} = (SELECT {pk} FROM {Catalog} WHERE {id} = 'myProductCatalog')
)

# Find products without prices
SELECT {p.code} FROM {Product AS p}
WHERE {p.catalogVersion} = ?cv
AND NOT EXISTS (
  SELECT 1 FROM {PriceRow AS pr} WHERE {pr.product} = {p.pk}
)

# Find products without categories
SELECT {p.code} FROM {Product AS p}
WHERE {p.catalogVersion} = ?cv
AND NOT EXISTS (
  SELECT 1 FROM {CategoryProductRelation AS r} WHERE {r.target} = {p.pk}
)

# Find products without images
SELECT {p.code} FROM {Product AS p}
WHERE {p.catalogVersion} = ?cv
AND {p.picture} IS NULL
AND {p.thumbnail} IS NULL
```

```java
// Programmatic verification
@Component
public class MigrationVerifier {
    
    @Resource
    private FlexibleSearchService flexibleSearchService;
    
    public MigrationReport verify(CatalogVersionModel catalogVersion) {
        MigrationReport report = new MigrationReport();
        
        // Count totals
        report.setProductCount(countItems("Product", catalogVersion));
        report.setCategoryCount(countItems("Category", catalogVersion));
        report.setMediaCount(countItems("Media", catalogVersion));
        
        // Find orphans
        report.setProductsWithoutPrices(findProductsWithoutPrices(catalogVersion));
        report.setProductsWithoutCategories(findProductsWithoutCategories(catalogVersion));
        report.setProductsWithoutImages(findProductsWithoutImages(catalogVersion));
        
        // Validate critical fields
        report.setProductsWithEmptyNames(findProductsWithEmptyField("name", catalogVersion));
        
        return report;
    }
    
    private long countItems(String typeCode, CatalogVersionModel cv) {
        String query = "SELECT COUNT(*) FROM {" + typeCode + "} WHERE {catalogVersion} = ?cv";
        FlexibleSearchQuery fsq = new FlexibleSearchQuery(query);
        fsq.addQueryParameter("cv", cv);
        fsq.setResultClassList(Collections.singletonList(Long.class));
        return flexibleSearchService.<Long>search(fsq).getResult().get(0);
    }
}
```

---

## Migration Patterns for Specific Scenarios

### Pattern: Platform Upgrade Migration

When upgrading SAP Commerce versions, the data model may change:

```
Pre-Upgrade Checklist:
┌────┬──────────────────────────────────────────────────────────┐
│ 1  │ Backup database (full dump)                              │
│ 2  │ Export critical data as ImpEx (safety net)               │
│ 3  │ Review release notes for type system changes             │
│ 4  │ Run `ant updatesystem` on a copy of production data      │
│ 5  │ Verify data integrity after update                       │
│ 6  │ Test all critical business flows                         │
│ 7  │ Compare item counts before/after                         │
│ 8  │ Validate search indexes                                  │
└────┴──────────────────────────────────────────────────────────┘
```

### Pattern: Customer Password Migration

Customer passwords can't be migrated in plain text (you don't have them). Common approaches:

1. **Force password reset**: After migration, all customers must reset their password at first login.

```impex
# Import customers without passwords — they must reset
INSERT_UPDATE Customer;uid[unique=true];name;encodedPassword
;john@example.com;John Doe;{bcrypt}$FORCE_RESET$
```

2. **Hash compatibility**: If the source system uses a compatible hash algorithm, import the hashes directly.

3. **Dual-auth period**: During a transition period, authenticate against both the old and new systems.

### Pattern: Order History Migration

Historical orders are read-only data. They need different handling:

```impex
# Orders reference products, customers, and addresses
# Load in strict dependency order

# 1. Ensure referenced products exist (at least as stubs)
INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en];$approved
;LEGACY-PROD-001;;Legacy Product 1;
;LEGACY-PROD-002;;Legacy Product 2;

# 2. Import orders
INSERT_UPDATE Order;code[unique=true];user(uid);date[dateformat=yyyy-MM-dd];currency(isocode);totalPrice;status(code);store(uid);site(uid)
;ORD-2023-001;john@example.com;2023-06-15;USD;299.99;COMPLETED;electronics;electronics-spa
;ORD-2023-002;jane@example.com;2023-07-20;USD;149.99;COMPLETED;electronics;electronics-spa

# 3. Import order entries
INSERT_UPDATE OrderEntry;order(code)[unique=true];entryNumber[unique=true];product(code,$catalogVersion);quantity;basePrice;totalPrice
;ORD-2023-001;0;LEGACY-PROD-001;1;299.99;299.99
;ORD-2023-002;0;LEGACY-PROD-002;2;74.99;149.99
```

---

## Rollback Strategy

Every migration needs a rollback plan.

### Database-Level Rollback

```bash
# Pre-migration: Full database backup
mysqldump -h db-host -u admin -p commerce_db > pre_migration_backup.sql

# Or for PostgreSQL
pg_dump -h db-host -U admin commerce_db > pre_migration_backup.sql

# Rollback if migration fails
mysql -h db-host -u admin -p commerce_db < pre_migration_backup.sql
```

### ImpEx-Based Rollback

For targeted rollback, generate reverse ImpEx:

```impex
# Remove incorrectly imported products
REMOVE Product;code[unique=true];$catalogVersion
;BAD-PROD-001;
;BAD-PROD-002;
;BAD-PROD-003;

# Or remove by query
"#% impex.initScript(""

import de.hybris.platform.servicelayer.search.FlexibleSearchQuery;

query = new FlexibleSearchQuery(""SELECT {pk} FROM {Product} WHERE {creationtime} > '2024-01-15 10:00:00'"");
results = flexibleSearchService.search(query).getResult();
for (item : results) {
    modelService.remove(item);
}

"");"
```

---

## Testing the Migration

### Migration Dry Runs

Run the full migration against a copy of production data at least three times before go-live:

| Dry Run | Purpose | Environment |
|---------|---------|-------------|
| #1 | Identify data quality issues, fix transformations | Dev |
| #2 | Measure timing, fix performance issues | Staging (production-sized DB) |
| #3 | Final validation, rehearse go-live steps | Pre-production |

### Automated Verification Suite

```java
@Test
public void verifyProductMigration() {
    // Expected counts from source system analysis
    assertEquals(50000, countProducts(), "Product count mismatch");
    assertEquals(2000, countCategories(), "Category count mismatch");
    assertEquals(150000, countMedia(), "Media count mismatch");
}

@Test
public void verifyNoOrphanedProducts() {
    List<String> orphans = findProductsWithoutCategories();
    assertTrue(orphans.isEmpty(), 
        "Found " + orphans.size() + " products without categories: " + 
        orphans.subList(0, Math.min(10, orphans.size())));
}

@Test
public void verifyPriceIntegrity() {
    List<String> noPriceProducts = findProductsWithoutPrices();
    assertTrue(noPriceProducts.isEmpty(),
        "Found " + noPriceProducts.size() + " products without prices");
}

@Test
public void verifySampleProducts() {
    // Spot-check known products from source system
    Product camera = productService.getProductForCode("CAM-001");
    assertNotNull(camera);
    assertEquals("Professional DSLR", camera.getName());
    assertNotNull(camera.getPicture(), "Missing product image");
    assertFalse(camera.getSupercategories().isEmpty(), "Missing categories");
}
```

---

## Best Practices

1. **Never migrate directly to Online catalog** — always import into Staged and synchronize to Online after verification.

2. **Idempotent imports** — use `INSERT_UPDATE` so you can re-run imports safely if they fail partway through.

3. **Validate before importing** — catch data issues in the transformation layer, not during import.

4. **Keep source data snapshots** — archive the exact source data used for each migration run. You'll need it for debugging.

5. **Document every transformation** — when you convert `weight_lbs` to kilograms or map `active=true` to `approvalStatus=approved`, document it. Six months later, someone will ask why a value looks wrong.

6. **Plan for delta migrations** — there's always a gap between the initial load and go-live. Plan how changes made in the source system during this window will be captured.

7. **Test with production-scale data** — migration that works with 1,000 products may fail with 500,000 due to memory, timeouts, or performance.

8. **Have a rollback plan** — and test it. A backup you've never restored is not a backup.

---

## Summary

Data migration is project-critical work that determines whether go-live succeeds or fails. The key principles:

1. **Plan thoroughly** — inventory all data, map dependencies, document transformations
2. **Build automated pipelines** — manual migration doesn't scale and isn't repeatable
3. **Validate at every stage** — before import, during import, and after import
4. **Respect dependency order** — categories before products, products before prices
5. **Use ImpEx properly** — `INSERT_UPDATE` for idempotency, batch processing for volume
6. **Rehearse the full migration** — at least three dry runs with production-scale data
7. **Always have a rollback plan** — database backups plus targeted ImpEx reversal
8. **Import to Staged, verify, then sync** — never import directly to the Online catalog

The teams that treat migration as a first-class engineering effort — with automated testing, repeatable pipelines, and thorough validation — are the ones that have smooth go-lives.