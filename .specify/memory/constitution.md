<!--
Sync Impact Report
==================
Version change: N/A → 1.0.0
Modified principles: (initial ratification — all principles new)
  - I. Pipeline-First Architecture (new)
  - II. Async-First Processing (new)
  - III. AI Integration Discipline (new)
  - IV. Type Safety & Validation (new)
  - V. Observability & Resilience (new)
Added sections:
  - Core Principles (5 principles)
  - Technology Constraints
  - Development Workflow
  - Governance
Removed sections: none
Templates requiring updates:
  - .specify/templates/plan-template.md — ✅ no changes needed (Constitution Check section already generic)
  - .specify/templates/spec-template.md — ✅ no changes needed (requirements align with principles)
  - .specify/templates/tasks-template.md — ✅ no changes needed (task phases align with workflow)
  - .specify/templates/agent-file-template.md — ✅ no changes needed
  - .specify/templates/checklist-template.md — ✅ no changes needed
Follow-up TODOs: none
-->

# AutoClip Constitution

## Core Principles

### I. Pipeline-First Architecture

Every processing feature MUST be implemented as a discrete pipeline step.
Pipeline steps MUST be independently executable, independently testable,
and composable via the orchestration service. No step MAY directly invoke
another step; all coordination MUST go through the processing service.
Each step MUST define clear input/output contracts (Pydantic models on the
backend, typed interfaces on the frontend).

Rationale: The AI video pipeline (outline → timeline → scoring → video
generation) is the core value of AutoClip. Decoupled steps enable
parallel development, easier debugging, and the ability to re-run or skip
individual steps without redoing the entire pipeline.

### II. Async-First Processing

All I/O-bound operations MUST use async/await on the backend. Long-running
tasks (video download, AI analysis, clip generation) MUST be dispatched as
Celery tasks — never executed synchronously in a request handler. Real-time
progress MUST be communicated to the frontend via WebSocket. API endpoints
MUST return immediately after queuing work, providing a task identifier for
status tracking.

Rationale: Video processing and AI calls are inherently slow (seconds to
minutes). Blocking request handlers would make the system unusable under
concurrent load. Async patterns are already established in the codebase and
MUST be preserved.

### III. AI Integration Discipline

All AI/LLM interactions MUST go through the centralized `llm_client`. Direct
API calls to DashScope or any LLM provider from business logic are
prohibited. AI calls MUST implement retry logic with exponential backoff and
configurable timeouts. Prompts MUST be maintained as versioned, reviewable
artifacts — never embedded inline in business logic. AI-generated results
MUST be validated against expected schemas before acceptance. Cost-sensitive
operations (e.g., full-video analysis) MUST be gated behind explicit user
action.

Rationale: LLM calls are expensive, non-deterministic, and a common source
of failures. Centralization enables monitoring, cost control, model
swapping, and consistent error handling.

### IV. Type Safety & Validation

Backend: All API request/response schemas MUST be Pydantic models with
explicit type annotations. No untyped `dict` may cross an API boundary.
Frontend: TypeScript strict mode is mandatory. All API responses MUST be
typed; use Zod or equivalent for runtime validation of external data.
Database models MUST align with Pydantic schemas; drift between them is a
bug. Environment configuration MUST be validated at startup via the config
module — never accessed as raw `os.environ` outside `core/config.py`.

Rationale: AutoClip processes external data (video metadata, AI responses,
platform APIs) that is inherently unpredictable. Type safety at boundaries
prevents silent corruption and makes refactoring safe.

### V. Observability & Resilience

All pipeline steps and service methods MUST emit structured logs at
appropriate levels (DEBUG for flow, INFO for milestones, WARNING for
degraded operation, ERROR for failures). Processing MUST be resumable:
if a step fails, the system MUST be able to retry that step without
repeating earlier ones. WebSocket notifications MUST be the primary
mechanism for frontend progress updates; polling is acceptable only as a
fallback. Error states MUST surface actionable messages to the user, not
raw stack traces. Temporary files MUST be cleaned up on both success and
failure paths.

Rationale: Video processing involves long-running, multi-step operations
that are prone to partial failure. Observability enables debugging;
resumability prevents wasted time and compute; clean resource management
prevents storage leaks.

## Technology Constraints

- **Backend language**: Python 3.9+ with type hints enforced by ruff.
- **Backend framework**: FastAPI with async handlers; no synchronous
  blocking calls in route handlers.
- **Task queue**: Celery with Redis broker; all long-running work dispatched
  as Celery tasks.
- **Database**: SQLite for development; schema MUST be compatible with
  PostgreSQL for production migration.
- **AI provider**: DashScope (Qwen) via centralized `llm_client`; no direct
  provider SDK calls outside the client module.
- **Frontend framework**: React 18+ with TypeScript strict mode, Ant Design
  for UI, Zustand for state management.
- **Video processing**: FFmpeg exclusively; no other video processing
  libraries.
- **Video download**: yt-dlp for YouTube and Bilibili; platform-specific
  logic isolated in dedicated service modules.
- **Build & deploy**: Docker Compose for all environments; no bare-metal
  production deployments without containerization.

## Development Workflow

1. **Lint before commit**: Run `ruff check backend/` and `cd frontend &&
   npm run lint`. All warnings MUST be resolved before merge.
2. **Test requirements**: Run `pytest backend/` and `cd frontend && npm
   test`. New features MUST include corresponding tests.
3. **Commit format**: Conventional Commits (`feat(scope):`, `fix(scope):`,
   `docs:`, `refactor:`, `chore:`). Scope reflects the module affected.
4. **Branching**: Feature branches named `feature/<short-description>`.
   Main branch MUST always be in a deployable state.
5. **Code review**: All changes require review. Reviewers MUST verify
   constitution compliance — especially Principles I (pipeline steps), III
   (centralized AI calls), and IV (typed boundaries).
6. **Environment secrets**: API keys, credentials, and tokens MUST NEVER be
   committed. All secrets MUST be in `.env` (git-ignored) and accessed via
   `core/config.py`.
7. **Documentation**: Public APIs MUST have docstrings. Pipeline steps MUST
   document their input/output contracts. README updates are required for
   user-facing changes.

## Governance

This constitution is the authoritative source for architectural and
process decisions in the AutoClip project. In case of conflict between
this document and other guides, this constitution takes precedence.

**Amendment procedure**: Any change to principles or technology constraints
requires a written proposal, team review, and explicit approval. The
proposal MUST document: (a) the change, (b) the rationale, and (c) a
migration plan for existing code that violates the new rule.

**Versioning policy**: MAJOR version for principle removals or
backward-incompatible redefinitions. MINOR version for new principles or
materially expanded guidance. PATCH version for clarifications, wording
fixes, or non-semantic refinements.

**Compliance review**: Code reviews MUST include a constitution compliance
check. Complexity that violates simplicity (Principle V spirit) MUST be
justified in the PR description. Use `AGENTS.md` for runtime development
guidance; this constitution governs principles and constraints.

**Version**: 1.0.0 | **Ratified**: 2026-04-15 | **Last Amended**: 2026-04-15
