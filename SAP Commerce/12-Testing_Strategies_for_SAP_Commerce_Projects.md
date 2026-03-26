# Testing Strategies for SAP Commerce Projects: Unit, Integration, and End-to-End

Testing SAP Commerce code presents unique challenges. The platform's deep dependency injection, tight coupling to the type system, and reliance on database-backed models mean that standard unit testing practices from vanilla Spring Boot don't translate directly. Many teams end up with either no tests or tests that boot the entire platform for every assertion — both approaches fail at scale.

This article covers practical testing strategies that work for real SAP Commerce projects: unit tests with proper mocking, integration tests against the running platform, Solr search testing, ImpEx validation, and end-to-end testing of OCC APIs.

---

## The Testing Pyramid for SAP Commerce

```
                    ┌─────────┐
                    │  E2E    │  Few: OCC API tests, Storefront smoke tests
                    │  Tests  │  Slow, expensive, run in CI pipeline
                   ┌┴─────────┴┐
                   │ Integration│  Moderate: Service tests against running platform
                   │   Tests    │  Boot platform once, run many tests
                  ┌┴────────────┴┐
                  │    Unit       │  Many: Business logic, converters, populators
                  │    Tests      │  Fast, no platform needed, run on every save
                  └───────────────┘
```

### What Goes Where

| Test Type | What to Test | Platform Required | Speed |
|-----------|-------------|-------------------|-------|
| Unit | Populators, converters, validators, business logic | No | Milliseconds |
| Integration | Services, facades, FlexibleSearch, ImpEx | Yes | Seconds |
| Solr | Search queries, indexing, facets | Yes + Solr | Seconds |
| E2E / API | OCC endpoints, full request lifecycle | Yes + Web | Seconds |
| Performance | Load testing, response times under concurrency | Full environment | Minutes |

---

## Unit Testing

Unit tests are the foundation. They test individual classes in isolation by mocking all dependencies.

### Testing a Populator

Populators are the most unit-testable code in SAP Commerce. They transform model objects into data objects.

```java
public class LoyaltyPointsPopulator implements Populator<CustomerModel, CustomerData> {
    
    private LoyaltyService loyaltyService;
    
    @Override
    public void populate(CustomerModel source, CustomerData target) {
        if (source == null || target == null) {
            throw new ConversionException("Source or target is null");
        }
        
        LoyaltyAccount account = loyaltyService.getAccountForCustomer(source);
        if (account != null) {
            target.setLoyaltyPoints(account.getPoints());
            target.setLoyaltyTier(account.getTier().name());
            target.setPointsToNextTier(calculatePointsToNextTier(account));
        }
    }
    
    private int calculatePointsToNextTier(LoyaltyAccount account) {
        Map<String, Integer> thresholds = Map.of(
            "BRONZE", 1000, "SILVER", 5000, "GOLD", 20000, "PLATINUM", Integer.MAX_VALUE
        );
        int nextThreshold = thresholds.getOrDefault(account.getTier().name(), Integer.MAX_VALUE);
        return Math.max(0, nextThreshold - account.getPoints());
    }
    
    // Setter for injection
    public void setLoyaltyService(LoyaltyService loyaltyService) {
        this.loyaltyService = loyaltyService;
    }
}
```

**Test:**

```java
@UnitTest
@RunWith(MockitoJUnitRunner.class)
public class LoyaltyPointsPopulatorTest {
    
    @InjectMocks
    private LoyaltyPointsPopulator populator;
    
    @Mock
    private LoyaltyService loyaltyService;
    
    @Mock
    private CustomerModel customerModel;
    
    private CustomerData customerData;
    
    @Before
    public void setUp() {
        customerData = new CustomerData();
    }
    
    @Test
    public void shouldPopulateLoyaltyPoints() {
        // Given
        LoyaltyAccount account = new LoyaltyAccount();
        account.setPoints(3500);
        account.setTier(LoyaltyTier.SILVER);
        
        when(loyaltyService.getAccountForCustomer(customerModel)).thenReturn(account);
        
        // When
        populator.populate(customerModel, customerData);
        
        // Then
        assertEquals(3500, customerData.getLoyaltyPoints());
        assertEquals("SILVER", customerData.getLoyaltyTier());
        assertEquals(1500, customerData.getPointsToNextTier()); // 5000 - 3500
    }
    
    @Test
    public void shouldHandleNullLoyaltyAccount() {
        when(loyaltyService.getAccountForCustomer(customerModel)).thenReturn(null);
        
        populator.populate(customerModel, customerData);
        
        assertEquals(0, customerData.getLoyaltyPoints());
        assertNull(customerData.getLoyaltyTier());
    }
    
    @Test(expected = ConversionException.class)
    public void shouldThrowOnNullSource() {
        populator.populate(null, customerData);
    }
    
    @Test(expected = ConversionException.class)
    public void shouldThrowOnNullTarget() {
        populator.populate(customerModel, null);
    }
    
    @Test
    public void shouldCalculatePointsToNextTierForBronze() {
        LoyaltyAccount account = new LoyaltyAccount();
        account.setPoints(250);
        account.setTier(LoyaltyTier.BRONZE);
        
        when(loyaltyService.getAccountForCustomer(customerModel)).thenReturn(account);
        
        populator.populate(customerModel, customerData);
        
        assertEquals(750, customerData.getPointsToNextTier()); // 1000 - 250
    }
}
```

