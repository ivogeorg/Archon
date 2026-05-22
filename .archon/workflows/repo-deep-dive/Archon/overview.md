# Archon — Overview

> Generated from codebase at commit `5b250a4c` (branch `repo-deep-dive`)

---

## What It Does

Archon is a deterministic workflow engine for AI coding agents. It wraps Claude (via `@anthropic-ai/claude-agent-sdk`) and Codex (via `@openai/codex-sdk`) in structured YAML-defined DAG workflows so that AI-assisted development processes are repeatable, auditable, and portable.

**Concrete capabilities:**

- **Run multi-step AI coding workflows** — Define `plan → implement → validate → review → PR` as a YAML file, invoke with a natural-language message, and let the engine execute each node in order. Example: `archon workflow run archon-fix-github-issue "Fix login redirect bug"`.

- **Parallel DAG node execution** — Independent nodes in the same topological layer run concurrently via `Promise.allSettled`. Example: five PR reviewers (`archon-comprehensive-pr-review`) run in parallel and results are synthesized.

- **Git worktree isolation** — Every workflow run gets an isolated git worktree under `~/.archon/workspaces/{owner}/{repo}/worktrees/`. Running `archon workflow run implement --branch feature-auth "Add auth"` creates branch `feature-auth` in a worktree so the main checkout is never touched.

- **Loop nodes with human approval gates** — A `loop` node type iterates until a completion signal (e.g., `APPROVED`, `ALL_TASKS_COMPLETE`). `interactive: true` loops pause the workflow and wait for `archon workflow approve <run-id>` from a human.

- **Resume failed runs** — A run that fails mid-DAG records completed node outputs. Re-invoking the same workflow on the same worktree detects the prior run and skips already-completed nodes, continuing from the failure point.

- **Bash nodes (deterministic steps)** — A `bash:` node type runs a shell script with no AI involvement; stdout is captured as `$nodeId.output` for use in downstream prompts. Example: `bash: "bun run validate"` after an `implement` node.

- **Router AI for workflow selection** — When a user sends a natural-language message, a Claude routing call (with `tools: []` to prevent tool use) reads all registered projects and discovered workflows and emits `/invoke-workflow <name> --project <project> --prompt "..."`. This eliminates the need for users to remember workflow names.

- **Multi-platform message delivery** — The same workflow engine is reachable from Web UI (SSE streaming), CLI (stdout), Slack (Socket Mode), Telegram (Bot API polling), Discord (discord.js WebSocket), GitHub (webhooks), GitLab (webhooks), and Gitea (webhooks). All platforms share one DB and one orchestrator.

- **Per-project environment variable injection** — Env vars configured via Web UI (`codebase_env_vars` table) or `.archon/config.yaml`'s `env:` section are merged into the Claude/Codex subprocess environment at workflow execution time. Sensitive keys in the target repo's `.env` are blocked by the env-leak gate unless consent is explicitly granted.

- **Structured output from AI nodes** — Nodes with `output_format: {type: object, ...}` instruct the Claude API to return validated JSON, which is captured and available as `$nodeId.output` in downstream prompts.

- **17 bundled default workflows** embedded in the binary:

  | Workflow                         | Purpose                                                      |
  | -------------------------------- | ------------------------------------------------------------ |
  | `archon-assist`                  | General Q&A / full Claude Code agent                         |
  | `archon-fix-github-issue`        | Issue → investigate → implement → validate → PR → review     |
  | `archon-idea-to-pr`              | Idea → plan → implement → validate → PR → 5 parallel reviews |
  | `archon-plan-to-pr`              | Execute plan → implement → validate → PR → review            |
  | `archon-issue-review-full`       | Full fix + multi-agent review pipeline                       |
  | `archon-smart-pr-review`         | Classify PR → targeted review agents → synthesize            |
  | `archon-comprehensive-pr-review` | 5 parallel reviewers + auto-fix                              |
  | `archon-create-issue`            | Classify → investigate → create GitHub issue                 |
  | `archon-validate-pr`             | Test both main and feature branches                          |
  | `archon-resolve-conflicts`       | Detect → analyze → resolve → validate → commit               |
  | `archon-feature-development`     | Implement feature → validate → PR                            |
  | `archon-architect`               | Architectural sweep + complexity reduction                   |
  | `archon-refactor-safely`         | Refactor with type-check hooks + behavior verification       |
  | `archon-ralph-dag`               | PRD story loop until all stories done                        |
  | `archon-remotion-generate`       | Generate/modify Remotion video compositions                  |
  | `archon-test-loop-dag`           | Loop node smoke test                                         |
  | `archon-piv-loop`                | Plan → Implement → Validate with human review gates          |

