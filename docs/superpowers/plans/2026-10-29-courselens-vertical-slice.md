# CourseLens Vertical-Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a runnable CourseLens vertical slice where a signed-in student creates a course, uploads and locally indexes a PDF or text document, asks a course-scoped question, and receives an honest cited answer.

**Architecture:** Use a small workspace containing a Next.js App Router web app, a FastAPI API, a Celery-compatible worker, shared contracts, and Supabase-compatible migrations. Keep extraction, retrieval, embeddings, answer generation, storage, and auth behind explicit interfaces so demo adapters and configured providers share the same route behavior.

**Tech Stack:** Next.js, React, TypeScript strict mode, Tailwind CSS, accessible UI primitives, Motion, FastAPI, Pydantic, Python, PostgreSQL/Supabase-compatible SQL with pgvector/RLS, Redis/Celery-compatible jobs, PyMuPDF, pytest, and frontend/e2e test tooling selected during setup.

**Spec:** `docs/superpowers/specs/2026-10-29-courselens-vertical-slice-design.md`

## Global Constraints

- “Accuracy and useful evidence over impressive but unsupported answers.”
- “Do not falsely claim that real AI processing or real video generation has occurred.”
- “Do not perform expensive transcription or rendering synchronously inside ordinary HTTP requests.”
- “All sensitive data access must be enforced by the backend and database policies.”
- “Never commit actual credentials or expose server secrets to the browser.”
- Demo mode must genuinely extract and index uploaded PDFs locally.
- No-retrieval responses must be clearly labelled no-evidence responses without invented citations.
- Configured credentials must route through real embedding and generation providers.
- The first slice includes sign-up/sign-in and cross-user isolation coverage.

## Review Focus

- Malformed or oversized uploads must be rejected before persistence and expose an explicit error; pin in the document upload API test.
- A document that is queued, processing, failed, or has no relevant chunks must never produce a fabricated cited answer; pin in worker/chat tests.
- Provider credentials present versus absent must select real versus demo adapters without leaking configuration; pin in provider-selection tests.
- A user attempting to read or mutate another user’s course resources must receive denial from both API ownership checks and RLS; pin in integration/RLS tests.
- Refreshing or retrying a job must not duplicate chunks or corrupt status; pin in ingestion idempotency tests.

---

### Task 1: Workspace and developer foundation

**Files:**
- Create: `package.json`, workspace configuration, `apps/web/`, `apps/api/`, `apps/worker/`, `packages/contracts/`, `infra/supabase/`
- Create: `.env.example`, `.gitignore`, `README.md`
- Create: service Docker/local orchestration configuration
- Test: startup smoke checks for web, API health, and worker import

**Interfaces:**
- Produces commands for installing dependencies, starting services, running tests, and formatting.
- Produces environment names for database, auth, storage, Redis, LLM, embedding, and app URLs.

- [ ] **Step 1: Write failing startup smoke checks** asserting API health shape, web route availability, and worker task import.
- [ ] **Step 2: Run the smoke checks** and confirm failure because the workspace does not exist.
- [ ] **Step 3: Scaffold the workspace** with strict TypeScript, FastAPI/Pydantic, worker entrypoint, shared package boundaries, local orchestration, `.env.example`, and README setup/demo limitations.
- [ ] **Step 4: Run the smoke checks** and confirm all services load.
- [ ] **Step 5: Commit** with `chore: scaffold courselens workspace`.

### Task 2: Database schema, auth boundary, and isolation policies

**Files:**
- Create: `infra/supabase/migrations/*_initial_schema.sql`
- Create: `apps/api/app/auth/ports.py`, `apps/api/app/auth/demo.py`, `apps/api/app/auth/supabase.py`
- Create: `apps/api/tests/test_auth_and_isolation.py`

**Interfaces:**
- `AuthProvider.sign_up(email: str, password: str) -> AuthUser`
- `AuthProvider.sign_in(email: str, password: str) -> Session`
- `CurrentUser` exposes `id: UUID` and `email: str`.
- Tables expose ownership fields for courses, documents, chunks, conversations, messages, citations, and processing jobs.

