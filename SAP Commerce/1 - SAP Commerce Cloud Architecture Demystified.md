---
layout: default
title: "SAP Commerce Cloud Architecture Demystified: A Practical Guide for Developers"
description: "A deep dive into SAP Commerce Cloud architecture — extensions, type system, service layer, and design patterns explained for Java developers."
---

# SAP Commerce Cloud Architecture Demystified: A Practical Guide for Developers

If you're a Java developer stepping into the SAP Commerce world — or a mid-level Commerce developer who wants to truly understand what's happening under the hood — this article is for you. We'll go beyond surface-level overviews and dig into the internals: the class hierarchies, the extension loading mechanism, the type system's relationship to the database, and the design patterns that make the platform tick.

---

## What Is SAP Commerce Cloud?

SAP Commerce Cloud (formerly Hybris) is a Java-based enterprise e-commerce platform. It sits within the SAP Customer Experience (CX) portfolio alongside SAP CDC (Customer Data Cloud), SAP Emarsys (marketing), SAP Sales Cloud, and SAP Service Cloud.

At its core, SAP Commerce is:

- A **Spring-based application** running on a Servlet container (embedded Tomcat)
- A **dynamic type system** that generates Java model classes and database schemas from XML definitions
- An **extension framework** that allows modular, composable customization
- A **headless commerce engine** exposing OCC REST APIs consumed by Spartacus (Composable Storefront) or any custom frontend

SAP Commerce is not a lightweight microservice — it's a monolithic (or "modulith") application designed for complex B2C and B2B commerce scenarios where product modeling, pricing, promotions, order management, and content management all need deep integration.

---

## High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│   Composable Storefront (Spartacus)  │  Mobile  │  Custom SPA  │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTPS (REST/JSON)
┌──────────────────────▼──────────────────────────────────────────┐
│                    OCC REST API LAYER                            │
│   @Controller classes in ycommercewebservices / custom OCC ext   │
│   WsDTO objects  │  Orika Mappers  │  OAuth2 (spring-security)  │
└──────────────────────┬──────────────────────────────────────────┘
                       │ Java method calls
┌──────────────────────▼──────────────────────────────────────────┐
│                    APPLICATION LAYER                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Facades     │  │  Services    │  │  Strategies / Hooks    │ │
│  │ (DTOs out)   │→ │ (Business    │→ │ (FindPriceStrategy,    │ │
│  │              │  │  Logic)      │  │  AddToCartStrategy...) │ │
│  └─────────────┘  └──────┬───────┘  └────────────────────────┘ │
│                          │                                      │
│  ┌───────────────────────▼───────────────────────────────────┐  │
│  │  DAOs (FlexibleSearchQuery → GenericSearchService)        │  │
│  └───────────────────────┬───────────────────────────────────┘  │
│                          │                                      │
│  ┌───────────────────────▼───────────────────────────────────┐  │
│  │  Model Objects (generated from items.xml)                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────────┘
                       │ JDBC
