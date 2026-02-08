---
name: golang-microservice
description: >
  Go microservice architecture skill. Covers Clean Architecture layering, gRPC communication,
  Saga patterns (sync & async), Transactional Outbox, caching/circuit-breaker/idempotency,
  and full microservice lifecycle management.
  Use when: creating a new Go microservice, restructuring existing services, making architecture
  decisions, reviewing code, or planning staged evolution (MVP → Async → Hardening → Infrastructure).
  Trigger keywords: Go microservice, gRPC service, Clean Architecture, Saga pattern, Outbox pattern,
  microservice directory structure, microservice evolution strategy, domain-driven design in Go.
  Not for: monolith architecture, non-Go projects, pure frontend development.
---

# Go Microservice Architecture Skill

Guide for building Go microservices from MVP to production-grade,
built on Clean Architecture + gRPC + PostgreSQL + RabbitMQ.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Go 1.24+ |
| API Gateway | Apache APISIX |
| Service Communication | gRPC + Protocol Buffers |
| Database | PostgreSQL 17 |
| Cache | Redis 8 |
| Message Queue | RabbitMQ |
| Object Storage | MinIO |
| Container | Docker Compose → Kubernetes |
| Dependency Injection | Uber Fx |
| Configuration | Native `os.Getenv` + struct |
| Logging | Zap (JSON stdout) |
| Schema Management | Atlas |
| Database Access | sqlc (SQL-first, type-safe) |
| Collection Ops | `github.com/samber/lo` (Application/Adapter layers only) |
| Singleflight | `golang.org/x/sync/singleflight` |
| Circuit Breaker | `github.com/sony/gobreaker` |

## Core Architecture Principle

```
Dependency direction (inward only): outer layers → inner layers

internal/
├── domain/           # Entity, Value Object, Repository Interface, Domain Event (zero external deps)
├── service/          # Reusable business logic (cross-UseCase)
├── usecase/          # Orchestration + DTO + transactions (consumer-defined local interfaces)
├── repository/postgres/  # Repository Impl + gen/ (sqlc) + mapper.go
├── client/           # External gRPC clients (by service name)
├── grpc/             # gRPC Handler (server.go + handler + mapper)
├── consumer/         # MQ Consumer (Async stage)
├── config/           # Config struct + DI module
└── app/              # Assembles all fx.Modules
```

**Domain Layer is the stable core across all stages** — Entity, Value Object, and state machines
never change due to infrastructure changes. UseCase orchestration signatures remain stable;
only internal implementations evolve with each stage.

**Go-style flat directories** — No `adapter/inbound/outbound/` or `application/port/input/output/`.
Each package is named by its concern (`grpc/`, `repository/postgres/`, `client/`).
UseCase uses consumer-defined local interfaces for its dependencies (Go idiom).

## Stage Evolution Overview

Every new project follows these four stages. Determine the current stage to decide implementation scope:

| Stage | Goal | When to advance |
|-------|------|-----------------|
| **MVP** | Feature-complete, architecture established | Core business flows validated, ready for decoupling |
| **Async & Resilience** | Async decoupling, fault tolerance | System needs production-grade stability |
| **Hardening** | Production-grade hardening | Ready for K8s deployment |
| **Infrastructure** | Deployment & operations automation | Ongoing operations |

### MVP — Must Do

1. **Architecture skeleton** → Read [references/architecture.md](references/architecture.md)
   - Flat Go-style directory structure + naming conventions
   - Monorepo structure (shared `go.mod`; evaluate `go.work` when services > 5–8)
   - **Uber Fx DI** (`var Module = fx.Module(...)` per package, `fx.Annotate` + `fx.As` for interface binding, assembled in `app/app.go`)
   - **Proto / buf tooling** (`buf.yaml`, `buf.gen.yaml`, Proto design conventions, `buf lint` + `buf breaking`)
   - **Local dev environment** (Docker Compose: PG + Redis + RabbitMQ + init-db.sh + Makefile)

2. **Domain design** → Read [references/domain-layer.md](references/domain-layer.md)
   - Entity state machine (whitelist transitions + inject `now time.Time`)
   - Value Object (immutable, smallest unit)
   - Repository Interface (co-located at bottom of Entity file in `domain/{entity}.go`)
   - Domain Event collection (Entity collects events on state transitions)

