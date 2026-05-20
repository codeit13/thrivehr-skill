# Code Quality Standards

## ESLint (Node.js + Frontend)

- Shared ESLint config extending across FE and BE packages
- TypeScript-specific rules enabled
- No `any` type — `@typescript-eslint/no-explicit-any: error`
- Strict mode in `tsconfig.json`

Recommended extends:

```json
{
  "extends": [
    "eslint:recommended",
    "@typescript-eslint/recommended",
    "@typescript-eslint/recommended-type-checked",
    "prettier"
  ]
}
```

---

## Prettier

Shared config across all TypeScript packages:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

---

## Ruff (Python AI Services)

- Replaces pylint, flake8, isort, black for Python services
- Config in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM"]
```

---

## Husky + lint-staged

Pre-commit hooks enforced via Husky:

```json
// package.json (root)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

### Pre-commit Hook Chain

1. **lint-staged**: ESLint fix + Prettier on changed files
2. **GitLeaks**: scan for secrets in staged changes
3. **commitlint**: validate commit message format (optional but recommended)

---

## Commit Convention

Use **Conventional Commits** format, especially in squash merge titles:

```
<type>(<scope>): <short description>
```

### Types

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes nor adds |
| `chore` | Build, CI, tooling changes |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `perf` | Performance improvement |

### Scope

Use service name or module: `feat(job):`, `fix(ats):`, `chore(ci):`, `test(resume):`

### Examples

```
feat(job): add listing search with pagination
fix(ats): handle null candidate email in notification
chore(ci): add dependency scan stage to pipeline
refactor(auth): extract token validation to middleware
```

---

## Git Branching Strategy

### Promotion Flow

```
feature/* → develop → uat → release
```

### Branch Rules

| Rule | Standard |
|------|----------|
| Long-lived branches | `develop` (integration), `uat` (QA), `release` (production) |
| Feature branches | `feature/{ticket}-short-desc` — branched from `develop` |
| Bugfix branches | `bugfix/{ticket}-short-desc` — from `develop` |
| Hotfix branches | `hotfix/{ticket}-short-desc` — from `release`; merge to `release` AND back-merge to `develop` |
| Merge to `develop` | **Squash merge** (one commit per feature, clean history) |
| `develop` → `uat` | **Merge commit** (preserve feature boundaries for QA tracing) |
| `uat` → `release` | **Merge commit** (auditable promotion) |
| Branch protection | All 3 long-lived branches: require PR, 1+ approval, passing CI, no direct push |
| Cleanup | Auto-delete merged feature/bugfix branches; no branch lives > 1 sprint |

---

## PR Template

Recommended `.github/pull_request_template.md`:

```markdown
## Summary

<!-- Brief description of what this PR does -->

## Type

- [ ] Feature
- [ ] Bug fix
- [ ] Refactor
- [ ] Chore / CI
- [ ] Documentation

## Linked Ticket

<!-- JIRA / GitHub Issue link -->

## Checklist

- [ ] Tests added/updated (unit + integration where applicable)
- [ ] No PII logged or exposed
- [ ] Swagger/OpenAPI docs updated (if new/changed endpoints)
- [ ] Database migration is reversible
- [ ] Linter passes (`eslint` / `ruff`)
- [ ] Coverage remains above 80%
- [ ] Follows naming conventions (`TM_EXT_` on DB tables only; service names without prefix — see `core.md`)
- [ ] UI responsive (mobile + desktop) and tested in light + dark mode (light default)
- [ ] UX: clear labels, loading/empty/error states, professional layout (see `frontend.md`)
- [ ] Response envelope format verified
- [ ] Reviewed for security (no secrets, input validated)

## Screenshots / Notes

<!-- Optional: UI changes, architecture diagrams -->
```

---

## Code Review Expectations

- **Minimum 1 approval** required for all PRs
- **2 approvals recommended** for cross-service PRs
- Reviewer checks: security, envelope compliance, test coverage, naming conventions, Swagger updates
- Use **"Request changes"** for blockers; **"Comment"** for suggestions
- Review within 1 business day of PR creation

---

## TypeScript Strict Mode

All `tsconfig.json` files must include:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```