┌──────────────────────▼──────────────────────────────────────────┐
│                    PERSISTENCE LAYER                             │
│   RDBMS (SAP HANA / MySQL / HSQLDB / Oracle / PostgreSQL)       │
│   + Solr (product search)  + Media Storage (local / S3 / Azure) │
└─────────────────────────────────────────────────────────────────┘
```

### Client Layer

The client layer is anything that consumes Commerce APIs. In modern projects, this is typically **Composable Storefront (Spartacus)** — an Angular SPA that communicates exclusively via OCC REST APIs. Legacy projects may still use the JSP-based Accelerator storefront, which runs inside the Commerce JVM itself.

### OCC REST API Layer

The OCC (Omni Commerce Connect) layer exposes RESTful endpoints under `/occ/v2/`. The entry point classes live in the `ycommercewebservices` extension (or your custom OCC extension). Key internals:

- **Controllers** extend Spring's `@Controller` and use `@RequestMapping` annotations
- **WsDTO** classes (Data Transfer Objects) are the API contract — generated or hand-crafted POJOs annotated for serialization
- **Orika Mappers** (`Converter<Source, Target>`) translate between internal Models/Data objects and WsDTOs
- **OAuth2** authentication is handled via Spring Security OAuth, configured in `ycommercewebservices` Spring context

### Application Layer

This is where business logic lives. The critical class hierarchy:

- **Facades** (e.g., `DefaultProductFacade`, `DefaultCartFacade`) — orchestrate multiple services and convert Models into Data objects (DTOs). Facades implement interfaces like `ProductFacade`, `CartFacade`.
- **Services** (e.g., `DefaultProductService`, `DefaultCartService`) — contain business logic. They extend `AbstractBusinessService` or implement interfaces like `ProductService`, `CartService`. They operate on **Model** objects.
- **DAOs** (e.g., `DefaultProductDao`) — execute `FlexibleSearchQuery` objects via `FlexibleSearchService` and return Model objects.
- **Strategies** — pluggable business rules. For example, `FindPriceStrategy` determines product pricing, `CommerceAddToCartStrategy` handles add-to-cart logic.

### Platform Layer

Under all the commerce-specific code lies the **platform** itself. Key platform classes:

- `Registry` — the bootstrap class. `Registry.activateStandaloneMode()` or `Registry.activateMasterTenant()` starts the platform.
- `Tenant` — represents an isolated runtime context. `MasterTenant` is the primary tenant.
- `TypeManager` — manages the runtime type system, loaded from `items.xml` definitions across all extensions.
- `ModelService` — the persistence gateway. `modelService.save(model)`, `modelService.remove(model)`, `modelService.create(MyModel.class)`.

### Persistence Layer

SAP Commerce supports multiple databases through JDBC. In CCv2 (Cloud), **SAP HANA** is the standard database. The platform generates DDL from the type system definitions — you never write `CREATE TABLE` statements manually.

**Solr** (via the `solrfacetsearch` extension) provides full-text product search with faceting. **ZooKeeper** manages the Solr cluster in CCv2.

---

## The Extension System

The extension system is SAP Commerce's module architecture. Everything — from the platform kernel to your custom code — is packaged as an extension.

### Extension Anatomy

Every extension follows this structure:

```
myextension/
├── extensioninfo.xml          # Extension metadata, dependencies
├── buildcallbacks.xml         # Ant build hooks
├── resources/
│   ├── myextension-items.xml  # Type definitions
│   ├── myextension-spring.xml # Spring bean definitions
│   ├── localization/          # i18n property files
│   └── impex/                 # ImpEx data files
│       ├── essentialdata-myextension.impex
│       └── projectdata-myextension.impex
├── src/                       # Java source code
├── testsrc/                   # Test source code
├── web/
│   ├── src/                   # Web-tier Java source
│   ├── webroot/               # JSP, static resources
│   └── WEB-INF/
│       └── myextension-web-spring.xml
└── gensrc/                    # Generated model classes (output)
```

The `extensioninfo.xml` is the extension's manifest:

```xml
<extensioninfo>
    <extension abstractclassprefix="Generated"
               classprefix="MyExtension"
               name="myextension"
               jaloclass="com.mycompany.jalo.MyExtensionManager"
               managersuperclass="de.hybris.platform.jalo.extension.Extension">
        <requires-extension name="commerceservices"/>
        <requires-extension name="catalog"/>
        <meta key="backoffice-module" value="true"/>
    </extension>
