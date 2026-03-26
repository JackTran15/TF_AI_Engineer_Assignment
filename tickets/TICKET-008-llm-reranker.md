# TICKET-008: LLM Reranker

## Phase

**Phase 2 — Retrieval and Ranking Pipeline**  
Ref: `implementation-plan.md §7 Phase 2` — "Implement recommendation trigger worker and RAG retrieval/ranking."

## Assignment Reference

- **assigment.md — Context (Phase 1):** "The pipeline should utilize LLM where it adds value." LLM reranking adds value by incorporating nuanced understanding of teacher-student fit beyond what deterministic scoring captures.
- **assigment.md — Deliverables — `output.md`:** "Pipeline trace — each step taken (filtering, scoring, LLM calls) with inputs and outputs shown." The reranker is a traced LLM call.
- **implementation-plan.md §3 — External API Constraints:** Rate limiter and token budget guard around LLM calls. Fallback to deterministic ranking when LLM is unavailable.

## Design Document References

- [ai-pipeline.md — §2 End-to-End Pipeline Flow](../ai-pipeline.md): `LLM Reranker` sits between `Deterministic Score Engine` and `Select top 4`.
- [technical-proposal.md — §5 Step 5](../technical-proposal.md): "Run reranker for top candidate window."
- [implementation-plan.md §3 — Fallback mode](../implementation-plan.md): "Deterministic ranking remains available; explanation uses constrained template when LLM is unavailable."
- [technical-proposal.md — §9 Concurrency and Backpressure](../technical-proposal.md): "Token budget and QPS throttling per LLM provider key."

## Description

Implement an LLM-based reranking step that takes the top N candidates from the deterministic scoring engine and applies a model-based relevance judgment to refine the final top-4 selection.

The reranker uses the LLM to evaluate nuanced fit factors that deterministic scoring cannot capture — such as pedagogical alignment described in bios, specific methodology matches, and teaching philosophy compatibility.

## Acceptance Criteria

- [ ] `rerank_candidates(student, scored_candidates[], config)` returns the top-4 teachers reranked by the LLM.
- [ ] Input to the reranker is the top N candidates (default N=10) from the deterministic scoring engine.
- [ ] The LLM receives a structured prompt containing the student profile and each candidate's profile summary, scores, and evidence chunks.
- [ ] The LLM returns a ranked list with a `rerank_rationale` for each position.
- [ ] Final score is a weighted blend: `final_score = alpha * deterministic_score + (1 - alpha) * llm_rerank_score` where alpha is configurable (default: 0.6).
- [ ] Top-4 is selected from the blended scores: rank 1 (best match) + ranks 2-4 (alternatives).
- [ ] **Fallback behavior:** If the LLM call fails (timeout, rate limit, error), the system falls back to deterministic-only ranking and logs a warning. The recommendation is still produced.
- [ ] **Rate limiting:** LLM reranker calls are throttled to stay within provider limits (configurable RPM/TPM).
- [ ] **Token budget:** The reranker prompt is capped at a configurable token limit (default: 4000 tokens). Candidate profiles are truncated if necessary.
- [ ] **Cost-control model tiering:** Reranker supports a cheaper model tier (for example, `gpt-4o-mini`) via config and records selected model in trace output.
- [ ] A `pipeline_trace_steps` entry records `step_name='llm_reranking'` with the prompt, LLM response, model used, latency, and token usage.
- [ ] Reranking latency P95 is under 3 seconds.

## Technical Details

### Reranker Prompt Design

```
You are a teacher-student matching expert. Given a student profile and a list
of candidate teachers, rerank the teachers by overall fit quality.

STUDENT PROFILE:
- Name: {name}
- Level: {current_level}
- Learning style: {preferred_learning_style}
- Goals: {learning_goals}
- Weak areas: {weak_areas}

CANDIDATE TEACHERS (pre-ranked by initial score):
{for each candidate:}
  [{rank}] Teacher: {name} (ID: {teacher_id})
  - Subjects: {subjects}
  - Style: {teaching_style}
  - Experience: {experience_years} years
  - Bio: {bio}
  - Skill scores: {scores summary}
  - Initial score: {deterministic_score}
{end for}

Rerank these teachers from best to worst fit for this student.
Return a JSON array with the format:
[
  { "teacher_id": "...", "rank": 1, "rerank_score": 0.0-1.0, "rationale": "..." },
  ...
]

Consider: subject expertise, teaching style alignment, experience with
the student's level, and how the teacher's strengths address the student's
specific weak areas.
```

### LLM Call Configuration

```python
rerank_response = await llm_client.chat(
    model=config.llm_model,  # e.g., "gpt-4o"
    messages=[{"role": "system", "content": RERANKER_SYSTEM_PROMPT},
              {"role": "user", "content": reranker_user_prompt}],
    response_format={"type": "json_object"},
    temperature=0.1,  # Low temperature for consistency
    max_tokens=1500,
    timeout=5.0
)
```

