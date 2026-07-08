# ARCH-02 --- RabbitMQ Architecture (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies how RabbitMQ is used inside Rustkins V1.

RabbitMQ is **not** the source of truth.

Responsibilities:

-   transport execution orders
-   transport cancellation requests
-   transport log chunks
-   decouple API, Scheduler and Workers

PostgreSQL remains the authoritative state.

RabbitMQ publication is driven by a PostgreSQL transactional outbox.

The Scheduler writes durable OutboxMessages inside runtime transactions.

The Outbox Publisher publishes pending messages to RabbitMQ.

The Outbox Publisher is a Scheduler subcomponent.

------------------------------------------------------------------------

# 2. Architecture

``` mermaid
flowchart LR

API --> Scheduler
Scheduler --> PostgreSQL
PostgreSQL --> OutboxPublisher
OutboxPublisher --> RabbitMQ

RabbitMQ --> Worker
Worker --> PostgreSQL
Worker --> RustFS

Worker --> RabbitMQ
RabbitMQ --> API
API --> Browser
```

------------------------------------------------------------------------

# 3. Exchanges

Three exchanges are defined.

  Exchange           Type     Purpose
  ------------------ -------- -----------------
  rustkins.jobs      direct   Job execution
  rustkins.control   fanout   Runtime control
  rustkins.logs      topic    Real-time logs

------------------------------------------------------------------------

# 4. Queues

  Queue                       Consumer            Description
  --------------------------- ------------------- -------------------------
  rustkins.jobs.ready.q       Workers             Runnable jobs
  rustkins.jobs.retry.q       Scheduler           Delayed retry
  rustkins.jobs.dlq           Operators           Dead Letter Queue
  rustkins.control.cancel.q   Workers             Cancellation broadcast
  rustkins.logs.sse.q         API                 Real-time log streaming

------------------------------------------------------------------------

# 5. Routing Keys

## rustkins.jobs

``` text
job.ready
job.retry
```

## rustkins.control

``` text
cancel.job
cancel.run
```

## rustkins.logs

``` text
logs.{job_run_id}
```

------------------------------------------------------------------------

# 6. Job message

``` json
{
  "message_id":"uuid",
  "pipeline_run_id":"uuid",
  "job_run_id":"uuid",
  "pipeline_version_job_id":"uuid",
  "project_id":"uuid",
  "attempt":1,
  "idempotency_key":"run:job",
  "created_at":"2026-01-01T12:00:00Z"
}
```

Only identifiers are transported.

Workers always reload runtime information from PostgreSQL.

------------------------------------------------------------------------

# 7. Cancel message

``` json
{
  "type":"cancel_job",
  "pipeline_run_id":"uuid",
  "job_run_id":"uuid",
  "requested_by":"uuid"
}
```

Cancellation is broadcast to all workers.

Only the worker executing the job reacts.

Cancel messages are also published through the transactional outbox.

------------------------------------------------------------------------

# 8. Log message

``` json
{
  "job_run_id":"uuid",
  "sequence":42,
  "stream":"stdout",
  "content":"Running tests...",
  "timestamp":"2026-01-01T12:00:00Z"
}
```

Maximum chunk size:

``` text
16 KB
```

------------------------------------------------------------------------

# 9. Transactional outbox

RabbitMQ messages that affect execution state are never published
directly inside the scheduling transaction.

Instead, the Scheduler persists an OutboxMessage in PostgreSQL.

OutboxMessage fields:

``` text
exchange
routing_key
event_type
payload_json
status
attempts
next_attempt_at
last_error
created_at
published_at
```

Status values:

``` text
pending
publishing
published
failed
```

Publication loop:

1.  Load pending OutboxMessages.
2.  Publish the payload to RabbitMQ.
3.  Mark the OutboxMessage as `published`.
4.  If publication fails, increment attempts and schedule a retry.

RabbitMQ downtime never rolls back committed runtime state.

It only delays outbox publication.

