# Tasks: KYC Retail Application Form

**Input**: Design documents from `specs/001-kyc-retail-form/`

**Prerequisites**:
- `spec.md` ✅ — Feature specification with 10 clarifications resolved (incl. FR-017, FR-012 timeline, notification channel, retry behaviors)
- `plan.md` ✅ — Full architecture (20 sections, 9 Mermaid diagrams); updated 2026-06-17
- `research.md` ✅ — 9 technology decisions
- `data-model.md` ✅ — ERD, state machine, schema design; updated with FR-017 ssn_hash index
- `contracts/kyc-service.yaml` ✅ — OpenAPI 3.1, 7 endpoints
- `contracts/document-service.yaml` ✅ — OpenAPI 3.1, 5 endpoints
- `contracts/notification-service.yaml` ✅ — OpenAPI 3.1, 2 endpoints (added 2026-06-17)
- `quickstart.md` ✅ — 12 validation scenarios

> **⚠ Admin Dashboard**: The KYC reviewer/admin dashboard is **out of scope** for this
> feature per `spec.md`. Review and approval flows are downstream processes. Admin tasks
> require a separate feature specification. See "Out of Scope" section at the end.

---

## Task Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (no shared file conflicts)
- **[Story]**: US1 / US2 / US3 / FOUND / SETUP

---

## Phase 1: Setup

**Purpose**: Monorepo scaffolding, tooling, and local development environment.

---

- [ ] T001 [P] Initialize monorepo project structure

  **Description**: Create the top-level directory layout for all packages — frontend, three
  backend services, AWS CDK infrastructure, and CI/CD workflows — as specified in
  plan.md §Project Structure. Initialize root package.json with npm workspaces.

  **Files**:
  - `package.json` (create — root workspace config)
  - `frontend/package.json` (create)
  - `services/kyc-service/package.json` (create)
  - `services/document-service/package.json` (create)
  - `services/notification-service/package.json` (create)
  - `infrastructure/package.json` (create)
  - `.gitignore` (create — node_modules, dist, .env, *.local)

  **Acceptance criteria**:
  - `npm install` at root installs all workspace dependencies
  - Directory structure matches plan.md §Project Structure exactly
  - Each workspace has its own `tsconfig.json` extending a shared base config

  **Dependencies**: —

---

- [ ] T002 [P] Configure frontend tooling (Vite, TypeScript, ESLint, Prettier)

  **Description**: Scaffold the React 18 + TypeScript 5 frontend using Vite. Configure
  ESLint (react, hooks, a11y plugins) and Prettier. Set up path aliases.

  **Files**:
  - `frontend/vite.config.ts` (create)
  - `frontend/tsconfig.json` (create)
  - `frontend/.eslintrc.cjs` (create)
  - `frontend/.prettierrc` (create)
  - `frontend/src/app/App.tsx` (create)
  - `frontend/index.html` (create)

  **Acceptance criteria**:
  - `npm run dev` in `frontend/` starts Vite dev server on port 5173
  - `npm run lint` passes with no errors on empty scaffold
  - `npm run typecheck` passes

  **Dependencies**: T001

---

- [ ] T003 [P] Configure backend service tooling (TypeScript, ESLint, Prettier — all three services)

  **Description**: Apply consistent TypeScript 5 + ESLint + Prettier configuration to all
  three services. Use a shared `tsconfig.base.json` at repo root. Configure `ts-node` /
  `tsx` for local development.

  **Files**:
  - `tsconfig.base.json` (create — shared strict TypeScript config)
  - `services/kyc-service/tsconfig.json` (create)
  - `services/document-service/tsconfig.json` (create)
  - `services/notification-service/tsconfig.json` (create)
  - `.eslintrc.base.cjs` (create — shared lint config)

  **Acceptance criteria**:
  - `npm run typecheck` passes in each service
  - ESLint runs clean on empty service scaffold
  - All services share strict TypeScript settings (no `any`, strict null checks)

  **Dependencies**: T001

---

- [ ] T004 [P] Docker Compose for local development

  **Description**: Create `docker-compose.yml` at repo root providing PostgreSQL 16 and
  LocalStack (for S3, KMS, SES simulation). Configure per-service `.env.local` templates.

  **Files**:
  - `docker-compose.yml` (create)
  - `services/kyc-service/.env.local.example` (create)
  - `services/document-service/.env.local.example` (create)
  - `services/notification-service/.env.local.example` (create)

  **Acceptance criteria**:
  - `docker compose up -d postgres localstack` starts without errors
  - PostgreSQL accessible on `localhost:5432`
  - LocalStack accessible on `localhost:4566`
  - S3 bucket `kyc-demo-identity-documents-local` created on LocalStack startup

  **Dependencies**: T001

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure all user stories depend on.

**⚠️ CRITICAL**: No user story implementation can begin until this phase is complete.

---

- [ ] T005 [FOUND] KYC Service — Fastify server skeleton with middleware

  **Description**: Initialize the Fastify 4 server for kyc-service. Register core middleware:
  JWT validation (jose), correlation ID injection, structured Winston logging, and the
  standard error handler that produces the `{code, message, details, correlationId}` shape.
  Add `/health` endpoint.

  **Files**:
  - `services/kyc-service/src/infrastructure/http/server.ts` (create)
  - `services/kyc-service/src/infrastructure/http/middleware/auth.ts` (create — JWT validation)
  - `services/kyc-service/src/infrastructure/http/middleware/correlation.ts` (create)
  - `services/kyc-service/src/infrastructure/http/middleware/errorHandler.ts` (create)
  - `services/kyc-service/src/shared/logging/logger.ts` (create — Winston structured)
  - `services/kyc-service/src/index.ts` (create — entry point)

  **Acceptance criteria**:
  - `GET /health` returns `{"status":"ok","service":"kyc-service"}`
  - Request with valid JWT proceeds; invalid JWT returns 401 with standard error shape
  - Every response includes `x-correlation-id` header
  - All log lines include `correlationId`, `service`, `level`, `timestamp`
  - SSN, JWT tokens never appear in logs

  **Dependencies**: T003

---

- [ ] T006 [P] [FOUND] Document Service — Fastify server skeleton with middleware

  **Description**: Mirror T005 for document-service. Same middleware stack (JWT, correlation
  ID, Winston, error handler). Add `/health` endpoint.

  **Files**:
  - `services/document-service/src/infrastructure/http/server.ts` (create)
  - `services/document-service/src/infrastructure/http/middleware/auth.ts` (create)
  - `services/document-service/src/infrastructure/http/middleware/correlation.ts` (create)
  - `services/document-service/src/infrastructure/http/middleware/errorHandler.ts` (create)
  - `services/document-service/src/shared/logging/logger.ts` (create)
  - `services/document-service/src/index.ts` (create)

  **Acceptance criteria**: Same criteria as T005 for document-service.

  **Dependencies**: T003

---

- [ ] T007 [P] [FOUND] Notification Service — Fastify server skeleton with middleware

  **Description**: Mirror T005 for notification-service. Add `/health` endpoint.

  **Files**:
  - `services/notification-service/src/infrastructure/http/server.ts` (create)
  - `services/notification-service/src/infrastructure/http/middleware/auth.ts` (create)
  - `services/notification-service/src/infrastructure/http/middleware/correlation.ts` (create)
  - `services/notification-service/src/infrastructure/http/middleware/errorHandler.ts` (create)
  - `services/notification-service/src/shared/logging/logger.ts` (create)
  - `services/notification-service/src/index.ts` (create)

  **Acceptance criteria**: Same criteria as T005 for notification-service.

  **Dependencies**: T003

---

- [ ] T008 [FOUND] KYC Service — Database migration: `kyc` schema

  **Description**: Create the Drizzle schema definition and initial migration for the `kyc`
  schema. Includes `kyc.applications` table with all columns, CHECK constraints, partial
  unique index, and secondary indexes per data-model.md §PostgreSQL Schema Design.

  **Files**:
  - `services/kyc-service/src/infrastructure/persistence/schema.ts` (create — Drizzle schema)
  - `services/kyc-service/src/infrastructure/persistence/migrations/0001_create_schema_and_applications.sql` (create)
  - `services/kyc-service/src/infrastructure/persistence/migrations/0002_add_partial_unique_index.sql` (create)
  - `services/kyc-service/drizzle.config.ts` (create)

  **Acceptance criteria**:
  - `npx drizzle-kit migrate` runs without errors on clean PostgreSQL database
  - `kyc.applications` table created with all columns from data-model.md
  - Partial unique index `uq_kyc_active_per_customer` enforced: INSERT two draft rows for
    same `customer_id` → second INSERT raises unique violation
  - All CHECK constraints validated (invalid status value rejected at DB level)

  **Dependencies**: T004, T005

