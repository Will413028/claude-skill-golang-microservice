# Observability (Grafana LGTM Stack)

Complete guide for implementing observability in Go microservices using **Grafana LGTM** stack:
- **L**oki (Logs)
- **G**rafana (Visualization)
- **T**empo (Traces)
- **M**imir (Metrics)

---

## Table of Contents

1. [Overview](#overview)
2. [Logging (Zap + Loki)](#logging-zap--loki)
3. [Tracing (OTel + Tempo)](#tracing-otel--tempo)
4. [Metrics (Prometheus + Mimir) `[Async]`](#metrics-prometheus--mimir-async)
5. [Correlation (Logs ↔ Traces ↔ Metrics)](#correlation-logs--traces--metrics)
6. [Grafana Dashboards & Alerts](#grafana-dashboards--alerts)
7. [Quick Reference](#quick-reference)

---

## Overview

### Three Pillars of Observability

| Pillar | Tool | Purpose | Query Language |
|--------|------|---------|----------------|
| **Logs** | Loki | Event details, debugging | LogQL |
| **Traces** | Tempo | Request flow across services | TraceQL |
| **Metrics** | Mimir | Aggregated measurements, alerting | PromQL |

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Go Microservice                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                         │
│  │   Zap   │  │  OTel   │  │ Prom    │                         │
│  │ Logger  │  │ Tracer  │  │ Client  │                         │
│  └────┬────┘  └────┬────┘  └────┬────┘                         │
└───────┼────────────┼────────────┼──────────────────────────────┘
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Promtail│  │  OTel   │  │ Prom    │
   │         │  │Collector│  │ Server  │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  Loki   │  │  Tempo  │  │  Mimir  │
   └────┬────┘  └────┬────┘  └────┬────┘
        │            │            │
        └────────────┼────────────┘
                     ▼
               ┌─────────┐
               │ Grafana │
               └─────────┘
```

### Dependencies

```go
// go.mod
require (
    go.uber.org/zap v1.27.0
    go.opentelemetry.io/otel v1.28.0
    go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.28.0
    go.opentelemetry.io/otel/sdk v1.28.0
    github.com/prometheus/client_golang v1.19.0
)
```

---

## Logging (Zap + Loki)

### Logger Initialization

```go
// pkg/logger/logger.go
package logger

import (
    "os"
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

type Config struct {
    ServiceName string
    Environment string // "local", "stage", "prod"
    Level       string // "debug", "info", "warn", "error"
}

func New(cfg Config) (*zap.Logger, error) {
    // Parse log level
    level, err := zapcore.ParseLevel(cfg.Level)
    if err != nil {
        level = zapcore.InfoLevel
    }

    // Production config (JSON output)
    encoderCfg := zap.NewProductionEncoderConfig()
    encoderCfg.TimeKey = "ts"
    encoderCfg.LevelKey = "level"
    encoderCfg.MessageKey = "msg"
    encoderCfg.EncodeTime = zapcore.ISO8601TimeEncoder
    encoderCfg.EncodeLevel = zapcore.LowercaseLevelEncoder

    core := zapcore.NewCore(
        zapcore.NewJSONEncoder(encoderCfg),
        zapcore.AddSync(os.Stdout),
        level,
    )

    // Add static fields
    logger := zap.New(core).With(
        zap.String("service", cfg.ServiceName),
        zap.String("env", cfg.Environment),
    )

    // Replace global logger
    zap.ReplaceGlobals(logger)

    return logger, nil
}
```

### Log Schema Standard

All logs must include these fields for Loki querying:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ts` | string | Yes | ISO8601 timestamp |
| `level` | string | Yes | debug/info/warn/error |
| `msg` | string | Yes | Human-readable message |
| `service` | string | Yes | Service name |
| `env` | string | Yes | Environment |
| `trace_id` | string | Conditional | Present when in request context |
| `span_id` | string | Conditional | Present when in request context |
| `event` | string | Recommended | Structured event name |
| `error` | string | Conditional | Error message (for warn/error) |
| `error_code` | string | Conditional | Domain error code |

```json
{
  "ts": "2024-01-15T10:30:45.123Z",
  "level": "info",
  "msg": "order created successfully",
  "service": "order-service",
  "env": "prod",
  "trace_id": "abc123def456",
  "span_id": "789xyz",
  "event": "order.created",
  "order_id": 12345,
  "buyer_id": 67890,
  "total_amount": 1500
}
```

### Context-Aware Logging

```go
// pkg/logger/context.go
package logger

import (
    "context"
    "go.opentelemetry.io/otel/trace"
    "go.uber.org/zap"
)

type ctxKey struct{}

// WithLogger stores logger in context
func WithLogger(ctx context.Context, l *zap.Logger) context.Context {
    return context.WithValue(ctx, ctxKey{}, l)
}

// FromContext retrieves logger from context, enriched with trace info
func FromContext(ctx context.Context) *zap.Logger {
    l, ok := ctx.Value(ctxKey{}).(*zap.Logger)
    if !ok {
        l = zap.L()
    }

    // Enrich with trace context
    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        l = l.With(
            zap.String("trace_id", sc.TraceID().String()),
            zap.String("span_id", sc.SpanID().String()),
        )
    }

    return l
}

// Ctx is a shorthand for FromContext
func Ctx(ctx context.Context) *zap.Logger {
    return FromContext(ctx)
}
```

**Usage in UseCase:**

```go
func (u *OrderUseCase) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    log := logger.Ctx(ctx)

    log.Info("creating order",
        zap.String("event", "order.create.start"),
        zap.Int64("buyer_id", req.BuyerID),
        zap.Int("item_count", len(req.Items)),
    )

    order, err := u.orderRepo.Create(ctx, req)
    if err != nil {
        log.Error("failed to create order",
            zap.String("event", "order.create.failed"),
            zap.Error(err),
        )
        return nil, err
    }

    log.Info("order created",
        zap.String("event", "order.created"),
        zap.Int64("order_id", order.ID),
        zap.Int64("total_amount", order.TotalAmount),
    )

    return order, nil
}
```

### Log Level Guidelines

| Level | When to Use | Example |
|-------|-------------|---------|
| **DEBUG** | Detailed debugging info (disabled in prod) | SQL queries, cache hits/misses |
| **INFO** | Normal operations, business events | Order created, payment received |
| **WARN** | Recoverable issues, degraded state | Retry succeeded, fallback used, deprecated API called |
| **ERROR** | Failures requiring attention | DB connection failed, external API error |

**Rules:**
- **INFO is the default** — Use INFO for happy path events
- **WARN is not ERROR** — Use WARN when the system recovered or can continue
- **ERROR means action needed** — Someone should investigate ERROR logs
- **Never log sensitive data** — Mask passwords, tokens, PII

### Error Logging with Context

```go
// pkg/logger/error.go
package logger

import (
    "errors"
    "go.uber.org/zap"

    pkgerrors "github.com/yourproject/go-pkg/errors"
)

// ErrorFields extracts structured fields from an error
// Uses the shared DomainError interface from pkg/errors (see grpc-patterns.md)
func ErrorFields(err error) []zap.Field {
    fields := []zap.Field{
        zap.Error(err),
    }

    // Extract domain error code
    var domErr pkgerrors.DomainError
    if errors.As(err, &domErr) {
        fields = append(fields, zap.Int("error_code", int(domErr.DomainCode())))
    }

    return fields
}

// LogError logs an error with proper context
func LogError(ctx context.Context, msg string, err error, extraFields ...zap.Field) {
    log := FromContext(ctx)
    fields := append(ErrorFields(err), extraFields...)
    log.Error(msg, fields...)
}
```

**Usage:**

```go
if err := u.paymentClient.Charge(ctx, req); err != nil {
    logger.LogError(ctx, "payment failed", err,
        zap.String("event", "payment.charge.failed"),
        zap.Int64("order_id", orderID),
        zap.Int64("amount", amount),
    )
    return err
}
```

### Event Catalog (Standardized Event Names)

Use consistent event names for easier Loki querying:

```go
// pkg/logger/events/events.go
package events

// Event naming convention: <domain>.<entity>.<action>
const (
    // Order events
    OrderCreateStart   = "order.create.start"
    OrderCreated       = "order.created"
    OrderCreateFailed  = "order.create.failed"
    OrderCancelled     = "order.cancelled"
    OrderShipped       = "order.shipped"

    // Payment events
    PaymentChargeStart = "payment.charge.start"
    PaymentCharged     = "payment.charged"
    PaymentChargeFailed = "payment.charge.failed"
    PaymentRefunded    = "payment.refunded"

    // User events
    UserLoginSuccess   = "user.login.success"
    UserLoginFailed    = "user.login.failed"
    UserRegistered     = "user.registered"

    // System events
    ServiceStarted     = "service.started"
    ServiceStopping    = "service.stopping"
    HealthCheckFailed  = "health.check.failed"
)
```

**Loki Query by Event:**

```logql
{service="order-service"} |= `"event":"order.created"`
{service="order-service"} | json | event = "order.create.failed"
```

### Sensitive Data Masking

```go
// pkg/logger/mask.go
package logger

import (
    "regexp"
    "strings"
)

var (
    emailRegex = regexp.MustCompile(`[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`)
    phoneRegex = regexp.MustCompile(`09\d{8}`)
    cardRegex  = regexp.MustCompile(`\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}`)
)

// MaskEmail masks email: test@example.com -> t***@example.com
func MaskEmail(email string) string {
    parts := strings.Split(email, "@")
    if len(parts) != 2 || len(parts[0]) == 0 {
        return "***"
    }
    return string(parts[0][0]) + "***@" + parts[1]
}

// MaskPhone masks phone: 0912345678 -> 0912***678
func MaskPhone(phone string) string {
    if len(phone) < 10 {
        return "***"
    }
    return phone[:4] + "***" + phone[len(phone)-3:]
}

// MaskCard masks card number: 1234-5678-9012-3456 -> ****-****-****-3456
func MaskCard(card string) string {
    cleaned := strings.ReplaceAll(strings.ReplaceAll(card, "-", ""), " ", "")
    if len(cleaned) < 4 {
        return "****"
    }
    return "****-****-****-" + cleaned[len(cleaned)-4:]
}

// MaskString masks sensitive patterns in a string
func MaskString(s string) string {
    s = emailRegex.ReplaceAllStringFunc(s, MaskEmail)
    s = phoneRegex.ReplaceAllStringFunc(s, MaskPhone)
    s = cardRegex.ReplaceAllStringFunc(s, MaskCard)
    return s
}
```

### Log Sampling (Production)

For high-traffic services, sample logs to reduce volume:

```go
// pkg/logger/sampler.go
package logger

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
    "time"
)

// NewSampledLogger creates a logger that samples INFO/DEBUG logs
func NewSampledLogger(cfg Config) (*zap.Logger, error) {
    base, err := New(cfg)
    if err != nil {
        return nil, err
    }

    // Sample INFO logs: first 10, then 1 per 100 in each second
    // WARN and ERROR are never sampled
    sampler := zap.WrapCore(func(core zapcore.Core) zapcore.Core {
        return zapcore.NewSamplerWithOptions(
            core,
            time.Second,    // interval
            10,             // first N messages per interval
            100,            // thereafter, 1 per M messages
        )
    })

    return base.WithOptions(sampler), nil
}
```

### Loki Labels Design

Loki uses labels for indexing. Keep labels **low cardinality**:

| Label | Cardinality | Example | Include? |
|-------|-------------|---------|----------|
| `service` | Low (~20) | order-service | Yes |
| `env` | Low (3) | prod, stage, dev | Yes |
| `level` | Low (4) | info, warn, error, debug | Yes |
| `trace_id` | High (millions) | abc123 | **No** (use filter) |
| `user_id` | High (millions) | 12345 | **No** (use filter) |
| `order_id` | High (millions) | 67890 | **No** (use filter) |

**Promtail Config:**

```yaml
# promtail-config.yaml
scrape_configs:
  - job_name: microservices
    static_configs:
      - targets: [localhost]
        labels:
          __path__: /var/log/app/*.log
    pipeline_stages:
      - json:
          expressions:
            level: level
            service: service
            env: env
      - labels:
          level:
          service:
          env:
```

---

## Tracing (OTel + Tempo)

### Tracer Initialization

```go
// pkg/tracer/tracer.go
package tracer

import (
    "context"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

type Config struct {
    ServiceName    string
    ServiceVersion string
    Environment    string
    OTLPEndpoint   string  // e.g., "tempo:4317"
    SampleRate     float64 // 0.0 to 1.0
}

func Init(ctx context.Context, cfg Config) (func(), error) {
    // Create OTLP exporter
    conn, err := grpc.NewClient(cfg.OTLPEndpoint,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }

    exporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
    if err != nil {
        return nil, err
    }

    // Create resource
    res, err := resource.Merge(
        resource.Default(),
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(cfg.ServiceName),
            semconv.ServiceVersion(cfg.ServiceVersion),
            semconv.DeploymentEnvironmentName(cfg.Environment),
        ),
    )
    if err != nil {
        return nil, err
    }

    // Create sampler
    var sampler sdktrace.Sampler
    if cfg.Environment == "local" || cfg.Environment == "dev" {
        sampler = sdktrace.AlwaysSample()
    } else {
        sampler = sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(cfg.SampleRate),
        )
    }

    // Create TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sampler),
    )

    // Set global TracerProvider
    otel.SetTracerProvider(tp)

    // Set global propagator (W3C Trace Context)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    // Return shutdown function
    shutdown := func() {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = tp.Shutdown(ctx)
    }

    return shutdown, nil
}
```

### Span Naming Convention

| Layer | Format | Example |
|-------|--------|---------|
| gRPC Server | `<package>.<Service>/<Method>` | `order.v1.OrderService/CreateOrder` |
| gRPC Client | `<package>.<Service>/<Method>` | `payment.v1.PaymentService/Charge` |
| HTTP Server | `<METHOD> <route>` | `POST /api/v1/orders` |
| HTTP Client | `HTTP <METHOD>` | `HTTP POST` |
| Database | `<operation> <table>` | `SELECT orders`, `INSERT orders` |
| MQ Publish | `<exchange> publish` | `order.events publish` |
| MQ Consume | `<queue> process` | `order.created.payment process` |

### gRPC Interceptor (Server + Client)

```go
// pkg/grpc/interceptor/tracing.go
package interceptor

import (
    "context"

    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

// TracingServerInterceptor adds tracing to gRPC server
func TracingServerInterceptor() grpc.UnaryServerInterceptor {
    return otelgrpc.UnaryServerInterceptor()
}

// TracingClientInterceptor adds tracing to gRPC client
func TracingClientInterceptor() grpc.UnaryClientInterceptor {
    return otelgrpc.UnaryClientInterceptor()
}
```

### HTTP Middleware (Gin)

```go
// pkg/middleware/tracing.go
package middleware

import (
    "github.com/gin-gonic/gin"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

// Tracing returns Gin middleware for tracing
func Tracing(serviceName string) gin.HandlerFunc {
    return otelgin.Middleware(serviceName)
}
```

### MQ Trace Propagation `[Async]`

See [grpc-patterns.md — MQ Trace Context Propagation](grpc-patterns.md#mq-trace-context-propagation-async) for the canonical `AMQPCarrier` implementation (`pkg/mq/rabbitmq/trace.go`) and publisher/consumer usage examples.

### Sampling Strategy `[Hardening]`

| Environment | Strategy | Rate | Rationale |
|-------------|----------|------|-----------|
| **Local/Dev** | AlwaysSample | 100% | Full visibility for debugging |
| **Stage** | TraceIDRatioBased | 50% | Balance visibility and volume |
| **Prod (low traffic)** | TraceIDRatioBased | 10-20% | Reasonable coverage |
| **Prod (high traffic)** | TraceIDRatioBased | 1-5% | Avoid storage explosion |

**Parent-Based Sampling:**

Always use `ParentBased` sampler in production. This ensures:
- If parent span is sampled → child is sampled
- If parent span is not sampled → child is not sampled
- Consistent sampling across service boundaries

```go
sampler := sdktrace.ParentBased(
    sdktrace.TraceIDRatioBased(0.1), // 10% for root spans
)
```

---

## Metrics (Prometheus + Mimir) `[Async]`

### Metrics Types

| Type | Use Case | Example |
|------|----------|---------|
| **Counter** | Cumulative count | Total requests, total errors |
| **Gauge** | Current value | Active connections, queue size |
| **Histogram** | Distribution | Request latency, response size |
| **Summary** | Quantiles | P50/P95/P99 latency |

### Standard Metrics (RED Method)

Every service should expose these metrics:

```go
// pkg/metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Rate: requests per second
    RequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    // Errors: error rate
    ErrorsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_errors_total",
            Help: "Total number of HTTP errors",
        },
        []string{"method", "path", "error_code"},
    )

    // Duration: request latency
    RequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
)

// gRPC metrics
var (
    GRPCRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "grpc_requests_total",
            Help: "Total number of gRPC requests",
        },
        []string{"method", "status"},
    )

    GRPCRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "grpc_request_duration_seconds",
            Help:    "gRPC request duration in seconds",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5},
        },
        []string{"method"},
    )
)
```

### Business Metrics

```go
// internal/infrastructure/metrics/business.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Order metrics
    OrdersCreated = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
        []string{"merchant_id", "payment_method"},
    )

    OrderAmount = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "order_amount",
            Help:    "Order amount distribution",
            Buckets: []float64{100, 500, 1000, 2000, 5000, 10000, 50000},
        },
        []string{"merchant_id"},
    )

    // Queue metrics
    QueueSize = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_size",
            Help: "Current queue size",
        },
        []string{"queue_name"},
    )

    // Cache metrics
    CacheHits = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "cache_hits_total",
            Help: "Total cache hits",
        },
        []string{"cache_name"},
    )

    CacheMisses = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "cache_misses_total",
            Help: "Total cache misses",
        },
        []string{"cache_name"},
    )
)
```

### Histogram Buckets Design

Choose buckets based on expected latency distribution:

| Service Type | Expected Latency | Buckets |
|--------------|------------------|---------|
| In-memory cache | 0.1-10ms | `.0001, .0005, .001, .005, .01, .025, .05` |
| Database query | 1-100ms | `.001, .005, .01, .025, .05, .1, .25, .5` |
| Internal gRPC | 5-500ms | `.005, .01, .025, .05, .1, .25, .5, 1` |
| External API | 50ms-5s | `.05, .1, .25, .5, 1, 2.5, 5, 10` |
| HTTP request | 10ms-2s | `.01, .025, .05, .1, .25, .5, 1, 2.5, 5` |

### Metrics Middleware (Gin)

```go
// pkg/middleware/metrics.go
package middleware

import (
    "strconv"
    "time"

    "github.com/gin-gonic/gin"
    "your-project/pkg/metrics"
)

func Metrics() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.FullPath() // Use route pattern, not actual path
        if path == "" {
            path = "unknown"
        }

        c.Next()

        status := strconv.Itoa(c.Writer.Status())
        duration := time.Since(start).Seconds()

        metrics.RequestsTotal.WithLabelValues(c.Request.Method, path, status).Inc()
        metrics.RequestDuration.WithLabelValues(c.Request.Method, path).Observe(duration)

        if c.Writer.Status() >= 400 {
            metrics.ErrorsTotal.WithLabelValues(c.Request.Method, path, status).Inc()
        }
    }
}
```

### Metrics Endpoint

```go
// main.go
import (
    "github.com/gin-gonic/gin"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    r := gin.Default()

    // Metrics endpoint (separate from main API for security)
    r.GET("/metrics", gin.WrapH(promhttp.Handler()))

    // Or run on separate port
    go func() {
        metricsRouter := gin.New()
        metricsRouter.GET("/metrics", gin.WrapH(promhttp.Handler()))
        metricsRouter.Run(":9090")
    }()
}
```

---

## Correlation (Logs ↔ Traces ↔ Metrics)

### TraceID in Logs

The key to correlation is including `trace_id` in logs:

```go
// Logs with trace_id can be found from Tempo
log.Info("order created",
    zap.String("trace_id", span.SpanContext().TraceID().String()),
    zap.Int64("order_id", order.ID),
)
```

### Exemplars (Metrics → Traces) `[Async]`

Link metrics to traces using exemplars:

```go
// pkg/metrics/exemplar.go
package metrics

import (
    "context"

    "github.com/prometheus/client_golang/prometheus"
    "go.opentelemetry.io/otel/trace"
)

// ObserveWithExemplar records a histogram observation with trace exemplar
func ObserveWithExemplar(ctx context.Context, hist *prometheus.HistogramVec, value float64, labels ...string) {
    observer := hist.WithLabelValues(labels...)

    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        observer.(prometheus.ExemplarObserver).ObserveWithExemplar(
            value,
            prometheus.Labels{"trace_id": sc.TraceID().String()},
        )
    } else {
        observer.Observe(value)
    }
}
```

### Grafana Correlation Queries

**From Logs → Traces (Loki to Tempo):**

1. In Grafana, query Loki: `{service="order-service"} | json | level="error"`
2. Click on a log line with `trace_id`
3. Grafana automatically links to Tempo trace

**From Traces → Logs (Tempo to Loki):**

1. In Tempo, view a trace
2. Click "Logs for this span"
3. Grafana queries Loki: `{service="order-service"} |= "trace_id=<trace_id>"`

**From Metrics → Traces (Mimir to Tempo):**

1. In Grafana, view a metric with exemplars enabled
2. Click on an exemplar point
3. Grafana links to the trace

---

## Grafana Dashboards & Alerts

### Service Overview Dashboard `[Async]`

```json
{
  "title": "Service Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(http_requests_total[5m])) by (service)"
      }]
    },
    {
      "title": "Error Rate",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])) * 100"
      }]
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))"
      }]
    },
    {
      "title": "Recent Errors",
      "type": "logs",
      "targets": [{
        "expr": "{service=~\".*\"} | json | level=\"error\""
      }]
    }
  ]
}
```

### Alert Rules `[Hardening]`

```yaml
# prometheus/alerts/service_alerts.yml
groups:
  - name: service_health
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_errors_total[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency on {{ $labels.service }}"
          description: "P99 latency is {{ $value | humanizeDuration }}"

      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"

      # Many errors in logs
      - alert: ManyLogErrors
        expr: |
          sum(count_over_time({level="error"}[5m])) by (service) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Many errors in {{ $labels.service }} logs"
```

### Loki Alert Rules `[Hardening]`

```yaml
# loki/alerts/log_alerts.yml
groups:
  - name: log_alerts
    rules:
      - alert: CriticalErrorLogged
        expr: |
          count_over_time({service=~".+"} |= "CRITICAL" [1m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Critical error logged"

      - alert: PanicDetected
        expr: |
          count_over_time({service=~".+"} |= "panic" [1m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Panic detected in {{ $labels.service }}"
```

---

## Quick Reference

### Log Query Patterns (LogQL)

```logql
# Filter by service and level
{service="order-service", level="error"}

# Search text
{service="order-service"} |= "payment failed"

# Parse JSON and filter
{service="order-service"} | json | order_id > 1000

# Count errors per minute
sum(count_over_time({service="order-service", level="error"}[1m]))

# Find logs by trace ID
{service=~".+"} |= "trace_id=abc123"
```

### Metric Query Patterns (PromQL) `[Async]`

```promql
# Request rate
sum(rate(http_requests_total[5m])) by (service)

# Error rate percentage
sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Apdex score (target 500ms)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  + sum(rate(http_request_duration_seconds_bucket{le="2"}[5m]))
) / 2 / sum(rate(http_request_duration_seconds_count[5m]))
```

### Trace Query Patterns (TraceQL)

```traceql
# Find slow traces
{ duration > 1s }

# Find error traces
{ status = error }

# Find traces by service
{ resource.service.name = "order-service" }

# Find traces with specific span
{ name = "POST /api/v1/orders" && duration > 500ms }
```
