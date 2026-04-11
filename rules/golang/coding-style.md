# Go Coding Style & Architecture Rules

> These rules apply to all Go services and libraries. They are non-negotiable unless explicitly overridden in a project-specific `.cursorrules` or `AGENTS.md`.

> **Related documents:** [`testing.md`](./testing.md) — TDD, test types, race detection, coverage · [`security.md`](./security.md) — full security rules, secret management, incident response · [`performance.md`](./performance.md) — profiling, GC tuning, leak patterns, DB optimization.

---

## 1. Immutability (CRITICAL)

**Always create new objects. Never mutate existing ones.**

```go
// ❌ Wrong — mutates in place
func updateUser(u *User, name string) {
    u.Name = name
}

// ✅ Correct — returns a new copy with the change
func withName(u User, name string) User {
    u.Name = name
    return u
}
```

Rationale: Immutable data prevents hidden side effects, makes debugging easier, and enables safe concurrency. In Go, prefer returning updated value copies over pointer mutation except where performance profiling proves otherwise.

---

## 2. General Principles

### KISS — Keep It Simple, Stupid
- Prefer the simplest solution that works. Resist premature abstraction.
- If a function, type, or package is hard to name, it probably has too many responsibilities.
- Avoid clever one-liners when a clear multi-line block is more readable.
- Optimize for clarity over cleverness.

### DRY — Don't Repeat Yourself
- Extract repeated logic into a shared function or package — but only after it appears **at least twice**.
- Do not DRY prematurely. Duplicate once, abstract on the second repetition.
- Shared logic lives in `/pkg` (public) or `/internal/shared` (private).
- Avoid copy-paste implementation drift.

### YAGNI — You Aren't Gonna Need It
- Do not build features or abstractions before they are needed.
- Avoid speculative generality — start simple, then refactor when the pressure is real.
- Every layer of abstraction must justify its existence with a concrete, present need.

### Explicit over implicit
- No magic. No hidden side effects. No global mutable state.
- Prefer verbose clarity over terse ambiguity.

---

## 3. Functional Options Pattern

Use the **functional options** pattern when a constructor has optional or evolving configuration. It avoids boolean flags, growing parameter lists, and config structs that break callers on every field addition.

### When to use
- A constructor has more than 2-3 optional parameters.
- Configuration may grow over time without breaking existing call sites.
- You want discoverable, self-documenting API options.

### Pattern

```go
// options.go

type ServerConfig struct {
    host           string
    port           int
    timeout        time.Duration
    maxConnections int
    logger         *zap.Logger
}

// Option is a function that mutates ServerConfig.
// This is the ONE place mutation is acceptable — during construction only.
type Option func(*ServerConfig)

func WithHost(host string) Option {
    return func(c *ServerConfig) { c.host = host }
}

func WithPort(port int) Option {
    return func(c *ServerConfig) { c.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(c *ServerConfig) { c.timeout = d }
}

func WithMaxConnections(n int) Option {
    return func(c *ServerConfig) { c.maxConnections = n }
}

func WithLogger(l *zap.Logger) Option {
    return func(c *ServerConfig) { c.logger = l }
}
```

```go
// server.go

type Server struct{ cfg ServerConfig }

// NewServer applies defaults first, then caller options on top.
func NewServer(opts ...Option) *Server {
    cfg := ServerConfig{
        host:           "0.0.0.0",
        port:           8080,
        timeout:        30 * time.Second,
        maxConnections: 100,
        logger:         zap.NewNop(),
    }
    for _, opt := range opts {
        opt(&cfg)
    }
    return &Server{cfg: cfg}
}
```

```go
// Call site — zero-friction, backward-compatible
srv := NewServer(
    WithPort(9090),
    WithTimeout(10 * time.Second),
    WithLogger(logger),
)
```

### Rules
- Defaults must be set inside the constructor before applying options — callers should always get a working zero-config object.
- Option functions must be pure and side-effect-free — they only set fields on the config struct.
- Export option functions, not the config struct fields.
- Do **not** use functional options for required parameters — those stay as positional constructor args.
- Combine with `uber-go/dig`: register option slices as providers when options come from config/env.

