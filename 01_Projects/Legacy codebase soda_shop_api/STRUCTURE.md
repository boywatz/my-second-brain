# Codebase Structure

**Analysis Date:** 2026-04-11

## Directory Layout

```
soda_shop_api/
├── main.go                  # Application entry point
├── go.mod                   # Go module definition (go 1.21.1)
├── go.sum                   # Dependency checksums
├── Dockerfile               # Container build
├── docker-compose.yml       # Local dev orchestration
├── Makefile                 # Build/run commands
├── nginx.conf               # Reverse proxy config
├── assets/                  # Static assets
│   └── fonts/               # Font files (for PDF generation)
├── certs/                   # TLS certificates
├── cmd/                     # Application commands
│   ├── api/                 # HTTP server
│   │   ├── server.go        # Route setup & DI wiring
│   │   ├── controllers/     # HTTP handlers (one per domain)
│   │   ├── middlewares/      # Auth, CORS, logging middleware
│   │   └── response/        # API response envelope
│   └── database/            # Database connection & migration
│       └── database.go
├── config/                  # Application configuration
│   └── config.go            # Env var loading, accessor functions
├── entity/                  # Domain models & DTOs
│   ├── common_field.go      # Shared audit fields (CommonFields)
│   ├── account.go           # Account-related types
│   ├── auth.go              # JWT claims, credentials, auth response
│   ├── collection.go        # Collection entities
│   ├── content.go           # Content, menu, storefront DTOs
│   ├── dashboard.go         # Dashboard types
│   ├── file.go              # File upload entity
│   ├── order.go             # Order, OrderDetail, OrderPromotion, OrderStock
│   ├── product.go           # Product, Brand, Category, Size, Color, SKU, Gallery
│   ├── promotion.go         # Promotion entity
│   ├── register.go          # Registration request DTO
│   ├── shipping.go          # Shipping cost entities
│   └── user.go              # User, Address, CorporateAddress, roles
├── html-template/           # HTML templates (email receipts)
├── internal/                # Internal packages (placeholder, mostly empty)
│   ├── event/               # Event system (README only, not implemented)
│   └── http/                # HTTP utilities (README only, not implemented)
├── pkg/                     # Domain packages (repo + usecase per domain)
│   ├── auth/                # Authentication
│   │   ├── auth_repo.go
│   │   └── auth_usecase.go
│   ├── brand/               # Brand management
│   │   ├── brand_repo.go
│   │   └── brand_usecase.go
│   ├── category/            # Category management
│   │   ├── category_repo.go
│   │   └── category_usecase.go
│   ├── collection/          # Collection management
│   │   ├── collection_repo.go
│   │   └── collection_usecase.go
│   ├── color/               # Color management
│   │   ├── color_repo.go
│   │   └── color_usecase.go
│   ├── content/             # Storefront content
│   │   ├── content_repo.go
│   │   └── content_usecase.go
│   ├── file/                # File upload management
│   │   ├── file_repo.go
│   │   └── file_usecase.go
│   ├── order/               # Order management
│   │   ├── order_repo.go
│   │   └── order_usecase.go
│   ├── product/             # Product management
│   │   ├── product_repo.go
│   │   └── product_usecase.go
│   ├── promotion/           # Promotion management
│   │   ├── promotion_repo.go
│   │   └── promotion_usecase.go
│   ├── shipping/            # Shipping cost management
│   │   ├── shipping_repo.go
│   │   └── shipping_usecase.go
│   ├── size/                # Size management
│   │   ├── size_repo.go
│   │   └── size_usecase.go
│   └── user/                # User management
│       ├── user_repo.go
│       ├── user_usecase.go
│       └── interfaces/      # Additional interfaces
│           └── change_password.go
├── services/                # External service integrations
│   ├── email.go             # SMTP email sending
│   ├── email_template.go    # HTML email template generation
│   ├── google_storage.go    # Google Cloud Storage upload/download/delete
│   ├── omise.go             # Omise payment gateway (charge, refund)
│   ├── order_number_generator.go  # Sequential order number generation
│   ├── password.go          # bcrypt password hashing
│   └── pdf_generator.go     # PDF invoice generation (maroto)
└── utils/                   # Shared utilities
    └── db_error.go          # PostgreSQL error translation
```

