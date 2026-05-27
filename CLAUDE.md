# CLAUDE.md — n8n Codebase Guide for AI Assistants

## Overview

n8n is an open-source workflow automation platform. This is a large TypeScript monorepo managed with **pnpm workspaces** and **Turborepo**. The platform supports 300+ integrations, a visual workflow editor, and an AI-native workflow builder.

---

## Repository Structure

```
n8n/
├── packages/
│   ├── cli/                    # Main backend server (Express + oclif CLI)
│   ├── core/                   # Workflow execution engine
│   ├── workflow/               # Shared interfaces and expression engine
│   ├── nodes-base/             # 300+ built-in integration nodes
│   ├── node-dev/               # CLI tool for developing custom nodes
│   ├── frontend/               # All frontend packages
│   │   ├── editor-ui/          # Main Vue 3 workflow editor
│   │   ├── @n8n/design-system/ # Reusable Vue components
│   │   ├── @n8n/chat/          # Chat UI components
│   │   ├── @n8n/composables/   # Vue composables
│   │   ├── @n8n/rest-api-client/
│   │   ├── @n8n/i18n/
│   │   ├── @n8n/stores/        # Pinia stores
│   │   └── @n8n/codemirror-lang/
│   └── @n8n/                   # Scoped backend/shared packages
│       ├── api-types/          # TypeScript API contracts
│       ├── backend-common/     # Shared backend utilities
│       ├── config/             # Config management (Zod schema)
│       ├── constants/          # Shared constants
│       ├── db/                 # TypeORM database layer
│       ├── decorators/         # Custom TypeScript decorators
│       ├── di/                 # Dependency injection (tsyringe)
│       ├── permissions/        # RBAC permission system
│       ├── task-runner/        # Task execution runtime
│       ├── nodes-langchain/    # LangChain AI nodes
│       ├── ai-workflow-builder/# AI-powered workflow generation
│       ├── eslint-config/      # Shared ESLint config
│       ├── typescript-config/  # Shared tsconfig
│       └── vitest-config/      # Shared Vitest config
├── cypress/                    # E2E tests
├── docker/                     # Docker build configuration
├── .github/workflows/          # CI/CD pipelines
├── scripts/                    # Setup/build helper scripts
├── turbo.json                  # Turborepo task orchestration
├── pnpm-workspace.yaml         # pnpm workspace config
├── biome.jsonc                 # JS/TS/JSON formatter+linter
├── lefthook.yml                # Pre-commit hooks
└── vitest.workspace.ts         # Vitest workspace config
```

---

## Prerequisites

- **Node.js**: 22.16+
- **pnpm**: 10.2+ (use `corepack enable` to activate)
- Do **not** use `npm install` — a preinstall script blocks it

---

## Development Workflow

### Initial Setup

```bash
pnpm install
pnpm build
```

### Starting Development

```bash
pnpm dev          # Full stack (frontend + backend)
pnpm dev:be       # Backend only
pnpm dev:fe       # Frontend only
pnpm dev:ai       # AI workflow builder
```

The app runs on **http://localhost:5678** by default.

### Building

```bash
pnpm build              # Full build
pnpm build:backend      # Backend packages only
pnpm build:frontend     # Frontend packages only
pnpm build:nodes        # nodes-base only
```

---

## Testing

### Running Tests

```bash
pnpm test               # All unit tests (SQLite)
pnpm test:backend       # Backend unit tests (concurrency=1)
pnpm test:frontend      # Frontend unit tests
pnpm test:nodes         # Node-specific tests
```

### E2E Tests

```bash
pnpm dev:e2e            # Start dev server for E2E
pnpm test:e2e:dev       # Run E2E with hot reload
pnpm test:e2e:ui        # Run E2E against built UI
pnpm test:e2e:all       # Run E2E headless
```

### Test Frameworks

- **Jest** (v29) — primary unit test framework for backend packages
- **Vitest** (v3) — used for frontend packages
- **Cypress** — end-to-end tests in `cypress/`

### Database Variants for Tests

Set `DB_TYPE` to switch databases:
- `sqlite` (default)
- `postgresdb`
- `mysqldb`
- `mariadb`

---

## Code Quality

### Linting & Formatting

```bash
pnpm format             # Format all files (Biome + Prettier)
pnpm format:check       # Check formatting without writing
pnpm lint               # Lint all
pnpm lint:backend       # Backend lint only
pnpm lint:frontend      # Frontend lint only
pnpm lint:nodes         # Nodes lint only
pnpm lintfix            # Auto-fix lint issues
pnpm typecheck          # TypeScript type checking
```

