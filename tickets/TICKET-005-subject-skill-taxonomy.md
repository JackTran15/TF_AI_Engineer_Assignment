# TICKET-005: Subject and Skill Taxonomy

## Phase

**Phase 1 — Data Model and Profile Indexing Pipeline**  
Ref: `implementation-plan.md §7 Phase 1` — "Finalize schema and ingestion contracts."

## Assignment Reference

- **assigment.md — Context (Phase 1):** Teachers have subjects and students have weak areas. A controlled taxonomy is needed to map between them reliably for matching.
- **assigment.md — Goal:** "Automatically find the best-match teacher" requires that student weak areas can be mapped to teacher subjects and skills for scoring.

## Design Document References

- [data-model.md — §2.1 Core Entities](../data-model.md): `subjects` table (`subject_id`, `code`, `display_name`, `is_active`), `skills` table (`skill_id`, `code`, `subject_id FK`, `display_name`), `skill_aliases` table (`alias_id`, `skill_id FK`, `alias_text`).
- [data-model.md — §2.1](../data-model.md): "Controlled subject taxonomy (replaces free-text drift)" and "Synonyms map raw text (for example `newton's laws`) to canonical skills."
- [ai-pipeline.md — §3.2 Student Upload Flow](../ai-pipeline.md): Normalization Service normalizes goals, weak areas, and preferences using the taxonomy.

## Description

Create and seed the controlled taxonomy tables (`subjects`, `skills`, `skill_aliases`) that power normalization of student goals and weak areas into canonical terms. This taxonomy is the shared vocabulary that enables deterministic matching between students and teachers.

Without this taxonomy, the system would rely on fuzzy text matching, leading to inconsistent scoring and unreliable recommendations.

## Acceptance Criteria

- [ ] `subjects` table is seeded with all subjects found in `teachers.json`: Math, Physics, English, Chemistry, Programming.
- [ ] `skills` table is seeded with granular skills mapped to parent subjects, covering all `weak_areas` in `new_students.json`.
- [ ] `skill_aliases` table maps common synonyms and variant spellings to canonical skills.
- [ ] The normalization function `normalizeWeakAreas(rawAreas: string[]) -> { skill_id, confidence }[]` resolves all 9 weak areas from the 3 test students.
- [ ] The normalization function `normalizeGoals(rawGoals: string[]) -> { subject_id, confidence }[]` resolves all 6 learning goals from the 3 test students.
- [ ] Unrecognized terms return a `low` confidence mapping or `null` with a flag for manual review.
- [ ] Taxonomy supports CRUD operations via admin endpoints (stretch goal) or direct DB manipulation.
- [ ] Adding a new skill or alias does not require code changes — only a DB insert.

## Technical Details

### Seed Data

**Subjects:**

| code | display_name |
|---|---|
| `math` | Math |
| `physics` | Physics |
| `english` | English |
| `chemistry` | Chemistry |
| `programming` | Programming |

**Skills (with parent subject):**

| code | display_name | subject |
|---|---|---|
| `algebra` | Algebra | Math |
| `geometry` | Geometry | Math |
| `statistics` | Statistics | Math |
| `newtons_laws` | Newton's Laws | Physics |
| `python_basics` | Python basics | Programming |
| `data_structures` | Data structures | Programming |
| `japanese_grammar` | Japanese grammar | _(no parent — unmatched)_ |
| `kanji_writing` | Kanji writing | _(no parent — unmatched)_ |
| `modern_asian_history` | Modern Asian history | _(no parent — unmatched)_ |

**Skill Aliases:**

| alias_text | maps_to_skill |
|---|---|
| `newton's laws` | `newtons_laws` |
| `newtonian mechanics` | `newtons_laws` |
| `python` | `python_basics` |
| `basic python` | `python_basics` |
| `stats` | `statistics` |
| `maths` | → subject-level alias for `math` |

### Normalization Logic

```
function normalizeWeakAreas(rawAreas: string[]): NormalizedSkill[] {
  for each area in rawAreas:
    1. Exact match against skills.display_name (case-insensitive) -> confidence: high
    2. Exact match against skill_aliases.alias_text -> confidence: high
    3. Fuzzy/substring match against skills or aliases -> confidence: medium
    4. No match -> store raw, flag for review, confidence: low
  return mapped skills with confidence scores
}
```

### Edge Case: Student S004 (Japanese/History)

