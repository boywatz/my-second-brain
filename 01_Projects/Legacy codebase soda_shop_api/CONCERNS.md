# Codebase Concerns

**Analysis Date:** 2026-04-11

## Tech Debt

**Zero Test Coverage:**
- Issue: The entire codebase has no test files whatsoever -- no unit tests, no integration tests, no e2e tests
- Files: All files under `pkg/`, `cmd/api/controllers/`, `services/`
- Impact: Any code change risks breaking existing functionality with no safety net; refactoring is extremely risky
- Fix approach: Start with table-driven unit tests for `pkg/*/` usecase and repo layers using interfaces for mocking. Add integration tests for critical flows (order creation, payment webhook, auth). Prioritize `pkg/order/order_repo.go`, `pkg/auth/auth_usecase.go`, and `cmd/api/controllers/content_controller.go`

**God Controller -- content_controller.go (639 lines):**
- Issue: `cmd/api/controllers/content_controller.go` handles public storefront concerns (products, menus, orders, payments, registration, webhooks, collections) all in a single controller with 10+ dependencies injected
- Files: `cmd/api/controllers/content_controller.go`
- Impact: Extremely difficult to modify, test, or reason about. Any change to public API surfaces risks side effects across unrelated features
- Fix approach: Split into separate controllers: `StorefrontController` (products, menus, collections), `CheckoutController` (order creation, payment), `AccountController` (guest creation, registration), `WebhookController` (payment webhooks)

**God Controller -- report_controller.go (629 lines):**
- Issue: `cmd/api/controllers/report_controller.go` contains massive report-generation logic directly in controller handlers, mixing data aggregation, formatting, and PDF generation
- Files: `cmd/api/controllers/report_controller.go`
- Impact: Business logic is untestable because it lives in HTTP handlers. Report logic cannot be reused
- Fix approach: Extract report aggregation into dedicated service/usecase layer. Controller should only parse request, delegate, and return response

**Massive Preload Chains (N+1 risk and memory):**
- Issue: Nearly every repository query eagerly preloads 8-12 nested associations. `FindAll` endpoints load ALL records with ALL associations into memory
- Files: `pkg/order/order_repo.go` (lines 17-27, 35-44, 62-76, 82-94, 111-123, 139-152, 157-170), `pkg/product/product_repo.go` (lines 127-142, 146-162)
- Impact: As data grows, these queries will cause severe memory pressure and slow response times. `FindAll` on orders or products with no pagination will eventually OOM
- Fix approach: Add pagination to all `FindAll` methods. Use selective preloading -- only load what each endpoint actually needs. Consider query-specific DTOs instead of loading full entity graphs

**Duplicated Code Across Controllers:**
- Issue: Account-building logic (mapping User to Account struct) is copy-pasted across `ClientLogin`, `LoadClientData`, and `UpdateClientData` in `cmd/api/controllers/auth_controller.go` (lines 73-98, 114-135)
- Files: `cmd/api/controllers/auth_controller.go`
- Impact: Bugs in account mapping must be fixed in multiple places; easy to miss one
- Fix approach: Extract a `buildAccountFromUser(user, orders)` helper function

**Duplicated Email Sending Logic:**
- Issue: Two nearly identical `sendEmail` and `sendMailToAdmin` functions exist
- Files: `cmd/api/controllers/content_controller.go` (lines 534-547), `cmd/api/controllers/order_controller.go` (lines 190-202)
- Impact: Email logic drift; changes to one are not reflected in the other
- Fix approach: Consolidate into a single email notification service

**Stock Update Not Atomic with Order Creation:**
- Issue: Order creation, stock creation, and stock decrement are separate database operations with no single transaction wrapping them all. If stock update fails mid-way, data is inconsistent
- Files: `cmd/api/controllers/content_controller.go` (lines 439-463), `cmd/api/controllers/content_controller.go` (lines 594-614)
- Impact: Race conditions on concurrent orders can oversell inventory. Partial failures leave stock in inconsistent state
- Fix approach: Wrap order creation + order stock creation + product stock update in a single database transaction

