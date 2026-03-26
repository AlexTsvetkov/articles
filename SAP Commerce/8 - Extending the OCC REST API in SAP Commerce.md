# Extending the OCC REST API in SAP Commerce: Custom Endpoints, DTOs, and Security

The OCC (Omni Commerce Connect) REST API is the standard interface for headless commerce with SAP Commerce. It powers Spartacus (the composable storefront), mobile apps, single-page applications, and third-party integrations. The out-of-the-box API covers most common commerce operations — products, carts, checkout, orders, user management. But every project needs custom endpoints: loyalty programs, custom search, B2B-specific workflows, integration callbacks.

This guide covers how to extend the OCC API properly — adding custom controllers, defining DTOs, configuring field-level mapping, securing endpoints, and following the patterns that make your extensions consistent with the standard API.

---

## OCC Architecture Overview

The OCC API follows a layered structure:

```
┌─────────────────────────────────────────────┐
│  HTTP Request                                │
├─────────────────────────────────────────────┤
│  OCC Controller (@RestController)            │  ← Request handling
├─────────────────────────────────────────────┤
│  Facade Layer                                │  ← Business orchestration
├─────────────────────────────────────────────┤
│  Data Mapping (Orika / Field-level config)   │  ← Model → DTO conversion
├─────────────────────────────────────────────┤
│  Service Layer                               │  ← Business logic
├─────────────────────────────────────────────┤
│  DAO / Persistence                           │  ← Data access
└─────────────────────────────────────────────┘
```

OCC uses several SAP Commerce-specific patterns:

- **WsDTO classes**: Data Transfer Objects annotated for serialization
- **Field-level configuration**: Controls which fields are returned at each detail level (BASIC, DEFAULT, FULL)
- **Orika mapper**: Handles conversion between models and DTOs
- **Spring Security OAuth2**: Secures endpoints with token-based authentication

### Standard URL Structure

```
/occ/v2/{baseSiteId}/products/{productCode}
/occ/v2/{baseSiteId}/users/{userId}/carts/{cartId}
/occ/v2/{baseSiteId}/orders/{orderCode}
```

The `{baseSiteId}` is mandatory and determines the commerce site context (catalog versions, currencies, languages).

---

## Creating a Custom OCC Extension

### Project Setup

Generate an OCC extension or add controllers to an existing web extension:

```
myprojectocc/
├── extensioninfo.xml
├── resources/
│   └── myprojectocc-spring.xml
├── web/
│   ├── src/
│   │   └── com/mycompany/occ/
│   │       ├── controllers/
│   │       │   └── LoyaltyController.java
│   │       ├── dto/
│   │       │   ├── LoyaltyAccountWsDTO.java
│   │       │   └── LoyaltyTransactionListWsDTO.java
│   │       └── validators/
│   │           └── LoyaltyRedeemValidator.java
│   ├── webroot/
│   │   └── WEB-INF/
│   │       ├── web.xml
│   │       └── config/
│   │           └── field-mapping-spring.xml
│   └── web-spring.xml
└── project.properties
```

The `extensioninfo.xml` should declare dependencies on the OCC infrastructure:

```xml
<extensioninfo>
    <extension name="myprojectocc" classprefix="myprojectocc">
        <requires-extension name="ycommercewebservices"/>
        <requires-extension name="commercewebservicescommons"/>
        <requires-extension name="myprojectfacades"/>
    </extension>
</extensioninfo>
```

---

## Writing Custom Controllers

### Basic REST Controller

