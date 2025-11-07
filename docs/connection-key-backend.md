# Connection Key Backend Architecture

## Overview
The Connection Key feature is powered by a modular backend that separates quiz generation, scoring, and messaging unlock orchestration. Services are deployed in a Kubernetes cluster and communicate through asynchronous events while exposing a consolidated GraphQL API to the mobile apps.

## Service Topology
- **Gateway/API**: NestJS GraphQL server with REST fallbacks. Handles authentication via Supabase JWTs, enforces rate limits with Redis, and orchestrates downstream calls.
- **Quiz Generator**: Python FastAPI service responsible for assembling question sets. Pulls from profile content, consent preferences, and a prompt template library stored in PostgreSQL.
- **Scoring Engine**: Kotlin Spring Boot microservice evaluating responses. Uses rule-based checks for consent items, similarity scoring for comprehension questions, and calls an LLM rubric endpoint for free responses.
- **Messaging Orchestrator**: Node.js worker subscribing to quiz completion events. Updates conversation unlock state, logs analytics, and publishes notifications to the WebSocket gateway.
- **Notification Gateway**: Serverless function (AWS Lambda) pushing push notifications via Firebase/APNs based on orchestrator events.

## Data Model Highlights
- `profiles`: user bio, prompts, consent preferences (PostgreSQL).
- `connection_keys`: ephemeral quiz payloads with expiry metadata.
- `connection_key_questions`: normalized questions with variants and difficulty metadata.
- `connection_key_attempts`: response payload, timing metrics, score breakdown, unlock status.
- `connection_key_events`: event stream stored in Kafka topic for analytics replay.

## API Surface
### GraphQL
- `mutation createConnectionKey(targetUserId: ID!): ConnectionKeyPayload` — Triggers quiz generation and returns signed payload + expiry.
- `mutation submitConnectionKey(input: ConnectionKeyResponseInput!): ConnectionKeyResult` — Validates responses, returns score, unlock state, feedback messages.
- `subscription connectionKeyStatus(conversationId: ID!): ConnectionKeyStatus` — Streams unlock updates to both participants.

### REST Fallbacks
- `POST /connection-keys` — Mirror of `createConnectionKey` for backend-to-backend integrations.
- `POST /connection-keys/{id}/responses` — Mirror of submission mutation, used by moderation tools.

## Workflow
1. **Create Quiz**
   1. Mobile client calls `createConnectionKey`.
   2. Gateway authenticates user, checks rate limits, and sends a job to the Quiz Generator via Redis queue.
   3. Quiz Generator builds questions, signs payload with HMAC, stores metadata in `connection_keys`, and returns payload to the gateway.
   4. Gateway caches payload in Redis and returns it to the client with expiry timestamp.
2. **Submit Quiz**
   1. Client calls `submitConnectionKey` with signed answers.
   2. Gateway validates signature and forwards answers to Scoring Engine.
   3. Scoring Engine computes score, updates `connection_key_attempts`, emits `QUIZ_COMPLETED` event to Kafka.
   4. Messaging Orchestrator consumes event, applies unlock rules, updates conversation state, and publishes `ConnectionKeyStatus` subscription payloads.
   5. Notification Gateway sends push if unlock achieved or retry needed.

## Security & Compliance
- All quiz payloads signed and expire within 5 minutes; refresh requires regeneration.
- PII stored encrypted-at-rest, with per-field encryption for consent preferences.
- Audit logs written to Append-only S3 bucket; immutable for 1 year minimum.
- Moderator tools require elevated scopes validated via JWT claims.

## Observability
- OpenTelemetry traces instrument Gateway, Generator, and Scoring Engine with trace context propagated through Kafka.
- Grafana dashboards display quiz success rate, latency percentiles, retry counts, and fairness metrics segmented by demographics.
- PagerDuty alerts on SLA breach: >1% quiz generation failures or >3s p95 submission latency.

## Scaling Considerations
- Quiz Generator can horizontally scale behind a queue; heavy LLM scoring is batched and backed by autoscaling GPU nodes.
- Scoring Engine caches recent profile embeddings to avoid recomputation.
- Messaging Orchestrator runs as stateless workers; idempotent event handling ensures retries are safe.

## Roadmap Extensions
- Add A/B testing hooks to Quiz Generator for template experimentation.
- Introduce progressive profiling to personalize difficulty.
- Integrate fairness monitors to flag disparate pass rates in near-real time.
