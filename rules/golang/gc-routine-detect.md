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