```go
// ❌ Wrong — required ID passed as option
NewService(WithID("abc"))

// ✅ Correct — required params positional, optional config as options
NewService(ctx, "abc", WithTimeout(5*time.Second), WithLogger(l))
```

## 4. Code Conventions

### Naming
| Element | Convention | Example |
|---|---|---|
| Variables | camelCase | `userID`, `httpClient` |
| Functions / Methods | camelCase | `getUserByID` |
| Exported symbols | PascalCase | `UserService`, `GetByID` |
| Constants | PascalCase preferred; `UPPER_SNAKE_CASE` only for true global consts | `MaxRetries`, `DEFAULT_TIMEOUT` |
| Packages | lowercase single word | `user`, `order`, `payment` |
| Interfaces | `-er` suffix or behavior noun | `Reader`, `UserStore`, `EventPublisher` |
| Receiver names | 1–2 char abbreviation | `u *User`, `s *Service` |
| Booleans | `is`, `has`, `should`, or `can` prefix | `isActive`, `hasPermission`, `canRetry` |

- Do **not** use underscores in names except in test functions (`Test_someFunc`).
- Do **not** stutter: `user.UserService` → `user.Service`.
- Acronyms follow Go style: `userID` not `userId`, `httpURL` not `httpUrl`.

### Comments & Documentation
- All exported symbols **must** have a godoc comment starting with the symbol name.
- Comments explain *why*, not *what*.
- Avoid commented-out code — delete it, use version control.

```go
// UserService handles business operations for user accounts.
type UserService struct { ... }

// GetByID returns a user by their unique identifier.
// Returns ErrNotFound if the user does not exist.
func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) { ... }
```

### Formatting
- Always run `gofmt` / `goimports`. CI must reject unformatted code.
- Max line length: **120 characters** (soft limit, enforced by `golangci-lint`).
- Group imports: stdlib → external → internal, separated by blank lines.

```go
import (
    "context"
    "fmt"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    "github.com/yourorg/service/internal/domain"
)
```

---

## 5. Function & File Constraints

### Function Rules
- **Maximum 120 lines** per function. If it exceeds this, split it.
- **Typical target: ≤ 50 lines** — keep functions focused and testable.
- **Maximum 4–5 input parameters**. If more are needed, use an options struct.
- `context.Context` **always goes first** and is **never stored as a struct field**.

```go
// ✅ Correct
func (s *OrderService) Create(ctx context.Context, userID string, req CreateOrderRequest) (*Order, error)

// ❌ Wrong — context stored in struct
type Service struct {
    ctx context.Context // NEVER do this
}

// ❌ Wrong — context not first
func Create(req CreateOrderRequest, userID string, ctx context.Context) (*Order, error)

// ✅ Use options struct when params grow
type CreateOrderOptions struct {
    UserID    string
    ProductID string
    Quantity  int
    Notes     string
}
func (s *OrderService) Create(ctx context.Context, opts CreateOrderOptions) (*Order, error)
```

### File Rules
- **Typical size: 200–400 lines**. Aim for high cohesion, low coupling.
- **Hard maximum: 800 lines** (1000 absolute ceiling for generated/legacy files).
- One primary type or concern per file. Extract utilities from large modules.
- Organize by **feature/domain**, not by type (avoid `handlers/`, `models/` flat dirs).
- File names are `snake_case.go`, matching the primary type or function group.

```
user_service.go       // UserService type and methods
user_repository.go    // UserRepository interface + implementation
user_handler.go       // HTTP handler for user routes
```

### Code Smells to Avoid

**Deep Nesting** — prefer early returns over nested conditionals:
```go
// ❌ Deep nesting
func process(u *User) error {
    if u != nil {
        if u.IsActive {
            if u.HasRole("admin") {
                // actual logic buried here
            }
        }
    }
    return nil
}

// ✅ Early returns
func process(u *User) error {
    if u == nil        { return ErrNilUser }
    if !u.IsActive     { return ErrInactiveUser }
    if !u.HasRole("admin") { return ErrUnauthorized }
    // actual logic here
    return nil
}
```

