# AI Coaching Recommendation System — Assignment

## Context

**Phase 1 — Build the pipeline**

Our platform currently has **100 teachers** (see `teachers.json`). Each teacher has:

- A **profile** (subjects, teaching style, experience)
- A **skill evaluation scoring metrics** (e.g. Math: 92, Communication: 78)

When **new students register** (see `new_students.json`), they provide their learning goals and weak areas.

**Goal:** Automatically find the **best-match teacher**, plus **3 alternatives** for the new students with an explanation for each match.

The pipeline should **utilize LLM** where it adds value.

**Phase 2 — Expose the pipeline to the world**

Once the pipeline is in place, the next step is making it accessible:

- **Expose an API** so external systems or clients can trigger the recommendation pipeline
- **Implement a UI** where a student can input their information and receive recommendations

**Remember**: The UI triggers the pipeline via the API, but the pipeline runs **in the background**

---

## Time expectation

**< 30 minutes** for the full assignment including the recording.

---

## Deliverables


| File                     | Content                                                                  |
| ------------------------ | ------------------------------------------------------------------------ |
| `data-model.md`          | Entities, relationships, schema design                                   |
| `architecture.md`        | System architecture diagram + component overview                         |
| `ai-pipeline.md`         | AI matching & recommendation pipeline                                    |
| `implementation-plan.md` | Implementation plan (see section below)                                  |
| `output.md`              | Pipeline trace, final selection + LLM-generated explanations (see below) |
| `screen-recording.mp4`   | Screen recording walking through your plan (< 30 min)                    |


### `output.md` — expected content

Run the pipeline against `new_student.json` and `teachers.json` and produce:

- **Pipeline trace** — an ASCII diagram + each step taken (filtering, scoring, LLM calls) with inputs and outputs shown
- **Final selection** — best match + 3 alternatives, clearly ranked
- **Explanations** — one LLM-generated explanation per selected teacher, tied to specific student and teacher data

## Implementation plan

In `implementation-plan.md`, cover your approach to the following:

- **Concurrency** — if 100 or 1,000 students trigger the pipeline simultaneously, how does your system keep up?
- **External API constraints** — third-party services have limits. What are the limits and how does your pipeline stay resilient under pressure?
- **Data freshness** — teacher profiles and scores change over time; how do you make sure recommendations don't go stale?
- **Evaluation** — how do you know the recommendations are actually good?

---

## Notes

- **No code required**
- **Record your screen for the full duration of the assignment**
- Use any AI tool you're comfortable with: Cursor, Codepilot, Claude Code, ...

