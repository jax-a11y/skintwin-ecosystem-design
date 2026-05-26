# E2E Test Fixtures Specification

This document specifies the test data fixtures used across E2E test scenarios. Fixtures provide deterministic, isolated test data for reproducible test execution.

## 1. Fixture Design Principles

### 1.1 Requirements

- **Deterministic**: Same input produces same output
- **Isolated**: No dependencies on external state
- **Comprehensive**: Cover all required entity relationships
- **Realistic**: Mirror production data patterns
- **Minimal**: Only include data needed for tests

### 1.2 Fixture Hierarchy

```
fixtures/
├── index.ts              # Main export + seed functions
├── tenants/              # Organization fixtures
├── users/                # User account fixtures
├── products/             # Product catalog fixtures
├── bookings/             # Appointment fixtures
├── courses/              # LMS content fixtures
├── erp/                  # ERP mapping fixtures
└── shopify/              # Shopify-specific fixtures
```

## 2. Core Fixtures

### 2.1 Tenant/Organization Fixtures

#### `tenants/salon_basic.json`
```json
{
  "id": "org_salon_basic",
  "name": "Beauty Studio",
  "type": "salon",
  "plan": "basic",
  "settings": {
    "timezone": "America/New_York",
    "currency": "USD",
    "locale": "en-US"
  },
  "address": {
    "line1": "123 Main Street",
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "US"
  },
  "contact": {
    "email": "contact@beautystudio.test",
    "phone": "+1-555-0100"
  },
  "integrations": {
    "shopify": {
      "shopDomain": "beauty-studio.myshopify.com",
      "accessToken": "test_token_basic"
    },
    "wix": null,
    "erp": null
  },
  "createdAt": "2024-01-01T00:00:00Z"
}
```

#### `tenants/salon_enterprise.json`
```json
{
  "id": "org_salon_enterprise",
  "name": "Luxe Beauty Group",
  "type": "salon_chain",
  "plan": "enterprise",
  "settings": {
    "timezone": "America/Los_Angeles",
    "currency": "USD",
    "locale": "en-US",
    "multiLocation": true,
    "ssoEnabled": true
  },
  "address": {
    "line1": "500 Corporate Drive",
    "city": "Los Angeles",
    "state": "CA",
    "postalCode": "90001",
    "country": "US"
  },
  "locations": [
    {
      "id": "loc_1",
      "name": "Downtown LA",
      "address": { "city": "Los Angeles", "state": "CA" }
    },
    {
      "id": "loc_2",
      "name": "Beverly Hills",
      "address": { "city": "Beverly Hills", "state": "CA" }
    }
  ],
  "integrations": {
    "shopify": {
      "shopDomain": "luxe-beauty.myshopify.com",
      "accessToken": "test_token_enterprise"
    },
    "wix": {
      "siteId": "wix_site_luxe",
      "apiKey": "test_wix_key"
    },
    "erp": {
      "provider": "quickbooks",
      "companyId": "qb_123",
      "syncEnabled": true
    }
  },
  "sso": {
    "provider": "workos",
    "connectionId": "conn_workos_luxe",
    "domain": "luxebeauty.com"
  },
  "createdAt": "2023-06-15T00:00:00Z"
}
```

### 2.2 User Fixtures

