# LLM Project Guide: Migrant Health Records

This document orients an LLM to the project structure, key files, responsibilities, and conventions so it can navigate, answer questions, and make safe changes. Paths are relative to the repository root.

## High-level Overview

- Frontend: `frontend/` — React + Vite + TypeScript single-page app
- Backend: `backend/` — FastAPI + SQLAlchemy + PostgreSQL + Alembic
- Auth: JWT-based (access + refresh). Tokens are used via Authorization header and optionally cookies. OTP login supported.
- Storage: S3-compatible presigned URLs for file uploads/downloads
- Security: Rate limiting (Redis), HTTPS enforcement, security headers, password policy, OTP throttles

---

## Repository Layout

- `backend/`
  - `app/`
    - `main.py`: FastAPI app creation, CORS, middleware wiring, health endpoint; mounts routers
    - `auth.py`: Authentication and authorization utilities and endpoints
    - `db.py`: SQLAlchemy engine/session configuration (PostgreSQL only)
    - `schemas.py`: Pydantic models (request/response validation)
    - `middleware.py`: Redis-based rate limiting + logging, HTTPS/security headers middleware, decorators
    - `core/`
      - `config.py`: Centralized runtime config via environment variables (ENVIRONMENT, SECRET_KEY, DB, Redis, CORS, OTP, storage, etc.)
    - `routers/`
      - `attachments.py`: Presign upload/download and attachment listing with RBAC/consent checks
      - Other routers (mounted in `main.py`): patients, encounters, consents, consent-requests, invitations, geo, admin_geo, stats, notifications, admin_notes, jurisdiction_change_requests, emergency_access
    - `deps.py`: Common dependencies (role checks, consent access checks, audit logging). Frequently used symbols include `require_roles()`, `can_view_patient()`, `ensure_access_to_patient()`, `log_action()`
    - `models.py`: SQLAlchemy models (Users, Patients, Encounters, Attachments, Consents, RefreshToken, OtpCode, etc.)
  - `alembic/`: Database migrations; `env.py` configures Alembic; `versions/` has migration scripts
  - `requirements.txt`: Backend dependencies and versions
  - `scripts/`: Operational scripts (e.g., seeding superadmin)

- `frontend/`
  - `src/`
    - `api/client.ts`: Typed API client calling backend endpoints
    - `services/http.ts`: Axios instance; sets Authorization header from localStorage; `withCredentials: true` to support cookie-based auth too
    - `context/AuthContext.tsx`: App auth state: login, logout, OTP flows, user info, session versioning
    - `pages/`: UI pages (e.g., `Login.tsx`, `Register.tsx`, `Consents.tsx`, Patients list/details, admin pages, etc.)
    - `components/`: UI components (e.g., `PasswordField`)
    - `types`: Shared TS types (API models, routes, constants like `API_BASE`)
  - `.env`: Frontend environment (e.g., API base URL)

- Project docs: `SYSTEM_FLOWS.md`, `migrant_health_records_system_design_documentation.md`, `colour.md`

---

## Backend: Key Modules and Responsibilities

### `app/core/config.py`
Centralized environment-driven configuration. Important keys:
- Environment and secrets
  - `ENVIRONMENT` (development|staging|production)
  - `SECRET_KEY` (required in production)
  - `ALGORITHM` (JWT)
  - `ACCESS_TOKEN_EXPIRE_MINUTES` (15 in production by default)
- Database
  - `DATABASE_URL` (PostgreSQL only; `FORCE_POSTGRES=true` by default)
- CORS
  - `ALLOWED_ORIGINS`, `ALLOW_CREDENTIALS`
- Redis + Rate limiting
  - `REDIS_URL` and global rate limits: window, anon/auth limits, method-awareness, enable flag
  - `ENABLE_RATE_LIMIT_MIDDLEWARE` defaults to true in production
- Proxy and HTTPS
  - `TRUST_PROXY` — read `X-Forwarded-Proto` for HTTPS enforcement
- OTP settings
  - `OTP_EXPIRE_MINUTES`, `OTP_CODE_LENGTH`, `OTP_MAX_ATTEMPTS` (lower in prod)
