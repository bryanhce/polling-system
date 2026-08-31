## 1. Backend Event Broadcaster & Lookup Table

- [ ] 1.1 Define `PollEventBroadcaster` interface and `SSEClient` types in `backend/src/poll/poll-event-broadcaster.ts`
- [ ] 1.2 Implement `InMemoryPollEventBroadcaster` with lookup table (`Map<string, Set<SSEClient>>`), periodic `: ping\n\n` heartbeat, broadcast methods for `answer` and `poll_closed`, and clean unregistering on socket close
- [ ] 1.3 Create unit tests for `InMemoryPollEventBroadcaster` in `backend/src/poll/poll-event-broadcaster.test.ts`

## 2. Backend SSE Route & Service Integration

- [ ] 2.1 Update `AnswerService` to invoke broadcaster after persisting new answers
- [ ] 2.2 Update `PollService` to invoke broadcaster on poll closure
- [ ] 2.3 Implement `GET /api/v1/polls/:pollId/events` endpoint in `backend/src/poll/poll.controller.ts` with validation (404 if not found, 403 if closed) and proper SSE response headers
- [ ] 2.4 Register broadcaster in `backend/src/app.ts` with graceful cleanup in Fastify `onClose` hook
- [ ] 2.5 Add integration tests in `backend/src/app.test.ts` verifying SSE endpoint connection, answer broadcasting, poll closure notification, and 404/403 validation

## 3. Frontend API & Real-Time Hook Integration

- [ ] 3.1 Add SSE connection utility / endpoint URL helper in `frontend/src/api/polls.ts`
- [ ] 3.2 Update `useActivePollPage` hook to subscribe to SSE for active polls, prepend incoming answers in real-time, and react to `poll_closed` events
- [ ] 3.3 Update `frontend/src/pages/ActivePoll/useActivePollPage.test.tsx` with test cases for SSE connection, live answer updates, and poll close handling

## 4. Verification & Testing

- [ ] 4.1 Run full backend test suite (`npm test`) and ensure all tests pass
- [ ] 4.2 Run full frontend test suite (`npm test`) and ensure all tests pass
- [ ] 4.3 Run TypeScript typechecking across backend and frontend