**Magic Numbers** — use named constants:
```go
// ❌
time.Sleep(30 * time.Second)
if attempts > 3 { ... }

// ✅
const (
    HealthCheckInterval = 30 * time.Second
    MaxRetryAttempts    = 3
)
```

**Long Functions** — split into focused pieces with clear responsibilities. If you need a comment to explain a block, that block should be its own function.

---

## 6. Error Handling

Use **`github.com/pkg/errors`** instead of `fmt.Errorf` — it captures a full stack trace at the point of origin, which `fmt.Errorf` does not.

```go
import "github.com/pkg/errors"
```

### Rules
- Always handle errors explicitly. Never use `_` to discard errors silently.
- **Wrap with context** using `errors.Wrap` or `errors.Wrapf` — never `fmt.Errorf`.
- **Create new errors** using `errors.New` or `errors.Errorf`.
- Define domain-level sentinel errors in the domain layer.
- Use `errors.Is` / `errors.As` (stdlib) for error inspection, never string comparison.
- Only wrap once per layer — do not re-wrap an already-wrapped error with the same context.

### When to use each function

| Function | Use case |
|---|---|
| `errors.New("msg")` | New sentinel / leaf error, no cause |
| `errors.Errorf("msg %s", val)` | New error with formatted message |
| `errors.Wrap(err, "msg")` | Wrap an existing error with context + stack trace |
| `errors.Wrapf(err, "msg %s", val)` | Wrap with formatted context message |
| `errors.Cause(err)` | Unwrap to root cause for sentinel comparison |

### Examples

```go
import (
    "github.com/pkg/errors"
    "github.com/yourorg/service/internal/domain"
)

// Domain sentinel errors — use stdlib errors.New, no stack trace needed here
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("conflict")
)

// ✅ Wrap at the repository boundary — stack trace captured here
func (r *postgresUserRepo) FindByID(ctx context.Context, id string) (*domain.User, error) {
    var user domain.User
    err := r.db.GetContext(ctx, &user, `SELECT * FROM users WHERE id = $1`, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound
        }
        return nil, errors.Wrapf(err, "postgresUserRepo.FindByID id=%s", id)
    }
    return &user, nil
}

// ✅ Wrap at the usecase boundary with additional context
func (s *UserService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, errors.Wrapf(err, "UserService.GetByID id=%s", id)
    }
    return user, nil
}

// ✅ Inspect at the handler boundary — use errors.Cause for sentinel match
func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.usecase.GetByID(c.Request.Context(), c.Param("id"))
    if err != nil {
        switch errors.Cause(err) {
        case domain.ErrNotFound:
            c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        case domain.ErrUnauthorized:
            c.JSON(http.StatusForbidden, gin.H{"error": "forbidden"})
        default:
            // Log full stack trace here
            h.logger.Error("unexpected error", zap.Error(err))
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        }
        return
    }
    c.JSON(http.StatusOK, toUserResponse(user))
}

// ✅ Log with stack trace using %+v
logger.Error(fmt.Sprintf("%+v", err)) // prints full stack
```

### Do NOT use `fmt.Errorf` for wrapping

```go
// ❌ No stack trace, loses origin
return nil, fmt.Errorf("userService.GetByID: %w", err)

// ✅ Stack trace captured
return nil, errors.Wrapf(err, "userService.GetByID id=%s", id)
```

---

## 7. Clean Architecture

### Layer Structure
```
/cmd
  /server           → main entrypoint, DI wiring only
/internal
  /domain           → entities, value objects, domain errors, interfaces
  /usecase          → business logic; depends only on domain interfaces
  /repository       → DB/cache implementations of domain interfaces
  /handler          → HTTP/gRPC; thin, delegates to usecases
  /middleware       → auth, logging, rate-limiting, recovery
/pkg
  /logger           → shared logger setup
  /config           → config loading
  /validator        → shared validation helpers
```