---

- [ ] T009 [P] [FOUND] Document Service — Database migration: `documents` schema

  **Description**: Drizzle schema and migration for `documents.identity_documents` per
  data-model.md. Includes CHECK constraints on `document_type`, `file_format`, `upload_status`,
  and `file_size_bytes` (max 5,242,880).

  **Files**:
  - `services/document-service/src/infrastructure/persistence/schema.ts` (create)
  - `services/document-service/src/infrastructure/persistence/migrations/0001_create_schema_and_identity_documents.sql` (create)
  - `services/document-service/drizzle.config.ts` (create)

  **Acceptance criteria**:
  - Migration runs without errors on clean database
  - `documents.identity_documents` table matches data-model.md schema exactly
  - CHECK constraint rejects `document_type` values other than `us_drivers_license` / `us_passport`
  - CHECK constraint rejects `file_size_bytes` > 5,242,880

  **Dependencies**: T004, T006

---

- [ ] T010 [P] [FOUND] Notification Service — Database migration: `notifications` schema

  **Description**: Drizzle schema and migration for `notifications.notification_log` per
  data-model.md. Includes idempotency key unique constraint.

  **Files**:
  - `services/notification-service/src/infrastructure/persistence/schema.ts` (create)
  - `services/notification-service/src/infrastructure/persistence/migrations/0001_create_schema_and_notification_log.sql` (create)
  - `services/notification-service/drizzle.config.ts` (create)

  **Acceptance criteria**:
  - Migration runs without errors
  - `notifications.notification_log` table matches data-model.md
  - UNIQUE constraint on `idempotency_key` prevents duplicate notification records

  **Dependencies**: T004, T007

---

- [ ] T011 [P] [FOUND] Frontend — App scaffold (router, providers, auth, query client)

  **Description**: Wire up React Router v6, TanStack QueryClient provider, and Cognito auth
  context provider. Configure Axios instance with base URL and Authorization header
  injection. Set up MSW for dev/test environments.

  **Files**:
  - `frontend/src/app/App.tsx` (update — add router + providers)
  - `frontend/src/app/router.tsx` (create — route definitions: /kyc, /confirmation)
  - `frontend/src/app/providers.tsx` (create — QueryClientProvider + AuthProvider)
  - `frontend/src/shared/lib/queryClient.ts` (create)
  - `frontend/src/shared/lib/apiClient.ts` (create — Axios instance, Bearer token injection)
  - `frontend/src/shared/lib/auth.ts` (create — Cognito OIDC setup via oidc-client-ts)
  - `frontend/tests/mocks/server.ts` (create — MSW Node server for tests)

  **Acceptance criteria**:
  - Navigating to `/kyc` renders the KYC feature shell (even if empty)
  - Unauthenticated access to `/kyc` redirects to Cognito login
  - Axios instance automatically attaches `Authorization: Bearer <token>` on every request
  - `correlationId` from response headers is accessible to error handling

  **Dependencies**: T002

---

**Checkpoint**: Phase 2 complete — all three services running, databases migrated, frontend
scaffolded with auth. User story implementation can now begin.

---

## Phase 3: User Story 1 — Submit Personal Information (Priority: P1) 🎯 MVP

**Goal**: Customer fills 4 required KYC fields + optional US phone number, auto-saves draft,
and can resume or discard it on return visit.

**Independent Test**: quickstart.md Scenarios 1–4, 8–10

---

### Domain + Application Layer (kyc-service)

- [ ] T012 [P] [US1] KYC Service — Domain entities and value objects

  **Description**: Implement `KYCApplication` entity, `SSN` value object (format validation
  only — no KMS at this layer), `ApplicationStatus` value object (valid transition rules),
  and `ReferenceNumber` value object (`KYC-YYYYMMDD-NNNNNN` format generator).

  **Files**:
  - `services/kyc-service/src/domain/entities/KYCApplication.ts` (create)
  - `services/kyc-service/src/domain/value-objects/SSN.ts` (create)
  - `services/kyc-service/src/domain/value-objects/ApplicationStatus.ts` (create)
  - `services/kyc-service/src/domain/value-objects/ReferenceNumber.ts` (create)

  **Acceptance criteria**:
  - `SSN` rejects values not matching `/^\d{3}-\d{2}-\d{4}$/`
  - `ApplicationStatus` throws on invalid transitions (e.g., `pending → draft`)
  - `KYCApplication.canSubmit()` returns false when any required field is null
  - `KYCApplication.canSubmit()` returns false when `documentId` is null
  - No imports from Fastify, Drizzle, or AWS SDK in this layer

  **Dependencies**: T005

---

- [ ] T013 [P] [US1] KYC Service — Repository interface (port)

  **Description**: Define `IKYCApplicationRepository` TypeScript interface with methods:
  `findById`, `findActiveByCustomerId`, `findByCustomerId` (paginated), `save`, `delete`.

  **Files**:
  - `services/kyc-service/src/domain/repositories/IKYCApplicationRepository.ts` (create)

  **Acceptance criteria**:
  - Interface is a pure TypeScript contract (no framework imports)
  - Method signatures match the use cases that depend on them

  **Dependencies**: T012

---

- [ ] T014 [US1] KYC Service — CreateDraftApplicationUseCase

  **Description**: Use case that checks for an existing active application (via
  `findActiveByCustomerId`), returns it if found as a draft, raises `ConflictError` if
  active non-draft exists, or creates and saves a new draft.

  **Files**:
  - `services/kyc-service/src/application/use-cases/CreateDraftApplicationUseCase.ts` (create)
  - `services/kyc-service/src/application/dtos/CreateApplicationDto.ts` (create)
  - `services/kyc-service/src/shared/errors/ConflictError.ts` (create)

  **Acceptance criteria**:
  - If no existing application: creates draft, calls `repository.save()`
  - If existing draft: returns existing draft without creating new record
  - If existing pending/under_review: throws `ConflictError` with `applicationId` and
    `referenceNumber` in payload
  - No Fastify or PostgreSQL imports in this file

  **Dependencies**: T013

---

- [ ] T015 [US1] KYC Service — UpdateDraftApplicationUseCase

  **Description**: Partial update of draft fields. Validates ownership (`customerId` matches
  JWT sub). Validates SSN format, phone format (if provided), DOB age (≥ 18) at use-case
  level. Does NOT apply KMS encryption here — that is the repository's responsibility.

  **Files**:
  - `services/kyc-service/src/application/use-cases/UpdateDraftApplicationUseCase.ts` (create)
  - `services/kyc-service/src/application/dtos/UpdateApplicationDto.ts` (create)
  - `services/kyc-service/src/shared/errors/ValidationError.ts` (create)
  - `services/kyc-service/src/shared/errors/ForbiddenError.ts` (create)

  **Acceptance criteria**:
  - Throws `ForbiddenError` if `customerId` in application ≠ authenticated user's `sub`
  - Throws `ValidationError` for invalid SSN format (pattern `^\d{3}-\d{2}-\d{4}$`)
  - Throws `ValidationError` for invalid phone format when phone is non-empty
  - Age validation is NOT performed here — age ≥ 18 is enforced only at submission (T031, FR-003)
  - Status must be `draft` — throws `ConflictError` otherwise
  - Partial update: only provided fields are updated; others unchanged

  **Dependencies**: T013, T014

---

- [ ] T016 [P] [US1] KYC Service — GetActiveApplicationUseCase

  **Description**: Returns the customer's active application (status IN draft, pending,
  under_review). Returns `null` if no active application exists.

  **Files**:
  - `services/kyc-service/src/application/use-cases/GetActiveApplicationUseCase.ts` (create)

  **Acceptance criteria**:
  - Returns application for matching `customerId`
  - Returns `null` if no matching active application found
  - Never returns `rejected` applications

  **Dependencies**: T013

---

