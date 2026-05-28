# A/B Testing Platform -- Development Plan

> Project: Candidate #031 · Plan Created: 2026-05-25

---

## Technology Decisions

### Data Model: Hybrid Relational + JSONB (Suggestion 3, adapted)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the foundation, with selective elements from Suggestion 1 (normalized results tables) and Suggestion 2 (append-only audit log).

**Rationale:**
- The hybrid model's 16-table footprint enables faster MVP delivery than the 36-table normalized model, while preserving foreign keys on core relationships (experiment-to-project, metric-to-project, experiment-to-flag).
- JSONB for targeting rules, variations, and SDK payloads mirrors the actual data access patterns: these structures are always read and written as a unit, never queried relationally across joins.
- Statistical results use JSONB within `experiment_snapshot` because the results UI always fetches the full variation-by-metric matrix. This avoids the 3-join pattern of the normalized model.
- The audit log borrows the append-only, JSONB-diff pattern from Suggestion 2's event design, but without the full CQRS/event-sourcing complexity. This gives us audit compliance without projection infrastructure.
- PostgreSQL JSONB with GIN indexes provides acceptable performance for all identified query patterns (find experiments by metric, find flags targeting a country, SRM detection scans).

**What we defer:** Full event sourcing (Suggestion 2) is architecturally superior for temporal debugging but adds 3-6 months of infrastructure work (event processors, projection rebuilders, eventual consistency handling). We can migrate to it post-v1 if the audit trail requirements justify the investment.

### Language & Framework

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend API** | Python 3.12+ / FastAPI | FastAPI provides async request handling, automatic OpenAPI doc generation, and Pydantic validation for JSONB payloads. Python is the natural choice for the statistics engine (SciPy, statsmodels). |
| **Statistics Engine** | Python (SciPy, statsmodels, custom) | SciPy for frequentist tests (t-tests, chi-squared, z-tests). statsmodels for CUPED regression. Custom implementation for sequential testing (SPRT) and Bayesian inference (PyMC or lightweight conjugate priors). |
| **Frontend** | React 18+ / TypeScript / Vite | React is the dominant framework for complex dashboard UIs. TypeScript prevents the class of bugs that JSONB-heavy data models are susceptible to. Vite for fast builds. |
| **Database** | PostgreSQL 16+ | Native JSONB support with GIN indexes, TIMESTAMPTZ, UUID generation, and the statistical functions (percentile_cont, etc.) needed for warehouse query generation. |
| **Cache / Pub-Sub** | Redis 7+ | SDK payload caching, flag change pub-sub for near-instant propagation, and rate limiting. |
| **Task Queue** | Celery + Redis | Asynchronous warehouse query execution, scheduled result snapshots, and bandit weight recalculation. |
| **SDK (initial)** | JavaScript/TypeScript (web) | First SDK targets the largest user base. Thin client that fetches cached payload from API and evaluates flags/experiments locally. |
| **Containerization** | Docker + Docker Compose | Self-hostable deployment from day one. Compose for local dev; Helm charts for Kubernetes in Phase 8. |

### Project Structure

```
ab-testing-platform/
|-- docker-compose.yml
|-- Dockerfile
|-- Makefile
|
|-- backend/
|   |-- alembic/                    # Database migrations
|   |   |-- versions/
|   |   +-- env.py
|   |-- app/
|   |   |-- main.py                 # FastAPI application entry point
|   |   |-- config.py               # Settings (env vars, defaults)
|   |   |-- dependencies.py         # Dependency injection (DB sessions, auth)
|   |   |-- models/                 # SQLAlchemy ORM models
|   |   |   |-- organization.py
|   |   |   |-- project.py
|   |   |   |-- user.py
|   |   |   |-- feature_flag.py
|   |   |   |-- experiment.py
|   |   |   |-- metric.py
|   |   |   |-- segment.py
|   |   |   |-- data_source.py
|   |   |   |-- experiment_snapshot.py
|   |   |   |-- exclusion_group.py
|   |   |   |-- sdk_connection.py
|   |   |   |-- audit_log.py
|   |   |   +-- webhook.py
|   |   |-- schemas/                # Pydantic request/response schemas
|   |   |   |-- experiment.py
|   |   |   |-- feature_flag.py
|   |   |   |-- metric.py
|   |   |   +-- ...
|   |   |-- api/                    # Route handlers
|   |   |   |-- v1/
|   |   |   |   |-- experiments.py
|   |   |   |   |-- feature_flags.py
|   |   |   |   |-- metrics.py
|   |   |   |   |-- projects.py
|   |   |   |   |-- sdk.py          # SDK payload endpoint
|   |   |   |   |-- auth.py
|   |   |   |   +-- webhooks.py
|   |   |   +-- router.py
|   |   |-- services/               # Business logic
|   |   |   |-- experiment_service.py
|   |   |   |-- flag_service.py
|   |   |   |-- flag_evaluator.py   # Server-side flag evaluation engine
|   |   |   |-- metric_service.py
|   |   |   |-- sdk_payload_service.py
|   |   |   +-- audit_service.py
|   |   |-- stats/                  # Statistics engine
|   |   |   |-- frequentist.py      # t-tests, z-tests, chi-squared
|   |   |   |-- bayesian.py         # Conjugate priors, posterior computation
|   |   |   |-- sequential.py       # SPRT implementation
|   |   |   |-- cuped.py            # CUPED variance reduction
|   |   |   |-- srm.py              # Sample Ratio Mismatch detection
|   |   |   |-- power.py            # Power analysis / sample size calculator
|   |   |   +-- bandit.py           # Thompson sampling, epsilon-greedy, UCB1
|   |   |-- warehouse/              # Data warehouse connectors
|   |   |   |-- base.py             # Abstract warehouse interface
|   |   |   |-- bigquery.py
|   |   |   |-- snowflake.py
|   |   |   |-- databricks.py
|   |   |   |-- redshift.py
|   |   |   |-- postgres.py
|   |   |   +-- clickhouse.py
|   |   |-- tasks/                  # Celery async tasks
|   |   |   |-- snapshot_runner.py  # Execute warehouse queries, compute results
|   |   |   |-- bandit_updater.py
|   |   |   +-- webhook_dispatcher.py
|   |   +-- utils/
|   |       |-- hashing.py          # Consistent hashing for traffic allocation
|   |       |-- encryption.py       # Credential encryption/decryption
|   |       +-- pagination.py
|   |-- tests/
|   |   |-- unit/
|   |   |   |-- stats/
|   |   |   |-- services/
|   |   |   +-- warehouse/
|   |   |-- integration/
|   |   +-- conftest.py
|   |-- requirements.txt
|   +-- pyproject.toml
|
|-- frontend/
|   |-- src/
|   |   |-- components/
|   |   |   |-- experiments/
|   |   |   |-- flags/
|   |   |   |-- metrics/
|   |   |   |-- common/
|   |   |   +-- layout/
|   |   |-- pages/
|   |   |-- hooks/
|   |   |-- api/                    # API client (auto-generated from OpenAPI)
|   |   |-- store/                  # State management (Zustand or React Query)
|   |   |-- utils/
|   |   +-- types/
|   |-- tests/
|   |-- package.json
|   +-- tsconfig.json
|
|-- sdks/
|   |-- javascript/                 # First SDK
|   |   |-- src/
|   |   |   |-- client.ts
|   |   |   |-- evaluator.ts        # Local flag/experiment evaluation
|   |   |   |-- hashing.ts          # MurmurHash3 for consistent assignment
|   |   |   +-- types.ts
|   |   |-- tests/
|   |   |-- package.json
|   |   +-- tsconfig.json
|   +-- python/                     # Second SDK (Phase 5)
|
|-- docs/
|   |-- api/
|   +-- sdk/
|
+-- scripts/
    |-- seed.py                     # Dev data seeding
    +-- migrate.sh
```

---

## Phase Dependency Graph

```
Phase 1 ──> Phase 2 ──> Phase 3 ──> Phase 4
               |            |           |
               |            v           v
               +-------> Phase 5 --> Phase 6
                            |           |
                            v           v
                         Phase 7 --> Phase 8
                            |           |
                            v           v
                         Phase 9 --> Phase 10
```

**Critical path:** 1 -> 2 -> 3 -> 4 -> 6 -> 8 -> 10
**Parallel tracks after Phase 2:** Phases 5 and 7 can proceed independently.
**Phase 9 (AI features)** requires Phase 7 (advanced statistics) to be complete.

---

## Phase 1: Foundation & Infrastructure

**Goal:** Establish the project skeleton, database schema, authentication, and CI/CD pipeline so all subsequent phases build on a working, tested foundation.

**Duration estimate:** 3 weeks

### Task 1.1: Project Scaffolding & Docker Setup

**What:** Create the project directory structure, Docker Compose environment (PostgreSQL 16, Redis 7, FastAPI app), and basic Makefile commands for build/run/test/migrate.

**Design:**

```python
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: abtesting
      POSTGRES_USER: abtesting
      POSTGRES_PASSWORD: ${DB_PASSWORD:-devpassword}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgresql+asyncpg://abtesting:${DB_PASSWORD:-devpassword}@db:5432/abtesting
      REDIS_URL: redis://redis:6379/0
    volumes:
      - ./backend:/app

volumes:
  pgdata:
```

```python
# backend/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.router import api_router
from app.config import settings

app = FastAPI(
    title="A/B Testing Platform",
    version="0.1.0",
    docs_url="/api/docs",
    openapi_url="/api/openapi.json",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router, prefix="/api/v1")

@app.get("/health")
async def health_check():
    return {"status": "ok"}
```

```python
# backend/app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    cors_origins: list[str] = ["http://localhost:3000"]
    secret_key: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    jwt_expiry_minutes: int = 60 * 24  # 24 hours
    encryption_key: str = "change-me-32-byte-key-for-fernet"

    class Config:
        env_file = ".env"

settings = Settings()
```

**Testing:**
- `docker compose up` starts all services without errors
- `curl http://localhost:8000/health` returns `{"status": "ok"}`
- `make test` runs pytest with zero tests collected (no failures)
- PostgreSQL is accessible on port 5432 with the configured credentials
- Redis responds to PING on port 6379

### Task 1.2: Database Schema & Migrations

**What:** Implement the complete Hybrid JSONB schema (all 16 tables from Data Model Suggestion 3) using Alembic migrations, with indexes and constraints.

**Design:**

```python
# backend/app/models/organization.py
from sqlalchemy import Column, String, text
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMP
from app.models.base import Base

class Organization(Base):
    __tablename__ = "organization"

    id = Column(UUID(as_uuid=True), primary_key=True, server_default=text("gen_random_uuid()"))
    name = Column(String(255), nullable=False)
    slug = Column(String(100), nullable=False, unique=True)
    settings = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    created_at = Column(TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"))
    updated_at = Column(TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"))
```

```python
# backend/app/models/experiment.py
from sqlalchemy import Column, String, Integer, ForeignKey, text
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMP
from app.models.base import Base

class Experiment(Base):
    __tablename__ = "experiment"

    id = Column(UUID(as_uuid=True), primary_key=True, server_default=text("gen_random_uuid()"))
    project_id = Column(UUID(as_uuid=True), ForeignKey("project.id", ondelete="CASCADE"), nullable=False)
    name = Column(String(500), nullable=False)
    slug = Column(String(255), nullable=False)
    description = Column(String)
    hypothesis = Column(String)
    type = Column(String(30), nullable=False, server_default="ab")
    status = Column(String(30), nullable=False, server_default="draft")
    feature_flag_id = Column(UUID(as_uuid=True), ForeignKey("feature_flag.id"))
    owner_id = Column(UUID(as_uuid=True), ForeignKey("app_user.id"))
    variations = Column(JSONB, nullable=False, server_default=text("'[]'::jsonb"))
    metric_assignments = Column(JSONB, nullable=False, server_default=text("'[]'::jsonb"))
    targeting = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    stats_config = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    bandit_config = Column(JSONB)
    started_at = Column(TIMESTAMP(timezone=True))
    ended_at = Column(TIMESTAMP(timezone=True))
    revision = Column(Integer, nullable=False, server_default="1")
    created_at = Column(TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"))
    updated_at = Column(TIMESTAMP(timezone=True), nullable=False, server_default=text("now()"))
```

**Testing:**
- `alembic upgrade head` applies all migrations without errors on a fresh database
- `alembic downgrade base` followed by `alembic upgrade head` round-trips cleanly
- All 16 tables exist with correct columns, types, and constraints
- Foreign key constraints reject inserts with invalid references (test with experiment referencing non-existent project)
- JSONB columns accept valid JSON and reject invalid payloads
- Unique constraints enforce slug uniqueness per project (test duplicate slug insert raises IntegrityError)
- GIN indexes exist on `metric.tags`, `feature_flag.tags`, `experiment.metric_assignments`

### Task 1.3: Authentication & User Management

**What:** Implement JWT-based authentication with local email/password signup and login. Create user CRUD endpoints and the membership model (user-to-organization binding with roles).

**Design:**

```python
# backend/app/api/v1/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas.auth import SignupRequest, LoginRequest, TokenResponse
from app.services.auth_service import AuthService

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/signup", response_model=TokenResponse, status_code=201)
async def signup(body: SignupRequest, auth_service: AuthService = Depends()):
    user = await auth_service.create_user(
        email=body.email, password=body.password, name=body.name
    )
    token = auth_service.create_access_token(user_id=str(user.id))
    return TokenResponse(access_token=token, token_type="bearer")

@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, auth_service: AuthService = Depends()):
    user = await auth_service.authenticate(email=body.email, password=body.password)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")
    token = auth_service.create_access_token(user_id=str(user.id))
    return TokenResponse(access_token=token, token_type="bearer")
```

```python
# backend/app/services/auth_service.py
from datetime import datetime, timedelta, timezone
from uuid import UUID
import jwt
from passlib.context import CryptContext
from app.config import settings

class AuthService:
    pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

    def create_access_token(self, user_id: str) -> str:
        payload = {
            "sub": user_id,
            "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expiry_minutes),
            "iat": datetime.now(timezone.utc),
        }
        return jwt.encode(payload, settings.secret_key, algorithm=settings.jwt_algorithm)

    def verify_password(self, plain: str, hashed: str) -> bool:
        return self.pwd_context.verify(plain, hashed)

    def hash_password(self, password: str) -> str:
        return self.pwd_context.hash(password)
```

**Testing:**
- POST `/api/v1/auth/signup` with valid email/password returns 201 with JWT token
- POST `/api/v1/auth/signup` with duplicate email returns 409 Conflict
- POST `/api/v1/auth/login` with valid credentials returns 200 with JWT token
- POST `/api/v1/auth/login` with wrong password returns 401 Unauthorized
- GET `/api/v1/users/me` with valid JWT returns user profile
- GET `/api/v1/users/me` without JWT returns 401
- GET `/api/v1/users/me` with expired JWT returns 401
- Passwords are stored as bcrypt hashes (verify raw password is not in database)
- Creating a membership assigns a role; role is included in JWT claims or user profile

### Task 1.4: Organization & Project CRUD

**What:** Implement REST endpoints for organization and project management, including the embedded environments JSONB structure.

**Design:**

