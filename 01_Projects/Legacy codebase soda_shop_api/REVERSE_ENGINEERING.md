# Soda Shop API — Reverse Engineering Report

**Analysis Date:** 2026-04-12
**Project:** E-commerce backend for SODA BKK (fashion/apparel brand)
**Stack:** Go 1.21 + Gin + GORM + PostgreSQL + Omise (Thai payment gateway)

---

## 1. Business Rules

### 1.1 Core Business Domain

This is a **B2C fashion e-commerce API** for "SODA BKK" — a Thai fashion brand selling apparel with size/color variants. The system supports:
- Product catalog with SKU-level variants (size × color)
- Online ordering with multiple payment methods (credit card, PromptPay)
- Shipping cost calculation by weight and country
- Promotion/coupon system
- Admin back-office (order management, reports, inventory)

### 1.2 Order State Machine

```
                  ┌──────────────┐
                  │   (Created)  │
                  └──────┬───────┘
                         │
              ┌──────────▼─────��────┐
              │   PAYMENT_PENDING   │  ← Initial state for async payments (PromptPay, 3DS)
              └���─────────┬─────────���┘
                         │ webhook: charge.complete (status=successful)
              ┌──────────▼──────────┐
              │   PAYMENT_SUCCESS   │  ← Payment confirmed (immediate for card, webhook for async)
              └─────���────┬──────────┘
                         │ Admin action: update status
              ┌──────────▼──────────┐
              │   SHIPPING_PENDING  │  ← Admin has acknowledged, preparing shipment
              └──────────┬──────────┘
                         │ Admin action: update status + tracking URL
              ┌──────────▼──────────��
              │ SHIPPING_COMPLETED  │  ← Delivered
              └─────────────────────┘

  CANCEL ← Admin can set this at any point (no stock reversal logic observed)
  VOID   ← Admin refunds via Omise → stock is restored to SKU
```

**Transitions:**
| From | To | Trigger | Side Effects |
|------|-----|---------|--------------|
| (new) | PAYMENT_PENDING | CreateOrder (async payment) | Order created, no stock deducted |
| (new) | PAYMENT_SUCCESS | CreateOrder (immediate card success) | Stock deducted, email sent |
| PAYMENT_PENDING | PAYMENT_SUCCESS | Omise webhook `charge.complete` | Stock deducted, email sent |
| PAYMENT_SUCCESS | SHIPPING_PENDING | Admin PATCH status | None |
| SHIPPING_PENDING | SHIPPING_COMPLETED | Admin PATCH status + trackingUrl | None |
| Any | VOID | Admin refund action | Omise refund API called, stock restored |
| Any | CANCEL | Admin PATCH status | None (no stock restoration) |

### 1.3 Validation Rules

| Rule | Location | Enforcement |
|------|----------|-------------|
| Email required + valid format for registration | `entity/register.go` | Gin binding tag `binding:"required,email"` |
| Password required for registration | `entity/register.go` | Gin binding tag `binding:"required"` |
| Unique email/username per user | `cmd/database/database.go` | DB unique index (case-insensitive) |
| Unique brand name | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Unique category name | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Unique promotion code | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Unique size code | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Unique color code | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Unique shipping country | `cmd/database/database.go` | DB unique index (case-insensitive, soft-delete aware) |
| Product must have primary gallery with colorID | `content_controller.go:254` | App-level: skips product from public listing if gallery is primary but colorID==0 |
| Product must have `isDisplay=true` | `content_controller.go:209` | App-level: filters out from storefront |
| Promotion must be published + within date range | `promotion_repo.go:17` | DB query filter |
| SKU defaults: price/weight/salePrice from parent product | `product_controller.go:41-49` | App-level on create |

### 1.4 Business Conditions

