# Technology Stack

**Analysis Date:** 2026-04-11

## Languages

**Primary:**
- Go 1.21.1 - All application code (`main.go`, `cmd/`, `pkg/`, `services/`, `entity/`, `utils/`)

**Secondary:**
- HTML - Email receipt templates (`html-template/receipt.html`)
- SQL - Inline unique index creation in migrations (`cmd/database/database.go`)

## Runtime

**Environment:**
- Go 1.21.1 (specified in `go.mod`)
- Docker base image: `golang:1.21.7-alpine3.19` (`Dockerfile`)

**Package Manager:**
- Go Modules
- Lockfile: `go.sum` present

## Frameworks

**Core:**
- Gin v1.9.1 (`github.com/gin-gonic/gin`) - HTTP web framework, routing, middleware
- GORM v1.25.5 (`gorm.io/gorm`) - ORM for PostgreSQL database operations

**Testing:**
- Not detected - no test framework configuration or test files found

**Build/Dev:**
- Docker + Docker Compose - containerization and local development
- Makefile - build automation (`Makefile`)
- Nginx - reverse proxy with TLS termination (`nginx.conf`, `docker-compose.yml`)

## Key Dependencies

**Critical:**
- `gorm.io/driver/postgres` v1.5.4 - PostgreSQL driver for GORM
- `github.com/jackc/pgx/v5` v5.5.1 - Underlying PostgreSQL driver
- `github.com/gin-gonic/gin` v1.9.1 - HTTP framework (all API routes)
- `github.com/golang-jwt/jwt/v4` v4.5.0 - JWT token creation and validation for auth
- `github.com/omise/omise-go` v1.4.1 - Omise payment gateway SDK
- `cloud.google.com/go/storage` v1.37.0 - Google Cloud Storage for file uploads
- `golang.org/x/crypto` v0.18.0 - bcrypt password hashing

**Infrastructure:**
- `github.com/joho/godotenv` v1.5.1 - `.env` file loading for local development
- `github.com/sirupsen/logrus` v1.9.3 - Structured JSON logging
- `github.com/google/uuid` v1.6.0 - UUID generation (receipt numbers, JWT IDs)
- `github.com/johnfercher/maroto` v1.0.0 - PDF report generation
- `github.com/dustin/go-humanize` v1.0.1 - Human-friendly number formatting
- `github.com/sethvargo/go-password` v0.2.0 - Password generation

## Configuration

**Environment:**
- Configuration loaded via environment variables through `config/config.go`
- `.env` file loaded by `godotenv` when `APP_ENV` is not set
- `.env` file present (existence noted, contents not read)

**Required environment variables:**
- `APP_ENV` - Environment mode (`development` / `production`)
- `APP_PORT` - Server port (defaults to `3000`)
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSL_MODE` - PostgreSQL connection
- `JWT_SECRET` - HMAC signing key for JWT tokens
- `GOOGLE_APPLICATION_CREDENTIALS` - GCS service account credentials path
- `GOOGLE_BUCKET_NAME`, `GOOGLE_PROJECT_ID`, `GOOGLE_UPLOAD_FOLDER`, `GOOGLE_PUBLIC_PATH` - GCS config
- `OMISE_PUBLIC_KEY`, `OMISE_SECRET_KEY` - Omise payment gateway keys
- `EMAIL_SERVER`, `EMAIL_HOST`, `EMAIL_FROM`, `EMAIL_PASSWORD` - SMTP email config
- `RETURN_URL` - Payment return URL after 3DS/redirect flows
- `ADMIN_USERNAME`, `ADMIN_PASSWORD` - Initial admin user seeding
- `LOCAL_UPLOAD_FOLDER` - Local file storage path
- `LOG_LEVEL` - Logrus log level

**Build:**
- `Dockerfile` - Alpine-based multi-stage-ish build with CGO disabled
- `Makefile` - `go-run`, `go-build`, `go-clean`, `docker-bun`, `database-up`, `database-down`
- `docker-compose.yml` - PostgreSQL + API + Nginx (TLS) orchestration

## Platform Requirements

**Development:**
- Go 1.21+
- PostgreSQL (via Docker or external, referenced in `../soda_shop_db/` in Makefile)
- `.env` file with all required environment variables
- Google Cloud service account credentials file (`config/service_credentials.json` present)

**Production:**
- Docker runtime
- Nginx reverse proxy with SSL certificates (`certs/cert.pem`, `certs/key.pem`)
- PostgreSQL database
- Google Cloud Storage bucket
- Omise payment account
- SMTP server for transactional emails

---

*Stack analysis: 2026-04-11*