### Dependency Rule
- Dependencies point **inward only**: `handler → usecase → domain ← repository`.
- The **domain layer has zero external dependencies**.
- Interfaces are **defined in the domain layer** and implemented in outer layers.

```go
// domain/port.go — interface defined in domain
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// repository/postgres_user_repo.go — implemented in repository
type postgresUserRepo struct { db *sqlx.DB }

func (r *postgresUserRepo) FindByID(ctx context.Context, id string) (*domain.User, error) { ... }
```

### Handlers are thin
Handlers do **only**: parse input → validate → call usecase → map response. No business logic.

```go
func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")
    user, err := h.usecase.GetByID(c.Request.Context(), id)
    if err != nil {
        h.respond.Error(c, err)
        return
    }
    c.JSON(http.StatusOK, toUserResponse(user))
}
```

### Dependency Injection

Use **`uber-go/dig`** as the preferred DI container — lightweight, reflection-based, and framework-agnostic. Use `uber-go/fx` only if you need full lifecycle hooks and module grouping at app-framework scale.

```go
import "go.uber.org/dig"

func BuildContainer() *dig.Container {
    c := dig.New()
    c.Provide(config.NewConfig)
    c.Provide(db.NewGormDB)
    c.Provide(repository.NewUserRepository)
    c.Provide(usecase.NewUserService)
    c.Provide(handler.NewUserHandler)
    return c
}

func main() {
    c := BuildContainer()
    if err := c.Invoke(func(h *handler.UserHandler) {
        // start server
    }); err != nil {
        log.Fatal(err)
    }
}
```

- Register constructors with `c.Provide`, never pre-built instances.
- Constructor parameters are resolved automatically by type — keep types distinct.
- No `init()` functions for side effects. No global singletons.
- For small services (≤ 3 layers), manual wiring in `main.go` is still acceptable and preferred over adding a DI dependency.

---

## 8. Schema-First Development

> **All contracts must be defined before implementation.** Code is derived from schemas, not the other way around.

### REST API — OpenAPI First
- Write `openapi.yaml` (OpenAPI 3.x) before writing any handler code.
- Use `oapi-codegen` or `ogen` to generate server stubs and request/response types.
- The OpenAPI spec is the source of truth for API contracts.

```yaml
# openapi.yaml
paths:
  /users/{id}:
    get:
      operationId: getUserByID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
```

### gRPC — Proto First
- Define `.proto` files before writing any service code.
- All `.proto` files live in `/proto` or a dedicated `proto` repository.
- Use `buf` for linting, breaking-change detection, and code generation.
- Never hand-write generated protobuf types.

```proto
// proto/user/v1/user.proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  string id = 1;
}
```

### Database — Schema First (Migrations)

Use **`golang-migrate/migrate`** for versioned schema migrations.