#### `users/customer_basic.json`
```json
{
  "id": "usr_customer_basic",
  "email": "jane.doe@example.test",
  "firstName": "Jane",
  "lastName": "Doe",
  "role": "customer",
  "verified": true,
  "profile": {
    "phone": "+1-555-0101",
    "dateOfBirth": "1990-05-15",
    "skinType": "combination",
    "concerns": ["aging", "hydration"]
  },
  "preferences": {
    "notifications": {
      "email": true,
      "sms": true,
      "marketing": false
    },
    "locale": "en-US"
  },
  "password": "TestPassword123!",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

#### `users/therapist_certified.json`
```json
{
  "id": "usr_therapist_certified",
  "email": "sarah.smith@beautystudio.test",
  "firstName": "Sarah",
  "lastName": "Smith",
  "role": "therapist",
  "organizationId": "org_salon_basic",
  "verified": true,
  "profile": {
    "phone": "+1-555-0102",
    "bio": "Certified skincare specialist with 10 years of experience.",
    "specialties": ["facials", "chemical_peels", "microdermabrasion"],
    "avatar": "https://cdn.example.test/avatars/sarah.jpg"
  },
  "certifications": [
    {
      "id": "cert_skincare_fundamentals",
      "name": "Skincare Fundamentals",
      "issuedAt": "2023-03-01",
      "expiresAt": "2025-03-01"
    },
    {
      "id": "cert_advanced_techniques",
      "name": "Advanced Treatment Techniques",
      "issuedAt": "2023-09-15",
      "expiresAt": "2025-09-15"
    }
  ],
  "schedule": {
    "timezone": "America/New_York",
    "workingHours": {
      "monday": { "start": "09:00", "end": "17:00" },
      "tuesday": { "start": "09:00", "end": "17:00" },
      "wednesday": { "start": "09:00", "end": "17:00" },
      "thursday": { "start": "09:00", "end": "17:00" },
      "friday": { "start": "09:00", "end": "15:00" }
    }
  },
  "password": "TherapistPass456!",
  "createdAt": "2023-01-10T08:00:00Z"
}
```

#### `users/admin_super.json`
```json
{
  "id": "usr_admin_super",
  "email": "admin@skintwin.test",
  "firstName": "Admin",
  "lastName": "User",
  "role": "admin",
  "permissions": ["*"],
  "verified": true,
  "mfaEnabled": true,
  "password": "AdminSecure789!",
  "createdAt": "2023-01-01T00:00:00Z"
}
```

### 2.3 Product Fixtures

#### `products/skincare_line.json`
```json
{
  "products": [
    {
      "id": "prod_hydrating_serum",
      "sku": "SKU-HYD-001",
      "name": "Hydrating Hyaluronic Serum",
      "description": "Intense hydration with 3 types of hyaluronic acid.",
      "category": "serums",
      "price": 79.99,
      "compareAtPrice": 99.99,
      "inventory": 150,
      "images": [
        "https://cdn.example.test/products/hydrating-serum-1.jpg"
      ],
      "variants": [
        { "id": "var_hyd_30ml", "size": "30ml", "price": 79.99, "sku": "SKU-HYD-001-30" },
        { "id": "var_hyd_50ml", "size": "50ml", "price": 119.99, "sku": "SKU-HYD-001-50" }
      ],
      "shopify": {
        "productId": "gid://shopify/Product/1001",
        "variantIds": {
          "var_hyd_30ml": "gid://shopify/ProductVariant/2001",
          "var_hyd_50ml": "gid://shopify/ProductVariant/2002"
        }
      },
      "erp": {
        "itemCode": "ERP-HYD-001",
        "costPrice": 25.00
      }
    },
    {
      "id": "prod_vitamin_c",
      "sku": "SKU-VIT-001",
      "name": "Brightening Vitamin C Serum",
      "description": "20% L-Ascorbic Acid for radiant skin.",
      "category": "serums",
      "price": 89.99,
      "inventory": 200,
      "images": [
        "https://cdn.example.test/products/vitamin-c-1.jpg"
      ],
      "variants": [
        { "id": "var_vitc_30ml", "size": "30ml", "price": 89.99, "sku": "SKU-VIT-001-30" }
      ],
      "shopify": {
        "productId": "gid://shopify/Product/1002",
        "variantIds": {
          "var_vitc_30ml": "gid://shopify/ProductVariant/2003"
        }
      }
    },
    {
      "id": "prod_cleanser",
      "sku": "SKU-CLN-001",
      "name": "Gentle Foaming Cleanser",
      "description": "Sulfate-free cleanser for all skin types.",
      "category": "cleansers",
      "price": 34.99,
      "inventory": 300,
      "variants": [
        { "id": "var_cln_150ml", "size": "150ml", "price": 34.99, "sku": "SKU-CLN-001-150" }
      ]
    }
  ]
}
```

### 2.4 Booking Fixtures

#### `bookings/upcoming_appointments.json`
```json
{
  "bookings": [
    {
      "id": "book_001",
      "customerId": "usr_customer_basic",
      "therapistId": "usr_therapist_certified",
      "organizationId": "org_salon_basic",
      "service": {
        "id": "svc_facial_classic",
        "name": "Classic Facial",
        "duration": 60,
        "price": 85.00
      },
      "scheduledAt": "{{tomorrow_10am}}",
      "status": "confirmed",
      "notes": "First-time client, sensitive skin",
      "wix": {
        "bookingId": "wix_book_001"
      },
      "createdAt": "{{today_minus_2days}}"
    },
    {
      "id": "book_002",
      "customerId": "usr_customer_basic",
      "therapistId": "usr_therapist_certified",
      "organizationId": "org_salon_basic",
      "service": {
        "id": "svc_peel_glycolic",
        "name": "Glycolic Peel",
        "duration": 45,
        "price": 120.00
      },
      "scheduledAt": "{{next_week_14pm}}",
      "status": "pending",
      "createdAt": "{{today}}"
    }
  ]
}
```

### 2.5 LMS Course Fixtures

#### `courses/certification_course.json`
```json
{
  "id": "course_skincare_cert",
  "title": "Skincare Fundamentals Certification",
  "description": "Comprehensive training on skincare theory and techniques.",
  "category": "certification",
  "level": "beginner",
  "duration": 480,
  "modules": [
    {
      "id": "mod_intro",
      "title": "Introduction to Skincare",
      "description": "Overview of skin anatomy and function.",
      "order": 1,
      "content": {
        "type": "video",
        "url": "https://cdn.example.test/lms/intro.mp4",
        "duration": 30
      },
      "quiz": {
        "questions": [
          {
            "id": "q1",
            "text": "What is the outermost layer of the skin?",
            "options": ["Dermis", "Epidermis", "Hypodermis", "Subcutis"],
            "correctAnswer": 1
          }
        ],
        "passingScore": 80
      }
    },
    {
      "id": "mod_skin_types",
      "title": "Understanding Skin Types",
      "order": 2,
      "content": {
        "type": "video",
        "url": "https://cdn.example.test/lms/skin-types.mp4",
        "duration": 45
      }
    },
    {
      "id": "mod_ingredients",
      "title": "Active Ingredients",
      "order": 3,
      "content": {
        "type": "interactive",
        "scormPackage": "ingredients-scorm.zip"
      }
    }
  ],
  "assessment": {
    "type": "final_exam",
    "questions": 50,
    "passingScore": 75,
    "timeLimit": 60,
    "attempts": 3
  },
  "certificate": {
    "template": "cert_template_standard",
    "validityPeriod": 730
  },
  "pricing": {
    "type": "one_time",
    "price": 299.00
  },
  "xapi": {
    "activityId": "https://skintwin.ai/courses/skincare-fundamentals"
  }
}
```

### 2.6 ERP Fixtures

#### `erp/erp_products.json`
```json
{
  "items": [
    {
      "itemCode": "ERP-HYD-001",
      "description": "Hydrating Serum 30ml",
      "category": "Finished Goods",
      "uom": "EA",
      "costPrice": 25.00,
      "listPrice": 79.99,
      "taxCode": "TAXABLE",
      "glAccounts": {
        "revenue": "4000-PRODUCT-REV",
        "cogs": "5000-COGS",
        "inventory": "1300-INVENTORY"
      },
      "vendors": [
        {
          "vendorCode": "VEND-001",
          "vendorSku": "VS-HYD-30",
          "leadTime": 14,
          "moq": 100
        }
      ]
    },
    {
      "itemCode": "ERP-VIT-001",
      "description": "Vitamin C Serum 30ml",
      "category": "Finished Goods",
      "uom": "EA",
      "costPrice": 30.00,
      "listPrice": 89.99,
      "taxCode": "TAXABLE"
    }
  ]
}
```

#### `erp/erp_inventory.json`
```json
{
  "warehouses": [
    {
      "code": "WH-MAIN",
      "name": "Main Warehouse",
      "location": "New York, NY"
    },
    {
      "code": "WH-WEST",
      "name": "West Coast DC",
      "location": "Los Angeles, CA"
    }
  ],
  "inventory": [
    {
      "itemCode": "ERP-HYD-001",
      "levels": [
        { "warehouse": "WH-MAIN", "onHand": 100, "allocated": 10, "available": 90 },
        { "warehouse": "WH-WEST", "onHand": 50, "allocated": 5, "available": 45 }
      ],
      "reorderPoint": 50,
      "reorderQty": 100
    },
    {
      "itemCode": "ERP-VIT-001",
      "levels": [
        { "warehouse": "WH-MAIN", "onHand": 150, "allocated": 0, "available": 150 }
      ],
      "reorderPoint": 75,
      "reorderQty": 150
    }
  ]
}
```

## 3. Fixture Loading

### `index.ts`
```typescript
import { readFileSync } from 'fs';
import { join } from 'path';

