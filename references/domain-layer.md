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
| **Domain** | `Entity` | `domain/{entity}.go` | Pure business logic, no DB tags |
| **Domain** | `ValueObject` | `domain/valueobject/` | Immutable, equality by value |
| **Domain** | `Enum` | `domain/{entity}_types.go` | Type-safe enums (co-located with Entity) |
| **Domain** | `Repository Interface` | `domain/{entity}.go` (bottom) | Interface only, co-located with Entity |
| **Application** | `Request DTO` | `usecase/dto/{feature}_req.go` | UseCase input |
| **Application** | `Response DTO` | `usecase/dto/{feature}_res.go` | UseCase output |
| **Repository** | `sqlc struct` | `repository/postgres/gen/` | Auto-generated (DO NOT EDIT) |

### Data Flow

```
gRPC Request → Handler → Request DTO → UseCase → Entity ← Repository ← gen.Model
                                           ↓
                             Response DTO ← UseCase
                                           ↓
                              Handler → gRPC Response
```

### Mapping Responsibilities

| From | To | Where |
|------|----|-------|
| `pb.Request` | `dto.XxxRequest` | gRPC Handler (`mapper.go`) |
| `dto.XxxRequest` | `domain.Entity` | UseCase |
| `gen.Model` | `domain.Entity` | Repository (`mapper.go` → `toDomain()`) |
| `domain.Entity` | `gen.XxxParams` | Repository (`mapper.go` → `toCreateParams()`) |
| `domain.Entity` | `dto.XxxResponse` | UseCase |
| `dto.XxxResponse` | `pb.Response` | gRPC Handler (`mapper.go`) |

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
    OrderStatusPending:   {OrderStatusConfirmed, OrderStatusCancelled},
    OrderStatusConfirmed: {OrderStatusPaid, OrderStatusCancelled},
    OrderStatusPaid:      {OrderStatusShipped, OrderStatusCancelled},
}

