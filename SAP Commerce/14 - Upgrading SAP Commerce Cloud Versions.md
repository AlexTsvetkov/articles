# Upgrading SAP Commerce Cloud: A Step-by-Step Migration Guide

Upgrading SAP Commerce to a new major or minor version is one of the highest-risk activities in a project's lifecycle. It touches every layer — the type system, database schema, Spring configuration, extension APIs, and storefront integration. Skipping versions amplifies the risk. Teams that stay two or three releases behind face exponentially more breaking changes, deprecated API removals, and compatibility issues when they finally upgrade.

This guide covers the complete upgrade process: planning, dependency analysis, code migration, testing, and deployment — drawn from real upgrade projects across SAP Commerce 1905 through 2211.

---

## Why Upgrade?

Before investing the effort, understand what you gain:

| Motivation | Examples |
|-----------|----------|
| **Security patches** | CVE fixes, updated libraries (Log4j, Spring, Jackson) |
| **Performance improvements** | Better caching, optimized queries, reduced memory |
| **New features** | Composable Storefront improvements, OCC enhancements |
| **SAP support** | Older versions reach end of mainstream maintenance |
| **Cloud compliance** | CCv2 may require minimum platform versions |
| **Third-party compatibility** | Java version support, database driver updates |

### SAP Commerce Release Cadence

```
Release naming: YYMM (e.g., 2211 = 2022 November)
Patch releases: 2211.0, 2211.1, ..., 2211.25

Support timeline (typical):
├── Mainstream maintenance: ~3 years from release
├── Extended maintenance: ~2 additional years (paid)
└── End of life: No patches, no support

Active versions (as of 2025):
  2211 — Current, actively maintained
  2205 — Maintenance mode
  2105 — Extended maintenance
  2011 — End of life (upgrade urgently)
```

---

## Upgrade Planning

### Step 1: Assess the Gap

```
Current Version: 2105.15
Target Version:  2211.25

Gap analysis:
┌────────────────────────────────┬─────────┐
│ Category                        │ Impact  │
├────────────────────────────────┼─────────┤
│ Removed/deprecated APIs         │ HIGH    │
│ Type system changes             │ MEDIUM  │
│ Spring framework upgrade        │ HIGH    │
│ Java version change (11 → 17)   │ HIGH    │
│ Backoffice widget changes       │ MEDIUM  │
│ OCC API changes                 │ LOW     │
│ Build system changes            │ LOW     │
│ Third-party library updates     │ MEDIUM  │
└────────────────────────────────┴─────────┘
```

### Step 2: Read the Release Notes

For every version between your current and target, read:

1. **Release notes** — new features, behavior changes
2. **Upgrade guides** — mandatory migration steps
3. **Deprecated and removed API lists** — code that will break
4. **Known issues** — problems SAP has documented

```
Sources:
- help.sap.com/docs/SAP_COMMERCE → Release Information
- SAP Note search for your version range
- CCv2 compatibility matrix
```

### Step 3: Inventory Your Customizations

```bash
# Count custom extensions
find . -name "extensioninfo.xml" -path "*/custom/*" | wc -l

# Find overridden Spring beans
grep -rn "override=\"true\"" --include="*-spring.xml" custom/

# Find overridden JSP/template files
find custom/ -name "*.jsp" -o -name "*.html" | wc -l

# Find deprecated API usage (run against deprecation list)
grep -rn "@Deprecated" --include="*.java" bin/platform/
# Cross-reference with your custom code usage
```

### Step 4: Estimate the Effort

| Customization Level | Patch Upgrade (2211.20→.25) | Minor Upgrade (2205→2211) | Major Skip (2105→2211) |
|--------------------|---------------------------|--------------------------|----------------------|
| Minimal (OOTB + config) | 1-2 days | 1-2 weeks | 3-4 weeks |
| Moderate (20-30 custom extensions) | 2-3 days | 3-4 weeks | 6-8 weeks |
| Heavy (50+ extensions, custom types) | 1 week | 6-8 weeks | 3-4 months |

---

## The Upgrade Process

### Phase 1: Environment Setup

```bash
# 1. Create a dedicated upgrade branch
git checkout -b upgrade/2211.25

# 2. Download the new platform version
# For CCv2: Update manifest.json
# For on-premise: Download from SAP Software Center

# 3. Update manifest.json (CCv2)
```

```json
{
  "commerceSuiteVersion": "2211.25",
  "extensions": [
    // Review each extension — some may be renamed or removed
  ]
}
```

```bash
# 4. Set up a clean database for the target version
# Run initialization to get a baseline schema
ant clean all initialize

# 5. Compare schemas between current and target
# This shows what database changes the upgrade will make
```

### Phase 2: Resolve Compilation Errors

After updating the platform version, your custom code will likely have compilation errors. Address them systematically.

**Java version changes:**

