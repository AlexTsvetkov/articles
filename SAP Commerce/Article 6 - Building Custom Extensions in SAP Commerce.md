# Building Custom Extensions in SAP Commerce: Architecture, Patterns, and Best Practices

Every SAP Commerce project requires custom extensions. Whether you're adding a loyalty program, integrating with an ERP system, or building a custom checkout flow, extensions are how you add functionality to the platform without modifying the core codebase.

But building extensions well — with clean architecture, proper layering, testability, and maintainability — is what separates a project that survives five years of production from one that becomes unmaintainable within months. This guide covers the architecture, patterns, and practical techniques for building extensions the right way.

---

## What Is an Extension?

An extension in SAP Commerce is a modular unit of functionality. It's a directory with a defined structure that the platform discovers, compiles, and loads at startup. Extensions can:

- Define new data types (via `items.xml`)
- Add business logic (Java services, DAOs, facades)
- Expose web endpoints (REST controllers, web pages)
- Include ImpEx data files
- Provide Spring bean configurations
- Package Backoffice customizations

Every extension declares its identity and dependencies in `extensioninfo.xml`.

### Extension Types

| Type | Purpose | Example |
|------|---------|---------|
| **Core** | Data model + services + DAOs | `myprojectcore` |
| **Facades** | DTOs + converters + populators | `myprojectfacades` |
| **Web/OCC** | REST API endpoints | `myprojectocc` |
| **Storefront** | Accelerator storefront pages | `myprojectstorefront` |
| **Backoffice** | Backoffice UI customizations | `myprojectbackoffice` |
| **Initial Data** | Sample/project data loading | `myprojectinitialdata` |
| **Test** | Integration test suites | `myprojecttest` |

---

## Creating a New Extension

### Using the `extgen` Template

SAP Commerce provides the `extgen` tool to scaffold extensions:

```bash
cd hybris/bin/platform
ant extgen

# Follow the prompts:
# Template: yempty (for a clean extension) or yaddon (for addons)
# Extension name: myprojectcore
# Package name: com.mycompany.myproject.core
```

### Extension Directory Structure

After generation, your extension looks like this:

```
myprojectcore/
├── extensioninfo.xml          # Extension metadata and dependencies
├── buildcallbacks.xml         # Custom Ant build hooks
├── project.properties         # Default properties for this extension
├── resources/
│   ├── myprojectcore-items.xml        # Type system definitions
│   ├── myprojectcore-spring.xml       # Spring bean definitions
│   ├── localization/
│   │   ├── myprojectcore-locales_en.properties
│   │   └── myprojectcore-locales_de.properties
│   └── impex/
│       ├── essentialdata-myprojectcore.impex
│       └── projectdata-myprojectcore.impex
├── src/
│   └── com/mycompany/myproject/core/
│       ├── setup/
│       │   └── MyProjectCoreSystemSetup.java
│       ├── dao/
│       ├── service/
│       │   └── impl/
│       └── interceptors/
├── testsrc/
│   └── com/mycompany/myproject/core/
│       ├── dao/
│       └── service/
│           └── impl/
├── web/
│   ├── src/
│   ├── webroot/
│   └── web-spring.xml
└── gensrc/                    # Auto-generated model classes (don't edit)
```

### extensioninfo.xml

This file declares the extension's identity and dependencies:

```xml
<extensioninfo xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="extensioninfo.xsd">
    <extension
        abstractionlayer="custom"
        classprefix="myprojectcore"
        name="myprojectcore"
        requires-extension="commerceservices">
        
        <requires-extension name="commerceservices"/>
        <requires-extension name="payment"/>
        <requires-extension name="promotions"/>
        
        <meta key="backoffice-module" value="true"/>
    </extension>
</extensioninfo>
```

**`requires-extension`**: Declares compile-time and runtime dependencies. The platform resolves the dependency graph and loads extensions in the correct order.