</extensioninfo>
```

The `requires-extension` elements define the dependency graph — the platform resolves these at build time and startup to determine load order.

### Extension Categories

1. **Platform Extensions** (`bin/platform/ext/`): Core infrastructure — `core`, `processing`, `scripting`, `validation`, `Europe1` (pricing), `catalog`, `commons`, etc.

2. **Commerce Extensions** (`bin/modules/`): Business functionality organized in module groups:
   - `bin/modules/commerce-services/` — cart, order, pricing, stock
   - `bin/modules/search-and-navigation/` — Solr integration
   - `bin/modules/web-content-management-system/` — CMS
   - `bin/modules/b2b-commerce/` — B2B-specific functionality
   - `bin/modules/coupon/`, `bin/modules/promotion-engine/`, etc.

3. **Custom Extensions** (`bin/custom/`): Your project-specific code. Created using the `extgen` ant target:

```bash
ant extgen -Dinput.template=ycommercewebservices -Dinput.name=myoccextension -Dinput.package=com.mycompany.occ
```

4. **AddOns**: A special extension type that overlays files onto another extension (typically a storefront). AddOns are installed via `ant addoninstall` and copy their web resources into the target extension. They're being phased out in favor of Composable Storefront customization, but remain relevant for Backoffice and legacy storefront projects.

### Extension Loading and Initialization

At startup, the platform:

1. Reads `localextensions.xml` to determine which extensions are active
2. Resolves the dependency graph from all `extensioninfo.xml` files
3. Loads extensions in dependency order
4. For each extension: loads Spring context (`*-spring.xml`), registers types from `*-items.xml`, and initializes the extension manager

The class `ExtensionManager` (accessed via `Registry.getCurrentTenantNoFallback().getExtensionManager()`) holds the list of loaded extensions. The `PlatformConfig` class reads `localextensions.xml` and resolves paths.

In CCv2, `localextensions.xml` is generated from the `extensions` list in `manifest.json`.

---

## The Type System

The type system is what makes SAP Commerce fundamentally different from a standard JPA/Hibernate application. Instead of writing `@Entity` classes and Flyway migrations, you define your data model in `items.xml` files, and the platform generates both Java Model classes and database DDL.

### items.xml Deep Dive

Here's a realistic `items.xml` example:

```xml
<items xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="items.xsd">

    <enumtypes>
        <enumtype code="LoyaltyTier" autocreate="true" generate="true" dynamic="true">
            <value code="BRONZE"/>
            <value code="SILVER"/>
            <value code="GOLD"/>
            <value code="PLATINUM"/>
        </enumtype>
    </enumtypes>

    <itemtypes>
        <itemtype code="Customer" autocreate="false" generate="false">
            <!-- Extending the existing Customer type -->
            <attributes>
                <attribute qualifier="loyaltyTier" type="LoyaltyTier">
                    <modifiers optional="true"/>
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="loyaltyPoints" type="java.lang.Integer">
                    <defaultvalue>Integer.valueOf(0)</defaultvalue>
                    <modifiers optional="true"/>
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="loyaltyScore" type="java.lang.Double">
                    <modifiers read="true" write="false" optional="true"/>
                    <persistence type="dynamic" attributeHandler="customerLoyaltyScoreHandler"/>
                </attribute>
            </attributes>
        </itemtype>

        <itemtype code="LoyaltyTransaction" autocreate="true" generate="true"
                  extends="GenericItem"
                  jaloclass="com.mycompany.jalo.LoyaltyTransaction">
            <deployment table="loyalty_transactions" typecode="25000"/>
            <attributes>
                <attribute qualifier="code" type="java.lang.String">
                    <modifiers unique="true" optional="false"/>
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="points" type="java.lang.Integer">
                    <modifiers optional="false"/>
                    <persistence type="property"/>
                </attribute>
                <attribute qualifier="transactionDate" type="java.util.Date">
                    <modifiers optional="false"/>
                    <persistence type="property"/>
                </attribute>
            </attributes>
            <indexes>
                <index name="codeIdx" unique="true">
                    <key attribute="code"/>
                </index>
                <index name="dateIdx">
                    <key attribute="transactionDate"/>
                </index>
            </indexes>
        </itemtype>
    </itemtypes>

    <relations>
        <relation code="Customer2LoyaltyTransaction" localized="false">
            <sourceElement type="Customer" qualifier="customer" cardinality="one">
                <modifiers optional="false"/>
            </sourceElement>
            <targetElement type="LoyaltyTransaction" qualifier="loyaltyTransactions" cardinality="many"
                           collectiontype="list" ordered="true">
                <modifiers partof="true"/>
            </targetElement>
        </relation>
    </relations>