- **Web dashboard** (`@archon/web`) — React + Vite + Tailwind v4 + shadcn/ui SPA with chat, mission-control dashboard, drag-and-drop workflow builder, and real-time step-by-step execution view. Served as static files from `packages/server`.

- **Workflow validation** — `archon validate workflows [name]` (or `POST /api/workflows/validate`) validates a YAML definition in memory against Zod schemas without writing any files. `archon validate commands [name]` validates command markdown files.

- **Codebase management** — Register a repo (local path or GitHub URL) via `POST /api/codebases`. The system clones it, sets up `~/.archon/workspaces/{owner}/{repo}/source/`, and auto-detects commands in `.archon/commands/`.

---

## Architecture

### Package Dependency Graph

```
┌─────────────────────────────────────────────────────────────────────┐
│  Consumers                                                          │
│  @archon/cli       @archon/server      @archon/web (browser only)  │
│       │                  │                      │                   │
│       └──────────────────┼──────────────────────┘                  │
│                          │ depends on                               │
├──────────────────────────▼──────────────────────────────────────────┤
│  @archon/adapters (Slack, Telegram, GitHub, Discord, GitLab, Gitea) │
│       │ depends on                                                  │
├───────▼─────────────────────────────────────────────────────────────┤
│  @archon/core (orchestrator, AI clients, DB, config, sessions)      │
│  depends on: @archon/workflows, @archon/isolation, @archon/git,     │
│              @archon/paths                                          │
├──────────────────────────────────────────────────────────────────── ┤
│  @archon/workflows (loader, executor, DAG, router, defaults)        │
│  depends on: @archon/git, @archon/paths (NO @archon/core)           │
├──────────────────────────────────────────────────────────────────── ┤
│  @archon/isolation (worktree provider, resolver, error classifiers) │
│  depends on: @archon/git, @archon/paths                             │
├──────────────────────────────────────────────────────────────────── ┤
│  @archon/git (branch, worktree, repo, exec wrappers)               │
│  depends on: @archon/paths                                          │
├──────────────────────────────────────────────────────────────────── ┤
│  @archon/paths (path resolution, Pino logger factory)               │
│  NO @archon/* dependencies                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Processing Flow

```
User message (any platform)
        │
        ▼
ConversationLockManager.acquireLock()          ← max 10 concurrent (configurable)
        │
        ▼
handleMessage()  [packages/core/src/orchestrator/orchestrator-agent.ts]
        │
        ├── Deterministic gate: /help /status /reset /workflow /worktree etc. (10 cmds)
        │         └── handleCommand() → CommandResult → platform.sendMessage()
        │
        └── Everything else → AI routing call (Claude with tools:[])
                  │
                  ├── Lists: codebases from DB + workflows from filesystem
                  ├── Builds prompt: buildOrchestratorPrompt() or buildProjectScopedPrompt()
                  ├── AI emits: /invoke-workflow <name> --project <p> --prompt "..."
                  └── parseOrchestratorCommands() → dispatchOrchestratorWorkflow()
                            │
                            ▼
                   validateAndResolveIsolation()   ← WorktreeProvider creates git worktree
                            │
                            ▼
                   executeWorkflow()  [packages/workflows/src/executor.ts]
                            │
                            ▼
                   executeDagWorkflow()  [packages/workflows/src/dag-executor.ts]
                            │
                            ├── Build topological layers
                            ├── Run independent nodes concurrently (Promise.allSettled)
                            ├── For each node:
                            │     ├── bash: → execFileAsync, capture stdout
                            │     ├── prompt:/command: → IAssistantClient.run()
                            │     └── loop: → iterate until completion signal
                            └── Emit WorkflowEvent to DB + SSE bridge on each transition
