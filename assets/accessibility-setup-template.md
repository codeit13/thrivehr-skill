# Accessibility Tooling Setup Template

Copy into `thrivehr-ui` when bootstrapping or adding WCAG enforcement. Aligns with [references/accessibility.md](../references/accessibility.md).

---

## Dependencies

```bash
pnpm add -D eslint-plugin-jsx-a11y vitest-axe @axe-core/playwright @axe-core/react
```

Optional (recommended):

```bash
pnpm add -D eslint-plugin-testing-library
```

---

## ESLint (`eslint.config.js` or `.eslintrc`)

```javascript
import jsxA11y from 'eslint-plugin-jsx-a11y';

export default [
  {
    plugins: { 'jsx-a11y': jsxA11y },
    rules: {
      ...jsxA11y.configs.recommended.rules,
      'jsx-a11y/anchor-is-valid': 'warn',
      'jsx-a11y/no-autofocus': ['warn', { ignoreNonDOM: true }],
      'jsx-a11y/click-events-have-key-events': 'error',
      'jsx-a11y/no-static-element-interactions': 'error',
      'jsx-a11y/label-has-associated-control': 'error',
    },
  },
];
```

For flat config with TypeScript + React, extend `plugin:jsx-a11y/recommended` in the React package eslint config.

---

## Vitest (`src/test/setup.ts`)

```typescript
import '@testing-library/jest-dom/vitest';
import 'vitest-axe/extend-expect';
import { configure } from '@testing-library/react';

configure({
  testIdAttribute: 'data-testid',
});
```

### Example component test

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { axe } from 'vitest-axe';
import { JobForm } from '@/features/jobs/components/JobForm';

describe('JobForm accessibility', () => {
  it('has no automated a11y violations', async () => {
    const { container } = render(<JobForm />);
    expect(await axe(container)).toHaveNoViolations();
  });

  it('exposes submit by accessible name', () => {
    render(<JobForm />);
    expect(screen.getByRole('button', { name: /create job/i })).toBeInTheDocument();
  });
});
```

---

## Playwright (`e2e/a11y.spec.ts`)

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

const CRITICAL_ROUTES = ['/login', '/dashboard', '/jobs'];

for (const path of CRITICAL_ROUTES) {
  test(`WCAG 2.1 AA scan: ${path}`, async ({ page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();

    expect(results.violations, formatViolations(results.violations)).toEqual([]);
  });
}

function formatViolations(violations: { id: string; impact?: string; description: string; nodes: unknown[] }[]) {
  return violations
    .map((v) => `[${v.impact}] ${v.id}: ${v.description} (${v.nodes.length} nodes)`)
    .join('\n');
}
```

Run with authenticated storage state when routes are protected.

---

## CI (GitHub Actions snippet)

```yaml
- name: Unit tests (incl. a11y)
  run: pnpm --filter thrivehr-ui test:coverage

- name: E2E accessibility
  run: pnpm --filter thrivehr-ui exec playwright test e2e/a11y.spec.ts
  continue-on-error: false # set true initially while fixing backlog
```

---

## globals.css — reduced motion

Add to `src/styles/globals.css`:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## Skip link styles

```css
.skip-link {
  position: absolute;
  left: -9999px;
  z-index: 999;
  padding: 0.75rem 1rem;
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
  border-radius: var(--radius);
}

.skip-link:focus {
  left: 1rem;
  top: 1rem;
}
```

See [skip-to-content-template.md](./skip-to-content-template.md).
