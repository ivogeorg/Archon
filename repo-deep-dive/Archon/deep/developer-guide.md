---
generated_by: repo-deep-dive workflow
date: 2026-04-09
depth: deep
---

# Developer Guide

> **Archon** — Remote Agentic Coding Platform  
> Bun + TypeScript monorepo, single-developer focus, no multi-tenant complexity.

---

## Prerequisites

### Runtime

| Tool        | Required Version                                                            | Install                                     |
| ----------- | --------------------------------------------------------------------------- | ------------------------------------------- |
| **Bun**     | `^1.3.0` (CI pins `1.3.11`)                                                 | `curl -fsSL https://bun.sh/install \| bash` |
| **Node.js** | Not required for dev; needed only inside Docker for `agent-browser` install | —                                           |
| **Git**     | Any modern version                                                          | System package manager                      |

### CLI Tools

| Tool                           | Required For                                        | Install                                           |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------------- |
| **GitHub CLI (`gh`)**          | GitHub adapter, PR creation, workflow commands      | `brew install gh` / `sudo apt install gh`         |
| **Claude Code**                | Primary AI assistant integration, setup wizard      | `curl -fsSL https://claude.ai/install.sh \| bash` |
| **Docker**                     | Container builds, `docker-compose`-based PostgreSQL | [docker.com](https://docker.com)                  |
| **PostgreSQL client (`psql`)** | Optional: manual migration runs against Postgres    | System package manager                            |

### System Dependencies (Docker image reference)

The production Docker image installs: `curl`, `git`, `bash`, `ca-certificates`, `gnupg`, `gosu`, `postgresql-client`, `chromium`, and GitHub CLI. For local development only Bun + Git are strictly required.

### AI Credentials (choose one)

```
# Option A — Claude global auth (recommended)
claude /login
CLAUDE_USE_GLOBAL_AUTH=true   # in .env

# Option B — explicit tokens
CLAUDE_CODE_OAUTH_TOKEN=...   # or CLAUDE_API_KEY=...

# Codex (optional, if using OpenAI Codex)
# After `codex login`, copy tokens from ~/.codex/auth.json
CODEX_ID_TOKEN=...
CODEX_ACCESS_TOKEN=...
CODEX_REFRESH_TOKEN=...
CODEX_ACCOUNT_ID=...
```

---

## Setup: Zero to Running

### 1. Clone and Install

```bash
git clone https://github.com/coleam00/Archon
cd Archon
bun install
```

Bun resolves all workspace dependencies in one pass. Expected output ends with something like:

```
bun install v1.3.x (...)
+ 87 packages installed
```

### 2. Configure Environment

```bash
cp .env.example .env
```

Minimum viable `.env` for local development:

```bash
CLAUDE_USE_GLOBAL_AUTH=true
GH_TOKEN=<your-github-pat>      # optional — only needed for GitHub adapter
PORT=3090                        # default server port
```

Database defaults to SQLite at `~/.archon/archon.db` — no extra setup needed.

#### Optional: PostgreSQL

```bash
docker-compose --profile with-db up -d postgres
# Then add to .env:
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/remote_coding_agent
# Run migrations (one-time):
psql $DATABASE_URL < migrations/000_combined.sql
```

### 3. Start the Application

```bash
bun run dev        # starts server (port 3090) + web UI (port 5173) with hot reload
```

Or individually:

```bash
bun run dev:server   # backend only  — port 3090
bun run dev:web      # frontend only — port 5173
```

The server prints a health endpoint on startup. Verify with:

```bash
curl http://localhost:3090/api/health
# → {"status":"ok"}
```

### 4. Register a Project

Open `http://localhost:5173`, click **+** next to "Project", and enter a GitHub URL or local path.  
Or from the CLI inside any git repo:

```bash
bun run cli workflow list       # auto-registers the current repo
```

### Platform Quirks

- **Bun `--filter` log truncation** — `bun run dev` from the repo root truncates logs per package with `[N lines elided]`. For full server logs: `cd packages/server && bun --watch src/index.ts`.
- **Windows** — Bun is supported but CI also runs on `windows-latest`; watch for path separator issues. The install script uses PowerShell: `irm https://archon.diy/install.ps1 | iex`.
- **Docker linker** — The Dockerfile uses `--linker=hoisted` because Vite/Rollup cannot resolve symlinks created by Bun's default isolated linker. Do not remove that flag.

---

## Development Workflow

### Day-to-Day Commands

```bash
bun run dev                  # hot-reload server + web UI
bun run dev:server           # server only (hot-reload)
bun run dev:web              # web only (Vite HMR)

bun run type-check           # TypeScript strict check across all packages
bun run lint                 # ESLint (zero warnings)
bun run lint:fix             # ESLint with auto-fix
bun run format               # Prettier write
bun run format:check         # Prettier check (used in CI)

bun run validate             # type-check + lint + format:check + test (pre-PR gate)
```

### Port Defaults

| Context                | Port                                |
| ---------------------- | ----------------------------------- |
| Server (main repo)     | `3090`                              |
| Web UI (Vite)          | `5173`                              |
| Worktree server (auto) | `3190–4089` (hash of worktree path) |

Override: `PORT=4000 bun run dev:server`. The same worktree always gets the same port (deterministic hash).

### Regenerating Frontend API Types

The web package derives all types from the OpenAPI spec. After changing server routes:

```bash
bun run dev:server            # server must be running on port 3090
bun --filter @archon/web generate:types
```

This writes `packages/web/src/lib/api.generated.d.ts`. Commit it alongside route changes.

### Log Levels

```bash
LOG_LEVEL=debug bun run dev:server    # server verbose
bun run cli --verbose workflow run …  # CLI debug (tool-level events)
bun run cli --quiet  workflow run …   # CLI errors only
```

Default level: `info`. Levels: `fatal > error > warn > info > debug > trace`.

### Structured Logging

All packages use Pino via `createLogger('domain-name')` from `@archon/paths`. Event naming convention: `{domain}.{action}_{state}` — e.g., `workflow.step_started`, `session.create_failed`. Always pair `_started` with `_completed` or `_failed` and include relevant IDs.

---

## Testing

### Framework

**Bun's built-in test runner** (`bun test`). No Jest, no Vitest.

### Running Tests

```bash
# Full suite — always use this form from the repo root
bun run test

# Single file
bun test packages/core/src/handlers/command-handler.test.ts

# Watch mode (single package)
bun test --watch

# Per-package (if you are inside a package directory)
bun test src/
```

> **Critical:** Never run `bun test` from the repo root. It discovers all test files across packages in one process and causes ~135 mock pollution failures. `bun run test` uses `bun --filter '*' test` for per-package process isolation.

### Test Organization

Tests live alongside source files (`*.test.ts`). `bunfig.toml` sets `root = "./packages"` and preloads `packages/core/src/test/setup.ts` before every test run. Coverage output goes to `coverage/`.

### Mock Isolation — Critical Quirk

Bun's `mock.module()` is **process-global and irreversible** — `mock.restore()` does NOT undo it ([oven-sh/bun#7823](https://github.com/oven-sh/bun/issues/7823)).

Rules:

- Do **not** add `afterAll(() => mock.restore())` for `mock.module()` — it has no effect.
- Use `spyOn()` for internal modules where other test files also import the module directly. `spy.mockRestore()` **does** work.
- Never call `mock.module()` on the same path with different implementations in two test files that share a process.
- When adding a new test file that uses `mock.module()`, add it as a **separate** `bun test` invocation in that package's `package.json` `test` script.

### Per-Package Test Batching

Packages with conflicting `mock.module()` calls split their `test` scripts into sequential `bun test` invocations:

| Package             | Batches |
| ------------------- | ------- |
| `@archon/core`      | 7       |
| `@archon/workflows` | 5       |
| `@archon/adapters`  | 4       |
| `@archon/isolation` | 3       |

See each `package.json` `test` script for the exact file groupings.

### Mocking Strategy

- **Database** — mocked via `IDatabase` interface; test helpers provide in-memory or SQLite implementations.
- **AI SDKs** — mocked via `mock.module()` at the process level; async generators simulate streaming.
- **Platform adapters** — mocked objects implementing `IPlatformAdapter`.
- **Git operations** — `spyOn()` against `@archon/git` functions.
- **External HTTP** — mocked inline; no live network calls in unit tests.

### CI Matrix

Tests run on both `ubuntu-latest` and `windows-latest` in GitHub Actions (`.github/workflows/test.yml`). The CI also runs a Docker smoke test: builds the image, starts the container, and hits `/api/health`.

---

## Build & Packaging

### Web UI (Vite)

```bash
bun run build:web
# Output: packages/web/dist/
```

The built static assets are served by the Hono server in production.

### CLI Binaries

```bash
bun run build:binaries
# Output: dist/binaries/
```

The script (`scripts/build-binaries.sh`) patches `packages/paths/src/bundled-build.ts` to embed `BUNDLED_IS_BINARY=true`, `BUNDLED_VERSION`, and `BUNDLED_GIT_COMMIT` at compile time, then restores the file via an EXIT trap. This ensures the dev tree is never left dirty.

Targets built locally (all 4 by default):

| Binary                | Target             |
| --------------------- | ------------------ |
| `archon-darwin-arm64` | `bun-darwin-arm64` |
| `archon-darwin-x64`   | `bun-darwin-x64`   |
| `archon-linux-x64`    | `bun-linux-x64`    |
| `archon-linux-arm64`  | `bun-linux-arm64`  |

Windows (`bun-windows-x64`) is built in CI but not locally by default. `--bytecode` is excluded for Windows targets due to inconsistent Bun support. All targets use `--minify`.

CI mode (single target, invoked by GitHub Actions matrix):

```bash
TARGET=bun-linux-x64 OUTFILE=archon-linux-x64 bun run build:binaries
```

Binaries must be ≥ 1 MB (sanity check); typical size is ~50 MB+.

### Docker

```bash
# Build image locally
docker build -t archon:local .

# Multi-stage: deps → web-build → production
# Production stage: oven/bun:1.3.11-slim + git + gh CLI + chromium + agent-browser
```

The production image:

1. Installs only production dependencies (`--production --ignore-scripts`).
2. Copies pre-built web assets from the `web-build` stage.
3. Runs as non-root `appuser` (uid 1001) via `gosu` in the entrypoint.
4. Exposes port `3000` (configurable via `PORT`).

```bash
# With docker-compose (recommended for production)
docker-compose up -d

# With PostgreSQL
docker-compose --profile with-db up -d

# With Caddy reverse proxy + TLS
DOMAIN=archon.example.com docker-compose --profile cloud up -d
```

### Release Process

Use the `/release` skill (or the `release` CLI command). It:

1. Compares `dev` to `main`.
2. Generates changelog entries.
3. Bumps `version` in the root `package.json`.
4. Creates a PR from `dev` → `main`.

Version semantics: `/release` = patch, `/release minor` = minor, `/release major` = major.

CI (`.github/workflows/release.yml`) builds binaries on every `v*` tag push using a matrix of OS/target pairs and attaches them to the GitHub Release.

---

## Configuration Reference

### Environment Variables

#### Database

| Variable       | Type   | Default | Required | Purpose                                                            |
| -------------- | ------ | ------- | -------- | ------------------------------------------------------------------ |
| `DATABASE_URL` | string | —       | No       | PostgreSQL DSN. If unset, SQLite at `~/.archon/archon.db` is used. |

#### AI Assistants

| Variable                  | Type            | Default     | Required    | Purpose                                                                     |
| ------------------------- | --------------- | ----------- | ----------- | --------------------------------------------------------------------------- |
| `CLAUDE_USE_GLOBAL_AUTH`  | bool            | auto-detect | No          | `true` = use `claude /login` global auth; `false` = require explicit tokens |
| `CLAUDE_CODE_OAUTH_TOKEN` | string          | —           | Conditional | OAuth token for Claude Code                                                 |
| `CLAUDE_API_KEY`          | string          | —           | Conditional | API key for Claude                                                          |
| `CODEX_ID_TOKEN`          | string          | —           | Conditional | Codex ID token                                                              |
| `CODEX_ACCESS_TOKEN`      | string          | —           | Conditional | Codex access token                                                          |
| `CODEX_REFRESH_TOKEN`     | string          | —           | Conditional | Codex refresh token                                                         |
| `CODEX_ACCOUNT_ID`        | string          | —           | Conditional | Codex account ID                                                            |
| `DEFAULT_AI_ASSISTANT`    | `claude\|codex` | `claude`    | No          | Default assistant for new conversations                                     |
| `TITLE_GENERATION_MODEL`  | string          | SDK default | No          | Lightweight model for conversation title generation (e.g., `haiku`)         |

#### Server

| Variable                       | Type                                     | Default                           | Required | Purpose                       |
| ------------------------------ | ---------------------------------------- | --------------------------------- | -------- | ----------------------------- |
| `PORT`                         | number                                   | `3000` (Docker) / `3090` (source) | No       | HTTP server bind port         |
| `HOST`                         | string                                   | `0.0.0.0`                         | No       | HTTP server bind address      |
| `MAX_CONCURRENT_CONVERSATIONS` | number                                   | `10`                              | No       | Max parallel AI conversations |
| `LOG_LEVEL`                    | `fatal\|error\|warn\|info\|debug\|trace` | `info`                            | No       | Pino log level                |

#### GitHub

| Variable                    | Type         | Default            | Required            | Purpose                                      |
| --------------------------- | ------------ | ------------------ | ------------------- | -------------------------------------------- |
| `GH_TOKEN` / `GITHUB_TOKEN` | string       | —                  | For GitHub adapter  | GitHub PAT                                   |
| `WEBHOOK_SECRET`            | string       | —                  | For GitHub webhooks | HMAC secret for webhook verification         |
| `GITHUB_ALLOWED_USERS`      | string (CSV) | —                  | No                  | Comma-separated whitelist; empty = allow all |
| `GITHUB_BOT_MENTION`        | string       | `BOT_DISPLAY_NAME` | No                  | @mention name for issue/PR detection         |

#### Telegram

| Variable                    | Type            | Default  | Required             | Purpose                |
| --------------------------- | --------------- | -------- | -------------------- | ---------------------- |
| `TELEGRAM_BOT_TOKEN`        | string          | —        | For Telegram adapter | BotFather token        |
| `TELEGRAM_ALLOWED_USER_IDS` | string (CSV)    | —        | No                   | Whitelist of user IDs  |
| `TELEGRAM_STREAMING_MODE`   | `stream\|batch` | `stream` | No                   | Response delivery mode |

#### Slack

| Variable                 | Type            | Default | Required          | Purpose                       |
| ------------------------ | --------------- | ------- | ----------------- | ----------------------------- |
| `SLACK_BOT_TOKEN`        | string          | —       | For Slack adapter | Bot OAuth token               |
| `SLACK_APP_TOKEN`        | string          | —       | For Slack adapter | App-level token (Socket Mode) |
| `SLACK_ALLOWED_USER_IDS` | string (CSV)    | —       | No                | Whitelist of user IDs         |
| `SLACK_STREAMING_MODE`   | `stream\|batch` | `batch` | No                | Response delivery mode        |

#### Discord

| Variable                   | Type            | Default | Required            | Purpose                |
| -------------------------- | --------------- | ------- | ------------------- | ---------------------- |
| `DISCORD_BOT_TOKEN`        | string          | —       | For Discord adapter | discord.js bot token   |
| `DISCORD_ALLOWED_USER_IDS` | string (CSV)    | —       | No                  | Whitelist of user IDs  |
| `DISCORD_STREAMING_MODE`   | `stream\|batch` | `batch` | No                  | Response delivery mode |

#### GitLab / Gitea

| Variable                | Type         | Default              | Required            | Purpose                       |
| ----------------------- | ------------ | -------------------- | ------------------- | ----------------------------- |
| `GITLAB_URL`            | string       | `https://gitlab.com` | For GitLab adapter  | GitLab instance URL           |
| `GITLAB_TOKEN`          | string       | —                    | For GitLab adapter  | Personal/project access token |
| `GITLAB_WEBHOOK_SECRET` | string       | —                    | For GitLab webhooks | Webhook secret                |
| `GITLAB_ALLOWED_USERS`  | string (CSV) | —                    | No                  | Username whitelist            |
| `GITEA_URL`             | string       | —                    | For Gitea adapter   | Gitea instance URL            |
| `GITEA_TOKEN`           | string       | —                    | For Gitea adapter   | Personal access token         |
| `GITEA_WEBHOOK_SECRET`  | string       | —                    | For Gitea webhooks  | Webhook secret                |
| `GITEA_ALLOWED_USERS`   | string (CSV) | —                    | No                  | Username whitelist            |

#### Misc

| Variable                               | Type   | Default       | Required          | Purpose                                         |
| -------------------------------------- | ------ | ------------- | ----------------- | ----------------------------------------------- |
| `BOT_DISPLAY_NAME`                     | string | `Archon`      | No                | Display name in batch-mode messages             |
| `ARCHON_HOME`                          | string | `~/.archon`   | No                | Override base directory for all Archon files    |
| `ARCHON_DATA`                          | string | Docker volume | No                | Docker host path for persistent data            |
| `SESSION_RETENTION_DAYS`               | number | `30`          | No                | Auto-delete inactive sessions older than N days |
| `DOMAIN`                               | string | —             | Cloud profile     | Domain for Caddy TLS                            |
| `CADDY_BASIC_AUTH`                     | string | —             | No                | Caddy basicauth directive (bcrypt hash)         |
| `AUTH_USERNAME` / `AUTH_PASSWORD_HASH` | string | —             | Form auth profile | Form-based login credentials                    |
| `COOKIE_SECRET`                        | string | —             | Form auth profile | 64-hex-char cookie signing secret               |

### Config File: `.archon/config.yaml`

Lives in the repo root (repo-level) or `~/.archon/.archon/config.yaml` (global).

```yaml
assistants:
  claude:
    model: sonnet # sonnet | opus | haiku | claude-* | inherit
    settingSources:
      - project # only project-level CLAUDE.md
      - user # also load ~/.claude/CLAUDE.md

  codex:
    model: gpt-5.3-codex
    modelReasoningEffort: medium # minimal | low | medium | high | xhigh
    webSearchMode: live # disabled | cached | live
    additionalDirectories:
      - /absolute/path/to/other/repo

defaults:
  loadDefaultCommands: true # set false to disable bundled commands
  loadDefaultWorkflows: true # set false to disable bundled workflows

docs:
  path: docs # default; used for $DOCS_DIR variable
```

---

## Contributing Conventions

### Branch Naming

- Feature branches: cut from `dev`, e.g., `feat/add-gitlab-adapter`
- Bug fixes: `fix/workflow-resume-race-condition`
- Releases: automated PR from `dev` → `main` via `/release` skill

`main` is the release branch — never commit directly to it. All work goes to `dev` first.

### Commit Messages

- Present tense: `Add GitLab adapter`, not `Added GitLab adapter`
- First line ≤ 72 characters
- Reference issues when applicable: `Fix workflow resume (#234)`
- Conventional prefix encouraged: `feat:`, `fix:`, `chore:`, `docs:`

### Pre-Commit Hook

Husky runs `lint-staged` on every commit (`.husky/pre-commit` → `bun x lint-staged`).  
`.lintstagedrc.json` applies:

- `*.{ts,tsx}` → `eslint --fix --max-warnings 0` + `prettier --write`
- `*.{json,md,yaml,yml}` → `prettier --write`

This means malformed TypeScript or any ESLint warning **blocks the commit**.

### PR Process

1. Branch from `dev`.
2. Make changes.
3. Run `bun run validate` — all four checks (type-check, lint, format, test) must pass.
4. Submit PR targeting `dev` (not `main`).
5. Fill in the PR template (`.github/pull_request_template.md`).
6. CI runs the full matrix on `ubuntu-latest` + `windows-latest`.

### Code Style Enforcement

| Tool        | Config                                                                                      | Invocation             |
| ----------- | ------------------------------------------------------------------------------------------- | ---------------------- |
| TypeScript  | `tsconfig.json` (strict mode + `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`) | `bun run type-check`   |
| ESLint      | `eslint.config.mjs`                                                                         | `bun run lint`         |
| Prettier    | `.prettierrc`                                                                               | `bun run format:check` |
| lint-staged | `.lintstagedrc.json`                                                                        | auto on commit         |

**ESLint zero-tolerance policy**: CI enforces `--max-warnings 0`. Inline `// eslint-disable-next-line` is acceptable only for incorrect external SDK types or intentional validated type assertions — never to make CI pass.

### TypeScript Rules

- `strict: true` always.
- All functions must have explicit return types.
- No `any` without a comment explaining why.
- Use `import type` for type-only imports.
- Derive types with `z.infer<typeof schema>` — never hand-craft parallel interfaces.
- Import `z` from `@hono/zod-openapi`, not `zod` directly.

---

## Common Failure Modes

### 1. `bun test` from repo root causes ~135 failures

**Symptom:** Hundreds of test failures immediately, often with cryptic "module already mocked" or wrong-implementation errors.  
**Cause:** `bun test` at the root runs all test files in one process; `mock.module()` is irreversible and bleeds across files.  
**Fix:** Always use `bun run test` (the workspace script), which runs each package in its own `bun test` process.

---

### 2. TypeScript error after adding a new API route

**Symptom:** `@archon/web` type errors referencing `WorkflowDefinition`, `DagNode`, etc.  
**Cause:** The web package derives all types from a generated file (`api.generated.d.ts`), not from `@archon/workflows` directly.  
**Fix:** Start the server (`bun run dev:server`), then run `bun --filter @archon/web generate:types` to regenerate the types file. Commit the updated file.

---

### 3. `ESLint: X warnings` blocks CI

**Symptom:** CI lint step fails with "X warnings found, 0 allowed."  
**Cause:** A new `any`, unused variable, or other lint issue was introduced. Inline disables are not allowed without justification.  
**Fix:** Run `bun run lint:fix` locally. If the issue is an unavoidable SDK type problem, add a narrow `// eslint-disable-next-line` with an explanatory comment.

---

### 4. SQLite database not initialised / missing tables

**Symptom:** `no such table: remote_agent_conversations` or similar on first run.  
**Cause:** SQLite auto-initialises on first use, but something prevented the migration from running (permissions, wrong `ARCHON_HOME`).  
**Fix:** Check `~/.archon/archon.db` exists and is writable. If using a custom `ARCHON_HOME`, ensure the directory exists. For PostgreSQL, run `psql $DATABASE_URL < migrations/000_combined.sql` explicitly.

---

### 5. Worktree server port collides

**Symptom:** `address already in use` when starting a worktree dev server.  
**Cause:** Another process (or a previous run) holds the deterministically-allocated port.  
**Fix:** `PORT=<free-port> bun run dev:server`. Or kill the occupying process: `lsof -ti :<port> | xargs kill`.

---

### 6. Docker build fails: "Web build produced no index.html"

**Symptom:** Docker build exits in the `web-build` stage with the error above.  
**Cause:** The Vite build failed silently, or a TypeScript error in `@archon/web` stopped the build.  
**Fix:** Run `bun run build:web` locally to see the actual Vite/TypeScript error. Fix the error, then retry the Docker build.

---

### 7. `mock.restore()` does not undo module mocks — tests still see stale implementation

**Symptom:** A test that explicitly calls `mock.restore()` in `afterAll` still sees a mocked module in a later test file.  
**Cause:** Bun's `mock.restore()` only resets `spyOn` spies, not `mock.module()` replacements.  
**Fix:** Remove the `afterAll(() => mock.restore())` call. Move the conflicting test file into its own `bun test` invocation in `package.json`. Use `spyOn()` instead of `mock.module()` where possible.

---

### 8. `git clean -fd` accidentally used — untracked files lost

**Symptom:** Files that were not yet committed disappear from the working tree.  
**Cause:** `git clean -fd` is explicitly prohibited in this project but may be run by habit or a tool.  
**Fix:** Recover from git stash, reflog, or IDE local history. Going forward, use `git checkout .` to discard unstaged changes to tracked files only. Never run `git clean -fd` in this codebase.
