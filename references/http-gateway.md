# HTTP Gateway Patterns

Patterns for a **standalone API Gateway** that translates REST → gRPC (Controller → gRPC Client, no UseCase).
For **service-internal** REST endpoints with business logic (Handler → UseCase → Repository), see [rest-api.md](rest-api.md).

## Table of Contents

- [Gateway Architecture](#gateway-architecture)
- [Router + Controller Pattern](#router--controller-pattern)
- [Rate Limiting](#rate-limiting)
- [Request Logging with Sensitive Field Masking](#request-logging-with-sensitive-field-masking)
- [External HTTP Client Logger](#external-http-client-logger)

## Gateway Architecture

HTTP API Gateway sits between clients and gRPC microservices, handling REST-to-gRPC translation, authentication, rate limiting, and request logging.

```
Client (HTTP/REST)
       │
       ▼
┌─────────────────────────────────────────┐
│            HTTP Gateway (Gin)           │
│  ┌─────────────────────────────────┐    │
│  │ Middleware Chain:               │    │
│  │  - Rate Limiting                │    │
│  │  - Request Logging              │    │
│  │  - Auth (JWT validation)        │    │
│  │  - Recovery                     │    │
│  └─────────────────────────────────┘    │
│                  │                      │
│                  ▼                      │
│  ┌─────────────────────────────────┐    │
│  │ Router → Controller → gRPC Client│   │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
       │
       ▼
  gRPC Microservices
```

## Router + Controller Pattern

When building an HTTP API Gateway (e.g., with Gin), use the **Router + Controller** pattern to separate routing configuration from request handling.

### Directory Structure

```
gateway/
├── internal/
│   ├── router/
│   │   ├── router.go          # Main router setup + middleware groups
│   │   ├── order.go           # Order routes
│   │   ├── user.go            # User routes
│   │   └── di.go              # Fx module
│   │
│   ├── controller/
│   │   ├── order.go           # Order controller
│   │   ├── order_dto.go       # Order DTOs
│   │   ├── user.go            # User controller
│   │   ├── user_dto.go        # User DTOs
│   │   ├── common_dto.go      # Shared DTOs (ErrorResponse, etc.)
│   │   └── di.go              # Fx module
│   │
│   └── client/                # gRPC clients to microservices
│       ├── order_client.go
│       └── user_client.go
```

### Router Implementation

```go
// internal/router/router.go
type RouterParams struct {
    fx.In
    Engine          *gin.Engine
    OrderController *controller.OrderController
    UserController  *controller.UserController
    AuthMiddleware  gin.HandlerFunc `name:"auth"`
}

type Router struct {
    in          RouterParams
    publicGroup *gin.RouterGroup
    authGroup   *gin.RouterGroup
    adminGroup  *gin.RouterGroup
}

func NewRouter(in RouterParams) *Router {
    r := &Router{in: in}

    // Route groups with different middleware
    r.publicGroup = in.Engine.Group("/api/v1")
    r.authGroup = in.Engine.Group("/api/v1", in.AuthMiddleware)
    r.adminGroup = in.Engine.Group("/api/v1/admin", in.AuthMiddleware, adminOnlyMiddleware)

    // Register routes
    r.orderRoutes()
    r.userRoutes()

    return r
}

// internal/router/order.go
func (r *Router) orderRoutes() {
    ctrl := r.in.OrderController

    // Public endpoints
    r.publicGroup.GET("/orders/:id", ctrl.GetOrder)

    // Authenticated endpoints
    r.authGroup.POST("/orders", ctrl.CreateOrder)
    r.authGroup.GET("/orders", ctrl.ListOrders)

    // Admin endpoints
    r.adminGroup.PUT("/orders/:id/status", ctrl.UpdateOrderStatus)
}
```

### Controller Implementation

```go
// internal/controller/order.go
type OrderParams struct {
    fx.In
    OrderClient client.OrderClient
    Logger      *zap.Logger
}

type OrderController struct {
    in OrderParams
}

func NewOrderController(in OrderParams) *OrderController {
    return &OrderController{in: in}
}

func (c *OrderController) CreateOrder(ctx *gin.Context) {
    var req CreateOrderRequest
    if err := ctx.ShouldBindJSON(&req); err != nil {
        ctx.JSON(http.StatusBadRequest, ErrorResponse{Code: 1, Msg: "invalid request"})
        return
    }

    // Get user from context (set by auth middleware)
    userID := ctx.GetString("user_id")

    // Call gRPC service
    result, err := c.in.OrderClient.Create(ctx.Request.Context(), userID, &req)
    if err != nil {
        c.handleError(ctx, err)
        return
    }

    ctx.JSON(http.StatusOK, SuccessResponse{Code: 0, Data: result})
}

func (c *OrderController) handleError(ctx *gin.Context, err error) {
    // Map gRPC status to HTTP status
    st, ok := status.FromError(err)
    if !ok {
        ctx.JSON(http.StatusInternalServerError, ErrorResponse{Code: 500, Msg: "internal error"})
        return
    }

    switch st.Code() {
    case codes.NotFound:
        ctx.JSON(http.StatusNotFound, ErrorResponse{Code: 404, Msg: st.Message()})
    case codes.InvalidArgument:
        ctx.JSON(http.StatusBadRequest, ErrorResponse{Code: 400, Msg: st.Message()})
    case codes.PermissionDenied:
        ctx.JSON(http.StatusForbidden, ErrorResponse{Code: 403, Msg: st.Message()})
    default:
        ctx.JSON(http.StatusInternalServerError, ErrorResponse{Code: 500, Msg: "internal error"})
    }
}
```

### DTO Definition

```go
// internal/controller/order_dto.go
type CreateOrderRequest struct {
    Items []OrderItemRequest `json:"items" binding:"required,min=1"`
}

type OrderItemRequest struct {
    ProductID string `json:"product_id" binding:"required"`
    Quantity  int    `json:"quantity" binding:"required,min=1"`
}

type OrderResponse struct {
    ID        string    `json:"id"`
    Status    string    `json:"status"`
    Total     int64     `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

// internal/controller/common_dto.go
type SuccessResponse struct {
    Code int         `json:"code"`
    Data interface{} `json:"data"`
}

type ErrorResponse struct {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}
```

### When to Use Router + Controller

| Scenario | Pattern | Reason |
|----------|---------|--------|
| HTTP API Gateway | Router + Controller | Flexible routing, middleware groups |
| gRPC Microservice | Handler only | Proto defines service/rpc structure |
| Mixed HTTP + gRPC | Both | Gateway uses Router, services use Handler |

### Key Differences from gRPC Handler

| Aspect | HTTP Gateway (Controller) | gRPC Microservice (Handler) |
|--------|--------------------------|----------------------------|
| Route definition | Explicit in Router | Proto service definition |
| Request binding | `ShouldBindJSON/URI/Query` | Auto from proto message |
| Response format | Manual JSON marshaling | Auto proto serialization |
| Error mapping | Controller maps to HTTP status | Interceptor maps to gRPC status |
| DTO location | `controller/xxx_dto.go` | `application/dto/` |

## Rate Limiting

Per-IP + Per-Path rate limiting for HTTP Gateway. Uses `golang.org/x/time/rate` with automatic cleanup of stale limiters.

### Implementation

```go
// pkg/middleware/ratelimit.go
package middleware

import (
    "context"
    "net/http"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

type limiterKey struct {
    IP   string
    Path string
}

type limiterEntry struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

type rateLimiterStore struct {
    entries map[limiterKey]*limiterEntry
    mu      sync.Mutex
}

// NewIPPathRateLimiter creates a rate limiter per IP+Path combination
// interval: minimum time between requests (e.g., 100ms = 10 req/sec)
// burst: max burst size
// ctx: used to stop the cleanup goroutine on shutdown
func NewIPPathRateLimiter(ctx context.Context, interval time.Duration, burst int) gin.HandlerFunc {
    store := &rateLimiterStore{
        entries: make(map[limiterKey]*limiterEntry),
    }

    getLimiter := func(ip, path string) *rate.Limiter {
        key := limiterKey{IP: ip, Path: path}
        store.mu.Lock()
        defer store.mu.Unlock()
        if e, exists := store.entries[key]; exists {
            e.lastSeen = time.Now()
            return e.limiter
        }
        l := rate.NewLimiter(rate.Every(interval), burst)
        store.entries[key] = &limiterEntry{limiter: l, lastSeen: time.Now()}
        return l
    }

    // Cleanup goroutine: remove stale limiters every 5 minutes
    // Stops when ctx is cancelled (graceful shutdown)
    go func() {
        const staleThreshold = 10 * time.Minute
        ticker := time.NewTicker(5 * time.Minute)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                now := time.Now()
                store.mu.Lock()
                for k, e := range store.entries {
                    if now.Sub(e.lastSeen) > staleThreshold {
                        delete(store.entries, k)
                    }
                }
                store.mu.Unlock()
            }
        }
    }()

    return func(c *gin.Context) {
        ip := c.ClientIP()
        path := c.FullPath()  // Use route pattern, not actual path
        limiter := getLimiter(ip, path)

        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "code":    -1,
                "message": "Too many requests, please try again later",
            })
            return
        }
        c.Next()
    }
}
```

### Usage

```go
// ctx is typically the application lifecycle context (cancelled on shutdown)

// Apply globally
r.Use(middleware.NewIPPathRateLimiter(ctx, 100*time.Millisecond, 10))  // 10 req/sec, burst 10

// Apply to specific routes
sensitiveGroup := r.Group("/api/v1/auth")
sensitiveGroup.Use(middleware.NewIPPathRateLimiter(ctx, time.Second, 5))  // 1 req/sec, burst 5
sensitiveGroup.POST("/login", authController.Login)
sensitiveGroup.POST("/reset-password", authController.ResetPassword)
```

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| **IP + Path key** | Same IP can access different endpoints at different rates |
| **`c.FullPath()` not `c.Request.URL.Path`** | Use route pattern `/users/:id` not `/users/123` — prevents per-resource explosion |
| **`lastSeen` tracking** | Track last access time instead of consuming tokens for cleanup. Prevents false positives and token loss |
| **`context.Context` for shutdown** | Cleanup goroutine stops when ctx is cancelled, preventing goroutine leak |
| **Token bucket** | Allows bursts while enforcing long-term rate |

### When to Use

| Scenario | Recommended Rate |
|----------|------------------|
| Login/Auth endpoints | 1 req/sec, burst 3 |
| Password reset | 1 req/10sec, burst 1 |
| General API | 10 req/sec, burst 20 |
| Public read endpoints | Skip rate limiting or 100 req/sec |

## Request Logging with Sensitive Field Masking

Log HTTP requests with automatic masking of sensitive fields (passwords, credit card numbers, etc.).

### Implementation

```go
// pkg/middleware/logger.go
package middleware

import (
    "bytes"
    "encoding/json"
    "io"
    "net/http"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

// Sensitive field names to mask
var sensitiveFields = map[string]bool{
    "password":        true,
    "confirmPassword": true,
    "oldPassword":     true,
    "newPassword":     true,
    "cardNo":          true,
    "cardNumber":      true,
    "cardCVC":         true,
    "cvv":             true,
    "token":           true,
    "accessToken":     true,
    "refreshToken":    true,
    "secret":          true,
    "apiKey":          true,
}

func RequestLogger(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        // Read and restore request body
        var bodyBytes []byte
        if c.Request.Body != nil {
            bodyBytes, _ = io.ReadAll(c.Request.Body)
            c.Request.Body = io.NopCloser(bytes.NewBuffer(bodyBytes))
        }

        // Process request
        c.Next()

        // Build log fields
        latency := time.Since(start)
        statusCode := c.Writer.Status()

        fields := []zap.Field{
            zap.String("method", c.Request.Method),
            zap.String("path", c.Request.URL.Path),
            zap.String("query", c.Request.URL.RawQuery),
            zap.String("clientIP", c.ClientIP()),
            zap.Int("status", statusCode),
            zap.Duration("latency", latency),
            zap.String("userAgent", c.Request.UserAgent()),
        }

        // Add masked body for POST/PUT/PATCH
        if len(bodyBytes) > 0 && shouldLogBody(c.Request.Method) {
            maskedBody := maskSensitiveFields(bodyBytes, c.ContentType())
            fields = append(fields, zap.String("body", truncate(maskedBody, 2000)))
        }

        // Log level based on status code
        switch {
        case statusCode >= 500:
            logger.Error("request", fields...)
        case statusCode >= 400:
            logger.Warn("request", fields...)
        default:
            logger.Info("request", fields...)
        }
    }
}

func shouldLogBody(method string) bool {
    return method == http.MethodPost || method == http.MethodPut || method == http.MethodPatch
}

func maskSensitiveFields(body []byte, contentType string) string {
    if !strings.Contains(contentType, "application/json") {
        return "[non-json body]"
    }

    var data map[string]interface{}
    if err := json.Unmarshal(body, &data); err != nil {
        return "[invalid json]"
    }

    maskMap(data)

    masked, _ := json.Marshal(data)
    return string(masked)
}

func maskMap(data map[string]interface{}) {
    for key, value := range data {
        if sensitiveFields[key] || sensitiveFields[strings.ToLower(key)] {
            data[key] = "***MASKED***"
            continue
        }
        // Recursively mask nested objects
        if nested, ok := value.(map[string]interface{}); ok {
            maskMap(nested)
        }
    }
}

func truncate(s string, maxLen int) string {
    if len(s) <= maxLen {
        return s
    }
    return s[:maxLen] + "...[truncated]"
}
```

### Output Example

```json
{
  "level": "info",
  "msg": "request",
  "method": "POST",
  "path": "/api/v1/auth/login",
  "clientIP": "192.168.1.1",
  "status": 200,
  "latency": "45.123ms",
  "body": "{\"email\":\"user@example.com\",\"password\":\"***MASKED***\"}"
}
```

### Best Practices

1. **Always mask**: password, token, card numbers, CVV, API keys
2. **Truncate large bodies**: Prevent log explosion (limit to 2KB)
3. **Skip binary content**: Don't log multipart/form-data file contents
4. **Log level by status**: Error for 5xx, Warn for 4xx, Info for 2xx/3xx
5. **Include trace ID**: Add `ctxutil.TraceID.ToLog(ctx)` for correlation

## External HTTP Client Logger

Log outbound HTTP requests in curl format for easy debugging and replay.

### Implementation

```go
// pkg/httpclient/logger.go
package httpclient

import (
    "context"
    "encoding/json"
    "net/http"
    "strings"
    "time"

    "github.com/go-resty/resty/v2"
    "go.uber.org/zap"
)

type RequestLogger struct {
    logger *zap.Logger
    topic  string  // e.g., "payuni", "stripe", "twilio"
}

func NewRequestLogger(logger *zap.Logger, topic string) *RequestLogger {
    return &RequestLogger{logger: logger, topic: topic}
}

// Log logs the HTTP request/response with curl command format
func (l *RequestLogger) Log(ctx context.Context, resp *resty.Response, err error) {
    fields := []zap.Field{
        zap.String("topic", l.topic),
        zap.Time("timestamp", time.Now()),
    }

    if resp == nil {
        fields = append(fields,
            zap.String("status", "failed"),
            zap.Error(err),
        )
        l.logger.Error("external request failed", fields...)
        return
    }

    req := resp.Request
    fields = append(fields,
        zap.String("method", req.Method),
        zap.String("url", req.URL),
        zap.Int("httpStatus", resp.StatusCode()),
        zap.Duration("latency", resp.Time()),
        zap.String("curl", l.buildCurlCommand(req)),
        zap.String("response", truncateResponse(resp.String(), 1000)),
    )

    if err != nil {
        fields = append(fields, zap.Error(err))
    }

    if resp.StatusCode() >= 400 {
        l.logger.Error("external request", fields...)
    } else {
        l.logger.Info("external request", fields...)
    }
}

// buildCurlCommand generates a curl command for easy replay
func (l *RequestLogger) buildCurlCommand(r *resty.Request) string {
    var sb strings.Builder
    sb.WriteString("curl -X ")
    sb.WriteString(r.Method)
    sb.WriteString(" '")
    sb.WriteString(r.URL)
    sb.WriteString("'")

    // Headers (mask sensitive ones)
    for k, v := range r.Header {
        for _, h := range v {
            if isSensitiveHeader(k) {
                h = "***MASKED***"
            }
            sb.WriteString(" -H '")
            sb.WriteString(k)
            sb.WriteString(": ")
            sb.WriteString(h)
            sb.WriteString("'")
        }
    }

    // Body
    if r.Body != nil {
        bodyBytes, _ := json.Marshal(r.Body)
        sb.WriteString(" -d '")
        sb.WriteString(string(bodyBytes))
        sb.WriteString("'")
    }

    return sb.String()
}

func isSensitiveHeader(name string) bool {
    lower := strings.ToLower(name)
    return lower == "authorization" || lower == "x-api-key" || lower == "x-auth-token"
}

func truncateResponse(s string, maxLen int) string {
    if len(s) <= maxLen {
        return s
    }
    return s[:maxLen] + "...[truncated]"
}
```

### Usage with Resty

```go
// pkg/client/payuni/client.go
package payuni

type Client struct {
    http   *resty.Client
    logger *httpclient.RequestLogger
}

func NewClient(baseURL string, logger *zap.Logger) *Client {
    return &Client{
        http:   resty.New().SetBaseURL(baseURL).SetTimeout(30 * time.Second),
        logger: httpclient.NewRequestLogger(logger, "payuni"),
    }
}

func (c *Client) CreateMerchant(ctx context.Context, req CreateMerchantRequest) (*CreateMerchantResponse, error) {
    var result CreateMerchantResponse

    resp, err := c.http.R().
        SetContext(ctx).
        SetHeader("Content-Type", "application/json").
        SetBody(req).
        SetResult(&result).
        Post("/api/merchants")

    // Always log, regardless of success/failure
    c.logger.Log(ctx, resp, err)

    if err != nil {
        return nil, fmt.Errorf("payuni request failed: %w", err)
    }
    if resp.StatusCode() >= 400 {
        return nil, fmt.Errorf("payuni error: status=%d body=%s", resp.StatusCode(), resp.String())
    }

    return &result, nil
}
```

### Output Example

```json
{
  "level": "info",
  "msg": "external request",
  "topic": "payuni",
  "method": "POST",
  "url": "https://api.payuni.com/api/merchants",
  "httpStatus": 200,
  "latency": "234.567ms",
  "curl": "curl -X POST 'https://api.payuni.com/api/merchants' -H 'Content-Type: application/json' -H 'Authorization: ***MASKED***' -d '{\"name\":\"Test Store\"}'",
  "response": "{\"code\":0,\"data\":{\"merchantId\":\"M12345\"}}"
}
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Easy replay** | Copy curl command directly to terminal for debugging |
| **Quick debugging** | See exact request sent to external API |
| **Audit trail** | Log all external API interactions |
| **Error correlation** | Track which external calls failed with trace ID |
| **Performance monitoring** | Latency tracking for external dependencies |

### Best Practices

1. **Always log**: Both success and failure — external calls are debugging goldmines
2. **Mask secrets**: Authorization headers, API keys
3. **Truncate responses**: Large responses can bloat logs
4. **Include topic**: Group logs by external service (payuni, stripe, etc.)
5. **Add trace ID**: Correlate with incoming request logs
