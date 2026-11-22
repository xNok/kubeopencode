# OpenCode 2.x SDK & Architecture Leverage Analysis

> **Date**: 2026-11-22  
> **Status**: Research / Future Improvement  
> **Target Subsystems**: Task Submitter, Agent Reconciler, Plugin System, Web UI

---

## 1. Overview & Context

With the release of OpenCode 2.x, the upstream project introduced `@opencode/sdk`, `@opencode/client`, and a redesigned `@opencode/plugin` subsystem. This document explores the technical architecture of these packages and evaluates how they can provide leverage for future enhancements in KubeOpenCode.

---

## 2. Technical Evaluation: In-Process Host vs. Network Daemon

### 2.1 What `@opencode/sdk` Is
`@opencode/sdk` is an in-process, in-memory host for TypeScript / JavaScript environments (Node.js, Bun, and Cloudflare Durable Objects via `@opencode/sdk/workerd`):
- It hosts the OpenCode server in-process without binding an HTTP port or TCP listener.
- It is designed for applications where the host process directly orchestrates AI sessions via memory routes.

### 2.2 Why the SDK Cannot Be Directly Imported in KubeOpenCode Core
1. **Language Barrier**: KubeOpenCode is built in Go 1.25 as a single static binary (`cmd/kubeopencode`). Importing `@opencode/sdk` into the Go controller or CLI would require bundling Node/Bun or a V8 runtime inside the container, violating the lightweight, zero-external-dependency architecture.
2. **Kubernetes Multi-Consumer Architecture**: KubeOpenCode's foundational design pattern is **`Agent` = running Kubernetes `Deployment` + `Service` running `opencode serve`** (see ADR 0003 & ADR 0028). Multiple Task Pods (`kubeopencode task-submit`), the Web UI terminal proxy, and CLI attach sessions (`kubeoc agent attach`) must connect concurrently to the running agent over the network. An in-memory SDK without an HTTP listener cannot serve remote Kubernetes clients.

---

## 3. High-Leverage Future Opportunities

While the TypeScript SDK itself cannot be used in Go, the **underlying OpenCode 2.x capabilities it exposes** provide high leverage for four major architectural improvements in KubeOpenCode:

### 3.1 Leverage 1: Native Git Worktree Isolation for Concurrent Tasks (High Value)

- **The Problem in KubeOpenCode**:
  Currently, persistent Agents support `spec.maxConcurrentTasks`. However, when multiple tasks run simultaneously against the same Agent, they execute in the same workspace directory (`workspaceDir`). This can cause Git index locks, unstaged file conflicts, and branch race conditions.

- **The OpenCode 2.x Worktree API**:
  OpenCode 2.x introduces native, first-class project worktree management (`POST /api/worktree` / `ctx.worktree`):
  ```ts
  const worktree = await opencode.worktree.create({ projectID, name: "task-abc" })
  await opencode.worktree.refresh({ projectID })
  await opencode.worktree.remove({ projectID, directory: worktree.directory, force: false })
  ```
  Sessions can be scoped directly to a specific worktree directory via `location.directory`.

- **Proposed Future Improvement**:
  When `kubeopencode task-submit` runs an attach-mode Task:
  1. The submitter calls `/api/worktree` to provision a dedicated worktree under `${WORKSPACE_DIR}/.worktrees/task-${TASK_NAME}`.
  2. The session is created pointing to that worktree directory.
  3. Upon task completion, the submitter prunes or merges the worktree.
  4. **Impact**: Unlocks true, safe multi-task concurrency on a single shared persistent Agent.

---

### 3.2 Leverage 2: KubeOpenCode Companion Plugin (`@opencode/plugin`)

OpenCode 2.x features a rebuilt server-side plugin system with deep lifecycle hooks. KubeOpenCode already provisions plugins via `Agent.spec.plugins` (init container installing packages into `/plugins`).

Building a first-party `@kubeopencode/agent-plugin` companion can provide:

1. **Prompt Admission & Security Governance**:
   - `ctx.session.hook("prompt", ...)` intercepts user prompts before execution.
   - Allows automatic secret redaction, regulatory audit logging, and injection of cluster-defined system policies and context files.
2. **Model Quota & Budget Enforcement**:
   - `ctx.model.transform` and `ctx.session.hook("context", ...)` can inspect token consumption and cost budgets before model requests are dispatched.
   - Enforces `Agent.spec.quota` hard stops proactively inside the agent rather than reactively via polling.
3. **Cluster Integration RPCs**:
   - `ctx.rpc.register(...)` allows defining custom RPC endpoints on the agent.
   - Allows the KubeOpenCode controller and agent to exchange structured heartbeats, task progress checkpoints, and container telemetry.

---

### 3.3 Leverage 3: Web UI (`ui/`) Modernization via `@opencode/client`

- KubeOpenCode's web interface is a React/Vite/TypeScript application.
- Rather than maintaining custom Fetch/WebSocket abstractions for OpenCode, the UI can adopt `@opencode/client` to talk to OpenCode endpoints through the server reverse-proxy:
  - Strongly typed session operations (`client.session.create`, `client.session.prompt`).
  - Native typed SSE event streaming (`for await (const event of client.event.subscribe())`).
  - Built-in session diff inspection (`client.session.diff()`).

---

### 3.4 Leverage 4: Contract-Driven Go Client Generation

- OpenCode 2.x maintains its protocol schemas in `@opencode/protocol` (JSON Schema / OpenAPI).
- Rather than manually maintaining and updating Go HTTP structs and routes when OpenCode releases new minor versions, KubeOpenCode can generate or lint Go client contracts against upstream JSON Schemas.

---

## 4. Prioritization & Roadmap

| Feature | Complexity | Impact | Recommended Phase |
|---|---|---|---|
| **Worktree Task Isolation** | Medium | **High** (solves concurrent task conflict) | v0.2.x |
| **Contract-Driven Go Client** | Low | Medium (maintenance reduction) | v0.2.x |
| **UI Event Streaming via `@opencode/client`** | Low | Medium (UI responsiveness) | v0.3.x |
| **KubeOpenCode Companion Plugin** | High | **High** (enterprise security & quota) | v0.3.x |