const FIXTURES_DIR = __dirname;

type FixtureName = 
  | 'salon_basic' 
  | 'salon_enterprise' 
  | 'customer_basic' 
  | 'therapist_certified'
  | 'admin_super'
  | 'skincare_line'
  | 'certification_course';

const FIXTURE_PATHS: Record<FixtureName, string> = {
  salon_basic: 'tenants/salon_basic.json',
  salon_enterprise: 'tenants/salon_enterprise.json',
  customer_basic: 'users/customer_basic.json',
  therapist_certified: 'users/therapist_certified.json',
  admin_super: 'users/admin_super.json',
  skincare_line: 'products/skincare_line.json',
  certification_course: 'courses/certification_course.json',
};

export function loadFixture<T>(name: FixtureName): T {
  const path = join(FIXTURES_DIR, FIXTURE_PATHS[name]);
  const content = readFileSync(path, 'utf-8');
  const data = JSON.parse(content);
  return processTemplates(data);
}

function processTemplates(data: any): any {
  const json = JSON.stringify(data);
  const processed = json
    .replace(/\{\{tomorrow_10am\}\}/g, getTomorrow10am())
    .replace(/\{\{next_week_14pm\}\}/g, getNextWeek14pm())
    .replace(/\{\{today\}\}/g, new Date().toISOString())
    .replace(/\{\{today_minus_2days\}\}/g, getDateOffset(-2));
  return JSON.parse(processed);
}

