# TICKET-000: Repository Initialization

## Phase

**Phase 0 — Repository and Infrastructure Bootstrap**

This ticket is a prerequisite for all subsequent work. It establishes the monorepo structure, dependency manifests, developer tooling, and CI skeleton before any feature code is written.

## Assignment Reference

- **assigment.md — Context (Phase 1 & Phase 2):** The repo must support both the AI pipeline (Phase 1) and the API + UI layer (Phase 2).
- **assigment.md — Deliverables table:** All deliverable files (`data-model.md`, `architecture.md`, `ai-pipeline.md`, `implementation-plan.md`, `output.md`) will live at the repo root.
- **implementation-plan.md §6 — Technical Selection Plan:** Node.js + Fastify for API/workers, Python + FastAPI + LangGraph for AI services, PostgreSQL + pgvector for storage.

## Design Document References

- [architecture.md — §3 C4 Container](../architecture.md): Defines the container boundaries that inform folder structure (API Gateway, Workers, RAG Orchestrator, Agent Runtime, Student UI, HITL Console).
- [implementation-plan.md — §6 Technical Selection Plan](../implementation-plan.md): Specifies the baseline stack and the Node.js / Python split rationale.
- [technical-proposal.md — §10 Technical Selections](../technical-proposal.md): Detailed pros/cons for each technology choice.

## Description

Set up a monorepo that cleanly separates the Node.js services (API gateway, workers) from the Python AI services (RAG orchestrator, agent runtime), with shared configuration for linting, formatting, environment variables, and CI.

### Folder Structure

```
/
├── docs/                          # Assignment deliverables (already exist at root — symlink or move)
├── packages/
│   ├── api/                       # Node.js Fastify API Gateway
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── workers/                   # Node.js batch workers (profileBatchWorker, recommendationBatchWorker)
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── student-ui/                # Frontend web app
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── services/
│   └── ai/                        # Python FastAPI + LangGraph AI services
│       ├── app/
│       ├── pyproject.toml
│       └── requirements.txt
├── infra/
│   ├── docker/                    # Dockerfiles for each service
│   ├── docker-compose.yml         # Local development stack (Postgres, pgvector, SQS-local)
│   └── migrations/                # SQL migration files
├── dataset/                       # Provided dataset (teachers.json, new_students.json)
├── scripts/                       # Utility scripts (seed, test-runner, etc.)
├── .env.example                   # Environment variable template
├── .gitignore
├── .eslintrc.js                   # Shared ESLint config for Node.js packages
├── .prettierrc                    # Shared Prettier config
├── turbo.json / nx.json           # Monorepo task runner (Turborepo or Nx)
├── package.json                   # Root workspace manifest
└── README.md
```

## Acceptance Criteria

- [ ] Root `package.json` declares workspaces for `packages/api`, `packages/workers`, `packages/student-ui`.
- [ ] Each Node.js package has its own `package.json` with `typescript`, `eslint`, `prettier` as dev dependencies.
- [ ] `services/ai/` has a `pyproject.toml` or `requirements.txt` with `fastapi`, `uvicorn`, `langchain`, `langgraph`, `pgvector`, `psycopg2-binary` pinned.
- [ ] `infra/docker-compose.yml` starts PostgreSQL 16 with pgvector extension, a local SQS substitute (e.g. ElasticMQ or LocalStack), and exposes standard ports.
- [ ] Running `docker compose up` from project root brings up Postgres and the queue with no errors.
- [ ] `.env.example` contains all required environment variables (`DATABASE_URL`, `QUEUE_URL`, `LLM_API_KEY`, `EMBEDDING_MODEL`, etc.) with placeholder values.
- [ ] `.gitignore` covers `node_modules/`, `__pycache__/`, `.env`, `dist/`, `.venv/`.
- [ ] A root `README.md` exists with setup instructions, prerequisites, and links to the assignment deliverables.
- [ ] Linting passes with zero errors on the empty scaffold (`npm run lint` and `ruff check` for Python).

