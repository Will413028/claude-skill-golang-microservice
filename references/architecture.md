# Architecture & Directory Structure

## Table of Contents

- [Single Service Directory Structure](#single-service-directory-structure)
  - [Application Layer: Service + UseCase 分層](#application-layer-service--usecase-分層)
  - [DTO Organization Pattern](#dto-organization-pattern)
- [Monorepo Structure](#monorepo-structure)
- [Shared Packages (pkg)](#shared-packages-pkg)
- [Naming Conventions](#naming-conventions)
- [Monorepo Scaling Strategy `[Infrastructure]`](#monorepo-scaling-strategy-infrastructure)
- [Uber Fx Dependency Injection](#uber-fx-dependency-injection)
- [Proto / buf Tooling](#proto--buf-tooling)
- [Local Development Environment](#local-development-environment)
- [Scheduled Jobs](#scheduled-jobs)

## Single Service Directory Structure

```
services/xxx-service/
├── cmd/
│   └── server/
│       └── main.go                     # Entry point (Main Injector)
│
├── internal/
│   ├── domain/                         # Domain Layer (core, zero external deps)
│   │   ├── order.go                    # Entity + Repository Interface + rich methods
│   │   ├── order_types.go              # Type-safe enums (enumer generated, co-located with Entity)
│   │   ├── order_event.go              # Domain Events for this aggregate
│   │   ├── valueobject/                # Value Object (immutable, with behavior logic)
│   │   ├── service/                    # Domain Service (cross-entity business logic, zero deps)
│   │   └── errors.go                   # Domain error definitions
│   │
│   ├── usecase/                        # Application Layer: business flow orchestration
│   │   ├── create_order.go             # UseCase implementation
│   │   ├── cancel_order.go
│   │   ├── dto/                        # Data Transfer Objects
│   │   │   ├── order_req.go
│   │   │   └── order_res.go
│   │   └── di.go                       # fx.Module (UseCase layer DI)
│   │
│   ├── service/                        # Application Service: reusable cross-UseCase logic
│   │   ├── address_service.go
│   │   ├── points_service.go
│   │   └── di.go                       # fx.Module (Service layer DI)
│   │
│   ├── repository/                     # Data access (Outbound Adapter)
│   │   ├── postgres/                   # Explicit technology naming
│   │   │   ├── gen/                    # 🤖 sqlc auto-generated (DO NOT EDIT)
│   │   │   │   ├── models.go           # DB Models (maps to table schema)
│   │   │   │   ├── query.sql.go        # DB Methods (auto-generated)
│   │   │   │   └── db.go              # DBTX Interface
│   │   │   ├── order.go                # Implements domain.OrderRepository
│   │   │   ├── mapper.go              # gen.Model ↔ domain.Entity conversion
│   │   │   └── di.go                   # fx.Module (Postgres Repository DI)
│   │   └── di.go                       # fx.Module (Repository module entry)
│   │
│   ├── client/                         # External service clients (Outbound Adapter)
│   │   ├── payment/
│   │   │   ├── client.go               # Payment API client
│   │   │   ├── dto.go                  # Request/Response types
│   │   │   └── mapper.go              # domain ↔ client DTO conversion
│   │   ├── inventory/
│   │   │   └── client.go               # Inventory gRPC client
│   │   └── di.go                       # fx.Module (Client layer DI)
│   │
│   ├── grpc/                           # gRPC interface (Inbound Adapter)
│   │   ├── server.go                   # gRPC Server setup
│   │   ├── handler.go                 # gRPC Handler (calls UseCase)
│   │   ├── mapper.go                  # Protobuf ↔ DTO conversion
│   │   └── di.go                       # fx.Module (gRPC layer DI)
│   │
│   ├── consumer/                       # MQ Consumer (Inbound Adapter, added in Async stage)
│   │   ├── order_consumer.go
│   │   └── di.go
│   │
│   ├── config/                         # Configuration
│   │   ├── config.go
│   │   └── di.go                       # fx.Module (Config DI)
│   │
│   └── app/                            # Application assembly
│       └── app.go                      # fx.New() — assembles all modules
│
├── db/                                 # Database-related (centralized)
│   ├── schema/schema.sql               # DDL (single source of truth)
│   ├── queries/                        # sqlc query definitions
│   │   ├── order.sql
│   │   └── outbox.sql
│   ├── migrations/                     # Atlas auto-generated migrations
│   ├── sqlc.yaml                       # sqlc configuration
│   └── atlas.hcl                       # Atlas configuration
│
├── tests/
├── scripts/
├── Makefile
└── Dockerfile
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Enum location** | `domain/*_types.go` (co-located with Entity) | High cohesion — Enum, Entity, and Repository Interface in same package |
| **Repository Interface** | Bottom of `domain/{entity}.go` | Go convention: define interface near the domain it serves; avoids separate `repository/` package bloat |
| **sqlc output** | `repository/postgres/gen/` | Named `gen/` (not `dao/`) to clearly indicate auto-generated code |
| **Flat directory** | No `adapter/inbound/outbound/` nesting | Go-style flat structure; name by technology (`grpc/`, `postgres/`, `client/`) not by direction |
| **DI per-layer** | Each package has `di.go` with `fx.Module` | Modular, self-contained; `app.go` only assembles modules |

> **Database Directory Convention**: All database-related files (`schema/`, `queries/`, `migrations/`, `sqlc.yaml`, `atlas.hcl`) are centralized in the `db/` directory to keep the service root clean. Run commands from within `db/`:
>
> ```bash
> cd db && sqlc generate           # Generate Go code to internal/repository/postgres/gen/
> cd db && atlas migrate diff ...  # Generate migration
> ```

### Application Layer: Service + UseCase 分層

Application 層採用 **Service + UseCase 分層架構**，分離可重用邏輯與業務流程編排：

```
internal/
├── service/                    # Application Service（可重用邏輯）
│   ├── address_service.go
│   ├── points_service.go
│   └── di.go                  # fx.Module
│
├── usecase/                    # UseCase（業務流程編排）
│   ├── checkout_usecase.go
│   ├── certification_usecase.go
│   ├── dto/                   # Data Transfer Objects
│   │   ├── checkout_req.go
│   │   └── checkout_res.go
│   └── di.go                  # fx.Module
```

| Layer | Location | Purpose | 方法數 |
|-------|----------|---------|--------|
| **Application Service** | `internal/service/` | 可重用的基礎操作，被多個 UseCase 共用 | 多個相關方法 |
| **UseCase** | `internal/usecase/` | 業務流程編排，組合多個 Services | 1-3 個公開方法 |
| **Domain Service** | `internal/domain/service/` | 純業務邏輯，跨多個 Entity，零外部依賴 | 依需求 |

### Application Service 設計

Service 封裝**可重用的基礎操作**，每個 Service 對應一個 Aggregate：

```go
// internal/service/address_service.go
type AddressService interface {
    Get(ctx context.Context, id uuid.UUID) (*dto.Address, error)
    List(ctx context.Context, accountID uuid.UUID) ([]*dto.Address, error)
    Create(ctx context.Context, req *dto.CreateAddressRequest) error
    Update(ctx context.Context, req *dto.UpdateAddressRequest) error
    Delete(ctx context.Context, id uuid.UUID) error
    SetDefault(ctx context.Context, accountID, addressID uuid.UUID) error
}

type addressService struct {
    addressRepo domain.AddressRepository  // 依賴 domain 層的 Repository 介面
    logger      *zap.Logger
}

func NewAddressService(repo domain.AddressRepository, logger *zap.Logger) AddressService {
    return &addressService{addressRepo: repo, logger: logger}
}
```

### UseCase 設計

UseCase 負責**業務流程編排**，組合多個 Services 完成完整流程：

```go
// internal/usecase/checkout_usecase.go
type CheckoutUseCase struct {
    addressSvc  service.AddressService
    pointsSvc   service.PointsService
    orderClient orderClient               // local interface (Go consumer-defined)
    txManager   txManager                  // local interface
    logger      *zap.Logger
}

// local interfaces — 消費者定義介面 (Go idiom)
// Only declare methods that this UseCase actually uses
type orderClient interface {
    Create(ctx context.Context, req *CreateOrderRequest) (*CreateOrderResponse, error)
}

type txManager interface {
    WithTx(ctx context.Context, fn func(ctx context.Context) error) error
}

func NewCheckoutUseCase(
    addressSvc service.AddressService,
    pointsSvc service.PointsService,
    orderClient orderClient,
    txManager txManager,
    logger *zap.Logger,
) *CheckoutUseCase {
    return &CheckoutUseCase{
        addressSvc:  addressSvc,
        pointsSvc:   pointsSvc,
        orderClient: orderClient,
        txManager:   txManager,
        logger:      logger,
    }
}

func (uc *CheckoutUseCase) Execute(ctx context.Context, req *dto.CheckoutRequest) (*dto.CheckoutResponse, error) {
    addr, err := uc.addressSvc.Get(ctx, req.AddressID)
    if err != nil { return nil, err }

    if req.UsePoints > 0 {
        if err := uc.pointsSvc.Deduct(ctx, req.AccountID, req.UsePoints); err != nil {
            return nil, err
        }
    }

    order, err := uc.orderClient.Create(ctx, &CreateOrderRequest{...})
    if err != nil { return nil, err }

    return &dto.CheckoutResponse{OrderID: order.ID}, nil
}
```

### Service vs UseCase 判斷規則

| 情境 | 放哪裡 | 範例 |
|------|--------|------|
| 單一 Aggregate 的 CRUD | **Service** | `AddressService.Create/Update/Delete` |
| 可被多個 UseCase 重用的邏輯 | **Service** | `PointsService.GetBalance` |
| 跨多個 Service 的流程編排 | **UseCase** | `CheckoutUseCase`（地址+積分+訂單）|
| 涉及外部服務呼叫 | **UseCase** | `GoogleOAuthUseCase`（呼叫 Google API）|
| 複雜的狀態機流程 | **UseCase** | `CertificationUseCase`（認證審核流程）|

### 簡單 Service 可省略 UseCase

如果業務邏輯簡單（純 CRUD，無跨服務流程），gRPC Handler 可直接呼叫 Service：

```go
// internal/grpc/address_handler.go
type AddressHandler struct {
    addressSvc service.AddressService  // 直接依賴 Service
}

func (h *AddressHandler) ListAddresses(ctx context.Context, req *pb.ListAddressesRequest) (*pb.ListAddressesResponse, error) {
    addresses, err := h.addressSvc.List(ctx, req.AccountId)
    // ...
}
```

**Rule**: 只有當需要**組合多個 Services** 或**複雜流程編排**時，才建立 UseCase。

### DTO Organization Pattern

DTOs live inside `usecase/dto/`, organized **by feature**, with each file containing paired request + response types.

```
usecase/dto/
├── merchant_req.go          # CreateMerchantRequest, UpdateProfileRequest
├── merchant_res.go          # MerchantResponse, MerchantProfile
├── certification_req.go     # CreateDraftRequest, SubmitApplicationRequest
├── certification_res.go     # CertificationResponse
├── crm_req.go               # ListCustomersRequest, UpdateCustomerStatusRequest
├── crm_res.go               # CustomerResponse, TagResponse
└── common.go                # PaginatedResult[T], shared types
```

**Organization Rules**:

| Principle | Guideline |
|-----------|-----------|
| Paired req/res files | `merchant_req.go` pairs with `merchant_res.go` |
| One pair per feature | Group related request/response types by the UseCase they serve |
| Shared generics | Place `PaginatedResult[T]` in `common.go` |
| No domain leakage | DTOs are flat data structures, never reference Domain entities directly |

**File Naming Convention**: `{feature}_req.go` + `{feature}_res.go`

## Monorepo Structure

```
project-root/
├── api/                                # API definitions
│   └── proto/                          # Protocol Buffers source files
│       ├── account/account.proto
│       ├── merchant/merchant.proto
│       └── common/{pagination,money}.proto
│
├── pkg/                                # Shared Go packages (single go.mod)
│   ├── go.mod                          # module github.com/yourproject/go-pkg
│   ├── proto/                          # Generated proto Go code
│   │   ├── account/                    # github.com/yourproject/go-pkg/proto/account
│   │   ├── merchant/
│   │   └── ...
│   ├── config/                         # Native os.Getenv + struct
│   ├── logger/                         # Zap + Log Schema
│   ├── errors/                         # ErrorCode + DomainError interface
│   ├── database/                       # PG connection + GetDBTX
│   ├── middleware/                     # gRPC Interceptors
│   ├── mq/                             # RabbitMQ connection + trace propagation
│   ├── redis/                          # Redis client + idempotency
│   ├── cache/                          # Cache + singleflight
│   └── circuitbreaker/                 # Circuit breaker
│
├── services/                           # Individual microservices
├── gateway/                            # API Gateway
├── scripts/                            # Build/deploy scripts
├── deploy/                             # K8s manifests, docker-compose
└── Makefile                            # Root-level commands
```

> **Proto Convention**: Proto source files live in `api/proto/`. Generated Go code lives in `pkg/proto/`.
> Services import via `github.com/yourproject/go-pkg/proto/<domain>` and use `replace` directive for local development:
> ```go
> // services/xxx-service/go.mod
> replace github.com/yourproject/go-pkg => ../../pkg
> ```

## Shared Packages (pkg)

| Package | Responsibility | Stage |
|---------|---------------|-------|
| `config` | `os.Getenv` + struct config | MVP |
| `logger` | Zap + Log Schema | MVP |
| `ctxutil` | correlation_id / request_id propagation | MVP |
| `errors` | ErrorCode + DomainError interface (contract) | MVP |
| `database` | PG connection pool + `GetDBTX` helper | MVP |
| `sqlutil` | pgtype nullable type helpers (Text, Int4, Timestamptz, etc.) | MVP |
| `mapper` | Manual mapping utilities | MVP |
| `middleware/grpc/interceptor` | gRPC Interceptor chain | MVP |
| `observability` | OTel tracing setup | MVP |
| `auth/jwt` | JWT validation | MVP |
| `cache` | Generic CacheLoader + singleflight | Async |
| `circuitbreaker` | gobreaker wrapper | Async |
| `mq/rabbitmq` | MQ connection + trace propagation | Async |
| `outbox` | Two-phase Outbox Poller | Async |
| `saga` | Saga timeout monitor | Async |

**Why `ErrorCode` lives in `pkg/errors` instead of `internal/domain`**: Avoids circular dependency. `pkg/errors` defines the `ErrorCode` type and `DomainError` interface. Domain layer imports it to implement; Interceptor imports it to map. `pkg/errors` contains only pure constants and interfaces — no runtime or transport protocol dependencies.

## Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Package | lowercase single word | `usecase`, `entity` |
| File | lowercase underscore | `create_order.go` |
| Struct | PascalCase | `CreateOrderUseCase` |
| Interface | PascalCase | `OrderRepository` |
| DB Name | lowercase underscore | `order_db` |
| DB Table | lowercase underscore plural | `orders` |
| Proto | PascalCase message + snake_case fields | `CreateOrderRequest` |
| Repository Method | `Get*` for retrieval, `List*` for collections | `GetByID`, `ListByMerchant` |

### Repository Method Naming

| Operation | Convention | Return |
|-----------|-----------|--------|
| Single lookup | `GetByID`, `GetBy<Field>` | `(*Entity, error)` — returns `nil, domain.ErrXxxNotFound` if not found |
| List with filter | `List<Criteria>` | `([]*Entity, error)` |
| Count | `Count<Criteria>` | `(int64, error)` |
| Create | `Create` | `error` (ID populated on entity) |
| Update | `Update`, `Update<Aspect>` | `error` |
| Delete | `Delete` | `error` |

### External Client Structure

If a service calls external APIs (REST, SDKs, gRPC), organize by technology/service:

```
internal/client/                     # External service clients
├── payment/
│   ├── client.go                    # Implementation
│   ├── dto.go                       # Request/Response types
│   └── mapper.go                    # domain ↔ client DTO conversion
├── inventory/
│   └── client.go                    # gRPC client wrapper
└── di.go                            # fx.Module
```

UseCase defines local interfaces for external dependencies (Go consumer-defined interface pattern):

```go
// internal/usecase/create_order.go
type paymentClient interface {
    Charge(ctx context.Context, req *ChargeRequest) (*ChargeResponse, error)
}
```

## Monorepo Scaling Strategy `[Infrastructure]`

Initially all services share a single `go.mod`. When service count exceeds 5–8, evaluate:

1. **Go Workspace** (`go.work`): Each service gets independent `go.mod`, workspace unifies dev experience
2. **Independent shared package versioning**: Extract `pkg/` as independent module with semantic versioning

**Decision signals**: Frequent dependency conflicts, build times too long, different services need different versions of shared packages.

## Uber Fx Dependency Injection

Uber Fx wires all layers together. Each package contains its own `di.go` with a `var Module` (using `fx.Module` for named modules).

### Module Layout

```
internal/
├── usecase/
│   ├── create_order.go
│   ├── cancel_order.go
│   └── di.go                     # var Module = fx.Module("usecase", ...)
├── service/
│   ├── address_service.go
│   └── di.go                     # var Module = fx.Module("service", ...)
├── repository/
│   ├── postgres/
│   │   ├── order.go
│   │   └── di.go                 # var Module = fx.Module("repository.postgres", ...)
│   └── di.go                     # var Module (aggregates sub-modules)
├── client/
│   ├── payment/client.go
│   └── di.go                     # var Module = fx.Module("client", ...)
├── grpc/
│   ├── handler.go
│   └── di.go                     # var Module = fx.Module("grpc", ...)
├── config/
│   ├── config.go
│   └── di.go                     # var Module = fx.Module("config", ...)
└── app/
    └── app.go                    # Assembles all modules
```

### Package-Level `di.go`

Each package exposes a `var Module` using `fx.Module()` for named module grouping. This produces clean error messages from Fx when dependency resolution fails.

**Convention**:

- Use `var Module = fx.Module(...)` (not `func Module()`) for cleaner import syntax
- Constructor 用大寫 (`NewXxx`) — 方便測試直接呼叫
- Repository `di.go` uses `fx.Annotate` + `fx.As` to bind concrete type to domain interface
- UseCase `di.go` provides concrete types directly (gRPC handler depends on concrete UseCase)
- 新增 UseCase 只改該 package 的 `di.go`，減少 merge conflicts

### Fx Key Rules

| Concept | When to Use |
|---------|-------------|
| `fx.Module("name", ...)` | Named module grouping — gives clear error messages |
| `fx.Provide` | Constructors that return types for others to depend on |
| `fx.Invoke` | Side-effects (register handlers, start pollers) — runs at startup |
| `fx.As(new(Interface))` | Bind concrete type to interface (e.g., `*postgres.OrderRepository` → `domain.OrderRepository`) |
| `fx.Annotate` + `fx.ParamTags` | Disambiguate multiple implementations of the same interface |
| `fx.Lifecycle` | Register `OnStart` / `OnStop` hooks (server listen, graceful shutdown) |

### Complete `app.go` Example

```go
// internal/app/app.go
package app

import (
    "context"
    "go.uber.org/fx"

    "github.com/yourproject/order-service/internal/client"
    "github.com/yourproject/order-service/internal/config"
    "github.com/yourproject/order-service/internal/grpc"
    "github.com/yourproject/order-service/internal/repository"
    "github.com/yourproject/order-service/internal/service"
    "github.com/yourproject/order-service/internal/usecase"
)

// New creates the application (very clean!)
func New() *fx.App {
    return fx.New(
        // Layer 1: Configuration
        config.Module,

        // Layer 2: Data access + External clients
        repository.Module,
        client.Module,

        // Layer 3: Business logic
        service.Module,
        usecase.Module,

        // Layer 4: Interface (gRPC server)
        grpc.Module,

        // Start the application
        fx.Invoke(run),
    )
}

func run(lifecycle fx.Lifecycle, srv *grpc.Server) {
    lifecycle.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go srv.Start()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            return srv.Stop()
        },
    })
}
```

```go
// cmd/server/main.go
package main

import "github.com/yourproject/order-service/internal/app"

func main() {
    app.New().Run()
}
```

`fx.New().Run()` handles the full lifecycle: dependency injection → `OnStart` hooks → block on OS signal (SIGINT/SIGTERM) → `OnStop` hooks. No manual signal handling needed.

### Complete `di.go` Examples

**UseCase — Provide concrete types directly**:

```go
// internal/usecase/di.go
package usecase

import "go.uber.org/fx"

// Module UseCase 層的依賴注入模組
var Module = fx.Module("usecase",
    fx.Provide(
        NewCreateOrderUseCase,
        NewCancelOrderUseCase,
        NewListOrdersUseCase,
    ),
    // Background workers use fx.Invoke
    fx.Invoke(startSyncWorker),
)

func startSyncWorker(uc *SyncUseCase) {
    uc.StartWorker()
}
```

**Repository — Bind to domain interface with `fx.Annotate` + `fx.As`**:

```go
// internal/repository/postgres/di.go
package postgres

import (
    "context"
    "go.uber.org/fx"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/yourproject/order-service/internal/config"
    "github.com/yourproject/order-service/internal/domain"
)

// Module Postgres Repository 層
var Module = fx.Module("repository.postgres",
    // Database connection pool
    fx.Provide(NewDBPool),

    // Bind concrete repos to domain interfaces
    fx.Provide(
        fx.Annotate(NewOrderRepository, fx.As(new(domain.OrderRepository))),
        fx.Annotate(NewPaymentRepository, fx.As(new(domain.PaymentRepository))),
    ),
)

func NewDBPool(cfg *config.Config) (*pgxpool.Pool, error) {
    return pgxpool.New(context.Background(), cfg.Database.DSN())
}
```

```go
// internal/repository/di.go
package repository

import (
    "go.uber.org/fx"
    "github.com/yourproject/order-service/internal/repository/postgres"
)

// Module Repository 總模組（聚合子模組）
var Module = fx.Module("repository",
    postgres.Module,
    // redis.Module,  // 未來擴展
)
```

**gRPC Handler — Registration via `fx.Invoke`**:

```go
// internal/grpc/di.go
package grpc

import (
    "go.uber.org/fx"
    pb "github.com/yourproject/go-pkg/proto/order/v1"
    grpclib "google.golang.org/grpc"
)

var Module = fx.Module("grpc",
    fx.Provide(NewHandler),
    fx.Provide(NewServer),
    fx.Invoke(func(server *grpclib.Server, h *Handler) {
        pb.RegisterOrderServiceServer(server, h)
    }),
)
```

**Config — Simple provider**:

```go
// internal/config/di.go
package config

import "go.uber.org/fx"

var Module = fx.Module("config",
    fx.Provide(Load),
)
```

### Dependency Graph

```
app.go → fx.New()
  ├─ config.Module              → *config.Config
  ├─ repository.Module
  │   └─ postgres.Module        → *pgxpool.Pool, domain.OrderRepository (impl)
  ├─ client.Module              → *PaymentClient, *InventoryClient
  ├─ service.Module             → service.AddressService, service.PointsService
  ├─ usecase.Module             → *CreateOrderUseCase, *CancelOrderUseCase
  ├─ grpc.Module                → *Handler, *Server (+ fx.Invoke registers to grpc.Server)
  └─ fx.Invoke(run)             → Lifecycle hooks (start/stop server)
```

### Common Mistake: Circular Dependencies

Fx detects circular dependencies at startup with clear error messages. Fix by:
1. Extracting shared logic into a separate `fx.Provide`
2. Using `fx.Invoke` for side-effect-only registration (breaks the cycle)
3. Introducing an interface to invert the dependency direction

## Proto / buf Tooling

### Directory Structure

```
api/proto/
├── buf.yaml              # Module config (lint + breaking rules)
├── buf.gen.yaml          # Code generation config
├── common/v1/
│   ├── pagination.proto  # Shared pagination messages
│   └── money.proto       # Shared Money value object
├── order/v1/
│   └── order_service.proto
└── inventory/v1/
    └── inventory_service.proto
```

### buf.yaml

```yaml
version: v2
modules:
  - path: .
lint:
  use:
    - STANDARD               # Enforces Google API style guide
  except:
    - FIELD_NOT_REQUIRED      # Allow `required` keyword (proto3 optional)
    - PACKAGE_NO_IMPORT_CYCLE
breaking:
  use:
    - WIRE_JSON               # Detect wire-format breaking changes
```

### buf.gen.yaml

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen/go
    opt: paths=source_relative
```

### Proto Design Conventions

| Rule | Rationale |
|------|-----------|
| Package = `{service}.v1` | Versioned namespace, enables backward-compatible evolution |
| Service name = singular noun + `Service` | `OrderService`, not `OrdersService` |
| RPC naming = verb + noun | `CreateOrder`, `GetOrder`, `ListOrders` |
| Request/Response = RPC name + `Request`/`Response` | `CreateOrderRequest`, `CreateOrderResponse` |
| Field numbering: reserve 1-15 for frequent fields | Wire format uses 1 byte for tags 1-15, 2 bytes for 16+ |
| Never reuse or reassign field numbers | Breaking change — use `reserved` instead |
| Use `google.protobuf.Timestamp` for times | Don't use `int64` epoch or `string` ISO format |
| Enums: first value must be `_UNSPECIFIED = 0` | Proto3 default; enables distinguishing "not set" vs "explicitly set to first value" |

### Workflow

```bash
# Lint proto files
buf lint

# Check backward compatibility against main branch
buf breaking --against '.git#branch=main'

# Generate Go code
buf generate

# Verify generated code matches (CI)
buf generate --output /tmp/gen && diff -r gen/go /tmp/gen/go
```

### Shared Proto (common/v1/)

Shared messages like Pagination and Money live in `common/v1/`. Services import them:

```protobuf
import "common/v1/pagination.proto";

service OrderService {
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
}

message ListOrdersRequest {
  string user_id = 1;
  common.v1.CursorPaginationRequest pagination = 2;
}
```

## Local Development Environment

### Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./scripts/init-db.sh:/docker-entrypoint-initdb.d/init-db.sh  # Create per-service DBs
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:8-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  rabbitmq:
    image: rabbitmq:4-management-alpine
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASSWORD: guest
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 10s

  # Observability stack (optional, enable when needed)
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./deploy/otel/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    ports: ["4317:4317"]   # gRPC OTLP

  tempo:
    image: grafana/tempo:latest
    ports: ["3200:3200"]

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin

volumes:
  pg-data:
```

### init-db.sh (Per-Service Database Setup)

```bash
#!/bin/bash
set -e

# Create database and user for each service
create_service_db() {
    local service=$1
    local db="${service}_db"
    local user="${service}_svc"
    local password="${service}_password"  # Use secrets in production

    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE USER ${user} WITH PASSWORD '${password}';
        CREATE DATABASE ${db} OWNER ${user};
        REVOKE ALL ON DATABASE ${db} FROM PUBLIC;
        GRANT CONNECT ON DATABASE ${db} TO ${user};
EOSQL
}

create_service_db "order"
create_service_db "inventory"
create_service_db "wallet"
```

### Makefile Targets

```makefile
# Local development workflow
.PHONY: infra-up infra-down migrate-all generate test

infra-up:
	docker compose up -d postgres redis rabbitmq
	@echo "Waiting for services..."
	@sleep 3

infra-down:
	docker compose down

migrate-all:
	@for dir in services/*/; do \
		if [ -d "$$dir/db/migrations" ]; then \
			echo "Migrating $$(basename $$dir)..."; \
			atlas migrate apply --dir "file://$$dir/db/migrations" \
				--url "postgres://...$$(basename $$dir)_db?sslmode=disable"; \
		fi; \
	done

generate:
	buf generate
	@for dir in services/*/; do \
		if [ -f "$$dir/db/sqlc.yaml" ]; then \
			echo "sqlc generate $$(basename $$dir)..."; \
			(cd $$dir/db && sqlc generate); \
		fi; \
	done

test:
	go test ./... -race -cover -count=1

dev-%:  ## Run a specific service: make dev-order
	go run ./services/$*-service/cmd/server/main.go
```

### Development Workflow

1. `make infra-up` — Start PG + Redis + RabbitMQ
2. `make migrate-all` — Apply all migrations
3. `make generate` — Generate Proto + sqlc code
4. `make dev-order` — Run a specific service locally
5. Services connect to `localhost:5432`, `localhost:6379`, `localhost:5672`
6. For full observability stack: `docker compose --profile observability up -d`

## Scheduled Jobs

See [scheduled-jobs.md](scheduled-jobs.md) for complete scheduled job implementation including:

- Dual entry points (Cron + API)
- Job UseCase pattern
- Distributed lock (Redis)
- Job execution history (audit log)
- Monitoring & alerting
