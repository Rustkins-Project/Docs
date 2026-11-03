# ARCH-01 --- Runtime & Execution Lifecycle (Rustkins V1)

**Status:** Draft (Implementation Reference)\
**Scope:** Rustkins V1

------------------------------------------------------------------------

# 1. Purpose

This document specifies the runtime execution model of Rustkins V1.

It is the reference for implementing:

-   API
-   Scheduler
-   Worker
-   RabbitMQ integration
-   Pipeline execution
-   Retry mechanism
-   Cancellation
-   Resource scheduling

It intentionally does **not** cover:

-   UI
-   Observability
-   Metrics
-   Kubernetes
-   Matrix builds

------------------------------------------------------------------------

# 2. Vocabulary

  -----------------------------------------------------------------------
  Term                    Definition
  ----------------------- -----------------------------------------------
  Project                 Repository managed by Rustkins

  Pipeline                Logical pipeline configured in Rustkins

  Pipeline Version        Immutable snapshot of a YAML definition for one
                          Git SHA

  Pipeline Run            One execution of one Pipeline Version

  Job Run                 Runtime execution of one job

  Step Run                Runtime execution of one step

  Worker                  Machine executing Docker containers

  Scheduler               Component deciding which job can start

  Artifact                File produced by a JobRun
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 3. High level architecture

``` mermaid
flowchart LR

Git --> API
API --> PipelineVersion
PipelineVersion --> Scheduler
Scheduler --> PostgreSQL
PostgreSQL --> OutboxPublisher
OutboxPublisher --> RabbitMQ
RabbitMQ --> Worker
Worker --> Docker
Docker --> RustFS
Worker --> PostgreSQL
```

Responsibilities:

  Component    Responsibility
  ------------ -----------------------------------------
  API          Receive requests and webhooks
  Scheduler    Build execution graph and dispatch jobs
  OutboxPublisher Publish durable outbox messages to RabbitMQ
  Worker       Execute Docker containers
  PostgreSQL   Source of truth
  RabbitMQ     Transport layer
  RustFS       Persistent storage

------------------------------------------------------------------------

# 4. Runtime model

Execution hierarchy

``` text
Project
 └── Pipeline
      └── Pipeline Version
           └── Pipeline Run
                ├── Job Run
                │      ├── Step Run
                │      ├── Logs
                │      └── Artifacts
                └── Job Run
```

Each PipelineVersion is immutable.

Each PipelineRun references exactly one PipelineVersion.

A PipelineVersion may be executed many times.

------------------------------------------------------------------------

# 5. Pipeline Run lifecycle

## States

  State       Meaning
  ----------- ------------------------------------------------------------
  queued      Created but not started
  running     At least one job running
  success     All mandatory jobs succeeded
  warning     No mandatory failure occurred, but at least one job ended in warning
  failed      Mandatory execution failed
  cancelled   Execution cancelled

State machine

``` mermaid
stateDiagram-v2
[*] --> queued
queued --> running
queued --> cancelled
running --> success
running --> warning
running --> failed
running --> cancelled
```

Transition rules

### queued → running

Conditions

-   pipeline version validated
-   DAG created
-   first runnable job queued

### running → success

Conditions

-   no JobRun = warning
-   no failed mandatory jobs
-   no dead_lettered jobs
-   skipped jobs are only skipped because their `run_if` condition was not satisfied

### running → warning

Conditions

-   no mandatory JobRun = failed
-   no JobRun = dead_lettered
-   one or more JobRuns = warning

### running → failed

Conditions

-   mandatory job failed
-   dead_lettered reached
-   cancellation timeout

### running → cancelled

User explicitly cancelled execution.

------------------------------------------------------------------------

# 6. Job lifecycle

States

``` text
waiting_dependencies
waiting_resources
queued
running
success
warning
failed
skipped
cancel_requested
cancelled
retrying
dead_lettered
```

``` mermaid
stateDiagram-v2
[*] --> waiting_dependencies
waiting_dependencies --> waiting_resources
waiting_dependencies --> queued
waiting_resources --> queued
queued --> running
queued --> cancelled
running --> success
running --> warning
running --> failed
running --> retrying
running --> cancel_requested
cancel_requested --> cancelled
cancel_requested --> failed
retrying --> queued
retrying --> dead_lettered
```

Detailed description

### waiting_dependencies

The DAG still blocks execution.

### waiting_resources

Dependencies satisfied but scheduler found no compatible worker.

### queued

Resources reserved.

A durable OutboxMessage exists for RabbitMQ dispatch.

The RabbitMQ message may already be published or still pending in the
outbox.

### running

Worker acknowledged message. Container started.

### success

Container finished with exit code 0.

### warning

Execution completed with a non blocking issue.

Examples:

-   a `continue_on_error` step failed
-   an optional job encountered a business failure
-   an artifact operation on an optional job failed

### failed

Business execution failure.

Examples

-   tests failed
-   build failed
-   script exit code != 0

If the JobRun is optional, a business failure produces `warning`
instead of `failed`.

Infrastructure failures still follow the retry policy.

### retrying

Infrastructure failure.

### dead_lettered

Maximum retries exceeded.

------------------------------------------------------------------------

# 7. Warning semantics

`warning` is a final non-blocking runtime state.

It preserves diagnostic information without failing the entire
PipelineRun.

## Step warning

A StepRun becomes `warning` when:

-   the command exits with a non-zero code
-   the PipelineVersionStep has `continue_on_error = true`

The Worker continues with the next step.

If no later blocking failure occurs, the parent JobRun finishes as
`warning`.

## Job warning

A JobRun becomes `warning` when:

-   at least one StepRun is `warning` and no blocking step failed
-   the PipelineVersionJob is `optional = true` and a business failure occurs
-   an artifact operation on an optional job fails

JobRun `warning` is treated as completed and non-blocking by the DAG.

## Pipeline warning

A PipelineRun becomes `warning` when:

-   at least one JobRun is `warning`
-   no mandatory JobRun is `failed`
-   no JobRun is `dead_lettered`

## Dependency evaluation

For `run_if`:

-   `all_success` accepts dependencies in `success` or `warning`
-   `always` accepts dependencies in any final state
-   `on_failure` only matches dependencies in `failed` or `dead_lettered`

------------------------------------------------------------------------

# 8. Scheduler responsibilities

The Scheduler is responsible for:

-   validating dependencies
-   evaluating resource availability
-   reserving worker capacity
-   releasing worker capacity
-   creating durable RabbitMQ outbox messages
-   updating runtime state

The Scheduler NEVER executes containers.

The Outbox Publisher is responsible for publishing pending outbox
messages to RabbitMQ.

------------------------------------------------------------------------

# 9. Retry strategy

Maximum attempts

``` text
3
```

Delays

``` text
Attempt 1 -> 30 s
Attempt 2 -> 2 min
Attempt 3 -> 5 min
```

Retry only infrastructure failures.

Retryable examples

-   Docker daemon unavailable
-   RustFS unavailable
-   RabbitMQ connection lost
-   PostgreSQL timeout
-   Worker crash

Non retryable

-   Tests failed
-   Exit code != 0
-   YAML validation
-   Build failed

State transition

``` mermaid
flowchart LR
FailedInfra --> Retrying
Retrying --> Queue
Retrying --> DLQ
```

------------------------------------------------------------------------

# 10. Cancellation

Queued jobs

-   status = cancelled
-   no RabbitMQ publication

Running jobs

1.  status = cancel_requested
2.  create cancel OutboxMessage
3.  worker stops container
4.  worker confirms cancellation
5.  scheduler releases resources
6.  status = cancelled

If worker disappears

status becomes failed.

------------------------------------------------------------------------

# 11. Idempotency

RabbitMQ provides at-least-once delivery.

Workers must therefore validate JobRun before execution.

Rules

-   already success -\> ignore
-   already failed -\> ignore
-   already cancelled -\> ignore
-   already running elsewhere -\> ignore

Duplicate messages are ACKed without execution.

------------------------------------------------------------------------

# 12. Runtime events

Pipeline events

-   pipeline_created
-   pipeline_started
-   pipeline_finished
-   pipeline_cancelled

Job events

-   job_waiting_dependencies
-   job_waiting_resources
-   job_queued
-   job_started
-   job_warning
-   job_retry
-   job_finished
-   job_cancel_requested
-   job_cancelled
-   job_dead_lettered

Storage events

-   artifact_uploaded
-   logs_completed

------------------------------------------------------------------------

# 13. Typical execution sequence

``` mermaid
sequenceDiagram
participant API
participant Scheduler
participant OutboxPublisher
participant RabbitMQ
participant Worker
participant Docker

API->>Scheduler: Create PipelineRun
Scheduler->>Scheduler: Build DAG
Scheduler->>Scheduler: Reserve resources
Scheduler->>Scheduler: Insert OutboxMessage(job.ready)
OutboxPublisher->>RabbitMQ: Publish outbox message
RabbitMQ->>Worker: Job message
Worker->>Docker: Start container
Docker-->>Worker: Exit code
Worker->>Scheduler: Update JobRun
Scheduler->>Scheduler: Release resources
Scheduler->>Scheduler: Unlock dependent jobs
```

------------------------------------------------------------------------

# 14. Runtime invariants

The following rules MUST always remain true.

-   One PipelineRun references exactly one PipelineVersion.
-   PipelineVersion objects are immutable.
-   One JobRun belongs to exactly one PipelineRun.
-   One StepRun belongs to exactly one JobRun.
-   Scheduler is the only component allowed to enqueue jobs.
-   Scheduler is the only component allowed to reserve or release worker capacity.
-   Every runtime RabbitMQ message is first persisted as an OutboxMessage.
-   Workers never modify DAG topology.
-   A completed JobRun never returns to running.
-   Dead-lettered jobs require manual retry.
-   PostgreSQL is the single source of truth.

------------------------------------------------------------------------

# 15. ADR

## ADR-001

Decision

Pipeline definitions are immutable through PipelineVersion.

Consequence

Historical executions remain reproducible.

------------------------------------------------------------------------

## ADR-002

Decision

Scheduler owns execution ordering.

Consequence

Workers remain stateless executors.

------------------------------------------------------------------------

## ADR-003

Decision

Retry only infrastructure failures.

Consequence

Business failures remain deterministic.

------------------------------------------------------------------------

## ADR-004

Decision

Dedicated waiting_resources state.

Consequence

Scheduler can distinguish dependency blocking from capacity blocking.

------------------------------------------------------------------------

# 16. Future evolution

Planned V2

-   Worker labels
-   Affinity
-   Priority scheduling
-   Matrix execution
-   Dynamic DAG generation
-   Multi-queue scheduling