```java
// If upgrading Java 11 → 17:

// REMOVED: Nashorn JavaScript engine
// Replace ScriptEngine usage with GraalVM or alternative

// REMOVED: javax.* packages moved to jakarta.* (if applicable)
// import javax.servlet.http.HttpServletRequest;
// becomes:
// import jakarta.servlet.http.HttpServletRequest;

// REMOVED: Security Manager
// Remove any System.setSecurityManager() calls

// NEW: Sealed classes, records, pattern matching (optional to adopt)
```

**Deprecated API replacements:**

```java
// Example: ServiceLayer API changes between versions

// OLD (removed in 2211):
// ItemModel model = modelService.get(pk);
// modelService.detach(model);

// NEW:
ItemModel model = modelService.get(pk);
modelService.detachAll(); // or use specific detach methods

// OLD: Direct Jalo layer access
// JaloSession.getCurrentSession().getSessionContext()

// NEW: Use ServiceLayer
SessionService sessionService = Registry.getApplicationContext()
    .getBean("sessionService", SessionService.class);
```

**Spring configuration changes:**

```xml
<!-- OLD: Spring beans using removed parent beans -->
<bean id="myService" parent="removedParentBean">
    <property name="oldProperty" ref="oldBean"/>
</bean>

<!-- NEW: Updated parent and property references -->
<bean id="myService" parent="newParentBean">
    <property name="newProperty" ref="newBean"/>
</bean>
```

### Phase 3: Type System Migration

Type system changes require careful handling — they affect the database schema.

```xml
<!-- items.xml changes to review -->

<!-- CHECK: Are any of your custom types extending types that changed? -->
<itemtype code="CustomProduct" extends="Product">
    <!-- Verify all inherited attributes still exist -->
</itemtype>

<!-- CHECK: Enum values that were removed or renamed -->
<enumtype code="ArticleApprovalStatus">
    <!-- Verify all values your code references still exist -->
</enumtype>

<!-- CHECK: Relation changes -->
<relation code="Product2CustomFeatureRelation">
    <!-- Verify source/target types haven't changed -->
</relation>
```

**Running the update:**

```bash
# Dry run — shows what will change without modifying the database
ant updatesystem -DdryRun=true

# Review the output:
# - New tables to create
# - Columns to add
# - Types to register
# - Potential data migration needed

# Execute the update
ant updatesystem
```

### Phase 4: Extension Compatibility

```
For each custom extension:

1. Does it compile against the new platform?
   → Fix compilation errors

2. Does it reference removed platform extensions?
   → Update extensioninfo.xml dependencies

3. Does it override platform Spring beans that changed?
   → Update bean definitions

4. Does it use ImpEx scripts that reference changed types?
   → Update ImpEx headers and references

5. Does it depend on third-party libraries conflicting
   with platform-bundled versions?
   → Align library versions
```

**Common extension compatibility fixes:**

```xml
<!-- extensioninfo.xml — update required extensions -->
<extensioninfo>
    <extension name="mycustomextension">
        <requires-extension name="commerceservices"/>
        <!-- This extension may have been renamed or split -->
        <!-- Check release notes for extension name changes -->
    </extension>
</extensioninfo>
```

### Phase 5: Backoffice Configuration

Backoffice widgets and configurations often break during upgrades:

```xml
<!-- Check custom Backoffice configurations -->
<!-- backoffice-config.xml changes -->

<!-- Verify widget definitions still match platform expectations -->
<context type="Product" component="editor-area" module="platformbackoffice">
    <!-- Custom editor configurations may need updating -->
</context>

<!-- Check for removed widget types -->
<!-- Some widgets are replaced in newer versions -->
```

### Phase 6: Storefront/OCC API Compatibility

```
If using Composable Storefront (Spartacus):

1. Check Spartacus compatibility matrix
   - Spartacus version X requires Commerce version Y

2. Review OCC API changes
   - New required fields
   - Changed response structures
   - Removed endpoints

3. Update Spartacus version if needed
   ng update @spartacus/schematics@latest
```

---

## Testing the Upgrade

### Test Strategy

```
┌─────────────────────────────────────────────────────┐
│ Level 1: Compilation & Startup                       │
│ Platform builds, starts, and initializes             │
├─────────────────────────────────────────────────────┤
│ Level 2: Unit Tests Pass                             │
│ All existing unit tests pass without modification    │
├─────────────────────────────────────────────────────┤
│ Level 3: Integration Tests Pass                      │
│ Services, FlexibleSearch, ImpEx work correctly       │
├─────────────────────────────────────────────────────┤
│ Level 4: Functional Smoke Test                       │
│ Browse catalog, add to cart, checkout, search        │
├─────────────────────────────────────────────────────┤
│ Level 5: Full Regression                             │
│ Complete test suite including edge cases             │
├─────────────────────────────────────────────────────┤
│ Level 6: Performance Baseline                        │
│ Response times match or improve vs. old version      │
├─────────────────────────────────────────────────────┤
│ Level 7: Data Migration Validation                   │
│ Production data snapshot works with new version      │
└─────────────────────────────────────────────────────┘
```

