---
generated_by: repo-deep-dive workflow
date: 2026-04-09
depth: deep
---

# Archon API Reference

Generated from source: `packages/server/src/routes/api.ts`, `packages/server/src/index.ts`, `packages/cli/src/cli.ts`, `packages/workflows/src/store.ts`, `packages/workflows/src/event-emitter.ts`, and related schema files.

---

## Table of Contents

1. [REST / HTTP API](#rest--http-api)
   - [Conversations](#conversations)
   - [Messages](#messages)
   - [Workflows](#workflows)
   - [Workflow Runs](#workflow-runs)
   - [Dashboard](#dashboard)
   - [Codebases](#codebases)
   - [Codebase Env Vars](#codebase-env-vars)
   - [Configuration](#configuration)
   - [Artifacts](#artifacts)
   - [Streaming (SSE)](#streaming-sse)
   - [Health Checks](#health-checks)
   - [Webhooks](#webhooks)
   - [OpenAPI Spec](#openapi-spec)
2. [CLI Interface](#cli-interface)
3. [Events / Streaming](#events--streaming)
4. [SDK / Library API](#sdk--library-api)
5. [Authentication & Authorization](#authentication--authorization)
6. [Error Reference](#error-reference)

---

## REST / HTTP API

Base URL: `http://localhost:3090` (default; worktrees use hash-allocated ports in range 3190–4089).

All JSON endpoints return `Content-Type: application/json`. Error responses use the common error schema `{ "error": string }`. CORS is `*` by default; override with `WEB_UI_ORIGIN` env var.

The full OpenAPI 3.0 spec is available at `GET /api/openapi.json`.

---

### Conversations

#### `GET /api/conversations`

List conversations, newest first. Returns up to 50 records.

**Source:** `packages/server/src/routes/api.ts:1022`

| Query param  | Type                | Description                                                          |
| ------------ | ------------------- | -------------------------------------------------------------------- |
| `platform`   | `string` (optional) | Filter by platform type (`web`, `slack`, `telegram`, `github`, etc.) |
| `codebaseId` | `string` (optional) | Filter by codebase UUID                                              |

**Response 200** — `ConversationListResponse` (array):

```json
[
  {
    "id": "uuid",
    "platform_type": "web",
    "platform_conversation_id": "web-1703123456789-a7f3bc",
    "codebase_id": "uuid | null",
    "cwd": "/path/to/repo | null",
    "isolation_env_id": "uuid | null",
    "ai_assistant_type": "claude",
    "title": "Fix login bug | null",
    "hidden": false,
    "deleted_at": "ISO8601 | null",
    "last_activity_at": "ISO8601 | null",
    "created_at": "ISO8601",
    "updated_at": "ISO8601"
  }
]
```

**Schema:** `packages/server/src/routes/schemas/conversation.schemas.ts:7`

---

#### `GET /api/conversations/{id}`

Fetch a single conversation by its platform conversation ID.

**Source:** `packages/server/src/routes/api.ts:1040`

| Path param | Type     | Description                                                |
| ---------- | -------- | ---------------------------------------------------------- |
| `id`       | `string` | Platform conversation ID (e.g. `web-1703123456789-a7f3bc`) |

**Response 200** — `Conversation` object (same schema as list entry)

**Response 404** — `{ "error": "Conversation not found" }`

---

#### `POST /api/conversations`

Create a new conversation. Optionally provide a `message` to create-and-send atomically.

**Source:** `packages/server/src/routes/api.ts:1056`

**Request body** (`application/json`, optional):

```json
{
  "codebaseId": "uuid (optional)",
  "message": "string (optional)"
}
```

Body uses `.strict()` — unknown fields are rejected with 400. `codebaseId` must reference an existing codebase.

**Response 200** — `CreateConversationResponse`:

```json
{
  "conversationId": "web-1703123456789-a7f3bc",
  "id": "uuid",
  "dispatched": true
}
```

`dispatched: true` is present only when `message` was provided and was dispatched to the orchestrator.

**Schema:** `packages/server/src/routes/schemas/conversation.schemas.ts:40`

---

#### `PATCH /api/conversations/{id}`

Update a conversation's title.

**Source:** `packages/server/src/routes/api.ts:1119`

**Request body:**

```json
{ "title": "string (min 1, max 255 after truncation)" }
```

**Response 200** — `{ "success": true }`

**Response 404** — Conversation not found

---

#### `DELETE /api/conversations/{id}`

Soft-delete a conversation (sets `deleted_at`).

**Source:** `packages/server/src/routes/api.ts:1141`

**Response 200** — `{ "success": true }`

**Response 404** — Conversation not found

---

### Messages

#### `GET /api/conversations/{id}/messages`

List message history for a conversation.

**Source:** `packages/server/src/routes/api.ts:1160`

| Query param | Type                | Description                               |
| ----------- | ------------------- | ----------------------------------------- |
| `limit`     | `string` (optional) | Max records, clamped to 500. Default: 200 |

**Response 200** — `MessageListResponse` (array):

```json
[
  {
    "id": "uuid",
    "conversation_id": "uuid",
    "role": "user | assistant",
    "content": "string",
    "metadata": "{}",
    "created_at": "ISO8601"
  }
]
```

`metadata` is always a JSON string (normalized from JSONB for PostgreSQL).

**Schema:** `packages/server/src/routes/schemas/conversation.schemas.ts:66`

---

#### `POST /api/conversations/{id}/message`

Send a message to a conversation. Supports both JSON and multipart form-data (file uploads).

**Source:** `packages/server/src/routes/api.ts:1185`

**JSON variant** (`application/json`):

```json
{ "message": "string (min 1)" }
```

**Multipart variant** (`multipart/form-data`):

| Field     | Type     | Constraints                       |
| --------- | -------- | --------------------------------- |
| `message` | `string` | Required, non-empty               |
| `files`   | `File[]` | Optional, max 5 files, 10 MB each |

Allowed file types: all `text/*` MIME types, `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `application/pdf`, `application/json`, and common code/config extensions (`.md`, `.ts`, `.py`, `.go`, `.rs`, `.sql`, etc.). Uploaded files are saved to `~/.archon/artifacts/uploads/{conversationId}/` and cleaned up after the AI subprocess reads them.

Conversation IDs must match `[\w-]+` (alphanumeric + hyphen). Path traversal is rejected.

**Response 200** — `DispatchResponse`:

```json
{ "accepted": true, "status": "queued-conversation | queued-capacity | dispatched" }
```

**Response 400** — Message empty, too many files, unsupported file type, file too large, invalid JSON

**Schema:** `packages/server/src/routes/schemas/conversation.schemas.ts:86–108`

---

### Workflows

#### `GET /api/workflows`

Discover available workflows for the given working directory.

**Source:** `packages/server/src/routes/api.ts:1724`

| Query param | Type                | Description                                                                            |
| ----------- | ------------------- | -------------------------------------------------------------------------------------- |
| `cwd`       | `string` (optional) | Must be rooted at a registered codebase path. Falls back to first codebase if omitted. |

The `cwd` parameter is validated against registered codebase paths to prevent path traversal.

**Response 200** — `WorkflowListResponse`:

```json
{
  "workflows": [
    {
      "workflow": {
        /* WorkflowDefinition */
      },
      "source": "project | bundled"
    }
  ],
  "errors": [
    {
      "filename": "broken.yaml",
      "error": "reason",
      "errorType": "read_error | parse_error | validation_error"
    }
  ]
}
```

`errors` is omitted when there are none.

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:32`

---

#### `GET /api/workflows/{name}`

Fetch a single workflow definition by name.

**Source:** `packages/server/src/routes/api.ts` (via `getWorkflowRoute`, line 189)

| Path param | Type     | Description                                                 |
| ---------- | -------- | ----------------------------------------------------------- |
| `name`     | `string` | Workflow name (case-insensitive lookup via 4-tier fallback) |

| Query param | Type                |
| ----------- | ------------------- |
| `cwd`       | `string` (optional) |

**Response 200** — `GetWorkflowResponse`:

```json
{
  "workflow": {
    /* WorkflowDefinition */
  },
  "filename": "my-workflow.yaml",
  "source": "project | bundled"
}
```

**Response 404** — Workflow not found

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:40`

---

#### `POST /api/workflows/validate`

Validate a workflow definition in-memory without saving it.

**Source:** `packages/server/src/routes/api.ts` (via `validateWorkflowRoute`, line 168)

**Request body:**

```json
{
  "definition": {
    /* raw workflow object */
  }
}
```

**Response 200** — `ValidateWorkflowResponse`:

```json
{
  "valid": true,
  "errors": ["error message 1", "error message 2"]
}
```

`errors` is omitted when `valid: true`.

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:57`

---

#### `PUT /api/workflows/{name}`

Save (create or update) a workflow YAML file. Validates the definition before writing.

**Source:** `packages/server/src/routes/api.ts` (via `saveWorkflowRoute`, line 209)

| Query param | Type                | Description                                        |
| ----------- | ------------------- | -------------------------------------------------- |
| `cwd`       | `string` (optional) | Target directory; must match a registered codebase |

**Request body:**

```json
{
  "definition": {
    /* raw workflow object */
  }
}
```

**Response 200** — `GetWorkflowResponse` (the saved workflow)

**Response 400** — Invalid definition or invalid `cwd`

---

#### `DELETE /api/workflows/{name}`

Delete a user-defined workflow. Bundled default workflows cannot be deleted.

**Source:** `packages/server/src/routes/api.ts` (via `deleteWorkflowRoute`, line 229)

| Query param | Type                |
| ----------- | ------------------- |
| `cwd`       | `string` (optional) |

**Response 200** — `DeleteWorkflowResponse`:

```json
{ "deleted": true, "name": "workflow-name" }
```

**Response 404** — Workflow not found

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:66`

---

#### `POST /api/workflows/{name}/run`

Run a workflow via the orchestrator. Translates to an internal `/workflow run <name> <message>` dispatch.

**Source:** `packages/server/src/routes/api.ts:1759`

| Path param | Type     | Description                                         |
| ---------- | -------- | --------------------------------------------------- |
| `name`     | `string` | Workflow name (validated with `isValidCommandName`) |

**Request body:**

```json
{
  "conversationId": "string",
  "message": "string"
}
```

**Response 200** — `DispatchResponse`:

```json
{ "accepted": true, "status": "queued-conversation | queued-capacity | dispatched" }
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:198`

---

#### `GET /api/commands`

List available command names (both bundled defaults and project-defined).

**Source:** `packages/server/src/routes/api.ts` (via `getCommandsRoute`, line 249)

| Query param | Type                |
| ----------- | ------------------- |
| `cwd`       | `string` (optional) |

**Response 200** — `CommandListResponse`:

```json
{
  "commands": [
    { "name": "execute", "source": "bundled" },
    { "name": "my-command", "source": "project" }
  ]
}
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:78`

---

### Workflow Runs

#### `GET /api/workflows/runs`

List workflow runs with optional filtering.

**Source:** `packages/server/src/routes/api.ts` (via `listWorkflowRunsRoute`, line 610)

| Query param      | Type                | Description                                                        |
| ---------------- | ------------------- | ------------------------------------------------------------------ |
| `conversationId` | `string` (optional) | Filter by conversation UUID                                        |
| `status`         | `string` (optional) | `pending \| running \| completed \| failed \| cancelled \| paused` |
| `codebaseId`     | `string` (optional) | Filter by codebase UUID                                            |
| `limit`          | `string` (optional) | Max results                                                        |

**Response 200** — `WorkflowRunListResponse`:

```json
{
  "runs": [
    {
      "id": "uuid",
      "workflow_name": "archon-fix-github-issue",
      "conversation_id": "uuid",
      "parent_conversation_id": "uuid | null",
      "codebase_id": "uuid | null",
      "status": "running",
      "user_message": "Fix the login bug",
      "metadata": {},
      "started_at": "ISO8601",
      "completed_at": "ISO8601 | null",
      "last_activity_at": "ISO8601 | null",
      "working_path": "/path/to/worktree | null"
    }
  ]
}
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:110`

---

#### `GET /api/workflows/runs/{runId}`

Get workflow run details including all events.

**Source:** `packages/server/src/routes/api.ts` (via `getWorkflowRunRoute`, line 733)

**Response 200** — `WorkflowRunDetail`:

```json
{
  "run": {
    /* WorkflowRun fields */
    "worker_platform_id": "string (optional)",
    "parent_platform_id": "string (optional)",
    "conversation_platform_id": "string | null"
  },
  "events": [
    {
      "id": "uuid",
      "workflow_run_id": "uuid",
      "event_type": "node_started",
      "step_index": 0,
      "step_name": "plan",
      "data": {},
      "created_at": "ISO8601"
    }
  ]
}
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:128`

---

#### `GET /api/workflows/runs/by-worker/{platformId}`

Look up a workflow run by its worker conversation platform ID. Used to resolve child conversation IDs back to their parent run.

**Source:** `packages/server/src/routes/api.ts` (via `getWorkflowRunByWorkerRoute`, line 594)

**Response 200** — `WorkflowRunByWorkerResponse`:

```json
{
  "run": {
    /* WorkflowRun */
  }
}
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:140`

---

#### `POST /api/workflows/runs/{runId}/cancel`

Cancel a workflow run. Only valid for runs in `running`, `pending`, or `paused` status.

**Source:** `packages/server/src/routes/api.ts:1845`

**Response 200** — `CancelWorkflowRunResponse`:

```json
{ "success": true, "message": "Cancelled workflow: workflow-name" }
```

**Response 400** — Cannot cancel in current status

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:145`

---

#### `POST /api/workflows/runs/{runId}/resume`

Mark a failed/paused run as ready for auto-resume. The next `/workflow run` invocation on the same path automatically resumes from completed nodes.

**Source:** `packages/server/src/routes/api.ts:1864`

Only valid for runs in `RESUMABLE_WORKFLOW_STATUSES` (`failed`, `paused`).

**Response 200** — `WorkflowRunActionResponse`:

```json
{
  "success": true,
  "message": "Workflow run ready to resume: workflow-name at /path/to/worktree. Re-run the workflow to auto-resume from completed nodes."
}
```

---

#### `POST /api/workflows/runs/{runId}/abandon`

Abandon a non-terminal run (marks as `cancelled`). Not applicable to already-terminal runs.

**Source:** `packages/server/src/routes/api.ts:1887`

**Response 200** — `WorkflowRunActionResponse`

**Response 400** — Run is already in a terminal status

---

#### `POST /api/workflows/runs/{runId}/approve`

Approve a paused workflow run (at an approval gate or interactive loop checkpoint).

**Source:** `packages/server/src/routes/api.ts:1906`

Only valid for runs in `paused` status.

**Request body** (optional):

```json
{ "comment": "Looks good, proceed" }
```

Defaults to `"Approved"` if comment is absent.

**Approval gate behavior:** Writes `node_completed` event (unless this is an interactive loop pause); resets rejection state; marks run as `failed` so resume picks it up.

**Interactive loop behavior:** Stores `loop_user_input` in metadata; does NOT write `node_completed`.

**Response 200** — `WorkflowRunActionResponse`

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:155`

---

#### `POST /api/workflows/runs/{runId}/reject`

Reject a paused workflow run. Triggers `on_reject` prompt if configured; cancels if max attempts reached.

**Source:** `packages/server/src/routes/api.ts:1961`

Only valid for runs in `paused` status.

**Request body** (optional):

```json
{ "reason": "The implementation is missing error handling" }
```

Defaults to `"Rejected"` if reason is absent.

**Behavior:**

- If `on_reject` prompt is configured: increments `rejection_count`; marks as `failed` so on-reject runs on next resume
- If `rejection_count + 1 >= maxAttempts` (default 3): cancels the run
- If no `on_reject` configured: cancels immediately

**Response 200** — `WorkflowRunActionResponse`

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:161`

---

#### `DELETE /api/workflows/runs/{runId}`

Delete a terminal workflow run and all its events.

**Source:** `packages/server/src/routes/api.ts` (via `deleteWorkflowRunRoute`, line 716)

Only terminal runs can be deleted (`completed`, `failed`, `cancelled`).

**Response 200** — `WorkflowRunActionResponse`

**Response 400** — Run is not in a terminal status

---

### Dashboard

#### `GET /api/dashboard/runs`

Enriched workflow run listing for the Command Center dashboard. Supports server-side filtering, full-text search, date range, and offset pagination.

**Source:** `packages/server/src/routes/api.ts:1802`

| Query param  | Type                | Description                                                        |
| ------------ | ------------------- | ------------------------------------------------------------------ |
| `status`     | `string` (optional) | `pending \| running \| completed \| failed \| cancelled \| paused` |
| `codebaseId` | `string` (optional) | Filter by codebase UUID                                            |
| `search`     | `string` (optional) | Full-text search across workflow name and user message             |
| `after`      | `string` (optional) | ISO8601 date — only runs started after this date                   |
| `before`     | `string` (optional) | ISO8601 date — only runs started before this date                  |
| `limit`      | `string` (optional) | Max 200, default 50                                                |
| `offset`     | `string` (optional) | Pagination offset, default 0                                       |

**Response 200** — `DashboardRunsResponse`:

```json
{
  "runs": [
    {
      /* All WorkflowRun fields, plus: */
      "codebase_name": "my-repo | null",
      "platform_type": "web | null",
      "worker_platform_id": "string | null",
      "parent_platform_id": "string | null",
      "current_step_name": "implement | null",
      "total_steps": 5,
      "current_step_status": "running | completed | failed | null",
      "agents_completed": 3,
      "agents_failed": 0,
      "agents_total": 5
    }
  ],
  "total": 42,
  "counts": {
    "all": 42,
    "running": 3,
    "completed": 30,
    "failed": 5,
    "cancelled": 2,
    "pending": 1,
    "paused": 1
  }
}
```

**Schema:** `packages/server/src/routes/schemas/workflow.schemas.ts:165`

---

### Codebases

#### `GET /api/codebases`

List all registered codebases. Deduplicates by `repository_url` (keeps most recently updated), sorted alphabetically by name.

**Source:** `packages/server/src/routes/api.ts:1437`

**Response 200** — `CodebaseListResponse` (array of `Codebase`):

```json
[
  {
    "id": "uuid",
    "name": "my-repo",
    "repository_url": "https://github.com/owner/repo | null",
    "default_cwd": "/home/user/.archon/workspaces/owner/repo/source",
    "ai_assistant_type": "claude",
    "allow_env_keys": false,
    "commands": {
      "execute": { "path": "/path/to/command.md", "description": "Run tests" }
    },
    "created_at": "ISO8601",
    "updated_at": "ISO8601"
  }
]
```

**Schema:** `packages/server/src/routes/schemas/codebase.schemas.ts:12`

---

#### `GET /api/codebases/{id}`

Fetch a single codebase by UUID.

**Source:** `packages/server/src/routes/api.ts:1481`

**Response 200** — `Codebase` object

**Response 404** — Codebase not found

---

#### `POST /api/codebases`

Register a codebase by cloning from a GitHub URL or registering a local path. Exactly one of `url` or `path` must be provided.

**Source:** `packages/server/src/routes/api.ts:1503`

**Request body:**

```json
{
  "url": "https://github.com/owner/repo",
  "path": "/local/path/to/repo",
  "allowEnvKeys": false
}
```

Providing both `url` and `path`, or neither, is rejected with 400.

`allowEnvKeys`: grants consent for the env-leak gate — allows the AI subprocess to see `.env` keys from the target repo. Audit-logged at `warn` level on every grant.

**Response 200** — `Codebase` (already existed)

**Response 201** — `Codebase` (newly created)

**Response 422** — `EnvLeakError`: target repo's `.env` contains sensitive keys; set `allowEnvKeys: true` to proceed

**Schema:** `packages/server/src/routes/schemas/codebase.schemas.ts:32`

---

#### `PATCH /api/codebases/{id}`

Update codebase consent flags (currently: `allowEnvKeys`).

**Source:** `packages/server/src/routes/api.ts:1536`

**Request body:**

```json
{ "allowEnvKeys": true }
```

Audit-logged at `warn` level as `env_leak_consent_granted` or `env_leak_consent_revoked` with `codebaseId`, `path`, `files`, `keys`, `scanStatus`, and `actor`.

**Response 200** — Updated `Codebase` object

**Response 404** — Codebase not found

---

#### `DELETE /api/codebases/{id}`

Delete a codebase and clean up associated resources (worktrees, workspace directory for Archon-managed repos).

**Source:** `packages/server/src/routes/api.ts:1601`

**Response 200** — `{ "success": true }`

**Response 404** — Codebase not found

---

#### `GET /api/codebases/{id}/environments`

List isolation environments (worktrees) for a codebase.

**Source:** `packages/server/src/routes/api.ts` (via `getCodebaseEnvironmentsRoute`, line 792)

**Response 200** — `CodebaseEnvironmentsResponse`:

```json
{
  "environments": [
    {
      "id": "uuid",
      "codebase_id": "uuid",
      "branch_name": "archon/task-implement-1703123456789",
      "working_path": "/home/user/.archon/workspaces/owner/repo/worktrees/branch",
      "status": "active",
      "created_at": "ISO8601",
      "updated_at": "ISO8601",
      "days_since_activity": 2
    }
  ]
}
```

**Schema:** `packages/server/src/routes/schemas/config.schemas.ts:64`

---

### Codebase Env Vars

#### `GET /api/codebases/{id}/env`

List env var **keys** for a codebase. Values are never returned.

**Source:** `packages/server/src/routes/api.ts:1651`

**Response 200** — `CodebaseEnvVarsResponse`:

```json
{ "keys": ["ANTHROPIC_API_KEY", "DATABASE_URL"] }
```

**Schema:** `packages/server/src/routes/schemas/codebase.schemas.ts:57`

---

#### `PUT /api/codebases/{id}/env`

Upsert an env var key-value pair for a codebase.

**Source:** `packages/server/src/routes/api.ts:1665`

**Request body:**

```json
{ "key": "MY_SECRET", "value": "secret-value" }
```

`key`: 1–255 characters. `value`: any string.

**Response 200** — `{ "success": true }`

**Schema:** `packages/server/src/routes/schemas/codebase.schemas.ts:64`

---

#### `DELETE /api/codebases/{id}/env/{key}`

Delete an env var from a codebase.

**Source:** `packages/server/src/routes/api.ts:1680`

**Response 200** — `{ "success": true }`

---

### Configuration

#### `GET /api/config`

Get the read-only safe configuration subset (no filesystem paths or secrets).

**Source:** `packages/server/src/routes/api.ts:2506`

**Response 200** — `ConfigResponse`:

```json
{
  "config": {
    "botName": "Archon",
    "assistant": "claude",
    "assistants": {
      "claude": { "model": "sonnet" },
      "codex": {
        "model": "gpt-5.3-codex",
        "modelReasoningEffort": "medium",
        "webSearchMode": "live"
      }
    },
    "streaming": {
      "telegram": "stream",
      "discord": "batch",
      "slack": "batch"
    },
    "concurrency": { "maxConversations": 10 },
    "defaults": {
      "copyDefaults": true,
      "loadDefaultCommands": true,
      "loadDefaultWorkflows": true
    }
  },
  "database": "sqlite"
}
```

**Schema:** `packages/server/src/routes/schemas/config.schemas.ts:53`

---

#### `PATCH /api/config/assistants`

Update assistant configuration. All fields are optional (partial update).

**Source:** `packages/server/src/routes/api.ts` (via `patchAssistantConfigRoute`, line 771)

**Request body:**

```json
{
  "assistant": "claude | codex (optional)",
  "claude": { "model": "sonnet | opus | haiku | claude-* (optional)" },
  "codex": {
    "model": "string (optional)",
    "modelReasoningEffort": "minimal | low | medium | high | xhigh (optional)",
    "webSearchMode": "disabled | cached | live (optional)"
  }
}
```

**Response 200** — `ConfigResponse` (same as `GET /api/config`)

**Schema:** `packages/server/src/routes/schemas/config.schemas.ts:35`

---

### Artifacts

#### `GET /api/artifacts/{runId}/*`

Serve a workflow artifact file by run ID and relative path.

**Source:** `packages/server/src/routes/api.ts:2416`

| Path segment | Description                                                                             |
| ------------ | --------------------------------------------------------------------------------------- |
| `runId`      | Workflow run UUID                                                                       |
| `*`          | Relative path within the run's artifacts directory (e.g. `plan.md`, `subdir/report.md`) |

Path traversal is blocked: any `..` segment returns 400. The `runId` must resolve to a run with a `working_path` inside `~/.archon/workspaces/`.

**Response 200** — Raw file content

- `Content-Type: text/markdown; charset=utf-8` for `.md` files
- `Content-Type: text/plain; charset=utf-8` for all others

**Response 400** — Invalid filename or path traversal attempt

**Response 404** — Run not found, `working_path` missing, or file not found

---

### Streaming (SSE)

#### `GET /api/stream/{conversationId}`

Server-Sent Events stream for a conversation. Opens an SSE connection that receives all events for the given conversation (AI responses, tool calls, workflow progress, lock state, errors).

**Source:** `packages/server/src/routes/api.ts:1395`

**Protocol:** HTTP `text/event-stream`. Each event is `data: {JSON}\n\n`.

**Initial event:** `{ "type": "heartbeat", "timestamp": 1703123456789 }` — sent immediately to flush HTTP headers and prevent EventSource from staying in `CONNECTING` state.

**Keepalive:** Heartbeat events every 30 seconds while stream is open.

**Buffering:** The `SSETransport` buffers up to 500 events with 60-second TTL. Clients reconnecting within 5 seconds receive buffered events.

**Cleanup:** `stream.onAbort()` triggers `webAdapter.removeStream()`. Uses `expectedStream` reference to prevent race conditions in React StrictMode double-mount.

**Event types forwarded to SSE streams:**

| Event `type`                     | Description                                                                 |
| -------------------------------- | --------------------------------------------------------------------------- |
| `heartbeat`                      | Keepalive ping, `{ timestamp }`                                             |
| `lock`                           | `{ locked: boolean }` — UI processing indicator                             |
| `error`                          | `{ message, classification, timestamp }`                                    |
| All `WorkflowEmitterEvent` types | Forwarded from workflow executor (see [Events section](#events--streaming)) |

---

#### `GET /api/stream/__dashboard__`

Multiplexed SSE stream receiving workflow events from **all** conversations. Used by the Command Center dashboard for real-time updates across all active runs.

**Source:** `packages/server/src/routes/api.ts:1360`

Must be registered before `/api/stream/:conversationId` to avoid param capture by the parameterized route.

Same heartbeat/keepalive behavior as the per-conversation stream.

---

### Health Checks

#### `GET /health`

Basic liveness check.

**Source:** `packages/server/src/index.ts:560`

**Response 200** — `{ "status": "ok" }`

---

#### `GET /health/db`

Database connectivity check.

**Source:** `packages/server/src/index.ts:564`

**Response 200** — `{ "status": "ok", "database": "connected" }`

**Response 500** — `{ "status": "error", "database": "disconnected" }`

---

#### `GET /health/concurrency`

Concurrency lock manager state.

**Source:** `packages/server/src/index.ts:574`

**Response 200** — `{ "status": "ok", "active": 2, "queuedTotal": 5, "maxConcurrent": 10 }`

---

#### `GET /api/health`

Detailed health status (registered via OpenAPI route).

**Source:** `packages/server/src/routes/api.ts` (via `getHealthRoute`, line 808)

**Response 200:**

```json
{
  "status": "ok",
  "adapter": "web",
  "concurrency": {},
  "runningWorkflows": 3,
  "version": "0.3.2",
  "is_docker": false
}
```

---

### Webhooks

All webhook endpoints return `OK` (200) immediately and process the payload asynchronously. Use `c.req.text()` for raw body to enable HMAC signature verification.

#### `POST /webhooks/github`

Receive GitHub webhook events (issue comments, PR events, etc.).

**Source:** `packages/server/src/index.ts:476`

Only registered when `GITHUB_TOKEN` / `GH_TOKEN` is configured.

| Header                | Required | Description                                        |
| --------------------- | -------- | -------------------------------------------------- |
| `x-github-event`      | Yes      | Event type (`issue_comment`, `pull_request`, etc.) |
| `x-github-delivery`   | No       | Delivery UUID for idempotency                      |
| `x-hub-signature-256` | Yes      | HMAC-SHA256 signature: `sha256=<hex>`              |

Signature is verified with `timingSafeEqual` using `WEBHOOK_SECRET`. Missing signature returns 400.

**Response 200** — `OK`

**Response 400** — Missing signature header

---

#### `POST /webhooks/gitea`

Receive Gitea webhook events.

**Source:** `packages/server/src/index.ts:507`

Only registered when `GITEA_TOKEN` is configured.

| Header              | Required |
| ------------------- | -------- |
| `x-gitea-event`     | No       |
| `x-gitea-signature` | Yes      |

**Response 200** — `OK`

**Response 400** — Missing signature header

---

#### `POST /webhooks/gitlab`

Receive GitLab webhook events.

**Source:** `packages/server/src/index.ts:535`

Only registered when `GITLAB_TOKEN` is configured.

| Header           | Required |
| ---------------- | -------- |
| `x-gitlab-event` | No       |
| `x-gitlab-token` | Yes      |

**Response 200** — `OK`

**Response 400** — Missing token header

---

### OpenAPI Spec

#### `GET /api/openapi.json`

Machine-generated OpenAPI 3.0 specification for all Zod-validated routes.

**Source:** `packages/server/src/routes/api.ts:1714`

```json
{
  "openapi": "3.0.0",
  "info": { "title": "Archon API", "version": "1.0.0" }
}
```

Frontend types are generated from this spec: `bun --filter @archon/web generate:types`.

---

## CLI Interface

All commands require being run from within a git repository (subdirectories work — CLI resolves to repo root via `git rev-parse --show-toplevel`).

**Source:** `packages/cli/src/cli.ts`

### Global Usage

```
archon <command> [subcommand] [options] [arguments]
```

### Global Options

| Option              | Alias           | Description                                                           |
| ------------------- | --------------- | --------------------------------------------------------------------- |
| `--cwd <path>`      |                 | Override working directory (default: current directory)               |
| `--branch <name>`   | `-b`            | Create worktree for branch (or reuse existing)                        |
| `--from <name>`     | `--from-branch` | Start point for new branch                                            |
| `--no-worktree`     |                 | Run directly in live checkout, no isolation                           |
| `--resume`          |                 | Resume most recent failed run (mutually exclusive with `--branch`)    |
| `--spawn`           |                 | Open setup wizard in a new terminal                                   |
| `--quiet`           | `-q`            | Warnings and errors only (suppresses Pino logs and workflow progress) |
| `--verbose`         | `-v`            | Debug-level output                                                    |
| `--json`            |                 | Machine-readable JSON (workflow list only)                            |
| `--workflow <name>` |                 | Workflow for `continue` command (default: `archon-assist`)            |
| `--no-context`      |                 | Skip context injection for `continue`                                 |
| `--allow-env-keys`  |                 | Grant env-key consent during auto-registration (audit-logged)         |

**Mutually exclusive pairs (enforced at pre-flight):**

- `--branch` + `--no-worktree`
- `--from` + `--no-worktree`
- `--resume` + `--branch`

---

### `archon chat <message>`

Send a single-shot message to the orchestrator agent. Streams response to stdout and exits.

**Source:** `packages/cli/src/commands/chat.ts:14`

```bash
archon chat "What does the orchestrator do?"
archon chat "Explain the DAG executor" --cwd /path/to/repo
```

---

### `archon workflow list`

List available workflows in the current directory.

**Source:** `packages/cli/src/commands/workflow.ts:145`

```bash
archon workflow list
archon workflow list --json
archon workflow list --cwd /path/to/repo
```

**JSON output fields per workflow:** `name`, `description`, `provider`, `model`, `modelReasoningEffort`, `webSearchMode`

---

### `archon workflow run <name> [message]`

Run a workflow. By default creates a worktree with an auto-generated branch name (`archon/task-{workflow}-{timestamp}`).

**Source:** `packages/cli/src/commands/workflow.ts`

```bash
# Default: isolated worktree, auto-generated branch
archon workflow run archon-fix-github-issue "Fix the login bug"

# Explicit branch name
archon workflow run implement --branch feature-auth "Implement OAuth"

# Override base branch for the worktree
archon workflow run implement --from dev "Add feature"

# No isolation — run in live checkout
archon workflow run quick-fix --no-worktree "Fix typo"

# Resume last failed run of this workflow
archon workflow run implement --resume

# Different cwd
archon workflow run assist --cwd /path/to/repo "What does this do?"
```

**Progress output** (to stderr): `node_started`, `node_completed`, `node_failed`, `approval_pending`, loop iteration events.

---

### `archon workflow status [runId]`

Show status of running workflows.

**Source:** `packages/cli/src/commands/workflow.ts`

```bash
archon workflow status
archon workflow status a1b2c3d4-...
```

---

### `archon workflow resume <runId>`

Resume a failed or paused workflow run.

```bash
archon workflow resume a1b2c3d4-...
```

---

### `archon workflow cancel <runId>`

Cancel a running workflow.

```bash
archon workflow cancel a1b2c3d4-...
```

---

### `archon workflow abandon <runId>`

Abandon (mark as failed) a non-terminal workflow run.

```bash
archon workflow abandon a1b2c3d4-...
```

---

### `archon workflow approve <runId>`

Approve a paused workflow run at an approval gate.

```bash
archon workflow approve a1b2c3d4-...
archon workflow approve a1b2c3d4-... --comment "Looks good, proceed"
```

---

### `archon workflow reject <runId>`

Reject a paused workflow run.

```bash
archon workflow reject a1b2c3d4-...
archon workflow reject a1b2c3d4-... --reason "Missing error handling"
```

---

### `archon workflow cleanup`

Remove orphaned workflow-related files.

```bash
archon workflow cleanup
```

---

### `archon workflow event emit`

Emit a workflow event manually (for testing).

**Source:** `packages/cli/src/commands/workflow.ts` (`workflowEventEmitCommand`)

```bash
archon workflow event emit --run-id <uuid> --type node_started --data '{"nodeId":"plan"}'
```

Valid `--type` values: all values in `WORKFLOW_EVENT_TYPES` (see [Events section](#events--streaming)).

---

### `archon isolation list`

List all active isolation environments (worktrees).

**Source:** `packages/cli/src/commands/isolation.ts:32`

```bash
archon isolation list
```

Output: grouped by codebase, shows branch, path, type, platform, last activity.

---

### `archon isolation cleanup [days]`

Remove stale isolation environments. Default: 7 days of inactivity.

**Source:** `packages/cli/src/commands/isolation.ts:77`

```bash
archon isolation cleanup
archon isolation cleanup 14
archon isolation cleanup --merged           # branches merged into main
archon isolation cleanup --merged --include-closed  # also includes closed PRs
```

`--merged`: deletes remote branches as well as local worktrees.

---

### `archon continue <branch> [message]`

Continue work on an existing worktree with prior conversation context injected.

**Source:** `packages/cli/src/commands/continue.ts`

```bash
archon continue fix/issue-42
archon continue fix/issue-42 --workflow archon-smart-pr-review "Review the changes"
archon continue fix/issue-42 --no-context "Start fresh"
```

Default workflow: `archon-assist`.

---

### `archon complete <branch>`

Complete branch lifecycle: removes the worktree, then deletes local and remote branches.

**Source:** `packages/cli/src/commands/isolation.ts` (`isolationCompleteCommand`)

```bash
archon complete fix/issue-42
archon complete fix/issue-42 --force   # skip uncommitted-changes check
```

Without `--force`, refuses if there are uncommitted changes in the worktree.

---

### `archon validate workflows [name]`

Validate all workflows or a specific workflow, including referenced commands, MCP configs, and skill directories.

**Source:** `packages/cli/src/commands/validate.ts:79`

```bash
archon validate workflows
archon validate workflows my-workflow
archon validate workflows my-workflow --json
archon validate workflows --cwd /path/to/repo
```

**Exit codes:** 0 = all valid, 1 = one or more errors

---

### `archon validate commands [name]`

Validate command files.

**Source:** `packages/cli/src/commands/validate.ts`

```bash
archon validate commands
archon validate commands my-command
archon validate commands --json
```

**Exit codes:** 0 = all valid, 1 = errors

---

### `archon setup`

Interactive setup wizard. Configures credentials, platform tokens, database choice, and copies the Archon skill to target repos.

**Source:** `packages/cli/src/commands/setup.ts`

```bash
archon setup
archon setup --spawn   # open in new terminal window
```

Writes to `~/.archon/.env` and optionally `<repo>/.env`.

---

### `archon version`

Show version, platform, build type, database type, and git commit.

**Source:** `packages/cli/src/commands/version.ts:75`

```bash
archon version
```

---

### `archon help`

Print usage information (same as running `archon` with no arguments).

---

## Events / Streaming

### Workflow Event Types

**Source:** `packages/workflows/src/store.ts:10`

The canonical list of event type strings stored in the `workflow_events` database table and emitted to SSE streams:

```typescript
export const WORKFLOW_EVENT_TYPES = [
  'workflow_started',
  'workflow_completed',
  'workflow_failed',
  'node_started',
  'node_completed',
  'node_failed',
  'node_skipped',
  'node_skipped_prior_success',
  'loop_iteration_started',
  'loop_iteration_completed',
  'loop_iteration_failed',
  'tool_called',
  'tool_completed',
  'ralph_story_started',
  'ralph_story_completed',
  'approval_requested',
  'approval_received',
  'workflow_cancelled',
] as const;

export type WorkflowEventType = (typeof WORKFLOW_EVENT_TYPES)[number];
```

### WorkflowEmitterEvent Payloads

**Source:** `packages/workflows/src/event-emitter.ts:27`

These typed event objects are emitted by the executor and forwarded to SSE streams.

```typescript
export type WorkflowEmitterEvent =
  | WorkflowStartedEvent
  | WorkflowCompletedEvent
  | WorkflowFailedEvent
  | LoopIterationStartedEvent
  | LoopIterationCompletedEvent
  | LoopIterationFailedEvent
  | NodeStartedEvent
  | NodeCompletedEvent
  | NodeFailedEvent
  | NodeSkippedEvent
  | WorkflowArtifactEvent
  | ToolStartedEvent
  | ToolCompletedEvent
  | ApprovalPendingEvent
  | WorkflowCancelledEvent;
```

**Individual payload shapes:**

| Event type                 | Fields                                                                                                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `workflow_started`         | `runId`, `workflowName`, `conversationId`                                                                                 |
| `workflow_completed`       | `runId`, `workflowName`, `duration` (ms)                                                                                  |
| `workflow_failed`          | `runId`, `workflowName`, `error`                                                                                          |
| `loop_iteration_started`   | `runId`, `nodeId?`, `iteration`, `maxIterations`                                                                          |
| `loop_iteration_completed` | `runId`, `nodeId?`, `iteration`, `duration`, `completionDetected`                                                         |
| `loop_iteration_failed`    | `runId`, `nodeId?`, `iteration`, `error`                                                                                  |
| `node_started`             | `runId`, `nodeId`, `nodeName`                                                                                             |
| `node_completed`           | `runId`, `nodeId`, `nodeName`, `duration`, `costUsd?`, `stopReason?`, `numTurns?`                                         |
| `node_failed`              | `runId`, `nodeId`, `nodeName`, `error`                                                                                    |
| `node_skipped`             | `runId`, `nodeId`, `nodeName`, `reason` (`when_condition \| when_condition_parse_error \| trigger_rule \| prior_success`) |
| `workflow_artifact`        | `runId`, `artifactType`, `label`, `url?`, `path?`                                                                         |
| `tool_started`             | `runId`, `toolName`, `stepName`                                                                                           |
| `tool_completed`           | `runId`, `toolName`, `stepName`, `durationMs`                                                                             |
| `approval_pending`         | `runId`, `nodeId`, `message`                                                                                              |
| `workflow_cancelled`       | `runId`, `nodeId`, `reason`                                                                                               |

### Subscription API

**Source:** `packages/workflows/src/event-emitter.ts:249`

```typescript
import { getWorkflowEventEmitter } from '@archon/workflows/event-emitter';

const emitter = getWorkflowEventEmitter();

// Register run-to-conversation mapping (called by executor)
emitter.registerRun(runId, conversationId);

// Subscribe to all events
const unsubscribe = emitter.subscribe((event: WorkflowEmitterEvent) => {
  console.log(event.type, event.runId);
});

// Subscribe to events for one conversation only
const unsubscribe2 = emitter.subscribeForConversation(conversationId, event => {
  // receives only events where emitter.conversationMap.get(event.runId) === conversationId
});

// Call returned function to remove listener
unsubscribe();

// Unregister run mapping at workflow end
emitter.unregisterRun(runId);
```

Design guarantees:

- Listener errors never propagate to the executor (fire-and-forget with internal catch)
- Singleton via `getWorkflowEventEmitter()`; `resetWorkflowEventEmitter()` resets for testing
- Max 50 listeners (set via `EventEmitter.setMaxListeners`)

### IWorkflowStore Interface

**Source:** `packages/workflows/src/store.ts:33`

The database abstraction interface implemented by `@archon/core` (`createWorkflowStore()`):

```typescript
export interface IWorkflowStore {
  // Run lifecycle
  createWorkflowRun(data: {
    workflow_name;
    conversation_id;
    codebase_id?;
    user_message;
    metadata?;
    working_path?;
    parent_conversation_id?;
  }): Promise<WorkflowRun>;
  getWorkflowRun(id: string): Promise<WorkflowRun | null>;
  getActiveWorkflowRunByPath(workingPath: string): Promise<WorkflowRun | null>;
  findResumableRun(workflowName: string, workingPath: string): Promise<WorkflowRun | null>;
  failOrphanedRuns(): Promise<{ count: number }>;
  resumeWorkflowRun(id: string): Promise<WorkflowRun>;
  updateWorkflowRun(id, updates: Partial<Pick<WorkflowRun, 'status' | 'metadata'>>): Promise<void>;
  updateWorkflowActivity(id: string): Promise<void>;
  getWorkflowRunStatus(id: string): Promise<WorkflowRunStatus | null>;
  completeWorkflowRun(id, metadata?): Promise<void>;
  failWorkflowRun(id, error: string): Promise<void>;
  pauseWorkflowRun(id, approvalContext: ApprovalContext): Promise<void>;
  cancelWorkflowRun(id: string): Promise<void>;

  // Events (MUST NOT throw — treat as observable-only)
  createWorkflowEvent(data: {
    workflow_run_id;
    event_type;
    step_index?;
    step_name?;
    data?;
  }): Promise<void>;

  // DAG resume
  getPriorNodeOutputs(runId: string): Promise<Map<string, NodeOutput>>;
}
```

---

## SDK / Library API

### `@archon/workflows`

**Source:** `packages/workflows/src/`

```typescript
// Workflow execution
import { executeWorkflow } from '@archon/workflows/executor';
// signature: (deps: WorkflowDeps, platform: IWorkflowPlatform, conversationId: string,
//             cwd: string, workflow: WorkflowDefinition, args: string[], options?) => Promise<void>

// Workflow discovery
import { discoverWorkflowsWithConfig } from '@archon/workflows/workflow-discovery';
// signature: (cwd: string, loadConfig: LoadConfigFn, options?) => Promise<WorkflowLoadResult>

// YAML parsing + validation
import { parseWorkflow } from '@archon/workflows/loader';
// signature: (content: string, filename: string) => WorkflowDefinition  (throws on invalid)

// Name resolution (4-tier: exact → case-insensitive → suffix → substring)
import { resolveWorkflowName } from '@archon/workflows/router';

// Event emitter singleton
import {
  getWorkflowEventEmitter,
  resetWorkflowEventEmitter,
} from '@archon/workflows/event-emitter';

// Run state constants
import {
  TERMINAL_WORKFLOW_STATUSES, // ['completed', 'failed', 'cancelled']
  RESUMABLE_WORKFLOW_STATUSES, // ['failed', 'paused']
  WORKFLOW_EVENT_TYPES, // all event type strings
  isApprovalContext, // type guard for ApprovalContext
} from '@archon/workflows/schemas/workflow-run';

// Schema for runtime validation
import { workflowDefinitionSchema } from '@archon/workflows/schemas/workflow';
```

### `@archon/core`

**Source:** `packages/core/src/index.ts`

Key exports:

```typescript
// Message handling
import { handleMessage } from '@archon/core';
// signature: (platform: IPlatformAdapter, conversationId: string, message: string,
//             context?: HandleMessageContext) => Promise<void>

// Repository management
import { cloneRepository, registerRepository } from '@archon/core';
// cloneRepository(url: string, allowEnvKeys?: boolean): Promise<{ codebaseId, alreadyExisted }>
// registerRepository(path: string, allowEnvKeys?: boolean): Promise<{ codebaseId, alreadyExisted }>

// Configuration
import { loadConfig, toSafeConfig, updateGlobalConfig } from '@archon/core';

// Concurrency
import { ConversationLockManager } from '@archon/core';
// lockManager.acquireLock(conversationId, handler) → Promise<{ status }>
// lockManager.getStats() → { active, queuedTotal, maxConcurrent }

// Database
import { pool, getDatabase, getDatabaseType, closeDatabase } from '@archon/core';
import * as conversationDb from '@archon/core/db/conversations';
import * as codebaseDb from '@archon/core/db/codebases';
import * as workflowDb from '@archon/core/db/workflows';
import * as workflowEventDb from '@archon/core/db/workflow-events';
import * as messageDb from '@archon/core/db/messages';

// AI clients
import { ClaudeClient, CodexClient, getAssistantClient } from '@archon/core';

// Workflow store adapter (bridges IWorkflowStore ↔ core DB)
import { createWorkflowStore } from '@archon/core';

// Error utilities
import { ConversationNotFoundError, EnvLeakError, scanPathForSensitiveKeys } from '@archon/core';
import { classifyAndFormatError, sanitizeCredentials, sanitizeError } from '@archon/core';

// Services
import { startCleanupScheduler, stopCleanupScheduler, generateAndSetTitle } from '@archon/core';
```

### `@archon/git`

**Source:** `packages/git/src/`

```typescript
import {
  // Branded type constructors
  toRepoPath, // (path: string) => RepoPath
  toBranchName, // (name: string) => BranchName
  toWorktreePath, // (path: string) => WorktreePath

  // Subprocess
  execFileAsync, // (cmd: string, args: string[], options?) => Promise<{ stdout, stderr }>

  // Repo operations
  findRepoRoot, // (cwd: string) => Promise<string>
  getRemoteUrl, // (repoPath: RepoPath) => Promise<string>
  extractOwnerRepo, // (repoPath: RepoPath) => Promise<{ owner, repo }>
  syncWorkspace, // (repoPath: RepoPath) => Promise<void>
  cloneRepository, // (url: string, dest: string) => Promise<void>

  // Branch operations
  getDefaultBranch, // (repoPath: RepoPath) => Promise<string>
  checkout, // (repoPath: RepoPath, branch: BranchName) => Promise<void>
  hasUncommittedChanges, // (repoPath: RepoPath) => Promise<boolean>
  commitAllChanges, // (repoPath: RepoPath, message: string) => Promise<void>
  isBranchMerged, // (repoPath: RepoPath, branch: BranchName, base: BranchName) => Promise<boolean>
  isPatchEquivalent, // (repoPath: RepoPath, branch1: BranchName, branch2: BranchName) => Promise<boolean>
  isAncestorOf, // (repoPath: RepoPath, ancestor: BranchName, descendant: BranchName) => Promise<boolean>

  // Worktree operations
  worktreeExists, // (repoPath: RepoPath, branch: BranchName) => Promise<boolean>
  listWorktrees, // (repoPath: RepoPath) => Promise<WorktreeInfo[]>
  findWorktreeByBranch, // (repoPath: RepoPath, branch: BranchName) => Promise<WorktreeInfo | null>
  removeWorktree, // (repoPath: RepoPath, worktreePath: WorktreePath) => Promise<void>
} from '@archon/git';
```

### `@archon/isolation`

**Source:** `packages/isolation/src/`

```typescript
import {
  // Configuration
  configureIsolation, // (config: IsolationConfig) => void
  getIsolationProvider, // () => IIsolationProvider

  // Error handling
  IsolationBlockedError, // extends Error; thrown when user already notified — stop processing
  classifyIsolationError, // (err: Error) => string (user-friendly message)
  isKnownIsolationError, // (err: Error) => boolean

  // Types
  type IsolationRequest,
  type IsolatedEnvironment,
  type IIsolationProvider,
  type IsolationHints,
} from '@archon/isolation';
```

### `@archon/paths`

**Source:** `packages/paths/src/`

```typescript
import {
  createLogger, // (name: string) => pino.Logger
  setLogLevel, // (level: string) => void
  getArchonHome, // () => string  (~/.archon/ or ARCHON_HOME)
  getArchonWorkspacesPath, // () => string  (~/.archon/workspaces/)
  getWorkflowFolderSearchPaths, // (cwd: string) => string[]
  getCommandFolderSearchPaths, // (cwd: string) => string[]
  getDefaultWorkflowsPath,
  getDefaultCommandsPath,
  getRunArtifactsPath, // (owner, repo, runId) => string
  isDocker, // () => boolean
} from '@archon/paths';
```

---

## Authentication & Authorization

### AI Assistant Authentication

**Claude:**

| Method      | Configuration                     | Notes                                          |
| ----------- | --------------------------------- | ---------------------------------------------- |
| Global auth | `CLAUDE_USE_GLOBAL_AUTH=true`     | Uses `claude /login` credentials (recommended) |
| OAuth token | `CLAUDE_CODE_OAUTH_TOKEN=<token>` | Explicit token                                 |
| API key     | `CLAUDE_API_KEY=<key>`            | Anthropic API key                              |

Auto-detect: if neither `CLAUDE_API_KEY` nor `CLAUDE_CODE_OAUTH_TOKEN` is set, defaults to `CLAUDE_USE_GLOBAL_AUTH=true`.

**Codex:**

Requires tokens from `~/.codex/auth.json` (after `codex login`):

```
CODEX_ID_TOKEN=
CODEX_ACCESS_TOKEN=
CODEX_REFRESH_TOKEN=
CODEX_ACCOUNT_ID=
```

### Webhook Authentication

**GitHub:** HMAC-SHA256 signature verification using `WEBHOOK_SECRET`. Header: `x-hub-signature-256: sha256=<hex>`. Verified with `timingSafeEqual` to prevent timing attacks.

**Gitea:** HMAC signature in `x-gitea-signature` header. Uses `GITEA_WEBHOOK_SECRET`.

**GitLab:** Token in `x-gitlab-token` header. Uses `GITLAB_WEBHOOK_SECRET`.

### Platform Adapter Authorization

Authorization is encapsulated inside each platform adapter (not in the orchestrator). Silent rejection for unauthorized users (no error response). Unauthorized attempts are logged with masked user IDs.

| Platform | Env var                     | Format                                              |
| -------- | --------------------------- | --------------------------------------------------- |
| Slack    | `SLACK_ALLOWED_USER_IDS`    | Comma-separated Slack member IDs                    |
| Telegram | `TELEGRAM_ALLOWED_USER_IDS` | Comma-separated Telegram user IDs                   |
| Discord  | `DISCORD_ALLOWED_USER_IDS`  | Comma-separated Discord user IDs                    |
| GitHub   | `GITHUB_ALLOWED_USERS`      | Comma-separated GitHub usernames (case-insensitive) |
| Gitea    | `GITEA_ALLOWED_USERS`       | Comma-separated Gitea usernames                     |

When the env var is empty or unset, all users are accepted.

### Web UI Authentication

No built-in auth. Options when exposing to the internet:

- **Caddy Basic Auth:** Set `CADDY_BASIC_AUTH` (bcrypt hash); see `.env.example`
- **Caddy Form Auth:** HTML login page via `auth-service`; requires `AUTH_USERNAME`, `AUTH_PASSWORD_HASH`, `COOKIE_SECRET`
- **IP allowlist:** Use firewall rules and leave auth env vars unset

### Env-Leak Gate

When registering a codebase, Archon scans the target repo's `.env` for sensitive keys. If found, the request is rejected with 422 unless `allowEnvKeys: true` is set. Every grant/revoke is audit-logged at `warn` level.

---

## Error Reference

### Error Response Format

All JSON API errors use the standard error schema:

```json
{ "error": "human-readable error message" }
```

Some endpoints include a `detail` field for machine-parseable context:

```json
{ "error": "Codebase not found", "detail": "No codebase with id \"abc123\"" }
```

**Source:** `packages/server/src/routes/schemas/common.schemas.ts:8`

### HTTP Status Codes

| Code | Meaning               | Common causes                                                                             |
| ---- | --------------------- | ----------------------------------------------------------------------------------------- |
| 200  | OK                    | Success                                                                                   |
| 201  | Created               | New codebase registered                                                                   |
| 400  | Bad Request           | Missing/invalid fields, invalid `cwd`, path traversal attempt, wrong status for operation |
| 404  | Not Found             | Conversation, workflow, codebase, or run not found                                        |
| 422  | Unprocessable Entity  | Env-leak gate blocked registration (sensitive `.env` keys found)                          |
| 500  | Internal Server Error | Database error, workflow discovery failure, file I/O error                                |

### Status-Dependent Errors

| Operation   | Rejected statuses                              | Error                                          |
| ----------- | ---------------------------------------------- | ---------------------------------------------- |
| Cancel run  | `completed`, `failed`, `cancelled`             | `Cannot cancel workflow in '<status>' status`  |
| Resume run  | `pending`, `running`, `completed`, `cancelled` | `Cannot resume workflow in '<status>' status`  |
| Abandon run | `completed`, `failed`, `cancelled`             | `Cannot abandon workflow in '<status>' status` |
| Approve run | Any except `paused`                            | `Cannot approve workflow in '<status>' status` |
| Reject run  | Any except `paused`                            | `Cannot reject workflow in '<status>' status`  |
| Delete run  | Non-terminal (`pending`, `running`, `paused`)  | `Cannot delete workflow in '<status>' status`  |

### Isolation Errors

**Source:** `packages/isolation/src/errors.ts`

`classifyIsolationError(err: Error): string` maps git errors to user-friendly messages:

| Error pattern        | User message                             |
| -------------------- | ---------------------------------------- |
| Permission denied    | "Permission denied accessing repository" |
| Timeout              | "Git operation timed out"                |
| No space left        | "Insufficient disk space"                |
| Not a git repository | "Not a git repository"                   |
| Unknown              | "Failed to create isolated environment"  |

`IsolationBlockedError`: thrown when the user has already been notified of the error via the platform adapter. Callers must catch this and stop processing — do not re-notify.

### Validation Errors

OpenAPI route validation failures return 400 with the Zod error details via `defaultHook`:

```json
{
  "error": "Validation failed",
  "detail": [{ "path": ["message"], "message": "String must contain at least 1 character(s)" }]
}
```

### Workflow Definition Errors

`parseWorkflow()` throws with a structured error message for invalid YAML:

- `read_error` — file could not be read
- `parse_error` — invalid YAML syntax
- `validation_error` — schema validation failed (e.g. missing `name`, invalid `trigger_rule`, provider/model mismatch)

Discovery failures are non-fatal: broken files appear in the `errors` array of `GET /api/workflows` without blocking valid workflows.

### Recovery Strategies

| Error type                     | Recommended action                                                              |
| ------------------------------ | ------------------------------------------------------------------------------- |
| 400 Invalid `cwd`              | Verify `cwd` matches a path under a registered codebase's `default_cwd`         |
| 404 Conversation not found     | Create conversation first via `POST /api/conversations`                         |
| 422 Env-leak gate              | Review the flagged `.env` file; pass `allowEnvKeys: true` to explicitly consent |
| 500 Workflow discovery failed  | Check if `cwd` directory still exists; re-register codebase if needed           |
| `IsolationBlockedError`        | Stop processing; user already notified; no further action needed                |
| Run `paused` awaiting approval | Call `POST /api/workflows/runs/{runId}/approve` or `reject`                     |
| Run `failed`                   | Call `POST /api/workflows/runs/{runId}/resume`, then re-run the workflow        |