## Technical Details

### Node.js Stack

- **Runtime:** Node.js >= 20 LTS
- **Package manager:** pnpm (workspace support)
- **TypeScript:** Strict mode enabled
- **Linting:** ESLint + Prettier
- **Monorepo runner:** Turborepo (`turbo.json`) for parallel build/lint/test

### Python Stack

- **Runtime:** Python >= 3.11
- **Package manager:** pip + venv (or Poetry)
- **Linting:** ruff
- **Type checking:** mypy (optional, recommended)

### Docker Compose Services

| Service | Image | Port | Purpose |
|---|---|---|---|
| `postgres` | `pgvector/pgvector:pg16` | 5432 | Metadata DB + vector store |
| `queue` | `softwaremill/elasticmq` or `localstack/localstack` | 9324 | Local SQS substitute |

### Environment Variables (`.env.example`)

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/coaching_rec
QUEUE_URL=http://localhost:9324
LLM_API_KEY=sk-placeholder
LLM_MODEL=gpt-4o
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSION=1536
NODE_ENV=development
LOG_LEVEL=debug
```

## Dependencies

None — this is the first ticket.

## Test Plan

### Unit Tests
- **Lint config validation:** Run ESLint on a sample `.ts` file with intentional errors; verify errors are reported. Run on a clean file; verify zero errors.
- **Prettier config validation:** Run Prettier `--check` on a sample file; verify formatting rules are applied.
- **Ruff config validation:** Run `ruff check` on a sample `.py` file; verify Python linting rules are active.
- **Dependency manifest parsing:** Verify `package.json` (root and each workspace) parses without errors via `npm pack --dry-run` or `pnpm install --frozen-lockfile`.
- **`pyproject.toml` / `requirements.txt` parsing:** Verify `pip install --dry-run -r requirements.txt` succeeds without unresolvable dependencies.

### Integration Tests
- **CI skeleton smoke test:** Push a dummy commit to the repo and verify the CI pipeline runs all stages (lint, type-check, test placeholder) and passes.
- **Docker Compose startup:** Run `docker compose up -d` and verify both `postgres` and `queue` containers reach healthy status within 30 seconds.
- **Cross-workspace dependency resolution:** Run `pnpm install` at root and verify all workspace packages (`api`, `workers`, `student-ui`) resolve without peer dependency warnings.

### E2E / Manual Tests
- **Clean clone install:** Clone the repo into a fresh directory, run `pnpm install` and `pip install -r services/ai/requirements.txt`, verify zero errors.
- **Docker Compose full stack:** Run `docker compose up`, connect to Postgres on port 5432 with `psql`, verify the database accepts connections. Verify ElasticMQ/LocalStack responds on port 9324.
- **README verification:** Open `README.md` in a markdown renderer; confirm setup instructions, prerequisites, and deliverable links render correctly.
- **Environment template:** Copy `.env.example` to `.env`; verify all placeholder values are present and no variable is missing.

### Requirement Coverage Matrix
| Acceptance Criterion | Test Type | Test Description |
|---|---|---|
| AC: Root package.json declares workspaces | Integration | Cross-workspace dependency resolution passes |
| AC: Node packages have TS/ESLint/Prettier | Unit | Lint and Prettier config validation on sample files |
| AC: Python service has pinned dependencies | Unit | requirements.txt dry-run install succeeds |
| AC: Docker Compose starts Postgres + queue | Integration | Docker Compose startup health check |
| AC: `.env.example` has all required vars | E2E/Manual | Environment template verification |
| AC: README exists with setup instructions | E2E/Manual | README renders correctly with all sections |
| AC: Linting passes on empty scaffold | Unit | ESLint + ruff config validation with clean files |

## Dataset References

- `dataset/teachers.json` and `dataset/new_students.json` remain in the `dataset/` folder at project root. They are consumed by seed scripts created in TICKET-001.
