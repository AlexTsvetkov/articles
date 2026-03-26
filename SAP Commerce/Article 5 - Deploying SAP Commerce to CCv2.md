# Deploying SAP Commerce to Commerce Cloud v2 (CCv2): The Complete Guide

Commerce Cloud v2 (CCv2) is SAP's managed cloud platform for hosting SAP Commerce. It abstracts away infrastructure provisioning, gives you a CI/CD pipeline, and runs your code on a Kubernetes-based architecture backed by Microsoft Azure. But "managed" doesn't mean "magic" — you still need to understand the deployment model, configure your build correctly, structure your `manifest.json`, and know how to troubleshoot when things go wrong.

This guide covers everything from your first deployment to production release patterns, with practical configuration examples and lessons learned from real CCv2 projects.

---

## CCv2 Architecture Overview

CCv2 runs on Azure Kubernetes Service (AKS). Your SAP Commerce application is containerized and orchestrated by Kubernetes, but SAP abstracts most of the K8s complexity away from you.

### Key Components

```
┌──────────────────────────────────────────────────────┐
│  CCv2 Environment (Dev / Staging / Production)        │
│                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ Storefront   │  │ Backoffice  │  │ Background    │ │
│  │ Aspect       │  │ Aspect      │  │ Processing    │ │
│  │ (OCC API)    │  │ (hAC/BO)    │  │ Aspect        │ │
│  │ Replicas: 2+ │  │ Replicas: 1 │  │ Replicas: 1   │ │
│  └──────┬───────┘  └──────┬──────┘  └──────┬────────┘ │
│         │                  │                │          │
│  ┌──────┴──────────────────┴────────────────┴───────┐ │
│  │              Azure Database (HANA)                │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │ Solr Cloud  │  │ Azure Blob  │  │ Azure Front  │ │
│  │ (Search)    │  │ (Media)     │  │ Door (CDN)   │ │
│  └─────────────┘  └─────────────┘  └──────────────┘ │
└──────────────────────────────────────────────────────┘
```

**Aspects** are logical server groupings. Each aspect is a set of pods running the same Commerce application but configured for a specific purpose:

- **accstorefront / api**: Handles customer-facing traffic (OCC API, Accelerator storefront)
- **backoffice**: Runs Backoffice, HAC, and admin interfaces
- **backgroundProcessing**: Executes CronJobs, imports, catalog synchronization

**Why aspects matter**: They let you isolate workloads. A heavy catalog sync on the backgroundProcessing aspect won't impact storefront response times.

### Environments

CCv2 provides three standard environments:

| Environment | Purpose | Data | Access |
|-------------|---------|------|--------|
| Development (d1) | Development and testing | Test/sample data | Developers |
| Staging (s1) | Pre-production validation | Production-like data | QA + limited team |
| Production (p1) | Live commerce | Real customer data | Public |

Each environment has its own database, Solr instance, and media storage.

---

## The manifest.json File

The `manifest.json` is the central configuration file for CCv2 builds. It lives in the root of your repository and controls every aspect of the build and deployment.

### Minimal Working Example

