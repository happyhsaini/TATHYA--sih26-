# SIH26190 — Secure Case, Evidence & Legal Document Management System

Smart India Hackathon 2026 prototype
(**PS ID: SIH26190** · NCRB / Ministry of Home Affairs · Blockchain & Cybersecurity)

A case-centric platform for managing sensitive legal and investigation documents: secure
storage, versioning, SHA-256 integrity verification, blockchain hash anchoring, audit
trails, search and a permission-aware AI assistant.

---

## 1. Problem

Investigation and legal workflows still rely on fragmented, paper-heavy document
handling. Sensitive records — FIRs, investigation reports, witness statements, forensic
reports, charge sheets, court filings, evidence records — are hard to locate quickly, easy
to tamper with, and difficult to share securely between authorized stakeholders.

The official problem statement identifies the core pain points:  

* Difficulty locating documents quickly
* Unauthorized access to confidential information
* Document tampering risks
* Lack of version control
* Inefficient collaboration between departments
* Delays in legal/investigative processes
* Poor auditability and compliance tracking

---

## 2. Solution

**Secure Case & Evidence Digital Asset Management System**

The platform combines:

* **Secure document management** — files stored in private object storage (MinIO),
  never exposed through public URLs
* **Case management** — every document belongs to a case; access flows from case
  membership
* **Role-Based Access Control (RBAC)** — five prototype roles (Admin, Investigating
  Officer, Supervisor, Forensic Officer, Prosecutor) with backend-enforced authorization
* **Document versioning** — immutable versions instead of silent overwrites; full history
  with per-version download
* **SHA-256 integrity verification** — every version is cryptographically fingerprinted
* **Blockchain hash anchoring** — hashes anchored on a local Ethereum node for
  tamper-evident proof (the document itself is **never** on-chain)
* **Audit trail** — append-only log of every security-relevant action
* **OCR** — scanned documents are OCR'd and indexed for search
* **Full-text search** — across document names, metadata and extracted text
* **Permission-aware RAG/AI assistant** — answers drawn only from documents the
  current user is authorized to view, with source citations

---

## 3. Architecture

```text
   Next.js Frontend (React, Tailwind)
          │
          ▼
   FastAPI Backend (Python)
          │
          ├── PostgreSQL + pgvector   ← metadata, users, cases, documents,
          │                              versions, audit logs, embeddings
          │
          ├── MinIO                   ← actual document files (private bucket)
          │
          ├── OCR / Search / RAG      ← text extraction, chunking, embeddings,
          │                              retrieval, LLM answering
          │
          └── Hardhat / Ethereum      ← document hash anchoring ONLY
                                         (PDFs are NOT stored on blockchain)
```

**Key separation:** files live in MinIO, metadata in PostgreSQL, hashes on the blockchain.
The blockchain is an integrity layer — it stores only `documentId`, `hash` and `timestamp`,
never the document content.

---

## 4. SIH Requirement Mapping

| SIH Requirement                      | Implemented Solution                                    |
| ------------------------------------ | ------------------------------------------------------- |
| Centralized document storage         | PostgreSQL (metadata) + MinIO (files)                   |
| Secure access and confidentiality    | JWT + bcrypt + RBAC + case-level authorization          |
| Prevent unauthorized modification    | Immutable versions + SHA-256 hashing                    |
| Tamper detection                     | Hash verification + blockchain anchor                   |
| Complete audit trail                 | Append-only audit log (every security-relevant action)  |
| Efficient search and retrieval       | Full-text search + permission-aware RAG                 |
| Collaboration between stakeholders   | Case membership + role-based access                     |
| Intelligent document handling        | OCR + AI assistant with source citations               |
| Compliance-supporting traceability   | Version history + audit trail + integrity verification  |

---

## 5. Security

* **Authentication** — JWT with bcrypt password hashing
* **Authorization** — RBAC capability matrix + case membership + backend-enforced
  checks on every protected operation (frontend visibility is never a security control)
* **Private storage** — MinIO bucket is private; all file access is proxied through the
  backend, never via public URLs