**`float32` and `float64` Mixed for Monetary Values:**
- Issue: `ShippingCost` is `float32`, `TotalPrice`/`NetTotalPrice` are `float64`, `OrderDetail.Price` is `float32`, `OrderPromotion.Discount` is `float32`. Monetary calculations mix precision levels
- Files: `entity/order.go` (lines 22-23, 33, 49, 58)
- Impact: Floating-point precision errors in financial calculations. Currency rounding issues
- Fix approach: Use integer cents (int64) for all monetary values, or use a decimal library. Standardize on a single type

## Known Bugs

**Report VoidOrders Appends to Wrong Slice:**
- Symptoms: Void order data in daily sales report is incorrectly populated
- Files: `cmd/api/controllers/report_controller.go` (lines 289, 305, 321, 334)
- Trigger: Generate a daily sales report with voided orders
- Workaround: None -- the report is silently wrong. `content.VoidOrders = append(content.Orders, ...)` should be `content.VoidOrders = append(content.VoidOrders, ...)`

**Double Write in Report PDF Response:**
- Symptoms: PDF response body is written twice, potentially corrupting output
- Files: `cmd/api/controllers/report_controller.go` (lines 346-350, 505-510, 622-628)
- Trigger: Any report PDF generation. `c.Writer.Write(file.Bytes())` is called, then immediately called again
- Workaround: The second write likely fails silently because headers are already sent

**Google Storage Client Closed Prematurely:**
- Symptoms: All Google Cloud Storage operations (upload/download/delete) will fail with a closed client error
- Files: `services/google_storage.go` (line 103)
- Trigger: Any file upload or delete operation
- Workaround: Remove the `defer client.Close()` from `NewGoogleStorageService()` -- the client must remain open for the lifetime of the service

**Error After Use in ClientLogin:**
- Symptoms: If `userUsecase.FindByID` fails, the error is checked only after the user data has already been consumed (at line 100), leading to nil pointer dereference
- Files: `cmd/api/controllers/auth_controller.go` (lines 77-103)
- Trigger: Database error during user lookup after successful token generation
- Workaround: None

**Ignored Parse Errors on URL Parameters:**
- Symptoms: Invalid `id` parameters silently become 0, causing lookups of record ID 0
- Files: All controllers using `strconv.ParseUint(c.Param("id"), 10, 64)` with `_` error discard -- see 30+ occurrences across all controller files
- Trigger: Pass a non-numeric ID in any endpoint URL
- Workaround: None -- returns confusing "record not found" errors

## Security Considerations

**Certificates and Credentials Committed to Git:**
- Risk: Private TLS keys and Google service account credentials are tracked in version control
- Files: `certs/cert.pem`, `certs/key.pem`, `config/service_credentials.json`
- Current mitigation: None -- these files are committed and visible in git history
- Recommendations: Immediately rotate all exposed credentials. Add `certs/`, `*.pem`, `*.key`, `service_credentials.json` to `.gitignore`. Use environment-based secret injection or a secret manager. Scrub git history with `git filter-repo`

**Unauthenticated Payment Webhook Endpoint:**
- Risk: Anyone can POST to `/api/v1/contents/payment_webhook` and trigger order status changes, stock decrements, and email sends
- Files: `cmd/api/controllers/content_controller.go` (lines 571-618)
- Current mitigation: None -- no signature verification, no IP allowlist, no shared secret
- Recommendations: Verify the Omise webhook signature on every request. Omise sends a signature header that must be validated against the endpoint signing secret

**Unauthenticated Order Creation:**
- Risk: The `POST /api/v1/contents/order` endpoint does not require authentication. Anyone with a valid Omise token can create orders
- Files: `cmd/api/controllers/content_controller.go` (line 69)
- Current mitigation: None
- Recommendations: Require at minimum a valid user session (guest or registered). Validate that `order.UserID` matches the authenticated user

**CORS Allows All Origins:**
- Risk: `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` allows any website to make authenticated requests
- Files: `cmd/api/middlewares/cors_middleware.go` (line 10)
- Current mitigation: None
- Recommendations: Configure an explicit allowlist of permitted origins. `*` with credentials is rejected by browsers but signals a misconfiguration

