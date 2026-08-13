# Requirements

## Functional
- User should be able to create a poll
- User should be able to answer a poll
    - Default expected behavior is 1 vote per user; however, since there is no authentication/user identification system, the system does not enforce or block multiple submissions per user for now
- User should be able to view a poll result (not real time requirement for now)
- User should be able to close a poll

Extension: Poll can have multiple types - MCQ (single or multiople selections), open ended. For simplicity, we do open ended string for now.

## Non Functional
- System should be highly available for answering polls, NOT choosing strong consistency
- System should be secure and rate limited

# Core Entities
- Poll: represents the poll itself
- Answer: represents an answer to a poll, a poll can have multiple answers submitted by users

# API
- POST /api/v1/polls

payload:
```json
{
    question: string,
    description: string?,
    pollType: MCQ | OpenEnded [KIV]
}
```

Response:

```json
201 Created
{
    pollId: UUID
}
``` 

---

- GET /api/v1/polls/:pollId

Response:
- 404 Not Found
- 200 OK
```json
{
    pollId: UUID,
    question: string,
    description: string?,
    status: "active" | "closed"
}
```

---

- PATCH /api/v1/polls/:pollId/close

Response:
- 200 OK
- 404 Not Found -> unable to find poll
- 409 Conflict -> poll is already closed

---

- POST /api/v1/polls/:pollId/answers

Payload:
```json
{
    answer: string
}
```

Response: 
- 201 Created
- 403 Forbidden -> poll closed
- 404 Not found -> unable to find poll

---

- GET /api/v1/polls/:pollId/answers?limit={}&offset={}

Response: 
```json
200 OK
{
    answers: Array of {
        answer: string
    }
}
```

# High Level Design
[Link to design](https://drive.google.com/file/d/17EK7KgMs6o7cbQXLJoXQKqEQSVwmnoLg/view?usp=sharing)

## Database tables
Poll
- id: UUID PK
- question: string NON NULL
- description: string
- status: ENUM('active', 'closed') NON NULL DEFAULT 'active'
- createdAt: Date NON NULL
- updatedAt: Date

> Polls are created with a default status of `'active'`. A poll can be closed by calling `PATCH /polls/:pollId/close`, which sets `status` to `'closed'`. The application layer checks `status` on every vote submission and rejects votes to closed polls (`403 Forbidden`).

Answer
- id: BIGINT PK
- pollId: UUID FK Poll
- answer: string NON NULL
- answeredAt: Date NON NULL
- Index on pollId

Note: perhaps if user entity is introduced, we can use userId + pollId as surrogate ID for Answer table.

# Deep Dives

## Security and Rate Limiting
- Rate limit on APIGW per IP address / client connection to prevent abuse
- Input validation and sanitization on all user-supplied fields (question, description, answer) to prevent XSS
- Use parameterized queries / ORM to prevent SQL injection
- CSRF protection on state-changing endpoints

# Ignore (kept for posterity)

## High Availability
- Increase redundancy, have more instances of poll service. If 1 instance goes down, still have multiple instances left to handle traffic
- Have the APIGW do load balancing with health checks — route traffic away from unhealthy instances
- Database high availability: primary-replica setup with automatic failover

## Real-Time Results
- Use server-sent events (SSE)
    - SSE establishes a unidirectional channel from server to client, allowing the server to push updates to the client whenever a new answer related to the poll comes in
- Use pub/sub pattern (Redis Pub/Sub) to handle emitting new events across replicas
    - When poll service instance A receives a new answer for poll1, it publishes the event to Pub/Sub channel (e.g. `poll:<pollId>`)
    - All poll service instances subscribe to channels for the polls their connected clients are watching
    - Subscribing instances push updates to their connected SSE clients
- Connection management
    - Each SSE connection is a long-lived HTTP connection. At 300K concurrent users, that requires maintaining 300K open connections.
    - A single Node.js instance can comfortably handle 50K to 100K concurrent open connections
    - Math: `300,000 connections / 50,000 per instance = 6 instances` (or `300,000 / 100,000 = 3 instances`). Thus, 3 to 6 Node.js instances are required.

## Scaling to Handle High Concurrent Users
- Use sequential `BIGINT` (BIGSERIAL) PK for `Answer` table instead of random UUIDs to avoid B-Tree index fragmentation, cut key storage overhead in half (8 bytes vs 16 bytes), and maximize write throughput under high concurrency
- Query optimization on DB: no `select *`, return only needed columns
- Add appropriate index on frequently accessed columns like `pollId` in Answer table
- Add read replicas to DB. Polling system has a high read-to-write ratio, est 100:1
    - All writes go to 1 primary DB instance, reads go to read replicas
- In event single poll goes viral (eg 10K votes/sec), all writes hit the primary for the same `pollId`
    - Consider write batching: buffer votes in Redis, flush to DB periodically [KIV not shown in drawio]

## Failure Scenarios
- Redis Pub/Sub goes down: SSE clients stop receiving real-time updates. Clients can fall back to polling `GET /answers` on an interval, till pubsub is back up.
- Service instance crashes with active SSE connections: Clients auto-reconnect (SSE built-in behaviour).

## Capacity Estimates
- 10K concurrent polls × 30 users = 300K concurrent users
- Write load: default expected usage is ~1 vote per user. Bursty peak estimate — 300K votes over 10 minutes = 500 writes/sec -> manageable by a single write PostgreSQL primary
- Read load: 300K users viewing live results. With SSE, reads are push-based (no polling). Initial page load triggers `GET /polls/:pollId` + `GET /answers` → 600K requests at poll start, then SSE handles the rest
- SSE connections: 300K long-lived connections across 3 to 6 instances (`300,000 / 50,000 = 6` to `300,000 / 100,000 = 3`)
- Storage: 300K answers × 200 bytes each = 60MB per poll cycle -> minimal