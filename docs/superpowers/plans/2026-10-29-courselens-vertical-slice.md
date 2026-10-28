# CourseLens Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a credential-free, persistent CourseLens vertical slice covering course creation, document indexing, cited questions, and triggerable honest demo media jobs.

**Architecture:** A pnpm workspace contains a Next.js App Router web app, a FastAPI API, a Python worker, and shared contract/configuration packages. The API owns authorization and orchestration; SQLite-backed local repositories and demo providers preserve the interfaces needed for future Supabase, hosted AI, Redis/Celery, and media integrations.

**Tech Stack:** Next.js, React, TypeScript strict mode, Tailwind CSS, FastAPI, Pydantic, Python, SQLite, pytest, Vitest/Testing Library, pnpm.

**Spec:** [docs/superpowers/specs/2026-10-29-courselens-vertical-slice-design.md](../specs/2026-10-29-courselens-vertical-slice-design.md)

## Global Constraints

- “Accuracy and useful evidence over impressive but unsupported answers.”
- “Do not perform expensive transcription or rendering synchronously inside ordinary HTTP requests.”
- “Never assume that API credentials are already available.”
- “Do not falsely claim that real AI processing or real video generation has occurred.”
- “All sensitive data access must be enforced by the backend and database policies.”
- “Secrets must remain server-side and `.env.example` must document required variables.”
- Use an explicit, clearly labeled local demo identity when credentials are absent.
- Local data must persist across application restarts.

## Review Focus

- A question submitted before indexing completes must produce an explicit processing/demo state, never fabricated citations. Test in API question-service tests.
- A question with no relevant chunks must produce a labeled demo answer with zero fabricated citations. Test in API retrieval tests.
- A document belonging to another course/user must be rejected even when its ID is supplied directly. Test in repository/service authorization tests.
- Unsupported or oversized uploads must return a visible validation error and create no processing job. Test in API upload tests.
- Media job submission without configured providers must be triggerable but visibly resolve to provider-unavailable/demo status, not completed media. Test in media job tests and web state tests.

---