- Migration files live in `/migrations`, numbered sequentially: `000001_create_users.up.sql` / `000001_create_users.down.sql`.
- Never alter the DB schema manually in production.
- Run migrations at startup (before serving traffic) or via a dedicated `migrate` CLI step in CI/CD.

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(databaseURL string) error {
    m, err := migrate.New("file://migrations", databaseURL)
    if err != nil {
        return errors.Wrap(err, "migrate.New")
    }
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return errors.Wrap(err, "migrate.Up")
    }
    return nil
}
```

Migration file example:
```sql
-- migrations/000003_create_users.up.sql
CREATE TABLE users (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email      TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- migrations/000003_create_users.down.sql
DROP TABLE users;
```

### Database ORM & Query Strategy

| Scenario | Tool | Rationale |
|---|---|---|
| Standard CRUD, simple queries | **GORM** | Type-safe, SQL injection protection, readable |
| Complex queries, reporting, bulk ops | **sqlc** or raw GORM `Raw` | Full SQL control, no ORM magic |
| Never | Raw string-concatenated SQL | SQL injection risk |

**Use GORM as the default.** It provides type checking, parameterized queries by default, and reduces boilerplate for 80% of queries.

```go
import "gorm.io/gorm"

// ✅ GORM — type-safe, no injection risk
func (r *userRepo) FindByID(ctx context.Context, id string) (*domain.User, error) {
    var user User
    result := r.db.WithContext(ctx).First(&user, "id = ?", id)
    if errors.Is(result.Error, gorm.ErrRecordNotFound) {
        return nil, domain.ErrNotFound
    }
    if result.Error != nil {
        return nil, errors.Wrap(result.Error, "userRepo.FindByID")
    }
    return toDomain(&user), nil
}

// ✅ GORM for simple create
func (r *userRepo) Save(ctx context.Context, u *domain.User) error {
    row := toModel(u)
    if err := r.db.WithContext(ctx).Create(&row).Error; err != nil {
        return errors.Wrap(err, "userRepo.Save")
    }
    return nil
}

// ✅ sqlc or GORM Raw for complex queries
func (r *userRepo) FindActiveWithOrders(ctx context.Context) ([]*domain.User, error) {
    var results []UserWithOrderCount
    err := r.db.WithContext(ctx).Raw(`
        SELECT u.id, u.email, COUNT(o.id) AS order_count
        FROM users u
        LEFT JOIN orders o ON o.user_id = u.id
        WHERE u.is_active = true
        GROUP BY u.id
    `).Scan(&results).Error
    if err != nil {
        return nil, errors.Wrap(err, "userRepo.FindActiveWithOrders")
    }
    return toDomainList(results), nil
}
```

GORM rules:
- Always pass `WithContext(ctx)` — never call GORM methods without a context.
- Map GORM model structs to domain types at the repository boundary — do not leak GORM models into usecases or handlers.
- Use `gorm.ErrRecordNotFound` to detect missing records, not row count checks.
- Configure connection pool on the underlying `*sql.DB`:

```go
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(25)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(5 * time.Minute)
```

---

## 9. Security

> Full security rules, code examples, and response protocol are in **[`security.md`](./security.md)**.

Key rules summarised here for quick reference:

- **Secrets** — never hardcode; validate all required env vars at startup; fail fast.
- **SQL** — GORM or parameterized queries only; never concatenate user input.
- **Input validation** — validate at every system boundary using `go-playground/validator`.
- **Passwords** — `bcrypt` (cost ≥ 12) or `argon2id`; never plaintext.
- **JWT** — verify signing algorithm explicitly; require `exp`; load secret from env.
- **Authorization** — enforce in the usecase layer, not only at the handler.
- **Rate limiting** — every public endpoint; Redis-backed for multi-instance deployments.
- **Error responses** — never expose stack traces, DB errors, or internal paths to clients.
- **PII in logs** — never; use opaque IDs (`userID`, `traceID`) only.
- **Security headers** — `X-Content-Type-Options`, `X-Frame-Options`, `HSTS`, CSP.
- **Static analysis** — `gosec` via `golangci-lint`; mandatory CI gate; zero medium/high findings.
- **Dependencies** — `govulncheck ./...`; mandatory CI gate.
- **CSRF** — required for cookie-authenticated, state-changing endpoints.
- **XSS** — use `html/template`; set `X-Content-Type-Options: nosniff`.
- **Incident response** — CRITICAL issues: stop, contain, fix, rotate secrets, regression test, post-mortem.


## 10. Observability & Trace ID

Every request must carry a **trace ID** from the moment it enters the system to every log line, error, outbound call, and response header. Without it, debugging distributed failures across services or correlating logs for a single request is effectively impossible.

### Trace ID Rules

- Generate a trace ID at the **outermost boundary** (HTTP middleware or gRPC interceptor) if one is not already present in the incoming request headers.
- Propagate the trace ID via `context.Context` — never as a function parameter or struct field.
- Include the trace ID in **every log line** in every layer.
- Return the trace ID in the response header (`X-Trace-ID`) so clients can report it in bug reports.
- Forward the trace ID in all **outbound calls** (HTTP headers, gRPC metadata, message queue attributes).
- Never generate a new trace ID inside a usecase or repository — only read from context.

---

### Context Key

Define the trace ID context key in a shared package to avoid collisions:

```go
// pkg/trace/trace.go

package trace

import "context"

type contextKey struct{}

const Header = "X-Trace-ID"

// NewContext returns a new context carrying the given trace ID.
func NewContext(ctx context.Context, traceID string) context.Context {
    return context.WithValue(ctx, contextKey{}, traceID)
}

// FromContext extracts the trace ID from the context.
// Returns an empty string if not set.
func FromContext(ctx context.Context) string {
    v, _ := ctx.Value(contextKey{}).(string)
    return v
}
```

---

### HTTP Middleware — Inject at the Edge

```go
// middleware/trace.go

func TraceMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        traceID := c.GetHeader(trace.Header)
        if traceID == "" {
            traceID = uuid.NewString() // generate if client didn't send one
        }

        // Inject into context for downstream use
        ctx := trace.NewContext(c.Request.Context(), traceID)
        c.Request = c.Request.WithContext(ctx)

        // Return to caller so they can correlate client-side
        c.Header(trace.Header, traceID)

        c.Next()
    }
}
```

Register it as the **first** middleware — before auth, logging, or any handler:

```go
r := gin.New()
r.Use(middleware.TraceMiddleware())
r.Use(middleware.LoggingMiddleware())
r.Use(middleware.AuthMiddleware())
```

---

### Structured Logging with Trace ID

Always extract and attach the trace ID in every log call. Use a logger helper that does this automatically:

```go
// pkg/logger/logger.go