</items>
```

Key concepts:

- **`autocreate="false" generate="false"`** on `Customer`: We're extending an existing platform type, not creating a new one. The platform merges attributes from all extensions' `items.xml` files for the same type code.
- **`deployment`**: Maps the type to a specific database table and assigns a unique **typecode** (integer). Typecodes must be unique across the system (custom types should use 10000+).
- **`persistence type="dynamic"`**: The attribute isn't stored in the database. Instead, a `DynamicAttributeHandler` Spring bean computes the value at runtime.
- **`persistence type="property"`**: Standard database column storage.
- **`partof="true"`** on the relation target: Defines a composition relationship — deleting the Customer will cascade-delete their LoyaltyTransactions.

### How Types Map to the Database

When you run `ant initialize` or `ant updatesystem`, the platform's `SchemaGenerator` reads the merged type system and generates DDL:

- Each `itemtype` with a `deployment` gets its own table
- Types without a `deployment` store their attributes in the parent type's table (single-table inheritance)
- Localized attributes get a separate `*lp` table (e.g., `products` → `productslp` for localized product names)
- Many-to-many relations create link tables
- The `props` table stores attributes that overflow a type's dedicated table (configurable via `impex.legacy.mode`)

The internal class `TypeManagerImpl` builds the runtime `ComposedType` graph. You can inspect it at runtime:

```java
TypeManager typeManager = TypeManager.getInstance();
ComposedType customerType = typeManager.getComposedType("Customer");
Set<AttributeDescriptor> attributes = customerType.getAttributeDescriptors();
```

### Generated Model Classes

After modifying `items.xml` and running `ant build`, the platform generates:

- `GeneratedLoyaltyTransactionModel.java` in `gensrc/` — generated getters/setters, **do not edit**
- You create `LoyaltyTransactionModel.java` in `src/` extending the generated class — add custom logic here

The generated model class extends `ItemModel` and uses the `ModelService` internally for persistence:

```java
// Generated
public class GeneratedLoyaltyTransactionModel extends ItemModel {
    public static final String CODE = "code";
    public static final String POINTS = "points";
    
    public String getCode() {
        return getPersistenceContext().getPropertyValue(CODE);
    }
    
    public void setCode(String value) {
        getPersistenceContext().setPropertyValue(CODE, value);
    }
    // ...
}
```

The `PersistenceContext` handles dirty tracking and lazy loading transparently.

### Type System vs. Traditional ORM

| Aspect | JPA/Hibernate | SAP Commerce Type System |
|--------|--------------|--------------------------|
| Model definition | `@Entity` Java annotations | `items.xml` XML |
| Schema migration | Flyway/Liquibase | `ant updatesystem` |
| Model generation | N/A (you write entities) | Platform generates models from XML |
| Relationships | `@OneToMany`, `@ManyToMany` | `<relation>` elements in XML |
| Localization | Manual (separate tables or JSON) | Built-in via `localized:` prefix |
| Dynamic attributes | N/A | `DynamicAttributeHandler` |
| Query language | JPQL/HQL | FlexibleSearch |
| Inheritance | `@Inheritance` strategies | Type hierarchy in XML (`extends=`) |

---

## The Service Layer Pattern

The Service Layer is the standard architecture for all business logic in SAP Commerce. Understanding this layered pattern is essential.

### The Full Stack

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────┐
│  Controller                              │
│  Handles HTTP, validates request params  │
│  Delegates to Facade                     │
│  Returns WsDTO / View                    │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  Facade                                  │
│  Orchestrates multiple Services          │
│  Converts Models → Data DTOs             │
│  Uses Populators/Converters              │
│  Transaction boundary (typically)        │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  Service                                 │
│  Core business logic                     │
│  Operates on Models                      │
│  Calls DAOs for persistence              │
│  May call Strategies for pluggable logic │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  DAO (Data Access Object)                │
│  Executes FlexibleSearch queries         │
│  Returns Model objects                   │
│  No business logic here                  │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  ModelService                            │
│  Persistence gateway                     │
│  save(), remove(), create(), refresh()   │
│  Triggers Interceptor chain              │
└─────────────────────────────────────────┘
```

