# ARCH-10 --- Data Model Reference (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document explains the role of every entity composing the Rustkins
V1 data model.

The Entity Relationship Diagram (ERD) defines the structure. This
document explains the semantics, ownership rules, lifecycle and
relationships.

------------------------------------------------------------------------

# 2. Model Overview

``` mermaid
flowchart TD

PROJECT --> PIPELINE
PIPELINE --> PIPELINE_VERSION
PIPELINE_VERSION --> PIPELINE_RUN

PIPELINE_VERSION --> PIPELINE_VERSION_JOB
PIPELINE_VERSION_JOB --> PIPELINE_VERSION_STEP

PIPELINE_RUN --> JOB_RUN
JOB_RUN --> STEP_RUN

JOB_RUN --> ARTIFACT
JOB_RUN --> JOB_LOG_STREAM
PIPELINE_RUN --> OUTBOX_MESSAGE
```

The model is intentionally divided into three domains:

-   Identity & Security
-   Pipeline Configuration
-   Runtime Execution

------------------------------------------------------------------------

# 3. Identity Domain

## USER

Represents one authenticated Rustkins user.

Responsibilities

-   Login
-   Project membership
-   Manual pipeline execution
-   Pipeline cancellation

Lifecycle

Created once. Never deleted physically.

------------------------------------------------------------------------

## USER_REFRESH_TOKEN

Represents one refresh token bound to a device/session.

Characteristics

-   hashed
-   revocable
-   expires
-   one record per session

------------------------------------------------------------------------

## PROJECT_MEMBER

Associates a User with a Project.

Stores

-   role
-   creation date

Valid roles

``` text
owner
maintainer
developer
viewer
```

------------------------------------------------------------------------

# 4. Project Domain

## PROJECT

Root entity.

Represents one Git repository managed by Rustkins.

Owns

-   members
-   pipelines
-   API tokens
-   webhook configuration
-   retention policy
-   pipeline runs

------------------------------------------------------------------------

## API_TOKEN

Authentication for automation.

Characteristics

-   project scoped
-   hashed
-   scopes
-   optional expiration

------------------------------------------------------------------------

## WEBHOOK_CONFIG

Stores webhook provider configuration.

Contains

-   provider
-   encrypted HMAC secret
-   enabled flag

One project may expose several webhook configurations.

------------------------------------------------------------------------

## WEBHOOK_DELIVERY

Historical record of received webhook deliveries.

Purpose

-   deduplication
-   audit
-   diagnostics

Never deleted automatically.

------------------------------------------------------------------------

# 5. Pipeline Definition Domain

## PIPELINE

Logical CI pipeline.

Represents

``` text
.rustkins/ci.yml
```

Stores

-   name
-   yaml path
-   activation state

Does not store execution data.

------------------------------------------------------------------------

## PIPELINE_VERSION

Immutable snapshot of a pipeline.

Created for one Git SHA.

Stores

-   YAML
-   validation result
-   hash

One Pipeline owns many versions.

Versions never change.

------------------------------------------------------------------------

## PIPELINE_VERSION_JOB

Represents one parsed job.

Contains immutable execution definition:

-   image
-   resources
-   timeout
-   run_if
-   optional

`optional = true` means that business failures on this job produce a
JobRun `warning` instead of a PipelineRun-blocking `failed`.

------------------------------------------------------------------------

## PIPELINE_VERSION_STEP

Represents one immutable command.

Contains

-   command
-   order
-   continue_on_error

`continue_on_error = true` means that a non-zero command exit produces a
StepRun `warning` and does not stop the remaining steps of the same job.

------------------------------------------------------------------------

## PIPELINE_VERSION_JOB_DEPENDENCY

Represents the immutable DAG.

Purpose

Describe execution order exactly as defined in YAML.

------------------------------------------------------------------------

# 6. Runtime Domain

## PIPELINE_RUN

Represents one execution.

References

-   project
-   pipeline
-   pipeline version

Stores

-   trigger
-   status
-   branch
-   commit
-   author

------------------------------------------------------------------------

## JOB_RUN

Runtime execution of one PipelineVersionJob.

Stores only runtime state.

Examples

-   worker
-   status
-   retry count
-   exit code
-   timestamps

