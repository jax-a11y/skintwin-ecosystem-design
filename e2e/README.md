# SkinTwin Ecosystem E2E Testing Strategy

This document defines the end-to-end testing strategy for the SkinTwin-AI ecosystem, covering test scenarios, execution tiers, environment management, and reliability engineering practices.

## 1. Overview

End-to-end tests validate that the integrated ecosystem functions correctly from the user's perspective. Our E2E strategy covers:

- **Critical user journeys** across all applications
- **Cross-service integration** between frontend and backend
- **External platform interactions** (Shopify, Wix, payment providers)
- **Event-driven flows** through the message bus
- **Federated ERP synchronization** across business units

## 2. E2E Test Tiers

### 2.1 Tier Definitions

| Tier | Scope | Trigger | Blocking | Duration |
|------|-------|---------|----------|----------|
| **Smoke** | Critical paths only | PR | Yes | < 5 min |
| **Regression** | Core functionality | Push to main | Yes | < 15 min |
| **Exhaustive** | All scenarios + edge cases | Nightly | No (alert) | < 60 min |

### 2.2 Tier Distribution

```
┌─────────────────────────────────────────────────────────────┐
│                      EXHAUSTIVE                              │
│  Full coverage including edge cases, resilience tests,       │
│  and all browser/device combinations                         │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    REGRESSION                          │  │
│  │  Core user flows, happy paths + primary error states   │  │
│  │                                                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │                    SMOKE                         │  │  │
│  │  │  Login, critical purchase, core navigation       │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. Critical User Journeys

### 3.1 Shopify App Journeys

#### Journey: App Installation & Onboarding
```
@smoke @shopify
Scenario: Merchant installs and onboards to the app
  Given a Shopify merchant visits the app listing
  When they click "Add app" and authorize permissions
  Then the OAuth flow completes successfully
  And the app is installed on their store
  And the onboarding wizard is displayed
  And the merchant can configure initial settings
```

#### Journey: Session Management
```
@smoke @shopify
Scenario: Authenticated session persistence
  Given a merchant has the app installed
  When they access the embedded app from Shopify admin
  Then the session token is validated
  And they can access protected resources
  And the session persists across navigation
```

#### Journey: Webhook Processing
```
@regression @shopify
Scenario: Order webhook triggers ERP sync
  Given a customer places an order on Shopify
  When the order webhook is received
  Then the order is validated and acknowledged
  And the order is posted to the event bus
  And downstream services process the order
  And the ERP receives the financial posting
```

### 3.2 Customer Portal Journeys

#### Journey: Customer Registration & Login
```
@smoke @portal
Scenario: New customer registration
  Given a visitor on the customer portal
  When they complete the registration form
  Then their account is created
  And they receive a verification email
  And they can log in after verification

@smoke @portal
Scenario: Existing customer login
  Given a registered customer
  When they enter valid credentials
  Then they are authenticated via WorkOS SSO
  And they are redirected to their dashboard
```

#### Journey: Booking & Appointment Flow
```
@smoke @portal
Scenario: Customer books an appointment
  Given an authenticated customer
  When they search for a salon
  And they select an available time slot
  And they confirm the booking
  Then the appointment is created
  And the booking syncs to Wix Bookings
  And the customer receives a confirmation email
```

#### Journey: Product Purchase
```
@smoke @portal
Scenario: Customer purchases products
  Given an authenticated customer
  When they add products to cart
  And they proceed to checkout
  And they complete payment via Stripe
  Then the order is created in Shopify
  And the customer receives an order confirmation
  And inventory is updated across systems
```

### 3.3 Training LMS Journeys

#### Journey: Course Enrollment
```
@smoke @lms
Scenario: Therapist enrolls in a course
  Given an authenticated therapist
  When they browse the course catalog
  And they enroll in a certification course
  Then they are added to the course roster
  And they can access course materials
  And xAPI tracking is initialized
