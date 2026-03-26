# Mastering ImpEx in SAP Commerce: From Basics to Advanced Patterns

ImpEx is the language of SAP Commerce data. Whether you're setting up a fresh project, loading product catalogs, configuring promotions, migrating customer data, or debugging a production issue through HAC — ImpEx is the tool you'll reach for. It's the backbone of data management in the platform, and mastering it is non-negotiable for any serious SAP Commerce developer.

This article takes you from the first `INSERT_UPDATE` to scripted bulk imports with performance tuning. Every concept comes with practical examples you can copy, adapt, and use immediately.

---

## What Is ImpEx?

ImpEx (Import/Export) is SAP Commerce's proprietary data manipulation language. Think of it as a CSV-like DSL that maps directly to the platform's type system. Unlike raw SQL, ImpEx operates at the model layer — it understands type hierarchies, relations, catalog versions, localized attributes, and interceptors.

Under the hood, ImpEx is processed by the `ImpExImportService` (interface: `de.hybris.platform.servicelayer.impex.ImportService`). The core engine lives in `de.hybris.platform.impex.jalo.imp.DefaultImportProcessor` for the legacy layer and `de.hybris.platform.impex.impl.DefaultImpExImportStrategy` for the service layer. When you paste ImpEx into HAC or load it from a file, the engine:

1. Parses the header line to determine the target type and columns
2. For each data line, resolves references and translators
3. Creates or updates items via `ModelService` (which triggers interceptors)
4. Reports successes and failures line by line

ImpEx is used everywhere in the platform:
- **Essential data**: System setup (currencies, languages, user groups) — loaded during `ant initialize`
- **Project data**: Catalog structure, CMS pages, Solr configuration — loaded during update
- **Sample data**: Demo products, test users — loaded for development/demo environments
- **Data migration**: Bulk loading customer data, product catalogs, order history
- **Runtime operations**: Backoffice data fixes, one-time corrections via HAC

---

## Basic Syntax: The Four Operations

### INSERT

Creates new items. Fails if an item with the same unique key already exists.

```impex
INSERT;Language;isocode[unique=true];name[lang=en];active
;;de;German;true
;;fr;French;true
;;es;Spanish;true
```

### INSERT_UPDATE

Creates items if they don't exist, updates them if they do. This is the most commonly used operation — it's idempotent, making it safe to run repeatedly.

```impex
INSERT_UPDATE Currency;isocode[unique=true];name[lang=en];digits;symbol
;USD;US Dollar;2;$
;EUR;Euro;2;€
;GBP;British Pound;2;£
```

### UPDATE

Updates existing items only. Fails if the item doesn't exist. Use this when you want to ensure you're not accidentally creating new items.

```impex
UPDATE Product;code[unique=true];approvalStatus(code)
;PRODUCT-001;approved
;PRODUCT-002;approved
```

### REMOVE

Deletes items matching the specified criteria.

```impex
REMOVE Product;code[unique=true];catalogVersion(catalog(id),version)
;OBSOLETE-001;myProductCatalog:Staged
;OBSOLETE-002;myProductCatalog:Staged
```

**Important**: REMOVE is permanent. There's no undo. Always test on a non-production environment first.

### Syntax Anatomy

```
INSERT_UPDATE Product ; code[unique=true] ; name[lang=en]  ; catalogVersion(catalog(id),version)
                      ; PROD-001          ; My Product      ; myProductCatalog:Staged
```

- **Header line**: Starts with the operation, followed by the type name, then column definitions separated by semicolons
- **Data lines**: Start with a semicolon (the first column is reserved for special directives), then values separated by semicolons
- **Comments**: Lines starting with `#` are comments
- **Blank lines**: Separate ImpEx blocks (each block has its own header)

---

## Header Modifiers Deep Dive

Header modifiers control how each column is interpreted. They appear in square brackets after the attribute qualifier.

### `[unique=true]`

Marks the column as part of the item's unique key for lookup. The engine uses unique columns to determine whether to INSERT or UPDATE.

```impex
INSERT_UPDATE Product;code[unique=true];catalogVersion(catalog(id),version)[unique=true];name[lang=en]
;PROD-001;myProductCatalog:Staged;Widget Alpha
```

Multiple unique columns form a composite key. For products, `code` + `catalogVersion` is the natural unique key.

### `[lang=xx]`