```

### Data Model (8 tables, all prefixed `remote_agent_`)

| Table                    | Key Columns                                                                                                     |
| ------------------------ | --------------------------------------------------------------------------------------------------------------- |
| `codebases`              | `id`, `name` (owner/repo), `default_cwd`, `ai_assistant_type`, `allow_env_keys`, `commands` JSONB               |
| `codebase_env_vars`      | `codebase_id`, `key`, `value` — injected into subprocess env                                                    |
| `conversations`          | `platform_type`, `platform_conversation_id`, `codebase_id`, `isolation_env_id`, `title`, `deleted_at`, `hidden` |
| `sessions`               | `conversation_id`, `sdk_session_id`, `parent_session_id`, `transition_reason`, `active`                         |
| `isolation_environments` | `codebase_id`, `worktree_path`, `branch_name`, `workflow_type`, `workflow_id`, `status`                         |
| `workflow_runs`          | `workflow_name`, `conversation_id`, `status`, `working_path`, `metadata` JSONB, `last_activity_at`              |
| `workflow_events`        | `workflow_run_id`, `event_type`, `step_name`, `data` JSONB                                                      |
| `messages`               | `conversation_id`, `role`, `content`, `metadata` JSONB (tool calls)                                             |

---

## Public Surface

### HTTP/REST API (default port 3090)

All endpoints served by `@archon/server` via Hono + `@hono/zod-openapi`. OpenAPI spec at `GET /api/openapi.json`.

**Conversations**

| Method   | Path                              | Purpose                                                               |
| -------- | --------------------------------- | --------------------------------------------------------------------- |
| `GET`    | `/api/conversations`              | List conversations (supports `?platform=`, `?codebaseId=`, `?limit=`) |
| `POST`   | `/api/conversations`              | Create conversation; returns `{ id, ... }`                            |
| `GET`    | `/api/conversations/:id`          | Get single conversation                                               |
| `PATCH`  | `/api/conversations/:id`          | Update title, soft-delete                                             |
| `DELETE` | `/api/conversations/:id`          | Soft-delete                                                           |
| `POST`   | `/api/conversations/:id/message`  | Send message; body: `{ message, attachedFiles? }`                     |
| `GET`    | `/api/conversations/:id/messages` | Fetch message history                                                 |
| `GET`    | `/api/stream/:id`                 | SSE stream — emits `{ type, data }` JSON events                       |

**Workflows**

| Method   | Path                                 | Purpose                                                                   |
| -------- | ------------------------------------ | ------------------------------------------------------------------------- |
| `GET`    | `/api/workflows`                     | List workflows; optional `?cwd=`; returns `{ workflows, errors? }`        |
| `POST`   | `/api/workflows/validate`            | Validate YAML in-memory; body: `{ definition }`                           |
| `GET`    | `/api/workflows/:name`               | Fetch single workflow + source (`project` \| `bundled`)                   |
| `PUT`    | `/api/workflows/:name`               | Save (create or update) workflow YAML; requires `?cwd=`                   |
| `DELETE` | `/api/workflows/:name`               | Delete user-defined workflow (bundled workflows cannot be deleted)        |
| `POST`   | `/api/workflows/runs`                | Trigger a workflow run; body: `{ workflowName, conversationId, message }` |
| `GET`    | `/api/workflows/runs`                | List workflow runs; supports `?status=`, `?codebaseId=`, date filters     |
| `GET`    | `/api/workflows/runs/:runId`         | Get run detail including events                                           |
| `POST`   | `/api/workflows/runs/:runId/resume`  | Mark failed run ready for auto-resume                                     |
| `POST`   | `/api/workflows/runs/:runId/abandon` | Mark non-terminal run as cancelled                                        |
| `POST`   | `/api/workflows/runs/:runId/approve` | Approve interactive loop gate; body: `{ comment? }`                       |
| `POST`   | `/api/workflows/runs/:runId/reject`  | Reject approval gate; body: `{ reason? }`                                 |
| `POST`   | `/api/workflows/runs/:runId/cancel`  | Cancel running workflow                                                   |
| `DELETE` | `/api/workflows/runs/:runId`         | Delete terminal run record                                                |

**Codebases**

| Method   | Path                 | Purpose                                                                       |
| -------- | -------------------- | ----------------------------------------------------------------------------- |
| `GET`    | `/api/codebases`     | List registered codebases                                                     |
| `GET`    | `/api/codebases/:id` | Get single codebase                                                           |
| `POST`   | `/api/codebases`     | Register codebase (clone or local path); body: `{ url\|path, allowEnvKeys? }` |
| `PATCH`  | `/api/codebases/:id` | Update `allow_env_keys` consent; body: `{ allowEnvKeys: boolean }`            |
| `DELETE` | `/api/codebases/:id` | Delete codebase and clean up resources                                        |

**Artifacts, Commands, Config**

| Method  | Path                      | Purpose                                             |
| ------- | ------------------------- | --------------------------------------------------- |
| `GET`   | `/api/artifacts/:runId/*` | Serve artifact file by run ID + relative path       |
| `GET`   | `/api/commands`           | List command names; optional `?cwd=`                |
| `GET`   | `/api/config`             | Get safe config (model, assistant, streaming prefs) |
| `PATCH` | `/api/config`             | Update global config fields                         |

**Health**

| Method | Path                  | Purpose                                                      |
| ------ | --------------------- | ------------------------------------------------------------ |
| `GET`  | `/health`             | `{ status: "ok" }`                                           |
| `GET`  | `/health/db`          | Database connectivity check                                  |
| `GET`  | `/health/concurrency` | Lock manager stats: `{ active, queuedTotal, maxConcurrent }` |

**Webhooks**

| Method | Path               | Purpose                                                                 |
| ------ | ------------------ | ----------------------------------------------------------------------- |
| `POST` | `/webhooks/github` | GitHub issue/PR comment events; verified via HMAC `X-Hub-Signature-256` |
| `POST` | `/webhooks/gitea`  | Gitea events; verified via `X-Gitea-Signature`                          |
| `POST` | `/webhooks/gitlab` | GitLab events; verified via `X-Gitlab-Token`                            |

### SSE Event Types

The SSE stream at `/api/stream/:conversationId` delivers server-sent events with JSON data:

| `type`                                    | When emitted                                            |
| ----------------------------------------- | ------------------------------------------------------- |
| `message`                                 | New chat message (assistant text chunk or full message) |
| `workflow_started`                        | Workflow run begins                                     |
| `workflow_completed`                      | Workflow run completes                                  |
| `workflow_failed`                         | Workflow run fails                                      |
| `node_started`                            | Individual DAG node begins                              |
| `node_completed`                          | Individual DAG node completes                           |
| `node_failed`                             | Individual DAG node fails                               |
| `node_skipped`                            | Node skipped (condition false)                          |
| `loop_iteration_started/completed/failed` | Loop node iteration                                     |
| `tool_called / tool_completed`            | AI tool invocation                                      |
| `approval_requested`                      | Interactive loop paused waiting for human               |
| `approval_received`                       | Approval gate cleared                                   |
| `workflow_cancelled`                      | Run was cancelled                                       |

### CLI Commands

Binary: `archon` (or `bun run cli` from repo root). Requires being inside a git repository for `workflow` and `isolation` commands.

```bash
# Workflow
archon workflow list [--json]
archon workflow run <name> [message] [--branch <b>] [--from <base>] [--no-worktree] [--resume] [--allow-env-keys] [--cwd <path>]
archon workflow status [--json] [--verbose]
archon workflow resume <run-id>
archon workflow abandon <run-id>
archon workflow approve <run-id> [comment]
archon workflow reject <run-id> [--reason "..."]
archon workflow cleanup [days]                # default 7; deletes terminal runs
archon workflow event emit --run-id <uuid> --type <event-type> [--data <json>]

# Isolation / worktrees
archon isolation list
archon isolation cleanup [days]              # default 7; removes stale worktrees
archon isolation cleanup --merged [--include-closed]
archon complete <branch> [--force]           # remove worktree + local + remote branch

# Validation
archon validate workflows [name] [--json]
archon validate commands [name] [--json]

# Misc
archon chat <message>
archon continue <branch> [message] [--workflow <name>] [--no-context]
archon setup [--spawn]
archon version
archon help
```

Global flags: `--cwd <path>`, `--quiet / -q`, `--verbose / -v`.

### Workflow YAML Schema

A workflow file (`.archon/workflows/<name>.yaml`) has these top-level fields:

```yaml
name: my-workflow
description: |
  What this workflow does.
  Use when: ...
provider: claude # or codex; inferred from model if omitted
model: sonnet # or opus, haiku, claude-*, gpt-5.3-codex, inherit
interactive: false # true forces foreground execution in web UI
effort: medium # EffortLevel: minimal | low | medium | high | xhigh (Codex)
systemPrompt: '...' # Prepended system prompt for all nodes
nodes:
  - id: plan
    prompt: '...' # Inline prompt
    # OR
    command: my-command # Named .archon/commands/my-command.md
    # OR
    bash: 'bun run validate' # Shell script; stdout → $plan.output
    # OR
    loop:
      prompt: '...'
      until: APPROVED # Completion signal
      max_iterations: 10
      interactive: true # Pause and wait for human approval

    depends_on: [plan] # DAG edges
    when: "$plan.output.type == 'BUG'" # Skip condition
    trigger_rule: all_success # all_success | one_success | all_done | none_failed_min_one_success
    output_format: # JSON Schema for structured output (Claude only)
      type: object
      properties:
        type: { type: string }
    allowed_tools: [Read, Edit] # Claude tool allowlist (omit = all tools)
    denied_tools: [Bash] # Claude tool denylist
    model: opus # Per-node model override
    provider: claude # Per-node provider override
    idle_timeout: 300000 # ms; default 120000
    context: fresh # New AI session for this node
    hooks: # Claude SDK PostToolUse hooks
      PostToolUse:
        - matcher: { tool_name: Bash }
          hooks: [{ type: command, command: '...' }]
    mcp: # MCP config file path (Claude only)
      config_path: .archon/mcp.json
    skills: [archon] # AgentDefinition skill names
```

### Configuration

**Environment Variables (`.env` in repo root)**

| Variable                       | Type            | Default              | Purpose                                                               |
| ------------------------------ | --------------- | -------------------- | --------------------------------------------------------------------- |
| `DATABASE_URL`                 | string          | unset                | PostgreSQL connection string; unset = SQLite at `~/.archon/archon.db` |
| `CLAUDE_USE_GLOBAL_AUTH`       | boolean         | `true` (CLI default) | Use `claude /login` OAuth                                             |
| `CLAUDE_CODE_OAUTH_TOKEN`      | string          | unset                | Explicit Claude OAuth token                                           |
| `CLAUDE_API_KEY`               | string          | unset                | Anthropic API key                                                     |
| `CODEX_ID_TOKEN`               | string          | required for Codex   | Codex ID token                                                        |
| `CODEX_ACCESS_TOKEN`           | string          | required for Codex   | Codex access token                                                    |
| `CODEX_REFRESH_TOKEN`          | string          | unset                | Codex refresh token                                                   |
| `CODEX_ACCOUNT_ID`             | string          | unset                | Codex account ID                                                      |
| `DEFAULT_AI_ASSISTANT`         | `claude\|codex` | `claude`             | Default assistant for new conversations                               |
| `TITLE_GENERATION_MODEL`       | string          | SDK default          | Lightweight model for title generation (e.g. `haiku`)                 |
| `GH_TOKEN` / `GITHUB_TOKEN`    | string          | unset                | GitHub API + webhook auth                                             |
| `WEBHOOK_SECRET`               | string          | required for GitHub  | HMAC secret for GitHub webhook verification                           |
| `GITHUB_ALLOWED_USERS`         | string          | unset                | Comma-separated whitelist of GitHub usernames                         |
| `GITHUB_BOT_MENTION`           | string          | `BOT_DISPLAY_NAME`   | @mention trigger in GitHub issues                                     |
| `GITLAB_TOKEN`                 | string          | required for GitLab  | GitLab Personal Access Token                                          |
| `GITLAB_WEBHOOK_SECRET`        | string          | required for GitLab  | GitLab webhook secret                                                 |
| `GITLAB_URL`                   | string          | `https://gitlab.com` | Self-hosted GitLab URL                                                |
| `GITLAB_ALLOWED_USERS`         | string          | unset                | Comma-separated GitLab username whitelist                             |
| `GITEA_URL`                    | string          | required for Gitea   | Gitea instance URL                                                    |
| `GITEA_TOKEN`                  | string          | required for Gitea   | Gitea personal access token                                           |
| `GITEA_WEBHOOK_SECRET`         | string          | required for Gitea   | Gitea webhook secret                                                  |
| `GITEA_ALLOWED_USERS`          | string          | unset                | Comma-separated Gitea username whitelist                              |
| `TELEGRAM_BOT_TOKEN`           | string          | unset                | Telegram Bot API token                                                |
| `TELEGRAM_ALLOWED_USER_IDS`    | string          | unset                | Comma-separated Telegram user ID whitelist                            |
| `TELEGRAM_STREAMING_MODE`      | `stream\|batch` | `stream`             | Message delivery mode                                                 |
| `DISCORD_BOT_TOKEN`            | string          | unset                | Discord bot token                                                     |
| `DISCORD_ALLOWED_USER_IDS`     | string          | unset                | Comma-separated Discord user ID whitelist                             |
| `DISCORD_STREAMING_MODE`       | `stream\|batch` | `batch`              | Message delivery mode                                                 |
| `SLACK_BOT_TOKEN`              | string          | unset                | Slack bot token                                                       |
| `SLACK_APP_TOKEN`              | string          | unset                | Slack app-level token (Socket Mode)                                   |
| `SLACK_ALLOWED_USER_IDS`       | string          | unset                | Comma-separated Slack user ID whitelist                               |
| `SLACK_STREAMING_MODE`         | `stream\|batch` | `batch`              | Message delivery mode                                                 |
| `BOT_DISPLAY_NAME`             | string          | `Archon`             | Bot display name in messages                                          |
| `PORT`                         | number          | `3090`               | HTTP server port (worktrees auto-allocate in 3190–4089 range)         |
| `HOST`                         | string          | `0.0.0.0`            | HTTP server bind address                                              |
| `DOMAIN`                       | string          | unset                | Public domain for Caddy TLS (cloud profile)                           |
| `MAX_CONCURRENT_CONVERSATIONS` | number          | `10`                 | Conversation lock manager capacity                                    |
| `SESSION_RETENTION_DAYS`       | number          | `30`                 | Days before inactive sessions are purged                              |
| `LOG_LEVEL`                    | string          | `info`               | Pino log level: `fatal\|error\|warn\|info\|debug\|trace`              |
| `ARCHON_HOME`                  | string          | `~/.archon`          | Override Archon home directory                                        |
| `ARCHON_DATA`                  | string          | Docker volume        | Docker host path for Archon data                                      |

**Config Files**

`~/.archon/config.yaml` (global) and `.archon/config.yaml` (repo-level) use this structure:

```yaml
# Repo-level: .archon/config.yaml
assistant: claude # claude | codex
assistants:
  claude:
    model: sonnet # sonnet | opus | haiku | claude-* | inherit
    settingSources: [project] # project | user
  codex:
    model: gpt-5.3-codex
    modelReasoningEffort: medium
    webSearchMode: live
commands:
  folder: .archon/commands # custom command folder
  autoLoad: true
worktree:
  baseBranch: main
  copyFiles: ['.env', '.archon'] # git-ignored files to copy to new worktrees
docs:
  path: docs/
defaults:
  loadDefaultCommands: true
  loadDefaultWorkflows: true
allow_target_repo_keys: false # bypass env-leak gate (USE WITH CAUTION)
env:
  MY_VAR: value # injected into subprocess env
```

---

## Integration Guide

### Prerequisites and Setup

```bash
# Install Bun (runtime)
curl -fsSL https://bun.sh/install | bash

# Install GitHub CLI (needed for PR workflows)
brew install gh && gh auth login

# Install Claude Code CLI
curl -fsSL https://claude.ai/install.sh | bash

# Install Archon CLI (quick install)
curl -fsSL https://archon.diy/install | bash

# OR clone and run from source
git clone https://github.com/coleam00/Archon && cd Archon
bun install
bun run dev          # starts server (port 3090) + web UI (port 5173)
```

### Authentication Configuration

Claude (recommended — uses existing `claude /login` session):

```bash
# .env
CLAUDE_USE_GLOBAL_AUTH=true
```

Claude with API key:

```bash
# .env
CLAUDE_API_KEY=sk-ant-...
```

GitHub webhooks:

```bash
# .env
GITHUB_TOKEN=ghp_...
WEBHOOK_SECRET=your-random-secret
# Configure webhook in GitHub repo → Settings → Webhooks
# URL: https://your-server/webhooks/github
# Content-Type: application/json
# Events: Issue comments
```

### Integration Pattern 1: Trigger a workflow from the REST API

Register a project, then POST a message to run a workflow:

```bash
# Register project
curl -X POST http://localhost:3090/api/codebases \
  -H 'Content-Type: application/json' \
  -d '{"path": "/path/to/my-repo"}'

# Create conversation
CONV=$(curl -s -X POST http://localhost:3090/api/conversations \
  -H 'Content-Type: application/json' -d '{}' | jq -r .id)

# Trigger workflow via natural language (router AI selects the workflow)
curl -X POST http://localhost:3090/api/conversations/$CONV/message \
  -H 'Content-Type: application/json' \
  -d '{"message": "Use archon to fix issue #42"}'

# Stream progress (SSE)
curl -N http://localhost:3090/api/stream/$CONV
```

### Integration Pattern 2: CLI in CI/CD pipelines

```bash
cd /path/to/repo

# Run a code review workflow on the current branch, no worktree isolation
archon workflow run archon-smart-pr-review --no-worktree "Review my changes"

# Run with explicit branch (safe for parallel CI jobs)
archon workflow run archon-fix-github-issue --branch ci-fix-$ISSUE_NUM \
  "Fix issue #$ISSUE_NUM"

# Check workflow status (machine-readable)
archon workflow status --json | jq '.runs[] | select(.status == "running")'
```

### Integration Pattern 3: Embed the workflow engine in a custom server

`@archon/workflows` has zero `@archon/core` dependency. Inject deps via the `WorkflowDeps` interface:

```typescript
import { executeWorkflow } from '@archon/workflows/executor';
import { discoverWorkflowsWithConfig } from '@archon/workflows/workflow-discovery';
import type { WorkflowDeps, IWorkflowPlatform } from '@archon/workflows/deps';

// Your platform adapter
const platform: IWorkflowPlatform = {
  sendMessage: async (convId, msg) => {
    /* your delivery */
  },
  getPlatformType: () => 'custom',
};

