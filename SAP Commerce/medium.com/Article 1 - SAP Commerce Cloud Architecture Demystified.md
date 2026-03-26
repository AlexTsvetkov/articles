# How I Finally Understood SAP Commerce Cloud Architecture (And You Can Too)

## A deep-dive into the internals of one of enterprise e-commerce's most complex platforms — from type systems to interceptor chains

---

I remember my first day on an SAP Commerce project. I opened the codebase and was immediately lost. Extensions everywhere. XML files defining data models instead of JPA annotations. Something called "FlexibleSearch" that looked like SQL but wasn't. A "Jalo layer" that everyone said not to touch but was still everywhere.

It took months to piece together how everything fit. **This is the article I wish I had on day one.**

If you're a Java developer entering the SAP Commerce world — or a mid-level developer who wants to truly understand the platform internals — let's walk through the architecture together.

---

## What Is SAP Commerce Cloud, Really?

SAP Commerce Cloud (formerly Hybris) is a **Java-based enterprise e-commerce platform** within the SAP CX ecosystem. At its core, it's:

- A **Spring-based monolith** running on embedded Tomcat
- A **dynamic type system** that generates Java models and database DDL from XML
- An **extension framework** for modular customization
- A **headless commerce engine** with OCC REST APIs

It's not a lightweight microservice. It's a "modulith" — a monolithic application with a modular internal architecture — handling everything from product modeling and pricing to order management and CMS.

---

## The Architecture at 30,000 Feet

The platform is organized into four layers, each with a strict dependency direction (top calls down, never up):

**1. Client Layer** — Composable Storefront (Spartacus), mobile apps, or any custom SPA communicating over REST/JSON.

**2. OCC API Layer** — Spring `@Controller` classes exposing RESTful endpoints under `/occ/v2/`. Uses WsDTO objects as the API contract, Orika Mappers for translation, and OAuth2 for authentication.

**3. Application Layer** — Where business logic lives. Follows the chain: **Facades** (orchestrate services, convert Models to DTOs) → **Services** (core business logic on Models) → **DAOs** (FlexibleSearch queries) → **Model objects** (generated from `items.xml`). Strategies and hooks provide pluggable business rules.

**4. Persistence Layer** — RDBMS (SAP HANA in CCv2, MySQL/HSQLDB for local dev) accessed via JDBC. Solr handles product search with faceting. Media storage supports local filesystem, S3, or Azure blob.

Each layer has a specific responsibility. Let me break down what actually matters.

---

## The Extension System: Everything Is a Module

The most important concept to grasp: **everything in SAP Commerce is an extension**. The platform core, commerce features, your custom code — all extensions.

Every extension follows a standard layout:

```
myextension/
├── extensioninfo.xml          # Dependencies, metadata
├── resources/
│   ├── myextension-items.xml  # Data model (Type System)
│   └── myextension-spring.xml # Spring beans
├── src/                       # Java source
├── web/                       # Web layer
└── gensrc/                    # Generated models (DON'T EDIT)
```

The `extensioninfo.xml` declares dependencies via `<requires-extension>`. At startup, the platform resolves the full dependency graph and loads extensions in order.

**Extensions come in four flavors:**

1. **Platform extensions** (`bin/platform/ext/`) — core infrastructure
2. **Commerce extensions** (`bin/modules/`) — business features
3. **Custom extensions** (`bin/custom/`) — your project code
4. **AddOns** — overlays on other extensions (legacy pattern)

> **Key insight**: In CCv2 (Cloud), `localextensions.xml` is auto-generated from the `extensions` list in your `manifest.json`. You control your extension set from one place.

---

## The Type System: Where SAP Commerce Gets Unique

This is where SAP Commerce diverges fundamentally from standard Java development. **You don't write `@Entity` classes.** You define your data model in `items.xml`, and the platform generates everything.

Here's what a real-world type definition looks like:

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
            <persistence type="dynamic" 
                         attributeHandler="loyaltyScoreHandler"/>
        </attribute>
    </attributes>