### Testing a Validator

```java
public class OrderQuantityValidator implements Validator {
    
    private static final int MAX_QUANTITY = 999;
    private StockService stockService;
    
    @Override
    public boolean supports(Class<?> clazz) {
        return AddToCartParams.class.isAssignableFrom(clazz);
    }
    
    @Override
    public void validate(Object target, Errors errors) {
        AddToCartParams params = (AddToCartParams) target;
        
        if (params.getQuantity() <= 0) {
            errors.rejectValue("quantity", "cart.quantity.invalid",
                "Quantity must be greater than zero");
        }
        
        if (params.getQuantity() > MAX_QUANTITY) {
            errors.rejectValue("quantity", "cart.quantity.exceeded",
                new Object[]{MAX_QUANTITY}, "Maximum quantity is {0}");
        }
        
        // Check stock availability
        Long available = stockService.getAvailableStock(params.getProductCode());
        if (available != null && params.getQuantity() > available) {
            errors.rejectValue("quantity", "cart.quantity.nostock",
                new Object[]{available}, "Only {0} items available");
        }
    }
}
```

**Test:**

```java
@UnitTest
@RunWith(MockitoJUnitRunner.class)
public class OrderQuantityValidatorTest {
    
    @InjectMocks
    private OrderQuantityValidator validator;
    
    @Mock
    private StockService stockService;
    
    private AddToCartParams params;
    private BeanPropertyBindingResult errors;
    
    @Before
    public void setUp() {
        params = new AddToCartParams();
        params.setProductCode("PROD-001");
        params.setQuantity(1);
        errors = new BeanPropertyBindingResult(params, "params");
    }
    
    @Test
    public void shouldAcceptValidQuantity() {
        when(stockService.getAvailableStock("PROD-001")).thenReturn(100L);
        
        validator.validate(params, errors);
        
        assertFalse(errors.hasErrors());
    }
    
    @Test
    public void shouldRejectZeroQuantity() {
        params.setQuantity(0);
        
        validator.validate(params, errors);
        
        assertTrue(errors.hasFieldErrors("quantity"));
        assertEquals("cart.quantity.invalid", 
            errors.getFieldError("quantity").getCode());
    }
    
    @Test
    public void shouldRejectNegativeQuantity() {
        params.setQuantity(-5);
        
        validator.validate(params, errors);
        
        assertTrue(errors.hasFieldErrors("quantity"));
    }
    
    @Test
    public void shouldRejectQuantityExceedingMax() {
        params.setQuantity(1000);
        when(stockService.getAvailableStock("PROD-001")).thenReturn(5000L);
        
        validator.validate(params, errors);
        
        assertTrue(errors.hasFieldErrors("quantity"));
        assertEquals("cart.quantity.exceeded",
            errors.getFieldError("quantity").getCode());
    }
    
    @Test
    public void shouldRejectQuantityExceedingStock() {
        params.setQuantity(10);
        when(stockService.getAvailableStock("PROD-001")).thenReturn(5L);
        
        validator.validate(params, errors);
        
        assertTrue(errors.hasFieldErrors("quantity"));
        assertEquals("cart.quantity.nostock",
            errors.getFieldError("quantity").getCode());
    }
    
    @Test
    public void shouldAllowWhenStockIsNull() {
        // Null stock means stock tracking is disabled
        params.setQuantity(50);
        when(stockService.getAvailableStock("PROD-001")).thenReturn(null);
        
        validator.validate(params, errors);
        
        assertFalse(errors.hasErrors());
    }
}
```

