# Performance Optimization — Go

> These rules apply to all Go services and libraries.
---

## Build Troubleshooting

If build fails:
1. Use **build-error-resolver** agent
2. Analyze error messages
3. Fix incrementally
4. Verify after each fix

## The Golden Rule

> **Profile first. Never guess.**

Every optimization must be backed by a measurement. Premature optimization is the most common source of unnecessary complexity in Go codebases. The workflow is always:

```
Measure → Identify bottleneck → Optimize → Measure again → Verify improvement
```

If you cannot show a before/after benchmark or profile diff, the optimization is speculative and should not be merged.

---

## 1. Profiling — Find the Real Bottleneck

### pprof Setup

Expose `pprof` in non-production environments only. Never expose it publicly in production.

### Benchmarks

Write benchmarks for every hot path before optimizing. A benchmark without a baseline measurement proves nothing.

```go
// user_service_bench_test.go
func BenchmarkUserService_GetByID(b *testing.B) {
    svc := setupBenchService(b)
    ctx := context.Background()

    b.ResetTimer()            // exclude setup time
    b.ReportAllocs()          // show allocation count and bytes

    for i := 0; i < b.N; i++ {
        _, _ = svc.GetByID(ctx, "abc123")
    }
}

// Run with:
// go test -bench=BenchmarkUserService_GetByID -benchmem -count=5 ./...
// -benchmem: show allocs/op and bytes/op
// -count=5:  run 5 times to reduce variance
```

```bash
# Compare before and after using benchstat
go install golang.org/x/perf/cmd/benchstat@latest

go test -bench=. -benchmem -count=5 ./... > before.txt
# make changes
go test -bench=. -benchmem -count=5 ./... > after.txt
benchstat before.txt after.txt
```

### Profiling Checklist

- [ ] Profile in production-like load conditions — not on your laptop with zero traffic
- [ ] Use `benchstat` to confirm improvements are statistically significant
- [ ] Profile CPU *and* heap — a faster function that allocates more may hurt overall throughput
- [ ] Check goroutine and mutex profiles for concurrency bottlenecks
- [ ] Re-profile after changes — optimizations can shift the bottleneck elsewhere

---

## 2. Memory & Allocation Optimization

### Slice and Map Preallocation

```go
// ❌ Grows repeatedly — O(n) reallocations
var results []User
for _, id := range ids {
    u, _ := repo.FindByID(ctx, id)
    results = append(results, u)
}

// ✅ Preallocate — zero reallocations
results := make([]User, 0, len(ids))
for _, id := range ids {
    u, _ := repo.FindByID(ctx, id)
    results = append(results, u)
}

// ✅ Preallocate map
counts := make(map[string]int, len(items))
for _, item := range items {
    counts[item.Key]++
}
```

### Buffer Reuse with sync.Pool

`sync.Pool` eliminates heap allocations on hot paths by reusing objects. Use for: JSON encoder buffers, byte slices, request-scoped scratch structs.

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func encode(v any) ([]byte, error) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return nil, errors.Wrap(err, "json encode")
    }
    // Copy before returning — buf goes back to pool
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result, nil
}
```

**`sync.Pool` rules:**
- Objects in the pool may be evicted by the GC at any time — never assume they persist.
- Always call `Reset()` or zero the object before putting it back.
- Do not store pointers to pooled objects outside the function scope.
- Do not pool objects that hold open resources (connections, file handles).

### String Building

```go
// ❌ Quadratic — allocates new string on each +
var result string
for _, s := range parts {
    result += s
}

// ✅ strings.Builder — O(n) single allocation
var sb strings.Builder
sb.Grow(estimatedLen) // optional hint to avoid growth
for _, s := range parts {
    sb.WriteString(s)
}
result := sb.String()
```

### Struct and Pointer Decisions

```go
// Pass by value for small structs (≤ ~64 bytes) — avoids heap escape
func process(u User) error { ... }         // ✅ if User is small

// Pass by pointer for large structs — avoids copy cost
func process(u *LargeReport) error { ... } // ✅ if LargeReport > 64 bytes

// ❌ Pointer in large slice — each element requires GC scanning
type Cache struct {
    items []*Item // GC must scan every pointer
}

// ✅ Value slice — single GC scan for the whole slice
type Cache struct {
    items []Item  // no pointer chasing
}
```

### Escape Analysis

Use the escape analysis flag to see what is being heap-allocated:

```bash
go build -gcflags="-m=2" ./... 2>&1 | grep "escapes to heap"
```

Key causes of unexpected heap escapes:
- Storing a pointer in an interface (`any`)
- Returning a pointer to a local variable
- Passing a local slice/map to a goroutine
- Closures capturing outer variables

```go
// ❌ Escapes — stored in interface{}
func process(v any) { ... }
process(myStruct) // myStruct escapes to heap

