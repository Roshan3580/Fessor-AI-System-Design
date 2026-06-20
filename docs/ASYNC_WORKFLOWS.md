# Async Workflows

Fessor offloads long-running AI and media work to Celery workers backed by Redis. This document describes how async jobs are structured, retried, scaled, and recovered. For the end-to-end request view see [System Design](SYSTEM_DESIGN.md). For pipeline stages and model routing see [AI Pipeline](AI_PIPELINE.md).

---

## Overview

```text
Django API → enqueue task → Redis broker → Celery worker → AI provider / storage → PostgreSQL
                                    ↓
                            progress events → Channels → frontend
```

The API returns as soon as a job is validated and enqueued. Workers own all provider calls that exceed HTTP timeout budgets.

---

## Celery Workers

Celery workers are separate processes from Django web workers.

**Responsibilities**

- Execute book generation pipelines
- Run image generation jobs
- Process attachments (PDF extraction, OCR)
- Generate quizzes, flashcards, and exams
- Run scheduled maintenance via Celery Beat

**Why workers are separate**

- Provider latency is unpredictable and often measured in seconds or minutes
- Retries and backoff should not block API threads
- Worker count can scale independently from user-facing request capacity

---

## Redis Broker

Redis acts as the Celery message broker.

**Roles**

- Holds pending and in-flight task messages
- Supports worker acknowledgment and redelivery on crash
- Also serves as cache and Django Channels layer (separate logical use, shared infrastructure in production via Memorystore)

**Implications**

- Broker availability is required for new async work to start
- Task visibility timeout and acknowledgment settings affect retry behavior
- Queue depth in Redis is the primary signal for backpressure

---

## Book Generation Tasks

A book generation job is not a single monolithic task. It is a chain or group of tasks that mirror the pipeline stages in [AI Pipeline](AI_PIPELINE.md).

**Typical task breakdown**

1. **Planning** — Produce chapter and section outline
2. **Section generation** — One or more tasks per section (or batched by chapter)
3. **Image generation** — Tasks tied to sections or figures that require imagery
4. **Assessment generation** — Quizzes, flashcards, and exams derived from section content
5. **Finalization** — Update book readiness state and emit completion events

**Incremental persistence**

Each completed section is written to PostgreSQL before the full book finishes. The reader and progress channel can reflect partial completion without waiting for the entire pipeline.

---

## Image Generation Tasks

Image generation runs as distinct async tasks so text generation is not blocked.

**Flow**

1. Section content and image metadata identify what to generate
2. Worker calls the image provider
3. Result is stored in Cloud Storage
4. PostgreSQL record is updated with the storage reference
5. Progress event notifies the frontend

**Isolation**

Image tasks are slower and more failure-prone than text tasks. Keeping them separate allows retries without re-running successful section writes.

---

## Retries

Celery retries handle transient failures.

**Retry triggers**

- Provider timeout or rate limit
- Temporary network error between worker and provider
- Short-lived database connection issues

**Retry behavior**

- Exponential backoff between attempts
- Retries scoped to the failed task unit (e.g., a single section) rather than the entire book where possible
- Validation failures may retry with the same prompt or fail fast if output is repeatedly malformed

**Idempotency**

Workers write with stable identifiers for sections and assets so a redelivered task updates the same logical record instead of creating duplicates.

---

## Queue Backpressure

When generation demand exceeds worker throughput, tasks accumulate in Redis.

**Signals**

- Growing queue depth
- Increasing time-from-enqueue-to-start
- User-visible delays before progress events begin

**Responses**

- Scale worker processes horizontally
- Optionally split queues (e.g., `generation` vs `images`) so one workload type does not starve another
- Rate-limit or queue new generation requests at the API when the system is saturated, returning a clear message rather than accepting jobs that will sit idle

Backpressure protects provider rate limits and prevents unbounded memory growth in the broker.

---

## Worker Scaling

**Scale up when**

- Queue depth sustains above baseline
- P95 time-to-first-progress rises
- Provider rate limits are not the sole bottleneck

**Scale cautiously when**

- Database write throughput or Cloud Storage upload rate limits appear
- Provider quotas cap effective parallelism regardless of worker count

**Deployment pattern**

- Worker fleet on Compute Engine separate from Daphne/API instances
- Autoscaling based on queue depth or CPU where supported
- Celery Beat on a single dedicated process to avoid duplicate scheduled execution

---

## Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Redis unavailable | New tasks cannot enqueue; in-flight tasks may stall | Managed Memorystore; monitor broker health |
| Worker crash mid-task | Task returned to broker after visibility timeout | Celery redelivery; idempotent section writes |
| Provider hard failure | Section or image task fails after retries | Mark unit failed; surface status via progress channel; completed sections remain |
| Validation rejection | Model output unusable | Retry limited times; fail section without blocking whole book |
| Credit exhaustion mid-job | Later stages cannot proceed | Upfront balance check reduces risk; remaining stages halt with user-visible state |
| Storage upload failure | Image or attachment not persisted | Retry upload; text content unaffected |

---

## Related Documents

- [System Design](SYSTEM_DESIGN.md)
- [AI Pipeline](AI_PIPELINE.md)
- [Realtime Architecture](REALTIME_ARCHITECTURE.md)
- [Architecture](ARCHITECTURE.md)
