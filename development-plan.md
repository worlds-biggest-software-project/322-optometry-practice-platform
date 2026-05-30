# Optometry Practice Platform — Phased Development Plan

> Project: 322-optometry-practice-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 2 (Hybrid Relational + JSONB)** as the primary schema: the patient → appointment → encounter → dispensing → claim pipeline stays in typed relational columns for integrity, while variable clinical and product data (exam findings, device results, lens configurations, payer-specific fields) live in GIN-indexed JSONB. This balances HIPAA-grade referential integrity with the adaptability an AI-native MVP needs to absorb new exam types and evolving diagnostic instruments without schema migrations. Where analytics demand columnar aggregation (IOP trending, denial-rate reporting), read-side materialised views derived from the JSONB are introduced in later phases — borrowing the strengths of Suggestion 1 without its 22-table maintenance cost.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The product is API + multi-role web UI heavy with light ML orchestration (LLM calls, not training). One language across server and client reduces context-switching and lets FHIR resource types be shared between API and frontend. The healthcare integration ecosystem (FHIR clients, X12 parsers) has mature TS/JS libraries. |
| API framework | Fastify 5 + `@fastify/swagger` | Fastify's JSON-schema-first routing generates OpenAPI 3.1 automatically (a `standards.md` requirement) and validates request/response bodies natively. Higher throughput than Express for the high-concurrency portal + device-webhook surface. |
| Validation / types | Zod 3 + `fastify-type-provider-zod` | Single source of truth for runtime validation and compile-time types. Zod schemas double as the JSONB structure validators that the hybrid data model requires at the application layer. |
| Database | PostgreSQL 16 | The hybrid model depends on native JSONB + GIN `jsonb_path_ops` indexing and `PARTITION BY RANGE` for the audit log — both Postgres-specific. Strong transactional integrity is mandatory for the claim lifecycle. Row-level security is used to enforce practice-level tenant isolation. |
| ORM / query layer | Drizzle ORM | Type-safe SQL with first-class JSONB column typing and explicit migrations (no hidden magic — important for a regulated audit trail). Generates TS types from the schema that align with the Zod validators. |
| Migrations | drizzle-kit | Deterministic, reviewable SQL migrations checked into git — required for ONC/HIPAA change-control evidence. |
| Task queue | BullMQ on Redis 7 | Async workloads dominate: EDI submission/polling, e-prescribing round-trips, recall messaging, LLM scribe transcription, device-message ingestion. BullMQ gives retries, scheduled (delayed) jobs for recalls, and rate limiting for payer/clearinghouse APIs. |
| Cache / sessions | Redis 7 | Shared with BullMQ; backs session store, eligibility-response caching, and idempotency keys for EDI submissions. |
| Frontend | Next.js 16 (App Router) + React 19 + TypeScript | Server components for fast role-based dashboards; client components for the live exam charting surface. App Router middleware enforces auth and MFA gates. shadcn/ui + Tailwind for an accessible, modern UI (the explicit gap versus legacy incumbents). |
| Auth (staff) | Auth.js (NextAuth) credential + WebAuthn provider | WebAuthn (W3C, per `standards.md`) gives phishing-resistant MFA expected of HIPAA-aligned systems. Sessions are JWT (RFC 7519) backed by Redis for revocation. |
| Auth (API / third-party) | SMART on FHIR (OAuth 2.0 + PKCE, RFC 6749/7519) | Required for the FHIR R4 API surface. Implemented with `node-oauth2-server` semantics over Fastify; scopes follow SMART `patient/*.read` conventions. |
| LLM provider | Anthropic Claude (via `@anthropic-ai/sdk`) behind a provider abstraction | The AI-native differentiators (ambient scribe, coding suggestions, denial prediction) need a strong instruction-following model with structured-output (tool) support. The abstraction allows swapping/failover. Prompt caching is used for the static clinical-rules system prompt. |
| Speech-to-text | Deepgram (streaming) behind an STT abstraction | Real-time ambient scribe transcription with medical vocabulary support; abstraction permits a self-hosted Whisper fallback for self-hosted deployments. |
| FHIR | `@medplum/fhirtypes` + custom resource mappers | Provides validated FHIR R4 + US Core typings without adopting a full server we cannot certify; mappers convert internal JSONB to `VisionPrescription`, `Observation`, `Claim`, etc. |
| EDI (X12) | `node-x12` + custom 837P/835/270-271 builders/parsers | No turnkey vision-EDI library exists; X12 segment-level library plus payer-profile config (VSP, EyeMed, Davis) handles the wire format. |
| HL7 v2 | `simple-hl7` | Parses inbound ORU/MDM device messages from the device-adapter gateway. |
| Object storage | S3-compatible (AWS S3 / MinIO self-hosted) | Stores DICOM image blobs, scanned intake forms, and ERA files. Encrypted with SSE-KMS; objects referenced by URI from JSONB `device_results`. |
| Containerisation | Docker + docker-compose (dev), Helm chart (prod) | Self-hostable requirement from README; compose stack runs api, worker, web, postgres, redis, minio locally. |
| Testing | Vitest (unit/integration) + Playwright (E2E) + Testcontainers | Vitest for fast unit/integration with mocked externals; Testcontainers spins real Postgres/Redis for integration; Playwright drives multi-role browser flows. |
| Code quality | Biome (lint + format) + `tsc --noEmit` | Single fast tool for lint+format; strict TypeScript catches data-model drift. |
| Package manager | pnpm (workspace monorepo) | Workspaces for `apps/api`, `apps/web`, `apps/worker`, `packages/db`, `packages/fhir`, `packages/edi`, `packages/ai`, `packages/core` — shared types without publishing. |
| Audit / security | OWASP ASVS Level 2 checklist, RLS, field-level encryption | ASVS L2 (per `standards.md`) for PHI-handling apps; pgcrypto/app-layer encryption for `ssn_last4` and similar; tenant isolation via Postgres RLS. |

### Project Structure

