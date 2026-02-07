# Infrastructure

## Table of Contents

- [Observability](#observability)
- [Tracing Sampling Strategy](#tracing-sampling-strategy)
- [Alert Rules](#alert-rules)
- [Testing Strategy](#testing-strategy)
- [CI/CD Pipeline](#cicd-pipeline)
- [gRPC Health Check Service](#grpc-health-check-service)
- [Kubernetes Deployment](#kubernetes-deployment)

## Observability

| Pillar | Tooling | Stage |
|--------|---------|-------|
| Logs | Zap JSON → Loki | MVP |
| Traces | OTel SDK → Tempo | MVP |
| Metrics | Prometheus → Grafana | Async |
| Alerting | Alertmanager | Hardening |

## Tracing Sampling Strategy

| Environment | Strategy | Sampling Rate | Rationale |
|-------------|----------|---------------|-----------|
| Development | AlwaysOn | 100% | Full traces for debugging |
| Staging | AlwaysOn | 100% | Load testing, E2E tests need complete data |
| Production | TraceIDRatioBased | 10–20% | Balance observability vs performance overhead |
| Production (high-traffic QPS > 10K) | TraceIDRatioBased | 1–5% | Further reduce overhead |

```go
// pkg/observability/tracer.go
func NewTracerProvider(cfg *config.Config) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(context.Background(),
        otlptracegrpc.WithEndpoint(cfg.OTel.Endpoint),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil { return nil, fmt.Errorf("create exporter: %w", err) }

    var sampler sdktrace.Sampler
    switch cfg.Environment {
    case "production":
        // ParentBased ensures:
        // 1. Upstream already sampled → continue sampling (preserve trace completeness)
        // 2. Upstream not sampled → don't sample (follow upstream decision)
        // 3. No upstream (root span) → sample by ratio
        sampler = sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(cfg.OTel.SamplingRate), // Recommend 0.1–0.2
        )
    default:
        sampler = sdktrace.AlwaysSample()
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithSampler(sampler),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(cfg.ServiceName),
            attribute.String("environment", cfg.Environment),
        )),
    )
    return tp, nil
}
```

### Advanced: Tail-Based Sampling

Head-Based Sampling decides at trace start — may miss important error traces. When observability requirements increase, configure Tail-Based Sampling at OTel Collector layer (no application code changes needed):

```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    policies:
      - name: error-policy
        type: status_code
        status_code: { status_codes: [ERROR] }  # All error traces 100% retained
      - name: slow-policy
        type: latency
        latency: { threshold_ms: 1000 }          # > 1s traces 100% retained
      - name: default-policy
        type: probabilistic
        probabilistic: { sampling_percentage: 10 } # Rest 10% sampled
```

## Alert Rules

```yaml
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: >
          rate(grpc_server_handled_total{grpc_code!="OK"}[5m]) /
          rate(grpc_server_handled_total[5m]) > 0.05
        for: 5m

      - alert: SlowRequests
        expr: histogram_quantile(0.95, rate(grpc_server_handling_seconds_bucket[5m])) > 1
        for: 10m

      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state{state="open"} == 1
        for: 1m

  - name: saga
    rules:
      - alert: SagaStuck
        expr: saga_stuck_total > 0
        for: 5m

      - alert: OutboxBacklog
        expr: outbox_unsent_events_total > 100
        for: 3m

  - name: infrastructure
    rules:
      - alert: HighDBConnectionUsage
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.8
        for: 5m
```

## Testing Strategy

| Type | Tools | Scope | Stage |
|------|-------|-------|-------|
| Unit tests | `testing` + `testify` + `mockery` | Entity state machine, UseCase | MVP |
| Integration tests | `testcontainers-go` | Repository | MVP |
| Contract tests | `buf breaking` | Proto backward compatibility (prevent breaking changes) | MVP |
| Idempotency tests | — | Duplicate consumption of same event | Async |
| Load tests | `k6` / `vegeta` / `ghz` (gRPC) | Throughput, P99 latency, connection pool saturation | Hardening |
| Chaos tests | `Litmus` / `Chaos Mesh` | Saga timeout, CB trigger, DB failover, pod kill | Hardening |
| E2E tests | — | Cross-service happy path | Hardening |

### Testing Priorities by Layer

| Layer | What to Test | How |
|-------|-------------|-----|
| Entity | State machine transitions (valid + invalid), Domain Event collection | Pure unit tests, no mocks needed |
| UseCase | Orchestration logic, error handling, TX boundary | Mock Repository + Output Ports |
| Repository | SQL correctness, mapping, optimistic lock behavior | Integration tests with testcontainers |
| gRPC Handler | Request validation, DTO mapping | Test mapper functions as unit tests |

## CI/CD Pipeline

```yaml
stages:
  - lint:
      - golangci-lint run ./...
      - buf lint api/proto/
      - buf breaking api/proto/ --against '.git#branch=main'  # Proto backward compatibility
  - codegen-verify:
      - sqlc diff  # Ensure generated code matches queries
  - test:
      - go test ./... -race -cover
      - atlas migrate lint --dir migrations/
  - migration-safety:   # Added in Hardening stage
      - atlas migrate lint --dir migrations/ --dev-url "postgres://..."
      # Checks for destructive changes (DROP TABLE / DROP COLUMN)
  - build:
      - docker build -t ${SERVICE_NAME}:${SHA} .
  - deploy:
      - kubectl apply -k deploy/k8s/
```

### CI Scope per Stage

| Stage | CI Pipeline |
|-------|-------------|
| MVP | lint + test + build |
| Async | Same as MVP |
| Hardening | Add migration-safety checks |
| Infrastructure | Full pipeline including deploy |

## gRPC Health Check Service

K8s probes use `grpc: { port: 50051 }` which requires the standard `grpc.health.v1.Health` service to be registered.

### Implementation

```go
// internal/infrastructure/server/health.go
import (
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
)

func RegisterHealthCheck(server *grpc.Server, pool *pgxpool.Pool, redis *redis.Client) *health.Server {
    healthServer := health.NewServer()
    healthpb.RegisterHealthServer(server, healthServer)

    // Set initial status
    healthServer.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)

    // Optional: background health monitor for dependencies
    go monitorDependencies(healthServer, pool, redis)

    return healthServer
}

func monitorDependencies(hs *health.Server, pool *pgxpool.Pool, redis *redis.Client) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    for range ticker.C {
        status := healthpb.HealthCheckResponse_SERVING

        // Check DB
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        if err := pool.Ping(ctx); err != nil {
            status = healthpb.HealthCheckResponse_NOT_SERVING
        }
        cancel()

        // Check Redis
        ctx, cancel = context.WithTimeout(context.Background(), 2*time.Second)
        if err := redis.Ping(ctx).Err(); err != nil {
            status = healthpb.HealthCheckResponse_NOT_SERVING
        }
        cancel()

        hs.SetServingStatus("", status)
    }
}
```

### Registration in Fx

```go
// internal/infrastructure/fx/grpc_module.go
func RegisterGRPCServices(server *grpc.Server, orderHandler *handler.OrderHandler,
    pool *pgxpool.Pool, redis *redis.Client) {
    pb.RegisterOrderServiceServer(server, orderHandler)
    RegisterHealthCheck(server, pool, redis)  // Health check for K8s probes
}
```

### K8s Probe Behavior

| Probe | Purpose | Health Check Semantics |
|-------|---------|----------------------|
| `startupProbe` | Wait for app to be ready at boot | Passes once service is registered |
| `livenessProbe` | Detect deadlocks/frozen processes | Passes if gRPC server responds |
| `readinessProbe` | Route traffic only to healthy pods | Reflects DB + Redis connectivity |

**Key**: Set readiness to NOT_SERVING during graceful shutdown (before `GracefulStop`) to remove pod from Service endpoints before draining connections.

```go
// In OnStop hook, before GracefulStop:
healthServer.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
time.Sleep(5 * time.Second)  // Wait for K8s endpoint removal (matches preStop)
server.GracefulStop()
```

## Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: some-service
spec:
  replicas: 2
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: some-service
          resources:
            # Go service with gRPC + DB pool + Redis needs at least 256Mi request
            # Consider omitting CPU limit (set request only) to avoid CFS throttling
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { memory: 512Mi }
          # preStop hook: wait for endpoint removal from Service, prevents traffic disruption
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
          startupProbe:
            grpc: { port: 50051 }
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 6
          livenessProbe:
            grpc: { port: 50051 }
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            grpc: { port: 50051 }
            initialDelaySeconds: 5
            periodSeconds: 5
---
# PodDisruptionBudget: keep at least 1 pod available during rolling updates or node maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: some-service-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: some-service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: some-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: some-service
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### K8s Deployment Notes

| Config | Rationale |
|--------|-----------|
| `preStop: sleep 5` | Wait for endpoint removal from Service, prevent in-flight request drops |
| `terminationGracePeriodSeconds: 30` | Give Graceful Shutdown enough time to drain |
| CPU limit omitted (request only) | Avoid CFS throttling that limits Go runtime performance |
| Memory limit = 2x request | Allow burst while preventing OOM |
| `minReplicas: 2` | Ensure high availability |
| PDB `minAvailable: 1` | At least 1 pod survives rolling updates or node drain |