### Tools

| Tool | Handles | Config |
|------|---------|--------|
| **Biome** v1.9 | `.js`, `.ts`, `.json` | `biome.jsonc` |
| **Prettier** v3 | `.vue`, `.yml`, `.md`, `.css`, `.scss` | `.prettierrc` |
| **ESLint** | TypeScript-specific rules | `packages/@n8n/eslint-config/` |

### Formatting Rules

- **Tabs** (width 2), not spaces
- **100 character** line width
- Single quotes for strings
- Trailing commas in multi-line structures
- Semicolons required

### Pre-commit Hooks (Lefthook)

On each commit:
- `biome check --write` runs on staged JS/TS/JSON files
- `prettier --write` runs on staged Vue/YAML/Markdown/CSS files
- Fixed files are auto-staged

---

## TypeScript Configuration

- **Target**: ES2021
- **Strict mode**: fully enabled (`strict`, `noUnusedLocals`, `noUnusedParameters`, `strictNullChecks`)
- **No `ts-ignore`**: enforced via lint rules
- Root extends `packages/@n8n/typescript-config/tsconfig.common.json`
- Each package has its own `tsconfig.json` and `tsconfig.build.json`
- Path aliases are resolved via `tsc-alias` after compilation

---

## Package Architecture

### Package Naming Conventions

- `n8n`, `n8n-core`, `n8n-workflow`, `n8n-nodes-base` — top-level core packages
- `@n8n/package-name` — scoped infrastructure/shared packages
- Frontend packages live under `packages/frontend/`

### Core Package Responsibilities

| Package | Role |
|---------|------|
| `packages/cli` | Backend HTTP server, REST API controllers, CLI commands (oclif), auth, webhooks |
| `packages/core` | Workflow execution engine, node runner, binary data handling |
| `packages/workflow` | Interfaces, types, expression engine — shared between front and back |
| `packages/nodes-base` | All standard integration nodes (300+) |
| `packages/@n8n/db` | TypeORM entities, migrations, repositories |
| `packages/@n8n/config` | All environment-based configuration via Zod schemas |
| `packages/@n8n/di` | Dependency injection container (tsyringe-based) |
| `packages/frontend/editor-ui` | Vue 3 + Vite workflow editor |

---

## Writing Nodes

Nodes live in `packages/nodes-base/nodes/{NodeName}/`. A standard node has:

```
nodes/{NodeName}/
├── {NodeName}.node.ts        # Main node class
├── {NodeName}.node.json      # Categories, docs URL, codex metadata
├── {NodeName}.svg            # Icon (light)
├── {NodeName}.dark.svg       # Icon (dark, optional)
├── {NodeName}Description.ts  # Operations/fields definitions (for complex nodes)
└── GenericFunctions.ts       # Shared API call helpers
```

### Node Class Structure

```typescript
import { IExecuteFunctions, INodeType, INodeTypeDescription } from 'n8n-workflow';

export class MyNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Node',
    name: 'myNode',
    icon: 'file:myNode.svg',
    group: ['transform'],
    version: 1,
    description: 'What this node does',
    defaults: { name: 'My Node' },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [...],
    properties: [...],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    // implementation
  }
}
```

### Node Conventions

- Operations are defined as `options` on a `resource` property
- Credentials are typed and declared in `packages/nodes-base/credentials/`
- Node categories: `Development`, `Finance & Accounting`, `Communication`, `Data & Storage`, etc.
- Use `GenericFunctions.ts` for shared API helpers within a node group
- Versioned nodes use `{NodeName}V1.node.ts`, `{NodeName}V2.node.ts`, etc.

---

## Backend Architecture (`packages/cli/src/`)

```
src/
├── commands/           # oclif CLI commands (start, webhook, worker)
├── controllers/        # REST API route handlers
├── services/           # Business logic (ActiveWorkflowManager, ExecutionService, etc.)
├── auth/               # JWT, OAuth2, LDAP authentication
├── databases/          # TypeORM entities, migrations, repositories
├── eventbus/           # Internal event system for workflow lifecycle
├── webhooks/           # Incoming webhook routing
├── credentials/        # Credential storage and encryption
└── middleware/         # Express middleware
```

Key patterns:
- **Dependency Injection** via `@n8n/di` (tsyringe) — use `@Service()` decorator
- **TypeORM** for all database access — entities in `packages/@n8n/db/`
- **Controllers** use `@RestController`, `@Get`, `@Post` decorators from `@n8n/decorators`

