# Backend Standards (Fastify + TypeScript)

## Canonical Stack

| Component | Standard | Do not use |
|-----------|----------|------------|
| HTTP framework | **Fastify + TypeScript** | Express, NestJS, plain `http` |
| Database | **PostgreSQL** | MySQL, MongoDB for transactional data |
| ORM | **Prisma** | TypeORM, Drizzle, Knex, raw SQL in app code |
| Migrations | **Prisma Migrate** | Ad-hoc SQL scripts |
| Validation | **Zod** + `fastify-type-provider-zod` | Joi, class-validator |
| Logging | **Pino** (Fastify built-in) | Winston, Bunyan |
| Security headers | **@fastify/helmet** | Manual header setting |
| API docs | **@fastify/swagger** + **@fastify/swagger-ui** (password-protected) | Public docs |

---

## Project Structure

```
{servicename}-service/
├── src/
│   ├── app.ts                             # Fastify instance, plugin registration
│   ├── server.ts                          # Bootstrap, load settings, start server
│   ├── plugins/
│   │   ├── cors.ts
│   │   ├── swagger.ts                     # Password-protected (see core.md)
│   │   ├── zod-provider.ts                # @fastify/type-provider-zod
│   │   ├── auth.ts                        # [AUTH_SWAP] Day 1: passthrough
│   │   ├── error-handler.ts
│   │   └── correlation-id.ts
│   ├── common/
│   │   ├── cache/
│   │   │   ├── cache.interface.ts
│   │   │   └── in-memory-cache.store.ts   # [CACHE_SWAP]
│   │   ├── config/env.config.ts           # Only infra configs (PORT, DB_URL)
│   │   ├── middleware/
│   │   │   └── rate-limit.ts
│   │   ├── schemas/
│   │   │   └── response.schema.ts         # Zod schemas for envelope
│   │   └── utils/
│   ├── settings/
│   │   ├── settings.routes.ts
│   │   ├── settings.service.ts            # Singleton, in-memory after boot
│   │   ├── settings.repository.ts
│   │   └── settings.schema.ts
│   ├── health/
│   │   └── health.routes.ts
│   ├── modules/{domain}/
│   │   ├── {domain}.routes.ts
│   │   ├── {domain}.service.ts
│   │   ├── {domain}.repository.ts
│   │   └── {domain}.schema.ts             # Zod request/response schemas
│   └── events/
│       ├── producers/
│       └── consumers/
├── prisma/
│   └── schema.prisma                      # Source of truth for DB schema
├── tests/
│   ├── unit/
│   └── integration/
├── .env.example
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── vitest.config.ts
├── Dockerfile
├── docker-compose.yml
└── package.json
```

---

## Prisma + PostgreSQL Rules

- One Prisma schema per service; PostgreSQL schema: `TM_EXT_{SERVICE}` (e.g. `TM_EXT_JOB`)
- UUID primary keys on all tables
- **Mandatory columns** on every table:

```prisma
model JobListing {
  id         String   @id @default(uuid())
  // ... domain fields ...
  createdAt  DateTime @default(now()) @map("created_at")
  updatedAt  DateTime @updatedAt @map("updated_at")
  createdBy  String   @map("created_by")
  isDeleted  Boolean  @default(false) @map("is_deleted")

  @@map("TM_EXT_JOB_LISTINGS")
}
```

- **Soft delete only** — never hard delete; filter `is_deleted = false` in repositories
- Column-level encryption for PII (Aadhaar, PAN, etc.)
- Migrations via `prisma migrate dev` / `prisma migrate deploy` — never raw SQL

### Database Indexing

- Add `@@index` in Prisma schema for columns used in WHERE, ORDER BY, or JOIN
- Composite indexes for candidate search and reporting queries
- Run `EXPLAIN ANALYZE` on new queries touching large tables
- Avoid N+1: use `include` / `select` deliberately; prefer explicit joins over lazy loading

```prisma
@@index([createdAt])
@@index([isDeleted, createdAt])
@@index([status, createdAt])  // composite for filtered listings
```

---

## Repository Pattern

All DB access goes through repository classes. Route handlers never call Prisma directly.

