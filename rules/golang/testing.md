# Testing Requirements & Standards

> These rules apply to all Go services and libraries. They are non-negotiable unless explicitly overridden in a project-specific `.cursorrules` or `AGENTS.md`.

---

## 1. Minimum Test Coverage: 80%

- **80% minimum** on `domain` and `usecase` packages — enforced in CI as a hard gate.
- Coverage of `handler` and `repository` packages is satisfied by integration/E2E tests.
- Do **not** write meaningless tests just to hit a number. Coverage is a signal, not a goal.
- Uncovered lines in critical paths (auth, payments, data mutation) must be justified in a comment or covered by an integration test.

```bash
# Basic coverage
go test -cover ./...

# HTML report for line-by-line inspection
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Coverage + race detection (required in CI)
go test -race -coverprofile=coverage.out ./...
```

---

## 2. Test Types (ALL Required)

### Unit Tests
- Cover individual functions, domain logic, and usecase business rules.
- Mock **all** external dependencies — no real DB, no real HTTP calls.
- Live alongside the code they test: `user_service_test.go`.
- Fast: entire unit suite should complete in seconds.

### Integration Tests
- Cover API endpoints and database operations against a real DB.
- Use `testcontainers-go` to spin up a real Postgres/Redis instance per test run.
- Test the full repository layer — confirm queries, migrations, and constraints work correctly.

### E2E Tests
- Cover critical user flows end-to-end through the full HTTP stack.
- Validate handler behavior against the OpenAPI / proto contract.
- Run in a staging-like environment in CI — not on every commit if slow.

| Test Type | Scope | DB | Speed | Coverage Target |
|---|---|---|---|---|
| Unit | domain, usecase | Mocked | Fast (< 5s) | 80%+ |
| Integration | repository, handler | Real (testcontainers) | Medium | Key paths |
| E2E | Full HTTP flows | Real | Slow | Critical flows |

---

## 3. Test-Driven Development (TDD)

**Mandatory workflow for all new features and bug fixes:**

```
RED   → Write the test first. Run it. It must FAIL.
GREEN → Write the minimal implementation. Run it. It must PASS.
IMPROVE → Refactor. Re-run. Verify coverage ≥ 80%.
```

1. **Write test first (RED)** — define the expected behavior before writing a single line of implementation.
2. **Run test — it must FAIL** — a test that passes before implementation exists is not testing anything.
3. **Write minimal implementation (GREEN)** — write only enough code to make the test pass. No more.
4. **Run test — it must PASS** — confirm the implementation satisfies the contract.
5. **Refactor (IMPROVE)** — clean up code, eliminate duplication, improve naming. Tests must still pass.
6. **Verify coverage (80%+)** — run `go test -race -cover ./...` and confirm the gate is met.

> If you feel the urge to write implementation before the test — stop. Write the test first.

---

## 4. Test Structure — AAA Pattern

All tests must follow the **Arrange-Act-Assert** structure. This makes tests readable, debuggable, and intention-clear.

```go
func TestCalculateCosineSimilarity_OrthogonalVectors(t *testing.T) {
    // Arrange
    vector1 := []float64{1, 0, 0}
    vector2 := []float64{0, 1, 0}

    // Act
    similarity := calculateCosineSimilarity(vector1, vector2)

    // Assert
    assert.Equal(t, 0.0, similarity)
}
```

Never mix arrange, act, and assert phases — keep them visually separated with blank lines or comments.

---

## 5. Test Naming

Use **descriptive names that explain the behavior under test**, not the implementation detail.

Format: `Test<Subject>_<Scenario>` or `Test<Subject>_<Condition>_<ExpectedOutcome>`

```go
// ✅ Describes behavior
func TestUserService_GetByID_ReturnsNotFoundWhenUserDoesNotExist(t *testing.T)
func TestOrderService_Create_FailsWhenInventoryIsInsufficient(t *testing.T)
func TestAuthMiddleware_RejectsRequestWithExpiredToken(t *testing.T)

// ❌ Describes implementation, not behavior
func TestGetUser(t *testing.T)
func TestCreate(t *testing.T)
func TestMiddleware(t *testing.T)
```

More examples of good test names:

```go
func TestSearchService_ReturnsEmptyArrayWhenNoMarketsMatchQuery(t *testing.T)
func TestHTTPClient_FallsBackToSubstringSearchWhenRedisIsUnavailable(t *testing.T)
func TestPaymentProcessor_ThrowsErrorWhenAPIKeyIsMissing(t *testing.T)
func TestUserRepo_SaveReturnsDuplicateErrorOnConflict(t *testing.T)
```

---

## 6. Table-Driven Tests

Prefer table-driven tests for any function with multiple input/output combinations. They reduce boilerplate and make adding new cases trivial.

```go
func TestUserService_GetByID(t *testing.T) {
    tests := []struct {
        name    string
        id      string
        mock    func(*mockUserRepo)
        want    *domain.User
        wantErr error
    }{
        {
            name: "returns user when found",
            id:   "abc123",
            mock: func(m *mockUserRepo) {
                m.On("FindByID", mock.Anything, "abc123").
                    Return(&domain.User{ID: "abc123"}, nil)
            },
            want: &domain.User{ID: "abc123"},
        },
        {
            name: "returns ErrNotFound when user does not exist",
            id:   "missing",
            mock: func(m *mockUserRepo) {
                m.On("FindByID", mock.Anything, "missing").
                    Return(nil, domain.ErrNotFound)
            },
            wantErr: domain.ErrNotFound,
        },
        {
            name: "returns error on repository failure",
            id:   "boom",
            mock: func(m *mockUserRepo) {
                m.On("FindByID", mock.Anything, "boom").
                    Return(nil, errors.New("db timeout"))
            },
            wantErr: errors.New("db timeout"),
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            // Arrange
            repo := new(mockUserRepo)
            tc.mock(repo)
            svc := usecase.NewUserService(repo)

            // Act
            got, err := svc.GetByID(context.Background(), tc.id)

            // Assert
            if tc.wantErr != nil {
                assert.Error(t, err)
                assert.True(t, errors.Is(err, tc.wantErr) || err.Error() == tc.wantErr.Error())
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tc.want, got)
            }
        })
    }
}
```