### Data Migration Testing

```bash
# 1. Take a snapshot of production data
#    (anonymized for non-production environments)

# 2. Restore to upgrade test environment

# 3. Run updatesystem against production data
ant updatesystem

# 4. Verify data integrity
# Run validation queries:
```

```sql
-- Check product counts match
SELECT COUNT(*) FROM products;

-- Check no orphaned price rows
SELECT COUNT(*) FROM pricerows pr
LEFT JOIN products p ON pr.p_product = p.pk
WHERE p.pk IS NULL;

-- Check catalog versions are intact
SELECT cv.p_version, COUNT(p.pk)
FROM products p
JOIN catalogversions cv ON p.p_catalogversion = cv.pk
GROUP BY cv.p_version;
```

---

## Deploying the Upgrade

### CCv2 Deployment Strategy

```
Recommended approach: Blue-Green via CCv2

1. Build the new version
   → Cloud Portal → Builds → Create Build

2. Deploy to staging environment
   → Full testing with production-like data

3. Run performance tests on staging
   → Compare with production baselines

4. Schedule production deployment
   → Low-traffic window
   → Team on standby

5. Deploy to production
   → CCv2 handles rolling deployment

6. Monitor for 24 hours
   → Error rates, response times, memory
   → Cache hit rates, CronJob success

7. Keep previous build available for rollback
   → Don't delete until new version is stable
```

### Rollback Plan

```
Rollback triggers:
- Error rate > 2× baseline for 15+ minutes
- P95 response time > 3× baseline
- Critical business flow broken (checkout, search)
- Data corruption detected

Rollback steps:
1. Deploy previous build from CCv2 Cloud Portal
2. If DB schema changed: Assess if rollback build
   is compatible with modified schema
3. If incompatible: Restore from pre-upgrade DB backup
4. Communicate timeline to stakeholders

Important: Database schema changes (new columns, tables)
are forward-only. The old build must tolerate new columns
it doesn't use. Removing columns requires the old version
to not reference them.
```

---

## Version-Specific Migration Notes

### Upgrading to 2211 (from 2205 or earlier)

Key changes:
- **Java 17 required** — update build tools, CI pipelines, Docker images
- **Spring 5.3.x** — review custom Spring configurations
- **Solr 9.x** — solrconfig.xml may need updates
- **Removed legacy accelerator templates** — migrate to Composable Storefront
- **Improved OCC API** — some endpoint signatures changed

### Upgrading to 2205 (from 2105 or earlier)

Key changes:
- **Java 11 minimum** — if coming from Java 8
- **Ant build changes** — review custom build callbacks
- **Backoffice redesign** — custom widget configurations may break
- **CronJob framework updates** — verify custom CronJob implementations

### Patch Upgrades (e.g., 2211.20 → 2211.25)

Lower risk but still requires testing:
- No type system changes (usually)
- Bug fixes and security patches
- Possible behavior changes in edge cases
- Always test with production data snapshot

---

## Best Practices

1. **Never skip reading the upgrade guide** — SAP provides version-specific instructions for a reason. Skipping them leads to missed mandatory steps.

2. **Upgrade incrementally** — if you're on 2011, don't jump to 2211. Go 2011→2105→2205→2211, validating at each step. Some teams prefer skipping to the latest directly, but incremental upgrades make debugging easier.

3. **Automate your test suite** — manual regression testing for an upgrade is slow and error-prone. Invest in automated tests before starting the upgrade.

4. **Test with production data** — synthetic test data misses edge cases. Anonymized production data reveals issues with real product catalogs, customer data volumes, and order history.

5. **Keep customizations minimal** — every override, every copied platform class, every Backoffice widget customization is a potential upgrade blocker. Prefer extension points over overrides.

6. **Maintain a dependency inventory** — know exactly which platform APIs you use, which beans you override, and which types you extend. This inventory makes impact analysis faster.

7. **Plan for rollback** — assume the upgrade will need to be rolled back at least once during testing. Design the deployment so rollback is a one-click operation.

8. **Communicate with stakeholders** — upgrades require freeze periods, testing windows, and deployment coordination. Set expectations early about timelines and potential disruptions.

---

## Summary

Upgrading SAP Commerce is a project, not a task. It requires:

1. **Analysis** — understand what changed between versions and how it affects your customizations
2. **Code migration** — fix compilation errors, replace deprecated APIs, update configurations
3. **Type system validation** — ensure database schema changes don't break existing data
4. **Comprehensive testing** — from unit tests through full regression with production data
5. **Careful deployment** — staged rollout with monitoring and rollback capability

The teams that upgrade regularly (every 6-12 months) spend less total effort than those who defer upgrades for years. Each version jump is smaller, the deprecated API list is shorter, and the team maintains familiarity with the upgrade process. Treat upgrades as routine maintenance, not a crisis — and they'll stay manageable.