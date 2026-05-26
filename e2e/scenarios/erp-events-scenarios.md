# ERP Integration E2E Test Scenarios

This document specifies the end-to-end test scenarios for federated ERP synchronization and cross-service event flows.

## Event Bus Integration Scenarios

### `events/propagation.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { EventBusMonitor } from '../utils/event-bus-monitor';
import { ServiceHealthMonitor } from '../utils/service-health';
import { seedUser, seedOrder, seedProduct } from '../fixtures';

test.describe('@regression @events Event Propagation', () => {
  let eventMonitor: EventBusMonitor;
  let healthMonitor: ServiceHealthMonitor;

  test.beforeAll(async () => {
    eventMonitor = new EventBusMonitor();
    healthMonitor = new ServiceHealthMonitor();
    await Promise.all([
      eventMonitor.connect(),
      healthMonitor.verifyAllServicesHealthy(),
    ]);
  });

  test.afterAll(async () => {
    await eventMonitor.disconnect();
  });

  test('user.created event provisions LMS access', async ({ request }) => {
    // Create a new user
    const response = await request.post('/api/users', {
      data: {
        email: 'newtherapist@example.com',
        role: 'therapist',
        organizationId: 'org_123',
      },
    });
    const user = await response.json();

    // Wait for event propagation
    const event = await eventMonitor.waitForEvent('user.created', {
      userId: user.id,
    });
    expect(event.payload.role).toBe('therapist');

    // Verify LMS received event
    const lmsEvent = await eventMonitor.waitForEvent('lms.user.provisioned', {
      userId: user.id,
    });
    expect(lmsEvent.payload.accessLevel).toBe('learner');

    // Verify user can access LMS
    const lmsAccess = await request.get(`/api/lms/users/${user.id}/access`);
    expect(lmsAccess.ok()).toBeTruthy();
  });

  test('order.placed event triggers inventory update', async ({ request }) => {
    const product = await seedProduct({ inventory: 100 });
    const order = {
      customerId: 'cust_123',
      items: [{ productId: product.id, quantity: 5 }],
    };

    // Place order
    const response = await request.post('/api/orders', { data: order });
    const placedOrder = await response.json();

    // Wait for order event
    const orderEvent = await eventMonitor.waitForEvent('order.placed', {
      orderId: placedOrder.id,
    });

    // Wait for inventory update event
    const inventoryEvent = await eventMonitor.waitForEvent('inventory.updated', {
      productId: product.id,
    });
    expect(inventoryEvent.payload.previousQuantity).toBe(100);
    expect(inventoryEvent.payload.newQuantity).toBe(95);
  });

  test('certification.awarded event updates therapist profile', async ({ request }) => {
    const therapist = await seedUser({ role: 'therapist' });

    // Publish certification event (simulated from LMS)
    await request.post('/api/events/publish', {
      data: {
        type: 'certification.awarded',
        payload: {
          userId: therapist.id,
          certificationId: 'cert_skincare_advanced',
          courseId: 'course_123',
          awardedAt: new Date().toISOString(),
        },
      },
    });

    // Wait for profile update event
    const profileEvent = await eventMonitor.waitForEvent('therapist.profile.updated', {
      userId: therapist.id,
    });
    expect(profileEvent.payload.newCertifications).toContain('cert_skincare_advanced');

    // Verify certification visible in portal
    const profile = await request.get(`/api/therapists/${therapist.id}`);
    const data = await profile.json();
    expect(data.certifications).toContainEqual(
      expect.objectContaining({ id: 'cert_skincare_advanced' })
    );
  });

  test('events are delivered exactly once (idempotency)', async ({ request }) => {
    const eventId = `evt_${Date.now()}`;

    // Publish same event twice
    await request.post('/api/events/publish', {
      data: {
        id: eventId,
        type: 'test.event',
        payload: { test: true },
      },
    });

    await request.post('/api/events/publish', {
      data: {
        id: eventId,
        type: 'test.event',
        payload: { test: true },
      },
    });

    // Verify only processed once
    const processedEvents = await eventMonitor.getProcessedEvents('test.event', { id: eventId });
    expect(processedEvents).toHaveLength(1);
  });
});