- Storage/S3
  - `STORAGE_PROVIDER`, `S3_BUCKET`, AWS creds/region/endpoint
  - `PRESIGN_EXPIRE_SECONDS`, `MAX_UPLOAD_SIZE_BYTES`, `ALLOWED_UPLOAD_CONTENT_TYPES`
- Consent and session TTLs

### `app/db.py`
- SQLAlchemy engine creation (PostgreSQL enforced) and `SessionLocal`
- Connection pool settings to avoid stale connections

### `app/middleware.py`
- `RateLimitAndLogMiddleware`: Redis-based rate limiting with per-route overrides and request logging
- `rate_limit_policy(limit, window)`: decorator to set per-endpoint limits
- `SecurityHeadersAndHTTPSMiddleware`: In production, enforces HTTPS (redirect GET/HEAD; reject others) and adds headers: HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy

### `app/schemas.py`
- Pydantic v2 schemas for all API domains
- Strong password policy on `UserCreate.password` (min 12; must include upper/lower/digit/symbol)
- Models include: `UserCreate/Read`, `TokenPair`, OTP models, Patient/Encounter/Vitals, Attachments, Consents, Diseases, Invitations, JurisdictionChange, Emergency Access, Notifications, Admin Notes, Geo hierarchy

### `app/auth.py`
Primary auth flows:
- Password login: `POST /auth/token` — issues access + refresh
- OTP login: `POST /auth/otp/request`, `POST /auth/otp/verify`
- Refresh: `POST /auth/refresh` — rotates refresh tokens
- Logout: `POST /auth/logout` — revokes a refresh token
- Logout-all: `POST /auth/logout_all` — revokes all refresh tokens for user
- Guards: Blocks login/OTP verify if request already bears a valid JWT (via header or cookies)
- Rate limits: Per-endpoint limits via `rate_limit_policy`
- Password hashing: Passlib bcrypt
- Email: OTP email send (works when SMTP envs set)

Key helpers:
- `_extract_token_from_request()` — reads token from Authorization header or common cookie names
- `get_current_user_optional()` — resolves a user if token is valid; used to block re-login

### `app/routers/attachments.py`
- `POST /attachments/presign-upload`: Validates filename, content-type, and size; checks RBAC/consents; persists placeholder `Attachment`; returns S3 presign URL
- `GET /attachments/`: Lists attachments with RBAC scoping (migrant, provider with consent, scoped admins)
- `GET /attachments/{id}/presign-download`: Returns S3 presign URL for download
- `PATCH /attachments/{id}/status`: Update status
- `DELETE /attachments/{id}`: Best-effort deletion

Security controls for uploads:
- `MAX_UPLOAD_SIZE_BYTES` and `ALLOWED_UPLOAD_CONTENT_TYPES` enforced server-side
- Filename sanitization to prevent path traversal

### Other Routers (not exhaustive; mounted in `app/main.py`)
- `patients`, `encounters`, `consents`, `consent-requests`, `invitations`, `geo` (public and admin), `stats`, `notifications`, `admin_notes`, `jurisdiction_change_requests`, `emergency_access`
- Access control typically via `require_roles()` and `can_view_patient()` dependencies

### `app/main.py`
- Creates `FastAPI` app
- Adds CORS middleware with `ALLOW_CREDENTIALS` support when origins are explicit
- Adds `SecurityHeadersAndHTTPSMiddleware` in production
- Adds `RateLimitAndLogMiddleware` when enabled by config
- Mounts routers and exposes `/healthz`

---

## Frontend: Key Modules and Responsibilities

### `src/services/http.ts`
- Axios instance with `baseURL: API_BASE` and `withCredentials: true` (sends cookies)
- Adds Authorization header from `localStorage` when present
- Centralized response interceptor hooks (e.g., special handling on 422 during register)

### `src/api/client.ts`
- Typed wrapper functions for backend endpoints:
  - Auth: `/auth/token`, `/auth/register`, OTP flows, `/auth/logout`
  - Attachments: presign upload/download, list, update status
  - Patients/Encounters/Diseases/Stats
  - Consents, Consent Requests
  - Invitations, Notifications, Profiles, Geo queries, Jurisdiction changes, Emergency access

