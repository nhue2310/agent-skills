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