| Condition | Logic | Location |
|-----------|-------|----------|
| Payment method detection | Token starts with `tokn` → credit card; otherwise → source-based (PromptPay/banking) | `content_controller.go:393` |
| Stock deduction timing | Only on PAYMENT_SUCCESS (immediate or webhook) | `content_controller.go:439-460`, `content_controller.go:596-614` |
| Stock restoration | Only on VOID (refund), not on CANCEL | `order_controller.go:69-90` |
| Free shipping threshold | Per-country `freeShippingOnCost` field on ShippingCost | `entity/shipping.go:7` |
| Promotion validity | `is_published=true` AND `published_start_at <= now` AND `published_end_at >= now` | `promotion_repo.go:17` |
| Dashboard: "new orders today" | `PAYMENT_SUCCESS` + `CreatedAt.Day() == today` | `dashboard_controller.go:45` |
| Dashboard: "waiting confirm" | Status is `PAYMENT_SUCCESS` or `SHIPPING_PENDING` | `dashboard_controller.go:47` |
| Report location grouping | Bangkok vs Other Thai Province vs International (by address country code + province name "กรุงเทพมหานคร") | `report_controller.go:131-155` |
| Report VOID exclusion | VOID orders excluded from sales total, shown in separate section | `report_controller.go:121-122` |
| Collection visibility | Only `isPublished=true` collections shown to public, sorted by `order` field | `content_controller.go:628-638` |

### 1.5 Business Workflows

**Order Creation Flow:**
1. Client submits order with items, payment token, user/address info
2. System generates order number: `{sequential}-{userId}-{uuid8}`
3. System generates order item codes: `{orderNumber}-{productId}-{skuId}`
4. Payment charged via Omise (card or source)
5. If immediate success → deduct stock, send receipt email to customer + admin
6. If pending (3DS/PromptPay) → return payment details for client-side handling
7. Webhook later confirms → deduct stock, send email

**Refund Flow:**
1. Admin initiates refund on order
2. Omise refund API called with full order amount
3. Order status set to VOID
4. OrderStock records deleted for this order
5. Product SKU stock restored (qty added back)

**User Registration Flow:**
1. Guest: auto-generated password, role=GUEST, username=guestId
2. Registered: email as username, role=REGISTERED, bcrypt hashed password

**Client Profile Update:**
1. Can update firstName, lastName, tel
2. Can update/create first address (only index [0] is used)
3. Can update corporate address (only if companyName is non-empty)

---

## 2. API Inventory

### 2.1 Response Envelope

All endpoints return:
```json
{
  "success": true|false,
  "data": <payload | null>,
  "error": "<error message | empty>",
  "timestamp": "<ISO datetime>"
}
```

### 2.2 Public Endpoints (No Auth)

| Method | Path | Purpose | Request | Response Data |
|--------|------|---------|---------|---------------|
| GET | `/` | Health check | — | `"Service is up and running"` |
| POST | `/api/auth/login` | Admin login | Basic Auth header | `AuthResponse` |
| POST | `/api/auth/client-login` | Client login | Basic Auth header | `Account` (with token + orders) |
| DELETE | `/api/auth/logout` | Logout | JWT Bearer | `null` |
| GET | `/api/auth/client-profile` | Load client data | JWT Bearer | `Account` |
| PUT | `/api/auth/client-profile` | Update client data | JWT Bearer + `UpdateClientDataRequest` | `User` |
| GET | `/api/v1/contents/banner` | Get banners | — | `[]Content` |
| GET | `/api/v1/contents/menu` | Get navigation menu | — | `[]Menu` (dynamic from brands×categories) |
| GET | `/api/v1/contents/product` | Get storefront products | — | `[]ProductContent` |
| GET | `/api/v1/contents/shipping` | Get shipping rates | — | `[]ShippingContent` |
| GET | `/api/v1/contents/promotion` | Get active promotions | — | `[]Promotion` |
| GET | `/api/v1/contents/collections` | Get published collections | — | `[]Collection` |
| GET | `/api/v1/contents/order?userId=N` | Get user's orders | query: `userId` | `[]Order` |
| GET | `/api/v1/contents/charge/:chargeID` | Get Omise charge | path: `chargeID` | Omise Charge object |
| GET | `/api/v1/contents/checkPaymentStatus/:chargeID` | Check payment status | path: `chargeID` | `Order` |
| POST | `/api/v1/contents/account/guest` | Create guest | `User{guestId}` | `User` |
| POST | `/api/v1/contents/account/register` | Register user | `RegisterRequest` | `"Register success."` |
| POST | `/api/v1/contents/order` | Create order | `Order` (full) | `CreditCardResponse` or `PayWithSourceResponse` |
| POST | `/api/v1/contents/payment_webhook` | Omise webhook | Omise Event JSON | `"success"` |
| POST | `/api/v1/contents/test_email` | Test email | `Order{id}` | HTML content |
| PUT | `/api/v1/contents/banner` | Update banners (JWT required) | `[]Content` | `"success"` |

### 2.3 Authenticated Admin Endpoints (JWT Required)

