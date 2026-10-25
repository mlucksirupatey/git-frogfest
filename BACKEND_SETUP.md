# ClaimSetu Backend Architecture & Setup Guide

This guide documents the complete backend services for ClaimSetu so any frontend developer or autonomous coding assistant can spin up, seed, verify, and interact with the full backend stack in minutes.

---

## 1. System Architecture

```text
┌────────────────────────────────────────────────────────────────────────┐
│                          Next.js App (Port 3000)                        │
│                                                                        │
│  [POST /api/claims/:id/documents]  --> Upload, Deduplicate, Structure  │
│  [POST /api/claims/:id/evaluate]   --> Deterministic HIPML Eval        │
│  [POST /api/claims/:id/reason]     --> Evidence Reasoning & Reports    │
└──────────────┬──────────────────┬─────────────────┬────────────────────┘
               │                  │                 │
               v                  v                 v
     ┌─────────────────┐ ┌────────────────┐ ┌───────────────────────────┐
     │ MinIO S3 (9000) │ │ HIPML (3001)   │ │ RAG Retrieval (4100)      │
     │ claim-docs      │ │ Vitraya Engine │ │ Express + pgvector        │
     │ SHA-256 Storage │ │ .pml Rule Eval │ │ Gemini 3072 Embeddings    │
     └─────────────────┘ └────────────────┘ └─────────────┬─────────────┘
                                                          │
                                                          v
                                            ┌───────────────────────────┐
                                            │ PostgreSQL + pgvector     │
                                            │ Port: 5432                │
                                            │ Database: claimsetu       │
                                            │ Tables & Pre-seeded Embeds│
                                            └───────────────────────────┘
```

### Services & Port Assignments

| Service | Port | Description | Managed By |
| :--- | :--- | :--- | :--- |
| **PostgreSQL 16** | `5432` | Primary database with `pgvector` extension and pre-seeded embeddings | `docker-compose.yml` (`postgres-local`) |
| **MinIO Object Storage** | `9000` (API)<br>`9001` (Console) | S3-compatible content-addressed storage for original PDF uploads | `docker-compose.yml` (`minio-local`) |
| **Vitraya HIPML Engine** | `3001` | Standalone deterministic rule parser & evaluator (Node 14 legacy runtime) | `docker-compose.yml` (`hipml-service`) |
| **RAG Policy Retrieval** | `4100` | Express REST API for pgvector semantic search against policy clauses | Node.js process (`rag_demo_claim_setu`) |
| **Next.js Web App** | `3000` | Frontend UI + Claims pipeline, aggregator, reasoning engine API routes | Node.js process (`nextjs`) |

---

## 2. Quickstart in 3 Steps

