# VesselBridge

VesselBridge provides a comprehensive digital platform for vessel operations, compliance, and crew welfare, integrating real-time bridge signals with regulatory requirements and operational tools.

## Run & Operate

- **Run API server locally**: `pnpm --filter @workspace/api-server run dev`
- **Typecheck all packages**: `pnpm run typecheck`
- **Build all packages**: `pnpm run build`
- **Regenerate API client from OpenAPI spec**: `pnpm --filter @workspace/api-spec run codegen`
- **Push DB schema changes**: `pnpm --filter @workspace/db run push` (development only)

**Required Environment Variables**:
- `API_KEY`: Shared secret for API authentication (min 32 chars).
- `VITE_API_KEY`: Frontend API key (must match `API_KEY`).
- `REPLIT_DOMAINS`: Comma-separated list of Replit domains for CORS allowlist.
- `CORS_ALLOWED_ORIGINS`: (Optional) Additional domains for CORS allowlist.

## Stack

- **Monorepo**: pnpm workspaces
- **Node.js**: 24
- **TypeScript**: 5.9
- **API Framework**: Express 5
- **Database**: PostgreSQL
- **ORM**: Drizzle ORM
- **Validation**: Zod, drizzle-zod
- **API Codegen**: Orval (from OpenAPI spec)
- **Build Tool**: esbuild

## Where things live

- **API Server Routes**: `artifacts/api-server/src/routes/`
- **Database Schema**: `lib/db/src/schema/vessels.ts` (for vessels)
- **OpenAPI Specification**: `artifacts/api-spec/openapi.yaml`
- **Bridge Signal Bus (Source of Truth)**: `artifacts/vessel-docs/src/lib/bridge-signal-bus.ts`
- **Compliance Catalogs**: `artifacts/vessel-docs/src/lib/*-catalog.ts` (e.g., `bwm-catalog.ts`, `sopep-catalog.ts`, `medical-catalog.ts`)
- **Clinical Decision Support**: `artifacts/vessel-docs/src/lib/triage-engine.ts` (8 symptom-driven protocols, NEWS2 / GCS / Rule of 9s / Parkland / dosing helpers, KOTC HSE-MED-01..10 forms register) + `artifacts/vessel-docs/src/pages/medical-clinical.tsx` (TriageTab, AssessmentTab, KotcFormsTab)
- **Inventory Brain (Ship Hospital)**: `artifacts/vessel-docs/src/lib/medical-inventory-engine.ts` (batch / consumption / attachment types, par+rate×days reorder algorithm, INDUSTRY_BULLETINS + TRUSTED_MEDICAL_SOURCES, KOTC HSE-MED-06/07 + RFQ HTML builders) + `artifacts/vessel-docs/src/pages/medical-inventory.tsx` (InventoryTab consumed by `medical.tsx`)
- **Office Bulletins & Circulars**: `artifacts/vessel-docs/src/lib/office-bulletins-engine.ts` (Bulletin / BulletinAck / BulletinAttachment types, ALL_RANKS, severity helpers, SEED_BULLETINS) + `artifacts/vessel-docs/src/pages/office-bulletins.tsx` (route `/office/bulletins`)
- **Standing Orders (per rank)**: `artifacts/vessel-docs/src/lib/standing-orders-catalog.ts` (RankStandingOrderSpec for 12 ranks with KOTC mandatory topics) + `artifacts/vessel-docs/src/pages/standing-orders.tsx` (route `/standing-orders`, per-rank tabs, coverage panel, ack ledger, print)
- **Master's Document Brain**: `artifacts/vessel-docs/src/lib/master-brain-engine.ts` (heuristic extractor — ISO/human dates, IMO, cert numbers, keywords; rules engine for expiry / missing / stale / conflict / process / vessel mismatch / crew gaps; BrainAlert + summarise) + `artifacts/vessel-docs/src/pages/master-brain.tsx` (route `/master-brain`, prioritised alerts panel + per-category extracted-fields tables)
- **KOTC HSE Catalog**: `artifacts/vessel-docs/src/lib/kotc-hse-catalog.ts` (KOTC_RANKS / KOTC_RANK_GROUPS / KOTC_SECTIONS — Office Manual + PAG + Forms + Shipboard sections with role tagging, refresh cadence, source-PDF refs) + `artifacts/vessel-docs/src/pages/kotc-hse.tsx` (route `/kotc-hse`, role+search filter, sections/forms tabs, per-rank). Source PDFs mirrored under `artifacts/vessel-docs/public/kotc/`. Master Brain's `REQUIRED_DOC_KINDS` cites OM/PAG refs from this catalog.
- **API Client**: `lib/api-client-react/src/`
- **Frontend Pages**: `artifacts/vessel-docs/src/pages/`
- **Frontend Components**: `artifacts/vessel-docs/src/components/`
- **Security Middleware**: `artifacts/api-server/src/middleware/auth.ts`, `artifacts/api-server/src/middleware/rateLimiter.ts`