Student S004 has goals and weak areas in Japanese and East Asian history. No teachers in the current pool cover these subjects. The taxonomy should:
1. Still create skill entries (even without a parent subject match to a teacher).
2. Mark these as `is_active = true` so future teachers can be mapped.
3. The recommendation pipeline (TICKET-007) will handle the "no matching teacher" case gracefully.

## Dependencies

- **TICKET-001** — Database schema must exist (tables: `subjects`, `skills`, `skill_aliases`).

## Test Plan

### Unit Tests
- **Alias resolution — exact match:** Call `normalizeWeakAreas(["Algebra"])` ; verify it returns `skill_id` for `algebra` with `confidence: high`.
- **Alias resolution — case-insensitive:** Call `normalizeWeakAreas(["newton's laws"])`; verify it resolves via `skill_aliases` to `newtons_laws` with `confidence: high`.
- **Alias resolution — variant spelling:** Call `normalizeWeakAreas(["newtonian mechanics"])`; verify it maps to `newtons_laws`.
- **Alias resolution — Python shorthand:** Call `normalizeWeakAreas(["python"])`; verify it maps to `python_basics`.
- **Unmatched term:** Call `normalizeWeakAreas(["quantum entanglement"])`; verify it returns `confidence: low` or `null` with a review flag.
- **Goal normalization:** Call `normalizeGoals(["Understand core Math concepts"])`; verify it returns `subject_id` for `math`.
- **Duplicate subject code rejection:** Attempt to insert a second subject with `code = 'math'`; verify the UNIQUE constraint rejects it.
- **is_active filtering:** Mark a skill as `is_active = false`; verify normalization excludes it from match results.

### Integration Tests
- **Seed data completeness:** Run the taxonomy seed script; verify `subjects` table contains exactly 5 entries (Math, Physics, English, Chemistry, Programming). Verify `skills` table contains all 9 skills from `new_students.json` weak areas. Verify `skill_aliases` table contains expected mappings.
- **Normalization against all test students:** Run `normalizeWeakAreas` for each student's weak areas from `new_students.json`:
  - S002: Algebra -> `algebra` (high), Geometry -> `geometry` (high), Newton's Laws -> `newtons_laws` (high).
  - S003: Python basics -> `python_basics` (high), Statistics -> `statistics` (high), Data structures -> `data_structures` (high).
  - S004: Japanese grammar -> `japanese_grammar` (low — no parent subject with teachers), Kanji writing -> `kanji_writing` (low), Modern Asian history -> `modern_asian_history` (low).
- **Goal normalization against all test students:** Run `normalizeGoals` for each student's learning goals; verify subject IDs are returned for known subjects.

### E2E / Manual Tests
- **Manual DB inspection:** After seeding, connect to the database and query:
  - `SELECT * FROM subjects ORDER BY code;` — verify 5 rows matching teachers.json subjects.
  - `SELECT sk.code, sk.display_name, sub.code AS parent FROM skills sk LEFT JOIN subjects sub ON sk.subject_id = sub.subject_id;` — verify parent mappings and 3 orphaned skills (Japanese grammar, Kanji writing, Modern Asian history).
  - `SELECT * FROM skill_aliases;` — verify alias entries.
- **Add new skill without code change:** Insert a new skill `calculus` under `math` directly into the DB; run normalization for `["calculus"]`; verify it resolves with `confidence: high`.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: subjects seeded with 5 teacher subjects | Integration | Seed data completeness — subjects check |
| AC: skills seeded covering new_students.json | Integration | Seed data completeness — skills check |
| AC: skill_aliases maps synonyms | Unit | Alias resolution tests (multiple variants) |
| AC: normalizeWeakAreas resolves all 9 weak areas | Integration | Normalization against all test students |
| AC: normalizeGoals resolves all 6 goals | Integration | Goal normalization against all test students |
| AC: Unrecognized terms return low confidence | Unit | Unmatched term test |
| AC: No code changes needed for new skills | E2E/Manual | Add new skill without code change |
| AC: is_active flag filters correctly | Unit | is_active filtering test |

## Dataset References

- **`dataset/teachers.json`:** Subjects extracted: Math, Physics, English, Chemistry, Programming. These define the initial subject taxonomy.
- **`dataset/new_students.json`:** Weak areas to normalize: Algebra, Geometry, Newton's Laws, Python basics, Statistics, Data structures, Japanese grammar, Kanji writing, Modern Asian history. These define the initial skill taxonomy and test the normalization function.
