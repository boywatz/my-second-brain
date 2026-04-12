# Architecture

**Analysis Date:** 2026-04-11

## Pattern Overview

**Overall:** Layered Architecture with Repository-Usecase-Controller pattern (Clean Architecture variant)

**Key Characteristics:**
- Three-layer design: Controller (HTTP) -> Usecase (Business Logic) -> Repository (Data Access)
- Domain entities defined in a shared `entity` package, used across all layers
- Dependency injection via constructor functions wired in the server setup
- Interface-based abstractions for usecase and repository layers
- Public API routes (content) and authenticated API routes split at the router level

## Layers

**Controller Layer (HTTP Handlers):**
- Purpose: Parse HTTP requests, delegate to usecases, format responses
- Location: `cmd/api/controllers/`
- Contains: Route registration, request binding, response formatting, user ID extraction from JWT context
- Depends on: Usecase interfaces from `pkg/*/`, entity structs, `cmd/api/response/`, `services/`
- Used by: Gin router groups in `cmd/api/server.go`

**Usecase Layer (Business Logic):**
- Purpose: Orchestrate business operations, delegating persistence to repositories
- Location: `pkg/*/` (each domain has a `*_usecase.go` file)
- Contains: Usecase interface + private struct implementing it
- Depends on: Repository interfaces (same package), `entity/`
- Used by: Controllers
- Note: Most usecases are thin pass-throughs to the repository with minimal business logic

**Repository Layer (Data Access):**
- Purpose: Execute GORM database queries and transactions
- Location: `pkg/*/` (each domain has a `*_repo.go` file)
- Contains: Repository interface + private struct holding `*gorm.DB`
- Depends on: `entity/`, `gorm.io/gorm`, `utils/`
- Used by: Usecases

**Entity Layer (Domain Models):**
- Purpose: Define shared data structures (GORM models, request/response DTOs, enums)
- Location: `entity/`
- Contains: GORM-annotated structs, JSON-tagged DTOs, constants (roles, statuses)
- Depends on: `gorm.io/gorm` (for `DeletedAt`), `github.com/golang-jwt/jwt/v4`
- Used by: All other layers

**Services Layer (External Integrations):**
- Purpose: Encapsulate third-party service interactions (payments, storage, email, PDF)
- Location: `services/`
- Contains: Interface + implementation pairs for Omise, Google Cloud Storage, email, PDF generation, password hashing
- Depends on: `config/`, third-party SDKs
- Used by: Controllers directly (not injected through usecases)

**Middleware Layer:**
- Purpose: Cross-cutting HTTP concerns (auth, CORS, logging)
- Location: `cmd/api/middlewares/`
- Contains: Gin middleware functions
- Depends on: `config/`, `cmd/api/response/`, `github.com/golang-jwt/jwt/v4`
- Used by: `cmd/api/server.go` route setup

## Data Flow

**Authenticated Admin API Request (e.g., Create Product):**

1. HTTP request hits Gin router at `/api/v1/products`
2. `middlewares.JwtAuthentication()` validates Bearer token, sets `UserID` in Gin context
3. `productController.CreateProduct()` binds JSON body to `entity.Product`
4. Controller enriches entity with `CreatedBy` from context, adjusts SKU defaults
5. `productUsecase.CreateProduct()` delegates to `productRepo.Create()`
6. Repository wraps GORM `Create` in a transaction, calls `utils.HandleDBError()` for constraint violations
7. Controller returns `response.ApiResponse("success", "")` as JSON

**Public Content API Request (e.g., Create Order with Payment):**

1. HTTP request hits `/api/v1/contents/order` (no auth middleware)
2. `contentController.CreateOrder()` binds JSON to `entity.Order`
3. Controller generates order number via `services.GenerateOrderNumber()`
4. Controller charges customer via `omiseService.Charge()` or `omiseService.ChargeWithSource()`
5. Controller creates order via `orderUsecase.CreateOrder()`
6. If payment successful: creates order stock records, decrements product stock
7. Deferred email notification via `services.SendEmail()`

**Payment Webhook Flow:**