```json
{
  "commerceSuiteVersion": "2211.28",
  "useCloudExtensionPack": true,
  "extensions": [
    "modeltacceleratorservices",
    "ycommercewebservices",
    "backoffice",
    "hac",
    "solrserver",
    "cloudhotfolder",
    "azurecloudhotfolder"
  ],
  "storefrontAddons": [
    {
      "addon": "smarteditaddon",
      "storefront": "yacceleratorstorefront",
      "template": "yacceleratorstorefront"
    }
  ],
  "aspects": [
    {
      "name": "backoffice",
      "properties": [
        {
          "key": "cluster.node.groups",
          "value": "integration,yHotfolderCandidate,backgroundProcessing"
        }
      ],
      "webapps": [
        {
          "name": "hac",
          "contextPath": "/hac"
        },
        {
          "name": "backoffice",
          "contextPath": ""
        }
      ]
    },
    {
      "name": "accstorefront",
      "properties": [
        {
          "key": "cluster.node.groups",
          "value": "storefrontRestApi"
        }
      ],
      "webapps": [
        {
          "name": "ycommercewebservices",
          "contextPath": "/occ"
        },
        {
          "name": "hac",
          "contextPath": "/hac"
        }
      ]
    },
    {
      "name": "backgroundProcessing",
      "properties": [
        {
          "key": "cluster.node.groups",
          "value": "integration,yHotfolderCandidate,backgroundProcessing"
        },
        {
          "key": "cronjob.timertask.loadonstartup",
          "value": "true"
        }
      ],
      "webapps": [
        {
          "name": "hac",
          "contextPath": "/hac"
        }
      ]
    }
  ],
  "properties": [
    {
      "key": "modeltacceleratorservices.check.uncategorized.products",
      "value": "false"
    }
  ],
  "enableImageProcessingService": true,
  "solrVersion": "8.11.2",
  "useConfig": {
    "extensions": {
      "location": "core-customize/hybris/config/localextensions.xml"
    },
    "properties": [
      {
        "location": "core-customize/hybris/config/local.properties"
      }
    ],
    "solr": {
      "location": "core-customize/solr"
    }
  }
}
```

### Key Fields Explained

**`commerceSuiteVersion`**: The SAP Commerce version. This determines which base platform image is used. Format: `YYYY.PATCH` (e.g., `2211.28`).

**`extensions`**: List of extensions to include in the build. Only listed extensions (and their dependencies) are compiled and deployed. Keep this list minimal — fewer extensions = faster builds and smaller images.

**`aspects`**: The server configurations. Each aspect defines which webapps to deploy, which properties to apply, and which cluster groups to join.

**`properties`**: Global properties applied to all aspects. Aspect-level properties override global ones.

**`useConfig`**: Points to configuration files in your repository. This is how you include `localextensions.xml`, `local.properties`, and custom Solr configurations.

### Properties Hierarchy

Properties are resolved in this order (later overrides earlier):

1. Platform defaults (`project.properties` in each extension)
2. `manifest.json` global `properties`
3. `manifest.json` aspect-level `properties`
4. `useConfig.properties` files
5. Environment-specific properties (set via Cloud Portal)

---

## Repository Structure

CCv2 expects a specific repository layout:

```
my-commerce-project/
├── manifest.json                          # Build configuration
├── core-customize/
│   └── hybris/
│       ├── config/
│       │   ├── localextensions.xml        # Extension list
│       │   ├── local.properties           # Global properties
│       │   ├── local-dev.properties       # Dev overrides
│       │   ├── local-stag.properties      # Staging overrides
│       │   └── local-prod.properties      # Production overrides
│       └── bin/
│           └── custom/
│               ├── mycore/                # Core extension
│               │   ├── src/
│               │   ├── resources/
│               │   ├── testsrc/
│               │   ├── web/
│               │   ├── buildcallbacks.xml
│               │   └── extensioninfo.xml
│               ├── myfacades/             # Facades extension
│               ├── mystorefront/          # Storefront extension
│               └── myinitialdata/         # Initial data extension
├── js-storefront/                         # Spartacus/composable storefront
│   ├── package.json
│   ├── angular.json
│   └── src/
└── solr/                                  # Custom Solr config (optional)
    └── server/
        └── solr/
            └── configsets/
```

### The `core-customize` Directory

This is where your custom Commerce code lives. CCv2 builds look for the `hybris` directory inside `core-customize/`.

**`localextensions.xml`** — Lists all extensions to be loaded:

```xml
<hybrisconfig xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="resources/schemas/extensions.xsd">
    <extensions>
        <!-- Platform extensions -->
        <extension name="backoffice"/>
        <extension name="hac"/>
        <extension name="ycommercewebservices"/>
        
        <!-- Custom extensions -->
        <extension dir="${HYBRIS_BIN_DIR}/custom/mycore"/>
        <extension dir="${HYBRIS_BIN_DIR}/custom/myfacades"/>
        <extension dir="${HYBRIS_BIN_DIR}/custom/mystorefront"/>
        <extension dir="${HYBRIS_BIN_DIR}/custom/myinitialdata"/>
    </extensions>
</hybrisconfig>
```

---

## Build Process

### How CCv2 Builds Work

When you trigger a build in the Cloud Portal (or via the CCv2 API), the following happens:

```
1. Source code is pulled from the Git repository (specified branch/commit)
2. manifest.json is parsed
3. Commerce Suite image (specified version) is pulled
4. Custom extensions are compiled (ant build)
5. Docker image is created with compiled code + platform
6. Image is pushed to the Azure Container Registry
7. Build artifacts (logs, metadata) are stored
8. Build is marked as AVAILABLE for deployment
```

### Triggering Builds

**Via Cloud Portal**: Navigate to Builds → Create Build → Select branch.

**Via CCv2 API**:

```bash
curl -X POST "https://portalrotapi.hana.ondemand.com/v2/builds" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "branch": "develop",
    "name": "Sprint-42-Build",
    "subscriptionCode": "your-subscription-code"
  }'
```

### Build Optimization Tips

**1. Minimize the extension list.** Every extension adds compile time and image size:

```json
// BAD: Including everything
"extensions": ["adaptivesearch", "adaptivesearchbackoffice", "adaptivesearchfacades", 
               "adaptivesearchsolr", "addonsupport", "apiregistrybackoffice", ...]

// GOOD: Only what you actually use
"extensions": ["ycommercewebservices", "backoffice", "hac", "solrserver",
               "cloudhotfolder", "azurecloudhotfolder", "mycore", "myfacades"]
```

**2. Use `.ccv2ignore` to exclude files from the build:**

```
# .ccv2ignore
core-customize/hybris/bin/custom/*/testsrc/**
core-customize/hybris/bin/custom/*/.git/**
*.md
docs/
```

**3. Cache dependencies.** If you use Gradle or Maven for custom builds, configure caching in the build callbacks.

---

## Deployment Process

### Deploying a Build

Deployment assigns a build to an environment and starts the rollout:

**Via Cloud Portal**: Navigate to Environments → Select environment → Deploy → Choose build.

**Via API**:

```bash
curl -X POST "https://portalrotapi.hana.ondemand.com/v2/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "buildCode": "20250315.1",
    "environmentCode": "d1",
    "databaseUpdateMode": "UPDATE",
    "strategy": "ROLLING_UPDATE"
  }'
```

### Database Update Modes

| Mode | Behavior | When to Use |
|------|----------|-------------|
| `NONE` | No database changes | Hotfix with no type system changes |
| `UPDATE` | Runs `ant updatesystem` | Most deployments (safe, additive) |
| `INITIALIZE` | Runs `ant initialize` (drops and recreates) | First deployment, major restructuring |

**WARNING**: `INITIALIZE` destroys all data. Never use it on production unless you have a full data migration plan.

### Deployment Strategies

**Rolling Update** (default): Pods are replaced one at a time. Zero-downtime deployment if your application handles graceful shutdown.

**Recreate**: All pods are stopped, then new pods are started. Causes downtime but ensures clean state.

For production, always use **Rolling Update** with proper health checks:

```json
{
  "aspects": [
    {
      "name": "accstorefront",
      "properties": [
        {
          "key": "tomcat.graceful.shutdown.timeout",
          "value": "30"
        }
      ]
    }
  ]
}
```

---

## Environment-Specific Configuration

### Managing Properties Across Environments

Use the Cloud Portal to set environment-specific properties that shouldn't be in source control:

- **API keys and secrets**: Payment gateway credentials, third-party API keys
- **Endpoint URLs**: Backend system URLs that differ per environment
- **Feature flags**: Enable/disable features per environment

In the Cloud Portal: Environments → Select environment → Configuration → Service Properties.

### Property Files Strategy

For properties that can be in source control, use environment-specific files:

```
core-customize/hybris/config/
├── local.properties              # Shared across all environments
├── local-dev.properties          # Development overrides
├── local-stag.properties         # Staging overrides
└── local-prod.properties         # Production overrides
```

Reference them in `manifest.json`:

```json
{
  "useConfig": {
    "properties": [
      {
        "location": "core-customize/hybris/config/local.properties"
      },
      {
        "location": "core-customize/hybris/config/local-dev.properties",
        "aspect": "accstorefront",
        "persona": "development"
      },
      {
        "location": "core-customize/hybris/config/local-prod.properties",
        "aspect": "accstorefront",
        "persona": "production"
      }
    ]
  }
}
```

**Personas** map to environments:
- `development` → d1
- `staging` → s1
- `production` → p1

---

## Data Initialization and Updates

### First Deployment (Initialize)

Your first deployment to an environment must initialize the database. This typically involves:

1. **Type system creation** — tables generated from `items.xml`
2. **Essential data** — system-level configuration (currencies, languages, user groups)
3. **Project data** — catalogs, CMS pages, Solr configuration

Structure your initial data loading:

```java
@SystemSetup(extension = "myinitialdata")
public class MyInitialDataSetup extends AbstractSystemSetup {
    
    @SystemSetup(type = Type.PROJECT, process = Process.ALL)
    public void createProjectData(final SystemSetupContext context) {
        // Order matters — dependencies first
        importImpexFile(context, "/myinitialdata/import/coredata/stores/mystore/store.impex");
        importImpexFile(context, "/myinitialdata/import/coredata/stores/mystore/site.impex");
        importImpexFile(context, "/myinitialdata/import/sampledata/catalogs/products.impex");
        importImpexFile(context, "/myinitialdata/import/sampledata/catalogs/prices.impex");
        importImpexFile(context, "/myinitialdata/import/sampledata/cms/pages.impex");
    }
}
```

### Subsequent Deployments (Update)

Most deployments use `UPDATE` mode, which:

1. Applies type system changes (new types, new attributes)
2. Runs essential data imports (files matching `essentialdata-*.impex`)
3. Runs project data imports (files matching `projectdata-*.impex`) — only on `ant initialize`, NOT on `ant updatesystem`

**Critical**: Project data files only run during initialization, not during updates. For data that needs to change on every deployment, use essential data or custom update scripts.

### Safe Schema Migration Patterns

**Adding a new attribute**: Safe. The column is added to the table. No data loss.

```xml
<!-- items.xml -->
<itemtype code="Product" autocreate="false" generate="false">
    <attributes>
        <attribute qualifier="loyaltyPoints" type="java.lang.Integer">
            <persistence type="property"/>
        </attribute>
    </attributes>
</itemtype>
```

**Adding a new type**: Safe. A new table is created.

**Removing an attribute**: SAP Commerce doesn't delete columns on update. The column remains in the database but is no longer mapped. Clean up via manual SQL if needed.

**Changing attribute type**: Dangerous. Can cause data loss. Create a new attribute, migrate data, then deprecate the old one.

---

## Monitoring and Logging

### Accessing Logs

CCv2 provides log access through the Cloud Portal and through Kibana (ELK stack):

**Cloud Portal**: Environments → Select environment → Logging → Kibana

**Log types available**:
- Application logs (your `LOG.info()` statements)
- Tomcat access logs
- GC logs
- Platform logs (system events, CronJob execution)

### Kibana Query Examples