func WithTrace(ctx context.Context, logger *zap.Logger) *zap.Logger {
    traceID := trace.FromContext(ctx)
    if traceID == "" {
        return logger
    }
    return logger.With(zap.String("traceID", traceID))
}
```

Usage in every layer:

```go
// Handler
func (h *UserHandler) GetUser(c *gin.Context) {
    ctx := c.Request.Context()
    log := logger.WithTrace(ctx, h.logger)

    user, err := h.usecase.GetByID(ctx, c.Param("id"))
    if err != nil {
        log.Error("GetUser failed", zap.Error(err))
        // ...
        return
    }
    log.Info("GetUser success", zap.String("userID", user.ID))
    c.JSON(http.StatusOK, toUserResponse(user))
}

// Usecase
func (s *UserService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    log := logger.WithTrace(ctx, s.logger)
    log.Info("fetching user", zap.String("userID", id))

    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        log.Warn("user not found", zap.String("userID", id), zap.Error(err))
        return nil, errors.Wrapf(err, "UserService.GetByID id=%s", id)
    }
    return user, nil
}
```

Every log line now includes `traceID` automatically — no manual threading of the ID through parameters.

---

### Error Responses — Include Trace ID

Return the trace ID in error responses so clients can quote it in support tickets:

```go
// pkg/respond/respond.go

type ErrorResponse struct {
    Error   string `json:"error"`
    TraceID string `json:"traceId,omitempty"`
}

func Error(c *gin.Context, err error) {
    traceID := trace.FromContext(c.Request.Context())
    statusCode := toHTTPStatus(err)

    c.JSON(statusCode, ErrorResponse{
        Error:   userFacingMessage(err),
        TraceID: traceID,
    })
}
```

Client-visible error example:
```json
{
  "error": "user not found",
  "traceId": "4a7f3c91-2b1e-4d08-9e6a-1c3f7a2b5d9e"
}
```

---

### Outbound HTTP Calls — Forward the Trace ID

When calling external services, forward the trace ID in the request header:

```go
func (c *httpClient) Get(ctx context.Context, url string) (*http.Response, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, errors.Wrap(err, "http.NewRequest")
    }

    // Forward trace ID to downstream service
    if traceID := trace.FromContext(ctx); traceID != "" {
        req.Header.Set(trace.Header, traceID)
    }

    return c.client.Do(req)
}
```

---

### OpenTelemetry Integration (Optional but Recommended)

For services that need distributed tracing across multiple services, integrate with OpenTelemetry. The trace ID from context maps directly to the OTel span:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func (h *UserHandler) GetUser(c *gin.Context) {
    ctx, span := otel.Tracer("user-service").Start(c.Request.Context(), "GetUser")
    defer span.End()

    // OTel span ID can be used as the trace ID for log correlation
    traceID := span.SpanContext().TraceID().String()
    ctx = trace.NewContext(ctx, traceID)

    // ... rest of handler
}
```

