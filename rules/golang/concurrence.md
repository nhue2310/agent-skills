## 1. Concurrency & Goroutine Performance

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

