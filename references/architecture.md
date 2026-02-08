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
├── .air.toml                           # ⚡ Air 熱重載設定 (開發用)
├── buf.yaml                            # 🔧 Buf 設定 (per-service)
├── buf.gen.yaml                        # 🔧 生成 Go & Gateway & Swagger
├── .go-arch-lint.yml                   # 🛡️ 架構防腐 (層依賴規則)
├── Dockerfile                          # 🐳 Multi-stage (dev + prod)
│
├── cmd/
│   └── xxx-service/                    # 🚀 以服務名命名 (非 server/)
│       └── main.go                     # 啟動點 (依賴注入)
│
├── internal/                           # 🔒 私有核心: Clean Architecture
│   ├── domain/                         # 🏛️ 核心層 (Entity, Interface, Enum)
│   │   ├── order.go                    # Entity + Repository Interface + rich methods
│   │   ├── order_types.go              # Type-safe enums (enumer generated, co-located with Entity)
│   │   ├── order_event.go              # Domain Events for this aggregate
│   │   ├── valueobject/                # Value Object (immutable, with behavior logic)
│   │   ├── service/                    # Domain Service (cross-entity business logic, zero deps)
│   │   └── errors.go                   # Domain error definitions
│   │
│   ├── usecase/                        # 🎯 應用層: business flow orchestration
│   │   ├── create_order.go             # UseCase implementation
│   │   ├── cancel_order.go
│   │   ├── dto/                        # Data Transfer Objects
│   │   │   ├── order_req.go
│   │   │   └── order_res.go
│   │   └── di.go                       # fx.Module (UseCase layer DI)
│   │
│   ├── repository/                     # 💾 資料存取層
│   │   ├── postgres/                   # 🐘 SQL 實作
│   │   │   ├── gen/                    # 🤖 sqlc auto-generated (DO NOT EDIT)
│   │   │   │   ├── models.go          # DB Models (maps to table schema)
│   │   │   │   ├── query.sql.go       # DB Methods (auto-generated)
│   │   │   │   └── db.go              # DBTX Interface
│   │   │   ├── order.go               # Implements domain.OrderRepository
│   │   │   └── mapper.go              # gen.Model ↔ domain.Entity conversion
│   │   ├── redis/                      # ⚡ 快取層 (optional)
│   │   │   ├── order_proxy.go          # Cache Decorator (使用 pkg/redis)
│   │   │   └── cache.go               # Shared cache helpers
│   │   └── di.go                       # fx.Module (aggregates postgres + redis)
│   │
│   ├── infrastructure/                 # 🏗️ 基建層: server setup, external infra
│   │   ├── server.go                   # gRPC/HTTP Server bootstrap
│   │   └── address_impl.go            # External infra adapters
│   │
│   ├── client/                         # 🌐 外部適配: external service clients
│   │   ├── payment/
│   │   │   ├── client.go               # Payment API client (e.g., PayUni adapter)
│   │   │   ├── dto.go                  # Request/Response types
│   │   │   └── mapper.go              # domain ↔ client DTO conversion
│   │   ├── inventory/
│   │   │   └── client.go               # Inventory gRPC client
│   │   └── di.go                       # fx.Module (Client layer DI)
│   │
│   ├── grpc/                           # 📡 傳輸層: gRPC interface (Inbound)
│   │   ├── server.go                   # gRPC Server setup
│   │   ├── handler.go                 # gRPC Handler (calls UseCase)
│   │   ├── mapper.go                  # Protobuf ↔ DTO conversion
│   │   └── di.go                       # fx.Module (gRPC layer DI)
│   │
│   ├── worker/                         # ⚙️ 背景任務 (Outbox Publisher, async jobs)
│   │   ├── outbox_publisher.go
│   │   └── di.go
│   │
│   └── app/                            # 🧩 組裝層: application assembly
│       └── app.go                      # fx.New() — assembles all modules
│
├── db/                                 # 🗄️ 服務專屬 DB 定義
│   ├── migrations/                     # Atlas 遷移檔
│   ├── schema.hcl                      # Atlas Schema (HCL, single source of truth)
│   └── query.sql                       # sqlc 查詢語句
│
└── test/                               # 🧪 測試 (Integration, Mocks)
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Enum location** | `domain/*_types.go` (co-located with Entity) | High cohesion — Enum, Entity, and Repository Interface in same package |
| **Repository Interface** | Bottom of `domain/{entity}.go` | Go convention: define interface near the domain it serves; avoids separate `repository/` package bloat |
| **sqlc output** | `repository/postgres/gen/` | Named `gen/` under `postgres/` to clearly indicate auto-generated DB code |
| **Cache Decorator** | `repository/redis/xxx_proxy.go` | Proxy pattern — intercepts Repository interface, checks Redis first, falls through to Postgres |
| **Flat directory** | No `adapter/inbound/outbound/` nesting | Go-style flat structure; name by concern (`grpc/`, `repository/`, `client/`) not by direction |
| **DI per-layer** | Each package has `di.go` with `fx.Module` | Modular, self-contained; `app.go` only assembles modules |
| **cmd naming** | `cmd/{service-name}/` (not `cmd/server/`) | Matches service identity, supports multi-binary if needed |
| **worker package** | Separate `internal/worker/` | Background tasks (Outbox Publisher) isolated from request handlers |
| **Architecture guard** | `.go-arch-lint.yml` | Automated enforcement of layer dependency rules |