3. **Communication & errors** → Read [references/grpc-patterns.md](references/grpc-patterns.md)
   - gRPC + base Interceptor Chain (OTel → correlation → logging → recovery → metrics → auth → error mapping)
   - DomainError + ErrorCode + centralized Interceptor mapping
   - **Context Deadline / Timeout Budget** (deadline propagation, per-layer budget subtraction)
   - **Authentication / Authorization** (JWT Interceptor, RBAC Permission Interceptor, user context propagation, cross-service auth forwarding)
   - API pagination (Offset / Cursor strategy selection) + Batch API (partial failure pattern)
   - **Proto Versioning Strategy** (package naming, backward-compatible evolution, `buf breaking`)
   - DTO mapping (manual mapper functions in `mapper.go`, compile-time safe)

4. **HTTP Gateway** → Read [references/http-gateway.md](references/http-gateway.md)
   - Router + Controller pattern (Gin-based)
   - Rate limiting (per-IP + per-Path)
   - Request logging with sensitive field masking
   - External HTTP client logger (curl format)

5. **REST API** → Read [references/rest-api.md](references/rest-api.md)
   - RESTful URL design (resource naming, HTTP methods)
   - Response format: HTTP status + business code (`{"code": 0, "data": {...}}`)
   - Request DTO validation (Gin binding tags)
   - DomainError → HTTP status mapping
   - Batch API (per-item result, partial failure)

6. **Data layer** → Read [references/data-layer.md](references/data-layer.md)
   - Database-per-Service (logical isolation)
   - sqlc (SQL-first, type-safe) + Atlas (declarative migration)
   - **Zero-Downtime Schema Migration** (expand-and-contract, backfill strategy, deployment ordering)
   - **Keyset Pagination** (composite cursor, N+1 fetch pattern)
   - Config management (`os.Getenv` + struct + Fail-Fast Validation)

7. **Cross-service transactions** → Read [references/async-patterns.md](references/async-patterns.md)
   - Synchronous Saga + DB-backed step tracking
   - Stale Record Scanner (safety net)
   - All compensation operations must be idempotent

8. **Idempotency** → Read [references/resilience.md](references/resilience.md)
   - Critical business operations use DB `processed_events` table

9. **Observability** → Read [references/observability.md](references/observability.md)
   - Logging (Zap + Loki): log schema, log levels, error logging, event catalog, sensitive masking
   - Tracing (OTel + Tempo): tracer init, span naming, context propagation, sampling
   - Metrics (Prometheus + Mimir): RED method, business metrics, histogram buckets
   - Correlation: TraceID in logs, exemplars, Grafana queries

10. **Quality**: Unit tests + integration tests (testcontainers) + `buf breaking` (Proto contract)

11. **Scheduled Jobs** → Read [references/scheduled-jobs.md](references/scheduled-jobs.md)
    - Dual entry points (Cron + API for backfill/debug)
    - Distributed lock (Redis) for K8s replicas
    - Job execution history (audit log)
    - Monitoring & alerting (Prometheus metrics)

### Async & Resilience — Must Do

1. **Async Saga + Outbox** → Read [references/async-patterns.md](references/async-patterns.md)
   - Single TX writes business record + Saga state + Outbox events
   - Two-phase Outbox Poller (pick vs publish separation)
   - TxManager (Application Port interface + Infrastructure impl)
   - Event versioning for progressive consumer migration

2. **MQ Consumer patterns** → Read [references/async-patterns.md](references/async-patterns.md)
   - Prefetch tuning, concurrent workers, ack/nack strategy (transient vs permanent errors)
   - Event version handling in consumer (v1/v2 switch, unknown version best-effort)
   - Consumer graceful shutdown (drain in-flight → close channel)

3. **Cache + singleflight** → Read [references/resilience.md](references/resilience.md)
   - Generic CacheLoader (Redis + empty-value anti-penetration + singleflight merge)
   - `context.WithoutCancel` to prevent shared-goroutine cancel cascade
   - **Cache invalidation strategy** (write-through, event-driven, always-delete-not-update)

4. **Circuit Breaker** → Read [references/resilience.md](references/resilience.md)
   - gobreaker + singleflight gRPC Client
   - MQ Trace Context propagation (AMQP Header Carrier)

5. **Dispatcher Pattern** → Read [references/resilience.md](references/resilience.md)
   - Generic worker pool for parallel batch processing
   - Panic-Safe errgroup (recover panics in goroutines)

6. **Distributed Lock (Redlock)** → Read [references/resilience.md](references/resilience.md)
   - redsync with WatchDog auto-renewal
   - Application Port (interface) + Adapter (impl)

7. **Saga timeout monitor** + enhanced idempotency (Redis SET NX)

