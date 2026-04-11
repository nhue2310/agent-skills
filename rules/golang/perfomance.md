# Performance Optimization — Go

> These rules apply to all Go services and libraries.
>
> **Related documents:** [`coding-style.md`](./coding-style.md) · [`testing.md`](./testing.md) · [`security.md`](./security.md)

---



## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.6** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## Extended Thinking + Plan Mode

Extended thinking is enabled by default, reserving up to 31,999 tokens for internal reasoning.

Control extended thinking via:
- **Toggle**: Option+T (macOS) / Alt+T (Windows/Linux)
- **Config**: Set `alwaysThinkingEnabled` in `~/.claude/settings.json`
- **Budget cap**: `export MAX_THINKING_TOKENS=10000`
- **Verbose mode**: Ctrl+O to see thinking output

For complex tasks requiring deep reasoning:
1. Ensure extended thinking is enabled (on by default)
2. Enable **Plan Mode** for structured approach
3. Use multiple critique rounds for thorough analysis
4. Use split role sub-agents for diverse perspectives

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

```go
// cmd/server/main.go — guarded by build tag or env flag
import _ "net/http/pprof"

if cfg.EnableProfiling {
    go func() {
        log.Println("pprof listening on :6060")
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}
```

### Profiling Workflows

```bash
# CPU profile — what is the code spending time on?
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile — what is allocating memory?
go tool pprof http://localhost:6060/debug/pprof/heap

# Allocation count — how many objects are being allocated?
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap

# Goroutine profile — are goroutines leaking?
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Mutex contention — where are goroutines blocking on locks?
go tool pprof http://localhost:6060/debug/pprof/mutex

# Block profile — where are goroutines blocked on channel ops?
go tool pprof http://localhost:6060/debug/pprof/block

# Flame graph — visual CPU profile (most useful for hotspot discovery)
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30
```

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

## 3. Concurrency & Goroutine Performance

### Worker Pool Pattern

For bounded fan-out over large inputs — never launch unbounded goroutines:

```go
func processAll(ctx context.Context, items []Item, concurrency int) error {
    g, ctx := errgroup.WithContext(ctx)
    sem := make(chan struct{}, concurrency)

    for _, item := range items {
        item := item // capture loop variable
        sem <- struct{}{}

        g.Go(func() error {
            defer func() { <-sem }()
            return processOne(ctx, item)
        })
    }
    return g.Wait()
}

// Usage
if err := processAll(ctx, items, runtime.NumCPU()); err != nil {
    return errors.Wrap(err, "processAll")
}
```

### Channel Sizing

```go
// Unbuffered — synchronous handoff, use when sender must wait for receiver
ch := make(chan Result)

// Buffered — decouple producer and consumer; size to expected burst
ch := make(chan Result, 100)

// ❌ Wrong — channel sized to 1 is almost always a bug
// it gives a false sense of buffering and causes subtle blocking
ch := make(chan Result, 1)
```

### Mutex vs RWMutex

```go
// sync.Mutex — for write-heavy or mixed workloads
type Counter struct {
    mu    sync.Mutex
    value int
}
func (c *Counter) Inc() { c.mu.Lock(); defer c.mu.Unlock(); c.value++ }

// sync.RWMutex — for read-heavy workloads (many reads, few writes)
type Registry struct {
    mu    sync.RWMutex
    items map[string]Item
}
func (r *Registry) Get(key string) (Item, bool) {
    r.mu.RLock()  // multiple readers can hold RLock simultaneously
    defer r.mu.RUnlock()
    v, ok := r.items[key]
    return v, ok
}
func (r *Registry) Set(key string, val Item) {
    r.mu.Lock() // exclusive write lock
    defer r.mu.Unlock()
    r.items[key] = val
}
```

### sync.Once and sync/atomic for Hot Paths

```go
// sync.Once for one-time expensive initialisation
var (
    instance *Service
    once     sync.Once
)
func GetService() *Service {
    once.Do(func() { instance = newExpensiveService() })
    return instance
}

// sync/atomic for simple counters — no mutex overhead
type Metrics struct {
    requests  atomic.Int64
    errors    atomic.Int64
}
func (m *Metrics) RecordRequest()  { m.requests.Add(1) }
func (m *Metrics) RecordError()    { m.errors.Add(1) }
func (m *Metrics) RequestCount() int64 { return m.requests.Load() }
```

---

## 4. Goroutine Leaks (CRITICAL)

A goroutine leak occurs when a goroutine is started but never exits. It holds memory, may hold locks, and degrades performance over time — a common source of production memory growth.

### Common Causes and Fixes

```go
// ❌ Leak — goroutine blocks on send if caller returns early
func process(jobs []Job) {
    ch := make(chan Result)
    for _, j := range jobs {
        go func(j Job) { ch <- doWork(j) }(j) // stuck if nobody reads ch
    }
}

// ✅ errgroup — goroutines exit when context is cancelled or work is done
func process(ctx context.Context, jobs []Job) error {
    g, ctx := errgroup.WithContext(ctx)
    for _, j := range jobs {
        j := j
        g.Go(func() error {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
                return doWork(ctx, j)
            }
        })
    }
    return g.Wait()
}

// ❌ Leak — background goroutine with no stop signal
func startWorker() {
    go func() {
        for { doWork() } // runs forever
    }()
}

// ✅ Always provide a stop mechanism
func startWorker(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
                doWork()
            }
        }
    }()
}
```

### Detect Leaks in Tests

```go
// main_test.go — goleak fails the test suite if any goroutines survive
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Per-test leak check
func TestSomeFunc(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ... test body
}
```

