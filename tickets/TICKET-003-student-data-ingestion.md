# TICKET-003: Student Data Ingestion

## Phase

**Phase 1 — Data Model and Profile Indexing Pipeline**  
Ref: `implementation-plan.md §7 Phase 1` — "Finalize schema and ingestion contracts."

## Assignment Reference

- **assigment.md — Context (Phase 1):** "When new students register (see `new_students.json`), they provide their learning goals and weak areas."
- **assigment.md — Context (Phase 2):** "Implement a UI where a student can input their information and receive recommendations" — this endpoint backs the student form submission.
- **implementation-plan.md §4 — Data Freshness:** "Increment `profile_version` on profile or sales note updates."

## Design Document References

- [data-model.md — §2.1 Core Entities](../data-model.md): `students`, `student_goal_subjects`, `student_weak_skills` table definitions. Student status is `CHECK ('looking_for_new_coach','matched','paused')`.
- [architecture.md — §4 Component Overview — API Gateway](../architecture.md): Validates payloads, creates async requests and returns `request_id`.
- [ai-pipeline.md — §3.2 Student Upload Flow](../ai-pipeline.md): Sequence diagram showing student/sales -> API Gateway -> Job Queue -> profileBatchWorker -> normalization -> metadata + embedding.
- [technical-proposal.md — §4.2 Student Upload](../technical-proposal.md): Full sequence chart for student upload/update flow.
- [technical-proposal.md — §3 RAG Data Model — Student Profile Document](../technical-proposal.md): Fields including `learning_goals[]`, `weak_areas[]`, `preferred_style`, `constraints{}`.

## Description

Build the API endpoint and validation layer that accepts student profile submissions, persists them to the metadata database with a normalization step for learning goals and weak areas, sets the student status to `looking_for_new_coach`, and enqueues a profile indexing job.

The normalization step maps free-text learning goals and weak areas to canonical subjects and skills from the taxonomy (TICKET-005). This is critical for the downstream deterministic scoring engine (TICKET-007).

## Acceptance Criteria

- [ ] `POST /api/students` accepts a student profile JSON payload and returns `201 Created` with `{ student_id, profile_version, status }`.
- [ ] Request validation rejects payloads missing `learning_goals` or `weak_areas` with `400 Bad Request`.
- [ ] `current_level` must be one of `beginner`, `intermediate`, `advanced`.
- [ ] `preferred_learning_style` must be one of `structured`, `exploratory`.
- [ ] On successful upload:
  - A row is inserted into `students` with `status = 'looking_for_new_coach'`.
  - `learning_goals_raw` and `weak_areas_raw` store the original arrays as JSONB.
  - Normalized subject mappings are written to `student_goal_subjects`.
  - Normalized skill mappings are written to `student_weak_skills`.
- [ ] A profile indexing job is enqueued with `{ entity_type: 'student', entity_id: student_id, profile_version }`.
- [ ] `PUT /api/students/:student_id` updates an existing student, increments `profile_version`, and re-enqueues.
- [ ] `GET /api/students/:student_id` returns the student profile with both raw and normalized fields.
- [ ] Normalization handles the provided test data correctly:
  - S002: "Algebra", "Geometry" -> Math subject; "Newton's Laws" -> Physics subject.
  - S003: "Python basics", "Data structures" -> Programming subject; "Statistics" -> Math subject.
  - S004: "Japanese grammar", "Kanji writing" -> no matching teacher subject (graceful handling).

## Technical Details

### API Contract

**Create Student**
```
POST /api/students
Content-Type: application/json

{
  "id": "S002",
  "name": "Student 1",
  "age": 15,
  "learning_goals": [
    "Understand core Math concepts",
    "Build confidence in Physics"
  ],
  "weak_areas": ["Algebra", "Geometry", "Newton's Laws"],
  "current_level": "beginner",
  "preferred_learning_style": "structured"
}
```

**Response**
```json
{
  "student_id": "S002",
  "profile_version": 1,
  "status": "looking_for_new_coach",
  "indexing_job_id": "job_def456",
  "normalized_subjects": ["Math", "Physics"],
  "normalized_weak_skills": ["Algebra", "Geometry", "Newton's Laws"]
}
```

### Normalization Service

The normalization service maps free-text student inputs to canonical taxonomy entries:

1. **Goal-to-subject mapping:** Parse `learning_goals[]` and extract subject references. Use keyword matching against `subjects.display_name` and LLM-assisted extraction for ambiguous goals.
2. **Weak-area-to-skill mapping:** Match `weak_areas[]` against `skills.display_name` and `skill_aliases.alias_text`. Record severity (default: `medium`).
3. **Confidence tagging:** Each mapping gets a `confidence` score (`high` for exact match, `medium` for alias match, `low` for LLM-inferred).
4. **Unmatched handling:** Store unmatched items in raw fields; flag for HITL review if critical.