#### Users
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/users` | List all users |
| GET | `/api/v1/users/:id` | Get user by ID |
| POST | `/api/v1/users` | Create user |
| PUT | `/api/v1/users/:id` | Update user |
| DELETE | `/api/v1/users/:id` | Soft delete user |

#### Brands
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/brands` | List all brands |
| GET | `/api/v1/brands/:id` | Get brand by ID |
| POST | `/api/v1/brands` | Create brand |
| PUT | `/api/v1/brands/:id` | Update brand |
| DELETE | `/api/v1/brands/:id` | Soft delete brand |

#### Categories
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/categories` | List all categories |
| GET | `/api/v1/categories/:id` | Get category by ID |
| POST | `/api/v1/categories` | Create category |
| PUT | `/api/v1/categories/:id` | Update category |
| DELETE | `/api/v1/categories/:id` | Soft delete category |

#### Sizes
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/sizes` | List all sizes |
| GET | `/api/v1/sizes/:id` | Get size by ID |
| POST | `/api/v1/sizes` | Create size |
| PUT | `/api/v1/sizes/:id` | Update size |
| DELETE | `/api/v1/sizes/:id` | Soft delete size |

#### Colors
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/colors` | List all colors |
| GET | `/api/v1/colors/:id` | Get color by ID |
| POST | `/api/v1/colors` | Create color |
| PUT | `/api/v1/colors/:id` | Update color |
| DELETE | `/api/v1/colors/:id` | Soft delete color |

#### Products
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/products` | List all products (admin view) |
| GET | `/api/v1/products/:id` | Get product by ID |
| POST | `/api/v1/products` | Create product with SKUs, galleries, size charts |
| PUT | `/api/v1/products/:id` | Full update product + nested |
| DELETE | `/api/v1/products/:id` | Soft delete product |
| PATCH | `/api/v1/products/:id` | Toggle `isDisplay` flag |

#### Collections
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/collections` | List all collections |
| GET | `/api/v1/collections/:id` | Get collection by ID |
| POST | `/api/v1/collections` | Create collection |
| PUT | `/api/v1/collections/:id` | Update collection |
| DELETE | `/api/v1/collections/:id` | Soft delete collection |

#### Promotions
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/promotions` | List all promotions |
| GET | `/api/v1/promotions/:id` | Get promotion by ID |
| POST | `/api/v1/promotions` | Create promotion |
| PUT | `/api/v1/promotions/:id` | Update promotion |
| DELETE | `/api/v1/promotions/:id` | Soft delete promotion |

#### Orders
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/orders` | List all orders |
| GET | `/api/v1/orders/:id` | Get order by ID |
| PATCH | `/api/v1/orders/:id/status` | Update order status + tracking |
| PATCH | `/api/v1/orders/:id/refund` | Refund order via Omise |
| POST | `/api/v1/orders/:id/invoice` | Send invoice email |
| DELETE | `/api/v1/orders/:id` | Soft delete order |

#### Shipping
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/shippings` | List shipping costs |
| GET | `/api/v1/shippings/:id` | Get shipping cost by ID |
| POST | `/api/v1/shippings` | Create shipping cost |
| PUT | `/api/v1/shippings/:id` | Update shipping cost |
| DELETE | `/api/v1/shippings/:id` | Soft delete shipping cost |

#### Files
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/files` | List files |
| POST | `/api/v1/files` | Upload file |
| DELETE | `/api/v1/files/:id` | Delete file |

#### Dashboard
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/dashboard` | Get dashboard metrics |

#### Reports (PDF generation)
| Method | Path | Request Body | Purpose |
|--------|------|--------------|---------|
| POST | `/api/v1/reports/daily-sales` | `{fromDate, toDate}` | Daily sales PDF |
| POST | `/api/v1/reports/daily-sales-group-model` | `{fromDate, toDate}` | Sales grouped by model PDF |
| POST | `/api/v1/reports/stock-group-model` | `{modelFrom, modelTo}` | Stock valuation PDF |

### 2.4 Key Request/Response Schemas

**RegisterRequest:**
```json
{
  "firstName": "string",
  "lastName": "string",
  "email": "string (required, valid email)",
  "password": "string (required)"
}
```