- [ ] T017 [US1] KYC Service — PostgreSQL repository implementation

  **Description**: Implement `KYCApplicationRepository` using Drizzle ORM against the `kyc`
  schema. Apply KMS encryption of SSN on `save()` and decryption on reads (via
  `KMSEncryptionService` injected as a dependency). Map DB rows to domain entities.

  **Files**:
  - `services/kyc-service/src/infrastructure/persistence/KYCApplicationRepository.ts` (create)
  - `services/kyc-service/src/infrastructure/persistence/mappers/applicationMapper.ts` (create)
  - `services/kyc-service/src/infrastructure/security/KMSEncryptionService.ts` (create)

  **Acceptance criteria**:
  - `save()` encrypts raw SSN via KMS before INSERT/UPDATE; stores as `BYTEA`
  - `findById()` decrypts SSN from KMS before returning domain entity
  - SSN never logged in repository layer
  - `findActiveByCustomerId()` uses the partial unique index efficiently (confirm via EXPLAIN)
  - Integration test with real PostgreSQL container confirms round-trip (save + findById)

  **Dependencies**: T008, T012, T013

---

- [ ] T018 [US1] KYC Service — HTTP routes: POST, GET /active, GET /{id}, PATCH /{id}, DELETE /{id}

  **Description**: Wire Fastify routes to use cases. Implement request/response schema
  validation (Zod via fastify-zod or JSON Schema). Ensure SSN is `writeOnly: true` and
  never serialized in responses. Apply JWT auth + correlation middleware to all routes.

  **Files**:
  - `services/kyc-service/src/infrastructure/http/routes/applicationRoutes.ts` (create)
  - `services/kyc-service/src/infrastructure/http/controllers/ApplicationController.ts` (create)
  - `services/kyc-service/src/infrastructure/http/schemas/applicationSchemas.ts` (create — Zod)

  **Acceptance criteria**:
  - `POST /api/v1/kyc-applications` → 201 (new draft) or 409 (active non-draft exists)
  - `GET /api/v1/kyc-applications/active` → 200 (with application) or 404
  - `GET /api/v1/kyc-applications/{id}` → 200 or 403 (wrong customer) or 404
  - `PATCH /api/v1/kyc-applications/{id}` → 200 (updated) or 400/403/409/422
  - `DELETE /api/v1/kyc-applications/{id}` → 204 or 409 (not draft)
  - Response never includes `ssn_encrypted`, `ssn_hash`, or raw SSN
  - All response shapes match `contracts/kyc-service.yaml` exactly

  **Dependencies**: T014, T015, T016, T017

---

### Frontend (User Story 1)

- [ ] T019 [P] [US1] Frontend — Shared form components (Input, Select)

  **Description**: Build accessible `Input` and `Select` components meeting WCAG AA.
  Include visible labels, `aria-describedby` for error messages, error state styling,
  and `aria-required`. Write Storybook stories for each.

  **Files**:
  - `frontend/src/shared/components/Input.tsx` (create)
  - `frontend/src/shared/components/Select.tsx` (create)
  - `frontend/src/shared/components/Input.stories.tsx` (create)
  - `frontend/src/shared/components/Select.stories.tsx` (create)

  **Acceptance criteria**:
  - Keyboard navigable; focus ring visible
  - Error message associated via `aria-describedby`
  - Storybook renders: default, filled, error, disabled variants
  - Passes `axe-core` accessibility scan with zero violations

  **Dependencies**: T002

---

- [ ] T020 [US1] Frontend — KYCForm component

  **Description**: Build the main personal information form using React Hook Form + Zod.
  Fields: Full Legal Name (required), Date of Birth (required, min age 18), SSN (required,
  format XXX-XX-XXXX), Home Address (required), US Phone Number (optional, NANP format).
  Show inline field-level validation errors. Disable submit button until all required fields
  are valid.

  **Files**:
  - `frontend/src/features/kyc/components/KYCForm.tsx` (create)
  - `frontend/src/features/kyc/components/KYCForm.stories.tsx` (create)
  - `frontend/src/features/kyc/schemas/kycFormSchema.ts` (create — Zod schema)

  **Acceptance criteria**:
  - All 4 required fields show inline errors when invalid
  - Phone field shows error only when non-empty and invalid format
  - Empty phone field does NOT block form submission
  - Age validation: DOB within 18 years of today shows error
  - Calling `onSubmit` prop fires only when all required fields are valid
  - Storybook renders: empty, partially filled, all errors, valid states

  **Dependencies**: T019

---

- [ ] T021 [US1] Frontend — useKYCApplication TanStack Query hooks

  **Description**: Implement hooks: `useGetActiveApplication` (query), `useCreateApplication`
  (mutation), `useUpdateApplication` (mutation, debounced 500 ms), `useDiscardApplication`
  (mutation). Connect to `kycApi.ts` Axios functions.

  **Files**:
  - `frontend/src/features/kyc/hooks/useKYCApplication.ts` (create)
  - `frontend/src/features/kyc/api/kycApi.ts` (create)

  **Acceptance criteria**:
  - `useGetActiveApplication` caches result; `enabled: false` when no auth token
  - `useUpdateApplication` debounces 500 ms before firing PATCH call
  - `useCreateApplication` on 409 response surfaces the `referenceNumber` from error payload
  - All hooks surface `correlationId` from error responses

  **Dependencies**: T011, T018

---

- [ ] T022 [US1] Frontend — DraftResume and ApplicationBlocked components

  **Description**: `DraftResume` shows resume/discard choice when a draft is detected.
  `ApplicationBlocked` shows a non-form view with reference number + status when a
  pending/under_review application exists. Both meet WCAG AA.

  **Files**:
  - `frontend/src/features/kyc/components/DraftResume.tsx` (create)
  - `frontend/src/features/kyc/components/ApplicationBlocked.tsx` (create)
  - `frontend/src/features/kyc/components/DraftResume.stories.tsx` (create)
  - `frontend/src/features/kyc/components/ApplicationBlocked.stories.tsx` (create)

  **Acceptance criteria**:
  - `DraftResume`: "Resume" pre-populates form; "Start Over" calls `useDiscardApplication`
    then resets form
  - `ApplicationBlocked`: displays reference number, status, and message "Your application
    is under review"
  - Neither component renders form inputs

  **Dependencies**: T020, T021

---

- [ ] T023 [US1] Frontend — KYCFormPage (orchestrator)

  **Description**: Page component that orchestrates the full US1 flow: check active
  application on mount → render correct state (DraftResume, ApplicationBlocked, or fresh
  form). Wire auto-save (500 ms debounce on field change via `useUpdateApplication`).
  Include `beforeunload` handler that fires `sendBeacon` to flush unsaved draft.

  **Files**:
  - `frontend/src/features/kyc/pages/KYCFormPage.tsx` (create)
  - `frontend/src/app/router.tsx` (update — add /kyc → KYCFormPage)

  **Acceptance criteria**:
  - On mount: calls `GET /api/v1/kyc-applications/active`
  - If 404: calls `POST /api/v1/kyc-applications` to create draft
  - If draft returned: shows DraftResume
  - If pending/under_review: shows ApplicationBlocked
  - Auto-save fires PATCH on field change (debounced 500 ms)
  - Tab-close triggers `sendBeacon` with latest unsaved field values
  - On `POST /{id}/submissions` 4xx/5xx error: all form field values are retained in state,
    a visible "Try Again" button appears inline with the error message, and the customer
    can re-attempt submission without navigating away (FR-013, US3 AC-3)

  **Dependencies**: T022, T021

---

**Checkpoint**: User Story 1 fully functional. Customer can fill form, auto-save draft,
resume draft, and see blocked state for existing active application.

---

## Phase 4: User Story 2 — Upload Identity Document (Priority: P2)

**Goal**: Customer selects document type from a two-option selector, uploads JPEG/PNG/PDF
≤ 5 MB directly to S3 via presigned URL, and the document is linked to their application.

**Independent Test**: quickstart.md Scenarios 5–6

---

### Domain + Application Layer (document-service)