```python
# backend/app/schemas/project.py
from pydantic import BaseModel, Field
from uuid import UUID
from datetime import datetime

class EnvironmentConfig(BaseModel):
    slug: str = Field(max_length=100)
    name: str = Field(max_length=255)

class ProjectCreate(BaseModel):
    name: str = Field(max_length=255)
    slug: str = Field(max_length=100, pattern=r"^[a-z0-9-]+$")
    description: str | None = None
    environments: list[EnvironmentConfig] = [
        EnvironmentConfig(slug="production", name="Production"),
        EnvironmentConfig(slug="staging", name="Staging"),
    ]

class ProjectResponse(BaseModel):
    id: UUID
    organization_id: UUID
    name: str
    slug: str
    description: str | None
    environments: list[EnvironmentConfig]
    settings: dict
    created_at: datetime
    updated_at: datetime
```

```python
# backend/app/api/v1/projects.py
from fastapi import APIRouter, Depends, HTTPException, status
from uuid import UUID
from app.schemas.project import ProjectCreate, ProjectResponse, ProjectUpdate
from app.services.project_service import ProjectService
from app.dependencies import get_current_user, get_project_service

router = APIRouter(prefix="/projects", tags=["projects"])

@router.post("/", response_model=ProjectResponse, status_code=201)
async def create_project(
    body: ProjectCreate,
    org_id: UUID,
    user=Depends(get_current_user),
    service: ProjectService = Depends(get_project_service),
):
    return await service.create(org_id=org_id, data=body, created_by=user.id)

@router.get("/{project_id}", response_model=ProjectResponse)
async def get_project(
    project_id: UUID,
    user=Depends(get_current_user),
    service: ProjectService = Depends(get_project_service),
):
    project = await service.get(project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    return project
```

**Testing:**
- POST `/api/v1/projects/` creates a project with default environments (production, staging)
- POST with duplicate slug within same org returns 409
- POST with invalid slug format (uppercase, spaces) returns 422
- GET `/api/v1/projects/{id}` returns project with embedded environments
- PATCH `/api/v1/projects/{id}` updates name/description, returns updated project
- DELETE `/api/v1/projects/{id}` removes project and cascades to child entities
- Organization CRUD follows the same patterns (create, read, update, list)

### Task 1.5: CI/CD Pipeline

**What:** Configure GitHub Actions for automated testing (pytest, frontend unit tests), linting (ruff, mypy), and Docker image builds on every push.

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  backend-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: abtesting_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r backend/requirements.txt
      - run: cd backend && python -m pytest tests/ -v --cov=app --cov-report=xml
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/abtesting_test
          REDIS_URL: redis://localhost:6379/0

  backend-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install ruff mypy
      - run: cd backend && ruff check app/
      - run: cd backend && mypy app/ --ignore-missing-imports

  frontend-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: cd frontend && npm ci
      - run: cd frontend && npm run lint
      - run: cd frontend && npm test -- --run
```

**Testing:**
- Pushing to any branch triggers CI pipeline
- Pipeline fails if any test fails (verify with intentionally broken test)
- Ruff reports linting errors for PEP 8 violations
- mypy catches type errors in Python code
- Docker build produces a runnable image (`docker build -t abtesting .`)

### Definition of Done -- Phase 1
- [ ] `docker compose up` starts PostgreSQL, Redis, and FastAPI with zero manual configuration
- [ ] All 16 database tables created via Alembic migration with correct constraints and indexes
- [ ] User signup/login works end-to-end with JWT authentication
- [ ] Organization and project CRUD endpoints return correct responses
- [ ] CI pipeline runs on GitHub Actions with test, lint, and build stages
- [ ] Test coverage baseline established (>80% for auth and project services)
- [ ] API documentation auto-generated at `/api/docs` (Swagger UI)

---

## Phase 2: Feature Flags Engine

**Goal:** Build the complete feature flag system -- creation, targeting rules, percentage rollouts, per-environment state, user overrides, and the server-side evaluation engine that SDKs will replicate.

**Duration estimate:** 4 weeks

### Task 2.1: Feature Flag CRUD

**What:** Implement REST endpoints for creating, reading, updating, listing, and archiving feature flags. Flags store per-environment configuration (enabled state, targeting rules, rollout percentages, overrides) in the `environment_settings` JSONB column.

**Design:**

```python
# backend/app/schemas/feature_flag.py
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Literal

class TargetingCondition(BaseModel):
    attribute: str = Field(max_length=255)
    operator: Literal["$eq", "$ne", "$in", "$nin", "$gt", "$gte", "$lt", "$lte", "$regex", "$exists"]
    value: str | int | float | bool | list  # JSON-encoded in DB

class RuleVariation(BaseModel):
    value: str  # JSON-encoded variation value
    weight: int = Field(ge=0, le=10000)  # basis points

class TargetingRule(BaseModel):
    description: str | None = None
    type: Literal["targeting", "force", "rollout", "experiment"] = "targeting"
    enabled: bool = True
    conditions: list[TargetingCondition] = []
    variations: list[RuleVariation] = []

class EnvironmentSetting(BaseModel):
    enabled: bool = False
    rules: list[TargetingRule] = []
    overrides: dict[str, str] = {}  # context_key -> variation_value

class FeatureFlagCreate(BaseModel):
    key: str = Field(max_length=255, pattern=r"^[a-z0-9_-]+$")
    description: str | None = None
    value_type: Literal["boolean", "string", "number", "json"] = "boolean"
    default_value: str = "false"
    tags: list[str] = []

class FeatureFlagResponse(BaseModel):
    id: UUID
    project_id: UUID
    key: str
    description: str | None
    value_type: str
    default_value: str
    tags: list[str]
    status: str
    environment_settings: dict[str, EnvironmentSetting]
    revision: int
    created_at: str
    updated_at: str
```

**Testing:**
- POST `/api/v1/projects/{id}/flags` creates a flag with empty environment_settings for each project environment
- GET `/api/v1/projects/{id}/flags` lists all flags with pagination (limit, offset)
- GET `/api/v1/projects/{id}/flags/{flag_id}` returns full flag with environment_settings
- PATCH updates key, description, tags, default_value; bumps revision
- PATCH with stale revision returns 409 Conflict (optimistic concurrency)
- DELETE archives flag (sets status to "archived"), does not hard-delete
- Flag key uniqueness enforced per project (duplicate key returns 409)
- Invalid key format (uppercase, spaces) returns 422

### Task 2.2: Targeting Rule Management

**What:** Implement endpoints to add, update, reorder, and remove targeting rules within a flag's environment settings. Validate that variation weights sum to 10000 (100.00%) within each rule.

**Design:**

```python
# backend/app/api/v1/feature_flags.py (targeting endpoints)

@router.put(
    "/{flag_id}/environments/{env_slug}/rules",
    response_model=FeatureFlagResponse,
)
async def set_environment_rules(
    flag_id: UUID,
    env_slug: str,
    body: SetRulesRequest,  # { enabled: bool, rules: TargetingRule[] }
    user=Depends(get_current_user),
    service: FlagService = Depends(),
):
    """Replace all targeting rules for a flag in a specific environment."""
    return await service.set_environment_rules(
        flag_id=flag_id,
        env_slug=env_slug,
        enabled=body.enabled,
        rules=body.rules,
        updated_by=user.id,
        expected_revision=body.revision,
    )

@router.put(
    "/{flag_id}/environments/{env_slug}/overrides",
    response_model=FeatureFlagResponse,
)
async def set_overrides(
    flag_id: UUID,
    env_slug: str,
    body: SetOverridesRequest,  # { overrides: dict[str, str] }
    user=Depends(get_current_user),
    service: FlagService = Depends(),
):
    """Set user-level overrides for a flag in a specific environment."""
    return await service.set_overrides(
        flag_id=flag_id,
        env_slug=env_slug,
        overrides=body.overrides,
        updated_by=user.id,
        expected_revision=body.revision,
    )
```

```python
# backend/app/services/flag_service.py
class FlagService:
    async def set_environment_rules(self, flag_id, env_slug, enabled, rules, updated_by, expected_revision):
        flag = await self._get_flag(flag_id)
        if flag.revision != expected_revision:
            raise ConflictError("Flag has been modified by another user")

        # Validate weights sum to 10000 for each rule with variations
        for rule in rules:
            if rule.variations:
                total_weight = sum(v.weight for v in rule.variations)
                if total_weight != 10000:
                    raise ValidationError(f"Variation weights must sum to 10000, got {total_weight}")

        # Update the JSONB
        env_settings = dict(flag.environment_settings)
        env_settings[env_slug] = {
            "enabled": enabled,
            "rules": [r.model_dump() for r in rules],
            "overrides": env_settings.get(env_slug, {}).get("overrides", {}),
        }
        flag.environment_settings = env_settings
        flag.revision += 1
        flag.updated_at = utcnow()

        await self._save(flag)
        await self._audit("feature_flag", flag.id, "rules_updated", updated_by, {env_slug: rules})
        await self._invalidate_sdk_cache(flag.project_id, env_slug)
        return flag
```

**Testing:**
- PUT rules with valid conditions and weights=10000 succeeds
- PUT rules with weights summing to 9999 returns 422 validation error
- PUT rules with invalid operator (e.g., "$invalid") returns 422
- Overrides set for specific context_keys are returned in GET
- Setting rules triggers SDK cache invalidation (verify Redis key deleted/updated)
- Audit log entry created for every rule change
- Concurrent updates with same revision: first succeeds, second returns 409

### Task 2.3: Flag Evaluation Engine

**What:** Build the server-side flag evaluation engine that resolves a feature flag to a value given an evaluation context (user attributes). This is the reference implementation that SDKs will replicate in their respective languages.

**Design:**

```python
# backend/app/services/flag_evaluator.py
import hashlib
import struct
from typing import Any

class FlagEvaluator:
    """Evaluates feature flags given an evaluation context.

    Evaluation order:
    1. Check overrides (exact context_key match)
    2. Evaluate rules in sort order (first matching rule wins)
    3. Fall back to default value
    """

    def evaluate(
        self,
        flag: dict,  # flag configuration from SDK payload
        context: dict[str, Any],  # user attributes: {"id": "user-123", "country": "US", ...}
        environment: str = "production",
    ) -> tuple[str, str]:  # (value, source) where source = "override"|"rule"|"default"
        env_config = flag.get("environment_settings", {}).get(environment, {})

        if not env_config.get("enabled", False):
            return flag["default_value"], "disabled"

        # 1. Check overrides
        context_key = str(context.get("id", ""))
        overrides = env_config.get("overrides", {})
        if context_key in overrides:
            return overrides[context_key], "override"

        # 2. Evaluate rules
        for rule in env_config.get("rules", []):
            if not rule.get("enabled", True):
                continue
            if self._matches_conditions(rule.get("conditions", []), context):
                return self._select_variation(rule["variations"], flag["key"], context_key), "rule"

        # 3. Default
        return flag["default_value"], "default"

    def _matches_conditions(self, conditions: list[dict], context: dict) -> bool:
        """All conditions must match (AND logic)."""
        if not conditions:
            return True  # empty conditions = match all
        return all(self._eval_condition(c, context) for c in conditions)

    def _eval_condition(self, condition: dict, context: dict) -> bool:
        attr_value = context.get(condition["attribute"])
        op = condition["operator"]
        target = condition["value"]

        if op == "$eq":
            return attr_value == target
        elif op == "$ne":
            return attr_value != target
        elif op == "$in":
            return attr_value in target
        elif op == "$nin":
            return attr_value not in target
        elif op == "$gt":
            return attr_value is not None and attr_value > target
        elif op == "$gte":
            return attr_value is not None and attr_value >= target
        elif op == "$lt":
            return attr_value is not None and attr_value < target
        elif op == "$lte":
            return attr_value is not None and attr_value <= target
        elif op == "$exists":
            return (attr_value is not None) == target
        elif op == "$regex":
            import re
            return attr_value is not None and bool(re.search(target, str(attr_value)))
        return False

    def _select_variation(self, variations: list[dict], flag_key: str, context_key: str) -> str:
        """Consistent hashing to select a variation based on weights."""
        hash_value = self._hash(flag_key, context_key)  # 0-9999
        cumulative = 0
        for v in variations:
            cumulative += v["weight"]
            if hash_value < cumulative:
                return v["value"]
        return variations[-1]["value"]  # safety fallback

    def _hash(self, flag_key: str, context_key: str) -> int:
        """MurmurHash3-like consistent hash returning 0-9999."""
        seed = f"{flag_key}__{context_key}"
        h = hashlib.md5(seed.encode()).digest()
        n = struct.unpack("<I", h[:4])[0]
        return n % 10000
```

**Testing:**
- Disabled flag returns default value with source="disabled"
- Override for specific user returns override value with source="override"
- User matching targeting condition gets rule variation
- User not matching any conditions gets default value
- Consistent hashing: same user + flag always returns same variation
- 50/50 split: over 10,000 random user IDs, each variation gets ~50% +/- 2%
- 90/10 split: over 10,000 random user IDs, distribution matches within 2% tolerance
- Multiple conditions: all must match (AND logic)
- `$in` operator with list of values works correctly
- `$regex` operator matches partial strings
- `$exists` operator handles None/missing attributes
- Empty conditions list matches all users (100% rollout)
- Rules evaluated in order: first match wins, subsequent rules skipped

### Task 2.4: SDK Payload Endpoint

**What:** Build the GET `/api/v1/sdk/{api_key}` endpoint that returns the complete feature flag + experiment configuration payload for a given SDK connection. Cache the payload in Redis; invalidate on flag changes.

**Design:**

```python
# backend/app/api/v1/sdk.py
from fastapi import APIRouter, Depends, HTTPException, Response
from app.services.sdk_payload_service import SDKPayloadService

router = APIRouter(prefix="/sdk", tags=["sdk"])

@router.get("/{api_key}")
async def get_sdk_payload(
    api_key: str,
    response: Response,
    service: SDKPayloadService = Depends(),
):
    payload = await service.get_payload(api_key)
    if payload is None:
        raise HTTPException(status_code=401, detail="Invalid SDK key")

    response.headers["ETag"] = f'"{payload["version"]}"'
    response.headers["Cache-Control"] = "public, max-age=30"
    return payload
```

```python
# backend/app/services/sdk_payload_service.py
import json
from app.config import settings

class SDKPayloadService:
    CACHE_TTL = 60  # seconds

    async def get_payload(self, api_key: str) -> dict | None:
        # 1. Check Redis cache
        cache_key = f"sdk_payload:{api_key}"
        cached = await self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 2. Look up SDK connection
        conn = await self.db.get_sdk_connection_by_key(api_key)
        if not conn:
            return None

        # 3. Build payload
        payload = await self._build_payload(conn.project_id, conn.environment)
        payload["version"] = conn.id  # or a computed hash

        # 4. Cache
        await self.redis.set(cache_key, json.dumps(payload), ex=self.CACHE_TTL)
        return payload

    async def _build_payload(self, project_id, environment) -> dict:
        flags = await self.db.get_active_flags(project_id)
        experiments = await self.db.get_running_experiments(project_id)

        return {
            "features": {
                f.key: {
                    "defaultValue": f.default_value,
                    "rules": (f.environment_settings.get(environment, {}).get("rules", [])),
                }
                for f in flags
                if f.environment_settings.get(environment, {}).get("enabled", False)
            },
            "experiments": [
                {
                    "key": e.slug,
                    "variations": e.variations,
                    "hashAttribute": e.targeting.get("hash_attribute", "id"),
                    "hashVersion": e.targeting.get("hash_version", 2),
                }
                for e in experiments
                if e.targeting.get("environments", {}).get(environment, {}).get("enabled", False)
            ],
        }

    async def invalidate(self, project_id, environment):
        """Called when flags or experiments change."""
        connections = await self.db.get_sdk_connections(project_id, environment)
        for conn in connections:
            await self.redis.delete(f"sdk_payload:{conn.api_key}")