### Why This Layering Matters

1. **Facades** are the transaction boundary. A single facade method call typically represents one business transaction. This is where you annotate `@Transactional` if needed.

2. **Services** are reusable across different facades and channels. A `CartService` method can be called from both a storefront facade and a backoffice facade.

3. **DAOs** isolate data access. If you ever need to switch from FlexibleSearch to a stored procedure or an external service, only the DAO changes.

4. **Separation of Models and DTOs**: Models are the internal persistence representation. Data DTOs (e.g., `ProductData`, `CartData`) are the external contract. The `Populator`/`Converter` pattern bridges them (detailed below).

### Concrete Example

```java
// Controller (OCC layer)
@Controller
@RequestMapping("/users/{userId}/loyalty")
public class LoyaltyController {
    
    @Resource
    private LoyaltyFacade loyaltyFacade;
    
    @GetMapping("/transactions")
    @ResponseBody
    public LoyaltyTransactionListWsDTO getTransactions(
            @PathVariable String userId,
            @RequestParam(defaultValue = "DEFAULT") String fields) {
        List<LoyaltyTransactionData> transactions = loyaltyFacade.getTransactions(userId);
        return dataMapper.map(transactions, LoyaltyTransactionListWsDTO.class, fields);
    }
}

// Facade
public class DefaultLoyaltyFacade implements LoyaltyFacade {
    
    @Resource
    private LoyaltyService loyaltyService;
    @Resource
    private UserService userService;
    @Resource
    private Converter<LoyaltyTransactionModel, LoyaltyTransactionData> loyaltyTransactionConverter;
    
    @Override
    public List<LoyaltyTransactionData> getTransactions(String userId) {
        UserModel user = userService.getUserForUID(userId);
        List<LoyaltyTransactionModel> models = loyaltyService.getTransactionsForCustomer(user);
        return loyaltyTransactionConverter.convertAll(models);
    }
}

// Service
public class DefaultLoyaltyService implements LoyaltyService {
    
    @Resource
    private LoyaltyTransactionDao loyaltyTransactionDao;
    @Resource
    private ModelService modelService;
    
    @Override
    public List<LoyaltyTransactionModel> getTransactionsForCustomer(UserModel customer) {
        return loyaltyTransactionDao.findByCustomer(customer);
    }
    
    @Override
    public LoyaltyTransactionModel createTransaction(UserModel customer, int points) {
        LoyaltyTransactionModel tx = modelService.create(LoyaltyTransactionModel.class);
        tx.setCustomer(customer);
        tx.setPoints(points);
        tx.setTransactionDate(new Date());
        tx.setCode(generateUniqueCode());
        modelService.save(tx);
        return tx;
    }
}

// DAO
public class DefaultLoyaltyTransactionDao implements LoyaltyTransactionDao {
    
    @Resource
    private FlexibleSearchService flexibleSearchService;
    
    private static final String FIND_BY_CUSTOMER = 
        "SELECT {pk} FROM {LoyaltyTransaction} WHERE {customer} = ?customer ORDER BY {transactionDate} DESC";
    
    @Override
    public List<LoyaltyTransactionModel> findByCustomer(UserModel customer) {
        FlexibleSearchQuery query = new FlexibleSearchQuery(FIND_BY_CUSTOMER);
        query.addQueryParameter("customer", customer);
        SearchResult<LoyaltyTransactionModel> result = flexibleSearchService.search(query);
        return result.getResult();
    }
}
```

---

## Configuration Hierarchy

SAP Commerce has a sophisticated configuration system with a clear precedence order. Understanding this is critical for debugging "why is my property not taking effect?"

```
Priority (highest wins):
─────────────────────────────────────────────
1. System properties      (-Dproperty=value)
2. Environment variables  (via y_ prefix convention in CCv2)
3. local.properties        (project-level overrides)
4. Extension properties   (myextension/project.properties)
5. Platform defaults      (platform/project.properties)
─────────────────────────────────────────────
```

