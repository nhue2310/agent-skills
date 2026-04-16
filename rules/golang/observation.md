## 7. Observability & Trace ID

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