---

## 5. Memory Leaks (CRITICAL)

| Source | Symptom | Fix |
|---|---|---|
| Unclosed HTTP response bodies | Heap growth, exhausted connections | `defer resp.Body.Close()` immediately after nil check |
| Forgotten `context.CancelFunc` | Goroutine and timer accumulation | Always `defer cancel()` after `WithCancel`/`WithTimeout` |
| Unbounded in-memory caches | Monotonic heap growth | TTL-based cache (`ristretto`, `freecache`) |
| Sub-slice retaining backing array | Large arrays never freed | Copy sub-slice: `append([]T{}, big[a:b]...)` |
| Timers/tickers never stopped | Goroutine accumulation | `defer t.Stop()` after every `time.NewTimer`/`time.NewTicker` |
| `context.Value` holding large objects | Unexpected long-lived references | Keep context values small; pass large data as explicit params |
| Goroutines blocking on full channel | Memory and goroutine accumulation | Use buffered channel or `select` with `ctx.Done()` |
| Global maps growing without bound | Monotonic memory growth | Use `sync.Map` with expiry, or a bounded LRU |

```go
// ❌ Every common memory leak pattern

// 1. Unclosed body
resp, _ := http.Get(url)
data, _ := io.ReadAll(resp.Body) // Body never closed

// 2. Forgotten cancel
ctx, _ := context.WithTimeout(parent, 5*time.Second) // cancel leaked

// 3. Sub-slice retaining huge backing array
all := loadGigabyteSlice()
first100 := all[:100] // all stays alive as long as first100 is alive

// 4. Ticker not stopped
ticker := time.NewTicker(time.Second)
go func() { for range ticker.C { work() } }()
// ticker goroutine runs forever

// ✅ All fixed

// 1.
resp, err := http.Get(url)
if err != nil { return errors.Wrap(err, "http.Get") }
defer resp.Body.Close()

// 2.
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// 3.
first100 := make([]Item, 100)
copy(first100, all[:100])

// 4.
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
go func() {
    for {
        select {
        case <-ticker.C: work()
        case <-ctx.Done(): return
        }
    }
}()
```

---

## 6. GC Optimization

Go's GC is a concurrent tri-color mark-and-sweep. Performance is determined by two factors: **heap size** (triggers GC frequency) and **pointer density** (determines GC scan cost).

### Tuning Knobs

```bash
# GOGC=100 (default) — GC runs when live heap doubles since last collection
# Lower GOGC → more frequent GC → lower peak memory, more CPU overhead
# Higher GOGC → less frequent GC → higher throughput, higher peak memory

GOGC=50   # containerized services where memory is the constraint
GOGC=200  # batch jobs or services where throughput matters more than memory

# GOMEMLIMIT (Go 1.19+) — hard ceiling on total Go memory usage
# Prevents OOM kills in containers. Set to ~80% of container memory limit.
GOMEMLIMIT=400MiB  # for a 512MiB container
```

```go
// Programmatic — use only when you cannot set environment variables
import "runtime/debug"

func init() {
    debug.SetMemoryLimit(400 * 1024 * 1024) // 400 MiB
    // GOGC is best set via env; avoid SetGCPercent in production
}
```

**Recommendation:** Always set `GOMEMLIMIT` in production containers. It is the single highest-impact GC tuning lever — it prevents the GC from being too conservative and triggering OOM kills.

### Reduce Pointer Density

Every pointer in a heap-allocated object must be scanned by the GC. Reducing pointers reduces scan time.

```go
// ❌ High pointer density — GC must scan every element
type Cache struct {
    entries map[string]*Entry  // map of pointers — all scanned
}

// ✅ Lower density — values stored inline, no per-element pointer scan
type Cache struct {
    entries map[string]Entry   // map of values — only map header scanned
}

// ❌ Slice of pointers — N pointers to scan
items := make([]*Item, n)

// ✅ Slice of values — single scan for the backing array
items := make([]Item, n)
```

### Avoid Interface Boxing in Hot Paths

Storing a concrete value in an interface (`any`, `error`) causes a heap allocation — the value is "boxed":

```go
// ❌ Boxes every integer — heap allocation per call in a hot loop
var pool []any
pool = append(pool, myInt) // myInt escapes to heap

// ✅ Typed slice — no boxing
var pool []int
pool = append(pool, myInt)
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

## 10. Context Window Management (AI-Assisted Development)

When working with AI coding assistants (Claude, Copilot, etc.), context window efficiency directly affects output quality.

### What to Avoid at High Context Usage (Last 20%)

Avoid starting these tasks when the context window is near capacity — the AI loses access to earlier instructions and code:

- Large-scale refactoring spanning multiple files
- Feature implementation that touches many layers
- Debugging complex multi-service interactions
- Architecture decisions requiring broad context

### Lower-Context Tasks (Safe at Any Point)

These tasks are self-contained and context-efficient:

- Single-file edits and bug fixes
- Independent utility or helper functions
- Documentation updates
- Schema or migration files
- Writing tests for a specific function

### Extended Thinking for Complex Problems

For architectural decisions, algorithm design, or deep debugging, enable extended thinking for more thorough analysis:

```bash
# Claude Code — toggle extended thinking
Option+T (macOS) / Alt+T (Windows/Linux)

# Set budget cap to avoid excessive token usage on routine tasks
export MAX_THINKING_TOKENS=10000

# View thinking output
Ctrl+O (verbose mode)
```

Use **Plan Mode** for multi-step implementations — it forces a structured approach before writing any code, reducing expensive mid-implementation corrections.

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