```
optometry-practice-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml                # postgres, redis, minio, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── deploy/
│   └── helm/optometry-platform/      # production Helm chart
├── packages/
│   ├── core/                         # shared domain types, errors, result helpers, config loader
│   │   └── src/{config.ts,errors.ts,result.ts,types/*.ts}
│   ├── db/                           # Drizzle schema, migrations, RLS policies, seed
│   │   ├── src/schema/{practices.ts,patients.ts,appointments.ts,encounters.ts,dispensing.ts,claims.ts,audit.ts,ai.ts}
│   │   ├── src/jsonb/                # Zod schemas validating each JSONB column shape
│   │   ├── migrations/
│   │   └── src/{client.ts,rls.ts,seed.ts}
│   ├── fhir/                         # FHIR R4 / US Core mappers + SMART scopes
│   │   └── src/{mappers/*.ts,scopes.ts}
│   ├── edi/                          # X12 837P/835/270-271 builders & parsers, payer profiles
│   │   └── src/{x837p.ts,x835.ts,x270_271.ts,payers/*.ts}
│   ├── ai/                           # LLM + STT abstractions, prompts, structured-output schemas
│   │   └── src/{llm.ts,stt.ts,prompts/*.ts,schemas/*.ts}
│   └── ui/                           # shared shadcn/ui components, design tokens
├── apps/
│   ├── api/                          # Fastify server
│   │   └── src/{server.ts,plugins/*.ts,routes/*.ts,services/*.ts,middleware/*.ts}
│   ├── worker/                       # BullMQ processors
│   │   └── src/{index.ts,queues/*.ts,processors/*.ts}
│   ├── web/                          # Next.js 16 app
│   │   └── src/app/{(staff)/,(patient)/,api/auth/,middleware.ts}
│   └── device-gateway/               # HL7 v2 / DICOM ingest listener
│       └── src/{mllp-server.ts,dicom-scp.ts,normaliser.ts}
└── test/
    ├── fixtures/                     # sample HL7 ORU, X12 837P/835, FHIR bundles, exam JSON
    └── e2e/                          # Playwright specs
```

The structure is grouped by concern, not by phase. Each phase adds files within these directories without restructuring.

---

## Phase 1: Foundation, Tenancy & Security Spine

### Purpose
Establish the monorepo, database with multi-tenant isolation, configuration, and the cross-cutting security primitives (audit logging, encryption, RLS) that every later phase depends on. HIPAA and OWASP ASVS L2 obligations are foundational, not retrofitted — so the audit log and access-control middleware ship first. After this phase the system can authenticate a staff user, scope all queries to a practice, and record every PHI access.

### Tasks

#### 1.1 — Monorepo, tooling, and config loader

**What**: Initialise the pnpm workspace, Biome, TypeScript project references, Docker compose stack, and a typed config loader.

**Design**:
- `packages/core/src/config.ts` loads and validates environment with Zod:
```ts
export const ConfigSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  S3_ENDPOINT: z.string().url(),
  S3_BUCKET: z.string(),
  S3_ACCESS_KEY: z.string(),
  S3_SECRET_KEY: z.string(),
  JWT_SIGNING_KEY: z.string().min(32),
  FIELD_ENCRYPTION_KEY: z.string().length(64), // hex, 32 bytes
  ANTHROPIC_API_KEY: z.string().optional(),
  DEEPGRAM_API_KEY: z.string().optional(),
  PORT: z.coerce.number().default(3001),
});
export type Config = z.infer<typeof ConfigSchema>;
export const loadConfig = (env = process.env): Config => ConfigSchema.parse(env);
```
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, plus build targets for `api`, `worker`, `web`.
- Biome config enforces format on commit; `tsc --noEmit` runs in CI per package.

**Testing**:
- `Unit: loadConfig with complete env → Config object with defaults applied`
- `Unit: loadConfig missing DATABASE_URL → ZodError naming "DATABASE_URL"`
- `Unit: loadConfig FIELD_ENCRYPTION_KEY of wrong length → ZodError`
- `Integration: docker compose up → all containers report healthy within 60s`

#### 1.2 — Database schema, JSONB validators, and migrations

**What**: Implement the hybrid schema from Data Model Suggestion 2 as Drizzle tables with Zod validators for every JSONB column, plus the initial migration.

**Design**:
- Relational tables (typed columns): `practices`, `providers`, `patients`, `appointments`, `encounters`, `dispensing_orders`, `claims`, `frame_inventory`, `audit_log` (range-partitioned by `created_at`), `ai_analyses`.
- JSONB columns and their Zod validators in `packages/db/src/jsonb/`:
  - `practices.settings` → `PracticeSettingsSchema` (locations[], exam_templates, fee_schedule[], insurance_plans[])
  - `providers.credentials` → `ProviderCredentialsSchema` (npi, license_number, license_state, dea_number, surescripts_spi, location_ids[], specialties[])
  - `patients.contact|clinical_history|insurance` → `ContactSchema`, `ClinicalHistorySchema`, `InsuranceArraySchema`
  - `encounters.exam_data|diagnoses|prescriptions|device_results|e_prescriptions` → `ExamDataSchema`, `DiagnosisArraySchema`, `PrescriptionArraySchema`, `DeviceResultArraySchema`, `EPrescriptionArraySchema`
  - `dispensing_orders.specs` → `DispensingSpecsSchema`
  - `claims.payer|service_lines|denial|era_835` → `PayerSchema`, `ServiceLineArraySchema`, `DenialSchema`, `Era835Schema`
- Example validator (refraction-bearing prescription element):
```ts
const EyeRxSchema = z.object({
  sphere: z.number().multipleOf(0.25).min(-30).max(30).nullable(),
  cylinder: z.number().multipleOf(0.25).min(-12).max(12).nullable(),
  axis: z.number().int().min(0).max(180).nullable(),
  add: z.number().multipleOf(0.25).min(0).max(4).nullable(),
});
export const PrescriptionSchema = z.object({
  id: z.string().uuid(),
  type: z.enum(['eyeglasses', 'contact_lenses']),
  prescribed_at: z.string().datetime(),
  od: EyeRxSchema.extend({ bc: z.number().optional(), diameter: z.number().optional(), brand: z.string().optional() }),
  os: EyeRxSchema.extend({ bc: z.number().optional(), diameter: z.number().optional(), brand: z.string().optional() }),
  pd: z.object({ distance: z.number(), near: z.number().optional() }).optional(),
  expiration_date: z.string().date(),
});
```
- Indexes per Suggestion 2: GIN `jsonb_path_ops` on `patients.insurance`, `encounters.diagnoses`, `encounters.device_results`, `claims.payer`; btree on name/dob, appointment time, claim status partial index.
- All timestamps `TIMESTAMPTZ` (ISO 8601); all money as `BIGINT` cents (avoid float).
- A `withJsonbValidation()` helper validates JSONB through the Zod schema before any insert/update — the application-layer guarantee the hybrid model requires.

