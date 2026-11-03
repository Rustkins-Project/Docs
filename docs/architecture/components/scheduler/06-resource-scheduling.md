---
id: DOC-0007
title: "Resource Scheduling (Rustkins V1)"
type: architecture
status: draft
summary: "Allocation et réservation de CPU et mémoire pour répartir les jobs entre workers sans dépasser leurs capacités."
topics: [scheduler, resources, worker, cpu, memory]
applies_to: [V1]
---

# ARCH-06 --- Resource Scheduling (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies the resource scheduling model of Rustkins V1.

Objectives:

-   Prevent worker overload
-   Schedule jobs according to available CPU and memory
-   Reserve resources before execution
-   Release resources after execution
-   Keep scheduling deterministic and simple

Out of scope:

-   Kubernetes
-   Affinity
-   Node labels
-   Priority scheduling
-   GPU scheduling

------------------------------------------------------------------------

# 2. Architecture Overview

``` mermaid
flowchart LR

PipelineRun --> Scheduler
Scheduler --> PostgreSQL
PostgreSQL --> OutboxPublisher
OutboxPublisher --> RabbitMQ
RabbitMQ --> Worker
Worker --> Docker

Worker --> PostgreSQL
```

The Scheduler is the only component allowed to decide where a job
executes.

Workers never schedule themselves.

The Scheduler is also the only component allowed to reserve and release
worker capacity.

------------------------------------------------------------------------

# 3. Worker Capacity Model

Each Worker exposes a fixed capacity.

Example

``` text
worker-1
---------
CPU    : 4000 mCPU
Memory : 8192 MB
```

Database fields

``` text
total_cpu_milli
total_memory_mb
reserved_cpu_milli
reserved_memory_mb
```

Available resources are calculated as:

``` text
available_cpu    = total_cpu - reserved_cpu
available_memory = total_memory - reserved_memory
```

------------------------------------------------------------------------

# 4. Job Resource Definition

Resources are declared in the pipeline YAML.

Example

``` yaml
resources:
  cpu: 1000
  memory: 1024
```

Units

  Resource   Unit
  ---------- ----------
  CPU        milliCPU
  Memory     MB

Default values

``` text
cpu    = 1000
memory = 1024
```

------------------------------------------------------------------------

# 5. Scheduling Algorithm

Before a JobRun can be queued:

1.  Dependencies must be satisfied.
2.  Worker must be online.
3.  Worker must have enough available CPU.
4.  Worker must have enough available memory.

If no worker matches:

``` text
JobRun.status = waiting_resources
```

------------------------------------------------------------------------

# 6. Reservation Flow

``` mermaid
sequenceDiagram

participant Scheduler
participant PostgreSQL
participant Publisher as OutboxPublisher
participant RabbitMQ
participant Worker

Scheduler->>PostgreSQL: Find compatible worker
Scheduler->>PostgreSQL: Reserve CPU & Memory
Scheduler->>PostgreSQL: JobRun = queued
Scheduler->>PostgreSQL: Insert OutboxMessage(job.ready)

Publisher->>RabbitMQ: Publish job.ready
RabbitMQ->>Worker: Job message
Worker->>PostgreSQL: JobRun = running
```

Reservation and OutboxMessage creation occur in the same PostgreSQL
transaction.

RabbitMQ publication occurs after commit through the Outbox Publisher.

------------------------------------------------------------------------

# 7. Resource Release

Resources are released by the Scheduler when a JobRun reaches or is
forced into one of these final states:

-   success
-   warning
-   failed
-   cancelled
-   dead_lettered

The Worker does not modify `reserved_cpu_milli` or
`reserved_memory_mb`.

Normal completion flow:

1.  Worker updates the JobRun execution result.
2.  Scheduler observes the final JobRun state.
3.  Scheduler releases the reserved CPU and memory.
4.  Scheduler unlocks dependent jobs or completes the PipelineRun.

Release occurs before any dependent JobRun is queued.

------------------------------------------------------------------------