Specifies the language for localized attributes:

```impex
INSERT_UPDATE Product;code[unique=true];catalogVersion(catalog(id),version)[unique=true];name[lang=en];name[lang=de];name[lang=fr]
;PROD-001;myProductCatalog:Staged;Widget Alpha;Widget Alpha;Widget Alpha
```

### `[default=value]`

Sets a default value for the column. If a data line has an empty value for this column, the default is used:

```impex
INSERT_UPDATE Product;code[unique=true];catalogVersion(catalog(id),version)[unique=true][default=myProductCatalog:Staged];approvalStatus(code)[default=approved]
;PROD-001;;
;PROD-002;;
;PROD-003;;
```

All three products get `catalogVersion=myProductCatalog:Staged` and `approvalStatus=approved` without repeating the values.

### `[translator=class]`

Specifies a custom translator class for value interpretation. Translators implement `de.hybris.platform.impex.jalo.translators.AbstractValueTranslator`:

```impex
INSERT_UPDATE MediaContainer;qualifier[unique=true];catalogVersion(catalog(id),version)[unique=true];medias(code,catalogVersion(catalog(id),version))[translator=de.hybris.platform.impex.jalo.translators.MediaContainerTranslator]
;container-001;myContentCatalog:Staged;image-001:myContentCatalog:Staged,image-002:myContentCatalog:Staged
```

Common built-in translators:
- `de.hybris.platform.impex.jalo.translators.ConvertPlaintextToEncodedUserPasswordTranslator` — hashes passwords
- `de.hybris.platform.impex.jalo.translators.MediaDataTranslator` — handles media binary data
- `de.hybris.platform.impex.jalo.translators.UserRightsTranslator` — manages user access rights

### `[mode=append]`

For collection attributes, appends values instead of replacing them:

```impex
UPDATE Product;code[unique=true];supercategories(code, catalogVersion(catalog(id),version))[mode=append]
;PROD-001;newCategory:myProductCatalog:Staged
```

Without `mode=append`, the supercategories collection would be replaced entirely.

### `[allownull=true]`

Permits null/empty values. Without this modifier, empty values in data lines are skipped (the attribute is not modified):

```impex
UPDATE Product;code[unique=true];description[allownull=true]
;PROD-001;
```

This clears the `description` field. Without `[allownull=true]`, the empty value would be ignored.

### `[forceWrite=true]`

Forces the attribute to be written even if the model's modifier says otherwise (e.g., read-only attributes):

```impex
INSERT_UPDATE Order;code[unique=true];creationtime[forceWrite=true,dateformat=dd.MM.yyyy HH:mm:ss]
;ORDER-001;15.03.2025 14:30:00
```

### `[dateformat=pattern]`

Specifies the date format for date attributes:

```impex
INSERT_UPDATE CronJob;code[unique=true];startTime[dateformat=yyyy-MM-dd HH:mm:ss]
;myCronJob;2025-06-01 03:00:00
```

### `[numberformat=pattern]`

Specifies number formatting:

```impex
INSERT_UPDATE PriceRow;product(code)[unique=true];price[numberformat=#,##0.00];currency(isocode)[unique=true]
;PROD-001;1299.99;USD
```

### `[cellDecorator=class]`

Applies a decorator to cell values before processing. Decorators implement `de.hybris.platform.impex.jalo.imp.DefaultImportCellDecorator`:

```impex
INSERT_UPDATE Product;code[unique=true][cellDecorator=com.mycompany.impex.UpperCaseDecorator];name[lang=en]
;prod-001;My Product
```

### `[virtual=true]`

Marks a column as virtual — it doesn't map to a real attribute. Useful with BeanShell scripting:

```impex
INSERT_UPDATE Product;code[unique=true];@customLogic[virtual=true]
;PROD-001;someValue
```

---

## Referencing Related Items

One of ImpEx's strengths is its ability to reference existing items through their attributes.

### Simple Reference

Reference an item by its unique attribute:

```impex
INSERT_UPDATE Product;code[unique=true];unit(code);catalogVersion(catalog(id),version)
;PROD-001;pieces;myProductCatalog:Staged
```

Here `unit(code)` means: "find the Unit whose `code` equals the provided value."

### Nested Reference

Reference through a chain of attributes:

```impex
;catalogVersion(catalog(id),version)
```

