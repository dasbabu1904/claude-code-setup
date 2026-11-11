# Dhee Project: Technical Specification

Dhee is a multi-tenant learning management platform built from independently deployed services. Its main areas include identity and tenant administration, course delivery, exams and assessments, payments, reporting, personalization, and live learning.

This specification is based on the current workspace manifests and deployment Compose files. **Versions below are repository declarations**: some are exact pins, some are version ranges, and some use floating tags. They may differ from versions currently running in a deployed environment.

## Architecture

- **Style:** Microservices, with a shared API gateway and independently deployed service and worker processes.
- **Tenant model:** Multi-tenant, with tenant-aware services and subdomain-based routing configured at the proxy.
- **Request path:** Client → NGINX → Dhee gateway → domain services.
- **Background processing:** Celery workers, RabbitMQ, Redis, and Kafka are used across services for asynchronous jobs and events.
- **Persistence:** PostgreSQL for relational service data; MongoDB for selected document and reporting workloads; Neo4j and Qdrant for personalization-related graph and vector data.
- **Files and media:** MinIO-compatible object storage, plus Gotenberg for document rendering.

## Applications and services

| Area | Components | Technology |
|---|---|---|
| Main LMS | `dhee-app` | Next.js, React, TypeScript |
| Tenant administration | `tenant-management-app` | Next.js, React, TypeScript |
| API entry point | Gateway | Python, FastAPI |
| Identity and access | TOAuth, profile, tenant management | Python, FastAPI; JWT/OAuth and RBAC |
| Learning domain | Course management, content management/delivery, question bank, exams, certificates, assessment, tutoring | Python services; primarily FastAPI |
| Business services | Finance, payment wallet, events, training, notifications | Python services; primarily FastAPI |
| Analytics and personalization | Reporting, personalization | Python services; MongoDB, Neo4j and Qdrant integrations |
| Live learning | DheeLive web, conference orchestration, proctoring, recording/egress | Next.js, Python services, LiveKit |
| Deployment | `dhee-deployment` | Docker Compose, NGINX, deployment scripts |

The deployment configuration also includes workflow and collaboration services. Some services are present in the workspace but may not be enabled in every environment.

## Technologies and declared versions

### Application runtimes and frameworks

| Technology | Version in repository |
|---|---|
| Python | Mixed: many service Dockerfiles use `3.14.6-slim`; DheeLive services declare Python `>=3.12`; collaboration uses `3.11-slim`; other service metadata allows Python `>=3.11` or gives a broader range. |
| FastAPI | Mixed by service. Examples include exact pins `0.115.0`, `0.115.5`, and `0.104.1`; other services specify ranges such as `>=0.110.0` or leave it unpinned. |
| Next.js, main LMS app | `15.4.8` |
| React, main LMS app | `19.1.0` |
| Next.js, tenant admin app | `^15.5.12` |
| React, tenant admin app | `^19.2.4` |
| Node.js, main LMS app | `>=22`; Dockerfile uses `node:22-slim` |
| Node.js, tenant admin app | `>=22`; Dockerfile uses `node:22-alpine` |
| Node.js, DheeLive web | Dockerfile uses `node:20-slim` |
| pnpm, main LMS app | `9.15.4` |
| Tailwind CSS | `^4.2.1` in both main and tenant admin app manifests |
| TypeScript, main LMS app | `5.9.3` |

Backend dependency versions are **not uniform across services**. For example, course management pins FastAPI `0.115.0`, while the gateway uses a lower-bound range. SQLAlchemy, Pydantic, and Alembic are also pinned or constrained differently across service manifests.

### Data, messaging, and infrastructure

The Dhee deployment Compose files declare these image tags:

| Component | Image version/tag |
|---|---|
| PostgreSQL | `18-alpine` |
| MongoDB | `8` |
| Redis | `8-alpine`; a separate collaboration Redis uses `7-alpine` |
| RabbitMQ | `4.3.4-management-alpine` |
| Apache Kafka | `4.3.1` |
| Elasticsearch | `9.5.0` |
| Neo4j | `2026-community` |
| Qdrant | `v1.19.0` |
| Ollama | `0.32.5` |
| NGINX, Dhee deployment | `1.31-alpine` |
| Gotenberg | `8` |
| MinIO | `latest` in Dhee deployment |
| LiveKit server and egress | `latest` in DheeLive deployment |

Several infrastructure tags are major-version tags or `latest`, so they do not lock deployments to a specific patch release. Service images themselves commonly use branch-based tags such as `master`, `shared`, or `multitenancy`, rather than semantic versions.

### Supporting technologies

- **Database migrations:** Alembic.
- **ORM:** SQLAlchemy in relational-data services.
- **Authentication:** JWT/OAuth through TOAuth; RBAC is used for permission checks.
- **Task processing:** Celery workers; broker/backend configuration varies by service and can use RabbitMQ or Redis.
- **Observability:** OpenTelemetry instrumentation appears in service dependencies; exact versions vary.
- **Live classes and recording:** LiveKit server, egress, and client SDKs.
- **Frontend data and forms:** TanStack Query, Axios, React Hook Form; the main app also uses Zustand and next-intl.

## Version caveats

Treat this as a **workspace-level inventory**, not a single centrally enforced bill of materials. Python and backend library requirements differ among services, and some requirements files are unpinned. Container tags such as `latest`, major-only tags, and branch names can resolve to different builds over time. The deployment configuration and service lockfiles or dependency manifests are the best sources for an environment-specific version report.

## Source files

This summary draws primarily from `knowledge.md`, `dhee-deployment/compose/docker-compose.dev.yml`, `lms/dhee-app/package.json`, `tenant-management-app/package.json`, and individual service `pyproject.toml`, `requirements.txt`, and Dockerfiles.
