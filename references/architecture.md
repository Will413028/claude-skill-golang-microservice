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
├── cmd/main.go                         # Entry point
│
├── internal/
│   ├── domain/                         # Domain Layer
│   │   ├── entity/                     # Entity + state machine
│   │   ├── valueobject/                # Value Object (immutable)
│   │   ├── repository/                 # Repository Interface (interface only)
│   │   ├── event/                      # Domain Event definitions (with EventType() + Version())
│   │   └── service/                    # Domain Service (cross-entity business logic, zero deps)
│   │
│   ├── application/
│   │   ├── usecase/                    # Business logic orchestration
│   │   ├── dto/{request,response}/     # Data Transfer Objects
│   │   ├── port/
│   │   │   ├── input/                  # Input Port (UseCase interfaces)
│   │   │   └── output/                 # Output Port (external dependency interfaces, incl. TxManager)
│   │   └── service/                    # Application Service (reusable cross-UseCase logic)
│   │
│   ├── adapter/
│   │   ├── inbound/
│   │   │   ├── grpc/                   # gRPC Handler + Mapper
│   │   │   └── consumer/              # MQ Consumer (added in Async stage)
│   │   └── outbound/
│   │       ├── persistence/           # Repository implementation (sqlc)
│   │       ├── external/              # External service clients (non-gRPC: REST APIs, SDKs)
│   │       ├── grpcclient/            # gRPC Client implementation
│   │       └── publisher/             # MQ Publisher (added in Async stage)
│   │
│   └── infrastructure/
│       ├── config/config.go
│       ├── server/grpc_server.go
│       └── fx/                         # DI modules (Uber Fx)
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
├── sqlcgen/                            # sqlc auto-generated Go code
├── configs/
├── tests/
└── Dockerfile
```

> **Database Directory Convention**: All database-related files (`schema/`, `queries/`, `migrations/`, `sqlc.yaml`, `atlas.hcl`) are centralized in the `db/` directory to keep the service root clean. Run commands from within `db/`:
>
> ```bash
> cd db && sqlc generate           # Generate Go code
> cd db && atlas migrate diff ...  # Generate migration
> ```

### Application Layer: Service + UseCase 分層

Application 層採用 **Service + UseCase 分層架構**，分離可重用邏輯與業務流程編排：

```
application/
├── service/                    # Application Service（可重用邏輯）
│   ├── account_service.go      # AccountService: 帳號相關操作
│   ├── address_service.go      # AddressService: 地址 CRUD
│   └── points_service.go       # PointsService: 積分操作
│
├── usecase/                    # UseCase（業務流程編排）
│   ├── google_oauth.go         # GoogleOAuthUseCase: OAuth 登入流程
│   ├── checkout.go             # CheckoutUseCase: 結帳流程
│   └── certification.go        # CertificationUseCase: 認證審核流程
│
├── dto/{request,response}/     # Data Transfer Objects
└── port/output/                # Output Port interfaces
```

| Layer | Location | Purpose | 方法數 |
|-------|----------|---------|--------|
| **Application Service** | `application/service/` | 可重用的基礎操作，被多個 UseCase 共用 | 多個相關方法 |
| **UseCase** | `application/usecase/` | 業務流程編排，組合多個 Services | 1-3 個公開方法 |
| **Domain Service** | `domain/service/` | 純業務邏輯，跨多個 Entity，零外部依賴 | 依需求 |

### Application Service 設計

Service 封裝**可重用的基礎操作**，每個 Service 對應一個 Aggregate：

```go
// application/service/address_service.go
type AddressService interface {
    Get(ctx context.Context, id uuid.UUID) (*response.Address, error)
    List(ctx context.Context, accountID uuid.UUID) ([]*response.Address, error)
    Create(ctx context.Context, req *request.CreateAddress) error
    Update(ctx context.Context, req *request.UpdateAddress) error
    Delete(ctx context.Context, id uuid.UUID) error
    SetDefault(ctx context.Context, accountID, addressID uuid.UUID) error
}

type addressService struct {
    addressRepo repository.AddressRepository
    logger      *zap.Logger
}

func NewAddressService(repo repository.AddressRepository, logger *zap.Logger) AddressService {
    return &addressService{addressRepo: repo, logger: logger}
}
```

### UseCase 設計

UseCase 負責**業務流程編排**，組合多個 Services 完成完整流程：

```go
// application/usecase/checkout.go
type CheckoutUseCase interface {
    Execute(ctx context.Context, req *request.Checkout) (*response.CheckoutResult, error)
}