---

## Integration Testing

Integration tests run against a booted SAP Commerce platform. They test that services, the database, and the type system work together correctly.

### ServicelayerTransactionalTest Base Class

SAP Commerce provides `ServicelayerTransactionalTest` as the base class for integration tests. It boots the platform, provides Spring dependency injection, and wraps each test in a transaction that rolls back automatically.

```java
@IntegrationTest
public class DefaultLoyaltyServiceIntegrationTest extends ServicelayerTransactionalTest {
    
    @Resource
    private LoyaltyService loyaltyService;
    
    @Resource
    private UserService userService;
    
    @Resource
    private ModelService modelService;
    
    private CustomerModel customer;
    
    @Before
    public void setUp() throws Exception {
        // Load test data via ImpEx
        importCsv("/test/testdata/loyalty-test-data.impex", "utf-8");
        
        customer = userService.getUserForUID("testcustomer@example.com", CustomerModel.class);
    }
    
    @Test
    public void shouldCreateLoyaltyAccount() {
        // When
        LoyaltyAccountModel account = loyaltyService.getOrCreateAccount(customer);
        
        // Then
        assertNotNull(account);
        assertEquals(0, account.getPoints().intValue());
        assertEquals(LoyaltyTier.BRONZE, account.getTier());
        assertEquals(customer, account.getCustomer());
    }
    
    @Test
    public void shouldAccruePoints() {
        LoyaltyAccountModel account = loyaltyService.getOrCreateAccount(customer);
        
        // When
        loyaltyService.accruePoints(customer, 500, "Order ORD-001");
        
        // Then
        modelService.refresh(account);
        assertEquals(500, account.getPoints().intValue());
    }
    
    @Test
    public void shouldUpgradeTierWhenThresholdReached() {
        loyaltyService.getOrCreateAccount(customer);
        
        // Accrue enough points for Silver tier (threshold: 1000)
        loyaltyService.accruePoints(customer, 1200, "Large order");
        
        LoyaltyAccountModel account = loyaltyService.getAccount(customer);
        assertEquals(LoyaltyTier.SILVER, account.getTier());
    }
    
    @Test(expected = InsufficientPointsException.class)
    public void shouldRejectRedemptionWithInsufficientPoints() {
        loyaltyService.getOrCreateAccount(customer);
        loyaltyService.accruePoints(customer, 100, "Small order");
        
        // Try to redeem more than available
        loyaltyService.redeemPoints(customer, 500, "Redemption");
    }
}
```

### Test ImpEx Data

```impex
# /test/testdata/loyalty-test-data.impex
$catalogVersion = catalogVersion(catalog(id[default='testCatalog']),version[default='Online'])

INSERT_UPDATE Customer;uid[unique=true];name;groups(uid)
;testcustomer@example.com;Test Customer;customergroup

INSERT_UPDATE Product;code[unique=true];name[lang=en];$catalogVersion;approvalStatus(code)
;TEST-PROD-001;Test Product 1;;approved
;TEST-PROD-002;Test Product 2;;approved

INSERT_UPDATE PriceRow;product(code,$catalogVersion)[unique=true];price;currency(isocode)[unique=true];net
;TEST-PROD-001;99.99;USD;false
;TEST-PROD-002;49.99;USD;false
```

---

## Testing FlexibleSearch Queries

FlexibleSearch queries need integration tests because they depend on the type system and database schema.

```java
@IntegrationTest
public class ProductQueryServiceIntegrationTest extends ServicelayerTransactionalTest {
    
    @Resource
    private ProductQueryService productQueryService;
    
    @Resource
    private CatalogVersionService catalogVersionService;
    
    @Before
    public void setUp() throws Exception {
        importCsv("/test/testdata/product-query-test-data.impex", "utf-8");
    }
    
    @Test
    public void shouldFindProductsByPriceRange() {
        CatalogVersionModel cv = catalogVersionService.getCatalogVersion("testCatalog", "Online");
        
        List<ProductModel> results = productQueryService.findByPriceRange(
            cv, BigDecimal.valueOf(50), BigDecimal.valueOf(150));
        
        assertEquals(2, results.size());
        assertTrue(results.stream()
            .allMatch(p -> p.getCode().startsWith("TEST-PROD")));
    }
    
    @Test
    public void shouldReturnEmptyForNonMatchingRange() {
        CatalogVersionModel cv = catalogVersionService.getCatalogVersion("testCatalog", "Online");
        
        List<ProductModel> results = productQueryService.findByPriceRange(
            cv, BigDecimal.valueOf(10000), BigDecimal.valueOf(20000));
        
        assertTrue(results.isEmpty());
    }
    
    @Test
    public void shouldFindRecentlyModifiedProducts() {
        CatalogVersionModel cv = catalogVersionService.getCatalogVersion("testCatalog", "Online");
        
        // Modify a product
        ProductModel product = productService.getProductForCode(cv, "TEST-PROD-001");
        product.setDescription("Updated description");
        modelService.save(product);
        
        // Query for recently modified
        List<ProductModel> results = productQueryService.findModifiedSince(
            cv, Date.from(Instant.now().minus(1, ChronoUnit.HOURS)));
        
        assertFalse(results.isEmpty());
        assertTrue(results.stream()
            .anyMatch(p -> "TEST-PROD-001".equals(p.getCode())));
    }
}
```

