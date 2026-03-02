# MedCHR.ai — Engineering Notes (Engineering + Compliance)

This document is a **living engineering notebook** for MedCHR.ai. It captures:
- how the repo is structured,
- how to run/test it,
- and the **enterprise / HIPAA-hardening plan** we are implementing.

If you are new to the project, start with `README.md`, then come back here for deeper engineering conventions and security posture.

---

## Objective (1-liner)

MedCHR is a clinician-in-the-loop pipeline that turns raw patient records (PDFs/images/notes) into **editable Client Health Reports (CHRs)** with **citations back to source data**, then clinician review + sign-off.

---

## Repository Map

- `backend/app/` — FastAPI application (API + server-rendered UI)
  - `gap_features.py` — Clinical analytics API router (trends, timeline, suggestions, genetics, rules)
  - `trends.py` — Longitudinal lab value trend analysis with direction/severity
  - `timeline.py` — Patient event extraction and timeline formatting
  - `alerts.py` — Critical lab values, drug interactions, allergy contraindications
  - `clinical.py` — Clinical feature endpoints
  - `ip_whitelist.py` — Tenant IP allowlist enforcement (uses `allowed_ips` column)
- `frontend/templates/` — Jinja2 templates for `/ui/*`
- `backend/sql/migrations/` — SQL migrations applied by `backend/scripts/migrate.py`
- `backend/scripts/` — ops scripts (migrate, worker, import mock data, data generation, etc.)
  - `generate_4_serangkai.py` — Simulate realistic patient histories (hypertension, diabetes, etc.)
  - `generate_scale_data.py` — Generate 100 hospital-grade patient records
- `doc/` — product + operational docs (HIPAA notes, k8s snippets, etc.)
- `doc/ARCHITECTURE.md` — current vs target enterprise architecture diagrams
- `doc/CLOUD_AGNOSTIC_HEALTHCARE_BASELINE.md` — provider-neutral healthcare control baseline

---

## Current Architecture (High Level)

1) Clinician uses `/ui/*` (cookie session) to manage patients/documents and draft CHRs  
2) Documents are stored in **Supabase Storage** (private bucket)  
3) Text extraction uses **pdfplumber/Tesseract**, and optional **Vision OCR**  
4) Extraction/embeddings/CHR drafting use an LLM provider (OpenAI by default)  
5) Background jobs run via a worker polling `jobs` table (MVP)

See `security_best_practices_report.md` for security gaps and the “after” architecture.

---

## Running Locally

Primary instructions: `README.md`

Common commands:
- Install deps: `pip install -r backend/requirements.txt`
- Apply schema/migrations: `python -m backend.scripts.init_db`
- Run API: `uvicorn app.main:app --app-dir backend --reload`
- Run worker (optional): `python -m backend.scripts.worker`

---

## Testing

- Unit tests live under `backend/tests/`
- Run: `pytest`

Hardening rule: when adding security/authz controls, add at least one test that would have failed before the change.

---

## Security + HIPAA Mode (What We Mean)

HIPAA compliance is not “a toggle”; it is a combination of:
- technical safeguards (authz, audit logging, encryption, monitoring, egress controls),
- administrative safeguards (policies, training, access reviews, incident response),
- and vendor contracts (BAAs for any PHI processor).

In this repo:
- `HIPAA_MODE=true` enables stricter startup checks and secure defaults (`doc/PRODUCTION_READINESS.md`).
- `PHI_PROCESSORS` is the allowlist of vendors that are allowed to receive PHI.
- `PHI_REDACTION_ENABLED=true` enables best-effort redaction before egress (not a guarantee).

**Important:** Any outbound call that might include PHI must be routed through a single policy gate (work-in-progress).
Current implementation note: **all OpenAI client creation must go through** `backend/app/llm_gateway.py` (it enforces `PHI_PROCESSORS` in HIPAA mode and applies optional redaction).
API key implementation note: API auth now supports DB-backed hashed keys in `api_keys` (provision with `python -m backend.scripts.create_api_key`), in addition to tenant-scoped env `API_KEYS`.
Env key format note: in `HIPAA_MODE`/`APP_ENV=prod`, env keys must be tenant-scoped: `API_KEYS="<tenant_uuid>:<key>,..."` (unscoped keys are rejected).
Session implementation note: UI sessions are now server-side in `ui_sessions`; client cookies store only a signed opaque session id.

---

## Enterprise / Healthcare-Grade Hardening Plan

We are implementing the following phases. “Phase 0” items are **blockers** for serious HIPAA posture.