```

**Testing:**
- GET with valid API key returns JSON payload with features and experiments
- GET with invalid API key returns 401
- Response includes ETag header for client-side caching
- Payload only includes enabled flags for the connection's environment
- Payload only includes running experiments enabled in the connection's environment
- Modifying a flag invalidates the Redis cache (subsequent GET rebuilds from DB)
- Cache hit: second request within TTL is served from Redis (verify with Redis MONITOR)
- Payload structure matches the expected SDK contract (validate against JSON schema)

### Task 2.5: Audit Log Service

**What:** Implement the audit log system that records every mutation (create, update, delete, start, stop) with the acting user, entity type/ID, action, and a JSONB diff of changes.

**Design:**

```python
# backend/app/services/audit_service.py
from uuid import UUID
from datetime import datetime, timezone

class AuditService:
    async def log(
        self,
        organization_id: UUID,
        project_id: UUID | None,
        user_id: UUID | None,
        entity_type: str,
        entity_id: UUID,
        action: str,
        details: dict | None = None,
        ip_address: str | None = None,
    ):
        await self.db.execute(
            """
            INSERT INTO audit_log (organization_id, project_id, user_id, entity_type, entity_id, action, details, ip_address)
            VALUES (:org_id, :project_id, :user_id, :entity_type, :entity_id, :action, :details::jsonb, :ip)
            """,
            {
                "org_id": organization_id,
                "project_id": project_id,
                "user_id": user_id,
                "entity_type": entity_type,
                "entity_id": entity_id,
                "action": action,
                "details": details,
                "ip": ip_address,
            },
        )

    @staticmethod
    def compute_diff(old: dict, new: dict) -> dict:
        """Compute field-level diff between old and new state."""
        diff = {}
        all_keys = set(old.keys()) | set(new.keys())
        for key in all_keys:
            old_val = old.get(key)
            new_val = new.get(key)
            if old_val != new_val:
                diff[key] = {"old": old_val, "new": new_val}
        return diff
```

**Testing:**
- Creating a flag generates an audit log entry with action="created"
- Updating a flag generates an entry with action="updated" and details showing the diff
- Deleting/archiving generates an entry with action="archived"
- Audit log entries include the acting user's ID and IP address
- GET `/api/v1/projects/{id}/audit-log` returns paginated audit entries
- Filter by entity_type works (e.g., `?entity_type=feature_flag`)
- Filter by date range works (e.g., `?since=2026-01-01`)
- Diff correctly captures old and new values for changed fields

### Definition of Done -- Phase 2
- [ ] Feature flags can be created, updated, listed, and archived via REST API
- [ ] Targeting rules support all 10 operators ($eq, $ne, $in, $nin, $gt, $gte, $lt, $lte, $regex, $exists)
- [ ] Flag evaluation engine correctly resolves flags given user context
- [ ] Consistent hashing produces deterministic, evenly distributed variant assignments
- [ ] SDK payload endpoint returns cached, environment-scoped flag configurations
- [ ] Cache invalidation triggers on every flag mutation
- [ ] Audit log captures every mutation with user, action, and field-level diff
- [ ] Per-environment flag state (enabled/disabled, rules, overrides) works independently
- [ ] Optimistic concurrency prevents lost updates on concurrent edits
- [ ] >90% test coverage on flag evaluation engine

---

## Phase 3: Experiment Engine (Core)

**Goal:** Build the experiment lifecycle -- creation, variation management, metric attachment, start/stop, and the warehouse query pipeline that computes statistical results.

**Duration estimate:** 5 weeks

### Task 3.1: Experiment CRUD & Lifecycle

**What:** Implement REST endpoints for experiment management including creation (with variations and metric assignments), status transitions (draft -> running -> paused -> stopped -> completed), and linking experiments to feature flags.

**Design:**

```python
# backend/app/schemas/experiment.py
from pydantic import BaseModel, Field, field_validator
from uuid import UUID
from typing import Literal

class VariationInput(BaseModel):
    key: str = Field(max_length=100, pattern=r"^[a-z0-9-]+$")
    name: str = Field(max_length=255)
    value: str  # JSON-encoded
    weight: int = Field(ge=0, le=10000)
    is_control: bool = False

class MetricAssignment(BaseModel):
    metric_id: UUID
    role: Literal["primary", "secondary", "guardrail"]

class StatsConfig(BaseModel):
    engine: Literal["frequentist", "bayesian"] = "frequentist"
    significance_level: float = Field(default=0.05, ge=0.001, le=0.20)
    power: float = Field(default=0.80, ge=0.50, le=0.99)
    sequential_testing: dict | None = None  # {"enabled": true, "tuning_parameter": 0.0001}
    min_sample_size: int | None = None
    max_duration_days: int | None = None

class ExperimentCreate(BaseModel):
    name: str = Field(max_length=500)
    slug: str = Field(max_length=255, pattern=r"^[a-z0-9-]+$")
    description: str | None = None
    hypothesis: str | None = None
    type: Literal["ab", "multivariate", "split_url", "bandit"] = "ab"
    feature_flag_id: UUID | None = None
    variations: list[VariationInput] = []
    metric_assignments: list[MetricAssignment] = []
    stats_config: StatsConfig = StatsConfig()

    @field_validator("variations")
    @classmethod
    def validate_variations(cls, v):
        if v:
            weights_sum = sum(var.weight for var in v)
            if weights_sum != 10000:
                raise ValueError(f"Variation weights must sum to 10000, got {weights_sum}")
            controls = [var for var in v if var.is_control]
            if len(controls) != 1:
                raise ValueError("Exactly one variation must be marked as control")
        return v
```

```python
# backend/app/services/experiment_service.py
class ExperimentService:
    VALID_TRANSITIONS = {
        "draft": ["running"],
        "running": ["paused", "stopped"],
        "paused": ["running", "stopped"],
        "stopped": ["completed", "archived"],
        "completed": ["archived"],
    }

    async def transition(self, experiment_id: UUID, new_status: str, user_id: UUID, reason: str = None):
        experiment = await self._get(experiment_id)
        if new_status not in self.VALID_TRANSITIONS.get(experiment.status, []):
            raise InvalidTransitionError(
                f"Cannot transition from '{experiment.status}' to '{new_status}'"
            )

        # Pre-start validation
        if new_status == "running":
            await self._validate_for_start(experiment)

        experiment.status = new_status
        if new_status == "running" and experiment.started_at is None:
            experiment.started_at = utcnow()
        if new_status in ("stopped", "completed"):
            experiment.ended_at = utcnow()

        experiment.revision += 1
        await self._save(experiment)
        await self._audit("experiment", experiment.id, new_status, user_id, {"reason": reason})
        await self._invalidate_sdk_cache(experiment.project_id)

    async def _validate_for_start(self, experiment):
        """Ensure experiment is ready to run."""
        errors = []
        if not experiment.variations:
            errors.append("At least 2 variations required")
        elif len(experiment.variations) < 2:
            errors.append("At least 2 variations required")
        if not any(ma["role"] == "primary" for ma in experiment.metric_assignments):
            errors.append("At least one primary metric required")
        if not experiment.targeting.get("environments"):
            errors.append("At least one environment must be enabled")
        if errors:
            raise ValidationError(errors)
```

**Testing:**
- POST creates experiment in "draft" status
- Variations weights must sum to 10000 (422 on invalid sum)
- Exactly one control variation required (422 otherwise)
- Status transitions follow state machine: draft->running allowed, draft->completed rejected (400)
- Starting requires: >= 2 variations, >= 1 primary metric, >= 1 enabled environment
- Starting sets `started_at` timestamp
- Stopping sets `ended_at` timestamp
- Pausing and resuming preserves `started_at`
- Linking to a feature flag validates the flag exists and belongs to the same project
- Duplicate slug within project returns 409
- Audit log captures every status transition with reason

### Task 3.2: Metric Registry

**What:** Implement the standardized metric CRUD system. Metrics define what to measure (conversion rate, revenue, etc.) with their aggregation type, source fact table, and statistical parameters (cap value, conversion window, CUPED covariates).

**Design:**

```python
# backend/app/schemas/metric.py
from pydantic import BaseModel, Field
from uuid import UUID
from typing import Literal

class CovariateConfig(BaseModel):
    metric_id: UUID
    lookback_days: int = Field(default=7, ge=1, le=90)

class MetricConfig(BaseModel):
    fact_table_id: UUID | None = None
    aggregation: Literal["sum", "count", "avg", "countDistinct", "max", "min"] = "sum"
    value_column: str | None = None
    sql_filter: str | None = None
    cap_value: float | None = None  # Winsorization cap
    conversion_window_hours: int = Field(default=72, ge=1, le=720)
    delay_hours: int = Field(default=0, ge=0, le=168)
    is_inverse: bool = False  # True if lower is better (bounce rate, load time)
    min_sample_size: int = Field(default=100, ge=10)
    denominator_metric_id: UUID | None = None  # For ratio metrics
    covariates: list[CovariateConfig] = []

class MetricCreate(BaseModel):
    name: str = Field(max_length=255)
    slug: str = Field(max_length=255, pattern=r"^[a-z0-9-]+$")
    description: str | None = None
    type: Literal["binomial", "count", "duration", "revenue", "ratio", "quantile"]
    tags: list[str] = []
    config: MetricConfig = MetricConfig()
```

**Testing:**
- POST creates metric with all config fields stored in JSONB
- Binomial metrics do not require value_column; count/revenue do
- Ratio metrics require denominator_metric_id (422 if missing)
- Metric slug unique per project
- GET `/api/v1/projects/{id}/metrics` lists all metrics with filtering by type, tag, status
- PATCH updates config fields without overwriting unspecified fields
- Archiving a metric in use by a running experiment returns 400
- Covariate self-reference (metric A is covariate of itself) returns 422
- Tags are searchable via GIN index (verify with `?tag=revenue` filter)

### Task 3.3: Data Source & Warehouse Connectors

**What:** Build the data source connection management and the abstract warehouse connector interface. Implement the first connector (PostgreSQL) for development/testing, with BigQuery and Snowflake connectors following.

**Design:**

```python
# backend/app/warehouse/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class QueryResult:
    columns: list[str]
    rows: list[list]
    row_count: int
    execution_time_ms: float

class WarehouseConnector(ABC):
    """Abstract interface for warehouse connectors."""

    @abstractmethod
    async def test_connection(self) -> bool:
        """Verify the connection is valid."""
        ...

    @abstractmethod
    async def execute_query(self, sql: str, params: dict | None = None) -> QueryResult:
        """Execute a SQL query and return results."""
        ...

    @abstractmethod
    async def get_tables(self) -> list[str]:
        """List available tables/views."""
        ...

    @abstractmethod
    async def get_columns(self, table: str) -> list[dict]:
        """Get column names and types for a table."""
        ...

    @abstractmethod
    def generate_experiment_query(
        self,
        assignment_sql: str,
        metric_sql: str,
        experiment_id: str,
        start_date: str,
        end_date: str,
        conversion_window_hours: int,
    ) -> str:
        """Generate the warehouse-specific SQL for computing experiment results."""
        ...
```

```python
# backend/app/warehouse/postgres.py
import asyncpg
from app.warehouse.base import WarehouseConnector, QueryResult

class PostgresConnector(WarehouseConnector):
    def __init__(self, connection_params: dict):
        self.dsn = connection_params["dsn"]

    async def test_connection(self) -> bool:
        try:
            conn = await asyncpg.connect(self.dsn)
            await conn.execute("SELECT 1")
            await conn.close()
            return True
        except Exception:
            return False

    async def execute_query(self, sql: str, params: dict | None = None) -> QueryResult:
        conn = await asyncpg.connect(self.dsn)
        try:
            import time
            start = time.monotonic()
            rows = await conn.fetch(sql, *(params or {}).values())
            elapsed = (time.monotonic() - start) * 1000
            return QueryResult(
                columns=[k for k in rows[0].keys()] if rows else [],
                rows=[list(r.values()) for r in rows],
                row_count=len(rows),
                execution_time_ms=elapsed,
            )
        finally:
            await conn.close()

    def generate_experiment_query(
        self, assignment_sql, metric_sql, experiment_id, start_date, end_date, conversion_window_hours
    ) -> str:
        return f"""
        WITH assignments AS (
            SELECT user_id, variation_id, timestamp AS assigned_at
            FROM ({assignment_sql}) a
            WHERE experiment_id = '{experiment_id}'
              AND timestamp >= '{start_date}'
              AND timestamp <= '{end_date}'
        ),
        metrics AS (
            SELECT user_id, timestamp AS event_at, value
            FROM ({metric_sql}) m
        ),
        joined AS (
            SELECT
                a.user_id,
                a.variation_id,
                m.value
            FROM assignments a
            LEFT JOIN metrics m
              ON a.user_id = m.user_id
              AND m.event_at >= a.assigned_at
              AND m.event_at <= a.assigned_at + interval '{conversion_window_hours} hours'
        )
        SELECT
            variation_id,
            COUNT(DISTINCT user_id) AS users,
            AVG(COALESCE(value, 0)) AS mean,
            STDDEV(COALESCE(value, 0)) AS stddev,
            SUM(COALESCE(value, 0)) AS total_value
        FROM joined
        GROUP BY variation_id
        """
```

**Testing:**
- POST `/api/v1/projects/{id}/data-sources` stores encrypted connection params
- Test connection endpoint validates warehouse is reachable
- PostgreSQL connector executes queries and returns structured results
- Fact table discovery lists available tables
- Column introspection returns correct types
- Experiment query generation produces valid SQL (execute against test data)
- Connection params are encrypted at rest (verify raw DB value is not readable)
- Invalid credentials return descriptive error (not a stack trace)

### Task 3.4: Experiment Result Computation (Snapshot Runner)

**What:** Build the async task that executes warehouse queries to compute experiment results. For each experiment, it: queries assignments, queries metric events, joins them, computes per-variation statistics, detects SRM, and stores the results as an `experiment_snapshot`.

**Design:**

```python
# backend/app/tasks/snapshot_runner.py
from celery import shared_task
from app.stats.frequentist import FrequentistEngine
from app.stats.srm import SRMDetector

