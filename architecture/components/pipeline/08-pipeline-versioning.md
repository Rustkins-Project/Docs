# ARCH-08 --- Pipeline Versioning (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document defines the Pipeline Versioning model used by Rustkins V1.

Goals

-   Immutable pipeline definitions
-   Reproducible executions
-   Historical consistency
-   Separation between configuration and execution
-   Reliable debugging of past runs

Out of scope

-   Version comparison UI
-   Manual edition
-   Rollback
-   Pipeline templates

------------------------------------------------------------------------

# 2. Motivation

A pipeline evolves over time.

Example

Commit A

``` yaml
jobs:
  build:
    image: node:20
```

Commit B

``` yaml
jobs:
  build:
    image: node:22
```

A PipelineRun executed on Commit A must continue referencing the exact
definition that existed at that moment.

For this reason every Git revision produces an immutable
PipelineVersion.

------------------------------------------------------------------------

# 3. Architecture

``` mermaid
flowchart LR

Pipeline --> PipelineVersion
PipelineVersion --> VersionJob
VersionJob --> VersionStep

PipelineVersion --> PipelineRun

PipelineRun --> JobRun
JobRun --> StepRun
```

Pipeline definitions are immutable.

Runtime objects remain mutable.

------------------------------------------------------------------------

# 4. Runtime Hierarchy

``` text
PROJECT
 └── PIPELINE
      ├── PIPELINE_VERSION
      │      ├── PIPELINE_VERSION_JOB
      │      │      ├── PIPELINE_VERSION_STEP
      │      │      └── PIPELINE_VERSION_JOB_DEPENDENCY
      │      └── PIPELINE_RUN
      │             ├── JOB_RUN
      │             └── STEP_RUN
```

------------------------------------------------------------------------

# 5. Logical Pipeline

PIPELINE represents the logical CI pipeline.

Contains

-   name
-   yaml_path
-   activation status

Never stores runtime information.

------------------------------------------------------------------------

# 6. Pipeline Version

Each PipelineVersion represents one immutable YAML snapshot.

Fields

-   git_sha
-   yaml_hash
-   yaml_content
-   validation_status
-   validation_errors

PipelineVersion is never modified.

If the YAML changes

A new PipelineVersion is created.

------------------------------------------------------------------------

# 7. Version Jobs

PIPELINE_VERSION_JOB stores the parsed definition of one job.

Contains

-   job_key
-   name
-   image
-   run_if
-   optional
-   cpu
-   memory
-   timeout

These definitions never change.

`optional` is part of the immutable job definition because it changes how
runtime failures are classified: an optional job may end in `warning`
instead of failing the PipelineRun.

------------------------------------------------------------------------

# 8. Version Steps

PIPELINE_VERSION_STEP stores every command.

Contains

-   step_key
-   name
-   command
-   continue_on_error
-   execution order

Steps are immutable.

`continue_on_error` is part of the immutable step definition because it
changes how a non-zero command exit is classified: the StepRun becomes
`warning` and the Worker continues with the next step.

------------------------------------------------------------------------

# 9. Version DAG

Dependencies are stored independently.

PIPELINE_VERSION_JOB_DEPENDENCY

Purpose

Represent the execution graph exactly as defined in YAML.

Example

``` text
build
  ↓
tests
  ↓
deploy
```

------------------------------------------------------------------------

# 10. Pipeline Run

PipelineRun references exactly one PipelineVersion.

Relationship

``` text
PipelineVersion 1 ---- N PipelineRuns
```

Several executions may reuse the same version.

Example

Manual retry

Nightly execution

Manual trigger

All reference the same PipelineVersion.

------------------------------------------------------------------------

# 11. Job Runs

JobRun references

``` text
PIPELINE_VERSION_JOB
```

JobRun stores only runtime state.

Examples

-   status
-   exit_code
-   worker
-   timestamps
-   retry count

The execution definition remains in PipelineVersionJob.

------------------------------------------------------------------------

# 12. Step Runs

StepRun references

``` text
PIPELINE_VERSION_STEP
```

Stores

-   status
-   exit code
-   timestamps

No command duplication exists.

------------------------------------------------------------------------

# 13. Version Creation

PipelineVersion creation sequence

``` mermaid
sequenceDiagram

participant API
participant Git
participant Parser
participant PostgreSQL

API->>Git: Read YAML at SHA
Git-->>API: YAML
API->>Parser: Parse
Parser-->>API: DAG
API->>PostgreSQL: Store PipelineVersion
```

Creation occurs only if the version does not already exist.

------------------------------------------------------------------------

# 14. Version Identification

Uniqueness

``` text
pipeline_id
git_sha
```

If the same SHA already exists

Existing PipelineVersion is reused.

------------------------------------------------------------------------

# 15. Validation

Every version is validated before execution.

Possible results

``` text
valid
invalid
```

Validation errors are preserved.

PipelineRuns may still be created in failed state.

------------------------------------------------------------------------

# 16. Advantages

PipelineVersion provides

-   immutable history
-   reproducible executions
-   simplified debugging
-   runtime consistency
-   deterministic retries

------------------------------------------------------------------------

# 17. Runtime Separation

Configuration

``` text
PIPELINE
PIPELINE_VERSION
PIPELINE_VERSION_JOB
PIPELINE_VERSION_STEP
PIPELINE_VERSION_JOB_DEPENDENCY
```

Execution

``` text
PIPELINE_RUN
JOB_RUN
STEP_RUN
JOB_RUN_DEPENDENCY
```

Configuration never changes during execution.

------------------------------------------------------------------------

# 18. Failure Scenarios

Missing YAML

PipelineRun = failed.

Invalid YAML

PipelineVersion created.

Validation status = invalid.

PipelineRun fails immediately.

Unknown Git SHA

No PipelineVersion created.

------------------------------------------------------------------------

# 19. ADR

## ADR-001

Decision

Pipeline definitions are immutable.

Consequence

Historical executions remain reproducible.

------------------------------------------------------------------------

## ADR-002

Decision

Configuration separated from runtime.

Consequence

Runtime tables remain lightweight.

------------------------------------------------------------------------

## ADR-003

Decision

PipelineVersion reused when SHA already exists.

Consequence

Storage duplication reduced.

------------------------------------------------------------------------

## ADR-004

Decision

PipelineRun always references one PipelineVersion.

Consequence

Every execution is historically traceable.

------------------------------------------------------------------------

# 20. Future Evolution

Candidates

-   Version diff
-   Version browser
-   Rollback
-   Pipeline comparison
-   YAML visualizer
-   Version promotion