When using OTel, the middleware-generated UUID trace ID is replaced by the OTel trace ID — use the same `trace.NewContext` / `trace.FromContext` helpers so all logging stays consistent.

---

### Summary — Trace ID at Every Layer

```
Inbound request
    └── TraceMiddleware         → generate/read X-Trace-ID, inject into ctx
          └── Handler           → log with traceID, call usecase
                └── Usecase     → log with traceID, call repository
                      └── Repo  → log with traceID, query DB
                └── HTTP client → forward X-Trace-ID header to downstream
    └── Response                → return X-Trace-ID header + traceId in error body
```

## 11. Performance & Optimization

> Full profiling workflows, benchmarking, GC tuning, memory/goroutine leak patterns, DB pool tuning, caching strategy, and build optimization are in **[`performance.md`](./performance.md)**.

Key rules summarised here for quick reference:

- **Profile first** — never optimize without a measurement. `pprof` CPU + heap before any change.
- **Benchmarks** — `go test -bench=. -benchmem -count=5`; use `benchstat` for before/after comparison.
- **Preallocate** — `make([]T, 0, n)` when size is known; `make(map[K]V, n)` for maps.
- **Buffer reuse** — `sync.Pool` for hot-path buffers (JSON, byte slices); always `Reset()` before reuse.
- **Strings** — `strings.Builder` with `Grow` hint; never `+` in loops.
- **Goroutine leaks** — every goroutine needs a stop path (context cancellation or channel close); `goleak.VerifyTestMain` in every test suite.
- **Memory leaks** — `defer resp.Body.Close()`, `defer cancel()`, `defer ticker.Stop()`, copy sub-slices.
- **GC** — set `GOMEMLIMIT` to ~80% of container memory limit in production; prefer `[]struct` over `[]*struct`.
- **DB pool** — `SetMaxOpenConns(25)`, `SetMaxIdleConns(10)`, `SetConnMaxLifetime(5m)`.
- **N+1 queries** — `Preload`, JOINs, or batch fetch; never query inside a loop.
- **Cache** — repository/usecase layer only; always TTL; invalidate on write; document strategy.
- **http.Client** — shared singleton with tuned `Transport`; never create per-request.


## 12. Tooling & Linting

Required tools in every project:

| Tool | Purpose |
|---|---|
| `gofmt` / `goimports` | Formatting |
| `golangci-lint` | Linting (see config below) |
| `buf` | Proto linting & generation |
| `gorm.io/gorm` | ORM for standard CRUD (default) |
| `sqlc` | Type-safe SQL for complex queries |
| `golang-migrate/migrate` | DB migrations (preferred) |
| `uber-go/dig` | Dependency injection (preferred) |
| `oapi-codegen` or `ogen` | OpenAPI code generation |
| `govulncheck` | Dependency vulnerability scanning |
| `goleak` | Goroutine leak detection in tests |