If the Outbox Publisher crashes after publishing to RabbitMQ but before
marking the OutboxMessage as `published`, the message may be published
again.

This is acceptable because workers always reload JobRun state from
PostgreSQL and ACK duplicate deliveries without executing them twice.

------------------------------------------------------------------------

# 10. Message lifecycle

``` mermaid
sequenceDiagram
participant Scheduler
participant DB
participant Publisher as Outbox Publisher
participant MQ
participant Worker

Scheduler->>DB: Reserve resources
Scheduler->>DB: Insert OutboxMessage(job.ready)
Publisher->>DB: Load pending OutboxMessage
Publisher->>MQ: Publish job.ready
Publisher->>DB: Mark OutboxMessage published
MQ->>Worker: Deliver message
Worker->>DB: Load JobRun
Worker->>Worker: Execute container
Worker->>DB: Update status
Worker->>MQ: ACK
Scheduler->>DB: Release resources
```

------------------------------------------------------------------------

# 11. ACK / NACK policy

## ACK

ACK is sent only after:

-   JobRun updated
-   artifacts uploaded
-   logs finalized

Resource release is performed by the Scheduler after it observes the
final JobRun state.

## Business failure

Examples:

-   tests failed
-   exit code != 0

Action:

``` text
Status = failed
ACK
```

## Infrastructure failure

Examples:

-   Docker unavailable
-   RustFS unavailable
-   Worker crash

Action:

``` text
Status = failed with infrastructure error
ACK current message
Scheduler schedules retry
```

------------------------------------------------------------------------

# 12. Retry policy

Maximum attempts

``` text
3
```

Retry delays

``` text
30 seconds
2 minutes
5 minutes
```

Retryable failures

-   Docker daemon unavailable
-   RustFS unavailable
-   PostgreSQL temporary outage
-   RabbitMQ temporary outage
-   Network timeout
-   Worker crash

Non retryable

-   Build failed
-   Tests failed
-   YAML invalid
-   Exit code != 0

------------------------------------------------------------------------

# 13. Dead Letter Queue

After maximum retries:

``` text
JobRun.status = dead_lettered
```

PipelineRun becomes:

``` text
failed
```

Dead-lettered jobs require manual retry.

------------------------------------------------------------------------

# 14. Idempotency

RabbitMQ guarantees at-least-once delivery.

Workers must always:

1.  Load JobRun.
2.  Validate current status.
3.  Ignore already completed executions.
4.  ACK duplicate deliveries.

Duplicate execution must never occur.

------------------------------------------------------------------------

# 15. Ordering

RabbitMQ ordering is never trusted.

Execution ordering is entirely controlled by:

-   DAG
-   Scheduler
-   Resource manager

Workers execute only received jobs.

------------------------------------------------------------------------

# 16. Failure scenarios

## Worker crashes

-   heartbeat lost
-   resources released by the Scheduler
-   retry if attempts remain
-   otherwise dead_lettered

## RabbitMQ restart

Outbox Publisher republishes `pending` or `publishing` messages using
OutboxMessage state.

Workers still reload JobRun state from PostgreSQL, so duplicate
publication is safe.

## API unavailable

Workers continue executing.

Real-time logs are temporarily unavailable.

------------------------------------------------------------------------

# 17. ADR

## ADR-001

RabbitMQ is transport only.

Consequence:

Database remains the single source of truth.

------------------------------------------------------------------------

## ADR-002

Workers are stateless.

Consequence:

Every execution starts by reading PostgreSQL.

------------------------------------------------------------------------

## ADR-003

Transactional outbox is used for runtime messages.

Consequence:

RabbitMQ outages delay publication without corrupting scheduling state.

------------------------------------------------------------------------

## ADR-004

Cancellation uses broadcast.

Reason:

Simpler multi-worker implementation for V1.

------------------------------------------------------------------------

# 18. Future evolution

V2 candidates

-   Priority queues
-   Worker-specific queues
-   Delayed exchanges
-   Multi-tenant isolation
-   Queue metrics
