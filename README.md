# Jira Tickets — Worker Execution POC

## Epic

**Title:** Worker Execution POC — Event-Driven Async Agent Execution via Redis Streams

**Description:**
Implement a proof-of-concept demonstrating event-driven worker execution architecture for async agent execution. The system uses FastAPI as the API layer, Redis Streams as the message broker, and LangChain ChatOpenAI for LLM invocation. A client submits a query via REST API, the message is queued in Redis Streams, and a background worker consumes and processes it asynchronously.

**Labels:** `poc`, `backend`, `redis`, `async`, `llm`
**Priority:** Medium
**Components:** `api-graph`, `worker-execution`

---

## Story 1: Project Setup & Configuration

**Title:** [POC] Set up project structure, Docker Compose, and configuration management

**Story:**
As a developer, I want a containerized project setup with Docker Compose so that I can run the POC locally with Redis and the API server.

**Acceptance Criteria:**
- [ ] Project structure created with FastAPI application
- [ ] `Dockerfile` with Python 3.11-slim base image
- [ ] `docker-compose.yml` with Redis and API services
- [ ] `requirements.txt` with all dependencies (fastapi, uvicorn, redis, langchain-openai, pydantic-settings)
- [ ] Configuration management via pydantic-settings loading from `.env`
- [ ] Environment variables documented: `OPENAI_*`, `REDIS_*`, `LOCAL_SERVER`
- [ ] `docker compose up --build` starts both services successfully
- [ ] Health endpoint `GET /health` returns `{"status": "healthy"}`

**Story Points:** 3
**Priority:** High
**Labels:** `poc`, `setup`, `docker`

---

## Story 2: Redis Streams Integration

**Title:** [POC] Implement Redis Streams client for message publishing and consuming

**Story:**
As a developer, I want a Redis Streams wrapper that supports publishing messages (XADD), consuming messages (XREADGROUP), and acknowledging messages (XACK) so that the async message pipeline works reliably.

**Acceptance Criteria:**
- [ ] `RedisStreamClient` class with async Redis connection
- [ ] `connect()` method creates consumer group with MKSTREAM option
- [ ] `publish_message()` uses XADD to add messages to the stream
- [ ] `start_processing()` uses XREADGROUP with block timeout for consuming
- [ ] `acknowledge_message()` uses XACK to confirm processing
- [ ] `stop_processing()` and `close()` for graceful shutdown
- [ ] Consumer group creation handles "already exists" case gracefully
- [ ] Messages include: `correlation_id`, `agent_id`, `query`, `conversation_id`

**Story Points:** 5
**Priority:** High
**Labels:** `poc`, `redis`, `streams`

---

## Story 3: API Endpoint for Agent Query Submission

**Title:** [POC] Implement POST /agents/{agent_id}/query endpoint for async query submission

**Story:**
As a client, I want to submit a query to an agent via REST API and receive an immediate "queued" response with a `correlation_id` so that I know my request has been accepted for processing.

**Acceptance Criteria:**
- [ ] `POST /agents/{agent_id}/query` endpoint implemented
- [ ] Request body accepts `{"query": "string", "conversation_id": "optional string"}`
- [ ] Response returns `{"status": "queued", "correlation_id": "uuid", "message": "string"}`
- [ ] Message is published to Redis Streams with all required fields
- [ ] Unique `correlation_id` (UUID) generated for each request
- [ ] Endpoint returns immediately without waiting for processing

**Story Points:** 3
**Priority:** High
**Labels:** `poc`, `api`, `fastapi`

---

## Story 4: Worker Implementation with LLM Invocation

**Title:** [POC] Implement background worker that consumes messages and invokes LLM

**Story:**
As a system, I want a background worker that consumes messages from Redis Streams and invokes the LLM (ChatOpenAI) so that queries are processed asynchronously.

**Acceptance Criteria:**
- [ ] `WorkerApplication` class with setup/run/shutdown lifecycle
- [ ] Worker spawned in a daemon thread during FastAPI lifespan startup
- [ ] Worker consumes messages from Redis Streams via XREADGROUP
- [ ] `ExecuteAgentHandler` invokes `ChatOpenAI.ainvoke()` with the query
- [ ] Messages are acknowledged (XACK) after successful processing
- [ ] Worker logs: message received, LLM response (truncated), acknowledgment
- [ ] Worker handles errors gracefully without crashing
- [ ] Worker stops cleanly during application shutdown

**Story Points:** 5
**Priority:** High
**Labels:** `poc`, `worker`, `llm`, `langchain`

---

## Story 5: End-to-End Testing & Verification

**Title:** [POC] Verify end-to-end pipeline: API → Redis → Worker → LLM → ACK

**Story:**
As a developer, I want to verify the complete message pipeline works end-to-end so that I can confirm the POC architecture is viable.

**Acceptance Criteria:**
- [ ] `docker compose up --build` starts all services without errors
- [ ] Health endpoint returns 200 OK
- [ ] Query endpoint returns queued response with `correlation_id`
- [ ] Worker consumes the message from Redis Streams
- [ ] LLM is invoked and returns a valid response
- [ ] Message is acknowledged (0 pending messages in consumer group)
- [ ] Redis Streams state verified via `XLEN` and `XINFO GROUPS`
- [ ] All logs captured and documented

**Story Points:** 3
**Priority:** High
**Labels:** `poc`, `testing`, `verification`

**Test Results (Completed):**
- Docker Compose build: ✅ Success
- Health endpoint: ✅ `{"status": "healthy"}`
- Query endpoint: ✅ Returns `{"status": "queued", "correlation_id": "af648ad9-...", "message": "Query queued for agent test-agent"}`
- Worker processing: ✅ Message consumed, LLM invoked successfully
- LLM Response: ✅ "The capital of France is Paris."
- Message ACK: ✅ 0 pending messages, lag: 0

---

## Story 6: Documentation

**Title:** [POC] Create technical documentation for Confluence

**Story:**
As a team member, I want comprehensive technical documentation so that I can understand the POC architecture, how to run it, and what the results were.

**Acceptance Criteria:**
- [ ] Confluence page created with architecture diagrams (Mermaid)
- [ ] Technology stack documented
- [ ] API endpoints documented with request/response schemas
- [ ] Environment variables documented with defaults
- [ ] Step-by-step "How to Run" guide
- [ ] Test results documented
- [ ] Known limitations and future work listed
- [ ] Key source code highlights included

**Story Points:** 3
**Priority:** Medium
**Labels:** `poc`, `documentation`

---

## Summary Table

| Ticket | Title | Points | Priority | Status |
|--------|-------|--------|----------|--------|
| Epic | Worker Execution POC | — | Medium | Done |
| Story 1 | Project Setup & Configuration | 3 | High | Done |
| Story 2 | Redis Streams Integration | 5 | High | Done |
| Story 3 | API Endpoint for Query Submission | 3 | High | Done |
| Story 4 | Worker with LLM Invocation | 5 | High | Done |
| Story 5 | E2E Testing & Verification | 3 | High | Done |
| Story 6 | Documentation | 3 | Medium | Done |
| **Total** | | **22** | | |
