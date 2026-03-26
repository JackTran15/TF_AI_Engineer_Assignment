# TICKET-002: Teacher Data Ingestion

## Phase

**Phase 1 — Data Model and Profile Indexing Pipeline**  
Ref: `implementation-plan.md §7 Phase 1` — "Finalize schema and ingestion contracts."

## Assignment Reference

- **assigment.md — Context (Phase 1):** "Our platform currently has 100 teachers (see `teachers.json`). Each teacher has a profile (subjects, teaching style, experience) and a skill evaluation scoring metrics."
- **assigment.md — Context (Phase 2):** "Expose an API so external systems or clients can trigger the recommendation pipeline" — the teacher upload endpoint is part of the API surface.
- **implementation-plan.md §6:** API and workers run on Node.js + Fastify.

## Design Document References

- [data-model.md — §2.1 Core Entities](../data-model.md): `teachers`, `teacher_skill_scores`, `teacher_subjects` table definitions.
- [architecture.md — §4 Component Overview — API Gateway](../architecture.md): "Validates payloads, creates async requests."
- [ai-pipeline.md — §3.1 Teacher Upload Flow](../ai-pipeline.md): Sequence diagram showing teacher ops -> API Gateway -> Job Queue -> profileBatchWorker.
- [technical-proposal.md — §4.1 Teacher Upload](../technical-proposal.md): Full sequence chart for teacher upload -> queue -> embedding.
- [technical-proposal.md — §3 RAG Data Model — Teacher Profile Document](../technical-proposal.md): Fields for the teacher profile document.

## Description

Build the API endpoint and validation layer that accepts teacher profile uploads, persists them to the metadata database, and enqueues a profile indexing job for downstream embedding by the `profileBatchWorker` (TICKET-004).

This endpoint serves two purposes:
1. **Bulk seed:** Load all 10 teachers from `teachers.json` at startup or via a seed command.
2. **Runtime upload:** Accept individual teacher profile creation and updates via REST API.

## Acceptance Criteria

- [ ] `POST /api/teachers` endpoint accepts a teacher profile JSON payload and returns `201 Created` with `{ teacher_id, profile_version }`.
- [ ] Request validation rejects payloads missing required fields (`name`, `subjects`, `teaching_style`, `scores`) with `400 Bad Request` and descriptive error messages.
- [ ] `teaching_style` must be one of `structured` or `exploratory`; other values are rejected.
- [ ] Score values are validated: `subject_knowledge`, `communication`, `problem_solving`, `patience` must be 0-100; `student_satisfaction` must be 0-5.
- [ ] On successful upload, a row is inserted into `teachers`, corresponding rows into `teacher_subjects` (with subject upsert into `subjects`), and rows into `teacher_skill_scores`.
- [ ] `profile_version` is set to 1 for new teachers and incremented on update.
- [ ] After database writes, a profile indexing job is enqueued to the job queue with payload `{ entity_type: 'teacher', entity_id: teacher_id, profile_version }`.
- [ ] `PUT /api/teachers/:teacher_id` updates an existing teacher, increments `profile_version`, and enqueues a re-index job.
- [ ] `GET /api/teachers/:teacher_id` returns the current teacher profile.
- [ ] `GET /api/teachers` returns a paginated list of active teachers.
- [ ] Duplicate upload with the same `teacher_id` is handled as an upsert (update + version bump), not a conflict error.

## Technical Details

### API Contract

**Create Teacher**
```
POST /api/teachers
Content-Type: application/json

{
  "id": "T001",
  "name": "Sarah Mitchell",
  "subjects": ["Math", "Physics"],
  "teaching_style": "structured",
  "experience_years": 8,
  "preferred_student_level": ["beginner", "intermediate"],
  "scores": {
    "subject_knowledge": 92,
    "communication": 85,
    "problem_solving": 88,
    "patience": 90,
    "student_satisfaction": 4.8
  },
  "bio": "Specialises in breaking down complex Math and Physics concepts step by step."
}
```

**Response**
```json
{
  "teacher_id": "T001",
  "profile_version": 1,
  "status": "accepted",
  "indexing_job_id": "job_abc123"
}
```

### Validation Schema (Fastify JSON Schema or Zod)

- `id`: string, required
- `name`: string, required
- `subjects`: array of strings, min 1 item
- `teaching_style`: enum `['structured', 'exploratory']`
- `experience_years`: integer >= 0
- `preferred_student_level`: array of enum `['beginner', 'intermediate', 'advanced']`
- `scores`: object with required keys and range constraints
- `bio`: string, optional

