# Research: KYC Retail Application Form

**Feature**: 001-kyc-retail-form
**Phase**: 0 — Technology & Design Research
**Date**: 2026-06-17

---

## Decision 1: Backend Language and Framework

**Decision**: TypeScript 5.4 on Node.js 20 LTS with Fastify 4

**Rationale**: Consistent language across frontend and backend reduces context-switching
and enables type sharing between contracts/DTOs. Node.js 20 LTS is supported until 2026.
Fastify 4 outperforms Express on throughput benchmarks (30k+ req/s on commodity hardware),
has first-class TypeScript support, and supports JSON Schema validation natively — aligning
with OpenAPI-first development. Fastify's plugin architecture maps well to Hexagonal
Architecture (infrastructure adapters as plugins).

**Alternatives considered**:
- Java 21 + Spring Boot 3: Enterprise-standard, more KYC-industry precedent, but higher
  context-switch cost for a TypeScript-heavy team and heavier container images (~500 MB vs
  ~80 MB for Node.js Alpine).
- Python 3.12 + FastAPI: Strong OpenAPI tooling but less type safety than TypeScript at
  scale; team uniformity argument favors TypeScript.

---

## Decision 2: ORM / Database Access

**Decision**: Drizzle ORM with Drizzle Kit for migrations

**Rationale**: Drizzle is type-safe, uses TypeScript-first schema definitions (not
decorators), produces zero-overhead SQL (no N+1 issues from lazy loading), and has
first-class PostgreSQL schema-isolation support (namespace per schema). Drizzle Kit
generates and applies SQL migrations deterministically — committed alongside source code
(aligns with constitution migration requirement).

**Alternatives considered**:
- Prisma: Excellent DX but requires a separate schema definition language (`.prisma`) and
  Prisma Client is a heavy runtime dependency; schema isolation requires per-service Prisma
  instances which is workable but less seamless.
- Knex.js: Mature query builder but not type-safe without additional wrappers; migration
  tooling is lower-level.
- TypeORM: Active Record pattern encourages domain contamination (constitution violation:
  domain layer must be framework-independent).

---

## Decision 3: Identity Provider

**Decision**: AWS Cognito User Pool (OAuth 2.1 / OIDC)

**Rationale**: Native AWS service; no additional vendor dependency; integrates directly
with API Gateway JWT Authorizer (no custom Lambda authorizer required); supports
Authorization Code + PKCE flow for SPAs; RBAC via custom attributes (`custom:role`).
Cognito is free for up to 50,000 MAU — appropriate for demo scale.

**Alternatives considered**:
- Auth0: Better DX and admin UI, but adds a paid third-party dependency; overkill for a
  demo.
- Keycloak (self-hosted): Open source but requires additional ECS service + operational
  overhead.
- AWS IAM Identity Center: Enterprise SSO focus; not designed for retail customer
  authentication.

---

## Decision 4: SSN Storage

**Decision**: Encrypt SSN with AWS KMS (AES-256 envelope encryption), store as `BYTEA`.
Additionally store a salted SHA-256 hash for duplicate detection.

**Rationale**: SSN is the most sensitive PII in the KYC form. Reversible encryption (via
KMS) is required for authorized reviewer access. A separate hash enables efficient
duplicate detection (`WHERE ssn_hash = $1`) without decrypting all records. KMS key
rotation is automated. SSN never appears in logs, error messages, or HTTP responses.

**Security controls**:
- Encryption/decryption occurs in kyc-service infrastructure layer only
- SSN value object in domain layer carries the raw value only in-memory during a single
  request lifecycle
- Hash salt stored in Secrets Manager (rotatable)

**Alternatives considered**:
- Tokenization (e.g., Vault): More secure but adds significant operational overhead for
  a demo.
- Hashing only: Irreversible — reviewer cannot see SSN for manual verification.
- Plaintext: Unacceptable for any production-adjacent KYC system.

---

## Decision 5: Document Upload Architecture

**Decision**: Presigned S3 PUT URLs — frontend uploads directly to S3, document-service
handles metadata only.