8. **CQRS Read Model Projection** → Read [references/async-patterns.md](references/async-patterns.md)
   - Denormalized read table (zero JOIN list queries)
   - Event Projector (MQ Consumer maintains read model)
   - QueryService pattern (bypass Domain Entity for reads)
   - Read-your-writes consistency (Redis flag fallback)

### Hardening — Must Do

1. **Graceful Shutdown** → Read [references/infrastructure.md](references/infrastructure.md)
   - Shutdown sequence: Health NOT_SERVING → gRPC GracefulStop → Consumer drain → Poller Stop → MQ/Redis/DB Close
   - Fx Lifecycle hooks for coordinated shutdown
2. **gRPC Health Check Service** → Read [references/infrastructure.md](references/infrastructure.md)
   - `grpc.health.v1.Health` registration (required for K8s probes)
   - Dependency health monitoring (DB + Redis → readiness status)
   - Set NOT_SERVING before GracefulStop for clean endpoint removal
3. **Dead Letter Queue**: `x-delivery-limit` + DLQ routing
4. **gRPC Retry Policy**: `retryableStatusCodes: [UNAVAILABLE, DEADLINE_EXCEEDED]`
5. **Alerting**: HighErrorRate, SlowRequests, CircuitBreakerOpen, SagaStuck, OutboxBacklog
6. **Tracing Sampling tuning**: Production 10–20%, high-traffic 1–5% (ParentBased)
7. **Load testing + chaos testing**

### Infrastructure — Must Do

1. **K8s deployment** → Read [references/infrastructure.md](references/infrastructure.md)
   - Deployment + PDB + HPA
   - preStop hook (`sleep 5`, wait for endpoint removal)
   - Resource requests/limits (consider omitting CPU limit to avoid CFS throttling)
2. **Full CI/CD**: lint → codegen-verify → test → migration-safety → build → deploy
3. **Evaluate Monorepo scaling**: `go.work` / independent versioning for shared packages

### Stage Priority Cross-Reference

| Topic | MVP | Async & Resilience | Hardening | Infrastructure |
|-------|-----|-----|-----|-----|
| Clean Architecture layering | ✅ Must | — | — | — |
| Directory structure + Monorepo | ✅ Must | — | — | Evaluate `go.work` |
| Uber Fx Module Wiring | ✅ Must | — | — | — |
| Proto / buf Tooling | ✅ Must | — | — | — |
| Local Dev (Docker Compose) | ✅ Must | — | — | — |
| Naming conventions | ✅ Must | — | — | — |
| Error handling (DomainError + Interceptor) | ✅ Must | — | — | — |
| Domain Layer (Entity + VO + Repo Interface) | ✅ Must | — | — | — |
| DTO mapping + consumer-defined interfaces | ✅ Must | — | — | — |
| Context Deadline / Timeout Budget | ✅ Must | — | — | — |
| Authentication / Authorization | ✅ Must | — | — | — |
| gRPC Interceptor Chain | ✅ Must (base) | Add MQ Trace propagation | — | — |
| API Pagination + Keyset Pagination | ✅ Must | — | — | — |
| Batch API (gRPC + REST) | ✅ Must | — | — | — |
| Proto Versioning Strategy | ✅ Must | — | — | — |
| RBAC Permission Interceptor | ✅ Must | — | — | — |
| Config management | ✅ Must | — | — | — |
| Structured logging | ✅ Must | — | — | — |
| Database (sqlc + Atlas) | ✅ Must | — | — | — |
| Zero-Downtime Schema Migration | ✅ Must | — | — | — |
| Sync Saga + step persistence | ✅ Must | Replace with Async Saga | — | — |
| Async Saga + Outbox + TxManager | — | ✅ Must | — | — |
| MQ Consumer patterns | — | ✅ Must | — | — |
| Cache + singleflight + invalidation | Optional (high-read scenarios) | ✅ Must | — | — |
| Circuit Breaker + singleflight Client | — | ✅ Must | — | — |
| Dispatcher Pattern + Panic-Safe errgroup | — | ✅ Must | — | — |
| Distributed Lock (Redlock + WatchDog) | — | ✅ Must | — | — |
| Idempotency | ✅ Must (DB) | Add Redis SET NX | — | — |
| Saga timeout monitor | — | ✅ Must | — | — |
| CQRS Read Model Projection | — | Optional (high-read + cross-service) | — | — |
| gRPC Health Check Service | — | — | ✅ Must | — |
| Dead Letter Queue | — | — | ✅ Must | — |
| Graceful Shutdown | Optional | — | ✅ Must | — |
| gRPC Retry Policy | — | — | ✅ Must | — |
| Observability | ✅ Must (Logs + Traces) | Add Metrics + Saga alerts | Sampling tuning + full alerts | — |
| Testing | ✅ Must (unit + integration + contract) | Add idempotency tests | Load + chaos testing | — |
| CI/CD | ✅ Must (lint + test + build) | — | Add migration safety | Full pipeline |
| K8s deployment | — | — | — | ✅ Must |