```

#### Journey: Course Completion & Certification
```
@regression @lms
Scenario: Therapist completes certification
  Given a therapist enrolled in a course
  When they complete all modules
  And they pass the final assessment
  Then they receive a certificate
  And their certification is recorded
  And the completion syncs to customer portal
```

#### Journey: SCORM Content Playback
```
@regression @lms
Scenario: SCORM package loads and tracks
  Given a course with SCORM content
  When a learner opens the SCORM module
  Then the package loads in the LMS player
  And progress is tracked via SCORM API
  And completion status is persisted
```

### 3.4 Business Directory Journeys

#### Journey: Directory Search & Discovery
```
@smoke @directory
Scenario: User searches for a salon
  Given a user on the business directory
  When they enter a location and search
  Then relevant results are displayed
  And results can be filtered by service type
  And the map shows business locations
```

#### Journey: Business Profile View
```
@smoke @directory
Scenario: User views business details
  Given search results are displayed
  When the user clicks on a business
  Then the business profile is shown
  And services, hours, and contact info are displayed
  And the user can initiate a booking
```

### 3.5 Cross-Service Integration Journeys

#### Journey: AI Skin Analysis
```
@regression @ai
Scenario: Customer receives skin analysis
  Given an authenticated customer
  When they upload a skin image
  Then the cognitive service analyzes the image
  And skin concerns are identified
  And personalized product recommendations are generated
  And results are displayed in the portal
```

#### Journey: Event Bus Propagation
```
@regression @events
Scenario: Events propagate across services
  Given a new customer registers
  When the registration event is published
  Then the LMS service receives the event
  And the customer is provisioned for training access
  And the integration service updates external platforms
```

### 3.6 Federated ERP Journeys

#### Journey: Catalog Synchronization
```
@regression @erp
Scenario: Product catalog syncs from ERP
  Given products are updated in the enterprise ERP
  When the sync job runs
  Then products are mapped to canonical entities
  And Shopify product listings are updated
  And internal services reflect the changes
```

#### Journey: Order Financial Posting
```
@regression @erp
Scenario: Order posts to ERP finance
  Given an order is fulfilled
  When the fulfillment event is processed
  Then the invoice is generated
  And the financial posting is sent to ERP
  And the posting is confirmed and reconciled
```

#### Journey: Inventory Synchronization
```
@regression @erp
Scenario: Inventory levels sync bidirectionally
  Given inventory is updated in a warehouse
  When the inventory event is received
  Then all storefronts reflect the new levels
  And low stock alerts are triggered if applicable
  And replenishment recommendations are generated
```

## 4. Test Environment Strategy

### 4.1 Environment Types

| Environment | Purpose | Data | External Services |
|-------------|---------|------|-------------------|
| **Preview** | PR validation | Seeded fixtures | Mocked/stubbed |
| **Staging** | Pre-production | Anonymized production | Sandbox accounts |
| **Production** | Synthetic monitoring | Synthetic users | Live (read-only) |

### 4.2 Ephemeral Preview Environments

For each PR that affects runtime behavior:

```yaml
preview:
  trigger: pull_request
  infrastructure:
    - Deploy services to isolated namespace
    - Provision test database with fixtures
    - Configure mock external services
    - Generate unique preview URL
  lifecycle:
    creation: On PR open
    update: On PR push
    teardown: On PR close/merge
```

### 4.3 External Service Handling

| Service | Preview Mode | Staging Mode |
|---------|--------------|--------------|
| Shopify | App Bridge mock | Partner sandbox |
| Wix | API mock | Developer sandbox |
| Stripe | Mock responses | Test mode |
| QuickBooks | Stubbed adapter | Sandbox company |
| WorkOS | Mock SSO | Test organization |

## 5. Test Data Management

### 5.1 Fixture Categories

```
e2e/fixtures/
├── tenants/              # Organization/tenant fixtures
│   ├── salon_basic.json
│   ├── salon_enterprise.json
│   └── therapist_solo.json
├── users/                # User account fixtures
│   ├── customer_basic.json
│   ├── therapist_certified.json
│   └── admin_super.json
├── products/             # Product catalog fixtures
│   ├── skincare_line.json
│   └── treatment_packages.json
├── bookings/             # Appointment fixtures
│   ├── upcoming_appointments.json
│   └── historical_bookings.json
├── courses/              # LMS content fixtures
│   ├── certification_course.json
│   └── scorm_package.json
└── erp/                  # ERP mapping fixtures
    ├── erp_products.json
    ├── erp_inventory.json
    └── erp_financial_accounts.json
