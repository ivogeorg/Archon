---
generated_by: repo-deep-dive workflow
date: 2026-04-09
depth: deep
---

# Archon — Module Architecture

> Generated 2026-04-09 from commit `1d581b5a`

---

## Module Map

| Package             | Purpose                                                                     | Key Exports                                                                                                                                                  | Depends On                                                               |
| ------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `@archon/paths`     | Path resolution utilities and Pino logger factory                           | `createLogger`, `getArchonHome`, `getProjectSourcePath`, `getRunArtifactsPath`, `BUNDLED_VERSION`                                                            | _(zero @archon/_ deps)\*                                                 |
| `@archon/git`       | Git operations — worktrees, branches, repos, exec wrappers                  | `execFileAsync`, `listWorktrees`, `checkout`, `syncWorkspace`, `findWorktreeByBranch`, branded constructors (`toRepoPath`, `toBranchName`, `toWorktreePath`) | `@archon/paths`                                                          |
| `@archon/isolation` | Worktree isolation types, providers, resolver, error classifiers            | `IsolationResolver`, `WorktreeProvider`, `IIsolationProvider`, `IIsolationStore`, `classifyIsolationError`, `IsolationBlockedError`                          | `@archon/git`, `@archon/paths`                                           |
| `@archon/workflows` | Workflow YAML engine — loader, router, DAG executor, bundled defaults       | `executeWorkflow`, `discoverWorkflowsWithConfig`, `findWorkflow`, `IWorkflowStore`, `WorkflowDeps`, all Zod schemas                                          | `@archon/git`, `@archon/paths`                                           |
| `@archon/core`      | Shared business logic — AI clients, DB, orchestrator, slash command handler | `handleMessage`, `IPlatformAdapter`, `IAssistantClient`, `IDatabase`, `createWorkflowStore`, `ClaudeClient`, `CodexClient`, `ConversationLockManager`        | `@archon/git`, `@archon/paths`, `@archon/isolation`, `@archon/workflows` |
| `@archon/adapters`  | Platform adapters — Slack, Telegram, GitHub, Discord, Gitea, GitLab         | `TelegramAdapter`, `SlackAdapter`, `GitHubAdapter`, `DiscordAdapter`                                                                                         | `@archon/core`                                                           |
| `@archon/server`    | Hono HTTP server + Web adapter (SSE streaming) + REST API routes            | `WebAdapter`, `SSETransport`, `registerApiRoutes`, `WorkflowEventBridge`                                                                                     | `@archon/adapters`, `@archon/core`                                       |
| `@archon/cli`       | Command-line interface — workflow run, isolation management                 | `workflowRunCommand`, `isolationCleanupCommand`, `CLIAdapter`                                                                                                | `@archon/core`, `@archon/workflows`, `@archon/isolation`, `@archon/git`  |
| `@archon/web`       | React + Vite frontend — chat, workflow builder, dashboard                   | React components, Zustand stores, `api.generated.d.ts` types                                                                                                 | HTTP API (generated OpenAPI spec)                                        |
| `@archon/docs-web`  | Astro-based documentation site                                              | Static docs                                                                                                                                                  | _(standalone)_                                                           |

---

## Key Abstractions

### 1. `IPlatformAdapter` — `packages/core/src/types/index.ts:117`

The universal interface every platform adapter implements. The orchestrator never talks to a platform directly.

**Methods:** `sendMessage`, `ensureThread`, `getStreamingMode`, `getPlatformType`, `start`, `stop`, and optional `sendStructuredEvent?`, `emitRetract?`.

**Implementors:** `WebAdapter` (`packages/server/src/adapters/web.ts`), `TelegramAdapter`, `SlackAdapter`, `DiscordAdapter` (all in `@archon/adapters`), `GitHubAdapter`, `GiteaAdapter`, `GitLabAdapter` (forge adapters in `@archon/adapters`), `CLIAdapter` (`packages/cli/src/adapters/`).

**Used by:** `handleMessage` in `packages/core/src/orchestrator/orchestrator-agent.ts`, `executeWorkflow` in `packages/workflows/src/executor.ts` (via the narrower `IWorkflowPlatform`).