</itemtype>
```

Key things happening here:

- **`deployment`** maps the type to a database table with a unique typecode
- **`persistence type="property"`** = stored in database
- **`persistence type="dynamic"`** = computed at runtime by a Spring bean (no DB column)
- You can **extend existing types** by setting `autocreate="false" generate="false"` — the platform merges attributes across extensions

After `ant build`, the platform generates `GeneratedLoyaltyTransactionModel.java` with getters/setters. You extend it in your own `LoyaltyTransactionModel.java`.

**The comparison with JPA is stark:**

- In **JPA/Hibernate** you use `@Entity` annotations → in **SAP Commerce** you use `items.xml` XML
- In **JPA** you run Flyway migrations → in **SAP Commerce** you run `ant updatesystem`
- In **JPA** you write entity classes yourself → in **SAP Commerce** the platform generates models from XML
- In **JPA** you query with JPQL → in **SAP Commerce** you query with FlexibleSearch

---

## The Service Layer: The Architecture You Must Follow

SAP Commerce enforces a layered architecture. Deviating from it causes pain during upgrades and testing.

**The stack: Controller → Facade → Service → DAO → ModelService**

Each layer has a clear contract:

- **Controllers** — HTTP concerns only. Delegate to Facades.
- **Facades** — orchestrate Services, convert Models to DTOs using **Populators/Converters**
- **Services** — core business logic, operate on Models
- **DAOs** — FlexibleSearch queries, return Models
- **ModelService** — the persistence gateway (`save()`, `create()`, `remove()`)

### Why does this matter?

**Facades are the transaction boundary.** Services are reusable across channels. DAOs isolate data access. This separation makes it possible to test each layer independently and extend behavior without modifying existing code.

---

## Design Patterns That Power Everything

Three patterns are everywhere in SAP Commerce. Understanding them is non-negotiable.

### 1. Populator/Converter Pattern

This is how Models become DTOs. A `Converter` delegates to a list of `Populator` beans:

```xml
<bean id="productConverter" parent="abstractPopulatingConverter">
    <property name="populators">
        <list>
            <ref bean="productBasicPopulator"/>
            <ref bean="productPricePopulator"/>
        </list>
    </property>
</bean>
```

**The beauty**: Any extension can add a populator to this list using `<list merge="true">`, extending the conversion without touching original code.

### 2. Strategy Pattern

Business decisions are implemented as pluggable strategies. `FindPriceStrategy` for pricing, `CommerceAddToCartStrategy` for cart logic, `CommercePlaceOrderStrategy` for order placement. Swap the Spring bean, change the behavior.

### 3. Interceptor Chain

Lifecycle hooks on `ModelService` operations:

1. **InitDefaultsInterceptor** — during `create()`
2. **PrepareInterceptor** — before save (transform data)
3. **ValidateInterceptor** — before save (enforce rules)
4. **RemoveInterceptor** — before delete

```java
public class LoyaltyTransactionPrepareInterceptor 
    implements PrepareInterceptor<LoyaltyTransactionModel> {
    
    @Override
    public void onPrepare(LoyaltyTransactionModel model, 
                           InterceptorContext ctx) {
        if (model.getCode() == null) {
            model.setCode("LT-" + UUID.randomUUID()
                .toString().substring(0, 8).toUpperCase());
        }
    }
}
```

You can even **disable interceptors** for bulk operations — critical for performance during data migrations.

---

## The Mistakes Everyone Makes

After working on multiple SAP Commerce projects, I've seen the same mistakes repeated:

1. **Business logic in controllers** — it belongs in Services/Facades
2. **FlexibleSearch in Services** — use DAOs for data access
3. **Editing generated files** in `gensrc/` — they're regenerated every build
4. **Using the Jalo layer** — it's deprecated; use the ServiceLayer
5. **Ignoring catalog version context** — products live in Staged/Online versions; forget to set the context and items "disappear"
6. **Over-customizing** — copy-pasting an entire service to change one method instead of overriding, using strategies, or adding interceptors

---

## When to Customize vs. Extend vs. Use OOTB

Here's how I think about the customization spectrum:

- **Out of the Box** — Use when it matches 80%+ of requirements. Risk: Low.
- **Configuration** (properties/ImpEx) — Use when behavior is adjustable without code. Risk: Low.
- **Extend** (add populator/strategy/interceptor) — Use when you need additive behavior. Risk: Medium.
- **Override** (replace Spring bean) — Use when you need to change existing behavior. Risk: Medium-High.
- **Custom Extension** — Use for new domains not covered by OOTB. Risk: High.
- **Modify OOTB Code** — **NEVER do this.** Risk: Critical. It breaks upgrades.

**Golden rule**: Never modify `bin/platform/` or `bin/modules/`. Always work through your own extensions.

---

## What I Wish Someone Had Told Me

SAP Commerce is a powerful platform, but its learning curve is steep because the architecture is genuinely different from standard Spring Boot applications. The type system replaces ORM. Extensions replace modules. Interceptors replace event listeners. Populators replace manual DTO mapping.

Once these patterns click, everything starts making sense — and you'll be able to build, debug, and maintain Commerce applications with confidence.

**The key is understanding *why* the platform is structured this way.** It's built for enterprise extensibility, safe upgrades, and multi-channel commerce at scale. Every architectural choice serves that goal.

---

*What's the most surprising thing you discovered when you first worked with SAP Commerce? I'd love to hear your experiences in the comments.*