**Testing**:
- `Unit: PrescriptionSchema valid eyeglasses object → parses`
- `Unit: PrescriptionSchema sphere = -2.30 (not multiple of 0.25) → ZodError`
- `Unit: ExamDataSchema with unknown section key → allowed (passthrough) but typed sections validated`
- `Integration (Testcontainers Postgres): run migration → all tables/indexes/partitions exist (query pg_indexes, pg_partitions)`
- `Integration: insert encounter with malformed device_results via withJsonbValidation → rejected before SQL`

#### 1.3 — Multi-tenant isolation via Row-Level Security

**What**: Enforce practice-level data isolation so a query can never read another practice's PHI.

**Design**:
- Every tenant table carries `practice_id`. Enable RLS and add policy:
```sql
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON patients
  USING (practice_id = current_setting('app.practice_id')::uuid);
```
- `packages/db/src/client.ts` exposes `withTenant(practiceId, fn)` that opens a transaction and runs `SET LOCAL app.practice_id = $1` before executing `fn`. All request-scoped DB access goes through this.
- A non-superuser DB role (`app_user`) is used at runtime so RLS is never bypassed.

**Testing**:
- `Integration: insert patients for practice A and B; query within withTenant(A) → only A's rows returned`
- `Integration: attempt cross-tenant read by passing A's id while bound to B → zero rows`
- `Integration: connection without app.practice_id set → query errors (no implicit all-access)`

#### 1.4 — Audit logging and field encryption

**What**: Append-only PHI audit trail and transparent encryption of sensitive small fields.

**Design**:
- `audit_log` (partitioned) written via a `recordAudit({practiceId, userId, action, resourceType, resourceId, details, ipAddress})` service. Actions enum: `view | create | update | delete | export | login | login_failed | print`.
- A Fastify `onResponse` hook auto-logs `view` for any successful `GET /patients/:id`-style PHI route; mutations log explicitly within their service.
- Field encryption util in `packages/core`: AES-256-GCM using `FIELD_ENCRYPTION_KEY`, applied to `patients` SSN-last4 and emergency contact before persistence; returns `{ciphertext, iv, tag}` packed base64.
- A monthly cron (BullMQ repeatable job) pre-creates the next partition.

**Testing**:
- `Unit: encrypt then decrypt → original plaintext; tampered tag → throws`
- `Integration: GET /patients/:id → exactly one audit row with action=view, correct resource_id`
- `Integration: audit_log is append-only (UPDATE/DELETE blocked by trigger) → error`
- `Integration: partition cron for next month → new partition table exists`

#### 1.5 — Staff authentication, WebAuthn MFA, RBAC

**What**: Login, session issuance, WebAuthn second factor, and role-based authorisation middleware.

**Design**:
- Auth.js in `apps/web` with credentials + WebAuthn providers; session JWT (RFC 7519) signed with `JWT_SIGNING_KEY`, session record in Redis for revocation and automatic logoff (15-min idle, HIPAA technical safeguard).
- Roles (from schema CHECK): `optometrist | ophthalmologist | optician | technician | receptionist | billing | admin`.
- `requireRole(...roles)` Fastify preHandler; `requireScope(...)` for API tokens. A capability matrix maps role → allowed actions (e.g. only `optometrist|ophthalmologist` may `sign` an encounter; only `billing|admin` may `submit` claims).
- WebAuthn registration/assertion via `@simplewebauthn/server`; credentials stored per provider.

**Testing**:
- `Integration (mocked): valid password, registered passkey assertion → session issued`
- `Integration: valid password, missing/invalid passkey → 401, no session`
- `Integration: receptionist calls POST /encounters/:id/sign → 403`
- `Integration: session idle > 15 min → next request 401 (auto logoff)`
- `Unit: requireRole capability matrix denies billing from signing encounter`

### Definition of Done
All Phase-1 DoD checklist items (see end) pass; a seeded admin can log in with passkey, every PHI read produces an audit row, and cross-tenant access is provably impossible.

---

## Phase 2: Core Domain — Patients, Scheduling, Encounters

### Purpose
Build the relational spine of the practice: patient records, the appointment calendar, and the encounter (visit) document that all clinical and billing work hangs off. This is the heart of the EHR and the substrate for the exam workflow in Phase 3. After this phase, front-desk staff can register patients, schedule appointments, and the system can open/lock encounters.

### Tasks

#### 2.1 — Patient registration and records

**What**: CRUD for patients with USCDI v3 demographics and clinical history.

**Design**:
- Routes: `POST /patients`, `GET /patients/:id`, `PATCH /patients/:id`, `GET /patients?q=&dob=&page=` (search by name/dob/phone).
- USCDI v3 fields added beyond Suggestion 2 base: `sexual_orientation`, `gender_identity`, `race`, `ethnicity`, `sdoh_screening` stored under `clinical_history` JSONB (USCDI v3 mandate, per `standards.md`).
- `mrn` auto-generated per practice (`P-{practice_seq}`); unique per practice.
- Search uses btree name/dob index; paginated with RFC 8288 `Link` headers.

**Testing**:
- `Integration: POST /patients valid → 201, mrn assigned, audit create row`
- `Integration: POST duplicate mrn within practice → 409`
- `Integration: GET /patients?q=smith&dob=1980-01-01 → matching rows, Link header for next page`
- `Unit: ClinicalHistorySchema accepts USCDI sdoh_screening block`

#### 2.2 — Insurance coverage management

**What**: Manage the patient's `insurance` JSONB array (primary/secondary/tertiary) and the practice's `insurance_plans` catalogue.

