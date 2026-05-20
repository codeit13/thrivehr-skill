# CI/CD Standards

## Current: GitHub Actions

All CI/CD runs on **GitHub Actions**. Future plan: sync repo to Bitbucket, Jenkins pipelines with identical stages.

### Pipeline Stages

```
1. Lint
2. Unit Tests + Coverage Gate
3. SAST (Static Analysis)
4. Secret Scan
5. Dependency Scan
6. Docker Build
7. Integration / Smoke Tests
```

### Stage Details

#### 1. Lint

- **Node/FE**: `eslint .` + `prettier --check .`
- **Python**: `ruff check .` + `ruff format --check .`
- Fail the pipeline on any lint error

#### 2. Unit Tests + Coverage

- **Node**: `vitest run --coverage` — 80% minimum (statements, branches, functions, lines)
- **Python**: `pytest --cov --cov-fail-under=80`
- Coverage report uploaded as artifact

#### 3. SAST

- **SonarQube** analysis on every PR
- Quality gate: no new critical/blocker issues
- Track code smells and technical debt

#### 4. Secret Scan

- **GitLeaks** scans committed code for secrets
- Also runs as pre-commit hook (Husky)
- Fail pipeline on any detected secret

#### 5. Dependency Scan

- **Snyk** or **Trivy** for known vulnerabilities
- Block on critical/high severity CVEs
- Automated PR for dependency updates (Renovate or Dependabot)

#### 6. Docker Build

- Multi-stage Dockerfile (build → runtime)
- Minimal base image (e.g., `node:20-slim`, `python:3.12-slim`)
- Health check instruction in Dockerfile
- Image tagged with commit SHA + branch name

#### 7. Integration / Smoke Tests

- Spin up service with test database
- Hit health endpoints (`/health`, `/health/ready`)
- Run basic API smoke tests
- Tear down after completion

---

## Branch-Pipeline Mapping

| Branch | Trigger | Stages |
|--------|---------|--------|
| `feature/*`, `bugfix/*` | PR to `develop` | All 7 stages |
| `develop` | Push (after merge) | All 7 + deploy to dev environment |
| `uat` | Push (after merge from develop) | All 7 + deploy to UAT environment |
| `release` | Push (after merge from uat) | All 7 + deploy to production |
| `hotfix/*` | PR to `release` | All 7 stages |

---

## Planned: Bitbucket + Jenkins

### Migration Strategy

- GitHub repo syncs to Bitbucket via mirror
- Jenkins pipelines replicate the same 7 stages with identical semantics
- **Do not diverge** job logic between GitHub Actions and Jenkins
- Both pipelines should produce the same pass/fail for the same commit

### Jenkins Pipeline Structure

```
Jenkinsfile
├── stage('Lint')
├── stage('Test')
├── stage('SAST')
├── stage('Secret Scan')
├── stage('Dependency Scan')
├── stage('Docker Build')
└── stage('Integration Test')
```

---

## Docker Standards

- Multi-stage builds: separate build and runtime stages
- `.dockerignore`: exclude `node_modules`, `.git`, `tests/`, `.env`
- Health check: `HEALTHCHECK CMD curl -f http://localhost:${PORT}/health || exit 1`
- Image naming: `{servicename}-service:{branch}-{commit_sha}` (e.g. `job-service:develop-abc123`)
- No secrets baked into images — use runtime env/Vault injection
