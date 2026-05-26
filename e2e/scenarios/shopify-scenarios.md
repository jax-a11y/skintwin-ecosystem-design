# Shopify App E2E Test Scenarios

This document specifies the end-to-end test scenarios for Shopify app integration in the SkinTwin ecosystem.

## Test Suite: Installation & Onboarding

### `installation.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { ShopifyMock } from '../utils/shopify-mock';
import { seedMerchant } from '../fixtures/merchants';

test.describe('@smoke @shopify Installation Flow', () => {
  let shopifyMock: ShopifyMock;

  test.beforeAll(async () => {
    shopifyMock = new ShopifyMock();
    await shopifyMock.start();
  });

  test.afterAll(async () => {
    await shopifyMock.stop();
  });

  test('merchant can install app from Shopify App Store', async ({ page }) => {
    // Navigate to app listing
    await page.goto('/api/auth/shopify');

    // Expect OAuth redirect to Shopify
    await expect(page).toHaveURL(/accounts\.shopify\.com/);

    // Complete OAuth flow (mocked)
    await shopifyMock.completeOAuth(page);

    // Verify redirect back to app
    await expect(page).toHaveURL(/\/app\/onboarding/);

    // Verify installation recorded
    const response = await page.request.get('/api/shop/current');
    expect(response.ok()).toBeTruthy();
  });

  test('app handles OAuth errors gracefully', async ({ page }) => {
    // Simulate OAuth denial
    await shopifyMock.simulateOAuthDenial();
    await page.goto('/api/auth/shopify');

    // Verify error page is shown
    await expect(page.locator('[data-testid="oauth-error"]')).toBeVisible();
    await expect(page.locator('[data-testid="retry-button"]')).toBeVisible();
  });

  test('app validates HMAC signature on install', async ({ page }) => {
    // Attempt install with invalid HMAC
    const invalidParams = shopifyMock.generateInvalidHmac();
    await page.goto(`/api/auth/callback?${invalidParams}`);

    // Verify rejection
    await expect(page).toHaveURL(/\/error/);
    await expect(page.locator('text=Invalid signature')).toBeVisible();
  });
});

test.describe('@regression @shopify Onboarding Wizard', () => {
  test.beforeEach(async ({ page }) => {
    // Seed a newly installed merchant
    await seedMerchant({ status: 'installed', onboarded: false });
    await page.goto('/app/onboarding');
  });

  test('merchant completes full onboarding flow', async ({ page }) => {
    // Step 1: Welcome
    await expect(page.locator('h1')).toContainText('Welcome');
    await page.click('[data-testid="next-button"]');

    // Step 2: Business Details
    await page.fill('[data-testid="business-name"]', 'Test Salon');
    await page.selectOption('[data-testid="business-type"]', 'salon');
    await page.click('[data-testid="next-button"]');

    // Step 3: ERP Configuration
    await page.selectOption('[data-testid="erp-provider"]', 'quickbooks');
    await page.click('[data-testid="connect-erp"]');
    await expect(page.locator('[data-testid="erp-connected"]')).toBeVisible();
    await page.click('[data-testid="next-button"]');

    // Step 4: Complete
    await expect(page.locator('h1')).toContainText('Ready');
    await page.click('[data-testid="go-to-dashboard"]');

    // Verify dashboard loads
    await expect(page).toHaveURL(/\/app\/dashboard/);
  });

  test('onboarding state persists across sessions', async ({ page, context }) => {
    // Complete first step
    await page.click('[data-testid="next-button"]');
    await page.fill('[data-testid="business-name"]', 'Test Salon');

    // Close and reopen
    await page.close();
    const newPage = await context.newPage();
    await newPage.goto('/app/onboarding');

    // Verify state persisted
    await expect(newPage.locator('[data-testid="business-name"]')).toHaveValue('Test Salon');
  });
});
```

## Test Suite: Session Management

### `session.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { createAuthenticatedContext } from '../utils/auth';
import { ShopifyAppBridge } from '../utils/app-bridge-mock';

test.describe('@smoke @shopify Session Management', () => {
  test('embedded app validates session token', async ({ browser }) => {
    // Create context with valid session
    const context = await createAuthenticatedContext(browser, {
      shopDomain: 'test-store.myshopify.com',
    });
    const page = await context.newPage();

    // Access embedded app
    await page.goto('/app/dashboard');

    // Verify session is valid
    await expect(page.locator('[data-testid="shop-name"]')).toContainText('test-store');
    await expect(page.locator('[data-testid="auth-status"]')).toContainText('Authenticated');
  });

  test('expired session triggers re-authentication', async ({ page }) => {
    // Set expired session token
    await page.route('/api/**', (route) => {
      route.fulfill({
        status: 401,
        json: { error: 'Session expired' },
      });
    });

    await page.goto('/app/dashboard');

    // Verify redirect to auth
    await expect(page).toHaveURL(/\/api\/auth/);
  });

  test('session refreshes automatically before expiry', async ({ browser }) => {
    const context = await createAuthenticatedContext(browser, {
      sessionExpiresIn: 60, // 1 minute
    });
    const page = await context.newPage();

    await page.goto('/app/dashboard');

    // Wait for refresh to occur
    await page.waitForRequest((req) => req.url().includes('/api/auth/refresh'));

    // Verify still authenticated
    await expect(page.locator('[data-testid="auth-status"]')).toContainText('Authenticated');
  });
});

