# TICKET-007: Deterministic Scoring Engine

## Phase

**Phase 2 — Retrieval and Ranking Pipeline**  
Ref: `implementation-plan.md §7 Phase 2` — "Return top-4 teachers with deterministic score."

## Assignment Reference

- **assigment.md — Context (Phase 1):** "Automatically find the best-match teacher, plus 3 alternatives" — the scoring engine produces the ranked list.
- **assigment.md — Deliverables — `output.md`:** "Pipeline trace — each step taken (filtering, scoring, LLM calls) with inputs and outputs shown." The scoring engine must produce traceable inputs and outputs.
- **implementation-plan.md §3 — External API Constraints — Fallback mode:** "Deterministic ranking remains available" when LLM is unavailable. This engine is the fallback path.

## Design Document References

- [ai-pipeline.md — §2 End-to-End Pipeline Flow](../ai-pipeline.md): `Deterministic Score Engine` sits between `Hybrid Retriever` and `LLM Reranker`.
- [technical-proposal.md — §5 Retrieval, Ranking, and Explanation Pipeline](../technical-proposal.md): Step 4 — "Calculate deterministic score: skill-gap coverage, teaching-style fit, experience suitability, policy and business constraints."
- [ai-pipeline.md — §5.1 Recommendation Runtime Contract](../ai-pipeline.md): Step 3 — "Score and rerank modules produce top candidates (rank 1-4)."

## Description

Implement a deterministic scoring engine that computes a composite match score for each teacher candidate against a student profile. This engine uses structured data (not LLM calls) to produce reproducible, auditable scores.

The deterministic score serves two purposes:
1. **Primary ranking signal:** Combined with LLM reranking (TICKET-008), it determines the final top-4.
2. **Fallback ranking:** When the LLM is unavailable, the deterministic score alone determines the recommendation.

## Acceptance Criteria

- [ ] `score_candidates(student, candidates[])` returns candidates with a `deterministic_score` (0-1) and a `score_breakdown` object.
- [ ] Score breakdown includes individual dimension scores: `skill_gap_coverage`, `teaching_style_fit`, `experience_suitability`, `communication_score`, `satisfaction_score`.
- [ ] **Skill-gap coverage** (weight: 0.35): Measures how well the teacher's scored skills cover the student's weak areas. A teacher with high scores in a student's weak subjects scores higher.
- [ ] **Teaching-style fit** (weight: 0.20): Exact match between student `preferred_learning_style` and teacher `teaching_style` scores 1.0; mismatch scores 0.3.
- [ ] **Experience suitability** (weight: 0.15): Teachers whose `preferred_student_level` includes the student's `current_level` score 1.0; otherwise 0.4. Bonus for higher `experience_years` (normalized 0-1 over pool range).
- [ ] **Communication score** (weight: 0.15): Teacher's `communication` score normalized to 0-1.
- [ ] **Satisfaction score** (weight: 0.15): Teacher's `student_satisfaction` normalized to 0-1 (raw value / 5.0).
- [ ] Weights are configurable via environment or config file, not hardcoded.
- [ ] Score is deterministic: same inputs always produce the same output.
- [ ] Scoring produces a `pipeline_trace_steps` entry with `step_name='deterministic_scoring'`, recording all inputs and the full score breakdown.
- [ ] Candidates are sorted by `deterministic_score` descending.
- [ ] Scoring handles the edge case where a teacher has no overlapping subjects with the student (skill_gap_coverage = 0).

## Technical Details

### Scoring Formula

```
deterministic_score = 
    w_skill * skill_gap_coverage
  + w_style * teaching_style_fit
  + w_experience * experience_suitability
  + w_communication * communication_normalized
  + w_satisfaction * satisfaction_normalized
```

Default weights: `w_skill=0.35, w_style=0.20, w_experience=0.15, w_communication=0.15, w_satisfaction=0.15`

### Skill-Gap Coverage Calculation

```python
def compute_skill_gap_coverage(student: Student, teacher: Teacher) -> float:
    student_subjects = {s.subject.code for s in student.goal_subjects}
    teacher_subjects = {ts.subject.code for ts in teacher.subjects}
    
    overlap = student_subjects & teacher_subjects
    if not student_subjects:
        return 0.0
    
    coverage_ratio = len(overlap) / len(student_subjects)
    
    # Bonus for high skill scores in overlapping areas
    relevant_scores = [
        s.score for s in teacher.skill_scores
        if s.skill_name in SUBJECT_TO_SKILL_MAP.get(overlap, [])
    ]
    avg_relevant_score = mean(relevant_scores) / 100.0 if relevant_scores else 0.5
    
    return coverage_ratio * 0.6 + avg_relevant_score * 0.4
```

### Score Breakdown Example (Student S002 vs Teacher T001)

```json
{
  "teacher_id": "T001",
  "deterministic_score": 0.91,
  "score_breakdown": {
    "skill_gap_coverage": 0.95,
    "teaching_style_fit": 1.0,
    "experience_suitability": 0.87,
    "communication_normalized": 0.85,
    "satisfaction_normalized": 0.96
  },
  "weights": {
    "skill_gap": 0.35,
    "style": 0.20,
    "experience": 0.15,
    "communication": 0.15,
    "satisfaction": 0.15
  }
}
```