### `src/context/AuthContext.tsx`
- Exposes `user`, `login`, `logout`, `register`, `loginOtpStart`, `loginOtpVerify`, `sessionVersion`
- Loads `user` on boot if `access_token` in localStorage; compatible with cookie auth when backend sets cookies
- `logout()` revokes refresh token server-side (best-effort) and clears local tokens

### `src/pages/Login.tsx`
- Redirects authenticated users away from `/login` to `/patients`
- Supports both password and OTP login flows

### Other Pages and Components
- `src/pages/Register.tsx`, `src/pages/Consents.tsx`, patients pages, admin pages, etc.
- `src/components/` for shared UI pieces (e.g., `PasswordField`)

---

## Auth Model

- Access Token: Short-lived JWT (15 minutes in production by default). Used for API calls.
- Refresh Token: Persistent token stored in DB; rotated on refresh; revocation supported via `/auth/logout` and `/auth/logout_all`.
- Token Carriage:
  - Authorization header: `Bearer <access_token>` from localStorage
  - Or cookies (if backend issues them). Frontend `http.ts` is configured to send cookies too.
- Re-login Guard: If a valid access token is present, `/auth/token` and `/auth/otp/verify` reject with `Already authenticated`.
- OTP: Request + Verify flow with throttles and attempt limits (stricter in production).

---

## Security Controls Summary

- **Password Policy**: Strong validation in `schemas.py` (`UserCreate.password`)
- **Rate Limiting**: Redis-based global middleware + per-endpoint decorators (`middleware.py`)
- **HTTPS Enforcement**: Production middleware redirects GET/HEAD to HTTPS and rejects non-idempotent HTTP; HSTS and security headers added
- **CORS**: Use explicit origins and `ALLOW_CREDENTIALS=true` when using cookies
- **Uploads**: Content-type allow-list and max size; filename sanitization; S3 presign
- **Access Control**: Role checks and consent gating via shared dependencies (`deps.py`)

---

## Running Locally

- Backend
  - Ensure PostgreSQL is available; set `DATABASE_URL`
  - Set `ENVIRONMENT=development`
  - Optional: configure `REDIS_URL` for rate limiting (middleware defaults off in dev)
  - Run `uvicorn app.main:app --reload`

- Frontend
  - Set `.env` with `VITE_API_BASE` (consumed by `API_BASE` in `types`)
  - Run `npm install` and `npm run dev` in `frontend/`

- Production Configuration (important keys)
  - `ENVIRONMENT=production`
  - `SECRET_KEY` (required), `TRUST_PROXY=true` (behind reverse proxy)
  - `ENABLE_RATE_LIMIT_MIDDLEWARE=true`
  - `ACCESS_TOKEN_EXPIRE_MINUTES=15`
  - CORS with explicit `ALLOWED_ORIGINS` and `ALLOW_CREDENTIALS=true` if using cookies

---

## How to Add a New Feature (LLM Playbook)

1. Identify the domain area:
   - Backend: add schema(s) in `schemas.py`, model(s) in `models.py`, and an API router in `routers/<feature>.py`
   - Frontend: add client calls in `src/api/client.ts`, page(s) under `src/pages/`, and component(s) under `src/components/`
2. Wire backend router in `app/main.py` and ensure access control dependencies are applied (`require_roles`, `can_view_patient`)
3. Update `types` used by frontend where needed
4. Add rate limits for sensitive endpoints via `@rate_limit_policy`
5. If dealing with uploads, enforce `content_type`/`size` rules and sanitize filenames
6. Add tests (if test scaffold present) and update docs

---

## Conventions

- Paths: backend routers live under `app/routers/` with RESTful routes; use nouns and pluralization
- Security: default-deny posture; always add role/consent checks for data access
- Config: never hardcode secrets; read from `core/config.py`
- Responses: Pydantic models from `schemas.py`

---

## Quick Index of Important Files

- Backend
  - `backend/app/main.py`
  - `backend/app/auth.py`
  - `backend/app/core/config.py`
  - `backend/app/middleware.py`
  - `backend/app/schemas.py`
  - `backend/app/db.py`
  - `backend/app/routers/attachments.py`
  - `backend/app/models.py`
  - `backend/app/deps.py`

