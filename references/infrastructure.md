# Infrastructure

## Table of Contents

- [Observability](#observability)
- [Graceful Shutdown](#graceful-shutdown)
- [Alert Rules](#alert-rules)
- [Testing Strategy](#testing-strategy)
- [CI/CD Pipeline](#cicd-pipeline)
- [gRPC Health Check Service](#grpc-health-check-service)
- [Kubernetes Deployment](#kubernetes-deployment)

## Observability

See [observability.md](observability.md) for complete observability implementation including:
- Logging (Zap + Loki)
- Tracing (OTel + Tempo) with sampling strategies
- Metrics (Prometheus + Mimir)
- Grafana dashboards and alerts

## Graceful Shutdown

Proper shutdown order ensures no requests are dropped and all resources are cleaned up correctly.

### Shutdown Sequence

```
1. Set Health Check to NOT_SERVING
   ↓ (wait for K8s endpoint removal)
2. Stop accepting new requests (gRPC/HTTP)
   ↓
3. Drain in-flight requests
   ↓
4. Stop Cron Scheduler
   ↓
5. Stop MQ Consumers (drain in-flight messages)
   ↓
6. Stop Outbox Poller
   ↓
7. Close MQ Connection
   ↓
8. Close Redis Connection
   ↓
9. Close DB Connection Pool
```

### Implementation with Uber Fx

```go
// cmd/main.go
func main() {
    app := fx.New(
        fx.Provide(config.Load),
        fx.Provide(logger.New),
        infrastructure.Module,
        adapter.Module,
        application.Module,
        fx.Invoke(registerLifecycle),
    )
    app.Run()
}

// internal/infrastructure/fx/lifecycle.go
func registerLifecycle(
    lc fx.Lifecycle,
    cfg *config.Config,
    logger *zap.Logger,
    grpcServer *grpc.Server,
    healthServer *health.Server,
    scheduler *cron.Scheduler,
    mqConn *amqp.Connection,
    consumer *mq.Consumer,
    poller *outbox.Poller,
    redisClient *redis.Client,
    dbPool *pgxpool.Pool,
) {
    var listener net.Listener

    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            var err error
            listener, err = net.Listen("tcp", ":"+cfg.Server.GRPCPort)
            if err != nil {
                return err
            }

            // Start components
            go grpcServer.Serve(listener)
            scheduler.Start()
            consumer.Start()
            poller.Start()

            logger.Info("service started",
                zap.String("grpc_port", cfg.Server.GRPCPort),
            )
            return nil
        },
        OnStop: func(ctx context.Context) error {
            logger.Info("shutdown initiated")

            // 1. Set health to NOT_SERVING (K8s removes from endpoints)
            healthServer.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
            logger.Info("health set to NOT_SERVING")

            // 2. Wait for K8s endpoint removal (matches preStop hook)
            time.Sleep(5 * time.Second)

            // 3. Stop accepting new gRPC requests, drain in-flight
            grpcServer.GracefulStop()
            logger.Info("gRPC server stopped")

            // 4. Stop scheduler
            <-scheduler.Stop().Done()
            logger.Info("cron scheduler stopped")

            // 5. Stop MQ consumer (drain in-flight messages)
            consumer.Stop()
            logger.Info("MQ consumer stopped")

            // 6. Stop outbox poller
            poller.Stop()
            logger.Info("outbox poller stopped")

            // 7. Close MQ connection
            if err := mqConn.Close(); err != nil {
                logger.Warn("failed to close MQ connection", zap.Error(err))
            }
            logger.Info("MQ connection closed")

            // 8. Close Redis
            if err := redisClient.Close(); err != nil {
                logger.Warn("failed to close Redis", zap.Error(err))
            }
            logger.Info("Redis connection closed")

            // 9. Close DB pool
            dbPool.Close()
            logger.Info("DB pool closed")

            logger.Info("shutdown complete")
            return nil
        },
    })
}
```

### MQ Consumer Graceful Shutdown

```go
// internal/adapter/inbound/mq/consumer.go
type Consumer struct {
    channel    *amqp.Channel
    done       chan struct{}
    wg         sync.WaitGroup
    handlers   map[string]MessageHandler
}

func (c *Consumer) Start() {
    c.done = make(chan struct{})

    for queue, handler := range c.handlers {
        c.wg.Add(1)
        go c.consume(queue, handler)
    }
}

func (c *Consumer) Stop() {
    close(c.done)  // Signal all consumers to stop
    c.wg.Wait()    // Wait for in-flight messages to complete
}

func (c *Consumer) consume(queue string, handler MessageHandler) {
    defer c.wg.Done()

    msgs, err := c.channel.Consume(queue, "", false, false, false, false, nil)
    if err != nil {
        zap.L().Error("failed to start consumer", zap.String("queue", queue), zap.Error(err))
        return
    }

    for {
        select {
        case <-c.done:
            // Graceful shutdown: stop accepting new messages
            // In-flight message (if any) will complete before wg.Done()
            return
        case msg, ok := <-msgs:
            if !ok {
                return
            }
            if err := handler.Handle(context.Background(), msg); err != nil {
                msg.Nack(false, !isPermanentError(err))
            } else {
                msg.Ack(false)
            }
        }
    }
}
```

### Outbox Poller Graceful Shutdown

```go
// internal/infrastructure/outbox/poller.go
type Poller struct {
    done   chan struct{}
    wg     sync.WaitGroup
    ticker *time.Ticker
}

func (p *Poller) Start() {
    p.done = make(chan struct{})
    p.ticker = time.NewTicker(5 * time.Second)

    p.wg.Add(1)
    go p.poll()
}

func (p *Poller) Stop() {
    p.ticker.Stop()
    close(p.done)
    p.wg.Wait()
}

func (p *Poller) poll() {
    defer p.wg.Done()

    for {
        select {
        case <-p.done:
            return
        case <-p.ticker.C:
            p.processOutbox()
        }
    }
}
```

### Key Points

| Component | Shutdown Action | Wait For |
|-----------|-----------------|----------|
| Health Check | Set NOT_SERVING | K8s endpoint removal (5s) |
| gRPC Server | GracefulStop() | In-flight requests complete |
| Cron Scheduler | Stop() | Current job completes |
| MQ Consumer | Close done channel | WaitGroup (in-flight messages) |
| Outbox Poller | Close done channel | WaitGroup (current batch) |
| MQ Connection | Close() | — |
| Redis | Close() | — |
| DB Pool | Close() | — |

### Common Mistakes

1. **Closing DB before draining requests** → In-flight requests fail with "connection closed"
2. **Not waiting for K8s endpoint removal** → Traffic still routes to terminating pod
3. **Force killing consumers** → Messages lost or redelivered
4. **Not logging shutdown progress** → Hard to debug shutdown issues

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