**Rule**: Only declare direct dependencies. If `myprojectcore` uses classes from `commerceservices`, declare it. Don't declare transitive dependencies that `commerceservices` already pulls in.

---

## The items.xml — Defining Your Data Model

The `items.xml` is where you define your custom types (tables), attributes (columns), relations, and enums. The platform generates Java model classes from this file.

### Adding a New Type

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
        <itemtype code="LoyaltyAccount"
                  jaloclass="com.mycompany.core.jalo.LoyaltyAccount"
                  extends="GenericItem"
                  autocreate="true"
                  generate="true">
            <deployment table="LoyaltyAccounts" typecode="25000"/>
            <attributes>
                <attribute qualifier="accountId" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers read="true" write="true" optional="false" unique="true"/>
                </attribute>
                <attribute qualifier="customer" type="Customer">
                    <persistence type="property"/>
                    <modifiers read="true" write="true" optional="false"/>
                </attribute>
                <attribute qualifier="points" type="java.lang.Integer">
                    <persistence type="property"/>
                    <modifiers read="true" write="true"/>
                    <defaultvalue>Integer.valueOf(0)</defaultvalue>
                </attribute>
                <attribute qualifier="tier" type="LoyaltyTier">
                    <persistence type="property"/>
                    <modifiers read="true" write="true"/>
                    <defaultvalue>em().getEnumerationValue("LoyaltyTier", "BRONZE")</defaultvalue>
                </attribute>
                <attribute qualifier="joinDate" type="java.util.Date">
                    <persistence type="property"/>
                    <modifiers read="true" write="false"/>
                </attribute>
            </attributes>
            <indexes>
                <index name="accountIdIdx" unique="true">
                    <key attribute="accountId"/>
                </index>
                <index name="customerIdx">
                    <key attribute="customer"/>
                </index>
            </indexes>
        </itemtype>

        <itemtype code="LoyaltyTransaction"
                  jaloclass="com.mycompany.core.jalo.LoyaltyTransaction"
                  extends="GenericItem"
                  autocreate="true"
                  generate="true">
            <deployment table="LoyaltyTransactions" typecode="25001"/>
            <attributes>
                <attribute qualifier="transactionId" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers read="true" write="false" optional="false" unique="true"/>
                </attribute>
                <attribute qualifier="account" type="LoyaltyAccount">
                    <persistence type="property"/>
                    <modifiers read="true" write="false" optional="false"/>
                </attribute>
                <attribute qualifier="points" type="java.lang.Integer">
                    <persistence type="property"/>
                    <modifiers read="true" write="false" optional="false"/>
                </attribute>
                <attribute qualifier="order" type="Order">
                    <persistence type="property"/>
                    <modifiers read="true" write="false"/>
                </attribute>
                <attribute qualifier="description" type="java.lang.String">
                    <persistence type="property">
                        <columntype>
                            <value>HYBRIS.LONG_STRING</value>
                        </columntype>
                    </persistence>
                </attribute>
            </attributes>
        </itemtype>
    </itemtypes>
</items>
```

### Extending Existing Types

Add attributes to existing platform types without modifying their source:

```xml
<itemtype code="Customer" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="loyaltyAccount" type="LoyaltyAccount">
            <persistence type="property"/>
            <modifiers read="true" write="true"/>
        </attribute>
        <attribute qualifier="preferredContactMethod" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
    </attributes>
</itemtype>

<itemtype code="Order" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="loyaltyPointsEarned" type="java.lang.Integer">
            <persistence type="property"/>
            <defaultvalue>Integer.valueOf(0)</defaultvalue>
        </attribute>
        <attribute qualifier="loyaltyPointsRedeemed" type="java.lang.Integer">
            <persistence type="property"/>
            <defaultvalue>Integer.valueOf(0)</defaultvalue>
        </attribute>
    </attributes>