```typescript
// src/modules/job/job.repository.ts
export class JobRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string) {
    return this.prisma.jobListing.findFirst({
      where: { id, isDeleted: false },
    });
  }

  async findMany(params: ListParams) {
    const { page = 1, limit = 20, sort_by = 'createdAt', sort_order = 'desc' } = params;
    const [data, total] = await Promise.all([
      this.prisma.jobListing.findMany({
        where: { isDeleted: false },
        orderBy: { [sort_by]: sort_order },
        skip: (page - 1) * limit,
        take: limit,
      }),
      this.prisma.jobListing.count({ where: { isDeleted: false } }),
    ]);
    return { data, total, page, limit };
  }

  async softDelete(id: string, userId: string) {
    return this.prisma.jobListing.update({
      where: { id },
      data: { isDeleted: true, updatedAt: new Date() },
    });
  }
}
```

---

## Route Handler Pattern

Route handlers validate input (via Zod schemas) and delegate to services. No business logic in handlers.

```typescript
// src/modules/job/job.routes.ts
import { FastifyPluginAsyncZod } from 'fastify-type-provider-zod';
import { buildPaginationMeta } from '../../common/schemas/pagination.schema';
import { createJobSchema, listJobsQuerySchema } from './job.schema';

const jobRoutes: FastifyPluginAsyncZod = async (app) => {
  app.get('/job/api/v1/listings', {
    schema: { querystring: listJobsQuerySchema },
  }, async (request, reply) => {
    const { page = 1, limit = 20 } = request.query;
    const result = await jobService.list(request.query);
    return reply.send({
      success: true,
      data: result.data,
      meta: {
        timestamp: new Date().toISOString(),
        requestId: request.id,
        version: '1.0.0',
        pagination: buildPaginationMeta(page, limit, result.total),
      },
      error: null,
    });
  });
};
```

---

## Cache Abstraction `[CACHE_SWAP]`

All cache usage through `ICacheStore` interface:

```typescript
export interface ICacheStore {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttlSeconds?: number): Promise<void>;
  del(key: string): Promise<void>;
  flush(): Promise<void>;
}
```

Day 1: `InMemoryCacheStore`. Swap to Redis by searching `[CACHE_SWAP]` — only instantiation changes.

Cache key format: `TM_EXT:{SERVICE}:{ENTITY}:{id}` (see `core.md`).

---

## Message Queue `[MQ_SWAP]`

Day 1: **BullMQ** → Later: **RabbitMQ**

Queue interface:

```typescript
export interface IQueueProducer {
  publish(queueName: string, message: QueueMessage, opts?: JobOpts): Promise<void>;
}

export interface QueueMessage {
  messageId: string;
  type: string;
  source: string;
  correlationId: string;
  timestamp: string;
  payload: Record<string, unknown>;
  metadata: { userId: string; tenantId: string };
}
```

Queue settings in Settings Table: `queue_default_retries`, `queue_retry_delay_ms`, `queue_concurrency`, `queue_dead_letter_enabled`.

---

## Swagger (password-protected)

Register at `src/plugins/swagger.ts`. Protect with `onRequest` hook validating HTTP Basic against `TM_EXT_SWAGGER_USER` / `TM_EXT_SWAGGER_PASSWORD`. See [assets/fastify-swagger-auth-template.md](../assets/fastify-swagger-auth-template.md).

Health routes (`/health`, `/health/ready`) are exempt from auth.

---

## API Versioning

- Routes: `/{servicename}/api/v1/{resource}`
- Breaking changes → `v2` prefix; never silent breaks
- Non-breaking additions (new optional fields) stay in current version

---

## Vector Data (Qdrant boundary)

Business Fastify services do **not** talk to Qdrant directly. All vector operations go through `ai-orchestrator` (see `ai.md`).

- Self-hosted Qdrant cluster URL in Settings Table (`qdrant_host`)
- Collections: `TM_EXT_RESUMES`, `TM_EXT_JOBS`, `TM_EXT_SKILLS` (or as configured in settings)
- No Qdrant client in any business service (`job`, `ats`, etc.)

---

## Security

- RBAC via Auth Service, deny by default
- AES-256 at rest, TLS 1.3 in transit
- Parameterized queries only (Prisma handles this)
- Input validation: Zod on every endpoint
- `@fastify/helmet` for security headers
- No secrets in code — use Vault / `.env`
- GitLeaks in pre-commit hooks
- Service-to-service: mTLS (Zero Trust)