### In Practice

```properties
# platform/project.properties (platform defaults)
db.pool.maxActive=30

# myextension/project.properties (extension defaults)
my.feature.enabled=false

# config/local.properties (project overrides, NOT in source control for local dev)
db.pool.maxActive=50
my.feature.enabled=true
db.url=jdbc:mysql://localhost:3306/hybris

# In CCv2 manifest.json (environment-specific)
{
  "properties": [
    { "key": "db.pool.maxActive", "value": "100", "persona": "production" },
    { "key": "my.feature.enabled", "value": "true" }
  ]
}
```

Access configuration in code via:

```java
Config.getParameter("my.feature.enabled");  // returns String
Config.getBoolean("my.feature.enabled", false);  // with default
Config.getInt("db.pool.maxActive", 30);
```

The `Config` class reads from `ConfigIntf`, which loads all property sources at startup and merges them according to the precedence rules. In CCv2, environment-specific properties from `manifest.json` are injected as system properties or written to `local.properties` during the build.

---

## Multi-Tenancy Model

SAP Commerce supports multi-tenancy at the JVM level. Each tenant has its own:
- Database schema (or database)
- Type system cache
- Spring application context
- Session management
- Cache regions

### Master vs. Slave Tenants

- **Master Tenant** (`MasterTenant`): The primary tenant, always present. Your main commerce application runs here.
- **Slave Tenants**: Additional isolated tenants in the same JVM. The `junit` tenant is the most common slave tenant — it's used for integration tests with its own database and type system.

```properties
# Slave tenant configuration in local.properties
installed.tenants=junit

# junit tenant uses a separate database
junit.db.url=jdbc:hsqldb:mem:testDB
junit.db.driver=org.hsqldb.jdbcDriver
junit.db.username=sa
junit.db.password=
```

Tenant switching happens via `Registry.setCurrentTenant(tenant)`. The `JaloSession` is always tenant-scoped. In practice, most developers only interact with multi-tenancy through the junit tenant for testing.

---

## Key Design Patterns

### Populator / Converter Pattern

This is the most pervasive pattern in SAP Commerce. It separates the concern of converting Models to DTOs into small, composable units.

```java
// Converter interface (from platform)
public interface Converter<SOURCE, TARGET> extends Populator<SOURCE, TARGET> {
    TARGET convert(SOURCE source);
    TARGET convert(SOURCE source, TARGET prototype);
}

// Populator interface
public interface Populator<SOURCE, TARGET> {
    void populate(SOURCE source, TARGET target);
}
```

A typical converter delegates to a list of populators:

```java
// Spring configuration
<bean id="loyaltyTransactionConverter" parent="abstractPopulatingConverter">
    <property name="targetClass" value="com.mycompany.data.LoyaltyTransactionData"/>
    <property name="populators">
        <list>
            <ref bean="loyaltyTransactionBasicPopulator"/>
            <ref bean="loyaltyTransactionPointsPopulator"/>
        </list>
    </property>
</bean>

<bean id="loyaltyTransactionBasicPopulator" 
      class="com.mycompany.facades.populators.LoyaltyTransactionBasicPopulator"/>
```

```java
public class LoyaltyTransactionBasicPopulator 
    implements Populator<LoyaltyTransactionModel, LoyaltyTransactionData> {
    
    @Override
    public void populate(LoyaltyTransactionModel source, LoyaltyTransactionData target) {
        target.setCode(source.getCode());
        target.setTransactionDate(source.getTransactionDate());
    }
}
```

**Why this pattern matters**: You can add new populators via Spring without modifying existing code. An extension can add a `loyaltyTransactionCustomPopulator` to the converter's populator list using Spring `<list merge="true">`, extending the DTO conversion without touching the original code. This is critical for upgradeability.

### Strategy Pattern

Many business decisions in SAP Commerce are implemented as strategies — pluggable beans that can be swapped via Spring configuration.