</itemtype>
```

**`autocreate="false"`** means "don't create a new type — modify an existing one." **`generate="false"`** means "don't regenerate the model class — the platform's model class already exists."

### Defining Relations

For one-to-many relationships:

```xml
<relations>
    <relation code="LoyaltyAccount2TransactionRelation"
              localized="false"
              generate="true"
              autocreate="true">
        <sourceElement type="LoyaltyAccount"
                       qualifier="account"
                       cardinality="one">
            <modifiers read="true" write="true" optional="false"/>
        </sourceElement>
        <targetElement type="LoyaltyTransaction"
                       qualifier="transactions"
                       cardinality="many"
                       collectiontype="list"
                       ordered="true">
            <modifiers read="true" write="true"/>
        </targetElement>
    </relation>
</relations>
```

### Type Code Allocation

Every custom type needs a unique `typecode` in the `<deployment>` tag. The range **25000–32767** is reserved for customer extensions. Track your type codes in a central document to avoid collisions across extensions:

| Type Code | Type | Extension |
|-----------|------|-----------|
| 25000 | LoyaltyAccount | myprojectcore |
| 25001 | LoyaltyTransaction | myprojectcore |
| 25002 | WishlistItem | myprojectcore |
| 25010 | CustomPaymentInfo | myprojectcore |

---

## Layered Architecture

SAP Commerce follows a strict layered architecture. Understanding and respecting these layers is essential for maintainability.

```
┌─────────────────────────────────────────┐
│  Web / OCC Layer (Controllers)           │  ← HTTP request handling
├─────────────────────────────────────────┤
│  Facade Layer (Facades + Converters)     │  ← Orchestration, DTO conversion
├─────────────────────────────────────────┤
│  Service Layer (Business Logic)          │  ← Core business rules
├─────────────────────────────────────────┤
│  DAO Layer (Data Access)                 │  ← FlexibleSearch queries
├─────────────────────────────────────────┤
│  Model Layer (Generated Models)          │  ← Auto-generated from items.xml
├─────────────────────────────────────────┤
│  Persistence Layer (Platform)            │  ← Database operations
└─────────────────────────────────────────┘
```

### Layer Rules

1. **Controllers** call only **Facades**
2. **Facades** call **Services** and use **Converters/Populators**
3. **Services** call **DAOs** and other **Services**
4. **DAOs** use **FlexibleSearch** to query data
5. **Never skip layers** — a controller should never call a DAO directly

### DAO Layer

```java
public interface LoyaltyAccountDao {
    LoyaltyAccountModel findByAccountId(String accountId);
    LoyaltyAccountModel findByCustomer(CustomerModel customer);
    List<LoyaltyAccountModel> findByTier(LoyaltyTier tier);
}
```

```java
public class DefaultLoyaltyAccountDao implements LoyaltyAccountDao {
    
    private static final String FIND_BY_ACCOUNT_ID = 
        "SELECT {pk} FROM {LoyaltyAccount} WHERE {accountId} = ?accountId";
    
    private static final String FIND_BY_CUSTOMER = 
        "SELECT {pk} FROM {LoyaltyAccount} WHERE {customer} = ?customer";
    
    private static final String FIND_BY_TIER = 
        "SELECT {pk} FROM {LoyaltyAccount} WHERE {tier} = ?tier ORDER BY {points} DESC";
    
    @Resource
    private FlexibleSearchService flexibleSearchService;
    
    @Override
    public LoyaltyAccountModel findByAccountId(String accountId) {
        FlexibleSearchQuery query = new FlexibleSearchQuery(FIND_BY_ACCOUNT_ID);
        query.addQueryParameter("accountId", accountId);
        SearchResult<LoyaltyAccountModel> result = flexibleSearchService.search(query);
        return result.getResult().stream().findFirst().orElse(null);
    }
    