**No Authorization / Role-Based Access Control:**
- Risk: JWT authentication checks identity only -- any authenticated user can access any admin endpoint (create products, delete orders, generate reports, manage users)
- Files: `cmd/api/middlewares/auth_middleware.go` -- no role check; `cmd/api/server.go` -- all admin routes share same auth middleware
- Current mitigation: None
- Recommendations: Add role-checking middleware. The JWT claims already contain `role` -- use it to restrict admin endpoints to `SUPER_ADMIN` and `ADMIN` roles

**No Rate Limiting:**
- Risk: All endpoints are vulnerable to brute force (login), resource exhaustion (report generation), and abuse (order creation)
- Files: `cmd/api/server.go` -- no rate limiting middleware configured
- Current mitigation: None
- Recommendations: Add rate limiting middleware per-IP and per-user, especially on `/auth/login`, `/contents/order`, and report endpoints

**Hardcoded Test Email in Production Code:**
- Risk: Test email endpoint sends to a hardcoded personal email address
- Files: `cmd/api/controllers/content_controller.go` (line 507) -- `"thanawat.settapong.37@gmail.com"`
- Current mitigation: None -- endpoint is publicly accessible
- Recommendations: Remove the `/test_email` endpoint or restrict it to development environment only

**JWT Token Not Validated Against Stored Tokens:**
- Risk: The auth middleware validates JWT signature and expiry but does not check if the token is still active in the `log_authens` table. Logged-out tokens remain valid until they expire
- Files: `cmd/api/middlewares/auth_middleware.go` -- no database lookup for active token
- Current mitigation: Token expiry is 24 hours
- Recommendations: Check token against `log_authens` table on each request, or implement a token blacklist cache

**No Input Length/Size Validation:**
- Risk: No maximum length validation on any user input fields (names, addresses, emails). No file size limit on uploads
- Files: All controllers using `ShouldBindJSON` without size constraints; `cmd/api/controllers/file_controller.go` (line 37)
- Current mitigation: Database column limits may reject at DB level
- Recommendations: Add request body size limits via Gin middleware. Add validation tags to entity structs. Add file size limits

**Unauthenticated User Order History:**
- Risk: `GET /api/v1/contents/order?userId=X` allows anyone to view any user's order history by guessing user IDs
- Files: `cmd/api/controllers/content_controller.go` (lines 478-490)
- Current mitigation: None
- Recommendations: Require authentication and validate that the requested userId matches the authenticated user

## Performance Bottlenecks

**No Pagination on List Endpoints:**
- Problem: All `FindAll` queries fetch every record from the database with full association preloading
- Files: `pkg/order/order_repo.go` (lines 138-153), `pkg/product/product_repo.go` (lines 145-162), `pkg/user/user_repo.go` (lines 66-68)
- Cause: No `LIMIT`/`OFFSET` in queries. No pagination parameters accepted by controllers
- Improvement path: Add `page` and `limit` query parameters to all list endpoints. Pass through to repository layer with `Limit()` and `Offset()` GORM clauses

**Full Product Catalog Loaded for Storefront:**
- Problem: `GetProducts` loads ALL products with ALL SKUs, galleries, and size charts into memory, then filters in Go code
- Files: `cmd/api/controllers/content_controller.go` (lines 186-285)
- Cause: Filtering (`IsDisplay`, gallery validation) happens in application code after full DB load
- Improvement path: Add `WHERE is_display = true` to the database query. Move validation logic to a background job that sets a `valid` flag

**Report Generation Loads All Orders Then Iterates O(n*m):**
- Problem: `DailySalesReportGroupByModel` groups by model using nested loops over all orders for each unique model, resulting in O(n*m) complexity
- Files: `cmd/api/controllers/report_controller.go` (lines 420-456)
- Cause: Grouping done in Go code instead of SQL
- Improvement path: Use SQL `GROUP BY` with aggregate functions. Or use a map-based grouping in Go for O(n) complexity

**Database Logger Set to Info Level in Production:**
- Problem: GORM logger is hardcoded to `logger.Info` which logs all SQL queries
- Files: `cmd/database/database.go` (line 19)
- Cause: Logger level not environment-dependent
- Improvement path: Use `logger.Warn` or `logger.Error` in production. Only use `logger.Info` in development

