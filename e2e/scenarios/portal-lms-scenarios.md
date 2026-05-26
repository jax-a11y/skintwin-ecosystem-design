# Portal & LMS E2E Test Scenarios

This document specifies the end-to-end test scenarios for the Customer Portal and Training LMS applications.

## Customer Portal Test Scenarios

### `portal/auth.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { WorkOSMock } from '../utils/workos-mock';
import { seedUser, seedOrganization } from '../fixtures';

test.describe('@smoke @portal Authentication', () => {
  let workos: WorkOSMock;

  test.beforeAll(async () => {
    workos = new WorkOSMock();
    await workos.start();
  });

  test.afterAll(async () => {
    await workos.stop();
  });

  test('new customer can register', async ({ page }) => {
    await page.goto('/register');

    // Fill registration form
    await page.fill('[data-testid="email"]', 'newcustomer@example.com');
    await page.fill('[data-testid="password"]', 'SecurePass123!');
    await page.fill('[data-testid="confirm-password"]', 'SecurePass123!');
    await page.fill('[data-testid="first-name"]', 'Jane');
    await page.fill('[data-testid="last-name"]', 'Doe');

    // Submit
    await page.click('[data-testid="register-button"]');

    // Verify success
    await expect(page.locator('[data-testid="verification-sent"]')).toBeVisible();
    await expect(page).toHaveURL(/\/verify-email/);
  });

  test('existing customer can login', async ({ page }) => {
    const user = await seedUser({ verified: true });

    await page.goto('/login');
    await page.fill('[data-testid="email"]', user.email);
    await page.fill('[data-testid="password"]', user.password);
    await page.click('[data-testid="login-button"]');

    // Verify redirected to dashboard
    await expect(page).toHaveURL(/\/dashboard/);
    await expect(page.locator('[data-testid="user-name"]')).toContainText(user.firstName);
  });

  test('SSO login via WorkOS', async ({ page }) => {
    const org = await seedOrganization({ ssoEnabled: true });

    await page.goto('/login');
    await page.fill('[data-testid="email"]', `user@${org.domain}`);
    await page.click('[data-testid="continue-button"]');

    // Should redirect to WorkOS
    await expect(page).toHaveURL(/api\.workos\.com/);

    // Complete SSO (mocked)
    await workos.completeSSOFlow(page, org);

    // Verify logged in
    await expect(page).toHaveURL(/\/dashboard/);
  });

  test('password reset flow works', async ({ page }) => {
    const user = await seedUser({ verified: true });

    await page.goto('/forgot-password');
    await page.fill('[data-testid="email"]', user.email);
    await page.click('[data-testid="reset-button"]');

    // Verify email sent message
    await expect(page.locator('[data-testid="reset-email-sent"]')).toBeVisible();

    // Simulate clicking reset link
    const resetToken = await workos.getResetToken(user.email);
    await page.goto(`/reset-password?token=${resetToken}`);

    // Set new password
    await page.fill('[data-testid="new-password"]', 'NewSecurePass456!');
    await page.fill('[data-testid="confirm-password"]', 'NewSecurePass456!');
    await page.click('[data-testid="update-password"]');

    // Verify success
    await expect(page.locator('[data-testid="password-updated"]')).toBeVisible();
  });
});

test.describe('@regression @portal Multi-tenant Access', () => {
  test('user sees only their organization data', async ({ browser }) => {
    const org1 = await seedOrganization({ name: 'Salon A' });
    const org2 = await seedOrganization({ name: 'Salon B' });
    const user1 = await seedUser({ organizationId: org1.id });
    const user2 = await seedUser({ organizationId: org2.id });

    // Login as user1
    const context1 = await browser.newContext();
    const page1 = await context1.newPage();
    await loginAs(page1, user1);
    await page1.goto('/dashboard');

    // Verify sees only org1 data
    await expect(page1.locator('[data-testid="org-name"]')).toContainText('Salon A');

    // Login as user2
    const context2 = await browser.newContext();
    const page2 = await context2.newPage();
    await loginAs(page2, user2);
    await page2.goto('/dashboard');

    // Verify sees only org2 data
    await expect(page2.locator('[data-testid="org-name"]')).toContainText('Salon B');
  });
});
```

### `portal/booking.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { WixMock } from '../utils/wix-mock';
import { seedUser, seedSalon, seedTherapist, seedService } from '../fixtures';
import { loginAs } from '../utils/auth';