// ✅ Concrete type — stays on stack
func process(v MyStruct) { ... }
process(myStruct)
```

---

## 7. Database Performance

### Connection Pool Tuning

Configure GORM's underlying `*sql.DB` at startup. Wrong pool settings are a common cause of connection exhaustion and latency spikes under load.

```go
sqlDB, err := db.DB()
if err != nil {
    return errors.Wrap(err, "db.DB()")
}

// Tune based on load testing, not guesswork
sqlDB.SetMaxOpenConns(25)             // max simultaneous connections to DB
sqlDB.SetMaxIdleConns(10)             // keep this many connections warm
sqlDB.SetConnMaxLifetime(5 * time.Minute)  // recycle connections to avoid stale state
sqlDB.SetConnMaxIdleTime(2 * time.Minute)  // close idle connections after this duration
```

Guidelines:
- `MaxOpenConns` × (number of service instances) should not exceed the DB's `max_connections`.
- Start with `MaxOpenConns=25` and tune with load testing.
- `MaxIdleConns` should be ≤ `MaxOpenConns`. A ratio of 40% is a reasonable starting point.

### N+1 Query Prevention

```go
// ❌ N+1 — 1 query for orders + N queries for users
orders, _ := repo.FindAllOrders(ctx)
for _, o := range orders {
    o.User, _ = repo.FindUserByID(ctx, o.UserID) // query per order
}

// ✅ Preload in GORM — 2 queries total
db.WithContext(ctx).Preload("User").Find(&orders)

// ✅ JOIN — 1 query
db.WithContext(ctx).
    Joins("JOIN users ON users.id = orders.user_id").
    Select("orders.*, users.name as user_name").
    Find(&orders)

// ✅ Batch fetch — 2 queries for any N
db.WithContext(ctx).Find(&orders)
userIDs := extractUserIDs(orders)
db.WithContext(ctx).Where("id IN ?", userIDs).Find(&users)
userMap := indexByID(users)
for i := range orders { orders[i].User = userMap[orders[i].UserID] }
```

### Query Analysis

```sql
-- Run EXPLAIN ANALYZE on any query touching > 10k rows
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = $1 AND status = 'pending';

-- Look for: Seq Scan (missing index), high rows estimate, high actual rows
-- Fix: add index on (user_id, status) for this query
CREATE INDEX idx_orders_user_status ON orders (user_id, status)
    WHERE status = 'pending'; -- partial index — even cheaper
```

Index guidelines:
- Index every foreign key column.
- Index columns in `WHERE`, `ORDER BY`, and `JOIN ON` clauses that appear in slow queries.
- Composite index column order matters — most selective column first.
- Use `EXPLAIN ANALYZE` output, not intuition, to decide on indexes.

---

## 8. Caching Strategy

Cache at the **repository or usecase layer** — never in handlers.

```go
// repository/cached_user_repo.go

type cachedUserRepo struct {
    db    UserRepository
    cache *ristretto.Cache // or freecache, redis
    ttl   time.Duration
}

func (r *cachedUserRepo) FindByID(ctx context.Context, id string) (*domain.User, error) {
    // Check cache first
    if val, ok := r.cache.Get(id); ok {
        return val.(*domain.User), nil
    }

    // Miss — fetch from DB
    user, err := r.db.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // Store with TTL
    r.cache.SetWithTTL(id, user, 1, r.ttl)
    return user, nil
}

func (r *cachedUserRepo) Save(ctx context.Context, user *domain.User) error {
    if err := r.db.Save(ctx, user); err != nil {
        return err
    }
    // Invalidate on write
    r.cache.Del(user.ID)
    return nil
}
```

Cache rules:
- **Always set a TTL.** Never cache indefinitely.
- **Invalidate on write** — not on a timer. Stale reads after writes erode trust.
- **Document the invalidation strategy** next to the cache logic — in a comment.
- **Use a bounded cache** (`ristretto`, `freecache`) for in-process caching — never a plain `map` without eviction.
- Use Redis for shared cache across multiple service instances.
- Cache keys must be deterministic and unique: `"user:v1:" + id`.

---

## 9. HTTP Performance

### Reuse http.Client

```go
// ❌ Creates a new client per request — no connection reuse
func callAPI(ctx context.Context, url string) error {
    client := &http.Client{Timeout: 5 * time.Second}
    // ...
}

// ✅ Shared client — connection pooling via Transport
var httpClient = &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
        DisableCompression:  false,
    },
}
```

### JSON Encoding

```go
// ❌ json.Marshal — allocates a new []byte every time
data, err := json.Marshal(v)

// ✅ Encode directly to response writer — zero intermediate allocation
json.NewEncoder(c.Writer).Encode(v)

// ✅ For repeated encoding of same type — use sync.Pool (see Section 2)
```

### Response Compression

Enable gzip compression for text responses > 1KB:

```go
import "github.com/gin-contrib/gzip"