- [ ] **Step 1: Write failing tests** for sign-up/sign-in and denial when user B accesses user A’s course and document IDs.
- [ ] **Step 2: Run the tests** and confirm missing schema/auth behavior.
- [ ] **Step 3: Add migrations and auth adapters** with explicit status/check constraints, indexes, pgvector-ready chunk fields, RLS policies, and server-side ownership helpers.
- [ ] **Step 4: Run the auth/isolation tests** against the local database and confirm pass.
- [ ] **Step 5: Commit** with `feat: add auth schema and ownership isolation`.

### Task 3: Course and document lifecycle API

**Files:**
- Create: `apps/api/app/courses/routes.py`, `apps/api/app/courses/service.py`
- Create: `apps/api/app/documents/routes.py`, `apps/api/app/documents/service.py`
- Create: `apps/api/app/storage/ports.py`, local/private storage adapter
- Create: `apps/api/tests/test_courses_documents.py`

**Interfaces:**
- `POST /courses` creates a course owned by the current user.
- `GET /courses` lists only the current user’s courses.
- `POST /courses/{course_id}/documents` accepts PDF/plain text metadata and content upload.
- `GET /courses/{course_id}/documents` returns document status and error details.
- Upload limits and supported MIME types are centralized validation constants.

- [ ] **Step 1: Write failing tests** for course creation/listing, valid PDF/text uploads, invalid type/size rejection, and foreign-course denial.
- [ ] **Step 2: Run the targeted tests** and confirm failure.
- [ ] **Step 3: Implement typed request/response models, ownership checks, private storage writes, document records, and `queued` job creation.
- [ ] **Step 4: Run the targeted tests** and confirm pass, including the malformed/oversized review-focus case.
- [ ] **Step 5: Commit** with `feat: add course and document lifecycle`.

### Task 4: Local document extraction, chunking, and job processing

**Files:**
- Create: `apps/api/app/ingestion/ports.py`, `extractors.py`, `chunking.py`
- Create: `apps/worker/app/tasks/ingest_document.py`
- Create: `apps/api/tests/test_ingestion.py`, `apps/worker/tests/test_ingest_document.py`

**Interfaces:**
- `DocumentExtractor.extract(path: str) -> list[ExtractedPage]`
- `Chunker.chunk(pages: list[ExtractedPage]) -> list[DocumentChunkInput]`
- `ingest_document(document_id: UUID) -> None`
- Status transitions are `queued -> processing -> ready|failed`, with structured error details.

- [ ] **Step 1: Write failing tests** using a small PDF fixture for page text/source locations, deterministic chunking, successful status transition, failure status, and retry idempotency.
- [ ] **Step 2: Run the tests** and confirm failure.
- [ ] **Step 3: Implement PyMuPDF extraction, plain-text extraction, bounded chunking, persistence, and idempotent worker retry behavior.
- [ ] **Step 4: Run ingestion tests** and confirm pass.
- [ ] **Step 5: Commit** with `feat: add local document ingestion`.

### Task 5: Retrieval and answer-provider interfaces

**Files:**
- Create: `apps/api/app/ai/ports.py`, `provider_factory.py`
- Create: `apps/api/app/ai/local_retriever.py`, `local_answerer.py`
- Create: `apps/api/app/ai/embedding_provider.py`, `llm_provider.py`
- Create: `apps/api/tests/test_ai_providers.py`, `tests/fixtures/course.pdf`

**Interfaces:**
- `Retriever.search(user_id: UUID, course_id: UUID, query: str, limit: int) -> list[RetrievedChunk]`
- `Answerer.answer(query: str, context: list[RetrievedChunk]) -> AnswerResult`
- `ProviderFactory.from_settings(settings: Settings) -> ProviderBundle`
- `AnswerResult` contains `text`, `mode`, and structured citations with document title and page/location.