**CreateOrder Request (POST /api/v1/contents/order):**
```json
{
  "userId": 123,
  "totalPrice": 2500.00,
  "netTotalPrice": 2300.00,
  "totalDiscount": 200.00,
  "addressId": 1,
  "corporateAddressId": null,
  "omiseToken": "tokn_xxxx | src_xxxx",
  "shippingCost": 50.0,
  "orderDetails": [
    { "productId": 1, "skuId": 5, "qty": 2, "price": 1250.0 }
  ],
  "promotions": [
    { "promotionId": 1, "promotionName": "SALE20", "discount": 200 }
  ]
}
```

**CreditCardResponse:**
```json
{
  "paymentStatus": "PAYMENT_SUCCESS|PAYMENT_PENDING",
  "paymentMethod": "credit_card",
  "authUrl": "https://3ds-redirect-url..."
}
```

**PayWithSourceResponse (PromptPay):**
```json
{
  "sourceId": "src_xxx",
  "paymentId": "chrg_xxx",
  "paymentStatus": "PAYMENT_PENDING",
  "paymentMethod": "promptpay",
  "qrCode": "https://qr-image-url...",
  "expireAt": "2026-04-12T..."
}
```

**AuthResponse (Login):**
```json
{
  "token": "jwt-string",
  "userId": 1,
  "username": "admin",
  "firstName": "John",
  "lastName": "Doe",
  "isAdmin": true,
  "role": "SUPER_ADMIN"
}
```

**Account (Client Login/Profile):**
```json
{
  "id": 1,
  "guestId": "",
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane@example.com",
  "tel": "0812345678",
  "role": "REGISTERED",
  "orders": [],
  "addressId": 1,
  "addresses": [],
  "corporateAddressId": null,
  "corporateAddress": null,
  "countryCode": "TH",
  "token": "jwt-string"
}
```

**Dashboard:**
```json
{
  "totalNewOrderToday": 5,
  "totalOrderWaitingAdminConfirm": 12,
  "totalCompleted": 45,
  "totalUser": 200
}
```

---

## 3. Data Mapping (Database Structure)

### 3.1 Entity Relationship Diagram (Textual)

```
┌─────────────────────────────���───────────────────────────────────────┐
│                          USERS & AUTH                                │
├───────────────────────────────────────────────��─────────────────────┤
│ users ─── 1:N ──→ addresses                                         │
│ users ─── 1:1 ──→ corporate_addresses                               │
│ users ─── 1:N ──→ log_authens (JWT session tracking)                │
│ users ─── 1:N ──→ orders                                            │
└─────────���──────────────────────────────────────��────────────────────┘

┌────────────���───────────────────────────��────────────────────────────��
│                          PRODUCT CATALOG                             │
├──────────��──────────────────────────────────────────────────────────┤
│ products ─── N:1 ──→ brands                                         │
│ products ─── N:1 ──→ categories                                     │
│ products ─── 1:N ──→ product_skus (size × color variants)           │
│ products ─── 1:N ──→ product_galleries (images)                     │
│ products ─── 1:N ──→ product_size_charts                            │
│ products ─── 1:1 ──→ size_chart_images                              │
│ product_skus ─── N:1 ──→ sizes                                      │
│ product_skus ─── N:1 ──→ colors                                     │
│ product_galleries ─── N:1 ──→ file_uploads                          │
│ product_galleries ─── N:1 ──→ colors                                │
│ product_size_charts ──�� N:1 ──→ size_charts                         │
��� product_size_charts ─── N:1 ──→ sizes                               │
│ categories ─── 1:N ──→ size_charts                                  │
└─────────────���──────────────────────────────────��────────────────────┘

┌─────────────────��───────────────────────────────────────────────────┐
│                          ORDERS & PAYMENTS                           │
├─────────��─────────────────────────���───────────────────────────��─────┤
│ orders ─── N:1 ──→ users                                            │
│ orders ─── N:1 ──→ addresses (nullable)                             │
│ orders ─── N:1 ──→ corporate_addresses (nullable)                   │
│ orders ─── 1:N ──→ order_details                                    │
│ orders ─── 1:N ──→ order_promotions                                 │
│ orders ─── 1:N ──→ order_stocks (stock deduction records)           │
│ order_details ─── N:1 ──→ products                                  │
│ order_details ─── N:1 ──→ product_skus                              │
│ order_promotions ─── N:1 ──→ promotions                             │
└─────────────────��──────────────────────────────���────────────────────┘

┌──────────────��───────────────────────────────���──────────────────────┐
│                        PROMOTIONS & COLLECTIONS                      │
├───────────────────────────────────────────────────────��─────────────┤
│ promotions ─── M:N ──→ products (via promotion_products)            │
│ collections ─── 1:N ──→ collection_products                         │
│ collection_products ─── N:1 ──→ products                            │
└───────────────────────────────────────────────���─────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                            SHIPPING                                  │
├───────────────────────────────────���─────────────────────────���───────┤
│ shipping_costs ─── 1:N ──→ shipping_cost_configs (weight tiers)     │
└────────────��─────────────────────────────────────��──────────────────┘

┌────���────────────────────────────────────��───────────────────────────┐
│                          CONTENT & FILES                             │
├────────────────────────────���────────────────────────────────────────┤
│ contents (key-value CMS store for banners etc.)                     │
│ file_uploads (uploaded file metadata)                                │
└───────────────────────���─────────────────────────────────────────────┘
```