test.describe('@exhaustive @events @resilience Event Resilience', () => {
  test('events are retried on consumer failure', async ({ request }) => {
    const eventMonitor = new EventBusMonitor();

    // Simulate consumer failure for first attempt
    await eventMonitor.simulateConsumerFailure('inventory.updated', { failCount: 2 });

    // Publish event
    const product = await seedProduct({ inventory: 50 });
    await request.post('/api/events/publish', {
      data: {
        type: 'inventory.updated',
        payload: { productId: product.id, newQuantity: 45 },
      },
    });

    // Wait for successful delivery (after retries)
    const event = await eventMonitor.waitForEvent('inventory.updated', {
      productId: product.id,
      processed: true,
    }, { timeout: 30000 });

    expect(event.retryCount).toBe(2);
    expect(event.status).toBe('processed');
  });

  test('dead letter queue captures failed events', async ({ request }) => {
    const eventMonitor = new EventBusMonitor();

    // Simulate permanent consumer failure
    await eventMonitor.simulateConsumerFailure('test.permanent.failure', { permanent: true });

    const eventId = `evt_${Date.now()}`;
    await request.post('/api/events/publish', {
      data: {
        id: eventId,
        type: 'test.permanent.failure',
        payload: { data: 'test' },
      },
    });

    // Wait for DLQ
    await new Promise(r => setTimeout(r, 10000));

    // Verify in dead letter queue
    const dlq = await request.get('/api/events/dlq');
    const dlqData = await dlq.json();
    expect(dlqData.events).toContainEqual(
      expect.objectContaining({ id: eventId })
    );
  });

  test('events preserve ordering within partition', async ({ request }) => {
    const eventMonitor = new EventBusMonitor();
    const orderId = `order_${Date.now()}`;

    // Publish ordered events
    const events = [
      { type: 'order.created', payload: { orderId, step: 1 } },
      { type: 'order.paid', payload: { orderId, step: 2 } },
      { type: 'order.fulfilled', payload: { orderId, step: 3 } },
    ];

    for (const event of events) {
      await request.post('/api/events/publish', {
        data: { ...event, partitionKey: orderId },
      });
    }

    // Wait and verify ordering
    const processedEvents = await eventMonitor.getProcessedEventsInOrder(orderId);
    expect(processedEvents.map(e => e.payload.step)).toEqual([1, 2, 3]);
  });
});
```

## Federated ERP Scenarios

### `erp/catalog-sync.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { ERPMock } from '../utils/erp-mock';
import { ShopifyMock } from '../utils/shopify-mock';
import { EventBusMonitor } from '../utils/event-bus-monitor';

