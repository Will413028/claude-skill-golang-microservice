# Architecture & Directory Structure

## Table of Contents

- [Single Service Directory Structure](#single-service-directory-structure)
  - [Application Layer: Service + UseCase 分層](#application-layer-service--usecase-分層)
  - [DTO Organization Pattern](#dto-organization-pattern)
- [Monorepo Structure](#monorepo-structure)
- [Shared Packages (pkg)](#shared-packages-pkg)
- [Naming Conventions](#naming-conventions)
- [Monorepo Scaling Strategy](#monorepo-scaling-strategy)
- [Uber Fx Dependency Injection](#uber-fx-dependency-injection)
- [Proto / buf Tooling](#proto--buf-tooling)
- [Local Development Environment](#local-development-environment)
- [Scheduled Jobs Pattern](#scheduled-jobs-pattern)

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
    Get(ctx context.Context, id int64) (*response.Address, error)
    List(ctx context.Context, accountID int64) ([]*response.Address, error)
    Create(ctx context.Context, req *request.CreateAddress) error
    Update(ctx context.Context, req *request.UpdateAddress) error
    Delete(ctx context.Context, id int64) error
    SetDefault(ctx context.Context, accountID, addressID int64) error
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
    return &checkoutUseCase{...}
}

func (u *checkoutUseCase) Execute(ctx context.Context, req *request.Checkout) (*response.CheckoutResult, error) {
    // 1. 驗證地址
    addr, err := u.addressSvc.Get(ctx, req.AddressID)
    if err != nil { return nil, err }

    // 2. 扣除積分（如有使用）
    if req.UsePoints > 0 {
        if err := u.pointsSvc.Deduct(ctx, req.AccountID, req.UsePoints); err != nil {
            return nil, err
        }
    }

    // 3. 建立訂單（跨服務呼叫）
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

```go
// infrastructure/fx/service_module.go
var ServiceModule = fx.Options(
    fx.Provide(service.NewAddressService),
    fx.Provide(service.NewPointsService),
    fx.Provide(service.NewAccountService),
)

// infrastructure/fx/usecase_module.go
var UseCaseModule = fx.Options(
    fx.Provide(usecase.NewCheckoutUseCase),
    fx.Provide(usecase.NewGoogleOAuthUseCase),
    fx.Provide(usecase.NewCertificationUseCase),
)

// infrastructure/fx/module.go
var Module = fx.Options(
    ConfigModule,
    DatabaseModule,
    RepositoryModule,
    ServiceModule,      // Service 先於 UseCase
    UseCaseModule,
    GRPCModule,
)
```

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
| Single lookup | `GetByID`, `GetBy<Field>` | `(*Entity, error)` — returns `nil, nil` if not found |
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

## Monorepo Scaling Strategy

Initially all services share a single `go.mod`. When service count exceeds 5–8, evaluate:

1. **Go Workspace** (`go.work`): Each service gets independent `go.mod`, workspace unifies dev experience
2. **Independent shared package versioning**: Extract `pkg/` as independent module with semantic versioning

**Decision signals**: Frequent dependency conflicts, build times too long, different services need different versions of shared packages.

## Uber Fx Dependency Injection

Uber Fx wires all layers together. Each layer exposes an `fx.Option` (Module) that provides its implementations.

### Module Organization Strategies

Two approaches for organizing Fx modules. Choose based on project size:

#### Strategy A: Centralized (Small Projects < 10 UseCases)

All DI configuration in `infrastructure/fx/`:

```
internal/infrastructure/fx/
├── module.go            # Root module: combines all sub-modules
├── config_module.go     # Config loading
├── database_module.go   # PG pool + TxManager
├── logger_module.go     # Zap logger
├── grpc_module.go       # gRPC server + interceptors
├── usecase_module.go    # UseCases
├── repository_module.go # Repository implementations
└── ...
```

```go
// internal/infrastructure/fx/usecase_module.go
var UseCaseModule = fx.Options(
    fx.Provide(usecase.NewCreateOrderUseCase),
    fx.Provide(usecase.NewCancelOrderUseCase),
)
```

#### Strategy B: Distributed (Medium-Large Projects ≥ 10 UseCases) — Recommended

Each package contains its own `di.go` with a `Module()` function:

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
│   └── outbound/
│       └── persistence/
│           ├── di.go              # Repository module
│           ├── order_repository.go
│           └── payment_repository.go
└── infrastructure/
    └── fx/
        └── module.go              # Root: combines all sub-modules
```

```go
// internal/application/usecase/orderuc/di.go
package orderuc

import "go.uber.org/fx"

func Module() fx.Option {
    return fx.Options(
        fx.Provide(newCreateOrderUseCase),  // lowercase: unexported
        fx.Provide(newCancelOrderUseCase),
    )
}

// internal/infrastructure/fx/module.go
var Module = fx.Options(
    // Infrastructure
    ConfigModule,
    DatabaseModule,

    // Adapter — Outbound
    persistence.Module(),

    // Application — UseCase
    orderuc.Module(),
    paymentuc.Module(),

    // Adapter — Inbound
    grpchandler.Module(),
)
```

#### Strategy Comparison

| Aspect | Centralized | Distributed `di.go` |
|--------|-------------|---------------------|
| Add new UseCase | Edit `usecase_module.go` | Edit package's `di.go` |
| Constructor visibility | Must be exported (`NewXxx`) | Can be unexported (`newXxx`) |
| Merge conflicts | High (shared file) | Low (package-local) |
| Discoverability | One place to see all | Each package self-contained |
| Large team | ❌ Bottleneck | ✅ Independent |
| Circular dependency risk | Medium | Low |

**Recommendation**: Start with Centralized for MVP. Migrate to Distributed when UseCase count exceeds 10 or team grows.

### Wiring Pattern

```go
// internal/infrastructure/fx/module.go
var Module = fx.Options(
    ConfigModule,
    DatabaseModule,
    ObservabilityModule,
    RepositoryModule,
    ClientModule,
    UseCaseModule,
    GRPCModule,
    // Async stage additions:
    // MQModule,
    // PollerModule,
)
```

```go
// internal/infrastructure/fx/database_module.go
var DatabaseModule = fx.Options(
    fx.Provide(func(cfg *config.Config) (*pgxpool.Pool, error) {
        return database.NewPool(context.Background(), cfg.Database.URL())
    }),
    fx.Provide(func(pool *pgxpool.Pool) port.TxManager {
        return database.NewTxManager(pool)
    }),
)
```

```go
// internal/infrastructure/fx/repository_module.go
var RepositoryModule = fx.Options(
    // fx.As binds concrete impl to Domain interface (dependency inversion)
    fx.Provide(fx.Annotate(
        postgres.NewOrderRepository,
        fx.As(new(repository.OrderRepository)),
    )),
    fx.Provide(fx.Annotate(
        postgres.NewOutboxRepository,
        fx.As(new(repository.OutboxRepository)),
    )),
)
```

```go
// internal/infrastructure/fx/usecase_module.go
var UseCaseModule = fx.Options(
    // Fx auto-resolves constructor params from provided types
    fx.Provide(fx.Annotate(
        usecase.NewCreateOrderUseCase,
        fx.As(new(input.CreateOrderUseCase)),
    )),
)
```

```go
// internal/infrastructure/fx/grpc_module.go
var GRPCModule = fx.Options(
    fx.Provide(NewGRPCServer),
    fx.Invoke(RegisterGRPCServices),  // Invoke: side-effect only (register handlers)
)

func NewGRPCServer(logger *zap.Logger, metrics *prometheus.Registry) *grpc.Server {
    return grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            otelgrpc.UnaryServerInterceptor(),
            interceptor.ServerCorrelationInterceptor(),
            interceptor.LoggingInterceptor(logger),
            interceptor.RecoveryInterceptor(logger),
            interceptor.MetricsInterceptor(metrics),
            interceptor.AuthInterceptor(jwtValidator),   // See grpc-patterns.md Auth section
            interceptor.ErrorMappingInterceptor(),
        ),
    )
}

func RegisterGRPCServices(server *grpc.Server, orderHandler *handler.OrderHandler) {
    pb.RegisterOrderServiceServer(server, orderHandler)
    // Register gRPC Health Check (see infrastructure.md)
}
```

### Fx Key Rules

| Concept | When to Use |
|---------|-------------|
| `fx.Provide` | Constructors that return types for others to depend on |
| `fx.Invoke` | Side-effects (register handlers, start pollers) — runs at startup |
| `fx.As(new(Interface))` | Bind concrete type to interface (dependency inversion at DI layer) |
| `fx.Annotate` + `fx.ParamTags` | Disambiguate multiple implementations of the same interface |
| `fx.Lifecycle` | Register `OnStart` / `OnStop` hooks (server listen, graceful shutdown) |

### Lifecycle Hooks

```go
fx.Invoke(func(lc fx.Lifecycle, server *grpc.Server, cfg *config.Config) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            lis, err := net.Listen("tcp", ":"+cfg.Server.Port)
            if err != nil { return err }
            go server.Serve(lis)
            return nil
        },
        OnStop: func(ctx context.Context) error {
            server.GracefulStop()
            return nil
        },
    })
})
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
    image: postgres:18-alpine
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
		if [ -d "$$dir/migrations" ]; then \
			echo "Migrating $$(basename $$dir)..."; \
			atlas migrate apply --dir "file://$$dir/migrations" \
				--url "postgres://...$$(basename $$dir)_db?sslmode=disable"; \
		fi; \
	done

generate:
	buf generate
	@for dir in services/*/; do \
		if [ -f "$$dir/db/sqlc.yaml" ]; then \
			echo "sqlc generate $$(basename $$dir)..."; \
			cd $$dir/db && sqlc generate && cd ../../..; \
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

## Scheduled Jobs Pattern

Design scheduled jobs as UseCases with dual entry points: **Cron** (automatic) + **API** (manual trigger). This enables backfilling, debugging, and manual retry without code changes.

### Architecture

```
                    ┌─────────────────┐
                    │   JobUseCase    │
                    │  Handle(param)  │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                                 ▼
    ┌───────────────┐                ┌───────────────┐
    │  Cron Handler │                │  API Handler  │
    │ (fixed param) │                │ (custom param)│
    └───────────────┘                └───────────────┘
```

### Directory Structure

```
internal/
├── application/
│   └── usecase/
│       └── jobuc/
│           ├── di.go
│           ├── stats_content_hourly.go    # Job UseCase
│           └── purge_cache.go
├── adapter/
│   └── inbound/
│       ├── grpc/                          # gRPC handlers
│       └── job/                           # Job controller (cron + API)
│           ├── controller.go
│           └── dto.go
└── infrastructure/
    └── cron/
        └── scheduler.go                   # Cron registration
```

### UseCase Implementation

```go
// internal/application/usecase/jobuc/stats_content_hourly.go
package jobuc

type StatsContentHourlyJob interface {
    Handle(ctx context.Context, param *StatsContentHourlyParam) error
}

type StatsContentHourlyParam struct {
    StatsTimeStart time.Time
    StatsTimeEnd   time.Time
}

type statsContentHourlyJob struct {
    contentRepo repository.ContentRepository
    statsRepo   repository.StatsRepository
}

func newStatsContentHourlyJob(
    contentRepo repository.ContentRepository,
    statsRepo repository.StatsRepository,
) StatsContentHourlyJob {
    return &statsContentHourlyJob{
        contentRepo: contentRepo,
        statsRepo:   statsRepo,
    }
}

func (j *statsContentHourlyJob) Handle(ctx context.Context, param *StatsContentHourlyParam) error {
    // Business logic: aggregate stats for the given time range
    // This method is idempotent - same params produce same results

    // 1. Delete existing stats for this time range (idempotent)
    if err := j.statsRepo.DeleteByTimeRange(ctx, param.StatsTimeStart, param.StatsTimeEnd); err != nil {
        return err
    }

    // 2. Aggregate and insert new stats
    stats, err := j.contentRepo.AggregateStats(ctx, param.StatsTimeStart, param.StatsTimeEnd)
    if err != nil {
        return err
    }

    return j.statsRepo.BulkInsert(ctx, stats)
}
```

### Controller (Dual Entry Points)

```go
// internal/adapter/inbound/job/controller.go
package job

type Controller struct {
    statsContentHourlyJob jobuc.StatsContentHourlyJob
}

// API Handler - accepts custom time range for backfill/debug
func (c *Controller) StatsContentHourly(ctx *gin.Context) {
    var req StatsContentHourlyRequest
    if err := ctx.ShouldBindJSON(&req); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    param := &jobuc.StatsContentHourlyParam{
        StatsTimeStart: req.StatsTimeStart,
        StatsTimeEnd:   req.StatsTimeEnd,
    }

    if err := c.statsContentHourlyJob.Handle(ctx.Request.Context(), param); err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    ctx.Status(http.StatusNoContent)
}

// Cron Handler - fixed parameters (previous hour)
func (c *Controller) StatsContentHourlyCron() {
    ctx := context.Background()
    ctx = ctxutil.TraceID.Set(ctx, uuid.New().String())

    now := time.Now().Truncate(time.Hour)
    param := &jobuc.StatsContentHourlyParam{
        StatsTimeStart: now.Add(-time.Hour),
        StatsTimeEnd:   now,
    }

    zap.L().Info("job start", zap.String("job", "StatsContentHourly"), ctxutil.TraceID.ToLog(ctx))
    defer zap.L().Info("job end", zap.String("job", "StatsContentHourly"), ctxutil.TraceID.ToLog(ctx))

    if err := c.statsContentHourlyJob.Handle(ctx, param); err != nil {
        zap.L().Error("job failed", zap.String("job", "StatsContentHourly"), zap.Error(err))
    }
}
```

### DTO

```go
// internal/adapter/inbound/job/dto.go
package job

type StatsContentHourlyRequest struct {
    StatsTimeStart time.Time `json:"statsTimeStart" binding:"required"`
    StatsTimeEnd   time.Time `json:"statsTimeEnd" binding:"required"`
}
```

### Cron Scheduler

```go
// internal/infrastructure/cron/scheduler.go
package cron

import (
    "github.com/robfig/cron/v3"
)

type Scheduler struct {
    cron       *cron.Cron
    controller *job.Controller
}

func NewScheduler(controller *job.Controller) *Scheduler {
    c := cron.New(
        cron.WithLocation(time.FixedZone("Asia/Taipei", 8*60*60)),
        cron.WithSeconds(),
    )
    return &Scheduler{cron: c, controller: controller}
}

func (s *Scheduler) Start() {
    // Every hour at :10 (給資料寫入 buffer)
    s.cron.AddFunc("0 10 * * * *", s.controller.StatsContentHourlyCron)

    // Daily at 03:30
    s.cron.AddFunc("0 30 3 * * *", s.controller.DailyReportCron)

    s.cron.Start()
}

func (s *Scheduler) Stop() context.Context {
    return s.cron.Stop()
}
```

### Router (API Endpoints)

```go
// internal/adapter/inbound/job/router.go
func RegisterRoutes(r *gin.RouterGroup, ctrl *Controller, authMiddleware gin.HandlerFunc) {
    jobGroup := r.Group("/jobs", authMiddleware, adminOnlyMiddleware)
    {
        jobGroup.POST("/stats-content-hourly", ctrl.StatsContentHourly)
        jobGroup.POST("/daily-report", ctrl.DailyReport)
        jobGroup.POST("/purge-cache", ctrl.PurgeCache)
    }
}
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Backfill** | API accepts custom time range to re-process historical data |
| **Debug/Test** | Trigger immediately without waiting for cron schedule |
| **Manual retry** | Re-run failed jobs with same or adjusted parameters |
| **Idempotent** | Same parameters always produce same results |
| **Unified logic** | UseCase doesn't know who called it — stays pure |
| **Observability** | Both entry points log with TraceID for correlation |

### Design Rules

1. **UseCase must be idempotent**: Delete-then-insert pattern, or upsert with version check
2. **Cron handler sets default params**: Previous hour, previous day, etc.
3. **API handler validates custom params**: Time range limits, authorization
4. **Both handlers set TraceID**: For log correlation
5. **API protected by auth**: Admin-only middleware for manual triggers
6. **Graceful shutdown**: Stop scheduler before closing DB connections

### Distributed Lock (Prevent Duplicate Execution)

When running multiple instances (K8s replicas), use **Redis distributed lock** to ensure only one instance executes a job:

```go
// pkg/redislock/lock.go
package redislock

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

type DistributedLock struct {
    client *redis.Client
}

func NewDistributedLock(client *redis.Client) *DistributedLock {
    return &DistributedLock{client: client}
}

// TryLock attempts to acquire a lock. Returns true if lock acquired.
// lockKey: unique job identifier (e.g., "job:stats-content-hourly:2024-01-15T10:00")
// ttl: lock auto-expires after ttl to prevent deadlock
func (d *DistributedLock) TryLock(ctx context.Context, lockKey string, ttl time.Duration) (bool, error) {
    // SET NX: only set if not exists
    result, err := d.client.SetNX(ctx, lockKey, "locked", ttl).Result()
    if err != nil {
        return false, err
    }
    return result, nil
}

func (d *DistributedLock) Unlock(ctx context.Context, lockKey string) error {
    return d.client.Del(ctx, lockKey).Err()
}
```

```go
// Controller with distributed lock
func (c *Controller) StatsContentHourlyCron() {
    ctx := context.Background()
    ctx = ctxutil.TraceID.Set(ctx, uuid.New().String())

    now := time.Now().Truncate(time.Hour)
    lockKey := fmt.Sprintf("job:stats-content-hourly:%s", now.Add(-time.Hour).Format(time.RFC3339))

    // Try to acquire lock (TTL = max job duration + buffer)
    acquired, err := c.lock.TryLock(ctx, lockKey, 30*time.Minute)
    if err != nil {
        zap.L().Error("failed to acquire lock", zap.String("job", "StatsContentHourly"), zap.Error(err))
        return
    }
    if !acquired {
        zap.L().Info("job skipped - another instance is running", zap.String("job", "StatsContentHourly"))
        return
    }
    // Note: Don't unlock manually - let TTL expire to prevent re-execution within window

    param := &jobuc.StatsContentHourlyParam{
        StatsTimeStart: now.Add(-time.Hour),
        StatsTimeEnd:   now,
    }

    zap.L().Info("job start", zap.String("job", "StatsContentHourly"), ctxutil.TraceID.ToLog(ctx))
    if err := c.statsContentHourlyJob.Handle(ctx, param); err != nil {
        zap.L().Error("job failed", zap.String("job", "StatsContentHourly"), zap.Error(err))
        return
    }
    zap.L().Info("job end", zap.String("job", "StatsContentHourly"), ctxutil.TraceID.ToLog(ctx))
}
```

**Lock Key Design**:
- Include job name + execution window: `job:{name}:{window}`
- Window granularity matches job frequency (hourly job → hour, daily → date)
- Prevents re-execution within the same window, even after restart

**TTL Strategy**:
| Job Duration | Recommended TTL |
|--------------|-----------------|
| < 1 minute | 5 minutes |
| 1-10 minutes | 30 minutes |
| 10-60 minutes | 2 hours |
| > 1 hour | job duration × 2 |

### Job Execution History (Audit Log)

Track job executions for debugging and compliance:

```sql
-- migrations/000X_create_job_executions.sql
CREATE TABLE job_executions (
    id              BIGSERIAL PRIMARY KEY,
    job_name        VARCHAR(100) NOT NULL,
    trace_id        VARCHAR(36) NOT NULL,
    status          VARCHAR(20) NOT NULL,  -- 'running', 'completed', 'failed'
    trigger_type    VARCHAR(20) NOT NULL,  -- 'cron', 'api'
    triggered_by    VARCHAR(100),          -- user_id for API, 'system' for cron
    params          JSONB,
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL,
    finished_at     TIMESTAMPTZ,
    duration_ms     INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_job_executions_job_name ON job_executions(job_name);
CREATE INDEX idx_job_executions_started_at ON job_executions(started_at);
CREATE INDEX idx_job_executions_status ON job_executions(status);
```

```go
// internal/domain/repository/job_execution_repository.go
type JobExecutionRepository interface {
    Create(ctx context.Context, exec *entity.JobExecution) error
    Update(ctx context.Context, exec *entity.JobExecution) error
    FindByJobName(ctx context.Context, jobName string, limit int) ([]*entity.JobExecution, error)
    FindRunning(ctx context.Context) ([]*entity.JobExecution, error)
}
```

```go
// internal/adapter/inbound/job/executor.go
package job

type JobExecutor struct {
    execRepo repository.JobExecutionRepository
}

// WithTracking wraps job execution with audit logging
func (e *JobExecutor) WithTracking(
    ctx context.Context,
    jobName string,
    triggerType string,  // "cron" or "api"
    triggeredBy string,  // user_id or "system"
    params any,
    fn func(ctx context.Context) error,
) error {
    traceID := ctxutil.TraceID.Get(ctx)
    if traceID == "" {
        traceID = uuid.New().String()
        ctx = ctxutil.TraceID.Set(ctx, traceID)
    }

    paramsJSON, _ := json.Marshal(params)
    exec := &entity.JobExecution{
        JobName:     jobName,
        TraceID:     traceID,
        Status:      "running",
        TriggerType: triggerType,
        TriggeredBy: triggeredBy,
        Params:      paramsJSON,
        StartedAt:   time.Now(),
    }

    if err := e.execRepo.Create(ctx, exec); err != nil {
        zap.L().Error("failed to create job execution record", zap.Error(err))
        // Continue execution even if logging fails
    }

    // Execute the job
    jobErr := fn(ctx)

    // Update execution record
    exec.FinishedAt = time.Now()
    exec.DurationMs = int(exec.FinishedAt.Sub(exec.StartedAt).Milliseconds())
    if jobErr != nil {
        exec.Status = "failed"
        exec.ErrorMessage = jobErr.Error()
    } else {
        exec.Status = "completed"
    }

    if err := e.execRepo.Update(ctx, exec); err != nil {
        zap.L().Error("failed to update job execution record", zap.Error(err))
    }

    return jobErr
}
```

```go
// Usage in controller
func (c *Controller) StatsContentHourlyCron() {
    ctx := context.Background()
    now := time.Now().Truncate(time.Hour)
    param := &jobuc.StatsContentHourlyParam{
        StatsTimeStart: now.Add(-time.Hour),
        StatsTimeEnd:   now,
    }

    err := c.executor.WithTracking(ctx, "StatsContentHourly", "cron", "system", param,
        func(ctx context.Context) error {
            return c.statsContentHourlyJob.Handle(ctx, param)
        },
    )
    if err != nil {
        // Error already logged by executor
    }
}
```

### Job Monitoring & Alerting

#### Structured Log Schema

Consistent log fields enable log-based alerting:

```go
// Standard job log fields
zap.L().Info("job event",
    zap.String("event", "job_start"),    // job_start, job_end, job_failed
    zap.String("job", "StatsContentHourly"),
    zap.String("trigger", "cron"),        // cron, api
    zap.String("trace_id", traceID),
    zap.Int64("duration_ms", durationMs), // for job_end
    zap.Error(err),                       // for job_failed
)
```

#### Prometheus Metrics

```go
// internal/infrastructure/metrics/job_metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    JobExecutionTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "job_execution_total",
            Help: "Total number of job executions",
        },
        []string{"job_name", "status", "trigger_type"},
    )

    JobExecutionDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "job_execution_duration_seconds",
            Help:    "Job execution duration in seconds",
            Buckets: []float64{1, 5, 10, 30, 60, 120, 300, 600},
        },
        []string{"job_name"},
    )

    JobCurrentlyRunning = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "job_currently_running",
            Help: "Number of currently running jobs",
        },
        []string{"job_name"},
    )
)
```

```go
// Updated executor with metrics
func (e *JobExecutor) WithTracking(...) error {
    metrics.JobCurrentlyRunning.WithLabelValues(jobName).Inc()
    defer metrics.JobCurrentlyRunning.WithLabelValues(jobName).Dec()

    startTime := time.Now()

    jobErr := fn(ctx)

    duration := time.Since(startTime).Seconds()
    metrics.JobExecutionDuration.WithLabelValues(jobName).Observe(duration)

    status := "completed"
    if jobErr != nil {
        status = "failed"
    }
    metrics.JobExecutionTotal.WithLabelValues(jobName, status, triggerType).Inc()

    return jobErr
}
```

#### Alert Rules (Prometheus/Grafana)

```yaml
# prometheus/alerts/job_alerts.yml
groups:
  - name: scheduled_jobs
    rules:
      # Job failed
      - alert: JobExecutionFailed
        expr: increase(job_execution_total{status="failed"}[5m]) > 0
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Job {{ $labels.job_name }} failed"
          description: "Job {{ $labels.job_name }} failed in the last 5 minutes"

      # Job running too long
      - alert: JobRunningTooLong
        expr: job_currently_running > 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Job {{ $labels.job_name }} running for over 30 minutes"

      # Job not executed (missed schedule)
      - alert: JobMissedSchedule
        expr: |
          time() - max(job_execution_total{job_name="StatsContentHourly"}) by (job_name) > 7200
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Job {{ $labels.job_name }} missed scheduled execution"
          description: "No execution in the last 2 hours"

      # Job duration anomaly
      - alert: JobDurationAnomaly
        expr: |
          job_execution_duration_seconds{quantile="0.99"} >
          avg_over_time(job_execution_duration_seconds{quantile="0.99"}[7d]) * 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Job {{ $labels.job_name }} taking longer than usual"
```

### When to Use This Pattern

| Scenario | Use Dual Entry? |
|----------|-----------------|
| Data aggregation (hourly/daily stats) | ✅ Yes — backfill is common |
| Cache cleanup | ✅ Yes — manual trigger for debugging |
| Report generation | ✅ Yes — regenerate on demand |
| One-time migration | ❌ No — use dedicated migration script |
| Real-time event processing | ❌ No — use MQ consumer |