> **Database Directory Convention**: All database-related files (`schema.hcl`, `query.sql`, `migrations/`) are centralized in the `db/` directory to keep the service root clean. Atlas uses HCL for declarative schema:
>
> ```bash
> cd db && sqlc generate           # Generate Go code to internal/repository/postgres/gen/
> cd db && atlas migrate diff ...  # Generate migration from schema.hcl
> ```

### Cache Decorator Pattern (RepoRedis)

UseCase only depends on the Repository interface. A Redis cache decorator wraps the Postgres implementation, keeping business logic completely clean:

```
UseCase → Repository Interface → RepoRedis (Proxy) → Redis Cache
                                                    ↓ (miss)
                                              RepoPG (Postgres)
```

```go
// internal/repository/redis/order_proxy.go
type orderRedisProxy struct {
    pg    domain.OrderRepository  // actual Postgres impl
    redis *redis.Client
    ttl   time.Duration
}

func NewOrderRedisProxy(pg domain.OrderRepository, redis *redis.Client) domain.OrderRepository {
    return &orderRedisProxy{pg: pg, redis: redis, ttl: 5 * time.Minute}
}

func (r *orderRedisProxy) GetByID(ctx context.Context, id uuid.UUID) (*domain.Order, error) {
    // 1. Check Redis
    cached, err := r.getFromCache(ctx, id)
    if err == nil { return cached, nil }

    // 2. Cache miss → query Postgres
    order, err := r.pg.GetByID(ctx, id)
    if err != nil { return nil, err }

    // 3. Populate cache
    r.setCache(ctx, order)
    return order, nil
}
```

DI wiring with decorator:

```go
// internal/repository/di.go — aggregates postgres + redis sub-modules
var Module = fx.Module("repository",
    postgres.Module,               // provides concrete Postgres impls
    fx.Provide(
        fx.Annotate(redis.NewOrderRedisProxy, fx.As(new(domain.OrderRepository))),
    ),
)
```

### Architecture Guard (.go-arch-lint.yml)

Enforce layer dependency rules to prevent architecture erosion:

```yaml
# .go-arch-lint.yml
allow:
  domain: []                          # domain depends on nothing
  usecase: [domain]                   # usecase only depends on domain
  repository: [domain]               # repository implements domain interfaces
  infrastructure: [domain, usecase]  # infra can depend on domain + usecase
  client: [domain]                    # client adapts external to domain
  grpc: [domain, usecase]            # grpc calls usecase
  worker: [domain, usecase]          # worker calls usecase
  app: [domain, usecase, repository, infrastructure, client, grpc, worker]
```

Run: `go-arch-lint check --project-path .`

### UseCase vs Domain Service

| Layer | Location | 職責 | 依賴 |
|-------|----------|------|------|
| **UseCase** | `internal/usecase/` | 應用程式流程編排（輸入 → 驗證 → 交易 → 存檔 → 通知）| Repository Interface, Client, Domain Service |
| **Domain Service** | `internal/domain/service/` | 純領域規則（手續費算法、風險判定、跨 Entity 業務邏輯）| 零外部依賴，透過參數傳入資料或注入 Repository Interface |