    @Override
    public LoyaltyAccountModel findByCustomer(CustomerModel customer) {
        FlexibleSearchQuery query = new FlexibleSearchQuery(FIND_BY_CUSTOMER);
        query.addQueryParameter("customer", customer);
        SearchResult<LoyaltyAccountModel> result = flexibleSearchService.search(query);
        return result.getResult().stream().findFirst().orElse(null);
    }
    
    @Override
    public List<LoyaltyAccountModel> findByTier(LoyaltyTier tier) {
        FlexibleSearchQuery query = new FlexibleSearchQuery(FIND_BY_TIER);
        query.addQueryParameter("tier", tier);
        return flexibleSearchService.<LoyaltyAccountModel>search(query).getResult();
    }
}
```

### Service Layer

```java
public interface LoyaltyService {
    LoyaltyAccountModel getOrCreateAccount(CustomerModel customer);
    void earnPoints(LoyaltyAccountModel account, int points, OrderModel order);
    boolean redeemPoints(LoyaltyAccountModel account, int points, OrderModel order);
    LoyaltyTier calculateTier(int totalPoints);
}
```

```java
public class DefaultLoyaltyService implements LoyaltyService {
    
    private static final Logger LOG = LoggerFactory.getLogger(DefaultLoyaltyService.class);
    
    private static final int SILVER_THRESHOLD = 1000;
    private static final int GOLD_THRESHOLD = 5000;
    private static final int PLATINUM_THRESHOLD = 20000;
    
    @Resource
    private LoyaltyAccountDao loyaltyAccountDao;
    @Resource
    private ModelService modelService;
    @Resource
    private KeyGenerator loyaltyAccountIdGenerator;
    
    @Override
    public LoyaltyAccountModel getOrCreateAccount(CustomerModel customer) {
        LoyaltyAccountModel existing = loyaltyAccountDao.findByCustomer(customer);
        if (existing != null) {
            return existing;
        }
        
        LoyaltyAccountModel account = modelService.create(LoyaltyAccountModel.class);
        account.setAccountId(loyaltyAccountIdGenerator.generate().toString());
        account.setCustomer(customer);
        account.setPoints(0);
        account.setTier(LoyaltyTier.BRONZE);
        account.setJoinDate(new Date());
        modelService.save(account);
        
        LOG.info("Created loyalty account {} for customer {}", 
            account.getAccountId(), customer.getUid());
        return account;
    }
    
    @Override
    public void earnPoints(LoyaltyAccountModel account, int points, OrderModel order) {
        Preconditions.checkArgument(points > 0, "Points must be positive");
        
        LoyaltyTransactionModel transaction = modelService.create(LoyaltyTransactionModel.class);
        transaction.setTransactionId(UUID.randomUUID().toString());
        transaction.setAccount(account);
        transaction.setPoints(points);
        transaction.setOrder(order);
        transaction.setDescription("Points earned from order " + order.getCode());
        
        account.setPoints(account.getPoints() + points);
        account.setTier(calculateTier(account.getPoints()));
        
        modelService.saveAll(transaction, account);
        LOG.info("Account {} earned {} points. Total: {}", 
            account.getAccountId(), points, account.getPoints());
    }
    
    @Override
    public boolean redeemPoints(LoyaltyAccountModel account, int points, OrderModel order) {
        if (account.getPoints() < points) {
            LOG.warn("Insufficient points for account {}. Has: {}, Requested: {}", 
                account.getAccountId(), account.getPoints(), points);
            return false;
        }
        
        LoyaltyTransactionModel transaction = modelService.create(LoyaltyTransactionModel.class);
        transaction.setTransactionId(UUID.randomUUID().toString());
        transaction.setAccount(account);
        transaction.setPoints(-points);
        transaction.setOrder(order);
        transaction.setDescription("Points redeemed for order " + order.getCode());
        
        account.setPoints(account.getPoints() - points);
        account.setTier(calculateTier(account.getPoints()));
        
        modelService.saveAll(transaction, account);
        return true;
    }
    
