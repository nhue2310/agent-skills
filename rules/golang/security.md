# Security Guidelines — Go

> These rules apply to all Go services and libraries. They are non-negotiable unless explicitly overridden in a project-specific `.cursorrules` or `AGENTS.md`.
---

## Pre-Commit Security Checklist

Run this before every commit. CI will enforce it — but catching issues locally is faster and cheaper.

- [ ] No hardcoded secrets (API keys, passwords, tokens, connection strings)
- [ ] All user inputs validated at system boundary
- [ ] SQL uses GORM or parameterized queries — no string concatenation
- [ ] HTML output sanitized — XSS prevented
- [ ] CSRF protection enabled on state-changing endpoints
- [ ] Authentication checked — JWT validated, not just decoded
- [ ] Authorization checked in usecase layer, not only handler
- [ ] Rate limiting applied to all public endpoints
- [ ] Error responses do not leak stack traces, internal paths, or DB details
- [ ] No PII in logs — opaque IDs only
- [ ] `gosec ./...` passes with zero medium/high findings
- [ ] All secrets loaded from env or secrets manager — validated at startup
- [ ] Dependency vulnerabilities checked: `govulncheck ./...`

---

## 1. Secret Management

### Rules
- **Never** hardcode secrets in source code, config files, or test fixtures.
- **Always** load secrets from environment variables or a secrets manager (HashiCorp Vault, AWS SSM / Secrets Manager, GCP Secret Manager).
- **Validate** that all required secrets are present at startup — fail fast with a clear error before the server starts serving traffic.
- **Rotate** any secret that may have been exposed — immediately, without waiting to confirm the breach.
- Never log secret values, even at `DEBUG` level.
- Never store secrets in `.env` files committed to version control. Use `.env.example` with placeholder values instead.

### Startup Validation

Validate all required secrets at boot — never let a misconfigured service start silently

### Secret Rotation Protocol

If a secret is exposed (accidental commit, log leak, breach):

1. **STOP** — do not continue development until resolved.
2. **Rotate immediately** — revoke the old secret, issue a new one.
3. **Audit** — check logs for any unauthorized use of the exposed secret.
4. **Deploy** — roll out the new secret to all environments.
5. **Review** — scan the entire codebase for similar patterns: `git log -S "hardcoded_value"`.
6. **Post-mortem** — document what happened and how it will be prevented.

### .gitignore — Minimum Required Entries

```gitignore
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.pfx
secrets/
```

---

## 2. SQL Injection Prevention

Use **GORM** for all standard queries — it parameterizes by default. For complex queries, use `sqlc` or GORM `Raw` with placeholders. Never concatenate user input into SQL strings.

```go
// ❌ CRITICAL — SQL injection risk
query := "SELECT * FROM users WHERE email = '" + email + "'"
db.Raw(query).Scan(&user)

// ❌ Also dangerous — Sprintf into SQL
db.Raw(fmt.Sprintf("SELECT * FROM users WHERE id = %s", id)).Scan(&user)

// ✅ GORM — parameterized by default
db.WithContext(ctx).Where("email = ?", email).First(&user)

// ✅ GORM Raw with placeholders
db.WithContext(ctx).Raw("SELECT * FROM users WHERE email = ? AND active = ?", email, true).Scan(&user)

// ✅ sqlc — SQL injection impossible by design (compile-time checked)
user, err := queries.GetUserByEmail(ctx, email)
```

**GORM-specific pitfalls to avoid:**

```go
// ❌ Raw user input passed directly to Where string — still injectable
db.Where(userInput).Find(&users)

// ✅ Always use placeholder syntax
db.Where("status = ?", userInput).Find(&users)

// ❌ Order by user input — injectable
db.Order(c.Query("sort")).Find(&users)

// ✅ Allowlist sort columns
allowedSorts := map[string]bool{"created_at": true, "name": true}
sortCol := c.Query("sort")
if !allowedSorts[sortCol] {
    sortCol = "created_at"
}
db.Order(sortCol + " DESC").Find(&users)
```

---

## 3. Input Validation

**Validate at every system boundary** — HTTP handlers, gRPC interceptors, message queue consumers. Usecases and repositories must never receive unvalidated input.

```go
// Use go-playground/validator with struct tags
type CreateUserRequest struct {
    Name     string `json:"name"     validate:"required,min=2,max=100,alpha"`
    Email    string `json:"email"    validate:"required,email,max=255"`
    Password string `json:"password" validate:"required,min=8,max=72"`
    Age      int    `json:"age"      validate:"omitempty,min=13,max=130"`
    Role     string `json:"role"     validate:"required,oneof=admin user viewer"`
}

// Validation middleware / handler helper
func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest

    // Reject unknown fields
    dec := json.NewDecoder(c.Request.Body)
    dec.DisallowUnknownFields()
    if err := dec.Decode(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    if err := h.validator.Struct(req); err != nil {
        c.JSON(http.StatusUnprocessableEntity, gin.H{"error": toValidationErrors(err)})
        return
    }

    // Safe to pass to usecase
    user, err := h.usecase.CreateUser(c.Request.Context(), req)
    // ...
}
```