```java
@RestController
@RequestMapping(value = "/{baseSiteId}/loyalty")
@Api(tags = "Loyalty")
public class LoyaltyController extends BaseController {

    @Resource
    private LoyaltyFacade loyaltyFacade;
    
    @Resource
    private DataMapper dataMapper;

    @GetMapping(value = "/account")
    @ResponseStatus(HttpStatus.OK)
    @ApiOperation(value = "Get current customer's loyalty account")
    public LoyaltyAccountWsDTO getLoyaltyAccount(
            @ApiParam(value = "Response field configuration") 
            @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
        
        LoyaltyAccountData accountData = loyaltyFacade.getCurrentCustomerLoyaltyAccount();
        return dataMapper.map(accountData, LoyaltyAccountWsDTO.class, fields);
    }

    @GetMapping(value = "/transactions")
    @ResponseStatus(HttpStatus.OK)
    @ApiOperation(value = "Get loyalty transaction history")
    public LoyaltyTransactionListWsDTO getTransactions(
            @RequestParam(defaultValue = "0") final int currentPage,
            @RequestParam(defaultValue = "10") final int pageSize,
            @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
        
        SearchPageData<LoyaltyTransactionData> transactions = 
            loyaltyFacade.getTransactionHistory(currentPage, pageSize);
        return dataMapper.map(transactions, LoyaltyTransactionListWsDTO.class, fields);
    }

    @PostMapping(value = "/earn")
    @ResponseStatus(HttpStatus.OK)
    @ApiOperation(value = "Earn loyalty points for an order")
    public LoyaltyAccountWsDTO earnPoints(
            @RequestParam final String orderCode,
            @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
        
        loyaltyFacade.earnPointsForOrder(orderCode);
        LoyaltyAccountData accountData = loyaltyFacade.getCurrentCustomerLoyaltyAccount();
        return dataMapper.map(accountData, LoyaltyAccountWsDTO.class, fields);
    }

    @PostMapping(value = "/redeem")
    @ResponseStatus(HttpStatus.OK)
    @ApiOperation(value = "Redeem loyalty points")
    public LoyaltyAccountWsDTO redeemPoints(
            @RequestBody final LoyaltyRedeemRequestWsDTO redeemRequest,
            @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
        
        validate(redeemRequest, "redeemRequest", loyaltyRedeemValidator);
        
        LoyaltyAccountData accountData = loyaltyFacade.redeemPoints(redeemRequest.getPoints());
        return dataMapper.map(accountData, LoyaltyAccountWsDTO.class, fields);
    }
}
```

### Key Patterns

**1. Extend `BaseController`**: Provides utility methods like `validate()` and common field set constants (`DEFAULT_FIELD_SET`, `BASIC_FIELD_SET`, `FULL_FIELD_SET`).

**2. Use `DataMapper`**: Never return model objects directly. Always map through DTOs using the `DataMapper` (Orika-based).

**3. Accept `fields` parameter**: This controls the response detail level. Clients can request `BASIC`, `DEFAULT`, or `FULL` fields, or specify individual fields.

**4. Follow RESTful conventions**:
- `GET` for reads
- `POST` for creates and actions
- `PUT` for full updates
- `PATCH` for partial updates
- `DELETE` for removals

---

## Defining WsDTO Classes

WsDTOs are the response/request objects serialized to JSON/XML.

### Response DTO

```java
@ApiModel(value = "LoyaltyAccount", description = "Loyalty account data")
public class LoyaltyAccountWsDTO implements Serializable {
    
    @ApiModelProperty(value = "Unique account identifier")
    private String accountId;
    
    @ApiModelProperty(value = "Current loyalty points balance")
    private Integer points;
    
    @ApiModelProperty(value = "Current loyalty tier")
    private String tier;
    
    @ApiModelProperty(value = "Date the account was created")
    private Date joinDate;
    
    @ApiModelProperty(value = "Customer display name")
    private String customerName;
    
    @ApiModelProperty(value = "Recent transactions")
    private List<LoyaltyTransactionWsDTO> recentTransactions;
    
    @ApiModelProperty(value = "Points needed for next tier")
    private Integer pointsToNextTier;
    
    // getters and setters
}
```

### Request DTO

```java
@ApiModel(value = "LoyaltyRedeemRequest", description = "Request to redeem loyalty points")
public class LoyaltyRedeemRequestWsDTO implements Serializable {
    
    @ApiModelProperty(value = "Number of points to redeem", required = true)
    private Integer points;
    
    @ApiModelProperty(value = "Cart to apply the discount to")
    private String cartId;
    
    // getters and setters
}
```

### List Wrapper DTO

For paginated results, wrap the list in a container DTO:

```java
@ApiModel(value = "LoyaltyTransactionList")
public class LoyaltyTransactionListWsDTO implements Serializable {
    
    private List<LoyaltyTransactionWsDTO> transactions;
    private PaginationWsDTO pagination;
    private List<SortWsDTO> sorts;
    
    // getters and setters
}
```

---

## Field-Level Mapping Configuration

One of the OCC API's most useful features is field-level configuration. Clients can control exactly which fields they receive, reducing payload size and improving performance.

### Configuration File

Create `field-mapping-spring.xml` in your web config:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- LoyaltyAccountWsDTO field configuration -->
    <bean parent="fieldSetLevelMapping">
        <property name="dtoClass" 
                  value="com.mycompany.occ.dto.LoyaltyAccountWsDTO"/>
        <property name="levelMapping">
            <map>
                <entry key="BASIC" value="accountId,points,tier"/>
                <entry key="DEFAULT" 
                       value="accountId,points,tier,joinDate,customerName,pointsToNextTier"/>
                <entry key="FULL" 
                       value="accountId,points,tier,joinDate,customerName,pointsToNextTier,recentTransactions(DEFAULT)"/>
            </map>
        </property>
    </bean>

    <!-- LoyaltyTransactionWsDTO field configuration -->
    <bean parent="fieldSetLevelMapping">
        <property name="dtoClass" 
                  value="com.mycompany.occ.dto.LoyaltyTransactionWsDTO"/>
        <property name="levelMapping">
            <map>
                <entry key="BASIC" value="transactionId,points"/>
                <entry key="DEFAULT" value="transactionId,points,description,date"/>
                <entry key="FULL" value="transactionId,points,description,date,orderCode"/>
            </map>
        </property>
    </bean>

    <!-- LoyaltyTransactionListWsDTO field configuration -->
    <bean parent="fieldSetLevelMapping">
        <property name="dtoClass" 
                  value="com.mycompany.occ.dto.LoyaltyTransactionListWsDTO"/>
        <property name="levelMapping">
            <map>
                <entry key="BASIC" value="transactions(BASIC),pagination"/>
                <entry key="DEFAULT" value="transactions(DEFAULT),pagination,sorts"/>
                <entry key="FULL" value="transactions(FULL),pagination,sorts"/>
            </map>
        </property>
    </bean>
</beans>
```

### How Field Levels Work

When a client calls:

```
GET /occ/v2/electronics/loyalty/account?fields=BASIC
```

They receive only `accountId`, `points`, and `tier`. With `fields=DEFAULT`, they also get `joinDate`, `customerName`, and `pointsToNextTier`. With `fields=FULL`, nested `recentTransactions` are included too.

Clients can also request specific fields:

```
GET /occ/v2/electronics/loyalty/account?fields=points,tier,pointsToNextTier
```

This returns only the three requested fields — useful for bandwidth-constrained mobile clients.

---

## Orika Mapper Configuration

The `DataMapper` uses Orika under the hood. For simple field-to-field mapping, it works automatically. For complex mappings, register custom converters.

### Custom Mapper

```java
@Component
public class LoyaltyAccountDataToWsDTOMapper implements Mapper<LoyaltyAccountData, LoyaltyAccountWsDTO> {

    @Override
    public Class<LoyaltyAccountData> getAType() {
        return LoyaltyAccountData.class;
    }

    @Override
    public Class<LoyaltyAccountWsDTO> getBType() {
        return LoyaltyAccountWsDTO.class;
    }

    @Override
    public void mapAtoB(LoyaltyAccountData source, LoyaltyAccountWsDTO target, MappingContext context) {
        target.setAccountId(source.getAccountId());
        target.setPoints(source.getPoints());
        target.setTier(source.getTier());
        target.setJoinDate(source.getJoinDate());
        target.setCustomerName(source.getCustomerName());
        
        // Calculate derived field
        target.setPointsToNextTier(calculatePointsToNextTier(source));
    }
    
    private Integer calculatePointsToNextTier(LoyaltyAccountData account) {
        int points = account.getPoints() != null ? account.getPoints() : 0;
        if ("PLATINUM".equals(account.getTier())) return 0;
        if ("GOLD".equals(account.getTier())) return 20000 - points;
        if ("SILVER".equals(account.getTier())) return 5000 - points;
        return 1000 - points;
    }