---

### 2. `IAssistantClient` — `packages/core/src/types/index.ts:364`

Single method interface for AI assistant clients. `sendQuery` returns an `AsyncGenerator<MessageChunk>` so callers can stream tokens.

```typescript
interface IAssistantClient {
  sendQuery(prompt, cwd, resumeSessionId?, options?): AsyncGenerator<MessageChunk>;
  getType(): string;
}
```

**Implementors:** `ClaudeClient` (`packages/core/src/clients/claude.ts`) wraps `@anthropic-ai/claude-agent-sdk`; `CodexClient` (`packages/core/src/clients/codex.ts`) wraps `@openai/codex-sdk`.

**Used by:** `getAssistantClient` factory in `packages/core/src/clients/factory.ts`; the workflow engine accesses clients only via the injected `AssistantClientFactory`.

---

### 3. `IDatabase` — `packages/core/src/db/adapters/types.ts:16`

Minimal DB abstraction that two concrete backends satisfy: `postgres` (pg pool) and `sqlite` (Bun SQLite). All SQL uses `$1, $2` placeholders on both backends.

**Key methods:** `query<T>`, `withTransaction`, `close`, `dialect: 'postgres' | 'sqlite'`, `sql: SqlDialect`.

`SqlDialect` (`packages/core/src/db/adapters/types.ts:56`) provides helper expressions for UUID generation, `NOW()`, JSON merge, and interval arithmetic — the only place dialect-specific SQL lives.

**Used by:** every module in `packages/core/src/db/` (conversations, sessions, codebases, workflows, messages, etc.).

---

### 4. `IWorkflowStore` — `packages/workflows/src/store.ts:33`

Narrow DB interface the workflow engine depends on — completely decoupled from the real DB. The full contract covers run lifecycle (`createWorkflowRun`, `failWorkflowRun`, `completeWorkflowRun`, `pauseWorkflowRun`, `cancelWorkflowRun`, `resumeWorkflowRun`), event appending (`createWorkflowEvent`), and DAG resume support (`getCompletedDagNodeOutputs`).

**Implementor:** `createWorkflowStore()` in `packages/core/src/workflows/store-adapter.ts:28` — a thin adapter that maps the interface onto the real `workflowDb.*` and `workflowEventDb.*` functions.

---

### 5. `WorkflowDeps` — `packages/workflows/src/deps.ts:273`

Single injection point for everything the workflow engine needs from outside. Contains three fields:

```typescript
interface WorkflowDeps {
  store: IWorkflowStore;
  getAssistantClient: AssistantClientFactory; // (provider) => IWorkflowAssistantClient
  loadConfig: (cwd: string) => Promise<WorkflowConfig>;
}
```

`@archon/workflows` has **zero** `@archon/core` dependency; all real-world effects flow through this injection. `createWorkflowDeps()` in `packages/core/src/workflows/store-adapter.ts` builds the concrete instance used by the orchestrator and CLI.

---

### 6. `IIsolationProvider` / `IIsolationStore` — `packages/isolation/src/types.ts`, `packages/isolation/src/store.ts`

`IIsolationProvider` defines `create(request)` and `destroy(env, options)` for managing isolated environments. The only implementation is `WorktreeProvider` (`packages/isolation/src/providers/worktree.ts`).

`IIsolationStore` is the narrow DB interface for persisting environment records (create, mark destroyed, list active, etc.).

**Used by:** `IsolationResolver` (`packages/isolation/src/resolver.ts:67`) which is instantiated in `packages/core/src/orchestrator/orchestrator.ts` with both injected.

---

### 7. `IsolationResolver` — `packages/isolation/src/resolver.ts:67`

Class that encodes the 7-step resolution order (existing env → no codebase → workflow reuse → linked issue sharing → PR branch adoption → limit check/auto-cleanup → create new). Returns a rich discriminated union `IsolationResolution` without any platform messaging — the orchestrator handles all user-facing messages.

---

### 8. `DagNode` (discriminated union) — `packages/workflows/src/schemas/dag-node.ts`