- [ ] T024 [P] [US2] Document Service — Domain entities and value objects

  **Description**: `IdentityDocument` entity, `DocumentType` value object (enforces
  `us_drivers_license | us_passport`), `FileMetadata` value object (validates format in
  `jpeg|png|pdf` and size ≤ 5,242,880 bytes).

  **Files**:
  - `services/document-service/src/domain/entities/IdentityDocument.ts` (create)
  - `services/document-service/src/domain/value-objects/DocumentType.ts` (create)
  - `services/document-service/src/domain/value-objects/FileMetadata.ts` (create)

  **Acceptance criteria**:
  - `DocumentType` throws `ValidationError` for any value outside the two accepted types
  - `FileMetadata` throws `ValidationError` for `fileSizeBytes` > 5,242,880
  - `FileMetadata` throws `ValidationError` for formats outside `jpeg|png|pdf`
  - No infrastructure imports in domain layer

  **Dependencies**: T006

---

- [ ] T025 [P] [US2] Document Service — Repository and storage interfaces

  **Description**: Define `IIdentityDocumentRepository` (CRUD) and `IDocumentStorage`
  (generatePresignedPutUrl, headObject) as TypeScript interfaces.

  **Files**:
  - `services/document-service/src/domain/repositories/IIdentityDocumentRepository.ts` (create)
  - `services/document-service/src/domain/ports/IDocumentStorage.ts` (create)

  **Acceptance criteria**:
  - Both interfaces are pure TypeScript with no AWS SDK or Drizzle imports
  - `IDocumentStorage.generatePresignedPutUrl()` returns `{url: string, expiresAt: Date}`

  **Dependencies**: T024

---

- [ ] T026 [US2] Document Service — UploadDocumentUseCase (initiate + confirm)

  **Description**: Two use-case flows:
  1. **Initiate**: validates DocumentType + FileMetadata, creates `IdentityDocument`
     (status: pending) via repository, generates presigned S3 PUT URL via storage port,
     returns `{documentId, presignedUrl, expiresAt}`.
  2. **Confirm**: calls `IDocumentStorage.headObject()` to verify file exists, updates
     `upload_status` to `uploaded`.

  **Files**:
  - `services/document-service/src/application/use-cases/InitiateUploadUseCase.ts` (create)
  - `services/document-service/src/application/use-cases/ConfirmUploadUseCase.ts` (create)
  - `services/document-service/src/application/dtos/InitiateUploadDto.ts` (create)

  **Acceptance criteria**:
  - Initiate with `documentType = "utility_bill"` → throws `ValidationError`
  - Initiate with `fileSizeBytes = 6000000` → throws `ValidationError` before S3 call
  - Confirm when S3 object missing → throws `UnprocessableError`
  - Confirm when already confirmed → throws `ConflictError`

  **Dependencies**: T025

---

- [ ] T027 [US2] Document Service — PostgreSQL repository and S3 storage adapter

  **Description**: `IdentityDocumentRepository` using Drizzle. `S3DocumentStorage` adapter
  using `@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`. S3 key pattern:
  `documents/{applicationId}/{documentId}/{fileName}`. Presigned URL TTL: 15 minutes.

  **Files**:
  - `services/document-service/src/infrastructure/persistence/IdentityDocumentRepository.ts` (create)
  - `services/document-service/src/infrastructure/storage/S3DocumentStorage.ts` (create)
  - `services/document-service/src/infrastructure/persistence/mappers/documentMapper.ts` (create)

  **Acceptance criteria**:
  - `generatePresignedPutUrl()` returns a URL scoped to exact S3 key (application_id +
    document_id embedded in path)
  - Presigned URL expires in 15 minutes (verified in unit test with mock clock)
  - `headObject()` returns `true` when object exists, `false` when 404 from S3
  - S3 bucket name read from environment variable (not hardcoded)

  **Dependencies**: T009, T024, T025, T026

---

- [ ] T028 [US2] Document Service — HTTP routes

  **Description**: Wire Fastify routes per `contracts/document-service.yaml`:
  `GET /document-types`, `POST /documents` (initiate), `GET /documents/{id}`,
  `POST /documents/{id}/upload-confirmations` (confirm). Apply auth middleware.

  **Files**:
  - `services/document-service/src/infrastructure/http/routes/documentRoutes.ts` (create)
  - `services/document-service/src/infrastructure/http/controllers/DocumentController.ts` (create)
  - `services/document-service/src/infrastructure/http/schemas/documentSchemas.ts` (create)

  **Acceptance criteria**:
  - `GET /document-types` → 200 with exactly two types (no auth required)
  - `POST /documents` with invalid `documentType` → 400 with `INVALID_DOCUMENT_TYPE` code
  - `POST /documents` with `fileSizeBytes > 5242880` → 400 with `EXCEEDS_MAX_SIZE` code
  - `POST /documents` with unsupported format → 400 with `UNSUPPORTED_FORMAT` code
  - `POST /documents/{id}/upload-confirmations` when S3 missing → 422
  - All responses match `contracts/document-service.yaml` exactly

  **Dependencies**: T026, T027

---

### Frontend (User Story 2)

- [ ] T029 [P] [US2] Frontend — FileUpload shared component

  **Description**: Accessible file input component with drag-and-drop support. Client-side
  validates format (from `accept` attribute) and size (≤ 5 MB) before firing `onChange`.
  Shows file name, size, and a progress indicator during upload. WCAG AA compliant.

  **Files**:
  - `frontend/src/shared/components/FileUpload.tsx` (create)
  - `frontend/src/shared/components/FileUpload.stories.tsx` (create)

  **Acceptance criteria**:
  - Accepts only `image/jpeg`, `image/png`, `application/pdf`
  - Rejects files > 5 MB with inline error (no API call triggered)
  - Shows file name and formatted size after selection
  - Progress bar visible during S3 upload (via `onUploadProgress` Axios callback)
  - Keyboard accessible (Enter/Space to open file picker)

  **Dependencies**: T002

---

- [ ] T030 [US2] Frontend — DocumentUpload component

  **Description**: Combines a `Select` (document type: "US Driver's License" / "US Passport")
  with `FileUpload`. Document type selector MUST be selected before the file input is
  enabled. On file select: calls `POST /api/v1/documents` to initiate upload, PUTs file to
  presigned URL, then calls `POST /api/v1/documents/{id}/upload-confirmations`. On success:
  calls `PATCH /api/v1/kyc-applications/{id}` to link `documentId`.

  **Files**:
  - `frontend/src/features/kyc/components/DocumentUpload.tsx` (create)
  - `frontend/src/features/kyc/components/DocumentUpload.stories.tsx` (create)
  - `frontend/src/features/kyc/hooks/useDocumentUpload.ts` (create)
  - `frontend/src/features/kyc/api/documentApi.ts` (create)

  **Acceptance criteria**:
  - File input disabled until document type selected
  - Exactly two options in document type selector: "US Driver's License", "US Passport"
  - Upload flow: initiate → S3 PUT → confirm → link to application (three sequential calls)
  - If any upload step fails: silently retry up to 3 times with exponential backoff; if all
    3 retries fail, show inline error with "Try Again" button; personal info fields are
    unaffected (US2 AC-6, spec.md)
  - On success: parent `KYCFormPage` receives `documentId` for form submission readiness

  **Dependencies**: T021, T028, T029

---

**Checkpoint**: User Story 2 fully functional. Customer can select document type, upload
file to S3, and have it linked to their application.

---

## Phase 5: User Story 3 — Confirm KYC Application Submission (Priority: P3)

**Goal**: Customer submits the completed application, receives a unique reference number,
and sees a confirmation screen. Notification is dispatched to contact on file.

**Independent Test**: quickstart.md Scenarios 1, 7

---

- [ ] T031 [US3] KYC Service — SubmitApplicationUseCase

  **Description**: Validates all required fields and `documentId` are non-null. Validates
  DOB age ≥ 18 at submission time. Generates `ReferenceNumber`. Transitions status to
  `pending`. Sets `submitted_at`. Calls `NotificationServiceClient` (REST) to dispatch
  confirmation — with 3-second timeout; notification failure does NOT fail submission.

  **Files**:
  - `services/kyc-service/src/application/use-cases/SubmitApplicationUseCase.ts` (create)
  - `services/kyc-service/src/infrastructure/http-client/NotificationServiceClient.ts` (create)

  **Acceptance criteria**:
  - Missing required field → throws `UnprocessableError` listing all missing fields
  - `documentId` null → throws `UnprocessableError` with `DOCUMENT_REQUIRED`
  - Age < 18 → throws `UnprocessableError` with `AGE_BELOW_MINIMUM`
  - `ssn_hash` matches an active application under a different `customer_id` → throws
    `ConflictError` with code `SSN_ALREADY_ACTIVE` and generic message (no conflicting
    customer details revealed) per FR-017; application reverts to draft status
  - On success: `reference_number` set, `status = pending`, `submitted_at` set
  - If notification-service call times out (3 s): submission still returns 200 with
    reference number; timeout logged as warning
  - Idempotent: calling submit on an already-pending application → throws `ConflictError`

  **Dependencies**: T017, T018