```

### 5.2 Fixture Design Principles

1. **Deterministic**: Same fixtures produce same test outcomes
2. **Isolated**: Fixtures don't depend on external state
3. **Comprehensive**: Cover all entity relationships
4. **Realistic**: Mirror production data patterns
5. **Minimal**: Only include data needed for tests

### 5.3 Data Seeding

```typescript
// Example fixture loader
import { seedDatabase } from '@skintwin/e2e-fixtures';

beforeAll(async () => {
  await seedDatabase({
    tenants: ['salon_basic'],
    users: ['customer_basic', 'therapist_certified'],
    products: ['skincare_line'],
  });
});

afterAll(async () => {
  await cleanupDatabase();
});
```

## 6. E2E Reliability Engineering

### 6.1 Flaky Test Management

```
┌─────────────────────────────────────────────────────────────┐
│                    FLAKE DETECTION PIPELINE                  │
├─────────────────────────────────────────────────────────────┤
│  1. Test fails → Automatic retry (up to 2 times)            │
│  2. Passes on retry → Tagged as "potentially flaky"         │
│  3. Flaky 3+ times/week → Quarantined                       │
│  4. Quarantined tests → Run in isolation, not blocking      │
│  5. Weekly triage → Fix or permanently skip                 │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Retry Strategy

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results.json' }],
    // Custom flake reporter
    ['./reporters/flake-detector.ts'],
  ],
});
```

### 6.3 Approved Retry Classes

Only retry for these transient failure types:

| Failure Type | Retry | Reason |
|--------------|-------|--------|
| Network timeout | ✅ | Transient infrastructure |
| Element not found (within timeout) | ✅ | Render timing |
| Database connection | ✅ | Pool exhaustion |
| External API 5xx | ✅ | Third-party instability |
| Assertion failure | ❌ | Actual test failure |
| Authentication error | ❌ | Logic/config issue |

### 6.4 Failure Artifact Collection

On every failure, capture:

```yaml
artifacts:
  - screenshot: Full page at failure point
  - video: Full test recording (if enabled)
  - trace: Playwright trace file
  - network: HAR file of all requests
  - console: Browser console logs
  - service-logs: Backend service logs
  - event-bus: Relevant event traces
```

## 7. E2E Metrics & Reporting

### 7.1 Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Pass rate | > 99% | < 95% |
| Flake rate | < 1% | > 5% |
| Smoke duration | < 5 min | > 8 min |
| Regression duration | < 15 min | > 25 min |
| Test coverage (journeys) | 100% critical | < 90% |

### 7.2 Dashboard Components

```
┌─────────────────────────────────────────────────────────────┐
│                    E2E HEALTH DASHBOARD                      │
├─────────────────────────────────────────────────────────────┤
│  [Pass Rate Trend]  [Flake Rate Trend]  [Duration Trend]    │
├─────────────────────────────────────────────────────────────┤
│  Top Failing Tests (Last 7 Days)                            │
│  1. shopify/webhook-processing.spec.ts:42  (5 failures)     │
│  2. portal/booking-flow.spec.ts:108        (3 failures)     │
│  3. lms/scorm-tracking.spec.ts:67          (2 failures)     │
├─────────────────────────────────────────────────────────────┤
│  Flaky Tests Under Investigation                            │
│  • erp/inventory-sync.spec.ts (quarantined, assigned: @dev) │
│  • ai/analysis-timeout.spec.ts (investigating)              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 Notification Strategy