This means: "find the CatalogVersion whose `catalog`'s `id` equals the first value and whose `version` equals the second value."

The syntax is: `attribute(nestedAttribute1, nestedAttribute2)`, and nested values are separated by colons in the data:

```impex
;myProductCatalog:Staged
```

### Multi-Level Nesting

```impex
INSERT_UPDATE Product;code[unique=true];catalogVersion(catalog(id),version);supercategories(code,catalogVersion(catalog(id),version))
;PROD-001;myProductCatalog:Staged;electronics:myProductCatalog:Staged
```

### Collection References

For multi-valued attributes, separate items with commas:

```impex
UPDATE Product;code[unique=true];supercategories(code,catalogVersion(catalog(id),version))
;PROD-001;category1:myProductCatalog:Staged,category2:myProductCatalog:Staged,category3:myProductCatalog:Staged
```

### PK References

You can reference items by their PK (primary key) directly, though this is rare and fragile:

```impex
UPDATE Product;code[unique=true];unit(&unitRef)
;PROD-001;8796093054980
```

Using `&` prefix creates a named reference. More commonly, `&` is used for document ID references within the same ImpEx file:

```impex
INSERT_UPDATE Address;&addrRef;owner(uid);streetname;town
;addr1;john@example.com;123 Main St;Berlin

INSERT_UPDATE Customer;uid[unique=true];defaultPaymentAddress(&addrRef)
;john@example.com;addr1
```

The `&addrRef` creates an in-document reference that lets you link items created in the same ImpEx execution.

---

## Macros and Variables

Macros make ImpEx files clean, maintainable, and DRY. A macro is defined with `$` prefix and expanded at parse time.

### Basic Macros

```impex
$productCatalog=myProductCatalog
$catalogVersion=catalogVersion(catalog(id[default=$productCatalog]),version[default='Staged'])[unique=true,default=$productCatalog:Staged]
$lang=en

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=$lang];description[lang=$lang]
;PROD-001;;Widget Alpha;A high-quality widget
;PROD-002;;Widget Beta;An even better widget
;PROD-003;;Widget Gamma;The ultimate widget
```

### Catalog Version Macro (The Most Common Pattern)

Almost every ImpEx file dealing with products or content uses this pattern:

```impex
$productCatalog=electronicsProductCatalog
$catalogVersion=catalogVersion(catalog(id[default=$productCatalog]),version[default='Staged'])[unique=true,default=$productCatalog:Staged]
$supercategories=supercategories(code,catalogVersion(catalog(id[default=$productCatalog]),version[default='Staged']))
$prices=Europe1prices[translator=de.hybris.platform.europe1.jalo.impex.Europe1PricesTranslator]
$taxGroup=Europe1PriceFactory_PTG(code)[default=eu-vat-full]

INSERT_UPDATE Product;code[unique=true];$catalogVersion;$supercategories;$prices;$taxGroup;approvalStatus(code)[default=approved]
;PROD-001;;electronics;1 pieces = 29.99 USD N;
;PROD-002;;electronics,accessories;1 pieces = 49.99 USD N;
```

### Content Catalog Macro

```impex
$contentCatalog=electronicsContentCatalog
$contentCV=catalogVersion(CatalogVersion.catalog(Catalog.id[default=$contentCatalog]),CatalogVersion.version[default=Staged])[default=$contentCatalog:Staged]
```

### Environment-Specific Macros

Macros can be overridden by system properties or configuration. In the ImpEx file:

```impex
$siteUrl=https://localhost:9002
```

Then in `local.properties`:

```properties
impex.import.siteUrl=https://production.mysite.com
```

The platform resolves `$siteUrl` from the configuration, falling back to the value defined in the ImpEx file.

---

## Advanced Patterns

### Collections and Maps

**Collection types (List, Set):**

```impex
# Setting a collection of delivery countries
INSERT_UPDATE BaseStore;uid[unique=true];deliveryCountries(isocode)
;electronics;US,DE,FR,GB,JP
```

**Map types:**

```impex
INSERT_UPDATE Product;code[unique=true];$catalogVersion;customAttributes[map-delimiter=|][key2value-delimiter=->]
;PROD-001;;color->red|size->large|material->cotton
```

### Media Import

**From the classpath (JAR/extension resources):**

