---
title: "SAP Commerce Cloud Architecture Demystified: A Practical Guide for Developers"
published: true
description: "Deep dive into SAP Commerce internals — type system, extensions, service layer, and design patterns"
tags: java, architecture, cloud, tutorial
cover_image: https://placeholder-url.com/sap-commerce-architecture.png
---

## TL;DR

SAP Commerce Cloud (formerly Hybris) is a Java/Spring-based enterprise e-commerce platform with a unique architecture:
- **Extension system** — everything is a module (extension), from platform core to your custom code
- **Type system** — define data models in `items.xml`, platform generates Java classes and database DDL (no JPA/Hibernate)
- **Service layer** — strict Controller → Facade → Service → DAO → ModelService layering
- **Design patterns** — Populator/Converter, Strategy, and Interceptor patterns enable extensibility without modifying core code
- **Golden rule** — never modify OOTB code; always extend through your own extensions

---

## What Is SAP Commerce Cloud?

SAP Commerce Cloud is a Java-based enterprise e-commerce platform. It's a Spring application running on embedded Tomcat with:

- A **dynamic type system** generating Java models and DB schemas from XML
- An **extension framework** for modular customization
- **OCC REST APIs** consumed by Composable Storefront (Spartacus) or any frontend
- Support for complex B2C and B2B scenarios

It's a monolith ("modulith") — not microservices — designed for deep integration between product modeling, pricing, promotions, orders, and CMS.

## High-Level Architecture

```
CLIENT: Spartacus (Angular) / Mobile / Custom SPA
    │ REST/JSON
OCC API: Controllers → WsDTOs → OAuth2
    │
APPLICATION: Facades → Services → DAOs → Models
    │ JDBC
PERSISTENCE: SAP HANA / MySQL + Solr + Media Storage
```

## The Extension System

**Everything in SAP Commerce is an extension.** Platform core, commerce features, your code — all extensions.

### Extension Structure

```
myextension/
├── extensioninfo.xml          # Dependencies, metadata
├── resources/
│   ├── myextension-items.xml  # Type definitions
│   └── myextension-spring.xml # Spring beans
├── src/                       # Java source
├── gensrc/                    # Generated models (DON'T EDIT)
└── web/                       # Web layer
```

`extensioninfo.xml` declares dependencies:

```xml
<extensioninfo>
    <extension name="myextension">
        <requires-extension name="commerceservices"/>
        <requires-extension name="catalog"/>
    </extension>
</extensioninfo>
```

### Extension Categories

1. **Platform** (`bin/platform/ext/`) — core: `core`, `processing`, `Europe1` (pricing)
2. **Commerce** (`bin/modules/`) — `commerce-services`, `search-and-navigation`, `cms`, `b2b-commerce`
3. **Custom** (`bin/custom/`) — your project code, created via `ant extgen`
4. **AddOns** — legacy overlay mechanism, being phased out

### Loading Order

At startup:
1. Reads `localextensions.xml` (in CCv2, generated from `manifest.json`)
2. Resolves dependency graph from all `extensioninfo.xml` files
3. Loads extensions in dependency order
4. For each: loads Spring context, registers types from `items.xml`, initializes extension manager

## The Type System

This is where SAP Commerce differs fundamentally from standard Java apps. **No `@Entity` classes, no Flyway.** You define models in `items.xml`.

### Example

```xml
<itemtype code="LoyaltyTransaction" autocreate="true" generate="true"
          extends="GenericItem">
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
        <attribute qualifier="loyaltyScore" type="java.lang.Double">
            <persistence type="dynamic" attributeHandler="loyaltyScoreHandler"/>
        </attribute>
    </attributes>
</itemtype>
```

Key concepts:
- **`deployment`** — maps type to a DB table with unique typecode
- **`persistence type="property"`** — standard DB column
- **`persistence type="dynamic"`** — computed at runtime by a Spring bean (no column)
- **Extending existing types** — set `autocreate="false" generate="false"` to add attributes to platform types

### How It Maps to the Database

`ant initialize` or `ant updatesystem` triggers `SchemaGenerator`:
- Each type with `deployment` → dedicated table
- Types without deployment → parent table (single-table inheritance)
- Localized attributes → separate `*lp` table
- Many-to-many relations → link tables

### Type System vs JPA

| JPA/Hibernate | SAP Commerce |
|---|---|
| `@Entity` annotations | `items.xml` XML |
| Flyway/Liquibase | `ant updatesystem` |
| You write entities | Platform generates models |
| `@OneToMany` | `<relation>` elements |
| JPQL/HQL | FlexibleSearch |
| `@Inheritance` | Type hierarchy via `extends=` |

