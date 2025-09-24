# Migrant Health Records - Frontend

A React + Vite + TypeScript SPA integrated with the FastAPI backend.

## Prerequisites

- Node.js 18+
- Backend running at `http://127.0.0.1:8000` (default), with CORS allowing the frontend origin

## Configure

Create a `.env` file in this folder to override defaults if needed:

```
VITE_API_BASE=http://127.0.0.1:8000/api/v1
```

The backend already defaults `ALLOWED_ORIGINS` to `*`. If you have locked it down, set:

Windows PowerShell (in the backend shell):
```
$env:ALLOWED_ORIGINS = "http://127.0.0.1:5173"
```
Then restart the backend server.

## Install & Run (dev)

In a terminal:
```
npm install
npm run dev
```
Open http://127.0.0.1:5173

## Features

- Auth: register (migrant/provider/admin), login, logout
- Patients: list (consent-scoped), detail
- Encounters: list, create (consent enforced server-side)
- Attachments (S3): presign upload, upload via signed URL, mark available, download via presigned URL
- Consents: grant by provider email, list mine, list granted-to-me, revoke
- Audit logs: per-patient view of actions

## Notes

- For S3 uploads, configure backend env: `STORAGE_PROVIDER=s3`, `S3_BUCKET`, and AWS creds/region or `S3_ENDPOINT_URL` (MinIO). If not configured, upload endpoints return 501/500.
- Rate limiting is enabled on the backend. Hammering endpoints may return 429 depending on your settings.

## Routes and Roles

The app uses route-level guards via `ProtectedRoute` in `src/App.tsx`.

- `/` — Redirects to `/dashboard` (admins) or `/patients` (others) when authenticated; `/login` otherwise
- `/login` — Login
- `/admin/login` — Admin login
- `/admin/register-by-invite` — Admin registration via invite code
- `/register` — Register (migrant/provider/hospital)
- `/forgot-password` — Forgot/Reset password via Health ID + OTP

- `/patients` — Protected (any authenticated role)
- `/patients/:id` — Protected (access-scoped by backend)
- `/consents` — Protected (migrant, provider, hospital)
- `/emergency-access` — Protected (provider, hospital)
- `/emergency-access/log` — Protected (migrant)
- `/chat` — Protected (migrant) — Clinical RAG chatbot
- `/profile` — Protected (migrant)
- `/practice` — Protected (provider, hospital, lab)
- `/lab/jobs` — Protected (lab)
- `/profile/jurisdiction` — Protected (provider, hospital)

- `/dashboard` — Protected (admin)
- `/admin/providers` — Protected (admin)
- `/admin/hospitals` — Protected (admin)
- `/admin/labs` — Protected (admin)
- `/admin/diseases` — Protected (admin; superadmin or state_admin with full_control)
- `/admin/vaccines` — Protected (admin; superadmin or state_admin with full_control)
- `/admin/invitations` — Protected (admin; superadmin or any admin with full_control)
- `/admin/notifications` — Protected (admin)
- `/admin/jurisdiction-requests` — Protected (admin)
- `/admin/manage-admins` — Protected (admin; superadmin or any admin with full_control)

- `/profile/:healthId` — Public profile view (QR landing)

## Chatbot (Clinical RAG)

- Page: `src/pages/Chat.tsx`
- Backend endpoint: `POST /api/v1/chat/ask`
- Health: `GET /api/v1/chat/health` (reports MedCAT/SBERT/ocr status)
- Behavior:
  - Retrieves a personalized patient summary and scans recent attachments
  - Uses SBERT embeddings for similarity; MedCAT 2.1.0 for clinical NER (spaCy fallback)
  - Extracts text via PyMuPDF; falls back to Tesseract OCR (and `pdf2image` if PyMuPDF is not available)
  - Generates answers with Gemini 2.5 Flash (Google Search tools enabled; falls back to 1.5 Flash)

## i18n

- Config: `src/i18n.ts` loads translations from `public/locales/<lng>/translation.json`
- Many pages use `useTranslation()` and keys like `t('login.title')`