Key strategies in the platform:

| Strategy Interface | Purpose | Default Implementation |
|---|---|---|
| `CommercePlaceOrderStrategy` | Place order logic | `DefaultCommercePlaceOrderStrategy` |
| `CommerceAddToCartStrategy` | Add to cart validation & logic | `DefaultCommerceAddToCartStrategy` |
| `FindPriceStrategy` | Price resolution | `Europe1FindPriceStrategy` |
| `FindDiscountValuesStrategy` | Discount resolution | `Europe1FindDiscountValuesStrategy` |
| `DeliveryModeLookupStrategy` | Available delivery modes | `DefaultDeliveryModeLookupStrategy` |
| `CommerceStockLevelCalculationStrategy` | Stock level calculation | `DefaultCommerceStockLevelCalculationStrategy` |

To customize pricing logic, you'd implement `FindPriceStrategy` and register it in Spring:

```xml
<bean id="findPriceStrategy" class="com.mycompany.strategies.CustomFindPriceStrategy"/>
```

The platform resolves strategies by bean ID convention or via explicit injection.

### Interceptor Chain

Interceptors are lifecycle hooks on `ModelService` operations. When you call `modelService.save(model)`, the platform executes a chain of interceptors before and after persistence.

The interceptor types, in execution order:

1. **`InitDefaultsInterceptor`** — Called during `modelService.create()`. Sets default values.
2. **`PrepareInterceptor`** — Called before save. Transforms data (e.g., generate a code, compute a derived field).
3. **`ValidateInterceptor`** — Called before save, after prepare. Validates business rules. Throws `InterceptorException` to abort.
4. **`RemoveInterceptor`** — Called before `modelService.remove()`. Can prevent deletion or clean up related data.
5. **`LoadInterceptor`** — Called when a model is loaded from the database. Rarely used.

```java
public class LoyaltyTransactionPrepareInterceptor implements PrepareInterceptor<LoyaltyTransactionModel> {
    
    @Override
    public void onPrepare(LoyaltyTransactionModel model, InterceptorContext ctx) throws InterceptorException {
        if (model.getCode() == null) {
            model.setCode("LT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
        }
    }
}
```

Registration in Spring:

```xml
<bean id="loyaltyTransactionPrepareInterceptor"
      class="com.mycompany.interceptors.LoyaltyTransactionPrepareInterceptor"/>

<bean id="loyaltyTransactionPrepareInterceptorMapping"
      class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
    <property name="interceptor" ref="loyaltyTransactionPrepareInterceptor"/>
    <property name="typeCode" value="LoyaltyTransaction"/>
</bean>
```

**Important internal detail**: The `InterceptorRegistry` class maintains mappings from type codes to interceptor lists. The `DefaultModelService` iterates through `InterceptorExecutionPolicy` to execute the chain. You can disable interceptors for bulk operations:

```java
Map<String, Object> context = new HashMap<>();
context.put(InterceptorExecutionPolicy.DISABLED_INTERCEPTOR_BEANS, 
    Set.of("loyaltyTransactionPrepareInterceptor"));
modelService.save(model, context);
```

### Event System

SAP Commerce has an internal event system based on `AbstractEvent` and `AbstractEventListener`:

```java
// Custom event
public class LoyaltyPointsEarnedEvent extends AbstractEvent {
    private final String customerUid;
    private final int points;
    
    public LoyaltyPointsEarnedEvent(String customerUid, int points) {
        this.customerUid = customerUid;
        this.points = points;
    }
    // getters...
}

// Listener
public class LoyaltyPointsEarnedListener extends AbstractEventListener<LoyaltyPointsEarnedEvent> {
    @Override
    protected void onEvent(LoyaltyPointsEarnedEvent event) {
        // Send notification, update analytics, etc.
    }
}

// Publishing
eventService.publishEvent(new LoyaltyPointsEarnedEvent("john.doe", 100));
```

Events can be **cluster-aware** (broadcast to all nodes) by extending `ClusterAwareEvent` and implementing `publish(int sourceNodeId)`.