// Seed functions for database
export async function seedDatabase(config: {
  tenants?: FixtureName[];
  users?: FixtureName[];
  products?: FixtureName[];
}) {
  const db = await getTestDatabase();
  
  for (const tenant of config.tenants || []) {
    await db.tenants.insert(loadFixture(tenant));
  }
  
  for (const user of config.users || []) {
    await db.users.insert(loadFixture(user));
  }
  
  for (const product of config.products || []) {
    const data = loadFixture<{ products: any[] }>(product);
    await db.products.insertMany(data.products);
  }
}

export async function cleanupDatabase() {
  const db = await getTestDatabase();
  await db.truncateAll();
}

// Individual seed helpers
export async function seedUser(overrides?: Partial<User>): Promise<User> {
  const base = loadFixture<User>('customer_basic');
  const user = { ...base, id: `usr_${Date.now()}`, ...overrides };
  await getTestDatabase().then(db => db.users.insert(user));
  return user;
}

export async function seedProduct(overrides?: Partial<Product>): Promise<Product> {
  const base = loadFixture<{ products: Product[] }>('skincare_line').products[0];
  const product = { ...base, id: `prod_${Date.now()}`, ...overrides };
  await getTestDatabase().then(db => db.products.insert(product));
  return product;
}

export async function seedOrganization(overrides?: Partial<Organization>): Promise<Organization> {
  const base = loadFixture<Organization>('salon_basic');
  const org = { ...base, id: `org_${Date.now()}`, ...overrides };
  await getTestDatabase().then(db => db.organizations.insert(org));
  return org;
}

export async function seedCourse(overrides?: Partial<Course>): Promise<Course> {
  const base = loadFixture<Course>('certification_course');
  const course = { ...base, id: `course_${Date.now()}`, ...overrides };
  await getTestDatabase().then(db => db.courses.insert(course));
  return course;
}
```

## 4. Dynamic Data Generation

### `generators.ts`
```typescript
import { faker } from '@faker-js/faker';

export function generateUser(role: 'customer' | 'therapist' | 'admin' = 'customer') {
  return {
    id: `usr_${faker.string.nanoid()}`,
    email: faker.internet.email(),
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    role,
    verified: true,
    password: 'TestPassword123!',
    createdAt: faker.date.past().toISOString(),
  };
}

export function generateProduct(category: string = 'skincare') {
  const name = faker.commerce.productName();
  const sku = `SKU-${faker.string.alphanumeric(6).toUpperCase()}`;
  
  return {
    id: `prod_${faker.string.nanoid()}`,
    sku,
    name,
    description: faker.commerce.productDescription(),
    category,
    price: parseFloat(faker.commerce.price({ min: 20, max: 200 })),
    inventory: faker.number.int({ min: 50, max: 500 }),
    createdAt: faker.date.past().toISOString(),
  };
}

export function generateOrder(customerId: string, items: OrderItem[]) {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const tax = subtotal * 0.08;
  const total = subtotal + tax;
  
  return {
    id: `ord_${faker.string.nanoid()}`,
    customerId,
    items,
    subtotal,
    tax,
    total,
    status: 'pending',
    createdAt: new Date().toISOString(),
  };
}

export function generateBooking(customerId: string, therapistId: string, serviceId: string) {
  const scheduledAt = faker.date.soon({ days: 14 });
  
  return {
    id: `book_${faker.string.nanoid()}`,
    customerId,
    therapistId,
    serviceId,
    scheduledAt: scheduledAt.toISOString(),
    status: 'confirmed',
    createdAt: new Date().toISOString(),
  };
}
```

## 5. Environment-Specific Configuration

### `config.ts`
```typescript
export const fixtureConfig = {
  preview: {
    database: 'skintwin_test',
    truncateBeforeEach: true,
    useMocks: true,
  },
  staging: {
    database: 'skintwin_staging',
    truncateBeforeEach: false,
    useMocks: false,
    seedOnce: true,
  },
  production: {
    database: null, // Synthetic users only
    truncateBeforeEach: false,
    useMocks: false,
    syntheticUsersOnly: true,
  },
};
```