---

- [ ] T032 [US3] KYC Service — HTTP route: POST /api/v1/kyc-applications/{id}/submissions

  **Description**: Wire submit route to `SubmitApplicationUseCase`. Maps
  `UnprocessableError` → 422, `ConflictError` → 409.

  **Files**:
  - `services/kyc-service/src/infrastructure/http/routes/applicationRoutes.ts` (update — add submissions sub-route)
  - `services/kyc-service/src/infrastructure/http/controllers/ApplicationController.ts` (update — add submit handler)

  **Acceptance criteria**:
  - `POST /{id}/submissions` on valid complete application → 200 with `referenceNumber`
  - Missing required field → 422 with `details` listing each missing field + issue code
  - Already pending → 409
  - Response shape matches `contracts/kyc-service.yaml` `SubmissionResult` schema

  **Dependencies**: T031, T018

---

- [ ] T033 [US3] Notification Service — Domain + Application layer

  **Description**: `NotificationLog` entity. `SendConfirmationUseCase`: accepts
  `{applicationId, customerId, channel, recipient}`, checks idempotency key, creates
  `NotificationLog` record (pending), dispatches via `INotificationChannel`, updates status.

  **Files**:
  - `services/notification-service/src/domain/entities/NotificationLog.ts` (create)
  - `services/notification-service/src/domain/repositories/INotificationLogRepository.ts` (create)
  - `services/notification-service/src/domain/ports/INotificationChannel.ts` (create)
  - `services/notification-service/src/application/use-cases/SendConfirmationUseCase.ts` (create)

  **Acceptance criteria**:
  - Duplicate `applicationId + submission_confirmation` → returns existing log (idempotent)
  - Channel dispatch failure → status set to `failed`, `error_message` logged (sanitized)
  - No PII (SSN, document content) in notification content or logs

  **Dependencies**: T010

---

- [ ] T034 [P] [US3] Notification Service — SES email channel adapter

  **Description**: `EmailNotificationChannel` implementing `INotificationChannel` using
  `@aws-sdk/client-ses`. Sends confirmation email with application reference number.
  In local dev: uses LocalStack SES or logs to console.

  **Files**:
  - `services/notification-service/src/infrastructure/channels/EmailNotificationChannel.ts` (create)

  **Acceptance criteria**:
  - Sends email to `recipient` with `referenceNumber` in body
  - LocalStack environment: email logged, no real send attempted
  - `SES_FROM_ADDRESS` read from environment variable

  **Dependencies**: T033

---

- [ ] T035 [P] [US3] Notification Service — PostgreSQL repository + HTTP route

  **Description**: `NotificationLogRepository` using Drizzle. Fastify route:
  `POST /api/v1/notifications` (internal — validated by JWT + service-to-service header).

  **Files**:
  - `services/notification-service/src/infrastructure/persistence/NotificationLogRepository.ts` (create)
  - `services/notification-service/src/infrastructure/http/routes/notificationRoutes.ts` (create)
  - `services/notification-service/src/infrastructure/http/controllers/NotificationController.ts` (create)

  **Acceptance criteria**:
  - `POST /api/v1/notifications` → 201 on success
  - Duplicate `idempotency_key` → 200 (already sent)
  - Response includes `notificationId` and `status`

  **Dependencies**: T033, T034

---