@shared_task(bind=True, max_retries=3)
def compute_experiment_snapshot(self, experiment_id: str):
    """Compute a statistical snapshot for an experiment."""
    experiment = db.get_experiment(experiment_id)
    data_source = db.get_data_source(experiment.project_id)
    connector = get_connector(data_source)

    results_by_variation = {}
    queries_run = []

    for metric_assignment in experiment.metric_assignments:
        metric = db.get_metric(metric_assignment["metric_id"])
        fact_table = db.get_fact_table(metric.config["fact_table_id"])

        sql = connector.generate_experiment_query(
            assignment_sql=data_source.queries["assignment"]["sql"],
            metric_sql=fact_table.sql_query,
            experiment_id=experiment_id,
            start_date=str(experiment.started_at),
            end_date=str(utcnow()),
            conversion_window_hours=metric.config.get("conversion_window_hours", 72),
        )
        queries_run.append(sql)
        raw_results = connector.execute_query(sql)

        # Compute statistics
        stats_engine = FrequentistEngine(
            significance_level=experiment.stats_config.get("significance_level", 0.05)
        )
        control_data = find_control(raw_results, experiment.variations)
        for row in raw_results.rows:
            variation_key = row[0]
            stats = stats_engine.compute(
                control_users=control_data["users"],
                control_mean=control_data["mean"],
                control_stddev=control_data["stddev"],
                treatment_users=row[1],
                treatment_mean=row[2],
                treatment_stddev=row[3],
            )
            results_by_variation.setdefault(variation_key, {})[metric.slug] = stats

    # SRM detection
    srm = SRMDetector.check(
        expected_weights=[v["weight"] for v in experiment.variations],
        observed_counts=[results_by_variation[v["key"]].get("users", 0) for v in experiment.variations if v["key"] in results_by_variation],
    )

    # Store snapshot
    snapshot = {
        "experiment_id": experiment_id,
        "status": "success",
        "summary": {
            "total_users": sum(r.get("users", 0) for r in results_by_variation.values()),
            "srm": {"p_value": srm.p_value, "detected": srm.detected},
            "queries_run": queries_run,
        },
        "results": format_results(results_by_variation, experiment.variations),
    }
    db.create_snapshot(snapshot)
```

**Testing:**
- Snapshot runner executes warehouse query and stores results
- Results include per-variation user count, mean, stddev, p-value, CI bounds
- SRM detection triggers when observed ratios deviate from expected (chi-squared p < 0.01)
- SRM not triggered for valid 50/50 split with balanced observations
- Snapshot stores the actual SQL queries executed (for auditability)
- Failed warehouse query creates snapshot with status="error" and error message
- Metrics with cap_value apply winsorization before computing statistics
- Conversion window correctly limits metric events to N hours after assignment
- Multiple metrics per experiment: each metric computed independently
- Control variation identified correctly from variations list

### Task 3.5: Frequentist Statistics Engine

**What:** Implement the frequentist statistical testing engine: two-sample t-test for continuous metrics, two-proportion z-test for binomial metrics, confidence intervals, and relative uplift calculations.

**Design:**

```python
# backend/app/stats/frequentist.py
from dataclasses import dataclass
from scipy import stats as scipy_stats
import math

@dataclass
class StatResult:
    users: int
    mean: float
    stddev: float
    cr: float | None           # conversion rate (binomial only)
    ci_lower: float | None     # CI of the difference (treatment - control)
    ci_upper: float | None
    p_value: float | None
    uplift_percent: float | None
    uplift_mean: float | None  # absolute difference
    is_statistically_significant: bool

class FrequentistEngine:
    def __init__(self, significance_level: float = 0.05):
        self.alpha = significance_level

    def compute_binomial(
        self,
        control_users: int, control_conversions: int,
        treatment_users: int, treatment_conversions: int,
    ) -> StatResult:
        """Two-proportion z-test for binomial metrics."""
        p_c = control_conversions / control_users if control_users > 0 else 0
        p_t = treatment_conversions / treatment_users if treatment_users > 0 else 0

        # Pooled proportion
        p_pool = (control_conversions + treatment_conversions) / (control_users + treatment_users)
        se = math.sqrt(p_pool * (1 - p_pool) * (1/control_users + 1/treatment_users))

        if se == 0:
            z_stat, p_value = 0.0, 1.0
        else:
            z_stat = (p_t - p_c) / se
            p_value = 2 * (1 - scipy_stats.norm.cdf(abs(z_stat)))

        # Confidence interval of the difference
        se_diff = math.sqrt(p_c*(1-p_c)/control_users + p_t*(1-p_t)/treatment_users)
        z_crit = scipy_stats.norm.ppf(1 - self.alpha / 2)
        diff = p_t - p_c

        return StatResult(
            users=treatment_users,
            mean=p_t,
            stddev=math.sqrt(p_t * (1 - p_t)) if p_t > 0 else 0,
            cr=p_t,
            ci_lower=diff - z_crit * se_diff,
            ci_upper=diff + z_crit * se_diff,
            p_value=p_value,
            uplift_percent=((p_t - p_c) / p_c * 100) if p_c > 0 else None,
            uplift_mean=diff,
            is_statistically_significant=p_value < self.alpha,
        )

    def compute_continuous(
        self,
        control_users: int, control_mean: float, control_stddev: float,
        treatment_users: int, treatment_mean: float, treatment_stddev: float,
    ) -> StatResult:
        """Welch's t-test for continuous metrics."""
        se = math.sqrt(
            (control_stddev**2 / control_users) + (treatment_stddev**2 / treatment_users)
        )

        if se == 0:
            t_stat, p_value = 0.0, 1.0
            df = control_users + treatment_users - 2
        else:
            t_stat = (treatment_mean - control_mean) / se
            # Welch-Satterthwaite degrees of freedom
            num = (control_stddev**2/control_users + treatment_stddev**2/treatment_users)**2
            den = (
                (control_stddev**2/control_users)**2 / (control_users - 1)
                + (treatment_stddev**2/treatment_users)**2 / (treatment_users - 1)
            )
            df = num / den if den > 0 else control_users + treatment_users - 2
            p_value = 2 * (1 - scipy_stats.t.cdf(abs(t_stat), df))

        diff = treatment_mean - control_mean
        t_crit = scipy_stats.t.ppf(1 - self.alpha / 2, df)

        return StatResult(
            users=treatment_users,
            mean=treatment_mean,
            stddev=treatment_stddev,
            cr=None,
            ci_lower=diff - t_crit * se,
            ci_upper=diff + t_crit * se,
            p_value=p_value,
            uplift_percent=((treatment_mean - control_mean) / control_mean * 100) if control_mean != 0 else None,
            uplift_mean=diff,
            is_statistically_significant=p_value < self.alpha,
        )
```

```python
# backend/app/stats/srm.py
from scipy.stats import chisquare

class SRMDetector:
    SRM_THRESHOLD = 0.01  # p < 0.01 indicates SRM

    @staticmethod
    def check(expected_weights: list[int], observed_counts: list[int]) -> dict:
        """Chi-squared test for Sample Ratio Mismatch."""
        total_observed = sum(observed_counts)
        total_weight = sum(expected_weights)

        expected_counts = [
            (w / total_weight) * total_observed for w in expected_weights
        ]

        chi2, p_value = chisquare(observed_counts, f_exp=expected_counts)

        return {
            "p_value": float(p_value),
            "detected": p_value < SRMDetector.SRM_THRESHOLD,
            "chi_squared": float(chi2),
            "variation_counts": [
                {"expected": int(e), "observed": o}
                for e, o in zip(expected_counts, observed_counts)
            ],
        }
```

**Testing:**
- Binomial test: known conversion rates (10% vs 12%, n=5000 each) produce expected p-value range
- Binomial test: identical rates produce p-value > 0.05 (not significant)
- Continuous test: known means (100 vs 105, stddev=20, n=1000) produce expected significance
- Continuous test: Welch's df correction applied (verified against scipy.stats.ttest_ind)
- Confidence interval at 95%: 95 out of 100 simulated experiments contain true difference
- Uplift calculation: 10% control, 12% treatment -> 20% uplift
- Edge cases: zero users (returns neutral result), zero stddev (no division error)
- SRM: 50/50 split with 4900/5100 does not trigger (p > 0.01)
- SRM: 50/50 split with 4000/6000 triggers (p < 0.01)
- SRM chi-squared matches scipy.stats.chisquare output

### Task 3.6: Results API & Scheduled Computation

**What:** Build the REST API for fetching experiment results (latest snapshot, historical snapshots) and configure scheduled snapshot computation via Celery Beat.

**Design:**

```python
# backend/app/api/v1/experiments.py (results endpoints)
@router.get("/{experiment_id}/results", response_model=ExperimentResultsResponse)
async def get_results(
    experiment_id: UUID,
    user=Depends(get_current_user),
    service: ExperimentService = Depends(),
):
    """Get the latest experiment results snapshot."""
    experiment = await service.get(experiment_id)
    snapshot = await service.get_latest_snapshot(experiment_id)
    return ExperimentResultsResponse(
        experiment=experiment,
        snapshot=snapshot,
        is_stale=snapshot and (utcnow() - snapshot.run_at).total_seconds() > 3600,
    )

@router.post("/{experiment_id}/results/refresh")
async def refresh_results(
    experiment_id: UUID,
    user=Depends(get_current_user),
    service: ExperimentService = Depends(),
):
    """Trigger an immediate results recomputation."""
    task = compute_experiment_snapshot.delay(str(experiment_id))
    return {"task_id": str(task.id), "status": "queued"}

@router.get("/{experiment_id}/results/history")
async def get_results_history(
    experiment_id: UUID,
    limit: int = 20,
    user=Depends(get_current_user),
    service: ExperimentService = Depends(),
):
    """Get historical snapshots for result trend analysis."""
    snapshots = await service.get_snapshots(experiment_id, limit=limit)
    return snapshots
```

**Testing:**
- GET results returns the most recent successful snapshot
- GET results for experiment with no snapshots returns null snapshot
- POST refresh queues a Celery task and returns task ID
- GET history returns snapshots in reverse chronological order
- `is_stale` flag is true when snapshot is older than 1 hour
- Celery Beat triggers snapshot computation every 6 hours for running experiments
- Failed snapshot (warehouse error) does not overwrite last successful snapshot

### Definition of Done -- Phase 3
- [ ] Experiments can be created with variations, metrics, and statistical configuration
- [ ] Lifecycle state machine enforces valid transitions (draft->running->stopped->completed)
- [ ] Pre-start validation ensures experiments have variations, metrics, and environments
- [ ] Metric registry supports binomial, count, revenue, ratio, and quantile types
- [ ] Data source connections store encrypted credentials and support connection testing
- [ ] PostgreSQL warehouse connector generates and executes experiment queries
- [ ] Frequentist statistics engine produces correct p-values and confidence intervals (validated against scipy)
- [ ] SRM detection uses chi-squared test with p < 0.01 threshold
- [ ] Experiment snapshots store complete results as JSONB with query audit trail
- [ ] Scheduled snapshot computation runs every 6 hours for running experiments
- [ ] Results API serves latest snapshot with staleness indicator

---

## Phase 4: JavaScript SDK

**Goal:** Ship the first client-side SDK that fetches the cached payload, evaluates flags and experiments locally (no server round-trip per evaluation), and tracks experiment exposures.

**Duration estimate:** 3 weeks

### Task 4.1: SDK Core (Payload Fetching & Caching)

**What:** Build the JavaScript/TypeScript SDK client that fetches the feature flag and experiment payload from the API, caches it locally, and provides a typed interface for flag evaluation.

**Design:**

```typescript
// sdks/javascript/src/client.ts
export interface ABTestingClientOptions {
  apiKey: string;
  apiHost?: string;       // default: "https://api.abtesting.dev"
  attributes?: Record<string, any>;  // user context
  refreshInterval?: number;  // ms, default 60000
  enableDevMode?: boolean;
}

export interface FeatureResult {
  value: any;
  source: "override" | "rule" | "default" | "disabled" | "experiment";
  experimentKey?: string;
  variationKey?: string;
}

export class ABTestingClient {
  private payload: SDKPayload | null = null;
  private attributes: Record<string, any>;
  private refreshTimer: ReturnType<typeof setInterval> | null = null;
  private listeners: Set<() => void> = new Set();

  constructor(private options: ABTestingClientOptions) {
    this.attributes = options.attributes || {};
  }

  async init(): Promise<void> {
    await this.fetchPayload();
    if (this.options.refreshInterval) {
      this.refreshTimer = setInterval(
        () => this.fetchPayload(),
        this.options.refreshInterval
      );
    }
  }

  setAttributes(attrs: Record<string, any>): void {
    this.attributes = { ...this.attributes, ...attrs };
  }

  getFeatureValue(key: string, defaultValue?: any): FeatureResult {
    if (!this.payload) {
      return { value: defaultValue, source: "default" };
    }
    return this.evaluator.evaluate(
      this.payload.features[key],
      this.attributes,
      key
    );
  }

  isOn(key: string): boolean {
    const result = this.getFeatureValue(key, false);
    return result.value === true || result.value === "true";
  }

  destroy(): void {
    if (this.refreshTimer) clearInterval(this.refreshTimer);
  }

  private async fetchPayload(): Promise<void> {
    const res = await fetch(
      `${this.options.apiHost || "https://api.abtesting.dev"}/api/v1/sdk/${this.options.apiKey}`
    );
    if (res.ok) {
      this.payload = await res.json();
      this.listeners.forEach((fn) => fn());
    }
  }
}
```

**Testing:**
- `init()` fetches payload from API and stores locally
- `getFeatureValue("my-flag")` returns correct value without network call
- `isOn("my-flag")` returns boolean for boolean flags
- `setAttributes()` updates user context for subsequent evaluations
- Auto-refresh fetches updated payload at configured interval
- `destroy()` clears refresh timer (no memory leak)
- SDK works offline after initial fetch (evaluates from cached payload)
- Network error during refresh does not clear existing payload

### Task 4.2: SDK Flag & Experiment Evaluator

**What:** Port the server-side flag evaluation engine to TypeScript for local evaluation. Include MurmurHash3 for consistent hashing (must produce identical assignments to the server-side Python implementation).

**Design:**

```typescript
// sdks/javascript/src/evaluator.ts
import { murmurhash3 } from "./hashing";

export class Evaluator {
  evaluate(
    feature: FeatureConfig | undefined,
    attributes: Record<string, any>,
    key: string
  ): FeatureResult {
    if (!feature) return { value: undefined, source: "default" };

    // Rules are already scoped to the SDK connection's environment
    const rules = feature.rules || [];

    // Check each rule in order
    for (const rule of rules) {
      if (rule.enabled === false) continue;
      if (this.matchesConditions(rule.conditions || [], attributes)) {
        const value = this.selectVariation(rule.variations, key, attributes);
        return { value: value.value, source: "rule" };
      }
    }

    return { value: feature.defaultValue, source: "default" };
  }

  private matchesConditions(
    conditions: Condition[],
    attributes: Record<string, any>
  ): boolean {
    if (conditions.length === 0) return true;
    return conditions.every((c) => this.evalCondition(c, attributes));
  }