**Rationale**: API Gateway has a 10 MB payload limit (HTTP API). Documents up to 5 MB
would fit but consuming binary streams through ECS is inefficient and unnecessary. Presigned
PUT URLs allow the browser to upload directly to S3 with a time-limited (15-minute) signed
credential. document-service generates the URL, frontend uploads, frontend notifies
document-service on completion, document-service confirms object existence via S3 HeadObject.

**Security**:
- Presigned URL is scoped to a specific S3 key (path includes application_id + document_id)
- 15-minute TTL prevents stale URL reuse
- S3 bucket is fully private; no public access block exceptions
- KMS SSE applied at S3 level

**Alternatives considered**:
- Multipart upload through API Gateway: Works but hits payload limits and adds unnecessary
  load on ECS tasks for binary streaming.
- S3 Transfer Acceleration: Unnecessary for demo scale.

---

## Decision 6: Frontend Testing Stack

**Decision**: Vitest + React Testing Library + MSW 2

**Rationale**: Vitest is Vite-native (faster than Jest for Vite projects), compatible with
Jest API, supports TypeScript out of the box. RTL enforces behavior-over-implementation
testing (aligns with constitution). MSW 2 (Service Worker + Node adapter) intercepts fetch
at the network boundary — tests are maximally realistic without hitting real APIs. MSW
handlers are derived from OpenAPI specs to prevent contract drift.

**Alternatives considered**:
- Jest + Enzyme: Enzyme tests implementation details; deprecated for React 18.
- Cypress Component Testing: Heavier setup; E2E tool better suited for full flows.

---

## Decision 7: Infrastructure as Code

**Decision**: AWS CDK v2 (TypeScript)

**Rationale**: CDK is the most expressive IaC for AWS; TypeScript CDK reuses the same
language as the application code. CDK Constructs for ECS Fargate, RDS, Cognito, CloudFront
are well-maintained. CDK `diff` + `deploy` integrates cleanly into GitHub Actions CI/CD.

**Alternatives considered**:
- Terraform: Language-agnostic and more portable, but HCL adds another language; for an
  AWS-only demo, CDK is simpler.
- AWS SAM: Focused on Lambda/serverless; not suited for ECS Fargate-based services.
- CloudFormation YAML: Verbose; CDK generates it anyway.

---

## Decision 8: Inter-service Communication

**Decision**: Synchronous REST HTTP (Axios client, per constitution)

**Rationale**: Constitution mandates REST for inter-service communication. For the
notification trigger (kyc-service → notification-service after submission), a synchronous
POST is straightforward and sufficient at demo scale. Idempotency is handled by the
`idempotency_key` field in `notifications.notification_log` — retries are safe.

**Risk**: If notification-service is unavailable, submission response is delayed.
**Mitigation**: kyc-service uses a short timeout (3 s) and returns 200 with reference
number regardless of notification outcome — notification failure is logged but does not
fail the submission.

**Alternatives considered**:
- SNS/SQS async messaging: More resilient but adds AWS message bus dependency and
  violates the constitution's explicit REST requirement.
- AWS EventBridge: Same concern as SNS/SQS.

---

## Decision 9: Draft Auto-Save Mechanism

**Decision**: PATCH `/api/v1/kyc-applications/{id}` debounced on the frontend (500 ms
after last keystroke), plus `beforeunload` handler for explicit saves on tab close.

**Rationale**: Server-side draft persistence requires a PATCH call per field-change batch.
Debouncing at 500 ms prevents excessive API calls during typing. The `beforeunload` handler
fires a synchronous `sendBeacon` (non-blocking) to flush the latest state before the page
unloads. The backend PATCH is idempotent — repeated PATCHes with the same data have no
side effects.

**Draft lifecycle**:
- Created: `POST /api/v1/kyc-applications` on form mount (if no existing draft)
- Updated: `PATCH /api/v1/kyc-applications/{id}` on each debounced save
- Submitted: `POST /api/v1/kyc-applications/{id}/submissions`
- Discarded: `DELETE /api/v1/kyc-applications/{id}` (new endpoint for discard flow)

---

## Resolved NEEDS CLARIFICATION items

All technical context fields were derived from the project constitution and above research.
Zero outstanding NEEDS CLARIFICATION items at end of Phase 0.