### 3.2 Table Definitions

#### `users`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK, auto-increment | |
| guest_id | string | nullable | UUID for guest accounts |
| email | string | not null, unique (CI) | Immutable after creation |
| username | string | not null, unique (CI) | Immutable after creation |
| password | string | not null | bcrypt hash (cost 14) |
| first_name | string | nullable | |
| last_name | string | nullable | |
| tel | string | nullable | |
| is_admin | bool | default false | Immutable after creation |
| is_foreign | bool | default false | |
| role | enum | not null | SUPER_ADMIN, ADMIN, GUEST, REGISTERED |
| created_at, updated_at, deleted_at | timestamps | soft delete | |
| created_by, updated_by, deleted_by | uint | audit | |

#### `addresses`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| user_id | uint | FK → users |
| country_name | string | not null |
| country_code | string | not null |
| address | string | not null |
| province | string | not null |
| city | string | not null |
| post_code | string | not null |
| tambon | string | nullable |

#### `corporate_addresses`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| user_id | uint | FK → users |
| company_name | string | not null |
| tax_id | string | not null |
| company_tel | string | not null |
| company_email | string | not null |
| address, province, city, tambon, post_code | string | billing address |
| shipping_address, shipping_province, shipping_city, shipping_tambon, shipping_post_code | string | shipping address |

#### `products`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| model_code | string | not null | Brand's internal model code |
| name | string | not null | |
| category_id | uint | FK → categories, not null | |
| brand_id | uint | FK → brands, not null | |
| detail | string | not null | HTML description |
| spec | string | not null | HTML specifications |
| price | float32 | not null | Base price |
| sale_price | float32 | default 0 | Discounted price (0 = no sale) |
| weight | float32 | not null | Base weight (kg) |
| is_new | bool | default false | "NEW" badge |
| is_display | bool | default true | Visibility on storefront |
| new_end_at | timestamp | nullable | Auto-remove NEW badge after this |

#### `product_skus`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| sku | string | not null | Generated SKU code |
| product_id | uint | FK → products | |
| size_id | uint | FK → sizes | |
| color_id | uint | FK → colors | |
| weight | float32 | not null | SKU-specific weight |
| stock | float32 | not null | Current inventory quantity |
| price | float32 | not null | SKU-specific price |
| sale_price | float32 | default 0 | SKU-specific sale price |

#### `orders`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| order_number | string | not null, immutable | Format: `{seq}-{userId}-{uuid8}` |
| order_status | enum | not null | PAYMENT_PENDING, PAYMENT_SUCCESS, SHIPPING_PENDING, SHIPPING_COMPLETED, CANCEL, VOID |
| total_price | float64 | not null, default 0 | Before discounts |
| net_total_price | float64 | not null, default 0 | After discounts |
| total_discount | float64 | not null, default 0 | |
| user_id | uint | FK → users, not null | |
| address_id | uint | FK → addresses, nullable | |
| corporate_address_id | uint | FK → corporate_addresses, nullable | |
| omise_token | string | | Payment token used |
| payment_id | string | nullable | Omise charge ID |
| payment_method | string | nullable | credit_card, promptpay, etc. |
| shipping_cost | float32 | nullable | |
| tracking_url | string | nullable | Delivery tracking link |
| order_date | timestamp | nullable | |
| receipt_number | string | | For invoice reference |

#### `order_details`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| order_id | uint | FK → orders, not null |
| order_item_code | string | not null, immutable |
| product_id | uint | FK → products, not null |
| sku_id | uint | FK → product_skus, not null |
| qty | int | not null, default 1 |
| price | float32 | not null |

