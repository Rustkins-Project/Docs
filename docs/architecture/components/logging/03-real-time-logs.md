# ARCH-03 --- Real-time Logs (Rustkins V1)

**Status:** Draft (Implementation Reference)

------------------------------------------------------------------------

# 1. Purpose

This document specifies the real-time log architecture of Rustkins V1.

Goals:

-   Near real-time log streaming
-   Low infrastructure complexity
-   Persistent logs
-   Automatic reconnection
-   Support multiple UI clients

Out of scope:

-   Redis
-   WebSocket
-   Infinite replay
-   Log indexing

------------------------------------------------------------------------

# 2. Global Architecture

``` mermaid
flowchart LR

Docker --> Worker
Worker --> RabbitMQ
Worker --> RustFS

RabbitMQ --> API
API --> SSE
SSE --> Browser

Browser -. reconnect .-> API
Browser -. history .-> RustFS
```

RabbitMQ is used only for live transport.

RustFS stores the complete log history.

PostgreSQL stores metadata only.

------------------------------------------------------------------------

# 3. Why Server-Sent Events (SSE)

Advantages

-   Native browser support
-   HTTP compatible
-   Automatic reconnection
-   One-way communication perfectly matches log streaming
-   Simpler than WebSocket

Rejected alternatives

### Polling

Problems:

-   unnecessary requests
-   latency
-   higher API load

### WebSocket

Rejected because:

-   bidirectional communication unnecessary
-   more infrastructure complexity
-   more connection state to maintain

------------------------------------------------------------------------

# 4. Runtime flow

``` mermaid
sequenceDiagram
participant Docker
participant Worker
participant MQ
participant API
participant Browser
participant RustFS

Docker->>Worker: stdout/stderr
Worker->>RustFS: append log
Worker->>MQ: publish chunk
MQ->>API: chunk
API->>Browser: SSE event

Browser->>API: reconnect
API->>RustFS: read history
API->>Browser: replay
```

------------------------------------------------------------------------

# 5. Log lifecycle

1.  Container writes stdout/stderr.
2.  Worker receives output.
3.  Output appended to RustFS.
4.  Chunk published to RabbitMQ.
5.  API forwards chunk through SSE.
6.  Browser renders chunk.

RustFS is always the persistent source.

RabbitMQ is best-effort.

------------------------------------------------------------------------

# 6. Storage model

## JOB_LOG_STREAM

Purpose

Metadata for one log stream.

Contains

-   storage_key
-   last_sequence
-   timestamps

## LOG_EVENT

Runtime representation

``` text
sequence
stream
content
emitted_at
```

LOG_EVENT is **not** persisted in PostgreSQL in V1.

It represents the structure exchanged through RabbitMQ and SSE.

------------------------------------------------------------------------

# 7. Chunk format

``` json
{
  "job_run_id":"uuid",
  "sequence":15,
  "stream":"stdout",
  "content":"Running integration tests...",
  "timestamp":"2026-01-01T12:00:00Z"
}
```

Maximum chunk size

``` text
16 KB
```

Streams

``` text
stdout
stderr
system
```

Sequence numbers are strictly increasing.

------------------------------------------------------------------------

# 8. SSE endpoint

Endpoint

``` text
GET /api/jobs/{jobRunId}/logs/stream
```

Response

``` text
Content-Type: text/event-stream
```

Each event contains

-   sequence
-   stream
-   content
-   timestamp

------------------------------------------------------------------------

# 9. Reconnection

Browser reconnects automatically.

Preferred mechanism

``` text
Last-Event-ID
```

If replay is unavailable:

1.  API reloads missing content from RustFS.
2.  Stream resumes.

------------------------------------------------------------------------

# 10. Memory fallback

API maintains a very short in-memory buffer.

Purpose

-   absorb temporary latency
-   avoid immediate RustFS reads
-   simplify reconnection

This cache is **volatile**.

Loss is acceptable because RustFS remains authoritative.

------------------------------------------------------------------------

# 11. Failure scenarios

## API unavailable

Worker continues:

-   writing RustFS
-   publishing RabbitMQ

When API returns:

Browser reloads logs from RustFS.

------------------------------------------------------------------------

## RabbitMQ unavailable

Worker continues writing RustFS.

Live updates temporarily unavailable.

------------------------------------------------------------------------

## Worker crash

Current JobRun ends.

Logs already stored in RustFS remain available.

------------------------------------------------------------------------

## Browser refresh

UI reconnects.

Missing history loaded from RustFS.

------------------------------------------------------------------------

# 12. Security

Access requires project authorization.

Viewer role may read logs.

Logs inherit PipelineRun permissions.

SSE endpoint requires JWT.

------------------------------------------------------------------------

# 13. Performance assumptions

Design target

-   multiple concurrent viewers
-   one writer per JobRun
-   sequential append
-   no random writes

Chunk transport is lightweight.

Large histories are read from RustFS only when necessary.

------------------------------------------------------------------------

# 14. ADR

## ADR-001

Decision

SSE replaces WebSocket.

Consequence

Lower implementation complexity.

------------------------------------------------------------------------

## ADR-002

Decision

RabbitMQ transports only live chunks.

Consequence

Persistent history remains independent from messaging.

------------------------------------------------------------------------

## ADR-003

Decision

RustFS is the authoritative log storage.

Consequence

Logs survive API restarts and browser reconnections.

------------------------------------------------------------------------

## ADR-004

Decision

Small in-memory API buffer.

Consequence

Redis is unnecessary in V1.

------------------------------------------------------------------------

# 15. Future evolution

Candidates

-   Redis replay cache
-   Log compression
-   Search
-   ANSI color support
-   Download complete logs
-   Log retention policies