**關鍵區分**：
- **UseCase** 關注「應用程式流程」— 協調 Repository、Client、Domain Service 完成一個完整業務流程
- **Domain Service** 關注「領域規則」— 純業務邏輯計算，不包含 DB 操作或 HTTP 呼叫
- UseCase 呼叫 Domain Service，但 Domain Service **不呼叫** UseCase

```go
// internal/domain/service/fee_calculator.go
// Domain Service — 純業務邏輯，零外部依賴
type FeeCalculator struct{}

func (fc *FeeCalculator) Calculate(order *domain.Order, rules []domain.FeeRule) domain.Money {
    // 純計算邏輯，資料透過參數傳入
}
```

```go
// internal/usecase/create_order.go
// UseCase — 應用程式流程編排
type CreateOrderUseCase struct {
    orderRepo    domain.OrderRepository      // 注入 Repository Interface
    feeCalc      *domainservice.FeeCalculator // 注入 Domain Service
    payClient    paymentClient                // consumer-defined local interface
    txManager    txManager                    // consumer-defined local interface
    logger       *zap.Logger
}

func (uc *CreateOrderUseCase) Execute(ctx context.Context, req *dto.CreateOrderRequest) (*dto.CreateOrderResponse, error) {
    // 1. 查詢資料
    rules, err := uc.orderRepo.ListFeeRules(ctx)
    if err != nil { return nil, err }

    // 2. 調用 Domain Service 計算（純邏輯）
    fee := uc.feeCalc.Calculate(req.ToOrder(), rules)

    // 3. 交易 + 存檔
    err = uc.txManager.WithTx(ctx, func(txCtx context.Context) error {
        return uc.orderRepo.Create(txCtx, order)
    })
    if err != nil { return nil, err }

    // 4. 呼叫外部服務
    _, err = uc.payClient.Charge(ctx, &ChargeRequest{Amount: fee})
    return &dto.CreateOrderResponse{OrderID: order.ID}, err
}

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
project-root/                            # 📦 Monorepo 根目錄
├── go.mod                               # 🌍 Workspace / Root Module
├── buf.work.yaml                        # 🌍 Buf Workspace 設定
├── Makefile                             # 🛠️ 全域指令 (make dev-up, make lint)
├── docker-compose.yaml                  # 🐳 本地開發 (Infra + Services + LGTM)
├── .env                                 # 🔐 環境變數 (給 docker-compose)
│
├── monitoring/                          # 🔭 可觀測性設定中心
│   ├── grafana/                         # 📊 Grafana Dashboards & Datasources
│   ├── prometheus/                      # 📈 Prometheus 設定 (Metrics)
│   ├── loki/                            # 🪵 Loki 設定 (Logs)
│   ├── tempo/                           # ⏱️ Tempo 設定 (Traces)
│   └── otel-collector/                  # 📡 OpenTelemetry Collector (資料轉運站)
│
├── pkg/                                 # 🧱 全域共用基建 (shared library)
│   ├── config/                          # ⚙️ 統一 Config (Viper)
│   ├── database/                        # 🗄️ 統一 Postgres 連線池設定
│   ├── redis/                           # ⚡ 統一 Redis Client & Lock
│   ├── logger/                          # 📝 統一 Zap/Slog 格式
│   ├── errors/                          # ❌ 全域錯誤碼 (Domain Errors)
│   ├── middleware/                      # 🛡️ gRPC/HTTP Interceptor (Auth, Trace, Log)
│   ├── otel/                            # 🔍 Tracing 初始化封裝
│   ├── twaddr/                          # 📮 [通用業務] 台灣地址解析
│   └── payuni/                          # 💳 [通用業務] PayUni SDK 封裝
│
├── api/                                 # 📋 介面合約
│   ├── proto/
│   │   ├── merchant/                    # merchant.proto (Source)
│   │   └── common/                      # Shared proto (pagination, money)
│   └── openapi/                         # 自動生成的 Swagger JSON
│
└── services/                            # 🏭 微服務群
    ├── merchant-service/                # 各服務 (見 Single Service 結構)
    └── (future-service)/                # 未來的其他服務
```