Recommended `golangci-lint` linters:
```yaml
linters:
  enable:
    - errcheck        # unchecked errors
    - gosimple        # simplify code
    - govet           # vet checks
    - ineffassign     # ineffectual assignment
    - staticcheck     # static analysis
    - unused          # unused code
    - gocognit        # cognitive complexity
    - funlen          # function length (120 lines hard max)
    - gocritic        # opinionated checks
    - gosec           # SECURITY: hardcoded creds, weak crypto, SQL injection, etc.
    - noctx           # missing context in HTTP calls
    - bodyclose       # unclosed HTTP response bodies
    - exhaustive      # exhaustive enum switches
    - containedctx    # context stored in struct (banned)
    - contextcheck    # context propagation correctness
    - nilerr          # returning nil with non-nil error
    - wrapcheck       # errors from external packages must be wrapped

linters-settings:
  funlen:
    lines: 120
    statements: 80
  gosec:
    severity: medium
    confidence: medium
  gocognit:
    min-complexity: 15
```

---

## Quick Reference — Rules at a Glance

| Rule | Limit / Standard |
|---|---|
| Function length | ≤ 50 lines typical, 120 hard max |
| Function parameters | ≤ 5 (context counts as 1st) |
| File length | 200–400 typical, 800 hard max |
| `context.Context` position | Always 1st param, never a struct field |
| Mutation | Never mutate — return new copies |
| SQL (simple) | GORM with `WithContext` — no raw strings |
| SQL (complex) | sqlc or GORM `Raw` — never string concat |
| Migrations | `golang-migrate` — versioned up/down files |
| DI | `uber-go/dig` — constructor-based providers |
| Error wrapping | `pkg/errors` — `Wrap`/`Wrapf`, never `fmt.Errorf` |
| Secrets | Never hardcoded, always env/secrets manager |
| CORS origins | Never wildcard `*` in production |
| Schema | OpenAPI / proto / migration first, always |
| Abstractions | YAGNI — DRY after 2nd repetition, not before |
| Nesting depth | Max 4 levels — use early returns |
| Goroutines | Always have a termination path via context |
| Memory | Close bodies, stop tickers, copy sub-slices |
| GC | Set `GOMEMLIMIT` in containers; profile before tuning |
| PII in logs | Never — use opaque IDs only |
| Context timeouts | Always — no network/DB call without a deadline |
| Security scan | `gosec ./...` via `golangci-lint` — mandatory in CI |
| Functional options | Use for optional config; required params stay positional |
| Trace ID | Generate at edge, propagate via ctx, log in every layer |
| Testing standards | See `testing.md` — TDD, AAA, race detection, coverage |

---

## Code Quality Checklist

Before marking work complete:

- [ ] Code is readable and well-named
- [ ] No mutation — new objects returned, not modified in place
- [ ] Functions are small (≤ 50 lines typical, ≤ 120 hard max)
- [ ] Files are focused (≤ 800 lines)
- [ ] No deep nesting (> 4 levels) — use early returns
- [ ] Proper and explicit error handling — `pkg/errors` used, no silent `_` discard
- [ ] No hardcoded values — use named constants or config
- [ ] `context.Context` is first param, never stored in structs
- [ ] Schema defined before implementation (OpenAPI / proto / migration)
- [ ] Input validated at system boundaries
- [ ] No secrets in code or version control
- [ ] GORM used for SQL with `WithContext` — no raw string concatenation
- [ ] Migrations managed via `golang-migrate`
- [ ] DI wired via `uber-go/dig` constructors
- [ ] Every goroutine has a termination condition (context / channel close)
- [ ] HTTP response bodies closed, tickers stopped, sub-slices copied
- [ ] `GOMEMLIMIT` set for containerized deployments — see `performance.md` for full investigation checklist
- [ ] No PII in logs — opaque IDs only (`userID`, `requestID`)
- [ ] All network/DB calls have a context deadline (`WithTimeout` or `WithDeadline`)
- [ ] `gosec` passes with no medium/high findings — see `security.md` for full pre-commit checklist
- [ ] Functional options used for constructors with optional config
- [ ] Trace ID middleware registered as first middleware
- [ ] Every log line includes `traceID` via `logger.WithTrace(ctx, logger)`
- [ ] Trace ID returned in `X-Trace-ID` response header and error body
- [ ] Outbound HTTP/gRPC calls forward the trace ID header
- [ ] All testing requirements satisfied — see **`testing.md`** for the full checklist