**Rationale for T001 scoring high for S002:**
- S002 needs Math and Physics (beginner, structured).
- T001 teaches Math and Physics, structured style, 8 years experience, preferred levels include beginner.
- Skill scores: subject_knowledge=92, communication=85, patience=90, satisfaction=4.8.

### Trace Output

Each scoring run writes a `pipeline_trace_steps` row:

```json
{
  "step_name": "deterministic_scoring",
  "step_order": 3,
  "input_summary": {
    "student_id": "S002",
    "candidate_count": 8,
    "weight_config": { ... }
  },
  "output_summary": {
    "scored_count": 8,
    "top_score": 0.91,
    "bottom_score": 0.32
  },
  "duration_ms": 12
}
```

## Dependencies

- **TICKET-001** — Database schema (`teacher_skill_scores`, `pipeline_trace_steps`).
- **TICKET-005** — Subject taxonomy for subject-to-skill mapping.
- **TICKET-006** — Hybrid Retrieval provides the candidate set to score.

## Test Plan

### Unit Tests
- **Skill-gap coverage — full overlap:** T001 (Math, Physics) vs S002 (Math, Physics goals); verify `skill_gap_coverage` close to 0.95 (high coverage + high scores).
- **Skill-gap coverage — partial overlap:** T005 (English, Math) vs S002 (Math, Physics); verify `coverage_ratio = 0.5` (1/2 subjects matched).
- **Skill-gap coverage — no overlap:** T009 (English only) vs S002 (Math, Physics); verify `skill_gap_coverage = 0.0`.
- **Teaching-style fit — exact match:** S002 (structured) vs T001 (structured); verify `teaching_style_fit = 1.0`.
- **Teaching-style fit — mismatch:** S002 (structured) vs T002 (exploratory); verify `teaching_style_fit = 0.3`.
- **Experience suitability — level match:** T001 (preferred: beginner, intermediate) vs S002 (beginner); verify `experience_suitability` includes level bonus of 1.0.
- **Experience suitability — level mismatch:** T006 (preferred: advanced only) vs S002 (beginner); verify penalty applied (0.4).
- **Communication + satisfaction normalization:** T001 communication=85 -> 0.85; satisfaction=4.8 -> 0.96; verify normalized values.
- **Configurable weights:** Override weights via config; verify score changes accordingly.
- **Determinism:** Run scoring twice with identical inputs; verify identical outputs.
- **Edge case — no subjects:** Teacher with no overlapping subjects; verify `skill_gap_coverage = 0.0` and total score is not NaN.

### Integration Tests
- **Full scoring pipeline:** Pass S002's candidate set (from retrieval) into `score_candidates()`; verify T001 scores highest. Verify T001 > T007 > T005 in ranking for S002.
- **Score determinism across runs:** Score the same candidate set twice; verify all scores match exactly (bit-for-bit).
- **Trace output:** After scoring, query `pipeline_trace_steps` for `step_name='deterministic_scoring'`; verify the row exists with correct `input_summary` (student_id, candidate_count) and `output_summary` (scored_count, top_score).

### E2E / Manual Tests
- **Manual score verification for T001 vs S002:** Compute expected score by hand using the formula with default weights: `0.35 * skill_gap + 0.20 * style_fit + 0.15 * experience + 0.15 * communication + 0.15 * satisfaction`. Compare against engine output. Document the calculation.
- **Full ranking of all teachers for S002:** Score all 10 teachers; verify the ordering is sensible (Math/Physics structured teachers rank highest for S002).

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: Returns deterministic_score (0-1) + breakdown | Unit | All dimension score tests |
| AC: Skill-gap coverage weight 0.35 | Unit | Skill-gap coverage tests (full/partial/none) |
| AC: Teaching-style fit weight 0.20 | Unit | Style fit tests (match/mismatch) |
| AC: Experience suitability weight 0.15 | Unit | Experience suitability tests |
| AC: Communication weight 0.15 | Unit | Communication normalization |
| AC: Satisfaction weight 0.15 | Unit | Satisfaction normalization |
| AC: Configurable weights | Unit | Configurable weights test |
| AC: Deterministic (same input = same output) | Unit + Integration | Determinism tests |
| AC: Trace entry written to pipeline_trace_steps | Integration | Trace output verification |
| AC: Candidates sorted by score descending | Integration | Full scoring pipeline — ranking check |
| AC: Handles no-overlap edge case | Unit | Edge case — no subjects |

## Dataset References

- Teacher scores from `dataset/teachers.json` are the primary inputs to the scoring formula. Example: T001 has `subject_knowledge: 92, communication: 85, problem_solving: 88, patience: 90, student_satisfaction: 4.8`.
- Student profiles from `dataset/new_students.json` provide the weak areas and preferences that drive skill-gap and style-fit calculations.
- Expected top matches for S002 (Math/Physics, beginner, structured): T001 > T007 > T005 > T010 (all structured, math-covering teachers with high scores).
