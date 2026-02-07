# gRPC Patterns

## Table of Contents

- [Error Handling Architecture](#error-handling-architecture)
- [Output Ports (Dependency Inversion)](#output-ports-dependency-inversion)
- [gRPC Interceptor Chain](#grpc-interceptor-chain)
- [Tracing Protocol](#tracing-protocol)
- [MQ Trace Context Propagation `[Async]`](#mq-trace-context-propagation-async)
- [API Pagination](#api-pagination)
- [DTO Mapping](#dto-mapping)
- [Context Deadline / Timeout Budget](#context-deadline--timeout-budget)
- [Authentication / Authorization](#authentication--authorization)

## Error Handling Architecture

### Mapping Chain

```
Domain Error (ErrorCode) → Interceptor (ErrorCode → codes.Code) → gRPC Status Code
```

| Layer | Handling |
|-------|----------|
| Domain | Define `Error` struct + `ErrorCode` enum (zero external deps) |
| Application | Return Domain Error directly, no mapping |
| Adapter (Interceptor) | Centrally map `ErrorCode` to gRPC Status |
| Infrastructure | LoggingInterceptor logs full error chain (incl. Unwrap) |

### Shared Error Interface (`pkg/errors/`)

`ErrorCode` is defined in `pkg/errors` (not `internal/domain`) to avoid circular dependency.
Domain layer imports `pkg/errors` to implement the interface; Interceptor imports it for mapping.
`pkg/errors` contains only pure constants and interfaces — no runtime or transport deps.

```go
// pkg/errors/domain.go
package errors

type ErrorCode int

const (
    ErrCodeNotFound           ErrorCode = iota // Resource not found
    ErrCodeAlreadyExists                       // Resource already exists (duplicate creation)
    ErrCodeInvalidInput                        // Invalid input parameters
    ErrCodePreconditionFailed                  // Business precondition not met
    ErrCodeConflict                            // Concurrent conflict (optimistic lock)
    ErrCodeAborted                             // Operation aborted (e.g., saga rollback)
    ErrCodeInternal                            // Unexpected internal error
    ErrCodeUnauthorized                        // Missing or invalid auth credentials
    ErrCodeForbidden                           // Authenticated but not authorized
    ErrCodeRateLimited                         // Rate limit exceeded
)

// DomainError — any error implementing this interface gets auto-mapped by Interceptor
type DomainError interface {
    error
    DomainCode() ErrorCode
}

// ClientMessage — optional interface to control client-facing message
// When not implemented, Interceptor uses default code description (prevents internal detail leakage)
type ClientMessage interface {
    ClientMsg() string
}
```

### Domain Error Definition (per-service `internal/domain/`)

Named `Error` (not `DomainError`) to avoid collision with `pkg/errors.DomainError` interface.

```go
// internal/domain/errors.go
type Error struct {
    code pkgerrors.ErrorCode
    msg  string
}
func (e *Error) Error() string                   { return e.msg }
func (e *Error) DomainCode() pkgerrors.ErrorCode { return e.code }
func (e *Error) ClientMsg() string               { return e.msg } // Business errors safe to expose

var (
    ErrInvalidTransition   = &Error{pkgerrors.ErrCodePreconditionFailed, "invalid status transition"}
    ErrOrderNotFound       = &Error{pkgerrors.ErrCodeNotFound, "order not found"}
    ErrOrderAlreadyExists  = &Error{pkgerrors.ErrCodeAlreadyExists, "order already exists"}
    ErrInsufficientBalance = &Error{pkgerrors.ErrCodePreconditionFailed, "insufficient balance"}
    ErrCurrencyMismatch    = &Error{pkgerrors.ErrCodeInvalidInput, "currency mismatch"}
    ErrOptimisticLock      = &Error{pkgerrors.ErrCodeConflict, "concurrent modification detected"}
)
```

### Error Wrapping with Stack Trace

使用 `WrapErr` 在錯誤中捕捉 file:line，方便 debug：

```go
// pkg/errors/wrap.go
import (
    "fmt"
    "path/filepath"
    "runtime"

    "github.com/pkg/errors"
)

// WrapErr 包裝錯誤並捕捉呼叫位置
func WrapErr(err error, msg string) error {
    if err == nil {
        return nil
    }
    _, file, line, ok := runtime.Caller(1)
    if !ok {
        return errors.Wrap(err, msg)
    }
    // 格式: "users.go:403-msg: original error"
    location := fmt.Sprintf("%s:%d", filepath.Base(file), line)
    if msg != "" {
        return errors.Wrap(err, fmt.Sprintf("%s-%s", location, msg))
    }
    return errors.Wrap(err, location)
}
```

**Usage**:
```go
func (s *userService) GetByID(ctx context.Context, id uuid.UUID) (*entity.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, pkgerrors.WrapErr(err, "repo.GetByID failed")
        // 輸出: "user_service.go:25-repo.GetByID failed: connection refused"
    }
    return user, nil
}
```

**Log 輸出範例**:
```json
{
    "level": "error",
    "msg": "user_service.go:25-repo.GetByID failed: connection refused",
    "stack": "..."
}
```

### Centralized Error Mapping Interceptor

```go
// pkg/middleware/grpc/interceptor/error_mapping.go
var domainCodeToGRPC = map[pkgerrors.ErrorCode]codes.Code{
    pkgerrors.ErrCodeNotFound:           codes.NotFound,
    pkgerrors.ErrCodeAlreadyExists:      codes.AlreadyExists,
    pkgerrors.ErrCodeInvalidInput:       codes.InvalidArgument,
    pkgerrors.ErrCodePreconditionFailed: codes.FailedPrecondition,
    pkgerrors.ErrCodeConflict:           codes.Aborted,
    pkgerrors.ErrCodeAborted:            codes.Aborted,
    pkgerrors.ErrCodeInternal:           codes.Internal,
    pkgerrors.ErrCodeUnauthorized:       codes.Unauthenticated,
    pkgerrors.ErrCodeForbidden:          codes.PermissionDenied,
    pkgerrors.ErrCodeRateLimited:        codes.ResourceExhausted,
}

func ErrorMappingInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (interface{}, error) {
        resp, err := handler(ctx, req)
        if err == nil { return resp, nil }

        // 1. Check DomainError → map to gRPC Status
        var domErr pkgerrors.DomainError
        if stderrors.As(err, &domErr) {
            grpcCode, ok := domainCodeToGRPC[domErr.DomainCode()]
            if !ok { grpcCode = codes.Internal }
            msg := domErr.Error()
            if cm, ok := err.(pkgerrors.ClientMessage); ok { msg = cm.ClientMsg() }
            return nil, status.Error(grpcCode, msg)
        }
        // 2. Already a gRPC Status Error (propagated from downstream) → pass through
        if _, ok := status.FromError(err); ok { return nil, err }
        // 3. Unknown error → Internal (don't expose details)
        return nil, status.Error(codes.Internal, "internal error")
    }
}
```

**Key**: UseCase returns Domain Errors directly — no manual mapping. `errors.As` can unwrap through `fmt.Errorf("...: %w", err)` chains.

### Why Not Use `grpc/codes` Directly in Domain

Domain semantics get hijacked by transport protocol. For example, `FailedPrecondition` maps to `400 Bad Request` in gRPC-HTTP transcoding, but business logic might need `409 Conflict` or `422 Unprocessable Entity`. Using Domain-level `ErrorCode` lets each Adapter (gRPC Interceptor, HTTP Middleware) define its own mapping rules. Domain layer stays truly portable.

## Output Ports (Dependency Inversion)

Define an interface for every external dependency in Application layer. Enables testing and replacement.

```go
// internal/application/port/output/some_client.go
type SomeClient interface {
    DoSomething(ctx context.Context, id uuid.UUID) error
}

// internal/application/port/output/tx_manager.go
type TxManager interface {
    WithTx(ctx context.Context, fn func(txCtx context.Context) error) error
}
```

UseCase depends on Output Port interfaces, never on concrete implementations. Adapter layer implements these interfaces and Uber Fx wires them together.

## gRPC Interceptor Chain

Order matters — outermost executes first on request, last on response:

```go
grpc.ChainUnaryInterceptor(
    otelgrpc.UnaryServerInterceptor(),          // 1. OTel tracing (outermost)
    interceptor.ServerCorrelationInterceptor(),  // 2. correlation_id / request_id
    interceptor.LoggingInterceptor(logger),      // 3. Request logging (logs unauthenticated attempts too)
    interceptor.RecoveryInterceptor(logger),     // 4. Panic recovery
    interceptor.MetricsInterceptor(metrics),     // 5. Prometheus metrics
    interceptor.RateLimitInterceptor(limiter),   // 6. Rate limiting (before auth to reject floods early)
    interceptor.AuthInterceptor(jwtValidator),   // 7. Authentication (after rate limit)
    interceptor.ErrorMappingInterceptor(),       // 8. Error mapping (innermost)
)
```

### RateLimitInterceptor Implementation

Server-side rate limiting per method to protect against traffic floods:

```go
// pkg/middleware/grpc/interceptor/rate_limit.go
type RateLimiter struct {
    limiters map[string]*rate.Limiter  // per-method limiter
    global   *rate.Limiter             // global fallback
}

func NewRateLimiter(globalRPS float64, globalBurst int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        global:   rate.NewLimiter(rate.Limit(globalRPS), globalBurst),
    }
}

// WithMethodLimit sets a per-method rate limit
func (rl *RateLimiter) WithMethodLimit(method string, rps float64, burst int) *RateLimiter {
    rl.limiters[method] = rate.NewLimiter(rate.Limit(rps), burst)
    return rl
}

func RateLimitInterceptor(rl *RateLimiter) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (interface{}, error) {
        limiter := rl.global
        if ml, ok := rl.limiters[info.FullMethod]; ok {
            limiter = ml
        }
        if !limiter.Allow() {
            return nil, status.Error(codes.ResourceExhausted, "rate limit exceeded")
        }
        return handler(ctx, req)
    }
}
```

## Tracing Protocol

| ID | Purpose | Producer |
|----|---------|----------|
| `trace_id` | Distributed tracing ID | OTel SDK auto-generated |
| `correlation_id` | Business correlation ID | Gateway generates UUIDv4 |
| `request_id` | Single entry request ID | Gateway generates UUIDv4 |

## MQ Trace Context Propagation `[Async]`

gRPC trace context is auto-propagated by OTel interceptor. MQ (RabbitMQ) requires manual injection/extraction, otherwise publisher → consumer trace chain breaks in Tempo.

```go
// pkg/mq/rabbitmq/trace.go
type amqpHeaderCarrier amqp.Table

func (c amqpHeaderCarrier) Get(key string) string {
    if val, ok := c[key]; ok {
        if s, ok := val.(string); ok { return s }
    }
    return ""
}
func (c amqpHeaderCarrier) Set(key, val string) { c[key] = val }
func (c amqpHeaderCarrier) Keys() []string {
    keys := make([]string, 0, len(c))
    for k := range c { keys = append(keys, k) }
    return keys
}

func InjectTraceContext(ctx context.Context, headers amqp.Table) {
    otel.GetTextMapPropagator().Inject(ctx, amqpHeaderCarrier(headers))
}

func ExtractTraceContext(ctx context.Context, headers amqp.Table) context.Context {
    return otel.GetTextMapPropagator().Extract(ctx, amqpHeaderCarrier(headers))
}
```

### Publisher Implementation

```go
func (p *RabbitMQPublisher) Publish(ctx context.Context, eventType string, eventVersion int, payload []byte) error {
    headers := amqp.Table{
        "event_type":    eventType,
        "event_version": int32(eventVersion),
    }
    // Inject trace context (traceparent, tracestate) + correlation_id
    rabbitmq.InjectTraceContext(ctx, headers)
    if cid, ok := ctxutil.CorrelationID(ctx); ok {
        headers["correlation_id"] = cid
    }

    return p.channel.PublishWithContext(ctx, p.exchange, eventType, false, false,
        amqp.Publishing{
            ContentType: "application/json",
            Headers:     headers,
            Body:        payload,
        })
}
```

### Consumer Implementation

```go
func (c *Consumer) handleDelivery(msg amqp.Delivery) {
    // Restore trace context from AMQP headers — continues upstream trace
    ctx := rabbitmq.ExtractTraceContext(context.Background(), msg.Headers)

    // Create consumer span — automatically becomes child of publisher span
    ctx, span := otel.Tracer("consumer").Start(ctx, "consume."+msg.RoutingKey)
    defer span.End()

    // Restore correlation_id
    if cid, ok := msg.Headers["correlation_id"].(string); ok {
        ctx = ctxutil.WithCorrelationID(ctx, cid)
    }

    // Read event version for progressive migration handling
    // getHeaderInt: see async-patterns.md — MQ Consumer Patterns for implementation
    eventVersion := getHeaderInt(msg.Headers, "event_version", 1)

    // Normal processing with idempotency check...
}
```

## API Pagination

### Strategy Selection

| Strategy | Use Case | Trade-offs |
|----------|----------|------------|
| **Offset-based** | Admin dashboard | Supports page jumping; `OFFSET` slow on large datasets |
| **Cursor-based** | Consumer-facing lists (infinite scroll) | O(1) performance; no page jumping |

### Proto Definitions

```protobuf
// Offset-based
message PaginationRequest {
  int32 page_size = 1;  // Server must clamp to [1, max_size]
  int32 page = 2;       // Starting from 1
}
message PaginationResponse {
  int32 total_count = 1;
  int32 total_pages = 2;
  int32 current_page = 3;
  int32 page_size = 4;
}

// Cursor-based
message CursorPaginationRequest {
  int32 page_size = 1;
  string cursor = 2;    // Opaque token, empty string = first page
}
message CursorPaginationResponse {
  string next_cursor = 1;  // Empty string = no more data
  bool has_more = 2;
}
```

### Implementation Rules

1. **Page size limits**: Default (e.g., 20) and max (e.g., 100). Clamp above max — do NOT return error. Use default for `<= 0`.
2. **Cursor safety (opaque token)**: Never expose DB primary key or raw offset. Recommended: `base64(url_safe_json({"id":"...","sort_val":"..."}))`. Prevents client from guessing cursor structure; preserves server-side flexibility.
3. **Sort field whitelist**: Each RPC must explicitly define allowed `sort_by` fields. NEVER concatenate client-provided strings into SQL `ORDER BY` (prevents SQL injection and performance issues).
4. **Cursor SQL (Keyset Pagination)**: Use `(sort_column, unique_id)` composite key. Sort MUST include a unique key (usually PK), otherwise duplicate sort values cause missing or duplicate rows.

### Pagination Helper (防 DOS + 標準化)

```go
// pkg/pagination/helper.go
const (
    DefaultPageIndex = 1
    DefaultPageSize  = 20
    MaxPageSize      = 100  // 防止 DOS 攻擊
    MinPageSize      = 1
)

// NormalizePageParams 標準化分頁參數，防止無效或惡意輸入
func NormalizePageParams(pageIndex, pageSize *int) (idx int, size int) {
    idx = DefaultPageIndex
    if pageIndex != nil && *pageIndex > 0 {
        idx = *pageIndex
    }

    size = DefaultPageSize
    if pageSize != nil {
        switch {
        case *pageSize > MaxPageSize:
            size = MaxPageSize  // Clamp，不要報錯
        case *pageSize < MinPageSize:
            size = MinPageSize
        default:
            size = *pageSize
        }
    }
    return idx, size
}

// CalculateOffset 計算 SQL OFFSET
func CalculateOffset(pageIndex, pageSize int) int {
    return (pageIndex - 1) * pageSize
}
```

**Usage**:
```go
func (u *OrderUseCase) List(ctx context.Context, req *request.ListOrders) (*response.OrderList, error) {
    pageIdx, pageSize := pagination.NormalizePageParams(req.PageIndex, req.PageSize)
    offset := pagination.CalculateOffset(pageIdx, pageSize)
    // ...
}
```

```sql
-- Cursor-based example
SELECT * FROM orders
WHERE user_id = $1
  AND (created_at, id) < ($2, $3)  -- Decoded cursor values
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

### Architecture Boundary

Pagination is a data query infrastructure detail — it must NOT pollute Domain Logic. Cursor struct is defined in Application layer DTO or Adapter layer. Repository interface accepts pagination parameters; Domain layer is unaware of cursor implementation.

```go
type OrderRepository interface {
    ListByUserID(ctx context.Context, userID uuid.UUID, cursor *pagination.Cursor, pageSize int) ([]*entity.Order, error)
}
```

## DTO Mapping

Use manual mapper functions for compile-time type safety. Pair with `lo.Map` for collection mapping.

```go
// internal/adapter/inbound/grpc/mapper/order_mapper.go

// Entity → gRPC Response
func OrderToResponse(e *entity.Order) *pb.OrderResponse {
    return &pb.OrderResponse{
        Id: e.ID.String(), Status: e.Status.String(),
        Amount: e.TotalAmount.Amount, Currency: e.TotalAmount.Currency,
        CreatedAt: timestamppb.New(e.CreatedAt),
    }
}

func OrdersToResponses(orders []*entity.Order) []*pb.OrderResponse {
    return lo.Map(orders, func(e *entity.Order, _ int) *pb.OrderResponse {
        return OrderToResponse(e)
    })
}

// gRPC Request → Application DTO (with validation)
func CreateRequestToDTO(req *pb.CreateOrderRequest) (*dto.CreateOrderRequest, error) {
    userID, err := uuid.Parse(req.UserId)
    if err != nil {
        return nil, fmt.Errorf("invalid user_id: %w", err)
    }
    items := make([]dto.OrderItem, 0, len(req.Items))
    for _, i := range req.Items {
        productID, err := uuid.Parse(i.ProductId)
        if err != nil {
            return nil, fmt.Errorf("invalid product_id: %w", err)
        }
        items = append(items, dto.OrderItem{ProductID: productID, Quantity: int(i.Quantity)})
    }
    return &dto.CreateOrderRequest{UserID: userID, Items: items}, nil
}
```

**Benefits**: Compiler catches field additions/removals/renames. External input (gRPC Request) uses `uuid.Parse` returning error — avoids panics from invalid input.

## Context Deadline / Timeout Budget

Microservice cascading failures often start from missing or mismanaged timeouts. Every outbound call must have a deadline.

### Timeout Budget Propagation

```
Gateway (10s total)
  → Service A (gRPC deadline = 10s, inherited from gateway)
       → Service B (gRPC call, must < remaining budget)
            → DB query (must < remaining budget)
```

gRPC propagates deadline automatically via metadata. The key rule is: **each layer subtracts its own processing overhead and passes a shorter deadline downstream.**

### Implementation

```go
// UseCase: compute remaining budget before calling downstream
func (uc *UseCase) Execute(ctx context.Context, req *Request) (*Response, error) {
    // Check if deadline already expired before doing work
    if err := ctx.Err(); err != nil {
        return nil, fmt.Errorf("context already expired: %w", err)
    }

    // Reserve buffer for own processing (DB write, event collection, etc.)
    const ownOverhead = 500 * time.Millisecond
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline) - ownOverhead
        if remaining <= 0 {
            return nil, status.Error(codes.DeadlineExceeded, "insufficient time budget")
        }
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, remaining)
        defer cancel()
    }

    // Call downstream with reduced budget (gRPC auto-propagates ctx deadline)
    result, err := uc.inventoryClient.Reserve(ctx, req.Items)
    if err != nil { return nil, err }

    // ... remaining work uses same ctx
}
```

### Timeout Guidelines

| Call Type | Recommended Timeout | Notes |
|-----------|-------------------|-------|
| Gateway → Service | 10–30s | Total request budget |
| Service → Service (gRPC) | Inherit from ctx | gRPC propagates deadline automatically |
| DB query | 3–5s | Set at pool level (`config.ConnConfig.QueryTimeout`) |
| Redis operation | 1–3s | Set via `redis.Options.ReadTimeout` |
| MQ publish | 3–5s | Set via `amqp.Channel.PublishWithContext` |
| Saga step (sync) | 5s per step | As defined in saga-patterns.md |

### Common Mistakes

1. **No timeout at all**: Goroutine blocks forever → goroutine leak → OOM
2. **Hardcoded timeout ignoring incoming deadline**: Service A gives 3s, but Service B creates fresh 10s context → B keeps working after A already returned error to client
3. **Same timeout for all steps**: 3 sequential calls each with 5s = 15s possible, but incoming deadline is 10s
4. **Compensation with caller's context**: Original context cancelled → compensation fails. Always use `context.WithoutCancel` for compensation (see saga-patterns.md)

## Authentication / Authorization

### JWT Interceptor

```go
// pkg/middleware/grpc/interceptor/auth.go

type AuthInterceptor struct {
    jwtValidator *jwt.Validator
    // Methods that skip authentication (e.g., health check, public endpoints)
    publicMethods map[string]bool
}

func NewAuthInterceptor(v *jwt.Validator, publicMethods []string) *AuthInterceptor {
    pm := make(map[string]bool, len(publicMethods))
    for _, m := range publicMethods { pm[m] = true }
    return &AuthInterceptor{jwtValidator: v, publicMethods: pm}
}

func (i *AuthInterceptor) Unary() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (interface{}, error) {

        // Skip auth for public methods
        if i.publicMethods[info.FullMethod] {
            return handler(ctx, req)
        }

        // Extract token from metadata
        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, "missing metadata")
        }
        tokens := md.Get("authorization")
        if len(tokens) == 0 {
            return nil, status.Error(codes.Unauthenticated, "missing authorization header")
        }
        tokenStr := strings.TrimPrefix(tokens[0], "Bearer ")

        // Validate and extract claims
        claims, err := i.jwtValidator.Validate(tokenStr)
        if err != nil {
            return nil, status.Error(codes.Unauthenticated, "invalid token")
        }

        // Inject user context for downstream use
        ctx = ctxutil.WithUserID(ctx, claims.UserID)
        ctx = ctxutil.WithUserRoles(ctx, claims.Roles)

        return handler(ctx, req)
    }
}
```

### User Context Propagation

```go
// pkg/ctxutil/auth.go
type ctxKey string
const (
    userIDKey   ctxKey = "user_id"
    userRolesKey ctxKey = "user_roles"
)

func WithUserID(ctx context.Context, id uuid.UUID) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}
func UserIDFrom(ctx context.Context) (uuid.UUID, error) {
    id, ok := ctx.Value(userIDKey).(uuid.UUID)
    if !ok { return uuid.Nil, fmt.Errorf("user_id not found in context") }
    return id, nil
}

func WithUserRoles(ctx context.Context, roles []string) context.Context {
    return context.WithValue(ctx, userRolesKey, roles)
}
func UserRolesFrom(ctx context.Context) []string {
    roles, _ := ctx.Value(userRolesKey).([]string)
    return roles
}
```

### Generic Context Utilities (進階)

使用泛型實現類型安全的 context Get/Set，避免 key 衝突：

```go
// pkg/ctxutil/generic.go
package ctxutil

import "context"

// typedKey uses Go's type system as the context key.
// Different T types produce different key types → no collisions, no reflection.
type typedKey[T any] struct{}

// Get 從 context 取得指定類型的值
func Get[T any](ctx context.Context) (T, bool) {
    v, ok := ctx.Value(typedKey[T]{}).(T)
    return v, ok
}

// Set 在 context 中設定指定類型的值
func Set[T any](ctx context.Context, val T) context.Context {
    return context.WithValue(ctx, typedKey[T]{}, val)
}
```

**Why `typedKey[T]` instead of `reflect.TypeOf`**: The reflect-based approach panics when `T` is an interface type (zero value of interface → `reflect.TypeOf` returns nil). Using a generic struct key leverages Go's type system directly — each concrete `T` produces a unique key type at compile time, with zero runtime cost and no reflection.

**Usage**:
```go
// 定義 context 攜帶的資料類型
type RequestInfo struct {
    TraceID   string
    UserAgent string
    IP        string
}

// 在 middleware 中設定
func RequestInfoMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        info := RequestInfo{
            TraceID:   r.Header.Get("X-Trace-ID"),
            UserAgent: r.UserAgent(),
            IP:        r.RemoteAddr,
        }
        ctx := ctxutil.Set(r.Context(), info)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 在 handler/usecase 中取得
func (h *Handler) Handle(ctx context.Context, req *Request) error {
    info, ok := ctxutil.Get[RequestInfo](ctx)
    if ok {
        log.Info("processing request", zap.String("trace_id", info.TraceID))
    }
    // ...
}
```

**優點**: 類型自動作為 key，不需要手動定義 key 常數，避免 typo 和衝突。

### TraceID Context with ToLog()

提供 TraceID 的 context helper，附帶 `ToLog()` 方法方便在任何地方輸出 structured log。

> **Note**: For complete logging and tracing implementation, see [observability.md](observability.md).

```go
// pkg/ctxutil/trace_id.go
package ctxutil

import (
    "context"
    "go.uber.org/zap"
)

var TraceID = new(traceIDKey)

type traceIDModel struct {
    val string
}

type traceIDKey struct{}

func (t *traceIDKey) Set(ctx context.Context, val string) context.Context {
    return Set(ctx, traceIDModel{val})
}

func (t *traceIDKey) Get(ctx context.Context) string {
    if v, ok := Get[traceIDModel](ctx); ok {
        return v.val
    }
    return "unknown"
}

// ToLog 返回 zap.Field，方便在任何地方 log trace ID
func (t *traceIDKey) ToLog(ctx context.Context) zap.Field {
    return zap.String("traceID", t.Get(ctx))
}
```

**Usage**:
```go
// 在 Interceptor 中設定
func TraceIDInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (interface{}, error) {

        traceID := uuid.New().String()
        if md, ok := metadata.FromIncomingContext(ctx); ok {
            if ids := md.Get("x-trace-id"); len(ids) > 0 {
                traceID = ids[0]
            }
        }
        ctx = ctxutil.TraceID.Set(ctx, traceID)
        return handler(ctx, req)
    }
}

// 在任何地方 log
func (u *OrderUseCase) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    zap.L().Info("creating order",
        ctxutil.TraceID.ToLog(ctx),  // 自動輸出 "traceID": "xxx"
        zap.Int64("buyerID", req.BuyerID),
    )
    // ...
}
```

### RequestInfo Context

攜帶請求的 metadata（IP、UserAgent 等），用於 audit log 或安全檢查：

```go
// pkg/ctxutil/request_info.go
package ctxutil

