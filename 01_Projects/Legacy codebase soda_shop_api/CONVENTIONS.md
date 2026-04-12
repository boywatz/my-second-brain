# Coding Conventions

**Analysis Date:** 2026-04-11

## Naming Patterns

**Files:**
- Use `snake_case` for all Go source files: `product_controller.go`, `order_usecase.go`, `auth_repo.go`
- Suffix files by layer role: `_controller.go`, `_usecase.go`, `_repo.go`
- Service files use descriptive names without suffix: `email.go`, `omise.go`, `password.go`

**Packages:**
- Single lowercase word: `entity`, `config`, `services`, `utils`
- Domain packages under `pkg/` use singular nouns: `product`, `order`, `auth`, `user`, `brand`
- Controller/middleware packages use plural: `controllers`, `middlewares`

**Structs:**
- PascalCase: `Product`, `OrderDetail`, `ProductSKU`
- Private implementation structs use lowercase: `repository`, `usecase`, `server`, `authRepo`, `omiseClient`

**Interfaces:**
- PascalCase with descriptive suffix: `ProductRepository`, `ProductUsecase`, `AuthUsecase`, `OmiseService`
- Defined in the same file as the implementation, at the bottom of the file

**Functions:**
- Constructor functions: `NewXxxYyy()` pattern -- `NewProductRepository()`, `NewAuthUsecase()`, `NewOmiseService()`
- Controller constructors: `NewXxxController()` -- registers routes inline

**Variables:**
- camelCase for local variables: `productId`, `orderId`, `accessToken`
- Struct fields use PascalCase with JSON tags in camelCase: `ModelCode` -> `json:"modelCode"`

**Constants:**
- UPPER_SNAKE_CASE for enum-like constants: `PAYMENT_PENDING`, `SUPER_ADMIN`, `CANCEL`
- Custom string types for enums: `type OrderStatus string`, `type UserRole string`

## Code Style

**Formatting:**
- Standard `gofmt` formatting (no explicit config files present)
- No `.editorconfig` or explicit formatting tooling configured

**Linting:**
- No linting tools configured (no `golangci-lint` config, no `.golangci.yml`)

**Import Organization:**
- Standard library first
- Third-party packages second
- Internal project packages third
- Example from `cmd/api/server.go`:
```go
import (
    "net/http"

    "github.com/boywatz/soda_shop_api/cmd/api/controllers"
    "github.com/boywatz/soda_shop_api/cmd/api/middlewares"
    "github.com/boywatz/soda_shop_api/config"
    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
)
```
- Package aliases used sparingly: `log "github.com/sirupsen/logrus"` in `main.go`, `server "github.com/boywatz/soda_shop_api/cmd/api"` in `main.go`

**Path Aliases:**
- Full module path: `github.com/boywatz/soda_shop_api/...`
- No path aliases or shorteners

## Architecture Pattern: Repository + Usecase

Each domain feature in `pkg/` follows a strict two-file pattern:

**Repository (data access):**
- File: `pkg/{domain}/{domain}_repo.go`
- Private struct: `type repository struct { DB *gorm.DB }`
- Public interface: `type {Domain}Repository interface { ... }`
- Constructor: `func New{Domain}Repository(db *gorm.DB) {Domain}Repository`
- All database operations wrapped in `r.DB.Transaction(func(tx *gorm.DB) error { ... })`

**Usecase (business logic):**
- File: `pkg/{domain}/{domain}_usecase.go`
- Private struct: `type usecase struct { repo {Domain}Repository }`
- Public interface: `type {Domain}Usecase interface { ... }`
- Constructor: `func New{Domain}Usecase(repo {Domain}Repository) {Domain}Usecase`
- Usecases are thin wrappers that delegate to repositories

**Example pattern from** `pkg/product/`:
```go
// product_repo.go
type repository struct { DB *gorm.DB }
type ProductRepository interface {
    FindAll(products *[]entity.Product) error
    FindById(product *entity.Product, id uint) error
    Create(product *entity.Product) error
    // ...
}
func NewProductRepository(db *gorm.DB) ProductRepository { return &repository{DB: db} }

// product_usecase.go
type usecase struct { repo ProductRepository }
type ProductUsecase interface {
    CreateProduct(product *entity.Product) error
    FindAll(products *[]entity.Product) error
    // ...
}
func NewProductUsecase(repo ProductRepository) ProductUsecase { return &usecase{repo: repo} }
```

## Controller Pattern

Controllers live in `cmd/api/controllers/` and follow this pattern:

1. Private struct holding usecase dependencies
2. Constructor `NewXxxController()` that takes a `*gin.RouterGroup` and registers routes
3. Route handlers as methods on the controller struct
4. Routes grouped under `/api/v1/{resource}`

```go
// Example from cmd/api/controllers/product_controller.go
type productController struct {
    productUsecase product.ProductUsecase
}

func NewProductController(c *gin.RouterGroup, productUsecase product.ProductUsecase) {
    controllers := &productController{productUsecase: productUsecase}
    v1 := c.Group("/v1/products")
    v1.POST("", controllers.CreateProduct)
    v1.GET("", controllers.GetProducts)
    v1.GET("/:id", controllers.GetProduct)
    v1.PUT("/:id", controllers.UpdateProduct)
    v1.DELETE("/:id", controllers.DeleteProduct)
    v1.PATCH("/:id", controllers.UpdateIsDisplay)
}
```

