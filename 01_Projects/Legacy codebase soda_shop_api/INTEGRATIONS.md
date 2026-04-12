# External Integrations

**Analysis Date:** 2026-04-11

## APIs & External Services

**Payment Processing:**
- Omise - Thai payment gateway for credit card charges, source-based payments, and refunds
  - SDK: `github.com/omise/omise-go` v1.4.1
  - Client: `services/omise.go` - `OmiseService` interface
  - Auth: `OMISE_PUBLIC_KEY`, `OMISE_SECRET_KEY` env vars (read via `config/config.go`)
  - Currency: THB (hardcoded in `services/omise.go`)
  - Operations:
    - `CreateCustomer()` - Create Omise customer with card token
    - `Charge()` - Card-based charge with 3DS return URI
    - `ChargeWithSource()` - Source-based charge (e.g., PromptPay, internet banking)
    - `Refund()` - Refund a charge by payment ID
    - `GetCharge()` - Retrieve charge status
  - Return URI: Configured via `RETURN_URL` env var for 3DS redirects

**Cloud Storage:**
- Google Cloud Storage - File upload, download, and deletion
  - SDK: `cloud.google.com/go/storage` v1.37.0
  - Client: `services/google_storage.go` - `StorageService` interface
  - Auth: `GOOGLE_APPLICATION_CREDENTIALS` env var (service account JSON path)
  - Credentials file: `config/service_credentials.json` (existence noted)
  - Config env vars:
    - `GOOGLE_PROJECT_ID` - GCP project
    - `GOOGLE_BUCKET_NAME` - Storage bucket name
    - `GOOGLE_UPLOAD_FOLDER` - Object prefix/folder path
    - `GOOGLE_PUBLIC_PATH` - Public URL base path
  - Operations:
    - `Upload()` - Upload multipart file to bucket with 60s timeout
    - `Download()` - Download object to local filesystem
    - `Delete()` - Delete object from bucket

## Data Storage

**Database:**
- PostgreSQL (latest, via Docker)
  - Connection: DSN built from `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSL_MODE`
  - Client: GORM v1.25.5 with `gorm.io/driver/postgres` v1.5.4 (pgx v5 driver)
  - Connection setup: `cmd/database/database.go`
  - Auto-migration: Runs in development mode only (`config.IsDevelopment()`)
  - Logger: GORM logger at `Info` level (always on)
  - Entities migrated:
    - `User`, `Address`, `LogAuthen`
    - `Product`, `Brand`, `Category`, `SizeChart`, `ProductSizeChart`
    - `Size`, `Color`, `ProductSKU`, `ProductGallery`
    - `Collection`, `CollectionProduct`
    - `Promotion`
    - `Order`, `OrderDetail`, `OrderPromotion`, `OrderStock`
    - `ShippingCost`, `ShippingCostConfig`
    - `FileUpload`, `SizeChartImage`
    - `Content`, `CorporateAddress`
  - Custom indexes: Unique indexes with case-insensitive collation and soft-delete awareness on brands, products, categories, promotions, users, sizes, colors, shipping costs

**File Storage:**
- Google Cloud Storage (primary, for production uploads)
- Local filesystem (secondary, via `LOCAL_UPLOAD_FOLDER` env var, served as static files at `/public`)

**Caching:**
- None detected

## Authentication & Identity

**Auth Provider:**
- Custom JWT-based authentication
  - Implementation: `pkg/auth/auth_usecase.go`, `cmd/api/middlewares/auth_middleware.go`
  - Password hashing: bcrypt with cost 14 (`services/password.go`)
  - Token signing: HMAC-SHA256 via `JWT_SECRET` env var
  - Token expiry: 24 hours
  - Token storage: `LogAuthen` table tracks active tokens with `issued_at`, `expired_at`, `is_active`
  - Claims: `userId`, `username`, `role` in JWT payload
  - Middleware: `middlewares.JwtAuthentication()` extracts Bearer token, validates, sets `UserID`, `Username`, `Token` in Gin context
  - Admin seeding: First admin user created on startup from `ADMIN_USERNAME`, `ADMIN_PASSWORD` env vars (`cmd/database/database.go`)
  - Role system: Entity defines roles (e.g., `SUPER_ADMIN`)

