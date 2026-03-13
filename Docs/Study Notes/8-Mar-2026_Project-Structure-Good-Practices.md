# Project Structure Best Practices for AI Web-Based Projects

**Date:** 8 March 2026  
**Topic:** How to organize and structure an AI-powered web application project  
**Context:** Applied to the "Real DigiChecker" project (Frontend + Backend + AI/ML)

---

## Table of Contents

1. [Why Project Structure Matters](#1-why-project-structure-matters)
2. [High-Level Architecture: Monorepo vs Polyrepo](#2-high-level-architecture-monorepo-vs-polyrepo)
3. [Recommended Top-Level Folder Structure](#3-recommended-top-level-folder-structure)
4. [Backend Structure (Python / FastAPI)](#4-backend-structure-python--fastapi)
5. [Frontend Structure (React / Next.js)](#5-frontend-structure-react--nextjs)
6. [AI/ML Module Structure](#6-aiml-module-structure)
7. [The Twelve-Factor App Principles](#7-the-twelve-factor-app-principles)
8. [Separation of Concerns](#8-separation-of-concerns)
9. [Configuration Management](#9-configuration-management)
10. [Testing Strategy & Structure](#10-testing-strategy--structure)
11. [Documentation Practices](#11-documentation-practices)
12. [DevOps & Deployment Structure](#12-devops--deployment-structure)
13. [Common Anti-Patterns to Avoid](#13-common-anti-patterns-to-avoid)
14. [Applied Example: Real DigiChecker](#14-applied-example-real-digichecker)
15. [References & Further Reading](#15-references--further-reading)

---

## 1. Why Project Structure Matters

> "Repository structure is a crucial part of your project's architecture."  
> — *The Hitchhiker's Guide to Python*

- **First Impressions:** When someone lands on your repository, they see a project name, description, and a bunch of files. If it's a nested mess, they leave before reading the README.
- **Maintainability:** You and your team will spend countless hours navigating this codebase. A clean layout reduces cognitive load.
- **Onboarding:** New developers can understand the project faster when files are in predictable locations.
- **Scalability:** Good structure lets you add features without reorganizing everything.
- **Separation of Concerns:** Each folder/module should have a clear, single responsibility.

**Key Principle:**
> "Dress for the job you want, not the job you have." — Structure your project as if it's already production-grade.

---

## 2. High-Level Architecture: Monorepo vs Polyrepo

### Monorepo (Recommended for most AI web projects)
All code lives in one repository with clear folder boundaries.

| Pros | Cons |
|------|------|
| Easier to share code between frontend, backend, and ML | Can become large over time |
| Atomic commits across the full stack | Requires discipline in folder organization |
| Single CI/CD pipeline configuration | Build times can increase |
| Easier dependency management | Access control is all-or-nothing |

### Polyrepo
Each component (frontend, backend, ML) has its own repository.

| Pros | Cons |
|------|------|
| Clear ownership boundaries | Cross-repo changes are harder to coordinate |
| Independent deployment | Code sharing requires publishing packages |
| Smaller repos, faster clones | Multiple CI/CD pipelines to maintain |

**Recommendation for AI Web Projects:**  
Use a **monorepo** with clear internal boundaries. AI web apps typically have tight coupling between the API layer and the ML models, making a monorepo more practical.

---

## 3. Recommended Top-Level Folder Structure

```
project-root/
├── .github/                    # GitHub Actions, issue templates, PR templates
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── ml-pipeline.yml
│
├── docs/                       # Project documentation
│   ├── architecture/           # Architecture Decision Records (ADRs)
│   ├── api/                    # API documentation
│   └── study-notes/            # Research & learning notes
│
├── src/                        # All source code
│   ├── backend/                # Backend API service
│   ├── frontend/               # Frontend web application
│   └── ml/                     # Machine Learning / AI modules
│
├── tests/                      # All test code (mirrors src/ structure)
│   ├── backend/
│   ├── frontend/
│   └── ml/
│
├── scripts/                    # Utility scripts (setup, data download, etc.)
├── data/                       # Data files (or .gitkeep if data is external)
│   ├── raw/                    # Unprocessed data
│   ├── processed/              # Cleaned/transformed data
│   └── external/               # Third-party data
│
├── models/                     # Trained model artifacts (.gitkeep, actual models in cloud)
├── notebooks/                  # Jupyter notebooks for exploration/prototyping
│
├── infrastructure/             # IaC (Terraform, Docker, K8s manifests)
│   ├── docker/
│   │   ├── Dockerfile.backend
│   │   ├── Dockerfile.frontend
│   │   └── Dockerfile.ml
│   └── k8s/
│
├── .env.example                # Template for environment variables (NEVER commit .env)
├── .gitignore
├── docker-compose.yml          # Local development orchestration
├── Makefile                    # Common commands (build, test, lint, run)
├── pyproject.toml              # Python project config & dependencies
├── package.json                # Node.js/Frontend dependencies (if at root)
├── README.md                   # Project overview, setup instructions
├── LICENSE
└── CONTRIBUTING.md             # Contribution guidelines
```

### Key Rules:
1. **`src/` is sacred** — only source code goes here, organized by domain
2. **`tests/` mirrors `src/`** — every module in `src/` has a corresponding test module
3. **`docs/` is always present** — even if it starts with just a README
4. **`data/` and `models/`** should use `.gitkeep` files; actual data/models live in cloud storage
5. **Configuration files** (`.env`, `docker-compose.yml`, `Makefile`) stay at root

---

## 4. Backend Structure (Python / FastAPI)

Based on FastAPI's official recommended structure for bigger applications:

```
src/backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app entry point
│   ├── dependencies.py         # Shared dependencies (auth, DB sessions)
│   ├── config.py               # App configuration (from env vars)
│   │
│   ├── api/                    # API layer
│   │   ├── __init__.py
│   │   ├── v1/                 # API versioning
│   │   │   ├── __init__.py
│   │   │   ├── router.py       # Aggregates all v1 routers
│   │   │   └── endpoints/
│   │   │       ├── __init__.py
│   │   │       ├── auth.py
│   │   │       ├── users.py
│   │   │       ├── documents.py
│   │   │       └── predictions.py  # AI inference endpoints
│   │   └── v2/                 # Future API version
│   │
│   ├── core/                   # Core business logic
│   │   ├── __init__.py
│   │   ├── security.py         # JWT, hashing, auth logic
│   │   └── exceptions.py       # Custom exception classes
│   │
│   ├── models/                 # Database models (SQLAlchemy / ORM)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── document.py
│   │
│   ├── schemas/                # Pydantic schemas (request/response validation)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── document.py
│   │
│   ├── services/               # Business logic services
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   ├── document_service.py
│   │   └── ml_service.py       # Bridge to ML module
│   │
│   ├── repositories/           # Data access layer (DB queries)
│   │   ├── __init__.py
│   │   ├── user_repo.py
│   │   └── document_repo.py
│   │
│   └── utils/                  # Utility functions
│       ├── __init__.py
│       └── helpers.py
│
├── alembic/                    # Database migrations
│   ├── versions/
│   └── env.py
│
├── requirements.txt            # or pyproject.toml
└── pyproject.toml
```

### Key Principles:
- **API Versioning:** Use `/api/v1/`, `/api/v2/` prefixes for forward compatibility
- **Repository Pattern:** Separate data access (`repositories/`) from business logic (`services/`)
- **Schemas ≠ Models:** Pydantic schemas handle validation; ORM models handle persistence
- **FastAPI Routers:** Use `APIRouter` to organize endpoints by domain (users, documents, etc.)
- **Dependency Injection:** Use FastAPI's `Depends()` for shared logic (auth, DB sessions)

---

## 5. Frontend Structure (React / Next.js)

```
src/frontend/
├── public/                     # Static assets served as-is
│   ├── favicon.ico
│   └── images/
│
├── src/
│   ├── app/                    # App-level setup (Next.js App Router / React Router)
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── (routes)/           # Route-based pages
│   │       ├── dashboard/
│   │       ├── upload/
│   │       └── results/
│   │
│   ├── components/             # Reusable UI components
│   │   ├── ui/                 # Primitive components (Button, Input, Modal)
│   │   ├── forms/              # Form-specific components
│   │   └── layout/             # Layout components (Header, Sidebar, Footer)
│   │
│   ├── hooks/                  # Custom React hooks
│   │   ├── useAuth.ts
│   │   └── useApi.ts
│   │
│   ├── services/               # API client functions
│   │   ├── api.ts              # Base API client (axios/fetch wrapper)
│   │   ├── authService.ts
│   │   └── documentService.ts
│   │
│   ├── store/                  # State management (Zustand / Redux / Context)
│   │   ├── authStore.ts
│   │   └── documentStore.ts
│   │
│   ├── types/                  # TypeScript type definitions
│   │   ├── api.ts
│   │   └── models.ts
│   │
│   ├── utils/                  # Utility functions
│   │   ├── formatters.ts
│   │   └── validators.ts
│   │
│   └── styles/                 # Global styles, themes
│       ├── globals.css
│       └── theme.ts
│
├── package.json
├── tsconfig.json
├── next.config.js              # or vite.config.ts
└── .eslintrc.json
```

### Key Principles:
- **Feature-based or Layer-based:** Group by feature (dashboard/, upload/) or by layer (components/, hooks/, services/)
- **Colocation:** Keep related files close together (component + its styles + its tests)
- **Barrel Exports:** Use `index.ts` files to simplify imports
- **API Layer Abstraction:** Never call `fetch()` directly in components; use a service layer
- **Type Safety:** Use TypeScript; define shared types in `types/`

---

## 6. AI/ML Module Structure

Based on best practices from "Deep Learning in Production" (AI Summer) and the Cookiecutter Data Science template:

```
src/ml/
├── __init__.py
│
├── configs/                    # All configurable parameters
│   ├── __init__.py
│   ├── model_config.py         # Model architecture hyperparameters
│   ├── training_config.py      # Training hyperparameters
│   └── inference_config.py     # Inference settings
│
├── data/                       # Data loading & preprocessing
│   ├── __init__.py
│   ├── dataloader.py           # Data loading classes
│   ├── preprocessing.py        # Data cleaning, normalization
│   ├── augmentation.py         # Data augmentation strategies
│   └── dataset.py              # Custom dataset classes
│
├── models/                     # Model architectures
│   ├── __init__.py
│   ├── base_model.py           # Abstract base class for all models
│   ├── classifier.py           # Specific model implementation
│   └── components/             # Reusable model building blocks
│       ├── __init__.py
│       ├── attention.py
│       └── layers.py
│
├── training/                   # Training logic
│   ├── __init__.py
│   ├── trainer.py              # Training loop
│   ├── callbacks.py            # Training callbacks (early stopping, checkpointing)
│   └── losses.py               # Custom loss functions
│
├── evaluation/                 # Model evaluation
│   ├── __init__.py
│   ├── metrics.py              # Custom metrics
│   └── evaluator.py            # Evaluation pipeline
│
├── inference/                  # Inference / serving
│   ├── __init__.py
│   ├── predictor.py            # Prediction interface (used by backend)
│   └── postprocessing.py       # Post-processing of model outputs
│
├── pipelines/                  # End-to-end ML pipelines
│   ├── __init__.py
│   ├── train_pipeline.py       # Full training pipeline
│   └── inference_pipeline.py   # Full inference pipeline
│
└── utils/                      # ML-specific utilities
    ├── __init__.py
    ├── visualization.py        # Plotting, visualization helpers
    └── io.py                   # Model saving/loading utilities
```

### Key Principles:

1. **Abstract Base Classes:** Define a `BaseModel` with abstract methods (`load_data`, `build`, `train`, `evaluate`) so all models share a consistent interface:

```python
from abc import ABC, abstractmethod

class BaseModel(ABC):
    """Abstract Model class inherited by all models"""
    def __init__(self, cfg):
        self.config = cfg

    @abstractmethod
    def build(self):
        pass

    @abstractmethod
    def train(self):
        pass

    @abstractmethod
    def evaluate(self):
        pass
```

2. **Configuration as Code:** Store all hyperparameters in config files, never hard-code them:

```python
CFG = {
    "model": {
        "architecture": "resnet50",
        "input_shape": [224, 224, 3],
        "num_classes": 10
    },
    "training": {
        "batch_size": 32,
        "epochs": 100,
        "learning_rate": 0.001,
        "optimizer": "adam"
    }
}
```

3. **Separate Training from Inference:** Training code should never be loaded in production; inference should be lightweight
4. **Reproducibility:** Pin random seeds, log all hyperparameters, version your data
5. **Notebooks are for Exploration Only:** Never put production code in notebooks; use them for EDA and prototyping, then move finalized code to proper modules

---

## 7. The Twelve-Factor App Principles

From [12factor.net](https://12factor.net/) — these are essential for any web service, especially AI-powered ones:

| # | Factor | What It Means for AI Web Apps |
|---|--------|-------------------------------|
| I | **Codebase** | One repo tracked in Git, many deploys (dev, staging, prod) |
| II | **Dependencies** | Explicitly declare all deps in `requirements.txt` / `package.json`. Never rely on system-wide packages |
| III | **Config** | Store config (API keys, DB URLs, model paths) in environment variables, NOT in code |
| IV | **Backing Services** | Treat databases, ML model stores, message queues as attached resources that can be swapped |
| V | **Build, Release, Run** | Strictly separate building the app, creating a release (build + config), and running it |
| VI | **Processes** | Run the app as stateless processes. Don't store state in memory between requests |
| VII | **Port Binding** | Export services via port binding (e.g., FastAPI on port 8000) |
| VIII | **Concurrency** | Scale out via the process model (multiple workers/containers, not bigger machines) |
| IX | **Disposability** | Fast startup, graceful shutdown. Models should load quickly or be pre-loaded |
| X | **Dev/Prod Parity** | Keep dev, staging, and prod as similar as possible. Use Docker for consistency |
| XI | **Logs** | Treat logs as event streams. Don't write to log files; stream to stdout |
| XII | **Admin Processes** | Run admin tasks (DB migrations, model retraining) as one-off processes |

### Critical for AI Projects:
- **Factor III (Config):** Model file paths, API keys for ML services, feature flags for A/B testing different models — all in env vars
- **Factor VI (Processes):** ML model inference should be stateless; load the model once at startup, serve predictions statelessly
- **Factor X (Dev/Prod Parity):** Use the SAME model serving code locally and in production. Docker is your best friend here

---

## 8. Separation of Concerns

### The Layered Architecture Pattern

```
┌─────────────────────────────────────────────┐
│              Presentation Layer              │ ← Frontend (React/Next.js)
│         (UI Components, Pages, Forms)        │
├─────────────────────────────────────────────┤
│                 API Layer                    │ ← FastAPI Routers/Endpoints
│        (Request handling, validation)        │
├─────────────────────────────────────────────┤
│              Service Layer                   │ ← Business Logic
│      (Orchestration, business rules)         │
├─────────────────────────────────────────────┤
│            ML / AI Layer                     │ ← Model Inference
│     (Prediction, pre/post-processing)        │
├─────────────────────────────────────────────┤
│             Data Access Layer                │ ← Repositories
│         (Database queries, ORM)              │
├─────────────────────────────────────────────┤
│               Data Layer                     │ ← Database, File Storage
│      (PostgreSQL, S3, Model Registry)        │
└─────────────────────────────────────────────┘
```

### Rules:
- Each layer only communicates with the layer directly below it
- Never let the API layer talk directly to the database — go through the service layer
- The ML layer exposes a clean interface (`predict(input) -> output`) that the service layer calls
- Frontend talks ONLY to the API layer via HTTP/WebSocket

### Signs of Poor Separation:
- **Circular dependencies:** Module A imports Module B, which imports Module A
- **Hidden coupling:** Changing one module breaks unrelated tests
- **Heavy global state:** Global variables modified by multiple modules
- **Spaghetti code:** Nested if/else with copy-pasted logic across files

---

## 9. Configuration Management

### Environment-Based Configuration

```python
# src/backend/app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # App
    APP_NAME: str = "DigiChecker"
    DEBUG: bool = False
    API_VERSION: str = "v1"

    # Database
    DATABASE_URL: str
    
    # ML Model
    MODEL_PATH: str = "./models/latest"
    MODEL_VERSION: str = "1.0.0"
    CONFIDENCE_THRESHOLD: float = 0.85
    
    # External Services
    STORAGE_BUCKET: str = ""
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### `.env.example` (committed to Git as a template):
```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/digichecker

# ML Model
MODEL_PATH=./models/latest
MODEL_VERSION=1.0.0
CONFIDENCE_THRESHOLD=0.85

# External Services  
STORAGE_BUCKET=my-bucket
```

### Rules:
- **NEVER** commit `.env` files — add to `.gitignore`
- **ALWAYS** commit `.env.example` as a template
- Use `pydantic-settings` (Python) or `dotenv` (Node.js) to validate config at startup
- Fail fast: if a required config is missing, crash immediately with a clear error

---

## 10. Testing Strategy & Structure

### Test Directory Structure (mirrors `src/`):
```
tests/
├── backend/
│   ├── unit/                   # Fast, isolated tests
│   │   ├── test_user_service.py
│   │   ├── test_document_service.py
│   │   └── test_ml_service.py
│   ├── integration/            # Tests with real DB/services
│   │   ├── test_api_auth.py
│   │   └── test_api_documents.py
│   └── conftest.py             # Shared fixtures
│
├── frontend/
│   ├── components/             # Component unit tests
│   │   └── Button.test.tsx
│   ├── hooks/                  # Hook tests
│   │   └── useAuth.test.ts
│   └── e2e/                    # End-to-end tests (Playwright/Cypress)
│       └── upload-flow.spec.ts
│
├── ml/
│   ├── test_preprocessing.py
│   ├── test_model_inference.py
│   └── test_pipeline.py
│
└── conftest.py                 # Root-level shared config
```

### Testing Pyramid for AI Web Apps:

```
        /\
       /  \        E2E Tests (few, slow, expensive)
      /    \       - Full user flows through UI
     /──────\
    /        \     Integration Tests (moderate)
   /          \    - API endpoints with real DB
  /   ────────  \  - ML model with real data
 /              \
/________________\ Unit Tests (many, fast, cheap)
                   - Service logic, utils, preprocessing
                   - Model architecture (shape checks)
                   - Data validation
```

### ML-Specific Tests:
- **Data Validation Tests:** Check that input data meets expected schema/format
- **Model Shape Tests:** Verify input/output tensor shapes are correct
- **Prediction Sanity Tests:** Known inputs should produce expected outputs
- **Performance Regression Tests:** Model accuracy shouldn't drop below a threshold
- **Inference Latency Tests:** Predictions should complete within an SLA

---

## 11. Documentation Practices

### What to Document:

| Document | Location | Purpose |
|----------|----------|---------|
| `README.md` | Root | Project overview, quick start, tech stack |
| `CONTRIBUTING.md` | Root | How to contribute, code style, PR process |
| `docs/architecture/` | Docs | Architecture Decision Records (ADRs) |
| `docs/api/` | Docs | API documentation (auto-generated from OpenAPI) |
| Docstrings | In code | Function/class purpose, args, returns |
| `CHANGELOG.md` | Root | Version history and changes |

### Code Documentation (Python Docstrings — Google Style):

```python
def predict(self, image: np.ndarray) -> dict:
    """Run inference on a single image.

    Args:
        image: Input image as numpy array with shape (H, W, C).

    Returns:
        Dictionary with keys:
            - 'class': predicted class label (str)
            - 'confidence': prediction confidence (float, 0-1)
            - 'processing_time_ms': inference time in milliseconds (float)

    Raises:
        ValueError: If image dimensions don't match expected input shape.
    """
```

### Rules:
- **Descriptive names > comments:** `normalize_image()` is better than `n()` with a comment
- **Type hints everywhere:** Use Python type hints and TypeScript types
- **Keep docs close to code:** Docstrings > separate wiki pages
- **Auto-generate API docs:** FastAPI generates OpenAPI/Swagger docs automatically

---

## 12. DevOps & Deployment Structure

### Docker Setup:

```dockerfile
# infrastructure/docker/Dockerfile.backend
FROM python:3.11-slim

WORKDIR /app

# Install dependencies first (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY src/backend/ ./src/backend/
COPY src/ml/ ./src/ml/

# Copy model artifacts
COPY models/ ./models/

EXPOSE 8000
CMD ["uvicorn", "src.backend.app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose for Local Development:

```yaml
# docker-compose.yml
version: "3.8"

services:
  backend:
    build:
      context: .
      dockerfile: infrastructure/docker/Dockerfile.backend
    ports:
      - "8000:8000"
    env_file: .env
    volumes:
      - ./src/backend:/app/src/backend  # Hot reload
    depends_on:
      - db

  frontend:
    build:
      context: .
      dockerfile: infrastructure/docker/Dockerfile.frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: digichecker
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Makefile for Common Commands:

```makefile
.PHONY: install test lint run build deploy

install:
	pip install -r requirements.txt
	cd src/frontend && npm install

test:
	pytest tests/ -v
	cd src/frontend && npm test

lint:
	ruff check src/backend/ src/ml/
	cd src/frontend && npm run lint

run-backend:
	uvicorn src.backend.app.main:app --reload

run-frontend:
	cd src/frontend && npm run dev

build:
	docker-compose build

up:
	docker-compose up -d

down:
	docker-compose down
```

---

## 13. Common Anti-Patterns to Avoid

### 1. The "Notebook-as-Production" Anti-Pattern
- **Problem:** Jupyter notebooks used as the main codebase
- **Fix:** Use notebooks ONLY for exploration; move finalized code to proper Python modules

### 2. The "God File" Anti-Pattern
- **Problem:** A single `app.py` or `main.py` with thousands of lines
- **Fix:** Split by responsibility — routes, services, models, utils

### 3. The "Circular Import" Anti-Pattern
- **Problem:** Module A imports B, B imports A
- **Fix:** Extract shared code into a third module; use dependency injection

### 4. The "Hardcoded Config" Anti-Pattern
- **Problem:** `model_path = "/home/alex/models/v3.pkl"` in source code
- **Fix:** Use environment variables and a config class

### 5. The "Mixed Concerns" Anti-Pattern
- **Problem:** API endpoint that does DB queries, ML inference, and email sending all inline
- **Fix:** Layer your code — endpoint → service → repository/ML

### 6. The "No Tests" Anti-Pattern
- **Problem:** "It works on my machine" with zero tests
- **Fix:** Write tests from day one; at minimum, test your ML inference pipeline

### 7. The "Data in Git" Anti-Pattern
- **Problem:** Large datasets and model files committed to Git
- **Fix:** Use Git LFS, DVC (Data Version Control), or cloud storage (S3, GCS)

### 8. The "Spaghetti Dependencies" Anti-Pattern
- **Problem:** No `requirements.txt`, dependencies installed ad-hoc
- **Fix:** Pin all dependencies with versions; use virtual environments

---

## 14. Applied Example: Real DigiChecker

Applying all the above principles to our project:

```
Real DigiChecker/
├── .github/
│   └── workflows/
│       ├── ci.yml              # Lint + Test on every PR
│       └── deploy.yml          # Deploy on merge to main
│
├── docs/
│   ├── architecture/
│   │   └── adr-001-tech-stack.md
│   ├── api/
│   └── study-notes/
│       └── 8-Mar-2026_Project-Structure-Good-Practices.md  ← (this file!)
│
├── src/
│   ├── backend/
│   │   ├── app/
│   │   │   ├── __init__.py
│   │   │   ├── main.py         # FastAPI entry
│   │   │   ├── config.py       # Settings from env
│   │   │   ├── dependencies.py
│   │   │   ├── api/v1/endpoints/
│   │   │   │   ├── auth.py
│   │   │   │   ├── documents.py
│   │   │   │   └── verify.py   # AI verification endpoint
│   │   │   ├── models/         # DB models
│   │   │   ├── schemas/        # Pydantic schemas
│   │   │   ├── services/       # Business logic
│   │   │   │   └── verification_service.py
│   │   │   └── repositories/   # DB access
│   │   ├── alembic/            # DB migrations
│   │   └── pyproject.toml
│   │
│   ├── frontend/
│   │   ├── src/
│   │   │   ├── app/            # Pages/routes
│   │   │   ├── components/     # UI components
│   │   │   ├── hooks/          # Custom hooks
│   │   │   ├── services/       # API client
│   │   │   └── types/          # TypeScript types
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── ml/
│       ├── configs/            # Model hyperparameters
│       ├── data/               # Data loading & preprocessing
│       ├── models/             # Model architectures
│       │   └── base_model.py
│       ├── training/           # Training loops
│       ├── inference/          # Prediction interface
│       │   └── predictor.py    # Called by backend's verification_service
│       └── utils/
│
├── tests/
│   ├── backend/
│   ├── frontend/
│   └── ml/
│
├── notebooks/                  # Exploration notebooks
├── models/                     # .gitkeep (actual models in cloud storage)
├── data/                       # .gitkeep (actual data in cloud storage)
│
├── infrastructure/
│   └── docker/
│       ├── Dockerfile.backend
│       └── Dockerfile.frontend
│
├── .env.example
├── .gitignore
├── docker-compose.yml
├── Makefile
├── README.md
└── LICENSE
```

### How the Layers Connect:

```
User → Frontend (React)
         ↓ HTTP Request
       Backend API (FastAPI)
         ↓ Service Call
       Verification Service
         ↓ Inference Call
       ML Predictor (loads model, runs prediction)
         ↓ Returns result
       Backend formats response
         ↓ HTTP Response
       Frontend displays result
```

---

## 15. References & Further Reading

### Documentation & Guides
- **The Hitchhiker's Guide to Python — Structuring Your Project:** https://docs.python-guide.org/writing/structure/
- **FastAPI — Bigger Applications (Multiple Files):** https://fastapi.tiangolo.com/tutorial/bigger-applications/
- **The Twelve-Factor App:** https://12factor.net/

### Books & Courses
- **"Deep Learning in Production"** by Sergios Karagiannakos (AI Summer) — Covers project structure, OOP, type checking, documentation for ML code
- **"Building Machine Learning Powered Applications"** by Emmanuel Ameisen (O'Reilly)
- **Full Stack Deep Learning Course:** https://fullstackdeeplearning.com/

### Templates & Cookiecutters
- **Cookiecutter Data Science:** https://cookiecutter-data-science.drivendata.org/ — The gold standard template for data science projects
- **FastAPI Full Stack Template:** https://github.com/fastapi/full-stack-fastapi-template
- **The AI Summer Deep Learning in Production repo:** https://github.com/The-AI-Summer/Deep-Learning-In-Production

### Articles
- **"Best Practices to Write Deep Learning Code" (AI Summer):** https://theaisummer.com/best-practices-deep-learning-code/ — Project structure, OOP patterns, type checking, and documentation for ML code

### Key Takeaways
1. **Structure is architecture** — invest time upfront, it pays off exponentially
2. **Separate concerns** — frontend, backend, ML, and data each get their own space
3. **Config in environment** — never hardcode paths, keys, or hyperparameters
4. **Test everything testable** — especially your ML inference pipeline
5. **Document as you go** — type hints, docstrings, README, ADRs
6. **Use abstract base classes** for ML models to enforce consistent interfaces
7. **Notebooks are NOT production code** — they're for exploration only
8. **Docker for consistency** — same environment everywhere
9. **Makefile for common tasks** — `make test`, `make run`, `make build`
10. **Follow the Twelve-Factor App** — it's the industry standard for a reason