- Frontend
  - `frontend/src/services/http.ts`
  - `frontend/src/api/client.ts`
  - `frontend/src/context/AuthContext.tsx`
  - `frontend/src/pages/Login.tsx`, `frontend/src/pages/Register.tsx`, `frontend/src/pages/Consents.tsx`
  - `frontend/src/components/`
  - `frontend/src/types`

---

## Notes for LLMs

- Always read and respect `app/core/config.py` when adding features; config toggles are commonly used and gate production-only behaviors.
- When modifying authentication flows, update both the backend endpoints (`auth.py`) and frontend integration (`AuthContext.tsx`, `services/http.ts`, and `api/client.ts`).
- For any new endpoints, consider adding `@rate_limit_policy` with appropriate limits.
- For data access to patient info or attachments, ensure consent checks via dependencies.
- When touching uploads, validate filename, content-type, and size server-side; never rely on client-side checks only.


---

## Endpoint Inventory (API surface)

All application routers are mounted under `/api/v1` in `backend/app/main.py`. Health checks exist at `/healthz` and `/check` (root-level).

- Auth (`backend/app/auth.py`)
  - POST `/api/v1/auth/token` — Password login; rate-limited; blocks if already authenticated
  - POST `/api/v1/auth/otp/request` — Request login OTP; rate-limited
  - POST `/api/v1/auth/otp/verify` — Verify OTP and issue tokens; rate-limited; blocks if already authenticated
  - POST `/api/v1/auth/refresh` — Rotate/refresh access token
  - POST `/api/v1/auth/logout` — Revoke presented refresh token
  - POST `/api/v1/auth/logout_all` — Revoke all refresh tokens for current user
  - (Also registration endpoints if present in auth router)

- Users (`backend/app/routers/users.py`)
  - GET `/api/v1/users/me` — Current user info
  - GET `/api/v1/users/{user_id}` — Read user by id

- Patients (`backend/app/routers/patients.py`)
  - GET `/api/v1/patients/` — List patients (scoped by role/consent/admin jurisdiction)
  - POST `/api/v1/patients/` — Create patient (providers/hospitals/admin)
  - GET `/api/v1/patients/{patient_id}` — Read patient (RBAC/consent check)
  - GET `/api/v1/patients/by-migrant/{migrant_id}` — Read patient for migrant user
  - PUT `/api/v1/patients/{patient_id}` — Update patient (ensure_access_to_patient)
  - DELETE `/api/v1/patients/{patient_id}` — Delete patient (ensure_access_to_patient)
  - GET `/api/v1/patients/{patient_id}/audit-logs` — Related audit logs

- Encounters (`backend/app/routers/encounters.py`)
  - GET `/api/v1/encounters/` — List encounters; optional `patient_id`
  - POST `/api/v1/encounters/` — Create encounter (writes require explicit consent)
  - GET `/api/v1/encounters/{encounter_id}` — Read encounter (RBAC/consent)
  - PUT `/api/v1/encounters/{encounter_id}` — Update encounter (writes require consent)
  - DELETE `/api/v1/encounters/{encounter_id}` — Delete encounter (writes require consent)

- Profiles (`backend/app/routers/profiles.py`)
  - GET `/api/v1/profiles/me` — Migrant's own profile (auto-backfills health_id/qr_token)
  - POST `/api/v1/profiles/` — Upsert migrant profile
  - GET `/api/v1/profiles/public/health-id/{health_id}` — Public profile by health id
  - GET `/api/v1/profiles/public/qr/{qr_token}` — Public profile by QR token

- Consents (`backend/app/routers/consents.py`)
  - POST `/api/v1/consents/grant`
  - POST `/api/v1/consents/grant-by-email`
  - GET `/api/v1/consents/mine`
  - GET `/api/v1/consents/granted-to-me`
  - POST `/api/v1/consents/{consent_id}/revoke`

- Consent Requests (`backend/app/routers/consent_requests.py`)
  - POST `/api/v1/consent-requests/` — Provider creates request
  - GET `/api/v1/consent-requests/mine` — Migrant's requests
  - GET `/api/v1/consent-requests/my-requests` — Provider's requests
  - PATCH `/api/v1/consent-requests/{id}/approve` — Migrant approves; returns code
  - PATCH `/api/v1/consent-requests/{id}/reject` — Migrant rejects
  - POST `/api/v1/consent-requests/{id}/enter-code` — Provider enters code to obtain session
  - PATCH `/api/v1/consent-requests/{id}/revoke` — Migrant revokes granted session