import (
    "context"
    "time"
    "go.uber.org/zap"
)

var RequestInfo = new(requestInfoKey)

type RequestInfoModel struct {
    RequestTime   time.Time // 請求當下時間
    Host          string
    ClientIP      string
    ConnectingIP  string    // Load balancer 連接 IP
    IPCountry     string    // GeoIP 國家代碼
    UserAgent     string
    XForwardedFor string
}

type requestInfoKey struct{}

func (t *requestInfoKey) Set(ctx context.Context, val RequestInfoModel) context.Context {
    return Set(ctx, val)
}

func (t *requestInfoKey) Get(ctx context.Context) *RequestInfoModel {
    if v, ok := Get[RequestInfoModel](ctx); ok {
        return &v
    }
    return nil
}

// ToLog 返回常用的 log fields
func (t *requestInfoKey) ToLog(ctx context.Context) []zap.Field {
    info := t.Get(ctx)
    if info == nil {
        return nil
    }
    return []zap.Field{
        zap.String("clientIP", info.ClientIP),
        zap.String("userAgent", info.UserAgent),
    }
}
```

**Usage**:
```go
// HTTP Gateway middleware
func RequestInfoMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        info := ctxutil.RequestInfoModel{
            RequestTime:   time.Now(),
            Host:          c.Request.Host,
            ClientIP:      c.ClientIP(),
            ConnectingIP:  c.Request.RemoteAddr,
            UserAgent:     c.Request.UserAgent(),
            XForwardedFor: c.GetHeader("X-Forwarded-For"),
            IPCountry:     c.GetHeader("CF-IPCountry"), // Cloudflare
        }
        ctx := ctxutil.RequestInfo.Set(c.Request.Context(), info)
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}