### Database Operations (single transaction)

1. Upsert `teachers` row.
2. For each subject: upsert `subjects` row, upsert `teacher_subjects` mapping.
3. Mark previous `teacher_skill_scores` as `is_current = false` for this teacher.
4. Insert new `teacher_skill_scores` rows with `is_current = true` and incremented `scoring_version`.
5. Enqueue indexing job (outside transaction, with idempotency key `teacher:{teacher_id}:v{profile_version}`).

## Dependencies

- **TICKET-000** — Repo structure, `packages/api` scaffold.
- **TICKET-001** — Database schema must exist (tables: `teachers`, `teacher_skill_scores`, `subjects`, `teacher_subjects`).
- **TICKET-005** — Subject taxonomy seed data should be in place for subject normalization.

## Test Plan

### Unit Tests
- **Validation — missing required fields:** Send POST with missing `name`; verify 400 response. Repeat for missing `subjects`, `teaching_style`, `scores`.
- **Validation — invalid teaching_style:** Send POST with `teaching_style: "hybrid"`; verify 400 with descriptive error.
- **Validation — score range enforcement:** Send POST with `subject_knowledge: 101`; verify 400. Send with `student_satisfaction: 5.5`; verify 400. Send with `score: -1`; verify 400.
- **Validation — empty subjects array:** Send POST with `subjects: []`; verify 400.
- **Profile version increment:** Create teacher T001, then PUT to update; verify `profile_version` incremented from 1 to 2.
- **Upsert behavior:** POST teacher T001 twice; verify second request updates (not duplicates) and increments `profile_version`.

### Integration Tests
- **POST + DB verification:** POST a valid teacher payload (T001 from teachers.json); query `teachers` table for `teacher_id = 'T001'`; verify row exists with correct fields. Query `teacher_subjects` for T001; verify Math and Physics mappings. Query `teacher_skill_scores` for T001; verify 5 score rows with `is_current = true`.
- **Indexing job enqueue:** After POST, verify a profile indexing job exists in the queue with payload `{ entity_type: 'teacher', entity_id: 'T001', profile_version: 1 }`.
- **PUT update + version:** PUT an update to T001 changing `experience_years`; verify `profile_version = 2` in DB and a new indexing job is enqueued.
- **GET single teacher:** POST T001, then GET `/api/teachers/T001`; verify response matches posted data.
- **GET teacher list:** POST all 10 teachers, then GET `/api/teachers`; verify paginated response with 10 entries.
- **Score versioning:** POST T001, then PUT with changed scores; verify old `teacher_skill_scores` rows have `is_current = false` and new rows have `is_current = true`.

### E2E / Manual Tests
- **Bulk upload all teachers:** POST all 10 teachers from `dataset/teachers.json` via API. Query DB to confirm: 10 rows in `teachers`, correct `teacher_subjects` mappings (e.g., T001 -> Math+Physics, T006 -> Programming only), 50 `teacher_skill_scores` rows with `is_current = true`.
- **Verify T001 data integrity:** GET `/api/teachers/T001`; compare response field-by-field against the T001 entry in `teachers.json`.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: POST returns 201 with teacher_id + version | Integration | POST + DB verification |
| AC: Rejects missing required fields | Unit | Validation — missing fields |
| AC: teaching_style must be structured/exploratory | Unit | Validation — invalid teaching_style |
| AC: Score range validation (0-100, 0-5) | Unit | Validation — score range enforcement |
| AC: Inserts teachers, teacher_subjects, scores | Integration | POST + DB verification (all 3 tables) |
| AC: profile_version set to 1, incremented on update | Unit + Integration | Profile version increment + PUT update |
| AC: Indexing job enqueued after write | Integration | Indexing job enqueue verification |
| AC: PUT updates + re-indexes | Integration | PUT update + version test |
| AC: GET single + GET list work | Integration | GET single teacher + GET teacher list |
| AC: Duplicate upload = upsert | Unit | Upsert behavior test |

## Dataset References

- **`dataset/teachers.json`:** 10 teacher records. Each has `id` (T001-T010), `name`, `subjects[]`, `teaching_style`, `experience_years`, `preferred_student_level[]`, `scores{}` with 5 metrics, and `bio`. These records define the full teacher data shape that the ingestion endpoint must accept.