### Phase 0 — Blockers (must-do)
- [x] Tenant isolation enforced everywhere (no cross-tenant IDOR)
- [x] DB-level tenant enforcement via RLS (`007_tenant_rls.sql`) + request/worker tenant context propagation
- [x] RLS bypass protection via forced policies (`011_force_rls.sql`)
- [x] PHI egress gate for all model calls (including Vision OCR/auditor/suggestions)
- [x] Tenant PHI policy + egress audit events (`008_phi_egress_policy.sql`, `phi_egress_events`)
- [x] Reproducible migrations (fresh DB bootstraps match application expectations)
- [x] Server-side sessions and safe MFA setup flows (no MFA secrets in cookie sessions)
- [x] Encrypted long-lived MFA secrets at rest (`users.mfa_secret_encrypted`, `012_encrypt_mfa_secret.sql`)
- [x] Tenant-scoped API keys (hashed, scoped, rotated; no shared global secrets)
- [x] Enforced API key scopes and per-key rate windows (`010_api_key_rate_windows.sql`)
- [x] Tenant IP allowlist enforcement for API and UI auth contexts
- [x] Reverse-proxy trust boundary for tenant IP controls (`TRUST_PROXY_HEADERS`, `TRUSTED_PROXY_IPS`)
- [x] Presigned storage flow for regular document upload/download (`/patients/{id}/documents/presign-upload`, `/documents/{id}/download-url`)
- [x] Remove direct service-role upload from regular request path (`upload_bytes_via_signed_url`)
- [x] Account-scoped, DB-backed MFA lockouts (`016_mfa_lockouts.sql`)

### Phase 1 — Identity and access
- [x] OIDC/SAML SSO scaffolding + tenant domain mapping (`sso.py`, `/ui/sso/*`, `/auth/callback/*`)
- [x] SSO trust hardening (`email_verified`, allowed domains/providers policy)
- [x] Step-up auth for sensitive UI actions (`/ui/step-up`, share/finalize protection)
- [x] RBAC permissions matrix helpers (`authz.py`) with route enforcement for admin/sensitive actions

### Phase 2 — Audit + observability + governance
- [x] Append-only audit events with immutability triggers (`009_audit_events_immutability.sql`)
- [x] Unified audit write path (`_log_action` + `audit_events.py` + legacy `audit_logs` compatibility)
- [x] Concurrency-safe per-tenant audit hash chain state + verifier utility (`013_audit_chain_locking.sql`, `verify_audit_chain.py`)
- [x] Prometheus-style metrics endpoint + request latency/status middleware (`/metrics`)
- [x] Immutable retention manifests and enforced export-before-purge gate (`014_retention_manifests.sql`, `purge_data.py`)
- [x] PHI endpoint audit coverage test guard (`backend/tests/test_audit_coverage.py`)
- [x] AST-based audit coverage check (less brittle than source-string markers)

### Phase 3 — Deployment hardening
- [x] Remove baked `.env` from container image; enforce runtime-injected config (`Dockerfile`)
- [x] CI hardening: migration smoke test + audit-chain verifier + static scan + vulnerability gate (`.github/workflows/ci.yml`)
- [x] SBOM generation + build artifact attestation in CI (`cyclonedx`, `actions/attest-build-provenance`)
- [x] Dependency pinning and weekly refresh PR automation (`backend/requirements.txt`, `.github/workflows/dependency-refresh.yml`)
- [x] Migration integrity checksums enforced at apply time (`schema_migrations.checksum`, `015_schema_migration_checksums.sql`)
- [x] Queue-backed jobs with Redis support and DB fallback (`jobs.py`, `worker.py`)
- [x] Nonce-based CSP script policy + inline handler removal in UI templates

### Phase 4 — Operational readiness
- [x] Key rotation, access review, incident response, and breach-notification runbooks (`doc/runbooks/*.md`)
- [x] One-command compliance evidence pack generation (`backend/scripts/generate_evidence_pack.py`)
- [x] Scheduled controls evidence artifact workflow (`.github/workflows/controls-evidence.yml`)
- [x] Cloud-agnostic control validator (`backend/scripts/validate_controls.py`)
- [x] Backup/restore drill artifact generator (`backend/scripts/backup_restore_drill.py`)
- [x] Runtime policy gate + LLM gateway centralization gate (`validate_runtime_policy.py`, `validate_llm_gateway_usage.py`)
- [x] Secret scan gate and container vulnerability gate in CI (`scan_secrets.py`, Trivy workflow step)

---

## Contribution Rules (Security-Sensitive Repo)

- Never commit secrets (`.env` is gitignored).
- Prefer least-privilege patterns: authz checks close to data access.
- Avoid logging PHI; log only metadata (request_id, tenant_id, ids).
- When introducing a new external processor, update:
  - `doc/PRODUCTION_READINESS.md` (PHI processor + BAA notes),
  - and the PHI egress policy code.

## Execution Backlog Reference

For the concrete build sequence to reach enterprise/HIPAA readiness, use:
- `doc/implementation_plans/07_enterprise_hipaa_execution_plan.md`