## Directory Purposes

**`cmd/api/`:**
- Purpose: HTTP server layer
- Contains: Server struct, route registration, controllers, middleware, response helpers
- Key files: `server.go` is the central wiring point where all dependencies are constructed and routes registered

**`cmd/api/controllers/`:**
- Purpose: HTTP request handlers, one file per domain
- Contains: 15 controller files matching domain areas
- Key files: `content_controller.go` (largest, handles all public storefront endpoints including order creation and payment), `order_controller.go`, `product_controller.go`

**`cmd/database/`:**
- Purpose: Database connection, migration, and seeding
- Contains: Single `database.go` with Connect, migrate, createFirstAdmin functions

**`config/`:**
- Purpose: Centralized environment variable access
- Contains: `Config` struct, `DBConfig` struct, accessor functions for all env vars (JWT, Google, Omise, Email)

**`entity/`:**
- Purpose: All domain models, DTOs, and type constants shared across layers
- Contains: GORM-annotated model structs, request/response DTOs, enum constants
- Key files: `product.go` (Product, Brand, Category, Size, Color, SKU, Gallery), `order.go` (Order, OrderDetail, OrderPromotion, OrderStock), `user.go` (User, Address, roles), `common_field.go` (audit fields)

**`pkg/`:**
- Purpose: Domain business logic organized by bounded context
- Contains: One subdirectory per domain, each with `*_repo.go` (interface + GORM implementation) and `*_usecase.go` (interface + implementation)

**`services/`:**
- Purpose: Third-party service wrappers
- Contains: Omise payments, Google Cloud Storage, SMTP email, PDF generation, password hashing, order number generation

**`utils/`:**
- Purpose: Shared utility functions
- Contains: Database error translation (PostgreSQL error codes to user-friendly messages)

**`internal/`:**
- Purpose: Planned internal packages (currently placeholder)
- Contains: Only README files in `event/` and `http/` subdirectories

## Key File Locations

**Entry Points:**
- `main.go`: Application bootstrap (config, logging, DB, server, run)
- `cmd/api/server.go`: All route registration and dependency injection wiring

**Configuration:**
- `config/config.go`: All env var loading and accessor functions
- `.env`: Environment variables (existence noted, contents not read)
- `docker-compose.yml`: Local development container orchestration

**Core Logic:**
- `cmd/api/controllers/content_controller.go`: Public API (storefront, orders, payments, webhooks, registration)
- `cmd/api/controllers/order_controller.go`: Admin order management
- `cmd/api/controllers/product_controller.go`: Admin product CRUD
- `pkg/order/order_repo.go`: Order data access with complex queries
- `pkg/product/product_repo.go`: Product data access with eager loading

**Authentication:**
- `cmd/api/middlewares/auth_middleware.go`: JWT validation middleware
- `pkg/auth/auth_usecase.go`: Login/logout logic, JWT token generation

**Testing:**
- No test files detected in the codebase

## Naming Conventions

**Files:**
- `snake_case.go` for all Go source files
- Domain files follow `{domain}_repo.go` and `{domain}_usecase.go` pattern
- Controller files follow `{domain}_controller.go` pattern
- Middleware files follow `{purpose}_middleware.go` pattern

**Directories:**
- `lowercase` single-word directories for domains under `pkg/`
- Plural nouns for container directories (`controllers/`, `middlewares/`, `services/`)

**Go Identifiers:**
- Interfaces: `PascalCase` with descriptive suffix (`ProductRepository`, `ProductUsecase`, `OmiseService`, `StorageService`)
- Private structs: `camelCase` (`repository`, `usecase`, `productController`, `omiseClient`)
- Constructors: `New{InterfaceName}` pattern (`NewProductRepository`, `NewProductUsecase`, `NewOmiseService`)
- Entity fields: `PascalCase` Go fields with `camelCase` JSON tags