### Database Operations (single transaction)

1. Upsert `students` row with raw fields.
2. Run normalization service on `learning_goals_raw` and `weak_areas_raw`.
3. Upsert `student_goal_subjects` with `(student_id, subject_id, source, confidence)`.
4. Upsert `student_weak_skills` with `(student_id, skill_id, severity)`.
5. Enqueue indexing job with idempotency key `student:{student_id}:v{profile_version}`.

## Dependencies

- **TICKET-000** — Repo structure, `packages/api` scaffold.
- **TICKET-001** — Database schema must exist (tables: `students`, `student_goal_subjects`, `student_weak_skills`).
- **TICKET-005** — Subject and skill taxonomy must be seeded for normalization to work.

## Test Plan

### Unit Tests
- **Validation — missing required fields:** Send POST with missing `learning_goals`; verify 400. Repeat for missing `weak_areas`.
- **Validation — invalid current_level:** Send POST with `current_level: "expert"`; verify 400 with descriptive error. Verify `beginner`, `intermediate`, `advanced` all accepted.
- **Validation — invalid learning_style:** Send POST with `preferred_learning_style: "hybrid"`; verify 400. Verify `structured` and `exploratory` are accepted.
- **Normalization — Newton's Laws:** Pass `"Newton's Laws"` through `normalizeWeakAreas()`; verify it maps to canonical skill `newtons_laws` via alias with `confidence: high`.
- **Normalization — direct match:** Pass `"Algebra"` through normalization; verify it maps to skill `algebra` with `confidence: high`.
- **Normalization — unmatched term:** Pass `"Japanese grammar"` through normalization; verify it returns `confidence: low` or `null` with a review flag.
- **Status default:** Verify new student records are created with `status = 'looking_for_new_coach'`.

### Integration Tests
- **POST + DB verification:** POST student S002 payload; query `students` table and verify row exists with `status = 'looking_for_new_coach'`, `learning_goals_raw` as JSONB, and `profile_version = 1`. Query `student_goal_subjects` for S002; verify Math and Physics subject mappings. Query `student_weak_skills` for S002; verify 3 skill entries (Algebra, Geometry, Newton's Laws).
- **Indexing job enqueue:** After POST S002, verify a profile indexing job is enqueued with `{ entity_type: 'student', entity_id: 'S002', profile_version: 1 }`.
- **PUT update + re-normalization:** PUT update to S002 adding a new weak area; verify `profile_version = 2`, new skills added to `student_weak_skills`, and re-index job enqueued.
- **GET student:** POST S002, then GET `/api/students/S002`; verify response includes both raw and normalized fields.

### E2E / Manual Tests
- **Upload all 3 students:** POST all 3 students from `dataset/new_students.json` via API. Verify:
  - S002: `student_goal_subjects` contains Math and Physics; `student_weak_skills` contains Algebra, Geometry, Newton's Laws.
  - S003: `student_goal_subjects` contains Programming and Math; `student_weak_skills` contains Python basics, Statistics, Data structures.
  - S004: `student_goal_subjects` may be empty or contain low-confidence entries (no matching teacher subjects); `student_weak_skills` contains Japanese grammar, Kanji writing, Modern Asian history with `confidence: low`.
- **Graceful handling of S004:** Verify S004 is accepted and stored despite having no matching teacher subjects. Verify `status = 'looking_for_new_coach'` is set.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: POST returns 201 with student_id + version + status | Integration | POST + DB verification |
| AC: Rejects missing learning_goals/weak_areas | Unit | Validation — missing required fields |
| AC: current_level must be beginner/intermediate/advanced | Unit | Validation — invalid current_level |
| AC: preferred_learning_style must be structured/exploratory | Unit | Validation — invalid learning_style |
| AC: Inserts students with status=looking_for_new_coach | Unit + Integration | Status default + POST+DB verification |
| AC: Stores raw JSONB + normalized mappings | Integration | POST+DB verification (raw + normalized) |
| AC: Indexing job enqueued | Integration | Indexing job enqueue verification |
| AC: PUT updates + re-enqueues | Integration | PUT update + re-normalization |
| AC: S002 normalization correct | E2E | Upload all 3 students — S002 check |
| AC: S003 normalization correct | E2E | Upload all 3 students — S003 check |
| AC: S004 graceful no-match handling | E2E | Graceful handling of S004 |

## Dataset References

- **`dataset/new_students.json`:** 3 student records:
  - **S002** (age 15, beginner, structured): Goals in Math and Physics, weak in Algebra/Geometry/Newton's Laws.
  - **S003** (age 19, beginner, structured): Goals in Python/data science and statistics for ML, weak in Python basics/Statistics/Data structures.
  - **S004** (age 22, beginner, structured): Goals in Japanese and East Asian history — tests edge case where no matching teacher subjects exist in the current teacher pool.