## Fragile Areas

**Order Creation and Payment Flow:**
- Files: `cmd/api/controllers/content_controller.go` (lines 377-477), `cmd/api/controllers/content_controller.go` (lines 571-618)
- Why fragile: Payment charging, order creation, stock management, and email sending are interleaved in a single controller method with no transaction safety. Payment webhook duplicates much of the same logic
- Safe modification: Extract into a dedicated `OrderService` with proper transaction boundaries. Test the service layer independently before touching controllers
- Test coverage: Zero

**Stock Management:**
- Files: `pkg/product/product_repo.go` (lines 49-63, 65-79), `cmd/api/controllers/content_controller.go` (lines 440-459), `cmd/api/controllers/order_controller.go` (lines 72-95)
- Why fragile: Stock updates use read-then-write pattern without row-level locking. `float32` for stock quantity allows fractional values on what should be integer counts. Refund stock logic in order_controller differs from the stock update path
- Safe modification: Add `SELECT ... FOR UPDATE` row locking. Change stock to integer type. Consolidate stock operations into a single service
- Test coverage: Zero

**User Update with Address Handling:**
- Files: `pkg/user/user_repo.go` (lines 72-91), `cmd/api/controllers/auth_controller.go` (lines 153-195)
- Why fragile: The update logic conditionally saves addresses and corporate addresses with complex branching. The address update uses `Where("user_id = ?").Save()` which can create or update unpredictably with GORM
- Safe modification: Use explicit `Create` vs `Updates` based on whether address ID exists. Add integration tests for each branch
- Test coverage: Zero

## Scaling Limits

**In-Memory Order Number Generator:**
- Current capacity: Single-instance deployment only
- Limit: Running multiple server instances will generate duplicate order numbers because the counter is in-memory with a mutex
- Scaling path: Use database sequence or Redis-based counter for distributed uniqueness
- Files: `services/order_number_generator.go`, `cmd/api/controllers/order_controller.go` (line 26)

**No Database Connection Pooling Configuration:**
- Current capacity: Default GORM connection pool settings
- Limit: Under high load, connection exhaustion
- Scaling path: Configure `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime` on the `sql.DB` instance
- Files: `cmd/database/database.go`

## Dependencies at Risk

**Google Cloud Storage Client Bug:**
- Risk: The storage client is closed immediately after construction via `defer client.Close()` in `NewGoogleStorageService()`, making all subsequent operations fail
- Impact: File uploads and deletions are broken
- Migration plan: Remove the `defer client.Close()` line; manage client lifecycle at application level
- Files: `services/google_storage.go` (line 103)

## Missing Critical Features

**No Request Validation Framework:**
- Problem: No structured validation beyond Gin's `ShouldBindJSON`. No custom validation rules, no field-level error messages
- Blocks: Cannot enforce business rules at input boundary (e.g., valid email format, password strength, positive quantities)

**No Structured Logging with Request Context:**
- Problem: Mix of `fmt.Println`, `log.Printf`, and `logrus` with no request ID correlation
- Blocks: Cannot trace issues across request lifecycle in production
- Files: `main.go` (logrus setup), `cmd/database/database.go` (fmt.Println), `services/omise.go` (log.Printf)

**No Graceful Shutdown:**
- Problem: Server starts with `gin.Run()` which does not handle OS signals for graceful shutdown
- Blocks: In-flight requests (especially payment processing) may be terminated abruptly on deploy
- Files: `main.go` (lines 40-48)

## Test Coverage Gaps

**Entire Codebase is Untested:**
- What's not tested: Every package, every controller, every service, every repository
- Files: All `.go` files under `pkg/`, `cmd/`, `services/`, `config/`, `utils/`
- Risk: Any code change can introduce regressions undetected. Payment processing, stock management, and financial calculations have zero automated verification
- Priority: High -- start with `pkg/auth/auth_usecase.go`, `pkg/order/order_usecase.go`, `services/omise.go`, then add controller-level integration tests for order creation and payment webhook flows

---

*Concerns audit: 2026-04-11*