    @Override
    public LoyaltyTier calculateTier(int totalPoints) {
        if (totalPoints >= PLATINUM_THRESHOLD) return LoyaltyTier.PLATINUM;
        if (totalPoints >= GOLD_THRESHOLD) return LoyaltyTier.GOLD;
        if (totalPoints >= SILVER_THRESHOLD) return LoyaltyTier.SILVER;
        return LoyaltyTier.BRONZE;
    }
}
```

### Facade Layer

Facades orchestrate service calls and convert Models to DTOs:

```java
public interface LoyaltyFacade {
    LoyaltyAccountData getCurrentCustomerLoyaltyAccount();
    List<LoyaltyTransactionData> getTransactionHistory(int page, int pageSize);
    void earnPointsForOrder(String orderCode);
}
```

```java
public class DefaultLoyaltyFacade implements LoyaltyFacade {
    
    @Resource
    private LoyaltyService loyaltyService;
    @Resource
    private UserService userService;
    @Resource
    private OrderService orderService;
    @Resource
    private Converter<LoyaltyAccountModel, LoyaltyAccountData> loyaltyAccountConverter;
    @Resource
    private Converter<LoyaltyTransactionModel, LoyaltyTransactionData> loyaltyTransactionConverter;
    
    @Override
    public LoyaltyAccountData getCurrentCustomerLoyaltyAccount() {
        CustomerModel customer = (CustomerModel) userService.getCurrentUser();
        LoyaltyAccountModel account = loyaltyService.getOrCreateAccount(customer);
        return loyaltyAccountConverter.convert(account);
    }
    
    @Override
    public void earnPointsForOrder(String orderCode) {
        CustomerModel customer = (CustomerModel) userService.getCurrentUser();
        OrderModel order = orderService.getOrderForCode(orderCode);
        LoyaltyAccountModel account = loyaltyService.getOrCreateAccount(customer);
        
        int pointsToEarn = calculatePointsForOrder(order);
        loyaltyService.earnPoints(account, pointsToEarn, order);
    }
    
    private int calculatePointsForOrder(OrderModel order) {
        // 1 point per dollar spent
        return order.getTotalPrice().intValue();
    }
}
```

### Converter and Populator Pattern

```java
public class LoyaltyAccountPopulator 
    implements Populator<LoyaltyAccountModel, LoyaltyAccountData> {
    
    @Override
    public void populate(LoyaltyAccountModel source, LoyaltyAccountData target) {
        target.setAccountId(source.getAccountId());
        target.setPoints(source.getPoints());
        target.setTier(source.getTier().getCode());
        target.setJoinDate(source.getJoinDate());
        
        if (source.getCustomer() != null) {
            target.setCustomerName(source.getCustomer().getName());
            target.setCustomerEmail(source.getCustomer().getUid());
        }
    }
}
```

---

## Spring Configuration

### Bean Definitions

Define your beans in `myprojectcore-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- DAOs -->
    <bean id="loyaltyAccountDao" 
          class="com.mycompany.core.dao.impl.DefaultLoyaltyAccountDao"/>
    
    <!-- Services -->
    <bean id="loyaltyService" 
          class="com.mycompany.core.service.impl.DefaultLoyaltyService">
        <property name="loyaltyAccountDao" ref="loyaltyAccountDao"/>
        <property name="modelService" ref="modelService"/>
        <property name="loyaltyAccountIdGenerator" ref="loyaltyAccountIdGenerator"/>
    </bean>
    
    <!-- Key Generator -->
    <bean id="loyaltyAccountIdGenerator" 
          class="de.hybris.platform.servicelayer.keygenerator.impl.PersistentKeyGenerator">
        <property name="key" value="loyalty_account_id"/>
        <property name="digits" value="8"/>
        <property name="start" value="00000000"/>
        <property name="type" value="alphanumeric"/>
    </bean>
    
    <!-- Converters -->
    <bean id="loyaltyAccountConverter" parent="abstractPopulatingConverter">
        <property name="targetClass" value="com.mycompany.facades.data.LoyaltyAccountData"/>
        <property name="populators">
            <list>
                <ref bean="loyaltyAccountPopulator"/>
            </list>
        </property>
    </bean>
    
    <bean id="loyaltyAccountPopulator" 
          class="com.mycompany.facades.populators.LoyaltyAccountPopulator"/>
    
    <!-- Facades -->
    <bean id="loyaltyFacade" 
          class="com.mycompany.facades.impl.DefaultLoyaltyFacade">
        <property name="loyaltyService" ref="loyaltyService"/>
        <property name="userService" ref="userService"/>
        <property name="loyaltyAccountConverter" ref="loyaltyAccountConverter"/>
    </bean>