## Error Handling

**Patterns:**
- Repository layer returns `error` from GORM operations, sometimes via `utils.HandleDBError(err)` for translating Postgres errors
- Usecase layer passes through repository errors without wrapping (thin delegation)
- Controller layer catches errors and returns JSON responses using `response.ApiResponse(nil, err.Error())`
- Only `auth_usecase.go` uses `fmt.Errorf("...: %w", err)` for error wrapping

**Standard error response:**
```go
if err != nil {
    c.JSON(http.StatusInternalServerError, response.ApiResponse(nil, err.Error()))
    return
}
```

**Error translation in** `utils/db_error.go`:
```go
func HandleDBError(err error) error {
    pgError, ok := err.(*pgconn.PgError)
    if !ok { return err }
    if pgError.Code == "23505" {
        return errors.New("data already exists in system")  // Thai: "ข้อมูลซ้ำกันในระบบ"
    }
    return err
}
```

**Issues to note:**
- `strconv.ParseUint()` errors are silently ignored with `_` throughout controllers (e.g., `productId, _ := strconv.ParseUint(...)`)
- Most errors return `http.StatusInternalServerError` regardless of cause (no 404 distinction)

## API Response Format

All responses use a consistent envelope via `cmd/api/response/response.go`:

```go
func ApiResponse(data interface{}, err string) gin.H {
    return gin.H{
        "success":   bool,
        "data":      data,       // nil on error
        "error":     &errString, // nil on success
        "timestamp": unixTimestamp,
    }
}
```

- Success: `response.ApiResponse(data, "")`
- Error: `response.ApiResponse(nil, err.Error())`
- Success with message: `response.ApiResponse("success", "")`

## Entity/Model Design

**Base model in** `entity/common_field.go`:
```go
type CommonFields struct {
    ID        uint           `gorm:"primary_key" json:"id"`
    CreatedAt time.Time      `json:"-"`
    CreatedBy uint           `json:"-"`
    UpdatedAt time.Time      `json:"-"`
    UpdatedBy uint           `json:"-"`
    DeletedAt gorm.DeletedAt `json:"-"`  // Soft delete
    DeletedBy uint           `json:"-"`
}
```

- All entities embed `CommonFields` for consistent ID, timestamps, and audit fields
- Audit fields (`CreatedAt`, `UpdatedAt`, `DeletedAt`, `CreatedBy`, `UpdatedBy`, `DeletedBy`) are hidden from JSON with `json:"-"`
- Soft deletes via GORM's `gorm.DeletedAt`

**Struct tag conventions:**
- JSON tags: camelCase (`json:"modelCode"`)
- GORM tags: snake_case constraints (`gorm:"not null"`, `gorm:"default:null"`)
- Foreign keys explicit in GORM tags: `gorm:"foreignKey:CategoryID"`

## Dependency Injection

- Manual constructor injection in `cmd/api/server.go`
- All usecases and repositories instantiated in `SetupRoutes()`
- No DI container or framework
- Dependencies flow: `server.go` -> `NewXxxRepository(db)` -> `NewXxxUsecase(repo)` -> `NewXxxController(router, usecase)`

## Logging

**Framework:** `github.com/sirupsen/logrus` (aliased as `log`)

**Configuration in** `main.go`:
```go
log.SetLevel(logLevel)
log.SetFormatter(&log.JSONFormatter{})
```

**Request logging middleware in** `cmd/api/middlewares/log_middleware.go`:
- Logs METHOD, URI, STATUS, LATENCY, CLIENT_IP, REQUEST body, RESPONSE body
- Masks file upload and report download payloads

**Service layer:** Uses stdlib `log` package (not logrus) in `services/omise.go`, `services/email.go`

## Comments

**When comments are used:**
- Inline comments explaining business logic: `// model code from soda`, `//readable order number code`
- Interface implementation markers: `// UpdateIsDisplay implements ProductRepository.`
- Commented-out code exists in several places (dead code)

**JSDoc/GoDoc:**
- No GoDoc comments on exported types, functions, or interfaces

## Function Design

**Size:** Most functions are under 50 lines. Exception: `UpdateProduct` controller at ~70 lines, `Update` repo method at ~50 lines

**Parameters:**
- Repository methods receive pointers to entity structs: `func (r *repository) Create(product *entity.Product) error`
- Output is populated via pointer parameters rather than return values: `FindAll(products *[]entity.Product) error`

**Return Values:**
- Single `error` return for mutations
- Pointer + error for queries that return a single result (only in auth usecase)
- Void mutation pattern: populate pointer param, return error

## Module Design

**Exports:**
- Each `pkg/{domain}/` package exports its interface types and constructor functions
- Implementation structs are unexported (lowercase)
- Entity structs in `entity/` are all exported

**Barrel Files:**
- Not used. Each package has explicit files for repo and usecase

---

*Convention analysis: 2026-04-11*