> **Key Monorepo Conventions**:
> - Root `go.mod` acts as workspace — all services share dependencies
> - `buf.work.yaml` at root orchestrates per-service buf configs
> - `pkg/` at root level (not nested) — shared across all services
> - `monitoring/` centralizes all observability configs (Grafana, Prometheus, Loki, Tempo, OTel Collector)
> - Each service has its own `buf.yaml` + `buf.gen.yaml` for proto generation (Go + Gateway + Swagger)
> - Proto source files live in `api/proto/`, generated code goes to service-local or `api/openapi/`

## Shared Packages (pkg)

| Package | Responsibility | Stage |
|---------|---------------|-------|
| `config` | 統一 Config (Viper-based) | MVP |
| `logger` | 統一 Zap/Slog 格式 | MVP |
| `errors` | 全域錯誤碼 + DomainError interface | MVP |
| `database` | 統一 Postgres 連線池設定 | MVP |
| `redis` | 統一 Redis Client & Lock | MVP |
| `middleware` | gRPC/HTTP Interceptor (Auth, Trace, Log) | MVP |
| `otel` | Tracing 初始化封裝 | MVP |
| `twaddr` | [通用業務] 台灣地址解析 | MVP |
| `payuni` | [通用業務] PayUni SDK 封裝 | MVP |
| `cache` | Generic CacheLoader + singleflight | Async |
| `circuitbreaker` | gobreaker wrapper | Async |
| `mq/rabbitmq` | MQ connection + trace propagation | Async |
| `outbox` | Two-phase Outbox Poller | Async |
| `saga` | Saga timeout monitor | Async |

**Why `ErrorCode` lives in `pkg/errors` instead of `internal/domain`**: Avoids circular dependency. `pkg/errors` defines the `ErrorCode` type and `DomainError` interface. Domain layer imports it to implement; Interceptor imports it to map. `pkg/errors` contains only pure constants and interfaces — no runtime or transport protocol dependencies.

**Business-specific shared packages** (`twaddr`, `payuni`): These contain reusable business logic shared across services. They follow the same zero-side-effect principle as infrastructure packages.

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
├── domain/
│   └── service/                  # Domain Service (純業務邏輯, zero deps)
├── usecase/
│   ├── create_order.go
│   ├── cancel_order.go
│   └── di.go                     # var Module = fx.Module("usecase", ...)
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
// cmd/order-service/main.go
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

### Directory Structure (Buf Workspace)

```
project-root/
├── buf.work.yaml              # 🌍 Root Buf Workspace (references all services)
│
├── api/proto/                 # 📋 Proto source files
│   ├── merchant/
│   │   └── merchant.proto
│   └── common/
│       ├── pagination.proto
│       └── money.proto
│
└── services/
    └── merchant-service/
        ├── buf.yaml           # 🔧 Per-service Buf module config
        └── buf.gen.yaml       # 🔧 Per-service code generation (Go + Gateway + Swagger)
```

### buf.work.yaml (Root)

```yaml
version: v1
directories:
  - api/proto
  - services/merchant-service
```

### buf.yaml (Per-Service)

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

### buf.gen.yaml (Per-Service, generates Go + Gateway + Swagger)

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen/go
    opt: paths=source_relative
  # HTTP Gateway (grpc-gateway)
  - remote: buf.build/grpc-ecosystem/gateway
    out: gen/go
    opt: paths=source_relative
  # Swagger/OpenAPI documentation
  - remote: buf.build/grpc-ecosystem/openapiv2
    out: ../../api/openapi
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

### Multi-Stage Dockerfile (Dev + Prod)

Each service has a Dockerfile supporting both development (with Air hot-reload) and production (minimal image):

```dockerfile
# services/xxx-service/Dockerfile

# === Stage 1: Base ===
FROM golang:1.23-alpine AS base
WORKDIR /app
RUN apk add --no-cache git
# Monorepo: copy root go.mod for dependency caching
COPY go.mod go.sum ./
RUN go mod download

# === Stage 2: Dev (with Air hot-reload) ===
FROM base AS dev
RUN go install github.com/air-verse/air@latest
RUN go install github.com/go-delve/delve/cmd/dlv@latest  # debugger (optional)
COPY . .
CMD ["air", "-c", "services/xxx-service/.air.toml"]

# === Stage 3: Builder (production build) ===
FROM base AS builder
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /server services/xxx-service/cmd/xxx-service/main.go

# === Stage 4: Production (minimal image) ===
FROM alpine:latest AS prod
WORKDIR /root/
COPY --from=builder /server .
CMD ["./server"]
```

