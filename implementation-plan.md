# Implementation Plan — AI Coaching Recommendation System

## 1) Delivery Approach

Implement in iterative phases with production-safe defaults:
1. Data model and profile indexing pipeline
2. Retrieval and ranking pipeline
3. Multi-agent explanation and citation validation
4. HITL workflow and operational hardening

## 2) Concurrency Plan (100 to 1,000 Students)

### Strategy
- Keep API asynchronous and queue-first.
- Use two workers:
  - `profileBatchWorker` for profile indexing
  - `recommendationBatchWorker` for recommendation triggers
- Batch recommendation jobs in size `5-10`.

### Capacity Controls
- Autoscale workers based on queue depth and processing latency.
- Separate queues for indexing and recommendation to avoid starvation.
- Enforce idempotency keys to prevent duplicate runs.

### Failure Handling
- Retry transient failures with exponential backoff.
- Route hard failures to DLQ and support replay after correction.

## 3) External API Constraints Plan

### Typical Constraints
- Requests-per-minute and tokens-per-minute limits from LLM providers
- Variable latency and occasional provider downtime
- Burst limits on embedding endpoints

### Mitigation
- Per-provider rate limiter and token budget guard.
- Circuit breaker and timeout policy around LLM and web search calls.
- Fallback mode:
  - deterministic ranking remains available
  - explanation uses constrained template when LLM is unavailable
- Optional secondary model provider for high availability.

## 4) Data Freshness Plan

### Freshness Rules
- Increment `profile_version` on profile or sales note updates.
- Re-embed affected entity after each material change.
- Track `embedding_version` and write snapshot to recommendation request.

### Operational Targets
- Freshness SLO: `99%` of profile updates indexed within `5` minutes.
- Nightly cleanup removes stale vectors and orphan chunks.
- Freshness alerts trigger when indexing lag breaches threshold.

## 5) Evaluation Plan

### Offline Evaluation
- `precision_at_4`
- `ndcg_at_4`
- citation coverage rate
- unsupported claim rate

### Online Evaluation
- recommendation acceptance rate
- rematch request rate
- HITL trigger rate and HITL override rate
- time-to-first-valid-recommendation

### Quality Gates
- Do not publish result when critical citation checks fail.
- Route low-confidence recommendations to HITL automatically.

## 6) Technical Selection Plan

### Baseline Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| API & Workers | `Node.js` + `Fastify` + `Swagger OpenAPI` | Strong async IO performance for API and worker orchestration |
| AI Services | `Python` + `FastAPI` + `LangGraph` + `LangChain` | Superior ecosystem depth for RAG and multi-agent workflows |
| Metadata DB | `PostgreSQL` | Battle-tested relational store for structured data |
| Vector DB | `pgvector` (phase-1) → `Pinecone` / `Weaviate` (at scale) | Reduces early complexity; upgrade path when volume demands it |
| Queue | `SQS` + `DLQ` | Managed, durable messaging with built-in dead-letter support |

## 7) Timeline (Execution Sequence)

### Phase 1
- Finalize schema and ingestion contracts.
- Implement `profileBatchWorker` and profile indexing flow.

### Phase 2
- Implement recommendation trigger worker and RAG retrieval/ranking.
- Return top-4 teachers with deterministic score.

### Phase 3
- Add explanation generation and citation validation agents.
- Add confidence gates and HITL case creation.

### Phase 4
- Add observability dashboards, SLO alerts, and load tests.
- Tune batch size (`5-10`) and retry policies under stress.