The union type for all DAG node variants: `CommandNode` (named file), `PromptNode` (inline prompt), `BashNode` (shell script, stdout captured), `LoopNode` (iterative AI), `ApprovalNode` (human gate), `CancelNode`. Each variant is parsed by `dagNodeSchema.safeParse()` in `loader.ts`.

**Type guards:** `isBashNode`, `isLoopNode`, `isApprovalNode`, `isCancelNode` (all in `dag-node.ts`, re-exported from `schemas/index.ts`).

---

### 9. `MessageChunk` — `packages/core/src/types/index.ts:196`

Discriminated union of all streaming events from an AI assistant: `assistant` (text), `system`, `thinking`, `tool`, `tool_result`, `result` (final + usage), `rate_limit`, `workflow_dispatch`. Flows from `IAssistantClient.sendQuery()` through the orchestrator to `IPlatformAdapter.sendStructuredEvent()` for the Web UI, or formatted text to `sendMessage()` for all other platforms.

---

### 10. `ConversationLockManager` — `packages/core/src/utils/conversation-lock.ts`

Per-conversation mutex that queues concurrent messages and enforces `MAX_CONCURRENT_CONVERSATIONS`. Returns `{ status: 'started' | 'queued-conversation' | 'queued-capacity' }` — callers use the return value to decide whether to emit a "queued" notice (TOCTOU-safe).

---

## Data Flow

### Use Case 1: Web UI message → workflow execution

```
Browser POST /api/conversations/:id/message
  → packages/server/src/routes/api.ts (route handler)
  → ConversationLockManager.acquireLock(conversationId)           [core/utils/conversation-lock.ts]
  → handleMessage(platform, conversationId, message, context)     [core/orchestrator/orchestrator-agent.ts:383]
      → db.getConversation(conversationId)                        [core/db/conversations.ts]
      → commandHandler.parseCommand(message)                      [core/handlers/command-handler.ts]
          → if slash command (one of 10 deterministic):
              handleCommand() → DB ops, returns CommandResult
          → if CommandResult.workflow set:
              dispatchBackgroundWorkflow(workflow, ...)           [core/orchestrator/orchestrator.ts:256]
                → create hidden worker conversation (DB)
                → webAdapter.setupEventBridge(worker, parent)
                → executeWorkflow(deps, platform, ...) [fire-and-forget]
          → else (free text or non-deterministic intent):
              discoverWorkflowsWithConfig(cwd)                   [workflows/workflow-discovery.ts]
              buildOrchestratorPrompt(workflows, codebases)      [core/orchestrator/prompt-builder.ts]
              getAssistantClient('claude').sendQuery(routerPrompt) [core/clients/claude.ts]
                → @anthropic-ai/claude-agent-sdk query()
                → yields MessageChunk events
              parseOrchestratorCommands(aiResponse)              [core/orchestrator/orchestrator-agent.ts:90]
                → if /invoke-workflow found:
                    findWorkflow(name, workflows)                 [workflows/router.ts]
                    dispatchOrchestratorWorkflow(...)             [core/orchestrator/orchestrator-agent.ts]
                      → executeWorkflow(deps, platform, ...)
                → else: send AI text to platform

executeWorkflow(deps, platform, conversationId, cwd, workflow)   [workflows/executor.ts]
  → deps.store.createWorkflowRun(...)                            [core/db/workflows.ts]
  → getWorkflowEventEmitter().registerRun(runId, conversationId) [workflows/event-emitter.ts]
  → executeDagWorkflow(deps, platform, run, nodes, ...)          [workflows/dag-executor.ts]
      → topological sort of nodes (respects depends_on)
      → for each layer: Promise.allSettled(layer.map(executeNode))
          → evaluateCondition(node.when, nodeOutputs)            [workflows/condition-evaluator.ts]
          → for BashNode: execFileAsync(shell, [script])         [git/exec.ts]
          → for PromptNode/CommandNode:
              deps.getAssistantClient(provider).sendQuery(prompt, cwd, resumeSessionId, options)
              → yields WorkflowMessageChunk events
              → platform.sendStructuredEvent(conversationId, chunk) [web] or
                platform.sendMessage(conversationId, formatted)  [other platforms]
          → deps.store.createWorkflowEvent({ event_type: 'node_completed', ... })
          → nodeOutputs.set(node.id, output)   // enables $nodeId.output substitution
  → deps.store.completeWorkflowRun(runId)

SSE stream (parallel, event-driven):
  getWorkflowEventEmitter().subscribeForConversation(conversationId, handler)
    → handler writes SSE events to WebAdapter → SSETransport → browser
```