**Design**:
- `PUT /patients/:id/insurance` replaces the ordered coverage array (validated by `InsuranceArraySchema`; at most one `primary`).
- Plans live in `practices.settings.insurance_plans` with `payer_id`, `edi_payer_id`, `plan_type ∈ {vision, medical}`.

**Testing**:
- `Integration: PUT two coverages both priority=primary → 422`
- `Integration: PUT references unknown plan_id → 422`
- `Integration: valid coverage array → stored, GIN index queryable`

#### 2.3 — Appointment scheduling

**What**: Calendar booking with conflict detection, statuses, and the visit lifecycle.

**Design**:
- Routes: `POST /appointments`, `GET /appointments?location=&provider=&from=&to=`, `PATCH /appointments/:id/status`.
- Status state machine: `scheduled → confirmed → checked_in → in_exam → in_dispensary → completed`; side branches `cancelled`, `no_show`. Illegal transitions rejected by a transition guard map.
- Double-booking guard: reject overlapping `scheduled_start/end` for the same `provider_id` unless provider config allows concurrency.
- `appointment_type` enum drives the default exam template selected in Phase 3.

**Testing**:
- `Integration: POST overlapping provider slot → 409`
- `Integration: PATCH status checked_in→completed (skipping in_exam) → 422 illegal transition`
- `Integration: GET range query → appointments ordered by scheduled_start`
- `Unit: transition guard matrix rejects no_show→in_exam`

#### 2.4 — Encounters (visit documents) and signing

**What**: Create encounters from appointments, hold the SOAP/exam JSONB, and lock on signature.

**Design**:
- `POST /appointments/:id/encounter` creates an `in_progress` encounter, seeding `exam_data` from the practice's exam template for that `appointment_type`.
- `PATCH /encounters/:id` updates `exam_data`/`diagnoses` while `status='in_progress'` (validated against template + Zod).
- `POST /encounters/:id/sign` requires `optometrist|ophthalmologist`; sets `signed_by`, `signed_at`, `status='locked'`. Locked encounters are immutable except via `POST /encounters/:id/addendum`, which appends an addendum referencing the original (malpractice/HIPAA amendment requirement).
- Encounter status: `in_progress → completed → locked`; `addendum` creates a linked record.

**Testing**:
- `Integration: create encounter from comprehensive appt → exam_data seeded with template sections`
- `Integration: PATCH locked encounter → 409`
- `Integration: sign by technician → 403; sign by optometrist → locked, signed_at set`
- `Integration: addendum on locked encounter → new linked record, original unchanged`

### Definition of Done
Full front-desk-to-clinician handoff works: register patient → schedule → check in → open encounter → sign. All transitions audited.

---

## Phase 3: Optometric Exam Workflow & Vision Prescriptions

### Purpose
Deliver the optometry-specific clinical core that differentiates this from a generic EHR: structured refraction, ocular health, and contact-lens fitting capture, plus legally significant vision prescriptions aligned to FHIR `VisionPrescription` and ANSI Z80. This is the primary value proposition and must ship early.

### Tasks

#### 3.1 — Exam templates and progressive-disclosure capture

**What**: Template-driven exam documentation for comprehensive, contact-lens, medical, pediatric, and dry-eye encounter types.

**Design**:
- Templates stored in `practices.settings.exam_templates`; each defines ordered `sections` (e.g. `visual_acuity`, `autorefraction`, `manifest_refraction`, `keratometry`, `iop`, `pachymetry`, `pupils`, `slit_lamp`, `fundus`, `cl_fitting`, `wear_schedule`, `assessment`, `plan`).
- Each section maps to a Zod sub-schema in `ExamDataSchema`; the web exam surface renders fields per section and reveals subsequent sections progressively (UX pattern from RevolutionEHR/Compulink).
- Refraction sub-schema enforces ANSI Z80.1 precision (0.25 D sphere/cyl steps, 1° axis steps).

**Testing**:
- `Unit: manifest_refraction with axis=181 → ZodError (>180)`
- `Unit: comprehensive template renders sections in defined order`
- `Integration: PATCH encounter exam_data section-by-section → persisted, retrievable in order`
- `Integration: cl_fitting section only valid on contact_lens/comprehensive types`

#### 3.2 — Vision prescriptions (eyeglasses & contact lens)

**What**: Generate prescriptions from exam data, store in encounter `prescriptions` JSONB, enforce expiry and laterality.

**Design**:
- `POST /encounters/:id/prescriptions` validates with `PrescriptionSchema`; auto-derives default `expiration_date` (eyeglasses +2y, CL +1y, configurable per state law).
- "Copy from manifest refraction" helper pre-fills `od/os` from `exam_data.manifest_refraction`.
- CL prescriptions require `bc`, `diameter`, `brand`; eyeglasses require `pd`.
- Each prescription has a generated `id` (UUID) referenced later by dispensing orders.

**Testing**:
- `Integration: create eyeglasses Rx without pd → 422`
- `Integration: create CL Rx without base curve → 422`
- `Integration: copy-from-manifest → od/os match exam_data`
- `Unit: expiration default eyeglasses = prescribed_at + 2y`

#### 3.3 — Diagnoses with ICD-10 and laterality

**What**: Attach ICD-10-CM diagnoses with eye laterality to encounters.

**Design**:
- `encounters.diagnoses` JSONB array: `{icd10, description, eye ∈ {od,os,ou,bilateral,na}, primary}`; exactly one `primary` enforced.
- A bundled ICD-10-CM ophthalmic subset (`packages/core/data/icd10-eye.json`) powers typeahead and validates codes; refreshed annually.

**Testing**:
- `Integration: two primary diagnoses → 422`
- `Integration: invalid ICD-10 code H99.999 → 422`
- `Unit: laterality enum rejects "left"`

#### 3.4 — FHIR R4 export of clinical data

**What**: Project encounter JSONB into FHIR R4 / US Core resources.

**Design**:
- `packages/fhir` mappers: encounter → `Encounter` + `Observation` (visual acuity, IOP, refraction), `Condition` (diagnoses), `VisionPrescription` (each prescription), bundled into a `Bundle` (transaction type).
- `GET /fhir/Patient/:id/$everything` returns a paginated FHIR Bundle (RFC 8288 paging links).
- `VisionPrescription.lensSpecification` maps sphere/cylinder/axis/add/prism/base-curve/diameter per the FHIR resource definition; US Core profiles applied where balloted, generic `Observation` + extension elsewhere (Eye Care IG not yet balloted — noted in `standards.md`).

