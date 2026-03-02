# MintralCHR.ai

Mistral-first clinical reporting platform that transforms raw medical records (PDFs, images, clinical notes) into structured, editable Client Health Reports (CHRs) with source-aware workflows.

## What It Does

- Ingests patient documents and notes
- Runs OCR + extraction into structured clinical data
- Generates embeddings and retrieval context (RAG)
- Produces draft CHRs for clinician review
- Provides clinical analytics (trends, alerts, timeline, suggestions)
- Includes admin/user/security controls for healthcare workflows

## Core Stack

- Backend: FastAPI + PostgreSQL + pgvector
- Storage: Supabase Storage
- LLM/OCR pipeline: Mistral API (OpenAI-compatible interface)
- Templates/UI: server-rendered Jinja pages (`/ui`)

## Mistral-First Configuration

This project is configured to use Mistral as the primary model provider.

Important: some internal modules still check `OPENAI_API_KEY` while requests are sent to Mistral-compatible endpoints.
For full feature coverage, set **both** `MISTRAL_API_KEY` and `OPENAI_API_KEY` to your Mistral key.

### Required environment values

```bash
# Database / storage
DATABASE_URL=postgresql://...
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
STORAGE_BUCKET=medchr-uploads

# Mistral (primary)
MISTRAL_API_KEY=<your_mistral_key>
OPENAI_API_KEY=<your_mistral_key>
OPENAI_BASE_URL=https://api.mistral.ai/v1
OPENAI_MODEL=mistral-large-latest
OPENAI_EMBEDDING_MODEL=mistral-embed

# App
APP_SECRET_KEY=<strong_random_secret>
APP_ENV=dev
APP_BASE_URL=http://127.0.0.1:8000
```

## Quick Start

Run from project root:

```bash
cd MintralCHR.ai
```

1. Create and activate virtualenv

```bash
python -m venv .venv
source .venv/bin/activate
```

2. Install dependencies

```bash
pip install -r backend/requirements.txt
```

3. Create `.env`

```bash
cp .env.example .env
# then edit .env using the Mistral-first config above
```

4. Ensure infrastructure

- Supabase Storage bucket exists: `medchr-uploads` (private)
- PostgreSQL has `vector` extension enabled

5. Initialize database

```bash
python -m backend.scripts.init_db
```

6. Start app

```bash
./run_dev.sh
```

7. Open UI

- `http://127.0.0.1:8000/ui`

8. Bootstrap admin user (first run)

```bash
python -m backend.scripts.bootstrap_admin
```

## Key Workflows

### Document extraction + embeddings + CHR

- Upload document via UI or API
- Extract text/structured data
- Embed extracted content
- Draft CHR from patient context

### Background jobs (optional)

```bash
# enable JOB_QUEUE_ENABLED=true in .env
python -m backend.scripts.worker
```

## Main Endpoints

- `GET /health` - liveness
- `GET /ready` - readiness (DB + storage)
- `POST /patients` - create patient
- `GET /patients` - list patients
- `POST /patients/{patient_id}/documents` - upload document
- `POST /documents/{document_id}/extract` - run extraction
- `POST /documents/{document_id}/embed` - generate/store embeddings
- `POST /chr/draft` - generate draft CHR
- `GET /jobs/{job_id}` - async job status

Clinical analytics endpoints:

- `GET /api/gap/patients/{patient_id}/trends`
- `GET /api/gap/patients/{patient_id}/timeline`
- `GET /api/gap/patients/{patient_id}/suggested-diagnoses`
- `GET /api/gap/patients/{patient_id}/genetics`
- `GET /api/gap/patients/{patient_id}/drug-interactions`
- `GET /api/gap/patients/{patient_id}/clinical-insights`

## Testing

Run full backend tests:

```bash
python -m pytest -q backend/tests
```

Run targeted validation scripts:

```bash
python -m backend.scripts.validate_runtime_policy
python -m backend.scripts.validate_llm_gateway_usage
```

## Security Notes

- API key, scope, and tenant controls are enforced in backend security/authz layers
- Session + MFA flows are implemented for UI-sensitive actions
- PHI policy checks and egress logging are enforced in LLM gateway paths

## Troubleshooting

- If startup fails on DB host resolution, verify network/DNS and `DATABASE_URL`
- If LLM features return empty output, confirm both `MISTRAL_API_KEY` and `OPENAI_API_KEY` are set
- If storage calls fail, verify `SUPABASE_URL` and service-role key

## Project Structure

- `backend/app/` - API, auth, security, LLM, RAG, OCR, analytics
- `backend/scripts/` - DB/bootstrap/ops utilities
- `backend/sql/migrations/` - schema migrations
- `backend/tests/` - backend test suite
- `frontend/templates/` - UI templates
- `data/` - sample/mock clinical data