test.describe('@smoke @portal Booking Flow', () => {
  let wixMock: WixMock;

  test.beforeAll(async () => {
    wixMock = new WixMock();
    await wixMock.start();
  });

  test('customer can search and find salons', async ({ page }) => {
    const salon = await seedSalon({ name: 'Beauty Studio', city: 'New York' });

    await loginAs(page, await seedUser());
    await page.goto('/find-salon');

    // Search by location
    await page.fill('[data-testid="location-search"]', 'New York');
    await page.click('[data-testid="search-button"]');

    // Verify results
    await expect(page.locator('[data-testid="salon-card"]')).toContainText('Beauty Studio');
  });

  test('customer can book an appointment', async ({ page }) => {
    const salon = await seedSalon();
    const therapist = await seedTherapist({ salonId: salon.id });
    const service = await seedService({ salonId: salon.id, duration: 60 });
    const user = await seedUser();

    await loginAs(page, user);
    await page.goto(`/salons/${salon.id}`);

    // Select service
    await page.click(`[data-testid="service-${service.id}"]`);

    // Select therapist
    await page.click(`[data-testid="therapist-${therapist.id}"]`);

    // Select date and time
    await page.click('[data-testid="date-next-available"]');
    await page.click('[data-testid="time-slot-1000"]');

    // Confirm booking
    await page.click('[data-testid="confirm-booking"]');

    // Verify confirmation
    await expect(page.locator('[data-testid="booking-confirmed"]')).toBeVisible();
    await expect(page.locator('[data-testid="booking-details"]')).toContainText(service.name);
  });

  test('booking syncs to Wix Bookings', async ({ page }) => {
    const salon = await seedSalon();
    const service = await seedService({ salonId: salon.id });
    const user = await seedUser();

    await loginAs(page, user);

    // Complete booking
    await page.goto(`/salons/${salon.id}`);
    await page.click(`[data-testid="service-${service.id}"]`);
    await page.click('[data-testid="date-next-available"]');
    await page.click('[data-testid="time-slot-1000"]');
    await page.click('[data-testid="confirm-booking"]');

    // Verify Wix sync
    const wixBooking = await wixMock.getLastBooking();
    expect(wixBooking.serviceId).toBe(service.wixServiceId);
    expect(wixBooking.customerEmail).toBe(user.email);
  });
});

test.describe('@regression @portal Booking Management', () => {
  test('customer can view upcoming appointments', async ({ page }) => {
    const user = await seedUser();
    const booking = await seedBooking({ userId: user.id, date: 'tomorrow' });

    await loginAs(page, user);
    await page.goto('/my-bookings');

    await expect(page.locator('[data-testid="booking-card"]')).toBeVisible();
    await expect(page.locator('[data-testid="booking-service"]')).toContainText(booking.serviceName);
  });

  test('customer can cancel appointment', async ({ page }) => {
    const user = await seedUser();
    const booking = await seedBooking({ userId: user.id, date: 'tomorrow', cancellable: true });

    await loginAs(page, user);
    await page.goto('/my-bookings');

    await page.click(`[data-testid="booking-${booking.id}-actions"]`);
    await page.click('[data-testid="cancel-booking"]');
    await page.click('[data-testid="confirm-cancel"]');

    // Verify cancelled
    await expect(page.locator(`[data-testid="booking-${booking.id}"]`)).toContainText('Cancelled');
  });

  test('customer can reschedule appointment', async ({ page }) => {
    const user = await seedUser();
    const booking = await seedBooking({ userId: user.id, date: 'tomorrow' });

    await loginAs(page, user);
    await page.goto('/my-bookings');

    await page.click(`[data-testid="booking-${booking.id}-actions"]`);
    await page.click('[data-testid="reschedule-booking"]');

    // Select new time
    await page.click('[data-testid="date-picker"]');
    await page.click('[data-testid="date-day-after-tomorrow"]');
    await page.click('[data-testid="time-slot-1400"]');
    await page.click('[data-testid="confirm-reschedule"]');

    // Verify rescheduled
    await expect(page.locator('[data-testid="reschedule-success"]')).toBeVisible();
  });
});
```

### `portal/purchase.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { StripeMock } from '../utils/stripe-mock';
import { ShopifyMock } from '../utils/shopify-mock';
import { seedUser, seedProduct, seedCart } from '../fixtures';
import { loginAs } from '../utils/auth';

