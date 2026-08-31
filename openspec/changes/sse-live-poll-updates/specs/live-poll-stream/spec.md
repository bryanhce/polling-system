## ADDED Requirements

### Requirement: SSE Endpoint Connection Establishment
The system SHALL provide an SSE endpoint at `GET /api/v1/polls/:pollId/events` that streams real-time updates for an active poll using `text/event-stream`.

#### Scenario: Client connects to an active poll
- **WHEN** a client sends a GET request to `/api/v1/polls/:pollId/events` for an existing active poll
- **THEN** the server responds with status 200, headers `Content-Type: text/event-stream`, `Cache-Control: no-cache, no-transform`, and `Connection: keep-alive`, and registers the client in the lookup table

#### Scenario: Client attempts connection to non-existent poll
- **WHEN** a client sends a GET request to `/api/v1/polls/:pollId/events` for a poll ID that does not exist
- **THEN** the server responds with status 404 and error code `POLL_NOT_FOUND` without establishing an SSE stream

#### Scenario: Client attempts connection to closed poll
- **WHEN** a client sends a GET request to `/api/v1/polls/:pollId/events` for a poll that is closed
- **THEN** the server responds with status 403 and error code `POLL_CLOSED` without establishing an SSE stream

### Requirement: Real-Time Answer Broadcast
The system SHALL broadcast every newly submitted answer to all connected SSE clients listening to that poll.

#### Scenario: New answer submitted to active poll with connected listeners
- **WHEN** an answer is successfully submitted via `POST /api/v1/polls/:pollId/answers`
- **THEN** all clients connected to `/api/v1/polls/:pollId/events` for that `pollId` receive an SSE event with name `answer` and payload `{"answer": "<submitted_answer>"}`

#### Scenario: Multiple clients connected to same poll
- **WHEN** multiple clients are subscribed to the same active `pollId` and an answer is submitted
- **THEN** every active client in the lookup table for that `pollId` receives the `answer` event

### Requirement: Poll Closure Event Broadcast and Connection Cleanup
The system SHALL broadcast a poll closure event and terminate client streams when a poll is closed.

#### Scenario: Creator closes an active poll
- **WHEN** an authorized creator closes a poll via `PATCH /api/v1/polls/:pollId/close`
- **THEN** all clients connected to `/api/v1/polls/:pollId/events` for that `pollId` receive an SSE event with name `poll_closed` and payload `{"pollId": "<pollId>"}`, their connections are closed by the server, and the poll is removed from the active lookup table

### Requirement: Connection Heartbeat and Disconnect Handling
The system SHALL maintain active connections with periodic keep-alive pings and clean up resources when clients disconnect.

#### Scenario: Periodic keep-alive ping
- **WHEN** an SSE connection remains open
- **THEN** the server sends periodic keep-alive comments `: ping\n\n` to prevent intermediate timeout

#### Scenario: Client disconnects
- **WHEN** a connected client closes the connection or navigates away
- **THEN** the server detects socket closure and removes the client from the lookup table

### Requirement: Frontend Real-Time Active Poll Stream Integration
The frontend active poll view SHALL connect to the poll events stream and reflect real-time changes.

#### Scenario: Real-time answer received on active poll page
- **WHEN** the frontend active poll page receives an `answer` SSE event
- **THEN** the new answer is prepended to the displayed list of answers in real-time

#### Scenario: Poll closed event received on active poll page
- **WHEN** the frontend active poll page receives a `poll_closed` SSE event
- **THEN** the page updates the poll status to closed, shows the poll closed notification, and closes the EventSource connection
