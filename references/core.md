# Core Standards

## Naming Conventions

**`TM_EXT_` is for database objects only** (tables, schemas, Prisma `@@map` names). Do **not** prefix service names, repo folders, or API path segments with `TM_EXT_`.

### Service Names (no `TM_EXT_` prefix)

| Type | Pattern | Examples |
|------|---------|----------|
| Business | `{servicename}` | `job`, `candidate`, `ats`, `auth`, `notification` |
| AI | `ai-{servicename}` | `ai-resume`, `ai-interview`, `ai-orchestrator`, `ai-skillmatch` |
| Repo / package folder | `{servicename}-service` or `ai-{servicename}` | `job-service/`, `ai-orchestrator/` |

Service names are lowercase, hyphenated for multi-word AI services. Use the same `{servicename}` segment in API routes (see below).

### Database Tables & Schemas (prefix `TM_EXT_`, ALL CAPS)

| Pattern | Examples |
|---------|----------|
| Domain tables | `TM_EXT_{SERVICE}_{MODULE}` | `TM_EXT_JOB_LISTINGS`, `TM_EXT_CANDIDATE_PROFILES`, `TM_EXT_ATS_APPLICATIONS` |
| Settings (mandatory) | `TM_EXT_{SERVICE}_SETTINGS` | `TM_EXT_JOB_SETTINGS`, `TM_EXT_ATS_SETTINGS` |
| AI / cross-cutting | `TM_EXT_{DESCRIPTIVE_NAME}` | `TM_EXT_AI_ORCHESTRATOR_USAGE` |
| PostgreSQL schema | `TM_EXT_{SERVICE}` | `TM_EXT_JOB`, `TM_EXT_ATS` |

In Prisma, use PascalCase model names in code and map to physical tables:

```prisma
model JobListing {
  id String @id @default(uuid())
  // ...
  @@map("TM_EXT_JOB_LISTINGS")
}
```

Column names in the database: `snake_case` (e.g. `created_at`, `is_deleted`). Only **table** (and schema) names use the `TM_EXT_` prefix.

### Cache / Redis Keys

```
TM_EXT:{SERVICE}:{ENTITY}:{id}
```

Examples: `TM_EXT:ATS:APPLICATION:uuid-123`, `TM_EXT:AUTH:SESSION:uuid-789`

### Queue Names

```
TM_EXT:{SERVICE}:{ACTION}
```

Examples: `TM_EXT:NOTIFICATION:SEND_EMAIL`, `TM_EXT:AI_RESUME:PARSE`

### API Routes

```
/{servicename}/api/v1/{resource}
```

Examples: `/job/api/v1/listings`, `/ai-resume/api/v1/parse` — route uses **service name**, not `TM_EXT_`.

### Environment Variables

```
TM_EXT_{SERVICENAME}_{KEY}
```

Examples: `TM_EXT_JOB_PORT=3001`, `TM_EXT_ATS_DATABASE_URL=...`, `TM_EXT_SWAGGER_USER`, `TM_EXT_SWAGGER_PASSWORD`

---

## Settings Table (mandatory per service)

Table name: `TM_EXT_{SERVICE}_SETTINGS` (e.g. `TM_EXT_JOB_SETTINGS`)

### Minimum Fields

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Primary key |
| `setting_key` | String (unique) | Lookup key |
| `setting_value` | Text | Stored value (cast by `value_type`) |
| `value_type` | Enum | `STRING`, `NUMBER`, `BOOLEAN`, `JSON`, `SECRET` |
| `category` | String | `general`, `integration`, `ai_model`, `limits`, `infrastructure` |
| `is_mutable` | Boolean | `false` = deploy-only; `true` = Admin Portal |
| `is_sensitive` | Boolean | `true` = masked in UI, never logged |
| `description` | Text | Human-readable explanation |
| `created_by` | String | Creator |
| `created_at` | Timestamp | Auto-set |
| `updated_at` | Timestamp | Auto-set |

