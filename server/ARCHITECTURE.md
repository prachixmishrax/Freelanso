# Architecture and security review

## Runtime shape

The browser authenticates with Firebase Authentication and sends Firebase ID
tokens to Express. `src/middleware/auth.js` verifies tokens with revocation
checking enabled. Controllers use validated request data and shape DTO
responses; services own authorization and business rules; repositories are the
only application layer that accesses Firestore. The Admin SDK bypasses
Firestore client rules, so API authentication and service authorization remain
mandatory.

The API app is exported separately from `src/server.js` for Supertest. Runtime
middleware adds request IDs, redacted structured Pino logs, Helmet, exact
origin CORS, HTTPS enforcement in production, compression, 100 KB JSON limits,
and global plus endpoint-specific rate limits. HTTP server request timeouts and
graceful shutdown are configured in `src/server.js`.

Firestore client access is denied by default in `firestore.rules`. RTDB
messaging remains client-accessible only for canonical conversation
participants; the API creates conversations using verified identities.

## OWASP API Security Top 10 (2023) mapping

| Risk | Mitigations | Verification |
|---|---|---|
| API1: Broken Object Level Authorization | Ownership checks in project, service, job, application, conversation, dashboard, support, and withdrawal services use verified `req.user`; private records return not-found or forbidden. | `tests/projects.test.js`, `tests/services.test.js`, `tests/jobs.test.js`, emulator suites |
| API2: Broken Authentication | Firebase Admin verifies ID tokens with revocation checks; private routes require Bearer auth; optional auth is used only for documented public detail/support endpoints. | `tests/foundation.test.js`, domain tests, emulator suites |
| API3: Broken Object Property Level Authorization | Strict Zod schemas whitelist mutable fields; DTO mappers omit internal/private fields; Firestore rules deny all client writes. | Schema and DTO tests in domain suites; `tests/firestoreRules.emulator.test.js` |
| API4: Unrestricted Resource Consumption | Global/write/AI/social/support rate limiters, bounded page sizes/search lengths, request timeout, JSON size cap, bounded AI candidates, daily AI quota. | Domain tests and `tests/support.emulator.test.js`; `src/scripts/loadTest.js` |
| API5: Broken Function Level Authorization | `requireRole` gates admin-only support/withdrawal operations; services enforce role and ownership independently of route middleware. | `tests/users.test.js`, `tests/support.emulator.test.js`, `tests/withdrawals.emulator.test.js` |
| API6: Unrestricted Access to Sensitive Business Flows | Transactional deterministic IDs protect likes, applications, votes, withdrawal idempotency, and support idempotency; throttles protect write and social endpoints. | Project/jobs/battles/support/withdrawals tests |
| API7: Server Side Request Forgery | User-controlled URLs are not fetched by the API; uploads use a server-signed Cloudinary configuration and fixed provider endpoints; AI output is treated as data, not a URL/action. | `tests/uploads.test.js`, `tests/geminiAIProvider.test.js`, `tests/smartSearch.service.test.js` |
| API8: Security Misconfiguration | Zod-validated environment, no wildcard CORS, production HTTPS, Helmet, disabled `x-powered-by`, explicit proxy setting, deny-by-default database rules. | `tests/env.test.js`, `tests/foundation.test.js`, emulator rules tests |
| API9: Improper Inventory Management | `openapi.yaml` documents the current API; `npm run openapi:check` compares registered Express methods/paths to the spec. | OpenAPI parity check, CI |
| API10: Unsafe Consumption of APIs | Gemini output is schema-validated and restricted to Firestore candidate IDs; Cloudinary asset URLs are validated; provider calls use bounded input, timeout, and retries; secrets are redacted. | Search, upload, and provider tests |

### Remaining security limitations

- The API uses the Firebase Admin SDK and therefore bypasses client rules. A
  compromised server identity or service-layer authorization defect remains
  high impact.
- Firestore rules deny even client reads; all current Firestore reads must go
  through the API. RTDB rules intentionally permit conversation participants
  to write their own messages directly.
