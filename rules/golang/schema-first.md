## 1. Schema-First Development

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