- [ ] T036 [US3] Frontend — ConfirmationPage

  **Description**: Page rendered after successful submission. Displays reference number
  (`KYC-YYYYMMDD-NNNNNN`), status message, estimated review timeline ("within 3 business
  days"), and instruction to watch for confirmation contact.

  **Files**:
  - `frontend/src/features/kyc/pages/ConfirmationPage.tsx` (create)
  - `frontend/src/app/router.tsx` (update — add /confirmation route)

  **Acceptance criteria**:
  - Reference number prominently displayed (heading level 1)
  - Estimated review timeline of "3 business days" displayed verbatim per FR-012
  - Page accessible without KYC form state (can be linked to directly)
  - No form fields or submit buttons on this page
  - "Return to Dashboard" link navigates to `/` (or bank home)

  **Dependencies**: T032, T011

---

**Checkpoint**: All 3 user stories complete. Full KYC flow functional end-to-end.

---

## Phase 6: Security Hardening

- [ ] T037 [P] JWT validation and RBAC middleware (all services)

  **Description**: Harden JWT middleware to validate: `iss` (Cognito User Pool URL), `aud`
  (App Client ID), `exp` (not expired), `custom:role = retail_customer`. Verify JWT using
  Cognito JWKS endpoint. Apply `customer_id` ownership check on all mutating routes.

  **Files**:
  - `services/kyc-service/src/infrastructure/http/middleware/auth.ts` (update)
  - `services/document-service/src/infrastructure/http/middleware/auth.ts` (update)
  - `services/notification-service/src/infrastructure/http/middleware/auth.ts` (update)

  **Acceptance criteria**:
  - Expired JWT → 401 `UNAUTHORIZED`
  - Valid JWT with `custom:role = kyc_reviewer` → 403 `FORBIDDEN` on customer endpoints
  - PATCH/DELETE on application owned by different customer → 403 `FORBIDDEN`
  - Cognito JWKS cached (TTL 1 hour) to avoid hot-path latency

  **Dependencies**: T005, T006, T007

---

- [ ] T038 [P] AWS Secrets Manager integration

  **Description**: At startup, each service reads DB connection string, KMS key ARN (kyc-service),
  and S3 bucket name (document-service) from Secrets Manager. Fail fast if any required
  secret is missing. For local dev, fall back to environment variables.

  **Files**:
  - `services/kyc-service/src/infrastructure/config/SecretsConfig.ts` (create)
  - `services/document-service/src/infrastructure/config/SecretsConfig.ts` (create)
  - `services/notification-service/src/infrastructure/config/SecretsConfig.ts` (create)

  **Acceptance criteria**:
  - Service fails to start if required secrets are unavailable (explicit error message)
  - Local dev: `USE_LOCAL_CONFIG=true` env var uses `.env.local` values instead
  - Secret values never logged

  **Dependencies**: T005, T006, T007

---

- [ ] T039 [P] SSN encryption via AWS KMS

  **Description**: Implement `KMSEncryptionService` using `@aws-sdk/client-kms`. Encrypt
  raw SSN bytes using KMS `Encrypt` API (AES-256-GCM envelope). Decrypt in `findById` /
  `findActiveByCustomerId`. For local dev: use LocalStack KMS.

  **Files**:
  - `services/kyc-service/src/infrastructure/security/KMSEncryptionService.ts` (update — add real KMS calls)

  **Acceptance criteria**:
  - Round-trip test: encrypt SSN → store → retrieve → decrypt → matches original
  - KMS key ARN configurable via Secrets Manager
  - SSN never stored as plaintext; DB column contains only encrypted BYTEA
  - LocalStack KMS key created in docker-compose startup script

  **Dependencies**: T017, T038

---

- [ ] T040 [P] S3 document bucket security configuration (CDK)

  **Description**: In the CDK infrastructure stack, configure the documents S3 bucket:
  block all public access, enable versioning, apply KMS SSE (customer-managed key), set
  bucket policy to allow only ECS task roles. Configure CloudFront WAF to rate-limit the
  `/api/v1/documents` endpoint.

  **Files**:
  - `infrastructure/lib/storage-stack.ts` (create)
  - `infrastructure/lib/networking-stack.ts` (create — VPC, security groups)

  **Acceptance criteria**:
  - S3 bucket has `BlockPublicAccess: ALL` confirmed in CDK synth output
  - KMS SSE key is customer-managed (not `aws/s3`)
  - ECS task execution roles have `s3:PutObject`, `s3:GetObject` scoped to bucket ARN only
  - WAF rule: rate limit `POST /api/v1/documents` to 10 req/min per IP

  **Dependencies**: T001

---

## Phase 7: Testing

- [ ] T041 [P] KYC Service — Domain unit tests

  **Description**: Unit tests for `KYCApplication` entity, `SSN`, `ApplicationStatus`,
  `ReferenceNumber` value objects using Vitest.

  **Files**:
  - `services/kyc-service/tests/unit/domain/KYCApplication.test.ts` (create)
  - `services/kyc-service/tests/unit/domain/SSN.test.ts` (create)
  - `services/kyc-service/tests/unit/domain/ApplicationStatus.test.ts` (create)
  - `services/kyc-service/tests/unit/domain/ReferenceNumber.test.ts` (create)

  **Acceptance criteria**:
  - `SSN` rejects all invalid formats (< 9 digits, wrong separators, all-zeros)
  - `ApplicationStatus` throws on all invalid transitions
  - `KYCApplication.canSubmit()` returns false with any null required field
  - `ReferenceNumber` format matches `KYC-YYYYMMDD-NNNNNN` regex
  - 100% branch coverage on value objects

  **Dependencies**: T012

---

- [ ] T042 [P] KYC Service — Use case unit tests

  **Description**: Unit tests for `CreateDraftApplicationUseCase`, `UpdateDraftApplicationUseCase`,
  `SubmitApplicationUseCase` using Vitest with mock repositories.

  **Files**:
  - `services/kyc-service/tests/unit/application/CreateDraftApplicationUseCase.test.ts` (create)
  - `services/kyc-service/tests/unit/application/UpdateDraftApplicationUseCase.test.ts` (create)
  - `services/kyc-service/tests/unit/application/SubmitApplicationUseCase.test.ts` (create)

  **Acceptance criteria**:
  - Create: returns existing draft when one exists; creates new when none; throws ConflictError for pending
  - Update: throws ForbiddenError on wrong customerId; throws ValidationError on invalid SSN or phone format; does NOT validate age (age check is submission-only)
  - Submit: throws UnprocessableError for each missing required field; idempotent on already-pending
  - Submit: throws ConflictError with `SSN_ALREADY_ACTIVE` when ssn_hash matches active application under different customer_id (FR-017)
  - Notification timeout does NOT cause submit to fail

  **Dependencies**: T014, T015, T031

---

- [ ] T043 [P] KYC Service — Integration tests (Supertest + real PostgreSQL)

  **Description**: Integration tests for all 7 HTTP endpoints using Supertest. Tests run
  against a real PostgreSQL Docker container (no mocks at DB layer). KMS mocked via
  LocalStack or a test stub.

  **Files**:
  - `services/kyc-service/tests/integration/kyc-applications.test.ts` (create)
  - `services/kyc-service/tests/integration/setup.ts` (create — DB setup/teardown)

  **Acceptance criteria**:
  - `POST /kyc-applications` → 201; second call same customer → 201 same draft; pending exists → 409
  - `PATCH` with invalid SSN format → 400; with age < 18 DOB → 422
  - `POST /{id}/submissions` missing documentId → 422; complete → 200 with referenceNumber
  - All response shapes validated against `contracts/kyc-service.yaml` JSON Schemas

  **Dependencies**: T018, T032

---

- [ ] T044 [P] Document Service — Domain + use case + integration tests

  **Description**: Unit tests for `IdentityDocument`, `DocumentType`, `FileMetadata`.
  Unit tests for `InitiateUploadUseCase`, `ConfirmUploadUseCase`. Integration tests for
  all 4 HTTP endpoints with LocalStack S3.

  **Files**:
  - `services/document-service/tests/unit/domain/IdentityDocument.test.ts` (create)
  - `services/document-service/tests/unit/domain/DocumentType.test.ts` (create)
  - `services/document-service/tests/unit/domain/FileMetadata.test.ts` (create)
  - `services/document-service/tests/unit/application/UploadDocumentUseCase.test.ts` (create)
  - `services/document-service/tests/integration/documents.test.ts` (create)

  **Acceptance criteria**:
  - `DocumentType` throws for every value outside two accepted types
  - `FileMetadata` throws for each invalid format and for size > 5,242,880
  - Integration: `POST /documents` with size 6 MB → 400; with unsupported format → 400
  - Integration: `POST /documents/{id}/upload-confirmations` when S3 missing → 422
  - All responses validated against `contracts/document-service.yaml`

  **Dependencies**: T026, T028

---

- [ ] T045 [P] Notification Service — Unit + integration tests

  **Description**: Unit tests for `SendConfirmationUseCase` (idempotency, channel failure
  handling). Integration test for `POST /api/v1/notifications` using LocalStack SES.

  **Files**:
  - `services/notification-service/tests/unit/application/SendConfirmationUseCase.test.ts` (create)
  - `services/notification-service/tests/integration/notifications.test.ts` (create)

  **Acceptance criteria**:
  - Duplicate call with same `applicationId` + type → returns existing log (no second send)
  - SES failure → status set to `failed`; use case does not throw (swallowed + logged)

  **Dependencies**: T033, T035

---

- [ ] T046 [P] Frontend — Component unit tests (Vitest + RTL)

  **Description**: Behavior-focused unit tests for all KYC feature components.

  **Files**:
  - `frontend/tests/unit/features/kyc/KYCForm.test.tsx` (create)
  - `frontend/tests/unit/features/kyc/DocumentUpload.test.tsx` (create)
  - `frontend/tests/unit/features/kyc/DraftResume.test.tsx` (create)
  - `frontend/tests/unit/features/kyc/ApplicationBlocked.test.tsx` (create)
  - `frontend/tests/unit/features/kyc/ConfirmationPage.test.tsx` (create)
  - `frontend/tests/unit/shared/Input.test.tsx` (create)
  - `frontend/tests/unit/shared/FileUpload.test.tsx` (create)

  **Acceptance criteria**:
  - KYCForm: submit blocked when required field empty; phone error shown only when non-empty
    and invalid; SSN masked in display
  - DocumentUpload: file input disabled until type selected; 5 MB rejection fires before API call
  - ApplicationBlocked: renders reference number from props
  - DraftResume: "Start Over" calls discard mutation
  - All tests use MSW handlers — no real HTTP calls

  **Dependencies**: T020, T022, T029, T030, T036

---

- [ ] T047 [P] Frontend — MSW API mock handlers

  **Description**: Implement MSW 2 request handlers mirroring all endpoints in
  `contracts/kyc-service.yaml` and `contracts/document-service.yaml`. Include happy-path
  and error-path handlers. Share handlers between test suite and Storybook.

  **Files**:
  - `frontend/tests/mocks/handlers/kycHandlers.ts` (create)
  - `frontend/tests/mocks/handlers/documentHandlers.ts` (create)
  - `frontend/tests/mocks/handlers/index.ts` (create)
  - `frontend/tests/mocks/fixtures/kycApplication.fixture.ts` (create)
  - `frontend/tests/mocks/fixtures/identityDocument.fixture.ts` (create)

  **Acceptance criteria**:
  - Handler response shapes derived from OpenAPI `contracts/` YAML files
  - Handlers cover: happy path, 400, 401, 403, 404, 409, 422 for key endpoints
  - Storybook decorators use same handlers as test suite

  **Dependencies**: T011

---

- [ ] T048 [P] OpenAPI contract tests (Dredd)

  **Description**: Configure Dredd to run against all three services using all three OpenAPI
  contracts. Run against services started with test DB (Docker). Fail CI if any response
  deviates from spec.

  **Files**:
  - `services/kyc-service/dredd.yml` (create)
  - `services/document-service/dredd.yml` (create)
  - `services/notification-service/dredd.yml` (create)
  - `.github/workflows/ci.yml` (update — add Dredd step for all three services)

  **Acceptance criteria**:
  - All documented response schemas for all three services pass Dredd validation
  - CI step fails if any endpoint returns a shape not matching `contracts/*.yaml`
  - notification-service Dredd covers `POST /api/v1/notifications` (201 + 200 idempotent)

  **Dependencies**: T043, T044

---

## Phase 8: CI/CD Pipeline

- [ ] T049 [P] Dockerfiles for all services and frontend

  **Description**: Multi-stage Dockerfiles for kyc-service, document-service,
  notification-service (Node 20 Alpine builder + runtime), and frontend (Vite build +
  nginx static server).

  **Files**:
  - `services/kyc-service/Dockerfile` (create)
  - `services/document-service/Dockerfile` (create)
  - `services/notification-service/Dockerfile` (create)
  - `frontend/Dockerfile` (create — Vite build + nginx)
  - `frontend/nginx.conf` (create — SPA fallback, gzip, cache headers)

  **Acceptance criteria**:
  - Each service image builds successfully with `docker build`
  - Production images are non-root user, no dev dependencies, no source maps exposed
  - Frontend nginx serves `index.html` for all unmatched routes (SPA routing)
  - Image sizes: services < 150 MB, frontend < 30 MB

  **Dependencies**: T005, T006, T007, T002

---

- [ ] T050 [P] GitHub Actions — CI workflow (lint, typecheck, test, contract validation)

  **Description**: `.github/workflows/ci.yml` runs on every push and pull request to
  `develop`. Jobs run in parallel per workspace: lint, typecheck, unit tests, integration
  tests (with Docker services), OpenAPI contract tests.

  **Files**:
  - `.github/workflows/ci.yml` (create)

  **Acceptance criteria**:
  - All jobs run in parallel per service/frontend
  - Integration test jobs start PostgreSQL and LocalStack as service containers
  - Dredd contract validation job runs after integration tests pass
  - `npm run typecheck` failure blocks merge
  - CI completes in < 10 minutes for a typical push

  **Dependencies**: T041, T042, T043, T044, T045, T046, T048, T049

---

- [ ] T051 [P] GitHub Actions — Build and ECR push workflow

  **Description**: `.github/workflows/build-push.yml` triggers on merge to `develop`.
  Builds Docker images, tags with `{service}-{sha}-{version}`, and pushes to Amazon ECR.

  **Files**:
  - `.github/workflows/build-push.yml` (create)

  **Acceptance criteria**:
  - Images pushed to ECR for all four packages (3 services + frontend)
  - Tags include both `latest` and the SHA-based tag
  - Workflow uses OIDC (not long-lived AWS keys) for ECR authentication

  **Dependencies**: T049, T050

---

- [ ] T052 [P] GitHub Actions — Deploy workflow (CDK + migrations + ECS rolling update)

  **Description**: `.github/workflows/deploy.yml` triggers on release tag `release/v*`.
  Runs `cdk diff`, applies DB migrations per service, `cdk deploy` (ECS rolling update),
  runs smoke tests (health checks), rolls back on failure.

  **Files**:
  - `.github/workflows/deploy.yml` (create)
  - `scripts/run-migrations.sh` (create — runs drizzle-kit migrate for each service in order)
  - `scripts/smoke-test.sh` (create — curls each service /health endpoint)

  **Acceptance criteria**:
  - Migration runs in order: kyc-service → document-service → notification-service
  - ECS rolling update: min 100% healthy during deploy
  - Health check endpoints return 200 before deploy is marked complete
  - CDK rollback triggered automatically if health check fails

  **Dependencies**: T051

---

## Phase 9: Documentation & Diagrams

- [ ] T053 [P] Storybook — Shared component stories

  **Description**: Publish Storybook stories for all shared components: Button, Input,
  Select, FileUpload. Each component has default, filled, error, disabled, and loading
  variants.

  **Files**:
  - `frontend/src/shared/components/Button.tsx` (create)
  - `frontend/src/shared/components/Button.stories.tsx` (create)
  - `frontend/src/shared/components/Input.stories.tsx` (update — ensure all variants)
  - `frontend/src/shared/components/Select.stories.tsx` (update)
  - `frontend/src/shared/components/FileUpload.stories.tsx` (update)

  **Acceptance criteria**:
  - `npm run storybook` builds without errors
  - All shared components have ≥ 4 story variants
  - `axe-core` Storybook addon reports zero accessibility violations

  **Dependencies**: T019, T029

---

- [ ] T054 [P] Storybook — Feature component stories

  **Description**: Storybook stories for KYCForm, DocumentUpload, DraftResume,
  ApplicationBlocked, ConfirmationPage. Use MSW Storybook addon with shared handlers.

  **Files**:
  - `frontend/src/features/kyc/components/KYCForm.stories.tsx` (update — MSW addon)
  - `frontend/src/features/kyc/components/DocumentUpload.stories.tsx` (update)
  - `frontend/src/features/kyc/components/DraftResume.stories.tsx` (update)
  - `frontend/src/features/kyc/components/ApplicationBlocked.stories.tsx` (update)
  - `frontend/src/features/kyc/pages/ConfirmationPage.stories.tsx` (create)

  **Acceptance criteria**:
  - Each story uses MSW handlers (not hardcoded mock data)
  - Stories cover happy path + error states
  - Upload progress story demonstrates file upload animation

  **Dependencies**: T020, T022, T030, T036, T047

---

- [ ] T055 [P] Project README

  **Description**: `README.md` at repo root covering: project overview, architecture
  summary, local dev setup (prerequisites + quick-start commands), service descriptions,
  links to spec artifacts, environment variable reference.

  **Files**:
  - `README.md` (create)

  **Acceptance criteria**:
  - `git clone` + `npm install` + `docker compose up` + `npm run dev` fully documented
  - Links to `specs/001-kyc-retail-form/plan.md` and `quickstart.md`
  - Environment variable table for each service

  **Dependencies**: T004, T023, T030, T036

---

## Phase 10: Polish & Cross-Cutting Concerns

- [ ] T056 [P] Observability — CloudWatch metric filters and alarms (CDK)

  **Description**: Add CloudWatch metric filters for 4XX rate, 5XX rate, and document
  upload failures to each service's log group. Configure alarms: 5XX rate > 1% over 5 min
  → SNS alert.

  **Files**:
  - `infrastructure/lib/observability-stack.ts` (create)

  **Acceptance criteria**:
  - Metric filters created for all three services
  - CloudWatch alarm created for 5XX rate threshold
  - SNS topic ARN configurable via CDK context

  **Dependencies**: T052

---

- [ ] T057 [P] Accessibility audit and WCAG AA compliance pass

  **Description**: Run `axe-core` against all KYC feature pages via Storybook and
  Playwright. Fix any violations. Ensure all form controls have visible labels and
  descriptive error messages.

  **Files**: Updates to components identified by audit (no predetermined files)

  **Acceptance criteria**:
  - Zero `axe-core` critical or serious violations on KYCFormPage and ConfirmationPage
  - All form fields keyboard-navigable with visible focus indicators
  - Colour contrast ratio ≥ 4.5:1 for all text

  **Dependencies**: T023, T036, T053, T054

---

- [ ] T058 [P] Run quickstart.md validation (all 10 scenarios)

  **Description**: Manually or via Playwright run all 10 validation scenarios from
  `quickstart.md` against the locally running stack. Document results. Fix any gaps.

  **Files**:
  - `quickstart.md` (update — mark scenarios as validated, note any deviations)

  **Acceptance criteria**:
  - All 10 quickstart scenarios pass
  - Any deviations from expected outcomes documented with fix or justification

  **Dependencies**: T052 (or local stack from T004)

---

## Out of Scope: Admin Dashboard

The following task categories were requested but are **out of scope for this feature**
per `spec.md` Assumptions: "KYC review and approval flows are handled by a downstream
review process, not by this feature."

To implement reviewer/admin functionality, create a new feature specification with:
- User stories for the KYC reviewer role (`kyc_reviewer` RBAC role defined in constitution)
- Requirements for application queue, document viewing, approve/reject flows
- Admin dashboard UI and associated API endpoints

Placeholder for future feature:
- [ ] ADMIN-001 ⛔ KYC Reviewer Dashboard — OUT OF SCOPE (requires separate spec)

---

## Phase 11: Post-Analysis Additions (2026-06-17)

*Tasks added after `/speckit-analyze` + `/speckit-clarify` identified gaps.*

---

- [ ] T059 [P] notification-service OpenAPI contract + Dredd hookup

  **Description**: The `contracts/notification-service.yaml` OpenAPI 3.1 spec was created
  during plan update (2026-06-17). This task ensures the notification-service implementation
  is wired to validate against it: add schema validation middleware to the
  `POST /api/v1/notifications` route, and add `notification-service/dredd.yml` config
  pointing to the contract. Required to satisfy constitution §IV: "Every service SHALL
  maintain an OpenAPI specification."

  **Files**:
  - `specs/001-kyc-retail-form/contracts/notification-service.yaml` ✅ (already created)
  - `services/notification-service/dredd.yml` (create)
  - `services/notification-service/src/infrastructure/http/schemas/notificationSchemas.ts` (create — Zod, derived from contract)

  **Acceptance criteria**:
  - `POST /api/v1/notifications` request body validated against `NotificationRequest` schema
  - Response shape matches `NotificationLog` schema for both 201 and 200
  - Dredd passes against the live notification-service in CI
  - `.github/workflows/ci.yml` Dredd step covers all three services

  **Dependencies**: T035, T048

---

- [ ] T060 [P] CDK — `database-stack.ts` (RDS PostgreSQL 16)

  **Description**: CDK stack creating the shared RDS PostgreSQL 16 instance (Multi-AZ for
  resilience), security group scoped to ECS compute layer, and Secrets Manager secret for
  the DB connection string. Creates all three per-service DB users with schema-scoped GRANTs
  via a custom resource or post-deploy script.

  **Files**:
  - `infrastructure/lib/database-stack.ts` (create)
  - `infrastructure/bin/kyc-demo.ts` (update — add DatabaseStack instantiation)
  - `scripts/create-db-users.sh` (create — per-service DB user + schema GRANT script)

  **Acceptance criteria**:
  - `cdk synth` produces RDS instance with Multi-AZ enabled
  - Security group allows inbound 5432 only from ECS security group
  - Three DB users created: `kyc_app`, `document_app`, `notification_app` each with
    USAGE + CRUD on their own schema only
  - DB connection string stored in Secrets Manager (not hardcoded)

  **Dependencies**: T001, T040 (networking-stack)

---

- [ ] T061 [P] CDK — `compute-stack.ts` (ECS Fargate cluster + services)

  **Description**: CDK stack creating the ECS Fargate cluster, task definitions (one per
  service), and Fargate services with rolling deployment config. Task definitions pull
  Docker images from ECR. Task execution roles scoped to Secrets Manager and KMS.

  **Files**:
  - `infrastructure/lib/compute-stack.ts` (create)
  - `infrastructure/bin/kyc-demo.ts` (update — add ComputeStack instantiation)

  **Acceptance criteria**:
  - Three Fargate services: kyc-service, document-service, notification-service
  - Rolling update: minimum 100% healthy during deploy
  - Task execution role: read-only access to own Secrets Manager paths only
  - Health check: `/health` endpoint; unhealthy threshold = 2 consecutive failures
  - ECS service discovery configured for internal service-to-service calls

  **Dependencies**: T060 (database-stack), T049 (Dockerfiles), T051 (ECR images)

---

- [ ] T062 [P] CDK — `api-stack.ts` (API Gateway HTTP API + CloudFront distribution)

  **Description**: CDK stack creating the HTTP API Gateway (JWT authorizer pointing to
  Cognito User Pool), CloudFront distribution (routes `/api/v1/*` to API Gateway, `/*` to
  S3 SPA bucket), WAF rate limiting for document upload endpoint.

  **Files**:
  - `infrastructure/lib/api-stack.ts` (create)
  - `infrastructure/bin/kyc-demo.ts` (update — add ApiStack instantiation)

  **Acceptance criteria**:
  - API Gateway JWT authorizer validates `iss` = Cognito User Pool URL
  - CloudFront routes `/api/v1/*` to API Gateway; all other paths to S3 SPA
  - WAF rule: rate limit `POST /api/v1/documents` to 10 req/min per IP
  - `x-correlation-id` header injected at API Gateway for all requests
  - HTTPS-only; HTTP → HTTPS redirect enforced

  **Dependencies**: T063 (auth-stack), T040 (storage-stack)

---

- [ ] T063 [P] CDK — `auth-stack.ts` (Cognito User Pool + App Client)

  **Description**: CDK stack creating the AWS Cognito User Pool with the `custom:role`
  attribute schema, App Client (Authorization Code + PKCE, no client secret), and hosted UI
  domain. Outputs User Pool ID and App Client ID consumed by the frontend and API Gateway
  JWT authorizer.

  **Files**:
  - `infrastructure/lib/auth-stack.ts` (create)
  - `infrastructure/bin/kyc-demo.ts` (update — add AuthStack instantiation)

  **Acceptance criteria**:
  - User Pool has `custom:role` custom attribute (string, mutable)
  - App Client: `ALLOW_USER_PASSWORD_AUTH` + `ALLOW_REFRESH_TOKEN_AUTH`; no client secret
  - Hosted UI domain configured for PKCE flow
  - User Pool ID + App Client ID exported as CDK stack outputs
  - Test user with `custom:role = retail_customer` creatable via `aws cognito-idp admin-create-user`

  **Dependencies**: T001

---

- [ ] T064 [P] DB migration: add `ssn_hash` index for FR-017 cross-customer check

  **Description**: Add a B-tree index on `kyc.applications.ssn_hash` filtered to active
  applications (`status IN ('pending', 'under_review')`) to support efficient cross-customer
  SSN deduplication at submission time (FR-017). This is a new migration separate from the
  initial schema migration.

  **Files**:
  - `services/kyc-service/src/infrastructure/persistence/migrations/0003_add_ssn_hash_index.sql` (create)

  **Acceptance criteria**:
  - Migration runs without errors on existing database
  - EXPLAIN on `SELECT id FROM kyc.applications WHERE ssn_hash = $1 AND status IN ('pending','under_review') AND customer_id != $2` shows index scan (not seq scan)
  - Existing data unaffected

  **Dependencies**: T008 (migration 0001 + 0002)

---

- [ ] T065 [US3] KYC Service — SubmitApplicationUseCase: FR-017 SSN cross-customer check

  **Description**: Extend `SubmitApplicationUseCase` (T031) to query `ssn_hash` against all
  active applications across all customer accounts before transitioning to pending. Add
  `findActiveBySSNHash(hash: string, excludeCustomerId: string)` method to
  `IKYCApplicationRepository` and its PostgreSQL implementation.

  **Files**:
  - `services/kyc-service/src/domain/repositories/IKYCApplicationRepository.ts` (update — add `findActiveBySSNHash` method)
  - `services/kyc-service/src/infrastructure/persistence/KYCApplicationRepository.ts` (update — implement `findActiveBySSNHash`)
  - `services/kyc-service/src/application/use-cases/SubmitApplicationUseCase.ts` (update — add FR-017 check before status transition)

  **Acceptance criteria**:
  - `findActiveBySSNHash` queries using the index from T064
  - Collision detected → throws `ConflictError` with code `SSN_ALREADY_ACTIVE`; error message "An application using this identity is already under review" (no customer info leaked)
  - No collision → submission proceeds normally
  - Unit test: mock repository returns a collision → verify error thrown with correct code
  - Unit test: mock repository returns no collision → verify submission proceeds

  **Dependencies**: T031, T064

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup) → no deps, start immediately
Phase 2 (Foundational) → depends on Phase 1
Phases 3–5 (User Stories) → each depends on Phase 2; can run in parallel across stories
Phase 6 (Security) → can run in parallel with Phases 3–5 (different files)
Phase 7 (Testing) → tests follow their corresponding implementation tasks
Phase 8 (CI/CD) → depends on Phase 7
Phase 9 (Docs) → depends on Phases 3–5
Phase 10 (Polish) → depends on all previous phases
```

### Critical Path

```
T001 → T003 → T005 → T008 → T017 → T018 → T032 → T043 → T050 → T052
```

### Parallel Opportunities

- T002, T003, T004 — all Phase 1 setup (different packages)
- T005, T006, T007 — three service skeletons (independent)
- T008, T009, T010 — three schema migrations (independent)
- T012, T024 — kyc-service and document-service domain layers
- T019, T029 — shared frontend components (independent)
- T041, T044, T045, T046, T047 — test suites (independent)
- T049, T053, T055 — Dockerfiles, Storybook, README (independent)

---

## Notes

- [P] = parallel-safe (different files, no conflict)
- SSN is always `writeOnly` in API responses — never serialized
- Mock data in MSW handlers must mirror `contracts/*.yaml` schemas exactly
- Each task's acceptance criteria maps to a specific validation scenario in `quickstart.md`
- Dredd in CI (T048) is the enforcement gate for OpenAPI spec compliance
