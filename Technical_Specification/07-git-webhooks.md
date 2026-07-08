# ARCH-07 --- Git Webhooks (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies how Rustkins receives, validates and processes
Git webhooks.

Supported providers:

-   GitHub
-   GitLab
-   Gitea

Objectives:

-   Automatic pipeline execution
-   Secure webhook validation
-   Pipeline discovery
-   Pipeline version creation
-   Duplicate protection

Out of scope:

-   Generic webhooks
-   Pull Request workflows
-   Release events
-   Tag events

------------------------------------------------------------------------

# 2. Global Architecture

``` mermaid
flowchart LR

Developer --> Git
Git --> Webhook
Webhook --> API
API --> PostgreSQL
API --> Scheduler
Scheduler --> PostgreSQL
PostgreSQL --> OutboxPublisher
OutboxPublisher --> RabbitMQ
RabbitMQ --> Worker
```

------------------------------------------------------------------------

# 3. Trigger Flow

``` mermaid
sequenceDiagram

participant Developer
participant Git
participant API
participant Scheduler
participant OutboxPublisher
participant RabbitMQ
participant Worker

Developer->>Git: git push
Git->>API: POST webhook
API->>API: Verify HMAC
API->>Scheduler: Build PipelineRun
Scheduler->>API: PipelineRun queued
Scheduler->>Scheduler: Create dispatch outbox messages
OutboxPublisher->>RabbitMQ: Publish job.ready
RabbitMQ->>Worker: Deliver job
```

------------------------------------------------------------------------

# 4. Supported Events

Rustkins V1 only reacts to:

``` text
push
```

All other events are ignored.

------------------------------------------------------------------------

# 5. Public Endpoints

One endpoint per provider.

Examples

``` text
POST /api/webhooks/github
POST /api/webhooks/gitlab
POST /api/webhooks/gitea
```

This isolates provider-specific payloads.

------------------------------------------------------------------------

# 6. Authentication

Every webhook must contain:

-   payload
-   signature

The API loads the configured webhook secret.

The signature is recomputed.

Comparison must use constant-time comparison.

Missing signature

``` text
401 Unauthorized
```

Invalid signature

``` text
401 Unauthorized
```

------------------------------------------------------------------------

# 7. Delivery Deduplication

Every received webhook has a provider delivery identifier.

Examples

GitHub

``` text
X-GitHub-Delivery
```

GitLab

``` text
X-Gitlab-Event-UUID
```

Stored in:

``` text
WEBHOOK_DELIVERY.external_delivery_id
```

If the delivery already exists:

Webhook is ignored.

------------------------------------------------------------------------

# 8. Internal Git Event

Provider payloads are normalized into one internal model.

Fields

``` text
provider
repository
branch
commit_sha
author_name
author_email
commit_message
```

Scheduler never consumes provider-specific payloads.

------------------------------------------------------------------------

# 9. Project Resolution

Repository URL received from the provider is matched against

``` text
PROJECT.repository_url
```

If no project exists:

``` text
404 Not Found
```

No PipelineRun is created.

------------------------------------------------------------------------

# 10. Pipeline Discovery

Pipelines are created from the Rustkins UI.

Each pipeline references

``` text
yaml_path
```

Examples

``` text
.rustkins/ci.yml
.rustkins/backend.yml
.rustkins/frontend.yml
```

Webhook processing never creates pipelines.

------------------------------------------------------------------------

# 11. Pipeline Version

Processing sequence

1.  Receive webhook
2.  Resolve project
3.  Locate active pipelines
4.  Read YAML at commit SHA
5.  Validate YAML
6.  Create PipelineVersion if needed
7.  Create PipelineRun
8.  Build execution DAG
9.  Schedule runnable jobs

PipelineVersion is immutable.

------------------------------------------------------------------------

# 12. Branch Conditions

Each pipeline evaluates branch rules.

Example

``` yaml
branches:
  include:
    - main
    - release/*
```

If the branch does not match

No PipelineRun is created.

------------------------------------------------------------------------

# 13. Multiple Pipelines

One project may contain several pipelines.

Example

``` text
.rustkins/
  ci.yml
  backend.yml
  frontend.yml
```

Every compatible pipeline generates its own PipelineRun.

------------------------------------------------------------------------

# 14. Invalid Pipeline

Invalid YAML

Result

``` text
PipelineRun = failed
```

Validation errors are attached to PipelineVersion.

History remains preserved.

------------------------------------------------------------------------

# 15. Missing Pipeline

If a configured YAML file no longer exists

Result

``` text
PipelineRun = failed
```

Reason

Configuration mismatch between Rustkins and repository.

------------------------------------------------------------------------

# 16. Failure Scenarios

Unknown repository

``` text
404
```

Invalid HMAC

``` text
401
```

Duplicate delivery

``` text
Ignored
```

Scheduler unavailable

``` text
Webhook accepted
Pipeline creation retried
```

------------------------------------------------------------------------

# 17. Webhook Delivery Lifecycle

``` mermaid
stateDiagram-v2

[*] --> received
received --> validated
validated --> processed
validated --> rejected
processed --> completed
```

Status values

``` text
received
validated
processed
completed
rejected
```

------------------------------------------------------------------------

# 18. Security

Every request validates

-   HMAC
-   provider endpoint
-   project existence

No pipeline execution occurs before validation succeeds.

------------------------------------------------------------------------

# 19. ADR

## ADR-001

Decision

One endpoint per provider.

Consequence

Simpler payload parsing.

------------------------------------------------------------------------

## ADR-002

Decision

Webhook never creates projects or pipelines.

Consequence

Rustkins configuration remains explicit.

------------------------------------------------------------------------

## ADR-003

Decision

PipelineVersion created from Git SHA.

Consequence

Historical executions remain reproducible.

------------------------------------------------------------------------

## ADR-004

Decision

Duplicate deliveries ignored.

Consequence

Idempotent webhook processing.

------------------------------------------------------------------------

# 20. Future Evolution

Candidates

-   Pull Request events
-   Tag events
-   Automatic webhook creation
-   Repository synchronization
-   Generic webhook endpoint
-   Organization integrations
