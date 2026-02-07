# Scheduled Jobs Pattern

Design scheduled jobs as UseCases with dual entry points: **Cron** (automatic) + **API** (manual trigger). This enables backfilling, debugging, and manual retry without code changes.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Directory Structure](#directory-structure)
3. [UseCase Implementation](#usecase-implementation)
4. [Controller (Dual Entry Points)](#controller-dual-entry-points)
5. [Cron Scheduler](#cron-scheduler)
6. [Distributed Lock](#distributed-lock-prevent-duplicate-execution)
7. [Job Execution History](#job-execution-history-audit-log)
8. [Monitoring & Alerting](#job-monitoring--alerting)
9. [When to Use This Pattern](#when-to-use-this-pattern)

---

## Architecture

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

---

## Directory Structure

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

---

## UseCase Implementation

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

func NewStatsContentHourlyJob(
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

---

## Controller (Dual Entry Points)

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

---

## Cron Scheduler

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

---

## Distributed Lock (Prevent Duplicate Execution)

When running multiple instances (K8s replicas), use **Redis distributed lock** to ensure only one instance executes a job.

> **Note**: This uses a simple `SetNX` + TTL pattern (fire-and-forget). For long-running business operations needing auto-renewal (WatchDog), use the **redsync-based Locker** from [resilience.md](resilience.md#distributed-lock-redlock-async). The simpler approach suffices here because: (1) job lock TTL is much longer than job duration, (2) we intentionally let TTL expire to prevent re-execution within the window, (3) no unlock needed.

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

### Controller with Distributed Lock

```go
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

### Lock Key Design

- Include job name + execution window: `job:{name}:{window}`
- Window granularity matches job frequency (hourly job → hour, daily → date)
- Prevents re-execution within the same window, even after restart

### TTL Strategy

| Job Duration | Recommended TTL |
|--------------|-----------------|
| < 1 minute | 5 minutes |
| 1-10 minutes | 30 minutes |
| 10-60 minutes | 2 hours |
| > 1 hour | job duration × 2 |

---

## Job Execution History (Audit Log)

Track job executions for debugging and compliance:

### Schema

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

### Repository Interface

```go
// internal/domain/repository/job_execution_repository.go
type JobExecutionRepository interface {
    Create(ctx context.Context, exec *entity.JobExecution) error
    Update(ctx context.Context, exec *entity.JobExecution) error
    FindByJobName(ctx context.Context, jobName string, limit int) ([]*entity.JobExecution, error)
    FindRunning(ctx context.Context) ([]*entity.JobExecution, error)
}
```

### Job Executor with Tracking

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

### Usage

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

---

## Job Monitoring & Alerting

### Structured Log Schema

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

### Prometheus Metrics

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

### Executor with Metrics

```go
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

### Alert Rules (Prometheus/Grafana)

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
      # Uses timestamp() of the last counter increment to detect staleness
      - alert: JobMissedSchedule
        expr: |
          time() - max(timestamp(job_execution_total{job_name="StatsContentHourly",status="completed"})) by (job_name) > 7200
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Job {{ $labels.job_name }} missed scheduled execution"
          description: "No successful execution in the last 2 hours"

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

---

## When to Use This Pattern

| Scenario | Use Dual Entry? |
|----------|-----------------|
| Data aggregation (hourly/daily stats) | Yes — backfill is common |
| Cache cleanup | Yes — manual trigger for debugging |
| Report generation | Yes — regenerate on demand |
| One-time migration | No — use dedicated migration script |
| Real-time event processing | No — use MQ consumer |