### Step 1: Start Docker Infrastructure (PostgreSQL, MinIO, HIPML)
In the repository root:
```bash
docker compose up -d
```
> **What this does:**
> - Spins up `postgres-local` with `pgvector` pre-initialized using `db/seed_full.sql` (contains all tables, policy #1, claim #1, and all 21 pre-computed 3072-dimensional policy vector embeddings).
> - Spins up `minio-local` (credentials: `minioadmin` / `miniopassword`).
> - Spins up `hipml-service` on port `3001` in an isolated Node 14 container, running the Vitraya rule parser.

Verify all 3 containers are healthy:
```bash
docker ps
```

#### Step 1.1: Run Database Migrations & Seeds
When spinning up or synchronizing an existing database volume with the latest schema, institutions, policies, and auth tables:
```bash
# Option A: One-command runner via pnpm
pnpm run db:migrate

# Option B: Direct Docker psql execution
docker exec -i postgres-local psql -U postgres -d claimsetu < db/004_seed_all_policies.sql
docker exec -i postgres-local psql -U postgres -d claimsetu < db/005_auth_system.sql
```

**Quick Verification:**
```bash
docker exec -it postgres-local psql -U postgres -d claimsetu -c "SELECT id, name, code FROM institutions;"
docker exec -it postgres-local psql -U postgres -d claimsetu -c "SELECT id, name, hipml_file_id FROM policies;"
docker exec -it postgres-local psql -U postgres -d claimsetu -c "SELECT id, email, role FROM users;"
```

---

### Step 2: Start the RAG Retrieval Service (Port 4100)
In a separate terminal:
```bash
cd rag_demo_claim_setu

# 1. Copy environment template if not present
cp -n .env.example .env

# 2. Ensure GEMINI_API_KEY is configured in .env (for runtime cosine similarity embeddings)

# 3. Install dependencies and start server
pnpm install
pnpm start
```
Verify RAG service health:
```bash
curl http://localhost:4100/health
# Expected: {"status":"ok","db":{"connected":true,"pgvectorVersion":"0.8.6"}}
```

---

### Step 3: Start the Next.js Dev Server (Port 3000)
In a separate terminal:
```bash
cd nextjs

# 1. Copy environment template if not present
cp -n .env.example .env.local

# 2. Ensure GEMINI_API_KEY is configured in .env.local

# 3. Install dependencies and start server
pnpm install
pnpm run dev
```

---

## 3. Environment Variables Reference

### Next.js (`nextjs/.env.local`)
```env
# PostgreSQL
POSTGRES_DB=claimsetu
POSTGRES_USER=postgres
POSTGRES_PASSWORD=devpassword
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
PORT=3000

# Auth Secrets
NEXTAUTH_SECRET="your-secret-key-min-32-chars"
NEXT_AUTH_SECRET="your-secret-key-min-32-chars"
NEXTAUTH_URL="http://localhost:3000"
NEXT_PUBLIC_APP_URL="http://localhost:3000"

# Gemini AI (Structured Extraction & Evidence Reasoning)
GEMINI_API_KEY=YOUR_GEMINI_API_KEY

# MinIO S3 Object Storage
MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_USE_SSL=false
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=miniopassword
MINIO_BUCKET=claim-docs

# Microservices
HIPML_SERVICE_URL=http://localhost:3001
RAG_SERVICE_URL=http://localhost:4100

# Models
EXTRACTION_MODEL=gemini-3.5-flash
REASONING_MODEL=gemini-3.5-flash
```

### RAG Service (`rag_demo_claim_setu/.env`)
```env
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
EMBEDDING_MODEL=gemini-embedding-001
EMBEDDING_DIMENSIONS=3072
DATABASE_URL=postgresql://postgres:devpassword@localhost:5432/claimsetu
PORT=4100
TOP_K=3
```

---

## 4. Key API Endpoints for Frontend Integration

### 1. Document Upload & Extraction Pipeline
```http
POST /api/claims/:id/documents
Content-Type: multipart/form-data
```
**Form Fields:**
- `claim`: PDF file (Claim Form)
- `discharge_summary`: PDF file (Discharge Summary)
- `bill`: PDF file (Hospital Invoice)
- `prescription`: (Optional) PDF file

**Behavior:**
- Computes SHA-256 hash.
- Uploads to MinIO.
- Extracts text via `pdf-parse`.
- Structures facts via Gemini with Zod validation.
- Mechanically aggregates facts into unified Claim JSON (with duplicate line-item conflict checks).
- Runs deterministic Vitraya HIPML rules.
- Retrieves RAG policy evidence for flagged findings.
- **Fast Deduplication Invariant:** Re-uploading identical bytes reuses existing document records and extractions with 0 new LLM calls (~600ms vs ~15s).

---

### 2. Deterministic Claim Evaluation
```http
POST /api/claims/:id/evaluate
Content-Type: application/json
```
**Behavior:**
- Runs the claim's structured data against Vitraya HIPML rule file (`claimsetu-mvp`).
- Returns condition verdicts (`Patient Eligibility`, `Claim Admissibility`) and sub-limits.

---

### 3. Post-Deterministic Reasoning & Report Generation
```http
POST /api/claims/:id/reason?dry_run=true|false
Content-Type: application/json
```
**Query Parameters:**
- `dry_run=true`: Evaluates reasoning, synthesizes reports, returns response **without** modifying PostgreSQL.
- `dry_run=false`: Evaluates reasoning, updates `claims.status` (`REJECTED_DETERMINISTIC`, `NEEDS_REVIEW`, `APPROVED_DETERMINISTIC`), and persists full report to `claims.reasoning_result`.

**Response JSON Payload:**
```json
{
  "success": true,
  "claimId": 1,
  "disposition": "failed",
  "analysis": {
    "report_title": "Claims Analysis Report",
    "analysis_status": "complete",
    "deterministic_disposition": "failed",
    "executive_assessment": "...",
    "claim_context": { ... },
    "findings": [ ... ],
    "cross_document_analysis": [ ... ],
    "financial_impact": [ ... ],
    "open_questions": [ ... ],
    "human_review_actions": [ ... ],
    "operational_conclusion": "...",
    "summary_report": "..."
  },
  "fullReport": "# Markdown 9-section report (~1,200 words)...",
  "summaryReport": "### Markdown executive summary (~220 words)...",
  "evidence": [
    { "evidence_id": "E1", "clause_ref": "6.1", "section_title": "Exclusions", ... }
  ],
  "citations": {
    "E1": { "clause_ref": "6.1", "section_title": "Section 6 - Exclusions", "page_start": 8 }
  },
  "persisted": false
}
```

---

## 5. Verification Commands & Test Suites

Run these from `nextjs/` to verify the entire system end-to-end:

```bash
# 1. Run all unit tests (PDF extraction, aggregator, HIPML mapper, MinIO storage, RAG retrieval, reasoning unit tests)
pnpm test

# 2. Run LLM structured extraction tests
pnpm run test:llm

# 3. Run multi-document upload and deduplication E2E pipeline tests
pnpm run test:e2e

# 4. Run post-deterministic reasoning unit tests (9 tests)
pnpm run test:reasoning:unit

# 5. Run live reasoning E2E integration tests (with RAG & Gemini)
pnpm run test:reasoning:e2e

# 6. Test reasoning via CLI script
node scripts/test_reasoning.mjs 1 --dry-run
```
