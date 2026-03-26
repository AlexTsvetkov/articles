# SAP Commerce Composable Storefront (Spartacus) Deep Dive: Architecture, Customization, and Production Patterns

SAP Commerce Composable Storefront — still widely known as Spartacus — is the Angular-based frontend framework for SAP Commerce Cloud. It replaces the Accelerator server-side storefronts with a decoupled, JavaScript-driven single-page application that communicates with the backend entirely through the OCC REST API. This architecture enables independent frontend deployments, CDN-friendly content delivery, and a modern development experience.

This article covers the architecture, customization model, state management, CMS integration, and production deployment patterns that every Spartacus developer needs to understand.

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────┐
│  Browser                                               │
│  ┌─────────────────────────────────────────────────┐  │
│  │  Spartacus Application (Angular SPA)             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐│  │
│  │  │ NgRx     │ │ CMS      │ │ Feature Modules  ││  │
│  │  │ Store    │ │ Mapping  │ │ (Cart, Checkout, ││  │
│  │  │          │ │ Engine   │ │  PDP, PLP, etc.) ││  │
│  │  └──────────┘ └──────────┘ └──────────────────┘│  │
│  │  ┌──────────────────────────────────────────────┐│  │
│  │  │  OCC Adapter Layer (HTTP → Backend)          ││  │
│  │  └──────────────────────────────────────────────┘│  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
          │ REST API calls (OCC v2)
          ▼
┌───────────────────────────────────────────────────────┐
│  SAP Commerce Cloud Backend                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │
│  │ OCC API  │ │ CMS      │ │ Commerce Services    │  │
│  │ Layer    │ │ Engine   │ │ (Cart, Pricing,      │  │
│  │          │ │          │ │  Checkout, Search)   │  │
│  └──────────┘ └──────────┘ └──────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **Headless**: The frontend knows nothing about the backend's Java code or database. All communication happens via HTTP/JSON through the OCC API.

2. **CMS-Driven Layout**: Page layout, component placement, and content are managed in the SAP Commerce CMS (SmartEdit). The frontend receives a page structure from the CMS API and dynamically renders Angular components.

3. **Feature Libraries**: Spartacus is modular. Features like cart, checkout, product display, and user management are packaged as separate Angular libraries (`@spartacus/cart`, `@spartacus/checkout`, etc.).

4. **NgRx State Management**: Global state (user session, cart, product data) is managed through NgRx stores with actions, effects, and reducers.

5. **Configurability Over Code**: Many behaviors can be changed through configuration (TypeScript objects) rather than writing new code.

---

## Project Setup

### Creating a Spartacus Project

```bash
# Create a new Angular workspace
ng new mystore --style=scss --routing=true

cd mystore

# Add Spartacus schematics
ng add @spartacus/schematics \
  --baseUrl=https://my-commerce-backend.com \
  --baseSite=electronics-spa \
  --occPrefix=/occ/v2 \
  --features=Checkout,Cart,User,Product,Navigation,SmartEdit,ASM
```

This scaffolds:

```
mystore/
├── src/
│   ├── app/
│   │   ├── app.module.ts
│   │   ├── app.component.ts
│   │   └── spartacus/
│   │       ├── spartacus-features.module.ts
│   │       ├── spartacus-configuration.module.ts
│   │       └── spartacus.module.ts
│   ├── styles.scss
│   └── index.html
├── angular.json
├── package.json
└── tsconfig.json
```

### Configuration Module

```typescript
// spartacus-configuration.module.ts
@NgModule({
  providers: [
    provideConfig(<OccConfig>{
      backend: {
        occ: {
          baseUrl: 'https://my-commerce-backend.com',
          prefix: '/occ/v2/',
        },
      },
    }),
    provideConfig(<SiteContextConfig>{
      context: {
        baseSite: ['electronics-spa'],
        language: ['en', 'de'],
        currency: ['USD', 'EUR'],
      },
    }),
    provideConfig(<RoutingConfig>{
      routing: {
        routes: {
          product: {
            paths: ['product/:productCode/:name'],
          },
        },
      },
    }),
  ],
})
export class SpartacusConfigurationModule {}
```