  private evalCondition(cond: Condition, attrs: Record<string, any>): boolean {
    const val = attrs[cond.attribute];
    switch (cond.operator) {
      case "$eq": return val === cond.value;
      case "$ne": return val !== cond.value;
      case "$in": return Array.isArray(cond.value) && cond.value.includes(val);
      case "$nin": return Array.isArray(cond.value) && !cond.value.includes(val);
      case "$gt": return val != null && val > cond.value;
      case "$gte": return val != null && val >= cond.value;
      case "$lt": return val != null && val < cond.value;
      case "$lte": return val != null && val <= cond.value;
      case "$exists": return (val != null) === cond.value;
      case "$regex": return val != null && new RegExp(cond.value).test(String(val));
      default: return false;
    }
  }

  private selectVariation(
    variations: Variation[],
    featureKey: string,
    attributes: Record<string, any>
  ): Variation {
    const userId = String(attributes.id || "");
    const hash = this.hash(featureKey, userId); // 0-9999
    let cumulative = 0;
    for (const v of variations) {
      cumulative += v.weight;
      if (hash < cumulative) return v;
    }
    return variations[variations.length - 1];
  }

  private hash(featureKey: string, userId: string): number {
    const seed = `${featureKey}__${userId}`;
    // Must match Python implementation exactly
    const h = murmurhash3(seed, 0);
    return ((h % 10000) + 10000) % 10000; // ensure non-negative
  }
}
```

```typescript
// sdks/javascript/src/hashing.ts
export function murmurhash3(key: string, seed: number): number {
  // MurmurHash3_x86_32 implementation
  // Must be identical to Python's mmh3.hash(key, seed, signed=False) % 10000
  let h = seed;
  const len = key.length;
  const nblocks = Math.floor(len / 4);

  for (let i = 0; i < nblocks; i++) {
    let k =
      (key.charCodeAt(i * 4) & 0xff) |
      ((key.charCodeAt(i * 4 + 1) & 0xff) << 8) |
      ((key.charCodeAt(i * 4 + 2) & 0xff) << 16) |
      ((key.charCodeAt(i * 4 + 3) & 0xff) << 24);

    k = Math.imul(k, 0xcc9e2d51);
    k = (k << 15) | (k >>> 17);
    k = Math.imul(k, 0x1b873593);

    h ^= k;
    h = (h << 13) | (h >>> 19);
    h = Math.imul(h, 5) + 0xe6546b64;
  }

  // Handle tail bytes
  let k = 0;
  const tail = nblocks * 4;
  switch (len & 3) {
    case 3: k ^= (key.charCodeAt(tail + 2) & 0xff) << 16;
    case 2: k ^= (key.charCodeAt(tail + 1) & 0xff) << 8;
    case 1:
      k ^= key.charCodeAt(tail) & 0xff;
      k = Math.imul(k, 0xcc9e2d51);
      k = (k << 15) | (k >>> 17);
      k = Math.imul(k, 0x1b873593);
      h ^= k;
  }

  h ^= len;
  h ^= h >>> 16;
  h = Math.imul(h, 0x85ebca6b);
  h ^= h >>> 13;
  h = Math.imul(h, 0xc2b2ae35);
  h ^= h >>> 16;

  return h >>> 0; // unsigned 32-bit
}
```

**Testing:**
- Cross-language hash consistency: for 1000 test inputs, JS hash output matches Python hash output exactly
- Evaluation matches server-side: same flag config + attributes produce identical result in JS and Python
- All 10 operators produce correct results (mirror the Python test suite)
- Empty conditions match all users
- Weight-based selection: 50/50 split distributes evenly across 10,000 user IDs
- Undefined attribute with `$exists: false` returns true
- `$regex` matches partial strings
- `$in` with empty array returns false

### Task 4.3: Exposure Tracking

**What:** Implement automatic exposure tracking in the SDK. When a user is assigned to an experiment variation, the SDK fires an exposure event that the backend records for attribution.

**Design:**

```typescript
// sdks/javascript/src/tracking.ts
export interface ExposureEvent {
  experimentKey: string;
  variationKey: string;
  userId: string;
  timestamp: string;  // ISO 8601
  attributes: Record<string, any>;
}

export class ExposureTracker {
  private queue: ExposureEvent[] = [];
  private seen: Set<string> = new Set();  // dedupe within session
  private flushTimer: ReturnType<typeof setInterval> | null = null;

  constructor(
    private apiHost: string,
    private apiKey: string,
    private flushInterval: number = 10000,  // 10s
    private maxQueueSize: number = 100,
  ) {
    this.flushTimer = setInterval(() => this.flush(), this.flushInterval);
  }

  track(event: ExposureEvent): void {
    const dedupeKey = `${event.experimentKey}__${event.variationKey}__${event.userId}`;
    if (this.seen.has(dedupeKey)) return;
    this.seen.add(dedupeKey);
    this.queue.push(event);

    if (this.queue.length >= this.maxQueueSize) {
      this.flush();
    }
  }

  async flush(): Promise<void> {
    if (this.queue.length === 0) return;
    const events = [...this.queue];
    this.queue = [];

    try {
      await fetch(`${this.apiHost}/api/v1/sdk/${this.apiKey}/track`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ events }),
      });
    } catch {
      // Re-queue on failure (will retry on next flush)
      this.queue.push(...events);
    }
  }

  destroy(): void {
    if (this.flushTimer) clearInterval(this.flushTimer);
    this.flush(); // final flush
  }
}
```

**Testing:**
- Exposure event fired when user first encounters an experiment variation
- Same user + experiment + variation: only one exposure tracked per session (deduplication)
- Events batched and flushed every 10 seconds
- Events flushed immediately when queue reaches maxQueueSize
- Failed flush re-queues events for retry
- `destroy()` triggers a final flush before cleanup
- Backend `/track` endpoint receives and stores events correctly
- Event includes timestamp, userId, experimentKey, variationKey, and attributes

### Task 4.4: SDK Build & Package

**What:** Configure the SDK build pipeline (TypeScript -> ESM/CJS), minification, tree-shaking, and npm package publishing. Target bundle size under 10KB gzipped.

**Design:**

```json
// sdks/javascript/package.json
{
  "name": "@abtesting/sdk-js",
  "version": "0.1.0",
  "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "types": "dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js",
      "types": "./dist/types/index.d.ts"
    }
  },
  "sideEffects": false,
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/index.ts --format esm,cjs --dts --minify",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/",
    "size": "size-limit"
  }
}
```

**Testing:**
- `npm run build` produces ESM and CJS bundles
- Bundle size < 10KB gzipped (measure with size-limit)
- TypeScript type definitions included in package
- Tree-shaking works: importing only `isOn` does not bundle exposure tracking
- Package installs and imports correctly in a fresh Node.js project
- Package installs and imports correctly in a Vite/React project
- No Node.js-specific APIs used (fetch, not http; no fs/path)

### Definition of Done -- Phase 4
- [ ] SDK fetches payload from API and evaluates flags locally without per-call latency
- [ ] Flag evaluation produces identical results in JS and Python for all test cases
- [ ] Consistent hashing produces identical bucket assignments across languages
- [ ] Exposure tracking deduplicates within session and batches events
- [ ] SDK auto-refreshes payload at configurable interval
- [ ] Bundle size < 10KB gzipped
- [ ] Published to npm as `@abtesting/sdk-js`
- [ ] SDK README documents installation, initialization, and all public methods
- [ ] >95% test coverage on evaluator and hashing modules

---

## Phase 5: Frontend Dashboard (MVP)

**Goal:** Build the React-based web UI for managing feature flags, experiments, metrics, and viewing results.

**Duration estimate:** 5 weeks

### Task 5.1: Layout, Navigation & Auth UI

**What:** Build the application shell: sidebar navigation, project/environment selector, login/signup forms, and the authenticated layout wrapper.

**Design:**

```typescript
// frontend/src/components/layout/AppShell.tsx
export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col">
        <TopBar />
        <main className="flex-1 overflow-auto p-6 bg-gray-50">
          {children}
        </main>
      </div>
    </div>
  );
}

// Sidebar navigation items:
// - Feature Flags
// - Experiments
// - Metrics
// - Segments
// - Data Sources
// - SDK Connections
// - Settings
// - Audit Log
```

**Testing:**
- Login form submits credentials, stores JWT, redirects to dashboard
- Invalid login shows error message (not raw API response)
- Sidebar highlights active section
- Project selector lists user's projects; switching project reloads content
- Environment selector filters flags/experiments by environment
- Unauthenticated access redirects to login page
- JWT expiry triggers automatic redirect to login

### Task 5.2: Feature Flags UI

**What:** Build the feature flags management pages: flag list (with search, filter, status badges), flag detail page (environment tabs, targeting rule editor, override management), and flag creation form.

**Design:**

```typescript
// frontend/src/pages/FlagDetailPage.tsx
export function FlagDetailPage() {
  const { flagId } = useParams();
  const { data: flag, isLoading } = useFlag(flagId);
  const [activeEnv, setActiveEnv] = useState("production");

  if (isLoading) return <Skeleton />;

  return (
    <div>
      <FlagHeader flag={flag} />
      <EnvironmentTabs
        environments={flag.project.environments}
        active={activeEnv}
        onSelect={setActiveEnv}
      />
      <div className="grid grid-cols-3 gap-6 mt-6">
        <div className="col-span-2">
          <EnableToggle
            enabled={flag.environment_settings[activeEnv]?.enabled}
            onToggle={(enabled) => updateEnvSetting(flagId, activeEnv, { enabled })}
          />
          <TargetingRulesEditor
            rules={flag.environment_settings[activeEnv]?.rules || []}
            onSave={(rules) => updateRules(flagId, activeEnv, rules)}
          />
        </div>
        <div>
          <OverridesPanel
            overrides={flag.environment_settings[activeEnv]?.overrides || {}}
            onSave={(overrides) => updateOverrides(flagId, activeEnv, overrides)}
          />
          <FlagHistory flagId={flagId} />
        </div>
      </div>
    </div>
  );
}
```

**Testing:**
- Flag list loads and displays all flags with status badges (draft/active/archived)
- Search filters flags by key or description
- Creating a flag via form creates it in draft status
- Environment tabs switch between production/staging targeting rules
- Enable/disable toggle updates flag state and invalidates SDK cache
- Targeting rule editor: add condition, select operator, enter value
- Rule variation weights validated to sum to 100% (UI prevents submission otherwise)
- Override: add user ID + value, save, verify in flag detail
- Archiving a flag prompts confirmation dialog
- Optimistic concurrency: concurrent edit shows conflict notification

### Task 5.3: Experiment Management UI

**What:** Build the experiment pages: experiment list, creation wizard (hypothesis, variations, metrics, targeting, stats config), lifecycle controls (start/pause/stop), and results dashboard.

**Design:**

```typescript
// frontend/src/pages/ExperimentWizard.tsx
// Multi-step creation wizard:
// Step 1: Basics (name, slug, hypothesis, type)
// Step 2: Variations (add/remove variants, set weights, mark control)
// Step 3: Metrics (select primary + guardrail + secondary metrics)
// Step 4: Targeting (segment, environment, exclusion group)
// Step 5: Stats Config (engine, significance level, sequential testing)
// Step 6: Review & Create

export function ExperimentWizard() {
  const [step, setStep] = useState(1);
  const [data, setData] = useState<ExperimentCreate>(defaults);

  return (
    <WizardLayout currentStep={step} totalSteps={6}>
      {step === 1 && <BasicsStep data={data} onChange={setData} onNext={() => setStep(2)} />}
      {step === 2 && <VariationsStep data={data} onChange={setData} onNext={() => setStep(3)} onBack={() => setStep(1)} />}
      {step === 3 && <MetricsStep data={data} onChange={setData} onNext={() => setStep(4)} onBack={() => setStep(2)} />}
      {step === 4 && <TargetingStep data={data} onChange={setData} onNext={() => setStep(5)} onBack={() => setStep(3)} />}
      {step === 5 && <StatsStep data={data} onChange={setData} onNext={() => setStep(6)} onBack={() => setStep(4)} />}
      {step === 6 && <ReviewStep data={data} onBack={() => setStep(5)} onSubmit={createExperiment} />}
    </WizardLayout>
  );
}
```

```typescript
// frontend/src/pages/ExperimentResultsPage.tsx
export function ExperimentResultsPage() {
  const { experimentId } = useParams();
  const { data: results } = useExperimentResults(experimentId);

  return (
    <div>
      <ResultsSummaryBar
        totalUsers={results?.snapshot?.summary?.total_users}
        srmDetected={results?.snapshot?.summary?.srm?.detected}
        isStale={results?.is_stale}
        onRefresh={() => refreshResults(experimentId)}
      />

      {results?.snapshot?.summary?.srm?.detected && (
        <SRMWarningBanner pValue={results.snapshot.summary.srm.p_value} />
      )}

      <MetricResultsTable
        variations={results?.experiment?.variations}
        results={results?.snapshot?.results}
        primaryMetricId={getPrimaryMetric(results?.experiment)}
      />

      <ConfidenceIntervalChart results={results?.snapshot?.results} />
    </div>
  );
}
```

**Testing:**
- Wizard enforces step completion (cannot skip to step 6 without filling steps 1-5)
- Variation weights slider updates in real-time and enforces 100% total
- Metric search/select populates from the metric registry
- Start button only enabled when experiment passes validation
- Start/Pause/Stop buttons reflect current lifecycle state
- Results page shows per-variation metrics in a table with CI bars
- SRM warning banner appears when SRM is detected (red alert)
- Refresh button triggers snapshot recomputation and shows loading state
- Stale results indicator appears when snapshot is > 1 hour old
- Results table correctly highlights statistically significant results (green/red)

### Task 5.4: Metrics & Data Sources UI

**What:** Build the metric registry management pages and data source configuration (connection setup, fact table definition, test query execution).

**Design:**

```typescript
// Metric list page with filtering by type, tag, status
// Metric detail page showing: definition, config, usage (which experiments use this metric)
// Data source connection form: select type -> enter credentials -> test connection -> save
// Fact table editor: SQL query input with column preview
```

**Testing:**
- Metric creation form validates required fields by type (binomial vs revenue vs ratio)
- Metric detail shows which experiments reference it
- Data source form masks sensitive credentials
- "Test Connection" button verifies warehouse reachability and shows success/error
- Fact table SQL input shows column preview after executing a LIMIT 10 query
- Assignment query validation ensures it returns required columns (user_id, timestamp, experiment_id, variation_id)

### Task 5.5: API Client Generation

**What:** Auto-generate the TypeScript API client from the FastAPI OpenAPI schema. Set up React Query hooks for data fetching, caching, and mutation.

**Design:**

```typescript
// frontend/src/api/generated/  -- auto-generated by openapi-typescript-codegen
// frontend/src/hooks/useExperiments.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { ExperimentsApi } from "../api/generated";

export function useExperiments(projectId: string) {
  return useQuery({
    queryKey: ["experiments", projectId],
    queryFn: () => ExperimentsApi.list(projectId),
  });
}