# 8. Waiting Resources

A JobRun enters `waiting_resources` when:

-   dependencies are complete
-   no worker currently has enough available resources

The Scheduler periodically reevaluates waiting jobs.

------------------------------------------------------------------------

# 9. Worker Selection

For V1 the first compatible worker is selected.

Compatibility requires:

-   online status
-   enough CPU
-   enough memory

No additional scheduling rules exist.

Future versions may implement smarter strategies.

------------------------------------------------------------------------

# 10. Docker Limits

Scheduler reservation is not sufficient.

Docker must also enforce limits.

Container receives:

``` text
CPU limit
Memory limit
```

This prevents one container from consuming the entire machine.

------------------------------------------------------------------------

# 11. Worker Heartbeat

Workers periodically update:

``` text
last_heartbeat_at
```

If heartbeat timeout expires:

``` text
Worker.status = offline
```

Consequences

-   no new jobs assigned
-   reserved resources released by the Scheduler
-   running jobs evaluated for retry

------------------------------------------------------------------------

# 12. Worker Failure

``` mermaid
flowchart TD

WorkerCrash --> HeartbeatTimeout
HeartbeatTimeout --> ReleaseResources
ReleaseResources --> Retry

Retry --> RemainingAttempts
RemainingAttempts --> DeadLetter
```

If retry attempts remain:

``` text
JobRun = retrying
```

Otherwise

``` text
JobRun = dead_lettered
```

------------------------------------------------------------------------

# 13. Oversized Jobs

If a JobRun requests more resources than any worker can provide:

Example

``` text
Worker
CPU 4000

Job
CPU 8000
```

Result

``` text
JobRun = failed
```

Reason

No compatible worker exists.

The Scheduler must not wait forever.

------------------------------------------------------------------------

# 14. Transactions

Resource reservation must be atomic.

Required sequence

1.  Lock worker row
2.  Read available resources
3.  Validate capacity
4.  Update reserved values
5.  Queue JobRun
6.  Insert OutboxMessage
7.  Commit transaction

This prevents over-allocation.

------------------------------------------------------------------------

# 15. Scheduling Rules

Scheduler MUST

-   never oversubscribe workers
-   reserve before creating dispatch outbox messages
-   release after execution
-   ignore offline workers
-   reevaluate waiting jobs

Workers MUST NOT

-   schedule jobs
-   reserve resources
-   release resources
-   modify worker capacity

------------------------------------------------------------------------

# 16. Runtime States

Scheduling-related states

``` text
waiting_dependencies
waiting_resources
queued
running
retrying
dead_lettered
```

------------------------------------------------------------------------

# 17. Failure Scenarios

## Worker offline

Retry if attempts remain.

------------------------------------------------------------------------

## Database unavailable

No reservation.

Scheduler retries later.

------------------------------------------------------------------------

## RabbitMQ unavailable

OutboxMessage remains pending.

JobRun remains queued with reserved resources.

Outbox Publisher retries publication later.

------------------------------------------------------------------------

## Docker unavailable

Infrastructure failure.

Retry policy applies.

------------------------------------------------------------------------

# 18. ADR

## ADR-001

Decision

Scheduler owns resource allocation and release.

Consequence

Workers remain stateless executors.

------------------------------------------------------------------------

## ADR-002

Decision

CPU and Memory are mandatory scheduling constraints.

Consequence

Simple and deterministic scheduling.

------------------------------------------------------------------------

## ADR-003

Decision

Reservation and dispatch outbox creation occur in one transaction.

Consequence

No worker can receive a job without reserved capacity, and RabbitMQ
outages do not lose dispatch messages.

------------------------------------------------------------------------

## ADR-004

Decision

Jobs exceeding every worker capacity fail immediately.

Consequence

Avoid infinite waiting.

------------------------------------------------------------------------

# 19. Future Evolution

Candidates

-   Worker labels
-   Affinity rules
-   Priority queues
-   Fair scheduling
-   Load balancing
-   GPU resources
-   Dynamic worker capacity