- [ ] **Step 1: Write failing tests** for local retrieval, no-context no-evidence output, configured-provider selection, and citation source integrity.
- [ ] **Step 2: Run the tests** and confirm failure.
- [ ] **Step 3: Implement local lexical/vector-ready retrieval and answer generation, provider interfaces, settings-based real-provider selection, and explicit provider error handling.
- [ ] **Step 4: Run provider tests** and confirm pass without requiring credentials; run configured-provider contract tests when credentials are available.
- [ ] **Step 5: Commit** with `feat: add course-aware AI provider boundary`.

### Task 6: Chat API and conversation persistence

**Files:**
- Create: `apps/api/app/chat/routes.py`, `apps/api/app/chat/service.py`
- Create: `apps/api/tests/test_chat.py`
- Modify: shared contracts for chat request/response types

**Interfaces:**
- `POST /courses/{course_id}/conversations` creates a conversation.
- `POST /conversations/{conversation_id}/messages` accepts `{content: str}` and returns answer text, mode, citations, and message IDs.
- Chat rejects inaccessible courses/conversations and returns a labelled `no_evidence` result when retrieval is empty.

- [ ] **Step 1: Write failing tests** for cited answers, conversation history, processing-state/no-evidence behavior, malformed messages, and foreign conversation denial.
- [ ] **Step 2: Run the targeted tests** and confirm failure.
- [ ] **Step 3: Implement chat orchestration, persistence, citation linking, validation, and explicit error responses.
- [ ] **Step 4: Run chat tests** and confirm pass.
- [ ] **Step 5: Commit** with `feat: add cited course chat`.

### Task 7: Public landing page and authenticated workspace

**Files:**
- Create: `apps/web/app/(marketing)/page.tsx`
- Create: `apps/web/app/(auth)/sign-in/page.tsx`, `sign-up/page.tsx`
- Create: `apps/web/app/(app)/dashboard/page.tsx`, `courses/[courseId]/page.tsx`
- Create: `apps/web/components/*`, `apps/web/lib/api-client.ts`, auth/session utilities
- Create: frontend component and route tests

**Interfaces:**
- Typed client methods mirror Tasks 2, 3, and 6.
- UI states expose `queued`, `processing`, `ready`, `failed`, `demo`, and `no evidence` explicitly.

- [ ] **Step 1: Write failing tests** for landing CTA, sign-in/sign-up validation, empty dashboard, course creation, upload status, cited chat rendering, and no-evidence rendering.
- [ ] **Step 2: Run frontend tests** and confirm failure.
- [ ] **Step 3: Implement responsive accessible pages using the CourseLens visual system, real API client calls, session handling, polling for document status, and no fake success interactions.
- [ ] **Step 4: Run frontend tests and accessibility checks** and confirm pass.
- [ ] **Step 5: Commit** with `feat: add courselens web experience`.

### Task 8: End-to-end acceptance, documentation, and release verification

**Files:**
- Create: `tests/e2e/course-learning-flow.*`
- Modify: `README.md`, API/web test configuration, CI configuration

**Interfaces:**
- The documented command starts the required local services in demo mode.
- E2E flow supports two independent signed-in users and a PDF fixture.

- [ ] **Step 1: Write failing E2E scenarios** for the full upload-to-cited-chat path, no-evidence response, and cross-user isolation.
- [ ] **Step 2: Run E2E tests** and confirm failures identify missing integration wiring.
- [ ] **Step 3: Wire fixtures, service startup, migrations, worker execution, and test cleanup; document provider setup and demo limitations.
- [ ] **Step 4: Run the smallest full verification set**: API tests, worker tests, frontend tests, E2E tests, type-check, lint, and documented startup smoke test.
- [ ] **Step 5: Commit** with `test: verify courselens vertical slice`.

## Self-review coverage

The plan covers the selected slice’s marketing/auth surfaces, course and
document lifecycle, local PDF extraction/indexing, asynchronous processing,
retrieval and real-provider seams, cited chat, RLS/API isolation, honest demo
states, responsive UI, documentation, and acceptance tests. Lecture clipping
and animated lessons remain intentionally deferred as specified rather than
represented by fake routes.
