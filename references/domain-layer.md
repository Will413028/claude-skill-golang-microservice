# Domain Layer Design Patterns

## Table of Contents

- [Data Object Naming Convention](#data-object-naming-convention)
- [Entity State Machine](#entity-state-machine)
- [Value Object](#value-object)
- [Type-Safe Enums with Enumer](#type-safe-enums-with-enumer)
- [Repository Interface](#repository-interface)
- [Domain Event](#domain-event)
- [Domain Service](#domain-service)
- [Domain Layer Constraints](#domain-layer-constraints)

## Data Object Naming Convention

Use **DTO (Data Transfer Object)** naming for application layer input/output.

| Layer | Object Type | Location | Description |
|-------|-------------|----------|-------------|
| **Domain** | `Entity` | `domain/entity/` | Pure business logic, no DB tags |
| **Domain** | `ValueObject` | `domain/valueobject/` | Immutable, equality by value |
| **Application** | `Request DTO` | `application/dto/request/` | UseCase input |
| **Application** | `Response DTO` | `application/dto/response/` | UseCase output |
| **Adapter** | `sqlc struct` | `sqlcgen/` | Auto-generated, DB mapping |

### Data Flow

```
gRPC Request → Handler → Request DTO → UseCase → Entity ← Repository ← sqlc struct
                                           ↓
                             Response DTO ← UseCase
                                           ↓
                              Handler → gRPC Response
```

### Mapping Responsibilities

| From | To | Where |
|------|----|-------|
| `pb.Request` | `request.DTO` | Handler (Mapper) |
| `request.DTO` | `Entity` | UseCase |
| `sqlc.Row` | `Entity` | Repository (`toEntity()`) |
| `Entity` | `response.DTO` | UseCase |
| `response.DTO` | `pb.Response` | Handler (Mapper) |

## Entity State Machine

### Design Principles

- Use a **whitelist** to define valid state transitions
- Inject `now time.Time` for testability
- Entity collects Domain Events during state transitions
- `Version` field for optimistic locking (incremented by Repository SQL)

### Concurrency Safety

Entity is NOT goroutine-safe. A single Entity instance must only be operated within one goroutine (including state transitions and Domain Event collection). For cross-goroutine access to the same business data, reload via Repository and use optimistic lock (Version) for consistency.

### Example

```go
type Order struct {
    ID        uuid.UUID
    UserID    uuid.UUID
    Status    OrderStatus
    TotalAmount valueobject.Money
    Version   int          // Optimistic lock
    CreatedAt time.Time
    UpdatedAt time.Time
    events []event.DomainEvent // Unpublished Domain Events
}

// --- Domain Event collection ---
func (o *Order) AddEvent(e event.DomainEvent) { o.events = append(o.events, e) }
func (o *Order) Events() []event.DomainEvent  { return o.events }
func (o *Order) ClearEvents()                 { o.events = nil }

// State transition whitelist
var validTransitions = map[OrderStatus][]OrderStatus{
    StatusPending:   {StatusConfirmed, StatusCancelled},
    StatusConfirmed: {StatusPaid, StatusCancelled},
    StatusPaid:      {StatusShipped, StatusRefunding},
}

func (o *Order) canTransitionTo(target OrderStatus) error {
    allowed, ok := validTransitions[o.Status]
    if !ok {
        return ErrInvalidTransition
    }
    for _, s := range allowed {
        if s == target { return nil }
    }
    return ErrInvalidTransition
}

func (o *Order) transitionTo(target OrderStatus, now time.Time) error {
    if err := o.canTransitionTo(target); err != nil {
        return err
    }
    from := o.Status
    o.Status = target
    o.UpdatedAt = now
    // Version is incremented by Repository SQL, not here

    // Collect Domain Event on state transition
    o.AddEvent(event.OrderStatusChanged{
        OrderID: o.ID, From: from, To: target, At: now,
    })
    return nil
}

// Public methods expose business operations, not raw transitions
func (o *Order) Confirm(now time.Time) error { return o.transitionTo(StatusConfirmed, now) }
func (o *Order) Cancel(now time.Time) error  { return o.transitionTo(StatusCancelled, now) }
```

## Value Object

Immutable, no identity, equality by value. Use smallest monetary unit (cents) to avoid floating-point precision issues.

```go
type Money struct {
    Amount   int64  // Smallest unit (cents)
    Currency string // ISO 4217
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, ErrCurrencyMismatch
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

func (m Money) Multiply(quantity int) Money {
    return Money{Amount: m.Amount * int64(quantity), Currency: m.Currency}
}
```

## Type-Safe Enums with Enumer

Use [enumer](https://github.com/dmarkham/enumer) to generate type-safe enum methods with JSON/SQL serialization support.

### Installation

```bash
go install github.com/dmarkham/enumer@latest
```

### Enum Definition

```go
// internal/domain/valueobject/order_status.go
package valueobject

//go:generate enumer -type=OrderStatus -trimprefix=OrderStatus -json -sql -transform=snake
type OrderStatus int32

const (
    OrderStatusPending   OrderStatus = iota // 0
    OrderStatusConfirmed                    // 1
    OrderStatusPaid                         // 2
    OrderStatusShipped                      // 3
    OrderStatusCancelled                    // 4
)
```

### Generated Methods

After running `go generate ./...`, enumer creates:

```go
// Auto-generated in order_status_enumer.go
func (i OrderStatus) String() string           // "pending", "confirmed", etc.
func OrderStatusString(s string) (OrderStatus, error)  // Parse from string
func (i OrderStatus) MarshalJSON() ([]byte, error)     // JSON serialization
func (i *OrderStatus) UnmarshalJSON(data []byte) error // JSON deserialization
func (i OrderStatus) Value() (driver.Value, error)     // SQL driver support
func (i *OrderStatus) Scan(value interface{}) error    // SQL driver support
```

### Common Enumer Flags

| Flag | Purpose | Example Output |
|------|---------|----------------|
| `-json` | JSON marshal/unmarshal | `"pending"` |
| `-sql` | SQL Value/Scan methods | Direct DB storage |
| `-text` | TextMarshaler/Unmarshaler | Config files |
| `-yaml` | YAML support | K8s configs |
| `-trimprefix=X` | Remove prefix from string | `OrderStatusPending` → `"pending"` |
| `-transform=snake` | String format | `"order_pending"` |
| `-transform=kebab` | String format | `"order-pending"` |

### Usage in sqlc

With `-sql` flag, enums work directly with sqlc-generated code:

```sql
-- queries/order.sql
-- name: UpdateOrderStatus :exec
UPDATE orders SET status = $2 WHERE id = $1;
```

```go
// Repository uses enum directly
func (r *repo) UpdateStatus(ctx context.Context, id uuid.UUID, status valueobject.OrderStatus) error {
    return r.q.UpdateOrderStatus(ctx, id, status)  // enum auto-converts via Value()
}
```

### Best Practices

1. **First value = zero/unknown**: Start with `Unspecified` or `Unknown` as `iota` value 0
2. **Prefix convention**: Use type name as prefix (`OrderStatusPending`), trim with `-trimprefix`
3. **Location**: Place enums in `domain/valueobject/` (they are value objects)
4. **Generate on CI**: Include `go generate ./...` in CI pipeline

## Repository Interface

Domain layer defines interface only. Implementation lives in Adapter layer.
Transaction management (`WithTx`) is a persistence detail — not in Domain layer. Handled by Application layer's TxManager.

```go
// internal/domain/repository/order_repository.go
type OrderRepository interface {
    // Create writes DB-generated ID, CreatedAt back to Entity (pointer = allows mutation, Go convention)
    Create(ctx context.Context, order *entity.Order) error
    GetByID(ctx context.Context, id uuid.UUID) (*entity.Order, error)
    // Update increments version via optimistic lock SQL, writes new Version back to Entity
    Update(ctx context.Context, order *entity.Order) error
    Delete(ctx context.Context, id uuid.UUID) error
}
```

### Repository Implementation Pattern (Adapter Layer)

```go
// internal/adapter/outbound/persistence/postgres/order_repository.go
type OrderRepository struct {
    pool *pgxpool.Pool
}

func (r *OrderRepository) Create(ctx context.Context, order *entity.Order) error {
    q := sqlcgen.New(database.GetDBTX(ctx, r.pool))  // Dynamically uses TX or pool
    row, err := q.CreateOrder(ctx, sqlcgen.CreateOrderParams{
        UserID: order.UserID, Status: string(order.Status),
        Amount: order.TotalAmount.Amount, Currency: order.TotalAmount.Currency,
    })
    if err != nil { return err }
    order.ID = row.ID
    order.CreatedAt = row.CreatedAt
    order.UpdatedAt = row.UpdatedAt
    return nil
}

func (r *OrderRepository) GetByID(ctx context.Context, id uuid.UUID) (*entity.Order, error) {
    q := sqlcgen.New(database.GetDBTX(ctx, r.pool))
    row, err := q.GetOrderByID(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, domain.ErrOrderNotFound  // Map to Domain Error
        }
        return nil, err
    }
    return toDomainOrder(row), nil
}

func (r *OrderRepository) Update(ctx context.Context, order *entity.Order) error {
    q := sqlcgen.New(database.GetDBTX(ctx, r.pool))
    result, err := q.UpdateOrderStatus(ctx, sqlcgen.UpdateOrderStatusParams{
        ID: order.ID, Status: string(order.Status), Version: int32(order.Version),
    })
    if err != nil { return err }
    if result.RowsAffected() == 0 {
        return domain.ErrOptimisticLock  // Optimistic lock conflict
    }
    order.Version++
    return nil
}

// sqlcgen struct → Domain Entity (thin one-way mapping)
func toDomainOrder(row *sqlcgen.Order) *entity.Order {
    return &entity.Order{
        ID: row.ID, UserID: row.UserID,
        Status:      entity.OrderStatus(row.Status),
        TotalAmount: valueobject.Money{Amount: row.Amount, Currency: row.Currency},
        Version: int(row.Version), CreatedAt: row.CreatedAt, UpdatedAt: row.UpdatedAt,
    }
}
```

### Pessimistic Locking (SELECT FOR UPDATE)

Use when optimistic lock retry cost is too high or exclusive access is required:

```sql
-- name: GetOrderForUpdate :one
SELECT * FROM orders WHERE id = $1 FOR UPDATE;
```

Must be used within a TX (`TxManager.WithTx`). Holds row lock until TX commits/rolls back. Prefer optimistic locking as default; use pessimistic only for high-contention critical sections.

## Domain Event

Domain Events are collected by Entity during state transitions. UseCase extracts them and writes to Outbox.

```go
// internal/domain/event/event.go
type DomainEvent interface {
    EventType() string  // e.g., "order.status_changed"
    Version() int       // Event schema version for progressive consumer migration
}

type OrderStatusChanged struct {
    OrderID uuid.UUID
    From    OrderStatus
    To      OrderStatus
    At      time.Time
}
func (e OrderStatusChanged) EventType() string { return "order.status_changed" }
func (e OrderStatusChanged) Version() int      { return 1 }
```

### Event Lifecycle Rules

1. Entity calls `AddEvent()` during state transition
2. UseCase writes `entity.Events()` to Outbox within TX
3. `entity.ClearEvents()` is called ONLY after TX commit succeeds
4. If TX rolls back, events remain on Entity — no inconsistent state

### Event Versioning Strategy

Every Domain Event carries a `Version()`. When event schema changes:

1. Increment the version number in the new event struct
2. Outbox stores `event_version` alongside the payload
3. Consumer reads `event_version` from MQ headers and handles accordingly:
   - Known version → process normally
   - Unknown future version → log warning, skip or apply best-effort handling
4. Maintain backward compatibility: add fields, don't remove or rename

This enables progressive migration — old consumers keep working while new ones handle enhanced payloads.

## Domain Service

Use when business logic spans multiple Entities or Value Objects and doesn't belong to any single Entity. Domain Service has zero external dependencies.

```go
// internal/domain/service/pricing_service.go
type PricingService struct{}

func (s *PricingService) CalculateTotal(items []entity.OrderItem, discounts []valueobject.Discount) (valueobject.Money, error) {
    // Pure business logic spanning multiple domain objects
}
```

## Domain Layer Constraints

- **Zero external dependencies**: No imports from Application, Adapter, or Infrastructure layers
- **No `samber/lo`**: Domain layer must not use any third-party utility libraries
- **No transport protocols**: Domain layer must not import `grpc/codes`, HTTP status codes, or any protocol-specific packages
- **No database concerns**: No SQL, no ORM tags, no connection management
- **Pure Go only**: Only standard library types + shared `pkg/errors` interface
