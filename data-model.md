# Data Model — AI Coaching Recommendation System

## 1) Entity Overview

This schema supports:
- Profile upload and versioning
- RAG indexing and vector retrieval
- Batch workers (`profileBatchWorker`, `recommendationBatchWorker`)
- Recommendation results, citations, and HITL audit

## 2) PostgreSQL Schema (Recommended)

### 2.1 Core Entities

| Table | Key Columns | Purpose |
|---|---|---|
| `teachers` | `teacher_id (PK)`, `profile_version`, `is_active`, `teaching_style CHECK ('structured','exploratory')` | Teacher profile master record |
| `teacher_skill_scores` | `teacher_id (FK->teachers)`, `scoring_version`, `is_current`, `UNIQUE (teacher_id, scoring_version)`, `INDEX (teacher_id) WHERE is_current=true` | Versioned teacher scoring metrics with score-range checks (`0-100`, satisfaction `0-5`) |
| `students` | `student_id (PK)`, `profile_version`, `status CHECK ('looking_for_new_coach','matched','paused')`, `current_level`, `preferred_learning_style`, `learning_goals_raw`, `weak_areas_raw` | Student profile + status and explicit mapping from `new_students.json` inputs |
| `profile_notes` | `note_id`, `entity_type`, `entity_id`, `note_source` | Sales notes and correction notes |
| `subjects` | `subject_id (PK)`, `code (UNIQUE)`, `display_name`, `is_active` | Controlled subject taxonomy (replaces free-text drift) |
| `skills` | `skill_id (PK)`, `code (UNIQUE)`, `subject_id (FK->subjects)`, `display_name`, `is_active` | Controlled skill taxonomy for weak area normalization |
| `skill_aliases` | `alias_id (PK)`, `skill_id (FK->skills)`, `alias_text (UNIQUE)` | Synonyms map raw text (for example `newton's laws`) to canonical skills |
| `teacher_subjects` | `teacher_id (FK->teachers)`, `subject_id (FK->subjects)`, `UNIQUE (teacher_id, subject_id)` | Many-to-many mapping of teacher teachable subjects |
| `student_goal_subjects` | `student_id (FK->students)`, `subject_id (FK->subjects)`, `source`, `confidence`, `UNIQUE (student_id, subject_id, source)` | Canonicalized subjects extracted from `learning_goals_raw` and `weak_areas_raw` |
| `student_weak_skills` | `student_id (FK->students)`, `skill_id (FK->skills)`, `severity`, `UNIQUE (student_id, skill_id)` | Canonical weak skills used by ranking and explanations |

### 2.2 RAG Indexing Entities

| Table | Key Columns | Purpose |
|---|---|---|
| `profile_chunks` | `chunk_id (PK)`, `entity_type`, `entity_id`, `profile_version`, `INDEX (entity_type, entity_id, profile_version)` | Chunked text for teacher/student profiles (`entity_type` determines FK target) |
| `embedding_jobs` | `job_id (PK)`, `job_type CHECK ('teacher_profile','student_profile','reindex')`, `status CHECK ('queued','processing','completed','failed')`, `batch_id` | Batch jobs consumed by `profileBatchWorker` |
| `embedding_versions` | `entity_type`, `entity_id`, `embedding_version`, `UNIQUE (entity_type, entity_id)` | Current embedding model and version state |

### 2.3 Recommendation Entities

| Table | Key Columns | Purpose |
|---|---|---|
| `recommendation_requests` | `request_id (PK)`, `student_id (FK->students)`, `status CHECK ('queued','processing','completed','failed','hitl_review')`, `INDEX (student_id, status, created_at DESC)` | Request lifecycle and retrieval path |
| `recommendation_results` | `result_id (PK)`, `request_id (FK->recommendation_requests)`, `teacher_id (FK->teachers)`, `rank CHECK (1-4)`, `UNIQUE (request_id, rank)`, `UNIQUE (request_id, teacher_id)`, `INDEX (request_id, rank)` | Top recommendation list (1 best + 3 alternatives) with rank uniqueness guaranteed |
| `recommendation_explanations` | `explanation_id (PK)`, `result_id (FK->recommendation_results)`, `llm_model` | LLM explanations per selected teacher |
| `recommendation_citations` | `citation_id (PK)`, `result_id (FK->recommendation_results)`, `source_id`, `chunk_id (FK->profile_chunks)` | Evidence links for each claim |
| `pipeline_trace_steps` | `trace_id (PK)`, `request_id (FK->recommendation_requests)`, `step_name`, `step_order`, `UNIQUE (request_id, step_order)`, `INDEX (request_id, step_order)` | Traceability and deterministic debugging order |