test.describe('@regression @erp Catalog Synchronization', () => {
  let erpMock: ERPMock;
  let shopifyMock: ShopifyMock;
  let eventMonitor: EventBusMonitor;

  test.beforeAll(async () => {
    erpMock = new ERPMock();
    shopifyMock = new ShopifyMock();
    eventMonitor = new EventBusMonitor();
    await Promise.all([
      erpMock.start(),
      shopifyMock.start(),
      eventMonitor.connect(),
    ]);
  });

  test.afterAll(async () => {
    await Promise.all([
      erpMock.stop(),
      shopifyMock.stop(),
      eventMonitor.disconnect(),
    ]);
  });

  test('new ERP product syncs to Shopify', async ({ request }) => {
    // Create product in ERP
    const erpProduct = await erpMock.createProduct({
      sku: 'SKU-001',
      name: 'Premium Serum',
      price: 79.99,
      inventory: 200,
      category: 'skincare',
    });

    // Trigger sync
    await request.post('/api/erp/sync/products');

    // Wait for sync event
    await eventMonitor.waitForEvent('erp.product.synced', {
      sku: erpProduct.sku,
    });

    // Verify in Shopify
    const shopifyProduct = await shopifyMock.getProductBySku(erpProduct.sku);
    expect(shopifyProduct.title).toBe('Premium Serum');
    expect(shopifyProduct.variants[0].price).toBe('79.99');
    expect(shopifyProduct.variants[0].inventory_quantity).toBe(200);
  });

  test('ERP product update syncs to all channels', async ({ request }) => {
    const erpProduct = await erpMock.createProduct({
      sku: 'SKU-002',
      name: 'Original Name',
      price: 49.99,
    });

    // Initial sync
    await request.post('/api/erp/sync/products');

    // Update in ERP
    await erpMock.updateProduct(erpProduct.id, {
      name: 'Updated Name',
      price: 59.99,
    });

    // Trigger update sync
    await request.post('/api/erp/sync/products');

    // Verify Shopify updated
    const shopifyProduct = await shopifyMock.getProductBySku(erpProduct.sku);
    expect(shopifyProduct.title).toBe('Updated Name');
    expect(shopifyProduct.variants[0].price).toBe('59.99');

    // Verify internal catalog updated
    const internalProduct = await request.get(`/api/products?sku=${erpProduct.sku}`);
    const data = await internalProduct.json();
    expect(data.products[0].name).toBe('Updated Name');
  });

  test('price list changes sync correctly', async ({ request }) => {
    const erpProduct = await erpMock.createProduct({ sku: 'SKU-003', price: 100 });
    await request.post('/api/erp/sync/products');

    // Update price list for specific region
    await erpMock.updatePriceList('US', [
      { sku: erpProduct.sku, price: 89.99 },
    ]);

    // Sync price lists
    await request.post('/api/erp/sync/price-lists');

    // Verify regional pricing
    const pricing = await request.get(`/api/pricing?sku=${erpProduct.sku}&region=US`);
    const data = await pricing.json();
    expect(data.price).toBe(89.99);
  });

  test('sync handles ERP downtime gracefully', async ({ request }) => {
    // Simulate ERP outage
    await erpMock.simulateOutage(5000);

    // Attempt sync
    const response = await request.post('/api/erp/sync/products');
    expect(response.status()).toBe(503);

    // Verify queued for retry
    const syncStatus = await request.get('/api/erp/sync/status');
    const status = await syncStatus.json();
    expect(status.pendingSyncs).toBeGreaterThan(0);

    // Wait for ERP recovery
    await new Promise(r => setTimeout(r, 6000));

    // Verify eventual sync
    const retryStatus = await request.get('/api/erp/sync/status');
    const retryData = await retryStatus.json();
    expect(retryData.lastSuccessfulSync).toBeDefined();
  });

  test('sync conflict resolution follows policy', async ({ request }) => {
    // Create product with same SKU in both systems
    const erpProduct = await erpMock.createProduct({
      sku: 'SKU-CONFLICT',
      name: 'ERP Name',
      price: 50,
    });

    await shopifyMock.createProduct({
      sku: 'SKU-CONFLICT',
      title: 'Shopify Name',
      price: '60',
    });

    // Sync with ERP-wins policy
    await request.post('/api/erp/sync/products', {
      data: { conflictPolicy: 'erp-wins' },
    });

    // Verify ERP data prevails
    const shopifyProduct = await shopifyMock.getProductBySku('SKU-CONFLICT');
    expect(shopifyProduct.title).toBe('ERP Name');
    expect(shopifyProduct.variants[0].price).toBe('50');
  });
});
```

### `erp/inventory-sync.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { ERPMock } from '../utils/erp-mock';
import { ShopifyMock } from '../utils/shopify-mock';
import { EventBusMonitor } from '../utils/event-bus-monitor';

