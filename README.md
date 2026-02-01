# Leader Board Architecture

[![Leader board architecture](./leaderboard-architecture.png)](./leaderboard-architecture.png)

## Overview ✅

This module specifies the architecture and execution flow for a Leader Board service: how scores are ingested, aggregated, and served to clients.

## Diagram 📊

Click or view the diagram below for an illustrated flow of execution:

[![Leader board architecture](./leaderboard-architecture.png)](./leaderboard-architecture.png)

> Diagram file: `leaderboard-architecture.png` (located at repository root)

## Flow of Execution (high-level) 🔁

1. Client submits a score → `POST /scores` (or websocket)
2. API Gateway validates & forwards to Score Service
3. Score Service writes to datastore and publishes event to message bus
4. Worker consumes events, updates aggregated leaderboards (cache + DB)
5. Clients request leaderboards → `GET /leaderboard` (served from cache for low latency)

## Components & Responsibilities 🔧

- **API Gateway**: Authentication, request validation, rate-limiting
- **Score Service**: Ingest scores, basic validation, event publishing
- **Workers**: Aggregate scores, update leaderboards, handle TTLs
- **Datastore**: Persistent storage for raw scores & snapshots (e.g., PostgreSQL)
- **Cache**: Fast reads for leaderboard queries (e.g., Redis sorted sets)
- **Message Bus**: Event-driven updates (e.g., Kafka/RabbitMQ)

## API Spec (for backend team) 🧾

- `POST /scores`
  - Body: `{ "user_id": "string", "score": number, "timestamp": "ISO8601" }`
  - Response: `201 Created`
- `GET /leaderboard?period=daily|weekly|all&limit=50`
  - Response: `200 OK` with ordered list of `{ user_id, rank, score }`

## Data Model (suggested)

- `Users(user_id, name, metadata)`
- `Scores(id, user_id, score, timestamp)`
- `Leaderboards(period, user_id, score, rank)`

## Non-functional Requirements ⚙️

- **Latency**: < 100ms for leaderboard reads when cached
- **Throughput**: Handle bursts of score submissions (scale workers horizontally)
- **Durability**: Persist raw events before acknowledging requests
- **Consistency**: Eventual consistency for aggregated leaderboards is acceptable

## Improvements & Notes (actionable) ✨

- Add automated tests around aggregation logic and ranking correctness.
- Implement retries and dead-letter queue for failed events.
- Add metrics: submission rate, processing lag, cache hit ratio, top-k stability.
- Consider sharding leaderboards by region/time-window for scalability.

## Implementation Checklist ✅

1. Create documentation for this module (`README.md`) — **done**
2. Create and include a diagram to illustrate flow — **image linked above**
3. Add additional comments and improvements — **see Improvements & Notes**
4. Provide clear spec for backend engineering team — **see API Spec & Data Model**