**Testing**:
- `Unit: prescription JSONB → VisionPrescription with correct lensSpecification (od/os)`
- `Unit: IOP exam_data → Observation with LOINC code, OD/OS components`
- `Integration: $everything → valid FHIR Bundle (schema-validated against @medplum/fhirtypes)`

### Definition of Done
A clinician can document a full comprehensive exam, write eyeglasses and CL prescriptions, code diagnoses, sign the encounter, and the visit exports as a valid FHIR R4 bundle.

---

## Phase 4: Optical Dispensary, Inventory & POS

### Purpose
Connect the clinical record to the retail business — the workflow incumbents like Crystal PM are strong at and where omnichannel is underserved. Frame/lens/CL inventory, dispensing orders linked to prescriptions, lab order tracking, and point-of-sale with ledger posting. After this phase a practice can sell and fulfil eyewear end-to-end.

### Tasks

#### 4.1 — Frame, lens, and contact-lens inventory

**What**: Multi-location inventory catalogue with SKU-level stock and reorder alerts.

**Design**:
- `frame_inventory` (per Suggestion 2) with `specs` JSONB (bridge, temple, lens_width, material, rim_type). Per-location `quantity`.
- Routes: `POST /inventory/frames`, `PATCH /inventory/frames/:id`, `GET /inventory/frames?location=&q=&brand=`, `POST /inventory/frames/:id/adjust` (delta with reason).
- Low-stock: when `quantity < reorder_point`, enqueue a `reorder-alert` job.
- Contact-lens stock tracked similarly with `specs` carrying `bc/diameter/power/modality`.

**Testing**:
- `Integration: adjust quantity below zero → 422`
- `Integration: search by brand uses idx_frames_brand`
- `Integration: drop below reorder_point → reorder-alert job enqueued`

#### 4.2 — Dispensing orders linked to prescriptions

**What**: Create lab/dispensing orders referencing a prescription, with the full lens/frame `specs` and lifecycle.

**Design**:
- `dispensing_orders` (Suggestion 2): `specs` JSONB for eyeglasses (frame, lenses, coatings, measurements seg-height/OC-height, lab) or CL (per-eye brand/bc/diameter/power/boxes, annual_supply).
- Status state machine: `ordered → in_lab → received → notified → dispensed`; branches `cancelled`, `returned`. Each transition timestamped and audited.
- `POST /dispensing-orders` requires a valid, non-expired `prescription_id` on the patient's encounters; decrements frame inventory on `ordered`.
- Warranty: `warranty_expiry` set on dispense; returns reference original order.

**Testing**:
- `Integration: order against expired prescription → 422`
- `Integration: order eyeglasses → frame inventory decremented by 1`
- `Integration: illegal transition ordered→dispensed (skip received) → 422`
- `Integration: return creates linked record, warranty checked`

#### 4.3 — Point-of-sale and ledger posting

**What**: Capture charges, take payments, and post to a patient ledger.

**Design**:
- `payments` table (from Suggestion 1, adopted here): `amount_cents`, `payment_method`, `reference_number`, optional `claim_id`/`dispensing_order_id`.
- A POS sale assembles line items (exam fees from `fee_schedule`, dispensing totals), computes patient responsibility (after insurance estimate from Phase 6), and records payment(s).
- Payment processing behind a `PaymentGateway` abstraction (Stripe Terminal/manual entry); idempotency key per transaction.
- Ledger view: running balance per patient from charges − payments − insurance postings.

**Testing**:
- `Integration: sale with split cash+card → two payment rows, ledger balanced`
- `Integration: duplicate idempotency key → single charge`
- `Unit: ledger balance = sum(charges) − sum(payments) − sum(insurance_paid)`

#### 4.4 — Lab order tracking

**What**: Track optical lab orders and inbound status updates.

**Design**:
- `dispensing_orders.specs.lab` holds `{name, order_number}`; a `LabConnector` abstraction posts orders (initially a manual/CSV connector; pluggable per lab) and ingests status callbacks that advance the dispensing status.
- Optician work queue: `GET /dispensing-orders?location=&status=in_lab|received` for the dispensary board.

**Testing**:
- `Integration (mocked lab): submit order → lab order_number recorded, status in_lab`
- `Integration: lab callback "received" → status advances, patient-notify job enqueued`

### Definition of Done
A patient's prescription can be turned into a frame+lens order, sent to a lab, tracked to dispensed, paid for at POS, with inventory and ledger correctly updated.

---

## Phase 5: Patient Engagement — Scheduling, Recall & Messaging

### Purpose
Deliver the patient-facing engagement layer that drives retention and fills the calendar: online self-booking, digital intake, automated recall, and two-way messaging. These run on the async queue built in Phase 1 and can be developed in parallel with Phase 4.

### Tasks

#### 5.1 — Online self-booking and digital intake

**What**: Public booking portal and pre-visit intake form feeding the patient record.

**Design**:
- `apps/web/(patient)` routes: slot search (`GET /public/availability?location=&type=`), book (`POST /public/appointments`), and an intake form whose JSON maps into `patients.clinical_history` and `contact`.
- Bookings create `scheduled` appointments; rate-limited and CAPTCHA-gated (OWASP ASVS).
- Intake submission attaches to the appointment; on check-in, staff review/merge into the patient record.

**Testing**:
- `Integration: public booking into open slot → appointment scheduled`
- `Integration: booking already-taken slot → 409`
- `Integration: intake submission → pending_intake linked to appointment`

#### 5.2 — Automated recall and reminder engine

**What**: Scheduled reminders (confirmation, day-before) and recall outreach (annual exam due) via SMS/email/voice.

**Design**:
- BullMQ delayed jobs scheduled at appointment creation (reminder T-24h) and at encounter close (recall at `last_exam + recall_interval`, default 12 months per `appointment_type`).
- `NotificationChannel` abstraction (Twilio SMS/voice, SendGrid email); template per message type; opt-out honoured.
- Recall list view for staff with due/overdue patients; sending logged to audit and a `messages` table.