test.describe('@smoke @portal Product Purchase', () => {
  let stripeMock: StripeMock;
  let shopifyMock: ShopifyMock;

  test.beforeAll(async () => {
    stripeMock = new StripeMock();
    shopifyMock = new ShopifyMock();
    await Promise.all([stripeMock.start(), shopifyMock.start()]);
  });

  test.afterAll(async () => {
    await Promise.all([stripeMock.stop(), shopifyMock.stop()]);
  });

  test('customer can browse products', async ({ page }) => {
    const products = await Promise.all([
      seedProduct({ name: 'Hydrating Serum', category: 'skincare' }),
      seedProduct({ name: 'Cleansing Foam', category: 'skincare' }),
    ]);

    await loginAs(page, await seedUser());
    await page.goto('/shop');

    // Verify products displayed
    await expect(page.locator('[data-testid="product-card"]')).toHaveCount(2);
    await expect(page.locator('text=Hydrating Serum')).toBeVisible();
    await expect(page.locator('text=Cleansing Foam')).toBeVisible();
  });

  test('customer can add to cart and checkout', async ({ page }) => {
    const product = await seedProduct({ name: 'Test Product', price: 49.99 });
    const user = await seedUser();

    await loginAs(page, user);
    await page.goto(`/shop/products/${product.id}`);

    // Add to cart
    await page.click('[data-testid="add-to-cart"]');
    await expect(page.locator('[data-testid="cart-count"]')).toContainText('1');

    // Go to checkout
    await page.click('[data-testid="cart-icon"]');
    await page.click('[data-testid="checkout-button"]');

    // Fill shipping
    await page.fill('[data-testid="address-line1"]', '123 Test St');
    await page.fill('[data-testid="city"]', 'New York');
    await page.fill('[data-testid="postal-code"]', '10001');
    await page.click('[data-testid="continue-to-payment"]');

    // Complete payment
    await stripeMock.fillCard(page, {
      number: '4242424242424242',
      expiry: '12/30',
      cvc: '123',
    });
    await page.click('[data-testid="place-order"]');

    // Verify order confirmation
    await expect(page.locator('[data-testid="order-confirmed"]')).toBeVisible();
    await expect(page.locator('[data-testid="order-total"]')).toContainText('$49.99');
  });

  test('order syncs to Shopify', async ({ page }) => {
    const product = await seedProduct({ shopifyProductId: 'gid://shopify/Product/123' });
    const user = await seedUser();

    // Complete purchase flow
    await loginAs(page, user);
    await page.goto(`/shop/products/${product.id}`);
    await page.click('[data-testid="add-to-cart"]');
    await page.click('[data-testid="cart-icon"]');
    await page.click('[data-testid="checkout-button"]');
    // ... complete checkout ...

    // Verify Shopify order created
    const shopifyOrder = await shopifyMock.getLastOrder();
    expect(shopifyOrder.email).toBe(user.email);
    expect(shopifyOrder.line_items[0].variant_id).toBe(product.shopifyVariantId);
  });
});
```

---

## Training LMS Test Scenarios

### `lms/enrollment.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { seedUser, seedCourse, seedModule } from '../fixtures';
import { loginAs } from '../utils/auth';

