# LlamaIndex RAG Stack — Implementation Study Notes

**Date:** 9 March 2026  
**Stack:** LlamaIndex + BGE-M3 + ChromaDB + Express.js + SQL + Neo4j  
**Paper:** Chen et al., *"M3-Embedding: Multi-Linguality, Multi-Functionality, Multi-Granularity Text Embeddings Through Self-Knowledge Distillation"* (arXiv:2402.03216, 2024)

---

## Table of Contents

1. [RAG Overview & Flow](#1-rag-overview--flow)
2. [LlamaIndex Pipeline Stages](#2-llamaindex-pipeline-stages)
3. [BGE-M3 Embedding Model](#3-bge-m3-embedding-model)
4. [ChromaDB Vector Database](#4-chromadb-vector-database)
5. [Express.js REST API Backend](#5-expressjs-rest-api-backend)
6. [SQL Database Layer](#6-sql-database-layer)
7. [Neo4j Graph Database — Plagiarism Detection](#7-neo4j-graph-database--plagiarism-detection)
8. [Full System Architecture Diagram](#8-full-system-architecture-diagram)
9. [References](#9-references)

---

## 1. RAG Overview & Flow

**Retrieval-Augmented Generation (RAG)** solves the core limitation of LLMs — they aren't trained on *your* data. RAG adds your data to the data LLMs already have access to at query time rather than re-training the model.

### High-Level RAG Flow

```
User Query
    │
    ▼
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│  Embedding  │───▶│  Vector DB   │───▶│  Top-K Retrieval│
│  (BGE-M3)   │    │  (ChromaDB)  │    │  (Similar Docs) │
└─────────────┘    └──────────────┘    └────────┬────────┘
                                                │
                                                ▼
                                    ┌───────────────────┐
                                    │   LLM Prompt      │
                                    │   Query + Context  │
                                    │   + System Prompt  │
                                    └────────┬──────────┘
                                             │
                                             ▼
                                    ┌───────────────────┐
                                    │   Generated       │
                                    │   Response        │
                                    └───────────────────┘
```

### Why RAG Over Fine-Tuning?

| Aspect | RAG | Fine-Tuning |
|--------|-----|-------------|
| Cost | Low — no retraining | High — GPU hours |
| Data freshness | Instant update — just re-index | Requires retraining |
| Hallucination | Reduced — grounded in retrieved docs | Can still hallucinate |
| Control | Easy to audit retrieved context | Black box |
| Best for | Dynamic, document-heavy use cases | Style/behavior changes |

---

## 2. LlamaIndex Pipeline Stages

LlamaIndex defines **5 key stages** in a RAG pipeline:

### Stage 1: Loading

Getting data from source into the pipeline.

- **Documents** — container around any data source (PDF, text, API output, database row)
- **Connectors (Readers)** — ingest data from different formats into Documents
- LlamaHub provides hundreds of connectors (PDF, CSV, Notion, SQL, etc.)

```python
from llama_index.core import SimpleDirectoryReader

# Load all documents from a directory
documents = SimpleDirectoryReader("./data/submissions").load_data()
```

### Stage 2: Indexing

Creating data structures that enable efficient querying.

- **Nodes** — atomic unit of data in LlamaIndex (a "chunk" of a Document). Nodes have metadata linking them to parent documents and sibling nodes.
- **Embeddings** — numerical representations of text meaning (BGE-M3 generates these)
- **Vector Store Index** — the most common index type, stores embeddings in a vector DB

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.node_parser import SentenceSplitter

# Parse documents into nodes (chunks)
parser = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = parser.get_nodes_from_documents(documents)

# Build index (embeds + stores)
index = VectorStoreIndex(nodes)
```

**Chunking Strategy Matters:**

| Strategy | Chunk Size | Overlap | Use Case |
|----------|-----------|---------|----------|
| Sentence | 256–512 | 20–50 | Short docs, precise retrieval |
| Paragraph | 512–1024 | 50–100 | Medium docs, context-heavy |
| Document | 1024–2048 | 100–200 | Long docs, broad context |

### Stage 3: Storing

Persisting indexed data to avoid re-indexing.

```python
from llama_index.vector_stores.chroma import ChromaVectorStore
import chromadb

# Persistent ChromaDB client
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("submissions")

vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
```

### Stage 4: Querying

Retrieving relevant context and generating responses.

Key components:
- **Retrievers** — define *how* to find relevant context (top-k similarity, keyword, hybrid)
- **Routers** — select which retriever to use for a given query
- **Node Postprocessors** — re-rank, filter, or transform retrieved nodes
- **Response Synthesizers** — generate final response from LLM using query + retrieved context

```python
# Create query engine from index
query_engine = index.as_query_engine(
    similarity_top_k=5,    # retrieve top 5 chunks
    response_mode="compact" # compact context into fewer LLM calls
)

response = query_engine.query("What are the key findings in submission SUB-1042?")
```

**Response Modes:**

| Mode | Description | Best For |
|------|------------|---------|
| `refine` | Iterate through each chunk, refining answer | Detailed answers |
| `compact` | Stuff as many chunks as possible per LLM call | Speed + cost savings |
| `tree_summarize` | Build tree of summaries, bottom-up | Long documents |
| `simple_summarize` | Truncate all chunks to fit one prompt | Quick summaries |

### Stage 5: Evaluation

Measuring RAG pipeline quality.

Key metrics:
- **Faithfulness** — Is the response grounded in the retrieved context?
- **Relevancy** — Are the retrieved documents relevant to the query?
- **Correctness** — Is the response factually accurate?
- **Latency** — How fast is the end-to-end retrieval + generation?

---

## 3. BGE-M3 Embedding Model

### What Makes BGE-M3 Special?

BGE-M3 stands for **Multi-Linguality, Multi-Functionality, Multi-Granularity**:

| "M" | Capability | Details |
|-----|-----------|---------|
| **Multi-Lingual** | 100+ languages | Built on XLM-RoBERTa backbone |
| **Multi-Functional** | 3 retrieval modes in one model | Dense, Sparse, ColBERT (multi-vector) |
| **Multi-Granular** | Short to long text | Up to 8192 tokens (vs 512 for most models) |

### Model Specs

| Property | Value |
|----------|-------|
| Base Model | XLM-RoBERTa (extended) |
| Embedding Dimension | 1024 |
| Max Sequence Length | 8192 tokens |
| Training | Self-Knowledge Distillation |
| License | MIT |
| Paper | arXiv:2402.03216 |

### Three Retrieval Modes

#### 1. Dense Retrieval
Maps entire text → single 1024-dim vector. Standard semantic similarity.

```python
from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)

# Generate dense embeddings
embeddings = model.encode(
    ["Student submission text here..."],
    batch_size=12,
    max_length=8192
)['dense_vecs']  # shape: (1, 1024)
```

#### 2. Sparse Retrieval (Lexical Matching)
Generates token-level weights — similar to BM25 but learned. Great for exact keyword matching.

```python
output = model.encode(
    ["Student submission text here..."],
    return_dense=True,
    return_sparse=True,
    return_colbert_vecs=False
)

# See token weights
token_weights = model.convert_id_to_token(output['lexical_weights'])
# {'Student': 0.12, 'submission': 0.25, 'text': 0.08, ...}
```

#### 3. Multi-Vector Retrieval (ColBERT)
Uses multiple vectors (one per token) for fine-grained matching. Highest accuracy but most expensive.

```python
output = model.encode(
    ["Student submission text here..."],
    return_colbert_vecs=True
)
# output['colbert_vecs'] → (1, seq_len, 1024)
```

### Hybrid Scoring (Recommended for RAG)

Combine all three modes for best results:

```python
scores = model.compute_score(
    sentence_pairs,
    weights_for_different_modes=[0.4, 0.2, 0.4]
    # 40% dense + 20% sparse + 40% colbert
)
```

### Integrating BGE-M3 with LlamaIndex

```python
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Use BGE-M3 as the embedding model in LlamaIndex
embed_model = HuggingFaceEmbedding(
    model_name="BAAI/bge-m3",
    max_length=8192,
    embed_batch_size=10
)

# Pass to the Settings or directly to index
from llama_index.core import Settings
Settings.embed_model = embed_model
```

### Self-Knowledge Distillation (Training Innovation)

BGE-M3's training uses a clever technique:
1. Compute scores from all three retrieval modes (dense, sparse, ColBERT)
2. Combine them into a unified "teacher" signal
3. Use this combined signal to train each individual mode
4. Result: each mode benefits from the strengths of the others

```
Dense Score ──┐
              ├──▶ Combined Teacher Signal ──▶ Train each mode individually
Sparse Score ─┤
              │
ColBERT Score ┘
```

---

## 4. ChromaDB Vector Database

### What is ChromaDB?

ChromaDB is an open-source, AI-native embedding database. It handles storing, indexing, and querying embeddings.

**Key Features:**
- Document + embedding + metadata storage
- Dense, sparse, and hybrid search
- Full-text and regex search
- Metadata filtering at query time
- Multi-modal support (text, images, audio)
- Apache 2.0 license

### Core Concepts

```
ChromaDB
  └── Collection (like a "table")
        ├── Documents (raw text)
        ├── Embeddings (vectors)
        ├── Metadata (key-value pairs)
        └── IDs (unique string identifiers)
```

### Setup & Basic Usage

```python
import chromadb

# Option 1: In-memory (for dev/testing)
client = chromadb.Client()

# Option 2: Persistent (for production — data survives restarts)
client = chromadb.PersistentClient(path="./chroma_db")

# Option 3: Client-server mode (remote)
client = chromadb.HttpClient(host="localhost", port=8000)
```

### Collection Operations

```python
# Create or get a collection
collection = client.get_or_create_collection(
    name="submissions",
    metadata={"hnsw:space": "cosine"}  # distance metric
)

# Add documents (ChromaDB auto-embeds if no embeddings provided)
collection.upsert(
    ids=["sub-1042", "sub-1041", "sub-1040"],
    documents=[
        "Lab Report — Physics 201: Analysis of projectile motion...",
        "Essay — Modern Literature: The role of symbolism in...",
        "Problem Set 5 — Calculus II: Integration by parts..."
    ],
    metadatas=[
        {"student": "Alice", "course": "PHYS201", "date": "2026-03-08"},
        {"student": "Bob", "course": "LIT301", "date": "2026-03-07"},
        {"student": "Charlie", "course": "MATH202", "date": "2026-03-06"}
    ]
)

# Query by semantic similarity
results = collection.query(
    query_texts=["physics experiment analysis"],
    n_results=3,
    where={"course": "PHYS201"}  # metadata filter
)
```

### Integrating ChromaDB with LlamaIndex

```python
import chromadb
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import StorageContext, VectorStoreIndex

# Setup ChromaDB
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.get_or_create_collection("submissions")

# Create LlamaIndex vector store backed by ChromaDB
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Build index — embeddings go into ChromaDB
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
    embed_model=embed_model  # BGE-M3
)

# Query — retrieves from ChromaDB
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("Summarize the physics lab findings")
```

### Distance Metrics

| Metric | Use When | ChromaDB Key |
|--------|----------|-------------|
| Cosine | Normalized embeddings (BGE-M3 default) | `cosine` |
| L2 (Euclidean) | Magnitude matters | `l2` |
| Inner Product | Pre-normalized, max inner product search | `ip` |

---

## 5. Express.js REST API Backend

### Role in the Architecture

Express.js serves as the **REST API gateway** between:
- Frontend (Next.js) → Express.js → LlamaIndex Python service
- Frontend → Express.js → SQL database (metadata, users, submissions)
- Frontend → Express.js → Neo4j (plagiarism checks)

### Project Structure

```
backend/
├── src/
│   ├── index.ts                 # Entry point
│   ├── config/
│   │   ├── database.ts          # SQL connection config
│   │   └── neo4j.ts             # Neo4j driver config
│   ├── routes/
│   │   ├── submissions.ts       # CRUD for submissions
│   │   ├── analysis.ts          # RAG query endpoints
│   │   └── plagiarism.ts        # Plagiarism check endpoints
│   ├── controllers/
│   │   ├── submissionController.ts
│   │   ├── analysisController.ts
│   │   └── plagiarismController.ts
│   ├── models/
│   │   ├── Submission.ts
│   │   └── User.ts
│   ├── services/
│   │   ├── ragService.ts        # Calls Python LlamaIndex service
│   │   ├── plagiarismService.ts # Calls Neo4j
│   │   └── embeddingService.ts  # Calls BGE-M3
│   └── middleware/
│       ├── auth.ts
│       └── errorHandler.ts
├── package.json
└── tsconfig.json
```

### Key API Endpoints

```typescript
// routes/submissions.ts
import { Router } from 'express';

const router = Router();

// CRUD
router.post('/submissions', submitFile);         // Upload + process
router.get('/submissions', getAll);              // List all
router.get('/submissions/:id', getById);         // Get one
router.delete('/submissions/:id', deleteById);   // Remove

// Analysis
router.post('/submissions/:id/analyze', analyzeSubmission);  // RAG analysis
router.get('/submissions/:id/grade', getGrade);              // Get grade

// Plagiarism
router.post('/submissions/:id/check-plagiarism', checkPlagiarism);
router.get('/submissions/:id/plagiarism-report', getPlagiarismReport);
```

### Communication with Python RAG Service

Express.js calls the Python LlamaIndex service via HTTP:

```typescript
// services/ragService.ts
const RAG_SERVICE_URL = process.env.RAG_SERVICE_URL || 'http://localhost:8000';

export async function analyzeDocument(documentId: string, query: string) {
  const response = await fetch(`${RAG_SERVICE_URL}/query`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ document_id: documentId, query })
  });
  return response.json();
}

export async function indexDocument(documentId: string, content: string) {
  const response = await fetch(`${RAG_SERVICE_URL}/index`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ document_id: documentId, content })
  });
  return response.json();
}
```

---

## 6. SQL Database Layer

### Purpose

SQL stores **structured metadata** — things that don't belong in a vector DB:
- User accounts & authentication
- Submission records (id, name, date, status, grade, file path)
- Course/class information
- Grading rubrics
- Audit logs

### Schema Design

```sql
-- Users
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(255) NOT NULL,
    role        VARCHAR(50) DEFAULT 'student',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Courses  
CREATE TABLE courses (
    id          SERIAL PRIMARY KEY,
    code        VARCHAR(20) UNIQUE NOT NULL,
    name        VARCHAR(255) NOT NULL,
    instructor_id INTEGER REFERENCES users(id)
);

-- Submissions
CREATE TABLE submissions (
    id              SERIAL PRIMARY KEY,
    submission_id   VARCHAR(20) UNIQUE NOT NULL,  -- e.g. "SUB-1042"
    user_id         INTEGER REFERENCES users(id),
    course_id       INTEGER REFERENCES courses(id),
    name            VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    status          VARCHAR(20) DEFAULT 'Pending', -- Pending | Checked
    grade           VARCHAR(10),
    chroma_doc_id   VARCHAR(255),  -- reference to ChromaDB document
    neo4j_node_id   VARCHAR(255),  -- reference to Neo4j node
    submitted_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    checked_at      TIMESTAMP
);

-- Plagiarism Reports
CREATE TABLE plagiarism_reports (
    id              SERIAL PRIMARY KEY,
    submission_id   INTEGER REFERENCES submissions(id),
    similarity_score DECIMAL(5,2),  -- 0.00 to 100.00
    matched_submissions TEXT[],     -- array of matching SUB-IDs
    report_data     JSONB,         -- detailed graph match data
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Data Flow: SQL ↔ ChromaDB

```
New Submission Upload
    │
    ├──▶ SQL: INSERT submission record (metadata)
    │
    ├──▶ ChromaDB: Store document embeddings (via BGE-M3)
    │         └── Store chroma_doc_id back to SQL
    │
    └──▶ Neo4j: Create document node + relationships
              └── Store neo4j_node_id back to SQL
```

---

## 7. Neo4j Graph Database — Plagiarism Detection

### Why a Graph Database for Plagiarism?

Plagiarism relationships are inherently **graph-shaped**:
- A document can be similar to multiple other documents
- Similarity is bidirectional and weighted
- You need to find clusters of similar submissions
- Path traversal reveals chains of plagiarism (A copied B, B copied C)
- Traditional SQL struggles with recursive relationship queries

### Graph Schema

```
(:Student {id, name, email})
    -[:SUBMITTED]->
(:Submission {id, submission_id, name, date, course})
    -[:HAS_SECTION]->
(:Section {id, type, content_hash, embedding_id})

(:Submission)-[:SIMILAR_TO {score, method}]->(:Submission)
(:Section)-[:MATCHES {score, overlap_pct}]->(:Section)
```

### Cypher Queries for Plagiarism Detection

#### 1. Create Document Nodes

```cypher
// Create a submission node
CREATE (s:Submission {
    submission_id: 'SUB-1042',
    name: 'Lab Report — Physics 201',
    course: 'PHYS201',
    student_id: 'user_1',
    submitted_at: datetime('2026-03-08T10:00:00'),
    content_hash: 'sha256_abc123'
})
RETURN s;

// Create sections (chunks of the document)
CREATE (sec:Section {
    id: 'SUB-1042-sec-1',
    type: 'introduction',
    content_hash: 'sha256_intro_hash',
    embedding_id: 'chroma_vec_001'
})
RETURN sec;

// Link submission to its sections
MATCH (s:Submission {submission_id: 'SUB-1042'})
MATCH (sec:Section {id: 'SUB-1042-sec-1'})
CREATE (s)-[:HAS_SECTION]->(sec);
```

#### 2. Create Similarity Relationships

After computing cosine similarity between embeddings (via ChromaDB/BGE-M3):

```cypher
// Create similarity edge between two submissions
MATCH (a:Submission {submission_id: 'SUB-1042'})
MATCH (b:Submission {submission_id: 'SUB-1038'})
CREATE (a)-[:SIMILAR_TO {
    score: 0.87,
    method: 'cosine_bge_m3',
    checked_at: datetime()
}]->(b);

// Create section-level match
MATCH (sa:Section {id: 'SUB-1042-sec-2'})
MATCH (sb:Section {id: 'SUB-1038-sec-3'})
CREATE (sa)-[:MATCHES {
    score: 0.93,
    overlap_pct: 78.5,
    method: 'dense+sparse_hybrid'
}]->(sb);
```

#### 3. Detect Plagiarism

```cypher
// Find all submissions similar to a given one (threshold > 0.8)
MATCH (s:Submission {submission_id: 'SUB-1042'})-[r:SIMILAR_TO]-(other:Submission)
WHERE r.score > 0.8
RETURN other.submission_id AS match,
       other.name AS title,
       r.score AS similarity,
       r.method AS method
ORDER BY r.score DESC;
```

```cypher
// Find plagiarism clusters — groups of highly similar submissions
MATCH (a:Submission)-[r:SIMILAR_TO]->(b:Submission)
WHERE r.score > 0.85
WITH a, COLLECT({target: b.submission_id, score: r.score}) AS matches
WHERE SIZE(matches) >= 2
RETURN a.submission_id AS source,
       a.name AS title,
       matches
ORDER BY SIZE(matches) DESC;
```

```cypher
// Detect transitive plagiarism chains (A→B→C)
MATCH path = (start:Submission)-[:SIMILAR_TO*2..4]-(end:Submission)
WHERE ALL(r IN relationships(path) WHERE r.score > 0.75)
  AND start <> end
RETURN [node IN nodes(path) | node.submission_id] AS chain,
       [r IN relationships(path) | r.score] AS scores;
```

```cypher
// Section-level deep comparison
MATCH (s:Submission {submission_id: 'SUB-1042'})-[:HAS_SECTION]->(sec:Section)
      -[m:MATCHES]->(other_sec:Section)<-[:HAS_SECTION]-(other:Submission)
WHERE m.score > 0.9
RETURN other.submission_id AS matched_submission,
       sec.type AS source_section,
       other_sec.type AS matched_section,
       m.score AS similarity,
       m.overlap_pct AS overlap
ORDER BY m.score DESC;
```

#### 4. Per-Course Analysis

```cypher
// Plagiarism heatmap for a course
MATCH (a:Submission)-[r:SIMILAR_TO]->(b:Submission)
WHERE a.course = 'PHYS201' AND b.course = 'PHYS201'
  AND a.submission_id < b.submission_id  // avoid duplicates
RETURN a.submission_id, b.submission_id, r.score
ORDER BY r.score DESC;
```

### Plagiarism Detection Pipeline

```
New Submission
    │
    ▼
┌──────────────────┐
│ 1. Chunk document │
│    into sections  │
└────────┬─────────┘
         │
         ▼
┌──────────────────────┐
│ 2. Embed sections    │
│    with BGE-M3       │
│    (dense + sparse)  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────────┐
│ 3. Store in ChromaDB     │
│    Query top-K similar   │
│    sections from other   │
│    submissions           │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ 4. Create Neo4j nodes    │
│    Submission + Sections │
│    + SIMILAR_TO edges    │
│    with similarity scores│
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ 5. Run Cypher queries    │
│    to detect clusters,   │
│    chains, and flag      │
│    suspicious matches    │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ 6. Store report in SQL   │
│    Update submission     │
│    status                │
└──────────────────────────┘
```

### Neo4j Node.js Driver Setup

```typescript
// config/neo4j.ts
import neo4j from 'neo4j-driver';

const driver = neo4j.driver(
    process.env.NEO4J_URI || 'bolt://localhost:7687',
    neo4j.auth.basic(
        process.env.NEO4J_USER || 'neo4j',
        process.env.NEO4J_PASSWORD || 'password'
    )
);

export async function runQuery(cypher: string, params: Record<string, unknown> = {}) {
    const session = driver.session();
    try {
        const result = await session.run(cypher, params);
        return result.records.map(record => record.toObject());
    } finally {
        await session.close();
    }
}

export default driver;
```

---

## 8. Full System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Next.js)                         │
│  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Dashboard │  │ Submissions  │  │ Reports  │  │ Upload File   │  │
│  └──────────┘  └──────────────┘  └──────────┘  └───────────────┘  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTP (REST API)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BACKEND (Express.js)                          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ Auth        │  │ Submission   │  │ Plagiarism                │  │
│  │ Middleware  │  │ Controller   │  │ Controller                │  │
│  └─────────────┘  └──────┬───────┘  └───────────┬───────────────┘  │
│                          │                      │                  │
│         ┌────────────────┼──────────────────────┤                  │
│         │                │                      │                  │
│         ▼                ▼                      ▼                  │
│  ┌────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ SQL DB     │  │ RAG Service  │  │ Neo4j Service             │  │
│  │ (Metadata) │  │ (HTTP call)  │  │ (Plagiarism)              │  │
│  └────────────┘  └──────┬───────┘  └───────────┬───────────────┘  │
└─────────────────────────┼──────────────────────┼───────────────────┘
                          │                      │
              ┌───────────▼────────┐    ┌────────▼──────────┐
              │  PYTHON SERVICE    │    │     NEO4J DB      │
              │  (LlamaIndex)      │    │  ┌─────────────┐  │
              │                    │    │  │ :Submission  │  │
              │  ┌──────────────┐  │    │  │  -[:SIMILAR] │  │
              │  │  BGE-M3      │  │    │  │ :Section    │  │
              │  │  Embeddings  │  │    │  │  -[:MATCHES] │  │
              │  └──────┬───────┘  │    │  └─────────────┘  │
              │         │          │    └───────────────────┘
              │         ▼          │
              │  ┌──────────────┐  │
              │  │  ChromaDB    │  │
              │  │  (Vectors)   │  │
              │  └──────────────┘  │
              └────────────────────┘
```

### Data Flow Summary

| Action | Flow |
|--------|------|
| Upload submission | Frontend → Express → SQL (metadata) → Python (embed + index in ChromaDB) → Neo4j (create node) |
| Analyze/grade submission | Frontend → Express → Python (LlamaIndex RAG query against ChromaDB) → Response |
| Check plagiarism | Frontend → Express → Python (embed new doc, query ChromaDB for similar) → Neo4j (create edges, run Cypher) → Report |
| View dashboard | Frontend → Express → SQL (stats, recent submissions) → Frontend |

---

## 9. References

### Papers
1. Chen et al., "M3-Embedding: Multi-Linguality, Multi-Functionality, Multi-Granularity Text Embeddings Through Self-Knowledge Distillation" — arXiv:2402.03216 (2024)
2. Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" — arXiv:2005.11401 (2020) — *foundational RAG paper*
3. Khattab & Zaharia, "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT" — arXiv:2004.12832 (2020)

### Documentation
4. LlamaIndex RAG Documentation — https://developers.llamaindex.ai/python/framework/understanding/rag/
5. LlamaIndex High-Level Concepts — https://developers.llamaindex.ai/python/framework/getting_started/concepts/
6. BGE-M3 Model Card — https://huggingface.co/BAAI/bge-m3
7. ChromaDB Documentation — https://docs.trychroma.com/docs/overview/introduction
8. Neo4j Cypher Manual — https://neo4j.com/docs/cypher-manual/current/introduction/
9. Neo4j Graph Data Science Library — https://neo4j.com/docs/graph-data-science/current/

### Tools & Libraries
10. LlamaIndex Python — https://github.com/run-llama/llama_index
11. FlagEmbedding (BGE-M3) — https://github.com/FlagOpen/FlagEmbedding
12. ChromaDB — https://github.com/chroma-core/chroma
13. Neo4j JavaScript Driver — https://github.com/neo4j/neo4j-javascript-driver
14. Express.js — https://expressjs.com/
