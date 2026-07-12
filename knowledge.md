# Project knowledge

This file gives Freebuff context about your project: goals, commands, conventions, and gotchas.

## Quickstart

### Development (all services via dhee-deployment)
```bash
cd dhee-deployment
cp env/examples/*.env env/dev/
./deploy.sh pull-code dev
./deploy.sh build
./deploy.sh setup-ssl dev
./deploy.sh bootstrap dev
./deploy.sh create-admin dev
# Access: https://localhost
```

### Individual service dev (Python FastAPI services)
```bash
cd <service-dir>
python3 -m venv .ve && . .ve/bin/activate
pip install -r requirements.txt
uvicorn main.app:app --reload --host 0.0.0.0 --port 5000
```

### Frontend apps (Next.js)
```bash
cd tenant-management-app  # or lms/dhee-app
pnpm install
pnpm dev
```

### Test
```bash
# dhee-deployment test suite
./deploy.sh test
./deploy.sh test <service-name>

# Individual Python services
pytest

# Frontend
pnpm lint
pnpm type-check
```

### Lint / Format (Python services)
```bash
# Via Makefile (available in most services)
make lint        # flake8 + isort + black + mypy
make format      # black
make style       # flake8
make types       # mypy
make migrate     # alembic upgrade head
make migration MESSAGE="description"  # generate alembic migration
```

### Database migrations
```bash
# Per-service in dev
docker exec -it dhee-toauth-service alembic upgrade head
docker exec -it dhee-notification-service alembic upgrade head
docker exec -it dhee-certificate-generation-service alembic upgrade head
# Others use: docker compose -f compose/docker-compose.dev.yml run --rm <service> alembic upgrade head
```

## Architecture

### What this is
A **multi-tenant LMS (Learning Management System)** platform called **Dhee** (also referenced as CertifySphere/CIRRS in older contexts). Monorepo with ~15+ microservices, 2 frontend apps, and deployment infrastructure.

### Key directories

| Directory | Description | Stack |
|-----------|-------------|-------|
| `lms/dhee-gateway-service/` | API Gateway — single entry point for all microservices | FastAPI, Python |
| `TOAuth/` | Auth service (JWT, OAuth, Celery, RBAC) | FastAPI, PostgreSQL, Celery, RabbitMQ |
| `user-profile-service/` | User profiles, RBAC (vendored rbac_core) | FastAPI, PostgreSQL, SQLAlchemy, Alembic |
| `tenant-management-service/` | SaaS tenant registry & lifecycle | FastAPI, PostgreSQL, Alembic |
| `notification-service/` | **Legacy/unused** — superseded by `cirrs-notification-service` | FastAPI, PostgreSQL, WebSocket |
| `cirrs-notification-service/` | **Active notification service** — this is the one in use | FastAPI, PostgreSQL |
| `lms/question-bank-service/` | Question repository for exams | FastAPI, MongoDB, Motor |
| `lms/exam-service/` | Exam creation, sessions, scoring | FastAPI, PostgreSQL, SQLAlchemy |
| `lms/lms-course-management-service/` | Course/module/section/activity/enrollment | FastAPI, PostgreSQL, SQLAlchemy |
| `lms/certificate-generation-service/` | Certificate generation with PDF caching | FastAPI, PostgreSQL |
| `lms/lms-content-management-service/` | Course content & curriculum | FastAPI |
| `lms/lms-content-delivery-service/` | Content streaming & delivery | FastAPI |
| `lms/lms-finance-service/` | Payments, billing | FastAPI |
| `lms/tutor-management-service/` | Tutor management | FastAPI, Python |
| `lms/lms-assessment-service/` | AI assessment, peer review | FastAPI |
| `lms/lms-reporting-service/` | Analytics, progress tracking | FastAPI, MongoDB |
| `lms/lms-personalization-service/` | Adaptive learning, recommendations | FastAPI, Neo4j, Qdrant |
| `tenant-management-app/` | Multi-tenant admin portal (Next.js 15) | Next.js 15, React 19, Tailwind 4, pnpm |
| `lms/dhee-app/` | Main LMS frontend (Next.js 15) | Next.js 15, React 19, Tailwind 4, pnpm |
| `dhee-deployment/` | Deployment configs (dev/staging/prod) | Docker Compose, NGINX, SSL |
| `deployment-toolkit/` | Shared deployment framework | Bash scripts |

### Data flow
```
Client → NGINX (SSL termination, tenant subdomain routing)
       → dhee-gateway-service (API gateway, auth verification)
       → individual microservices
       → PostgreSQL (per-service databases, 2 shared PG instances)
       → MongoDB (question bank, reporting events)
       → Redis (caching/sessions), RabbitMQ (async), Elasticsearch (search)
       → Neo4j/Qdrant (knowledge graph, ML embeddings for personalization)
```

### Database layout (dev)
- **dhee-toauth-postgres** (port 5433): toauth, tutor_management, tenant_management DBs
- **dhee-lms-shared-postgres** (port 5437): All LMS service DBs (exam, cms, question_bank, cert, finance, assessment, reporting, personalization, notification)

## Conventions

### Python services
- **Formatter**: black (line-length 88), isort
- **Linter**: flake8
- **Type checker**: mypy
- **Pre-commit**: enabled in most services (`.pre-commit-config.yaml`)
- **Framework**: FastAPI with SQLAlchemy + Alembic for migrations
- **Project layout**: `main/` package with `api/v1/routes/`, `models/`, `schemas/`, `services/`, `core/`, `db/`
- **Settings**: Pydantic `BaseSettings` with env-prefix (`APP_` for tenant-mgmt)
- **Auth**: JWT via TOAuth, `HTTPBearer` security scheme, `Depends(require_permission(...))`
- **RBAC**: Vendored `rbac_core` package (role hierarchies, context-based permissions, SoD constraints)

### Frontend apps (Next.js)
- **Package manager**: pnpm
- **Styling**: Tailwind CSS 4, shadcn/radix-ui components
- **State**: Zustand (dhee-app), React Query / TanStack Query
- **Forms**: React Hook Form
- **I18n**: next-intl (dhee-app), backend translations supported
- **HTTP**: Axios
- **Node**: >=20 (dhee-app), >=22 (tenant-management-app)

### Deployment
- **Docker Compose** per environment: `compose/docker-compose.{dev,staging,prod}.yml`
- **Central proxy network**: `central-proxy-net` for staging/prod
- **SSL**: self-signed (dev), Let's Encrypt (staging/prod)
- **Multi-tenant routing**: subdomain-based (`*.domain.com`)
- **Admin port**: 9443 with IP whitelisting
- **Environment files**: `env/{dev,staging,prod}/*.env`

### Patterns to follow
- Use `str_replace` for targeted edits, `write_file` for new files
- Read surrounding code before editing to match style
- Each service has its own `Makefile` with standard targets
- Use `docker exec -it <container> <command>` for service-specific ops
- Alembic for all PostgreSQL migrations

### Things to avoid
- Don't assume packages are available — check `requirements.txt` / `pyproject.toml` first
- Don't commit secrets or `.env` files
- Don't use `POSTGRES_DB` as a generic variable — multiple env files define it (use service-specific names like `TOAUTH_DATABASE_URL`)
- Don't mix up container names (`dhee-` prefix) vs compose service names (no prefix)
- Don't hardcode domains — use `DOMAIN` from `env/<env>/common.env` as single source of truth
- Don't edit `deployment-toolkit/` directly — edit project's `scripts/common.sh` instead
- Don't work in `notification-service/` — it's legacy/unused; all notification work goes in `cirrs-notification-service/`
