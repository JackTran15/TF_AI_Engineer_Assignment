# TICKET-001: Database Schema and Migrations

## Phase

**Phase 0 — Repository and Infrastructure Bootstrap**

Establishes the full PostgreSQL schema with pgvector extension, migration tooling, and seed scripts that load the provided dataset files.

## Assignment Reference

- **assigment.md — Context (Phase 1):** Teachers have profiles with subjects, teaching style, experience, and skill evaluation scoring metrics. Students provide learning goals and weak areas. The schema must capture all of these fields.
- **assigment.md — Deliverables — `data-model.md`:** This ticket implements the schema defined in `data-model.md`.
- **implementation-plan.md §1 — Phase 1:** "Finalize schema and ingestion contracts."

## Design Document References

- [data-model.md — §2 PostgreSQL Schema](../data-model.md): Complete table definitions for core entities, RAG indexing entities, recommendation entities, and HITL/audit entities.
- [data-model.md — §3 Vector Model](../data-model.md): pgvector column definition with `vector_id`, `entity_type`, `embedding`, metadata.
- [data-model.md — §5 Data Freshness and Versioning Rules](../data-model.md): `profile_version`, `embedding_version`, idempotency via deterministic `job_id`.
- [technical-proposal.md — §10.1 Metadata Database](../technical-proposal.md): PostgreSQL selection rationale.
- [technical-proposal.md — §10.2 Vector Database](../technical-proposal.md): pgvector as Phase-1 baseline.

## Description

Create SQL migration files that build the full database schema in a repeatable, version-controlled manner. Additionally, build seed scripts that load `teachers.json` and `new_students.json` into the appropriate tables so downstream tickets have working data from day one.

## Acceptance Criteria

- [ ] Migration tooling is configured (e.g. `node-pg-migrate`, `knex`, or raw SQL files with an execution script).
- [ ] Running migrations from a clean database produces all tables defined in `data-model.md §2`.
- [ ] pgvector extension is enabled (`CREATE EXTENSION IF NOT EXISTS vector`).
- [ ] The vector table includes an `embedding vector(1536)` column with an IVFFlat or HNSW index.
- [ ] All `CHECK` constraints from the data model are present (e.g. `teaching_style CHECK ('structured','exploratory')`, `rank CHECK (1-4)`, score ranges `0-100`, satisfaction `0-5`).
- [ ] All `UNIQUE` and `INDEX` constraints from the data model are present.
- [ ] A seed script reads `dataset/teachers.json` and inserts rows into `teachers`, `teacher_skill_scores`, `teacher_subjects`, and `subjects`.
- [ ] A seed script reads `dataset/new_students.json` and inserts rows into `students` with `status = 'looking_for_new_coach'`.
- [ ] Running `migrate up` then `seed` on a fresh Postgres instance completes without errors.
- [ ] A `migrate down` or rollback mechanism exists.

## Technical Details

### Migration Order

1. `001_enable_pgvector.sql` — Enable the vector extension.
2. `002_core_entities.sql` — `teachers`, `subjects`, `skills`, `skill_aliases`, `teacher_subjects`, `teacher_skill_scores`, `students`, `student_goal_subjects`, `student_weak_skills`, `profile_notes`.
3. `003_rag_indexing.sql` — `profile_chunks`, `embedding_jobs`, `embedding_versions`, vector table.
4. `004_recommendation.sql` — `recommendation_requests`, `recommendation_results`, `recommendation_explanations`, `recommendation_citations`, `pipeline_trace_steps`.
5. `005_hitl_audit.sql` — `hitl_cases`, `hitl_edits`, `batch_run_logs`.

### Key Table Definitions (from data-model.md)