- Rate limiting uses in-process memory storage. It is not globally consistent
  across multiple Cloud Run instances; use a shared store if abuse thresholds
  must be enforced fleet-wide.
- Error reporting relies on Cloud Logging/Error Reporting recognition of
  sanitized structured error entries. No external error-reporting project or
  notification channel can be provisioned from this repository alone.
- The support endpoint accepts optional unauthenticated submissions and stores
  contact information. Retention and deletion policy must be set operationally.
- The signed Cloudinary upload is restricted by API signature and configured
  Cloudinary presets; the preset restrictions must be independently verified
  in the Cloudinary console.
- A non-Git workspace snapshot does not contain evidence of prior Git history.
  Secret scanning this filesystem cannot establish that a credential was never
  committed in another clone or remote.

## Endpoint, authorization, schema, and test inventory

All request schemas are strict Zod schemas in the named module. Routes also
validate empty body/query/params schemas where a field is not expected.
`Optional` means a Bearer token may add caller-specific response data but is
not required. `Public` means no identity is required. `User` means a valid
Firebase token; role and ownership constraints are additionally enforced in
services. `Admin` means a verified token with a server-controlled admin role.

| Method and path | Auth | Validation schema | Covering tests |
|---|---|---|---|
| `GET /healthz` | Public | `health` empty schema | `foundation.test.js` |
| `GET /readyz` | Public | `health` empty schema | `foundation.test.js` |
| `GET /api/me` | User | `me` empty schema | `foundation.test.js`, `users.test.js` |
| `GET /api/dashboard` | User | `dashboardQuerySchema` | `dashboard.test.js` |
| `GET /api/users/me` | User | `emptyObjectSchema` | `users.test.js` |
| `GET /api/users/{uid}` | Public | `uidParamsSchema` | `users.test.js` |
| `PUT /api/users/{uid}` | User; owner/admin | `uidParamsSchema`, `profileUpdateSchema` | `users.test.js` |
| `POST /api/users/onboarding` | User | `onboardingSchema` | `users.test.js` |
| `POST /api/uploads/sign` | User | `uploadSignatureSchema` | `uploads.test.js` |
| `GET /api/projects` | Public | `projectListQuerySchema` | `projects.test.js`, `emulator.test.js` |
| `GET /api/projects/mine` | User | `commentsQuerySchema` | `projects.test.js`, `emulator.test.js` |
| `POST /api/projects` | User; freelancer/admin | `projectCreateSchema` | `projects.test.js`, `emulator.test.js` |
| `GET /api/projects/{id}` | Optional | `projectIdParamsSchema` | `projects.test.js`, `emulator.test.js` |
| `PUT /api/projects/{id}` | User; owner/admin | `projectIdParamsSchema`, `projectUpdateSchema` | `projects.test.js`, `emulator.test.js` |
| `DELETE /api/projects/{id}` | User; owner/admin | `projectIdParamsSchema` | `projects.test.js`, `emulator.test.js` |
| `POST /api/projects/{id}/like` | User | `projectIdParamsSchema`, empty body | `projects.test.js`, `emulator.test.js` |
| `GET /api/projects/{id}/comments` | Public | `projectIdParamsSchema`, `commentsQuerySchema` | `projects.test.js`, `emulator.test.js` |
| `POST /api/projects/{id}/comments` | User | `projectIdParamsSchema`, `commentCreateSchema` | `projects.test.js`, `emulator.test.js` |
| `DELETE /api/projects/{id}/comments/{commentId}` | User; author/owner/admin | `commentParamsSchema` | `projects.test.js`, `emulator.test.js` |
| `GET /api/meta/categories` | Public | `emptyObjectSchema` | `services.test.js`, `emulator.test.js` |
| `GET /api/services` | Public | `serviceListQuerySchema` | `services.test.js`, `emulator.test.js` |
| `GET /api/services/{id}` | Public | `serviceIdParamsSchema` | `services.test.js`, `emulator.test.js` |
| `POST /api/services` | User; freelancer/admin | `serviceCreateSchema` | `services.test.js`, `emulator.test.js` |
| `PUT /api/services/{id}` | User; owner/admin | `serviceIdParamsSchema`, `serviceUpdateSchema` | `services.test.js`, `emulator.test.js` |
| `DELETE /api/services/{id}` | User; owner/admin | `serviceIdParamsSchema` | `services.test.js`, `emulator.test.js` |
| `GET /api/jobs` | Public | `jobListQuerySchema` | `jobs.test.js`, `emulator.test.js` |
| `GET /api/jobs/{id}` | Public | `jobIdParamsSchema` | `jobs.test.js`, `emulator.test.js` |
| `POST /api/jobs` | User; client/admin | `jobCreateSchema` | `jobs.test.js`, `emulator.test.js` |
| `PUT /api/jobs/{id}` | User; owner/admin | `jobIdParamsSchema`, `jobUpdateSchema` | `jobs.test.js`, `emulator.test.js` |
| `DELETE /api/jobs/{id}` | User; owner/admin | `jobIdParamsSchema` | `jobs.test.js`, `emulator.test.js` |
| `POST /api/jobs/{id}/applications` | User; freelancer | `jobIdParamsSchema`, `applicationCreateSchema` | `jobs.test.js`, `emulator.test.js` |
| `GET /api/jobs/{id}/applications` | User; job owner | `jobIdParamsSchema`, `applicationListQuerySchema` | `jobs.test.js`, `emulator.test.js` |
| `GET /api/applications/me` | User | `applicationListQuerySchema` | `jobs.test.js`, `emulator.test.js` |
| `PATCH /api/applications/{id}/status` | User; applicant/job owner | `applicationIdParamsSchema`, `applicationStatusSchema` | `jobs.test.js`, `emulator.test.js` |
| `GET /api/battles` | Public | `battleListQuerySchema` | `battles.emulator.test.js` |
| `GET /api/battles/{id}` | Optional | `battleIdParamsSchema` | `battles.emulator.test.js` |
| `GET /api/battles/{id}/user-vote` | User | `battleIdParamsSchema` | `battles.emulator.test.js` |
| `POST /api/battles` | User | `battleCreateSchema` | `battles.emulator.test.js` |
| `POST /api/battles/{id}/vote` | User | `battleIdParamsSchema`, `battleVoteSchema` | `battles.emulator.test.js` |
| `POST /api/search/smart` | User | `smartSearchSchema` | `smartSearch.service.test.js`, `searchProvider.test.js`, `aiMatching.emulator.test.js` |
| `POST /api/matches/score-all` | User; authorized project context | `matchScoreAllSchema` | `matches.scoring.test.js`, `aiMatching.emulator.test.js` |
| `POST /api/support/tickets` | Optional | `supportTicketCreateSchema`, `Idempotency-Key` header | `support.emulator.test.js` |
| `GET /api/support/tickets/me` | User | `supportTicketPageSchema` | `support.emulator.test.js` |
| `GET /api/support/tickets` | Admin | `supportTicketPageSchema` | `support.emulator.test.js` |
| `PATCH /api/support/tickets/{id}/status` | Admin | `supportTicketParamsSchema`, `supportTicketStatusSchema` | `support.emulator.test.js` |
| `POST /api/conversations` | User; caller is a participant | `createConversationSchema` | `conversations.emulator.test.js` |
| `GET /api/wallet` | User | `withdrawalEmptySchema` | `withdrawals.emulator.test.js` |
| `GET /api/withdrawals/me` | User | `withdrawalListQuerySchema` | `withdrawals.emulator.test.js` |
| `POST /api/withdrawals` | User; `Idempotency-Key` required | `withdrawalCreateSchema` | `withdrawals.emulator.test.js` |
| `PATCH /api/withdrawals/{id}/status` | Admin | `withdrawalParamsSchema`, `withdrawalStatusSchema` | `withdrawals.emulator.test.js` |

The automated OpenAPI check compares the 50 registered Express operations to
the YAML `paths` keys/methods. Request/response schema details remain
hand-maintained and are reviewed in `openapi.yaml`.