## Email (SMTP)

**Transactional Email:**
- SMTP direct sending via Go standard library `net/smtp`
  - Implementation: `services/email.go`
  - Auth: `smtp.PlainAuth` with `EMAIL_FROM` and `EMAIL_PASSWORD`
  - Server: `EMAIL_SERVER` (host:port), `EMAIL_HOST` (SMTP host for auth)
  - Content-Type: HTML (`text/html; charset=UTF-8`)
  - Templates: HTML template for order receipts (`html-template/receipt.html`)
  - Template engine: Manual string replacement (`strings.Replace`) in `services/email_template.go`
  - Use case: Order completion receipt emails

## PDF Generation

**Report Generation:**
- Maroto PDF library (`github.com/johnfercher/maroto` v1.0.0)
  - Implementation: `services/pdf_generator.go`
  - Font: THSarabunNew (Thai font, stored in `assets/fonts/`)
  - Reports generated:
    - Daily Sales Report - orders with location breakdown (Bangkok, other provinces, international)
    - Daily Sales Report Group By Model - aggregated by product model
    - Stock Report By Model - inventory valuation report
  - Output: `bytes.Buffer` (PDF binary)

## Monitoring & Observability

**Error Tracking:**
- None detected (no Sentry, Datadog, or similar)

**Logs:**
- Logrus (`github.com/sirupsen/logrus`) with JSON formatter
  - Level: Configurable via `LOG_LEVEL` env var, defaults to `Info`
  - Setup: `main.go`
  - Custom middleware: `cmd/api/middlewares/log_middleware.go`

## CI/CD & Deployment

**Hosting:**
- Docker-based deployment with Nginx TLS reverse proxy
  - `docker-compose.yml` defines: PostgreSQL, API, Nginx services
  - Nginx listens on port 443 with SSL (`certs/cert.pem`, `certs/key.pem`)
  - Nginx proxies to `http://api:3000/`

**CI Pipeline:**
- Not detected (no GitHub Actions, GitLab CI, or similar config files)

## Environment Configuration

**Required env vars (grouped):**

Database:
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSL_MODE`

Application:
- `APP_ENV` (`development` | `production`)
- `APP_PORT` (default: `3000`)
- `LOG_LEVEL`

Authentication:
- `JWT_SECRET`
- `ADMIN_USERNAME`, `ADMIN_PASSWORD`

Google Cloud:
- `GOOGLE_APPLICATION_CREDENTIALS`
- `GOOGLE_PROJECT_ID`, `GOOGLE_BUCKET_NAME`, `GOOGLE_UPLOAD_FOLDER`, `GOOGLE_PUBLIC_PATH`

Payment:
- `OMISE_PUBLIC_KEY`, `OMISE_SECRET_KEY`
- `RETURN_URL`

Email:
- `EMAIL_SERVER`, `EMAIL_HOST`, `EMAIL_FROM`, `EMAIL_PASSWORD`

Storage:
- `LOCAL_UPLOAD_FOLDER`

**Secrets location:**
- `.env` file (local development, loaded by `godotenv`)
- `config/service_credentials.json` (Google Cloud service account)
- Docker `env_file` directive in `docker-compose.yml`

## Webhooks & Callbacks

**Incoming:**
- Not explicitly detected. Omise typically sends webhooks for payment status updates, but no webhook handler endpoint is visible in the route setup (`cmd/api/server.go`).

**Outgoing:**
- None detected

## Route Structure

**Public routes** (`/api` group, no auth):
- Auth controller - login/logout (`cmd/api/controllers/auth_controller.go`)
- Content controller - public storefront data (`cmd/api/controllers/content_controller.go`)

**Protected routes** (`/api` group, JWT auth required):
- User, Brand, Category, Size, Color, Product, Collection, Promotion, Order, Shipping, File, Dashboard, Report controllers
- All under `middlewares.JwtAuthentication()`

**Static files:**
- `/public` - serves local upload directory

---

*Integration audit: 2026-04-11*