---

## Common Architectural Mistakes

### 1. Putting Business Logic in Controllers

**Wrong:**
```java
@GetMapping("/products/{code}")
public ProductData getProduct(@PathVariable String code) {
    ProductModel product = productService.getProductForCode(code);
    if (product.getApprovalStatus() != ArticleApprovalStatus.APPROVED) {
        throw new UnknownIdentifierException("Product not found");
    }
    // MORE logic here...
}
```

**Right:** All business logic belongs in Services or Facades. Controllers should only handle HTTP concerns.

### 2. Using FlexibleSearch Directly in Services

Services should call DAOs, not execute queries directly. This makes services testable with mock DAOs.

### 3. Modifying Generated Files

Never edit files in `gensrc/`. They're regenerated on every build. Extend the generated class instead.

### 4. Using Jalo Layer

The Jalo layer (`jalo/` package) is the legacy API. Everything should use the **ServiceLayer** (services, models, `ModelService`). The Jalo layer exists for backward compatibility but is effectively deprecated.

```java
// WRONG - Jalo layer
JaloSession.getCurrentSession().getSessionContext();
ProductManager.getInstance().getProductsByCode(code);

// RIGHT - ServiceLayer
productService.getProductForCode(code);
```

### 5. Ignoring Catalog Version Context

Forgetting to set the catalog version context leads to "item not found" errors. Products exist in specific catalog versions (Staged/Online):

```java
CatalogVersionModel catalogVersion = catalogVersionService.getCatalogVersion("myProductCatalog", "Online");
catalogVersionService.setSessionCatalogVersion("myProductCatalog", "Online");
// Now FlexibleSearch queries will filter by this catalog version
```

### 6. Over-Customizing Instead of Extending

Copying an entire service implementation to change one method is a maintenance nightmare. Instead:
- Override just the method you need
- Use strategies for pluggable behavior
- Add populators instead of rewriting converters
- Use interceptors instead of modifying service code

### 7. Not Understanding the Build System

SAP Commerce uses Apache Ant, not Maven or Gradle. Key commands:

```bash
ant clean all          # Full rebuild
ant updatesystem       # Update database schema without data loss
ant initialize         # Full database initialization (DESTROYS DATA)
ant unittests          # Run unit tests
ant integrationtests   # Run integration tests (starts platform)
```

The `ant all` command: compiles all extensions in dependency order, generates model classes from `items.xml`, generates Spring bean definitions, and packages web applications.

---

## When to Customize vs. Extend vs. Use OOTB

| Approach | When to Use | Risk Level |
|----------|------------|------------|
| **OOTB (Out of the Box)** | Feature matches requirements 80%+ | Low |
| **Configuration** | Behavior can be adjusted via properties/ImpEx | Low |
| **Extend (Add Populator/Strategy/Interceptor)** | Need additional behavior without changing existing code | Medium |
| **Override (Replace Bean)** | Need to change existing behavior | Medium-High |
| **Custom Extension** | New domain/feature not covered by OOTB | High |
| **Modify OOTB Code** | **NEVER** — breaks upgrades | Critical |

**Golden Rule**: Never modify files in `bin/platform/` or `bin/modules/`. Always customize through your own extensions using Spring bean overriding, type system extension, and the patterns described above.

---

## Summary

SAP Commerce Cloud's architecture is built around a few core concepts:

1. **Extensions** provide modularity — everything is an extension, and custom code lives in custom extensions
2. **The Type System** replaces traditional ORM — `items.xml` defines your data model, the platform generates the rest
3. **The Service Layer** enforces a clean architecture: Controllers → Facades → Services → DAOs → Models
4. **Configuration cascades** from platform defaults through to environment-specific overrides
5. **Design patterns** (Populator/Converter, Strategy, Interceptor) enable extensibility without modifying core code

Understanding these architectural foundations will make you more effective at building, debugging, and maintaining SAP Commerce applications. When you understand *why* the platform is structured this way, you'll make better decisions about *how* to implement your requirements.