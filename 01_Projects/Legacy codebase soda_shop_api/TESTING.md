# Testing Patterns

**Analysis Date:** 2026-04-11

## Test Framework

**Runner:**
- Go standard `testing` package (available by default)
- No test files exist in the codebase

**Assertion Library:**
- None configured. No `testify`, `gomega`, or other assertion libraries in `go.mod`

**Run Commands:**
```bash
go test ./...              # Run all tests (none exist currently)
go test -race ./...        # Run with race detection
go test -cover ./...       # Run with coverage
go clean -testcache        # Clear test cache (available via Makefile: make go-clean)
```

## Test File Organization

**Location:**
- No test files exist anywhere in the codebase
- Convention would be co-located: `pkg/product/product_repo_test.go` alongside `product_repo.go`

**Naming:**
- Go standard: `*_test.go` suffix

**Expected structure based on codebase layout:**
```
pkg/product/
    product_repo.go
    product_repo_test.go      # Does not exist
    product_usecase.go
    product_usecase_test.go   # Does not exist
cmd/api/controllers/
    product_controller.go
    product_controller_test.go # Does not exist
```

## Current State: No Tests

There are zero test files in the entire codebase. This means:
- 0% test coverage
- No established testing patterns to follow
- No mocking infrastructure
- No test fixtures or factories

## Recommended Test Structure

Based on the repository/usecase/controller architecture, follow this approach when adding tests:

**Unit Tests for Usecases:**
```go
// pkg/product/product_usecase_test.go
package product

import (
    "testing"
    "github.com/boywatz/soda_shop_api/entity"
)

// Mock the repository interface
type mockProductRepo struct {
    findAllFn  func(products *[]entity.Product) error
    findByIdFn func(product *entity.Product, id uint) error
    createFn   func(product *entity.Product) error
}

func (m *mockProductRepo) FindAll(products *[]entity.Product) error {
    return m.findAllFn(products)
}

func (m *mockProductRepo) FindById(product *entity.Product, id uint) error {
    return m.findByIdFn(product, id)
}

func (m *mockProductRepo) Create(product *entity.Product) error {
    return m.createFn(product)
}
// ... implement remaining interface methods

func TestCreateProduct_Success(t *testing.T) {
    // Arrange
    repo := &mockProductRepo{
        createFn: func(product *entity.Product) error {
            return nil
        },
    }
    uc := NewProductUsecase(repo)
    product := &entity.Product{Name: "Test Product"}

    // Act
    err := uc.CreateProduct(product)

    // Assert
    if err != nil {
        t.Errorf("expected no error, got %v", err)
    }
}
```

**Integration Tests for Repositories:**
```go
// pkg/product/product_repo_test.go
// Requires a test database connection
// Use test containers or a dedicated test DB
```

## Mocking

**Framework:** None installed. Options to add:
- Hand-rolled mocks (define mock structs implementing interfaces) -- simplest approach given existing interface-based architecture
- `github.com/stretchr/testify/mock` -- popular, adds assertion helpers
- `github.com/golang/mock` (gomock) -- code generation approach

**What to Mock:**
- Repository interfaces when testing usecases (`ProductRepository`, `AuthRepository`, etc.)
- `OmiseService` interface when testing order controller
- External services: `services.SendEmail`, `services.HashPassword`

**What NOT to Mock:**
- Entity structs and value objects
- The `response.ApiResponse` helper
- Config functions (use env vars in test setup instead)

**Mockability assessment:**
- All repository and usecase layers use interfaces -- easy to mock
- `services/` package uses package-level functions (not interfaces) for `SendEmail`, `HashPassword`, `CheckHashPassword` -- harder to mock, would need refactoring to interfaces or function injection
- `services.OmiseService` is already an interface -- easy to mock

## Fixtures and Factories

**Test Data:**
- No fixtures, factories, or seed data files exist for testing
- `cmd/database/database.go` contains `createFirstAdmin()` which seeds a default admin user at startup -- this is production seeding, not test data

**Recommended location for test helpers:**
- `testutil/` or `internal/testutil/` for shared test helpers
- Co-locate fixtures with test files in each package

## Coverage

**Requirements:** None enforced. No CI/CD pipeline configured for coverage gates.

**View Coverage:**
```bash
go test -cover ./...                    # Summary
go test -coverprofile=coverage.out ./... # Generate profile
go tool cover -html=coverage.out         # HTML report
```

## Test Types

**Unit Tests:**
- Target: usecase layer (`pkg/{domain}/{domain}_usecase.go`) -- mock repository interfaces
- Target: service functions (`services/password.go`, `services/order_number_generator.go`)
- Target: utility functions (`utils/db_error.go`)
- Target: entity validation logic

**Integration Tests:**
- Target: repository layer (`pkg/{domain}/{domain}_repo.go`) -- requires test database
- Target: controller layer (`cmd/api/controllers/`) -- use `httptest` with Gin test mode
- Target: middleware (`cmd/api/middlewares/`) -- JWT parsing, CORS headers

**E2E Tests:**
- Not present. Would require a running server + test database
- Gin provides `gin.SetMode(gin.TestMode)` for test context

## Common Patterns (Recommended)

**Async Testing:**
- Not applicable currently. No goroutine-heavy code in business logic
- `services/order_number_generator.go` uses `sync.Mutex` -- test with `-race` flag

**Error Testing:**
```go
func TestFindById_NotFound(t *testing.T) {
    repo := &mockProductRepo{
        findByIdFn: func(product *entity.Product, id uint) error {
            return errors.New("record not found")
        },
    }
    uc := NewProductUsecase(repo)
    product := &entity.Product{}

    err := uc.FindById(product, 999)

    if err == nil {
        t.Error("expected error, got nil")
    }
}
```

**HTTP Handler Testing with Gin:**
```go
func TestGetProducts_Success(t *testing.T) {
    gin.SetMode(gin.TestMode)
    w := httptest.NewRecorder()
    c, r := gin.CreateTestContext(w)

    // Setup routes and mock dependencies
    // ...

    c.Request, _ = http.NewRequest("GET", "/api/v1/products", nil)
    r.ServeHTTP(w, c.Request)

    if w.Code != http.StatusOK {
        t.Errorf("expected 200, got %d", w.Code)
    }
}
```

## Priority Test Targets

Given zero coverage, prioritize testing in this order:

1. **`utils/db_error.go`** -- Simple pure function, easy first test
2. **`services/password.go`** -- `HashPassword` and `CheckHashPassword` are pure functions
3. **`pkg/auth/auth_usecase.go`** -- Login logic with JWT, critical security path
4. **`pkg/order/order_usecase.go`** -- Order processing, financial operations
5. **`cmd/api/middlewares/auth_middleware.go`** -- JWT validation, security boundary
6. **`cmd/api/controllers/`** -- HTTP handler tests for each resource

---

*Testing analysis: 2026-04-11*