// Your IWorkflowStore implementation (or use createWorkflowStore() from @archon/core)
const deps: WorkflowDeps = {
  store: myWorkflowStore,
  getAssistantClient: provider => new ClaudeClient(),
  loadConfig: async cwd => myConfigLoader(cwd),
};

const { workflows } = await discoverWorkflowsWithConfig(cwd);
const workflow = workflows.find(w => w.name === 'archon-fix-github-issue')!;

await executeWorkflow(deps, platform, conversationId, cwd, workflow, userMessage, dbConvId);
```

### Integration Pattern 4: Subscribe to workflow events (SSE bridge)

Use the event emitter for in-process observability without polling:

```typescript
import { getWorkflowEventEmitter } from '@archon/workflows/event-emitter';

const emitter = getWorkflowEventEmitter();
const unsubscribe = emitter.subscribeForConversation(conversationId, event => {
  // event.type: 'workflow_started' | 'node_completed' | 'tool_called' | ...
  console.log(event.type, event.runId);
});

// Cleanup when done
unsubscribe();
```

### Integration Pattern 5: Add a custom platform adapter

Implement `IPlatformAdapter` (from `@archon/core`) to add a new platform:

```typescript
import type { IPlatformAdapter, Conversation, MessageChunk } from '@archon/core';

