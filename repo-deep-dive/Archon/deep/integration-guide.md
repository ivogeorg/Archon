---
generated_by: repo-deep-dive workflow
date: 2026-04-09
depth: deep
---

# Archon Integration & Extension Guide

_Audience: engineers building a "God-mode" automation dashboard that orchestrates multiple AI coding projects in parallel._

---

## Table of Contents

1. [Integration Patterns](#integration-patterns)
   - [As a Service (REST/SSE)](#as-a-service-restsse)
   - [As a Library (direct import)](#as-a-library-direct-import)
   - [As a CLI Subprocess](#as-a-cli-subprocess)
   - [As a Platform Adapter](#as-a-platform-adapter)
2. [Extension Points](#extension-points)
   - [IPlatformAdapter — Custom Output Channels](#iplatformadapter--custom-output-channels)
   - [IAssistantClient — Custom AI Backends](#iassistantclient--custom-ai-backends)
   - [IWorkflowStore — Custom Persistence](#iworkflowstore--custom-persistence)
   - [IIsolationProvider — Custom Isolation Strategies](#iisolationprovider--custom-isolation-strategies)
   - [WorkflowEventEmitter — Observability Hooks](#workfloweventemitter--observability-hooks)
   - [Per-Node SDK Hooks (Claude only)](#per-node-sdk-hooks-claude-only)
   - [Configuration Overrides](#configuration-overrides)
3. [Automation Use Cases](#automation-use-cases)
   - [Batch / Bulk Operations](#batch--bulk-operations)
   - [Scheduled / Cron Workflows](#scheduled--cron-workflows)
   - [Event-Driven Triggers](#event-driven-triggers)
   - [Multi-Repo Parallel Execution](#multi-repo-parallel-execution)
4. [Parallel Deployment](#parallel-deployment)
5. [Limitations & Gotchas](#limitations--gotchas)

---

## Integration Patterns

### As a Service (REST/SSE)

**When to use:** Your dashboard is a separate process (different language, remote host, browser). You want Archon to remain a black box and communicate only through HTTP.

**Setup:** Run the server with `bun run dev:server` (port 3090 by default) or via Docker.

**Key endpoints:**

| Method | Path                                              | Purpose                                        |
| ------ | ------------------------------------------------- | ---------------------------------------------- |
| `GET`  | `/api/health`                                     | Liveness check                                 |
| `GET`  | `/api/workflows?cwd=/path/to/repo`                | List available workflows                       |
| `POST` | `/api/conversations`                              | Create a conversation                          |
| `POST` | `/api/conversations/:id/message`                  | Send user message / invoke workflow            |
| `GET`  | `/api/stream/:id`                                 | Server-Sent Events stream for real-time output |
| `GET`  | `/api/workflows/runs`                             | List workflow runs (dashboard history)         |
| `GET`  | `/api/dashboard/runs?codebaseId=X&status=running` | Filtered dashboard view                        |
| `POST` | `/api/workflows/runs/:runId/abandon`              | Cancel a run                                   |
| `POST` | `/api/workflows/runs/:runId/resume`               | Mark a failed run as resumable                 |
| `GET`  | `/api/openapi.json`                               | Full OpenAPI spec for client generation        |

**Minimal working example (TypeScript fetch):**

```typescript
const BASE = 'http://localhost:3090';

// 1. Create a conversation
const { id: convId } = await fetch(`${BASE}/api/conversations`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({}),
}).then(r => r.json());

// 2. Subscribe to SSE FIRST (before triggering work, to avoid missing early events)
const evtSource = new EventSource(`${BASE}/api/stream/${convId}`);
evtSource.onmessage = e => {
  const chunk = JSON.parse(e.data);
  // chunk.type: 'assistant' | 'tool' | 'tool_result' | 'result' | 'workflow_status' | ...
  console.log(chunk);
};

// 3. Trigger a workflow
await fetch(`${BASE}/api/conversations/${convId}/message`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: 'Use archon to fix issue #42' }),
});

// 4. Poll for terminal state if you prefer polling over SSE
const poll = async (runId: string) => {
  while (true) {
    const run = await fetch(`${BASE}/api/workflows/runs/${runId}`).then(r => r.json());
    if (['completed', 'failed', 'cancelled'].includes(run.status)) return run;
    await new Promise(res => setTimeout(res, 2000));
  }
};
```

**Caveats:**

- The SSE stream is ephemeral (in-process `EventEmitter`). If the server restarts while a workflow is running, the stream disconnects. Reconnecting replays nothing — subscribe before triggering.
- `POST /api/conversations/:id/message` returns `200 OK` immediately; the workflow runs asynchronously. Watch the SSE stream or poll `/api/workflows/runs`.
- Attach a codebase first (`POST /api/codebases`) if you want the workflow to run in a specific repo. Otherwise Archon has no `cwd` and will refuse most agentic operations.
- The OpenAPI spec at `/api/openapi.json` is generated from Zod schemas — use it to auto-generate a typed client with `openapi-typescript` or similar.

**Performance considerations:**

- SQLite is the default; it serializes writes. Under high parallelism (10+ simultaneous workflow runs), use PostgreSQL (`DATABASE_URL` env var).
- SSE streams are held open per conversation. Each stream is a kept-alive HTTP connection. For a dashboard with 50 active conversations, that is 50 open connections to the server.

---

### As a Library (direct import)

**When to use:** Your dashboard is also a Bun/Node TypeScript project in the same monorepo (or imports `@archon/core` and `@archon/workflows` as packages). You need fine-grained control over the execution lifecycle.

**Setup:** Add `@archon/core` and `@archon/workflows` as workspace dependencies.

**Minimal working example:**

```typescript
import { createWorkflowDeps } from '@archon/core/workflows/store-adapter';
import { executeWorkflow } from '@archon/workflows/executor';
import { parseWorkflow } from '@archon/workflows/loader';
import { discoverWorkflowsWithConfig } from '@archon/workflows/workflow-discovery';
import { CLIAdapter } from '@archon/cli/adapters/cli-adapter';
import * as conversationDb from '@archon/core/db/conversations';

// Build the dependency injection bundle (wires DB + AI clients + config)
const deps = createWorkflowDeps();

// Create a minimal platform adapter (stdout)
const platform = new CLIAdapter({ streamingMode: 'stream' });

// Discover workflows for a specific repo
const { workflows } = await discoverWorkflowsWithConfig('/path/to/repo');

// Parse a specific workflow
const wfResult = parseWorkflow(rawYaml, 'my-workflow.yaml');
if (!wfResult.ok) throw new Error(wfResult.error);
const workflow = wfResult.workflow;

// Create a DB conversation record
const conv = await conversationDb.createConversation(
  'cli',
  `cli-${Date.now()}`,
  null, // codebase_id (optional)
  null, // cwd (optional, workflow executor reads from cwd param)
  'claude',
  false // hidden
);

// Execute
const result = await executeWorkflow(
  deps,
  platform,
  conv.platform_conversation_id, // conversationId
  '/path/to/repo', // cwd
  workflow,
  'Fix issue #42', // userMessage
  conv.id, // conversationDbId
  undefined // codebaseId (optional)
);

console.log(result.status, result.summary);
```

**Key injection point — `WorkflowDeps`** (`packages/workflows/src/deps.ts:273`):

```typescript
interface WorkflowDeps {
  store: IWorkflowStore; // DB abstraction
  getAssistantClient: AssistantClientFactory; // 'claude' | 'codex' → IWorkflowAssistantClient
  loadConfig: (cwd: string) => Promise<WorkflowConfig>;
}
```

`createWorkflowDeps()` (in `packages/core/src/workflows/store-adapter.ts:69`) is the canonical factory — use it instead of assembling `WorkflowDeps` by hand.

**Caveats:**

- `@archon/workflows` must never import `@archon/core` — that would create a circular dependency. All wiring goes through the `WorkflowDeps` injection point.
- The database singleton is lazily initialized on first call to `getDatabase()`. It reads `DATABASE_URL` from environment. Set it before any import resolves, or use `dotenv` before importing `@archon/core`.
- `executeWorkflow` is not re-entrant for the same `cwd` path — it detects concurrent runs and refuses to start a second one unless the first is terminal.

**Performance considerations:**

- Library usage runs everything in-process; no HTTP overhead. Best for tightly integrated dashboards.
- Workflow runs are async but `executeWorkflow` itself is `await`-able. Parallelism requires `Promise.all` across different `cwd` paths.

---

### As a CLI Subprocess

**When to use:** Your dashboard orchestrates Archon from a non-TypeScript environment, or you want strict process isolation per workflow run (crash in one run does not affect others).

**Setup:** Install the binary (`curl -fsSL https://archon.diy/install | bash`) or use `bun run cli` from the repo root.

**Invocation patterns:**

```bash
# Run a workflow and capture output
archon workflow run fix-issue "Fix issue #42" --branch archon/issue-42

# Machine-readable output (JSON lines on stdout)
archon workflow run assist "What does the orchestrator do?" 2>/dev/null

# Check status of all running workflows (JSON)
archon workflow status --json

# Resume a failed run
archon workflow resume <run-id>

# Emit an event from within a workflow (for interactive loop gates)
archon workflow event emit --run-id <uuid> --type APPROVED

# List isolation environments (worktrees)
archon isolation list --json

# Clean up stale environments
archon isolation cleanup 7
```

**Output parsing:**

The CLI does not emit structured JSON natively for `workflow run` — output goes to stdout as human-readable text. For structured monitoring, use the REST API (`GET /api/workflows/runs`) which shares the same SQLite/PostgreSQL database.

Combine both: spawn the CLI subprocess for execution, then poll the REST API for status.

```typescript
import { spawn } from 'child_process';

function runWorkflow(name: string, message: string, cwd: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const proc = spawn('archon', ['workflow', 'run', name, message], {
      cwd,
      stdio: ['ignore', 'pipe', 'pipe'],
      env: { ...process.env },
    });

    proc.stdout.on('data', chunk => process.stdout.write(chunk));
    proc.stderr.on('data', chunk => process.stderr.write(chunk));
    proc.on('close', code => (code === 0 ? resolve() : reject(new Error(`Exit ${code}`))));
  });
}
```

**Caveats:**

- The CLI must run from inside a git repository (or a subdirectory of one). It calls `git rev-parse --show-toplevel` to find the repo root.
- The CLI loads `~/.archon/.env` with `override: true` before anything else. Set your tokens there, not in the system environment — the CLI will override them anyway.
- `--no-worktree` skips isolation and runs in the live checkout. Dangerous for parallel runs — two CLIs writing to the same directory will conflict. Never use for parallel execution.

---

### As a Platform Adapter

**When to use:** You want Archon to receive messages from a custom channel (internal chat, webhook, custom UI) and route them through the same orchestrator pipeline as Slack/Telegram/GitHub.

**Interface** (`packages/core/src/types/index.ts:117`):

```typescript
interface IPlatformAdapter {
  sendMessage(conversationId: string, message: string, metadata?: MessageMetadata): Promise<void>;
  ensureThread(originalConversationId: string, messageContext?: unknown): Promise<string>;
  getStreamingMode(): 'stream' | 'batch';
  getPlatformType(): string;
  start(): Promise<void>;
  stop(): void;

  // Optional — only needed for rich structured output (Web UI uses this):
  sendStructuredEvent?(conversationId: string, event: MessageChunk): Promise<void>;
  emitRetract?(conversationId: string): Promise<void>;
}
```

**Minimal implementation:**

```typescript
import type { IPlatformAdapter, MessageMetadata } from '@archon/core';

export class WebhookAdapter implements IPlatformAdapter {
  private handler: ((event: SlackMessageEvent) => Promise<void>) | null = null;

  constructor(private readonly webhookUrl: string) {}

  async sendMessage(
    conversationId: string,
    message: string,
    _meta?: MessageMetadata
  ): Promise<void> {
    // POST the message back to your webhook endpoint
    await fetch(this.webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ conversationId, message }),
    });
  }

  async ensureThread(id: string): Promise<string> {
    return id; // No threading — passthrough
  }

  getStreamingMode(): 'stream' | 'batch' {
    return 'batch'; // Batch: accumulate then send once
  }

  getPlatformType(): string {
    return 'webhook';
  }

  async start(): Promise<void> {
    // Start polling, WebSocket, etc.
  }

  stop(): void {
    // Teardown
  }
}
```

**Wiring into the orchestrator:**

```typescript
import { handleMessage, ConversationLockManager } from '@archon/core';
import * as conversationDb from '@archon/core/db/conversations';

const adapter = new WebhookAdapter('https://your-dashboard/callback');
const lockManager = new ConversationLockManager();

// On incoming message from your custom channel:
async function onIncomingMessage(conversationId: string, text: string) {
  // Ensure conversation exists in DB
  let conv = await conversationDb.getConversation('webhook', conversationId);
  if (!conv) {
    conv = await conversationDb.createConversation(
      'webhook',
      conversationId,
      null,
      null,
      'claude',
      false
    );
  }

  const result = await lockManager.acquireLock(conversationId);
  // 'started' | 'queued-conversation' | 'queued-capacity'
  if (result.status !== 'started') {
    await adapter.sendMessage(conversationId, `Your request is queued (${result.status}).`);
  }

  try {
    await handleMessage(adapter, conv, text, lockManager);
  } finally {
    lockManager.releaseLock(conversationId);
  }
}
```

**Caveats:**

- Authorization must live inside the adapter. The orchestrator never checks identity — it trusts whatever `conversationId` arrives. Silent-reject unauthorized users before calling `handleMessage`.
- `ensureThread()` is only meaningful for platforms that have threads (Slack, GitHub). For everything else, return `originalConversationId` unchanged.
- `getStreamingMode()` controls how the orchestrator buffers AI output: `'stream'` sends each text chunk immediately, `'batch'` accumulates and sends once at the end. Use `'batch'` for webhook adapters that cannot receive many rapid calls.

---

## Extension Points

### IPlatformAdapter — Custom Output Channels

**File:** `packages/core/src/types/index.ts:117`

See the [As a Platform Adapter](#as-a-platform-adapter) section for the interface and a complete implementation example.

**Key design rules:**

- Auth checks belong inside `sendMessage` / message handlers — never in middleware.
- Parse any whitelist env vars in the constructor so they are frozen at startup.
- Use the lazy logger pattern (`let cachedLog; function getLog() { ... }`) — this allows tests to mock `createLogger` before the logger is instantiated.
- `sendStructuredEvent` is optional and only needed if your platform can render rich structured data (tool call cards, workflow status chips). Standard text-only adapters omit it.

---

### IAssistantClient — Custom AI Backends

**File:** `packages/core/src/types/index.ts:364`

```typescript
interface IAssistantClient {
  sendQuery(
    prompt: string,
    cwd: string,
    resumeSessionId?: string,
    options?: AssistantRequestOptions
  ): AsyncGenerator<MessageChunk>;

  getType(): string;
}
```

**`MessageChunk` discriminated union** (`packages/core/src/types/index.ts:196`):

```typescript
type MessageChunk =
  | { type: 'assistant'; content: string }
  | { type: 'thinking'; content: string }
  | { type: 'tool'; toolName: string; toolInput?: Record<string, unknown>; toolCallId?: string }
  | { type: 'tool_result'; toolName: string; toolOutput: string; toolCallId?: string }
  | {
      type: 'result';
      sessionId?: string;
      tokens?: TokenUsage;
      isError?: boolean;
      stopReason?: string;
    }
  | { type: 'rate_limit'; rateLimitInfo: Record<string, unknown> }
  | { type: 'workflow_dispatch'; workerConversationId: string; workflowName: string };
```

**Registration:**

Implement `IAssistantClient`, then update `packages/core/src/clients/factory.ts` to return your client when `getType()` is called with a new identifier string. Register the new type in `WorkflowDeps.getAssistantClient` factory.

**Minimal skeleton:**

```typescript
import type { IAssistantClient, AssistantRequestOptions, MessageChunk } from '@archon/core';

export class MyAIClient implements IAssistantClient {
  getType(): string {
    return 'my-ai';
  }

  async *sendQuery(
    prompt: string,
    cwd: string,
    resumeSessionId?: string,
    options?: AssistantRequestOptions
  ): AsyncGenerator<MessageChunk> {
    // Call your AI API...
    yield { type: 'assistant', content: 'Hello from my AI!' };
    yield { type: 'result', stopReason: 'end_turn' };
  }
}
```

---

### IWorkflowStore — Custom Persistence

**File:** `packages/workflows/src/store.ts:33`

```typescript
interface IWorkflowStore {
  createWorkflowRun(data: { ... }): Promise<WorkflowRun>;
  getWorkflowRun(id: string): Promise<WorkflowRun | null>;
  updateWorkflowRun(id: string, updates: Partial<Pick<WorkflowRun, 'status' | 'metadata'>>): Promise<void>;
  completeWorkflowRun(id: string, metadata?: Record<string, unknown>): Promise<void>;
  failWorkflowRun(id: string, error: string): Promise<void>;
  pauseWorkflowRun(id: string, approvalContext: ApprovalContext): Promise<void>;
  cancelWorkflowRun(id: string): Promise<void>;
  // createWorkflowEvent MUST NOT throw — catch internally
  createWorkflowEvent(data: { workflow_run_id: string; event_type: WorkflowEventType; ... }): Promise<void>;
  getCompletedDagNodeOutputs(workflowRunId: string): Promise<Map<string, string>>;
  getCodebaseEnvVars(codebaseId: string): Promise<Record<string, string>>;
  getCodebase(id: string): Promise<{ id: string; name: string; ... } | null>;
  // ... (full interface in store.ts)
}
```

**Use case:** Replace the SQLite/PostgreSQL backend with a remote store (Redis, DynamoDB, custom API) for a distributed multi-instance deployment.

The canonical implementation is `createWorkflowStore()` in `packages/core/src/workflows/store-adapter.ts:28`. Build a wrapper or replacement that satisfies the same interface.

**Critical contract:** `createWorkflowEvent` must never throw. Swallow all errors and log them internally — the executor treats event persistence as optional observability, not critical path.

---

### IIsolationProvider — Custom Isolation Strategies

**File:** `packages/isolation/src/types.ts:168`

```typescript
interface IIsolationProvider {
  readonly providerType: IsolationProviderType; // 'worktree' | 'container' | 'vm' | 'remote'

  create(request: IsolationRequest): Promise<IsolatedEnvironment>;
  destroy(envId: string, options?: DestroyOptions): Promise<DestroyResult>;
  get(envId: string): Promise<IsolatedEnvironment | null>;
  list(codebaseId: string): Promise<IsolatedEnvironment[]>;
  healthCheck(envId: string): Promise<boolean>;
  adopt?(path: string): Promise<IsolatedEnvironment | null>; // optional
}
```

**Use case:** Container-based isolation (Docker, Podman) instead of git worktrees. Implement `IIsolationProvider`, return a `DestroyResult` describing what was cleaned up (partial failure is OK — use `warnings` for non-fatal issues).

**Wiring:** `getIsolationProvider()` in `packages/isolation/src/factory.ts` selects the provider. Add a new branch to return your custom provider when `providerType` matches.

**Error contract:**

- `create` throws on failure (caller surfaces to user via `classifyIsolationError`)
- `destroy` returns `DestroyResult` — never throws for partial cleanup failures
- `get` returns null if not found, throws for unexpected I/O errors

---

### WorkflowEventEmitter — Observability Hooks

**File:** `packages/workflows/src/event-emitter.ts:249`

This is the primary hook surface for building a real-time dashboard. It is a singleton in-process event bus — every workflow execution emits events through it.

```typescript
import { getWorkflowEventEmitter } from '@archon/workflows';

const emitter = getWorkflowEventEmitter();

// Subscribe to ALL workflow events
const unsubscribe = emitter.subscribe((event) => {
  switch (event.type) {
    case 'workflow_started':
      console.log(`Run ${event.runId} started for workflow ${event.workflowName}`);
      break;
    case 'node_completed':
      console.log(`Node ${event.nodeId} done in ${event.duration}ms, cost $${event.costUsd}`);
      break;
    case 'workflow_failed':
      console.error(`Run ${event.runId} failed: ${event.error}`);
      break;
  }
});

// Or subscribe per conversation (filters by runId→conversationId mapping)
emitter.registerRun(runId, conversationId); // called automatically by executor
const unsub = emitter.subscribeForConversation(conversationId, (event) => { ... });

// Cleanup
unsubscribe();
```

**Event types** (full `WorkflowEmitterEvent` union):

| Event type                        | Key fields                                                         |
| --------------------------------- | ------------------------------------------------------------------ |
| `workflow_started`                | `runId`, `workflowName`, `conversationId`                          |
| `workflow_completed`              | `runId`, `workflowName`, `duration`                                |
| `workflow_failed`                 | `runId`, `workflowName`, `error`                                   |
| `node_started`                    | `runId`, `nodeId`, `nodeName`                                      |
| `node_completed`                  | `runId`, `nodeId`, `duration`, `costUsd`, `stopReason`, `numTurns` |
| `node_failed`                     | `runId`, `nodeId`, `error`                                         |
| `node_skipped`                    | `runId`, `nodeId`, `reason`                                        |
| `loop_iteration_started`          | `runId`, `nodeId`, `iteration`, `maxIterations`                    |
| `loop_iteration_completed`        | `runId`, `iteration`, `completionDetected`                         |
| `tool_started` / `tool_completed` | `runId`, `toolName`, `stepName`, `durationMs`                      |
| `approval_pending`                | `runId`, `nodeId`, `message`                                       |
| `workflow_cancelled`              | `runId`, `nodeId`, `reason`                                        |
| `workflow_artifact`               | `runId`, `artifactType`, `label`, `url`, `path`                    |

**Listener errors never propagate to the executor.** If your listener throws, it is logged and the workflow continues.

---

### Per-Node SDK Hooks (Claude only)

**File:** `packages/workflows/src/schemas/hooks.ts`

Archon exposes Claude Agent SDK hooks at the YAML level, allowing you to intercept and respond to SDK lifecycle events without writing code.

```yaml
# .archon/workflows/my-workflow.yaml
nodes:
  - id: implement
    prompt: 'Implement the feature'
    hooks:
      PreToolUse:
        - matcher: 'Bash' # regex matched against tool name
          response:
            decision: 'allow' # or "block" with a reason
          timeout: 30 # seconds
      PostToolUse:
        - response:
            decision: 'allow'
```

**Supported hook events** (from `workflowHookEventSchema.options`):

`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PermissionRequest`, `Setup`, `TeammateIdle`, `TaskCompleted`, `Elicitation`, `ElicitationResult`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, `InstructionsLoaded`

**For programmatic hooks** (via library integration), use `AssistantRequestOptions.hooks` (`packages/core/src/types/index.ts:268`):

```typescript
const options: AssistantRequestOptions = {
  hooks: {
    PreToolUse: [
      {
        matcher: 'Bash',
        hooks: [
          async (input, toolUseID, { signal }) => {
            // Return an SDK SyncHookJSONOutput
            return { decision: 'allow' };
          },
        ],
        timeout: 30000, // ms
      },
    ],
  },
};
```

---

### Configuration Overrides

**Global config** (`~/.archon/config.yaml`):

```yaml
assistants:
  claude:
    model: sonnet # Default model for all Claude nodes
    settingSources: # Which CLAUDE.md files to load
      - project
      - user
  codex:
    model: gpt-5.3-codex
    modelReasoningEffort: medium
    webSearchMode: live

concurrency:
  maxConversations: 20 # Raise from default 10 for parallel dashboard

defaults:
  loadDefaultWorkflows: false # Only use repo-specific workflows
```

**Per-repo config** (`.archon/config.yaml`):

```yaml
assistant: claude
worktree:
  baseBranch: main
  copyFiles:
    - .env.local
    - .archon/
docs:
  path: docs/
env:
  MY_API_KEY: 'value' # Injected into Claude subprocess env
```

**Env var injection per codebase** (Web UI / API):

```bash
# Via API — persisted to DB, merged on top of config.yaml env:
curl -X PATCH http://localhost:3090/api/codebases/:id/env-vars \
  -H "Content-Type: application/json" \
  -d '{"key": "MY_TOOL_TOKEN", "value": "..."}'
```

**Priority (highest wins):** DB env vars → `config.yaml env:` section → process environment.

---

## Automation Use Cases

### Batch / Bulk Operations

**Use case:** Run the same workflow across 20 repositories, collect all results.

```typescript
import { createWorkflowDeps } from '@archon/core/workflows/store-adapter';
import { executeWorkflow } from '@archon/workflows/executor';
import { CLIAdapter } from '@archon/cli/adapters/cli-adapter';
import * as conversationDb from '@archon/core/db/conversations';
import { parseWorkflow } from '@archon/workflows/loader';
import { readFileSync } from 'fs';

const deps = createWorkflowDeps();
const yaml = readFileSync('.archon/workflows/archon-fix-github-issue.yaml', 'utf8');
const wfResult = parseWorkflow(yaml, 'archon-fix-github-issue.yaml');
if (!wfResult.ok) throw new Error(wfResult.error);
const workflow = wfResult.workflow;

const repos = [
  { path: '/home/user/projects/api', issue: 42 },
  { path: '/home/user/projects/ui', issue: 17 },
  // ...
];

const MAX_PARALLEL = 5; // Limited by Claude API rate limits and CPU

async function processRepo(repo: (typeof repos)[0]) {
  const adapter = new CLIAdapter({ streamingMode: 'batch' });
  const convId = `batch-${Date.now()}-${Math.random().toString(36).slice(2)}`;
  const conv = await conversationDb.createConversation(
    'cli',
    convId,
    null,
    repo.path,
    'claude',
    false
  );
  adapter.setConversationDbId(convId, conv.id);

  return executeWorkflow(
    deps,
    adapter,
    convId,
    repo.path,
    workflow,
    `Fix issue #${repo.issue}`,
    conv.id
  );
}

// Process in chunks to stay within rate limits
for (let i = 0; i < repos.length; i += MAX_PARALLEL) {
  const chunk = repos.slice(i, i + MAX_PARALLEL);
  const results = await Promise.allSettled(chunk.map(processRepo));
  results.forEach((r, j) => {
    const repo = chunk[j];
    if (r.status === 'fulfilled') {
      console.log(`✓ ${repo.path}: ${r.value.status}`);
    } else {
      console.error(`✗ ${repo.path}: ${r.reason}`);
    }
  });
}
```

**Key constraint:** `executeWorkflow` detects concurrent runs for the same `working_path` and refuses to start a duplicate. Each repo must have a distinct `cwd`. Do NOT run two workflows against the same directory simultaneously.

---

### Scheduled / Cron Workflows

**Use case:** Run `archon-architect` on every project every Sunday.

**Option 1 — Cron job calling CLI:**

```bash
# crontab entry
0 2 * * 0 /usr/local/bin/archon workflow run archon-architect "Weekly health check" --cwd /path/to/repo >> /var/log/archon-cron.log 2>&1
```

**Option 2 — REST API call from a scheduler (e.g., GitHub Actions cron):**

```yaml
# .github/workflows/weekly-audit.yml
on:
  schedule:
    - cron: '0 2 * * 0'
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Archon workflow
        run: |
          CONV=$(curl -sX POST https://archon.example.com/api/conversations \
            -H "Content-Type: application/json" -d '{}' | jq -r '.id')
          curl -sX POST https://archon.example.com/api/conversations/$CONV/message \
            -H "Content-Type: application/json" \
            -d '{"message": "Run archon-architect on this codebase"}'
```

**Option 3 — In-process scheduler (library mode):**

```typescript
import cron from 'node-cron';
import { runWorkflowForRepo } from './my-runner';

cron.schedule('0 2 * * 0', async () => {
  for (const repo of await getRegisteredRepos()) {
    await runWorkflowForRepo(repo.path, 'archon-architect', 'Weekly health check').catch(err =>
      log.error({ err, repo: repo.path }, 'scheduled_run_failed')
    );
  }
});
```

---

### Event-Driven Triggers

**Use case:** Automatically fix a GitHub issue the moment it is labeled `archon`.

**Via GitHub webhook** (built-in):

Archon's GitHub adapter already listens for `issue_comment.created` events. Add `@archon fix this` as a comment on any issue — no custom code needed.

**Custom webhook trigger:**

```typescript
import Fastify from 'fastify';

const app = Fastify();

app.post('/webhook/github', async (req, reply) => {
  const event = req.headers['x-github-event'];
  const payload = req.body as GitHubWebhookPayload;

  if (event === 'issues' && payload.action === 'labeled' && payload.label.name === 'archon') {
    const issue = payload.issue.number;
    const repo = payload.repository.full_name;

    // Trigger Archon via REST API
    await fetch(`http://archon-server:3090/api/conversations`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({}),
    }).then(async r => {
      const { id } = await r.json();
      return fetch(`http://archon-server:3090/api/conversations/${id}/message`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: `Fix GitHub issue #${issue} in ${repo}`,
        }),
      });
    });
  }

  reply.code(200).send('ok');
});
```

**File-watcher trigger** (respond to code changes):

```typescript
import { watch } from 'fs';

watch('/path/to/repo/src', { recursive: true }, async (eventType, filename) => {
  if (filename?.endsWith('.ts') && eventType === 'change') {
    await triggerWorkflow('archon-validate-pr', 'Validate changes after file modification');
  }
});
```

---

### Multi-Repo Parallel Execution

**Use case:** God-mode dashboard running workflows against 10 active projects simultaneously.

**Architecture:**

```
Dashboard Server
├── Project Registry (DB codebases table)
├── ConversationLockManager (per-conversation serialization)
├── WorkflowEventEmitter subscription (global event bus)
└── REST API / SSE (per-conversation streams)

Per Project (parallel):
├── Worktree A (archon/task-fix-auth) → Claude session
├── Worktree B (archon/task-dark-mode) → Claude session
└── Live checkout (for non-isolated queries)
```

**Key pattern — per-project conversation tracking:**

```typescript
// One conversation per active task per project
// conversationId encodes project + task for routing
const convId = `dashboard-${projectId}-${taskId}`;

// Subscribe to events for this conversation
const emitter = getWorkflowEventEmitter();
const unsubscribe = emitter.subscribeForConversation(convId, event => {
  dashboardWebSocket.send(JSON.stringify({ projectId, taskId, event }));
});

// Run workflow
await executeWorkflow(deps, adapter, convId, cwd, workflow, message, dbConvId, codebaseId);

unsubscribe();
```

**Port allocation for worktrees:** When running `bun dev` from within a worktree, Archon auto-allocates a unique port using a hash of the worktree path (range 3190–4089). The same worktree always gets the same port, so you can hard-code it after the first run or compute it: `3090 + (hash(worktreePath) % 900)`.

---

## Parallel Deployment

### Port and Network Configuration

| Scenario         | Port                                  | Override            |
| ---------------- | ------------------------------------- | ------------------- |
| Main repo server | 3090                                  | `PORT=NNNN` env var |
| Worktree server  | 3190–4089 (hash-based, deterministic) | `PORT=NNNN` env var |
| Docker (default) | 3000 external → container port        | `PORT` in `.env`    |
| Multi-instance   | Each instance needs a unique `PORT`   |                     |

For multiple Archon instances on one host (e.g., one per team project), use distinct ports and a reverse proxy (Caddy, nginx) to route by subdomain or path prefix.

```yaml
# docker-compose.override.yml for instance 2
services:
  app:
    ports:
      - '3091:3090'
    environment:
      PORT: '3090'
      DATABASE_URL: 'postgresql://postgres:postgres@postgres:5432/archon_project2'
      ARCHON_HOME: '/.archon-project2'
```

### Database and State Isolation

**Shared PostgreSQL (recommended for multi-instance dashboards):**

```
Instance 1 (Project A) ────┐
Instance 2 (Project B) ────┼──▶ PostgreSQL (shared)
Instance 3 (Project C) ────┘
```

All instances share the same `remote_agent_*` tables. Conversations, workflow runs, and isolation environments are globally visible — which is intentional for a God-mode dashboard.

**Fully isolated instances (separate databases):**

Set a distinct `DATABASE_URL` per instance. Instances will not see each other's runs. Use this when different teams should have completely independent environments.

**SQLite:** One file per `ARCHON_HOME`. Do not use SQLite with multiple instances pointing at the same file — SQLite has per-process locking and will deadlock under concurrent writes.

### Credential and Secret Management

**Environment variables per instance:**

```bash
# Instance for Project A (Claude)
CLAUDE_USE_GLOBAL_AUTH=true
GH_TOKEN=ghp_projectA...
ARCHON_HOME=/opt/archon/projectA

# Instance for Project B (Codex)
CODEX_ID_TOKEN=...
CODEX_ACCESS_TOKEN=...
GH_TOKEN=ghp_projectB...
ARCHON_HOME=/opt/archon/projectB
```

**Per-codebase env vars** (injected into Claude subprocess only):

Store project-specific secrets in the database via the API (`PATCH /api/codebases/:id/env-vars`). These are merged into `Options.env` when Claude is invoked for that codebase — they do not leak across projects.

**Env-leak gate:** Archon scans `.env` files in target repos for sensitive keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.). If found, registration is blocked unless `allow_env_keys` consent is granted. This is a security guardrail — grant it deliberately per codebase, not globally.

### Shared vs Isolated Filesystem Resources

| Resource                                     | Shared?                     | Notes                                                                  |
| -------------------------------------------- | --------------------------- | ---------------------------------------------------------------------- |
| `~/.archon/archon.db` (SQLite)               | Per `ARCHON_HOME`           | Never share across instances                                           |
| `~/.archon/workspaces/`                      | Per `ARCHON_HOME`           | Each instance manages its own worktrees                                |
| `~/.archon/config.yaml`                      | Per `ARCHON_HOME`           | Global config; per-instance if distinct `ARCHON_HOME`                  |
| `.archon/workflows/` (in repo)               | Shared (it is the git repo) | All instances serving the same repo see the same workflows             |
| Claude session files (`~/.claude/projects/`) | Process-local               | Archon sets `persistSession: false` by default to avoid disk pollution |

**For Docker multi-instance:** Mount distinct volumes:

```yaml
volumes:
  - /opt/archon-data-projectA:/.archon # Instance A
  - /opt/archon-data-projectB:/.archon # Instance B
```

### Concurrency and Rate-Limit Constraints

| Limit                        | Default                    | Override                                                                           |
| ---------------------------- | -------------------------- | ---------------------------------------------------------------------------------- |
| Max concurrent conversations | 10                         | `MAX_CONCURRENT_CONVERSATIONS` env var or `concurrency.maxConversations` in config |
| Claude API rate limits       | Model-dependent            | Back off exponentially; Archon does not retry automatically                        |
| Max worktrees per project    | 25 (auto-cleanup kicks in) | Hard-coded in `IsolationResolver` — `makeRoom()` tries to free space               |
| SQLite write serialization   | 1 writer at a time         | Use PostgreSQL for >5 concurrent workflow runs                                     |

---

## Limitations & Gotchas

**1. No re-entrancy for the same `cwd`.**
`executeWorkflow` calls `getActiveWorkflowRunByPath(cwd)` and refuses to start if an active run is already using that path. This prevents two Claude agents from writing to the same worktree simultaneously. Always use distinct worktrees (different branches) for parallel tasks on the same repo.

**2. SQLite write serialization under load.**
SQLite uses process-level locking. Under 5+ concurrent workflow runs, write contention causes perceptible delays and occasional `SQLITE_BUSY` errors. Switch to PostgreSQL for a proper multi-run dashboard. The switch is zero-code — set `DATABASE_URL` and restart.

**3. In-process event emitter — single process only.**
`WorkflowEventEmitter` is a Node.js `EventEmitter` singleton. If you run multiple Archon server processes, each has its own emitter — events from process B are invisible to a subscriber in process A. For multi-process observability, subscribe to the database (poll `workflow_events` table) or use a shared pub-sub (Redis Streams, PostgreSQL `LISTEN/NOTIFY`).

**4. SSE streams are not replayed on reconnect.**
The SSE transport (`packages/server/src/adapters/web/transport.ts`) holds an in-memory stream registry. On reconnect, the client receives only events emitted after reconnection. Build a polling fallback using `GET /api/workflows/runs/:runId` for any events missed during a disconnect.

**5. `mock.module()` pollution in tests.**
Bun's `mock.module()` is process-global and irreversible — `mock.restore()` does NOT undo it. Never run `bun test` from the repo root (it runs all packages in one process, causing ~135 failures). Always use `bun run test` which uses per-package isolation.

**6. Worktree port allocation is hash-based, not guaranteed unique.**
The hash of the worktree path maps to a port in `3190–4089` (900 possible ports). With many active worktrees, hash collisions are possible. Override with `PORT=NNNN` if you detect a collision.

**7. Approval-gate workflows pause the run.**
`loop` nodes with `interactive: true` pause and wait for a human to call `POST /api/workflows/runs/:runId/approve` (or use the CLI `archon workflow event emit`). A paused run holds its worktree and blocks that branch from being reused. Automate the approval step or set a timeout in your dashboard — there is no built-in timeout for paused runs.

**8. Config cache is not invalidated on file change.**
`loadConfig()` caches config per `cwd`. If you modify `.archon/config.yaml` while Archon is running, the change is not picked up until the process restarts (or `clearConfigCache()` is called explicitly). For dynamic config in a long-running dashboard, call `clearConfigCache()` before each `executeWorkflow`.

**9. `git clean -fd` is forbidden — never use it.**
Archon intentionally never calls `git clean -fd` because it permanently deletes untracked files including `.env`, `.archon/`, and other git-ignored state. If you need to reset a worktree, use `git checkout .` (reverts tracked changes only). This constraint applies to your own automation code too — if you shell out to git in a worktree, avoid `clean`.

**10. Binary builds embed defaults; source builds read from filesystem.**
Bundled default workflows and commands are embedded in the binary at compile time. In source builds (`bun run cli`), they are loaded from `packages/workflows/src/defaults/`. If you add a custom default and build the binary, you must rebuild — the filesystem is not consulted at runtime for defaults in binary mode.

**11. Claude API concurrency with `bypass-permissions` mode.**
Each workflow node spawns a `claude-agent-sdk` subprocess with `permissionMode: 'bypassPermissions'`. Claude's API has per-minute token limits per key. Running 10 parallel nodes on the same API key will hit rate limits. Use separate API keys per project (`CLAUDE_API_KEY` or OAuth tokens) or throttle your parallel dispatch.

**12. GitHub adapter responds only to `issue_comment.created`.**
The GitHub adapter intentionally does NOT respond to `issues.opened` or `pull_request.opened`. Issue and PR descriptions often contain example commands that look like invocations but are documentation — responding to them would create false positives. Only comments trigger the bot. See anthropics/claude-code#96 for the rationale.