test.describe('@regression @shopify App Bridge Integration', () => {
  let appBridge: ShopifyAppBridge;

  test.beforeEach(async ({ page }) => {
    appBridge = new ShopifyAppBridge(page);
    await appBridge.initialize();
  });

  test('app bridge toast notifications work', async ({ page }) => {
    await page.goto('/app/dashboard');

    // Trigger a toast
    await page.click('[data-testid="save-settings"]');

    // Verify App Bridge toast
    const toast = await appBridge.waitForToast();
    expect(toast.message).toContain('Settings saved');
  });

  test('app bridge navigation works', async ({ page }) => {
    await page.goto('/app/dashboard');

    // Use App Bridge navigation
    await page.click('[data-testid="nav-products"]');

    // Verify navigation event
    const navEvent = await appBridge.waitForNavigation();
    expect(navEvent.path).toBe('/products');
  });

  test('app bridge context bar actions work', async ({ page }) => {
    await page.goto('/app/products/edit/123');

    // Verify context bar shows
    const contextBar = await appBridge.waitForContextBar();
    expect(contextBar.actions).toContain('Save');
    expect(contextBar.actions).toContain('Discard');

    // Click save
    await contextBar.clickAction('Save');

    // Verify action executed
    await expect(page.locator('[data-testid="save-success"]')).toBeVisible();
  });
});
```

## Test Suite: Webhook Processing

### `webhooks.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { WebhookSimulator } from '../utils/webhook-simulator';
import { EventBusMonitor } from '../utils/event-bus-monitor';
import { seedOrder, seedProduct } from '../fixtures';

test.describe('@regression @shopify Webhook Processing', () => {
  let webhookSim: WebhookSimulator;
  let eventMonitor: EventBusMonitor;

  test.beforeAll(async () => {
    webhookSim = new WebhookSimulator({
      webhookSecret: process.env.SHOPIFY_WEBHOOK_SECRET,
    });
    eventMonitor = new EventBusMonitor();
    await eventMonitor.connect();
  });

  test.afterAll(async () => {
    await eventMonitor.disconnect();
  });

  test('orders/create webhook triggers order processing', async ({ request }) => {
    const order = seedOrder({ status: 'pending' });

    // Send webhook
    const response = await webhookSim.send(request, {
      topic: 'orders/create',
      payload: order,
    });

    expect(response.status()).toBe(200);

    // Verify event published to bus
    const event = await eventMonitor.waitForEvent('order.created', {
      orderId: order.id,
    });
    expect(event.payload.orderId).toBe(order.id);

    // Verify order stored
    const storedOrder = await request.get(`/api/orders/${order.id}`);
    expect(storedOrder.ok()).toBeTruthy();
  });

  test('orders/updated webhook triggers ERP sync', async ({ request }) => {
    // Create initial order
    const order = await seedOrder({ status: 'paid' });

    // Send fulfillment webhook
    const response = await webhookSim.send(request, {
      topic: 'orders/updated',
      payload: { ...order, fulfillment_status: 'fulfilled' },
    });

    expect(response.status()).toBe(200);

    // Verify ERP sync event
    const event = await eventMonitor.waitForEvent('erp.invoice.create', {
      orderId: order.id,
    });
    expect(event.payload.amount).toBe(order.total_price);
  });

  test('products/update webhook syncs inventory', async ({ request }) => {
    const product = await seedProduct({ inventory: 100 });

    // Send inventory update webhook
    const response = await webhookSim.send(request, {
      topic: 'products/update',
      payload: { ...product, variants: [{ ...product.variants[0], inventory_quantity: 50 }] },
    });

    expect(response.status()).toBe(200);

    // Verify inventory event
    const event = await eventMonitor.waitForEvent('inventory.updated', {
      productId: product.id,
    });
    expect(event.payload.newQuantity).toBe(50);
  });

  test('webhook with invalid signature is rejected', async ({ request }) => {
    const response = await webhookSim.send(request, {
      topic: 'orders/create',
      payload: seedOrder(),
      invalidSignature: true,
    });

    expect(response.status()).toBe(401);
  });

  test('webhook processing is idempotent', async ({ request }) => {
    const order = seedOrder();

    // Send same webhook twice
    await webhookSim.send(request, {
      topic: 'orders/create',
      payload: order,
    });

    const response = await webhookSim.send(request, {
      topic: 'orders/create',
      payload: order,
    });

    expect(response.status()).toBe(200);

    // Verify only one order created
    const orders = await request.get(`/api/orders?shopifyId=${order.id}`);
    const data = await orders.json();
    expect(data.orders).toHaveLength(1);
  });

  test('webhook failure triggers retry queue', async ({ request }) => {
    // Make downstream service unavailable
    await eventMonitor.simulateFailure('order.created');

    const order = seedOrder();
    const response = await webhookSim.send(request, {
      topic: 'orders/create',
      payload: order,
    });

    // Webhook acknowledged but queued for retry
    expect(response.status()).toBe(200);

    // Verify in retry queue
    const queueStatus = await request.get('/api/webhooks/queue');
    const queue = await queueStatus.json();
    expect(queue.pending).toContainEqual(
      expect.objectContaining({ orderId: order.id })
    );
  });
});