**Key**: Build context is the monorepo root (not the service dir) so it can access `go.mod` and `pkg/`.

### Air Configuration (.air.toml)

```toml
# services/xxx-service/.air.toml
root = "."
tmp_dir = "tmp"

[build]
# Path from monorepo root (Docker WORKDIR is /app)
cmd = "go build -o ./tmp/main services/xxx-service/cmd/xxx-service/main.go"
bin = "./tmp/main"
include_ext = ["go", "tpl", "tmpl", "html"]
exclude_dir = ["assets", "tmp", "vendor", "test"]

[log]
time = true
```

### Docker Compose (Infrastructure + Services + LGTM)

```yaml
# docker-compose.yaml
services:
  # =========================================
  # 🏭 1. Microservices
  # =========================================
  merchant-service:
    build:
      context: .                    # Monorepo root (for go.mod + pkg/)
      dockerfile: services/merchant-service/Dockerfile
      target: dev                   # Use dev stage (Air hot-reload)
    volumes:
      - .:/app                      # Mount source for hot-reload
    ports:
      - "8080:8080"                 # HTTP Gateway
      - "9090:9090"                 # gRPC
    environment:
      - APP_ENV=dev
      - DB_SOURCE=postgresql://user:pass@postgres:5432/payuni?sslmode=disable
      - REDIS_ADDR=redis:6379
      - OTEL_EXPORTER_OTLP_ENDPOINT=otel-collector:4317
    depends_on:
      - postgres
      - redis
      - otel-collector

  # =========================================
  # 🏗️ 2. Infrastructure
  # =========================================
  postgres:
    image: postgres:15-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: payuni
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  # =========================================
  # 🔭 3. Observability (LGTM Stack)
  # =========================================
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./monitoring/otel-collector/otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports: ["4317:4317", "4318:4318", "8888:8888"]

  prometheus:
    image: prom/prometheus:latest
    command: ["--config.file=/etc/prometheus/prometheus.yml"]
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9091:9090"]

  loki:
    image: grafana/loki:latest
    command: -config.file=/etc/loki/local-config.yaml
    ports: ["3100:3100"]

  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./monitoring/tempo/tempo-config.yaml:/etc/tempo.yaml
    ports: ["3200:3200"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning

volumes:
  postgres_data:
```

### Monitoring Directory Structure

```
monitoring/
├── grafana/
│   └── provisioning/          # Datasources + Dashboards auto-provisioning
├── prometheus/
│   └── prometheus.yml         # Scrape config
├── loki/
│   └── local-config.yaml
├── tempo/
│   └── tempo-config.yaml
└── otel-collector/
    └── otel-collector-config.yaml   # OTLP receivers → exporters
```

### Makefile Targets

```makefile
.PHONY: dev-up dev-down migrate-all generate test lint

# Start everything (infra + observability + services)
dev-up:
	docker compose up -d

# Start only infrastructure
infra-up:
	docker compose up -d postgres redis

# Stop everything
dev-down:
	docker compose down

migrate-all:
	@for dir in services/*/; do \
		if [ -d "$$dir/db/migrations" ]; then \
			echo "Migrating $$(basename $$dir)..."; \
			atlas migrate apply --dir "file://$$dir/db/migrations" \
				--url "postgres://...?sslmode=disable"; \
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

lint:
	golangci-lint run ./...
	buf lint
	go-arch-lint check

test:
	go test ./... -race -cover -count=1

dev-%:  ## Run a specific service locally: make dev-merchant
	go run ./services/$*-service/cmd/$*-service/main.go
```

### Development Workflow

1. `make dev-up` — Start all (Infra + LGTM + Services with Air hot-reload)
2. `make migrate-all` — Apply all migrations
3. `make generate` — Generate Proto + sqlc code
4. Edit code → Air auto-rebuilds and restarts the service
5. View traces at `http://localhost:3000` (Grafana → Tempo)
6. View logs at `http://localhost:3000` (Grafana → Loki)
7. View metrics at `http://localhost:3000` (Grafana → Prometheus)

## Scheduled Jobs

See [scheduled-jobs.md](scheduled-jobs.md) for complete scheduled job implementation including:

- Dual entry points (Cron + API)
- Job UseCase pattern
- Distributed lock (Redis)
- Job execution history (audit log)
- Monitoring & alerting