### File Path Validation

Never use user-supplied input directly in file paths:

```go
// ❌ Path traversal risk
filePath := filepath.Join("/uploads", userInput)
f, err := os.Open(filePath)

// ✅ Validate and clean the path
func safeFilePath(base, name string) (string, error) {
    // Clean and resolve to absolute path
    clean := filepath.Clean(filepath.Join(base, filepath.Base(name)))
    // Ensure the resolved path is still inside the base directory
    if !strings.HasPrefix(clean, filepath.Clean(base)+string(os.PathSeparator)) {
        return "", errors.New("invalid file path: traversal attempt detected")
    }
    return clean, nil
}
```

### URL / Redirect Validation

Never redirect to a user-supplied URL without validation:

```go
// ❌ Open redirect
http.Redirect(w, r, r.URL.Query().Get("next"), http.StatusFound)

// ✅ Allowlist or validate same-origin
func safeRedirect(next string) string {
    u, err := url.Parse(next)
    if err != nil || u.Host != "" {
        return "/dashboard" // default safe redirect
    }
    return next
}
```

---

## 4. XSS Prevention

Go's `html/template` package auto-escapes output — use it instead of `text/template` for any HTML rendering.

```go
// ❌ text/template — no escaping, XSS risk
import "text/template"
tmpl.Execute(w, userInput)

// ✅ html/template — context-aware auto-escaping
import "html/template"
tmpl.Execute(w, userInput)
```

For APIs returning JSON consumed by a frontend:
- Set `Content-Type: application/json` — browsers will not execute JSON as HTML.
- Set `X-Content-Type-Options: nosniff` to prevent MIME sniffing.

```go
// Security headers middleware
func SecurityHeadersMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        // For HTTPS-only services:
        c.Header("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
        c.Next()
    }
}
```

---

## 5. CSRF Protection

CSRF affects any endpoint that changes state and relies on cookie-based authentication.

```go
// Use gorilla/csrf or equivalent
import "github.com/gorilla/csrf"

// Setup — wrap the router
csrfMiddleware := csrf.Protect(
    []byte(cfg.CSRFSecret), // 32-byte random secret from env
    csrf.Secure(true),      // HTTPS only in production
    csrf.SameSite(csrf.SameSiteStrictMode),
    csrf.ErrorHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusForbidden)
        json.NewEncoder(w).Encode(map[string]string{"error": "CSRF token invalid"})
    })),
)

// Exempt: pure JSON APIs using Authorization header (not cookies) are not CSRF vulnerable
// Require CSRF token only on cookie-authenticated, state-changing endpoints
```

**APIs using `Authorization: Bearer <token>` header authentication are not vulnerable to CSRF** — only cookie-based auth requires CSRF protection.

---

## 7. Rate Limiting

Apply rate limiting at the **router level** as middleware. Every public endpoint must be rate-limited. Never rely solely on upstream infrastructure.

```go
import "golang.org/x/time/rate"

// Token bucket limiter — per IP
type RateLimiter struct {
    mu       sync.Mutex
    limiters map[string]*rate.Limiter
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    burst,
    }
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    if l, ok := rl.limiters[ip]; ok {
        return l
    }
    l := rate.NewLimiter(rl.rate, rl.burst)
    rl.limiters[ip] = l
    return l
}

func (rl *RateLimiter) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        ip := c.ClientIP()
        if !rl.getLimiter(ip).Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
            })
            return
        }
        c.Next()
    }
}
```

```go
// Register — different limits for different endpoint groups
limiter := middleware.NewRateLimiter(rate.Every(time.Second), 10) // 10 req/s burst

r.Use(limiter.Middleware()) // global

authGroup := r.Group("/auth")
authGroup.Use(middleware.NewRateLimiter(rate.Every(time.Minute), 5).Middleware()) // stricter on auth
```

**Use Redis-backed rate limiting** (e.g., `go-redis/redis_rate`) for multi-instance deployments — in-memory limiters are per-pod and ineffective at scale.

---

## 8. Safe Error Responses

Error messages returned to clients must **never** reveal:
- Stack traces
- Internal file paths or line numbers
- Database query details or table names
- Infrastructure details (server names, IPs, versions)
- Secret values or tokens

```go
// ❌ Leaks internal detail
c.JSON(500, gin.H{"error": err.Error()})
// output: "pq: duplicate key value violates unique constraint \"users_email_key\""

// ✅ Map to safe user-facing messages
func userFacingMessage(err error) string {
    switch errors.Cause(err) {
    case domain.ErrNotFound:
        return "resource not found"
    case domain.ErrUnauthorized:
        return "unauthorized"
    case domain.ErrConflict:
        return "resource already exists"
    default:
        return "an unexpected error occurred" // generic — log internally
    }
}

```

---

## 9. Cryptography

### Use Strong Algorithms Only