**Key files involved:**

- `packages/server/src/routes/api.ts` — HTTP entry
- `packages/core/src/orchestrator/orchestrator-agent.ts:383` — `handleMessage`
- `packages/workflows/src/executor.ts` — `executeWorkflow`
- `packages/workflows/src/dag-executor.ts` — `executeDagWorkflow`
- `packages/server/src/adapters/web.ts` — `WebAdapter.sendStructuredEvent`
- `packages/server/src/adapters/web/transport.ts` — `SSETransport`

---

### Use Case 2: CLI workflow run

```
bun run cli workflow run implement "Add auth"
  → packages/cli/src/cli.ts (parseArgs)
  → workflowRunCommand(name, message, options)                    [cli/commands/workflow.ts]
      → findRepoRoot(cwd)                                         [git/repo.ts]
      → discoverWorkflowsWithConfig(cwd)                         [workflows/workflow-discovery.ts]
      → findWorkflow(name, workflows)                             [workflows/router.ts]
      → IsolationResolver.resolve(request)                        [isolation/resolver.ts]
          → WorktreeProvider.create(request)                      [isolation/providers/worktree.ts]
              → syncWorkspace(repoPath)                           [git/repo.ts] — fetch + pull
              → git worktree add <path> -b <branch>              [git/exec.ts execFileAsync]
          → isolationStore.create(environment)                    [core/db/isolation-environments.ts via CLI adapter]
      → CLIAdapter.sendMessage(...)                               [cli/adapters/cli-adapter.ts]
      → createWorkflowDeps()                                      [core/workflows/store-adapter.ts]
      → executeWorkflow(deps, cliAdapter, conversationId, worktreePath, workflow, message)
          → [same as Use Case 1 from executeDagWorkflow onward]
      → stdout streaming output
```

**Key files:**

- `packages/cli/src/cli.ts` — argument parsing
- `packages/cli/src/commands/workflow.ts` — `workflowRunCommand`
- `packages/isolation/src/resolver.ts` — `IsolationResolver`
- `packages/isolation/src/providers/worktree.ts` — `WorktreeProvider`

---

### Use Case 3: GitHub webhook → issue comment execution

```
GitHub POST /webhooks/github (HMAC-signed)
  → packages/server/src/routes/api.ts (webhook route)
      → GitHubAdapter.handleWebhook(payload, signature)           [adapters/forge/github/index.ts]
          → timingSafeEqual HMAC verification
          → event type check: only `issue_comment.created`
          → authorization check against GITHUB_ALLOWED_USERS whitelist
          → ConversationLockManager.acquireLock(owner/repo#number)
          → cloneRepository or resolveLocalPath
          → loadCommandsFromDisk
          → buildIssueContext(issue, comments)
          → handleMessage(gitHubAdapter, "owner/repo#42", "@archon fix this", {
               issueContext, isolationHints: { workflowType: 'issue', identifier: '42' }
             })                                                    [core/orchestrator/orchestrator-agent.ts]
              → validateAndResolveIsolation(hints)                [core/orchestrator/orchestrator.ts:108]
                  → IsolationResolver.resolve(request)
                  → updates conversation.isolation_env_id in DB
              → [same routing + workflow dispatch as Use Case 1]
          → gitHubAdapter.postComment(owner, repo, issue, response)
```

**Key files:**

- `packages/adapters/src/forge/github/index.ts` — `GitHubAdapter.handleWebhook`
- `packages/core/src/orchestrator/orchestrator.ts:108` — `validateAndResolveIsolation`