// Entity (package entity) imports parent package domain for domain errors.
// No circular dependency: domain/errors.go has zero imports from entity.
func (o *Order) canTransitionTo(target OrderStatus) error {
    allowed, ok := validTransitions[o.Status]
    if !ok {
        return domain.ErrInvalidTransition
    }
    for _, s := range allowed {
        if s == target { return nil }
    }
    return domain.ErrInvalidTransition
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
func (o *Order) Confirm(now time.Time) error { return o.transitionTo(OrderStatusConfirmed, now) }
func (o *Order) Cancel(now time.Time) error  { return o.transitionTo(OrderStatusCancelled, now) }
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

Use [enumer](https://github.com/dmarkham/enumer) to generate type-safe enum methods. Enums are **co-located with their Entity** in `domain/{entity}_types.go`, keeping high cohesion.

### Installation

```bash
go install github.com/dmarkham/enumer@latest
```

### Directory Layout

```text
internal/domain/
├── order.go                          # Entity + Repository Interface + rich methods
├── order_types.go                    # Order-related enums (hand-written)
├── zzz_enumer_orderStatus.go         # Generated — sorts to bottom
├── zzz_enumer_paymentMethod.go       # Generated — sorts to bottom
├── merchant.go                       # Another entity
├── merchant_types.go                 # Merchant-related enums
├── zzz_enumer_merchantType.go        # Generated
├── valueobject/                      # Value Objects
├── service/                          # Domain Services
└── errors.go                         # Domain errors
```

**Convention**:
- Enum definitions in `{entity}_types.go` (co-located with Entity, same `domain` package)
- Generated files use `zzz_` prefix so they sort to the bottom of file explorers
- Repository Interface at the bottom of `{entity}.go`

### Enum Definition

```go
// internal/domain/order_types.go
package domain

//go:generate enumer -type=OrderStatus -trimprefix=OrderStatus -json -text -transform=snake --output=zzz_enumer_orderStatus.go
type OrderStatus int32

const (
    OrderStatusUnspecified OrderStatus = iota // 0 — proto3 default, distinguishes "not set"
    OrderStatusPending                        // 1
    OrderStatusConfirmed                      // 2
    OrderStatusPaid                           // 3
    OrderStatusShipped                        // 4
    OrderStatusCancelled                      // 5
)

//go:generate enumer -type=PaymentMethod -trimprefix=PaymentMethod -json -text -transform=snake --output=zzz_enumer_paymentMethod.go
type PaymentMethod int32

const (
    PaymentMethodUnspecified PaymentMethod = iota
    PaymentMethodCreditCard
    PaymentMethodBankTransfer
)
```

**Key points**:

- Enumer only supports `int`-based enums (iota), NOT string-based
- Group related enums in `{entity}_types.go` by domain concept (e.g., `order_types.go` has `OrderStatus` + `PaymentMethod`)
- Use `--output=zzz_enumer_<typeName>.go` for `zzz_` prefix convention
- Always include `-text` flag (needed for `encoding.TextMarshaler/TextUnmarshaler`)

### Generated Methods

After running `go generate ./...`, enumer creates:

```go
// Auto-generated in zzz_enumer_orderStatus.go
func (i OrderStatus) String() string                   // "pending", "confirmed", etc.
func OrderStatusString(s string) (OrderStatus, error)  // Parse from string
func (i OrderStatus) IsAOrderStatus() bool             // Validity check
func (i OrderStatus) MarshalJSON() ([]byte, error)     // JSON serialization
func (i *OrderStatus) UnmarshalJSON(data []byte) error // JSON deserialization
func (i OrderStatus) MarshalText() ([]byte, error)     // Text serialization
func (i *OrderStatus) UnmarshalText(data []byte) error // Text deserialization
```

### Common Enumer Flags

| Flag | Purpose | Example Output |
|------|---------|----------------|
| `-json` | JSON marshal/unmarshal | `"pending"` |
| `-text` | TextMarshaler/Unmarshaler | Standard serialization |
| `-sql` | SQL Value/Scan methods | Direct DB storage |
| `-yaml` | YAML support | K8s configs |
| `-trimprefix=X` | Remove prefix from string | `OrderStatusPending` → `"pending"` |
| `-transform=snake` | String format | `"order_pending"` |
| `-transform=lower` | String format | `"orderpending"` |
| `-transform=title-lower` | String format | `"orderPending"` |
| `--output=FILE` | Custom output filename | `zzz_enumer_orderStatus.go` |

### Enum vs Value Object

| Aspect | Enum (`domain/{entity}_types.go`) | Value Object (`domain/valueobject/`) |
| ------ | --------------------- | ------------------------------------- |
| Behavior | Pure label, no logic | Rich behavior (validation, comparison, arithmetic) |
| Generation | `enumer` auto-generated methods | Hand-written methods |
| Example | `OrderStatus`, `MerchantType` | `Phone` (validation), `Role` (bitmask), `Money` (arithmetic) |
| Location | Same `domain` package, in `*_types.go` | Sub-package `domain/valueobject/` |

**Rule of thumb**: If the type only has named constants and serialization → `*_types.go`. If it has validation, arithmetic, or complex equality → `valueobject/`.

### Usage in Repository (without -sql flag)

When using sqlc with `sqlutil` helpers (storing enums as strings in DB), parse in `toDomain()` inside `mapper.go`:

```go
// internal/repository/postgres/mapper.go
func toDomainOrder(row gen.Order) (*domain.Order, error) {
    status, err := domain.OrderStatusString(sqlutil.TextValue(row.Status))
    if err != nil {
        return nil, fmt.Errorf("invalid order status: %w", err)
    }
    return &domain.Order{
        Status: status,
        // ...
    }, nil
}
```

### Usage in Repository (with -sql flag)

With `-sql` flag, enums work directly with sqlc-generated code:

```go
func (r *OrderRepository) UpdateStatus(ctx context.Context, id uuid.UUID, status domain.OrderStatus) error {
    return r.q.UpdateOrderStatus(ctx, id, status)  // enum auto-converts via Value()
}
```

### Best Practices

1. **Co-located with Entity**: Place enums in `domain/{entity}_types.go`, same `domain` package as Entity
2. **Group by entity**: Related enums share one `_types.go` file (e.g., `order_types.go` has `OrderStatus` + `PaymentMethod`)
3. **`zzz_` prefix**: Always use `--output=zzz_enumer_<typeName>.go` so generated files sort to bottom
4. **First value = zero/unknown**: Start with `Unspecified` or `Unknown` as `iota` value 0
5. **Prefix convention**: Use type name as prefix (`OrderStatusPending`), trim with `-trimprefix`
6. **Standard flags**: Always include `-json -text`; add `-sql` only if DB column uses the enum type directly
7. **Generate on CI**: Include `go generate ./...` in CI pipeline
8. **Int-based only**: Enumer requires `int`-based types with `iota` — do NOT use `string` constants

## Repository Interface

Domain layer defines interface only, **co-located at the bottom of the Entity file**. Implementation lives in Repository layer (`repository/postgres/`).
Transaction management (`WithTx`) is a persistence detail — not in Domain layer. Handled by Application layer's TxManager.

```go
// internal/domain/order.go
package domain

import (
    "context"
    "github.com/google/uuid"
)

// ==================== Entity ====================

type Order struct {
    ID          uuid.UUID
    UserID      uuid.UUID
    Status      OrderStatus
    TotalAmount valueobject.Money
    Version     int
    CreatedAt   time.Time
    UpdatedAt   time.Time
    events      []DomainEvent
}

// ... Entity methods (Confirm, Cancel, etc.)

// ==================== Repository Interface ====================

// OrderRepository defines the persistence interface for orders.
// Co-located with Entity for high cohesion (Go convention: interface near its domain).
type OrderRepository interface {
    // Create writes DB-generated ID, CreatedAt back to Entity (pointer = allows mutation, Go convention)
    Create(ctx context.Context, order *Order) error
    GetByID(ctx context.Context, id uuid.UUID) (*Order, error)
    // Update increments version via optimistic lock SQL, writes new Version back to Entity
    Update(ctx context.Context, order *Order) error
    Delete(ctx context.Context, id uuid.UUID) error
}
```

**Why co-located**: If Repository Interfaces grow beyond 5+ interfaces in a single `repository.go`, they bloat. Placing each interface at the bottom of its Entity file keeps high cohesion and avoids a single file growing too large.

### Repository Implementation

See [data-layer.md — Repository Implementation Pattern](data-layer.md#repository-implementation-pattern) for the canonical implementation with `sqlutil` helpers, complete import block, and `toDomain()` mapping.

### Pessimistic Locking (SELECT FOR UPDATE)

Use when optimistic lock retry cost is too high or exclusive access is required:

```sql
-- name: GetOrderForUpdate :one
SELECT * FROM orders WHERE id = $1 FOR UPDATE;
```

Must be used within a TX (`TxManager.WithTx`). Holds row lock until TX commits/rolls back. Prefer optimistic locking as default; use pessimistic only for high-contention critical sections.

### Optimistic Lock Retry Pattern

When optimistic lock conflicts occur, UseCase should reload entity and retry:

```go
// internal/application/usecase/confirm_order.go
func (uc *ConfirmOrderUseCase) Execute(ctx context.Context, orderID uuid.UUID) error {
    const maxRetries = 3
    var lastErr error

    for attempt := 0; attempt < maxRetries; attempt++ {
        // 1. Always reload fresh entity from DB
        order, err := uc.orderRepo.GetByID(ctx, orderID)
        if err != nil {
            return err  // includes domain.ErrOrderNotFound
        }

        // 2. Apply business logic (state transition)
        if err := order.Confirm(time.Now()); err != nil {
            return err  // Business rule error — no retry
        }

        // 3. Attempt to persist with optimistic lock
        if err := uc.orderRepo.Update(ctx, order); err != nil {
            if errors.Is(err, domain.ErrOptimisticLock) {
                lastErr = err
                continue  // Retry: reload and try again
            }
            return err  // Other DB error — no retry
        }

        return nil  // Success
    }

    return fmt.Errorf("optimistic lock conflict after %d retries: %w", maxRetries, lastErr)
}
```

**Key points**:

- Always reload entity before each retry (stale data causes repeated conflicts)
- Only retry on `ErrOptimisticLock`, not on business rule errors
- Limit retries to prevent infinite loops under high contention
- Consider pessimistic locking if retry rate > 5%

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
    From    entity.OrderStatus
    To      entity.OrderStatus
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

### Event Versioning Strategy `[Async]`

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
