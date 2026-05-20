---
name: thrivehr
description: Applies ThriveHR (Axis Bank) standards for job, ats, ai-orchestrator, and related services. Database tables use TM_EXT_ prefix. Fastify, FastAPI, React, shadcn/ui, Prisma, PostgreSQL, Qdrant, Redux, Axios, Vitest. Use when writing ThriveHR backend, frontend, AI, tests, or CI/CD code.
---

# ThriveHR — Development Standards

> **Agent skill** for Cursor / Claude Code. The YAML block above is metadata (name + when to apply); it is not meant for Markdown preview. Read this document and linked `references/` files when generating code.

**Project:** Axis Bank Hiring & Onboarding Portal · **DB table prefix:** `TM_EXT_` (not used on service names)

## Technology Stack

| Layer | Choice |
|-------|--------|
| Business API | **Fastify + TypeScript** (e.g. `job`, `ats`, `candidate` services) |
| AI / Python | **FastAPI** (e.g. `ai-resume`, `ai-orchestrator`); Pydantic v2; SQLAlchemy + Alembic; Ruff |
| Frontend | **Vite + React + TypeScript**; **shadcn/ui**; **Tailwind CSS v4** |
| Database | **PostgreSQL** — tables **`TM_EXT_*`**; ORM: **Prisma** (Node) / **SQLAlchemy** (Python) |
| Vector DB | **Self-hosted Qdrant** (orchestrator-owned only) |
| Cache | Redis (Day 1: in-memory `[CACHE_SWAP]`) |
| Queue | BullMQ (Day 1, swap to RabbitMQ `[MQ_SWAP]`) |
| Object Storage | AWS S3 / MinIO |
| Validation | **Zod** (Fastify + React forms) / **Pydantic v2** (FastAPI) |
| Testing | **Vitest** + RTL (FE); **Vitest** + Supertest (BE); **Pytest** + httpx (AI) |
| Lint | **ESLint + Prettier** (Node/FE); **Ruff** (Python) |
| API Docs | **OpenAPI/Swagger** — mandatory, **password-protected** on every service |
| Logging | **Pino** (Node); **Loguru** (Python) |
| CI/CD | **GitHub Actions** (current); Bitbucket + Jenkins (planned) |
| Cross-cutting | **Langfuse** for LLM traces |

## Non-Negotiables (always apply)

- **GET and POST only** — no PATCH, PUT, DELETE (banking policy)
- **Response envelope** on every endpoint — `{ success, data, meta, error }`
- **Settings table** per service: `TM_EXT_{SERVICE}_SETTINGS` — business config in DB; `.env` only for infra
- **`TM_EXT_` on database tables/schemas only** — service names are plain (`job`, `ai-orchestrator`), not `TM_EXT_*`
- Swap tags: `// [AUTH_SWAP]`, `// [CACHE_SWAP]`, `// [MQ_SWAP]` on all abstracted code
- **No direct LLM calls** outside `ai-orchestrator` service
- **No PII in logs** — strip PII before external LLM APIs
- **shadcn/ui + Tailwind v4** for all UI; **React Hook Form + Zod** for forms; **TanStack Table** for data tables
- **Responsive UI** (mobile + desktop); **light + dark mode** (light default); clean, professional UX — see `frontend.md`
- **Auth**: access token in memory; refresh via httpOnly cookie; never localStorage for tokens
- **Vitest** for all new TypeScript tests (not Jest); 80% coverage minimum
- **Prisma** only for Node ORM (no TypeORM/Drizzle); **SQLAlchemy** only for Python AI services
- **Swagger/OpenAPI** on all services behind HTTP Basic auth (`TM_EXT_SWAGGER_USER` / `TM_EXT_SWAGGER_PASSWORD`); health routes exempt
- **Soft delete only** (`is_deleted` flag); UUID primary keys; all timestamps UTC ISO 8601
- No `any` in TypeScript — strict mode; async/await everywhere
- No business logic in route handlers — routes validate input and call services
- Repository pattern for all DB access
- **pnpm** for package management

## Frontend Stack (finalized)

| Area | Standard |
|------|----------|
| UI | **shadcn/ui** (Radix) — `src/components/ui/`; `cn()` for class composition |
| Styling | **Tailwind CSS v4** — `@theme` + CSS variables; shadcn tokens for brand |
| State | **Redux Toolkit** slices; server state pattern TBD (RTK Query vs TanStack Query) |
| Forms | **react-hook-form + Zod** via shadcn `<Form>` (`zodResolver`) |
| HTTP | **Axios** singleton — interceptors, envelope unwrap, correlation ID |
| Routing | **react-router-dom v6+** — `React.lazy` + `Suspense`, route-based splitting |
| Tables | **@tanstack/react-table** + shadcn Table |
| Dates | **date-fns** with `en-IN` locale |
| Icons | **lucide-react**; toasts: **sonner** |
| E2E | **Playwright** for critical flows |
| Accessibility | **WCAG 2.1 AA** minimum |
| Path alias | `@/` → `src/` |
| Brand colors | **Axis burgundy** `#971237` accent → shadcn `--primary`; tokens in `globals.css` only |
| Responsive | Mobile-first; works on phone through desktop |
| Themes | Light (default) + dark; `next-themes`, test both before merge |
| UX | Clean, professional layout; clear nav, labels, loading/empty/error states |

## Reference Index

Read the relevant file **before generating code** in that domain:

- **Building a Fastify service, API route, or Prisma schema** → [references/backend.md](references/backend.md)
- **Building React UI, forms, tables, Redux slices, or Axios calls** → [references/frontend.md](references/frontend.md)
- **Building AI workflows, orchestrator calls, PII guard, or evals** → [references/ai.md](references/ai.md)
- **Writing tests (any layer)** → [references/testing.md](references/testing.md)
- **API envelope, naming conventions, settings table, or Swagger auth** → [references/core.md](references/core.md)
- **Axios client template** → [assets/axios-client-template.md](assets/axios-client-template.md)
- **Fastify Swagger auth template** → [assets/fastify-swagger-auth-template.md](assets/fastify-swagger-auth-template.md)
- **FastAPI Swagger auth template** → [assets/fastapi-swagger-auth-template.md](assets/fastapi-swagger-auth-template.md)
- **CI/CD pipelines, Docker, or deployments** → [references/cicd.md](references/cicd.md)
- **ESLint, Prettier, Husky, git branching, or PR reviews** → [references/code-quality.md](references/code-quality.md)

## Conflict Resolutions

- **Testing**: Vitest + RTL (FE), Vitest + Supertest + `fastify.inject()` (BE); Jest only for legacy migration
- **Observability**: keep `TM_EXT_AI_ORCHESTRATOR_USAGE` table; add **Langfuse** as LLM tracing layer
- **CI/CD**: implement in **GitHub Actions** now; future Bitbucket mirror + Jenkins with identical stages
- **Data layer**: Prisma for Node business services; SQLAlchemy only inside Python AI services — not interchangeable
- **Swagger**: password-protected on both Fastify and FastAPI; never public