```impex
$siteResource=jar:com.mycompany.initialdata.setup.InitialDataSystemSetup&/myextension/import/images

INSERT_UPDATE Media;code[unique=true];@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime[default='image/jpeg'];$catalogVersion;folder(qualifier)[default='images']
;product-001-image;$siteResource/product-001.jpg;
;product-002-image;$siteResource/product-002.jpg;
;logo;$siteResource/logo.png;image/png
```

**From a URL:**

```impex
INSERT_UPDATE Media;code[unique=true];@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime[default='image/jpeg'];$catalogVersion
;banner-summer;https://cdn.mycompany.com/banners/summer-sale.jpg;
```

**From the file system (Hot Folder):**

```impex
INSERT_UPDATE Media;code[unique=true];@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime[default='image/jpeg'];$catalogVersion
;product-image;file:///opt/media/product-image.jpg;
```

### Conditional Operations with BeanShell

ImpEx supports `#%` directives for scripting:

```impex
#% impex.enableCodeExecution(true);

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en]
#% if: de.hybris.platform.jalo.c2l.C2LManager.getInstance().getLanguageByIsoCode("de") != null
;PROD-001;;Widget Alpha
#% endif:
```

### Conditional `if` with Groovy

```impex
#% impex.enableCodeExecution(true);
#% impex.info("Starting conditional import...");

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en]
#% groovy: if (de.hybris.platform.util.Config.getBoolean("myfeature.enabled", false)) {
;FEATURE-PROD-001;;Special Feature Product
#% groovy: }
```

---

## BeanShell and Groovy Scripting within ImpEx

ImpEx becomes extremely powerful when combined with scripting. You can execute arbitrary Java/Groovy code during import.

### Enabling Code Execution

```impex
#% impex.enableCodeExecution(true);
```

This must appear at the top of the ImpEx file. **Security note**: Code execution should be disabled in production ImpEx loaded via Hot Folders. It's primarily for development and HAC usage.

### `beforeEach` and `afterEach` Hooks

These hooks execute code before/after each line is processed:

```impex
#% impex.enableCodeExecution(true);

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en];approvalStatus(code)
#% beforeEach: line.clear();
#% java.lang.String code = line.getValueEntry("code").toString();
#% if (code.startsWith("TEST-")) { line.clear(); impex.info("Skipping test product: " + code); }
;PROD-001;;Real Product;approved
;TEST-001;;Test Product;approved
;PROD-002;;Another Real Product;approved
```

### Generating Data with Groovy

```impex
#% impex.enableCodeExecution(true);

INSERT_UPDATE Customer;uid[unique=true];name;customerID
#% groovy:
#% (1..100).each { i ->
#%   String uid = "test.user.${i}@example.com"
#%   String name = "Test User ${i}"
#%   String custId = String.format("CUST-%05d", i)
#%   impex.exportItems(";" + uid + ";" + name + ";" + custId)
#% }
```

### Using `impex` Object API

The `impex` object available in scripts provides useful methods:

```impex
#% impex.enableCodeExecution(true);

# Log information
#% impex.info("Starting product import at " + new java.util.Date());

# Include another ImpEx file
#% impex.includeExternalData("path/to/other-file.impex", "UTF-8");

# Get the current line number
#% impex.info("Processing line: " + impex.getCurrentLineNumber());

# Export items (useful for generating data)
#% impex.exportItems(";VALUE1;VALUE2;VALUE3");
```

### Post-Import Groovy Script

Execute a script after all lines are imported:

```impex
#% impex.enableCodeExecution(true);

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en]
;PROD-001;;Widget Alpha

"#% afterEach: end"
#% groovy:
#% import de.hybris.platform.core.Registry
#% def modelService = Registry.getApplicationContext().getBean("modelService")
#% def flexibleSearchService = Registry.getApplicationContext().getBean("flexibleSearchService")
#% impex.info("Import completed. Running post-import validation...")
#% def query = new de.hybris.platform.servicelayer.search.FlexibleSearchQuery("SELECT COUNT({pk}) FROM {Product}")
#% def result = flexibleSearchService.search(query)
#% impex.info("Total products in system: " + result.getResult().get(0))
```

---

## Performance Optimization

When importing large datasets (millions of rows), default ImpEx settings won't cut it. Here's how to tune for bulk operations.

### Batch Mode and Worker Threads