---

## Testing OCC API Endpoints

For end-to-end API testing, use REST-assured or Spring's `MockMvc` against the running webserver.

### REST-Assured Tests

```java
@IntegrationTest
@NeedsEmbeddedServer(webExtensions = {"commercewebservices"})
public class ProductOCCApiTest extends ServicelayerTransactionalTest {
    
    private static final String BASE_PATH = "/occ/v2/electronics-spa";
    
    @Before
    public void setUp() throws Exception {
        importCsv("/test/testdata/occ-test-data.impex", "utf-8");
    }
    
    @Test
    public void shouldReturnProductDetails() {
        given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
        .when()
            .get("/products/TEST-PROD-001")
        .then()
            .statusCode(200)
            .body("code", equalTo("TEST-PROD-001"))
            .body("name", equalTo("Test Product 1"))
            .body("price.value", notNullValue())
            .body("stock.stockLevelStatus", equalTo("inStock"));
    }
    
    @Test
    public void shouldReturn404ForNonExistentProduct() {
        given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
        .when()
            .get("/products/NONEXISTENT")
        .then()
            .statusCode(400)
            .body("errors[0].type", equalTo("UnknownIdentifierError"));
    }
    
    @Test
    public void shouldSearchProducts() {
        given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
            .queryParam("query", "test")
            .queryParam("fields", "FULL")
        .when()
            .get("/products/search")
        .then()
            .statusCode(200)
            .body("products.size()", greaterThan(0))
            .body("pagination.totalResults", greaterThan(0));
    }
    
    @Test
    public void shouldAddToCartAndRetrieve() {
        // Create anonymous cart
        String cartGuid = given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
        .when()
            .post("/users/anonymous/carts")
        .then()
            .statusCode(201)
            .extract().path("guid");
        
        // Add product to cart
        given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
            .contentType("application/json")
            .body("{\"product\":{\"code\":\"TEST-PROD-001\"},\"quantity\":2}")
        .when()
            .post("/users/anonymous/carts/" + cartGuid + "/entries")
        .then()
            .statusCode(200)
            .body("quantityAdded", equalTo(2));
        
        // Verify cart contents
        given()
            .baseUri("https://localhost:9002")
            .basePath(BASE_PATH)
            .relaxedHTTPSValidation()
        .when()
            .get("/users/anonymous/carts/" + cartGuid)
        .then()
            .statusCode(200)
            .body("entries.size()", equalTo(1))
            .body("entries[0].product.code", equalTo("TEST-PROD-001"))
            .body("entries[0].quantity", equalTo(2));
    }
}
```

---

## Testing ImpEx Scripts

ImpEx scripts are code — they should be tested too.