export function useCreateExperiment(projectId: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: ExperimentCreate) => ExperimentsApi.create(projectId, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["experiments", projectId] });
    },
  });
}
```

**Testing:**
- Generated client types match the Pydantic schemas (compile-time type checking)
- React Query caches prevent duplicate network requests
- Mutation success invalidates related query caches (list refreshes after create/update)
- Error responses are typed and display user-friendly messages
- Loading states render skeleton components

### Definition of Done -- Phase 5
- [ ] Users can log in, create organizations, create projects
- [ ] Feature flags can be created, toggled, and configured with targeting rules via UI
- [ ] Experiments can be created via multi-step wizard and started/stopped
- [ ] Experiment results display with per-metric statistics, CI visualization, and SRM warnings
- [ ] Metrics can be defined, tagged, and linked to experiments
- [ ] Data source connections can be configured and tested from the UI
- [ ] All pages are responsive and functional on desktop (1024px+)
- [ ] Frontend tests cover critical user flows (create flag, start experiment, view results)

---

## Phase 6: Warehouse Connectors (BigQuery & Snowflake)

**Goal:** Implement production-grade warehouse connectors for BigQuery and Snowflake, enabling real-world experiment analysis on the two most common data warehouses.

**Duration estimate:** 3 weeks

### Task 6.1: BigQuery Connector

**What:** Implement the BigQuery warehouse connector using the google-cloud-bigquery Python library. Handle authentication via service account JSON, dataset/table discovery, and experiment query generation with BigQuery-specific SQL dialect.

**Design:**

```python
# backend/app/warehouse/bigquery.py
from google.cloud import bigquery
from google.oauth2 import service_account
from app.warehouse.base import WarehouseConnector, QueryResult
import json

class BigQueryConnector(WarehouseConnector):
    def __init__(self, connection_params: dict):
        creds_json = connection_params["service_account_json"]
        credentials = service_account.Credentials.from_service_account_info(
            json.loads(creds_json)
        )
        self.client = bigquery.Client(
            project=connection_params["project"],
            credentials=credentials,
        )
        self.dataset = connection_params.get("dataset")

    async def test_connection(self) -> bool:
        try:
            query = "SELECT 1 AS test"
            result = self.client.query(query).result()
            return True
        except Exception:
            return False

    async def execute_query(self, sql: str, params: dict | None = None) -> QueryResult:
        import time
        start = time.monotonic()
        job_config = bigquery.QueryJobConfig()
        if params:
            job_config.query_parameters = [
                bigquery.ScalarQueryParameter(k, "STRING", v)
                for k, v in params.items()
            ]
        query_job = self.client.query(sql, job_config=job_config)
        rows = list(query_job.result())
        elapsed = (time.monotonic() - start) * 1000

        return QueryResult(
            columns=[field.name for field in query_job.schema],
            rows=[list(row.values()) for row in rows],
            row_count=len(rows),
            execution_time_ms=elapsed,
        )

    def generate_experiment_query(
        self, assignment_sql, metric_sql, experiment_id, start_date, end_date, conversion_window_hours
    ) -> str:
        return f"""
        WITH assignments AS (
            SELECT user_id, variation_id, timestamp AS assigned_at
            FROM ({assignment_sql})
            WHERE experiment_id = '{experiment_id}'
              AND timestamp >= TIMESTAMP('{start_date}')
              AND timestamp <= TIMESTAMP('{end_date}')
        ),
        metrics AS (
            SELECT user_id, timestamp AS event_at, value
            FROM ({metric_sql})
        ),
        joined AS (
            SELECT
                a.user_id,
                a.variation_id,
                m.value
            FROM assignments a
            LEFT JOIN metrics m
              ON a.user_id = m.user_id
              AND m.event_at >= a.assigned_at
              AND m.event_at <= TIMESTAMP_ADD(a.assigned_at, INTERVAL {conversion_window_hours} HOUR)
        )
        SELECT
            variation_id,
            COUNT(DISTINCT user_id) AS users,
            AVG(IFNULL(value, 0)) AS mean,
            STDDEV(IFNULL(value, 0)) AS stddev,
            SUM(IFNULL(value, 0)) AS total_value
        FROM joined
        GROUP BY variation_id
        """
```

**Testing:**
- Connection test succeeds with valid service account credentials
- Connection test fails gracefully with invalid credentials (descriptive error)
- Table discovery lists tables in the configured dataset
- Column introspection returns BigQuery types (STRING, INT64, FLOAT64, TIMESTAMP)
- Generated SQL uses BigQuery dialect (TIMESTAMP(), TIMESTAMP_ADD(), IFNULL())
- Query execution returns correct results against a test dataset
- Large result sets (>100K rows) complete within 60 seconds
- Connection params (service account JSON) never appear in logs or error messages

### Task 6.2: Snowflake Connector

**What:** Implement the Snowflake warehouse connector using the snowflake-connector-python library. Handle key-pair authentication, warehouse/database/schema selection, and Snowflake SQL dialect.

**Design:**

```python
# backend/app/warehouse/snowflake.py
import snowflake.connector
from app.warehouse.base import WarehouseConnector, QueryResult

class SnowflakeConnector(WarehouseConnector):
    def __init__(self, connection_params: dict):
        self.conn_params = {
            "account": connection_params["account"],
            "user": connection_params["user"],
            "password": connection_params.get("password"),
            "private_key": connection_params.get("private_key"),
            "warehouse": connection_params.get("warehouse"),
            "database": connection_params.get("database"),
            "schema": connection_params.get("schema", "PUBLIC"),
            "role": connection_params.get("role"),
        }

    async def test_connection(self) -> bool:
        try:
            conn = snowflake.connector.connect(**self.conn_params)
            conn.cursor().execute("SELECT 1")
            conn.close()
            return True
        except Exception:
            return False

    def generate_experiment_query(
        self, assignment_sql, metric_sql, experiment_id, start_date, end_date, conversion_window_hours
    ) -> str:
        return f"""
        WITH assignments AS (
            SELECT user_id, variation_id, timestamp AS assigned_at
            FROM ({assignment_sql})
            WHERE experiment_id = '{experiment_id}'
              AND timestamp >= '{start_date}'::TIMESTAMP_TZ
              AND timestamp <= '{end_date}'::TIMESTAMP_TZ
        ),
        metrics AS (
            SELECT user_id, timestamp AS event_at, value
            FROM ({metric_sql})
        ),
        joined AS (
            SELECT
                a.user_id,
                a.variation_id,
                m.value
            FROM assignments a
            LEFT JOIN metrics m
              ON a.user_id = m.user_id
              AND m.event_at >= a.assigned_at
              AND m.event_at <= DATEADD('hour', {conversion_window_hours}, a.assigned_at)
        )
        SELECT
            variation_id,
            COUNT(DISTINCT user_id) AS users,
            AVG(NVL(value, 0)) AS mean,
            STDDEV(NVL(value, 0)) AS stddev,
            SUM(NVL(value, 0)) AS total_value
        FROM joined
        GROUP BY variation_id
        """
```

**Testing:**
- Connection test with valid Snowflake credentials succeeds
- Snowflake-specific SQL dialect (DATEADD, NVL, ::TIMESTAMP_TZ) generates valid queries
- Warehouse/database/schema configuration applied correctly
- Query timeout handling for long-running warehouse queries
- Connection pooling does not leak connections

### Task 6.3: Connector Registry & Factory

**What:** Build the connector factory that instantiates the correct warehouse connector based on the data source type, with connection pooling and credential decryption.

**Design:**

```python
# backend/app/warehouse/factory.py
from app.warehouse.base import WarehouseConnector
from app.warehouse.postgres import PostgresConnector
from app.warehouse.bigquery import BigQueryConnector
from app.warehouse.snowflake import SnowflakeConnector
from app.utils.encryption import decrypt_connection_params

CONNECTORS = {
    "postgres": PostgresConnector,
    "bigquery": BigQueryConnector,
    "snowflake": SnowflakeConnector,
}

def get_connector(data_source) -> WarehouseConnector:
    connector_class = CONNECTORS.get(data_source.type)
    if not connector_class:
        raise ValueError(f"Unsupported data source type: {data_source.type}")
    params = decrypt_connection_params(data_source.connection_params_encrypted)
    return connector_class(params)
```

**Testing:**
- Factory returns correct connector class for each type string
- Unknown type raises descriptive ValueError
- Credentials decrypted before passing to connector
- Integration test: create data source, get connector, execute test query

### Definition of Done -- Phase 6
- [ ] BigQuery connector executes experiment queries against real BigQuery datasets
- [ ] Snowflake connector executes experiment queries against real Snowflake warehouses
- [ ] Both connectors produce identical result structures for the same logical data
- [ ] Connection credentials encrypted at rest and decrypted only in memory
- [ ] SQL dialect differences handled transparently (TIMESTAMP_ADD vs DATEADD, IFNULL vs NVL)
- [ ] Error messages from warehouse failures are descriptive without leaking credentials
- [ ] End-to-end test: create experiment, connect warehouse, run snapshot, view results

---

## Phase 7: Advanced Statistics

**Goal:** Implement the advanced statistical methods that differentiate this platform: CUPED variance reduction, Sequential testing (SPRT), Bayesian inference engine, and power analysis.

**Duration estimate:** 4 weeks

### Task 7.1: CUPED Variance Reduction

**What:** Implement CUPED (Controlled-experiment Using Pre-Experiment Data) to reduce experiment variance by 30-50%, enabling smaller sample sizes or shorter test durations.

**Design:**

```python
# backend/app/stats/cuped.py
import numpy as np
from dataclasses import dataclass

@dataclass
class CUPEDResult:
    adjusted_mean: float
    adjusted_stddev: float
    variance_reduction_pct: float  # how much variance was reduced
    theta: float                   # CUPED coefficient

class CUPEDEngine:
    """CUPED: Controlled-experiment Using Pre-Experiment Data.

    Uses pre-experiment covariate data (e.g., same metric measured before experiment)
    to reduce variance in the treatment effect estimate.

    Formula: Y_adjusted = Y - theta * (X - E[X])
    where theta = Cov(X, Y) / Var(X)
    """

    def adjust(
        self,
        post_values: np.ndarray,      # Y: metric values during experiment
        pre_values: np.ndarray,       # X: same metric values before experiment
    ) -> CUPEDResult:
        if len(post_values) != len(pre_values):
            raise ValueError("Pre and post arrays must have the same length")

        # Compute theta (regression coefficient)
        cov_xy = np.cov(pre_values, post_values, ddof=1)[0, 1]
        var_x = np.var(pre_values, ddof=1)

        if var_x == 0:
            # No variance in pre-experiment data; CUPED cannot help
            return CUPEDResult(
                adjusted_mean=float(np.mean(post_values)),
                adjusted_stddev=float(np.std(post_values, ddof=1)),
                variance_reduction_pct=0.0,
                theta=0.0,
            )

        theta = cov_xy / var_x

        # Adjust values
        mean_pre = np.mean(pre_values)
        adjusted = post_values - theta * (pre_values - mean_pre)

        original_var = np.var(post_values, ddof=1)
        adjusted_var = np.var(adjusted, ddof=1)
        reduction = (1 - adjusted_var / original_var) * 100 if original_var > 0 else 0

        return CUPEDResult(
            adjusted_mean=float(np.mean(adjusted)),
            adjusted_stddev=float(np.std(adjusted, ddof=1)),
            variance_reduction_pct=float(reduction),
            theta=float(theta),
        )
```

**Testing:**
- Synthetic data with known correlation: CUPED reduces variance by expected amount
- High correlation (r=0.8): variance reduction > 50%
- Zero correlation (r=0): no variance reduction, adjusted mean equals unadjusted mean
- Negative correlation: CUPED still reduces variance (theta is negative)
- Pre and post arrays of different lengths raise ValueError
- Zero-variance pre-data: graceful fallback (no adjustment)
- End-to-end: experiment with CUPED enabled produces adjusted results in snapshot

### Task 7.2: Sequential Testing (SPRT)

**What:** Implement Sequential Probability Ratio Test for valid early stopping without inflating false-positive rates. Supports continuous monitoring of p-values with always-valid confidence intervals.

**Design:**

```python
# backend/app/stats/sequential.py
import numpy as np
from scipy import stats as scipy_stats
from dataclasses import dataclass

@dataclass
class SequentialResult:
    is_significant: bool
    adjusted_p_value: float      # alpha-spending adjusted p-value
    adjusted_ci_lower: float
    adjusted_ci_upper: float
    information_fraction: float  # how much of planned sample collected (0-1)
    boundary_crossed: str | None  # "reject_null" | "accept_null" | None

class SequentialTestEngine:
    """Always-valid sequential testing using the mixture sequential probability ratio test.

    Uses an alpha-spending function to control the Type I error rate
    across multiple interim analyses.
    """

    def __init__(
        self,
        alpha: float = 0.05,
        tuning_parameter: float = 0.0001,  # rho: controls spending rate
    ):
        self.alpha = alpha
        self.rho = tuning_parameter

    def test(
        self,
        control_users: int,
        control_mean: float,
        control_var: float,
        treatment_users: int,
        treatment_mean: float,
        treatment_var: float,
        planned_sample_size: int,
    ) -> SequentialResult:
        n = control_users + treatment_users
        info_fraction = min(n / planned_sample_size, 1.0)

        # Alpha spending (O'Brien-Fleming-like)
        alpha_spent = self._alpha_spending(info_fraction)

        # Standard test statistic
        se = np.sqrt(control_var / control_users + treatment_var / treatment_users)
        if se == 0:
            return SequentialResult(
                is_significant=False,
                adjusted_p_value=1.0,
                adjusted_ci_lower=0.0,
                adjusted_ci_upper=0.0,
                information_fraction=info_fraction,
                boundary_crossed=None,
            )

        z = (treatment_mean - control_mean) / se
        raw_p = 2 * (1 - scipy_stats.norm.cdf(abs(z)))

        # Adjusted critical value
        z_crit = scipy_stats.norm.ppf(1 - alpha_spent / 2)
        is_significant = abs(z) > z_crit

        # Always-valid CI (wider than fixed-sample CI)
        diff = treatment_mean - control_mean
        ci_width = z_crit * se

        return SequentialResult(
            is_significant=is_significant,
            adjusted_p_value=raw_p,  # reported alongside the boundary
            adjusted_ci_lower=diff - ci_width,
            adjusted_ci_upper=diff + ci_width,
            information_fraction=info_fraction,
            boundary_crossed="reject_null" if is_significant else None,
        )

    def _alpha_spending(self, info_fraction: float) -> float:
        """O'Brien-Fleming-like alpha-spending function."""
        if info_fraction <= 0:
            return 0.0
        z = scipy_stats.norm.ppf(1 - self.alpha / 2)
        return 2 * (1 - scipy_stats.norm.cdf(z / np.sqrt(info_fraction)))
```

**Testing:**
- At full sample size (info_fraction=1.0), results match fixed-sample t-test
- Early stopping: true effect detected before full sample with valid Type I error rate
- Simulated null (no true effect): false positive rate <= alpha across 10,000 simulations
- Alpha-spending is conservative early (high info_fraction needed to reject at 20% of sample)
- Information fraction correctly computed as current_n / planned_n
- Tuning parameter rho affects sensitivity of early stopping
- Boundary crossing reported correctly in result object
- Zero standard error handled gracefully (no division by zero)

### Task 7.3: Bayesian Statistics Engine

**What:** Implement Bayesian A/B testing with conjugate priors, posterior computation, chance-to-beat-control, and expected loss calculations.

**Design:**

```python
# backend/app/stats/bayesian.py
import numpy as np
from scipy import stats as scipy_stats
from dataclasses import dataclass

