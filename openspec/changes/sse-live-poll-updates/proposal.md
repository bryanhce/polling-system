## Why

Live polls require real-time updates so that organizers and participants can immediately see answers submitted by others without needing manual page refreshes or aggressive polling. Introducing Server-Sent Events (SSE) enables lightweight, low-latency, unidirectional streaming of new answers and poll status changes from the server to connected clients.

## What Changes

- Add an SSE streaming endpoint `GET /api/v1/polls/:pollId/events` on the backend.
- Implement an in-memory connection manager / lookup table service that tracks active polls and their connected subscriber clients, structured with an interface allowing future swap to Redis Pub/Sub.
- Broadcast new answers (`event: answer`) to all clients listening to that specific active poll when submitted.
- Broadcast poll closure (`event: poll_closed`) and disconnect subscribers when a poll is closed.
- Emit periodic keep-alive comments (`: ping\n\n`) to prevent proxy/browser timeout drops.
- Pre-validate poll existence and state before establishing SSE (return `404` if not found, `403` if already closed).
- Integrate `EventSource` in the frontend `useActivePollPage` hook to automatically receive new answers in real-time and update the active poll UI immediately.

## Capabilities

### New Capabilities
- `live-poll-stream`: Server-Sent Events (SSE) real-time streaming endpoint and subscription manager for active polls, broadcasting newly submitted answers, closure events, and keep-alive pings to connected clients.

### Modified Capabilities
<!-- Existing capabilities whose REQUIREMENTS are changing (not just implementation).
     Only list here if spec-level behavior changes. Each needs a delta spec file.
     Use existing spec names from openspec/specs/. Leave empty if no requirement changes. -->

## Impact

- **Backend**: Adds `GET /api/v1/polls/:pollId/events`, adds `PollEventBroadcaster` / connection registry abstraction, updates `AnswerService.submit()` and `PollService.close()` to broadcast events to active subscribers, and adds integration tests.
- **Frontend**: Updates `useActivePollPage` to open an `EventSource` connection to the SSE endpoint when viewing an active poll, prepends real-time incoming answers, and reacts to poll closure events.
- **API**: New SSE endpoint `GET /api/v1/polls/:pollId/events` with Content-Type `text/event-stream`.