`warning` is a final non-blocking status used when a job completed with
non-blocking issues.

------------------------------------------------------------------------

## JOB_RUN_DEPENDENCY

Runtime DAG.

Allows scheduler to reason on execution state without modifying
immutable definitions.

------------------------------------------------------------------------

## JOB_RUN_ATTEMPT

One retry attempt.

Purpose

Keep retry history independently from JobRun.

------------------------------------------------------------------------

## STEP_RUN

Execution state of one PipelineVersionStep.

Contains

-   status
-   exit code
-   timestamps

`warning` is used when the step command failed but the corresponding
PipelineVersionStep has `continue_on_error = true`.

------------------------------------------------------------------------

## RUN_EVENT

Business events produced during execution.

Examples

``` text
job_started
job_finished
artifact_uploaded
retry
cancelled
```

Used for audit and diagnostics.

------------------------------------------------------------------------

## OUTBOX_MESSAGE

Durable message waiting to be published to RabbitMQ.

Purpose

-   bridge PostgreSQL transactions and RabbitMQ publication
-   avoid losing dispatch messages after a database commit
-   make RabbitMQ publication retryable

Stores

-   optional pipeline run reference
-   optional job run reference
-   exchange
-   routing key
-   event type
-   payload
-   status
-   attempts
-   next retry date
-   last error

Status values

``` text
pending
publishing
published
failed
```

The Scheduler creates OutboxMessages inside the same transaction as the
runtime state change.

The Outbox Publisher is the only component allowed to mark an
OutboxMessage as `published`.

------------------------------------------------------------------------

# 7. Infrastructure Domain

## WORKER_NODE

Represents one execution node.

Stores

-   capacity
-   reservations
-   heartbeat
-   status

Scheduler is the only component allowed to reserve and release capacity.

------------------------------------------------------------------------

## JOB_LOG_STREAM

Metadata describing one persistent log stream.

Binary log content is stored in RustFS.

------------------------------------------------------------------------

## LOG_EVENT

Transient log chunk.

Transported through RabbitMQ and SSE.

Not persisted in PostgreSQL.

------------------------------------------------------------------------

## ARTIFACT

Metadata describing one uploaded archive.

Stores

-   storage key
-   checksum
-   size
-   expiration

Archive bytes remain in RustFS.

------------------------------------------------------------------------

## ARTIFACT_RETENTION_POLICY

Project retention configuration.

Defines

-   default retention
-   maximum retention
-   size limits

------------------------------------------------------------------------

# 8. Entity Lifecycles

## Immutable

``` text
PIPELINE_VERSION
PIPELINE_VERSION_JOB
PIPELINE_VERSION_STEP
PIPELINE_VERSION_JOB_DEPENDENCY
```

## Mutable

``` text
PIPELINE_RUN
JOB_RUN
STEP_RUN
WORKER_NODE
OUTBOX_MESSAGE
```

## Ephemeral

``` text
LOG_EVENT
```

------------------------------------------------------------------------

# 9. Referential Rules

-   Every PipelineRun references exactly one PipelineVersion.
-   Every JobRun references exactly one PipelineVersionJob.
-   Every StepRun references exactly one PipelineVersionStep.
-   Every Artifact belongs to one JobRun.
-   Every Worker executes many JobRuns over time.
-   Every runtime RabbitMQ message is first persisted as an OutboxMessage.
-   Pipeline definitions never depend on runtime entities.

------------------------------------------------------------------------

# 10. ADR

## ADR-001

Configuration and runtime are separated.

Consequence

Pipeline history remains immutable.

------------------------------------------------------------------------

## ADR-002

Binary data is never stored in PostgreSQL.

Consequence

Database remains lightweight.

------------------------------------------------------------------------

## ADR-003

Runtime tables store only execution state.

Consequence

Configuration duplication is minimized.

------------------------------------------------------------------------

## ADR-004

PipelineVersion is the historical source of truth.

Consequence

Every execution remains reproducible.

------------------------------------------------------------------------

# 11. Future Evolution

Candidates

-   Organizations
-   Teams
-   Secret entities
-   Cache entities
-   Matrix execution
-   Kubernetes resources