@dataclass
class BayesianResult:
    chance_to_beat_control: float  # P(treatment > control)
    expected_loss: float           # E[max(control - treatment, 0)]
    credible_interval: tuple[float, float]  # 95% HDI
    posterior_mean: float
    posterior_stddev: float

class BayesianEngine:
    """Bayesian A/B testing using conjugate priors.

    Binomial metrics: Beta-Binomial model
    Continuous metrics: Normal-Normal model (known variance approximation)
    """
    N_SAMPLES = 100_000

    def compute_binomial(
        self,
        control_users: int, control_conversions: int,
        treatment_users: int, treatment_conversions: int,
        prior_alpha: float = 1.0, prior_beta: float = 1.0,  # Uniform prior
    ) -> BayesianResult:
        # Posterior distributions: Beta(alpha + successes, beta + failures)
        control_posterior = scipy_stats.beta(
            prior_alpha + control_conversions,
            prior_beta + control_users - control_conversions,
        )
        treatment_posterior = scipy_stats.beta(
            prior_alpha + treatment_conversions,
            prior_beta + treatment_users - treatment_conversions,
        )

        # Monte Carlo estimation
        control_samples = control_posterior.rvs(self.N_SAMPLES)
        treatment_samples = treatment_posterior.rvs(self.N_SAMPLES)

        chance_to_beat = float(np.mean(treatment_samples > control_samples))
        loss = float(np.mean(np.maximum(control_samples - treatment_samples, 0)))

        # Credible interval (95% HDI of treatment posterior)
        ci = (
            float(treatment_posterior.ppf(0.025)),
            float(treatment_posterior.ppf(0.975)),
        )

        return BayesianResult(
            chance_to_beat_control=chance_to_beat,
            expected_loss=loss,
            credible_interval=ci,
            posterior_mean=float(treatment_posterior.mean()),
            posterior_stddev=float(treatment_posterior.std()),
        )

    def compute_continuous(
        self,
        control_users: int, control_mean: float, control_stddev: float,
        treatment_users: int, treatment_mean: float, treatment_stddev: float,
    ) -> BayesianResult:
        # Normal approximation with known variance
        control_se = control_stddev / np.sqrt(control_users)
        treatment_se = treatment_stddev / np.sqrt(treatment_users)

        control_samples = np.random.normal(control_mean, control_se, self.N_SAMPLES)
        treatment_samples = np.random.normal(treatment_mean, treatment_se, self.N_SAMPLES)

        chance_to_beat = float(np.mean(treatment_samples > control_samples))
        loss = float(np.mean(np.maximum(control_samples - treatment_samples, 0)))

        ci = (
            float(np.percentile(treatment_samples, 2.5)),
            float(np.percentile(treatment_samples, 97.5)),
        )

        return BayesianResult(
            chance_to_beat_control=chance_to_beat,
            expected_loss=loss,
            credible_interval=ci,
            posterior_mean=float(np.mean(treatment_samples)),
            posterior_stddev=float(np.std(treatment_samples)),
        )
```

**Testing:**
- Binomial: 10% vs 12% conversion rate with n=5000 -> chance_to_beat > 0.95
- Binomial: identical rates -> chance_to_beat ~ 0.50 (+/- 0.02)
- Continuous: treatment mean significantly higher -> chance_to_beat > 0.95
- Expected loss < 0.001 when treatment clearly wins
- Credible interval contains true parameter in 95% of simulated experiments
- Uniform prior (alpha=1, beta=1): posterior driven by data
- Informative prior with small data: prior influences posterior
- Large sample: Bayesian and frequentist results converge
- Reproducibility: setting random seed produces identical results

### Task 7.4: Power Analysis & Sample Size Calculator

**What:** Build the pre-experiment power analysis tool that estimates the required sample size given baseline rate, minimum detectable effect, significance level, and power.

**Design:**

```python
# backend/app/stats/power.py
from scipy import stats as scipy_stats
import math

class PowerAnalysis:
    def sample_size_binomial(
        self,
        baseline_rate: float,          # e.g., 0.10 (10%)
        minimum_detectable_effect: float,  # relative: e.g., 0.05 (5% relative lift)
        alpha: float = 0.05,
        power: float = 0.80,
        num_variations: int = 2,
    ) -> dict:
        """Calculate required sample size per variation for a binomial metric."""
        p1 = baseline_rate
        p2 = baseline_rate * (1 + minimum_detectable_effect)

        z_alpha = scipy_stats.norm.ppf(1 - alpha / 2)
        z_beta = scipy_stats.norm.ppf(power)

        # Applying Bonferroni correction for multiple variations
        if num_variations > 2:
            z_alpha = scipy_stats.norm.ppf(1 - alpha / (2 * (num_variations - 1)))

        n = ((z_alpha * math.sqrt(2 * p1 * (1 - p1)) +
              z_beta * math.sqrt(p1 * (1 - p1) + p2 * (1 - p2)))**2) / (p2 - p1)**2

        n_per_variation = math.ceil(n)
        total = n_per_variation * num_variations

        return {
            "sample_size_per_variation": n_per_variation,
            "total_sample_size": total,
            "baseline_rate": p1,
            "expected_rate": p2,
            "minimum_detectable_effect": minimum_detectable_effect,
            "alpha": alpha,
            "power": power,
        }

    def estimated_duration_days(
        self,
        total_sample_size: int,
        daily_traffic: int,
        traffic_percentage: float = 1.0,  # fraction of traffic in experiment
    ) -> int:
        """Estimate days to reach required sample size."""
        effective_daily = daily_traffic * traffic_percentage
        if effective_daily <= 0:
            return -1
        return math.ceil(total_sample_size / effective_daily)
```

**Testing:**
- 10% baseline, 5% relative MDE, 80% power -> ~31,000 per variation (validate against standard tables)
- Higher power (90%) requires larger sample size
- Smaller MDE requires larger sample size
- Duration estimate: 62,000 total / 10,000 daily = 7 days
- Bonferroni correction increases sample size for 3+ variations
- Edge cases: 0% baseline rate, 0 daily traffic (handle gracefully)
- Results match online sample size calculators (Evan Miller, etc.)

### Definition of Done -- Phase 7
- [ ] CUPED reduces variance by 30-50% on synthetic data with known correlation
- [ ] Sequential testing maintains valid Type I error rate across continuous monitoring
- [ ] Bayesian engine produces chance-to-beat-control and expected loss
- [ ] Power analysis sample size matches published statistical tables
- [ ] Statistics engine selection (frequentist/bayesian) configurable per experiment
- [ ] CUPED enabled per metric via covariate configuration
- [ ] Sequential testing results include information fraction and boundary status
- [ ] Frontend displays Bayesian results (chance-to-beat, credible intervals) when selected
- [ ] All statistical methods validated against reference implementations (scipy, R)

---

## Phase 8: Multi-Armed Bandit & Adaptive Allocation

**Goal:** Implement multi-armed bandit experiments that dynamically allocate traffic to better-performing variations, maximizing business value during the test window.

**Duration estimate:** 3 weeks

### Task 8.1: Thompson Sampling Implementation

**What:** Implement Thompson sampling for adaptive traffic allocation. The algorithm samples from each variation's posterior distribution and allocates more traffic to variations that are likely better.

**Design:**

```python
# backend/app/stats/bandit.py
import numpy as np
from dataclasses import dataclass

@dataclass
class BanditAllocation:
    variation_key: str
    new_weight: int        # basis points (0-10000)
    reward_estimate: float
    sample_count: int

class ThompsonSampling:
    """Thompson Sampling for multi-armed bandit experiments.

    Uses Beta-Bernoulli model for binomial rewards.
    """

    def __init__(
        self,
        min_weight: int = 100,       # minimum basis points per variation (1%)
        prior_alpha: float = 1.0,
        prior_beta: float = 1.0,
    ):
        self.min_weight = min_weight
        self.prior_alpha = prior_alpha
        self.prior_beta = prior_beta

    def compute_allocations(
        self,
        variations: list[dict],  # [{"key": "control", "users": 5000, "conversions": 500}, ...]
    ) -> list[BanditAllocation]:
        n_variations = len(variations)
        n_samples = 10_000

        # Sample from posterior for each variation
        samples = {}
        for v in variations:
            alpha = self.prior_alpha + v.get("conversions", 0)
            beta = self.prior_beta + v.get("users", 0) - v.get("conversions", 0)
            samples[v["key"]] = np.random.beta(alpha, beta, n_samples)

        # Probability of being best
        all_samples = np.stack([samples[v["key"]] for v in variations], axis=1)
        best_indices = np.argmax(all_samples, axis=1)

        allocations = []
        remaining_weight = 10000 - (self.min_weight * n_variations)

        for i, v in enumerate(variations):
            prob_best = float(np.mean(best_indices == i))
            extra_weight = int(prob_best * remaining_weight)
            total_weight = self.min_weight + extra_weight

            allocations.append(BanditAllocation(
                variation_key=v["key"],
                new_weight=total_weight,
                reward_estimate=float(np.mean(samples[v["key"]])),
                sample_count=v.get("users", 0),
            ))

        # Normalize to exactly 10000
        total = sum(a.new_weight for a in allocations)
        if total != 10000:
            diff = 10000 - total
            allocations[0].new_weight += diff

        return allocations
```

**Testing:**
- Two variations with equal performance: approximately equal weights (~5000 each)
- Clear winner (10% vs 5%): winner gets majority of traffic (>7000 basis points)
- Every variation gets at least min_weight (100 basis points = 1%)
- Weights always sum to exactly 10000
- With zero data: weights are approximately equal (uniform prior)
- Increasing data for winner: weight allocation becomes more extreme
- Convergence: after enough data, nearly all traffic goes to winner

### Task 8.2: Bandit Scheduler & Weight Updates

**What:** Build the Celery task that periodically recomputes bandit weights for running bandit experiments and updates the experiment's variation weights.

**Design:**

```python
# backend/app/tasks/bandit_updater.py
from celery import shared_task
from app.stats.bandit import ThompsonSampling

@shared_task
def update_bandit_weights(experiment_id: str):
    experiment = db.get_experiment(experiment_id)
    if experiment.type != "bandit" or experiment.status != "running":
        return

    config = experiment.bandit_config
    if not config:
        return

    # Check burn-in period
    hours_running = (utcnow() - experiment.started_at).total_seconds() / 3600
    if hours_running < config.get("burn_in_period_hours", 24):
        return

    # Get current results
    snapshot = db.get_latest_snapshot(experiment_id)
    if not snapshot:
        return

    # Compute new allocations
    engine = ThompsonSampling(
        min_weight=config.get("min_weight", 100),
    )

    variation_data = []
    for v in experiment.variations:
        v_results = find_variation_results(snapshot, v["key"])
        variation_data.append({
            "key": v["key"],
            "users": v_results.get("users", 0),
            "conversions": v_results.get("conversions", 0),
        })

    allocations = engine.compute_allocations(variation_data)

    # Update experiment variation weights
    new_variations = []
    for v in experiment.variations:
        alloc = next(a for a in allocations if a.variation_key == v["key"])
        new_variations.append({**v, "weight": alloc.new_weight})

    experiment.variations = new_variations
    experiment.revision += 1
    db.save_experiment(experiment)

    # Log weight update in snapshot
    db.create_bandit_update_log(experiment_id, allocations)

    # Invalidate SDK cache
    invalidate_sdk_cache(experiment.project_id)
```

**Testing:**
- Bandit weights not updated during burn-in period
- After burn-in: weights updated according to Thompson sampling
- Weight updates logged for historical tracking
- SDK cache invalidated after weight update
- Celery Beat schedules updates at configured cadence (default 60 minutes)
- Non-bandit experiments are skipped
- Paused experiments are skipped
- Weight update history visible in experiment results UI

### Task 8.3: Bandit UI & Visualization

**What:** Add bandit-specific UI elements: weight history chart showing traffic allocation over time, reward estimate per variation, and burn-in period indicator.

**Design:**

```typescript
// frontend/src/components/experiments/BanditDashboard.tsx
// - Line chart: traffic allocation (weights) over time for each variation
// - Current allocation bars showing percentage per variation
// - Reward estimate table with confidence
// - Burn-in countdown timer
// - Toggle: "Lock weights" to freeze allocation (converts to fixed-allocation test)
```

**Testing:**
- Weight history chart shows allocation changes over time
- Current allocation bars update after each weight recomputation
- Burn-in period displayed as countdown timer
- "Lock weights" converts bandit to fixed experiment (confirmation dialog)
- Bandit creation wizard includes algorithm selection and burn-in configuration

### Definition of Done -- Phase 8
- [ ] Thompson sampling produces valid probability-proportional allocations
- [ ] Every variation receives at least minimum weight (no variation starved)
- [ ] Weights updated at configured cadence after burn-in period
- [ ] Weight history tracked and visualized over time
- [ ] SDK receives updated weights after each recomputation
- [ ] Bandit experiments can be converted to fixed-allocation experiments
- [ ] Burn-in period prevents premature optimization on insufficient data

---

## Phase 9: AI-Powered Features

**Goal:** Implement the AI-native capabilities that represent the platform's primary differentiation: hypothesis generation from telemetry, automated SRM root-cause analysis, and natural language experiment authoring.

**Duration estimate:** 4 weeks

### Task 9.1: AI Hypothesis Generation

**What:** Build an LLM-powered system that analyzes product telemetry (funnel drop-offs, retention curves, feature usage patterns) and generates experiment hypothesis suggestions ranked by estimated impact.

**Design:**

```python
# backend/app/ai/hypothesis_generator.py
from anthropic import Anthropic

class HypothesisGenerator:
    SYSTEM_PROMPT = """You are an experimentation expert analyzing product data.
    Given funnel metrics, user behavior patterns, and product telemetry,
    generate specific, testable A/B experiment hypotheses.

    For each hypothesis, provide:
    1. A clear hypothesis statement
    2. The specific change to test
    3. The primary metric to measure
    4. Estimated impact (low/medium/high) with reasoning
    5. Required sample size estimate
    6. Potential risks or guardrail metrics to monitor
    """

    async def generate(
        self,
        project_id: str,
        telemetry_summary: dict,  # funnel data, retention, feature usage
        existing_experiments: list[dict],  # avoid duplicating past experiments
        max_hypotheses: int = 5,
    ) -> list[dict]:
        client = Anthropic()

        prompt = self._build_prompt(telemetry_summary, existing_experiments)

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4000,
            system=self.SYSTEM_PROMPT,
            messages=[{"role": "user", "content": prompt}],
        )

        return self._parse_hypotheses(response.content[0].text)

    def _build_prompt(self, telemetry, existing) -> str:
        return f"""
        Analyze the following product telemetry and suggest experiment hypotheses:

        ## Funnel Data
        {self._format_funnel(telemetry.get('funnels', []))}

        ## Feature Usage
        {self._format_usage(telemetry.get('feature_usage', []))}

        ## Retention Curves
        {self._format_retention(telemetry.get('retention', {}))}

        ## Past Experiments (avoid duplicating)
        {self._format_experiments(existing)}

        Generate {5} hypotheses ranked by estimated impact.
        """