```properties
# In local.properties or passed as system properties

# Number of parallel import threads (default: 1)
impex.import.workers=8

# Batch size for parallel processing
impex.import.config.batchsize=1000

# Enable legacy mode for faster imports (bypasses some ServiceLayer overhead)
impex.legacy.mode=true
```

### Disabling Interceptors for Bulk Imports

Interceptors add overhead per item. For bulk migrations, you can disable them:

```impex
#% impex.enableCodeExecution(true);
#% impex.setValidationMode(ImpExManager.getImportStrictMode(de.hybris.platform.impex.constants.ImpExConstants.Enumerations.ImpExValidationModeEnum.IMPORT_STRICT));

# Disable specific interceptors
UPDATE GenericItem[disable.interceptor.beans='myPrepareInterceptor,myValidateInterceptor'];code[unique=true]
;dummy
```

Or use the `disable.interceptor.types` header modifier:

```impex
INSERT_UPDATE Product[disable.interceptor.types=validate];code[unique=true];$catalogVersion;name[lang=en]
;PROD-001;;Bulk Import Product
```

Interceptor type values: `validate`, `prepare`, `initdefaults`, `remove`, `load`.

### Reducing Database Round-Trips

```properties
# Enable distributed ImpEx (CCv2)
impex.distributed.enabled=true

# JDBC batch size
db.tableprefix=
jdbc.batch.size=100
```

### Caching During Import

```properties
# Increase type system cache during import
cache.main=200000
cache.main.eviction=LRU

# Increase query cache
flexiblesearch.cache.size=100000
```

### Large File Strategy

For very large ImpEx files (10M+ rows), split them:

```bash
# Split a large file into chunks of 100,000 lines each
split -l 100000 large-product-import.impex chunk_

# Add the header to each chunk
for file in chunk_*; do
    cat header.impex "$file" > "import_${file}.impex"
done
```

Then import each chunk sequentially or in parallel via Hot Folders.

---

## Organizing ImpEx Files

SAP Commerce has a convention-based structure for ImpEx files within extensions.

### Essential Data

Files matching `essentialdata-*.impex` in `resources/impex/` are loaded during **system initialization** (`ant initialize`) and **system update** (`ant updatesystem`). Use for:

- Type system configurations
- User groups and basic permissions
- System-level settings

```
myextension/
└── resources/
    └── impex/
        ├── essentialdata-myextension.impex
        └── essentialdata-user-groups.impex
```

### Project Data

Files matching `projectdata-*.impex` are loaded during **system initialization** only. Use for:

- Catalog structure
- CMS page templates and pages
- Solr search configuration
- Base store configuration

```
myextension/
└── resources/
    └── impex/
        └── projectdata-store-setup.impex
```

### Sample Data

Loaded by the setup classes (typically via `CoreSystemSetup` or custom `SystemSetup` annotations). Use for demo data, test data:

```java
@SystemSetup(extension = "myextension")
public class MyExtensionSystemSetup extends AbstractSystemSetup {
    
    @SystemSetup(type = Type.PROJECT, process = Process.ALL)
    public void createProjectData(final SystemSetupContext context) {
        importImpexFile(context, "/myextension/import/sampledata/products.impex");
        importImpexFile(context, "/myextension/import/sampledata/users.impex");
    }
}
```

### Recommended File Organization

```
myextension/
└── resources/
    └── myextension/
        └── import/
            ├── coredata/
            │   ├── common/
            │   │   ├── essential-data.impex
            │   │   ├── countries.impex
            │   │   └── currencies.impex
            │   └── stores/
            │       └── mystore/
            │           ├── store.impex
            │           ├── warehouses.impex
            │           └── delivery-modes.impex
            ├── sampledata/
            │   ├── productCatalogs/
            │   │   └── myProductCatalog/
            │   │       ├── categories.impex
            │   │       ├── products.impex
            │   │       ├── prices.impex
            │   │       ├── stock.impex
            │   │       └── media.impex
            │   ├── contentCatalogs/
            │   │   └── myContentCatalog/
            │   │       ├── cms-content.impex
            │   │       └── cms-responsive-content.impex
            │   └── stores/
            │       └── mystore/
            │           ├── promotions.impex
            │           └── solr.impex
            └── cockpit/
                └── backoffice-configuration.impex
```

---

## Export ImpEx

ImpEx isn't just for importing — you can export data too. This is invaluable for data migration, backup, and debugging.

### Export from HAC

