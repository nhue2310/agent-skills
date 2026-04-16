## 1. Clean Architecture

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
