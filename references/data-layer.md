# Data Layer

## Table of Contents

- [Database-per-Service](#database-per-service)
- [Schema Management: sqlc + Atlas](#schema-management-sqlc--atlas)
  - [Migration Safety & Rollback Strategy](#migration-safety--rollback-strategy)
  - [Nullable Type Helpers (pkg/sqlutil)](#nullable-type-helpers-pkgsqlutil)
  - [Repository Implementation Pattern](#repository-implementation-pattern)
- [Connection Pool Tuning](#connection-pool-tuning)
- [Configuration Management](#configuration-management)
- [Logging](#logging)

## Database-per-Service

Share a single PG instance, but each service gets an independent Database + User (logical isolation):

```sql
CREATE USER order_svc WITH PASSWORD '${ORDER_SVC_PASSWORD}';
CREATE DATABASE order_db OWNER order_svc;
REVOKE ALL ON DATABASE order_db FROM PUBLIC;
GRANT CONNECT ON DATABASE order_db TO order_svc;

-- Restrict search_path to prevent cross-schema queries
ALTER USER order_svc SET search_path TO public;

-- CONNECTION LIMIT prevents one service from exhausting all connections
ALTER USER order_svc CONNECTION LIMIT 50;
```

## Schema Management: sqlc + Atlas

### Directory Structure

```
services/xxx-service/
├── db/                         # Database-related (centralized)
│   ├── schema/schema.sql       # Desired complete schema (single source of truth)
│   ├── queries/                # sqlc query definitions
│   │   ├── order.sql
│   │   └── outbox.sql
│   ├── migrations/             # Atlas auto-generated migrations
│   ├── sqlc.yaml               # sqlc configuration
│   └── atlas.hcl               # Atlas configuration
│
├── sqlcgen/                    # sqlc auto-generated Go code (service root for import convenience)
```

> Run commands from within `db/`: `cd db && sqlc generate`, `cd db && atlas migrate diff ...`

### Schema Design Example

```sql
-- Shared trigger function: auto-update updated_at
CREATE OR REPLACE FUNCTION trigger_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    amount BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER set_orders_updated_at
    BEFORE UPDATE ON orders FOR EACH ROW
    EXECUTE FUNCTION trigger_set_updated_at();

CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);
```

### Outbox Table Design

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_version INT NOT NULL DEFAULT 1,  -- Event schema version for consumer migration
    payload JSONB NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    picked_at TIMESTAMPTZ,     -- Two-phase Poller: marks as claimed
    sent_at TIMESTAMPTZ,
    failed_at TIMESTAMPTZ
);

-- Partial index: accelerates Poller Phase 1 claim query
CREATE INDEX idx_outbox_unsent ON outbox_events (created_at ASC)
WHERE sent_at IS NULL AND failed_at IS NULL AND picked_at IS NULL;

-- Partial index: accelerates stuck event detection
CREATE INDEX idx_outbox_stuck ON outbox_events (picked_at ASC)
WHERE picked_at IS NOT NULL AND sent_at IS NULL AND failed_at IS NULL;
```

### sqlc Query Examples

```sql
-- name: CreateOrder :one
INSERT INTO orders (user_id, status, amount, currency)
VALUES ($1, $2, $3, $4) RETURNING *;

-- name: GetOrderByID :one
SELECT * FROM orders WHERE id = $1;

-- name: UpdateOrderStatus :execresult
UPDATE orders SET status = $2, version = version + 1
WHERE id = $1 AND version = $3;
-- Note: updated_at is auto-updated by trigger_set_updated_at

-- name: GetOrderForUpdate :one
SELECT * FROM orders WHERE id = $1 FOR UPDATE;

-- name: CreateOutboxEvent :one
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, event_version, payload)
VALUES ($1, $2, $3, $4, $5) RETURNING *;

-- name: PickUnsentEvents :many
-- Phase 1: Short TX claim with SKIP LOCKED
UPDATE outbox_events SET picked_at = NOW()
WHERE id IN (
    SELECT id FROM outbox_events
    WHERE sent_at IS NULL AND failed_at IS NULL AND picked_at IS NULL
    ORDER BY created_at ASC LIMIT $1
    FOR UPDATE SKIP LOCKED
) RETURNING *;

-- name: MarkEventAsSent :exec
UPDATE outbox_events SET sent_at = NOW() WHERE id = $1;

-- name: MarkEventAsFailed :exec
UPDATE outbox_events SET failed_at = NOW() WHERE id = $1;

-- name: IncrementRetryAndUnpick :exec
UPDATE outbox_events
SET retry_count = retry_count + 1, last_error = $2, picked_at = NULL
WHERE id = $1;

-- name: UnpickStuckEvents :execresult
-- Safety net: reset events stuck in processing too long
UPDATE outbox_events SET picked_at = NULL
WHERE picked_at IS NOT NULL AND sent_at IS NULL AND failed_at IS NULL
  AND picked_at < NOW() - INTERVAL '2 minutes';

-- name: DeleteSentEventsBefore :execresult
-- Data retention: clean up successfully sent events older than retention period
DELETE FROM outbox_events
WHERE sent_at IS NOT NULL AND sent_at < $1;
```

### Outbox Data Retention

Sent events accumulate over time. Schedule periodic cleanup (recommended: 7-day retention):

```go
// Run daily or as part of a cron job
cutoff := time.Now().Add(-7 * 24 * time.Hour)
result, err := outboxRepo.DeleteSentEventsBefore(ctx, cutoff)
```

### sqlc.yaml

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "queries/"
    schema: "schema/schema.sql"
    gen:
      go:
        package: "sqlcgen"
        out: "../sqlcgen"           # Output to service root (../sqlcgen/ from db/)
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_result_struct_pointers: true
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
```

### Workflow

```bash
# Run all commands from db/ directory: cd services/xxx-service/db

# 1. Edit schema/schema.sql (add columns, indexes, etc.)

# 2. Atlas auto-generates migration
atlas migrate diff add_new_column \
  --dir "file://migrations" \
  --to "file://schema/schema.sql" \
  --dev-url "postgres://localhost:5432/dev?sslmode=disable"

# 3. Review generated migration file

# 4. Update queries/*.sql (if needed)

# 5. Regenerate Go code (outputs to ../sqlcgen/)
sqlc generate

# 6. Apply migration
atlas migrate apply --dir "file://migrations" --url "$DATABASE_URL"
```

### Migration Safety & Rollback Strategy

Atlas generates forward-only migrations. For safe production rollbacks:

**Backward-compatible changes only** — deploy schema before code:

| Change Type | Safe Strategy | Unsafe |
|-------------|---------------|--------|
| Add column | Add as nullable or with default | Add as NOT NULL without default |
| Remove column | Deploy code first (stop reading), then drop column in next release | Drop column while code still reads it |
| Rename column | Add new column → copy data → deploy code → drop old column | `ALTER COLUMN RENAME` |
| Add index | `CREATE INDEX CONCURRENTLY` | `CREATE INDEX` (locks table) |
| Change type | Add new column → migrate → deploy → drop old | `ALTER COLUMN TYPE` |

**CI check for destructive changes**:

```bash
# atlas migrate lint detects DROP TABLE, DROP COLUMN, etc.
atlas migrate lint --dir "file://migrations" \
  --dev-url "postgres://localhost:5432/dev?sslmode=disable"
```

**Rollback strategy**: Since schema changes are backward-compatible, rollback = deploy previous code version. The schema supports both old and new code.

### Nullable Type Helpers (pkg/sqlutil)

sqlc generates `pgtype.Text`, `pgtype.Int4`, etc. for nullable columns. Centralize conversion helpers in `pkg/sqlutil` to avoid duplication across services:

```go
// pkg/sqlutil/nullable.go
package sqlutil

import (
    "encoding/json"
    "time"
    "github.com/jackc/pgx/v5/pgtype"
)

// --- To pgtype (for INSERT/UPDATE) ---

func Text(s string) pgtype.Text {
    if s == "" { return pgtype.Text{Valid: false} }
    return pgtype.Text{String: s, Valid: true}
}

func Int4(i int32) pgtype.Int4 {
    return pgtype.Int4{Int32: i, Valid: true}
}

func Int4From(i int) pgtype.Int4 {  // For services using int instead of int32
    return pgtype.Int4{Int32: int32(i), Valid: true}
}

func Int8(i int64) pgtype.Int8 {
    return pgtype.Int8{Int64: i, Valid: true}
}

func Bool(b bool) pgtype.Bool {
    return pgtype.Bool{Bool: b, Valid: true}
}

func Float8(f float64) pgtype.Float8 {
    return pgtype.Float8{Float64: f, Valid: true}
}

func Timestamptz(t time.Time) pgtype.Timestamptz {
    return pgtype.Timestamptz{Time: t, Valid: !t.IsZero()}
}

func TimestamptzPtr(t *time.Time) pgtype.Timestamptz {
    if t == nil { return pgtype.Timestamptz{Valid: false} }
    return pgtype.Timestamptz{Time: *t, Valid: true}
}

func Date(t *time.Time) pgtype.Date {
    if t == nil { return pgtype.Date{Valid: false} }
    return pgtype.Date{Time: *t, Valid: true}
}

func JSON(data json.RawMessage) []byte {
    if data == nil { return nil }
    return []byte(data)
}

// --- From pgtype (for SELECT) ---

func TextValue(t pgtype.Text) string {
    if !t.Valid { return "" }
    return t.String
}

func Int4Value(i pgtype.Int4) int32 {
    if !i.Valid { return 0 }
    return i.Int32
}

func Int4ToInt(i pgtype.Int4) int {  // For services using int instead of int32
    if !i.Valid { return 0 }
    return int(i.Int32)
}

func Int8Value(i pgtype.Int8) int64 {
    if !i.Valid { return 0 }
    return i.Int64
}

func BoolValue(b pgtype.Bool) bool {
    return b.Valid && b.Bool
}

func Float8Value(f pgtype.Float8) float64 {
    if !f.Valid { return 0 }
    return f.Float64
}

func TimestamptzValue(t pgtype.Timestamptz) time.Time {
    if !t.Valid { return time.Time{} }
    return t.Time
}

func TimestamptzToPtr(t pgtype.Timestamptz) *time.Time {
    if !t.Valid { return nil }
    return &t.Time
}

func DateValue(d pgtype.Date) *time.Time {
    if !d.Valid { return nil }
    return &d.Time
}

func JSONValue(data []byte) json.RawMessage {
    if data == nil { return nil }
    return json.RawMessage(data)
}
```

### Repository Implementation Pattern

Each repository directly inlines the `sqlcgen.New()` call with `database.GetDBTX()`:

```go
// adapter/outbound/persistence/order_repository.go
package persistence

import (
    "context"
    "errors"
    "fmt"
    "github.com/google/uuid"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/yourproject/go-pkg/database"
    "github.com/yourproject/go-pkg/sqlutil"
    "github.com/yourproject/order-service/internal/domain"
    "github.com/yourproject/order-service/internal/domain/entity"
    "github.com/yourproject/order-service/internal/domain/repository"
    "github.com/yourproject/order-service/internal/domain/valueobject"
    "github.com/yourproject/order-service/sqlcgen"
)

type orderRepository struct {
    pool *pgxpool.Pool
}

func NewOrderRepository(pool *pgxpool.Pool) repository.OrderRepository {
    return &orderRepository{pool: pool}
}

func (r *orderRepository) GetByID(ctx context.Context, id uuid.UUID) (*entity.Order, error) {
    // Inline pattern: sqlcgen.New() + database.GetDBTX() for TX support
    q := sqlcgen.New(database.GetDBTX(ctx, r.pool))
    row, err := q.GetOrderByID(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, domain.ErrOrderNotFound  // Map to Domain Error
        }
        return nil, err
    }
    return r.toEntity(row)
}

func (r *orderRepository) Create(ctx context.Context, o *entity.Order) error {
    q := sqlcgen.New(database.GetDBTX(ctx, r.pool))
    row, err := q.CreateOrder(ctx, sqlcgen.CreateOrderParams{
        UserID:   o.UserID,
        Status:   sqlutil.Text(o.Status.String()),
        Amount:   sqlutil.Int8(o.TotalAmount.Amount),
        Currency: sqlutil.Text(o.TotalAmount.Currency),
    })
    if err != nil {
        return err
    }
    o.ID = row.ID
    return nil
}

func (r *orderRepository) toEntity(row sqlcgen.Order) (*entity.Order, error) {
    status, err := valueobject.OrderStatusString(sqlutil.TextValue(row.Status))
    if err != nil {
        return nil, fmt.Errorf("parse order status %q: %w", sqlutil.TextValue(row.Status), err)
    }
    return &entity.Order{
        ID:     row.ID,
        UserID: row.UserID,
        Status: status,
        TotalAmount: valueobject.Money{
            Amount:   sqlutil.Int8Value(row.Amount),
            Currency: sqlutil.TextValue(row.Currency),
        },
        Version:   int(sqlutil.Int4Value(row.Version)),
        CreatedAt: sqlutil.TimestamptzValue(row.CreatedAt),
        UpdatedAt: sqlutil.TimestamptzValue(row.UpdatedAt),
    }, nil
}
```

**Key Points:**
- **No helpers.go**: Don't create a `getQueries()` helper function. Each service has its own `sqlcgen.Queries` type, so it can't be shared. Inline the call directly.
- **TX Support**: `database.GetDBTX(ctx, pool)` returns the transaction from context if available, otherwise the pool. This enables repositories to participate in transactions transparently.
- **Not Found Pattern**: Return `nil, domain.ErrXxxNotFound` for not found. Repository maps `pgx.ErrNoRows` to domain error; UseCase handles it directly via `errors.Is`.
- **Type Conventions**: Use `Int4From(int)` / `Int4ToInt()` if your Entity uses `int`, or `Int4(int32)` / `Int4Value()` if it uses `int32`.

## Connection Pool Tuning

```go
// Formula: PG CONNECTION LIMIT ≥ max_pods × MaxConns + buffer
config, _ := pgxpool.ParseConfig(databaseURL)
config.MaxConns = 5                          // Max connections per pod
config.MinConns = 2                          // Pre-warm to avoid cold start latency
config.MaxConnLifetime = 30 * time.Minute    // Periodic recycling (prevents stale connections to old PG nodes)
config.MaxConnIdleTime = 5 * time.Minute     // Idle connection reclaim
config.HealthCheckPeriod = 30 * time.Second  // Periodic health check, detect bad connections early

pool, err := pgxpool.NewWithConfig(ctx, config)
```

**Warning**: During sync Saga gRPC calls, DB connections are held while waiting for network I/O. Actual connection hold time can far exceed query time. Consider `pgbouncer` or decouple DB operations from remote calls in the Async stage.

## Configuration Management

Use native `os.Getenv` + struct. Zero dependency, compile-time safe.

```go
// pkg/config/config.go
type Config struct {
    Server    ServerConfig
    Database  DatabaseConfig
    Redis     RedisConfig
    RabbitMQ  RabbitMQConfig
    OTel      OTelConfig
    Environment string
    ServiceName string
}

type OTelConfig struct {
    Endpoint     string  // e.g., "otel-collector:4317"
    SamplingRate float64 // 0.0–1.0, default 1.0 (dev)
}

// Fail-Fast: collect all missing required vars at once
type envCollector struct { missing []string }

func (c *envCollector) require(key string) string {
    val := os.Getenv(key)
    if val == "" { c.missing = append(c.missing, key) }
    return val
}

func (c *envCollector) validate() error {
    if len(c.missing) == 0 { return nil }
    return fmt.Errorf("missing required environment variables: %s", strings.Join(c.missing, ", "))
}

func Load() (*Config, error) {
    c := &envCollector{}
    cfg := &Config{
        Server: ServerConfig{
            Port: getEnv("SERVER_PORT", "50051"),  // Has default → optional
        },
        Database: DatabaseConfig{
            Host:     c.require("DATABASE_HOST"),      // No default → required
            Port:     getEnv("DATABASE_PORT", "5432"),
            User:     c.require("DATABASE_USER"),
            Password: c.require("DATABASE_PASSWORD"),
            DBName:   c.require("DATABASE_NAME"),
        },
        Redis:    RedisConfig{Addr: c.require("REDIS_ADDR")},
        RabbitMQ: RabbitMQConfig{URL: c.require("RABBITMQ_URL")},
    }
    if err := c.validate(); err != nil { return nil, err }
    return cfg, nil
}

func getEnv(key, fallback string) string {
    if val := os.Getenv(key); val != "" { return val }
    return fallback
}
```

### Design Principles

- `getEnv(key, fallback)`: Optional with default (dev can start with zero config)
- `c.require(key)`: Required with no default (Load() returns error, no panic)
- All config in one struct, injected via DI — no global state
- **Fail-Fast**: Collect ALL missing required vars, report together. Eliminates repeated restart-to-debug cycles.

## Logging

See [observability.md](observability.md) for complete logging implementation including:
- Logger initialization (Zap + JSON output)
- Log schema standard
- Context-aware logging with trace correlation
- Log level guidelines
- Error logging patterns
- Sensitive data masking
- Log sampling for production