test.describe('@regression @erp Inventory Synchronization', () => {
  let erpMock: ERPMock;
  let shopifyMock: ShopifyMock;
  let eventMonitor: EventBusMonitor;

  test.beforeAll(async () => {
    erpMock = new ERPMock();
    shopifyMock = new ShopifyMock();
    eventMonitor = new EventBusMonitor();
    await Promise.all([
      erpMock.start(),
      shopifyMock.start(),
      eventMonitor.connect(),
    ]);
  });

  test('ERP inventory update syncs to Shopify', async ({ request }) => {
    const sku = 'INV-001';
    await erpMock.setInventory(sku, { warehouse: 'WH-1', quantity: 150 });

    // Trigger sync
    await request.post('/api/erp/sync/inventory');

    // Wait for event
    await eventMonitor.waitForEvent('inventory.synced', { sku });

    // Verify Shopify inventory
    const shopifyInventory = await shopifyMock.getInventoryBySku(sku);
    expect(shopifyInventory.available).toBe(150);
  });

  test('multi-warehouse inventory aggregates correctly', async ({ request }) => {
    const sku = 'INV-002';
    await erpMock.setInventory(sku, { warehouse: 'WH-1', quantity: 100 });
    await erpMock.setInventory(sku, { warehouse: 'WH-2', quantity: 75 });
    await erpMock.setInventory(sku, { warehouse: 'WH-3', quantity: 50 });

    // Sync
    await request.post('/api/erp/sync/inventory');

    // Verify aggregated inventory
    const shopifyInventory = await shopifyMock.getInventoryBySku(sku);
    expect(shopifyInventory.available).toBe(225);
  });

  test('low stock triggers alert', async ({ request }) => {
    const sku = 'INV-003';
    await erpMock.setInventory(sku, { warehouse: 'WH-1', quantity: 5 });
    await erpMock.setProductReorderPoint(sku, 10);

    // Sync
    await request.post('/api/erp/sync/inventory');

    // Verify low stock alert
    const alert = await eventMonitor.waitForEvent('inventory.low-stock', { sku });
    expect(alert.payload.currentQuantity).toBe(5);
    expect(alert.payload.reorderPoint).toBe(10);
  });

  test('Shopify order decrements ERP inventory', async ({ request }) => {
    const sku = 'INV-004';
    await erpMock.setInventory(sku, { warehouse: 'WH-1', quantity: 100 });

    // Create Shopify order
    await shopifyMock.createOrder({
      line_items: [{ sku, quantity: 10 }],
    });

    // Wait for sync back to ERP
    await eventMonitor.waitForEvent('erp.inventory.decremented', { sku });

    // Verify ERP inventory
    const erpInventory = await erpMock.getInventory(sku);
    expect(erpInventory.total).toBe(90);
  });
});
```

### `erp/order-posting.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { ERPMock } from '../utils/erp-mock';
import { ShopifyMock } from '../utils/shopify-mock';
import { EventBusMonitor } from '../utils/event-bus-monitor';