test.describe('@exhaustive @shopify @resilience Webhook Resilience', () => {
  test('handles burst of concurrent webhooks', async ({ request }) => {
    const webhookSim = new WebhookSimulator();
    const orders = Array.from({ length: 50 }, () => seedOrder());

    // Send all webhooks concurrently
    const responses = await Promise.all(
      orders.map((order) =>
        webhookSim.send(request, {
          topic: 'orders/create',
          payload: order,
        })
      )
    );

    // All should succeed
    responses.forEach((r) => expect(r.status()).toBe(200));

    // Verify all orders processed
    const allOrders = await request.get('/api/orders?limit=100');
    const data = await allOrders.json();
    expect(data.orders.length).toBeGreaterThanOrEqual(50);
  });

  test('recovers from database connection loss', async ({ request }) => {
    const webhookSim = new WebhookSimulator();

    // Simulate DB outage
    await request.post('/api/test/simulate-db-outage', {
      data: { duration: 5000 },
    });

    const order = seedOrder();
    const response = await webhookSim.send(request, {
      topic: 'orders/create',
      payload: order,
    });

    // Should still acknowledge
    expect(response.status()).toBe(200);

    // Wait for recovery
    await new Promise((r) => setTimeout(r, 6000));

    // Verify eventually processed
    const storedOrder = await request.get(`/api/orders/${order.id}`);
    expect(storedOrder.ok()).toBeTruthy();
  });
});
```

## Utility Classes

### `utils/shopify-mock.ts`

```typescript
export class ShopifyMock {
  private server: MockServer;

  async start() {
    this.server = await createMockServer({
      '/admin/oauth/authorize': this.handleOAuthAuthorize,
      '/admin/oauth/access_token': this.handleOAuthToken,
    });
  }

  async stop() {
    await this.server.close();
  }

  async completeOAuth(page: Page) {
    // Simulate clicking "Install app" button
    await page.click('[data-testid="install-app"]');
  }

  simulateOAuthDenial() {
    this.server.setHandler('/admin/oauth/authorize', (req, res) => {
      res.redirect('/api/auth/callback?error=access_denied');
    });
  }

  generateInvalidHmac(): string {
    return new URLSearchParams({
      shop: 'test-store.myshopify.com',
      hmac: 'invalid-hmac-value',
      timestamp: Date.now().toString(),
    }).toString();
  }
}
```

### `utils/webhook-simulator.ts`

```typescript
import crypto from 'crypto';

export class WebhookSimulator {
  private webhookSecret: string;

  constructor(options?: { webhookSecret?: string }) {
    this.webhookSecret = options?.webhookSecret || process.env.SHOPIFY_WEBHOOK_SECRET!;
  }

  async send(
    request: APIRequestContext,
    options: {
      topic: string;
      payload: any;
      invalidSignature?: boolean;
    }
  ) {
    const body = JSON.stringify(options.payload);
    const hmac = options.invalidSignature
      ? 'invalid'
      : this.generateHmac(body);

    return request.post('/api/webhooks/shopify', {
      headers: {
        'X-Shopify-Topic': options.topic,
        'X-Shopify-Hmac-Sha256': hmac,
        'X-Shopify-Shop-Domain': 'test-store.myshopify.com',
        'Content-Type': 'application/json',
      },
      data: body,
    });
  }

  private generateHmac(body: string): string {
    return crypto
      .createHmac('sha256', this.webhookSecret)
      .update(body, 'utf8')
      .digest('base64');
  }
}
```

## Test Data Fixtures

See `../fixtures/shopify/` for:
- `orders.ts` - Order payload generators
- `products.ts` - Product payload generators
- `customers.ts` - Customer payload generators
- `shops.ts` - Shop configuration fixtures
