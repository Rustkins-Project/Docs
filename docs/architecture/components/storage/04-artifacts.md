---
id: DOC-0010
title: "Artifacts (Rustkins V1)"
type: architecture
status: draft
summary: "Gestion des artéfacts produits par les jobs, transfert depuis les workers, stockage RustFS et accès aux résultats."
topics: [artifacts, rustfs, storage, worker]
applies_to: [V1]
---

# ARCH-04 --- Artifacts (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies the artifact management architecture of Rustkins
V1.

Artifacts are files produced by a JobRun and made available after
execution.

Typical examples:

-   compiled binaries
-   archives
-   reports
-   test results
-   coverage reports
-   generated documentation

Out of scope

-   Artifact versioning
-   Deduplication
-   Cache
-   Incremental uploads

------------------------------------------------------------------------

# 2. Global Architecture

``` mermaid
flowchart LR

Docker --> Worker
Worker --> TarArchive
TarArchive --> RustFS
RustFS --> Worker

Worker --> PostgreSQL

Browser --> API
API --> RustFS
```

RustFS is the persistent storage.

PostgreSQL stores metadata only.

------------------------------------------------------------------------

# 3. Artifact lifecycle

``` mermaid
stateDiagram-v2

[*] --> Created
Created --> Uploaded
Uploaded --> Available
Available --> Expired
Expired --> Deleted
Deleted --> [*]
```

------------------------------------------------------------------------

# 4. Artifact creation

Each JobRun may declare artifact patterns.

Example YAML

``` yaml
artifacts:
  paths:
    - build/**
    - reports/**
```

At the end of execution the Worker:

1.  searches matching files
2.  validates paths
3.  archives files
4.  uploads archive to RustFS
5.  creates metadata in PostgreSQL

------------------------------------------------------------------------

# 5. Supported patterns

Supported

``` text
*
**
?
```

Examples

``` text
build/**
dist/*.zip
reports/**/*.xml
```

Patterns outside the workspace are rejected.

------------------------------------------------------------------------

# 6. Archive format

Archive format

``` text
tar.gz
```

Reasons

-   portable
-   preserves permissions
-   efficient compression
-   easy extraction

One archive is produced per Artifact.

------------------------------------------------------------------------

# 7. Validation rules

Forbidden

-   path traversal (../)
-   absolute paths
-   broken symbolic links
-   unreadable files

Allowed

-   regular files
-   directories
-   empty directories

If one artifact fails validation

Result

``` text
JobRun = warning
PipelineRun = warning (optional job) or failed (mandatory upload failure)
```

------------------------------------------------------------------------

# 8. Size limits

Default values

``` text
Maximum artifact size:
100 MB

Maximum artifacts per PipelineRun:
500 MB
```

Limits are configurable through:

ArtifactRetentionPolicy

If exceeded

Artifact upload fails.

------------------------------------------------------------------------

# 9. Retention

Default retention

``` text
30 days
```

Each project defines

-   default retention
-   maximum retention

Expired artifacts become unavailable.

A background cleanup service removes expired archives.

------------------------------------------------------------------------

# 10. Automatic restore

Artifacts produced by dependency jobs are restored automatically before a
dependent JobRun starts.

This implements the pipeline specification rule:

``` text
jobs listed in needs expose their artifacts to the dependent job
```

Restore scope

-   only direct dependencies listed in `needs`
-   only artifacts produced by successful dependency JobRuns
-   only artifacts belonging to the same PipelineRun
-   no cross-pipeline artifact restore

Restore sequence

1.  Worker loads dependency JobRuns from PostgreSQL.
2.  Worker lists available Artifact metadata for these JobRuns.
3.  Worker downloads artifact archives from RustFS.
4.  Worker verifies SHA-256 checksums.
5.  Worker extracts archives into the job workspace.
6.  Worker starts the Docker container.

Artifacts are restored after repository checkout and before container
execution.

Archive paths are extracted relative to the workspace root.

Path traversal and absolute paths remain forbidden during restore.

If two restored archives contain the same destination path, restore fails.
V1 does not define conflict resolution or selective artifact download.

------------------------------------------------------------------------

# 11. Download

Endpoint

``` text
GET /api/artifacts/{artifactId}
```

Downloads never expose RustFS directly.

API generates a signed URL.

Flow

``` mermaid
sequenceDiagram

Browser->>API: Download artifact
API->>PostgreSQL: Validate permissions
API->>RustFS: Generate signed URL
RustFS-->>Browser: Download archive
```

------------------------------------------------------------------------

# 12. Security

Downloads require:

-   authenticated user OR API token
-   project authorization

Viewer role may download artifacts.

API Tokens require appropriate scopes.

Signed URLs have limited lifetime.

------------------------------------------------------------------------

# 13. Cleanup

Expired artifacts

``` text
deleted = true
deleted_at populated
```

RustFS object removed.

Metadata preserved for history.

------------------------------------------------------------------------

# 14. Worker responsibilities

Worker

-   locate files
-   validate patterns
-   archive files
-   calculate checksum
-   upload archive
-   create Artifact metadata
-   restore upstream artifact archives before dependent job execution

Worker never deletes artifacts.

------------------------------------------------------------------------

# 15. Metadata

Artifact table stores

-   archive name
-   storage key
-   size
-   checksum
-   content type
-   expiration date

Archive content is never stored in PostgreSQL.

------------------------------------------------------------------------

# 16. Failure scenarios

## Upload failure

Retryable.

Worker retries according to runtime retry policy.

------------------------------------------------------------------------

## RustFS unavailable

Retry.

------------------------------------------------------------------------

## Restore failure

Infrastructure restore failures are retryable.

Examples:

-   RustFS unavailable
-   temporary network timeout

Business restore failures are not retryable.

Examples:

-   checksum mismatch
-   path traversal
-   destination path collision
-   artifact expired or deleted

------------------------------------------------------------------------

## Artifact too large

Business failure.

No retry.

------------------------------------------------------------------------

## Invalid path

Business failure.

No retry.

------------------------------------------------------------------------

## Partial upload

Archive discarded.

Metadata not created.

------------------------------------------------------------------------

# 17. Checksums

Every uploaded archive generates

``` text
SHA-256
```

Purpose

-   integrity verification
-   corruption detection

Checksum is stored in PostgreSQL.

------------------------------------------------------------------------

# 18. ADR

## ADR-001

Decision

Artifacts are archived before upload.

Consequence

Single upload operation.

------------------------------------------------------------------------

## ADR-002

Decision

RustFS stores binary content.

Consequence

Database remains lightweight.

------------------------------------------------------------------------

## ADR-003

Decision

Downloads use signed URLs.

Consequence

RustFS stays private.

------------------------------------------------------------------------

## ADR-004

Decision

Metadata is preserved after deletion.

Consequence

Historical PipelineRuns remain consistent.

------------------------------------------------------------------------

# 19. Future evolution

Candidates

-   Artifact browser
-   Preview support
-   Compression choice
-   Partial downloads
-   Selective dependency artifact download
-   Object deduplication
-   Storage quotas