</beans>
```

### Overriding Platform Beans

To customize platform behavior, override existing beans using aliases:

```xml
<!-- Override the default CartService with your custom implementation -->
<alias name="myCustomCartService" alias="cartService"/>
<bean id="myCustomCartService" 
      class="com.mycompany.core.service.impl.MyCustomCartService"
      parent="defaultCartService">
    <property name="loyaltyService" ref="loyaltyService"/>
</bean>
```

The `alias` mechanism redirects all references to `cartService` to your custom bean, while `parent="defaultCartService"` inherits the original bean's configuration.

---

## Interceptors

Interceptors execute logic before or after model operations (load, save, remove, validate).

### Prepare Interceptor

Executes before `modelService.save()`:

```java
public class LoyaltyAccountPrepareInterceptor 
    implements PrepareInterceptor<LoyaltyAccountModel> {
    
    @Override
    public void onPrepare(LoyaltyAccountModel model, InterceptorContext ctx) 
        throws InterceptorException {
        // Auto-calculate tier before save
        if (ctx.isModified(model, LoyaltyAccountModel.POINTS)) {
            int points = model.getPoints() != null ? model.getPoints() : 0;
            if (points >= 20000) model.setTier(LoyaltyTier.PLATINUM);
            else if (points >= 5000) model.setTier(LoyaltyTier.GOLD);
            else if (points >= 1000) model.setTier(LoyaltyTier.SILVER);
            else model.setTier(LoyaltyTier.BRONZE);
        }
    }
}
```

### Validate Interceptor

Validates data before save:

```java
public class LoyaltyAccountValidateInterceptor 
    implements ValidateInterceptor<LoyaltyAccountModel> {
    
    @Override
    public void onValidate(LoyaltyAccountModel model, InterceptorContext ctx) 
        throws InterceptorException {
        if (model.getPoints() != null && model.getPoints() < 0) {
            throw new InterceptorException("Loyalty points cannot be negative");
        }
        if (StringUtils.isBlank(model.getAccountId())) {
            throw new InterceptorException("Account ID is required");
        }
    }
}
```

### Registering Interceptors

```xml
<bean id="loyaltyAccountPrepareInterceptor" 
      class="com.mycompany.core.interceptors.LoyaltyAccountPrepareInterceptor"/>

<bean id="loyaltyAccountPrepareInterceptorMapping"
      class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
    <property name="interceptor" ref="loyaltyAccountPrepareInterceptor"/>
    <property name="typeCode" value="LoyaltyAccount"/>
</bean>

<bean id="loyaltyAccountValidateInterceptor" 
      class="com.mycompany.core.interceptors.LoyaltyAccountValidateInterceptor"/>

<bean id="loyaltyAccountValidateInterceptorMapping"
      class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
    <property name="interceptor" ref="loyaltyAccountValidateInterceptor"/>
    <property name="typeCode" value="LoyaltyAccount"/>
</bean>
```

---

## Event Handling

SAP Commerce has an event system for decoupled communication between extensions.

### Defining a Custom Event

```java
public class LoyaltyTierChangedEvent extends AbstractEvent {
    private final String accountId;
    private final String previousTier;
    private final String newTier;
    