| Event | Channel | Recipients |
|-------|---------|------------|
| Smoke failure (PR) | PR comment | PR author |
| Regression failure (main) | Slack #ci-alerts | On-call team |
| Exhaustive failure (nightly) | Email digest | E2E owners |
| Flake threshold exceeded | Slack + issue | E2E team lead |

## 8. Test Organization

### 8.1 File Structure

```
e2e/
├── scenarios/                    # Test specifications
│   ├── shopify/
│   │   ├── installation.spec.ts
│   │   ├── session.spec.ts
│   │   └── webhooks.spec.ts
│   ├── portal/
│   │   ├── auth.spec.ts
│   │   ├── booking.spec.ts
│   │   └── purchase.spec.ts
│   ├── lms/
│   │   ├── enrollment.spec.ts
│   │   ├── completion.spec.ts
│   │   └── scorm.spec.ts
│   ├── directory/
│   │   ├── search.spec.ts
│   │   └── profile.spec.ts
│   ├── ai/
│   │   └── analysis.spec.ts
│   ├── events/
│   │   └── propagation.spec.ts
│   └── erp/
│       ├── catalog-sync.spec.ts
│       ├── order-posting.spec.ts
│       └── inventory-sync.spec.ts
├── fixtures/                     # Test data
├── pages/                        # Page objects
├── utils/                        # Test utilities
├── reporters/                    # Custom reporters
└── playwright.config.ts          # Playwright config
```

### 8.2 Test Tagging

Use tags to control test execution:

```typescript
test.describe('@smoke @portal', () => {
  test('customer can log in', async ({ page }) => {
    // ...
  });
});

test.describe('@regression @erp', () => {
  test('inventory syncs correctly', async ({ page }) => {
    // ...
  });
});

test.describe('@exhaustive @resilience', () => {
  test('handles network failure gracefully', async ({ page }) => {
    // ...
  });
});
```

### 8.3 Running Tests by Tag

```bash
# Run smoke tests only
pnpm exec playwright test --grep @smoke

# Run regression (includes smoke)
pnpm exec playwright test --grep "@smoke|@regression"

# Run exhaustive (all tests)
pnpm exec playwright test

# Run specific domain
pnpm exec playwright test --grep @shopify
```

## 9. Browser & Device Matrix

### 9.1 Smoke/Regression

| Browser | Viewport | Priority |
|---------|----------|----------|
| Chromium | Desktop (1280x720) | P0 |
| Firefox | Desktop (1280x720) | P1 |
| WebKit | Desktop (1280x720) | P1 |

### 9.2 Exhaustive (Nightly)

| Browser | Viewport | Device |
|---------|----------|--------|
| Chromium | Desktop | - |
| Chromium | Tablet (768x1024) | iPad |
| Chromium | Mobile (375x667) | iPhone 12 |
| Firefox | Desktop | - |
| WebKit | Desktop | - |
| WebKit | Mobile | iPhone 12 |

## 10. Implementation Checklist

### Phase 1: Foundation
- [ ] Set up Playwright project structure
- [ ] Create page objects for main applications
- [ ] Implement fixture loading system
- [ ] Configure CI workflow integration

### Phase 2: Smoke Tests
- [ ] Implement Shopify smoke scenarios
- [ ] Implement portal smoke scenarios
- [ ] Implement LMS smoke scenarios
- [ ] Implement directory smoke scenarios
- [ ] Add to PR gating

### Phase 3: Regression Tests
- [ ] Implement webhook/event scenarios
- [ ] Implement AI integration scenarios
- [ ] Implement ERP sync scenarios
- [ ] Add to main branch gating

### Phase 4: Exhaustive Tests
- [ ] Implement resilience scenarios
- [ ] Implement idempotency tests
- [ ] Implement browser matrix
- [ ] Configure nightly schedule

### Phase 5: Observability
- [ ] Set up metrics collection
- [ ] Create health dashboard
- [ ] Configure alerting
- [ ] Establish triage process