---

## Dependency Graph

```
                         ┌─────────────────────┐
                         │   @archon/paths      │
                         │  (zero @archon/* deps)│
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │    @archon/git       │
                         │  (only @archon/paths) │
                         └──────┬───────────────┘
                                │
               ┌────────────────┼──────────────────┐
               │                │                  │
    ┌──────────▼───────┐  ┌─────▼──────────┐       │
    │ @archon/isolation │  │@archon/workflows│       │
    │(@archon/git+paths)│  │(@archon/git+paths│      │
    └──────────┬────────┘  └────────┬───────┘       │
               │                   │                │
               └─────────┬─────────┘                │
                          │                          │
               ┌──────────▼───────────────────────┐ │
               │         @archon/core              │ │
               │  (git + isolation + workflows     │◄┘
               │   + paths + claude-agent-sdk)     │
               └──────┬───────────────────────────┘
                      │
         ┌────────────┼─────────────────┐
         │            │                 │
┌────────▼──────┐ ┌───▼───────────┐ ┌──▼─────────────┐
│@archon/adapters│ │ @archon/server │ │  @archon/cli   │
│ (core only)   │ │(adapters+core) │ │(core+workflows+│
└───────────────┘ └───────┬───────┘ │isolation+git)  │
                          │         └────────────────┘
               ┌──────────▼──────────┐
               │    @archon/web      │
               │  (HTTP API only,    │
               │  no @archon/* deps) │
               └─────────────────────┘
```

**No circular dependencies detected.** The package layering is strictly enforced: `@archon/workflows` explicitly avoids importing `@archon/core` (documented in `packages/workflows/src/deps.ts:19`), which would create a cycle since `@archon/core` imports from `@archon/workflows`.

---

## Notable Architectural Decisions

### 1. Dependency Inversion via `WorkflowDeps` prevents circular packages

`@archon/workflows` needs DB access, AI clients, and config — all of which live in `@archon/core`. Rather than creating a cycle, the engine defines narrow interfaces (`IWorkflowStore`, `IWorkflowAssistantClient`, `WorkflowConfig`) and receives concrete implementations through the `WorkflowDeps` injection point at call time.

Evidence: `packages/workflows/src/deps.ts:19` — explicit comment "copied to avoid circular dependency"; `packages/core/src/workflows/store-adapter.ts:28` — `createWorkflowStore()` is the glue layer.

`WorkflowAssistantOptions` in `deps.ts` is a structural duplicate of `AssistantRequestOptions` in `core/types/index.ts` for the same reason. A compile-time assertion at `store-adapter.ts:19` (`const assertConfigCompat: WorkflowConfig = {} as MergedConfig`) catches drift without runtime cost.

---

### 2. Narrow interface segregation on every package boundary

`IWorkflowPlatform` (`deps.ts:203`) is a 4-method subset of `IPlatformAdapter` (which has 8 methods including `start`, `stop`, `ensureThread`). The workflow engine doesn't manage platform lifecycle. Similarly, `IWorkflowAssistantClient` is a subset of `IAssistantClient`. This keeps the engine testable without full platform/AI mocks.

Evidence: `packages/workflows/src/deps.ts:196` — "Intentionally excludes ensureThread(), start(), and stop()".

---

### 3. DAG nodes in the same topological layer execute concurrently

`executeDagWorkflow` in `packages/workflows/src/dag-executor.ts` topologically sorts the DAG, groups nodes by layer, and runs each layer with `Promise.allSettled`. This means N independent nodes (e.g., 5 parallel PR review agents in `archon-comprehensive-pr-review`) run simultaneously without coordination overhead beyond the shared `nodeOutputs` map. `trigger_rule` (`all_success`, `one_success`, `all_done`, etc.) on downstream nodes controls join semantics.

Evidence: `packages/workflows/src/dag-executor.ts:1` comment and `Promise.allSettled` calls within `executeDagWorkflow`.

---

### 4. Isolation is centralized in the orchestrator, not in adapters