## The Service Layer

Strict layered architecture: **Controller → Facade → Service → DAO → ModelService**

```java
// DAO
public class DefaultLoyaltyTransactionDao implements LoyaltyTransactionDao {
    @Resource
    private FlexibleSearchService flexibleSearchService;
    
    private static final String FIND_BY_CUSTOMER = 
        "SELECT {pk} FROM {LoyaltyTransaction} WHERE {customer} = ?customer";
    
    @Override
    public List<LoyaltyTransactionModel> findByCustomer(UserModel customer) {
        FlexibleSearchQuery query = new FlexibleSearchQuery(FIND_BY_CUSTOMER);
        query.addQueryParameter("customer", customer);
        return flexibleSearchService.search(query).getResult();
    }
}

// Service
public class DefaultLoyaltyService implements LoyaltyService {
    @Resource private LoyaltyTransactionDao loyaltyTransactionDao;
    @Resource private ModelService modelService;
    
    @Override
    public LoyaltyTransactionModel createTransaction(UserModel customer, int points) {
        LoyaltyTransactionModel tx = modelService.create(LoyaltyTransactionModel.class);
        tx.setCustomer(customer);
        tx.setPoints(points);
        tx.setTransactionDate(new Date());
        modelService.save(tx);
        return tx;
    }
}

// Facade
public class DefaultLoyaltyFacade implements LoyaltyFacade {
    @Resource private LoyaltyService loyaltyService;
    @Resource private Converter<LoyaltyTransactionModel, LoyaltyTransactionData> converter;
    
    @Override
    public List<LoyaltyTransactionData> getTransactions(String userId) {
        UserModel user = userService.getUserForUID(userId);
        return converter.convertAll(loyaltyService.getTransactionsForCustomer(user));
    }
}
```

**Why this matters:**
- **Facades** = transaction boundary
- **Services** = reusable across channels
- **DAOs** = isolate data access (testable with mocks)

## Key Design Patterns

### Populator/Converter

Models → DTOs conversion via composable populators:

```xml
<bean id="loyaltyConverter" parent="abstractPopulatingConverter">
    <property name="populators">
        <list>
            <ref bean="loyaltyBasicPopulator"/>
            <ref bean="loyaltyPointsPopulator"/>
        </list>
    </property>
</bean>
```

Any extension can add populators via `<list merge="true">` — zero modification of existing code.

### Strategy Pattern

Pluggable business rules: `FindPriceStrategy`, `CommerceAddToCartStrategy`, `CommercePlaceOrderStrategy`. Override by registering your Spring bean with the same ID.

### Interceptor Chain

Lifecycle hooks on `ModelService` operations:

1. **InitDefaultsInterceptor** — `create()`
2. **PrepareInterceptor** — before save (transform)
3. **ValidateInterceptor** — before save (validate)
4. **RemoveInterceptor** — before delete

```java
public class LoyaltyPrepareInterceptor 
    implements PrepareInterceptor<LoyaltyTransactionModel> {
    @Override
    public void onPrepare(LoyaltyTransactionModel model, InterceptorContext ctx) {
        if (model.getCode() == null) {
            model.setCode("LT-" + UUID.randomUUID().toString().substring(0, 8));
        }
    }
}
```

## Configuration Hierarchy

```
Priority (highest wins):
1. System properties (-Dproperty=value)
2. Environment variables (CCv2 y_ prefix)
3. local.properties
4. Extension project.properties
5. Platform project.properties
```

In CCv2, `manifest.json` properties inject into `local.properties` or system properties per persona (development, staging, production).

## Common Mistakes

1. **Business logic in controllers** → use Services/Facades
2. **FlexibleSearch in Services** → use DAOs
3. **Editing `gensrc/` files** → they're regenerated every build
4. **Using Jalo layer** → use ServiceLayer
5. **Ignoring catalog version** → products live in Staged/Online versions
6. **Copy-pasting services** → override methods, use strategies, add interceptors
7. **Modifying OOTB code** → **never** touch `bin/platform/` or `bin/modules/`

## Customization Decision Matrix

| Approach | When | Risk |
|---|---|---|
| OOTB | Matches 80%+ requirements | Low |
| Configuration | Properties/ImpEx adjustable | Low |
| Extend | Add populator/strategy/interceptor | Medium |
| Override bean | Change existing behavior | Medium-High |
| Custom Extension | New domain | High |
| Modify OOTB | **NEVER** | Critical |

---

What's your experience with SAP Commerce architecture? Any patterns or pitfalls I missed? Let me know in the comments 👇