#### `order_stocks`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| product_id | uint | not null | |
| sku_id | uint | not null | |
| order_id | uint | not null | |
| order_detail_id | uint | not null | |
| qty | int | not null | Quantity deducted from stock |

#### `promotions`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| name | string | not null | Display name |
| code | string | not null, unique (CI) | Coupon code |
| type | enum | not null | DISCOUNT, COUPON, CODE, FREE_SHIPPING |
| discount_mode | enum | default ORDER_ITEM | ORDER_AMOUNT or ORDER_ITEM |
| discount_type | enum | not null | PERCENT or AMOUNT |
| discount | float32 | not null | Value (% or fixed) |
| limit | float32 | default 0 | Max discount cap |
| purchase_min | float32 | default 0 | Minimum purchase amount |
| detail | string | nullable | |
| is_published | bool | default false | |
| published_start_at | timestamp | not null | |
| published_end_at | timestamp | not null | |
| products | M:N | via `promotion_products` | Applicable products |

#### `shipping_costs`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| country_code | string | not null, unique (CI) |
| country_name | string | |
| free_shipping_on_cost | float32 | |

#### `shipping_cost_configs`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| shipping_cost_id | uint | FK → shipping_costs | |
| min | float32 | not null | Min weight (kg) |
| max | float32 | not null | Max weight (kg) |
| cost | float32 | not null | Shipping fee for this tier |

#### `collections`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| title | string | not null |
| sub_title | string | not null |
| order | uint | not null (display order) |
| is_published | bool | default true |

#### `contents`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| key | string | not null | e.g. "banner_1" |
| type | string | not null | e.g. "image", "text" |
| value | string | not null | Content value or URL |

#### `log_authens`
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| id | uint | PK | |
| token | string | not null | JWT string |
| user_id | uint | not null | |
| issued_at | timestamp | not null | |
| expired_at | timestamp | not null | 24h TTL |
| is_active | bool | default true | Set false on logout |

#### `file_uploads`
| Column | Type | Constraints |
|--------|------|-------------|
| id | uint | PK |
| name | string | not null |
| original_name | string | not null |
| url | string | not null |

### 3.3 Key Indexes (Unique, Case-Insensitive, Soft-Delete Aware)

All created in `cmd/database/database.go`:
- `idx_unique_brand_name` — `LOWER(name)` WHERE `deleted_at IS NULL`
- `idx_unique_product_name` — `LOWER(name)` WHERE `deleted_at IS NULL`
- `idx_unique_category_name` — `LOWER(name)` WHERE `deleted_at IS NULL`
- `idx_unique_promotion_code` — `LOWER(code)` WHERE `deleted_at IS NULL`
- `idx_unique_user_email` — `LOWER(email)` WHERE `deleted_at IS NULL`
- `idx_unique_size_code` — `LOWER(size_code)` WHERE `deleted_at IS NULL`
- `idx_unique_color_code` — `LOWER(color_code)` WHERE `deleted_at IS NULL`
- `idx_unique_shipping_country_code` �� `LOWER(country_code)` WHERE `deleted_at IS NULL`

### 3.4 Soft Delete Pattern

All entities embed `CommonFields`:
```go
type CommonFields struct {
    ID        uint           // PK
    CreatedAt time.Time      // Auto-set by GORM
    CreatedBy uint           // Set manually in controller
    UpdatedAt time.Time      // Auto-set by GORM
    UpdatedBy uint           // Set manually in controller
    DeletedAt gorm.DeletedAt // Soft delete (GORM auto-filters)
    DeletedBy uint           // Set manually before delete
}
```

---

## 4. Summary of Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| Omise payment gateway | Thai market; supports credit cards + PromptPay |
| GORM auto-migration (dev only) | Rapid development; no explicit migration files |
| JWT stored in DB table | Enables server-side session invalidation (logout) |
| Stock deducted on payment success only | Prevents stock holds on abandoned carts |
| Stock restored only on VOID (refund) | CANCEL doesn't restore; admin-controlled |
| All prices in THB (Thai Baht) | Hardcoded currency in Omise service |
| Order number includes userId + UUID | Unique + traceable |
| PDF reports generated server-side | Using Maroto library with Thai fonts |
| No test coverage | Technical debt; no test files exist |
| No caching layer | All queries hit PostgreSQL directly |
| Services injected directly in controllers | Not through usecase layer |
