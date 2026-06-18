<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 2.0.0
Type of bump: MAJOR — backward-incompatible redefinition; "Hexagonal Architecture (Ports and Adapters)"
removed as an accepted architectural pattern. Clean Architecture is now the sole governing model.

Modified principles:
  I. Architecture-First:
     - Removed: all occurrences of "Hexagonal Architecture (Ports and Adapters)" from introductory
       paragraph and Backend section
     - Added: Clean Architecture four-layer model (Domain, Application, Infrastructure, Interface/API)
       with explicit layer responsibilities and the inward-dependency rule

Added sections:
  Architecture-First > Backend > Clean Architecture Layers
  Architecture-First > Backend > Layer Dependency Rule

Removed sections:
  All references to "Hexagonal Architecture" and "Ports and Adapters"

Templates reviewed:
  ✅ .specify/templates/plan-template.md
       "Constitution Check" gate references constitution dynamically — no update needed
  ✅ .specify/templates/spec-template.md
       No Hexagonal Architecture references found — no update needed
  ✅ .specify/templates/tasks-template.md
       No Hexagonal Architecture references found — no update needed
  ⚠  .specify/templates/commands/*.md — directory not present; skipped
  ⚠  README.md — not present at repository root; skipped
  ✅ .specify/extensions/agent-context/README.md
       No Hexagonal Architecture references found — no update needed

Deferred items: None
-->

# KYC Demo Constitution

## Core Principles

### I. Architecture-First

The system MUST be built as a React SPA frontend backed by a microservices API layer.
All microservices MUST follow Clean Architecture. Domain logic MUST remain independent
of frameworks, databases, cloud SDKs, and transport protocols. Services MUST communicate
exclusively through REST APIs. OpenAPI-first development is mandatory — the specification
precedes implementation.

**Frontend**:

- React SPA is mandatory; TypeScript is mandatory
- Functional components and React Hooks only; class components are prohibited
- Feature-based folder structure; shared component library required
- Storybook required for all reusable UI components
- TanStack Query (React Query) required for server-state management
- ESLint and Prettier required
- Responsive design required; WCAG AA accessibility compliance required

**Backend**:

- Architecture SHALL be microservices-based
- Each microservice SHALL follow Clean Architecture
- Domain logic SHALL remain independent of frameworks, databases, cloud SDKs, and transport protocols
- Services SHALL communicate through REST APIs
- OpenAPI-first development is mandatory

**Clean Architecture Layers**:

- **Domain**: Core business entities and rules. No dependency on any outer layer, framework,
  database, or transport. This is the innermost layer.
- **Application**: Use cases and application services. Orchestrates domain objects to fulfill
  business workflows. No framework or infrastructure dependencies permitted.
- **Infrastructure**: Repositories, database adapters, cloud SDK clients, and external service
  integrations. Implements interfaces defined by the Application layer.
- **Interface / API**: HTTP controllers, REST handlers, and CLI entry points. Invokes Application
  layer use cases. Contains no business logic.

**Layer Dependency Rule**: Dependencies point inward only —
Interface/API → Infrastructure → Application → Domain.
The Domain layer MUST have zero outward dependencies.

**AWS Hosting**:

- React SPA SHALL be hosted on Amazon S3 and CloudFront
- APIs SHALL be exposed through API Gateway or Application Load Balancer
- Microservices SHALL run on ECS Fargate or EKS
- PostgreSQL SHALL be hosted on Amazon RDS

### II. Security by Design

Authentication and authorization MUST be enforced at every service boundary.
No service MUST bypass the security model. Credentials and secrets MUST NEVER
appear in source code, logs, or unencrypted storage.

- OAuth 2.1 and OpenID Connect (OIDC) are mandatory
- Authentication SHALL be delegated to an external Identity Provider
- JWT access tokens SHALL be used
- Role-Based Access Control (RBAC) SHALL be enforced
- JWT validation SHALL occur at service boundaries
- Sensitive configuration SHALL be stored in AWS Secrets Manager or Parameter Store
- HTTPS/TLS SHALL be required for all communications
- Sensitive information SHALL NOT be logged

### III. Data Integrity & Isolation

Each microservice owns exactly one database schema. Cross-schema access is strictly
prohibited regardless of implementation convenience. All cross-service data sharing
MUST occur via service APIs, preserving encapsulation and enabling independent evolution.

- PostgreSQL is the standard relational database; hosted on Amazon RDS
- A shared PostgreSQL instance MAY be used across microservices
- Each microservice SHALL own a dedicated PostgreSQL schema
- Database schema ownership SHALL be strictly enforced
- Cross-schema reads and writes SHALL NOT be permitted
- Cross-service data access SHALL occur only through service APIs
- Database permissions SHALL enforce schema isolation
- Database changes SHALL be version controlled through migrations
- Backward-compatible migration strategies SHOULD be preferred

### IV. API-First

APIs are the contracts between services and consumers. All APIs MUST be designed,
documented, and validated through an OpenAPI specification before implementation
begins. Breaking changes require a new API version.

**Design**:

- APIs SHALL be resource-oriented; verb-based endpoints SHALL NOT be allowed
- APIs SHALL be documented using OpenAPI; specifications SHALL be maintained alongside source code
- URI-based versioning SHALL be used (e.g., `/api/v1/users`)
- Resource names SHALL use plural nouns; multi-word resources SHALL use kebab-case
- camelCase URLs SHALL NOT be used

**HTTP Methods**: GET (retrieval), POST (creation), PUT (full replacement),
PATCH (partial update), DELETE (deletion)

**Standard Status Codes**: 200 OK, 201 Created, 204 No Content, 400 Bad Request,
401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict,
422 Unprocessable Entity, 500 Internal Server Error

**Pagination**: All collection endpoints SHALL support `page` and `limit` query
parameters; responses SHALL include pagination metadata.

**Error Standard**: All APIs SHALL return a consistent error response structure
containing: `code`, `message`, `details`, `correlationId`

**Contracts**:

- Every endpoint SHALL have request and response schemas
- Every endpoint SHALL have representative mock data
- Breaking API changes SHALL require a new API version

### V. Test-Driven Quality

Tests are non-negotiable. Every feature MUST have tests that validate both API
contracts and behavioral correctness. Specifications are the source of truth;
tests and mock data are derived from specifications, not authored independently.

**Backend Testing**:

- Unit tests SHALL be mandatory for Domain and Application layers
- Every API endpoint SHALL have a dedicated test suite
- Endpoint tests SHALL use mocked repositories and mocked dependencies
- Mock data fixtures SHALL be maintained for every endpoint
- Tests SHALL validate request and response contracts
- Tests SHALL align with OpenAPI specifications

**Frontend Testing**:

- React Testing Library SHALL be used
- Mock Service Worker (MSW) SHALL be used for API mocking
- Shared components SHALL include unit tests
- Feature components SHOULD include behavior-focused tests

**Continuous Integration**:

- All tests MUST pass before merge
- Pull Requests MUST include tests for new functionality
- CI SHALL fail when tests fail
- CI SHALL validate OpenAPI specifications

**Specification-Driven**:

- Specifications SHALL be the source of truth
- Tests SHALL be derived from specifications
- Mock data SHALL be derived from API contracts

## Development Standards

### Git Strategy

GitFlow is the mandatory branching strategy: `main`, `develop`, `feature/*`,
`release/*`, `hotfix/*`.

- All changes SHALL be delivered through Pull Requests
- Code review is mandatory before merge
- CI validation is mandatory before merge
- Direct commits to `main` SHALL NOT be allowed
- Direct commits to `develop` SHOULD be prohibited

### Code Quality

- SOLID principles SHALL be followed in all layers
- Separation of concerns SHALL be enforced across all Clean Architecture layers
- Dependency Injection SHALL be used
- Business logic SHALL NOT exist in Interface/API layer controllers
- Business logic SHALL NOT exist in React components
- Reusable code SHALL be extracted into shared libraries where appropriate

### Observability

- Structured logging is mandatory across all services
- Every request SHALL carry a correlation ID end-to-end
- Errors SHALL be logged consistently with sufficient context for diagnosis
- Logs SHALL be centralized in AWS CloudWatch
- Sensitive information SHALL NOT be logged

### Documentation

- Every service SHALL maintain an OpenAPI specification
- Architectural decisions SHOULD be documented (e.g., ADRs)
- Public APIs SHALL include usage examples
- Mock examples SHALL be maintained alongside specifications

## Governance

This constitution is the governing engineering standard for the KYC Demo project.
It supersedes all other practices and guidelines where conflicts exist.

**Amendment procedure**:

- Any deviation from these standards MUST be explicitly documented, reviewed,
  approved, and justified before implementation
- Amendments MUST increment the version according to semantic versioning:
  MAJOR — principle removal or backward-incompatible redefinition;
  MINOR — new principle or section added, or material expansion of guidance;
  PATCH — clarifications, wording, or non-semantic refinements
- `LAST_AMENDED_DATE` MUST be updated on every amendment to the current ISO date

**Compliance**:

- All Pull Requests and code reviews MUST verify compliance with this constitution
- The Constitution Check gate in `plan.md` MUST pass before Phase 0 research begins
- Complexity violations MUST be documented in the Complexity Tracking table of `plan.md`

**Version**: 2.0.0 | **Ratified**: 2026-06-17 | **Last Amended**: 2026-06-18
