# Async Patterns

Covers MQ consumer patterns, Saga orchestration, and Outbox pattern for reliable async processing.

## Table of Contents

- [MQ Consumer Patterns `[Async]`](#mq-consumer-patterns-async)
  - [Consumer Architecture](#consumer-architecture)
  - [Message Processing Pattern](#message-processing-pattern)
  - [Ack/Nack Decision Table](#acknack-decision-table)
  - [Transient vs Permanent Error](#transient-vs-permanent-error)
  - [Event Version Handling in Consumer](#event-version-handling-in-consumer)
  - [Consumer Graceful Shutdown](#consumer-graceful-shutdown)
  - [Connection Manager (Auto-Reconnect)](#connection-manager-auto-reconnect)
- [Synchronous Saga (MVP Stage)](#synchronous-saga-mvp-stage)
  - [Saga Execution Record Schema](#saga-execution-record-schema)
- [Stale Record Scanner](#stale-record-scanner)
- [Asynchronous Saga + Outbox `[Async]`](#asynchronous-saga--outbox-async)
- [Two-Phase Outbox Poller `[Async]`](#two-phase-outbox-poller-async)
- [TxManager `[Async]`](#txmanager-async)
- [Saga Timeout Monitor `[Async]`](#saga-timeout-monitor-async)
- [Repository Interface Segregation](#repository-interface-segregation)
- [CQRS Read Model Projection `[Async]`](#cqrs-read-model-projection-async)

## MQ Consumer Patterns `[Async]`

### Consumer Architecture

```go
// internal/adapter/inbound/consumer/order_event_consumer.go
type OrderEventConsumer struct {
    conn       *rabbitmq.ConnectionManager
    useCase    input.HandleOrderEventUseCase
    logger     *zap.Logger
    prefetch   int           // Recommended: 10–50 (tune based on processing time)
    maxRetries int           // Recommended: 5 (matches DLQ x-delivery-limit)
}

func (c *OrderEventConsumer) Start(ctx context.Context) error {
    ch, err := c.conn.Channel()
    if err != nil { return fmt.Errorf("open channel: %w", err) }

    // Prefetch: controls how many unacked messages the broker pushes to this consumer
    // Too high → memory pressure + unfair distribution across consumers
    // Too low → underutilization (idle between acks)
    if err := ch.Qos(c.prefetch, 0, false); err != nil {
        return fmt.Errorf("set qos: %w", err)
    }

    msgs, err := ch.ConsumeWithContext(ctx, "order.events.queue", "", false, false, false, false, nil)
    if err != nil { return fmt.Errorf("consume: %w", err) }

    // Concurrent consumer workers
    workerCount := 5
    for i := 0; i < workerCount; i++ {
        go c.worker(ctx, msgs)
    }
    return nil
}
```

### Message Processing Pattern

```go
func (c *OrderEventConsumer) worker(ctx context.Context, msgs <-chan amqp.Delivery) {
    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-msgs:
            if !ok { return }  // Channel closed
            c.handleMessage(ctx, msg)
        }
    }
}

// getHeaderInt safely extracts an int from AMQP headers with a default value.
func getHeaderInt(headers amqp.Table, key string, defaultVal int) int {
    v, ok := headers[key]
    if !ok {
        return defaultVal
    }
    switch n := v.(type) {
    case int64:
        return int(n)
    case int32:
        return int(n)
    default:
        return defaultVal
    }
}

func (c *OrderEventConsumer) handleMessage(ctx context.Context, msg amqp.Delivery) {
    // 1. Extract trace context from AMQP headers (see MQ Trace Context Propagation)
    ctx = rabbitmq.ExtractTraceContext(ctx, msg.Headers)
    ctx, span := otel.Tracer("consumer").Start(ctx, "consume."+msg.RoutingKey)
    defer span.End()

    // 2. Extract event metadata
    eventVersion := getHeaderInt(msg.Headers, "event_version", 1)

    // 3. Check retry count (consumer-side DLQ guard)
    retryCount := getHeaderInt(msg.Headers, "x-delivery-count", 0)
    if retryCount >= c.maxRetries {
        c.logger.Error("max retries exceeded, rejecting to DLQ",
            zap.String("routing_key", msg.RoutingKey),
            zap.Int("retry_count", retryCount))
        span.SetStatus(otelcodes.Error, "max retries exceeded")
        _ = msg.Reject(false)  // false = don't requeue → goes to DLQ
        return
    }

    // 4. Process with timeout
    processCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    err := c.useCase.Handle(processCtx, msg.RoutingKey, eventVersion, msg.Body)

    // 5. Ack/Nack strategy
    if err != nil {
        span.RecordError(err)
        span.SetStatus(otelcodes.Error, err.Error())
        c.logger.Error("failed to process message",
            zap.String("routing_key", msg.RoutingKey),
            zap.Int("retry_count", retryCount), zap.Error(err))

        if isTransient(err) {
            _ = msg.Nack(false, true)   // true = requeue for retry
        } else {
            _ = msg.Reject(false)       // Permanent failure → DLQ
        }
        return
    }

    _ = msg.Ack(false)
}
```

### Ack/Nack Decision Table

| Scenario | Action | Rationale |
|----------|--------|-----------|
| Processing succeeded | `Ack(false)` | Done, remove from queue |
| Transient error (timeout, connection) | `Nack(false, true)` | Requeue for retry |
| Permanent error (invalid payload, business rule) | `Reject(false)` | Send to DLQ, don't retry |
| Max retries exceeded | `Reject(false)` | Send to DLQ |
| Context cancelled (shutdown) | `Nack(false, true)` | Requeue — another consumer will pick it up |

### Transient vs Permanent Error

```go
func isTransient(err error) bool {
    // gRPC downstream errors
    if s, ok := status.FromError(err); ok {
        switch s.Code() {
        case codes.Unavailable, codes.DeadlineExceeded, codes.ResourceExhausted:
            return true
        }
    }
    // Database connection errors
    if errors.Is(err, context.DeadlineExceeded) { return true }
    if errors.Is(err, context.Canceled) { return true }
    // Network errors
    var netErr net.Error
    if errors.As(err, &netErr) { return true }
    return false
}
```

### Event Version Handling in Consumer

```go
func (uc *HandleOrderEventUseCase) Handle(ctx context.Context, eventType string, version int, payload []byte) error {
    switch eventType {
    case "order.status_changed":
        switch version {
        case 1:
            var evt OrderStatusChangedV1
            if err := json.Unmarshal(payload, &evt); err != nil { return fmt.Errorf("unmarshal v1: %w", err) }
            return uc.handleStatusChangedV1(ctx, evt)
        case 2:
            var evt OrderStatusChangedV2  // Added new fields
            if err := json.Unmarshal(payload, &evt); err != nil { return fmt.Errorf("unmarshal v2: %w", err) }
            return uc.handleStatusChangedV2(ctx, evt)
        default:
            // Unknown future version → log warning, attempt best-effort with latest known schema
            uc.logger.Warn("unknown event version, attempting best-effort",
                zap.String("event_type", eventType), zap.Int("version", version))
            var evt OrderStatusChangedV2
            if err := json.Unmarshal(payload, &evt); err != nil { return fmt.Errorf("unmarshal: %w", err) }
            return uc.handleStatusChangedV2(ctx, evt)
        }
    default:
        uc.logger.Warn("unknown event type", zap.String("event_type", eventType))
        return nil  // Don't reject unknown types — publisher may be ahead of consumer
    }
}
```

### Consumer Graceful Shutdown

Consumer shutdown must be coordinated with the overall shutdown sequence (see [Graceful Shutdown](infrastructure.md#graceful-shutdown-hardening)):

```go
func (c *OrderEventConsumer) Stop() error {
    // 1. Cancel consumer context → workers stop pulling new messages
    c.cancel()
    // 2. Wait for in-flight messages to complete (with timeout)
    c.wg.Wait()
    // 3. Close channel (remaining unacked messages go back to queue)
    return c.channel.Close()
}
```

**Shutdown order**: Stop accepting new gRPC requests → Stop consumers (drain in-flight) → Stop Outbox Poller → Close MQ connections → Close DB.

### Connection Manager (Auto-Reconnect)

RabbitMQ connections can drop (network blip, broker restart). Use a ConnectionManager that auto-reconnects with exponential backoff:

```go
// pkg/mq/rabbitmq/connection_manager.go
type ConnectionManager struct {
    url        string
    conn       *amqp.Connection
    mu         sync.RWMutex
    logger     *zap.Logger
    closeCh    chan struct{}
}

func NewConnectionManager(url string, logger *zap.Logger) (*ConnectionManager, error) {
    cm := &ConnectionManager{url: url, logger: logger, closeCh: make(chan struct{})}
    if err := cm.connect(); err != nil {
        return nil, fmt.Errorf("initial connection: %w", err)
    }
    go cm.watchAndReconnect()
    return cm, nil
}

func (cm *ConnectionManager) connect() error {
    conn, err := amqp.Dial(cm.url)
    if err != nil {
        return err
    }
    cm.mu.Lock()
    cm.conn = conn
    cm.mu.Unlock()
    return nil
}

func (cm *ConnectionManager) watchAndReconnect() {
    for {
        cm.mu.RLock()
        notifyClose := cm.conn.NotifyClose(make(chan *amqp.Error, 1))
        cm.mu.RUnlock()

        select {
        case <-cm.closeCh:
            return
        case err := <-notifyClose:
            if err == nil {
                return // Graceful close
            }
            cm.logger.Warn("RabbitMQ connection lost, reconnecting...", zap.Error(err))
            cm.reconnectWithBackoff()
        }
    }
}

func (cm *ConnectionManager) reconnectWithBackoff() {
    backoff := 1 * time.Second
    maxBackoff := 30 * time.Second

    for {
        select {
        case <-cm.closeCh:
            return
        case <-time.After(backoff):
        }

        if err := cm.connect(); err != nil {
            cm.logger.Warn("reconnect failed", zap.Error(err), zap.Duration("next_retry", backoff))
            backoff = min(backoff*2, maxBackoff)
            continue
        }
        cm.logger.Info("RabbitMQ reconnected")
        return
    }
}

func (cm *ConnectionManager) Channel() (*amqp.Channel, error) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    return cm.conn.Channel()
}

func (cm *ConnectionManager) Close() error {
    close(cm.closeCh)
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    return cm.conn.Close()
}
```

**Key**: Consumers must handle channel errors and re-create channels after reconnect. The `NotifyClose` pattern from amqp library signals when the connection is lost.

## Synchronous Saga (MVP Stage)

Orchestrator calls downstream services sequentially via synchronous gRPC. Any step failure triggers immediate reverse compensation.

**Saga execution records must be persisted even in MVP**: If orchestrator crashes after a step completes but before compensation, pure in-memory `completedSteps` are lost. DB persistence lets Stale Record Scanner trigger precise compensation.

### Saga Execution Record Schema

```sql
CREATE TABLE saga_executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type VARCHAR(50) NOT NULL,           -- e.g., 'create_order'
    aggregate_id UUID NOT NULL,               -- Associated business entity ID
    status VARCHAR(20) NOT NULL DEFAULT 'running',  -- running/completed/compensating/failed
    current_step VARCHAR(50),
    version INT NOT NULL DEFAULT 1,           -- Optimistic lock (Scanner CAS claim)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE saga_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_id UUID NOT NULL REFERENCES saga_executions(id),
    step_name VARCHAR(50) NOT NULL,           -- e.g., 'RESERVE_INVENTORY', 'DEDUCT_BALANCE'
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending/completed/compensated/compensation_failed
    executed_at TIMESTAMPTZ,
    compensated_at TIMESTAMPTZ,
    error_message TEXT,
    UNIQUE (saga_id, step_name)
);

-- Partial index for scanning stuck sagas
CREATE INDEX idx_saga_executions_status ON saga_executions (status, updated_at)
WHERE status IN ('running', 'compensating');

CREATE TRIGGER set_saga_executions_updated_at
    BEFORE UPDATE ON saga_executions FOR EACH ROW
    EXECUTE FUNCTION trigger_set_updated_at();
```

### Execution Flow

```go
func (uc *UseCase) executeSyncSaga(ctx context.Context, entity *Entity) error {
    const stepTimeout = 5 * time.Second

    // 1. Create saga execution record
    sagaID, err := uc.sagaRepo.Create(ctx, &SagaExecution{
        SagaType: "create_order", AggregateID: entity.ID, Status: SagaStatusRunning,
    })
    if err != nil { return fmt.Errorf("create saga: %w", err) }

    // 2. Execute steps sequentially, persist after each completion
    steps := []struct {
        Name    string
        Execute func(ctx context.Context) error
    }{
        {"RESERVE_INVENTORY", func(ctx context.Context) error { return uc.inventoryClient.Reserve(ctx, entity) }},
        {"DEDUCT_BALANCE", func(ctx context.Context) error { return uc.walletClient.Deduct(ctx, entity) }},
    }

    var completedSteps []string
    for _, step := range steps {
        stepCtx, cancel := context.WithTimeout(ctx, stepTimeout)
        err := step.Execute(stepCtx)
        cancel()

        if err != nil {
            _ = uc.sagaRepo.RecordStepError(ctx, sagaID, step.Name, err.Error())
            uc.compensate(ctx, sagaID, entity, completedSteps)
            return fmt.Errorf("%s: %w", step.Name, err)
        }

        if err := uc.sagaRepo.RecordStepCompleted(ctx, sagaID, step.Name); err != nil {
            // Persistence failed but step executed → still add to completedSteps
            uc.logger.Error("failed to record step completion",
                zap.String("step", step.Name), zap.Error(err))
        }
        completedSteps = append(completedSteps, step.Name)
    }

    _ = uc.sagaRepo.UpdateStatus(ctx, sagaID, SagaStatusCompleted)
    return nil
}
```

### Reverse Compensation

```go
func (uc *UseCase) compensate(ctx context.Context, sagaID uuid.UUID, entity *Entity, completedSteps []string) {
    _ = uc.sagaRepo.UpdateStatus(ctx, sagaID, SagaStatusCompensating)

    // Compensation uses independent context — prevents original request cancel from blocking compensation
    const compensateStepTimeout = 5 * time.Second

    allCompensated := true
    for i := len(completedSteps) - 1; i >= 0; i-- {
        compCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), compensateStepTimeout)
        var err error
        switch completedSteps[i] {
        case "RESERVE_INVENTORY":
            err = uc.inventoryClient.ReleaseReservation(compCtx, entity)
        case "DEDUCT_BALANCE":
            err = uc.walletClient.Refund(compCtx, entity)
        }
        cancel()

        if err != nil {
            allCompensated = false
            _ = uc.sagaRepo.RecordStepCompensationFailed(ctx, sagaID, completedSteps[i], err.Error())
            uc.logger.Error("compensation failed",
                zap.String("step", completedSteps[i]),
                zap.String("entity_id", entity.ID.String()), zap.Error(err))
        } else {
            _ = uc.sagaRepo.RecordStepCompensated(ctx, sagaID, completedSteps[i])
        }
    }

    if allCompensated {
        _ = uc.sagaRepo.UpdateStatus(ctx, sagaID, SagaStatusCompleted)
    } else {
        _ = uc.sagaRepo.UpdateStatus(ctx, sagaID, SagaStatusFailed)  // Needs manual intervention or Scanner retry
    }
}
```

**All compensation operations MUST be idempotent**: May be triggered multiple times (original caller + Scanner simultaneously).

### Known Trade-offs (Resolved in Async Stage)

| Problem | Impact |
|---------|--------|
| Sequential gRPC call latency | High P99 |
| DB connection held during network I/O | Connection pool pressure |
| Downstream failure = direct operation failure | Limited availability |
| Extra DB write per step | Minor latency (milliseconds, acceptable) |

## Stale Record Scanner

Periodically scans stuck saga records. Unlike pure in-memory approach, Scanner reads `saga_steps` table to precisely determine which steps need compensation:

```go
type StaleRecordScanner struct {
    sagaRepo       SagaRepository
    logger         *zap.Logger
    staleThreshold time.Duration // Recommended: 10 min
    interval       time.Duration // Recommended: 2 min
}

func (s *StaleRecordScanner) Start(ctx context.Context) {
    ticker := time.NewTicker(s.interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done(): return
        case <-ticker.C: s.scanAndRecover(ctx)
        }
    }
}

func (s *StaleRecordScanner) scanAndRecover(ctx context.Context) {
    stuckSagas, err := s.sagaRepo.GetStuckSagas(ctx, s.staleThreshold)
    if err != nil { s.logger.Error("failed to get stuck sagas", zap.Error(err)); return }

    for _, saga := range stuckSagas {
        // CAS claim: prevents multiple Scanner instances from processing same saga
        updated, err := s.sagaRepo.CompareAndSwapStatus(
            ctx, saga.ID, saga.Version, saga.Status, SagaStatusCompensating)
        if err != nil || !updated { continue }

        completedSteps, _ := s.sagaRepo.GetCompletedSteps(ctx, saga.ID)
        s.logger.Warn("recovering stuck saga",
            zap.String("saga_id", saga.ID.String()),
            zap.Strings("completed_steps", completedSteps))
        // Trigger compensation (shared with UseCase.compensate)
    }
}
```

## Asynchronous Saga + Outbox `[Async]`

Single TX writes business record + Saga state + Outbox events (atomicity). Returns PENDING immediately.
Events are collected from Entity (not manually constructed), ensuring state change and event consistency:

```go
func (uc *UseCase) Execute(ctx context.Context, req *Request) (*Response, error) {
    entity := buildEntity(req)

    err = uc.txManager.WithTx(ctx, func(txCtx context.Context) error {
        if err := uc.repo.Create(txCtx, entity); err != nil { return err }
        if err := uc.sagaRepo.Create(txCtx, sagaExec); err != nil { return err }

        // Extract Domain Events collected by Entity and write to Outbox
        for _, evt := range entity.Events() {
            if err := uc.outboxRepo.Create(txCtx, &OutboxEvent{
                AggregateType: "Order",
                AggregateID:   entity.ID,
                EventType:     evt.EventType(),
                EventVersion:  evt.Version(),  // Carried for consumer-side version handling
                Payload:       toJSON(evt),
            }); err != nil { return err }
        }
        // ⚠️ Do NOT call ClearEvents() inside TX closure
        // If TX rolls back, Entity events remain — no inconsistent state
        return nil
    })
    if err != nil { return nil, err }

    entity.ClearEvents()  // Clear ONLY after TX commit succeeds
    return response.FromEntity(entity), nil  // Return PENDING immediately
}
```

## Two-Phase Outbox Poller `[Async]`

**Design principle**: Separate "claim events" (short TX) from "publish + update status" (outside TX). Prevents long transactions from holding DB connections and row locks.

```go
type OutboxPoller struct {
    repo         OutboxRepository
    txManager    TxManager
    publisher    mq.Publisher
    logger       *zap.Logger
    maxRetries   int           // Recommended: 10
    retention    time.Duration // Recommended: 7 days
    minInterval  time.Duration // Recommended: 100ms (high backpressure)
    maxInterval  time.Duration // Recommended: 5s (idle)
    baseInterval time.Duration // Recommended: 1s
}

func (p *OutboxPoller) processBatch(ctx context.Context) int {
    // ═══ Phase 1: Short TX — claim events ═══
    var events []*OutboxEvent
    err := p.txManager.WithTx(ctx, func(txCtx context.Context) error {
        var err error
        events, err = p.repo.PickUnsentEvents(txCtx, batchSize)
        return err
    })
    // TX committed here — row locks released

    if err != nil { p.logger.Error("failed to pick outbox events", zap.Error(err)); return -1 }
    if len(events) == 0 { return 0 }

    // ═══ Phase 2: Publish + update status (no TX) ═══
    processed := 0
    for _, event := range events {
        if event.RetryCount >= p.maxRetries {
            _ = p.repo.MarkAsFailed(ctx, event.ID)
            processed++
            continue
        }

        if err := p.publisher.Publish(ctx, event.EventType, event.EventVersion, event.Payload); err != nil {
            p.logger.Error("publish failed, will retry next round",
                zap.String("event_id", event.ID.String()), zap.Error(err))
            _ = p.repo.IncrementRetryAndUnpick(ctx, event.ID, err.Error())
            continue
        }

        if err := p.repo.MarkAsSent(ctx, event.ID); err != nil {
            // Published but DB update failed → will re-publish next round
            // This is inherent at-least-once semantics — consumer handles via idempotency
            p.logger.Error("published but failed to mark as sent",
                zap.String("event_id", event.ID.String()), zap.Error(err))
        }
        processed++
    }
    return processed
}
```

### Adaptive Polling

```go
func (p *OutboxPoller) Start(ctx context.Context) {
    interval := p.baseInterval
    for {
        select {
        case <-ctx.Done(): return
        case <-time.After(interval):
            processed := p.processBatch(ctx)
            if processed < 0 {
                interval = min(interval*2, p.maxInterval)  // Query failed → exponential backoff
            } else if processed >= batchSize {
                interval = p.minInterval  // More pending → speed up
            } else if processed == 0 {
                interval = min(interval*2, p.maxInterval)  // Empty → slow down
            } else {
                interval = p.baseInterval
            }
        }
    }
}
```

### Stuck Events Safety Net

Run on Poller startup and periodically to handle events stuck due to Poller crash:

```go
func (p *OutboxPoller) recoverStuckEvents(ctx context.Context) {
    affected, err := p.repo.UnpickStuckEvents(ctx)
    if err != nil { p.logger.Error("failed to unpick stuck events", zap.Error(err)); return }
    if affected > 0 {
        p.logger.Warn("recovered stuck outbox events", zap.Int64("count", affected))
    }
}
```

### Two-Phase vs Single-TX Comparison

| Aspect | Single TX | Two-Phase |
|--------|-----------|-----------|
| TX hold time | Entire batch publish (seconds) | Claim only (milliseconds) |
| Row lock scope | Held during publish | Released after pick |
| Publish failure handling | Swallowed inside TX | Independent, errors logged explicitly |
| Commit failure + already published | Message sent but DB not updated — inconsistent | N/A — publish is outside TX |
| Crash recovery | Cannot distinguish "processing" vs "unprocessed" | `picked_at` timeout → `UnpickStuckEvents` resets |

## TxManager `[Async]`

Interface defined in Application Port. Implementation in Infrastructure (dependency inversion).

```go
// internal/application/port/output/tx_manager.go
type TxManager interface {
    WithTx(ctx context.Context, fn func(txCtx context.Context) error) error
}
```

```go
// pkg/database/dbtx.go
package database

type CtxKey string
const TxKey CtxKey = "tx"

// DBTX — shared interface satisfied by both pgxpool.Pool and pgx.Tx (Go structural typing).
// Each service's sqlcgen.DBTX is identical, but we define a shared one here to avoid
// coupling pkg/database to any service's generated code.
type DBTX interface {
    Exec(ctx context.Context, sql string, arguments ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

// GetDBTX lets Repository dynamically choose TX or pool
func GetDBTX(ctx context.Context, pool *pgxpool.Pool) DBTX {
    if tx, ok := ctx.Value(TxKey).(pgx.Tx); ok { return tx }
    return pool
}
```

```go
// internal/infrastructure/database/tx_manager.go
package database

import pkgdb "github.com/yourproject/go-pkg/database"

type PgxTxManager struct { pool *pgxpool.Pool }

func NewTxManager(pool *pgxpool.Pool) *PgxTxManager {
    return &PgxTxManager{pool: pool}
}

func (m *PgxTxManager) WithTx(ctx context.Context, fn func(txCtx context.Context) error) error {
    // Nested TX protection: if already in TX, execute within existing TX
    // Prevents UseCase A calling UseCase B from opening independent TX causing inconsistency
    if _, ok := ctx.Value(pkgdb.TxKey).(pgx.Tx); ok {
        return fn(ctx)
    }

    tx, err := m.pool.Begin(ctx)
    if err != nil { return fmt.Errorf("begin tx: %w", err) }

    txCtx := context.WithValue(ctx, pkgdb.TxKey, tx)

    if err := fn(txCtx); err != nil {
        // WithoutCancel ensures rollback executes even if original ctx is cancelled
        rollbackCtx := context.WithoutCancel(ctx)
        if rbErr := tx.Rollback(rollbackCtx); rbErr != nil {
            return fmt.Errorf("rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }
    // WithoutCancel ensures commit succeeds even if caller's ctx is cancelled after fn returns.
    // fn already completed successfully — losing the commit would silently discard the work.
    return tx.Commit(context.WithoutCancel(ctx))
}
```

## Saga Timeout Monitor `[Async]`

```go
type SagaTimeoutMonitor struct {
    sagaRepo  SagaRepository
    publisher mq.Publisher
    logger    *zap.Logger
    interval  time.Duration  // Recommended: 1 min
    timeout   time.Duration  // Recommended: 5 min
}

func (m *SagaTimeoutMonitor) checkStuckSagas(ctx context.Context) {
    stuckSagas, _ := m.sagaRepo.GetStuckSagas(ctx, m.timeout)

    for _, saga := range stuckSagas {
        switch saga.Status {
        case SagaStatusRunning:
            // Running but stuck → CAS claim then trigger compensation
            updated, err := m.sagaRepo.CompareAndSwapStatus(
                ctx, saga.ID, saga.Version, SagaStatusRunning, SagaStatusCompensating)
            if err != nil || !updated { continue }
            m.compensateStuckSaga(ctx, saga)

        case SagaStatusCompensating:
            // Compensation also stuck → mark as Failed (needs manual intervention)
            updated, err := m.sagaRepo.CompareAndSwapStatus(
                ctx, saga.ID, saga.Version, SagaStatusCompensating, SagaStatusFailed)
            if err != nil || !updated { continue }
            m.logger.Error("saga compensation stuck, marking as failed",
                zap.String("saga_id", saga.ID.String()),
                zap.String("saga_type", saga.SagaType))
        }
    }
}
```

## Repository Interface Segregation

Follow Interface Segregation Principle:

```go
// Business Repository (stable across stages)
type EntityRepository interface {
    Create(ctx context.Context, entity *Entity) error
    GetByID(ctx context.Context, id uuid.UUID) (*Entity, error)
    Update(ctx context.Context, entity *Entity) error
    Delete(ctx context.Context, id uuid.UUID) error
}

// Outbox Repository (two-phase design)
type OutboxRepository interface {
    Create(ctx context.Context, event *OutboxEvent) error
    PickUnsentEvents(ctx context.Context, batchSize int) ([]*OutboxEvent, error)
    MarkAsSent(ctx context.Context, id uuid.UUID) error
    MarkAsFailed(ctx context.Context, id uuid.UUID) error
    IncrementRetryAndUnpick(ctx context.Context, id uuid.UUID, errMsg string) error
    UnpickStuckEvents(ctx context.Context) (int64, error)
    DeleteSentEventsBefore(ctx context.Context, before time.Time) (int64, error)
}

// Saga Repository
type SagaRepository interface {
    Create(ctx context.Context, saga *SagaExecution) (uuid.UUID, error)
    GetByID(ctx context.Context, id uuid.UUID) (*SagaExecution, error)
    UpdateStatus(ctx context.Context, id uuid.UUID, status SagaStatus) error
    GetStuckSagas(ctx context.Context, threshold time.Duration) ([]*SagaExecution, error)
    CompareAndSwapStatus(ctx context.Context, id uuid.UUID, version int, from, to SagaStatus) (bool, error)
    RecordStepCompleted(ctx context.Context, sagaID uuid.UUID, stepName string) error
    RecordStepError(ctx context.Context, sagaID uuid.UUID, stepName, errMsg string) error
    RecordStepCompensated(ctx context.Context, sagaID uuid.UUID, stepName string) error
    RecordStepCompensationFailed(ctx context.Context, sagaID uuid.UUID, stepName, errMsg string) error
    GetCompletedSteps(ctx context.Context, sagaID uuid.UUID) ([]string, error)
}
```

## CQRS Read Model Projection `[Async]`

### When to Introduce

| Signal | Threshold |
|--------|-----------|
| List queries need JOIN 3+ tables or cross-service assembly | ✅ Introduce |
| Read/write ratio > 10:1 | ✅ Introduce |
| Read and write performance requirements diverge significantly | ✅ Introduce |
| Read model structure ≈ write model (single-table queries suffice) | ❌ Skip |

**Prerequisite**: Domain Event + MQ Consumer infrastructure must already exist (Async stage).

### Architecture

```
Command path:  gRPC → UseCase → Entity → Repository → Main DB (normalized)
                                    ↓
                              Domain Event (via Outbox)
                                    ↓
                              MQ Consumer (Projector)
                                    ↓
Query path:    gRPC → QueryService → Read DB (denormalized, zero JOIN)
```

### Denormalized Read Table

```sql
-- Read model: flatten all data needed for list API into one table
CREATE TABLE order_list_view (
    order_id       UUID PRIMARY KEY,
    buyer_id       UUID NOT NULL,
    buyer_name     TEXT NOT NULL,
    merchant_id    UUID NOT NULL,
    merchant_name  TEXT NOT NULL,
    status         TEXT NOT NULL,
    amount         BIGINT NOT NULL,
    item_count     INT NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL,
    updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_order_list_view_buyer ON order_list_view (buyer_id, created_at DESC, order_id DESC);
```

### Event Projector (MQ Consumer)

Projector listens to Domain Events and maintains the read model:

```go
// internal/adapter/inbound/mq/order_projector.go
type OrderProjector struct {
    viewRepo  OrderViewRepository
    userClient userv1.UserServiceClient  // Cross-service query for denormalization
    logger    *zap.Logger
}

func (p *OrderProjector) Handle(ctx context.Context, event DomainEvent) error {
    switch e := event.(type) {
    case *OrderCreatedEvent:
        buyer, err := p.userClient.GetUser(ctx, &userv1.GetUserRequest{Id: e.BuyerID.String()})
        if err != nil {
            return fmt.Errorf("fetch buyer for projection: %w", err)  // Transient → retry
        }
        return p.viewRepo.Upsert(ctx, &OrderListView{
            OrderID:      e.OrderID,
            BuyerID:      e.BuyerID,
            BuyerName:    buyer.Name,
            MerchantID:   e.MerchantID,
            MerchantName: e.MerchantName,
            Status:       e.Status.String(),
            Amount:       e.Amount,
            ItemCount:    e.ItemCount,
            CreatedAt:    e.CreatedAt,
            UpdatedAt:    e.CreatedAt,
        })

    case *OrderStatusChangedEvent:
        return p.viewRepo.UpdateStatus(ctx, e.OrderID, e.NewStatus.String(), e.OccurredAt)

    case *UserNameChangedEvent:
        // Propagate cross-service changes to denormalized fields
        return p.viewRepo.UpdateBuyerName(ctx, e.UserID, e.NewName)
    }
    return nil
}
```

### Query Service

Query Service bypasses Domain Entity — reads directly from the denormalized view:

```go
// internal/application/query/order_query_service.go
type OrderQueryService struct {
    viewRepo OrderViewRepository
}

func (q *OrderQueryService) ListByBuyer(ctx context.Context, buyerID uuid.UUID, cursor *pagination.Cursor, pageSize int) ([]*OrderListView, error) {
    return q.viewRepo.ListByBuyer(ctx, buyerID, cursor, pageSize)
}
```

**gRPC handler** routes commands to UseCase, queries to QueryService:

```go
func (s *OrderServer) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.CreateOrderResponse, error) {
    return s.useCase.Create(ctx, req)  // Command → UseCase → Entity
}

func (s *OrderServer) ListOrders(ctx context.Context, req *pb.ListOrdersRequest) (*pb.ListOrdersResponse, error) {
    return s.queryService.ListByBuyer(ctx, req)  // Query → QueryService → View
}
```

### Read-Your-Writes Consistency

Projection has eventual consistency delay. When a user writes and immediately reads, they may not see their own update. Two mitigation strategies:

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **Fallback to main DB** | After write, set a short-lived flag (Redis TTL 3s). Query checks flag — if set, read from main DB instead of view | Simple; adds slight main DB load for recently-written users |
| **Return write result** | `CreateOrder` response includes the created order data. Client uses this for immediate display, then switches to list API on next load | Zero extra DB cost; client must handle the logic |

**Fallback pattern**:

```go
func (q *OrderQueryService) ListByBuyer(ctx context.Context, buyerID uuid.UUID, cursor *pagination.Cursor, pageSize int) ([]*OrderListView, error) {
    // Check if user recently wrote (projection may lag)
    key := fmt.Sprintf("cqrs:written:%s", buyerID)
    if q.redis.Exists(ctx, key).Val() > 0 {
        // Fallback to main DB query (slower but consistent)
        return q.mainRepo.ListByBuyer(ctx, buyerID, cursor, pageSize)
    }
    return q.viewRepo.ListByBuyer(ctx, buyerID, cursor, pageSize)
}
```

### Architecture Placement

| Component | Location | Layer |
|-----------|----------|-------|
| Read table schema | `db/schema/` | Infrastructure |
| OrderViewRepository | `internal/repository/` | Infrastructure |
| Event Projector | `internal/adapter/inbound/mq/` | Adapter |
| OrderQueryService | `internal/application/query/` | Application |
| gRPC handler routing | `internal/adapter/inbound/grpc/` | Adapter |
