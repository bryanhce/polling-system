## Context

The polling system currently allows users to create polls, view active polls, submit answers, and close polls. However, participants and creators viewing an active poll must manually refresh or rely on periodic polling to see newly submitted answers. 

To deliver real-time interactivity, we are introducing Server-Sent Events (SSE). Clients viewing an active poll will establish a unidirectional HTTP connection to stream new answers and poll lifecycle events in real time.

## Goals / Non-Goals

**Goals:**
- Provide a persistent HTTP SSE streaming endpoint `GET /api/v1/polls/:pollId/events`.
- Implement an in-memory lookup table / connection manager (`PollEventBroadcaster`) that tracks active poll IDs and their connected HTTP response streams.
- Decouple the event broadcaster via an interface so it can be swapped with Redis Pub/Sub in multi-instance deployments in the future.
- Pre-validate poll status before opening the stream (404 if not found, 403 if closed).
- Push new answers (`event: answer`) immediately after persistence in `AnswerService.submit`.
- Push poll closure (`event: poll_closed`) and cleanly disconnect subscribers when `PollService.close` is called.
- Maintain connections with periodic keep-alive comments (`: ping\n\n`) to prevent timeouts.
- Update frontend hook `useActivePollPage` to connect via `EventSource`, prepend new answers in real-time, and react to poll closure.

**Non-Goals:**
- Bidirectional messaging or WebSockets (unidirectional server-to-client streaming via SSE meets all requirements).
- Full message replay / historical backfill over SSE (initial state is fetched via existing REST API `GET /api/v1/polls/:pollId/answers`).
- Redis implementation in the current scope (interface designed for future plug-in).

## Decisions

### 1. SSE Endpoint Path and Protocol
- **Decision**: Use `GET /api/v1/polls/:pollId/events` with standard SSE MIME type `text/event-stream`.
- **Rationale**: Follows RESTful URL hierarchy and standard browser `EventSource` protocol.
- **Alternatives Considered**: WebSockets (unnecessary protocol overhead and reconnection complexity for unidirectional updates); Long-polling (higher latency and connection overhead).

### 2. Broadcaster Service Abstraction
- **Decision**: Define a `PollEventBroadcaster` interface with methods:
  - `subscribe(pollId: string, client: SSEClient): () => void`
  - `broadcastAnswer(pollId: string, answer: Answer): void`
  - `broadcastPollClosed(pollId: string): void`
  - `closeAll(): void`
  And implement `InMemoryPollEventBroadcaster` holding `Map<string, Set<SSEClient>>`.
- **Rationale**: Keeps single-instance deployment simple with zero external dependencies while cleanly isolating connection lookup and broadcast logic, making it easy to swap with a Redis Pub/Sub implementation later.

### 3. Named Events and Message Framing
- **Decision**:
  - `event: answer\ndata: {"answer":"..."}\n\n`
  - `event: poll_closed\ndata: {"pollId":"..."}\n\n`
  - `: ping\n\n` (heartbeat sent every 15 seconds)
- **Rationale**: Named events allow the frontend to attach explicit event listeners via `EventSource.addEventListener('answer', ...)`. Standard `: ping` comment lines keep connections alive across proxies without triggering client event handlers.

### 4. Connection Lifecycle & Cleanup
- **Decision**:
  - Before establishing the SSE stream, query the database to ensure the poll exists (404 if missing) and is active (403 if status is `closed`).
  - When connection is accepted, set headers (`Content-Type: text/event-stream`, `Cache-Control: no-cache, no-transform`, `Connection: keep-alive`, `X-Accel-Buffering: no`), flush headers, and register the client.
  - Listen to `req.raw.on('close')` to automatically unregister disconnected clients.
  - When a poll is closed, send `poll_closed`, close all matching client response streams, and purge the poll entry from the lookup table.
  - Fastify `onClose` hook terminates all open streams.

### 5. Frontend Integration
- **Decision**: In `useActivePollPage`, instantiate `EventSource` once the poll is loaded and confirmed active.
  - On `answer` event: Prepend the new `PollAnswer` to `answers` state array.
  - On `poll_closed` event: Update `poll.status` to `'closed'`, display closed notice, and close the `EventSource`.
  - Clean up `EventSource` on component unmount or when poll status changes.

## Risks / Trade-offs

- **[Risk] Proxy/Gateway Connection Drops** → **Mitigation**: Periodic 15-second `: ping\n\n` heartbeat comments and `EventSource` built-in automatic reconnect.
- **[Risk] Memory Leaks from Orphaned Connections** → **Mitigation**: Hook into `req.raw.on('close')` and `res.on('finish')` to immediately remove subscribers; purge poll entries when empty; close all connections on server shutdown.
- **[Risk] Race Condition on Initial Load** → **Mitigation**: Client loads initial page via REST then connects to SSE; new answers prepend to list cleanly.