---

## Frontend Architecture (`packages/frontend/editor-ui/`)

- **Vue 3** + **Vite**
- **Pinia** for state management (`packages/frontend/@n8n/stores/`)
- **Element Plus** as UI component library
- **CodeMirror** for inline code editing
- **Monaco Editor** for advanced code panels

---

## Environment Variables

Configuration is managed via `packages/@n8n/config` with Zod schema validation. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_TYPE` | `sqlite` | Database: `sqlite`, `postgresdb`, `mysqldb`, `mariadb` |
| `DB_POSTGRESDB_SCHEMA` | `public` | Postgres schema |
| `DB_TABLE_PREFIX` | `` | Table name prefix |
| `N8N_LOG_LEVEL` | `info` | Log level: `error`, `warn`, `info`, `verbose`, `debug` |
| `NODE_ENV` | `development` | Environment mode |
| `N8N_PORT` | `5678` | HTTP port |

---

## CI/CD

### GitHub Actions Workflows (`.github/workflows/`)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci-master.yml` | Push to master | Full test suite on Node 20/22/24 |
| `ci-pull-requests.yml` | PR opened/updated | Lint, build, unit tests |
| `e2e-tests.yml` | PR / schedule | E2E test suite |
| `linting-reusable.yml` | Reusable | Biome + ESLint checks |
| `release-create-pr.yml` | Manual | Creates release PR with version bumps |
| `release-publish.yml` | Release tag | Publishes to npm + creates GitHub release |
| `docker-images-*.yml` | Release | Builds and pushes Docker images |

### Build Caching

- Turborepo remote cache enabled
- pnpm store cached between CI runs
- Build artifacts (`dist/`, `coverage/`) cached by Turbo

---

## Docker

```bash
# Build image
docker build -f docker/images/n8n/Dockerfile -t n8n .

# Run
docker run -p 5678:5678 n8n
```

Multi-stage Dockerfile:
1. **Builder**: Installs deps, runs `pnpm build`
2. **Production**: Minimal image, multi-arch (amd64 + arm64)
- Base: `n8nio/base:${NODE_VERSION}` (Node 22)
- Data volume: `/home/node`

---

## Common Patterns and Conventions

### Adding a New Package

1. Create `packages/@n8n/my-package/` with `package.json`, `tsconfig.json`, `tsconfig.build.json`
2. Reference `packages/@n8n/typescript-config` in tsconfig
3. Add to workspace — pnpm picks it up automatically
4. Wire into `turbo.json` if build caching is needed

### Making Database Changes

- Add TypeORM entities in `packages/@n8n/db/src/entities/`
- Write migrations in `packages/@n8n/db/src/migrations/`
- Migrations run automatically on startup in development

### Dependency Injection Pattern

```typescript
import { Service } from '@n8n/di';

@Service()
export class MyService {
  constructor(private readonly otherService: OtherService) {}
}
```

### REST Controller Pattern

```typescript
import { RestController, Get, Post } from '@/decorators';

@RestController('/my-resource')
export class MyController {
  @Get('/')
  async list() { ... }

  @Post('/')
  async create() { ... }
}
```

---

## What NOT to Do

- **Never** use `npm install` — use `pnpm`
- **Never** add `// @ts-ignore` — fix the type error properly
- **Never** skip pre-commit hooks with `--no-verify`
- **Never** push directly to `master` — open a PR
- **Don't** create new nodes without checking if one already exists for the service
- **Don't** add comments explaining what code does — use descriptive names instead; only comment the non-obvious *why*
- **Don't** introduce abstractions or error handling beyond what the task requires

---

## Useful Debugging Commands

```bash
# Check what's broken in a specific package
pnpm --filter @n8n/my-package typecheck

# Run a single test file
pnpm --filter n8n-core jest path/to/test.spec.ts

# Watch mode for a package
pnpm --filter n8n-workflow dev

# Lint only changed files
pnpm biome check --changed

# Inspect turbo task graph
pnpm turbo run build --dry-run
```

---

## Key External Resources

- Architecture docs: internal `docs/` (if present) and [docs.n8n.io](https://docs.n8n.io)
- Community: [community.n8n.io](https://community.n8n.io)
- Node development guide: [docs.n8n.io/integrations/creating-nodes/](https://docs.n8n.io/integrations/creating-nodes/)
- Contributing guide: `CONTRIBUTING.md` at repo root