- Attachments (`backend/app/routers/attachments.py`)
  - POST `/api/v1/attachments/presign-upload` — Validates file and returns presign URL
  - GET `/api/v1/attachments/` — List attachments (scoped)
  - GET `/api/v1/attachments/{attachment_id}` — Get attachment metadata
  - GET `/api/v1/attachments/{attachment_id}/presign-download` — Presign download URL
  - PATCH `/api/v1/attachments/{attachment_id}/status` — Update status
  - DELETE `/api/v1/attachments/{attachment_id}` — Delete

- Health Records (`backend/app/routers/health_records.py`)
  - GET `/api/v1/migrants/{migrant_id}/records` — List health records (RBAC + consent or emergency)
  - POST `/api/v1/migrants/{migrant_id}/records` — Create health record (providers/hospitals with consent; admins bypass)

- Invitations (`backend/app/routers/invitations.py`)
  - POST `/api/v1/invitations/` — Create invitation (admin)
  - GET `/api/v1/invitations/by-code/{code}` — Get by code
  - GET `/api/v1/invitations/mine` — List created by current admin
  - PATCH `/api/v1/invitations/{invitation_id}/revoke` — Revoke

- Emergency Access (`backend/app/routers/emergency_access.py`)
  - POST `/api/v1/emergency-access/` — Create emergency grant
  - GET `/api/v1/emergency-access/migrant/{migrant_id}` — List grants
  - PATCH `/api/v1/emergency-access/{grant_id}/revoke` — Revoke grant

- Diseases (`backend/app/routers/diseases.py`)
  - GET `/api/v1/diseases/` — List
  - POST `/api/v1/diseases/` — Create (admin)
  - GET `/api/v1/diseases/stats` — Aggregated stats (admin)
  - POST `/api/v1/diseases/seed` — Seed defaults (admin)

- Notifications (`backend/app/routers/notifications.py`)
  - GET `/api/v1/notifications/mine` — List current user's notifications
  - PATCH `/api/v1/notifications/{notification_id}/read` — Mark read
  - PATCH `/api/v1/notifications/read-all` — Mark all read
  - GET `/api/v1/notifications/stream` — SSE stream (token via header or query)

- Admin GEO (`backend/app/routers/admin_geo.py`)
  - CRUD for States, Districts, Subdivisions, Blocks under `/api/v1/admin/geo/*` (admin)
  - PATCH `/api/v1/admin/geo/users/{user_id}/assign-jurisdiction` — Assign admin/provider jurisdiction

- GEO Public (`backend/app/routers/geo_public.py`)
  - GET lists for states/districts/subdivisions/blocks under `/api/v1/geo/*`
  - GET single resource by id: `/api/v1/geo/states/{id}`, etc.

- Jurisdiction Change Requests (`backend/app/routers/jurisdiction_change_requests.py`)
  - POST `/api/v1/jurisdiction-change-requests/` — Provider/hospital submit
  - GET `/api/v1/jurisdiction-change-requests/mine` — List mine (provider/hospital)
  - GET `/api/v1/jurisdiction-change-requests/` — Admin list (scoped)
  - PATCH `/api/v1/jurisdiction-change-requests/{id}/approve|reject` — Admin actions

- Stats (`backend/app/routers/stats.py`) [admin]
  - GET `/api/v1/stats/kpis`
  - GET `/api/v1/stats/encounters_timeseries`
  - GET `/api/v1/stats/patients_timeseries`
  - GET `/api/v1/stats/attachments_timeseries`


---

## Data Model Overview (key entities and relationships)

- `User`
  - Fields: id, email, hashed_password, role (`migrant|provider|hospital|admin...`), phone, registration numbers, jurisdiction (state/district/subdivision/block)
  - Rels: has many `Patient` (created_by), has one `MigrantProfile`, has many `RefreshToken`, has many `Consent` (granted/received)

- `Patient`
  - Links to `migrant_user_id` (optional) and `created_by` (provider/hospital/admin)
  - Has many `Encounter`, `Attachment`, `Medication`, `LabResult`