    public LoyaltyTierChangedEvent(String accountId, String previousTier, String newTier) {
        this.accountId = accountId;
        this.previousTier = previousTier;
        this.newTier = newTier;
    }
    
    // getters...
}
```

### Publishing Events

```java
@Resource
private EventService eventService;

public void earnPoints(LoyaltyAccountModel account, int points, OrderModel order) {
    LoyaltyTier previousTier = account.getTier();
    
    account.setPoints(account.getPoints() + points);
    account.setTier(calculateTier(account.getPoints()));
    modelService.save(account);
    
    if (previousTier != account.getTier()) {
        eventService.publishEvent(new LoyaltyTierChangedEvent(
            account.getAccountId(),
            previousTier.getCode(),
            account.getTier().getCode()
        ));
    }
}
```

### Listening to Events

```java
public class LoyaltyTierChangedEventListener 
    extends AbstractEventListener<LoyaltyTierChangedEvent> {
    
    private static final Logger LOG = LoggerFactory.getLogger(LoyaltyTierChangedEventListener.class);
    
    @Resource
    private NotificationService notificationService;
    
    @Override
    protected void onEvent(LoyaltyTierChangedEvent event) {
        LOG.info("Loyalty tier changed for account {}: {} → {}", 
            event.getAccountId(), event.getPreviousTier(), event.getNewTier());
        
        // Send notification, trigger email, update external system, etc.
        notificationService.sendTierUpgradeNotification(
            event.getAccountId(), event.getNewTier());
    }
}
```

```xml
<bean id="loyaltyTierChangedEventListener" 
      class="com.mycompany.core.event.LoyaltyTierChangedEventListener"
      parent="abstractEventListener"/>
```

---

## CronJobs

Background jobs for scheduled processing.

### Defining a CronJob

```xml
<!-- items.xml -->
<itemtype code="LoyaltyTierRecalculationCronJob"
          extends="CronJob"
          jaloclass="com.mycompany.core.jalo.LoyaltyTierRecalculationCronJob"
          autocreate="true"
          generate="true">
    <attributes>
        <attribute qualifier="batchSize" type="java.lang.Integer">
            <persistence type="property"/>
            <defaultvalue>Integer.valueOf(1000)</defaultvalue>
        </attribute>
    </attributes>
</itemtype>
```

### Job Performable

```java
public class LoyaltyTierRecalculationJob 
    extends AbstractJobPerformable<LoyaltyTierRecalculationCronJobModel> {
    
    @Resource
    private LoyaltyAccountDao loyaltyAccountDao;
    @Resource
    private LoyaltyService loyaltyService;
    
    @Override
    public PerformResult perform(LoyaltyTierRecalculationCronJobModel cronJob) {
        int batchSize = cronJob.getBatchSize() != null ? cronJob.getBatchSize() : 1000;
        int processed = 0;
        int updated = 0;
        
        List<LoyaltyAccountModel> accounts = loyaltyAccountDao.findAll(batchSize);
        
        for (LoyaltyAccountModel account : accounts) {
            if (clearAbortRequestedIfNeeded(cronJob)) {
                LOG.info("CronJob aborted after processing {} accounts", processed);
                return new PerformResult(CronJobResult.UNKNOWN, CronJobStatus.ABORTED);
            }
            
            LoyaltyTier correctTier = loyaltyService.calculateTier(account.getPoints());
            if (correctTier != account.getTier()) {
                account.setTier(correctTier);
                modelService.save(account);
                updated++;
            }
            processed++;
        }
        
        LOG.info("Tier recalculation complete. Processed: {}, Updated: {}", processed, updated);
        return new PerformResult(CronJobResult.SUCCESS, CronJobStatus.FINISHED);
    }
    
    @Override
    public boolean isAbortable() {
        return true;
    }
}
```

### Spring Registration and ImpEx Setup

```xml
<bean id="loyaltyTierRecalculationJob" 
      class="com.mycompany.core.job.LoyaltyTierRecalculationJob"
      parent="abstractJobPerformable"/>