**teachers**
```sql
CREATE TABLE teachers (
  teacher_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  bio TEXT,
  teaching_style TEXT NOT NULL CHECK (teaching_style IN ('structured', 'exploratory')),
  experience_years INTEGER NOT NULL,
  profile_version INTEGER NOT NULL DEFAULT 1,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**teacher_skill_scores**
```sql
CREATE TABLE teacher_skill_scores (
  id SERIAL PRIMARY KEY,
  teacher_id TEXT NOT NULL REFERENCES teachers(teacher_id),
  skill_name TEXT NOT NULL,
  score INTEGER NOT NULL CHECK (score >= 0 AND score <= 100),
  scoring_version INTEGER NOT NULL DEFAULT 1,
  is_current BOOLEAN NOT NULL DEFAULT true,
  UNIQUE (teacher_id, skill_name, scoring_version)
);
CREATE INDEX idx_teacher_skill_current ON teacher_skill_scores(teacher_id) WHERE is_current = true;
```

**students**
```sql
CREATE TABLE students (
  student_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  age INTEGER,
  current_level TEXT,
  preferred_learning_style TEXT,
  learning_goals_raw JSONB,
  weak_areas_raw JSONB,
  status TEXT NOT NULL DEFAULT 'looking_for_new_coach'
    CHECK (status IN ('looking_for_new_coach', 'matched', 'paused')),
  profile_version INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Seed Script Logic

**Teacher seed (`scripts/seed-teachers.ts`):**
1. Read `dataset/teachers.json`.
2. For each teacher object:
   - Insert into `teachers` (`teacher_id=id`, `name`, `bio`, `teaching_style`, `experience_years`).
   - For each subject in `subjects[]`: upsert into `subjects` table, then insert into `teacher_subjects`.
   - For each key in `scores{}`: insert into `teacher_skill_scores` (`skill_name=key`, `score=value`). Map `student_satisfaction` as score * 20 to normalize to 0-100 range, or store raw in a separate column.

**Student seed (`scripts/seed-students.ts`):**
1. Read `dataset/new_students.json`.
2. For each student object:
   - Insert into `students` (`student_id=id`, `name`, `age`, `current_level`, `preferred_learning_style`, `learning_goals_raw=learning_goals`, `weak_areas_raw=weak_areas`, `status='looking_for_new_coach'`).

## Dependencies

- **TICKET-000** — Repo structure and Docker Compose (Postgres must be running).

## Test Plan

### Unit Tests
- **Migration idempotency:** Run `migrate up`, `migrate down`, `migrate up` in sequence on a clean database; verify no errors on any step and final schema matches expected state.
- **CHECK constraint — teaching_style:** Insert a teacher with `teaching_style = 'invalid'`; verify the INSERT is rejected with a constraint violation.
- **CHECK constraint — rank range:** Insert a `recommendation_results` row with `rank = 5`; verify rejection. Insert with `rank = 0`; verify rejection. Insert with `rank = 1..4`; verify acceptance.
- **CHECK constraint — status enum:** Insert a student with `status = 'unknown'`; verify rejection. Verify all valid values (`looking_for_new_coach`, `matched`, `paused`) are accepted.
- **CHECK constraint — score range:** Insert a `teacher_skill_scores` row with `score = -1`; verify rejection. Insert with `score = 101`; verify rejection. Insert with `score = 0` and `score = 100`; verify acceptance.
- **UNIQUE constraints:** Insert duplicate `(teacher_id, skill_name, scoring_version)` into `teacher_skill_scores`; verify rejection. Insert duplicate `(request_id, rank)` into `recommendation_results`; verify rejection.

### Integration Tests
- **Full schema creation:** Run all migrations on a fresh Postgres 16 instance; verify all tables from data-model.md exist using `SELECT table_name FROM information_schema.tables`.
- **pgvector extension:** Verify `CREATE EXTENSION IF NOT EXISTS vector` succeeds and the vector column type is available.
- **Teacher seed script:** Run `seed-teachers.ts` against `dataset/teachers.json`; verify 10 rows in `teachers`, correct subject mappings in `teacher_subjects`, and 50 rows in `teacher_skill_scores` (10 teachers x 5 metrics each) with `is_current = true`.
- **Student seed script:** Run `seed-students.ts` against `dataset/new_students.json`; verify 3 rows in `students` with `status = 'looking_for_new_coach'`, `learning_goals_raw` and `weak_areas_raw` as JSONB.
- **FK integrity:** Attempt to insert a `teacher_subjects` row referencing a non-existent `teacher_id`; verify FK violation.
- **Index verification:** Query `pg_indexes` to confirm all expected indexes exist (e.g., `idx_teacher_skill_current`).

### E2E / Manual Tests
- **Full migration + seed cycle:** On a freshly started Docker Compose Postgres, run `migrate up` then both seed scripts; connect with `psql` and run `\dt` to list all tables; spot-check `SELECT * FROM teachers LIMIT 3` and `SELECT * FROM students`.
- **Rollback verification:** After full migration, run `migrate down` to rollback the last migration; verify the dropped tables no longer exist; run `migrate up` again to restore.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: Migration tooling configured | Integration | Full schema creation from clean DB |
| AC: All data-model.md tables created | Integration | information_schema.tables query |
| AC: pgvector extension enabled | Integration | pgvector extension verification |
| AC: Vector column with HNSW index | Integration | Index verification via pg_indexes |
| AC: CHECK constraints present | Unit | Constraint tests for style, rank, status, scores |
| AC: UNIQUE/INDEX constraints present | Unit | Duplicate insertion rejection tests |
| AC: Teacher seed loads 10 teachers | Integration | Teacher seed script row count verification |
| AC: Student seed loads 3 students | Integration | Student seed script row count verification |
| AC: migrate up+seed completes clean | E2E/Manual | Full migration + seed cycle |
| AC: Rollback mechanism exists | E2E/Manual | Rollback verification |

## Dataset References

- **`dataset/teachers.json`:** 10 teacher records with fields `id`, `name`, `subjects[]`, `teaching_style`, `experience_years`, `preferred_student_level[]`, `scores{}`, `bio`. Each scores object has: `subject_knowledge`, `communication`, `problem_solving`, `patience`, `student_satisfaction` (0-5 scale).
- **`dataset/new_students.json`:** 3 student records with fields `id`, `name`, `age`, `learning_goals[]`, `weak_areas[]`, `current_level`, `preferred_learning_style`.