### Rules

- `is_mutable=false` → only changeable via code deployment
- `is_sensitive=true` → masked in UI, never logged
- `value_type=SECRET` → encrypted at rest, never plain text via API
- All business config (thresholds, model names, feature flags, timeouts) → Settings Table
- Only infra config (PORT, DATABASE_URL, MQ_URL, JWT_SECRET) → `.env`
- Index on `setting_key` (unique) and `category`
- Settings loaded into memory on startup via singleton `SettingsService` — no DB calls after boot until reload

Reload endpoint: `POST /{servicename}/api/v1/admin/settings/reload`

---

## API Standards

### HTTP Methods (banking policy)

**Only GET and POST.** No PATCH, PUT, DELETE.

| Action | Method | URL Pattern |
|--------|--------|-------------|
| Read / List | GET | `/{resource}` or `/{resource}/:id` |
| Create | POST | `/{resource}` |
| Update | POST | `/{resource}/:id/update` |
| Delete (soft) | POST | `/{resource}/:id/delete` |
| Actions | POST | `/{resource}/:id/{action}` |

### Response Envelope (all endpoints)

```json
{
  "success": true,
  "data": {},
  "meta": { "timestamp": "ISO8601", "requestId": "uuid", "version": "1.0.0" },
  "error": null
}
```

### Error Response

```json
{
  "success": false,
  "data": null,
  "meta": { "timestamp": "...", "requestId": "..." },
  "error": { "code": "VALIDATION_ERROR", "message": "Human readable", "details": [] }
}
```

### Standard Error Codes

| Code | HTTP | When |
|------|------|------|
| `VALIDATION_ERROR` | 400 | Schema validation failed |
| `BAD_REQUEST` | 400 | Malformed request |
| `UNAUTHORIZED` | 401 | Missing/invalid token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource missing or soft-deleted |
| `CONFLICT` | 409 | Duplicate key / state conflict |
| `RATE_LIMITED` | 429 | Too many requests |
| `TOKEN_BUDGET_EXCEEDED` | 429 | AI service daily token budget exhausted (orchestrator) |
| `SETTINGS_IMMUTABLE` | 403 | Updating `is_mutable=false` setting |
| `INTERNAL_ERROR` | 500 | Unhandled server error |
| `SERVICE_UNAVAILABLE` | 503 | Dependency down |

### Mandatory Endpoints (every service)

```
GET    /{servicename}/api/v1/admin/settings
POST   /{servicename}/api/v1/admin/settings/:key/update
POST   /{servicename}/api/v1/admin/settings/reload
GET    /{servicename}/api/v1/health
GET    /{servicename}/api/v1/health/ready
```

### Pagination (all list endpoints)

**Query params** (request):

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | Number | 1 | Page number (1-based) |
| `limit` | Number | 20 | Page size (max 100) |
| `sort_by` | String | `created_at` | Sort field |
| `sort_order` | String | `desc` | `asc` or `desc` |

**Response** — `meta.pagination` object (required on all list endpoints):

| Field | Type | Description |
|-------|------|-------------|
| `currentPage` | int | Current page (same as request `page`) |
| `pageSize` | int | Items per page (same as request `limit`) |
| `totalItems` | int | Total matching records |
| `totalPages` | int | `ceil(totalItems / pageSize)` |
| `hasNext` | bool | `currentPage < totalPages` |
| `hasPrevious` | bool | `currentPage > 1` |

Full paginated envelope example:

```json
{
  "success": true,
  "data": [],
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z",
    "requestId": "uuid",
    "version": "1.0.0",
    "pagination": {
      "currentPage": 1,
      "pageSize": 20,
      "totalItems": 150,
      "totalPages": 8,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "error": null
}
```

Zod schema (shared — Node services):