```

```impex
INSERT_UPDATE ServicelayerJob;code[unique=true];springId
;loyaltyTierRecalculationJob;loyaltyTierRecalculationJob

INSERT_UPDATE LoyaltyTierRecalculationCronJob;code[unique=true];job(code);batchSize;sessionLanguage(isocode)
;loyaltyTierRecalculationCronJob;loyaltyTierRecalculationJob;1000;en

INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression
;loyaltyTierRecalculationCronJob;0 0 2 * * ?
```

---

## Extension Best Practices

### Naming Conventions

Follow SAP's naming conventions consistently:

| Component | Pattern | Example |
|-----------|---------|---------|
| Extension | `{project}{layer}` | `myprojectcore`, `myprojectfacades` |
| Interface | Descriptive noun | `LoyaltyService`, `LoyaltyAccountDao` |
| Implementation | `Default{Interface}` | `DefaultLoyaltyService` |
| Spring bean ID | camelCase interface name | `loyaltyService` |
| Populator | `{Type}Populator` | `LoyaltyAccountPopulator` |
| Interceptor | `{Type}{Action}Interceptor` | `LoyaltyAccountPrepareInterceptor` |
| CronJob | `{Description}CronJob` | `LoyaltyTierRecalculationCronJob` |

### Multi-Extension Project Structure

For a real project, create multiple extensions with clear responsibilities:

```
custom/
├── myprojectcore/           # Data model, services, DAOs, interceptors
├── myprojectfacades/        # Facades, converters, populators, DTOs
├── myprojectocc/            # OCC REST API controllers
├── myprojectbackoffice/     # Backoffice customizations
├── myprojectinitialdata/    # ImpEx data loading
└── myprojecttest/           # Integration tests
```

**Rule**: Core should have zero dependency on facades or web. Facades depend on core. Web depends on facades.

### Avoiding Common Mistakes

**1. Don't put business logic in controllers or populators.** They belong in services:

```java
// BAD — logic in populator
public void populate(OrderModel source, OrderData target) {
    double discount = source.getTotalPrice() > 100 ? 0.1 : 0;
    target.setDiscount(discount); // Business logic in wrong layer
}

// GOOD — logic in service, populator just maps
public void populate(OrderModel source, OrderData target) {
    target.setDiscount(source.getDiscount()); // Just mapping
}
```

**2. Don't use `modelService.save()` in interceptors that save.** This creates infinite recursion. Use `InterceptorContext.registerElementFor()` instead.

**3. Don't hardcode catalog versions, sites, or currencies.** Use session context or configuration properties:

```java
// BAD
CatalogVersionModel cv = catalogVersionService.getCatalogVersion("myProductCatalog", "Online");

// GOOD
CatalogVersionModel cv = catalogVersionService.getSessionCatalogVersionForCatalog("myProductCatalog");
```

**4. Always make services thread-safe.** Spring beans are singletons by default. Don't use instance variables to store request-scoped state.

---

## Summary

Building custom extensions in SAP Commerce follows a clear, layered pattern. The key principles:

1. **Respect the layered architecture** — Controllers → Facades → Services → DAOs → Models
2. **Define your data model carefully in items.xml** — type codes, indexes, and relations are hard to change later
3. **Use interceptors for cross-cutting concerns** — validation, auto-calculation, audit logging
4. **Leverage events for decoupled communication** — don't create tight coupling between extensions
5. **Follow naming conventions** — consistency across the team reduces cognitive load
6. **Keep extensions focused** — separate concerns into distinct extensions (core, facades, web)
7. **Write services that are testable** — inject dependencies, avoid static methods, don't mix layers

Well-structured extensions are the foundation of a maintainable SAP Commerce project. The time invested in proper architecture pays dividends throughout the project lifecycle.