type checkoutUseCase struct {
    addressSvc  service.AddressService   // 依賴 Service，不是 Repository
    pointsSvc   service.PointsService
    orderClient output.OrderClient       // 外部服務 via Output Port
    txManager   output.TxManager
    logger      *zap.Logger
}

func NewCheckoutUseCase(
    addressSvc service.AddressService,
    pointsSvc service.PointsService,
    orderClient output.OrderClient,
    txManager output.TxManager,
    logger *zap.Logger,
) CheckoutUseCase {
    return &checkoutUseCase{
        addressSvc:  addressSvc,
        pointsSvc:   pointsSvc,
        orderClient: orderClient,
        txManager:   txManager,
        logger:      logger,
    }
}

func (u *checkoutUseCase) Execute(ctx context.Context, req *request.Checkout) (*response.CheckoutResult, error) {
    addr, err := u.addressSvc.Get(ctx, req.AddressID)
    if err != nil { return nil, err }

    if req.UsePoints > 0 {
        if err := u.pointsSvc.Deduct(ctx, req.AccountID, req.UsePoints); err != nil {
            return nil, err
        }
    }

    order, err := u.orderClient.Create(ctx, &output.CreateOrderRequest{...})
    if err != nil { return nil, err }

    return &response.CheckoutResult{OrderID: order.ID}, nil
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

### Fx Module Wiring

Uses per-package `di.go` pattern (see [Uber Fx Dependency Injection](#uber-fx-dependency-injection) for details):

```go
// application/service/di.go
package service

func Module() fx.Option {
    return fx.Options(
        fx.Provide(fx.Annotate(NewAddressService, fx.As(new(AddressService)))),
        fx.Provide(fx.Annotate(NewPointsService, fx.As(new(PointsService)))),
        fx.Provide(fx.Annotate(NewAccountService, fx.As(new(AccountService)))),
    )
}

// application/usecase/checkoutuc/di.go
package checkoutuc

func Module() fx.Option {
    return fx.Options(
        fx.Provide(fx.Annotate(NewCheckoutUseCase, fx.As(new(input.CheckoutUseCase)))),
    )
}
```

`main.go` 直接組合所有 Module（見 [Complete main.go Example](#complete-maingo-example)），不使用 Root Module 變數。

### 簡單 Service 可省略 UseCase

如果業務邏輯簡單（純 CRUD，無跨服務流程），gRPC Handler 可直接呼叫 Service：

```go
// adapter/inbound/grpc/address_handler.go
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

DTOs are organized **by UseCase/feature**, with each file containing related request/response types.

```
application/dto/
├── request/
│   ├── profile.go           # UpdateProfileRequest
│   ├── address.go           # AddAddressRequest, UpdateAddressRequest
│   ├── certification.go     # CreateCertificationDraft, SubmitApplication, SaveProgress
│   ├── review.go            # ReviewApplication
│   ├── customer.go          # ListCustomers, UpdateCustomerStatus, GetCustomerList
│   └── tag.go               # CreateTag, UpdateTag, ManageCustomerTag
└── response/
    ├── profile.go           # Profile
    ├── address.go           # Address
    ├── certification.go     # CertificationApplication
    ├── review.go            # CertificationReview
    ├── customer.go          # Customer
    ├── tag.go               # Tag
    └── pagination.go        # PaginatedResult[T] (shared generic)
```

**Organization Rules**:

| Principle | Guideline |
|-----------|-----------|
| One file per feature | Group related request/response types by the UseCase they serve |
| Matching names | `request/address.go` pairs with `response/address.go` |
| Shared generics | Place `PaginatedResult[T]` in `pagination.go` |
| No domain leakage | DTOs are flat data structures, never reference Domain entities directly |

**Why by-UseCase (not by-Domain or single file)**:

| Pattern | Pros | Cons |
|---------|------|------|
| Single file | Simple | Grows unwieldy (13+ types in one file) |
| By domain | Moderate grouping | Unclear boundaries, mixes unrelated UseCases |
| **By UseCase** | Clear boundaries, easy navigation, supports UseCase-per-file pattern | More files (manageable) |

**File Naming Examples**:

```
# Request files named after the operation
request/profile.go          → UpdateProfileRequest
request/certification.go    → CreateCertificationDraft, SubmitApplication, SaveProgress
request/points.go           → AddPointsRequest, DeductPointsRequest, UpdatePointsConfigRequest

# Response files named after the returned data
response/profile.go         → Profile, ProfileSummary
response/points.go          → PointsBalance, PointsRecord, PointsRule
```

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

### External Adapter Structure

If a service calls external APIs (REST, SDKs), it MUST have:

```
adapter/outbound/external/           # Implementation
application/port/output/             # Interface definition
infrastructure/fx/external_module.go # Fx wiring
```

## Monorepo Scaling Strategy `[Infrastructure]`

Initially all services share a single `go.mod`. When service count exceeds 5–8, evaluate:

1. **Go Workspace** (`go.work`): Each service gets independent `go.mod`, workspace unifies dev experience
2. **Independent shared package versioning**: Extract `pkg/` as independent module with semantic versioning

**Decision signals**: Frequent dependency conflicts, build times too long, different services need different versions of shared packages.

## Uber Fx Dependency Injection

Uber Fx wires all layers together. Each package contains its own `di.go` with a `Module()` function.

### Module Layout

```
internal/
├── application/
│   └── usecase/
│       ├── orderuc/
│       │   ├── di.go              # func Module() fx.Option
│       │   ├── create_order.go
│       │   └── cancel_order.go
│       └── paymentuc/
│           ├── di.go
│           └── process_payment.go
├── adapter/
│   ├── inbound/
│   │   └── grpc/
│   │       ├── di.go              # Handler module
│   │       └── order_handler.go
│   └── outbound/
│       └── persistence/
│           ├── di.go              # Repository module
│           ├── order_repository.go
│           └── payment_repository.go
└── infrastructure/
    └── fx/
        ├── config_module.go       # ConfigModule
        ├── logger_module.go       # LoggerModule
        ├── database_module.go     # DatabaseModule + TxManager
        ├── tracer_module.go       # TracerModule (OTel)
        └── grpc_module.go         # GRPCModule + NewGRPCServer + Lifecycle
```

### Package-Level `di.go`

Each package exposes a `Module()` function. See [Complete di.go Examples](#complete-digo-examples) for full code with `fx.Annotate` + `fx.As` interface binding.

**Convention**:

- Constructor 用大寫 (`NewXxx`) — 方便測試直接呼叫，di.go 內用 `fx.Annotate` 包裝綁定 interface
- 新增 UseCase 只改該 package 的 `di.go`，減少 merge conflicts
- Package 自包含，易於維護

### Fx Key Rules

| Concept | When to Use |
|---------|-------------|
| `fx.Provide` | Constructors that return types for others to depend on |
| `fx.Invoke` | Side-effects (register handlers, start pollers) — runs at startup |
| `fx.As(new(Interface))` | Bind concrete type to interface (dependency inversion at DI layer) |
| `fx.Annotate` + `fx.ParamTags` | Disambiguate multiple implementations of the same interface |
| `fx.Lifecycle` | Register `OnStart` / `OnStop` hooks (server listen, graceful shutdown) |

### Complete `main.go` Example

```go
// cmd/order-service/main.go
package main

import (
    "go.uber.org/fx"
    "go.uber.org/zap"

    grpchandler "github.com/yourproject/order-service/internal/adapter/inbound/grpc"
    "github.com/yourproject/order-service/internal/adapter/outbound/persistence"
    "github.com/yourproject/order-service/internal/application/usecase/orderuc"
    infrafx "github.com/yourproject/order-service/internal/infrastructure/fx"
)

func main() {
    fx.New(
        // Infrastructure (config, database, gRPC server, logger, tracer)
        infrafx.ConfigModule,
        infrafx.LoggerModule,
        infrafx.DatabaseModule,
        infrafx.TracerModule,
        infrafx.GRPCModule,

        // Adapter — Outbound (repository implementations)
        persistence.Module(),

        // Application — UseCase
        orderuc.Module(),

        // Adapter — Inbound (gRPC handlers)
        grpchandler.Module(),
    ).Run()
}
```

`fx.New().Run()` handles the full lifecycle: dependency injection → `OnStart` hooks → block on OS signal (SIGINT/SIGTERM) → `OnStop` hooks. No manual signal handling needed.

### Complete `di.go` Examples

**UseCase — Interface binding with `fx.As`**:

```go
// internal/application/usecase/orderuc/di.go
package orderuc

import (
    "go.uber.org/fx"
    "github.com/yourproject/order-service/internal/application/port/input"
)

func Module() fx.Option {
    return fx.Options(
        fx.Provide(fx.Annotate(NewCreateOrderUseCase, fx.As(new(input.CreateOrderUseCase)))),
        fx.Provide(fx.Annotate(NewGetOrderUseCase, fx.As(new(input.GetOrderUseCase)))),
        fx.Provide(fx.Annotate(NewListOrdersUseCase, fx.As(new(input.ListOrdersUseCase)))),
        fx.Provide(fx.Annotate(NewCancelOrderUseCase, fx.As(new(input.CancelOrderUseCase)))),
    )
}
```

**Repository — Interface binding**:

```go
// internal/adapter/outbound/persistence/di.go
package persistence

import (
    "go.uber.org/fx"
    "github.com/yourproject/order-service/internal/domain/repository"
)

func Module() fx.Option {
    return fx.Options(
        fx.Provide(fx.Annotate(NewOrderRepository, fx.As(new(repository.OrderRepository)))),
    )
}
```

**gRPC Handler — Registration via `fx.Invoke`**:

```go
// internal/adapter/inbound/grpc/di.go
package grpc

import (
    "go.uber.org/fx"
    pb "github.com/yourproject/go-proto/order/v1"
    "google.golang.org/grpc"
)

func Module() fx.Option {
    return fx.Options(
        fx.Provide(NewOrderHandler),
        fx.Invoke(func(server *grpc.Server, h *OrderHandler) {
            pb.RegisterOrderServiceServer(server, h)
        }),
    )
}
```

**Infrastructure — Config + Logger + Database + Tracer + gRPC Server**:

```go
// internal/infrastructure/fx/config_module.go
var ConfigModule = fx.Provide(config.Load)  // Fail-Fast: fx.New fails if config invalid

// internal/infrastructure/fx/logger_module.go
var LoggerModule = fx.Provide(func(cfg *config.Config) (*zap.Logger, error) {
    if cfg.Env == "production" {
        return zap.NewProduction()
    }
    return zap.NewDevelopment()
})

// internal/infrastructure/fx/database_module.go
var DatabaseModule = fx.Options(
    fx.Provide(func(cfg *config.Config) (*pgxpool.Pool, error) {
        return database.NewPool(context.Background(), cfg.Database.URL())
    }),
    fx.Provide(fx.Annotate(
        database.NewTxManager,
        fx.As(new(port.TxManager)),
    )),
)

// internal/infrastructure/fx/tracer_module.go
var TracerModule = fx.Options(
    fx.Provide(tracer.NewProvider),  // Returns *sdktrace.TracerProvider
    fx.Invoke(func(lc fx.Lifecycle, tp *sdktrace.TracerProvider) {
        otel.SetTracerProvider(tp)
        lc.Append(fx.Hook{
            OnStop: func(ctx context.Context) error {
                return tp.Shutdown(ctx)
            },
        })
    }),
)

// internal/infrastructure/fx/grpc_module.go
var GRPCModule = fx.Options(
    fx.Provide(NewGRPCServer),
    fx.Invoke(startGRPCServer),
)

func NewGRPCServer(
    logger *zap.Logger,
    metrics *prometheus.Registry,
    auth *interceptor.AuthInterceptor,
    rl *interceptor.RateLimiter,
) *grpc.Server {
    return grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            otelgrpc.UnaryServerInterceptor(),          // 1. OTel tracing
            interceptor.ServerCorrelationInterceptor(),  // 2. correlation_id
            interceptor.LoggingInterceptor(logger),      // 3. Request logging
            interceptor.RecoveryInterceptor(logger),     // 4. Panic recovery
            interceptor.MetricsInterceptor(metrics),     // 5. Prometheus metrics
            interceptor.RateLimitInterceptor(rl),        // 6. Rate limiting
            auth.Unary(),                                // 7. Authentication
            interceptor.ErrorMappingInterceptor(),       // 8. Error mapping (outermost = last to run)
        ),
    )
}

func startGRPCServer(lc fx.Lifecycle, server *grpc.Server, cfg *config.Config, logger *zap.Logger) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            lis, err := net.Listen("tcp", ":"+cfg.Server.GRPCPort)
            if err != nil { return err }
            logger.Info("gRPC server listening", zap.String("port", cfg.Server.GRPCPort))
            go server.Serve(lis)
            return nil
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("gRPC server shutting down")
            server.GracefulStop()
            return nil
        },
    })
}
```

### Dependency Graph

```
main.go
  └─ fx.New()
       ├─ ConfigModule          → *config.Config
       ├─ LoggerModule          → *zap.Logger
       ├─ DatabaseModule        → *pgxpool.Pool, port.TxManager
       ├─ TracerModule          → *sdktrace.TracerProvider
       ├─ GRPCModule            → *grpc.Server (+ Lifecycle hooks)
       ├─ persistence.Module()  → repository.OrderRepository (impl)
       ├─ orderuc.Module()      → input.CreateOrderUseCase, input.GetOrderUseCase, ...
       └─ grpchandler.Module()  → *OrderHandler (+ fx.Invoke registers to grpc.Server)
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