test.describe('@regression @erp Order Financial Posting', () => {
  let erpMock: ERPMock;
  let shopifyMock: ShopifyMock;
  let eventMonitor: EventBusMonitor;

  test.beforeAll(async () => {
    erpMock = new ERPMock();
    shopifyMock = new ShopifyMock();
    eventMonitor = new EventBusMonitor();
    await Promise.all([
      erpMock.start(),
      shopifyMock.start(),
      eventMonitor.connect(),
    ]);
  });

  test('fulfilled order creates ERP invoice', async ({ request }) => {
    // Create and fulfill Shopify order
    const order = await shopifyMock.createOrder({
      total_price: '150.00',
      line_items: [
        { sku: 'PROD-1', quantity: 2, price: '50.00' },
        { sku: 'PROD-2', quantity: 1, price: '50.00' },
      ],
      customer: { email: 'customer@example.com' },
    });

    await shopifyMock.fulfillOrder(order.id);

    // Wait for ERP posting
    const postingEvent = await eventMonitor.waitForEvent('erp.invoice.created', {
      sourceOrderId: order.id,
    });

    // Verify ERP invoice
    const invoice = await erpMock.getInvoice(postingEvent.payload.invoiceId);
    expect(invoice.amount).toBe(150.00);
    expect(invoice.lineItems).toHaveLength(2);
    expect(invoice.customerEmail).toBe('customer@example.com');
  });

  test('refund creates ERP credit note', async ({ request }) => {
    // Create fulfilled order
    const order = await shopifyMock.createOrder({ total_price: '100.00' });
    await shopifyMock.fulfillOrder(order.id);

    // Wait for invoice
    const invoiceEvent = await eventMonitor.waitForEvent('erp.invoice.created');

    // Process refund
    await shopifyMock.refundOrder(order.id, { amount: '50.00' });

    // Wait for credit note
    const creditEvent = await eventMonitor.waitForEvent('erp.credit-note.created', {
      sourceOrderId: order.id,
    });

    // Verify credit note
    const creditNote = await erpMock.getCreditNote(creditEvent.payload.creditNoteId);
    expect(creditNote.amount).toBe(50.00);
    expect(creditNote.relatedInvoiceId).toBe(invoiceEvent.payload.invoiceId);
  });

  test('posting uses correct GL accounts', async ({ request }) => {
    const order = await shopifyMock.createOrder({
      total_price: '200.00',
      line_items: [
        { sku: 'SERVICE-1', quantity: 1, price: '100.00', type: 'service' },
        { sku: 'PRODUCT-1', quantity: 1, price: '100.00', type: 'product' },
      ],
    });
    await shopifyMock.fulfillOrder(order.id);

    // Wait for posting
    const postingEvent = await eventMonitor.waitForEvent('erp.invoice.created');

    // Verify GL postings
    const journalEntries = await erpMock.getJournalEntries(postingEvent.payload.invoiceId);

    // Service revenue account
    expect(journalEntries).toContainEqual(
      expect.objectContaining({ account: '4100-SERVICE-REV', credit: 100 })
    );
    // Product revenue account
    expect(journalEntries).toContainEqual(
      expect.objectContaining({ account: '4000-PRODUCT-REV', credit: 100 })
    );
    // AR account
    expect(journalEntries).toContainEqual(
      expect.objectContaining({ account: '1200-AR', debit: 200 })
    );
  });

  test('posting is idempotent', async ({ request }) => {
    const order = await shopifyMock.createOrder({ total_price: '100.00' });
    await shopifyMock.fulfillOrder(order.id);

    // Wait for first posting
    await eventMonitor.waitForEvent('erp.invoice.created', {
      sourceOrderId: order.id,
    });

    // Simulate duplicate fulfillment webhook
    await shopifyMock.sendWebhook('orders/fulfilled', order);

    // Wait and check no duplicate
    await new Promise(r => setTimeout(r, 5000));
    const invoices = await erpMock.getInvoicesForOrder(order.id);
    expect(invoices).toHaveLength(1);
  });

  test('failed posting enters retry queue', async ({ request }) => {
    // Simulate ERP finance module down
    await erpMock.simulateModuleOutage('finance', 10000);

    const order = await shopifyMock.createOrder({ total_price: '100.00' });
    await shopifyMock.fulfillOrder(order.id);

    // Verify in retry queue
    await new Promise(r => setTimeout(r, 2000));
    const retryQueue = await request.get('/api/erp/posting/retry-queue');
    const queue = await retryQueue.json();
    expect(queue.pending).toContainEqual(
      expect.objectContaining({ orderId: order.id })
    );

    // Wait for recovery and retry
    await new Promise(r => setTimeout(r, 12000));

    // Verify eventually posted
    const invoices = await erpMock.getInvoicesForOrder(order.id);
    expect(invoices).toHaveLength(1);
  });
});

