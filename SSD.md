# Requirements

## Functional
- User should be able to create a poll
- User should be able to answer a poll
    - User should be able to answer once per poll
- User should be able to view a poll result in real-time
- User should be able to close a poll

Extension: Poll can have multiple types - MCQ, open ended. For simplicity, now we do open ended string.

Note: this design is a high level one and for the core of the application. As such, I am choosing to ignore functional requirements like creating a user, authN, authZ, sharing/searching for poll etc

## Non Functional
- System should be highly available for answering polls, over strong consistency
- System should be real-time, low latency for live poll
- System should scale to support 10K concurrent polls
    - est: 30 users per poll so 300K concurrent users
- System should be secure and rate limited

# Core Entities
- User: represents user of a system, either the one who created the poll or is answering a poll
- Poll: represents the poll itself
- Answer: represent an answer to a poll, a poll has mutliple answers, answer is submitted by a user

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

- GET /api/v1/polls/:pollId

Response:
```json
200 OK
{
    pollId: UUID,
    question: string,
    description: string?,
    status: "active" | "closed"
}
```

- PATCH /api/v1/polls/:pollId/close

Note: userId taken from JWT. Only the user who created the poll is authorized to close it.

Response:
- 200 OK
- 403 Forbidden -> user is not the poll creator
- 404 Not Found -> unable to find poll
- 409 Conflict -> poll is already closed

- POST /api/v1/polls/:pollId/answers

Payload:
```JSON
{
    answer: string
}
```
Note: userId taken from JWT

Response: 
- 201 Created
- 403 Forbidden -> poll closed
- 404 Not found -> unable to find poll
- 409 Conflict -> user has already answered this poll

- GET /api/v1/polls/:pollId/answers?limit={}&offset={}

Response: 
```JSON
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
User
- id: UUID PK
- name: string NON NULL
- email: string NON NULL
- createdAt: Date NON NULL
- updatedAt: Date

Poll
- id: UUID PK
- question: string NON NULL
- description: string
- status: ENUM('active', 'closed') NON NULL DEFAULT 'active'
- createdBy: UUID FK User
- createdAt: Date NON NULL
- updatedAt: Date

> Polls are created with a default status of `'active'`. Only the user who created the poll (`createdBy`) can close it by calling `PATCH /polls/:pollId/close`, which sets `status` to `'closed'`. The application layer checks `status` on every vote submission and rejects votes to closed polls (`403 Forbidden`).

Answer
- id: UUID PK
- pollId: UUID FK Poll
- UserId: UUID FK User
- answer: string NON NULL
- answeredAt: Date NON NULL
- UNIQUE CONSTRAINT (userId, pollId)
- Index on pollId

# Deep Dives

## Security and Rate Limiting
- All users need to sign up and have an account before answering or creating a poll
- Auth with JWT — access and refresh tokens, userId extracted from JWT
- Rate limit on APIGW, 10 answers/sec per user
- Input validation and sanitization on all user-supplied fields (question, description, answer) to prevent XSS
- Use parameterized queries / ORM to prevent SQL injection
- CSRF protection on state-changing endpoints

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
    - Each SSE connection is a long-lived HTTP connection. At 300K concurrent users, that is 300K open connections
    - A single Node.js instance can handle 10K concurrent connections (conservative est) -> need 30 instances

## Scaling to Handle High Concurrent Users
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
- Write load: each user votes once per poll. Bursty peak estimate — 300K votes over 10 minutes = 500 writes/sec -> manageable by a single write PostgreSQL primary
- Read load: 300K users viewing live results. With SSE, reads are push-based (no polling). Initial page load triggers `GET /polls/:pollId` + `GET /answers` → 600K requests at poll start, then SSE handles the rest
- SSE connections: 300K long-lived connections across 30 instances
- Storage: 300K answers × 200 bytes each = 60MB per poll cycle -> minimal