```
# Find errors in the last hour
level:ERROR AND @timestamp:[now-1h TO now]

# Find specific exception
message:"OutOfMemoryError"

# Filter by aspect
kubernetes.labels.aspect:accstorefront AND level:ERROR

# CronJob execution logs
message:"CronJob" AND message:"finished"
```

### Health Checks

Configure health endpoints for load balancer monitoring:

```properties
# Health check endpoint
tomcat.healthcheck.path=/healthcheck
tomcat.healthcheck.response=OK
```

The load balancer uses health checks to route traffic only to healthy pods. If a pod fails health checks, Kubernetes automatically restarts it.

---

## Hot Folders in CCv2

Hot Folders allow automated file-based data imports. In CCv2, they integrate with Azure Blob Storage.

### Configuration

```properties
# Enable cloud hot folders
cluster.node.groups=yHotfolderCandidate

# Azure Blob Storage configuration (set via Cloud Portal, not properties)
# azure.hotfolder.storage.account.connection-string=...
# azure.hotfolder.storage.container.name=...
```

### Import Flow

```
1. External system uploads CSV/ImpEx to Azure Blob Storage
2. Cloud Hot Folder service detects the new file
3. File is converted to ImpEx (if CSV) using mapping configuration
4. ImpEx is imported into the platform
5. File is moved to archive/error folder based on result
```

### Mapping Configuration

For CSV imports, define the mapping:

```impex
INSERT_UPDATE HotFolderMapping;code[unique=true];converter;impexHeader
;productImport;csvToImpExConverter;"INSERT_UPDATE Product;code[unique=true];name[lang=en];description[lang=en];catalogVersion(catalog(id),version)[default=myProductCatalog:Staged]"
```

---

## Troubleshooting Common CCv2 Issues

### Build Failures

**"Extension not found"**: The extension listed in `manifest.json` or `localextensions.xml` doesn't exist or isn't in the classpath.

**Fix**: Verify extension names match exactly. Check that custom extensions are in `core-customize/hybris/bin/custom/`.

**"Compilation error"**: Java code doesn't compile.

**Fix**: Build locally first (`ant clean all`). Ensure your code compiles against the same Commerce version specified in `manifest.json`.

### Deployment Failures

**"Database update failed"**: Type system changes conflict with existing data.

**Fix**: Check the deployment logs in Kibana. Common causes:
- Adding a `NOT NULL` attribute without a default value
- Changing the type of an existing attribute
- Circular type dependencies

**"Pod CrashLoopBackOff"**: The application starts but immediately crashes.

**Fix**: Check Kibana logs for the crash reason. Common causes:
- Out of memory (increase pod resources)
- Database connection failure (check connection settings)
- Missing required properties

### Performance Issues After Deployment

**Slow first requests**: Cache is cold after deployment. Expected behavior — caches warm up under load.

**Mitigation**: Implement cache warming scripts that run after deployment:

```java
@SystemSetup(type = Type.PROJECT, process = Process.UPDATE)
public void warmCaches(final SystemSetupContext context) {
    // Pre-load common data into cache
    catalogVersionService.getCatalogVersion("myProductCatalog", "Online");
    commonI18NService.getAllCurrencies();
    commonI18NService.getAllLanguages();
    // Pre-search common queries
    productSearchFacade.textSearch("", PageableData.DEFAULT);
}
```

---

## CI/CD Pipeline Integration

### Automating Builds via CCv2 API

The CCv2 API allows full automation:

```bash
#!/bin/bash
# build-and-deploy.sh

SUBSCRIPTION="your-subscription"
TOKEN=$(get-oauth-token)

# 1. Trigger build
BUILD_RESPONSE=$(curl -s -X POST \
  "https://portalrotapi.hana.ondemand.com/v2/subscriptions/$SUBSCRIPTION/builds" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"branch\": \"$BRANCH\", \"name\": \"CI-Build-$BUILD_NUMBER\"}")

BUILD_CODE=$(echo $BUILD_RESPONSE | jq -r '.code')

# 2. Wait for build to complete
while true; do
  STATUS=$(curl -s \
    "https://portalrotapi.hana.ondemand.com/v2/subscriptions/$SUBSCRIPTION/builds/$BUILD_CODE" \
    -H "Authorization: Bearer $TOKEN" | jq -r '.status')
  
  if [ "$STATUS" = "SUCCESS" ]; then break; fi
  if [ "$STATUS" = "FAIL" ]; then echo "Build failed!"; exit 1; fi
  sleep 30
done

# 3. Deploy to staging
curl -X POST \
  "https://portalrotapi.hana.ondemand.com/v2/subscriptions/$SUBSCRIPTION/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"buildCode\": \"$BUILD_CODE\",
    \"environmentCode\": \"s1\",
    \"databaseUpdateMode\": \"UPDATE\",
    \"strategy\": \"ROLLING_UPDATE\"
  }"
```

### Recommended Pipeline Stages

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Commit   │───>│  Build   │───>│  Deploy  │───>│  Test    │───>│  Deploy  │
│  & Push   │    │  (CCv2)  │    │  to Dev  │    │  (Auto)  │    │  to Stag │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                                      │
                                                                      ▼
                                                               ┌──────────┐
                                                               │  Deploy  │
                                                               │  to Prod │
                                                               │ (Manual) │
                                                               └──────────┘
```

- **Dev deployment**: Automatic on every merge to `develop` branch
- **Staging deployment**: Automatic after dev tests pass
- **Production deployment**: Manual approval gate

---

## Production Deployment Best Practices

### Pre-Deployment Checklist

- [ ] Build tested on staging environment with production-like data
- [ ] Database update mode verified (UPDATE vs. NONE)
- [ ] No `INITIALIZE` mode on production (double-check!)
- [ ] Rollback plan documented (previous build code noted)
- [ ] Monitoring dashboards open during deployment
- [ ] Team notified of deployment window

### Zero-Downtime Deployment

For true zero-downtime deployments:

1. **Use rolling update strategy** — pods are replaced one at a time
2. **Ensure backward-compatible database changes** — new code must work with old schema during rollout
3. **Avoid breaking API changes** — add new endpoints before removing old ones
4. **Configure graceful shutdown** — let in-flight requests complete before pod termination

```json
{
  "aspects": [
    {
      "name": "accstorefront",
      "properties": [
        { "key": "tomcat.graceful.shutdown.timeout", "value": "30" }
      ]
    }
  ]
}
```

### Rollback

If a deployment causes issues, rollback by deploying the previous build:

```bash
curl -X POST \
  "https://portalrotapi.hana.ondemand.com/v2/subscriptions/$SUBSCRIPTION/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "buildCode": "PREVIOUS_BUILD_CODE",
    "environmentCode": "p1",
    "databaseUpdateMode": "NONE",
    "strategy": "ROLLING_UPDATE"
  }'
```

**Note**: Database rollback is not automatic. If the failed deployment included schema changes, you may need to handle data migration manually.

---

## Summary

Deploying to CCv2 is straightforward once you understand the model. The key principles:

1. **`manifest.json` is your deployment contract** — it defines everything about your build, aspects, and configuration
2. **Aspects separate concerns** — storefront, backoffice, and background processing should have independent configurations
3. **Use UPDATE mode for most deployments** — INITIALIZE destroys data and should only be used for initial setup
4. **Environment-specific properties belong in the Cloud Portal** — secrets, API keys, and environment URLs should never be in source control
5. **Automate your pipeline** — use the CCv2 API to build CI/CD pipelines that reduce human error
6. **Plan for rollback** — always know your previous build code and have a rollback procedure documented
7. **Monitor during deployment** — watch Kibana logs and performance metrics during and after every deployment

CCv2 handles the infrastructure complexity, but deployment success still depends on how well you structure your code, configuration, and processes.