export class MyAdapter implements IPlatformAdapter {
  getConversationId(event: MyEvent): string {
    return event.threadId;
  }
  async sendMessage(conversationId: string, message: string): Promise<void> {
    await myApi.send(conversationId, message);
  }
  async sendMessageChunk(conversationId: string, chunk: MessageChunk): Promise<void> {
    /* streaming */
  }
  getStreamingMode(): 'stream' | 'batch' {
    return 'batch';
  }
  getPlatformType(): string {
    return 'my-platform';
  }
  onMessage(handler: (msg: { conversationId: string; message: string }) => Promise<void>): void {
    this.handler = handler;
  }
  async start(): Promise<void> {
    /* begin polling/websocket */
  }
  stop(): void {
    /* cleanup */
  }
}
```

### Extension Points

- **Platform adapters** — implement `IPlatformAdapter` in `packages/adapters/src/`
- **AI clients** — implement `IAssistantClient` in `packages/core/src/clients/`
- **Database backends** — implement `IDatabase` interface; current backends: SQLite (`packages/core/src/db/adapters/sqlite.ts`) and PostgreSQL (`packages/core/src/db/adapters/postgres.ts`)
- **Isolation providers** — implement `IIsolationProvider` in `packages/isolation/src/providers/`; currently only `WorktreeProvider`
- **Workflow store** — implement `IWorkflowStore` from `@archon/workflows/store` for a custom DB backend
- **Custom workflows** — place `.yaml` files in `.archon/workflows/`; same-named files override bundled defaults
- **Custom commands** — place markdown files in `.archon/commands/`; invoked by name from workflow nodes
- **Global workflows** — place workflows in `~/.archon/.archon/workflows/` to apply to all projects

### Known Limitations and Constraints

- `@archon/workflows` must not import `@archon/core` (circular dependency); all core deps are injected via `WorkflowDeps`.
- `@archon/web` must not import from `@archon/workflows`; use types from the generated `src/lib/api.generated.d.ts`.
- Bun's `mock.module()` is process-global and irreversible — never run `bun test` from the repo root (see CLAUDE.md testing section).
- `output_format` (structured JSON output) is supported only on Claude nodes; Codex nodes log a warning and ignore it.
- `allowed_tools` / `denied_tools` / `hooks` / `mcp` / `skills` are Claude SDK features only; Codex nodes ignore them.
- The env-leak gate blocks workflow execution if the target repo's `.env` contains sensitive keys (e.g., `ANTHROPIC_API_KEY`) until the user explicitly grants `allowEnvKeys` consent per codebase.
- Interactive loop gates (approval workflows) require the workflow to be started with `interactive: true` to work correctly in the Web UI; CLI always supports them.
- GitHub webhook processing triggers only on `issue_comment` events mentioning the bot (not issue/PR descriptions, per design — see anthropics/claude-code#96).

---

## Operational Notes

### Deployment Models

**Single binary (recommended for individuals):**

```bash
curl -fsSL https://archon.diy/install | bash
archon setup
```

Binary is self-contained — bundles default workflows and commands at compile time (`packages/workflows/src/defaults/bundled-defaults.ts`).

**Docker Compose:**

```bash
# SQLite (default)
docker compose up -d