---

## 7. Mocking

- Use `testify/mock` or `gomock` for interface mocking.
- **Never mock types you don't own** — wrap third-party types behind your own interfaces.
- Mock only at clean boundaries: domain interfaces defined in `domain/port.go`.
- Do not mock concrete structs — if you need to mock something, it should be behind an interface.

```go
// domain/port.go — the interface being mocked
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// mocks/user_repository_mock.go — generated or hand-written mock
type mockUserRepo struct{ mock.Mock }

func (m *mockUserRepo) FindByID(ctx context.Context, id string) (*domain.User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}
```

---

## 8. Test Isolation

- Every test must be **fully independent** — no shared state, no ordering dependencies.
- Use `t.Cleanup` to tear down resources created in a test.
- Use `t.Parallel()` for unit tests that have no shared state.
- Never use global variables in tests — declare everything locally inside each test or `setup` function.

```go
func TestUserRepo_FindByID(t *testing.T) {
    t.Parallel()

    // Arrange — isolated DB from testcontainers
    db := setupTestDB(t)
    t.Cleanup(func() { db.Close() })

    repo := repository.NewUserRepository(db)

    // seed test data
    seedUser(t, db, &domain.User{ID: "abc123", Email: "test@example.com"})

    // Act
    user, err := repo.FindByID(context.Background(), "abc123")

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "abc123", user.ID)
}
```

---

## 9. Race Detection

**Always run tests with the `-race` flag.** The race detector catches concurrent read/write conflicts that are otherwise invisible and non-deterministic in production.

```bash
# Required in CI — no exceptions
go test -race ./...
```

In CI, `-race` is **mandatory**. A pipeline without it is not a safety net. Race conditions caught locally cost minutes; those caught in production cost incidents.

```go
// ❌ Race — unsynchronized concurrent write
var counter int
func increment() { counter++ }

// ✅ Fix with mutex
var (
    mu      sync.Mutex
    counter int
)
func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

---

## 10. Goroutine Leak Detection

Register `goleak` in `TestMain` to catch goroutines that outlive their test:

```go
// main_test.go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

If a test leaks goroutines, `goleak` will fail it with a report of the offending stacks. This catches missing `cancel()` calls, unclosed channels, and runaway background workers.

---

## 11. Integration Tests with testcontainers-go

Use `testcontainers-go` to run integration tests against a real database — never mock the DB in integration tests.

```go
func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()

    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:16-alpine",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_USER":     "test",
            "POSTGRES_PASSWORD": "test",
            "POSTGRES_DB":       "testdb",
        },
        WaitingFor: wait.ForListeningPort("5432/tcp"),
    }

    pg, err := testcontainers.GenericContainer(ctx,
        testcontainers.GenericContainerRequest{
            ContainerRequest: req,
            Started:          true,
        })
    require.NoError(t, err)
    t.Cleanup(func() { pg.Terminate(ctx) })

    host, _ := pg.Host(ctx)
    port, _ := pg.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("host=%s port=%s user=test password=test dbname=testdb sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    require.NoError(t, err)

    // Run migrations
    require.NoError(t, RunMigrations(dsn))

    return db
}
```

---

## 12. Troubleshooting Test Failures

When a test fails, work through this checklist in order:

1. **Check test isolation** — does the test share state with another test? Use `t.Parallel()` safely or fix setup/teardown.
2. **Verify mocks are correct** — confirm mock return values match the scenario being tested.
3. **Fix implementation, not tests** — unless the test itself is wrong (wrong assertion, wrong scenario name). Tests define the contract; don't bend them to fit broken code.
4. **Run with verbose output** — `go test -v -run TestName ./...` to see the full failure trace.
5. **Check for races** — run `go test -race ./...` even if the test passes without it.
6. **Check for goroutine leaks** — if `goleak` reports a leak, find the missing `cancel()` or unread channel.

---

## CI Pipeline Requirements

Every CI run must execute all of the following — in this order:

```bash
# 1. Lint (includes gosec)
golangci-lint run ./...

# 2. Unit tests with race detection and coverage
go test -race -coverprofile=coverage.out ./...

# 3. Coverage gate — fail if below 80% on domain/usecase
go tool cover -func=coverage.out | grep -E "domain|usecase" | awk '{print $3}' 

# 4. Integration tests
go test -race -tags=integration ./...

# 5. Goroutine leak verification (via goleak in TestMain)
# — covered by step 2 automatically
```

---

## Test Quality Checklist

Before marking work complete:

- [ ] Tests written **before** implementation (TDD — RED first)
- [ ] All three test types present: unit, integration, E2E for critical flows
- [ ] AAA structure used — Arrange, Act, Assert clearly separated
- [ ] Test names describe behavior, not implementation
- [ ] Table-driven tests used for multi-case functions
- [ ] No shared state between tests
- [ ] Mocks only at domain interface boundaries
- [ ] `go test -race ./...` passes — no data races
- [ ] Coverage ≥ 80% on `domain` and `usecase` packages
- [ ] `goleak` registered in `TestMain`
- [ ] Integration tests use `testcontainers-go` — no mocked DB
- [ ] CI pipeline runs lint → unit → coverage gate → integration in order