## Where to Add New Code

**New Domain Feature (e.g., "wishlist"):**
1. Create entity structs in `entity/wishlist.go`
2. Create `pkg/wishlist/wishlist_repo.go` with `WishlistRepository` interface + GORM implementation
3. Create `pkg/wishlist/wishlist_usecase.go` with `WishlistUsecase` interface + implementation
4. Create `cmd/api/controllers/wishlist_controller.go` with route registration
5. Wire in `cmd/api/server.go`: instantiate repo, usecase, register controller on appropriate route group
6. Add GORM model to `cmd/database/database.go` `migrate()` function

**New External Service Integration:**
- Add to `services/` following interface + constructor pattern (see `services/omise.go` for reference)
- Add env var accessor in `config/config.go`

**New API Endpoint on Existing Domain:**
- Add route in the corresponding controller's constructor function (e.g., `NewProductController`)
- Add handler method on the controller struct
- If new data access needed: add method to repository interface and implementation, then usecase interface and implementation

**New Middleware:**
- Add to `cmd/api/middlewares/` following the `gin.HandlerFunc` pattern
- Apply in `cmd/api/server.go` via `c.Use()` or on specific route groups

**New Utility:**
- Add to `utils/` package

**Public (Unauthenticated) Endpoints:**
- Add to `cmd/api/controllers/content_controller.go` or create a new controller registered on the `pubr` (public) route group in `cmd/api/server.go`

**Authenticated (Admin) Endpoints:**
- Register on the `pvr` (private) route group in `cmd/api/server.go` which applies `middlewares.JwtAuthentication()`

## Special Directories

**`internal/`:**
- Purpose: Placeholder for future internal packages
- Generated: No
- Committed: Yes
- Status: Contains only README files; `event/` and `http/` subdirectories are empty stubs

**`assets/fonts/`:**
- Purpose: Font files used by PDF generator (`services/pdf_generator.go`)
- Generated: No
- Committed: Yes

**`html-template/`:**
- Purpose: HTML templates for email content generation
- Generated: No
- Committed: Yes

**`certs/`:**
- Purpose: TLS certificate files
- Generated: No
- Committed: Yes (should verify no private keys are committed)

## API Route Structure

**Public Routes (`/api/v1/contents/`):**
- `GET /banner` - Get banners
- `GET /menu` - Get dynamic navigation menu
- `GET /product` - Get storefront products
- `GET /shipping` - Get shipping costs
- `GET /promotion` - Get active promotions
- `GET /order` - Get orders by userId query param
- `GET /charge/:chargeID` - Get Omise charge details
- `GET /checkPaymentStatus/:chargeID` - Check payment status
- `GET /collections` - Get published collections
- `POST /account/guest` - Create guest account
- `POST /account/register` - Register new user
- `POST /order` - Create order with payment
- `PUT /banner` - Update banners (requires JWT)
- `POST /test_email` - Test email sending
- `POST /payment_webhook` - Omise payment webhook

**Auth Routes (`/api/v1/auth/`):**
- Login and logout (public)

**Authenticated Admin Routes (`/api/v1/`):**
- `/v1/users` - User CRUD
- `/v1/brands` - Brand CRUD
- `/v1/categories` - Category CRUD
- `/v1/sizes` - Size CRUD
- `/v1/colors` - Color CRUD
- `/v1/products` - Product CRUD + display toggle
- `/v1/collections` - Collection CRUD
- `/v1/promotions` - Promotion CRUD
- `/v1/orders` - Order management (list, detail, status update, refund, invoice, delete)
- `/v1/shipping` - Shipping cost management
- `/v1/files` - File upload management
- `/v1/dashboard` - Dashboard data
- `/v1/reports` - Report generation

---

*Structure analysis: 2026-04-11*