    @Override
    public void mapBtoA(LoyaltyAccountWsDTO source, LoyaltyAccountData target, MappingContext context) {
        // Reverse mapping (for request DTOs)
    }
}
```

Register the mapper:

```xml
<bean class="com.mycompany.occ.mappers.LoyaltyAccountDataToWsDTOMapper" parent="simpleWebMapper"/>
```

---

## Input Validation

Validate all incoming data before processing. OCC provides a validation framework.

### Custom Validator

```java
@Component
public class LoyaltyRedeemValidator implements Validator {
    
    @Override
    public boolean supports(Class<?> clazz) {
        return LoyaltyRedeemRequestWsDTO.class.isAssignableFrom(clazz);
    }
    
    @Override
    public void validate(Object target, Errors errors) {
        LoyaltyRedeemRequestWsDTO request = (LoyaltyRedeemRequestWsDTO) target;
        
        if (request.getPoints() == null || request.getPoints() <= 0) {
            errors.rejectValue("points", "field.invalid", 
                "Points must be a positive number");
        }
        
        if (request.getPoints() != null && request.getPoints() > 100000) {
            errors.rejectValue("points", "field.invalid", 
                "Cannot redeem more than 100,000 points at once");
        }
    }
}
```

### Using Validation in Controller

```java
@Resource
private Validator loyaltyRedeemValidator;

@PostMapping(value = "/redeem")
public LoyaltyAccountWsDTO redeemPoints(
        @RequestBody final LoyaltyRedeemRequestWsDTO request,
        @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
    
    validate(request, "redeemRequest", loyaltyRedeemValidator);
    // If validation fails, WebserviceValidationException is thrown automatically
    // and translated to a 400 Bad Request with error details
    
    LoyaltyAccountData data = loyaltyFacade.redeemPoints(request.getPoints());
    return dataMapper.map(data, LoyaltyAccountWsDTO.class, fields);
}
```

---

## Error Handling

Follow the OCC error response format for consistency:

### Standard Error Response

```json
{
    "errors": [
        {
            "type": "ValidationError",
            "message": "Points must be a positive number",
            "subject": "points",
            "subjectType": "parameter",
            "reason": "invalid"
        }
    ]
}
```

### Custom Exception Handling

```java
@ControllerAdvice
public class LoyaltyExceptionHandler {
    
    @ExceptionHandler(InsufficientLoyaltyPointsException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ResponseBody
    public ErrorListWsDTO handleInsufficientPoints(InsufficientLoyaltyPointsException ex) {
        ErrorWsDTO error = new ErrorWsDTO();
        error.setType("InsufficientPointsError");
        error.setMessage(ex.getMessage());
        error.setReason("insufficientPoints");
        
        ErrorListWsDTO errorList = new ErrorListWsDTO();
        errorList.setErrors(Collections.singletonList(error));
        return errorList;
    }
}
```

---

## Security Configuration

OCC endpoints are secured via OAuth2. You need to configure access control for your custom endpoints.

### OAuth2 Scopes

SAP Commerce defines standard OAuth2 scopes:

- `basic` — Read operations for anonymous users
- `extended` — Read operations for authenticated users
- `customer` — Customer-specific operations (carts, orders, profile)
- `admin` — Administrative operations

### Configuring Endpoint Security

In `web-spring.xml`:

```xml
<beans xmlns:security="http://www.springframework.org/schema/security">