## Architecture decisions

- **Monorepo Structure**: Uses pnpm workspaces for managing multiple packages, each with its own dependencies, ensuring clear separation of concerns.
- **Client-Side Per-Vessel State**: For Ship Hospital, state is persisted in client-side storage (`localStorage`) per-vessel, avoiding immediate backend persistence and simplifying multi-vessel isolation.
- **Configurable External Data Sources**: RSS feeds for compliance alerts and weather data sources are externalized and configurable (e.g., `TRUSTED_FEEDS`, `BWM_KEYWORDS`, `TRUSTED_SOURCES` for SOPEP), allowing easy updates without code changes.
- **Generic Record Book Engine**: A single `record-book-engine.ts` handles generic record book logic, persistence, and pre-filling from bridge signals, abstracting away domain-specific differences.
- **Bridge Signal Bus Abstraction**: All consumers of vessel signals (Navigation, Echo Sounder, BWM, SOPEP, Weather) read from a single `bridge-signal-bus.ts`, making it the sole integration point for real NMEA data.

## Product

- **Compliance Modules**: Ballast Water Management (BWM) and SOPEP/SMPEP with live trusted-source alerts, KOTC HSE forms, and record books.
- **Digital Record Books**: IMO-compliant Garbage and Ballast Water Record Books with auto-prefill from bridge signals, officer/master signatures, and amendment alerts.
- **Weather Bridge**: Integrates trusted weather feeds, provides 7-day forecasts, operational advisories (GO/CAUTION/HOLD), and abnormal weather alerts.
- **Ship Hospital**: Dashboard for medical readiness, medicine chest audit, equipment checks, crew health records, sick-bay log, controlled drug register, LPG-specific MFAG protocols, and medical drill tracking. Includes a **Clinical Decision Support** layer with symptom-driven Triage (8 KOTC-aligned protocols: chest pain, abdo pain, trauma, burn, anaphylaxis, heat stroke, severe SOB, suspected stroke), Patient Assessment (ABCDE survey + live NEWS2 / GCS / Wallace's rule of 9s + Parkland / weight-based dosing helper auto-filled from Bridge Signal Bus), and the full KOTC HSE-MED-01..10 forms register with one-click Sick-Bay log + MEDEVAC packet generation.
- **Vessel Management**: Multi-vessel support with isolated client-side state, active vessel switching, and a "Format brain" function for per-vessel data reset.
- **Bridge Signal Integration**: Real-time simulation of NMEA data for various vessel signals, forming a single source of truth for all modules.
- **Security Hardening**: API server incorporates Helmet, CORS allowlisting, rate limiting, secure authentication, and path traversal guards.

## User preferences

_Populate as you build_

## Gotchas

- **API Key Mismatch**: `API_KEY` (backend) and `VITE_API_KEY` (frontend) must be identical for the application to function.
- **DB Schema Push**: `pnpm --filter @workspace/db run push` is for development only; use proper migration strategies for production.
- **Vessel Isolation**: Formatting the brain (`VesselContext.formatBrain()`) only wipes data for the *active* vessel.
- **Backend Filtering**: Many backend routes currently lack `vesselId` filtering, relying on the `X-Vessel-Id` header to be consumed in future iterations.

## Pointers

- **pnpm workspaces**: [https://pnpm.io/workspaces](https://pnpm.io/workspaces)
- **TypeScript**: [https://www.typescriptlang.org/docs/](https://www.typescriptlang.org/docs/)
- **Express**: [https://expressjs.com/](https://expressjs.com/)
- **Drizzle ORM**: [https://orm.drizzle.team/](https://orm.drizzle.team/)
- **Zod**: [https://zod.dev/](https://zod.dev/)
- **Orval**: [https://orval.dev/](https://orval.dev/)
- **Helmet**: [https://helmetjs.github.io/](https://helmetjs.github.io/)
- **IMO MEPC Regulations**: Refer to relevant IMO MEPC documents for specifics on BWM, SOPEP, GRB, BWRB.
- **KOTC HSE-MS**: Refer to KOTC Health, Safety, and Environment Management System documentation for internal requirements.
- **Threat Model**: `threat_model.md` for known security weaknesses (e.g., shared bearer-key model).