```

**Testing:**
- Given funnel data with a significant drop-off, generates hypotheses targeting the drop-off step
- Hypotheses include measurable metrics and estimated impact
- Past experiments are not duplicated
- Response parsed into structured hypothesis objects
- API error (rate limit, timeout) returns graceful error to user
- Hypotheses stored and linked to project for future reference

### Task 9.2: Automated SRM Root-Cause Analysis

**What:** When SRM is detected, automatically analyze potential causes (bot traffic, SDK misconfiguration, browser-specific issues, geographic bias) and present likely root causes to the user.

**Design:**

```python
# backend/app/ai/srm_analyzer.py
class SRMRootCauseAnalyzer:
    """Automated root-cause analysis for Sample Ratio Mismatch."""

    async def analyze(
        self,
        experiment_id: str,
        snapshot: dict,
        assignment_data: dict,  # raw assignment data from warehouse
    ) -> list[dict]:
        causes = []

        # 1. Check for temporal patterns
        temporal = await self._check_temporal_drift(assignment_data)
        if temporal["detected"]:
            causes.append({
                "cause": "Temporal assignment drift",
                "confidence": temporal["confidence"],
                "description": "Assignment ratios changed over time, suggesting a deployment or configuration change",
                "evidence": temporal["details"],
            })

        # 2. Check for browser/platform skew
        platform = await self._check_platform_skew(assignment_data)
        if platform["detected"]:
            causes.append({
                "cause": "Platform-specific skew",
                "confidence": platform["confidence"],
                "description": f"Assignment is significantly skewed on {platform['affected_platform']}",
                "evidence": platform["details"],
            })

        # 3. Check for bot traffic
        bot = await self._check_bot_traffic(assignment_data)
        if bot["detected"]:
            causes.append({
                "cause": "Bot traffic",
                "confidence": bot["confidence"],
                "description": f"Estimated {bot['bot_pct']:.1f}% of assignments are from bot traffic",
                "evidence": bot["details"],
            })

        # 4. Check for redirect issues (split URL tests)
        # 5. Check for early exits before tracking fires

        return sorted(causes, key=lambda c: c["confidence"], reverse=True)
```

**Testing:**
- Temporal drift detection: assignment ratio changes mid-experiment flagged correctly
- Platform skew: iOS-only SRM detected when Chrome assignments are balanced
- Bot traffic: high-frequency single-user assignments identified as bot behavior
- Multiple causes: all detected causes returned, sorted by confidence
- No SRM: analyzer returns empty list
- Results displayed in the experiment results page as actionable cards

### Task 9.3: Natural Language Experiment Authoring

**What:** Allow users to describe an experiment in plain language and have the system generate the full experiment configuration (metrics, targeting, variations, stats settings).

**Design:**

```python
# backend/app/ai/experiment_author.py
class NaturalLanguageAuthor:
    SYSTEM_PROMPT = """You are an A/B testing configuration assistant.
    Given a natural language description of an experiment, generate a complete
    experiment configuration JSON that matches the platform's schema.

    Available metrics in this project: {metrics}
    Available segments: {segments}
    Available feature flags: {flags}
    """

    async def generate_config(
        self,
        project_id: str,
        description: str,  # "Test a new checkout button color. Blue vs green. Measure conversion rate."
    ) -> dict:
        # Fetch project context
        metrics = await self.db.list_metrics(project_id)
        segments = await self.db.list_segments(project_id)
        flags = await self.db.list_flags(project_id)

        client = Anthropic()
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2000,
            system=self.SYSTEM_PROMPT.format(
                metrics=self._format_metrics(metrics),
                segments=self._format_segments(segments),
                flags=self._format_flags(flags),
            ),
            messages=[{"role": "user", "content": description}],
        )

        config = self._parse_config(response.content[0].text)
        # Validate against Pydantic schema before returning
        validated = ExperimentCreate.model_validate(config)
        return validated.model_dump()
```

**Testing:**
- "Test a new checkout flow" generates experiment with relevant metrics (conversion rate)
- Generated config validates against ExperimentCreate schema
- Metric names matched to existing project metrics (fuzzy matching)
- Invalid description returns helpful error ("Could not determine variations")
- User can review and edit generated config before creating
- Generated hypothesis reflects the input description

### Definition of Done -- Phase 9
- [ ] AI hypothesis generator produces actionable, non-duplicate suggestions
- [ ] SRM root-cause analyzer identifies temporal drift, platform skew, and bot traffic
- [ ] Natural language experiment authoring generates valid experiment configurations
- [ ] All AI features accessible from the frontend dashboard
- [ ] AI features are optional (platform fully functional without them)
- [ ] LLM API errors handled gracefully with user-facing error messages
- [ ] AI-generated configurations require user review before activation

---

## Phase 10: Enterprise Features & Production Hardening

**Goal:** Implement the enterprise-grade features needed for production deployment: RBAC, mutual exclusion groups, global holdouts, Kubernetes deployment, and comprehensive monitoring.

**Duration estimate:** 4 weeks

### Task 10.1: Role-Based Access Control

**What:** Implement granular RBAC with predefined roles (admin, experimenter, analyst, viewer) and custom permission assignments. Enforce permissions on all API endpoints.

**Design:**

```python
# backend/app/services/rbac.py
from enum import Enum
from functools import wraps

class Permission(str, Enum):
    EXPERIMENT_CREATE = "experiment:create"
    EXPERIMENT_START = "experiment:start"
    EXPERIMENT_STOP = "experiment:stop"
    EXPERIMENT_READ = "experiment:read"
    FLAG_CREATE = "flag:create"
    FLAG_UPDATE = "flag:update"
    FLAG_TOGGLE = "flag:toggle"
    FLAG_READ = "flag:read"
    METRIC_MANAGE = "metric:manage"
    METRIC_READ = "metric:read"
    DATA_SOURCE_MANAGE = "data_source:manage"
    SETTINGS_MANAGE = "settings:manage"
    AUDIT_READ = "audit:read"

ROLE_PERMISSIONS = {
    "admin": list(Permission),
    "experimenter": [
        Permission.EXPERIMENT_CREATE, Permission.EXPERIMENT_START,
        Permission.EXPERIMENT_STOP, Permission.EXPERIMENT_READ,
        Permission.FLAG_CREATE, Permission.FLAG_UPDATE,
        Permission.FLAG_TOGGLE, Permission.FLAG_READ,
        Permission.METRIC_READ,
    ],
    "analyst": [
        Permission.EXPERIMENT_READ, Permission.FLAG_READ,
        Permission.METRIC_READ, Permission.AUDIT_READ,
    ],
    "viewer": [
        Permission.EXPERIMENT_READ, Permission.FLAG_READ,
        Permission.METRIC_READ,
    ],
}

def require_permission(permission: Permission):
    """FastAPI dependency that checks user has the required permission."""
    async def check(user=Depends(get_current_user), project_id: UUID = None):
        membership = await get_membership(user.id, project_id)
        if permission.value not in membership.permissions:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return Depends(check)
```

**Testing:**
- Admin can perform all actions
- Experimenter can create/start experiments but not manage data sources
- Analyst can view experiments and audit logs but not create/modify
- Viewer can only read (all mutation endpoints return 403)
- Custom permissions override role defaults
- Project-level role overrides organization-level role
- Unauthorized action returns 403 with descriptive message

### Task 10.2: Mutual Exclusion Groups

**What:** Implement mutual exclusion groups that prevent users from being in multiple conflicting experiments simultaneously by partitioning the traffic namespace.

**Design:**

```python
# backend/app/services/exclusion_group_service.py
class ExclusionGroupService:
    async def add_experiment(
        self,
        group_id: UUID,
        experiment_id: UUID,
        traffic_pct: int,  # basis points requested (e.g., 5000 = 50%)
    ) -> dict:
        group = await self._get(group_id)
        slots = group.slots or []

        # Find available range
        used_ranges = [(s["range_start"], s["range_end"]) for s in slots]
        available = self._find_available_range(used_ranges, traffic_pct)

        if available is None:
            raise ValidationError(
                f"Not enough traffic available. Requested {traffic_pct/100}%, "
                f"available: {self._available_pct(used_ranges)/100}%"
            )

        slots.append({
            "experiment_id": str(experiment_id),
            "range_start": available[0],
            "range_end": available[1],
        })
        group.slots = slots
        await self._save(group)

        # Update experiment targeting
        experiment = await self.experiment_service.get(experiment_id)
        targeting = dict(experiment.targeting)
        targeting["exclusion_group"] = {
            "id": str(group_id),
            "range_start": available[0],
            "range_end": available[1],
        }
        experiment.targeting = targeting
        await self.experiment_service.save(experiment)

    def _find_available_range(self, used: list[tuple], size: int) -> tuple | None:
        """Find first contiguous available range of given size."""
        sorted_ranges = sorted(used)
        start = 0
        for r_start, r_end in sorted_ranges:
            if r_start - start >= size:
                return (start, start + size)
            start = r_end
        if 10000 - start >= size:
            return (start, start + size)
        return None
```

**Testing:**
- Create exclusion group, add 2 experiments at 50% each -> full namespace allocated
- Adding a third experiment at 50% when full returns error with available percentage
- Users in experiment A's range never assigned to experiment B (verify with 10,000 hashes)
- Removing an experiment frees its namespace range
- Overlapping ranges prevented by validation
- Exclusion group visible in experiment detail page

### Task 10.3: Kubernetes Deployment & Monitoring

**What:** Create Helm chart for Kubernetes deployment, set up health checks, Prometheus metrics, and structured logging.

**Design:**

```python
# backend/app/middleware/metrics.py
from prometheus_client import Counter, Histogram, Gauge
import time

REQUEST_COUNT = Counter("http_requests_total", "Total requests", ["method", "path", "status"])
REQUEST_LATENCY = Histogram("http_request_duration_seconds", "Request latency", ["method", "path"])
ACTIVE_EXPERIMENTS = Gauge("active_experiments_total", "Currently running experiments")
SDK_PAYLOAD_REQUESTS = Counter("sdk_payload_requests_total", "SDK payload fetches", ["status"])
SNAPSHOT_DURATION = Histogram("snapshot_computation_seconds", "Time to compute experiment snapshot")

async def metrics_middleware(request, call_next):
    start = time.monotonic()
    response = await call_next(request)
    duration = time.monotonic() - start
    REQUEST_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    REQUEST_LATENCY.labels(request.method, request.url.path).observe(duration)
    return response
```

```yaml
# deploy/helm/values.yaml
replicaCount: 2
image:
  repository: abtesting/api
  tag: latest
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

**Testing:**
- Helm chart deploys to Kubernetes cluster without errors
- Health check endpoints respond within 100ms
- Prometheus metrics endpoint exposes all counters and histograms
- Auto-scaling triggers at 70% CPU utilization
- Rolling deployment with zero downtime (readiness probe prevents traffic to unready pods)
- Structured JSON logs include request ID, user ID, and timing

### Task 10.4: Python SDK

**What:** Build the server-side Python SDK with local evaluation, feature flag caching, and thread-safe operation for high-concurrency environments.

**Design:**

```python
# sdks/python/src/abtesting/client.py
import threading
import requests
from typing import Any, Optional
from abtesting.evaluator import Evaluator

class ABTestingClient:
    def __init__(
        self,
        api_key: str,
        api_host: str = "https://api.abtesting.dev",
        refresh_interval: int = 60,  # seconds
    ):
        self.api_key = api_key
        self.api_host = api_host
        self.refresh_interval = refresh_interval
        self._payload = None
        self._lock = threading.Lock()
        self._evaluator = Evaluator()
        self._refresh_thread = None

    def initialize(self) -> None:
        self._fetch_payload()
        self._start_refresh_loop()

    def get_feature_value(self, key: str, attributes: dict, default: Any = None) -> Any:
        with self._lock:
            if self._payload is None:
                return default
            feature = self._payload.get("features", {}).get(key)
            if not feature:
                return default
            result = self._evaluator.evaluate(feature, attributes, key)
            return result["value"]

    def is_on(self, key: str, attributes: dict) -> bool:
        value = self.get_feature_value(key, attributes, default=False)
        return value is True or value == "true"

    def destroy(self) -> None:
        if self._refresh_thread:
            self._stop_event.set()
            self._refresh_thread.join()
```

**Testing:**
- Client initializes and fetches payload
- `get_feature_value()` returns correct value without network call
- Thread safety: concurrent evaluations produce consistent results
- Refresh loop updates payload at configured interval
- `destroy()` stops refresh thread cleanly
- Offline operation: evaluates from cached payload when API unreachable
- Cross-language consistency: Python and JS produce identical evaluations for same inputs

### Definition of Done -- Phase 10
- [ ] RBAC enforces permissions on all API endpoints with 4 predefined roles
- [ ] Mutual exclusion groups prevent traffic overlap between conflicting experiments
- [ ] Helm chart deploys to Kubernetes with auto-scaling and zero-downtime updates
- [ ] Prometheus metrics track request count, latency, active experiments, and snapshot duration
- [ ] Python SDK provides thread-safe local flag evaluation
- [ ] Structured logging with request correlation IDs
- [ ] Load test: API handles 1000 req/s for SDK payload endpoint (p99 < 100ms)
- [ ] Security review: no credential leaks, SQL injection, or XSS vulnerabilities
- [ ] Documentation: API reference, SDK guides, deployment guide, and architecture overview

---

## Summary

| Phase | Name | Duration | Key Deliverables |
|-------|------|----------|------------------|
| 1 | Foundation & Infrastructure | 3 weeks | Project skeleton, DB schema, auth, CI/CD |
| 2 | Feature Flags Engine | 4 weeks | Flag CRUD, targeting rules, evaluation engine, SDK payload |
| 3 | Experiment Engine (Core) | 5 weeks | Experiment lifecycle, metrics, warehouse queries, frequentist stats |
| 4 | JavaScript SDK | 3 weeks | Client-side SDK with local evaluation and exposure tracking |
| 5 | Frontend Dashboard (MVP) | 5 weeks | React UI for flags, experiments, metrics, and results |
| 6 | Warehouse Connectors | 3 weeks | BigQuery and Snowflake production connectors |
| 7 | Advanced Statistics | 4 weeks | CUPED, sequential testing, Bayesian engine, power analysis |
| 8 | Multi-Armed Bandit | 3 weeks | Thompson sampling, adaptive allocation, weight scheduling |
| 9 | AI-Powered Features | 4 weeks | Hypothesis generation, SRM root-cause, NL experiment authoring |
| 10 | Enterprise & Production | 4 weeks | RBAC, exclusion groups, K8s deployment, Python SDK, monitoring |
| **Total** | | **~38 weeks** | |

**Critical path:** Phases 1 -> 2 -> 3 -> 4 (15 weeks to first usable product)
**MVP milestone:** End of Phase 5 (~20 weeks) -- full platform with UI, flags, experiments, and basic statistics
**Competitive differentiation:** Phase 7 (advanced stats) + Phase 9 (AI features) deliver the capabilities that distinguish this platform from GrowthBook and other open-source alternatives
