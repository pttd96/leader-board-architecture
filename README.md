# Leader Board Architecture


## Overview ✅

This module specifies the architecture and execution flow for a Leader Board with requirements as below:
1. We have a website with a score board, which shows the top 10 user’s scores.
2. We want live update of the score board.
3. User can do an action (which we do not need to care what the action is), completing this action will increase the user’s score.
4. Upon completion the action will dispatch an API call to the application server to update the score.
5. We want to prevent malicious users from increasing scores without authorisation.

## Diagram 📊

Click or view the diagram below for an illustrated flow of execution:

[![Leader board architecture](./leaderboard-architecture.png)](./leaderboard-architecture.png)

> Diagram file: `leaderboard-architecture.png` (located at repository root)

## Flow of Execution (high-level) 🔁

1. Frontend client (webapp) submits a user action → `POST /actions`
2. Load balancer routes the request to the most available instance of server, which in this case is Validator Service (VS).
3. VS validates the API request and user. 
  3.1. If validation fails, user is not authorized, VS returns error response.
  3.2. If validation succeeds, user is authorized, VS publishes a kafka message as the next step and finally returns success response.
4. Once message published to kafka, Score Service (SS) will be polling the messages to process the score. 
  4.1. Score is updated in Postgres DB.
  4.2. Score is updated in Redis Cache, which acts as the source for Leader Board.
5. Frontend client (webapp) will constantly be polling/fetching the Leader Board result from SS and showing them to the users.

## Components & Responsibilities 🔧

- **Load Balancer**: Balance request routing. Use Weighted round robin. We don't use Sticky LB here as we want to keep the server stateless. Additionally, providing and trusting session ids can introduce risks of users using tools like Postman to create unauthorized actions to increase score.
- **Validator Service**: Authentication, request validation, rate-limiting. 
- **Kafka**: Ingest scores, basic validation, event publishing
- **Score Service**: Aggregate scores, update leaderboards, handle TTLs
- **Postgres DB**: Persistent storage for raw scores & snapshots (e.g., PostgreSQL)
- **Redis Cache**: Fast reads for leaderboard queries (e.g., Redis sorted sets)

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