test.describe('@exhaustive @erp @resilience Posting Resilience', () => {
  test('handles concurrent postings correctly', async ({ request }) => {
    const erpMock = new ERPMock();
    const shopifyMock = new ShopifyMock();

    // Create 20 orders simultaneously
    const orders = await Promise.all(
      Array.from({ length: 20 }, (_, i) =>
        shopifyMock.createOrder({ total_price: `${(i + 1) * 10}.00` })
      )
    );

    // Fulfill all orders
    await Promise.all(orders.map(o => shopifyMock.fulfillOrder(o.id)));

    // Wait for all postings
    await new Promise(r => setTimeout(r, 30000));

    // Verify all invoices created
    for (const order of orders) {
      const invoices = await erpMock.getInvoicesForOrder(order.id);
      expect(invoices).toHaveLength(1);
    }
  });

  test('recovers from partial posting failure', async ({ request }) => {
    const erpMock = new ERPMock();
    const shopifyMock = new ShopifyMock();

    // Create order
    const order = await shopifyMock.createOrder({
      total_price: '100.00',
      line_items: [
        { sku: 'ITEM-1', price: '50.00' },
        { sku: 'ITEM-2', price: '50.00' },
      ],
    });

    // Simulate partial failure (first line item fails)
    await erpMock.simulateLineItemFailure('ITEM-1', { failCount: 1 });

    await shopifyMock.fulfillOrder(order.id);

    // Wait for recovery
    await new Promise(r => setTimeout(r, 15000));

    // Verify complete posting
    const invoices = await erpMock.getInvoicesForOrder(order.id);
    expect(invoices[0].lineItems).toHaveLength(2);
    expect(invoices[0].lineItems.every(li => li.status === 'posted')).toBe(true);
  });
});
```

---

## Utility Classes

### `utils/erp-mock.ts`

```typescript
export class ERPMock {
  private server: MockServer;
  private products: Map<string, ERPProduct> = new Map();
  private inventory: Map<string, InventoryRecord[]> = new Map();
  private invoices: Map<string, Invoice> = new Map();

  async start() {
    this.server = await createMockServer({
      '/api/products': this.handleProducts,
      '/api/inventory': this.handleInventory,
      '/api/invoices': this.handleInvoices,
      '/api/journal-entries': this.handleJournalEntries,
    });
  }

  async createProduct(data: CreateProductInput): Promise<ERPProduct> {
    const product = { id: `erp_${Date.now()}`, ...data };
    this.products.set(data.sku, product);
    return product;
  }

  async setInventory(sku: string, record: InventoryRecord) {
    const records = this.inventory.get(sku) || [];
    records.push(record);
    this.inventory.set(sku, records);
  }

  async simulateOutage(durationMs: number) {
    this.server.setUnavailable(true);
    setTimeout(() => this.server.setUnavailable(false), durationMs);
  }

  async simulateModuleOutage(module: string, durationMs: number) {
    this.server.setModuleUnavailable(module, true);
    setTimeout(() => this.server.setModuleUnavailable(module, false), durationMs);
  }
}
```

### `utils/event-bus-monitor.ts`

```typescript
export class EventBusMonitor {
  private connection: EventBusConnection;
  private receivedEvents: Event[] = [];

  async connect() {
    this.connection = await connectToEventBus(process.env.EVENT_BUS_URL);
    this.connection.on('event', (event) => this.receivedEvents.push(event));
  }

  async waitForEvent(
    type: string,
    matcher: Record<string, any>,
    options: { timeout?: number } = {}
  ): Promise<Event> {
    const timeout = options.timeout || 10000;
    const start = Date.now();

    while (Date.now() - start < timeout) {
      const event = this.receivedEvents.find(
        (e) => e.type === type && this.matchesPayload(e.payload, matcher)
      );
      if (event) return event;
      await new Promise((r) => setTimeout(r, 100));
    }

    throw new Error(`Timeout waiting for event ${type}`);
  }

  async simulateConsumerFailure(
    eventType: string,
    options: { failCount?: number; permanent?: boolean }
  ) {
    await this.connection.send('__test__.simulate-failure', {
      eventType,
      ...options,
    });
  }

  private matchesPayload(payload: any, matcher: Record<string, any>): boolean {
    return Object.entries(matcher).every(
      ([key, value]) => payload[key] === value
    );
  }
}
```
