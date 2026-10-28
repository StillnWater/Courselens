# CourseLens Vertical Slice Design

## Intent

CourseLens is a course-centered learning workspace for students. The first
release should prove the core loop: a student creates a course, uploads an
academic document, asks a question grounded in that document, and receives an
answer with source citations. It should also establish honest product surfaces
for lecture search/clipping and animated lessons without claiming that
provider-backed processing happened when providers are not configured.

## Goals

- Deliver one usable end-to-end course/document/cited-question workflow.
- Establish the requested Next.js, FastAPI, and worker service boundaries.
- Keep external AI, storage, authentication, and media providers behind
  replaceable interfaces.
- Make development possible without credentials using explicit demo adapters.
- Preserve a path to Supabase, pgvector, Redis/Celery, FFmpeg, transcription,
  TTS, and Remotion integrations.
- Provide clear local setup, environment placeholders, and tests for the core
  workflow.

## Non-goals for this slice

- Production billing, multi-tenant administration, or team collaboration.
- Full provider integrations for every AI/media service.
- Production-grade video rendering, OCR, or background processing.
- Foundation-model training.
- Pretending demo data or adapters are real provider results.

## Anti-goals

- Decorative buttons that imply unavailable functionality.
- Frontend-only access control.
- Coupling application logic directly to a single AI or storage vendor.
- Synchronous HTTP handling of expensive media work.

## Constraints

- The repository is empty.
- TypeScript should be strict; Python APIs should use Pydantic validation.
- The architecture should match the requested stack unless an incompatibility
  is discovered during implementation.
- Secrets must remain server-side and `.env.example` must document required
  variables.
- The first slice should remain runnable locally without external credentials.

## Approaches considered

### Next.js full-stack MVP

This would minimize setup and reach a demo quickly, but it would not establish
the requested FastAPI/worker boundaries and would make later extraction risky.

### Frontend-first prototype

This would optimize visual polish, but it would encourage mock interactions and
defer the data, authorization, and retrieval seams that make CourseLens useful.

### Service-oriented monorepo (selected)

Use a Next.js App Router frontend, a FastAPI API, and a worker package with
shared contracts/configuration. Implement local adapters for persistence,
embeddings, answer generation, and media jobs while defining interfaces that
can later map to Supabase, pgvector, hosted models, Redis/Celery, FFmpeg, and
Remotion. This costs more initial setup but best balances a working slice and
the requested production direction.

## Proposed architecture

```text
apps/web       Next.js App Router, Tailwind, accessible UI
apps/api       FastAPI routes, auth boundary, service orchestration
apps/worker    Background task entry points and provider adapters
packages/      Shared schemas, API types, and configuration contracts
infra/         Local development and future deployment configuration
docs/          Setup and architecture documentation
```

The browser communicates with the API through a typed client. The API owns
course/document/question authorization and orchestration. The worker owns
long-running document and media operations. A local development repository
implements the same service interfaces as the future Supabase-backed
repository, so the UI and route contracts do not change when infrastructure is
swapped.

## Core workflow

1. The student opens the workspace and creates or selects a course.
2. The student uploads a supported document with a course association.
3. The API validates metadata and stores the document through the storage
   abstraction; indexing is represented as a worker job.
4. The worker extracts text, creates chunks, and records source metadata using
   the local/demo retrieval adapter.
5. The student asks a course-scoped question.
6. The API retrieves relevant chunks, invokes the answer provider, and returns
   an answer plus citation records containing document name, page/section, and
   quoted source text.
7. If retrieval is incomplete or has no relevant chunks, the configured demo
   answer provider may return a clearly labeled demo answer; the response
   must distinguish it from a source-grounded answer and include no fabricated
   citations.
8. The UI presents loading, empty, unsupported-provider, demo-answer, and error
   states explicitly.
9. Lecture search/clip and animated-lesson jobs are user-triggerable. Without
   configured providers they move through honest demo/provider-unavailable
   states and never fabricate completed media.

## Data and security model

The initial entities are `User`, `Course`, `Document`, `DocumentChunk`,
`Question`, `Citation`, and `ProcessingJob`. Every course-scoped read and write
must carry an authenticated user identity and verify ownership in the API
service layer. The future Supabase schema should mirror these ownership
constraints with RLS; local development must use the same authorization checks.

## Verification

- API unit tests for ownership checks, document validation, retrieval, and
  citation shape.
- Frontend tests for course selection, upload/indexing states, cited answers,
  and honest demo-mode media states.
- Type checking/linting for the web and API-specific test/type commands.
- A documented local run path that works without provider credentials.

## Decisions confirmed during design review

- Local data must persist across application restarts.
- Media search, clipping, and animated-lesson jobs must be triggerable in the
  first slice, even when they resolve to explicit demo/provider-unavailable
  states.
- A question without usable indexed context may receive a clearly labeled demo
  answer, but it may not receive fabricated source citations.

## Builder decisions still open

- Exact package manager and monorepo tooling: choose the smallest stable setup
  compatible with the environment.
- Local persistence format: choose a lightweight implementation that preserves
  the repository interfaces and is easy to replace.
- Authentication in the credential-free slice: use an explicit local demo
  identity, clearly labeled and isolated from production auth.
- Document support in the first slice: start with text-readable PDF and plain
  text/Markdown if practical; reject unsupported files with visible errors.