### Task 1: Scaffold workspace and shared contracts

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.json`
- Create: `.env.example`
- Create: `packages/contracts/src/index.ts`
- Create: `packages/contracts/package.json`
- Create: `apps/web/package.json`
- Create: `apps/api/pyproject.toml`
- Create: `apps/worker/pyproject.toml`
- Create: `README.md`
- Test: `packages/contracts/src/index.test.ts`

**Interfaces:**
- Produces TypeScript `Course`, `Document`, `Citation`, `QuestionResponse`, and `ProcessingJob` schemas/types shared by the web client and API boundary.
- Produces environment names for local persistence, API URL, demo mode, and future provider credentials.

- [ ] **Step 1: Write the failing contract test** asserting the exported schemas accept the documented response shape and reject a citation without `source_text`.
- [ ] **Step 2: Run `pnpm --filter @courselens/contracts test`; expect failure because the package does not exist.**
- [ ] **Step 3: Implement the workspace manifests, strict TypeScript configuration, Zod schemas/types, Python project metadata, and `.env.example`.**
- [ ] **Step 4: Run `pnpm --filter @courselens/contracts test`; expect PASS.**
- [ ] **Step 5: Commit with `git add . && git commit -m "chore: scaffold courselens workspace"` and include the required Copilot co-author trailer.**

### Task 2: Build persistent local repositories and demo identity

**Files:**
- Create: `apps/api/app/config.py`
- Create: `apps/api/app/db.py`
- Create: `apps/api/app/models.py`
- Create: `apps/api/app/repositories/courses.py`
- Create: `apps/api/app/repositories/documents.py`
- Create: `apps/api/app/repositories/questions.py`
- Create: `apps/api/app/repositories/jobs.py`
- Create: `apps/api/app/auth.py`
- Test: `apps/api/tests/test_repositories.py`

**Interfaces:**
- `get_current_user() -> User`: returns the configured local demo identity.
- `CourseRepository.create(user_id: str, name: str, description: str | None) -> Course`.
- `DocumentRepository.create(course_id: str, filename: str, content: str, content_type: str) -> Document`.
- `DocumentRepository.list_for_course(course_id: str) -> list[Document]`.
- `QuestionRepository.create(course_id: str, text: str) -> Question`.
- `JobRepository.create(kind: str, resource_id: str) -> ProcessingJob`.
- Every repository method that accepts a course/resource ID verifies ownership through the database relationship.

- [ ] **Step 1: Write failing tests for persistence across repository re-instantiation, ownership rejection, and demo identity stability.**
- [ ] **Step 2: Run `cd apps/api && pytest tests/test_repositories.py -v`; expect failure.**
- [ ] **Step 3: Implement SQLite schema/bootstrap, repositories, Pydantic models, and local identity. Use a configured file path, not process memory.**
- [ ] **Step 4: Run the targeted pytest file; expect PASS, including a test that opens a second repository instance against the same database file.**
- [ ] **Step 5: Commit the repository layer.**

### Task 3: Implement document extraction, chunking, and retrieval

**Files:**
- Create: `apps/api/app/services/document_processing.py`
- Create: `apps/api/app/services/retrieval.py`
- Create: `apps/api/app/providers/embeddings.py`
- Create: `apps/api/app/providers/answering.py`
- Modify: `apps/api/app/repositories/documents.py`
- Test: `apps/api/tests/test_retrieval.py`

**Interfaces:**
- `extract_text(filename: str, content_type: str, raw: bytes) -> ExtractedDocument`.
- `chunk_document(document_id: str, text: str, max_chars: int = 1200) -> list[DocumentChunk]`.
- `retrieve(course_id: str, query: str, limit: int = 5) -> list[RetrievedChunk]`.
- `answer_question(question: str, chunks: list[RetrievedChunk]) -> AnswerResult`.

- [ ] **Step 1: Write failing tests for plain text/Markdown extraction, unsupported types, deterministic chunking with source sections, keyword retrieval, and no-context labeled demo answers with empty citations.**
- [ ] **Step 2: Run `cd apps/api && pytest tests/test_retrieval.py -v`; expect failure.**
- [ ] **Step 3: Implement the local extraction/chunking/retrieval adapter and demo answer provider. Keep provider protocols separate from their local implementations.**
- [ ] **Step 4: Run the targeted tests; expect PASS.**
- [ ] **Step 5: Commit the document/retrieval services.**

### Task 4: Add FastAPI course, document, question, and job routes

**Files:**
- Create: `apps/api/app/main.py`
- Create: `apps/api/app/routes/courses.py`
- Create: `apps/api/app/routes/documents.py`
- Create: `apps/api/app/routes/questions.py`
- Create: `apps/api/app/routes/jobs.py`
- Create: `apps/api/tests/test_api.py`

**Interfaces:**
- `POST /courses` creates a course.
- `GET /courses` lists only the current user’s courses.
- `POST /courses/{course_id}/documents` validates a supported upload and creates an indexing job.
- `GET /courses/{course_id}/documents` lists course documents and statuses.
- `POST /courses/{course_id}/questions` returns `QuestionResponse` with `answer_mode` (`grounded` or `demo`) and citations.
- `POST /courses/{course_id}/media-jobs` creates a triggerable lecture/animation job.
- `GET /jobs/{job_id}` returns status and explicit provider availability.

- [ ] **Step 1: Write failing API tests for course creation/listing, upload validation, indexing state, grounded/demo question responses, cross-course access rejection, and media job status.**
- [ ] **Step 2: Run `cd apps/api && pytest tests/test_api.py -v`; expect failure.**
- [ ] **Step 3: Implement FastAPI dependency wiring, route schemas, status codes, and service calls. Do not put parsing or media work directly in route handlers.**
- [ ] **Step 4: Run the targeted API tests; expect PASS, then run `cd apps/api && pytest -q`.**
- [ ] **Step 5: Commit the API routes.**

### Task 5: Implement worker entry points and local job transitions

**Files:**
- Create: `apps/worker/app/main.py`
- Create: `apps/worker/app/tasks/document_indexing.py`
- Create: `apps/worker/app/tasks/media_jobs.py`
- Create: `apps/worker/tests/test_tasks.py`
- Modify: `apps/api/app/repositories/jobs.py`

**Interfaces:**
- `index_document(job_id: str) -> ProcessingJob`.
- `run_media_job(job_id: str) -> ProcessingJob`.
- Job states are `queued`, `processing`, `completed`, `demo`, `provider_unavailable`, and `failed`; media jobs without provider credentials must end in `provider_unavailable` or `demo`, never `completed`.

- [ ] **Step 1: Write failing tests for document job transitions and provider-unavailable media jobs.**
- [ ] **Step 2: Run `cd apps/worker && pytest -q`; expect failure.**
- [ ] **Step 3: Implement worker tasks using the shared SQLite database and service interfaces; expose a local command that drains queued jobs without requiring Redis.**
- [ ] **Step 4: Run worker tests and an API-to-worker indexing smoke test; expect PASS.**
- [ ] **Step 5: Commit worker processing.**

### Task 6: Build the web workspace UI and typed API client

**Files:**
- Create: `apps/web/src/app/page.tsx`
- Create: `apps/web/src/app/courses/[courseId]/page.tsx`
- Create: `apps/web/src/components/course-sidebar.tsx`
- Create: `apps/web/src/components/document-uploader.tsx`
- Create: `apps/web/src/components/grounded-answer.tsx`
- Create: `apps/web/src/components/media-job-panel.tsx`
- Create: `apps/web/src/lib/api-client.ts`
- Create: `apps/web/src/lib/types.ts`
- Create: `apps/web/src/app/globals.css`
- Test: `apps/web/src/components/course-workspace.test.tsx`

**Interfaces:**
- `apiClient.listCourses()`, `createCourse(name, description)`, `uploadDocument(courseId, file)`, `askQuestion(courseId, text)`, `createMediaJob(courseId, kind, prompt)`, and `getJob(jobId)`.
- The workspace renders course selection, upload/indexing status, cited answers, demo-answer labels, and media provider-unavailable states.

- [ ] **Step 1: Write failing component tests for course creation/selection, upload errors, citation rendering, demo-answer labeling, and media job status.**
- [ ] **Step 2: Run `pnpm --filter @courselens/web test`; expect failure.**
- [ ] **Step 3: Implement the typed client and accessible responsive UI using the repository’s chosen styling primitives; use honest copy for demo/provider states and no fake completed media controls.**
- [ ] **Step 4: Run the targeted component tests; expect PASS.**
- [ ] **Step 5: Commit the web workspace.**

### Task 7: Connect local development, documentation, and end-to-end verification

**Files:**
- Create: `apps/api/README.md`
- Create: `apps/worker/README.md`
- Create: `infra/docker-compose.yml`
- Create: `scripts/dev.sh`
- Modify: `README.md`
- Modify: `.env.example`
- Test: `apps/api/tests/test_end_to_end.py`

**Interfaces:**
- `scripts/dev.sh` starts the web app, API, and local worker mode with documented prerequisites.
- The README documents credential-free demo limitations and the future provider configuration seams.

- [ ] **Step 1: Write an end-to-end test that creates a course, uploads text, drains indexing, asks a grounded question, and triggers a media job.**
- [ ] **Step 2: Run the test before final wiring; expect failure or missing-service errors.**
- [ ] **Step 3: Add startup configuration, CORS/API URL wiring, local worker command, optional Redis/Supabase service placeholders, and setup documentation.**
- [ ] **Step 4: Run `cd apps/api && pytest -q`, `cd apps/worker && pytest -q`, and `pnpm -r test`; expect PASS. Run web/API type checks and build commands from the package manifests.**
- [ ] **Step 5: Commit documentation and local orchestration.**

### Task 8: Final review and verification

**Files:**
- Modify only files required by verification findings.

- [ ] **Step 1: Run `git diff --check` and inspect `git status --short`.**
- [ ] **Step 2: Run all targeted tests, full package tests, type checks, and web build.**
- [ ] **Step 3: Manually verify the demo path: persistent data survives restart, unsupported upload errors are visible, grounded citations quote stored source text, demo answers have no citations, and media jobs do not claim provider-backed completion.**
- [ ] **Step 4: Fix only failures caused by this implementation and rerun the failing command.**
- [ ] **Step 5: Commit any verification fixes with the required co-author trailer.**