In the Hybris Administration Console (HAC), navigate to **Console → ImpEx Export**. Enter an export script:

{% raw %}
```impex
"#% impex.setTargetFile(""product-export.csv"");"
INSERT_UPDATE Product;code[unique=true];name[lang=en];description[lang=en];approvalStatus(code);catalogVersion(catalog(id),version)
"#% impex.exportItems(""SELECT {code},{name[en]},{description[en]},{approvalStatus},{catalogVersion} FROM {Product} WHERE {catalogVersion} IN ({{SELECT {pk} FROM {CatalogVersion} WHERE {catalog} IN ({{SELECT {pk} FROM {Catalog} WHERE {id}='myProductCatalog'}}) AND {version}='Online'}})"");"
```
{% endraw %}

### Export via FlexibleSearch

A simpler approach — run a FlexibleSearch query in HAC and use the results to generate ImpEx:

```sql
SELECT {p.code}, {p.name[en]}, {cv.version}, {cat.id}
FROM {Product AS p
    JOIN CatalogVersion AS cv ON {p.catalogVersion} = {cv.pk}
    JOIN Catalog AS cat ON {cv.catalog} = {cat.pk}}
WHERE {cat.id} = 'myProductCatalog' AND {cv.version} = 'Online'
```

### Programmatic Export

```java
@Resource
private ExportService exportService;

public void exportProducts() {
    ImpExExportMedia exportMedia = new ImpExExportMedia();
    exportMedia.setExportScript(
        "INSERT_UPDATE Product;code[unique=true];name[lang=en];catalogVersion(catalog(id),version)\n" +
        "#% impex.exportItems(\"SELECT {code},{name[en]},{catalogVersion} FROM {Product}\");"
    );
    ExportResult result = exportService.exportData(exportMedia);
    // Process result...
}
```

---

## Common Pitfalls and Troubleshooting

### "Could not resolve item expression"

**Symptom**: `Can not resolve any more lines... Could not resolve item expression 'myProductCatalog:Staged'`

**Causes**:
1. The referenced item doesn't exist. Check that the catalog, catalog version, or referenced item has been imported first.
2. Wrong attribute path. Verify the nested reference syntax matches the type system.
3. Case sensitivity. ImpEx references are case-sensitive.

**Fix**: Import items in dependency order — catalogs before catalog versions, catalog versions before products.

### Duplicate Unique Keys

**Symptom**: `item with same key attributes already exists`

**Causes**:
1. Using `INSERT` instead of `INSERT_UPDATE`
2. Incorrect unique key combination — missing a `[unique=true]` modifier

**Fix**: Use `INSERT_UPDATE` for idempotent imports. Ensure all attributes that form the natural key are marked `[unique=true]`.

### Encoding Issues

**Symptom**: Garbled characters, especially with German (ä, ö, ü), French (é, è), or Asian characters.

**Fix**: Always save ImpEx files in UTF-8 encoding. In HAC, the text area uses the browser's encoding. When importing from files:

```properties
# Ensure UTF-8 encoding
impex.import.file.encoding=UTF-8
```

In the ImpEx file itself:

```impex
#% impex.setLocale(new java.util.Locale("en","US"));
```

### Unresolved Lines (Two-Pass Resolution)

ImpEx processes lines in multiple passes. If item A references item B and both are in the same file, A might fail on the first pass (before B exists) but succeed on the second pass.

**Symptom**: Warning about unresolved lines, but eventual success.

**Problem**: If circular references exist, lines may never resolve.

**Fix**: Order your ImpEx blocks so dependencies come first. If you have circular dependencies, break them into separate files or use document ID references (`&ref`).

### Date Format Mismatches

**Symptom**: `java.text.ParseException: Unparseable date`

**Fix**: Always specify the date format explicitly:

```impex
INSERT_UPDATE CronJob;code[unique=true];startTime[dateformat=dd.MM.yyyy HH:mm:ss]
;myCronJob;01.06.2025 03:00:00
```

Default date format is `dd.MM.yyyy HH:mm:ss` but this varies by locale. Be explicit.

### Header Parsing Errors

**Symptom**: `Unknown attribute 'xyz' at position N`

**Causes**:
1. Attribute doesn't exist on the type. Check `items.xml` or use HAC's Type System viewer.
2. Typo in the attribute name.
3. Missing required extension — the type exists but the attribute is added by an extension that isn't loaded.