### Deferrable Items

These can wait beyond MVP without significant risk:

- **Cache / Circuit Breaker / singleflight**: Unless MVP already has high-concurrency read scenarios
- **Graceful Shutdown**: Minimal impact during Docker Compose development; must complete before K8s
- **DLQ / Retry Policy**: Hardening stage — ensure consumers have error logging in the meantime
- **Tail-Based Sampling**: Evaluate only when production traffic volume justifies the overhead
- **K8s / HPA / PDB**: Infrastructure stage

## Decision Guides

### New Service Checklist

1. Create directory structure (per architecture.md): `domain/`, `usecase/`, `service/`, `repository/postgres/`, `grpc/`, `client/`, `config/`, `app/`
2. Define Proto (`proto/{service}/`)
3. Design Domain Layer: Entity (`domain/{entity}.go`) + Enums (`domain/{entity}_types.go`) + Value Object (`domain/valueobject/`) + Repository Interface (bottom of Entity file)
4. Implement Service Layer (`service/`) for reusable business logic
5. Implement UseCase Layer (`usecase/`) with DTOs (`usecase/dto/`) and consumer-defined local interfaces
6. Implement Repository (`repository/postgres/`) with mapper.go + di.go
7. Implement gRPC Handler (`grpc/`) with mapper.go + di.go
8. Configure DI: each package has `di.go` with `var Module = fx.Module(...)`, assembled in `app/app.go`
9. Write Schema (`db/schema/`) + Queries (`db/queries/`) → `sqlc generate` → `repository/postgres/gen/`
10. Write tests (unit + integration)

### Sync vs Async Saga

| Factor | Sync Saga | Async Saga |
|--------|-----------|------------|
| Client needs immediate result | ✅ Suitable | ❌ Returns PENDING |
| Downstream availability requirement | High (direct failure) | Low (retry mechanism) |
| Latency sensitivity | Sequential accumulation | Non-blocking |
| Implementation complexity | Low | High (Outbox + Poller + Consumer) |

**Recommendation**: Use sync Saga in MVP. Switch to async after validating business flows. UseCase signatures remain unchanged.

### When to Introduce Caching

- Read/write ratio > 10:1 AND data tolerates short-term inconsistency → Introduce Redis Cache + singleflight
- Low read/write ratio OR strong consistency required → Skip caching, query DB directly

### When to Introduce Circuit Breaker

- Downstream service not 100% available (cross-team, external API) → Introduce
- Internal service within same team with SLA guarantees → Can defer

### Pagination Strategy

- Admin dashboard (needs page jumping) → Offset-based
- Consumer-facing lists (infinite scroll) → Cursor-based (Keyset Pagination)
- **Rules**: Never concatenate client-provided `sort_by` into SQL; Cursor must be opaque

### Optimistic vs Pessimistic Locking

| Scenario | Strategy | Mechanism |
|----------|----------|-----------|
| Most writes (default) | Optimistic lock | `WHERE version = $old_version`, retry on conflict |
| Short critical section, high contention | Pessimistic lock | `SELECT ... FOR UPDATE` within TX |
| Saga step requiring exclusive access | Pessimistic lock | `FOR UPDATE` to prevent concurrent saga execution |

**Default to optimistic locking.** Use pessimistic only when contention is high and retry cost is unacceptable.

## Code Quality Rules

- **Mapping**: Manual mapper functions (compile-time safe). No reflection-based tools (copier).
- **Collections**: `samber/lo` allowed in Application/Adapter layers only. Forbidden in Domain layer (zero external deps).
- **Error handling**: Domain Error carries ErrorCode. Interceptor maps centrally. UseCase must NEVER manually map gRPC Status.
- **Transactions**: TxManager interface in UseCase (consumer-defined), impl in Infrastructure. Repository uses `GetDBTX(ctx, pool)` for dynamic connection.
- **Domain Events**: `ClearEvents()` must be called ONLY after TX commit succeeds.
- **Entity concurrency**: Entity is NOT goroutine-safe. Cross-goroutine access requires Repository reload + optimistic lock.
- **Event versioning**: Every Domain Event carries `Version()`. Consumers must handle version-based progressive migration.
