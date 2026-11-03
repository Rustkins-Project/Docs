---
id: DOC-0011
title: "Worker Architecture (Rustkins V1)"
type: architecture
status: draft
summary: "Architecture et responsabilités du Worker V1 : récupération Git, exécution Docker, logs, artéfacts et états des jobs."
topics: [worker, docker, git, execution]
applies_to: [V1]
---

# ARCH-09 --- Worker Architecture (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies the internal architecture and responsibilities
of a Rustkins Worker.

The Worker is responsible for executing JobRuns assigned by the
Scheduler.

It is **not** responsible for scheduling, dependency resolution or
resource allocation.

Out of scope:

-   Scheduler
-   Kubernetes
-   Auto-scaling
-   Multi-runtime execution

------------------------------------------------------------------------

# 2. Responsibilities

The Worker is responsible for:

-   Consuming JobRun messages from RabbitMQ
-   Validating JobRun state
-   Cloning the repository
-   Checking out the requested Git SHA
-   Preparing the execution environment
-   Restoring upstream artifacts
-   Executing the Docker container
-   Streaming logs
-   Uploading artifacts
-   Updating runtime state
-   Reporting completion to the Scheduler

The Worker MUST NEVER:

-   Schedule jobs
-   Modify the execution DAG
-   Allocate resources
-   Release reserved resources
-   Create PipelineRuns

------------------------------------------------------------------------

# 3. High-Level Architecture

``` mermaid
flowchart LR

RabbitMQ --> Worker
Worker --> PostgreSQL
Worker --> Git
Worker --> Docker
Docker --> Worker
Worker --> RustFS
Worker --> RabbitMQ
```

------------------------------------------------------------------------

# 4. Execution Lifecycle

``` mermaid
stateDiagram-v2
[*] --> Idle
Idle --> JobReceived
JobReceived --> Preparing
Preparing --> Running
Running --> UploadingArtifacts
UploadingArtifacts --> Finalizing
Finalizing --> Idle
Running --> Failed
Failed --> Idle
```

------------------------------------------------------------------------

# 5. Job Processing Sequence

``` mermaid
sequenceDiagram

participant MQ as RabbitMQ
participant W as Worker
participant DB as PostgreSQL
participant Git
participant Docker
participant RustFS

MQ->>W: Job message
W->>DB: Load JobRun
W->>DB: Validate status
W->>Git: Clone repository
W->>Git: Checkout commit SHA
W->>RustFS: Restore upstream artifacts
W->>Docker: Start container
Docker-->>W: stdout / stderr
W->>RustFS: Persist logs
W->>RustFS: Upload artifacts
W->>DB: Update JobRun
W->>MQ: ACK
```

------------------------------------------------------------------------

# 6. Repository Preparation

Before execution the Worker:

1.  Clones the repository.
2.  Checks out the Git SHA referenced by the PipelineRun.
3.  Loads the workspace.
4.  Restores artifacts from dependency JobRuns.
5.  Starts the execution container.

The Worker must always execute the exact Git revision associated with
the PipelineVersion.

------------------------------------------------------------------------

# 7. Docker Execution

Each JobRun executes inside its own container.

The Worker applies:

-   CPU limits
-   Memory limits
-   Working directory
-   Environment variables
-   Mounted workspace

Container lifetime equals JobRun lifetime.

------------------------------------------------------------------------

# 8. Step Warning Handling

The Worker executes steps sequentially.

If a step exits with code `0`:

``` text
StepRun = success
```

If a step exits with a non-zero code and
`PipelineVersionStep.continue_on_error = true`:

``` text
StepRun = warning
```

The Worker continues with the next step.

If no later blocking failure occurs, the JobRun finishes as `warning`.

If a step exits with a non-zero code and
`PipelineVersionStep.continue_on_error = false`:

``` text
StepRun = failed
```

The Worker stops the remaining steps.

If the parent PipelineVersionJob is optional, the JobRun finishes as
`warning`.

Otherwise, the JobRun finishes as `failed`.

------------------------------------------------------------------------

# 9. Log Management

Container output is continuously captured.

For each chunk:

1.  Append to RustFS.
2.  Publish to RabbitMQ.
3.  Preserve ordering using sequence numbers.

Persistent storage always occurs before live transport.

------------------------------------------------------------------------

# 10. Artifact Upload

When execution finishes:

1.  Discover configured artifact paths.
2.  Validate paths.
3.  Create tar.gz archive(s).
4.  Compute SHA-256 checksum.
5.  Upload to RustFS.
6.  Persist Artifact metadata.

------------------------------------------------------------------------

# 11. Status Updates

The Worker updates JobRun states:

``` text
running
success
warning
failed
cancelled
```

The Worker reports infrastructure failures through the JobRun state and
error details.

The Scheduler decides whether the next state becomes `retrying` or
`dead_lettered`.

------------------------------------------------------------------------

# 12. Cancellation

When a cancel request is received:

1.  Mark execution as cancelling.
2.  Stop the Docker container.
3.  Flush remaining logs.
4.  Upload completed artifacts (if applicable).
5.  Persist final state.
6.  Let the Scheduler release reserved resources.

If the container cannot be stopped within the timeout, the JobRun
transitions to `failed`.

------------------------------------------------------------------------

# 13. Idempotency

Before processing a message the Worker reloads the JobRun.

If the JobRun is already completed:

-   Success
-   Failed
-   Cancelled

The message is acknowledged without execution.

Duplicate execution must never occur.

------------------------------------------------------------------------

# 14. Heartbeat

Each Worker periodically publishes its health.

Heartbeat updates:

-   last_heartbeat_at
-   status

Scheduler uses this information to detect offline Workers.

------------------------------------------------------------------------

# 15. Failure Handling

Typical infrastructure failures:

-   Docker unavailable
-   Git clone failure
-   RustFS unavailable
-   Network timeout

These failures follow the retry policy.

Business failures (tests, compilation, commands) do not trigger retries.

------------------------------------------------------------------------

# 16. Runtime Invariants

-   One Worker executes one JobRun at a time per reserved slot.
-   One JobRun executes on only one Worker.
-   One Docker container belongs to one JobRun.
-   Workers never modify scheduling decisions.
-   Workers never reserve or release worker capacity.
-   PostgreSQL remains the source of truth.

------------------------------------------------------------------------

# 17. ADR

## ADR-001

Decision

Workers are stateless executors.

Consequence

All execution state is stored externally.

------------------------------------------------------------------------

## ADR-002

Decision

Repository is checked out at the exact Git SHA.

Consequence

Execution remains reproducible.

------------------------------------------------------------------------

## ADR-003

Decision

Logs are persisted before live streaming.

Consequence

History survives API failures.

------------------------------------------------------------------------

## ADR-004

Decision

Artifact upload is performed by the Worker.

Consequence

Execution output remains local until completion.

------------------------------------------------------------------------

# 18. Future Evolution

Candidates

-   Workspace cache
-   Parallel Git clone optimization
-   Container reuse
-   Sandboxed execution
-   Multiple execution backends
-   Remote execution