    <!-- Require authentication for loyalty endpoints -->
    <security:http pattern="/#{configurationService.configuration.getString('occ.rewrite.overlapping.paths.enabled','false')=='true' ? '' : 'occ/v2/'}*/loyalty/**" 
                   use-expressions="true" 
                   entry-point-ref="oauthAuthenticationEntryPoint">
        <security:intercept-url pattern="/**" access="isAuthenticated() and hasRole('ROLE_CUSTOMERGROUP')" method="GET"/>
        <security:intercept-url pattern="/**" access="isAuthenticated() and hasRole('ROLE_CUSTOMERGROUP')" method="POST"/>
        <security:custom-filter ref="resourceServerFilter" before="PRE_AUTH_FILTER"/>
    </security:http>

</beans>
```

### Role-Based Access Control

```java
@PostMapping(value = "/admin/recalculate")
@PreAuthorize("hasRole('ROLE_TRUSTED_CLIENT') or hasRole('ROLE_CUSTOMERMANAGERGROUP')")
@ResponseStatus(HttpStatus.OK)
public void recalculateAllTiers() {
    loyaltyFacade.recalculateAllTiers();
}
```

### Anonymous Access

For endpoints that should be accessible without authentication (e.g., loyalty program info page):

```java
@GetMapping(value = "/program-info")
@ResponseStatus(HttpStatus.OK)
@ApiOperation(value = "Get loyalty program information (public)")
public LoyaltyProgramInfoWsDTO getProgramInfo(
        @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
    
    LoyaltyProgramInfoData data = loyaltyFacade.getProgramInfo();
    return dataMapper.map(data, LoyaltyProgramInfoWsDTO.class, fields);
}
```

Configure anonymous access in security:

```xml
<security:http pattern="/*/loyalty/program-info" security="none"/>
```

---

## API Versioning and Documentation

### Swagger/OpenAPI Documentation

OCC integrates with Swagger. Your `@Api` and `@ApiOperation` annotations are automatically picked up:

```java
@RestController
@RequestMapping(value = "/{baseSiteId}/loyalty")
@Api(tags = "Loyalty", description = "Loyalty program operations")
public class LoyaltyController extends BaseController {

    @GetMapping(value = "/account")
    @ApiOperation(
        value = "Get loyalty account",
        notes = "Returns the loyalty account for the currently authenticated customer. " +
                "Creates an account automatically if one doesn't exist.",
        response = LoyaltyAccountWsDTO.class
    )
    @ApiResponses({
        @ApiResponse(code = 200, message = "Loyalty account retrieved successfully"),
        @ApiResponse(code = 401, message = "Authentication required"),
        @ApiResponse(code = 403, message = "Access denied — customer role required")
    })
    public LoyaltyAccountWsDTO getLoyaltyAccount(
            @ApiParam(value = "Response field configuration level", 
                      allowableValues = "BASIC,DEFAULT,FULL")
            @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
        // ...
    }
}
```

Access Swagger UI at: `/occ/v2/swagger-ui.html`

### Versioning Strategy

The standard OCC API uses `/v2/` in the URL. For your custom endpoints, stay within the same version path. If you need breaking changes, consider:

1. **New endpoint path**: `/loyalty/v2/account` alongside the original `/loyalty/account`
2. **Query parameter versioning**: `?version=2` (less common in OCC)
3. **Backward-compatible changes**: Add new optional fields rather than changing existing ones

---

## Testing OCC Endpoints

### Integration Tests

```java
@IntegrationTest
@NeedsEmbeddedServer(webExtensions = {"myprojectocc", "oauth2"})
public class LoyaltyControllerIntegrationTest extends ServicelayerBaseTest {
    
    private static final String LOYALTY_ACCOUNT_ENDPOINT = "/occ/v2/electronics/loyalty/account";
    
    @Resource
    private OAuthTokenHelper oAuthTokenHelper;
    
    @Test
    public void testGetLoyaltyAccount_authenticatedCustomer() {
        // Get OAuth token for test customer
        String token = oAuthTokenHelper.getCustomerToken("testcustomer@test.com", "password");
        
        Response response = given()
            .header("Authorization", "Bearer " + token)
            .param("fields", "DEFAULT")
        .when()
            .get(LOYALTY_ACCOUNT_ENDPOINT)
        .then()
            .statusCode(200)
            .body("accountId", notNullValue())
            .body("points", greaterThanOrEqualTo(0))
            .body("tier", isOneOf("BRONZE", "SILVER", "GOLD", "PLATINUM"))
            .extract().response();
    }
    
    @Test
    public void testGetLoyaltyAccount_anonymous_returns401() {
        given()
        .when()
            .get(LOYALTY_ACCOUNT_ENDPOINT)
        .then()
            .statusCode(401);
    }
    
