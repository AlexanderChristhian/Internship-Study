# LLM RAG Project Structure — Best Practices & Guidelines

**Date:** 9 March 2026  
**Topic:** How to structure the DigiChecker LLM RAG project with Python (LlamaIndex), Express.js, ChromaDB, and Neo4j  
**Companion Note:** [9-Mar-2026_LlamaIndex-RAG-Stack-Implementation.md](9-Mar-2026_LlamaIndex-RAG-Stack-Implementation.md)  
**General Practices:** [8-Mar-2026_Project-Structure-Good-Practices.md](8-Mar-2026_Project-Structure-Good-Practices.md)

---

## Table of Contents

1. [Architecture Decision: Two-Service Backend](#1-architecture-decision-two-service-backend)
2. [Full Monorepo Project Structure](#2-full-monorepo-project-structure)
3. [Express.js API Service Structure](#3-expressjs-api-service-structure)
4. [Python RAG Service Structure](#4-python-rag-service-structure)
5. [Frontend Integration Points](#5-frontend-integration-points)
6. [Configuration & Environment Management](#6-configuration--environment-management)
7. [Database Layer Organization](#7-database-layer-organization)
8. [Service Communication Patterns](#8-service-communication-patterns)
9. [Error Handling & Logging Strategy](#9-error-handling--logging-strategy)
10. [Testing Structure for RAG Systems](#10-testing-structure-for-rag-systems)
11. [File/Document Storage Strategy](#11-filedocument-storage-strategy)
12. [Docker & Deployment Structure](#12-docker--deployment-structure)
13. [Naming Conventions & Code Standards](#13-naming-conventions--code-standards)
14. [Common RAG Project Anti-Patterns](#14-common-rag-project-anti-patterns)
15. [Checklist: Before You Start Coding](#15-checklist-before-you-start-coding)

---

## 1. Architecture Decision: Two-Service Backend

### Why Split Express.js and Python?

The DigiChecker project has two distinct backend concerns:

| Concern | Best Tool | Reason |
|---------|-----------|--------|
| REST API, auth, CRUD, file uploads | Express.js (Node.js) | Fast I/O, strong ecosystem for web APIs, TypeScript support |
| Embedding, indexing, querying, LLM calls | Python (LlamaIndex) | LlamaIndex/BGE-M3/ChromaDB are Python-native, ML ecosystem lives in Python |

Trying to force everything into one language leads to:
- Poor library support (LlamaIndex has no mature JS port)
- Reinventing the wheel (embedding models are Python-first)
- Maintenance burden (fighting the ecosystem instead of leveraging it)

### Why Neo4j Stays INSIDE Express.js (Not a Separate Service)

Unlike LlamaIndex/BGE-M3 which are Python-only, Neo4j has an **official first-class Node.js driver** (`neo4j-driver`). There is no ecosystem mismatch — Cypher queries run natively from TypeScript.

| Factor | Separate Neo4j Service | Merged with Express.js |
|--------|----------------------|------------------------|
| **Driver** | Unnecessary indirection — official Node.js driver exists | Direct Bolt protocol, no HTTP overhead |
| **Compute** | Cypher queries are lightweight app-side (Neo4j server does the work) | No performance reason to isolate |
| **Coupling** | Plagiarism is tightly coupled to submission CRUD (create/delete must sync graph nodes) | Transactional consistency in one process |
| **Deployment** | Extra container, networking, failure points | One fewer service to orchestrate |
| **Contrast** | Python service exists because LlamaIndex/BGE-M3 have NO JS support | Neo4j works natively in Node.js |

**Decision:** Neo4j lives inside the Express.js backend as a data access layer (like Prisma for SQL), NOT as a separate microservice.

### Communication Between Services

```
Frontend (Next.js)
    │
    │  All requests go through Express.js (single entry point)
    ▼
Express.js (API Gateway + Neo4j)
    │
    ├──▶ SQL DB (direct — via Prisma ORM)
    ├──▶ Neo4j (direct — via neo4j-driver, Bolt protocol)
    │
    └──▶ Python RAG Service (HTTP — internal network only)
              │
              ├──▶ BGE-M3 (in-process — loaded in memory)
              ├──▶ ChromaDB (direct — client connection)
              └──▶ LLM API (external — OpenAI/Ollama/etc.)
```

**Key Rule:** The frontend NEVER talks to the Python service directly. Express.js is the single gateway. This keeps auth, rate-limiting, and error handling in one place.

---

## 2. Full Monorepo Project Structure

```
Real DigiChecker/
│
├── Docs/
│   ├── Study Notes/                    # Research & learning (these notes)
│   ├── Architecture/                   # ADRs (Architecture Decision Records)
│   │   ├── 001-two-service-backend.md
│   │   ├── 002-bge-m3-over-openai.md
│   │   └── 003-neo4j-for-plagiarism.md
│   └── API/                            # API documentation
│       └── openapi.yaml                # OpenAPI spec (auto-generated or manual)
│
├── Submission_Checker/
│   ├── README.md                       # Project overview + setup instructions
│   │
│   ├── src/
│   │   ├── backend/                    # Express.js API service
│   │   │   └── (see Section 3)
│   │   │
│   │   ├── frontend/                   # Next.js app
│   │   │   └── (see Section 5)
│   │   │
│   │   └── rag-service/                # Python LlamaIndex service
│   │       └── (see Section 4)
│   │
│   ├── tests/
│   │   ├── backend/                    # Express.js tests
│   │   ├── frontend/                   # React/Next.js tests
│   │   ├── rag-service/                # Python RAG tests
│   │   └── integration/               # Cross-service tests
│   │
│   ├── infrastructure/
│   │   ├── docker/
│   │   │   ├── Dockerfile.backend
│   │   │   ├── Dockerfile.frontend
│   │   │   └── Dockerfile.rag
│   │   └── scripts/
│   │       ├── setup-dev.sh            # One-command dev environment setup
│   │       └── seed-db.sh              # Seed SQL + Neo4j with test data
│   │
│   ├── docker-compose.yml              # Orchestrate all services locally
│   ├── docker-compose.dev.yml          # Dev overrides (hot reload, volumes)
│   ├── .env.example                    # Template for required env vars
│   └── Makefile                        # Common commands
│
└── .gitignore
```

### Key Structural Rules

1. **Each service is self-contained** — has its own package.json / pyproject.toml, can run independently
2. **Tests mirror source** — `tests/backend/` corresponds to `src/backend/`
3. **Infrastructure is separate** — Docker, scripts, and config files don't mix with source code
4. **Docs live outside src/** — documentation is a first-class citizen, not buried in code folders
5. **Single `.env.example`** at the monorepo root — all services read from the same `.env` file via docker-compose

---

## 3. Express.js API Service Structure

```
src/backend/
├── package.json
├── tsconfig.json
├── .eslintrc.json
│
├── prisma/                         # Prisma ORM (SQL data access)
│   ├── schema.prisma               # Database schema definition
│   ├── migrations/                 # Auto-generated migration files
│   └── seed.ts                     # Database seed script
│
└── src/
    ├── index.ts                    # App entry point — starts server
    ├── app.ts                      # Express app setup (middleware, routes)
    │
    ├── config/
    │   ├── index.ts                # Central config (reads env vars, validates)
    │   ├── database.ts             # Prisma client singleton
    │   └── neo4j.ts                # Neo4j driver singleton
    │
    ├── routes/
    │   ├── index.ts                # Aggregates all route modules
    │   ├── auth.routes.ts
    │   ├── submission.routes.ts
    │   ├── analysis.routes.ts
    │   └── plagiarism.routes.ts
    │
    ├── controllers/                # Handle HTTP request/response
    │   ├── auth.controller.ts
    │   ├── submission.controller.ts
    │   ├── analysis.controller.ts
    │   └── plagiarism.controller.ts
    │
    ├── services/                   # Business logic (no HTTP or DB awareness)
    │   ├── auth.service.ts
    │   ├── submission.service.ts
    │   ├── rag.service.ts          # Calls Python RAG service via HTTP
    │   └── plagiarism.service.ts   # Orchestrates graph repo + SQL repo
    │
    ├── repositories/               # Data access — SQL (via Prisma)
    │   ├── user.repo.ts
    │   ├── submission.repo.ts
    │   └── plagiarism-report.repo.ts  # SQL plagiarism_reports table
    │
    ├── graph/                      # Data access — Neo4j (via neo4j-driver)
    │   ├── neo4j.client.ts         # Driver singleton + runQuery() helper
    │   ├── submission.graph.ts     # Create/delete Submission & Section nodes
    │   ├── similarity.graph.ts     # Create/query SIMILAR_TO & MATCHES edges
    │   └── queries/                # Raw Cypher query templates
    │       ├── create-nodes.cypher.ts
    │       ├── find-similar.cypher.ts
    │       ├── detect-clusters.cypher.ts
    │       └── course-heatmap.cypher.ts
    │
    ├── middleware/
    │   ├── auth.middleware.ts      # JWT verification
    │   ├── validation.middleware.ts # Request body validation (Zod)
    │   ├── upload.middleware.ts    # Multer file upload config
    │   └── error.middleware.ts     # Global error handler
    │
    ├── validators/                 # Zod schemas for request validation
    │   ├── auth.validator.ts
    │   ├── submission.validator.ts
    │   └── common.validator.ts
    │
    ├── types/                      # TypeScript type definitions
    │   ├── express.d.ts            # Extend Express Request type
    │   ├── submission.types.ts
    │   └── api.types.ts
    │
    └── utils/
        ├── logger.ts               # Winston/Pino logger setup
        ├── errors.ts               # Custom error classes (AppError, NotFoundError)
        └── helpers.ts              # Misc utility functions
```

### Why `graph/` Is Separate from `repositories/`

Both are data access layers, but they're fundamentally different:

| Aspect | `repositories/` (SQL) | `graph/` (Neo4j) |
|--------|----------------------|------------------|
| ORM | Prisma (type-safe, generated client) | Raw Cypher queries via neo4j-driver |
| Query language | Prisma Query API (TypeScript) | Cypher (string-based graph query language) |
| Data shape | Rows + columns (relational) | Nodes + edges (graph) |
| Purpose | CRUD metadata (users, submissions, reports) | Relationship traversal (similarity, clusters, chains) |

Merging them into one folder would be confusing — Prisma repos and Cypher graph modules have completely different patterns. Keeping `graph/` separate makes it clear which data access layer you're working with.

### How the Layers Connect for Plagiarism

```
plagiarism.controller.ts
    │
    ▼
plagiarism.service.ts          (business logic — orchestrates everything)
    │
    ├──▶ rag.service.ts            → calls Python service for similarity scores
    ├──▶ similarity.graph.ts       → writes SIMILAR_TO edges to Neo4j
    ├──▶ submission.graph.ts       → reads graph for clusters/chains
    └──▶ plagiarism-report.repo.ts → saves final report to SQL
```

### Layer Responsibility Rules

```
HTTP Request
    │
    ▼
┌─────────────┐  Validates request, extracts params, sends response
│  Controller  │  ✅ Knows about: req, res, HTTP status codes
│              │  ❌ No business logic, no DB queries
└──────┬──────┘
       │
       ▼
┌─────────────┐  Contains business rules, orchestrates operations
│   Service    │  ✅ Knows about: business rules, other services
│              │  ❌ No HTTP awareness, no direct DB access
└──────┬──────┘
       │
       ▼
┌─────────────┐  Executes DB queries, returns raw data
│  Repository  │  ✅ Knows about: database, queries, ORM
│              │  ❌ No business logic, no HTTP awareness
└─────────────┘
```

**Why this matters:** If you want to swap Prisma for Drizzle, you only change the repository layer. If you want to add a GraphQL API alongside REST, you add new controllers that call the same services. Each layer is independently testable.

### Example: Submission Flow

```typescript
// routes/submission.routes.ts
router.post(
    '/submissions',
    authMiddleware,                              // 1. Verify JWT
    uploadMiddleware.single('file'),             // 2. Handle file upload
    validate(createSubmissionSchema),            // 3. Validate body
    submissionController.create                  // 4. Handle request
);

// controllers/submission.controller.ts
async create(req: Request, res: Response, next: NextFunction) {
    try {
        const submission = await submissionService.create({
            userId: req.user.id,
            file: req.file,
            ...req.body
        });
        res.status(201).json(submission);
    } catch (error) {
        next(error);  // Goes to error middleware
    }
}

// services/submission.service.ts
async create(data: CreateSubmissionInput) {
    // 1. Save metadata to SQL
    const submission = await submissionRepo.create(data);

    // 2. Send document to RAG service for indexing
    await ragService.indexDocument(submission.id, data.file);

    // 3. Create Neo4j node for plagiarism graph
    await plagiarismService.createNode(submission);

    return submission;
}
```

---

## 4. Python RAG Service Structure

```
src/rag-service/
├── pyproject.toml                  # Dependencies + project config
├── requirements.txt                # Pinned dependencies (generated from pyproject.toml)
│
├── app/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app entry point
│   ├── config.py                   # Settings (from env vars, via pydantic-settings)
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py               # All API routes
│   │   └── schemas.py              # Pydantic request/response models
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── embeddings.py           # BGE-M3 model loading + encoding
│   │   ├── indexing.py             # Document chunking + indexing pipeline
│   │   ├── querying.py             # Query engine setup + execution
│   │   └── prompts.py              # System prompts + prompt templates
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── document_service.py     # Document processing (load, chunk, clean)
│   │   ├── index_service.py        # Index management (create, update, delete)
│   │   ├── query_service.py        # Query orchestration (retrieve + generate)
│   │   └── similarity_service.py   # Compute similarity for plagiarism
│   │
│   ├── stores/
│   │   ├── __init__.py
│   │   ├── chroma_store.py         # ChromaDB client + collection management
│   │   └── vector_store.py         # LlamaIndex VectorStore wrapper
│   │
│   └── utils/
│       ├── __init__.py
│       ├── text_processing.py      # Text cleaning, normalization
│       └── logger.py               # Structured logging setup
│
├── data/                           # Local data directory (gitignored)
│   ├── chroma_db/                  # ChromaDB persistent storage
│   └── uploads/                    # Temp uploaded files (processed then removed)
│
└── tests/
    ├── __init__.py
    ├── conftest.py                 # Shared fixtures
    ├── test_embeddings.py
    ├── test_indexing.py
    └── test_querying.py
```

### Key Design Decisions

#### 1. FastAPI as the HTTP layer (not Flask)

| Feature | FastAPI | Flask |
|---------|---------|-------|
| Async support | Native | Requires extensions |
| Auto docs (Swagger) | Built-in | Manual setup |
| Request validation | Pydantic (built-in) | Manual or extensions |
| Type hints | First-class | Optional |
| Performance | High (async, Uvicorn) | Medium |

#### 2. Separate `core/` from `services/`

```
core/        → HOW things work (embedding model, indexing pipeline, query engine)
services/    → WHAT to do (process this document, answer this query, find similar docs)
```

- `core/embeddings.py` knows how to load BGE-M3 and generate embeddings
- `services/index_service.py` knows that when a new submission arrives, it should chunk it, embed it, and store it in ChromaDB

This separation means you can swap BGE-M3 for another model by changing only `core/embeddings.py` — services don't care about the underlying model.

#### 3. Singleton Pattern for Expensive Resources

BGE-M3 and ChromaDB clients should be initialized once and reused:

```python
# core/embeddings.py
from functools import lru_cache
from FlagEmbedding import BGEM3FlagModel

@lru_cache(maxsize=1)
def get_embedding_model() -> BGEM3FlagModel:
    """Load BGE-M3 once and cache in memory."""
    return BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)
```

```python
# stores/chroma_store.py
from functools import lru_cache
import chromadb
from app.config import settings

@lru_cache(maxsize=1)
def get_chroma_client() -> chromadb.ClientAPI:
    """Create persistent ChromaDB client once."""
    return chromadb.PersistentClient(path=settings.CHROMA_DB_PATH)

def get_collection(name: str = "submissions"):
    client = get_chroma_client()
    return client.get_or_create_collection(
        name=name,
        metadata={"hnsw:space": "cosine"}
    )
```

#### 4. Prompt Templates in Their Own File

```python
# core/prompts.py

GRADING_SYSTEM_PROMPT = """You are an academic grading assistant.
Evaluate the student submission based on the provided rubric.
Be fair, specific, and constructive in your feedback.
Always cite specific parts of the submission in your evaluation."""

GRADING_QUERY_TEMPLATE = """
Rubric:
{rubric}

Student Submission (relevant sections):
{context}

Question: {query}

Provide a grade and detailed feedback.
"""

SIMILARITY_ANALYSIS_PROMPT = """Compare the following two text sections
and identify specific overlapping ideas, phrases, or structures.
Rate the similarity from 0 to 100."""
```

**Why a separate file?** Prompts are iterated on frequently. Keeping them in one place makes it easy to version, review, and A/B test different prompts without touching service logic.

---

## 5. Frontend Integration Points

The frontend should treat the Express.js API as a black box. It doesn't know about Python, ChromaDB, or Neo4j.

### API Client Structure

```
src/frontend/
├── services/                       # API client layer
│   ├── api.ts                      # Base fetch/axios client with auth headers
│   ├── submissionApi.ts            # Submission CRUD calls
│   ├── analysisApi.ts              # RAG analysis calls
│   └── plagiarismApi.ts            # Plagiarism check calls
│
├── types/                          # Shared TypeScript types
│   ├── submission.ts               # Submission, SubmissionStatus, etc.
│   ├── analysis.ts                 # AnalysisResult, Grade, etc.
│   └── plagiarism.ts               # PlagiarismReport, Match, etc.
```

### Type Definitions Should Mirror the API

```typescript
// types/submission.ts
export interface Submission {
    id: string;
    submissionId: string;    // "SUB-1042"
    name: string;
    date: string;
    status: 'Pending' | 'Checked';
    grade: string | null;
    fileUrl: string;
}

// services/submissionApi.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

export async function getSubmissions(): Promise<Submission[]> {
    const res = await fetch(`${API_BASE}/api/submissions`, {
        headers: { Authorization: `Bearer ${getToken()}` }
    });
    if (!res.ok) throw new Error('Failed to fetch submissions');
    return res.json();
}
```

**Rule:** Never put API URLs or backend logic in component files. Components call service functions; service functions call the API.

---

## 6. Configuration & Environment Management

### Single `.env.example` at Root

```bash
# .env.example — copy to .env and fill in values

# === Express.js Backend ===
NODE_ENV=development
PORT=3001
DATABASE_URL=postgresql://user:password@localhost:5432/digichecker
JWT_SECRET=your-secret-key-here

# === Neo4j ===
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password

# === Python RAG Service ===
RAG_SERVICE_URL=http://localhost:8000
CHROMA_DB_PATH=./data/chroma_db
BGE_M3_MODEL=BAAI/bge-m3

# === LLM ===
LLM_PROVIDER=openai            # openai | ollama | local
OPENAI_API_KEY=sk-...
OLLAMA_BASE_URL=http://localhost:11434

# === Frontend ===
NEXT_PUBLIC_API_URL=http://localhost:3001
```

### Configuration Validation (don't trust env vars blindly)

**Express.js (using Zod):**

```typescript
// config/index.ts
import { z } from 'zod';

const envSchema = z.object({
    NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
    PORT: z.coerce.number().default(3001),
    DATABASE_URL: z.string().url(),
    JWT_SECRET: z.string().min(32),
    NEO4J_URI: z.string(),
    NEO4J_USER: z.string(),
    NEO4J_PASSWORD: z.string(),
    RAG_SERVICE_URL: z.string().url(),
});

export const config = envSchema.parse(process.env);
```

**Python (using pydantic-settings):**

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    chroma_db_path: str = "./data/chroma_db"
    bge_m3_model: str = "BAAI/bge-m3"
    llm_provider: str = "openai"
    openai_api_key: str = ""
    ollama_base_url: str = "http://localhost:11434"
    host: str = "0.0.0.0"
    port: int = 8000

    class Config:
        env_file = ".env"

settings = Settings()
```

**Key Rule:** The app should fail fast at startup if a required config value is missing or invalid. Don't let it crash at runtime when it first tries to use the value.

---

## 7. Database Layer Organization

### Three Databases, Three Purposes

```
┌──────────────────────────────────────────────────────────┐
│                    DATABASE STRATEGY                      │
├──────────────┬───────────────────┬────────────────────────┤
│   SQL (PG)   │   ChromaDB        │   Neo4j                │
├──────────────┼───────────────────┼────────────────────────┤
│ Users        │ Document          │ Submission nodes       │
│ Submissions  │ embeddings        │ Section nodes          │
│ Courses      │ (BGE-M3 vectors)  │ SIMILAR_TO edges       │
│ Grades       │                   │ MATCHES edges          │
│ Reports      │ Chunk text +      │ (with similarity       │
│ Audit logs   │ metadata          │  scores)               │
├──────────────┼───────────────────┼────────────────────────┤
│ Structured   │ Unstructured      │ Relationships          │
│ metadata     │ semantic search   │ & graph traversal      │
└──────────────┴───────────────────┴────────────────────────┘
```

### Each DB Has Its Own Access Layer

Do NOT scatter database calls throughout your code. Create dedicated modules:

```
Express.js:
  repositories/          → SQL (via Prisma)       — relational data
  graph/                 → Neo4j (via neo4j-driver) — graph data

Python RAG Service:
  stores/chroma_store.py → ChromaDB                — vector data
```

The `graph/` folder is Neo4j's equivalent of `repositories/` for SQL. Services call both but never mix Prisma and Cypher in the same file.

### Cross-DB References

The three databases reference each other via IDs:

```
SQL submissions table:
  ├── chroma_doc_id  → points to ChromaDB document
  └── neo4j_node_id  → points to Neo4j node

Neo4j Submission node:
  ├── submission_id  → matches SQL submissions.submission_id
  └── embedding_id   → matches ChromaDB document ID

ChromaDB document:
  └── metadata.submission_id → matches SQL submissions.submission_id
```

**Rule:** SQL is the source of truth for submission metadata. ChromaDB and Neo4j are derived/secondary stores. If they get out of sync, SQL wins.

---

## 8. Service Communication Patterns

### Express.js → Python RAG Service

Use a dedicated service class with retry logic and timeout:

```typescript
// services/rag.service.ts
import { config } from '../config';

class RagService {
    private baseUrl: string;

    constructor() {
        this.baseUrl = config.RAG_SERVICE_URL;
    }

    async indexDocument(submissionId: string, content: string): Promise<void> {
        const response = await fetch(`${this.baseUrl}/index`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ submission_id: submissionId, content }),
            signal: AbortSignal.timeout(30000), // 30s timeout
        });

        if (!response.ok) {
            throw new AppError(
                `RAG indexing failed: ${response.statusText}`,
                response.status
            );
        }
    }

    async query(submissionId: string, query: string): Promise<QueryResult> {
        const response = await fetch(`${this.baseUrl}/query`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ submission_id: submissionId, query }),
            signal: AbortSignal.timeout(60000), // 60s timeout (LLM can be slow)
        });

        if (!response.ok) {
            throw new AppError(`RAG query failed: ${response.statusText}`, response.status);
        }

        return response.json();
    }

    async findSimilar(submissionId: string, topK: number = 10): Promise<SimilarityResult[]> {
        const response = await fetch(`${this.baseUrl}/similar`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ submission_id: submissionId, top_k: topK }),
            signal: AbortSignal.timeout(30000),
        });

        if (!response.ok) {
            throw new AppError(`Similarity search failed`, response.status);
        }

        return response.json();
    }
}

export const ragService = new RagService();
```

### Python RAG Service API Endpoints

```python
# api/routes.py
from fastapi import APIRouter, HTTPException
from app.api.schemas import IndexRequest, QueryRequest, SimilarityRequest
from app.services.index_service import IndexService
from app.services.query_service import QueryService
from app.services.similarity_service import SimilarityService

router = APIRouter()

@router.post("/index")
async def index_document(request: IndexRequest):
    """Chunk, embed, and store a document in ChromaDB."""
    service = IndexService()
    result = service.index(request.submission_id, request.content)
    return {"status": "indexed", "chunks": result.chunk_count}

@router.post("/query")
async def query_document(request: QueryRequest):
    """RAG query against indexed documents."""
    service = QueryService()
    result = service.query(request.submission_id, request.query)
    return {"answer": result.answer, "sources": result.sources}

@router.post("/similar")
async def find_similar(request: SimilarityRequest):
    """Find documents most similar to a given submission."""
    service = SimilarityService()
    matches = service.find_similar(request.submission_id, request.top_k)
    return {"matches": matches}

@router.get("/health")
async def health_check():
    return {"status": "ok"}
```

---

## 9. Error Handling & Logging Strategy

### Centralized Error Handling (Express.js)

```typescript
// utils/errors.ts
export class AppError extends Error {
    constructor(
        message: string,
        public statusCode: number = 500,
        public isOperational: boolean = true
    ) {
        super(message);
        this.name = 'AppError';
    }
}

export class NotFoundError extends AppError {
    constructor(resource: string) {
        super(`${resource} not found`, 404);
    }
}

export class ValidationError extends AppError {
    constructor(message: string) {
        super(message, 400);
    }
}

// middleware/error.middleware.ts
export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
    if (err instanceof AppError) {
        logger.warn(err.message, { statusCode: err.statusCode, path: req.path });
        return res.status(err.statusCode).json({
            error: err.message,
            statusCode: err.statusCode
        });
    }

    // Unexpected errors — log full stack trace
    logger.error('Unhandled error', { error: err, path: req.path, stack: err.stack });
    res.status(500).json({ error: 'Internal server error' });
}
```

### Structured Logging

Use structured JSON logs for easier parsing in production:

```typescript
// utils/logger.ts (using Pino — fastest Node.js logger)
import pino from 'pino';
import { config } from '../config';

export const logger = pino({
    level: config.NODE_ENV === 'production' ? 'info' : 'debug',
    transport: config.NODE_ENV !== 'production'
        ? { target: 'pino-pretty' }
        : undefined,
});

// Usage:
logger.info({ submissionId: 'SUB-1042', action: 'index' }, 'Indexing submission');
logger.error({ error, submissionId }, 'Failed to index submission');
```

### Python Logging (matching format)

```python
# utils/logger.py
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
        }
        if hasattr(record, 'submission_id'):
            log["submission_id"] = record.submission_id
        if record.exc_info:
            log["exception"] = self.formatException(record.exc_info)
        return json.dumps(log)

def get_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    return logger
```

---

## 10. Testing Structure for RAG Systems

### Test Pyramid for RAG

```
        ┌──────────────┐
        │   E2E Tests  │   Few — expensive, slow
        │  (Playwright) │   Test full user flows
        └──────┬───────┘
               │
      ┌────────▼────────┐
      │ Integration     │   Some — test service boundaries
      │ Tests           │   Express ↔ Python, Express ↔ SQL
      └────────┬────────┘
               │
    ┌──────────▼──────────┐
    │   Unit Tests        │   Many — fast, focused
    │   (Jest / Pytest)   │   Test business logic in isolation
    └─────────────────────┘
```

### What to Test in a RAG System

| Layer | What to Test | How |
|-------|-------------|-----|
| Chunking | Correct chunk sizes, overlap, metadata | Unit test with sample documents |
| Embedding | Correct dimensions, deterministic output | Unit test with known inputs |
| Indexing | Documents stored in ChromaDB correctly | Integration test with test collection |
| Retrieval | Correct top-K results for known queries | Integration test with seeded data |
| Response | LLM output matches expected format | Unit test with mocked LLM responses |
| Plagiarism | Correct Neo4j edges created | Integration test with test graph |
| API | Correct status codes, response shapes | Supertest (Express), httpx (FastAPI) |

### Test File Structure

```
tests/
├── backend/
│   ├── unit/
│   │   ├── services/
│   │   │   ├── submission.service.test.ts
│   │   │   └── plagiarism.service.test.ts
│   │   └── utils/
│   │       └── helpers.test.ts
│   ├── integration/
│   │   ├── routes/
│   │   │   ├── submission.routes.test.ts
│   │   │   └── plagiarism.routes.test.ts
│   │   └── repositories/
│   │       └── submission.repo.test.ts
│   └── fixtures/
│       ├── sample-submission.json
│       └── mock-rag-response.json
│
├── rag-service/
│   ├── unit/
│   │   ├── test_text_processing.py
│   │   ├── test_chunking.py
│   │   └── test_prompts.py
│   ├── integration/
│   │   ├── test_indexing_pipeline.py
│   │   ├── test_query_pipeline.py
│   │   └── test_similarity.py
│   └── fixtures/
│       ├── sample_document.txt
│       └── expected_chunks.json
│
└── integration/                     # Cross-service
    ├── test_upload_and_index.ts      # Upload via Express → indexed in Python
    └── test_plagiarism_flow.ts       # Upload → embed → Neo4j → report
```

### Testing Tip: Mock the LLM, Don't Mock Everything Else

```python
# tests/unit/test_query_service.py
from unittest.mock import patch, MagicMock

def test_query_returns_formatted_response():
    """Test that query service formats LLM response correctly."""
    mock_llm_response = "The submission demonstrates understanding of..."
    
    with patch('app.core.querying.get_llm') as mock_llm:
        mock_llm.return_value.complete.return_value = mock_llm_response
        
        service = QueryService()
        result = service.query("SUB-1042", "Evaluate this submission")
        
        assert result.answer == mock_llm_response
        assert len(result.sources) > 0
```

---

## 11. File/Document Storage Strategy

### Where Do Uploaded Files Go?

```
Upload Flow:
    User uploads PDF/DOCX
        │
        ▼
    Express.js saves to temp/ directory (Multer)
        │
        ▼
    Express.js sends file to Python RAG service
        │
        ├──▶ Python extracts text content
        ├──▶ Python chunks + embeds + stores in ChromaDB
        └──▶ Python returns success
        │
        ▼
    Express.js moves file to permanent storage
        │
        ├── Development: local uploads/ folder
        └── Production:  cloud storage (S3/GCS/Azure Blob)
        │
        ▼
    Express.js saves file URL to SQL database
```

### File Storage Structure

```
uploads/                          # gitignored, only for development
├── submissions/
│   ├── 2026/
│   │   └── 03/
│   │       ├── SUB-1042_physics-lab-report.pdf
│   │       ├── SUB-1041_modern-literature-essay.pdf
│   │       └── ...
│   └── ...
└── temp/                         # Multer temp directory (auto-cleaned)
```

### Storage Abstraction

```typescript
// services/storage.service.ts
interface StorageService {
    upload(file: Express.Multer.File, key: string): Promise<string>;  // returns URL
    getUrl(key: string): string;
    delete(key: string): Promise<void>;
}

// Implement LocalStorageService for dev, S3StorageService for prod
// Choose based on config.NODE_ENV
```

---

## 12. Docker & Deployment Structure

### docker-compose.yml (Development)

```yaml
version: '3.8'

services:
  # --- EXPRESS.JS API ---
  backend:
    build:
      context: ./src/backend
      dockerfile: ../../infrastructure/docker/Dockerfile.backend
    ports:
      - "3001:3001"
    env_file: .env
    depends_on:
      - postgres
      - neo4j
      - rag-service
    volumes:
      - ./src/backend:/app          # Hot reload
      - ./uploads:/app/uploads

  # --- PYTHON RAG SERVICE ---
  rag-service:
    build:
      context: ./src/rag-service
      dockerfile: ../../infrastructure/docker/Dockerfile.rag
    ports:
      - "8000:8000"
    env_file: .env
    volumes:
      - ./src/rag-service:/app      # Hot reload
      - ./data/chroma_db:/app/data/chroma_db
      - model-cache:/root/.cache    # Cache BGE-M3 model downloads

  # --- DATABASES ---
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: digichecker
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: digichecker
    volumes:
      - postgres-data:/var/lib/postgresql/data

  neo4j:
    image: neo4j:5-community
    ports:
      - "7474:7474"    # Browser UI
      - "7687:7687"    # Bolt protocol
    environment:
      NEO4J_AUTH: neo4j/devpassword
    volumes:
      - neo4j-data:/data

  # --- FRONTEND (optional — can also run with npm run dev) ---
  frontend:
    build:
      context: ./src/frontend
      dockerfile: ../../infrastructure/docker/Dockerfile.frontend
    ports:
      - "3000:3000"
    env_file: .env
    depends_on:
      - backend

volumes:
  postgres-data:
  neo4j-data:
  model-cache:
```

### Makefile (Common Commands)

```makefile
.PHONY: dev up down logs test seed

# Start everything
dev:
	docker compose up --build

# Start just the databases
db:
	docker compose up postgres neo4j -d

# Run Express.js locally (without Docker)
api:
	cd src/backend && npm run dev

# Run Python RAG service locally
rag:
	cd src/rag-service && uvicorn app.main:app --reload --port 8000

# Run frontend
web:
	cd src/frontend && npm run dev

# Stop everything
down:
	docker compose down

# View logs
logs:
	docker compose logs -f

# Run all tests
test:
	cd src/backend && npm test
	cd src/rag-service && pytest
	cd src/frontend && npm test

# Seed databases with test data
seed:
	cd src/backend && npx prisma db seed
	cd infrastructure/scripts && bash seed-neo4j.sh
```

---

## 13. Naming Conventions & Code Standards

### File Naming

| Language | Convention | Example |
|----------|-----------|---------|
| TypeScript (routes) | `kebab-case.routes.ts` | `submission.routes.ts` |
| TypeScript (components) | `PascalCase.tsx` | `StatCard.tsx` |
| TypeScript (utils) | `camelCase.ts` | `helpers.ts` |
| Python (modules) | `snake_case.py` | `query_service.py` |
| Python (classes) | `PascalCase` in `snake_case.py` | `class QueryService` in `query_service.py` |

### Variable & Function Naming

| Context | Convention | Example |
|---------|-----------|---------|
| Express.js routes | `verb + noun` | `getSubmissions`, `createSubmission` |
| Python services | `verb + noun` | `index_document`, `find_similar` |
| React components | `PascalCase` | `SubmissionsTable`, `StatCard` |
| Database columns | `snake_case` | `submission_id`, `created_at` |
| Environment vars | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `BGE_M3_MODEL` |
| API endpoints | `/kebab-case/plural-nouns` | `/api/submissions`, `/api/plagiarism-reports` |

### API Response Format (consistent across all endpoints)

```typescript
// Success
{
    "data": { ... },
    "meta": { "page": 1, "total": 67 }  // only for paginated responses
}

// Error
{
    "error": "Submission not found",
    "statusCode": 404
}
```

---

## 14. Common RAG Project Anti-Patterns

### 1. God Service

**Bad:** One service file that handles loading, chunking, embedding, indexing, querying, and response formatting.

```python
# ❌ Bad: 500-line do_everything.py
class RAGService:
    def process(self, file):
        text = extract_text(file)           # loading
        chunks = chunk_text(text)            # chunking
        embeddings = embed(chunks)           # embedding
        store(embeddings)                    # storing
        results = query(embeddings)          # querying
        return format_response(results)      # formatting
```

**Good:** Split by responsibility — each step is its own module.

### 2. Hardcoded Prompts Inside Service Logic

**Bad:** Prompt strings buried in service methods, mixed with business logic.

**Good:** All prompts in `core/prompts.py`, imported by services. Easy to iterate on prompts without touching logic.

### 3. No Abstraction Over Vector Store

**Bad:** ChromaDB API calls scattered throughout the codebase.

**Good:** A `stores/chroma_store.py` wrapper that all services import. If you switch to Pinecone later, you change one file.

### 4. Frontend Knows About Backend Architecture

**Bad:** Frontend has `/api/rag-service/query` URLs or references to Neo4j/ChromaDB.

**Good:** Frontend only knows about Express.js endpoints like `/api/submissions/:id/analyze`. It doesn't know or care that Python/ChromaDB/Neo4j exist behind the scenes.

### 5. No Health Checks

**Bad:** Express.js calls Python service, Python is down, entire app hangs.

**Good:** Each service has a `/health` endpoint. Express.js checks Python health before forwarding requests. docker-compose uses `healthcheck:` directives.

### 6. Embedding Model Loaded Per Request

**Bad:** `model = BGEM3FlagModel('BAAI/bge-m3')` called every time a request comes in (takes ~30 seconds each time).

**Good:** Load once at startup via singleton/`lru_cache`, reuse for all requests.

### 7. No Rate Limiting on LLM Calls

**Bad:** Every user can trigger unlimited LLM calls, burning through API credits.

**Good:** Rate limit at Express.js middleware level. Queue long-running analysis jobs instead of blocking HTTP requests.

---

## 15. Checklist: Before You Start Coding

Use this checklist to validate your project setup before writing feature code:

### Project Setup
- [ ] Monorepo structure created with `src/backend`, `src/frontend`, `src/rag-service`
- [ ] Each service has its own dependency file (`package.json` / `pyproject.toml`)
- [ ] `.env.example` created with all required variables documented
- [ ] `.gitignore` covers `node_modules/`, `__pycache__/`, `.env`, `chroma_db/`, `uploads/`
- [ ] `docker-compose.yml` runs all services + databases
- [ ] `Makefile` or equivalent with `dev`, `test`, `seed` commands

### Backend (Express.js)
- [ ] Layer separation: routes → controllers → services → repositories
- [ ] Prisma schema defined with all tables
- [ ] Error middleware catches and formats all errors
- [ ] Request validation (Zod) on all POST/PUT endpoints
- [ ] Auth middleware protects all non-public routes
- [ ] Logger setup (Pino) with structured JSON output
- [ ] Config validation at startup (fail fast on missing env vars)

### RAG Service (Python)
- [ ] FastAPI app with `/index`, `/query`, `/similar`, `/health` endpoints
- [ ] BGE-M3 loaded as singleton (not per-request)
- [ ] ChromaDB client as singleton with persistent storage
- [ ] Prompt templates in dedicated `core/prompts.py`
- [ ] Chunking strategy chosen and configurable
- [ ] Pydantic models for all request/response schemas

### Databases
- [ ] SQL schema covers: users, courses, submissions, plagiarism_reports
- [ ] ChromaDB collection created with cosine distance metric
- [ ] Neo4j constraints: `CREATE CONSTRAINT ON (s:Submission) ASSERT s.submission_id IS UNIQUE`
- [ ] Cross-DB references documented (SQL ↔ ChromaDB ↔ Neo4j IDs)
- [ ] Seed script for test data

### Frontend
- [ ] API client layer (`services/`) abstracts all HTTP calls
- [ ] Types match API response shapes
- [ ] Components don't contain API logic
- [ ] Environment variable for API base URL (`NEXT_PUBLIC_API_URL`)

### Testing
- [ ] At least one test file per service module
- [ ] Integration test for: upload → index → query flow
- [ ] Integration test for: upload → plagiarism check flow
- [ ] LLM calls mocked in unit tests (don't burn credits in CI)

---

## Quick Reference Card

```
┌────────────────────────────────────────────────────────┐
│              PROJECT STRUCTURE CHEAT SHEET              │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Express.js layers:                                    │
│    Route → Controller → Service → Repository           │
│                                                        │
│  Python layers:                                        │
│    API Route → Service → Core → Store                  │
│                                                        │
│  Database ownership:                                   │
│    SQL = metadata | ChromaDB = vectors | Neo4j = graph │
│                                                        │
│  Config: validate at startup, fail fast                │
│  Prompts: separate file, never inline                  │
│  Models: singleton, load once                          │
│  Errors: centralized middleware                        │
│  Logging: structured JSON, same format both services   │
│  Files: temp → process → permanent storage             │
│  Frontend: knows NOTHING about Python/ChromaDB/Neo4j   │
│                                                        │
└────────────────────────────────────────────────────────┘
```