```java
@IntegrationTest
public class ProductImportImpExTest extends ServicelayerTransactionalTest {
    
    @Resource
    private ProductService productService;
    
    @Resource
    private CatalogVersionService catalogVersionService;
    
    @Resource
    private ImportService importService;
    
    @Test
    public void shouldImportProductsSuccessfully() {
        // Given
        ImportConfig config = new ImportConfig();
        config.setScript(
            new StreamBasedImpExResource(
                getClass().getResourceAsStream("/import/products.impex"),
                "utf-8"
            )
        );
        config.setSynchronous(true);
        config.setFailOnError(true);
        
        // When
        ImportResult result = importService.importData(config);
        
        // Then
        assertTrue("Import should succeed", result.isSuccessful());
        assertFalse("Should have no unresolved lines", result.hasUnresolvedLines());
        
        // Verify imported data
        CatalogVersionModel cv = catalogVersionService
            .getCatalogVersion("myProductCatalog", "Staged");
        ProductModel product = productService.getProductForCode(cv, "CAM-001");
        
        assertNotNull("Product should exist", product);
        assertEquals("Professional DSLR", product.getName());
    }
    
    @Test
    public void shouldHandleDuplicateProductCodes() {
        // Import same file twice — INSERT_UPDATE should handle it
        ImportConfig config = new ImportConfig();
        config.setScript(new StreamBasedImpExResource(
            getClass().getResourceAsStream("/import/products.impex"), "utf-8"));
        config.setSynchronous(true);
        
        // First import
        ImportResult result1 = importService.importData(config);
        assertTrue(result1.isSuccessful());
        
        // Second import (should update, not fail)
        config.setScript(new StreamBasedImpExResource(
            getClass().getResourceAsStream("/import/products.impex"), "utf-8"));
        ImportResult result2 = importService.importData(config);
        assertTrue("Re-import should succeed with INSERT_UPDATE", result2.isSuccessful());
    }
}
```

---

## Test Configuration

### Running Tests

```bash
# Run unit tests only (fast, no platform boot)
ant unittests -Dtestclasses.packages=com.mycompany.*

# Run integration tests (boots platform)
ant integrationtests -Dtestclasses.packages=com.mycompany.*

# Run specific test class
ant integrationtests -Dtestclasses.packages=com.mycompany.core.service.DefaultLoyaltyServiceIntegrationTest

# Run all tests
ant alltests -Dtestclasses.packages=com.mycompany.*
```

### Test Properties

```properties
# local.properties for test execution

# Use HSQLDB for fast integration tests (instead of MySQL/HANA)
db.url=jdbc:hsqldb:mem:testDB
db.driver=org.hsqldb.jdbcDriver
db.username=sa
db.password=

# Disable Solr for tests that don't need it
solrserver.instances.default.autostart=false

# Reduce logging noise during tests
log4j2.logger.test.name = de.hybris.platform
log4j2.logger.test.level = WARN
```

---

## CI/CD Integration

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'ant clean all'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'ant unittests -Dtestclasses.packages=com.mycompany.*'
            }
            post {
                always {
                    junit '**/testresults/junit/*.xml'
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'ant integrationtests -Dtestclasses.packages=com.mycompany.*'
            }
            post {
                always {
                    junit '**/testresults/junit/*.xml'
                }
            }
        }
        
        stage('API Tests') {
            steps {
                sh 'ant integrationtests -Dtestclasses.packages=com.mycompany.*.api.*'
            }
        }
    }
    
    post {
        always {
            publishHTML(target: [
                reportDir: 'log/junit',
                reportFiles: 'index.html',
                reportName: 'Test Report'
            ])
        }
    }
}
```

---

## Best Practices

1. **Unit test everything that can be unit tested** — populators, converters, validators, and utility classes don't need the platform.

2. **Use `@UnitTest` and `@IntegrationTest` annotations** — they ensure correct test runner selection and classpath setup.

3. **Load minimal test data** — each test should import only the data it needs. Large shared test data files create fragile, slow tests.

4. **Use transactions for isolation** — `ServicelayerTransactionalTest` rolls back each test automatically, keeping the database clean.

5. **Test error paths** — null inputs, missing data, concurrent modifications, and invalid states are where bugs hide.

6. **Keep integration tests focused** — test one service behavior per test method. Don't write integration tests that validate 10 things at once.

7. **Mock external systems** — third-party APIs, payment gateways, and external services should be mocked in both unit and integration tests.

8. **Run unit tests on every commit, integration tests on every PR** — unit tests should complete in seconds, integration tests can take minutes.

---

## Summary

A well-tested SAP Commerce project needs tests at every level:

1. **Unit tests** for business logic, populators, validators — fast, no platform required
2. **Integration tests** for services, FlexibleSearch, and ImpEx — running against the platform with transactional rollback
3. **API tests** for OCC endpoints — verifying the full HTTP request/response cycle
4. **ImpEx tests** for data import scripts — ensuring imports are idempotent and produce correct data
5. **Automated in CI/CD** — unit tests on every commit, integration tests on PRs, full suites before deployment

The teams that invest in testing spend less time debugging production issues and more time building features. The platform's complexity makes testing harder, but it also makes testing more valuable — there are simply more places where things can go wrong.