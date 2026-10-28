# CourseLens Vertical-Slice Design

## Intent

CourseLens is a course-centered learning workspace for students. The first
slice should prove the core loop: a student creates a course, uploads academic
material, sees honest processing state, asks a course-aware question, and
receives an answer with source citations. The product should already feel like
the intended platform through a polished public landing page and authenticated
workspace, while expensive lecture and animation workflows remain ready to
extend.

## Goals

- Establish a runnable monorepo with a Next.js frontend, FastAPI API, worker
  boundary, shared contracts, local development configuration, and `.env.example`.
- Build the public landing page and authenticated course workspace using the
  CourseLens visual direction from the master prompt.
- Implement course creation, private document upload metadata, processing
  states, document listing, and a course-aware chat flow.
- Provide a real retrieval/provider abstraction with a clearly labelled
  development adapter when external credentials are unavailable.
- Enforce server-side ownership checks and structure the data model for
  Supabase/Postgres, pgvector, storage, and RLS.
- Make the seams for lecture transcription/clipping and animated lessons
  explicit without claiming those workflows are implemented in this slice.

## Non-goals

- Production-grade lecture transcription, topic search, clip rendering, or
  animated lesson rendering in the first slice.
- Training or hosting a foundation model.
- Billing, institution administration, or production observability beyond
  the interfaces and basic error handling needed by the slice.
- Claiming live AI/media processing when providers are not configured.

## Anti-goals

- Decorative buttons that imply an action but do nothing.
- Frontend-only authorization.
- Synchronous request handlers for document extraction or indexing.
- Hard-coding a single AI vendor into product or route logic.
- Building broad placeholder screens at the expense of one reliable workflow.

## Constraints

- Use Next.js App Router, React, TypeScript strict mode, Tailwind, accessible
  primitives, and Motion where interaction benefits from it.
- Use FastAPI and Pydantic for the API, with a service boundary between routes,
  persistence, ingestion, retrieval, and AI providers.
- Use Supabase-compatible Postgres/Auth/Storage/pgvector abstractions and
  ownership policies; local development may use a documented adapter.
- Use Redis/Celery-compatible background task boundaries for ingestion.
- Keep secrets server-side and list every required variable in `.env.example`.
- Preserve an honest demo mode with visible limitations when provider
  configuration is absent.

## First-slice user experience

1. A visitor lands on the CourseLens marketing page and understands the
   course-centered promise, evidence-first answers, and future media workflows.
2. The visitor signs in or uses the local development auth adapter.
3. The dashboard shows an empty state that leads to creating a course.
4. The student creates a course with a name and optional description.
5. The student uploads a supported document. The UI shows queued,
   processing, ready, and failed states; upload and extraction are not hidden
   behind a false success state.
6. Once content is ready, the student asks a question in the course workspace.
7. The API retrieves relevant chunks through a provider interface and returns a
   structured answer with source title/page references. Demo mode genuinely
   extracts and indexes uploaded PDFs locally; configured credentials enable
   real embedding and generation providers.
8. The student can inspect the cited source list and prior messages.

## Architecture

### Repository shape

```text
apps/
  web/                 Next.js application
  api/                 FastAPI application
  worker/              Celery-compatible task entrypoint
packages/
  contracts/           Shared API schemas and generated client boundary
  ui/                  Shared presentation primitives where useful
infra/
  supabase/            migrations, seed data, RLS policies
docs/
  superpowers/specs/   approved designs
```

The exact package-manager workspace format is a builder decision, provided
that frontend, API, worker, and shared contracts can be run independently and
documented with one local development command.

### Request and processing flow

- The web app calls a typed API client; it never calls provider SDKs directly.
- The API authenticates the request, validates input, checks course ownership,
  persists records, and enqueues ingestion work.
- The worker extracts text, creates chunks/embeddings through interfaces, and
  updates explicit processing state and error details.
- Chat requests use the course-scoped retriever and answer provider, returning
  citations as structured data.
- Provider implementations live behind interfaces for embeddings, extraction,
  generation, and later speech/media operations.

### Data model

The initial migration should include users/profiles as needed by the chosen
auth adapter, courses, documents, document_chunks, conversations,
messages, citations, and processing_jobs. IDs, timestamps, ownership fields,
status enums, and provider metadata should be explicit. Document chunks should
reserve vector storage and source location fields for pgvector-backed search.
RLS policies and API ownership checks must both prevent cross-user access.

### Honest demo mode

Provider configuration is detected at startup or request time and exposed in
health/config diagnostics without leaking secrets. In demo mode:

- uploads persist metadata and use a local extractor/indexer for PDF and plain
  text documents;
- chat responses identify themselves as demo responses and cite the uploaded
  sources they used;
- unavailable operations return a visible, actionable unavailable state rather
  than a success-shaped placeholder.

If documents are still processing or failed, the UI explains that state. If
retrieval finds no relevant context, chat returns a clearly labelled
no-evidence response and does not invent citations or unsupported claims.
When credentials are configured, the same interfaces route to real embedding
and generation providers, with provider failures surfaced explicitly.

The first slice includes sign-up and sign-in against the selected auth
boundary. End-to-end tests create two users and verify that each can access
only its own course data.

## Approaches considered

### Vertical-slice MVP — selected

Build the marketing surface, auth/workspace shell, course/document lifecycle,
and cited chat path deeply, while defining stable interfaces for later media
features. This best satisfies the accuracy and real-functionality principles
with the available empty workspace and provides a useful acceptance target.

### Full scaffold in one pass

Create every route, worker, media pipeline, and provider interface immediately,
using demo adapters where needed. This maximizes surface coverage but raises
integration risk and makes it harder to distinguish working workflows from
future seams.

### Frontend-first prototype

Build the marketing page and authenticated workspace with mocked data before
the backend. This is fastest for visual validation but does not prove the
course-aware learning loop and conflicts with the requirement to prioritize
real functionality.

## Acceptance criteria

- A clean checkout can start the documented local services with demo
  configuration.
- A user can create a course, upload a supported document, observe truthful
  processing state, and ask a question in that course.
- The answer has structured citations tied to source metadata.
- A second user cannot read or mutate the first user's course, document,
  chunks, conversation, or messages through the API.
- Invalid uploads, missing courses, failed jobs, unavailable providers, and
  malformed chat requests produce explicit errors and useful UI states.
- Targeted backend, frontend, and end-to-end tests cover the acceptance flow
  and ownership isolation.
- The README explains setup, demo limitations, provider configuration, and
  the planned extension points for lecture and animation workflows.

## Builder calls still open

- Select a stable package manager and local service orchestration approach.
- Use PDF and plain text as the initial supported formats, with local PDF
  extraction/indexing in demo mode.
- Use sign-up/sign-in for the first slice while keeping the Supabase boundary.
- Choose whether shared contracts are generated from OpenAPI or maintained as
  a small explicitly versioned package.
- Keep lecture and animation work as the next independently planned phases,
  not hidden TODOs inside the first slice.
