# ThriveHR Agent Skill

Team-shared coding standards for the ThriveHR (Axis Bank Hiring & Onboarding) project. This skill ensures all AI-generated code follows project conventions across Fastify, FastAPI, React, Prisma, and AI orchestrator services.

## Installation

### Cursor IDE

Copy this folder to one of the following locations:

**Project-level** (shared via repo):
```
cp -r thrivehr-skill/ .cursor/skills/thrivehr/
```

**User-level** (personal, all projects):
```
cp -r thrivehr-skill/ ~/.cursor/skills/thrivehr/
```

Verify: open Cursor Settings (Cmd+Shift+J) → Rules → Skills section.

### Claude Code

```
cp -r thrivehr-skill/ ~/.claude/skills/thrivehr/
```

### Codex

```
cp -r thrivehr-skill/ ~/.codex/skills/thrivehr/
```

## Usage

- The skill activates automatically when the agent detects relevant context (`TM_EXT_`, ThriveHR, Fastify, FastAPI, React, Prisma, etc.)
- Invoke manually by typing `/thrivehr` in Agent chat
- Reference files under `references/` are loaded on demand — only when the task needs that domain

### Viewing `SKILL.md`

`SKILL.md` is an **Agent Skill**, not a normal doc page. It starts with YAML frontmatter (`name`, `description`) that Cursor/Claude read for discovery. VS Code **Markdown Preview** shows that block as plain text — that is expected. For reading, use the editor view or scroll past the frontmatter to the `# ThriveHR` heading and tables below.

## Structure

```
thrivehr/
├── SKILL.md                  # Hub — stack, non-negotiables, reference index
├── README.md                 # This file
├── references/
│   ├── core.md               # Naming, settings table, API standards, Swagger auth
│   ├── frontend.md           # React, shadcn/ui, responsive UI, light/dark themes, UX
│   ├── accessibility.md    # WCAG 2.1 AA, ARIA, keyboard, contrast, a11y testing
│   ├── backend.md            # Fastify, Prisma, Zod, DB standards, project structure
│   ├── ai.md                 # AI inventory, orchestrator APIs, settings keys, Qdrant, security
│   ├── testing.md            # Vitest, RTL, Supertest, Pytest, coverage
│   ├── cicd.md               # GitHub Actions, future Jenkins pipeline
│   └── code-quality.md       # ESLint, Prettier, Husky, git branching, PR template
└── assets/
    ├── axios-client-template.md
    ├── accessibility-setup-template.md   # ESLint jsx-a11y, vitest-axe, Playwright axe
    ├── skip-to-content-template.md
    ├── index-html-a11y-template.md
    ├── fastify-swagger-auth-template.md
    └── fastapi-swagger-auth-template.md
```

## Updating

Edit the markdown files directly. Keep `SKILL.md` under 500 lines (hub only). Move detailed content to `references/`.