r.Use(gzip.Gzip(gzip.DefaultCompression))
```

---

## 11. Build & Compile Optimization

### Build Cache

Go's build cache is automatic but can be managed:

```bash
# View cache size
go env GOCACHE

# Clean if disk space is tight
go clean -cache

# Build with race detector (test only — ~2-20x slower runtime)
go build -race ./...
```

### Binary Size Reduction

```bash
# Strip debug info for production binaries — reduces size 20-30%
go build -ldflags="-s -w" -o service ./cmd/server

# -s: omit symbol table
# -w: omit DWARF debug info
```

### Trimpath for Reproducible Builds

```bash
# Remove local file paths from binary — improves reproducibility and security
go build -trimpath -ldflags="-s -w" -o service ./cmd/server
```

### CGO

```bash
# Disable CGO for fully static binaries (e.g., scratch Docker images)
CGO_ENABLED=0 go build -o service ./cmd/server
```

---

## 12. Build Troubleshooting

When a build or test fails unexpectedly:

1. **Read the full error output** — do not skim. Go errors are precise; the exact message is always meaningful.
2. **Fix incrementally** — fix one error at a time and rebuild. Cascading errors often disappear once the root cause is fixed.
3. **Check module state** — `go mod tidy` to sync `go.sum`, `go mod verify` to check integrity.
4. **Clear build cache** — `go clean -cache` if you suspect stale artifacts.
5. **Check Go version** — `go version`. Syntax or API differences between Go versions cause confusing errors.
6. **Isolate the failure** — `go build ./path/to/package` to narrow scope.
7. **Verify after each fix** — confirm the build passes before moving to the next issue.

```bash
# Common build debug commands
go mod tidy           # fix missing or unused dependencies
go mod verify         # verify dependency checksums
go clean -cache       # clear build cache
go vet ./...          # catch common mistakes the compiler misses
go build ./...        # build all packages without running
```

---

## Quick Reference — Performance Rules at a Glance

| Area | Rule |
|---|---|
| **Golden rule** | Profile first — never optimize without measurement |
| **Profiling** | `pprof` CPU + heap + goroutine before any optimization |
| **Benchmarks** | `go test -bench=. -benchmem -count=5`; use `benchstat` for comparison |
| **Slices** | Preallocate with `make([]T, 0, n)` when size is known |
| **Buffers** | `sync.Pool` for hot-path buffers; always `Reset()` before reuse |
| **Strings** | `strings.Builder` + `Grow` hint in loops |
| **Structs** | Value for ≤64 bytes; pointer for larger; `[]struct` over `[]*struct` |
| **Goroutines** | Bounded with `errgroup` + semaphore; every one must have a stop path |
| **Goroutine leaks** | `goleak.VerifyTestMain` in every test suite |
| **Memory leaks** | Close bodies, cancel contexts, stop tickers, copy sub-slices |
| **GC** | Set `GOMEMLIMIT=80% container limit`; tune `GOGC` based on workload |
| **Pointer density** | Prefer `[]struct` over `[]*struct`; avoid `any` in hot paths |
| **DB pool** | `SetMaxOpenConns(25)`, `SetMaxIdleConns(10)`, `SetConnMaxLifetime(5m)` |
| **N+1 queries** | `Preload`, JOINs, or batch fetch; never query inside a loop |
| **Query analysis** | `EXPLAIN ANALYZE` for any query on > 10k rows |
| **Cache** | Repository/usecase layer only; always TTL; invalidate on write |
| **http.Client** | Shared singleton with tuned `Transport`; never create per-request |
| **JSON** | Encode directly to `io.Writer`; pool encoder buffers for high throughput |
| **Build** | `-ldflags="-s -w" -trimpath` for production; `CGO_ENABLED=0` for static |

---

## Performance Investigation Checklist

When a service is slow or using unexpectedly high memory:

- [ ] Capture CPU profile under real load — identify top functions by cumulative time
- [ ] Capture heap profile — identify top allocators by `alloc_objects`
- [ ] Check goroutine profile — look for count > expected; sign of a leak
- [ ] Check mutex profile — look for high contention on hot locks
- [ ] Run `go test -bench=. -benchmem` on suspected hot paths
- [ ] Use `benchstat` to confirm improvements are statistically significant
- [ ] Check DB slow query log — identify queries without indexes
- [ ] Run `EXPLAIN ANALYZE` on top slow queries
- [ ] Verify connection pool settings are appropriate for load
- [ ] Confirm `GOMEMLIMIT` is set in production containers
- [ ] Check for N+1 patterns in hot code paths
- [ ] Confirm `http.Client` is shared, not created per request
- [ ] Check for unbounded goroutine growth via metrics or goroutine profile
- [ ] Run `goleak` tests to catch goroutine leaks in test suite