// 在 UseCase 中使用（安全檢查、audit log）
func (u *PaymentUseCase) ProcessPayment(ctx context.Context, req PaymentRequest) error {
    info := ctxutil.RequestInfo.Get(ctx)
    if info != nil && info.IPCountry == "CN" {
        return ErrRegionRestricted
    }

    // Audit log
    zap.L().Info("processing payment",
        ctxutil.TraceID.ToLog(ctx),
        zap.String("clientIP", info.ClientIP),
        zap.Int64("amount", req.Amount),
    )
    // ...
}
```

### Cross-Service Auth Propagation

When Service A calls Service B via gRPC, forward the auth token:

```go
// pkg/middleware/grpc/interceptor/auth_propagation.go
func AuthPropagationInterceptor() grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{},
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        // Extract token from incoming context and forward to outgoing
        md, ok := metadata.FromIncomingContext(ctx)
        if ok {
            if tokens := md.Get("authorization"); len(tokens) > 0 {
                ctx = metadata.AppendToOutgoingContext(ctx, "authorization", tokens[0])
            }
        }
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

### Architecture Placement

| Component | Location | Layer |
|-----------|----------|-------|
| JWT validation logic | `pkg/auth/jwt/` | Shared |
| Auth Interceptor | `pkg/middleware/grpc/interceptor/` | Shared |
| User context helpers | `pkg/ctxutil/` | Shared |
| Auth propagation (client) | `pkg/middleware/grpc/interceptor/` | Shared |
| Role-based access control | UseCase or dedicated Interceptor | Application |

---

**Related Documents:**
- [HTTP Gateway Architecture](http-gateway.md) - Rate limiting, request logging, external client patterns
- [Async Patterns](async-patterns.md) - MQ consumer patterns, Saga, Outbox
