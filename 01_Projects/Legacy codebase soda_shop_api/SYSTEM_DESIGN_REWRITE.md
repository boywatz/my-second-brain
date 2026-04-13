# เอกสารการออกแบบระบบ: SODA BKK API Rewrite
## System Design Document: SODA BKK API Rewrite Project

**เอกสารเวอร์ชัน:** 1.0  
**วันที่เขียน:** 12 เมษายน 2566 (2026-04-12)  
**สถานะ:** Design Phase  
**โปรเจคต์:** Soda Shop API — Cloudflare Stack Migration + Hexagonal Architecture  

---

## สารบัญ / Table of Contents

1. [ภาพรวมสถาปัตยกรรม](#ภาพรวมสถาปัตยกรรม)
2. [การตัดสินใจ Cloudflare Stack](#การตัดสินใจ-cloudflare-stack)
3. [กลยุทธ์การออกแบบฐานข้อมูล](#กลยุทธ์การออกแบบฐานข้อมูล)
4. [การออกแบบ Queue](#การออกแบบ-queue)
5. [API Gateway Pattern](#api-gateway-pattern)
6. [การปรับปรุงความเป็นไปได้ด้านความปลอดภัย](#การปรับปรุงความเป็นไปได้ด้านความปลอดภัย)
7. [กลยุทธ์การ Migration](#กลยุทธ์การอพเกรด)
8. [หัวข้ออื่นที่ควรพิจารณา](#หัวข้ออื่นที่ควรพิจารณา)
   - 8.1 Observability & Monitoring (Cloudflare Native Stack)
   - 8.2 Testing Strategy (Vitest)
   - 8.3 Local Development (Wrangler gotchas)
   - 8.4 CPU Time Limit & Long-Running Tasks
   - 8.5 Cost Estimation
   - 8.6 CI/CD Pipeline
   - 8.7 Image Optimization
   - 8.8 Full-Text Search
   - 8.9 Internationalization
   - 8.10 Type-Safe API Client (Hono RPC)
   - 8.11 Drizzle ORM
   - 8.12 Error Handling Strategy
   - 8.13 Input Validation (Zod)
   - 8.14 Product SEO (Backend Responsibilities)
9. [สรุปเทคโนโลยีที่แนะนำ](#สรุปเทคโนโลยีที่แนะนำ)
10. [ความเสี่ยงและคำถามที่ยังเปิดอยู่](#ความเสี่ยงและคำถามที่ยังเปิดอยู่)
11. [Authentication & Authorization Design](#authentication--authorization-design)
    - 11.1 Auth Provider Selection (Better Auth vs Clerk vs Auth.js)
    - 11.2 User Schema (Better Auth auto-managed tables)
    - 11.3 Authentication Flows (Better Auth: Login, Register, Logout, Password Change)
    - 11.4 Session Management (Better Auth DB-backed sessions + KV edge cache)
    - 11.5 RBAC — Better Auth plugin + 4 Roles + Permissions Matrix
    - 11.6 Anonymous Checkout Flow
    - 11.7 Auth Middleware Stack (Session-based)
    - 11.8 Route Protection Matrix
    - 11.9 Security Improvements vs Legacy

---

## ภาพรวมสถาปัตยกรรม

### 1.1 โครงสร้าง Monorepo

ระบบใหม่จะใช้ **Turborepo** เป็นตัวจัดการ monorepo ที่จะบรรจุไว้ 3 แอปพลิเคชันหลัก:

```
soda-bkk-monorepo/
├── packages/
│   ├── backend/                    # Hono API Backend
│   │   ├── src/
│   │   │   ├── domain/             # Business Logic (pure)
│   │   │   ├── application/        # Use Cases & Orchestration
│   │   │   ├── ports/              # Interfaces (Repository, Service contracts)
│   │   │   ├── adapters/           # Implementation (Cloudflare, DB, Payment)
│   │   │   ├── middleware/         # Auth, Rate-limit, Logging
│   │   │   └── routes.ts           # Hono route definitions
│   │   ├── wrangler.toml           # Cloudflare Workers config
│   │   └── vitest.config.ts
│   │
│   ├── web-storefront/             # E-commerce Frontend (React/Next.js)
│   │   └── ...
│   │
│   ├── web-backoffice/             # Back Office Frontend (React)
│   │   └── ...
│   │
│   └── shared/                      # Type definitions, constants
│       ├── types/
│       ├── utils/
│       └── api-client/             # Hono RPC client for type-safe calls
│
├── turbo.json
├── pnpm-workspace.yaml
└── README.md
```

**เหตุผล:** Turborepo ใช้ได้ดีกับ Cloudflare Workers (Fast refresh, Task caching) และช่วยให้ Frontend แชร์ Type definitions กับ Backend ผ่าน Hono RPC mode

---

### 1.2 Hexagonal Architecture (Ports & Adapters)

ประยุกต์ใช้ Hexagonal Architecture ให้เข้ากับโดเมน e-commerce:

```
┌─────────────────────────────────────────────────────┐
│                   OUTER RING (Adapters)             │
│  ┌─────────────────────────────────────────────┐    │
│  │  HTTP (Hono)  │ DB (D1/Drizzle) │ Queue   │    │
│  │  Storage (R2) │ Payment (Omise)  │ KV      │    │
│  └─────────────────────────────────────────────┘    │
│                         ▲                            │
│                    (Implements)                      │
│                         │                            │
├─────────────────────────┼──────────────────────────┤
│            PORTS (Interfaces)                       │
│  ┌─────────────────────────────────────────────┐    │
│  │ IProductRepository  │ IOrderService          │    │
│  │ IPaymentGateway     │ IQueueProducer         │    │
│  │ IFileStorage        │ IAuthenticator         │    │
│  └─────────────────────────────────────────────┘    │
│                         ▲                            │
│                    (uses)                            │
│                         │                            │
├─────────────────────────┼──────────────────────────┤
│          INNER RING (Domain & Application)         │
│  ┌─────────────────────────────────────────────┐    │
│  │  Entities:                                   │    │
│  │    - Product, ProductVariant                │    │
│  │    - Order, OrderItem                       │    │
│  │    - User, Cart                             │    │
│  │    - Payment, Transaction                   │    │
│  │                                              │    │
│  │  Use Cases:                                 │    │
│  │    - CreateOrder, ProcessPayment            │    │
│  │    - UpdateStock, FulfillOrder              │    │
│  │    - CatalogSearch, GetProductDetails       │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**ประโยชน์:**
- **ความเป็นอิสระ:** Domain logic ไม่ขึ้นกับ Cloudflare Workers runtime
- **ความสามารถในการทดสอบ:** Ports เป็น interfaces — ใช้ mock adapters ในการทดสอบ
- **ความยืดหยุ่น:** สามารถสลับ D1 ↔ PostgreSQL โดยไม่เปลี่ยน Domain logic

**โมดูล/โดเมนหลัก:**
1. **Catalog Module** — Product, ProductVariant, Category
2. **Order Module** — Order, OrderItem, Cart
3. **Payment Module** — Payment processing, Transaction audit
4. **User Module** — Authentication, Authorization, Profile
5. **Inventory Module** — Stock management, Reservations
6. **Fulfillment Module** — Shipping, Status tracking

---

### 1.3 Hono ในบริบท Hexagonal Architecture

Hono ทำหน้าที่เป็น **HTTP Adapter** — ตัวสร้าง endpoint HTTP ที่เรียก use cases:

```typescript
// adapters/http/hono-routes.ts
import { Hono } from 'hono';
import { CreateOrderUseCase } from '../../application/CreateOrderUseCase';
import { OrderRepositoryD1 } from '../persistence/OrderRepositoryD1';

export function setupOrderRoutes(app: Hono, ctx: ExecutionContext) {
  const orderRepo = new OrderRepositoryD1(ctx.env.DB);
  const createOrderUseCase = new CreateOrderUseCase(orderRepo, paymentGateway);

  app.post('/api/orders', async (c) => {
    const input = await c.req.json();
    const result = await createOrderUseCase.execute(input);
    return c.json(result);
  });
}
```

**ทำไม Hono?**
- Native support สำหรับ Cloudflare Workers
- Type-safe routing + RPC mode (ใช้ได้กับ Frontend)
- Lightweight middleware stack
- Edge-first design

---

## การตัดสินใจ Cloudflare Stack

### 2.1 Runtime: Cloudflare Workers vs Container

| เกณฑ | Cloudflare Workers | Container (DigitalOcean) |
|------|-------------------|-------------------------|
| **Cold Start** | ~0ms (V8 isolate) | 500ms - 2s |
| **Memory Limit** | 128MB (free), 256MB (workers-ai) | ไม่จำกัด |
| **CPU Time** | 30s (workers.dev), unlimited (workers named routes) | ไม่จำกัด |
| **Cost** | ฟรี: 100k reqs/day, $0.50/1M reqs | $5-10/month+ |
| **Scaling** | Instant (geographic) | Manual/Auto scaling |
| **Vendor Lock-in** | Medium (Workers runtime) | Low |
| **Local Dev** | `wrangler dev` | Docker |

**แนะนำ:** ✅ **Cloudflare Workers**

**เหตุผล:**
- SODA BKK มีปริมาณ traffic ปานกลาง (ไม่ใช่ high-concurrency e-commerce)
- 128MB memory เพียงพอสำหรับ Node.js runtime + Hono + GORM queries
- CPU time limit 30s ใช้ได้สำหรับ 99% ของ requests (ยกเว้น report generation)
- ไม่ต้องบริหาร infrastructure — เหมาะสำหรับทีมเล็ก
- Geographic distribution (กรุงเทพ ↔ ทั่วโลก) เสริมด้วย Cloudflare CDN

**ข้อ caveat:**
- Report generation (PDF) ที่ใช้ CPU มาก → offload to **Cloudflare Queues** + dedicated worker
- Large file uploads → อาจต้อง chunking + R2 presigned URLs

---

### 2.2 Database: D1 vs Hyperdrive vs Neon

#### Option A: Cloudflare D1 (SQLite-based)

**Pros:**
- ฟรี tier: 100k reads/writes per day
- Built-in replication (backups)
- Type-safe with Drizzle ORM
- No additional service to manage

**Cons:**
- SQLite มีข้อจำกัด (no JSON query functions native)
- Scaling: แต่ละ region มี local SQLite copy
- Complex aggregations อาจช้า

**ความเหมาะสม:** ✅ ต้องการ schema ที่ simple, ปริมาณ data < 10GB

---

#### Option B: Hyperdrive + Neon (PostgreSQL proxy)

**Pros:**
- PostgreSQL power (JSON, full-text search, advanced functions)
- Connection pooling through Hyperdrive
- Neon free tier: 3GB storage, 20k compute units

**Cons:**
- ต้องจ่ายเพิ่มเติมสำหรับ Neon
- Hyperdrive routing (latency +10-50ms)
- Connection pooling setup complexity

**ความเหมาะสม:** ✅ ต้องการ advanced query, complex schema

---

#### Option C: Traditional PostgreSQL + Hyperdrive

**Cons:**
- ต้องรักษา PostgreSQL เอง (DigitalOcean managed/$12+)
- ไม่ได้ประโยชน์จาก Cloudflare ecosystem

---

### 2.3 แนะนำ: D1 + Drizzle ORM

**ทำไม D1?**
1. **Cost-effective:** ฟรี tier ใช้ได้นาน ก่อนต้องจ่ายโดยแท้
2. **Schema Re-design:** เมื่อเขียน schema ใหม่ โดย denormalize ที่เหมาะสม → ลด JOIN complexity
3. **Type Safety:** Drizzle ORM ทำ SQL generation type-safe (ลด SQL injection risk)
4. **No schema migration overhead:** SQLite migration ง่ายกว่า PostgreSQL ใน Cloudflare context

**Trade-off:** ถ้าในอนาคตต้องการ full PostgreSQL feature → ใช้ Hyperdrive + Neon (ไม่ต้องเขียน adapter code ใหม่)

---

### 2.4 Storage: Cloudflare R2

**แนะนำ:** ✅ **Cloudflare R2** แทน Google Cloud Storage

**Pros:**
- ฟรี: 10GB storage/month, unlimited egress
- S3-compatible API → ย้ายจาก GCS ได้ง่าย
- Built-in CDN (no egress fee ให้ Cloudflare Images)
- Integration with Cloudflare transform workers

**Cons:**
- ไม่มี advanced query (vs GCS BigQuery)
- Regional availability (เมื่อเทียบกับ GCS)

**Implementation:**
```typescript
// adapters/storage/R2StorageAdapter.ts
interface IFileStorage {
  uploadFile(key: string, buffer: Buffer): Promise<string>;
  deleteFile(key: string): Promise<void>;
  getSignedUrl(key: string, expirySeconds: number): Promise<string>;
}

export class R2StorageAdapter implements IFileStorage {
  constructor(private r2: R2Bucket) {}

  async uploadFile(key: string, buffer: Buffer): Promise<string> {
    await this.r2.put(key, buffer);
    return `https://{r2-custom-domain}/${key}`;
  }
}
```

---

### 2.5 Queue: Cloudflare Queues

**Pros:**
- ฟรี: 1M messages/month
- Built-in producer/consumer pattern
- Integration with Workers
- Retry + dead-letter queue built-in

**Cons:**
- ฟรี tier: 1M/month → ~33k/day (เพียงพอสำหรับ SODA BKK)
- Message size limit: 128KB (โอเค สำหรับ order data)

**Cost Estimate:**
- 100 orders/day × 5 queue messages/order = 500 messages/day
- 500 × 30 = 15,000/month = ฟรี tier

**แนะนำ:** ✅ **Cloudflare Queues** (ไม่ต้องจ่ายเงิน ในระยะแรก)

---

### 2.6 API Gateway: Cloudflare Workers (Gateway Pattern)

**Option:** Single entry-point Worker vs Cloudflare API Gateway product

**แนะนำ:** ✅ **Cloudflare Workers as Gateway** (ประหยัดค่า)

ออกแบบ gateway worker ที่ router requests ไปยัง domain-specific workers:

```typescript
// gateway-worker/src/gateway.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { bearerAuth } from 'hono/bearer-auth';
import { setupOrderRoutes } from './routes/orders';
import { setupProductRoutes } from './routes/products';

const app = new Hono<{ Bindings: CloudflareEnv }>();

// Gateway-level middleware
app.use('*', cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') }));
app.use('/api/*', bearerAuth({ token: '' })); // Validate JWT

// Route to domain workers
app.route('/api/products', productRouter);
app.route('/api/orders', orderRouter);
app.route('/api/payments', paymentRouter);

// Webhook signature verification at gateway level (ปิด security gap!)
app.post('/webhooks/omise', omiseWebhookHandler);

export default app;
```

**Benefits:**
- ✅ Single auth layer
- ✅ Centralized rate-limiting
- ✅ Webhook signature verification (ปัญหาเดิมแก้ได้)
- ✅ CORS configuration per origin
- ✅ Request logging/monitoring

---

## กลยุทธ์การออกแบบฐานข้อมูล

### 3.1 ปัญหาของระบบเดิม

Current system (Go + GORM):
- God controller (`content_controller.go` 639 lines) → multiple concerns mixed
- Heavy preload chains (8-12 nested associations) → N+1 query problem
- Stock atomicity bug → order creation ไม่ lock stock
- Monetary values mixed float32/float64 → precision issues
- No pagination on product catalog → ต้อง load หลายพัน records

### 3.2 Schema Redesign Strategy

**เลือก Option B: Hybrid Approach** (Relational + JSON columns)

#### Schema Principles:

1. **Snapshot Pattern for Orders**
   ```sql
   -- ลบ FK reference ไปยัง products
   -- แทนที่เก็บ snapshot ของ price/name ไว้ใน order_items
   CREATE TABLE order_items (
     id TEXT PRIMARY KEY,
     order_id TEXT NOT NULL,
     -- Snapshot fields (ไม่ reference product)
     product_snapshot JSONB NOT NULL,  -- { sku, name, price, color, size }
     quantity INTEGER NOT NULL,
     unit_price DECIMAL(12, 2) NOT NULL,
     created_at TIMESTAMP DEFAULT now()
   );
   ```

2. **Product Catalog with JSON Variants**
   ```sql
   CREATE TABLE products (
     id TEXT PRIMARY KEY,
     name TEXT NOT NULL,
     description TEXT,
     category_id TEXT,
     base_price DECIMAL(12, 2),
     -- Embed variants + gallery in JSON to avoid joins
     variants JSONB NOT NULL,  -- [{ sku, color, size, stock }, ...]
     gallery JSONB NOT NULL,   -- [{ url, alt }, ...]
     metadata JSONB,           -- tags, attributes, etc.
     
     -- SEO fields (Product SEO)
     slug TEXT NOT NULL UNIQUE,          -- URL-friendly: "oversized-tee-black"
     meta_title TEXT,                     -- <title> tag (ถ้า NULL ใช้ name)
     meta_description TEXT,              -- <meta description> (ถ้า NULL ใช้ description ตัดสั้น)
     
     -- Status
     status TEXT NOT NULL DEFAULT 'draft',  -- draft, active, archived
     
     created_at TIMESTAMP,
     updated_at TIMESTAMP
   );
   
   CREATE UNIQUE INDEX idx_products_slug ON products(slug);
   ```
   
   **Slug History** (สำหรับ 301 redirect เมื่อเปลี่ยน URL):
   ```sql
   CREATE TABLE slug_redirects (
     old_slug TEXT NOT NULL,
     new_slug TEXT NOT NULL,
     entity_type TEXT NOT NULL,           -- 'product', 'collection', 'category'
     entity_id TEXT NOT NULL,
     created_at TIMESTAMP DEFAULT (datetime('now'))
   );
   
   CREATE UNIQUE INDEX idx_slug_redirects_old ON slug_redirects(entity_type, old_slug);
   ```

3. **Stock Management (Atomic with Order)**
   ```sql
   CREATE TABLE stock_reservations (
     id TEXT PRIMARY KEY,
     order_id TEXT UNIQUE,
     product_sku TEXT NOT NULL,
     reserved_quantity INTEGER NOT NULL,
     -- Atomic constraint: only 1 active reservation per SKU
     created_at TIMESTAMP,
     released_at TIMESTAMP -- NULL if still reserved
   );
   -- ทำให้ stock atomicity ได้ แม้ SQLite ไม่มี transaction isolation level control
   ```

4. **User & Authentication**
   ```sql
   CREATE TABLE users (
     id TEXT PRIMARY KEY,
     email TEXT UNIQUE NOT NULL,
     phone TEXT,
     password_hash TEXT,
     role TEXT DEFAULT 'customer',  -- customer, admin, staff
     profile JSONB,  -- { firstName, lastName, avatar }
     created_at TIMESTAMP,
     updated_at TIMESTAMP
   );
   ```

5. **Payment with Decimal Precision**
   ```sql
   CREATE TABLE payments (
     id TEXT PRIMARY KEY,
     order_id TEXT NOT NULL,
     amount DECIMAL(12, 2) NOT NULL,  -- Always use DECIMAL, NOT float
     currency TEXT DEFAULT 'THB',
     status TEXT,  -- pending, succeeded, failed
     gateway_ref TEXT,  -- Omise charge ID
     metadata JSONB,  -- gateway response
     created_at TIMESTAMP,
     updated_at TIMESTAMP
   );
   ```

### 3.3 Query Pattern Changes

**OLD (Go + GORM — 12 joins):**
```go
// god_controller.go
product.Preload("Category").
  Preload("Variants.Images").
  Preload("Tags").
  Preload("Inventory").
  Preload("Reviews.User").
  ...
```

**NEW (D1 + Drizzle):**
```typescript
// application/GetProductDetailsUseCase.ts
async execute(productId: string) {
  const product = await this.productRepo.findById(productId);
  // variants + gallery ตรงจาก JSON columns ไม่ต้อง join
  return {
    id: product.id,
    name: product.name,
    variants: product.variants,  // Already populated from JSONB
    gallery: product.gallery,
  };
}
```

**Performance improvement:** 1 query แทน 12 queries!

---

### 3.4 CQRS-Lite Pattern (Optional but Recommended)

สำหรับ read-heavy operations (product catalog browsing):

```typescript
// Write Side (Canonical)
async function updateProduct(productId: string, updates: unknown) {
  await db.update(products).set(updates);
  // Emit event
  await queue.send({ event: 'product.updated', productId });
}

// Read Side (Materialized View in KV Cache)
async function getProductDetails(productId: string) {
  const cached = await kv.get(`product:${productId}`);
  if (cached) return JSON.parse(cached);
  
  const product = await db.query.products.findFirst({...});
  await kv.put(`product:${productId}`, JSON.stringify(product), { expirationTtl: 3600 });
  return product;
}
```

**ประโยชน์:**
- ✅ Reads ไปยัง Cloudflare KV (edge) ไม่ต้อง hit D1
- ✅ Cache invalidation through queue events
- ✅ Reduce database load

---

## การออกแบบ Queue

### 4.1 Queue Flows

| Flow | Trigger | Worker | Output |
|------|---------|--------|--------|
| **Order Confirmation** | POST /orders success | email-worker | Send email to customer |
| **Payment Webhook** | Omise webhook | payment-processor | Update order status, trigger fulfillment |
| **Fulfillment** | Payment confirmed | fulfillment-worker | Create shipping label, notify warehouse |
| **Report Generation** | GET /admin/reports (async) | report-worker | Generate PDF, upload to R2, email link |
| **Stock Sync** | Product updated | inventory-sync | Update materialized view, clear KV cache |
| **Sitemap Rebuild** | Product/collection created/updated/deleted | sitemap-worker | Regenerate sitemap.xml, upload to KV |

### 4.2 Implementation Pattern

```typescript
// adapters/queue/QueueProducerCloudflare.ts
export class CloudflareQueueProducer implements IQueueProducer {
  constructor(private queue: Queue) {}

  async sendOrderCreated(order: Order): Promise<void> {
    await this.queue.send({
      type: 'order.created',
      orderId: order.id,
      timestamp: Date.now(),
    });
  }
}

// workers/email-worker/src/index.ts
export default {
  async queue(batch: MessageBatch<QueueMessage>, env: Env) {
    for (const message of batch.messages) {
      if (message.body.type === 'order.created') {
        const order = await env.DB.prepare(...)
          .bind(message.body.orderId).first();
        await sendEmail(order.user.email, orderConfirmationTemplate(order));
      }
      message.ack();
    }
  }
};
```

### 4.3 Dead Letter Queue + Retry

```typescript
// wrangler.toml
[[queues.producers]]
binding = "MY_QUEUE"
queue = "my-queue"

[[queues.consumers]]
queue = "my-queue"
max_retries = 3
max_concurrency = 100
dead_letter_queue = "my-queue-dlq"
```

**Retry strategy:**
- สำหรับ email failures → retry 3 times, exponential backoff
- สำหรับ payment webhook failures → retry indefinitely (อยู่ใน queue)
- สำหรับ DLQ failures → manual inspection + alerting to Slack

---

## API Gateway Pattern

### 5.1 Gateway Architecture

```
┌─────────────────┐
│  Client         │
│  (Web/Mobile)   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  Gateway Worker (gateway.soda.workers.dev)  │
│  - JWT verification                     │
│  - Rate limiting (Cloudflare rules)     │
│  - CORS enforcement                     │
│  - Webhook signature verification       │
│  - Request logging                      │
│  - Response transformation               │
└──────────────────┬──────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
     ▼             ▼             ▼
┌─────────────┐ ┌──────────┐ ┌──────────┐
│ Product     │ │ Order    │ │ Payment  │
│ Worker      │ │ Worker   │ │ Worker   │
└─────────────┘ └──────────┘ └──────────┘
```

### 5.2 Gateway Middleware Implementation

```typescript
// middleware/auth-middleware.ts
export async function verifyJWT(c: Context, next: Next) {
  const token = c.req.header('Authorization')?.replace('Bearer ', '');
  if (!token) return c.json({ error: 'Unauthorized' }, 401);
  
  try {
    const decoded = await verifyToken(token, c.env.JWT_SECRET);
    c.set('user', decoded);
    await next();
  } catch (e) {
    return c.json({ error: 'Invalid token' }, 401);
  }
}

// middleware/webhook-verification.ts
export async function verifyOmiseWebhook(c: Context, next: Next) {
  const signature = c.req.header('X-Omise-Signature');
  const body = await c.req.text();
  
  const expected = createHmac('sha256', c.env.OMISE_SECRET)
    .update(body)
    .digest('hex');
  
  if (signature !== expected) {
    return c.json({ error: 'Invalid signature' }, 401);  // ✅ FIX!
  }
  
  c.set('body', JSON.parse(body));
  await next();
}
```

### 5.3 SEO Routes (Gateway-level)

Gateway Worker serve SEO-critical routes โดยตรงเพื่อ response เร็วที่สุด:

```typescript
// routes/seo.ts — SEO routes ที่ Gateway Worker serve ตรง

// robots.txt — serve จาก Worker (ไม่ต้อง origin)
app.get('/robots.txt', (c) => {
  return c.text(`User-agent: *
Allow: /
Disallow: /api/
Disallow: /admin/

Sitemap: https://sodabkk.com/sitemap.xml`);
});

// sitemap.xml — serve จาก KV (pre-generated โดย sitemap-worker)
app.get('/sitemap.xml', async (c) => {
  const sitemap = await c.env.KV.get('seo:sitemap', 'text');
  if (!sitemap) return c.notFound();
  
  return c.body(sitemap, {
    headers: {
      'Content-Type': 'application/xml',
      'Cache-Control': 'public, max-age=3600',  // 1 ชม.
    },
  });
});

// Slug redirects — 301 redirect เมื่อ product/collection URL เปลี่ยน
app.get('/products/:slug', async (c) => {
  const slug = c.req.param('slug');
  
  // Check slug_redirects table สำหรับ old slugs
  const redirect = await c.env.DB.prepare(
    'SELECT new_slug FROM slug_redirects WHERE entity_type = ? AND old_slug = ?'
  ).bind('product', slug).first();
  
  if (redirect) {
    return c.redirect(`/products/${redirect.new_slug}`, 301);
  }
  
  // ไม่ใช่ redirect → pass ต่อไป Next.js frontend
  return c.next();
});
```

---

## การปรับปรุงความเป็นไปได้ด้านความปลอดภัย

### 6.1 Mapping Current Issues → New Architecture

| Issue (ระบบเดิม) | Risk | Fix (ระบบใหม่) |
|---|---|---|
| Unauthenticated webhook | ❌ Critical | ✅ Gateway: `verifyOmiseWebhook()` middleware |
| CORS wildcard | ❌ High | ✅ Cloudflare: cors() middleware + config per origin |
| No RBAC | ⚠️ Medium | ✅ Domain: Role enum, adapter: RBAC middleware |
| JWT not validated vs DB | ⚠️ Medium | ✅ Adapter: Token blacklist in Cloudflare KV |
| Credentials in git | ❌ Critical | ✅ Wrangler secrets + `--local` environment |
| Mixed float precision | ⚠️ Medium | ✅ Schema: DECIMAL(12,2) everywhere |
| Stock not atomic | ❌ High | ✅ Schema: `stock_reservations` table + constraints |

### 6.2 Token Blacklist Pattern

```typescript
// adapters/auth/TokenBlacklistAdapter.ts
export class TokenBlacklistAdapter implements ITokenBlacklist {
  constructor(private kv: KVNamespace) {}

  async blacklistToken(token: string, expirySeconds: number): Promise<void> {
    await this.kv.put(`blacklist:${token}`, '1', { expirationTtl: expirySeconds });
  }

  async isBlacklisted(token: string): Promise<boolean> {
    return (await this.kv.get(`blacklist:${token}`)) !== null;
  }
}

// usage: logout endpoint
app.post('/api/auth/logout', async (c) => {
  const token = c.get('token');
  const exp = c.get('user').exp;
  await tokenBlacklist.blacklistToken(token, exp - Math.floor(Date.now() / 1000));
  return c.json({ success: true });
});
```

### 6.3 Secret Management

```toml
# wrangler.toml
[env.production]
vars = { ENVIRONMENT = "production" }

# Secrets (ไม่ store ใน wrangler.toml)
# wrangler secret put JWT_SECRET --env production
# wrangler secret put OMISE_SECRET --env production
```

```bash
# Deploy with secrets
wrangler deploy --env production
```

---

## กลยุทธ์การอพเกรด

### 7.1 Strangler Fig Pattern

**Phase 1: Parallel Run** (Weeks 1-2)
```
Client → Load Balancer
         ├── 10% traffic → New Hono API (Cloudflare)
         └── 90% traffic → Old Go API (DigitalOcean)
```

- Deploy new Hono API on staging
- Run integration tests against old API schemas
- Gradual traffic shift (10% → 25% → 50%)

**Phase 2: Read Traffic Migration** (Weeks 2-3)
```
Reads (Products, Categories) → New API
Writes (Orders, Payments) → Old API
```

- Migrate product catalog (read-only)
- Validate data consistency
- Keep old API as fallback

**Phase 3: Write Traffic Migration** (Weeks 3-4)
```
All traffic → New API
Old API → Archive mode (read-only for audit)
```

- Data sync: users, orders, payments
- Dual-write temporary period:
  - New order → Write to both old + new DB
  - Verify consistency
  - Once verified → switch to new only

**Phase 4: Decommission** (Week 5)
```
Old API → Shutdown
DigitalOcean instance → Terminate
GCS bucket → Archive
```

### 7.2 Data Migration Strategy

```typescript
// migration/migrateUsers.ts
import { migrate } from 'cloudflare-d1-migration';

export async function migrateUsers(oldDB: any, newDB: D1Database) {
  const users = await oldDB.query('SELECT * FROM users');
  
  for (const user of users) {
    await newDB
      .prepare(`
        INSERT INTO users (id, email, phone, password_hash, role, profile, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?)
      `)
      .bind(
        user.id,
        user.email,
        user.phone,
        user.password_hash,
        user.role || 'customer',
        JSON.stringify({ firstName: user.first_name, lastName: user.last_name }),
        user.created_at
      )
      .run();
  }
}

// migration/migrateOrders.ts
export async function migrateOrders(oldDB: any, newDB: D1Database) {
  const orders = await oldDB.query('SELECT o.*, oi.* FROM orders o LEFT JOIN order_items oi ON o.id = oi.order_id');
  
  // Group by order_id
  const grouped = groupBy(orders, 'order_id');
  
  for (const [orderId, items] of grouped.entries()) {
    const order = items[0];
    
    // Insert order
    await newDB.prepare(`
      INSERT INTO orders (id, user_id, status, total_amount, currency, created_at)
      VALUES (?, ?, ?, ?, ?, ?)
    `).bind(orderId, order.user_id, order.status, order.total_amount, 'THB', order.created_at).run();
    
    // Insert order_items with snapshot
    for (const item of items) {
      const product = await oldDB.query('SELECT * FROM products WHERE id = ?', [item.product_id]).first();
      
      await newDB.prepare(`
        INSERT INTO order_items (id, order_id, product_snapshot, quantity, unit_price, created_at)
        VALUES (?, ?, ?, ?, ?, ?)
      `).bind(
        item.id,
        orderId,
        JSON.stringify({
          sku: product.sku,
          name: product.name,
          color: item.color,
          size: item.size,
        }),
        item.quantity,
        item.unit_price,
        item.created_at
      ).run();
    }
  }
}
```

### 7.3 Validation & Rollback

```typescript
// migration/validate.ts
export async function validateMigration(oldDB: any, newDB: D1Database) {
  const oldUserCount = await oldDB.query('SELECT COUNT(*) as cnt FROM users').first();
  const newUserCount = await newDB.prepare('SELECT COUNT(*) as cnt FROM users').first();
  
  if (oldUserCount.cnt !== newUserCount.cnt) {
    throw new Error(`User count mismatch: ${oldUserCount.cnt} vs ${newUserCount.cnt}`);
  }
  
  console.log('✅ Migration validation passed');
}
```

---

## หัวข้ออื่นที่ควรพิจารณา

### 8.1 Observability & Monitoring

#### Cloudflare Native Observability Stack (ฟรี)

Cloudflare มี observability tools ครบจบในตัว ลดการพึ่ง third-party service:

| Tool | หน้าที่ | Free Plan | Paid Plan ($5/mo) |
|------|---------|-----------|-------------------|
| **Workers Logs** | เก็บ log events อัตโนมัติจากทุก Worker | 6M events/mo, retention 3 วัน | 20M events/mo, retention 7 วัน (+$0.60/M) |
| **Workers Traces** | Distributed tracing (auto-instrumented) | ✅ ดูใน dashboard ฟรี | ✅ + export ไป OTLP destination |
| **Workers Analytics** | Request metrics, errors, latency dashboard | ✅ ฟรี | ✅ ฟรี |
| **Analytics Engine** | Custom metrics + SQL queries (ฟรีช่วง beta) | ✅ ฟรี (ยังไม่เริ่ม billing) | ✅ |
| **Real-time Logs** | `wrangler tail` — stream logs ใน terminal | ✅ ฟรี | ✅ ฟรี |
| **Tail Workers** | Custom log processing/filtering/sampling | ✅ ฟรี (ใช้ Worker requests) | ✅ |
| **Logpush** | Export logs ไป S3, R2, Datadog, Splunk etc. | ❌ Paid only | ✅ |

#### Workers Logs (Auto-collected)

Workers Logs เก็บ log events อัตโนมัติจาก `console.log()` ทุก Worker โดยไม่ต้อง config อะไรเพิ่ม — แค่เปิด observability setting (default on สำหรับ Workers ใหม่):

```typescript
// middleware/structured-logger.ts
// console.log() จะถูกเก็บใน Workers Logs อัตโนมัติ
app.use('*', async (c, next) => {
  const requestId = crypto.randomUUID();
  const startTime = performance.now();
  
  c.set('requestId', requestId);
  
  await next();
  
  // Log นี้จะปรากฏใน Workers Logs dashboard + queryable
  console.log(JSON.stringify({
    level: c.res.status >= 500 ? 'error' : c.res.status >= 400 ? 'warn' : 'info',
    requestId,
    method: c.req.method,
    path: c.req.path,
    status: c.res.status,
    durationMs: Math.round(performance.now() - startTime),
    userId: c.get('user')?.id ?? null,
    userAgent: c.req.header('user-agent'),
    cfRay: c.req.header('cf-ray'),           // Cloudflare ray ID
    cfCountry: c.req.header('cf-ipcountry'), // Country code
  }));
});
```

**Query logs ใน dashboard:**

Workers Logs รองรับ SQL-like queries ใน Cloudflare dashboard:
- Filter ตาม `status >= 500` เพื่อดู errors
- Filter ตาม `path` เพื่อ debug specific endpoints
- Filter ตาม `userId` เพื่อ trace user journey
- Sort ตาม `durationMs` เพื่อหา slow requests

#### Workers Traces (Distributed Tracing)

Cloudflare auto-instrument traces สำหรับ Workers — เห็น request flow ข้าม Workers, D1 queries, KV reads, R2 operations โดยไม่ต้องเขียน code เพิ่ม:

```
Request → Gateway Worker → D1 Query → KV Cache Check → Response
         ├─ 2ms           ├─ 15ms    ├─ 1ms           ├─ 18ms total
```

Traces แสดง:
- ⏱️ Duration ของแต่ละ sub-request (D1, KV, R2, fetch)
- 🔗 Waterfall view ของ request chain
- ❌ Error stack traces
- 📊 Cold start vs warm invocation

**Export traces ไป external provider (OTLP):**

```toml
# wrangler.toml — export traces ไป Grafana Cloud, Honeycomb, Axiom etc.
[observability]
enabled = true

[observability.logs]
enabled = true
head_sampling_rate = 1  # 1 = log ทุก request, 0.1 = sample 10%

# Optional: export ไป OTLP endpoint (Paid plan)
# [observability.traces]
# enabled = true
# head_sampling_rate = 0.1
```

#### Analytics Engine (Custom Metrics)

Analytics Engine ให้เราเก็บ custom business metrics แบบ time-series แล้ว query ด้วย SQL ได้ ฟรีช่วง beta:

```typescript
// adapters/analytics/CloudflareAnalytics.ts
export class CloudflareAnalytics implements IAnalytics {
  constructor(private ae: AnalyticsEngineDataset) {}
  
  trackOrder(order: Order): void {
    this.ae.writeDataPoint({
      blobs: [order.paymentMethod, order.status],
      doubles: [order.totalAmount, order.items.length],
      indexes: [order.id],
    });
  }
  
  trackProductView(productId: string, source: string): void {
    this.ae.writeDataPoint({
      blobs: [productId, source],  // source: 'search', 'collection', 'direct'
      doubles: [1],                 // view count
      indexes: [productId],
    });
  }
}
```

**ใช้ได้ฟรี สำหรับ:**
- จำนวน orders ต่อวัน/ชั่วโมง
- Revenue tracking
- Product view counts (ใช้กับ SEO analytics)
- Payment method breakdown
- Error rate per endpoint

#### Tail Workers (Advanced Log Processing)

สำหรับ use cases ที่ต้อง custom processing เช่น filter sensitive data, sample logs, หรือ forward ไป external service:

```typescript
// workers/tail-worker/src/index.ts
export default {
  async tail(events: TraceItem[], env: Env) {
    for (const event of events) {
      // Filter: เก็บเฉพาะ errors + slow requests (> 1s)
      const isError = event.outcome === 'exception';
      const isSlow = event.event?.some(
        e => e.type === 'fetch' && e.durationMs > 1000
      );
      
      if (isError || isSlow) {
        // Forward ไป R2 สำหรับ long-term storage (ฟรี egress)
        await env.R2_LOGS.put(
          `logs/${new Date().toISOString()}/${event.scriptName}/${crypto.randomUUID()}.json`,
          JSON.stringify(event)
        );
      }
    }
  }
};
```

```toml
# wrangler.toml — attach tail worker
[[tail_consumers]]
service = "tail-worker"
```

#### Error Tracking Strategy

แทนที่จะใช้ Sentry (มีค่าใช้จ่าย) เราใช้ Cloudflare native tools เป็นหลัก แล้วเพิ่ม Sentry เฉพาะเมื่อต้องการ:

| ระดับ | เครื่องมือ | ค่าใช้จ่าย |
|-------|----------|-----------|
| **Level 1: Basic** (MVP) | Workers Logs + Traces + Analytics | ฟรี |
| **Level 2: Enhanced** | + Analytics Engine (custom metrics) | ฟรี (beta) |
| **Level 3: Advanced** | + Tail Worker → R2 (long-term logs) | ฟรี (ใช้ R2 free tier) |
| **Level 4: Full** (ถ้าต้องการ) | + Sentry (error tracking + alerting) | $0-26/mo |

**MVP แนะนำ Level 1-2** — ฟรีทั้งหมด ครอบคลุม:

```typescript
// middleware/error-handler.ts
app.onError((err, c) => {
  const requestId = c.get('requestId');
  
  // Workers Logs จะเก็บ error นี้อัตโนมัติ
  console.error(JSON.stringify({
    level: 'error',
    requestId,
    error: err.message,
    stack: err.stack,
    path: c.req.path,
    method: c.req.method,
    userId: c.get('user')?.id,
  }));
  
  return c.json({ 
    error: 'Internal Server Error',
    requestId, // ให้ user report requestId เพื่อ debug
  }, 500);
});
```

**เพิ่ม Sentry ทีหลังได้** ถ้าต้องการ alerting (Slack/email notification เมื่อ error spike) และ error grouping:

```typescript
// เพิ่มทีหลังเมื่อต้องการ (optional)
import { Toucan } from 'toucan-js'; // Sentry SDK สำหรับ Cloudflare Workers

app.onError((err, c) => {
  const sentry = new Toucan({ dsn: c.env.SENTRY_DSN, context: c.executionCtx });
  sentry.captureException(err);
  // ... rest of error handling
});
```

#### wrangler.toml Configuration (Observability)

```toml
# wrangler.toml — เปิด observability ทั้งหมด
[observability]
enabled = true

[observability.logs]
enabled = true
head_sampling_rate = 1  # Production: ลดเป็น 0.1 ถ้า log volume สูง

# Tail Worker สำหรับ advanced processing
[[tail_consumers]]
service = "tail-worker"

# Analytics Engine binding
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "soda_bkk_metrics"
```

### 8.2 Testing Strategy

**Vitest + Hexagonal architecture = ง่ายทำ unit tests**

```typescript
// domain/__tests__/CreateOrderUseCase.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { CreateOrderUseCase } from '../CreateOrderUseCase';

describe('CreateOrderUseCase', () => {
  let useCase: CreateOrderUseCase;
  let mockOrderRepo: any;
  let mockPaymentGateway: any;
  
  beforeEach(() => {
    mockOrderRepo = { save: vi.fn() };
    mockPaymentGateway = { charge: vi.fn() };
    useCase = new CreateOrderUseCase(mockOrderRepo, mockPaymentGateway);
  });
  
  it('should create order with valid items', async () => {
    const input = {
      userId: 'user-1',
      items: [{ sku: 'SKU-001', quantity: 2 }],
      paymentMethod: 'card',
    };
    
    await useCase.execute(input);
    expect(mockOrderRepo.save).toHaveBeenCalled();
  });
  
  it('should reject order if payment fails', async () => {
    mockPaymentGateway.charge.mockRejectedValue(new Error('Declined'));
    
    await expect(
      useCase.execute({ ...input, paymentMethod: 'card' })
    ).rejects.toThrow('Payment failed');
  });
});
```

### 8.3 Local Development with Wrangler

```bash
# Start local dev environment
wrangler dev --remote

# Test with local D1 database
wrangler d1 create soda-shop-dev
wrangler dev --local

# Inspect KV locally
wrangler kv:key list --namespace-id <namespace-id> --local
```

**Gotchas:**
- Local D1 vs remote D1 → ต่างกัน ต้องทำ migration ในท้องถิ่นเหมือนทำในระดับไกล
- Local queue ทำงาน differently (synchronous vs asynchronous)

### 8.4 CPU Time Limit: Handling Long-Running Tasks

**Problem:** Report generation (PDF) อาจใช้ 45 seconds → exceed 30s limit

**Solutions:**

**A) Use Cloudflare Queues for async generation**
```typescript
app.post('/api/admin/reports', async (c) => {
  const reportId = generateId();
  await reportQueue.send({
    type: 'generate.report',
    reportId,
    filters: c.req.query(),
  });
  return c.json({ reportId, status: 'processing' });
});

// Separate worker for report generation
export default {
  async queue(batch: MessageBatch, env: Env) {
    for (const message of batch.messages) {
      const pdf = await generatePDFReport(...);
      await env.R2_BUCKET.put(`reports/${message.body.reportId}.pdf`, pdf);
      message.ack();
    }
  }
};
```

**B) Offload to external service (SendGrid, AWS Lambda)**
- ได้ไม่มีข้อจำกัด CPU time

---

### 8.5 Cost Estimation

**Free tier limits vs SODA BKK traffic:**

| Service | Free Tier | SODA Est. | Status |
|---------|-----------|-----------|--------|
| Workers | 100k req/day | ~5k orders/day | ✅ ฟรี |
| D1 | 100k reads/writes | ~20k/day | ✅ ฟรี |
| KV | 1M ops/month | ~2M cache hits | ⚠️ Borderline |
| Queues | 1M messages/month | ~15k/month | ✅ ฟรี |
| R2 | 10GB storage | ~2GB images | ✅ ฟรี |
| R2 egress | Free | ~50GB/month | ✅ ฟรี |

**Upgrade timing:**
- KV: When cache hit rate > 1M/month → $0.50/million ops
- D1: When writes > 100k/day → pay-as-you-go ($0.50/million)
- Workers: When requests > 100k/day → $0.50/million

**Estimated monthly cost (year 2):** $5-15/month (after exceeding free tiers)

---

### 8.6 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloudflare

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm run test  # Run Vitest
      - run: pnpm run lint

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm run build
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: deploy --env production
```

### 8.7 Image Optimization

**Cloudflare Images vs R2 + Transform Worker:**

| Feature | Cloudflare Images | R2 + Transform |
|---------|------------------|-----------------|
| **Resizing** | ✅ Built-in | ✅ Custom worker |
| **Format negotiation** | ✅ WebP/AVIF | ✅ Custom |
| **Caching** | ✅ Edge | ✅ Edge (R2) |
| **Cost** | $20/100k images | Free (R2 egress) |

**แนะนำ:** R2 + Transform Worker (ประหยัดค่า)

```typescript
// worker/image-transform.ts
export default {
  async fetch(request: Request, env: Env) {
    const url = new URL(request.url);
    const key = url.pathname.slice(1); // /products/image.jpg
    const width = url.searchParams.get('width');
    
    const image = await env.R2_BUCKET.get(key);
    if (!image) return new Response('Not found', { status: 404 });
    
    // Use Cloudflare Image Optimization
    return env.IMAGES.transform(image, {
      width: parseInt(width || '400'),
      quality: 80,
      format: 'webp',
    });
  }
};
```

**Image SEO สำหรับ Product:**

gallery JSON ของ product ต้องเก็บข้อมูล SEO ด้วย:

```typescript
// gallery JSON structure (ใน products.gallery column)
interface ProductImage {
  url: string;           // R2 path: "products/abc123/main.jpg"
  alt: string;           // Alt text สำหรับ SEO: "เสื้อยืด Oversized สีดำ - SODA BKK"
  width: number;         // Original width (สำหรับ srcset)
  height: number;        // Original height (ป้องกัน CLS)
  position: number;      // ลำดับ (0 = primary image สำหรับ og:image)
}
```

Backend ต้อง return `width`/`height` ให้ Frontend เพื่อป้องกัน Cumulative Layout Shift (CLS) — หนึ่งใน Core Web Vitals ที่ส่งผลต่อ SEO ranking

### 8.8 Search: Full-Text Search

**Option A: D1 SQLite FTS5**
```sql
CREATE VIRTUAL TABLE products_fts USING fts5(
  name, description
);
```

**Option B: Cloudflare Vectorize + ML**
- Use Cloudflare's embedding model
- Store vectors in D1/Vectorize
- Support semantic search

**แนะนำ:** D1 FTS5 (ก่อน scale)

```typescript
// application/SearchProductsUseCase.ts
async execute(query: string) {
  const results = await this.productRepo.search(query);
  // SELECT * FROM products_fts WHERE products_fts MATCH 'blue shirt'
  return results.map(r => this.toDTO(r));
}
```

### 8.9 Internationalization

SODA BKK ขายไปหลายประเทศ → ต้องคิด localization ตั้งแต่ design

```typescript
// domain/entities/Product.ts
interface Product {
  id: string;
  translations: Record<Locale, {
    name: string;
    description: string;
  }>;
  // ...
}

// adapters/http/middleware/locale.ts
app.use('*', (c, next) => {
  const locale = c.req.query('lang') || 'th';
  c.set('locale', locale as Locale);
  return next();
});
```

### 8.10 Type-Safe API Client (Hono RPC)

**Monorepo Super Power:** Frontend ได้ auto-complete API calls โดยไม่ต้องเขียน client code เองเลย — type ถูก generate จาก backend โดยตรง

```typescript
// packages/backend/src/routes.ts
const app = new Hono()
  .get('/api/products/:id', async (c) => {
    // ...
    return c.json({ id, name, price });
  })
  .post('/api/orders', async (c) => {
    const body = await c.req.json<CreateOrderInput>();
    return c.json({ orderId: 'ord_123', status: 'pending' });
  });

export type AppType = typeof app;

// packages/web-storefront/src/api.ts
import { hc } from 'hono/client';
import type { AppType } from '@soda-bkk/backend';

const client = hc<AppType>('https://api.soda.workers.dev');

// ✅ Full type-safe! IDE autocomplete ทั้ง request + response
const product = await client.api.products[':id'].$get({ 
  param: { id: '123' }
});

const order = await client.api.orders.$post({
  json: { items: [...], paymentToken: 'tokn_xxx' }
});
```

**ประโยชน์ใน monorepo:**
- เปลี่ยน response type ใน backend → TypeScript error ใน frontend ทันที
- ไม่ต้องเขียน/maintain API types แยกต่างหาก
- ลด risk ของ frontend/backend type mismatch ที่เป็นปัญหาบ่อยครั้ง

### 8.11 Drizzle ORM — Type-Safe Database Access

Drizzle เป็น ORM ที่เหมาะที่สุดสำหรับ Cloudflare D1 ในตอนนี้ ให้ผล type-safe ตั้งแต่ schema definition จนถึง query result

**Schema definition:**
```typescript
// packages/backend/src/db/schema.ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core';

export const products = sqliteTable('products', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  categoryId: text('category_id'),
  basePrice: real('base_price').notNull(),
  variants: text('variants', { mode: 'json' }).$type<ProductVariant[]>(),
  gallery: text('gallery', { mode: 'json' }).$type<GalleryImage[]>(),
  isDisplay: integer('is_display', { mode: 'boolean' }).default(true),
  createdAt: text('created_at').default(sql`CURRENT_TIMESTAMP`),
});

export const orders = sqliteTable('orders', {
  id: text('id').primaryKey(),
  orderNumber: text('order_number').notNull().unique(),
  userId: text('user_id').notNull(),
  status: text('status', { 
    enum: ['payment_pending', 'payment_success', 'shipping_pending', 'shipping_completed', 'cancel', 'void'] 
  }).notNull(),
  totalAmount: real('total_amount').notNull(),
  currency: text('currency').default('THB'),
  createdAt: text('created_at').default(sql`CURRENT_TIMESTAMP`),
});
```

**Migration workflow:**
```bash
# Generate migration file (จาก schema changes)
pnpm drizzle-kit generate:sqlite

# Apply to D1 local
wrangler d1 migrations apply soda-shop-dev --local

# Apply to D1 production
wrangler d1 migrations apply soda-shop-prod --remote
```

**Type-safe queries:**
```typescript
// adapters/persistence/ProductRepositoryD1.ts
import { drizzle } from 'drizzle-orm/d1';
import { eq, and, like } from 'drizzle-orm';
import { products } from '../db/schema';

export class ProductRepositoryD1 implements IProductRepository {
  private db: ReturnType<typeof drizzle>;

  constructor(d1: D1Database) {
    this.db = drizzle(d1);
  }

  async findById(id: string) {
    // ✅ Return type inferred automatically — no manual typing needed
    return this.db.select().from(products).where(eq(products.id, id)).get();
  }

  async findDisplayed() {
    return this.db
      .select()
      .from(products)
      .where(eq(products.isDisplay, true))
      .all();
  }
}
```

**ทำไม Drizzle แทน Prisma:**
| Feature | Drizzle | Prisma |
|---------|---------|--------|
| D1 support | ✅ Native | ⚠️ Limited |
| Bundle size | ~50KB | ~500KB+ |
| Workers memory | ✅ ไม่กิน | ⚠️ หนัก |
| SQL control | ✅ ใกล้ raw SQL | ❌ ORM abstraction |
| Type safety | ✅ Full | ✅ Full |

### 8.12 Error Handling Strategy (แก้ปัญหาเดิมที่ทุก error return 500)

ระบบเดิมมีปัญหาใหญ่คือ controller ส่ง `http.StatusInternalServerError` สำหรับทุก error ไม่ว่าจะเป็น not found, validation error หรือจริงๆ แล้ว 500 ในระบบใหม่ให้ใช้ structured error hierarchy ตั้งแต่ domain layer

```typescript
// domain/errors/DomainError.ts
export class DomainError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly statusCode: number = 400
  ) {
    super(message);
  }
}

export class NotFoundError extends DomainError {
  constructor(entity: string, id: string) {
    super('NOT_FOUND', `${entity} with id '${id}' not found`, 404);
  }
}

export class ValidationError extends DomainError {
  constructor(message: string) {
    super('VALIDATION_ERROR', message, 422);
  }
}

export class PaymentError extends DomainError {
  constructor(message: string) {
    super('PAYMENT_FAILED', message, 402);
  }
}

export class StockInsufficientError extends DomainError {
  constructor(sku: string) {
    super('STOCK_INSUFFICIENT', `SKU '${sku}' has insufficient stock`, 409);
  }
}
```

**Gateway-level error handler:**
```typescript
// middleware/error-handler.ts
app.onError((err, c) => {
  if (err instanceof DomainError) {
    return c.json({
      success: false,
      error: { code: err.code, message: err.message },
      timestamp: new Date().toISOString(),
    }, err.statusCode as StatusCode);
  }
  
  // Unexpected errors → Sentry + 500
  Sentry.captureException(err);
  return c.json({
    success: false,
    error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' },
    timestamp: new Date().toISOString(),
  }, 500);
});
```

### 8.13 Input Validation (Zod)

ระบบเดิมมีแค่ Gin binding tags ไม่มี custom validation — ในระบบใหม่ใช้ Zod ควบคู่กับ Hono validator middleware

```typescript
// adapters/http/validators/orderValidator.ts
import { z } from 'zod';
import { zValidator } from '@hono/zod-validator';

const createOrderSchema = z.object({
  userId: z.string().min(1),
  items: z.array(z.object({
    sku: z.string().min(1),
    quantity: z.number().int().positive().max(100),
    price: z.number().positive(),
  })).min(1),
  paymentToken: z.string().startsWith('tokn_').or(z.string().startsWith('src_')),
  shippingAddressId: z.string().optional(),
});

// ใช้งาน
app.post('/api/orders', 
  zValidator('json', createOrderSchema),  // ✅ Auto validate + type-safe body
  async (c) => {
    const body = c.req.valid('json');  // body is fully typed here
    // ...
  }
);
```

**ข้อดีเทียบกับระบบเดิม:**
- `strconv.ParseUint(id, 10, 64)` ที่ silently fail → ใช้ `z.string().uuid()` แทน
- float precision issues → Zod enforce `z.number().multipleOf(0.01)` สำหรับราคา
- No field-level error messages → Zod ให้ error message per-field โดยอัตโนมัติ

### 8.14 Product SEO (Backend Responsibilities)

Backend มีหน้าที่เตรียมข้อมูลที่ Frontend (Next.js) ต้องการสำหรับ SEO ทั้งหมด:

#### Storefront API Response (SEO-enriched)

```typescript
// GET /api/storefront/products/:slug
// Response ต้องมีข้อมูลครบสำหรับ SSR + structured data
interface ProductSEOResponse {
  // Product data
  id: string;
  name: string;
  slug: string;
  description: string;
  basePrice: number;
  variants: ProductVariant[];
  gallery: ProductImage[];     // มี alt, width, height สำหรับ SEO
  
  // SEO meta (Frontend ใช้สำหรับ <head> tags)
  seo: {
    title: string;             // meta_title หรือ fallback เป็น name
    description: string;       // meta_description หรือ fallback เป็น description ตัดสั้น
    canonicalUrl: string;      // https://sodabkk.com/products/{slug}
    ogImage: string;           // gallery[0] transformed URL (1200x630)
  };
  
  // Structured Data (Backend เตรียมข้อมูลให้ Frontend render เป็น JSON-LD)
  structuredData: {
    "@type": "Product";
    name: string;
    description: string;
    image: string[];
    brand: { "@type": "Brand"; name: "SODA BKK" };
    offers: {
      "@type": "Offer";
      price: number;
      priceCurrency: "THB";
      availability: "InStock" | "OutOfStock";
      url: string;
    };
    sku: string;               // SKU ของ default variant
    category: string;          // breadcrumb category name
  };
  
  // Breadcrumbs (สำหรับ BreadcrumbList schema)
  breadcrumbs: Array<{
    name: string;              // "หน้าแรก" > "เสื้อผ้า" > "เสื้อยืด"
    url: string;
  }>;
}
```

#### Sitemap Worker (Queue-based generation)

```typescript
// workers/sitemap-worker/src/index.ts
export default {
  async queue(batch: MessageBatch<QueueMessage>, env: Env) {
    // Triggered เมื่อ product/collection created/updated/deleted
    
    // 1. Query all active products + collections
    const products = await env.DB.prepare(
      'SELECT slug, updated_at FROM products WHERE status = ?'
    ).bind('active').all();
    
    // 2. Generate sitemap XML
    const sitemap = generateSitemapXml({
      staticPages: ['/', '/collections', '/about', '/contact'],
      products: products.results.map(p => ({
        loc: `https://sodabkk.com/products/${p.slug}`,
        lastmod: p.updated_at,
        changefreq: 'weekly',
        priority: 0.8,
      })),
    });
    
    // 3. Store ใน KV (Gateway Worker serve จาก KV)
    await env.KV.put('seo:sitemap', sitemap, {
      metadata: { generatedAt: new Date().toISOString() },
    });
  }
};
```

#### Slug Management (Use Case)

```typescript
// application/UpdateProductSlugUseCase.ts
export class UpdateProductSlugUseCase {
  async execute(productId: string, newSlug: string): Promise<void> {
    const product = await this.productRepo.findById(productId);
    const oldSlug = product.slug;
    
    if (oldSlug === newSlug) return;
    
    // 1. Update product slug
    await this.productRepo.updateSlug(productId, newSlug);
    
    // 2. บันทึก old slug → new slug redirect (301)
    await this.slugRedirectRepo.create({
      entityType: 'product',
      entityId: productId,
      oldSlug,
      newSlug,
    });
    
    // 3. Trigger sitemap rebuild
    await this.queue.send({ type: 'sitemap.rebuild' });
    
    // 4. Clear KV cache สำหรับ old slug
    await this.kv.delete(`product:${oldSlug}`);
  }
}
```

**สิ่งที่ Backend ต้องรับผิดชอบ vs Frontend:**

| ด้าน | Backend | Frontend (Next.js) |
|------|---------|-------------------|
| Product slug | ✅ Generate, store, redirect old slugs | ใช้ slug ใน URL routing |
| meta_title / meta_description | ✅ Store ใน DB, return ใน API | Render ใน `<head>` |
| Structured Data (JSON-LD) | ✅ เตรียมข้อมูลครบใน API response | Render เป็น `<script type="application/ld+json">` |
| og:image | ✅ Return transformed image URL (1200x630) | Render ใน `<meta property="og:image">` |
| Sitemap | ✅ Generate + serve จาก KV | ไม่เกี่ยว |
| robots.txt | ✅ Serve จาก Gateway Worker | ไม่เกี่ยว |
| 301 Redirects | ✅ slug_redirects table + Gateway route | ไม่เกี่ยว |
| Image alt text | ✅ Store ใน gallery JSON | Render ใน `<img alt="">` |
| Image dimensions (CLS) | ✅ Store width/height ใน gallery JSON | ใช้ใน `<img width="" height="">` |
| Canonical URL | ✅ Return ใน API response | Render ใน `<link rel="canonical">` |
| Breadcrumbs | ✅ Return category hierarchy | Render + BreadcrumbList schema |

---

## สรุปเทคโนโลยีที่แนะนำ

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Frontend (Storefront)** | React/Next.js | SEO + SSR for e-commerce |
| **Frontend (Back Office)** | React (Vite) | SPA เพียงพอสำหรับ admin |
| **Backend Runtime** | Cloudflare Workers | 0ms cold start, geographic distribution |
| **Backend Framework** | Hono + TypeScript | Edge-first, tiny bundle, RPC type-safe |
| **Architecture** | Hexagonal (Ports & Adapters) | Testability, modularity, infra independence |
| **Database** | Cloudflare D1 (SQLite) | Free tier, schema redesign, Workers native |
| **ORM** | Drizzle ORM | Type-safe SQL, migration support, D1 native, tiny bundle |
| **Validation** | Zod | Runtime + compile-time validation, Hono-compatible |
| **Storage** | Cloudflare R2 | S3-compatible, free egress, CDN integration |
| **Queue** | Cloudflare Queues | Free 1M/month, built-in retry + DLQ |
| **KV Cache** | Cloudflare KV | Token blacklist, product read cache, sessions |
| **Testing** | Vitest | ESM-native, fast, mock-friendly with Hexagonal |
| **Monorepo** | Turborepo + pnpm | Task caching, workspace, shared types |
| **API Gateway** | Cloudflare Workers | JWT auth, rate-limit, webhook verification, CORS |
| **Logging** | Workers Logs (Cloudflare native) | Auto-collected, queryable, ฟรี 6M events/mo |
| **Tracing** | Workers Traces (Cloudflare native) | Auto-instrumented distributed tracing, ฟรี |
| **Custom Metrics** | Analytics Engine (Cloudflare) | Business metrics + SQL query, ฟรี (beta) |
| **Error Tracking** | Cloudflare native (Level 1-2) → Sentry (Level 4, optional) | ฟรีเริ่มต้น, เพิ่ม Sentry ทีหลังถ้าต้องการ alerting |
| **Secrets** | Wrangler Secrets | Zero credential leak risk (ปิด git secret gap) |
| **CI/CD** | GitHub Actions + Wrangler | Monorepo-compatible, automated deploy |
| **Authentication** | Better Auth | MIT, self-hosted, Cloudflare D1 native, Hono first-class, built-in RBAC/2FA/passkeys, free |
| **API Client** | Hono RPC | Auto-generated type-safe client ใน Frontend |

---

## ความเสี่ยงและคำถามที่ยังเปิดอยู่

### 10.1 Open Questions

| Question | Impact | Decision Date |
|----------|--------|-----------------|
| ต้องการ advanced PostgreSQL features (JSON query, arrays) หรือ SQLite พอ? | Schema design | Week 1 |
| Product images จำนวนเท่าไร? (จะใช้ Cloudflare Images หรือ R2?) | Storage cost | Week 1 |
| ต้องการ realtime inventory updates (WebSocket/SSE) หรือ polling? | Architecture | Week 2 |
| การรองรับหลายสกุลเงิน (THB, USD) ต้องหรือไม่? | Schema, payment logic | Week 2 |

### 10.2 Risks & Mitigation

| Risk | Probability | Mitigation |
|------|-------------|-----------|
| D1 quota exceeded | Low | Monitor usage, upgrade early if > 50k writes/day |
| Workers CPU timeout on heavy queries | Low | Denormalize schema, use queue for async tasks |
| Cold start on first deploy (edge warming) | Low | Geographic replication automatic, no action needed |
| Data consistency during migration | Medium | Run validation, dual-write period, automated tests |
| Vendor lock-in (Cloudflare) | High | Hexagonal architecture isolates domain from infra |
| Team learning curve (Workers, Hono, D1) | Medium | Allocate 2-3 weeks for ramp-up, pair programming |

### 10.3 Success Metrics

| Metric | Target | Measurement |
|--------|--------|------------|
| **Latency** | < 200ms p99 | Cloudflare Analytics |
| **Uptime** | 99.9% | Sentry + Cloudflare monitoring |
| **Query count** | < 2 per request | Application logging |
| **Test coverage** | > 80% | Vitest coverage report |
| **Deployment time** | < 5 minutes | GitHub Actions logs |
| **Cost** | < $20/month | Cloudflare billing dashboard |

---

## สรุป

ระบบใหม่ (Cloudflare + Hono + D1 + Hexagonal) ให้คุณ:

✅ **เศษเสียของ infrastructure** — Workers ทำหมด, ไม่ต้อง manage servers  
✅ **ราคาถูก** — ฟรี tier ใช้ได้นาน ก่อน scale  
✅ **ความปลอดภัย** — ไม่ต้องกังวล credentials, webhook signature verification built-in  
✅ **ความเร็ว** — Geographic distribution, 0ms cold start, KV cache edge  
✅ **ความสามารถในการบำรุง** — Modular design, hexagonal arch, easy testing  
✅ **Developer Experience** — Type-safe API client (Hono RPC), Turborepo, Drizzle migrations  

**Next Step:** Approve database design (SQLite + denormalization approach) ก่อนเริ่มเขียนโค้ด

---

**เอกสารที่เกี่ยวข้อง:**
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Hono Framework](https://hono.dev/)
- [Drizzle ORM D1 Guide](https://orm.drizzle.team/docs/get-started-sqlite)
- [Hexagonal Architecture (Ports & Adapters)](https://alistair.cockburn.us/hexagonal-architecture/)

---

## Authentication & Authorization Design

### สรุปปัญหาระบบเดิม

| ปัญหา | ที่มา | ผลกระทบ |
|--------|-------|---------|
| Guest + Member + Admin อยู่ใน `users` table เดียว | `entity/user.go` | Role check ซับซ้อน, auto-generated password สำหรับ guest ไม่มีความหมาย |
| JWT middleware ไม่ verify token vs DB | `auth_middleware.go` | Logout แล้วยัง access ได้จน token หมดอายุ (24 ชม.) |
| ไม่มี RBAC จริง — แค่ on/off auth | `server.go` route groups | User คนไหนก็เข้า admin endpoints ได้หมด ขอแค่มี valid JWT |
| CORS wildcard + credentials | `cors_middleware.go` | ทุก origin เรียก API ได้ |
| Credentials ใน git | `certs/`, `service_credentials.json` | Secret หลุดใน repository |
| ไม่มี rate limiting | `server.go` | Brute-force login ได้ไม่จำกัด |

---

### Auth Provider Selection

#### ทำไมไม่สร้าง Auth เอง?

ระบบเดิมใช้ custom JWT + bcrypt ที่เขียนเอง ซึ่งมีปัญหาหลายจุด (ดูตารางด้านบน) การสร้าง auth จาก 0 ในระบบใหม่จะซ้ำปัญหาเดิม — ต้องจัดการ session, password hashing, email verification, forgot password, OAuth, 2FA เอง ซึ่งแต่ละอย่างมี edge case ที่อันตราย

#### เปรียบเทียบ Auth Provider

| เกณฑ์ | Better Auth | Clerk | Auth.js (v5) |
|--------|-------------|-------|-------------|
| **License** | MIT (open source) | Proprietary SaaS | MIT |
| **ราคา** | ฟรีทั้งหมด (self-hosted) | ฟรี 10K MAU, จ่ายหลังจากนั้น | ฟรี |
| **Cloudflare D1** | ✅ Native adapter | ❌ ไม่รองรับ D1 | ⚠️ ต้อง custom adapter |
| **Hono** | ✅ First-class integration | ❌ ต้อง adapter | ⚠️ Community adapter |
| **Self-hosted** | ✅ ข้อมูลอยู่ใน D1 ของเรา | ❌ ข้อมูลอยู่ Clerk cloud | ✅ |
| **RBAC** | ✅ Plugin built-in | ✅ | ❌ ต้องสร้างเอง |
| **2FA/Passkeys** | ✅ Plugin built-in | ✅ | ⚠️ ต้อง config เพิ่ม |
| **OAuth (Google, FB)** | ✅ 30+ providers | ✅ | ✅ |
| **Email verification** | ✅ Built-in | ✅ | ✅ |
| **Forgot password** | ✅ Built-in | ✅ | ⚠️ Manual |
| **Session management** | ✅ DB-backed sessions | ✅ | ✅ JWT or DB |
| **Drizzle ORM** | ✅ Auto-generate schema | ❌ | ⚠️ Adapter needed |
| **Bundle size** | เล็ก (tree-shakeable) | N/A (API call) | ปานกลาง |

#### ทำไมเลือก Better Auth

1. **Cloudflare native** — มี adapter สำหรับ D1, KV, R2 พร้อมใช้งาน รวมถึง CLI tool `better-auth-cloudflare` ที่ generate project พร้อม D1/KV/R2 provisioning
2. **Hono first-class** — มี official example จาก Hono team: `hono.dev/examples/better-auth-on-cloudflare`
3. **ข้อมูลอยู่กับเรา** — self-hosted, ข้อมูล user ทั้งหมดอยู่ใน D1 ของเรา ไม่พึ่ง third-party cloud
4. **ฟรีตลอด** — MIT license, ไม่มี usage limit, ไม่มีค่าใช้จ่ายเพิ่มเมื่อ scale
5. **Plugin ecosystem** — RBAC, 2FA, passkeys, magic link, OAuth ใช้ plugin เปิดปิดได้
6. **Drizzle integration** — auto-generate tables (`user`, `session`, `account`, `verification`) ผ่าน Drizzle schema ไม่ต้องเขียน migration เอง

#### Known Issues

- **Session refresh กับ secondary storage** (GitHub issue #4203) — workaround: ปิด `cookieCache` option
- ยังอยู่ใน active development — ควรล็อค version ใน `package.json`

---

### ภาพรวม Auth ระบบใหม่

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT REQUEST                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  GATEWAY WORKER (Hono)                                   │
│                                                          │
│  1. Rate Limiter  ─── Cloudflare Rate Limiting Rules     │
│  2. CORS          ─── Origin allowlist from config       │
│  3. Auth Router:                                         │
│     ├── Public routes       → ไม่ต้องมี token            │
│     ├── Customer routes     → JWT required (customer)    │
│     ├── Staff routes        → JWT required (staff+)      │
│     ├── Manager routes      → JWT required (manager+)    │
│     └── Admin routes        → JWT required (super_admin) │
│  4. Token Blacklist ─── Check KV before allow            │
│  5. RBAC Middleware ─── Verify role has permission        │
│                                                          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  APPLICATION LAYER (Use Cases)                           │
└─────────────────────────────────────────────────────────┘
```

**หลักการออกแบบ:**

- **Better Auth จัดการ core auth** — login, register, logout, session management, password hashing, email verification, forgot password, OAuth ผ่าน Better Auth library ไม่ต้องเขียนเอง
- **Registered users เท่านั้นที่มี account** — ไม่มี guest record ใน DB
- **Anonymous checkout ไม่ต้อง login** — ข้อมูลลูกค้าฝังใน order record (custom logic, ไม่ผ่าน Better Auth)
- **Session-based auth (Better Auth default)** — Better Auth ใช้ DB-backed sessions แทน stateless JWT; เสริมด้วย KV cache ที่ edge สำหรับ performance
- **RBAC 4 ระดับ** — customer, staff, manager, super_admin ผ่าน Better Auth RBAC plugin + custom permissions matrix
- **Secrets ไม่อยู่ใน code** — ใช้ Wrangler secrets ทั้งหมด

---

### User Schema

> **Note:** Better Auth auto-manages 4 core tables: `user`, `session`, `account`, `verification` ผ่าน Drizzle schema generation (`npx @better-auth/cli generate`). เราเพิ่ม custom fields ลงใน `user` table ผ่าน Better Auth config ไม่ต้องเขียน migration เอง

#### Better Auth Core Tables (Auto-managed)

```typescript
// auth.ts — Better Auth configuration (Drizzle + D1)
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "./db"; // Drizzle D1 instance

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "sqlite" }),
  emailAndPassword: { enabled: true },
  user: {
    // Custom fields เพิ่มเข้า user table
    additionalFields: {
      phone: { type: "string", required: false },
      role: { 
        type: "string", 
        required: true, 
        defaultValue: "customer",
        input: false, // ไม่ให้ user set เอง ต้องผ่าน admin
      },
    },
  },
  session: {
    // ปิด cookieCache เพื่อหลีกเลี่ยง bug #4203 กับ D1
    cookieCache: { enabled: false },
    expiresIn: 60 * 60 * 24 * 7, // 7 days
    updateAge: 60 * 60 * 24,     // refresh ทุกวัน
  },
});
```

#### Better Auth Auto-generated Tables

```sql
-- user table (auto-generated + custom fields)
CREATE TABLE user (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  email           TEXT NOT NULL UNIQUE,
  emailVerified   INTEGER NOT NULL DEFAULT 0,
  image           TEXT,
  createdAt       TEXT NOT NULL,
  updatedAt       TEXT NOT NULL,
  -- Custom fields (เพิ่มผ่าน additionalFields)
  phone           TEXT,
  role            TEXT NOT NULL DEFAULT 'customer'
);

-- session table (auto-generated, DB-backed sessions)
CREATE TABLE session (
  id              TEXT PRIMARY KEY,
  expiresAt       TEXT NOT NULL,
  token           TEXT NOT NULL UNIQUE,   -- session token (ส่งใน cookie)
  createdAt       TEXT NOT NULL,
  updatedAt       TEXT NOT NULL,
  ipAddress       TEXT,
  userAgent       TEXT,
  userId          TEXT NOT NULL REFERENCES user(id)
);

-- account table (auto-generated, สำหรับ OAuth providers)
CREATE TABLE account (
  id                  TEXT PRIMARY KEY,
  accountId           TEXT NOT NULL,
  providerId          TEXT NOT NULL,       -- 'credential', 'google', 'facebook'
  userId              TEXT NOT NULL REFERENCES user(id),
  accessToken         TEXT,
  refreshToken        TEXT,
  idToken             TEXT,
  accessTokenExpiresAt TEXT,
  refreshTokenExpiresAt TEXT,
  scope               TEXT,
  password            TEXT,               -- hashed password (สำหรับ credential provider)
  createdAt           TEXT NOT NULL,
  updatedAt           TEXT NOT NULL
);

-- verification table (auto-generated, สำหรับ email verification + forgot password)
CREATE TABLE verification (
  id              TEXT PRIMARY KEY,
  identifier      TEXT NOT NULL,          -- email address
  value           TEXT NOT NULL,           -- verification token
  expiresAt       TEXT NOT NULL,
  createdAt       TEXT,
  updatedAt       TEXT
);
```

**สิ่งที่เปลี่ยนจากระบบเดิม:**

- ไม่มี `guest_id`, `is_admin`, `username` — ลดความซ้ำซ้อน
- ไม่ต้องจัดการ password hashing เอง — Better Auth ใช้ scrypt (built-in) หรือ config เป็น Argon2id ได้
- `role` เป็น custom field ใน `user` table — เพียงพอสำหรับ 4 roles
- `session` table แทน stateless JWT — revoke session ได้ทันทีโดยไม่ต้อง blacklist
- `account` table รองรับ OAuth ในอนาคต (Google, Facebook) โดยไม่ต้อง schema change
- `verification` table จัดการ email verification + forgot password ให้อัตโนมัติ

#### Addresses Table (แยกจาก User)

```sql
CREATE TABLE addresses (
  id              TEXT PRIMARY KEY,
  user_id         TEXT NOT NULL REFERENCES users(id),
  label           TEXT DEFAULT 'default',  -- 'default', 'home', 'work'
  recipient_name  TEXT NOT NULL,
  phone           TEXT,
  country_code    TEXT NOT NULL DEFAULT 'TH',
  country_name    TEXT NOT NULL DEFAULT 'Thailand',
  province        TEXT NOT NULL,
  city            TEXT NOT NULL,
  district        TEXT,                    -- ตำบล/แขวง
  postal_code     TEXT NOT NULL,
  address_line    TEXT NOT NULL,           -- บ้านเลขที่ ถนน ฯลฯ
  is_default      INTEGER DEFAULT 0,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_addresses_user ON addresses(user_id);
```

#### Corporate Addresses (Optional, for Tax Invoice)

```sql
CREATE TABLE corporate_addresses (
  id              TEXT PRIMARY KEY,
  user_id         TEXT NOT NULL REFERENCES users(id),
  company_name    TEXT NOT NULL,
  tax_id          TEXT NOT NULL,
  branch          TEXT DEFAULT 'สำนักงานใหญ่',
  address_line    TEXT NOT NULL,
  province        TEXT NOT NULL,
  city            TEXT NOT NULL,
  postal_code     TEXT NOT NULL,
  phone           TEXT,
  email           TEXT,
  created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

### Authentication Flows

> **Note:** Better Auth จัดการ auth flows ให้ทั้งหมด — เราแค่ mount handler ใน Hono แล้ว Better Auth จะ handle login, register, logout, session, email verification, forgot password ให้อัตโนมัติ

#### Hono + Better Auth Integration

```typescript
// src/index.ts — mount Better Auth handler ใน Hono
import { Hono } from "hono";
import { auth } from "./auth"; // Better Auth instance

const app = new Hono();

// Mount Better Auth — handles all /api/auth/* routes automatically
app.on(["POST", "GET"], "/api/auth/**", (c) => {
  return auth.handler(c.req.raw);
});
```

Better Auth provides these endpoints automatically:

| Endpoint | Method | หน้าที่ |
|----------|--------|--------|
| `/api/auth/sign-up/email` | POST | Register ด้วย email + password |
| `/api/auth/sign-in/email` | POST | Login ด้วย email + password |
| `/api/auth/sign-out` | POST | Logout (ลบ session จาก DB) |
| `/api/auth/session` | GET | ดึง current session + user |
| `/api/auth/forget-password` | POST | ส่ง reset password email |
| `/api/auth/reset-password` | POST | Reset password ด้วย token |
| `/api/auth/change-password` | POST | เปลี่ยน password (ต้อง login) |
| `/api/auth/verify-email` | GET | Verify email จาก link |
| `/api/auth/sign-in/social` | POST | OAuth login (Google, Facebook) |

#### Login Flow (Better Auth)

```
Client                        Hono Worker                    D1 Database
  │                                │                               │
  │  POST /api/auth/sign-in/email  │                               │
  │  { email, password }           │                               │
  │ ─────────────────────────────► │                               │
  │                                │  Better Auth:                 │
  │                                │  1. Lookup user by email      │
  │                                │  2. Verify password (scrypt)  │
  │                                │  3. Create session record     │
  │                                │ ─────────────────────────────►│
  │                                │                               │
  │  Set-Cookie: session_token=... │                               │
  │  { user, session }             │                               │
  │ ◄───────────────────────────── │                               │
```

#### Registration Flow (Better Auth)

```
Client                        Hono Worker                    D1 Database
  │                                │                               │
  │  POST /api/auth/sign-up/email  │                               │
  │  { name, email, password }     │                               │
  │ ─────────────────────────────► │                               │
  │                                │  Better Auth:                 │
  │                                │  1. Validate input            │
  │                                │  2. Check email uniqueness    │
  │                                │  3. Hash password             │
  │                                │  4. INSERT user (role=customer)│
  │                                │  5. INSERT account (credential)│
  │                                │  6. Create session            │
  │                                │ ─────────────────────────────►│
  │                                │                               │
  │  Set-Cookie: session_token=... │                               │
  │  { user, session }             │                               │
  │ ◄───────────────────────────── │                               │
```

#### Logout Flow (Better Auth)

```
Client                        Hono Worker                    D1 Database
  │                                │                               │
  │  POST /api/auth/sign-out       │                               │
  │  Cookie: session_token=...     │                               │
  │ ─────────────────────────────► │                               │
  │                                │  Better Auth:                 │
  │                                │  1. ลบ session record จาก DB  │
  │                                │  2. Clear cookie              │
  │                                │ ─────────────────────────────►│
  │                                │                               │
  │  Clear-Cookie: session_token   │                               │
  │  { success: true }             │                               │
  │ ◄───────────────────────────── │                               │
```

**ข้อดีเทียบกับ JWT blacklist เดิม:** Logout ทำงานทันทีเพราะลบ session จาก DB โดยตรง ไม่ต้องพึ่ง KV blacklist

#### Password Change Flow (Better Auth)

Better Auth จัดการ change password ให้อัตโนมัติ:

```
POST /api/auth/change-password
Cookie: session_token=...
{ currentPassword, newPassword }

Better Auth handles:
1. Verify session → get userId
2. Verify currentPassword against stored hash
3. Hash newPassword → UPDATE account.password
4. Optionally revoke other sessions (revokeOtherSessions: true)
```

---

### Session Management (Better Auth)

> **Note:** Better Auth ใช้ DB-backed sessions แทน stateless JWT ทำให้ logout ทำงานทันที (ลบ session จาก DB) โดยไม่ต้องพึ่ง token blacklist

#### Session Strategy

```
┌──────────────────────────────────────────────────────────┐
│  Session Token                                            │
│  ├── เก็บใน HttpOnly cookie (secure, sameSite: lax)      │
│  ├── ส่งอัตโนมัติโดย browser ทุก request                  │
│  ├── Better Auth จัดการ cookie ให้                        │
│  └── Server verify โดย lookup session จาก D1              │
│                                                           │
│  Session Record (D1)                                      │
│  ├── expiresAt: 7 วัน (configurable)                     │
│  ├── updateAge: refresh ทุก 24 ชม.                       │
│  ├── เก็บ ipAddress + userAgent                           │
│  └── ลบ = logout ทันที (ไม่ต้อง blacklist)                │
└──────────────────────────────────────────────────────────┘
```

#### Session Configuration

| Parameter | Value | เหตุผล |
|-----------|-------|--------|
| Session TTL | 7 วัน | เหมาะกับ e-commerce — ไม่สั้นเกินไปสำหรับลูกค้า |
| Update Age | 24 ชม. | Refresh session ทุกวัน ลด D1 writes |
| Cookie Cache | ปิด | หลีกเลี่ยง bug #4203 กับ D1 secondary storage |
| Cookie Name | `better-auth.session_token` | Default ของ Better Auth |
| SameSite | Lax | รองรับ OAuth redirect |
| HttpOnly | true | ป้องกัน XSS |
| Secure | true | HTTPS only |

#### Edge Session Cache (KV Enhancement)

แม้ Better Auth จัดการ sessions ใน D1 แต่เราเพิ่ม KV cache สำหรับ session validation ที่ edge เพื่อลด D1 round-trips:

```typescript
// middleware/session-cache.ts
// Cache session validation result ใน KV เพื่อลด D1 queries
export async function getCachedSession(
  sessionToken: string, 
  kv: KVNamespace,
  auth: ReturnType<typeof betterAuth>
) {
  const cacheKey = `session:${sessionToken}`;
  
  // 1. Check KV cache first (~1ms)
  const cached = await kv.get(cacheKey, "json");
  if (cached) return cached;
  
  // 2. Cache miss → query D1 via Better Auth (~10-50ms)
  const session = await auth.api.getSession({ 
    headers: new Headers({ cookie: `better-auth.session_token=${sessionToken}` })
  });
  
  if (session) {
    // Cache for 5 minutes (shorter than session TTL)
    await kv.put(cacheKey, JSON.stringify(session), { expirationTtl: 300 });
  }
  
  return session;
}
```

#### Logout ทุกอุปกรณ์ (Logout All)

Better Auth รองรับ revoke all sessions ของ user:

```typescript
// POST /api/auth/revoke-all-sessions
app.post('/api/auth/revoke-all-sessions', async (c) => {
  const session = c.get('session');
  
  // Better Auth: ลบทุก session ของ user จาก D1
  await auth.api.revokeSessions({
    body: { userId: session.user.id },
  });
  
  // Clear KV cache for this user's sessions
  // (KV entries จะ expire เองใน 5 นาที แต่ clear ทันทีดีกว่า)
  
  return c.json({ success: true, message: 'All sessions revoked' });
});
```

**ข้อดีเทียบกับ JWT + KV blacklist เดิม:**

| เกณฑ์ | JWT + KV Blacklist | Better Auth Sessions |
|--------|-------------------|---------------------|
| Logout ทันที | ⚠️ ต้อง blacklist + TTL | ✅ ลบ session จาก DB |
| Logout all devices | ⚠️ ต้อง tokenValidAfter workaround | ✅ `revokeSessions` built-in |
| Token ขนาด | ใหญ่ (JWT payload ใน header ทุก request) | เล็ก (session token สั้นๆ ใน cookie) |
| Complexity | สูง (sign, verify, refresh, blacklist) | ต่ำ (Better Auth จัดการให้) |
| D1 dependency | ไม่มี (KV only) | ✅ ทุก request ต้อง check D1 (แก้ด้วย KV cache) |

---

### RBAC — 4 Roles + Permissions

> **Note:** ใช้ Better Auth RBAC plugin เป็น base สำหรับ role management แล้วเพิ่ม custom permissions matrix ด้านบน เนื่องจาก Better Auth RBAC plugin รองรับ roles + permissions แต่เราต้องการ hierarchical permission inheritance ที่ซับซ้อนกว่า

#### Better Auth RBAC Plugin Setup

```typescript
// auth.ts — เพิ่ม RBAC plugin
import { betterAuth } from "better-auth";
import { rbac } from "better-auth/plugins";

export const auth = betterAuth({
  // ...database config...
  plugins: [
    rbac({
      roles: {
        customer: { description: "สั่งซื้อ + ดูประวัติ + จัดการ profile" },
        staff: { description: "จัดการ products + orders + ดู dashboard" },
        manager: { description: "ดู reports + approve refunds + จัดการ orders" },
        super_admin: { description: "ทำได้ทุกอย่าง + จัดการ users + settings" },
      },
    }),
  ],
});
```

#### Role Hierarchy

```
super_admin  (ทำได้ทุกอย่าง + จัดการ users + settings)
    │
manager      (ดู reports + approve refunds + จัดการ orders)
    │
staff        (จัดการ products + orders + ดู dashboard)
    │
customer     (สั่งซื้อ + ดูประวัติ + จัดการ profile)
```

#### Permissions Matrix

| Permission | customer | staff | manager | super_admin |
|------------|----------|-------|---------|-------------|
| **Profile** | | | | |
| `profile:read` | ✅ | ✅ | ✅ | ✅ |
| `profile:write` | ✅ | ✅ | ✅ | ✅ |
| **Orders** | | | | |
| `orders:create` | ✅ | ✅ | ✅ | ✅ |
| `orders:read_own` | ✅ | ✅ | ✅ | ✅ |
| `orders:read_all` | ❌ | ✅ | ✅ | ✅ |
| `orders:update_status` | ❌ | ✅ | ✅ | ✅ |
| `orders:refund` | ❌ | ❌ | ✅ | ✅ |
| `orders:delete` | ❌ | ❌ | ❌ | ✅ |
| **Products** | | | | |
| `products:read` | ✅ | ✅ | ✅ | ✅ |
| `products:write` | ❌ | ✅ | ✅ | ✅ |
| `products:delete` | ❌ | ❌ | ✅ | ✅ |
| **Catalog (brands, categories, sizes, colors)** | | | | |
| `catalog:read` | ✅ | ✅ | ✅ | ✅ |
| `catalog:write` | ❌ | ✅ | ✅ | ✅ |
| `catalog:delete` | ❌ | ❌ | ✅ | ✅ |
| **Collections & Promotions** | | | | |
| `collections:read` | ✅ | ✅ | ✅ | ✅ |
| `collections:write` | ❌ | ✅ | ✅ | ✅ |
| `promotions:read` | ✅ | ✅ | ✅ | ✅ |
| `promotions:write` | ❌ | ✅ | ✅ | ✅ |
| **Reports** | | | | |
| `reports:read` | ❌ | ❌ | ✅ | ✅ |
| `reports:generate` | ❌ | ❌ | ✅ | ✅ |
| **Dashboard** | | | | |
| `dashboard:read` | ❌ | ✅ | ✅ | ✅ |
| **Users** | | | | |
| `users:read` | ❌ | ❌ | ✅ | ✅ |
| `users:write` | ❌ | ❌ | ❌ | ✅ |
| `users:delete` | ❌ | ❌ | ❌ | ✅ |
| `users:assign_role` | ❌ | ❌ | ❌ | ✅ |
| **Files** | | | | |
| `files:upload` | ❌ | ✅ | ✅ | ✅ |
| `files:delete` | ❌ | ❌ | ✅ | ✅ |
| **Shipping** | | | | |
| `shipping:read` | ✅ | ✅ | ✅ | ✅ |
| `shipping:write` | ❌ | ❌ | ✅ | ✅ |
| **Settings** | | | | |
| `settings:read` | ❌ | ❌ | ❌ | ✅ |
| `settings:write` | ❌ | ❌ | ❌ | ✅ |

#### RBAC Implementation

```typescript
// domain/auth/permissions.ts
export type UserRole = 'customer' | 'staff' | 'manager' | 'super_admin';
export type Permission = string;

const ROLE_PERMISSIONS: Record<UserRole, Permission[]> = {
  customer: [
    'profile:read', 'profile:write',
    'orders:create', 'orders:read_own',
    'products:read', 'catalog:read',
    'collections:read', 'promotions:read',
    'shipping:read',
  ],
  staff: [
    // Inherits all customer permissions +
    'orders:read_all', 'orders:update_status',
    'products:write',
    'catalog:write',
    'collections:write', 'promotions:write',
    'dashboard:read',
    'files:upload',
  ],
  manager: [
    // Inherits all staff permissions +
    'orders:refund',
    'products:delete', 'catalog:delete',
    'reports:read', 'reports:generate',
    'users:read',
    'files:delete',
    'shipping:write',
  ],
  super_admin: [
    // Inherits all manager permissions +
    'orders:delete',
    'users:write', 'users:delete', 'users:assign_role',
    'settings:read', 'settings:write',
  ],
};

// Flatten with inheritance
export function getPermissions(role: UserRole): Set<Permission> {
  const hierarchy: UserRole[] = ['customer', 'staff', 'manager', 'super_admin'];
  const roleIndex = hierarchy.indexOf(role);
  
  const allPerms = new Set<Permission>();
  for (let i = 0; i <= roleIndex; i++) {
    ROLE_PERMISSIONS[hierarchy[i]].forEach(p => allPerms.add(p));
  }
  return allPerms;
}

export function hasPermission(role: UserRole, permission: Permission): boolean {
  return getPermissions(role).has(permission);
}
```

#### RBAC Middleware

```typescript
// middleware/rbac.ts
import { hasPermission, Permission, UserRole } from '../domain/auth/permissions';

export function requirePermission(...permissions: Permission[]) {
  return async (c: Context, next: Next) => {
    const user = c.get('user') as JwtPayload;
    
    if (!user) {
      return c.json({ error: 'Authentication required' }, 401);
    }

    const hasAll = permissions.every(p => hasPermission(user.role as UserRole, p));
    
    if (!hasAll) {
      return c.json({ 
        error: 'Forbidden',
        message: `Required permissions: ${permissions.join(', ')}`,
      }, 403);
    }

    await next();
  };
}

// Usage in routes:
app.get('/api/admin/reports', 
  requirePermission('reports:read'),
  reportHandler
);

app.patch('/api/admin/orders/:id/refund',
  requirePermission('orders:refund'),
  refundHandler
);

app.post('/api/admin/users',
  requirePermission('users:write'),
  createUserHandler
);
```

---

### Anonymous Checkout Flow

#### Order Schema (รองรับ Anonymous)

```sql
CREATE TABLE orders (
  id                   TEXT PRIMARY KEY,
  order_number         TEXT NOT NULL UNIQUE,
  status               TEXT NOT NULL DEFAULT 'payment_pending',
  
  -- Customer Info (Snapshot — ไม่ FK ไป users)
  user_id              TEXT,              -- NULL สำหรับ anonymous
  customer_email       TEXT NOT NULL,     -- ต้องมีเสมอ (สำหรับส่ง receipt)
  customer_name        TEXT NOT NULL,
  customer_phone       TEXT,
  shipping_address     TEXT NOT NULL,     -- JSON: { line, province, city, ... }
  billing_address      TEXT,             -- JSON (nullable, for tax invoice)
  
  -- Payment
  payment_ref          TEXT,              -- Omise charge ID (chrg_xxx)
  payment_method       TEXT,              -- credit_card, promptpay
  payment_token        TEXT,              -- Token ที่ใช้ charge
  
  -- Amounts (DECIMAL ไม่ใช่ float!)
  subtotal             REAL NOT NULL,     -- ก่อนส่วนลด
  discount_total       REAL DEFAULT 0,
  shipping_cost        REAL DEFAULT 0,
  total_amount         REAL NOT NULL,     -- สุทธิ
  currency             TEXT DEFAULT 'THB',
  
  -- Fulfillment
  tracking_url         TEXT,
  
  -- Timestamps
  created_at           TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at           TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_orders_payment_ref ON orders(payment_ref);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_email ON orders(customer_email);
```

#### Anonymous Checkout Flow Diagram

```
┌───────────────────────────────────────────────────────────┐
│                  ANONYMOUS CHECKOUT                        │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  1. Client ส่ง POST /api/checkout                         │
│     {                                                      │
│       customerEmail: "jane@example.com",                   │
│       customerName: "Jane Doe",                            │
│       shippingAddress: { ... },                            │
│       items: [{ sku, qty, price }],                        │
│       paymentToken: "tokn_xxx"                             │
│     }                                                      │
│                                                            │
│  2. Zod validate ทุก field                                │
│     - email format ✓                                       │
│     - items.length > 0 ✓                                   │
│     - paymentToken starts with tokn_ or src_ ✓             │
│                                                            │
│  3. สร้าง Order record (user_id = NULL)                   │
│     - Generate order_number                                │
│     - Snapshot customer info ลงใน order                    │
│     - Snapshot items ลงใน order_items (product_snapshot)   │
│                                                            │
│  4. Charge via Omise                                       │
│     - Card → Omise.charge() → payment_ref = chrg_xxx      │
│     - PromptPay → Omise.chargeWithSource() → QR code      │
│                                                            │
│  5. Response:                                              │
│     - Card (3DS) → { authUrl, orderId }                   │
│     - Card (success) → { orderId, status: success }       │
│     - PromptPay → { orderId, qrCode, expireAt }           │
│                                                            │
│  6. Webhook (charge.complete):                             │
│     - Lookup order WHERE payment_ref = chrg_xxx            │
│     - Update status → deduct stock → send email            │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

#### Registered User Checkout (ต่างจาก Anonymous)

ถ้า user login อยู่ → ส่ง JWT มาด้วย:

```typescript
app.post('/api/checkout', optionalAuth(), async (c) => {
  const body = c.req.valid('json');
  const user = c.get('user'); // nullable — ไม่มี = anonymous
  
  const order = await createOrderUseCase.execute({
    userId: user?.sub ?? null,              // NULL for anonymous
    customerEmail: body.customerEmail,       // ต้องมีเสมอ
    customerName: body.customerName,
    shippingAddress: body.shippingAddress,
    items: body.items,
    paymentToken: body.paymentToken,
  });

  return c.json(order);
});
```

#### Order Lookup (Anonymous vs Registered)

```typescript
// Anonymous ดู order ผ่าน email + order number
app.get('/api/orders/lookup', async (c) => {
  const email = c.req.query('email');
  const orderNumber = c.req.query('orderNumber');
  
  // ต้องรู้ทั้ง 2 อย่าง → ป้องกัน enumeration
  const order = await orderRepo.findByEmailAndNumber(email, orderNumber);
  if (!order) return c.json({ error: 'Order not found' }, 404);
  
  return c.json(order);
});

// Registered user ดูทุก order ของตัวเอง
app.get('/api/me/orders',
  requirePermission('orders:read_own'),
  async (c) => {
    const user = c.get('user');
    const orders = await orderRepo.findByUserId(user.sub);
    return c.json(orders);
  }
);
```

---

### Auth Middleware Stack

#### Middleware Execution Order

```typescript
// gateway.ts — middleware ทำงานตามลำดับ:

// 1. Rate Limiting (ก่อนทุกอย่าง)
app.use('*', rateLimiter());

// 2. CORS
app.use('*', cors({
  origin: (origin) => {
    const allowed = env.ALLOWED_ORIGINS.split(',');
    return allowed.includes(origin) ? origin : null;
  },
  allowMethods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Authorization', 'Content-Type'],
  credentials: true,
  maxAge: 86400,
}));

// 3. Request logging
app.use('*', structuredLogger());

// 4. Auth — only on protected routes (Better Auth session-based)
app.use('/api/me/*', sessionAuth());           // Customer routes
app.use('/api/admin/*', sessionAuth());        // Admin routes

// 5. RBAC — per route group
app.use('/api/admin/reports/*', requirePermission('reports:read'));
app.use('/api/admin/users/*', requirePermission('users:read'));
```

#### Session Auth Middleware (Better Auth)

```typescript
// middleware/session-auth.ts
import { auth } from '../auth';

export function sessionAuth() {
  return async (c: Context, next: Next) => {
    // Better Auth: verify session จาก cookie → lookup D1 (หรือ KV cache)
    const session = await auth.api.getSession({
      headers: c.req.raw.headers,
    });
    
    if (!session) {
      return c.json({ error: 'Authentication required' }, 401);
    }
    
    // Set session + user in Hono context
    c.set('session', session.session);
    c.set('user', session.user);
    
    await next();
  };
}
```

#### Optional Auth Middleware (for checkout)

```typescript
// middleware/optional-auth.ts
// ไม่ reject ถ้าไม่มี session — แค่ set user = null (สำหรับ anonymous checkout)
export function optionalAuth() {
  return async (c: Context, next: Next) => {
    try {
      const session = await auth.api.getSession({
        headers: c.req.raw.headers,
      });
      
      if (session) {
        c.set('session', session.session);
        c.set('user', session.user);
      }
    } catch {
      // No valid session → treat as anonymous
    }
    
    c.set('user', c.get('user') ?? null);
    await next();
  };
}
```

---

### Route Protection Matrix

| Route Pattern | Auth | Middleware | หมายเหตุ |
|---------------|------|-----------|----------|
| `GET /` | ❌ | — | Health check |
| `GET /api/storefront/*` | ❌ | — | Public catalog, promotions, collections |
| `POST /api/checkout` | Optional | `optionalAuth()` | Anonymous + registered |
| `GET /api/orders/lookup` | ❌ | Rate limit (strict) | Email + order number required |
| `POST /api/webhooks/omise` | ❌ | `verifyOmiseSignature()` | Webhook signature check |
| `/api/auth/**` | ❌ | Better Auth handler | Better Auth จัดการ login/register/logout/session ให้ |
| `POST /api/auth/sign-in/email` | ❌ | Rate limit (strict) | 5 attempts/min/IP |
| `POST /api/auth/sign-up/email` | ❌ | Rate limit (moderate) | 10/min/IP |
| `GET /api/me/*` | ✅ Session | `sessionAuth()` | Customer profile, orders |
| `PUT /api/me/profile` | ✅ Session | `sessionAuth()` + `requirePermission('profile:write')` | |
| `GET /api/admin/dashboard` | ✅ Session | `sessionAuth()` + `requirePermission('dashboard:read')` | staff+ |
| `GET /api/admin/orders` | ✅ Session | `sessionAuth()` + `requirePermission('orders:read_all')` | staff+ |
| `PATCH /api/admin/orders/:id/refund` | ✅ Session | `sessionAuth()` + `requirePermission('orders:refund')` | manager+ |
| `POST /api/admin/products` | ✅ Session | `sessionAuth()` + `requirePermission('products:write')` | staff+ |
| `DELETE /api/admin/products/:id` | ✅ Session | `sessionAuth()` + `requirePermission('products:delete')` | manager+ |
| `GET /api/admin/reports/*` | ✅ Session | `sessionAuth()` + `requirePermission('reports:read')` | manager+ |
| `POST /api/admin/users` | ✅ Session | `sessionAuth()` + `requirePermission('users:write')` | super_admin |
| `PUT /api/admin/users/:id/role` | ✅ Session | `sessionAuth()` + `requirePermission('users:assign_role')` | super_admin |

---

### Security Improvements vs Legacy

| ปัญหาเดิม | สถานะ | วิธีแก้ในระบบใหม่ |
|------------|--------|-------------------|
| JWT logout ไม่ทำงาน | ✅ แก้แล้ว | Better Auth DB-backed sessions — ลบ session = logout ทันที |
| ไม่มี RBAC | ✅ แก้แล้ว | Better Auth RBAC plugin + custom 4 roles + permissions matrix |
| Guest ปนกับ registered | ✅ แก้แล้ว | Anonymous checkout ไม่มี user record |
| CORS wildcard | ✅ แก้แล้ว | Explicit origin allowlist |
| Credentials ใน git | ✅ แก้แล้ว | Wrangler secrets |
| ไม่มี rate limiting | ✅ แก้แล้ว | Cloudflare Rate Limiting + per-route config |
| bcrypt (vulnerable to GPU) | ✅ ปรับปรุง | Better Auth scrypt (default) หรือ config Argon2id |
| 24hr token → long exposure | ✅ ปรับปรุง | Better Auth session 7d + auto-refresh ทุก 24 ชม. |
| Unauthenticated webhook | ✅ แก้แล้ว | `verifyOmiseSignature()` middleware |
| No input validation | ✅ แก้แล้ว | Zod validators on all endpoints |
| ไม่มี email verification | ✅ เพิ่มใหม่ | Better Auth built-in email verification |
| ไม่มี forgot password | ✅ เพิ่มใหม่ | Better Auth built-in password reset flow |
| ไม่มี OAuth | ✅ พร้อมเพิ่ม | Better Auth รองรับ 30+ OAuth providers (เปิดเมื่อต้องการ) |
| Unauthenticated order history | ✅ แก้แล้ว | Requires email + order number (anonymous) หรือ session (registered) |

---

### Auth Open Questions

| คำถาม | ผลกระทบ | สถานะ | หมายเหตุ |
|--------|---------|--------|----------|
| ต้องการ OAuth (Google/Facebook login) หรือไม่? | เพิ่ม OAuth provider config | ⏳ Week 2 | Better Auth รองรับ 30+ providers — แค่เพิ่ม config เมื่อต้องการ |
| Email verification flow | ต้องการ email service (Resend/SES) | ⏳ Week 2 | Better Auth built-in — แค่ต้อง config email transport |
| Forgot password flow | ต้องการ email service | ⏳ Week 2 | Better Auth built-in — ใช้ email link (ไม่ใช่ OTP) |
| Admin user แรกสร้างอย่างไร? | Deployment strategy | ⏳ Week 1 | Seed script ผ่าน Better Auth API หรือ direct D1 insert |
| Session concurrency limit | D1 query strategy | ⏳ Week 3 | Better Auth เก็บ sessions ใน D1 — query ได้ว่ามีกี่ session ต่อ user |
| ต้องการ audit log สำหรับ admin actions? | Schema + storage | ⏳ Week 3 | Custom implementation (ไม่เกี่ยวกับ Better Auth) |
| Email transport เลือก provider ไหน? (Resend vs SES vs Mailgun) | ค่าใช้จ่าย + setup | ⏳ Week 2 | จำเป็นสำหรับ email verification + forgot password |
| 2FA/Passkeys เปิดใช้งานเมื่อไหร่? | UX complexity | ⏳ Week 4+ | Better Auth มี plugin พร้อม — เปิดเมื่อ MVP เสร็จ |