### 2.4 HITL and Worker Audit Entities

| Table | Key Columns | Purpose |
|---|---|---|
| `hitl_cases` | `case_id (PK)`, `request_id (FK->recommendation_requests)`, `trigger_reason`, `status` | Manual review workflow |
| `hitl_edits` | `edit_id (PK)`, `case_id (FK->hitl_cases)`, `editor_id`, `reason_code` | Sales adjustments and override reasons |
| `batch_run_logs` | `run_id`, `worker_name`, `batch_size`, `status` | Throughput and reliability for worker batches |

## 3) Vector Model (pgvector Phase-1)

| Field | Description |
|---|---|
| `vector_id` | Deterministic id (for example `teacher:T001:chunk:03:v2`) |
| `entity_type` | `teacher` or `student` |
| `entity_id` | Entity reference id |
| `chunk_id` | FK to `profile_chunks.chunk_id` |
| `embedding` | Vector column (`vector`) |
| `embedding_version` | Embedding model version |
| `metadata` | Filterable metadata (`subjects`, `teaching_style`, `language`, `is_active`) |
| `updated_at` | Freshness timestamp |

## 4) Relationship Diagram

```mermaid
classDiagram
    class "teachers" as teachers
    class "subjects" as subjects
    class "skills" as skills
    class "skill_aliases" as skill_aliases
    class "teacher_subjects" as teacher_subjects
    class "teacher_skill_scores" as teacher_skill_scores
    class "students" as students
    class "student_goal_subjects" as student_goal_subjects
    class "student_weak_skills" as student_weak_skills
    class "profile_notes" as profile_notes
    class "profile_chunks" as profile_chunks
    class "embedding_jobs" as embedding_jobs
    class "recommendation_requests" as recommendation_requests
    class "recommendation_results" as recommendation_results
    class "recommendation_explanations" as recommendation_explanations
    class "recommendation_citations" as recommendation_citations
    class "pipeline_trace_steps" as pipeline_trace_steps
    class "hitl_cases" as hitl_cases
    class "hitl_edits" as hitl_edits
    class "batch_run_logs" as batch_run_logs

    teachers --> teacher_subjects : "teaches subjects"
    subjects --> teacher_subjects : "subject mapping"
    subjects --> skills : "owns skills"
    skills --> skill_aliases : "alias map"
    teachers --> teacher_skill_scores : "versioned scores"
    teachers --> profile_chunks : "teacher chunks"
    students --> profile_chunks : "student chunks"
    students --> student_goal_subjects : "normalized goal subjects"
    subjects --> student_goal_subjects : "canonical subject"
    students --> student_weak_skills : "normalized weak skills"
    skills --> student_weak_skills : "canonical skill"
    students --> profile_notes : "sales notes"
    teachers --> profile_notes : "sales notes"
    students --> recommendation_requests : "submits request"
    recommendation_requests --> recommendation_results : "returns top 4"
    recommendation_results --> recommendation_explanations : "has explanation"
    recommendation_results --> recommendation_citations : "has citations"
    recommendation_requests --> pipeline_trace_steps : "execution trace"
    recommendation_requests --> hitl_cases : "manual review case"
    hitl_cases --> hitl_edits : "human adjustments"
    embedding_jobs --> batch_run_logs : "worker run logs"
```

## 5) Data Freshness and Versioning Rules

- `profile_version` increments on every profile or sales note change.
- `embedding_version` increments when embedding model changes.
- Recommendation request stores `profile_version` and `embedding_version` snapshot for reproducibility.
- Workers are idempotent via deterministic `job_id` and dedup checks.