**Testing**:
- `Integration: create appointment → reminder job scheduled at start−24h`
- `Integration: close encounter → recall job scheduled at +12m`
- `Integration (mocked Twilio): due recall fires → SMS sent, message row logged`
- `Integration: patient opted out → no send`

#### 5.3 — Two-way patient messaging

**What**: Secure inbound/outbound messaging thread per patient.

**Design**:
- `messages` table: `{patient_id, direction, channel, body, status, thread_id}`. Inbound SMS webhook (signature-verified) appends to thread; staff reply from the patient chart.
- Real-time updates to the staff inbox via SSE.

**Testing**:
- `Integration (mocked): inbound SMS webhook valid signature → message appended, thread updated`
- `Integration: inbound invalid signature → 401, nothing stored`
- `Integration: staff reply → outbound message queued and logged`

### Definition of Done
Patients can self-book and complete intake; the system automatically reminds and recalls them, and staff can hold two-way conversations — all opt-out-aware and audited.

---

## Phase 6: Insurance Billing — Eligibility, Claims & ERA

### Purpose
Implement the revenue-cycle engine that independent ODs rank highest: 270/271 eligibility, 837P claim generation for vision and medical, and 835 ERA auto-posting. This depends on encounters (Phase 3) and POS (Phase 4) and is the most standards-heavy phase (ANSI X12).

### Tasks

#### 6.1 — Eligibility verification (270/271)

**What**: Real-time benefit/eligibility checks against payers via a clearinghouse.

**Design**:
- `packages/edi/src/x270_271.ts` builds a 270 inquiry from patient coverage + provider NPI and parses the 271 response into a normalised `EligibilityResult` (`active`, `copay_cents`, `deductible_remaining_cents`, `vision_benefit_available`, `last_exam_date`).
- `ClearinghouseConnector` abstraction (TriZetto profile first); responses cached in Redis (TTL 24h) keyed by patient+payer+date.
- `POST /eligibility/check` enqueues a job; result streamed back / polled.

**Testing**:
- `Unit: build 270 from coverage → valid X12 segments (ISA/GS/ST/.../SE)`
- `Unit: parse fixture 271 → EligibilityResult with copay and deductible`
- `Integration (mocked clearinghouse): check → cached result, second call served from cache`

#### 6.2 — Claim generation (837P)

**What**: Generate professional claims for vision and medical from encounter + service lines.

**Design**:
- `claims` table (Suggestion 2): `payer`, `service_lines` (CPT, modifiers, ICD-10 pointers, charges), `denial`, `era_835` JSONB.
- Service-line builder derives lines from encounter (exam CPT e.g. 92004/92014, refraction 92015) and dispensing (V-codes for materials), attaching ICD-10 pointers from diagnoses.
- `packages/edi/src/x837p.ts` produces a valid 837P transaction per payer profile (VSP, EyeMed, Davis, generic medical). Claim lifecycle: `draft → submitted → accepted/rejected → paid/partially_paid/denied → appealed`.
- `POST /claims` (build draft), `POST /claims/:id/submit` (enqueue EDI submission with idempotency).

**Testing**:
- `Unit: build 837P from claim with two service lines → valid loops (2000A/2010BB/2300/2400)`
- `Unit: VSP payer profile applies correct payer_id and required segments`
- `Integration: submit claim → status submitted, idempotent on retry`
- `Integration: submit without primary insurance → 422`

#### 6.3 — ERA processing (835) and auto-posting

**What**: Parse remittance advice and post payments/adjustments to claims and ledger.

**Design**:
- `packages/edi/src/x835.ts` parses inbound 835 (from SFTP poller / clearinghouse) into `claims.era_835`; matches by claim number; updates `allowed_cents`, `paid_cents`, `patient_responsibility_cents`, sets status `paid|partially_paid|denied`, and writes a `payment` row (`payment_method='insurance_payment'`).
- Denials captured into `denial` JSONB (`reason`, CARC/RARC `code`, `appeal_deadline`).

**Testing**:
- `Unit: parse fixture 835 → claim matched, paid_cents set, status paid`
- `Unit: 835 with CO-96 denial → denial JSONB populated, status denied`
- `Integration: ERA posting creates insurance_payment row, ledger updated`

#### 6.4 — Patient responsibility estimation

**What**: Estimate patient out-of-pocket at POS using cached eligibility + fee schedule.

**Design**:
- Combines `EligibilityResult` (copay, deductible remaining) with the sale's covered/non-covered lines to compute estimated patient responsibility, surfaced in the POS flow (Phase 4 integration).

**Testing**:
- `Unit: copay $20 + non-covered material $150 → responsibility = $170`
- `Integration: POS sale shows estimate before payment`

### Definition of Done
Eligibility can be verified pre-visit, a clean 837P submitted for a documented encounter, an 835 auto-posted, and denials captured with appeal deadlines.

---

## Phase 7: Diagnostic Instrument & DICOM Integration

### Purpose
Auto-populate exam findings from devices (autorefractor, OCT, visual field, keratometer) — a baseline expectation the research flags, with a documented industry adoption gap that justifies a proprietary adapter layer alongside standards. Requires the encounter model (Phase 3).

### Tasks

#### 7.1 — Device gateway (HL7 v2 ORU/MDM ingest)

**What**: Receive device result messages and normalise them into encounter `device_results`.

**Design**:
- `apps/device-gateway` runs an MLLP listener (`simple-hl7`) accepting ORU^R01 messages; a `normaliser` maps OBX segments per device profile into the `DeviceResultArraySchema` shape (`type`, `device`, `performed_at`, per-eye values, `raw_message` preserved for audit).
- Matched to an open encounter by patient identifiers + time window; appended to `encounters.device_results` via `withJsonbValidation`.
- Per-device profiles (Nidek ARK, Zeiss Cirrus, Humphrey HFA) in `packages/core/data/device-profiles/` map vendor OBX codes → canonical fields (the proprietary adapter layer).