All worktree creation, reuse, and cleanup decisions happen in `validateAndResolveIsolation()` (`packages/core/src/orchestrator/orchestrator.ts:108`). Platform adapters only supply `IsolationHints` (preferred workflow type, identifier). This means isolation behavior is consistent across all platforms (Web, CLI, GitHub, Telegram) without per-adapter worktree logic.

Evidence: `packages/core/src/orchestrator/orchestrator.ts:108`; `.claude/rules/isolation-patterns.md` — "ALL isolation logic is centralized in the orchestrator — adapters are thin".

---

### 5. The routing AI call uses `tools: []` to prevent tool use

When the orchestrator needs to select a workflow, it calls Claude with an empty tools list (`tools: []`) so Claude cannot invoke code execution tools — it can only produce text (which may contain the `/invoke-workflow` protocol string). This prevents runaway tool calls during routing. If routing is ambiguous, `findWorkflow` falls back to `archon-assist`; if that's also unavailable, the raw AI text is sent to the user.

Evidence: `.claude/rules/workflows.md` — "Claude routing calls use `tools: []` to prevent tool use at the API level".

---

### 6. SSE event bridge for background workflows (Web only)

Background workflow dispatch (`dispatchBackgroundWorkflow`, `packages/core/src/orchestrator/orchestrator.ts:256`) creates a hidden "worker" conversation (prefixed `web-worker-`). `WebAdapter.setupEventBridge()` wires the worker's SSE events to the parent conversation's SSE stream, so the user sees real-time progress in the original chat thread while the workflow runs independently in the background.

Evidence: `packages/server/src/adapters/web.ts` — `setupEventBridge`; `packages/server/src/adapters/web/workflow-bridge.ts` — `WorkflowEventBridge`.

---

### 7. Immutable sessions with `parent_session_id` audit chain

Sessions are never mutated. When a transition occurs (e.g., `plan-to-execute`, `reset-requested`), the current session is deactivated (`active = false`, `ended_reason` set) and a new session is created with `parent_session_id` pointing to the previous one. The transition type is stored in `transition_reason`. This creates a full audit trail navigable through the linked chain.

Evidence: `packages/core/src/types/index.ts:74–88` — `Session` interface; `packages/core/src/state/session-transitions.ts` — `TransitionTrigger` type; `.claude/rules/database.md` — "Session Audit Trail".

---

### 8. Worktree-aware auto port allocation (deterministic hash)

When the server starts inside a git worktree, `getPort()` (`packages/core/src/utils/port-allocation.ts`) computes a hash of the worktree path and maps it to the 3190–4089 range. The same worktree always gets the same port, enabling predictable `curl` testing. The main repo defaults to 3090. This allows multiple simultaneous Archon instances (each worktree has its own server) sharing the same SQLite database.

Evidence: `.claude/rules/dx-quirks.md`; `packages/core/src/utils/port-allocation.ts`.

---

### 9. Env leak gate with audit log

Before any Claude subprocess runs in a codebase, `scanPathForSensitiveKeys()` (`packages/core/src/utils/env-leak-scanner.ts`) scans `.env` files for API keys. If secrets are found, the run is blocked with `EnvLeakError` unless the codebase has `allow_env_keys: true` (set via `PATCH /api/codebases/:id`). Every grant/revoke is audit-logged at `warn` level with `actor`, `codebaseId`, and the list of leaked keys.

Evidence: `packages/core/src/utils/env-leak-scanner.ts`; `packages/core/src/clients/claude.ts:39` — `scanPathForSensitiveKeys` call.

---

### 10. Resilient YAML workflow discovery

`discoverWorkflowsWithConfig()` (`packages/workflows/src/workflow-discovery.ts`) returns `{ workflows, errors }` — one broken YAML does not abort discovery for the rest. Callers surface errors via `/workflow list` without crashing. Bundled defaults (embedded at binary compile time via `scripts/build-binaries.sh`) are merged with repo-level files; repo files override by filename.

Evidence: `packages/workflows/src/schemas/workflow.ts` — `WorkflowLoadResult` type with `errors` field; `.claude/rules/workflows.md` — "Resilient discovery".