### Blended Score Calculation

```python
def compute_final_score(det_score: float, llm_score: float, alpha: float = 0.6) -> float:
    return alpha * det_score + (1 - alpha) * llm_score
```

### Fallback Path

```python
async def rerank_with_fallback(student, candidates, config):
    try:
        reranked = await llm_rerank(student, candidates, config)
        return blend_scores(candidates, reranked, config.alpha)
    except (LLMTimeoutError, RateLimitError, LLMError) as e:
        logger.warning(f"LLM reranker failed: {e}. Falling back to deterministic ranking.")
        trace.record_fallback(step="llm_reranking", reason=str(e))
        return candidates[:4]  # Top 4 by deterministic score
```

### Circuit Breaker

- Opens after 3 consecutive LLM failures within 60 seconds.
- Half-open after 30 seconds — allows one probe request.
- Fully open = all requests use deterministic fallback.

## Dependencies

- **TICKET-007** — Deterministic Scoring Engine provides the pre-ranked candidate list.
- **TICKET-000** — `services/ai` scaffold for Python FastAPI service.
- **TICKET-001** — Database schema (`pipeline_trace_steps`).

## Test Plan

### Unit Tests
- **Prompt construction:** Build the reranker prompt for S002 with 5 candidates; verify prompt includes student goals, weak areas, level, style, and all candidate fields (name, subjects, style, bio, scores).
- **LLM response parsing:** Mock LLM returning a valid JSON array of reranked teachers; verify the parser extracts `teacher_id`, `rank`, `rerank_score`, and `rationale` for each entry.
- **LLM response — malformed JSON:** Mock LLM returning invalid JSON; verify fallback to deterministic ranking is triggered.
- **Blended score calculation:** Given `deterministic_score=0.90`, `llm_rerank_score=0.80`, `alpha=0.6`; verify `final_score = 0.6*0.90 + 0.4*0.80 = 0.86`.
- **Token budget enforcement:** Build a prompt that exceeds 4000 tokens; verify candidate profiles are truncated to fit within the limit.
- **Fallback on timeout:** Simulate LLM timeout (>5s); verify `rerank_with_fallback` returns top-4 by deterministic score and logs a warning.
- **Fallback on rate limit:** Simulate rate limit error; verify fallback path and warning log.
- **Model-tier configuration:** Set reranker model to `gpt-4o-mini`; verify request uses configured model and trace stores that model value.

### Integration Tests
- **Live/mocked reranking:** Send 10 candidates for S002 to the LLM reranker (with real or mocked LLM); verify a top-4 list is returned with blended scores and rationales.
- **Trace output:** After reranking, query `pipeline_trace_steps` for `step_name='llm_reranking'`; verify the row includes prompt, model used, latency_ms, token usage, and the ranked output.
- **Circuit breaker:** Simulate 3 consecutive LLM failures within 60 seconds; verify circuit opens and subsequent requests use deterministic fallback without attempting LLM call. Wait 30 seconds; verify half-open state allows one probe.
- **Partial LLM failure:** If LLM returns only 3 teachers instead of 4, verify the 4th is filled from the deterministic ranking.

### E2E / Manual Tests
- **Full retrieval + scoring + reranking for S002:** Run the complete pipeline from retrieval through reranking; verify 4 teachers are returned with final blended scores. Verify the top-1 teacher is reasonable for S002's profile (Math/Physics, beginner, structured).
- **Graceful degradation test:** Stop or block the LLM endpoint; submit a recommendation for S002; verify the system still returns 4 teachers using deterministic-only ranking within acceptable latency.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: rerank_candidates returns top-4 | Integration | Live/mocked reranking |
| AC: Input is top N from deterministic scoring | Unit | Prompt construction (verifies candidates included) |
| AC: LLM receives structured prompt | Unit | Prompt construction test |
| AC: LLM returns ranked list with rationale | Unit | LLM response parsing |
| AC: Blended score formula applied | Unit | Blended score calculation |
| AC: Fallback on LLM failure | Unit | Fallback on timeout + rate limit |
| AC: Rate limiting / token budget | Unit | Token budget enforcement |
| AC: Cost-control model tiering | Unit | Model-tier configuration test |
| AC: Trace entry written | Integration | Trace output verification |
| AC: P95 latency under 3s | E2E | Full pipeline timing check |
| AC: Circuit breaker | Integration | Circuit breaker test |

## Dataset References

- The reranker processes candidates scored against `dataset/teachers.json` profiles for the students in `dataset/new_students.json`.
- Example: For S002, the deterministic engine might rank T001 > T007 > T005 > T001 > T010. The LLM reranker can adjust if, for example, T007's bio indicates a methodology that better addresses Algebra weakness specifically.
