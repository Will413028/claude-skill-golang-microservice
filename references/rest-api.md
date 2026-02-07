# REST API Patterns

Patterns for building RESTful APIs with Go (Gin), following Clean Architecture principles.

## Table of Contents

- [When to Use REST vs gRPC](#when-to-use-rest-vs-grpc)
- [Directory Structure](#directory-structure)
- [RESTful URL Design](#restful-url-design)
- [HTTP Method Semantics](#http-method-semantics)
- [Unified Response Format](#unified-response-format)
- [Request DTO with Validation](#request-dto-with-validation)
- [REST Handler (Controller)](#rest-handler-controller)
- [Error Handling: DomainError to HTTP Status](#error-handling-domainerror-to-http-status)
- [Middleware Chain](#middleware-chain)
- [Pagination](#pagination)
- [File Upload](#file-upload)
- [API Versioning](#api-versioning)

## When to Use REST vs gRPC

| Scenario | Recommended | Rationale |
|----------|-------------|-----------|
| Frontend → Backend | REST | Browser-native, easy debugging, wide tooling |
| Backend → Backend (internal) | gRPC | Type-safe, efficient, streaming support |
| Public API | REST | Universal compatibility, OpenAPI docs |
| Mobile App | REST or gRPC | REST simpler; gRPC if performance critical |
| Real-time / Streaming | gRPC or WebSocket | Bidirectional streaming |

**Hybrid Pattern**: Use REST for external-facing APIs, gRPC for internal service communication.

```
Frontend (REST) ──→ API Service (Gin) ──→ Internal Services (gRPC)
                         │
                         └── Direct DB access for simple CRUD
```

## Directory Structure

For a service exposing REST APIs directly (not as a gateway):

```
internal/
├── adapter/
│   └── inbound/
│       └── rest/                    # REST adapter layer
│           ├── handler/
│           │   ├── order_handler.go
│           │   ├── order_dto.go     # Request/Response DTOs
│           │   ├── user_handler.go
│           │   ├── user_dto.go
│           │   └── di.go
│           ├── middleware/
│           │   ├── auth.go
│           │   ├── logging.go
│           │   ├── recovery.go
│           │   ├── ratelimit.go
│           │   └── di.go
│           ├── router/
│           │   ├── router.go        # Route registration
│           │   └── di.go
│           └── response/
│               └── response.go      # Unified response helpers
├── application/
│   └── usecase/                     # Business logic (unchanged)
└── domain/                          # Domain layer (unchanged)
```

## RESTful URL Design

### Resource Naming Rules

| Rule | Good | Bad |
|------|------|-----|
| Use nouns, not verbs | `/orders` | `/getOrders` |
| Use plural nouns | `/users/:id` | `/user/:id` |
| Use kebab-case | `/order-items` | `/orderItems`, `/order_items` |
| Nest for relationships | `/users/:id/orders` | `/getUserOrders` |
| Max 2 levels of nesting | `/users/:id/orders` | `/users/:id/orders/:oid/items/:iid/details` |

### Common URL Patterns

```
# Collection
GET    /api/v1/orders              # List orders (with pagination)
POST   /api/v1/orders              # Create order

# Single Resource
GET    /api/v1/orders/:id          # Get order by ID
PUT    /api/v1/orders/:id          # Full update (replace)
PATCH  /api/v1/orders/:id          # Partial update
DELETE /api/v1/orders/:id          # Delete order

# Nested Resources
GET    /api/v1/users/:id/orders    # List user's orders
POST   /api/v1/users/:id/orders    # Create order for user

# Actions (when CRUD doesn't fit)
POST   /api/v1/orders/:id/cancel   # Cancel order (state transition)
POST   /api/v1/orders/:id/refund   # Refund order
POST   /api/v1/auth/login          # Login (action, not resource)
POST   /api/v1/auth/logout         # Logout

# Search / Filter
GET    /api/v1/orders?status=pending&created_after=2024-01-01
GET    /api/v1/products/search?q=keyword
```

## HTTP Method Semantics

| Method | Idempotent | Safe | Use Case |
|--------|------------|------|----------|
| GET | Yes | Yes | Retrieve resource(s) |
| POST | No | No | Create resource, trigger action |
| PUT | Yes | No | Full replacement of resource |
| PATCH | No* | No | Partial update |
| DELETE | Yes | No | Remove resource |

*PATCH can be idempotent if using JSON Merge Patch semantics.

### Method Selection Guide

```go
// GET: Retrieve (no side effects)
GET /orders/:id

// POST: Create new resource (server assigns ID)
POST /orders
Body: { "items": [...] }
Response: 201 Created + Location: /orders/123

// PUT: Full replacement (client provides complete resource)
PUT /orders/:id
Body: { "id": "123", "status": "confirmed", "items": [...] }

// PATCH: Partial update (only changed fields)
PATCH /orders/:id
Body: { "status": "confirmed" }

// DELETE: Remove resource
DELETE /orders/:id
Response: 204 No Content
```

## Unified Response Format

### Design Principle: HTTP Status + Business Code

Use **both** standard HTTP status codes AND custom business codes:

| Layer | Purpose | Example |
|-------|---------|---------|
| **HTTP Status** | Transport-level semantics (success/failure category) | 200, 400, 404, 500 |
| **Body `code`** | Business-level details (specific error reason) | 0, 1001, 1002 |

```
HTTP 404 Not Found
{"code": 1001, "msg": "order not found"}

HTTP 400 Bad Request
{"code": 1002, "msg": "invalid order status transition"}

HTTP 200 OK
{"code": 0, "data": {"id": "123", "status": "confirmed"}}
```

### Why NOT Return HTTP 200 for Everything?

**Anti-pattern**: Some APIs return HTTP 200 for all responses and rely solely on `code` field.

```json
// BAD: HTTP 200 OK
{"code": 1001, "msg": "order not found"}
```

**Problems with this approach:**
1. **Monitoring blind spots** — APM tools (Datadog, New Relic) can't detect error rates
2. **CDN/Proxy caching issues** — May incorrectly cache error responses
3. **Browser fetch API** — `response.ok` is true, won't trigger catch block
4. **Violates HTTP semantics** — Confusing for developers familiar with REST standards
5. **Load balancer health checks** — May route traffic to unhealthy instances

**Correct approach**: HTTP status reflects success/failure, `code` provides details.

### Response Structure

```go
// internal/adapter/inbound/rest/response/response.go
package response

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

// Success response: HTTP 2xx + {"code": 0, "data": {...}}
type SuccessResponse struct {
    Code int         `json:"code"`
    Data interface{} `json:"data"`
}

// Error response: HTTP 4xx/5xx + {"code": <error_code>, "msg": "..."}
type ErrorResponse struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}

// Paginated response: {"code": 0, "data": {...}, "meta": {...}}
type PaginatedResponse struct {
    Code int         `json:"code"`
    Data interface{} `json:"data"`
    Meta *PageMeta   `json:"meta,omitempty"`
}

type PageMeta struct {
    Page       int   `json:"page"`
    PageSize   int   `json:"page_size"`
    Total      int64 `json:"total"`
    TotalPages int   `json:"total_pages"`
}

// Helper functions
func OK(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, SuccessResponse{Code: 0, Data: data})
}

func Created(c *gin.Context, data interface{}) {
    c.JSON(http.StatusCreated, SuccessResponse{Code: 0, Data: data})
}

func NoContent(c *gin.Context) {
    c.Status(http.StatusNoContent)
}

func OKWithPage(c *gin.Context, data interface{}, meta *PageMeta) {
    c.JSON(http.StatusOK, PaginatedResponse{Code: 0, Data: data, Meta: meta})
}

func Error(c *gin.Context, httpStatus int, code int, msg string) {
    c.JSON(httpStatus, ErrorResponse{Code: code, Msg: msg})
}

func BadRequest(c *gin.Context, code int, msg string) {
    Error(c, http.StatusBadRequest, code, msg)
}

func NotFound(c *gin.Context, msg string) {
    Error(c, http.StatusNotFound, 404, msg)
}

func Unauthorized(c *gin.Context, msg string) {
    Error(c, http.StatusUnauthorized, 401, msg)
}

func Forbidden(c *gin.Context, msg string) {
    Error(c, http.StatusForbidden, 403, msg)
}

func InternalError(c *gin.Context) {
    Error(c, http.StatusInternalServerError, 500, "internal server error")
}
```

### Response Examples

```json
// Success
{
  "code": 0,
  "data": {
    "id": "ord_123",
    "status": "pending",
    "total": 9900
  }
}

// Success with pagination
{
  "code": 0,
  "data": [
    {"id": "ord_123", "status": "pending"},
    {"id": "ord_124", "status": "confirmed"}
  ],
  "meta": {
    "page": 1,
    "page_size": 20,
    "total": 42,
    "total_pages": 3
  }
}

// Error
{
  "code": 1001,
  "msg": "order not found"
}
```

## Request DTO with Validation

Use Gin's binding tags for validation:

```go
// internal/adapter/inbound/rest/handler/order_dto.go
package handler

import "time"

// === Request DTOs ===

type CreateOrderRequest struct {
    Items []CreateOrderItemRequest `json:"items" binding:"required,min=1,dive"`
}

type CreateOrderItemRequest struct {
    ProductID string `json:"product_id" binding:"required,uuid"`
    Quantity  int    `json:"quantity" binding:"required,min=1,max=100"`
}

type UpdateOrderRequest struct {
    Status string `json:"status" binding:"required,oneof=confirmed cancelled"`
}

type ListOrdersRequest struct {
    Page      int    `form:"page" binding:"omitempty,min=1"`
    PageSize  int    `form:"page_size" binding:"omitempty,min=1,max=100"`
    Status    string `form:"status" binding:"omitempty,oneof=pending confirmed shipped"`
    CreatedAt string `form:"created_after" binding:"omitempty,datetime=2006-01-02"`
}

// === Response DTOs ===

type OrderResponse struct {
    ID        string    `json:"id"`
    Status    string    `json:"status"`
    Total     int64     `json:"total"`
    Currency  string    `json:"currency"`
    CreatedAt time.Time `json:"created_at"`
}

// === Mapper functions ===

func ToOrderResponse(order *entity.Order) *OrderResponse {
    return &OrderResponse{
        ID:        order.ID.String(),
        Status:    order.Status.String(),
        Total:     order.TotalAmount.Amount,
        Currency:  order.TotalAmount.Currency,
        CreatedAt: order.CreatedAt,
    }
}

func ToOrderListResponse(orders []*entity.Order) []*OrderResponse {
    result := make([]*OrderResponse, len(orders))
    for i, order := range orders {
        result[i] = ToOrderResponse(order)
    }
    return result
}
```

### Common Validation Tags

| Tag | Description | Example |
|-----|-------------|---------|
| `required` | Field must be present | `binding:"required"` |
| `omitempty` | Skip validation if empty | `binding:"omitempty,min=1"` |
| `min`, `max` | Numeric/string length bounds | `binding:"min=1,max=100"` |
| `len` | Exact length | `binding:"len=11"` |
| `oneof` | Enum values | `binding:"oneof=pending confirmed"` |
| `email` | Email format | `binding:"email"` |
| `uuid` | UUID format | `binding:"uuid"` |
| `url` | URL format | `binding:"url"` |
| `datetime` | Date/time format | `binding:"datetime=2006-01-02"` |
| `dive` | Validate slice elements | `binding:"required,dive"` |

## REST Handler (Controller)

```go
// internal/adapter/inbound/rest/handler/order_handler.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    "myapp/internal/adapter/inbound/rest/response"
    "myapp/internal/application/port/input"
)

type OrderHandler struct {
    createOrderUC input.CreateOrderUseCase
    getOrderUC    input.GetOrderUseCase
    listOrdersUC  input.ListOrdersUseCase
    updateOrderUC input.UpdateOrderUseCase
    deleteOrderUC input.DeleteOrderUseCase
    logger        *zap.Logger
}

func NewOrderHandler(
    createOrderUC input.CreateOrderUseCase,
    getOrderUC input.GetOrderUseCase,
    listOrdersUC input.ListOrdersUseCase,
    updateOrderUC input.UpdateOrderUseCase,
    deleteOrderUC input.DeleteOrderUseCase,
    logger *zap.Logger,
) *OrderHandler {
    return &OrderHandler{
        createOrderUC: createOrderUC,
        getOrderUC:    getOrderUC,
        listOrdersUC:  listOrdersUC,
        updateOrderUC: updateOrderUC,
        deleteOrderUC: deleteOrderUC,
        logger:        logger,
    }
}

// POST /orders
func (h *OrderHandler) Create(c *gin.Context) {
    var req CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, 1001, "invalid request: "+err.Error())
        return
    }

    // Get user ID from auth middleware
    userID := c.GetString("user_id")
    if userID == "" {
        response.Unauthorized(c, "user not authenticated")
        return
    }

    // Call UseCase
    order, err := h.createOrderUC.Execute(c.Request.Context(), &input.CreateOrderInput{
        UserID: userID,
        Items:  toUseCaseItems(req.Items),
    })
    if err != nil {
        h.handleError(c, err)
        return
    }

    response.Created(c, ToOrderResponse(order))
}

// GET /orders/:id
func (h *OrderHandler) GetByID(c *gin.Context) {
    orderID := c.Param("id")

    order, err := h.getOrderUC.Execute(c.Request.Context(), orderID)
    if err != nil {
        h.handleError(c, err)
        return
    }

    response.OK(c, ToOrderResponse(order))
}

// GET /orders
func (h *OrderHandler) List(c *gin.Context) {
    var req ListOrdersRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        response.BadRequest(c, 1001, "invalid query params: "+err.Error())
        return
    }

    // Default pagination
    if req.Page == 0 {
        req.Page = 1
    }
    if req.PageSize == 0 {
        req.PageSize = 20
    }

    userID := c.GetString("user_id")

    result, err := h.listOrdersUC.Execute(c.Request.Context(), &input.ListOrdersInput{
        UserID:   userID,
        Status:   req.Status,
        Page:     req.Page,
        PageSize: req.PageSize,
    })
    if err != nil {
        h.handleError(c, err)
        return
    }

    response.OKWithPage(c, ToOrderListResponse(result.Orders), &response.PageMeta{
        Page:       req.Page,
        PageSize:   req.PageSize,
        Total:      result.Total,
        TotalPages: int((result.Total + int64(req.PageSize) - 1) / int64(req.PageSize)),
    })
}

// PATCH /orders/:id
func (h *OrderHandler) Update(c *gin.Context) {
    orderID := c.Param("id")

    var req UpdateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, 1001, "invalid request: "+err.Error())
        return
    }

    order, err := h.updateOrderUC.Execute(c.Request.Context(), &input.UpdateOrderInput{
        OrderID: orderID,
        Status:  req.Status,
    })
    if err != nil {
        h.handleError(c, err)
        return
    }

    response.OK(c, ToOrderResponse(order))
}

// DELETE /orders/:id
func (h *OrderHandler) Delete(c *gin.Context) {
    orderID := c.Param("id")

    if err := h.deleteOrderUC.Execute(c.Request.Context(), orderID); err != nil {
        h.handleError(c, err)
        return
    }

    response.NoContent(c)
}
```

## Error Handling: DomainError to HTTP Status

Centralized error mapping in handler:

```go
// internal/adapter/inbound/rest/handler/error.go
package handler

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"

    "myapp/internal/adapter/inbound/rest/response"
    pkgerrors "myapp/pkg/errors"
)

func (h *OrderHandler) handleError(c *gin.Context, err error) {
    var domErr pkgerrors.DomainError
    if errors.As(err, &domErr) {
        httpStatus, errCode := mapDomainErrorToHTTP(domErr.DomainCode())
        msg := domErr.Error()
        if cm, ok := err.(pkgerrors.ClientMessage); ok {
            msg = cm.ClientMsg()
        }
        response.Error(c, httpStatus, errCode, msg)
        return
    }

    // Unknown error — log and return generic 500
    h.logger.Error("unexpected error", zap.Error(err))
    response.InternalError(c)
}

func mapDomainErrorToHTTP(code pkgerrors.ErrorCode) (httpStatus int, errCode int) {
    switch code {
    case pkgerrors.ErrCodeNotFound:
        return http.StatusNotFound, 1001
    case pkgerrors.ErrCodeInvalidInput:
        return http.StatusBadRequest, 1002
    case pkgerrors.ErrCodePreconditionFailed:
        return http.StatusConflict, 1003
    case pkgerrors.ErrCodeUnauthorized:
        return http.StatusUnauthorized, 1004
    case pkgerrors.ErrCodeForbidden:
        return http.StatusForbidden, 1005
    case pkgerrors.ErrCodeConflict:
        return http.StatusConflict, 1006
    case pkgerrors.ErrCodeRateLimited:
        return http.StatusTooManyRequests, 1007
    case pkgerrors.ErrCodeAlreadyExists:
        return http.StatusConflict, 1008
    case pkgerrors.ErrCodeAborted:
        return http.StatusConflict, 1009
    default:
        return http.StatusInternalServerError, 5000
    }
}
```

### HTTP Status Code Guidelines

| HTTP Status | When to Use |
|-------------|-------------|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST (resource created) |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Invalid request format, validation failed |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | State conflict (e.g., duplicate, invalid state transition) |
| 422 Unprocessable Entity | Semantic error (valid format, invalid business logic) |
| 429 Too Many Requests | Rate limited |
| 500 Internal Server Error | Unexpected server error |

## Middleware Chain

```go
// internal/adapter/inbound/rest/router/router.go
package router

import (
    "github.com/gin-gonic/gin"

    "myapp/internal/adapter/inbound/rest/handler"
    "myapp/internal/adapter/inbound/rest/middleware"
)

type Router struct {
    engine       *gin.Engine
    orderHandler *handler.OrderHandler
    userHandler  *handler.UserHandler
    authMW       gin.HandlerFunc
    loggerMW     gin.HandlerFunc
    recoveryMW   gin.HandlerFunc
    rateLimitMW  gin.HandlerFunc
}

func NewRouter(
    engine *gin.Engine,
    orderHandler *handler.OrderHandler,
    userHandler *handler.UserHandler,
    authMW gin.HandlerFunc,
    loggerMW gin.HandlerFunc,
    recoveryMW gin.HandlerFunc,
    rateLimitMW gin.HandlerFunc,
) *Router {
    r := &Router{
        engine:       engine,
        orderHandler: orderHandler,
        userHandler:  userHandler,
        authMW:       authMW,
        loggerMW:     loggerMW,
        recoveryMW:   recoveryMW,
        rateLimitMW:  rateLimitMW,
    }
    r.setupRoutes()
    return r
}

func (r *Router) setupRoutes() {
    // Global middleware (order matters: outer → inner)
    r.engine.Use(r.recoveryMW)   // 1. Panic recovery (outermost)
    r.engine.Use(r.loggerMW)     // 2. Request logging
    r.engine.Use(r.rateLimitMW)  // 3. Rate limiting

    // API v1
    v1 := r.engine.Group("/api/v1")

    // Public routes (no auth)
    public := v1.Group("")
    {
        public.POST("/auth/login", r.userHandler.Login)
        public.POST("/auth/register", r.userHandler.Register)
    }

    // Protected routes (require auth)
    protected := v1.Group("")
    protected.Use(r.authMW)
    {
        // Orders
        protected.POST("/orders", r.orderHandler.Create)
        protected.GET("/orders", r.orderHandler.List)
        protected.GET("/orders/:id", r.orderHandler.GetByID)
        protected.PATCH("/orders/:id", r.orderHandler.Update)
        protected.DELETE("/orders/:id", r.orderHandler.Delete)

        // Users
        protected.GET("/users/me", r.userHandler.GetProfile)
        protected.PATCH("/users/me", r.userHandler.UpdateProfile)
    }

    // Admin routes
    admin := v1.Group("/admin")
    admin.Use(r.authMW, middleware.RequireRole("admin"))
    {
        admin.GET("/orders", r.orderHandler.AdminList)
        admin.PATCH("/orders/:id/status", r.orderHandler.AdminUpdateStatus)
    }
}
```

### Middleware Execution Order

```
Request → Recovery → Logger → RateLimit → Auth → Handler → Response
                                                      ↓
Response ← Recovery ← Logger ← RateLimit ← Auth ← Handler
```

## Pagination

### Offset-Based (Simple, for Admin UIs)

```go
// Request
GET /api/v1/orders?page=2&page_size=20

// Response
{
  "code": 0,
  "data": [...],
  "meta": {
    "page": 2,
    "page_size": 20,
    "total": 156,
    "total_pages": 8
  }
}
```

### Cursor-Based (Efficient, for Infinite Scroll)

```go
// Request
GET /api/v1/orders?cursor=eyJpZCI6MTIzfQ&limit=20

// Response
{
  "code": 0,
  "data": [...],
  "meta": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

```go
// Cursor implementation
type CursorMeta struct {
    NextCursor string `json:"next_cursor,omitempty"`
    HasMore    bool   `json:"has_more"`
}

func EncodeCursor(id string) string {
    return base64.StdEncoding.EncodeToString([]byte(id))
}

func DecodeCursor(cursor string) (string, error) {
    data, err := base64.StdEncoding.DecodeString(cursor)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

## File Upload

```go
// POST /api/v1/files
func (h *FileHandler) Upload(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        response.BadRequest(c, 1001, "file is required")
        return
    }

    // Validate file size (10MB max)
    if file.Size > 10*1024*1024 {
        response.BadRequest(c, 1002, "file too large, max 10MB")
        return
    }

    // Validate file type
    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
    }
    if !allowedTypes[file.Header.Get("Content-Type")] {
        response.BadRequest(c, 1003, "invalid file type")
        return
    }

    // Open file
    src, err := file.Open()
    if err != nil {
        response.InternalError(c)
        return
    }
    defer src.Close()

    // Upload to storage (e.g., MinIO, S3)
    url, err := h.storageUC.Upload(c.Request.Context(), src, file.Filename)
    if err != nil {
        h.handleError(c, err)
        return
    }

    response.Created(c, gin.H{"url": url})
}
```

## API Versioning

### URL Path Versioning (Recommended)

```
/api/v1/orders
/api/v2/orders
```

```go
func (r *Router) setupRoutes() {
    v1 := r.engine.Group("/api/v1")
    r.setupV1Routes(v1)

    v2 := r.engine.Group("/api/v2")
    r.setupV2Routes(v2)
}
```

### Header Versioning (Alternative)

```
GET /api/orders
Accept: application/vnd.myapp.v2+json
```

### Version Deprecation

```go
func DeprecationMiddleware(deprecatedAt, sunsetAt time.Time) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Deprecation", deprecatedAt.Format(time.RFC1123))
        c.Header("Sunset", sunsetAt.Format(time.RFC1123))
        c.Header("Link", `</api/v2/orders>; rel="successor-version"`)
        c.Next()
    }
}

// Apply to v1 routes
v1.Use(DeprecationMiddleware(
    time.Date(2024, 6, 1, 0, 0, 0, 0, time.UTC),
    time.Date(2024, 12, 1, 0, 0, 0, 0, time.UTC),
))
```

---

**Related Documents:**
- [HTTP Gateway Patterns](http-gateway.md) - Gateway layer (REST → gRPC translation)
- [gRPC Patterns](grpc-patterns.md) - Service-to-service communication
- [Domain Layer](domain-layer.md) - Entity, Value Object, Repository Interface