    @Test
    public void testRedeemPoints_insufficientBalance() {
        String token = oAuthTokenHelper.getCustomerToken("testcustomer@test.com", "password");
        
        given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .body("{\"points\": 999999}")
        .when()
            .post(LOYALTY_ACCOUNT_ENDPOINT.replace("account", "redeem"))
        .then()
            .statusCode(400)
            .body("errors[0].type", equalTo("InsufficientPointsError"));
    }
}
```

### Manual Testing with cURL

```bash
# 1. Get OAuth token
TOKEN=$(curl -s -X POST "https://localhost:9002/authorizationserver/oauth/token" \
  -d "client_id=mobile_android&client_secret=secret&grant_type=password&username=testcustomer@test.com&password=1234" \
  | jq -r '.access_token')

# 2. Get loyalty account
curl -H "Authorization: Bearer $TOKEN" \
  "https://localhost:9002/occ/v2/electronics/loyalty/account?fields=FULL"

# 3. Earn points
curl -X POST -H "Authorization: Bearer $TOKEN" \
  "https://localhost:9002/occ/v2/electronics/loyalty/earn?orderCode=00001001"

# 4. Redeem points
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"points": 500}' \
  "https://localhost:9002/occ/v2/electronics/loyalty/redeem"
```

---

## Performance Considerations

### Response Caching

For read-heavy endpoints, implement HTTP caching:

```java
@GetMapping(value = "/program-info")
public ResponseEntity<LoyaltyProgramInfoWsDTO> getProgramInfo(
        @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
    
    LoyaltyProgramInfoData data = loyaltyFacade.getProgramInfo();
    LoyaltyProgramInfoWsDTO dto = dataMapper.map(data, LoyaltyProgramInfoWsDTO.class, fields);
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic())
        .body(dto);
}
```

### Pagination

Always paginate list endpoints:

```java
@GetMapping(value = "/transactions")
public LoyaltyTransactionListWsDTO getTransactions(
        @RequestParam(defaultValue = "0") final int currentPage,
        @RequestParam(defaultValue = "20") final int pageSize,
        @RequestParam(defaultValue = "date:desc") final String sort,
        @RequestParam(defaultValue = DEFAULT_FIELD_SET) final String fields) {
    
    // Cap page size to prevent abuse
    int effectivePageSize = Math.min(pageSize, 100);
    
    PageableData pageableData = new PageableData();
    pageableData.setCurrentPage(currentPage);
    pageableData.setPageSize(effectivePageSize);
    pageableData.setSort(sort);
    
    SearchPageData<LoyaltyTransactionData> results = 
        loyaltyFacade.getTransactionHistory(pageableData);
    
    return dataMapper.map(results, LoyaltyTransactionListWsDTO.class, fields);
}
```

### Keep Field Sets Lean

The `BASIC` field set should be minimal — just the essential identifiers and key attributes needed for list views. The `DEFAULT` set adds commonly displayed fields. `FULL` includes everything, including nested objects and computed fields. Heavy computed fields should only be in `FULL` to avoid unnecessary processing on list pages.

---

## Summary

Extending the OCC REST API follows established patterns that keep your custom endpoints consistent with the platform's standard API. The key principles:

1. **Extend `BaseController`** — it provides validation utilities and field set constants
2. **Use the `DataMapper` for all conversions** — never return model objects directly from controllers
3. **Configure field-level mappings** — let clients control response payload size with BASIC/DEFAULT/FULL
4. **Validate all input** — use Spring's `Validator` interface and the `validate()` helper
5. **Secure endpoints with OAuth2** — configure role-based access in Spring Security
6. **Follow RESTful conventions** — proper HTTP methods, status codes, and URL structure
7. **Document with Swagger annotations** — they power the auto-generated API documentation
8. **Always paginate list endpoints** — cap page sizes to prevent abuse
9. **Write integration tests** — test the full HTTP stack, not just the Java methods

Well-designed OCC extensions feel like natural parts of the platform API, making life easier for the frontend teams and integration partners who consume them.