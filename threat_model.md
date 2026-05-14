# Threat Model

## Project Overview

VesselBridge is a maritime document-management and compliance application built as a pnpm TypeScript monorepo. The production surface consists of a React frontend in `artifacts/vessel-docs` and an Express 5 API in `artifacts/api-server`, backed by PostgreSQL through Drizzle ORM in `lib/db`. The API stores and processes vessel documents, crew records, certificates, activity logs, and AI-generated form-fill outputs. The `artifacts/mockup-sandbox` artifact is a development-only sandbox and is out of scope unless separately proven production-reachable.

The current production auth implementation is a single shared bearer key checked by `artifacts/api-server/src/middleware/auth.ts` and mirrored into the browser via `VITE_API_KEY` in `artifacts/vessel-docs/src/main.tsx`. The repository also contains a stronger OIDC/session flow in `artifacts/api-server/src/routes/auth.ts`, but it is not mounted by the production router.

## Assets

- **Maritime compliance documents** — uploaded files, document metadata, extracted fields, and validation results in `documents`. These can contain operational, regulatory, and voyage-sensitive data.
- **Crew personal data** — names, nationalities, passport numbers, certificate and medical expiry dates in `crew`. Exposure affects privacy and may support identity fraud.
- **Certificate records** — issuance, expiry, vessel association, and certificate numbers in `certificates`. Tampering can falsify compliance state.
- **Generated form-fill outputs** — AI-produced completed forms and embedded file payloads in `form_fills`. These may reproduce sensitive source content and operator-entered answers.
- **Audit/activity history** — operational history in `activity`. Integrity matters for compliance, investigations, and user accountability.
- **Application secrets and paid integrations** — PostgreSQL connection string and OpenAI integration credentials in environment variables. Abuse can expose data or generate external cost.
- **Server filesystem contents** — uploaded files under `artifacts/api-server/uploads` and any host-accessible files reachable by backend code. Unauthorized reads could disclose secrets or private documents.

## Trust Boundaries

- **Browser to API** — all client input crossing into `artifacts/api-server/src/routes/*` is untrusted. The API must authenticate callers, authorize every operation, validate all parameters, and bound resource usage.
- **API to PostgreSQL** — the API has broad read/write access to the application database via `lib/db/src/index.ts`. Route handlers must prevent unauthorized reads, writes, and destructive actions.
- **API to filesystem** — upload and form-fill flows read and write local files under the server working directory. Database-stored or request-derived paths must not escape intended directories.
- **API to OpenAI integration** — `/forms/*` and `/intelligence/answer` send selected document content to the OpenAI integration using server-side credentials. Only authorized callers should be able to trigger those calls or disclose source context.
- **Public vs authenticated operator boundary** — the frontend exposes role-oriented shells, but role selection in the client is not a security boundary. Real enforcement must exist in the backend. Under the current implementation, any party that can recover the browser-shipped shared API key effectively crosses this boundary.
- **Production vs dev-only boundary** — `artifacts/mockup-sandbox` is not production. Scan effort should stay focused on `artifacts/api-server`, `artifacts/vessel-docs`, and shared libraries they import.

## Scan Anchors

- **Production API entry points:** `artifacts/api-server/src/app.ts`, `artifacts/api-server/src/routes/*.ts`
- **Highest-risk code areas:** `artifacts/api-server/src/middleware/auth.ts`, `artifacts/api-server/src/routes/forms.ts`, `upload.ts`, `documents.ts`, `export.ts`, `intelligence.ts`, and shared DB access in `lib/db/src/index.ts`
- **Public/authenticated/admin surfaces:** the current API key model should be treated as effectively public because the frontend ships the same bearer secret to browsers; the frontend role system in `artifacts/vessel-docs` is presentation-only and must not be treated as authorization
- **Dormant but security-relevant code:** `artifacts/api-server/src/routes/auth.ts` contains a stronger auth flow but is currently unmounted from `routes/index.ts`
- **Dev-only areas usually ignored:** `artifacts/mockup-sandbox/**`, most generated `dist/**` outputs unless validating compiled reachability

## Mitigations Currently In Place

The following defences are implemented in the production API. They do **not** replace the need to mount the OIDC/session flow in `routes/auth.ts`, but they materially raise the cost of every category of attack listed below.