```typescript
export const paginationMetaSchema = z.object({
  currentPage: z.number().int().min(1),
  pageSize: z.number().int().min(1).max(100),
  totalItems: z.number().int().min(0),
  totalPages: z.number().int().min(0),
  hasNext: z.boolean(),
  hasPrevious: z.boolean(),
});

export function buildPaginationMeta(page: number, pageSize: number, totalItems: number) {
  const totalPages = pageSize > 0 ? Math.ceil(totalItems / pageSize) : 0;
  return {
    currentPage: page,
    pageSize,
    totalItems,
    totalPages,
    hasNext: page < totalPages,
    hasPrevious: page > 1,
  };
}
```

### Authentication `[AUTH_SWAP]`

Auth is a middleware/plugin that can be enabled or disabled.

- Day 1: passthrough mode (mock user context)
- Later: swap to real auth (JWT / SSO) — mark with `// [AUTH_SWAP]`

Request context injected by auth middleware:

```typescript
interface RequestContext {
  userId: string;
  tenantId: string;
  roles: string[];
}
```

RBAC via `roles` guard — services declare required roles, middleware enforces.

### Date & Time

| Type | Format | Example |
|------|--------|---------|
| Timestamps | ISO 8601 UTC | `2025-01-15T10:30:00Z` |
| Date-only | `YYYY-MM-DD` | `2025-01-15` |

---

## Swagger / OpenAPI (mandatory)

Every HTTP service must expose OpenAPI docs behind authentication.

| Rule | Detail |
|------|--------|
| Coverage | Every Fastify and FastAPI service exposes OpenAPI 3.x |
| Fastify | `@fastify/swagger` + `@fastify/swagger-ui` at `/documentation` |
| FastAPI | `/docs` + `/redoc` + `/openapi.json` |
| Protection | HTTP Basic auth — `TM_EXT_SWAGGER_USER` / `TM_EXT_SWAGGER_PASSWORD` from env/Vault |
| Health exempt | `GET /health`, `GET /health/ready` have no auth |
| Production | Same protection; disable only via `swagger_enabled=false` setting with approval |

**Forbidden**: public Swagger without auth, credentials in frontend, logging Basic auth headers.

See [assets/fastify-swagger-auth-template.md](../assets/fastify-swagger-auth-template.md) and [assets/fastapi-swagger-auth-template.md](../assets/fastapi-swagger-auth-template.md) for implementation patterns.

---

## Config Placement

| Type | Where |
|------|-------|
| DB URL, Port, MQ connection | `.env` |
| Feature flags, thresholds, timeouts | Settings Table (`is_mutable=true`) |
| AI model names, providers | Settings Table (`is_mutable=true`) |
| Encryption keys, JWT secrets | Vault / `.env` (`is_mutable=false`) |
| API keys for external services | Settings Table (`is_sensitive=true`) |
| Swagger credentials | `.env` / Vault (`TM_EXT_SWAGGER_USER`, `TM_EXT_SWAGGER_PASSWORD`) |

---

## Swap Tags

Mark all abstracted infrastructure code with these tags for easy find-and-replace:

| Tag | Purpose |
|-----|---------|
| `// [AUTH_SWAP]` or `# [AUTH_SWAP]` | Auth middleware usage |
| `// [CACHE_SWAP]` or `# [CACHE_SWAP]` | Cache store usage |
| `// [MQ_SWAP]` or `# [MQ_SWAP]` | Message queue usage |

---

## Code Style Rules

- Soft delete only (`is_deleted` flag) — never hard delete
- UUID for all primary keys
- All timestamps in UTC (ISO 8601)
- No business logic in route handlers — validate input and call services
- Repository pattern for all DB access
- Zod schemas (Node.js) / Pydantic schemas (Python) for all request/response
- No `any` type in TypeScript — strict mode enabled
- Async/await everywhere — no callbacks
- Never swallow errors silently — throw typed errors or log and re-throw
- No hardcoded magic strings/numbers — use constants or Settings Table
- Keep functions small and single-responsibility