| Purpose | Use | Never Use |
|---|---|---|
| Password hashing | `bcrypt` (cost ≥ 12) or `argon2id` | MD5, SHA1, plain SHA256 |
| Data signing | HMAC-SHA256 / SHA512, Ed25519 | HMAC-MD5, DSA |
| Encryption | AES-256-GCM | DES, 3DES, RC4, AES-ECB |
| Random tokens | `crypto/rand` | `math/rand` |
| TLS | TLS 1.2 minimum, 1.3 preferred | SSL, TLS 1.0/1.1 |
| Hashing (non-password) | SHA-256 / SHA-512 | MD5, SHA1 |
---

## 10. Dependency Security

Run `govulncheck` on every CI pipeline — it scans your dependency tree against the Go vulnerability database:

Rules:
- Update dependencies regularly — do not let `go.sum` drift for months.
- Review `go mod tidy` output after updating — watch for unexpected new transitive deps.
- Pin major versions explicitly in `go.mod`.
- Do not import packages with no active maintainers for security-sensitive functionality.

---

## 11. HTTP Security Headers

Register a security headers middleware on every server — before all other middleware:

---

## 12. Static Security Analysis — gosec

Run **`gosec`** on every CI pipeline as a mandatory gate:

Prefer running via `golangci-lint` so it is part of the unified lint pipeline:

| Rule | Issue | Fix |
|---|---|---|
| G101 | Hardcoded credentials | Move to env / secrets manager |
| G106 | `ssh.InsecureIgnoreHostKey` | Use proper host key verification |
| G201/G202 | SQL string formatting | GORM parameterized or `sqlc` |
| G304 | File path from user input | Validate with `safeFilePath` helper |
| G401 | Weak hash (MD5/SHA1) | Replace with `crypto/sha256` |
| G402 | TLS `InsecureSkipVerify` | Never disable in production |
| G403 | RSA key < 2048 bits | Use 2048-bit minimum, prefer 4096 |
| G501 | Imports `crypto/md5` | Replace with `crypto/sha256` |
| G601 | Implicit memory aliasing in loop | Capture loop variable |

**Never suppress a gosec finding without a comment explaining why it is a false positive.**

```go
// ✅ Documented suppression
// #nosec G304 — path is validated by safeFilePath before reaching here
f, err := os.Open(validatedPath)
```

---

## 13. Security Response Protocol

When a security issue is discovered:

### Severity Classification

| Severity | Examples | Response Time |
|---|---|---|
| **CRITICAL** | RCE, auth bypass, secret exposure, data breach | Immediate — stop all work |
| **HIGH** | SQL injection, broken authz, SSRF, privilege escalation | Same day |
| **MEDIUM** | XSS, CSRF, info disclosure, weak crypto in use | Within 48 hours |
| **LOW** | Missing headers, verbose errors, minor config issues | Next sprint |

### Response Steps

**For CRITICAL / HIGH:**

1. **STOP** — halt development on affected code immediately.
2. **Assess blast radius** — what data or systems are exposed? Who is affected?
3. **Contain** — disable the vulnerable endpoint or feature if possible.
4. **Fix** — implement the minimal safe fix. Do not refactor while fixing a security issue.
5. **Rotate** — if any secrets were exposed, rotate all of them now.
6. **Test** — write a security regression test that would have caught this issue.
7. **Deploy** — ship the fix via an expedited release.
8. **Review** — search the entire codebase for the same pattern: `grep`, `git log -S`, `gosec`.
9. **Document** — write an internal post-mortem. Include: what happened, root cause, timeline, fix, prevention.

**For MEDIUM / LOW:**

1. Create a tracked issue with the severity label.
2. Fix within the SLA above.
3. Add a regression test.
4. Include in the next regular release.

### Security Regression Tests

Every fixed security vulnerability **must** have a test that proves the fix works and prevents regression:

---

## Quick Reference — Security Rules at a Glance

| Area | Rule |
|---|---|
| Secrets | Never hardcode — env / secrets manager only |
| Secrets | Validate all required secrets at startup — fail fast |
| SQL | GORM or parameterized queries — never concatenate |
| Input | Validate at every system boundary with struct tags |
| File paths | Validate and resolve — prevent path traversal |
| Passwords | `bcrypt` cost ≥ 12 or `argon2id` |
| JWT | Verify algorithm, require `exp`, short TTL |
| Authorization | Enforce in usecase layer — not only handler |
| Rate limiting | Every public endpoint — Redis-backed for multi-instance |
| Error responses | No stack traces, DB details, or internal paths to clients |
| Random values | `crypto/rand` only — never `math/rand` |
| Crypto | AES-256-GCM, HMAC-SHA256+, TLS 1.2+ |
| PII in logs | Never — opaque IDs only |
| Headers | Security headers middleware — first in chain |
| Static analysis | `gosec` via `golangci-lint` — mandatory CI gate |
| Dependencies | `govulncheck ./...` — mandatory CI gate |
| Suppression | `#nosec` requires documented justification comment |
| Response protocol | CRITICAL → stop immediately, fix, rotate, regression test |