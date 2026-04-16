## 1. Authentication & Authorization

### Password Storage

```go
import "golang.org/x/crypto/bcrypt"

// ✅ Hash with bcrypt — cost ≥ 12
func HashPassword(plain string) (string, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(plain), 12)
    if err != nil {
        return "", errors.Wrap(err, "bcrypt.GenerateFromPassword")
    }
    return string(hash), nil
}

func CheckPassword(plain, hash string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(plain)) == nil
}
```

### JWT — Validation Rules

```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID string `json:"sub"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func ValidateToken(tokenStr, secret string) (*Claims, error) {
    claims := &Claims{}

    token, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (any, error) {
        // ✅ Verify signing method explicitly — prevent algorithm switching attacks
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return []byte(secret), nil
    },
        jwt.WithExpirationRequired(),       // reject tokens with no expiry
        jwt.WithIssuedAt(),                 // validate iat claim
    )
    if err != nil || !token.Valid {
        return nil, errors.Wrap(err, "invalid token")
    }
    return claims, nil
}
```

**JWT rules:**
- Always verify the signing algorithm — never accept `alg: none`.
- Set a short expiry (`exp`) — typically 15 minutes for access tokens.
- Use refresh tokens with longer TTL stored server-side (revocable).
- Store JWTs in `httpOnly`, `Secure`, `SameSite=Strict` cookies — not `localStorage`.
- JWT secret minimum 32 bytes, loaded from env only.

### Authorization — Enforce in Usecase Layer

```go
// ❌ Auth only at handler — usecase is unprotected if called from other places
func (h *UserHandler) DeleteUser(c *gin.Context) {
    if !isAdmin(c) { c.AbortWithStatus(403); return }
    h.usecase.Delete(c.Request.Context(), c.Param("id")) // no authz check here
}

// ✅ Authz enforced inside usecase — safe regardless of caller
func (s *UserService) Delete(ctx context.Context, requesterID, targetID string) error {
    requester, err := s.repo.FindByID(ctx, requesterID)
    if err != nil { return errors.Wrap(err, "fetch requester") }

    if requester.Role != domain.RoleAdmin && requesterID != targetID {
        return domain.ErrUnauthorized
    }
    return s.repo.Delete(ctx, targetID)
}
```

---