* **File validation** — MIME type + magic-byte checking, file size limits
* **Immutable versions** — old versions are preserved as read-only history
* **Integrity** — SHA-256 fingerprint of every document version
* **Blockchain anchoring** — hashes anchored on a local Ethereum node
* **Audit trail** — append-only; never logs passwords, JWT secrets or file contents
* **Permission-aware RAG** — retrieval is filtered by user permissions before any LLM
  call; prompt-injection input is treated as untrusted document data
* **No existence leak** — unauthorized access returns 404 (not 403) to avoid revealing
  which documents exist

---

## 6. Blockchain — How It Works (Judge-Facing Explanation)

> **The blockchain does NOT store the document.**

```text
Document (PDF)
      │
      ▼
  SHA-256 hash  ──────────────────►  PostgreSQL  +  Blockchain
      │                                    │
      │                                    │  stores: documentId, hash, timestamp
      │                                    │  (never the PDF)
      ▼
  Stored securely in MinIO
```

**Verification:**

```text
Current document
      │
      ▼
  SHA-256 hash  ──── compare ────  recorded hash (PostgreSQL + blockchain)
                                      │
                                      ├── match     → ✅ VERIFIED
                                      └── mismatch  → 🚨 INTEGRITY FAILURE
```

This makes tampering detectable: if a stored file is modified, its SHA-256 hash changes
and no longer matches the hash recorded in PostgreSQL and anchored on the blockchain.

---

## 7. Tech Stack

| Layer        | Technology                                            |
| ------------ | ----------------------------------------------------- |
| Frontend     | Next.js (App Router) + React + TypeScript + Tailwind  |
| Backend      | FastAPI + SQLAlchemy 2.x + Pydantic v2 (Python 3.11+) |
| Database     | PostgreSQL 16 + pgvector                              |
| Object store | MinIO                                                 |
| Auth         | JWT + bcrypt                                          |
| Blockchain   | Hardhat + Solidity (local Ethereum node)              |
| OCR          | Tesseract + PyMuPDF                                   |
| AI / RAG     | LLM + embeddings + pgvector retrieval                 |
| Container    | Docker Compose                                        |

---

## 8. Project Structure

```
/frontend    Next.js (App Router) + TypeScript + Tailwind CSS
/backend     FastAPI + SQLAlchemy 2.x + Pydantic v2 (Python 3.11+)
/blockchain  Hardhat + Solidity (DocumentIntegrity contract)
/docs        Requirements, architecture, database, security docs
/docker      docker-compose.yml — PostgreSQL (pgvector) + MinIO + Hardhat
```

---

## 9. Prerequisites

* **Docker Desktop** (WSL2 + Virtual Machine Platform on Windows)
* **Python 3.11+**
* **Node.js 18+**
* **Tesseract OCR** (optional — for scanned-document OCR; the system degrades
  gracefully if absent)

---

## 10. Local Setup

### 10.1 Environment configuration

```bash
# from the repository root
copy .env.example .env        # Windows
# cp .env.example .env        # macOS/Linux
```

Never commit the real `.env` — only `.env.example` is versioned.

### 10.2 Start infrastructure (PostgreSQL + MinIO + Hardhat)

```bash
docker compose -f docker/docker-compose.yml up -d
```

> **Windows prerequisite:** if Docker reports "Virtual Machine Platform not enabled",
> run `wsl --install --no-distribution` and **reboot** — the feature only takes effect
> after a restart.

This starts:

* **PostgreSQL** → `localhost:5432` (user `sih`, db `sih26190`)
* **MinIO** S3 API → `http://localhost:9000`, console → `http://localhost:9001`
* **Hardhat** local Ethereum node → `http://localhost:8545` (hash anchoring ONLY)

Enable the pgvector extension once inside the database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### 10.3 Deploy the integrity contract (once per fresh chain)

```bash
cd blockchain
npm install
npx hardhat run scripts/deploy.js --network localhost
```

Copy the printed address into `backend/.env`:

```
BLOCKCHAIN_CONTRACT_ADDRESS=0x...
```

> The address changes whenever the Hardhat chain resets. Re-deploy and update `.env`
> if uploads report `BLOCKCHAIN_REGISTRATION_FAILED`.