**Testing**:
- `Unit: parse fixture Nidek ORU → autorefraction device_result (od/os sphere/cyl/axis)`
- `Unit: unknown device profile → quarantined with raw_message, flagged for mapping`
- `Integration: ORU for patient with open encounter → device_results appended`

#### 7.2 — DICOM image ingest and reference

**What**: Store ophthalmic images (OCT, fundus, VF) and link them to device results.

**Design**:
- A C-STORE SCP (or DICOMweb STOW-RS endpoint) accepts images; blobs to S3 (encrypted), metadata extracted (`StudyInstanceUID`, `SeriesInstanceUID`, modality) and the `dicom_study_uid` recorded in the relevant `device_results` element.
- `GET /encounters/:id/images` returns presigned URLs (short-lived) for the staff viewer.

**Testing**:
- `Integration (mocked SCP): store OCT DICOM → S3 object created, study_uid linked to device_result`
- `Integration: image URL presigned and expires`

#### 7.3 — Result trending read model

**What**: Columnar materialised view for IOP/RNFL/VA trending despite JSONB storage.

**Design**:
- A materialised view `mv_measurement_trends` extracts `(patient_id, visit_date, measurement_type, od_value, os_value)` from `encounters.exam_data`/`device_results` via JSONB path expressions (the Suggestion-1 trending strength applied to the Suggestion-2 store). Refreshed on encounter sign.

**Testing**:
- `Integration: sign encounter with IOP → mv row appears; trend query returns ordered series`
- `Unit: JSONB path extraction handles missing IOP gracefully (no row)`

### Definition of Done
A device result arriving over HL7/DICOM auto-populates the matching encounter and is viewable; longitudinal IOP/RNFL trends render for glaucoma monitoring.

---

## Phase 8: AI-Native Layer — Scribe, Coding & Denial Prediction

### Purpose
Deliver the AI-native differentiators that justify the project: ambient scribe → structured SOAP, automated ICD-10/CPT coding suggestions, and pre-submission denial-risk scoring. Built on the encounter (Phase 3) and claim (Phase 6) models, grounded in real practice data via MCP-style context exposure.

### Tasks

#### 8.1 — Ambient scribe (speech → structured SOAP)

**What**: Transcribe exam-room dialogue and draft structured exam_data/SOAP for clinician review.

**Design**:
- `apps/web` streams audio to the API; the API streams to the STT abstraction (Deepgram) for a live transcript.
- On stop, `packages/ai` sends transcript + the encounter's exam template to Claude with a structured-output (tool) schema mirroring `ExamDataSchema`; the model returns a draft populating sections + S/O/A/P text.
- Draft stored in `ai_analyses` (`analysis_type='soap_draft'`, `model_version`) — never auto-applied; clinician accepts/edits into `exam_data`. Static clinical-rules system prompt uses prompt caching.

**Testing**:
- `Unit: transcript fixture → structured tool output validates against ExamDataSchema`
- `Integration (mocked LLM+STT): record→stop → soap_draft ai_analyses row created, not applied`
- `Integration: clinician accept → exam_data merged, audit logged`

#### 8.2 — Automated ICD-10 / CPT coding suggestions

**What**: Suggest diagnosis and procedure codes from documented exam data.

**Design**:
- `packages/ai` sends signed/near-complete `exam_data` + `diagnoses` to Claude with the ophthalmic code subset as grounding; returns ranked ICD-10 and CPT suggestions with rationale and laterality.
- Stored as `ai_analyses` (`coding_suggestion`); billing reviews before claim build. Suggestions feed the Phase 6 service-line builder when accepted.

**Testing**:
- `Unit: exam with elevated IOP + C/D 0.6 → suggests glaucoma-suspect ICD-10 ranked`
- `Integration (mocked LLM): suggestions stored, acceptance pre-fills claim service lines`

#### 8.3 — Denial-risk pre-screening

**What**: Score a draft claim's denial likelihood against payer rules before submission.

**Design**:
- A hybrid scorer: deterministic payer-rule checks (missing modifier, ICD/CPT mismatch, frequency limits e.g. refraction once/year) + an LLM rationale over the claim and payer profile, producing `score ∈ [0,1]` and a list of issues.
- Stored as `ai_analyses` (`denial_risk`, `score`); surfaced in the claim review UI with blocking issues vs. warnings.

**Testing**:
- `Unit: 92015 billed twice in 12 months → frequency-limit issue, high score`
- `Unit: missing modifier 25 on same-day E/M + procedure → issue flagged`
- `Integration (mocked LLM): high-risk draft shows blocking issues in review`

#### 8.4 — MCP context server for AI grounding

**What**: Expose patient chart, eligibility, inventory, and clinical-rules context to AI agents via MCP.

**Design**:
- An MCP server (per `standards.md`) exposes read-only, audited resources: `patient-chart`, `eligibility`, `clinical-rules`, `optical-inventory`. All access is tenant-scoped (RLS) and audit-logged like any PHI read.
- Internal scribe/coding/denial features consume these resources rather than bespoke prompt assembly.

**Testing**:
- `Integration: MCP patient-chart fetch → tenant-scoped data, audit row recorded`
- `Integration: cross-tenant MCP request → denied`

### Definition of Done
Scribe drafts a reviewable SOAP note from audio, coding suggestions pre-fill claims on acceptance, denial risk blocks bad claims pre-submission, and all AI features ground on audited, tenant-scoped context.

---

## Phase 9: E-Prescribing, Multi-Location & Quality Reporting

### Purpose
Round out the v1.1 should-have scope: Surescripts e-prescribing, consolidated multi-location reporting, and quality-registry/MIPS reporting hooks. These extend existing models and integrate external networks.

### Tasks

#### 9.1 — E-prescribing via Surescripts (NCPDP SCRIPT)

**What**: Send medication prescriptions electronically to pharmacies.

**Design**:
- `encounters.e_prescriptions` JSONB drives an NCPDP SCRIPT v2023011 NewRx message via a `SurescriptsConnector` (mTLS, vendor-credentialed). Pharmacy directory lookup by NCPDP id; status tracked (`pending→sent→dispensed/denied`).
- EPCS (controlled substances) gated behind WebAuthn two-factor and DEA number presence.

