# Testing Standards

## Testing Matrix

| Layer | Tools | Scope |
|-------|-------|-------|
| FE unit / UI | **Vitest + React Testing Library** | Components, hooks, reducers |
| FE integration | **Vitest + MSW** (or mocked Axios) | User flows, API contract |
| BE unit | **Vitest** | Services, repositories |
| BE HTTP | **Vitest + Supertest** + `app.inject()` | Routes, envelope, auth middleware |
| AI unit | **Pytest** + **httpx** `AsyncClient` | Schemas, service logic |
| AI evals | **Pytest** + eval harness | Golden datasets, regression, edge cases |
| E2E | **Playwright** + **@axe-core/playwright** | Critical user journeys + WCAG scans |
| A11y unit | **vitest-axe** | Component-level WCAG rule checks |
| Production | Synthetic monitors + health checks | Audits, uptime |

---

## Coverage Target

- **80% minimum** for all new code (FE + BE + AI)
- Measured per service in CI — build fails if coverage drops below threshold
- Focus on business logic and edge cases, not boilerplate getters

---

## Frontend Testing (Vitest + RTL)

### Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      thresholds: { statements: 80, branches: 80, functions: 80, lines: 80 },
    },
  },
  resolve: { alias: { '@': '/src' } },
});
```

### What to Test

- **Components**: render, user interactions, conditional rendering, **accessibility** (`vitest-axe`, `getByRole` / `getByLabelText`)
- **Hooks**: custom hook behavior with `renderHook`
- **Redux slices**: reducers and selectors (pure function tests)
- **API integration**: mock Axios or use MSW to verify request/response handling
- **Forms**: validation messages, submit behavior, error states

### Patterns

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('submits form with valid data', async () => {
  const user = userEvent.setup();
  render(<JobForm onSubmit={mockSubmit} />);

  await user.type(screen.getByLabelText(/title/i), 'Senior Engineer');
  await user.click(screen.getByRole('button', { name: /submit/i }));

  expect(mockSubmit).toHaveBeenCalledWith(
    expect.objectContaining({ title: 'Senior Engineer' })
  );
});
```

---

## Backend Testing (Vitest + Supertest)

### Unit Tests

Test services and repositories in isolation:

```typescript
import { describe, it, expect, vi } from 'vitest';

describe('JobService', () => {
  it('returns cached job if available', async () => {
    const mockCache = { get: vi.fn().mockResolvedValue(mockJob) };
    const service = new JobService(mockCache, mockRepo);
    const result = await service.getJob('uuid');
    expect(result).toEqual(mockJob);
    expect(mockCache.get).toHaveBeenCalledWith('TM_EXT:JOB:LISTING:uuid');
  });
});
```

### HTTP / Integration Tests

Use `app.inject()` (Fastify native) or Supertest:

```typescript
import { describe, it, expect } from 'vitest';
import { buildApp } from '../src/app';

describe('GET /job/api/v1/listings', () => {
  it('returns paginated envelope', async () => {
    const app = await buildApp();
    const res = await app.inject({
      method: 'GET',
      url: '/job/api/v1/listings?page=1&limit=10',
    });
    const body = JSON.parse(res.body);
    expect(res.statusCode).toBe(200);
    expect(body.success).toBe(true);
    expect(body.meta.pagination).toEqual({
      currentPage: 1,
      pageSize: 10,
      totalItems: expect.any(Number),
      totalPages: expect.any(Number),
      hasNext: expect.any(Boolean),
      hasPrevious: false,
    });
  });
});
```

### What to Verify

- Response envelope structure (`success`, `data`, `meta`, `error`)
- Error codes match `core.md` table
- Pagination metadata
- Auth middleware behavior (passthrough / enforcement)
- Settings routes (get, update, reload)

---

## AI Testing (Pytest + httpx)

### Unit Tests

```python
import pytest
from app.modules.resume.service import ResumeService

@pytest.mark.asyncio
async def test_resume_parse_returns_structured_data():
    service = ResumeService(mock_orchestrator_client)
    result = await service.parse(sample_resume_text)
    assert "skills" in result
    assert isinstance(result["skills"], list)
```

### API Tests

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_health_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/ai-resume/api/v1/health")
    assert response.status_code == 200
    assert response.json()["success"] is True
```

### Evaluation Harness

Every AI workflow needs structured evals:

1. **Golden dataset**: curated `(input, expected_output)` pairs in `tests/evals/`
2. **Schema pass rate**: percentage of outputs validating against Pydantic model
3. **Regression suite**: run on every prompt or model change
4. **Edge cases**: adversarial inputs, empty inputs, max-length inputs

```python
# tests/evals/test_resume_parse_eval.py
@pytest.mark.parametrize("case", load_golden_dataset("resume_parse"))
async def test_resume_parse_golden(case):
    result = await service.parse(case["input"])
    assert_schema_valid(result)
    assert result["skills"] == case["expected"]["skills"]
```

### Accessibility (Vitest + axe)

Every shared layout and form component should include at least one axe scan. Setup: [assets/accessibility-setup-template.md](../assets/accessibility-setup-template.md). Standards: [accessibility.md](accessibility.md).

```typescript
import { axe } from 'vitest-axe';
import { render } from '@testing-library/react';

test('Sidebar has no axe violations', async () => {
  const { container } = render(<Sidebar />);
  expect(await axe(container)).toHaveNoViolations();
});
```

Prefer accessible queries in all UI tests:

```typescript
// Good
screen.getByRole('button', { name: /save/i });
screen.getByLabelText(/job title/i);

// Avoid unless no semantic alternative
screen.getByTestId('save-btn');
```

---

## E2E Testing (Playwright)

- Located at `e2e/` in the app root
- Cover critical user journeys: login, apply to job, file upload, dashboard navigation
- Run at least one spec with **mobile viewport** (e.g. iPhone 14) and one with desktop
- Run in CI as an optional gate (not blocking, but tracked)

### E2E accessibility (axe)

Scan critical routes after auth setup. See [assets/accessibility-setup-template.md](../assets/accessibility-setup-template.md).

```typescript
import AxeBuilder from '@axe-core/playwright';

test('jobs page WCAG 2.1 AA', async ({ page }) => {
  await page.goto('/jobs');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();
  expect(results.violations).toEqual([]);
});
```

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('user can log in and see dashboard', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name="email"]', 'test@example.com');
  await page.fill('[name="password"]', 'password');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByText('Welcome')).toBeVisible();
});
```

---

## Production Confidence

- **Health checks**: `GET /health` (liveness), `GET /health/ready` (readiness with dependency checks)
- **Synthetic monitors**: automated probes hitting health endpoints from CI smoke stage
- **Audit endpoints**: service metadata, version, uptime

---

## New Feature Checklist

When adding a new feature, ensure:

- [ ] Unit tests for service/business logic
- [ ] Integration tests for API routes (envelope, error codes)
- [ ] If form-related: test validation and error states
- [ ] If AI workflow: add eval suite entry with golden data
- [ ] If new route: verify Swagger docs are generated
- [ ] Coverage stays above 80%
- [ ] UI: axe scan or manual a11y checklist per [accessibility.md](accessibility.md)
