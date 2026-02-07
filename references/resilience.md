# Resilience & Fault Tolerance

## Table of Contents

- [Cache + singleflight `[Async]`](#cache--singleflight-async)
  - [Cache Invalidation Strategy `[Async]`](#cache-invalidation-strategy-async)
- [Circuit Breaker + singleflight gRPC Client `[Async]`](#circuit-breaker--singleflight-grpc-client-async)
- [Dispatcher Pattern `[Async]`](#dispatcher-pattern-async)
- [Panic-Safe errgroup `[Async]`](#panic-safe-errgroup-async)
- [Distributed Lock (Redlock) `[Async]`](#distributed-lock-redlock-async)
- [Idempotency](#idempotency)
- [Dead Letter Queue `[Hardening]`](#dead-letter-queue-hardening)
- [gRPC Retry Policy `[Hardening]`](#grpc-retry-policy-hardening)
- [Graceful Shutdown `[Hardening]`](#graceful-shutdown-hardening)

## Cache + singleflight `[Async]`

### Generic CacheLoader

```go
type CacheLoader[T any] struct {
    redis     *redis.Client
    logger    *zap.Logger
    sfg       singleflight.Group
    ttl       time.Duration
    emptyTTL  time.Duration  // Empty-value cache TTL (anti-penetration), recommended: 30s–60s
    sfTimeout time.Duration  // singleflight internal timeout, recommended: 3s
}

// Empty marker distinguishes "key does not exist" from "cache miss"
var emptyMarker = []byte("__EMPTY__")

// loader returns (T, bool, error): bool explicitly indicates "found / not found"
// Avoids using isZero(T) which has edge cases with structs/pointers/slices in generics
func (c *CacheLoader[T]) Load(ctx context.Context, key string,
    loader func(ctx context.Context) (T, bool, error)) (T, bool, error) {
    var zero T

    // 1. Check cache (distinguish Redis error vs key miss)
    cached, err := c.redis.Get(ctx, key).Bytes()
    if err == nil {
        if bytes.Equal(cached, emptyMarker) { return zero, false, nil }  // Hit empty marker → not found
        val, err := unmarshal[T](cached)
        return val, true, err
    }
    if err != nil && !errors.Is(err, redis.Nil) {
        // Redis connection error → log and fallback to singleflight + loader
        // Still use singleflight to prevent thundering herd on DB
        c.logger.Warn("redis get failed, fallback to singleflight + loader", zap.Error(err))
    }

    // 2. Cache miss or Redis error → singleflight merges concurrent requests
    //
    // ⚠️ Context handling: singleflight shares one execution across goroutines.
    // If using outer ctx directly, first caller's cancel kills all waiters.
    // Use context.WithoutCancel to detach from original caller,
    // paired with independent timeout to prevent indefinite blocking.
    sfCtx, sfCancel := context.WithTimeout(context.WithoutCancel(ctx), c.sfTimeout)
    defer sfCancel()

    result, err, _ := c.sfg.Do(key, func() (interface{}, error) {
        data, found, err := loader(sfCtx)
        if err != nil { return nil, err }
        if !found {
            // Not found → write empty marker with short TTL (anti-penetration)
            c.redis.Set(sfCtx, key, emptyMarker, c.emptyTTL)
            return &cacheResult[T]{Value: data, Found: false}, nil
        }
        c.redis.Set(sfCtx, key, marshal(data), c.ttl)
        return &cacheResult[T]{Value: data, Found: true}, nil
    })
    if err != nil { return zero, false, err }
    r := result.(*cacheResult[T])
    return r.Value, r.Found, nil
}

// singleflight needs wrapper to pass found state through interface{}
type cacheResult[T any] struct {
    Value T
    Found bool
}
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `loader` returns `(T, bool, error)` | `bool` explicitly indicates found/not found. Avoids `isZero(T)` edge cases with generics. |
| Empty-value caching | Prevents cache penetration (non-existent keys repeatedly hitting DB) |
| `context.WithoutCancel` | singleflight shares execution. First cancel must not kill other waiters. |
| Independent timeout | Paired with `WithoutCancel` — prevents unbounded blocking |

### When to Introduce Caching

- Read/write ratio > 10:1 AND data tolerates short-term inconsistency → Introduce
- Low read/write ratio OR strong consistency required → Skip

### Cache Invalidation Strategy `[Async]`

Cache without invalidation leads to stale data. Choose the right strategy based on consistency requirements:

| Strategy | How | Use Case |
|----------|-----|----------|
| **TTL-only** | Set TTL on cache entries; no active invalidation | Low-sensitivity data (product catalog, configs). Simplest. |
| **Write-through** | Update cache in same UseCase that writes DB | Single-service owned data. Low write volume. |
| **Event-driven invalidation** | Consumer listens to Domain Events and deletes/updates cache | Cross-service data. Async but eventually consistent. |
| **Cache-aside with version check** | Cache stores entity version; on read, compare with DB version | High-consistency needs without full cache bypass |

#### Write-Through Example (Same Service)

```go
func (uc *UpdateOrderUseCase) Execute(ctx context.Context, req *Request) error {
    err := uc.txManager.WithTx(ctx, func(txCtx context.Context) error {
        order, err := uc.repo.GetByID(txCtx, req.OrderID)
        if err != nil { return err }
        order.UpdateStatus(req.Status, time.Now())
        return uc.repo.Update(txCtx, order)
    })
    if err != nil { return err }

    // Invalidate cache AFTER TX commit (not inside TX)
    // If cache delete fails, TTL will eventually expire — acceptable
    cacheKey := fmt.Sprintf("order:%s", req.OrderID)
    if err := uc.redis.Del(ctx, cacheKey).Err(); err != nil {
        uc.logger.Warn("cache invalidation failed", zap.String("key", cacheKey), zap.Error(err))
    }
    return nil
}
```

**Rule**: Always invalidate (delete) rather than update cache. Delete is idempotent and avoids race conditions where concurrent writes produce stale cache values. Next read triggers CacheLoader to fetch fresh data.

#### Event-Driven Invalidation (Cross-Service)

When Service A's data changes affect Service B's cache:

```go
// Service B consumer: listens to Service A's Domain Events
func (c *ProductCacheInvalidator) Handle(ctx context.Context, eventType string, _ int, payload []byte) error {
    switch eventType {
    case "product.price_changed", "product.updated", "product.deleted":
        var evt struct{ ProductID string `json:"product_id"` }
        if err := json.Unmarshal(payload, &evt); err != nil { return err }
        return c.redis.Del(ctx, "product:"+evt.ProductID).Err()
    }
    return nil
}
```

#### Common Mistakes

1. **Invalidate inside TX**: If TX rolls back, cache is already deleted → unnecessary cache miss (minor). If invalidate before commit and TX succeeds but cache write back races → stale data (serious). Always invalidate AFTER commit.
2. **Update cache instead of delete**: Two concurrent writes can race: Write A (old) updates cache after Write B (new) → stale. Delete is safe — next read fetches fresh.
3. **No TTL as safety net**: Even with active invalidation, always set TTL. If invalidation message is lost (MQ failure, consumer bug), TTL prevents permanent staleness.

## Circuit Breaker + singleflight gRPC Client `[Async]`

```go
type SomeGRPCClient struct {
    client pb.SomeServiceClient
    sfg    singleflight.Group
    cb     *gobreaker.CircuitBreaker
}

func (c *SomeGRPCClient) GetByIDs(ctx context.Context, ids []uuid.UUID) (Result, error) {
    // Sort IDs to ensure same set produces same key
    strIDs := toStrings(ids)
    sort.Strings(strIDs)
    // Use hash to avoid key explosion with many IDs (100 UUIDs ≈ 3600 bytes)
    h := sha256.Sum256([]byte(strings.Join(strIDs, ",")))
    cacheKey := "items:" + hex.EncodeToString(h[:16])

    // WithoutCancel: prevents one caller's cancel from failing all waiters
    // Must pair with independent timeout — otherwise goroutine blocks forever if downstream unresponsive
    detachedCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)
    defer cancel()

    result, err, _ := c.sfg.Do(cacheKey, func() (interface{}, error) {
        cbResult, cbErr := c.cb.Execute(func() (interface{}, error) {
            return c.client.GetByIDs(detachedCtx, &pb.Request{Ids: strIDs})
        })
        if cbErr != nil { return nil, fmt.Errorf("service call: %w", cbErr) }
        return mapResult(cbResult), nil
    })
    if err != nil { return nil, err }
    return result.(Result), nil
}
```

### When to Introduce Circuit Breaker

- Downstream not 100% available (cross-team, external API) → Introduce
- Internal service within same team with SLA guarantees → Can defer

## Dispatcher Pattern `[Async]`

Generic worker pool for parallel batch processing. Use when you need to process a collection of items concurrently with controlled parallelism.

### Generic Dispatcher Implementation

```go
// pkg/dispatcher/dispatcher.go
type Process[Param any, Result any] struct {
    handler     func(context.Context, Param) (Result, error)
    workerCount int
    strategy    func(ch <-chan Param, chans ...chan<- Param)
}

func NewProcess[Param any, Result any]() *Process[Param, Result] {
    return &Process[Param, Result]{
        workerCount: runtime.NumCPU(),
        strategy:    roundRobinStrategy[Param],
    }
}

func (p *Process[Param, Result]) Handler(h func(context.Context, Param) (Result, error)) *Process[Param, Result] {
    p.handler = h
    return p
}

func (p *Process[Param, Result]) WorkerCount(n int) *Process[Param, Result] {
    p.workerCount = n
    return p
}

// Run processes all items and returns collected results (blocking)
func (p *Process[Param, Result]) Run(ctx context.Context, items []Param) ([]Result, error) {
    if len(items) == 0 {
        return nil, nil
    }

    inCh := make(chan Param, len(items))
    for _, item := range items {
        inCh <- item
    }
    close(inCh)

    resultCh := p.Do(ctx, inCh)

    var results []Result
    for r := range resultCh {
        results = append(results, r)
    }
    return results, nil
}

// Do returns a channel of results (non-blocking, stream processing)
func (p *Process[Param, Result]) Do(ctx context.Context, inCh <-chan Param) <-chan Result {
    outCh := make(chan Result)

    var wg sync.WaitGroup
    workerChans := make([]chan Param, p.workerCount)

    for i := 0; i < p.workerCount; i++ {
        workerChans[i] = make(chan Param)
        wg.Add(1)
        go func(ch <-chan Param) {
            defer wg.Done()
            for param := range ch {
                result, err := p.handler(ctx, param)
                if err != nil {
                    continue  // Log error, skip result
                }
                outCh <- result
            }
        }(workerChans[i])
    }

    // Dispatch items to workers
    go func() {
        p.strategy(inCh, toSendOnly(workerChans)...)
        for _, ch := range workerChans {
            close(ch)
        }
    }()

    // Close output when all workers done
    go func() {
        wg.Wait()
        close(outCh)
    }()

    return outCh
}

func roundRobinStrategy[T any](in <-chan T, outs ...chan<- T) {
    i := 0
    for item := range in {
        outs[i%len(outs)] <- item
        i++
    }
}
```

### UseCase Example

```go
func (uc *BatchProcessUseCase) ProcessOrders(ctx context.Context, orderIDs []string) ([]*OrderResult, error) {
    results, err := dispatcher.NewProcess[string, *OrderResult]().
        Handler(func(ctx context.Context, orderID string) (*OrderResult, error) {
            order, err := uc.orderRepo.GetByID(ctx, orderID)
            if err != nil {
                return nil, err
            }
            // Process order...
            return &OrderResult{OrderID: orderID, Status: "processed"}, nil
        }).
        WorkerCount(5).
        Run(ctx, orderIDs)

    return results, err
}
```

### When to Use Dispatcher

| Scenario | Use Dispatcher? | Reason |
|----------|-----------------|--------|
| Batch processing (10+ items) | ✅ Yes | Controlled parallelism, prevent goroutine explosion |
| Fan-out to multiple services | ✅ Yes | Limit concurrent connections |
| Simple 2-3 concurrent calls | ❌ No | Use `errgroup.Group` directly |
| Sequential processing required | ❌ No | No parallelism benefit |

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| Builder pattern API | Fluent, readable configuration |
| Generic `[Param, Result]` | Type-safe, no reflection |
| `Run` vs `Do` | `Run` for batch, `Do` for streaming |
| Configurable worker count | Tune based on downstream capacity |
| Round-robin default | Simple, even distribution |

## Panic-Safe errgroup `[Async]`

Standard `errgroup.Group` does not recover panics — a panic in one goroutine crashes the entire process. Wrap it with panic recovery for production safety.

### Implementation

```go
// pkg/gogroup/gogroup.go
package gogroup

import (
    "fmt"

    "go.uber.org/zap"
    "golang.org/x/sync/errgroup"
)

type Option func(*group)

func WithPanicCallback(callback func(r any)) Option {
    return func(g *group) {
        g.panicCallback = callback
    }
}

type group struct {
    *errgroup.Group
    logger        *zap.Logger
    panicCallback func(r any)
}

func New(logger *zap.Logger, opts ...Option) *group {
    g := &group{Group: new(errgroup.Group), logger: logger}
    for _, opt := range opts {
        opt(g)
    }
    return g
}

func (g *group) Go(f func() error) {
    g.Group.Go(func() (err error) {
        defer func() {
            if r := recover(); r != nil {
                g.logger.Error("goroutine panic",
                    zap.Any("recover", r),
                    zap.Stack("stack"),
                )
                if g.panicCallback != nil {
                    g.panicCallback(r)
                }
                // Return error so Wait() reports the panic to caller
                err = fmt.Errorf("panic recovered: %v", r)
            }
        }()
        return f()
    })
}
```

### Usage

```go
func (uc *UseCase) ProcessConcurrently(ctx context.Context, items []Item) error {
    g := gogroup.New(uc.logger,
        gogroup.WithPanicCallback(func(r any) {
            // Send to Sentry, PagerDuty, etc.
            sentry.CaptureException(fmt.Errorf("panic: %v", r))
        }),
    )

    for _, item := range items {
        item := item  // capture loop variable
        g.Go(func() error {
            return uc.processItem(ctx, item)
        })
    }

    return g.Wait()
}
```

### When to Use

| Scenario | Use gogroup? | Reason |
|----------|--------------|--------|
| Production concurrent ops | ✅ Yes | Prevent single panic from crashing process |
| Development/testing | ❌ Optional | Panics help identify bugs faster |
| Critical background jobs | ✅ Yes | Must not crash the service |

### Relationship with Dispatcher

| Tool | Purpose | Panic Handling |
|------|---------|----------------|
| **Dispatcher** | Batch processing with worker pool | Should include panic recovery internally |
| **gogroup** | Simple concurrent execution (2-5 goroutines) | Wrap errgroup with panic recovery |

Use Dispatcher for batch processing (10+ items). Use gogroup for simple fan-out (2-5 concurrent calls).

## Distributed Lock (Redlock) `[Async]`

Use [redsync](https://github.com/go-redsync/redsync) for distributed locking across multiple service instances. Key feature: **WatchDog** auto-renewal to prevent lock timeout during long operations.

### Infrastructure Layer Implementation

```go
// pkg/redislock/lock.go
package redislock

import (
    "context"
    "time"

    "github.com/go-redsync/redsync/v4"
    "github.com/go-redsync/redsync/v4/redis/goredis/v9"
    "github.com/redis/go-redis/v9"
)

type Option func(*config)

type config struct {
    expiry time.Duration
    tries  int
    delay  time.Duration
}

func WithExpiry(d time.Duration) Option {
    return func(c *config) { c.expiry = d }
}

func WithTries(n int) Option {
    return func(c *config) { c.tries = n }
}

type Locker struct {
    rs *redsync.Redsync
}

func NewLocker(client *redis.Client) *Locker {
    pool := goredis.NewPool(client)
    return &Locker{rs: redsync.New(pool)}
}

func (l *Locker) NewMutex(ctx context.Context, key string, opts ...Option) *Mutex {
    cfg := &config{
        expiry: 1 * time.Second,
        tries:  32,
        delay:  200 * time.Millisecond,
    }
    for _, opt := range opts {
        opt(cfg)
    }

    ctx, cancel := context.WithCancel(ctx)
    return &Mutex{
        ctx:    ctx,
        cancel: cancel,
        mx: l.rs.NewMutex(
            key,
            redsync.WithExpiry(cfg.expiry),
            redsync.WithTries(cfg.tries),
            redsync.WithRetryDelay(cfg.delay),
        ),
        expiry: cfg.expiry,
    }
}

type Mutex struct {
    ctx    context.Context
    cancel context.CancelFunc
    mx     *redsync.Mutex
    expiry time.Duration
}

// Lock acquires the lock and starts WatchDog for auto-renewal
func (m *Mutex) Lock() error {
    if err := m.mx.LockContext(m.ctx); err != nil {
        return err
    }
    go m.watchDog()
    return nil
}

// UntilLock blocks until the lock is acquired
func (m *Mutex) UntilLock() error {
    for {
        select {
        case <-m.ctx.Done():
            return m.ctx.Err()
        default:
        }
        if err := m.Lock(); err != nil {
            if err == redsync.ErrFailed {
                time.Sleep(50 * time.Millisecond)
                continue
            }
            return err
        }
        return nil
    }
}

// Unlock releases the lock and stops WatchDog
func (m *Mutex) Unlock() error {
    defer m.cancel()
    // Use independent context — m.ctx may be cancelled by watchDog if extension failed
    unlockCtx, unlockCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer unlockCancel()
    _, err := m.mx.UnlockContext(unlockCtx)
    return err
}

// watchDog auto-renews lock every expiry/2 to prevent timeout
//
// ⚠️ IMPORTANT: If ExtendContext fails (Redis unavailable), context is cancelled
// but business logic may still be running. The lock will eventually expire,
// allowing other nodes to acquire it. Business logic MUST check ctx.Done()
// periodically and abort if cancelled to prevent concurrent execution.
func (m *Mutex) watchDog() {
    ticker := time.NewTicker(m.expiry / 2)
    defer ticker.Stop()

    for {
        select {
        case <-m.ctx.Done():
            return
        case <-ticker.C:
            if _, err := m.mx.ExtendContext(m.ctx); err != nil {
                // Extension failed — cancel context to signal business logic
                // Business logic should check ctx.Done() and abort gracefully
                m.cancel()
                return
            }
        }
    }
}
```

### Application Port (Interface)

```go
// internal/application/port/output/lock_service.go
package output

import "context"

type LockService interface {
    // WithLock executes fn while holding the lock, auto-unlock on return
    // fn receives enriched context for re-entrancy detection in nested calls
    WithLock(ctx context.Context, key string, fn func(ctx context.Context) error) error

    // WithUserTransaction locks user's financial operations
    WithUserTransaction(ctx context.Context, userID string, fn func(ctx context.Context) error) error
}
```

### Adapter Implementation

```go
// internal/adapter/outbound/external/lock_service_impl.go
package external

import (
    "context"
    "fmt"

    "your-project/pkg/redislock"
)

type lockService struct {
    locker *redislock.Locker
}

func NewLockService(locker *redislock.Locker) output.LockService {
    return &lockService{locker: locker}
}

func (s *lockService) WithLock(ctx context.Context, key string, fn func(ctx context.Context) error) error {
    mx := s.locker.NewMutex(ctx, key)
    if err := mx.Lock(); err != nil {
        return fmt.Errorf("acquire lock: %w", err)
    }
    defer mx.Unlock()

    return fn(ctx)
}

func (s *lockService) WithUserTransaction(ctx context.Context, userID string, fn func(ctx context.Context) error) error {
    return s.WithLock(ctx, fmt.Sprintf("lock:transaction:user:%s", userID), fn)
}
```

### UseCase Usage

```go
func (uc *TransferUseCase) Transfer(ctx context.Context, req TransferRequest) error {
    return uc.lockService.WithUserTransaction(ctx, req.FromUserID, func(ctx context.Context) error {
        // Critical section: deduct from sender
        if err := uc.walletRepo.Deduct(ctx, req.FromUserID, req.Amount); err != nil {
            return err
        }
        // Credit to receiver
        return uc.walletRepo.Credit(ctx, req.ToUserID, req.Amount)
    })
}
```

### Re-entrancy Support (Optional)

If nested lock calls are needed (e.g., UseCase A calls UseCase B, both need the same lock), track acquired locks in context:

```go
// pkg/redislock/reentrant.go
type lockedKeysKey struct{}

func IsLocked(ctx context.Context, key string) bool {
    if keys, ok := ctx.Value(lockedKeysKey{}).(map[string]struct{}); ok {
        _, exists := keys[key]
        return exists
    }
    return false
}

func MarkLocked(ctx context.Context, key string) context.Context {
    // Always create a new map (copy-on-write) to avoid mutating shared map across goroutines
    oldKeys, _ := ctx.Value(lockedKeysKey{}).(map[string]struct{})
    newKeys := make(map[string]struct{}, len(oldKeys)+1)
    for k, v := range oldKeys {
        newKeys[k] = v
    }
    newKeys[key] = struct{}{}
    return context.WithValue(ctx, lockedKeysKey{}, newKeys)
}

// WithLock with re-entrancy check
// fn receives enriched context so nested WithLock calls can detect the held lock
func (s *lockService) WithLock(ctx context.Context, key string, fn func(ctx context.Context) error) error {
    if IsLocked(ctx, key) {
        return fn(ctx)  // Already holding lock, skip acquire
    }

    mx := s.locker.NewMutex(ctx, key)
    if err := mx.Lock(); err != nil {
        return fmt.Errorf("acquire lock: %w", err)
    }
    defer mx.Unlock()

    ctx = MarkLocked(ctx, key)
    return fn(ctx)
}
```

### When to Use Distributed Lock

| Scenario | Use Lock? | Reason |
|----------|-----------|--------|
| Financial transactions (transfer, payment) | ✅ Yes | Prevent double-spend |
| Resource initialization (singleton across instances) | ✅ Yes | Prevent duplicate creation |
| Rate limiting per user | ❌ No | Use Redis counter instead |
| Idempotency check | ❌ No | Use SET NX with TTL |
| Saga step execution | ✅ Consider | Prevent concurrent saga on same entity |
| Cron job deduplication | ❌ Use simpler SetNX | See [scheduled-jobs.md](scheduled-jobs.md#distributed-lock-prevent-duplicate-execution) |

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| **WatchDog auto-renewal** | Long operations won't lose lock due to timeout |
| **redsync library** | Battle-tested Redlock implementation |
| **Application Port interface** | UseCase doesn't depend on Redis directly |
| **`WithLock(fn)` pattern** | Guarantees unlock via defer, prevents forgotten unlock |
| **Re-entrancy via context** | Nested calls to same lock don't deadlock |

### Lock Key Naming Convention

```
lock:<domain>:<resource>:<id>

Examples:
- lock:transaction:user:123
- lock:inventory:sku:ABC-001
- lock:order:create:buyer:456
```

## Idempotency

| Strategy | Use Case | Stage |
|----------|----------|-------|
| **DB `processed_events` table** | Critical business operations (inventory, payment) | MVP |
| **Redis SET NX** | Non-critical state updates, notifications | Async |

### Saga Compensation Idempotency

Saga compensation may be triggered multiple times (original caller + timeout monitor simultaneously). Every compensation operation **MUST be idempotent**:

1. **Check business state**: Downstream compensation API checks if already compensated → return success directly
2. **Idempotency key**: Use `saga_id + step_name` as the key to prevent duplicate compensation execution
3. **Set target state, not delta**: Compensation should "set to target state" not "increment/decrement". Example: "set inventory to X" not "add back N units"

### DB-based Idempotency Example

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```go
func (h *Handler) HandleEvent(ctx context.Context, eventID uuid.UUID, payload []byte) error {
    // Attempt to insert — if already exists, event was already processed
    err := h.processedRepo.Insert(ctx, eventID)
    if err != nil {
        if isUniqueViolation(err) { return nil }  // Already processed, skip
        return err
    }
    // Process the event...
}
```

### Redis-based Idempotency Example `[Async]`

```go
func (h *Handler) HandleEvent(ctx context.Context, eventID string) error {
    ok, err := h.redis.SetNX(ctx, "processed:"+eventID, "1", 24*time.Hour).Result()
    if err != nil { return err }
    if !ok { return nil }  // Already processed
    // Process the event...
}
```

## Dead Letter Queue `[Hardening]`

### RabbitMQ Configuration

```yaml
# Queue arguments
arguments:
  x-dead-letter-exchange: "dlx.exchange"
  x-dead-letter-routing-key: "dlq.{service}.{event}"
  x-delivery-limit: 5    # After 5 requeues → DLQ
```

### Consumer-side Handling

```go
func handleMessage(msg amqp.Delivery) {
    retryCount := getHeaderInt(msg.Headers, "x-delivery-count", 0)
    if retryCount >= 5 {
        msg.Reject(false)  // Send to DLQ, no requeue
        return
    }
    // Normal processing...
}
```

**Stage**: Hardening (must do). In MVP/Async stages, ensure consumers at minimum have error logging; DLQ can wait.

## gRPC Retry Policy `[Hardening]`

```go
retryPolicy := `{
    "methodConfig": [{
        "name": [{"service": "some.v1.SomeService"}],
        "retryPolicy": {
            "maxAttempts": 3,
            "initialBackoff": "0.1s",
            "maxBackoff": "1s",
            "backoffMultiplier": 2,
            "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
        }
    }]
}`
conn, _ := grpc.NewClient(address, grpc.WithDefaultServiceConfig(retryPolicy))
```

**Stage**: Hardening. Only retry idempotent operations. Non-idempotent calls should NOT use automatic retries.

## Graceful Shutdown `[Hardening]`

See [infrastructure.md → Graceful Shutdown](infrastructure.md#graceful-shutdown-hardening).