**Testing**:
- `Unit: build NewRx from e_prescription → valid NCPDP SCRIPT message`
- `Integration (mocked Surescripts): send NewRx → status sent, surescripts_id stored`
- `Integration: controlled substance without second factor → blocked`

#### 9.2 — Multi-location consolidated reporting

**What**: Cross-location dashboards for production, capture rate, AR, and inventory.

**Design**:
- Read-model materialised views aggregate revenue, claims, recall compliance, and stock across a practice's locations; KPI endpoints `GET /reports/kpis?from=&to=&location=`.
- Location switching in the UI without per-location logins (gap noted for Compulink/RevolutionEHR).

**Testing**:
- `Integration: KPI report aggregates two locations correctly`
- `Unit: capture-rate = exams_with_dispense / exams`

#### 9.3 — Quality registry & MIPS hooks

**What**: Export quality measures to AOA MORE / AAO IRIS and CMS MIPS.

**Design**:
- A measures engine computes eligible/numerator/denominator from encounter data; exports QRDA-III / registry payloads behind a `RegistryConnector`. Attestation tracking stored per provider/period.

**Testing**:
- `Unit: glaucoma measure numerator computed from IOP-documented encounters`
- `Integration (mocked registry): export → accepted receipt stored`

### Definition of Done
Optometrists can e-prescribe (with EPCS safeguards), administrators see consolidated multi-location KPIs, and quality measures export to registries.

---

## Phase 10: SMART on FHIR API, Hardening & Deployment

### Purpose
Expose the certified-style external API surface, complete the security/compliance hardening, and ship production deployment artefacts. This is the platform-maturity phase enabling third-party extensibility and self-hosting.

### Tasks

#### 10.1 — SMART on FHIR API + OAuth 2.0 authorisation server

**What**: Standards-compliant FHIR R4 API for third-party and patient-facing apps.

**Design**:
- OAuth 2.0 + PKCE authorisation server (RFC 6749/7519) issuing scoped tokens (`patient/*.read`, `user/*.read`); SMART discovery (`/.well-known/smart-configuration`).
- FHIR endpoints (read + search) for `Patient`, `Appointment`, `Encounter`, `Observation`, `Condition`, `VisionPrescription`, `Coverage`, `Claim`, `ClaimResponse`, with US Core profile conformance and Bundle paging (RFC 8288).
- OpenAPI 3.1 spec auto-published for the practice's own REST API.

**Testing**:
- `Integration: OAuth PKCE flow → scoped token; expired token → 401`
- `Integration: GET /fhir/VisionPrescription?patient= → US-Core-valid Bundle`
- `Integration: token with patient/*.read cannot write → 403`

#### 10.2 — Security hardening to OWASP ASVS L2

**What**: Complete the ASVS L2 controls and run an automated security pass.

**Design**:
- Rate limiting, security headers, input validation coverage audit, secrets management, dependency scanning, and a documented threat model + NIST SP 800-66 risk assessment. Penetration-test checklist executed against staff and patient portals.

**Testing**:
- `Integration: rate-limit exceeded → 429`
- `Integration: IDOR attempt on another practice's resource → 403/404 (RLS + authz)`
- `E2E: automated ASVS L2 checklist run green`

#### 10.3 — Production deployment, backups, and DR

**What**: Helm chart, encrypted backups, and disaster-recovery runbook.

**Design**:
- Helm chart for api/worker/web/postgres/redis/minio with secrets via external secret store; automated encrypted PITR backups of Postgres and S3; documented RPO/RTO and restore runbook. Self-host quickstart via docker-compose.

**Testing**:
- `Integration: helm install on kind cluster → all pods ready, health checks pass`
- `Integration: backup then restore into fresh DB → data intact, RLS preserved`
- `E2E: docker-compose self-host quickstart → seeded login works`

### Definition of Done
Third-party apps can authenticate via SMART on FHIR and read US-Core-conformant data; the platform passes ASVS L2 checks and deploys reproducibly with tested backup/restore.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Security      ─── required by everything
    │
Phase 2: Core Domain (patients/sched/encounter) ─── requires P1
    │
Phase 3: Exam Workflow & Prescriptions          ─── requires P2   ← core value ships here
    │
    ├── Phase 4: Optical Dispensary & POS        ─── requires P3 (┐ can parallel
    ├── Phase 5: Patient Engagement              ─── requires P2 (┘ with P4)
    │
    ├── Phase 6: Insurance Billing               ─── requires P3 + P4
    ├── Phase 7: Device & DICOM Integration      ─── requires P3 (can parallel with P4/P5/P6)
    │
Phase 8: AI-Native Layer                         ─── requires P3 + P6
    │
Phase 9: E-Rx, Multi-Location, Quality Reporting ─── requires P3 + P6 (+P2)
    │
Phase 10: SMART on FHIR API, Hardening, Deploy   ─── requires P3 + P6 (FHIR/data surface)
```

**Parallelism opportunities:**
- After Phase 3: Phases 4, 5, and 7 can be developed concurrently (distinct subsystems).
- Phase 6 needs Phase 4's POS/charges; Phase 7 is independent and can run alongside Phase 6.
- Phase 8 (AI) can begin scribe/coding (8.1/8.2) as soon as Phase 3 lands; denial prediction (8.3) waits for Phase 6.

**Estimated scope: large** (full-stack regulated platform, 10 phases, multiple external integrations).

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`vitest run`), including Testcontainers integration tests.
3. Relevant E2E tests pass (`playwright test`) where the phase touches user-facing flows.
4. Biome lint + format passes; `tsc --noEmit` passes across all affected packages.
5. Docker images build (`api`, `worker`, `web`) and the compose stack starts healthy.
6. The phase's feature works end-to-end against the seeded demo practice.
7. New configuration options are documented in `.env.example` and the README.
8. New REST endpoints appear in the auto-generated OpenAPI 3.1 spec; new FHIR resources validate against US Core.
9. Drizzle migrations are created, reviewed as SQL, and applied cleanly to a fresh database; RLS policies cover any new tenant tables.
10. Every new PHI access path writes an audit-log entry, and tenant isolation is verified by a cross-tenant negative test.
11. Any new JSONB column has a corresponding Zod validator enforced via `withJsonbValidation`.
```