1. Omise sends POST to `/api/v1/contents/payment_webhook`
2. `contentController.PaymentWebhook()` parses `omise.Event`
3. On `charge.complete` with status `successful`: updates order status, creates stock records, decrements inventory, sends email

**State Management:**
- All state persisted in PostgreSQL via GORM
- No in-memory caching layer
- JWT tokens stored in `log_authens` table for session tracking
- Order numbers generated with in-memory atomic counter (`services.OrderNumberGenerator`)

## Key Abstractions

**Repository Interface Pattern:**
- Purpose: Abstract database operations behind an interface
- Examples: `pkg/product/product_repo.go` (`ProductRepository`), `pkg/order/order_repo.go` (`OrderRepository`)
- Pattern: Interface defined in same file as private struct; constructor returns interface type

**Usecase Interface Pattern:**
- Purpose: Abstract business logic behind an interface
- Examples: `pkg/product/product_usecase.go` (`ProductUsecase`), `pkg/auth/auth_usecase.go` (`AuthUsecase`)
- Pattern: Interface defined in same file as private struct; depends on corresponding repository interface

**Service Interface Pattern:**
- Purpose: Abstract external service calls
- Examples: `services/omise.go` (`OmiseService`), `services/google_storage.go` (`StorageService`)
- Pattern: Interface + constructor; instantiated in controllers, not injected from server

**CommonFields Embedding:**
- Purpose: Shared audit fields across all GORM models
- Location: `entity/common_field.go`
- Pattern: All entities embed `CommonFields` which provides `ID`, `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`, `DeletedAt` (soft delete), `DeletedBy`

## Entry Points

**Application Entry:**
- Location: `main.go`
- Triggers: `go run main.go` or compiled binary
- Responsibilities: Load config, setup logging, connect database, create server, init order number generator, run Gin engine

**Server Setup / Route Registration:**
- Location: `cmd/api/server.go` (`SetupRoutes()`)
- Triggers: Called from `main.go`
- Responsibilities: Create all repositories and usecases, wire dependency injection, register all routes in public (`pubr`) and private (`pvr`) groups

**Database Connection & Migration:**
- Location: `cmd/database/database.go`
- Triggers: Called from `main.go`
- Responsibilities: Connect PostgreSQL via GORM, run auto-migrations (dev only), create unique indexes, seed admin user

## Error Handling

**Strategy:** Return `error` up the call stack; controllers translate to HTTP status codes

**Patterns:**
- Repository layer: Wraps GORM operations in transactions; returns raw errors or uses `utils.HandleDBError()` for PostgreSQL constraint violations (e.g., unique index → Thai-language error message)
- Usecase layer: Passes errors through from repository (minimal transformation)
- Controller layer: Returns `http.StatusBadRequest` for binding errors, `http.StatusInternalServerError` for all other errors, `http.StatusUnauthorized` for auth failures
- All API responses use `response.ApiResponse(data, errMsg)` envelope with `success`, `data`, `error`, `timestamp` fields
- No structured error codes or error type hierarchy

## Cross-Cutting Concerns

**Logging:**
- Logrus configured in `main.go` with JSON formatter
- Log level from `LOG_LEVEL` env var
- Custom log middleware at `cmd/api/middlewares/log_middleware.go`
- Services use standard `log` package directly

**Validation:**
- JSON binding via Gin's `ShouldBindJSON()` using struct tags
- No dedicated validation library beyond Gin's built-in binding
- Database-level constraints (unique indexes, not-null) as secondary validation

**Authentication:**
- JWT-based with HMAC-SHA256 signing
- Middleware at `cmd/api/middlewares/auth_middleware.go` extracts claims to Gin context
- Token stored in `log_authens` table; 24-hour expiry
- Two route groups: public (`/api/v1/contents/*`) and private (`/api/v1/*`) requiring JWT

**CORS:**
- Middleware at `cmd/api/middlewares/cors_middleware.go`

**Soft Deletes:**
- GORM `DeletedAt` field in `CommonFields` enables soft delete across all entities
- `DeletedBy` tracked manually in controller before calling usecase

---

*Architecture analysis: 2026-04-11*