---

## CMS-Driven Page Rendering

This is the most important concept in Spartacus. Pages are not hardcoded Angular routes with static templates. Instead:

1. A user navigates to `/product/123/camera`
2. Spartacus calls the CMS API to get the page structure for this route
3. The backend returns a page definition with **slots** and **components**
4. Spartacus maps each CMS component type to an Angular component
5. Components are rendered dynamically in the appropriate slots

### Page Structure from CMS

The OCC CMS API returns:

```json
{
  "uid": "productDetailPage",
  "template": "ProductDetailsPageTemplate",
  "contentSlots": {
    "contentSlot": [
      {
        "slotId": "ProductSummarySlot",
        "position": "Summary",
        "components": {
          "component": [
            { "uid": "ProductImagesComponent", "typeCode": "CMSFlexComponent" },
            { "uid": "ProductSummaryComponent", "typeCode": "CMSFlexComponent" },
            { "uid": "ProductAddToCartComponent", "typeCode": "CMSFlexComponent" }
          ]
        }
      },
      {
        "slotId": "ProductTabsSlot",
        "position": "Tabs",
        "components": {
          "component": [
            { "uid": "ProductDetailsTabComponent", "typeCode": "CMSTabParagraphContainer" },
            { "uid": "ProductReviewsTabComponent", "typeCode": "CMSFlexComponent" }
          ]
        }
      }
    ]
  }
}
```

### CMS Component Mapping

Spartacus maps CMS component types to Angular components:

```typescript
provideConfig(<CmsConfig>{
  cmsComponents: {
    ProductImagesComponent: {
      component: ProductImagesComponent,
    },
    ProductSummaryComponent: {
      component: ProductSummaryComponent,
    },
    ProductAddToCartComponent: {
      component: AddToCartComponent,
    },
    // Custom component mapping
    LoyaltyPointsDisplayComponent: {
      component: LoyaltyPointsComponent,
      providers: [
        {
          provide: LoyaltyService,
          useClass: LoyaltyService,
        },
      ],
    },
  },
})
```

### How Slots Render

In templates, the `<cx-page-slot>` directive renders all components assigned to a slot:

```html
<!-- Page layout template -->
<div class="product-detail">
  <div class="summary-section">
    <cx-page-slot position="Summary"></cx-page-slot>
  </div>
  <div class="tabs-section">
    <cx-page-slot position="Tabs"></cx-page-slot>
  </div>
  <div class="recommendations">
    <cx-page-slot position="CrossSelling"></cx-page-slot>
  </div>
</div>
```

The content of each slot is entirely determined by the CMS configuration in the backend. Business users can add, remove, or reorder components through SmartEdit without any frontend deployment.

---

## Customizing Components

### Replacing a Standard Component

To replace an out-of-the-box component with your custom implementation:

```typescript
@Component({
  selector: 'app-custom-add-to-cart',
  template: `
    <div class="custom-add-to-cart">
      <div class="quantity-selector">
        <button (click)="decrement()">-</button>
        <input type="number" [value]="quantity" (change)="onQuantityChange($event)"/>
        <button (click)="increment()">+</button>
      </div>
      <button 
        class="btn btn-primary" 
        (click)="addToCart()" 
        [disabled]="!product?.stock?.stockLevelStatus || product.stock.stockLevelStatus === 'outOfStock'">
        <span *ngIf="product?.stock?.stockLevelStatus === 'outOfStock'">Out of Stock</span>
        <span *ngIf="product?.stock?.stockLevelStatus !== 'outOfStock'">
          Add to Cart — {{ product?.price?.formattedValue }}
        </span>
      </button>
      <app-loyalty-points-preview 
        *ngIf="loyaltyPoints > 0"
        [points]="loyaltyPoints">
      </app-loyalty-points-preview>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class CustomAddToCartComponent extends AddToCartComponent {
  loyaltyPoints = 0;

  constructor(
    // Inject all parent dependencies
    protected currentProductService: CurrentProductService,
    protected activeCartFacade: ActiveCartFacade,
    private loyaltyService: LoyaltyService
  ) {
    super(currentProductService, activeCartFacade);
  }

  ngOnInit() {
    super.ngOnInit();
    this.currentProductService.getProduct().subscribe(product => {
      if (product?.price?.value) {
        this.loyaltyPoints = Math.floor(product.price.value);
      }
    });
  }
}
```

Register the override:

```typescript
provideConfig(<CmsConfig>{
  cmsComponents: {
    ProductAddToCartComponent: {
      component: CustomAddToCartComponent,
    },
  },
})
```

### Creating New CMS Components

For components that don't exist in the standard library:

**1. Define the CMS component in SAP Commerce (ImpEx):**

```impex
INSERT_UPDATE CMSFlexComponent;uid[unique=true];name;flexType;$catalogVersion
;LoyaltyDashboardComponent;Loyalty Dashboard;LoyaltyDashboardComponent;
```

**2. Create the Angular component:**

```typescript
@Component({
  selector: 'app-loyalty-dashboard',
  template: `
    <div class="loyalty-dashboard" *ngIf="account$ | async as account">
      <div class="tier-badge" [ngClass]="account.tier | lowercase">
        {{ account.tier }}
      </div>
      <div class="points-display">
        <span class="points-value">{{ account.points | number }}</span>
        <span class="points-label">Points</span>
      </div>
      <div class="progress-bar">
        <div class="progress" [style.width.%]="getProgress(account)"></div>
        <span class="next-tier">{{ account.pointsToNextTier | number }} points to next tier</span>
      </div>
      <div class="recent-transactions">
        <h3>Recent Activity</h3>
        <div *ngFor="let tx of account.recentTransactions" class="transaction">
          <span class="tx-date">{{ tx.date | date }}</span>
          <span class="tx-desc">{{ tx.description }}</span>
          <span class="tx-points" [class.positive]="tx.points > 0">
            {{ tx.points > 0 ? '+' : '' }}{{ tx.points }}
          </span>
        </div>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoyaltyDashboardComponent implements OnInit {
  account$: Observable<LoyaltyAccount>;

  constructor(private loyaltyService: LoyaltyService) {}

  ngOnInit() {
    this.account$ = this.loyaltyService.getAccount();
  }

  getProgress(account: LoyaltyAccount): number {
    const tierThresholds = { BRONZE: 1000, SILVER: 5000, GOLD: 20000, PLATINUM: 100000 };
    const nextThreshold = tierThresholds[account.tier] || 100000;
    return Math.min(100, (account.points / nextThreshold) * 100);
  }
}
```

**3. Register the CMS mapping:**

```typescript
provideConfig(<CmsConfig>{
  cmsComponents: {
    LoyaltyDashboardComponent: {
      component: LoyaltyDashboardComponent,
    },
  },
})
```

Now business users can place this component on any page via SmartEdit.

---

## State Management with NgRx

Spartacus uses NgRx for state management. Understanding the pattern is essential for custom features.

### Custom State: Loyalty Module

**Actions:**

```typescript
// loyalty.actions.ts
export const LOAD_LOYALTY_ACCOUNT = '[Loyalty] Load Account';
export const LOAD_LOYALTY_ACCOUNT_SUCCESS = '[Loyalty] Load Account Success';
export const LOAD_LOYALTY_ACCOUNT_FAIL = '[Loyalty] Load Account Fail';

export class LoadLoyaltyAccount implements Action {
  readonly type = LOAD_LOYALTY_ACCOUNT;
}

export class LoadLoyaltyAccountSuccess implements Action {
  readonly type = LOAD_LOYALTY_ACCOUNT_SUCCESS;
  constructor(public payload: LoyaltyAccount) {}
}

export class LoadLoyaltyAccountFail implements Action {
  readonly type = LOAD_LOYALTY_ACCOUNT_FAIL;
  constructor(public payload: any) {}
}
```

**Reducer:**

```typescript
// loyalty.reducer.ts
export interface LoyaltyState {
  account: LoyaltyAccount | null;
  loading: boolean;
  error: any;
}

const initialState: LoyaltyState = {
  account: null,
  loading: false,
  error: null,
};

export function loyaltyReducer(state = initialState, action: LoyaltyActions): LoyaltyState {
  switch (action.type) {
    case LOAD_LOYALTY_ACCOUNT:
      return { ...state, loading: true, error: null };
    case LOAD_LOYALTY_ACCOUNT_SUCCESS:
      return { ...state, account: action.payload, loading: false };
    case LOAD_LOYALTY_ACCOUNT_FAIL:
      return { ...state, error: action.payload, loading: false };
    default:
      return state;
  }
}
```

**Effects:**

```typescript
// loyalty.effects.ts
@Injectable()
export class LoyaltyEffects {
  
  loadAccount$ = createEffect(() =>
    this.actions$.pipe(
      ofType(LOAD_LOYALTY_ACCOUNT),
      switchMap(() =>
        this.loyaltyConnector.getAccount().pipe(
          map(account => new LoadLoyaltyAccountSuccess(account)),
          catchError(error => of(new LoadLoyaltyAccountFail(error)))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private loyaltyConnector: LoyaltyConnector
  ) {}
}
```

**Connector (OCC Adapter):**

```typescript
// loyalty.connector.ts
@Injectable({ providedIn: 'root' })
export class LoyaltyConnector {
  constructor(private http: HttpClient, private occEndpoints: OccEndpointsService) {}

  getAccount(): Observable<LoyaltyAccount> {
    const url = this.occEndpoints.buildUrl('loyaltyAccount');
    return this.http.get<LoyaltyAccount>(url);
  }

  redeemPoints(points: number): Observable<LoyaltyAccount> {
    const url = this.occEndpoints.buildUrl('loyaltyRedeem');
    return this.http.post<LoyaltyAccount>(url, { points });
  }
}
```

**Endpoint Configuration:**

```typescript
provideConfig(<OccConfig>{
  backend: {
    occ: {
      endpoints: {
        loyaltyAccount: 'users/${userId}/loyalty/account',
        loyaltyRedeem: 'users/${userId}/loyalty/redeem',
        loyaltyTransactions: 'users/${userId}/loyalty/transactions?currentPage=${currentPage}&pageSize=${pageSize}',
      },
    },
  },
})
```

---

## Layout Configuration

Spartacus layout is configured through TypeScript objects, not CSS grids alone.

### Page Layout

```typescript
provideConfig(<LayoutConfig>{
  layoutSlots: {
    ProductDetailsPageTemplate: {
      slots: ['Summary', 'UpSelling', 'Tabs', 'CrossSelling'],
      lg: {
        slots: [
          { slot: 'Summary', flex: '60' },
          { slot: 'UpSelling', flex: '40' },
          'Tabs',
          'CrossSelling',
        ],
      },
    },
    LandingPage2Template: {
      slots: [
        'Section1',
        'Section2A',
        'Section2B',
        'Section3',
        'Section4',
        'Section5',
      ],
    },
    // Custom page template
    LoyaltyPageTemplate: {
      slots: ['LoyaltyHeader', 'LoyaltyDashboard', 'LoyaltyHistory'],
    },
  },
})
```

### Header and Footer

```typescript
provideConfig(<LayoutConfig>{
  layoutSlots: {
    header: {
      lg: {
        slots: [
          'PreHeader',
          'SiteContext',
          'SiteLinks',
          'SiteLogo',
          'SearchBox',
          'SiteLogin',
          'MiniCart',
          'NavigationBar',
        ],
      },
      slots: ['PreHeader', 'SiteLogo', 'SearchBox', 'MiniCart', 'hamburger'],
    },
    footer: {
      slots: ['Footer'],
    },
  },
})
```

---

## Styling and Theming

Spartacus uses SCSS with a BEM-like naming convention. Override styles through the component style hierarchy.

### Global Theme Variables

```scss
// styles.scss
$primary: #0a6ed1;
$secondary: #354a5f;
$font-family: 'Open Sans', sans-serif;

// Override Spartacus variables
$cx-g-font-family: $font-family;
$cx-g-color-primary: $primary;
$cx-g-color-secondary: $secondary;

// Import Spartacus styles
@import '@spartacus/styles';
@import '@spartacus/styles/scss/theme';
```

### Component-Level Overrides

```scss
// Custom product card styling
cx-product-list-item {
  .cx-product-image {
    border-radius: 8px;
    overflow: hidden;
  }
  
  .cx-product-name {
    font-weight: 600;
    font-size: 1.1rem;
  }
  
  .cx-product-price {
    color: $primary;
    font-size: 1.2rem;
  }
}

// Loyalty dashboard custom styles
app-loyalty-dashboard {
  .loyalty-dashboard {
    padding: 2rem;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border-radius: 12px;
    color: white;
  }
  
  .tier-badge {
    &.bronze { background: #cd7f32; }
    &.silver { background: #c0c0c0; color: #333; }
    &.gold { background: #ffd700; color: #333; }
    &.platinum { background: #e5e4e2; color: #333; }
    
    display: inline-block;
    padding: 0.25rem 1rem;
    border-radius: 20px;
    font-weight: bold;
    text-transform: uppercase;
  }
}
```

---

## Server-Side Rendering (SSR)

SSR is critical for SEO and initial page load performance. Spartacus supports SSR via Angular Universal.

### Enabling SSR

```bash
ng add @spartacus/schematics --ssr
```

This adds:

```typescript
// server.ts
import { ngExpressEngine } from '@nguniversal/express-engine';
import * as express from 'express';
import { AppServerModule } from './src/main.server';

const app = express();
const PORT = process.env['PORT'] || 4000;

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModule,
}));

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));

// Serve static files
app.get('*.*', express.static(join(DIST_FOLDER, 'browser')));

// All routes use SSR
app.get('*', (req, res) => {
  res.render('index', { req, providers: [{ provide: APP_BASE_HREF, useValue: req.baseUrl }] });
});

app.listen(PORT, () => console.log(`SSR server listening on port ${PORT}`));
```

### SSR Transfer State

Spartacus uses Angular's `TransferState` to avoid duplicate API calls. Data fetched on the server is serialized into the HTML and reused by the browser:

```
Server renders page → embeds API responses as JSON in HTML → 
Browser boots Angular → reads transferred state → skips redundant API calls
```

### SSR Performance Tips

1. **Cache SSR responses**: Use a reverse proxy (Varnish, CDN) to cache rendered HTML for anonymous pages
2. **Set SSR timeouts**: Prevent slow API calls from blocking the render:
   ```typescript
   provideConfig({
     ssr: {
       timeout: 3000, // Fallback to client-side render after 3s
     },
   })
   ```
3. **Skip SSR for authenticated pages**: Cart, checkout, and account pages don't benefit from SSR since they're unique per user

---

## Lazy Loading and Performance

### Feature Module Lazy Loading

Spartacus lazy-loads feature modules automatically:

```typescript
// spartacus-features.module.ts
@NgModule({
  imports: [
    // These modules are lazy-loaded when the user navigates to relevant pages
    CartBaseFeatureModule,     // Loaded when accessing cart
    CheckoutFeatureModule,     // Loaded at checkout
    UserFeatureModule,         // Loaded for account pages
    ProductFeatureModule,      // Loaded for PDP/PLP
  ],
})
export class SpartacusFeaturesModule {}
```

### Lazy Loading Custom Modules

```typescript
// Register a lazy-loaded custom module
provideConfig({
  featureModules: {
    loyalty: {
      module: () => import('./loyalty/loyalty.module').then(m => m.LoyaltyModule),
      cmsComponents: ['LoyaltyDashboardComponent', 'LoyaltyPointsDisplayComponent'],
    },
  },
})
```

The loyalty module is only downloaded when a page contains one of the listed CMS components.

### Bundle Analysis

```bash
# Analyze bundle sizes
ng build --stats-json
npx webpack-bundle-analyzer dist/mystore/browser/stats.json
```

Target main bundle size under 300KB gzipped for good initial load performance.

---

## Production Deployment on CCv2

### Build Configuration

```json
// angular.json (production build)
{
  "configurations": {
    "production": {
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "500kb",
          "maximumError": "1mb"
        }
      ],
      "outputHashing": "all",
      "sourceMap": false,
      "optimization": true,
      "buildOptimizer": true,
      "aot": true
    }
  }
}
```

### CCv2 js-storefront Configuration

In the CCv2 manifest:

```json
{
  "storefrontAddons": [],
  "jsStorefronts": [
    {
      "name": "mystore",
      "storefront": "mystore",
      "contextRoot": "",
      "nodeVersion": "18"
    }
  ]
}
```

### Environment-Specific Configuration

```typescript
// environment.prod.ts
export const environment = {
  production: true,
  occBaseUrl: '', // Empty string — relative URLs in CCv2 (same origin)
};

// Use in config
provideConfig(<OccConfig>{
  backend: {
    occ: {
      baseUrl: environment.occBaseUrl,
    },
  },
})
```

On CCv2, the JS storefront is served by the same domain as the OCC API, so no CORS configuration is needed.

---

## Best Practices

1. **Use CMS mapping for component placement** — don't hardcode component positions in templates. Let the CMS drive layout.

2. **Prefer configuration over code** — routes, endpoints, feature flags, and layout can all be changed via `provideConfig()` without modifying component code.

3. **Follow the adapter/connector pattern** — isolate OCC API calls in connectors. Components should never call `HttpClient` directly.

4. **Use `OnPush` change detection** — every custom component should use `ChangeDetectionStrategy.OnPush` for performance.

5. **Lazy load everything possible** — custom feature modules should be lazy-loaded via the `featureModules` configuration.

6. **Test with SSR enabled** — components that use browser-only APIs (`window`, `document`, `localStorage`) break SSR. Use Angular's platform checks:
   ```typescript
   import { isPlatformBrowser } from '@angular/common';
   if (isPlatformBrowser(this.platformId)) {
     window.scrollTo(0, 0);
   }
   ```

7. **Don't fight the framework** — Spartacus has established patterns for customization. Replacing CMS component mappings is preferred over forking library code.

8. **Keep custom libraries separate** — organize custom features as Angular libraries within the workspace to maintain clean dependency boundaries.

---

## Summary

SAP Commerce Composable Storefront (Spartacus) brings modern frontend architecture to SAP Commerce. The key concepts:

1. **CMS-driven rendering** — page structure comes from the backend CMS, not hardcoded Angular routes
2. **Component mapping** — CMS component types map to Angular components via configuration
3. **NgRx state management** — global state flows through actions, reducers, and effects
4. **OCC adapters** — all backend communication goes through typed connectors
5. **Configuration-first** — routes, layouts, endpoints, and component mappings are configurable TypeScript objects
6. **SSR for SEO** — server-side rendering delivers crawlable HTML for search engines
7. **Lazy loading** — feature modules load on demand for faster initial page loads

The combination of headless architecture and CMS-driven rendering gives teams the flexibility to evolve the frontend independently from the backend — while giving business users the power to manage content without developer involvement.