test.describe('@smoke @lms Course Enrollment', () => {
  test('therapist can browse course catalog', async ({ page }) => {
    const courses = await Promise.all([
      seedCourse({ title: 'Skincare Fundamentals', category: 'certification' }),
      seedCourse({ title: 'Advanced Techniques', category: 'advanced' }),
    ]);

    const therapist = await seedUser({ role: 'therapist' });
    await loginAs(page, therapist);
    await page.goto('/lms/courses');

    // Verify courses displayed
    await expect(page.locator('[data-testid="course-card"]')).toHaveCount(2);
    await expect(page.locator('text=Skincare Fundamentals')).toBeVisible();
  });

  test('therapist can enroll in a course', async ({ page }) => {
    const course = await seedCourse({
      title: 'Certification Course',
      modules: [
        await seedModule({ title: 'Module 1' }),
        await seedModule({ title: 'Module 2' }),
      ],
    });
    const therapist = await seedUser({ role: 'therapist' });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}`);

    // Enroll
    await page.click('[data-testid="enroll-button"]');

    // Verify enrollment
    await expect(page.locator('[data-testid="enrolled-badge"]')).toBeVisible();
    await expect(page.locator('[data-testid="start-learning"]')).toBeVisible();
  });

  test('enrollment initializes xAPI tracking', async ({ page }) => {
    const course = await seedCourse();
    const therapist = await seedUser({ role: 'therapist' });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}`);
    await page.click('[data-testid="enroll-button"]');

    // Start first module
    await page.click('[data-testid="start-learning"]');

    // Verify xAPI statement sent
    const xapiStatements = await page.evaluate(() =>
      (window as any).__xapi_statements || []
    );
    expect(xapiStatements).toContainEqual(
      expect.objectContaining({
        verb: { id: 'http://adlnet.gov/expapi/verbs/launched' },
      })
    );
  });
});
```

### `lms/completion.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { EventBusMonitor } from '../utils/event-bus-monitor';
import { seedUser, seedCourse, seedEnrollment, seedProgress } from '../fixtures';
import { loginAs } from '../utils/auth';