### 10.4 Backend

```bash
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt        # Windows
# .venv/bin/pip install -r requirements.txt          # macOS/Linux
.venv\Scripts\uvicorn app.main:app --reload --port 8000
```

* API root → `http://localhost:8000`
* Health check → `http://localhost:8000/health`
* Interactive docs → `http://localhost:8000/docs`

### 10.5 Frontend

```bash
cd frontend
npm install
npm run dev
```

* App → `http://localhost:3000`

### 10.6 Seed demo data

```bash
cd backend
python -m app.seed_demo --reset     # clear existing demo data
python -m app.seed_demo             # create fresh demo data
```

### 10.7 Stop / clean the environment

```bash
docker compose -f docker/docker-compose.yml down          # stop, keep data
docker compose -f docker/docker-compose.yml down -v       # stop AND wipe data
```

---

## 11. Demo Data

> **All data is fictional.** No real police, legal or sensitive data is used anywhere in
> this prototype. Credentials are for development/demo only.

**Demo password (all users):** `Demo@2026!`

| Username            | Role                 | Notes                          |
| ------------------- | -------------------- | ------------------------------ |
| `admin_demo`        | ADMIN                | Full system access             |
| `io_reddy`          | INVESTIGATING_OFFICER| Assigned to several cases      |
| `supervisor_patil`  | SUPERVISOR           | Department-level review        |
| `forensic.kumar`    | FORENSIC_OFFICER     | Forensic/evidence access       |
| `prosecutor.sharma` | PROSECUTOR           | Legal/court document access    |

**Email pattern:** `<username>@demo.sih` (e.g. `io.reddy@demo.sih`).

**Demo cases:** `CASE-2026-001` through `CASE-2026-007` — covering fraud, narcotics,
assault, property crime, arms trafficking, missing persons and embezzlement scenarios.

**Reset/reseed:** `python -m app.seed_demo --reset` then `python -m app.seed_demo`.

---

## 12. Recommended Demo Flow (for SIH Judges)

```text
 1. Login as Investigating Officer (io.reddy@demo.sih)
 2. Open a case (e.g. CASE-2026-001)
 3. Show authorized documents (RBAC: IO sees only assigned cases)
 4. Open a document → show SHA-256 hash
 5. Verify integrity → ✅ VERIFIED (with blockchain transaction hash)
 6. Show version history + per-version download
 7. Upload a new version → old version preserved as read-only history
 8. Search documents (full-text across authorized cases)
 9. Ask the AI assistant a question → answer + source citations
10. Show the audit trail (every action logged)
11. Demonstrate tamper detection:
      modify a stored file → verify again → 🚨 INTEGRITY FAILURE
12. Login as a different role → show restricted access
```

**What to highlight at each step:**

* **Login** — JWT authentication, bcrypt password hashing
* **Case view** — case-centric design, RBAC enforcement
* **Document + hash** — SHA-256 fingerprint shown to the user
* **Verify** — three-layer check: file vs stored hash vs blockchain anchor
* **Versions** — immutable history, no silent overwrites
* **Search** — permission-filtered full-text search
* **AI** — permission-aware retrieval, source citations, honest "not found" instead of
  hallucination
* **Audit** — append-only trail, never logs secrets
* **Tamper** — the core differentiator: blockchain-anchored integrity proof
* **RBAC** — backend-enforced, not just hidden buttons

---

## 13. Status & Scope

This is an **SIH prototype**, not a production government deployment. It is a technically
credible proof of concept demonstrating the required capabilities. It does **not** claim
production-grade identity integration, real NCRB/court connectivity, legal certification
or national-scale deployment.

**Implemented and verified:** Auth, RBAC, case management, secure document upload/storage,
version control, SHA-256 integrity, blockchain anchoring, audit trail, search, OCR, and a
permission-aware RAG/AI assistant.

---

## 14. Documentation

See `docs/requirements.md`, `docs/architecture.md`, `docs/database.md`,
`docs/security.md` and `docs/implementation-plan.md` for the full design and roadmap.