### Performance Issues with Large Imports

**Symptom**: Import takes hours for a few hundred thousand records.

**Checklist**:
1. Are interceptors running unnecessary logic? Disable them for bulk imports.
2. Is the session catalog version set? Without it, every product lookup scans all catalog versions.
3. Are you importing media binaries inline? Split media import from data import.
4. Is the database under-provisioned? Check connection pool and query performance.
5. Are you using `INSERT_UPDATE` when `INSERT` would suffice? `INSERT` is faster because it skips the lookup phase.

---

## Real-World Tips from Production Experience

### Tip 1: Always Use Macros for Catalog Versions

Hard-coding catalog references in every line is error-prone. Define them once:

```impex
$productCatalog=myProductCatalog
$catalogVersion=catalogVersion(catalog(id[default=$productCatalog]),version[default='Staged'])[unique=true,default=$productCatalog:Staged]
```

### Tip 2: Include Cleanup Blocks

Before loading sample data, remove stale data:

```impex
REMOVE Product;code[unique=true];$catalogVersion
;OLD-PRODUCT-001;
;OLD-PRODUCT-002;

INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en]
;NEW-PRODUCT-001;;New Product
```

### Tip 3: Use Comments Liberally

```impex
# ================================================
# Product Catalog: Electronics
# Author: J. Smith
# Date: 2025-03-15
# Description: Initial product load for Q2 launch
# Dependencies: categories.impex must be loaded first
# ================================================
```

### Tip 4: Validate Before Bulk Import

Run a small subset first:

```impex
# Test with first 5 records
INSERT_UPDATE Product;code[unique=true];$catalogVersion;name[lang=en]
;TEST-001;;Test Product 1
;TEST-002;;Test Product 2
;TEST-003;;Test Product 3
;TEST-004;;Test Product 4
;TEST-005;;Test Product 5
```

If it works, load the full file. If it fails, you save hours of debugging.

### Tip 5: ImpEx for Quick Production Fixes

When you need to fix data in production quickly (with proper approvals), ImpEx via HAC is often the fastest path:

```impex
# Emergency fix: correct pricing for product PROD-XYZ
# JIRA: PROJ-1234
# Approved by: Product Manager
# Date: 2025-03-15

UPDATE PriceRow;product(code)[unique=true];currency(isocode)[unique=true];price;catalogVersion(catalog(id),version)[unique=true]
;PROD-XYZ;USD;29.99;myProductCatalog:Online
```

Always document the reason, approval, and ticket reference in comments.

### Tip 6: Avoid Storing Passwords in ImpEx Files

Use the password translator for user imports:

```impex
INSERT_UPDATE Employee;uid[unique=true];password[translator=de.hybris.platform.impex.jalo.translators.ConvertPlaintextToEncodedUserPasswordTranslator];groups(uid)
;admin-user;tempPassword123;admingroup
```

Never commit plain-text passwords to version control. For production, use SSO or other authentication mechanisms.

### Tip 7: Use ImpEx Includes for Modularity

Break large imports into focused files and include them:

```impex
#% impex.includeExternalDataMedia("categories.impex", "UTF-8");
#% impex.includeExternalDataMedia("products.impex", "UTF-8");
#% impex.includeExternalDataMedia("prices.impex", "UTF-8");
#% impex.includeExternalDataMedia("stock.impex", "UTF-8");
```

---

## Summary

ImpEx is deceptively simple in syntax but enormously powerful in practice. The key takeaways:

1. **`INSERT_UPDATE` is your default** — it's idempotent and safe for repeated execution
2. **Macros are essential** — define catalog versions, content catalogs, and repeated references as macros
3. **Header modifiers control behavior** — `[unique=true]`, `[default=value]`, `[translator=class]`, `[mode=append]` are your primary tools
4. **Reference syntax follows the type system** — nested references like `catalogVersion(catalog(id),version)` map directly to the data model
5. **Scripting unlocks advanced scenarios** — BeanShell and Groovy within ImpEx handle conditional logic, data generation, and post-import operations
6. **Performance tuning matters for bulk operations** — worker threads, batch sizes, and interceptor management make the difference between hours and minutes
7. **Organization and documentation are non-negotiable** — use the essentialdata/projectdata/sampledata convention and comment everything

Master ImpEx, and you'll be the person everyone comes to when data needs to move.