test.describe('@regression @lms Course Completion', () => {
  let eventMonitor: EventBusMonitor;

  test.beforeAll(async () => {
    eventMonitor = new EventBusMonitor();
    await eventMonitor.connect();
  });

  test('therapist can complete all modules', async ({ page }) => {
    const course = await seedCourse({ moduleCount: 3 });
    const therapist = await seedUser({ role: 'therapist' });
    const enrollment = await seedEnrollment({
      userId: therapist.id,
      courseId: course.id,
      progress: { completedModules: [course.modules[0].id, course.modules[1].id] },
    });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}/modules/${course.modules[2].id}`);

    // Complete final module
    await page.click('[data-testid="mark-complete"]');

    // Verify course completion
    await expect(page.locator('[data-testid="course-complete"]')).toBeVisible();
  });

  test('passing assessment awards certificate', async ({ page }) => {
    const course = await seedCourse({ hasAssessment: true, passingScore: 80 });
    const therapist = await seedUser({ role: 'therapist' });
    await seedEnrollment({
      userId: therapist.id,
      courseId: course.id,
      progress: { modulesComplete: true },
    });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}/assessment`);

    // Answer questions correctly (mocked for 90% score)
    for (const question of course.assessment.questions) {
      await page.click(`[data-testid="answer-${question.correctAnswer}"]`);
      await page.click('[data-testid="next-question"]');
    }

    await page.click('[data-testid="submit-assessment"]');

    // Verify certificate
    await expect(page.locator('[data-testid="assessment-passed"]')).toBeVisible();
    await expect(page.locator('[data-testid="certificate-available"]')).toBeVisible();
  });

  test('completion syncs to customer portal', async ({ page }) => {
    const course = await seedCourse();
    const therapist = await seedUser({ role: 'therapist' });
    await seedEnrollment({
      userId: therapist.id,
      courseId: course.id,
      progress: { modulesComplete: true, assessmentPassed: true },
    });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}`);

    // Claim certificate
    await page.click('[data-testid="claim-certificate"]');

    // Verify event published
    const event = await eventMonitor.waitForEvent('certification.awarded', {
      userId: therapist.id,
    });
    expect(event.payload.courseId).toBe(course.id);
    expect(event.payload.certificateId).toBeDefined();
  });
});
```

### `lms/scorm.spec.ts`

```typescript
import { test, expect } from '@playwright/test';
import { SCORMPackageMock } from '../utils/scorm-mock';
import { seedUser, seedCourse, seedSCORMContent } from '../fixtures';
import { loginAs } from '../utils/auth';

test.describe('@regression @lms SCORM Content', () => {
  let scormMock: SCORMPackageMock;

  test.beforeAll(async () => {
    scormMock = new SCORMPackageMock();
  });

  test('SCORM package loads in player', async ({ page }) => {
    const scormContent = await seedSCORMContent({ version: '2004' });
    const course = await seedCourse({ content: scormContent });
    const therapist = await seedUser({ role: 'therapist' });
    await seedEnrollment({ userId: therapist.id, courseId: course.id });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}/play`);

    // Verify SCORM player loaded
    await expect(page.frameLocator('[data-testid="scorm-player"]').locator('body')).toBeVisible();
  });

  test('SCORM progress is tracked', async ({ page }) => {
    const scormContent = await seedSCORMContent();
    const course = await seedCourse({ content: scormContent });
    const therapist = await seedUser({ role: 'therapist' });
    await seedEnrollment({ userId: therapist.id, courseId: course.id });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}/play`);

    // Simulate SCORM progress (50%)
    await scormMock.simulateProgress(page, 50);

    // Navigate away and back
    await page.goto('/lms/courses');
    await page.goto(`/lms/courses/${course.id}`);

    // Verify progress persisted
    await expect(page.locator('[data-testid="progress-bar"]')).toHaveAttribute('aria-valuenow', '50');
  });

  test('SCORM completion status is recorded', async ({ page }) => {
    const scormContent = await seedSCORMContent();
    const course = await seedCourse({ content: scormContent });
    const therapist = await seedUser({ role: 'therapist' });
    await seedEnrollment({ userId: therapist.id, courseId: course.id });

    await loginAs(page, therapist);
    await page.goto(`/lms/courses/${course.id}/play`);

    // Simulate SCORM completion
    await scormMock.simulateCompletion(page, {
      success: true,
      score: 95,
    });

    // Verify completion recorded
    await page.goto(`/lms/courses/${course.id}`);
    await expect(page.locator('[data-testid="module-status"]')).toContainText('Completed');
    await expect(page.locator('[data-testid="module-score"]')).toContainText('95%');
  });
});
```

---

## Utility Classes

### `utils/workos-mock.ts`

```typescript
export class WorkOSMock {
  private server: MockServer;

  async start() {
    this.server = await createMockServer({
      '/sso/authorize': this.handleSSOAuthorize,
      '/sso/callback': this.handleSSOCallback,
      '/user_management/password_reset': this.handlePasswordReset,
    });
  }

  async completeSSOFlow(page: Page, org: Organization) {
    // Simulate SSO completion
    await page.click('[data-testid="authorize-app"]');
  }

  async getResetToken(email: string): Promise<string> {
    return this.server.getStoredToken(email);
  }
}
```

### `utils/stripe-mock.ts`

```typescript
export class StripeMock {
  async fillCard(page: Page, card: CardDetails) {
    const frame = page.frameLocator('[data-testid="stripe-element"]');
    await frame.locator('[name="cardnumber"]').fill(card.number);
    await frame.locator('[name="exp-date"]').fill(card.expiry);
    await frame.locator('[name="cvc"]').fill(card.cvc);
  }
}
```
