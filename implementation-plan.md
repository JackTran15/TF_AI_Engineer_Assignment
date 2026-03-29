# Implementation Plan — AI Coaching Recommendation System

This document is the source of truth for operational policy details (model routing, fallback behavior, and LLM cost controls). Other deliverables should reference this file to avoid duplicated policy text.

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

### Capacity Estimates

Per-recommendation latency breakdown (P50):

| Step | Latency | LLM Calls |
|---|---|---|
| Metadata filter + vector retrieval | ~50 ms | 0 |
| Heuristic pre-ranking (10 candidates) | ~15 ms | 0 |
| LLM reranking (top-7 candidates) | ~800 ms | 1 |
| Explanation generation (4 teachers) | ~1,200 ms | 4 (parallel) |
| Citation validation | ~300 ms | 1 |
| **Total per student** | **~2,400 ms** | **6** |

Worker throughput (single `recommendationBatchWorker` instance):

| Scenario | Batch Size | Parallelism | Throughput | Queue Drain Time |
|---|---|---|---|---|
| Baseline (1 worker) | 5 | 1 batch at a time | ~25 students/min | 100 students in ~4 min |
| Scaled (3 workers) | 10 | 3 batches concurrent | ~75 students/min | 1,000 students in ~14 min |
| Peak (10 workers) | 10 | 10 batches concurrent | ~250 students/min | 1,000 students in ~4 min |

Scaling triggers:
- Scale up when queue depth exceeds `50` messages or P95 processing latency exceeds `5s`.
- Scale down when queue is empty for `5` consecutive minutes.
- Maximum worker count capped at `10` to stay within LLM provider rate limits.

## 3) External API Constraints Plan

### Typical Constraints
- Requests-per-minute and tokens-per-minute limits from LLM providers
- Variable latency and occasional provider downtime
- Burst limits on embedding endpoints

### Mitigation
- Per-provider rate limiter and token budget guard.
- Circuit breaker and timeout policy around LLM and web search calls.
- Fallback mode:
  - heuristic ordering remains available
  - explanation uses constrained template when LLM is unavailable
- Optional secondary model provider for high availability.
- Task-based model routing:
  - cheap model tier for simple language tasks (cleanup, formatting, low-risk rewriting)
  - balanced model tier for standard reranking and explanation generation
  - high-performance model tier only for high-ambiguity, high-impact adjudication (for example citation disputes or contradictory HITL constraints)

### LLM Cost Analysis

Token estimates per recommendation (GPT-4o-class model):

| LLM Call | Input Tokens | Output Tokens | Count per Student |
|---|---|---|---|
| Reranking (7 candidates) | ~2,000 | ~500 | 1 |
| Explanation generation (per teacher) | ~800 | ~300 | 4 |
| Citation validation | ~1,500 | ~400 | 1 |
| **Total per student** | **~6,700** | **~2,100** | **6 calls** |

Cost projections (at $2.50 / 1M input tokens, $10.00 / 1M output tokens):

| Batch Size | Input Cost | Output Cost | Total Cost | Cost per Student |
|---|---|---|---|---|
| 100 students | $1.68 | $2.10 | **$3.78** | $0.038 |
| 1,000 students | $16.75 | $21.00 | **$37.75** | $0.038 |

Cost reduction levers:
- **Template fallback:** When the LLM circuit breaker opens, skip reranking and explanation LLM calls. Use heuristic ordering + templated explanations. Reduces cost to ~$0 per student (no LLM calls).
- **Smaller model for reranking:** Use a cheaper model (GPT-4o-mini at ~10x lower cost) for the reranking step, saving ~40% on total LLM cost.
- **Cached explanations:** If a student-teacher pair has been recommended before and teacher profile has not changed (`profile_version` unchanged), reuse the cached explanation.
- **Tiered escalation:** Invoke high-performance model tier only when uncertainty gates fail; default path remains cheap/balanced tiers.

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

### Ground-Truth Methodology

**Phase 1 — Bootstrapping (no labeled data):**
- Use retrieval plus a lightweight heuristic only to generate an initial candidate shortlist; do not treat heuristic ranking as ground truth.
- Have internal subject-matter experts (sales team, education leads) manually label 50-100 student-teacher pairs as `good_match`, `acceptable`, or `poor_match`.
- Compute `precision_at_4` and `ndcg_at_4` against expert labels to establish a baseline, then recalibrate heuristic weights from those labels if the heuristic is kept.

**Phase 2 — Online feedback collection:**
- Track implicit signals: recommendation acceptance rate (student clicks "Accept"), rematch request rate (student requests a different teacher within 7 days), and session completion rate.
- Track explicit signals: post-session student rating (1-5) after the first 3 sessions with the matched teacher.
- Label a recommendation as `positive` if the student does not request a rematch within 14 days and rates the teacher >= 4/5.

**Phase 3 — A/B testing framework:**
- Split incoming students into control (retrieval + heuristic-only fallback ordering) and treatment (retrieval + LLM reranking + citation-gated output) groups.
- Primary metric: recommendation acceptance rate lift.
- Secondary metrics: rematch rate reduction, HITL trigger rate reduction, time-to-first-valid-recommendation improvement.
- Minimum sample size: 200 students per arm for statistical significance at 95% confidence.
- Run for at least 4 weeks to capture weekly seasonality effects.

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
- Return top-4 teachers using retrieval-first ranking, with the heuristic kept only for shortlist/fallback support.

### Phase 3
- Add explanation generation and citation validation agents.
- Add confidence gates and HITL case creation.

### Phase 4
- Add observability dashboards, SLO alerts, and load tests.
- Tune batch size (`5-10`) and retry policies under stress.