- **HTTP security headers** — Helmet with strict CSP (`default-src 'none'`, JSON-only API), HSTS (1 yr, includeSubDomains), `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, frameguard deny, COOP `same-origin`, CORP `same-site`, `x-powered-by` disabled. See `artifacts/api-server/src/app.ts`.
- **CORS allowlist** — only same-origin + explicit `REPLIT_DOMAINS` + opt-in `CORS_ALLOWED_ORIGINS`. All other origins rejected before reaching any route.
- **`trust proxy = 1`** — a single Replit hop is trusted so per-IP rate limits and auth lockout state see the real client IP rather than the loopback proxy.
- **Tiered rate limiting** — global 600 req/min per IP, AI endpoints 10/min, export 20/min, upload 300/min. See `artifacts/api-server/src/middleware/rateLimiter.ts`.
- **Constant-time bearer comparison** — `crypto.timingSafeEqual` in `middleware/auth.ts` defeats timing oracles; the middleware also fails closed if `API_KEY` is missing or shorter than 32 chars.
- **Per-IP exponential lockout** — 8 failed bearer attempts trigger a lockout that doubles each additional failure up to 1 h. Successful auth resets the counter. This bounds credential stuffing / brute-force even if part of the shared key leaks.
- **Path-traversal guard** — `lib/safe-path.ts` (`safeJoin`) is the single source of truth for joining user/DB-controlled values onto upload paths. Rejects `..`, absolute-path injection, NUL bytes. The existing `path.resolve` + prefix check in `routes/forms.ts` and `routes/upload.ts` should be migrated to call `safeJoin` so the policy lives in one place.
- **Tight body parsers** — JSON 1 MB / urlencoded 1 MB. File uploads go through multer with its own per-file cap.
- **Log redaction** — `req.headers.authorization`, `req.headers.cookie` and `res.headers['set-cookie']` are redacted in `lib/logger.ts` so secrets cannot leak into log aggregation.
- **Generic JSON error handler** — never leaks stack traces or internal error messages to clients; full error logged server-side with request id.

### Residual risks still open

- **Shared bearer model.** The frontend ships `VITE_API_KEY` to every browser. Any user able to read the bundle effectively crosses the operator/public boundary. Mitigation: mount the OIDC/session flow in `routes/auth.ts` and remove `VITE_API_KEY` from the frontend.
- **No object-level authorisation.** Once a request is authenticated it can read or mutate any document, certificate, crew record or activity entry. Mitigation: bind every record to an authenticated identity and enforce per-record ACL in the routes.
- **Audit actor fields are still client-supplied** in some paths (e.g. `uploadedBy`, `performedBy`). Mitigation: derive these server-side from the authenticated identity once OIDC is mounted.
- **In-process state** — rate-limit counters and auth lockouts reset on every server restart. Acceptable for the current single-instance deployment but must move to Redis or PostgreSQL if the API ever scales horizontally.

## Threat Categories

### Spoofing

The backend currently serves role-oriented operational workflows, but operator identity is not inherently trustworthy when supplied by the client. All API endpoints that access or mutate operational data MUST require a verifiable server-side identity, and any claimed role or actor name MUST be derived from that identity rather than request body fields or frontend route selection. A shared bearer key compiled into the frontend does not satisfy this guarantee because any browser user can recover and replay it.

### Tampering

The application allows creation, update, deletion, validation, export, and AI-assisted processing of documents, certificates, crew data, and activity records. The server MUST enforce object-level authorization on every mutation, must reject database-controlled or request-controlled filesystem paths that escape approved storage locations, and MUST not allow client input to rewrite compliance state, audit context, or stored file references without authorization. Audit actor fields such as `uploadedBy` and `performedBy` must come from a trusted authenticated identity rather than free-form request data.

### Information Disclosure

The system stores crew PII, compliance documents, validation metadata, audit history, and generated form outputs. API responses, exports, AI context assembly, and filesystem reads MUST be scoped to authorized users only; sensitive fields MUST not be exposed to callers who merely know a shared frontend secret; and local-file access MUST remain confined to intended upload directories.

### Denial of Service

The production API exposes upload, parsing, export, validation, and AI-backed endpoints that can consume disk, memory, CPU, database capacity, and paid third-party quota. Publicly reachable endpoints MUST enforce authentication or strong abuse controls, apply request-size and concurrency limits appropriate to the operation, and avoid unauthenticated fan-out to expensive AI or bulk-processing workflows. Per-IP rate limits are useful but are not a substitute for real caller identity when the bearer credential is shared across all users.

### Elevation of Privilege

The frontend role model does not protect backend resources, so an attacker who can reach the API may attempt privileged operations directly. The backend MUST not rely on client-side role routing for authorization, MUST apply per-route and per-record access control, and MUST prevent secondary privilege escalation paths such as rewriting stored file references to read arbitrary server files or invoking export endpoints that aggregate all records. Any user able to replay the browser-shipped shared key should be assumed to have broad API access unless stronger server-side authorization is added.