# With PostgreSQL
docker compose --profile with-db up -d postgres
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/remote_coding_agent

# Cloud profile (with Caddy TLS)
docker compose --profile cloud up -d
# Requires: DOMAIN=archon.example.com in .env
```

**Source:**

```bash
bun install && bun run dev    # port 3090 (server) + 5173 (vite dev)
bun run build && bun run start  # production build
```

### Port Allocation

- Main repo: `PORT` env var or `3090`
- Worktrees: Hash of worktree path → deterministic port in range `3190–4089`
- Same worktree always gets the same port; no port conflicts between parallel runs
- Override: `PORT=4000 bun dev`

### Observability

**Structured JSON logging via Pino** (`@archon/paths`). Event naming convention: `{domain}.{action}_{state}`, e.g., `workflow.step_started`, `isolation.create_failed`.

Log destinations:

- **Server stdout** — Pino JSON; pretty-print with `| pino-pretty`
- **Workflow JSONL logs** — `~/.archon/workspaces/{owner}/{repo}/logs/{runId}.jsonl`

Log levels: `fatal > error > warn > info (default) > debug > trace`

- CLI: `--quiet` → `warn`, `--verbose` → `debug`
- Server: `LOG_LEVEL=debug bun run start`

**No built-in metrics endpoint.** Workflow run status is queryable via `GET /api/workflows/runs`.

**Health endpoints:**

- `GET /health` — basic liveness
- `GET /health/db` — database connectivity
- `GET /health/concurrency` — lock manager saturation

### Resource Requirements

- **Runtime:** Bun ≥ 1.3.0
- **Memory:** Low base footprint; spikes per concurrent AI session (Claude SDK streams subprocess output)
- **Disk:** `~/.archon/` grows with worktrees, artifacts, logs; use `archon isolation cleanup` and `archon workflow cleanup` to reclaim space
- **Network:** Outbound HTTPS to Anthropic/OpenAI APIs and to GitHub/GitLab/Gitea/Telegram/Slack/Discord APIs as configured

### Data Persistence

| Data          | Location                                                       | Notes                                         |
| ------------- | -------------------------------------------------------------- | --------------------------------------------- |
| Database      | `~/.archon/archon.db` (SQLite) or `$DATABASE_URL` (PostgreSQL) | All conversations, runs, events, messages     |
| Cloned repos  | `~/.archon/workspaces/{owner}/{repo}/source/`                  | Full git clone                                |
| Worktrees     | `~/.archon/workspaces/{owner}/{repo}/worktrees/{branch}/`      | Isolated per run                              |
| Artifacts     | `~/.archon/workspaces/{owner}/{repo}/artifacts/runs/{runId}/`  | Workflow outputs; `$ARTIFACTS_DIR` in prompts |
| Workflow logs | `~/.archon/workspaces/{owner}/{repo}/logs/`                    | JSONL per run                                 |
| Config        | `~/.archon/config.yaml`                                        | Global user config                            |
| Global env    | `~/.archon/.env`                                               | `DATABASE_URL` and other infra vars           |

Override all base paths with `ARCHON_HOME=<path>` (default `~/.archon`). Docker deployments default to an `archon_data` named volume; override with `ARCHON_DATA=/host/path`.

**Cleanup commands:**

```bash
archon workflow cleanup 30         # delete terminal run records older than 30 days
archon isolation cleanup 14        # remove worktrees idle for 14+ days
archon isolation cleanup --merged  # remove worktrees whose branches are merged
```