- `Encounter`
  - Belongs to `Patient`, created by `User`
  - Has one `Vitals`
  - Many-to-many with `Disease` via `encounter_diseases`

- `Attachment`
  - Belongs to `Patient` and optional `Encounter`
  - Tracks storage info: bucket, storage_key, content_type, size, status

- `Consent`
  - Migrant grants consent to provider/admin (`migrant_id` -> `granted_to`), with `scope_json`, `expires_at`, `revoked`

- `ConsentRequest`
  - OTP-based session grant: `provider_id`, `migrant_id`, `status`, `code`, `code_valid_until`, `session_expires_at`
  - `ConsentAudit` events track lifecycle actions

- `RefreshToken`
  - Persistent refresh token per user with `revoked` flag and expiry

- `OtpCode`
  - Stores per-user OTPs, attempts, expiry

- `HealthRecord`
  - Append-only records of changes (JSONB) authored by providers/admins for a migrant

- `EmergencyAccessGrant`
  - Time-limited read-only access for emergencies with audit and notifications

- `AdminNote`, `Notification`
  - Admin-only patient notes; user-targeted notifications (+ SSE stream)

- GEO: `State`, `District`, `Subdivision`, `Block`
  - Hierarchical administrative geography for scoping admins and providers

- `MigrantProfile`
  - One-to-one with migrant `User`; includes `health_id`, `qr_token`, photo, address


---

## Testing Guide (pytest + Testcontainers)

- Recommended dev dependencies:
  - `pytest`, `pytest-asyncio`, `httpx`, `testcontainers[postgres]`, `pip-audit`, `bandit`

- Example approach:
  1. Create `backend/tests/conftest.py` to spin up a Postgres container and set `DATABASE_URL` for the app
  2. Initialize the schema via Alembic migrations (or `Base.metadata.create_all` for smoke tests)
  3. Use FastAPI `TestClient` or `httpx.AsyncClient` to hit `/api/v1/*`

- Critical tests:
  - Auth: weak password rejected; login/refresh/logout revocation; “Already authenticated” guard works
  - RBAC & Consents: providers need active consent; migrants only access self; admins scoped by jurisdiction
  - Attachments presign: rejects invalid filename/content-type/oversize
  - Rate limits: 429 after threshold on `/auth/*`


---

## Ops Runbook (env, migrations, deployment)

- Environment variables (see `backend/app/core/config.py`):
  - `ENVIRONMENT` (production enables HTTPS/HSTS middleware and R/L default)
  - `SECRET_KEY` (required in production)
  - `DATABASE_URL` (PostgreSQL; enforced)
  - CORS: `ALLOWED_ORIGINS`, `ALLOW_CREDENTIALS`
  - Rate Limiting: `REDIS_URL`, `ENABLE_RATE_LIMIT_MIDDLEWARE`, `RATE_LIMIT_*`
  - HTTPS behind proxy: `TRUST_PROXY=true`
  - Auth: `ACCESS_TOKEN_EXPIRE_MINUTES` (15 in prod default)
  - OTP: `OTP_EXPIRE_MINUTES`, `OTP_CODE_LENGTH`, `OTP_MAX_ATTEMPTS`
  - Storage: `S3_BUCKET`, `AWS_*`, `S3_ENDPOINT_URL`, `MAX_UPLOAD_SIZE_BYTES`, `ALLOWED_UPLOAD_CONTENT_TYPES`

- Migrations:
  - Use Alembic under `backend/alembic/`. Typical commands:
    - `alembic revision --autogenerate -m "message"`
    - `alembic upgrade head`

- Deployment checklist (backend):
  - Set env vars above, particularly `SECRET_KEY`, DB URL, CORS, Redis, HTTPS/proxy flags
  - `ENVIRONMENT=production`, `ENABLE_RATE_LIMIT_MIDDLEWARE=true`, `TRUST_PROXY=true`
  - Ensure reverse proxy sets `X-Forwarded-Proto=https` and terminates TLS
  - Monitor health: `/healthz` and `/check`

- Frontend:
  - `VITE_API_BASE` points to backend `/api/v1` origin
  - If using cookie auth, ensure backend CORS allows credentials and specific origins
