# OpenCode 2.x Migration Guide

This guide covers migrating agents and clusters to OpenCode 2.x in KubeOpenCode v0.2.0+, as well as details on dual-version backward compatibility with OpenCode 1.x.

---

## Overview

Starting with **v0.2.0**, KubeOpenCode defaults to **OpenCode 2.x** (v2.3.2+) for agent images and client communication.

### Dual-Version Support Guarantee

You do **not** need to migrate all your agents simultaneously. KubeOpenCode v0.2.0+ provides **concurrent dual-version compatibility**:
- OpenCode 1.x Agents and OpenCode 2.x Agents can run side-by-side in the same Kubernetes cluster.
- Existing Agent and AgentTemplate manifests pointing to 1.x images (such as `:v0.1.9`) continue to run without changes.
- The KubeOpenCode controller, task submitter, and web server automatically detect and negotiate both protocols.

---

## What Changed in OpenCode 2.x

| Feature / Area | OpenCode 1.x | OpenCode 2.x | How KubeOpenCode Handles It |
|---|---|---|---|
| **Attach Flag** | `opencode run --attach <url>` | `opencode run --server <url>` | `pod_builder` dynamically checks flag support at runtime and uses `--server` or `--attach` |
| **API Envelope** | Bare objects/arrays (`/session`) | Envelope format `{ "data": ... }` (`/api/session`) | `opencode_client` unwraps `{ "data": ... }` or reads bare responses transparently |
| **Session Endpoints** | `/session/:id`, `/session/:id/prompt_async` | `/api/session/:id`, `/api/session/:id/prompt` | Controller & proxy query 2.x endpoints first and fall back to 1.x endpoints on 404 |
| **Metrics & Cost** | Calculated by aggregating assistant messages | Included directly on session object (`session.Tokens`, `session.Cost`) | Controller reads direct fields when present, falling back to message aggregation |
| **Health Check** | `/session/status` (HTTP 200) | `/session/status` (HTTP 200) | Controller probes `/session/status` (unauthenticated on both versions) |
| **CPU Instruction Set** | Standard x86_64 / arm64 | AVX2 required by default on x64 | KubeOpenCode provides an optional `OPENCODE_VARIANT=baseline` build argument |

---

## Migrating an Agent to OpenCode 2.x

### Step 1: Update the Agent Image

In your `Agent` or `AgentTemplate` manifest, update `agentImage` to reference `v0.2.0` (or later):

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Agent
metadata:
  name: my-agent
  namespace: default
spec:
  # Update to v0.2.0+ for OpenCode 2.x
  agentImage: ghcr.io/kubeopencode/kubeopencode-agent-opencode:v0.2.0
  executorImage: ghcr.io/kubeopencode/kubeopencode-agent-devbox:v0.2.0
  workspaceDir: /workspace
```

Apply the updated manifest:

```bash
kubectl apply -f my-agent.yaml
```

The controller will roll out the new Deployment.

---

### Step 2: CPU Architecture & AVX2 Compatibility

OpenCode 2.x binary builds for `amd64` (x86_64) use **AVX2 vector instructions** by default for maximum performance.

#### When to use the Baseline Variant
If you are deploying on:
- Older hardware (pre-Haswell / pre-Excavator, older than ~2013)
- Virtualized cloud instances or hypervisors where AVX2 CPU flags are masked out
- Kind / Docker Desktop on certain older host CPUs

OpenCode 2.x may terminate with an `Illegal instruction` (`SIGILL` exit code 132) error.

#### Building the Baseline Image
KubeOpenCode includes a build argument for the baseline variant:

```bash
make -C agents AGENT=opencode OPENCODE_VARIANT=baseline build \
  IMG=ghcr.io/my-org/kubeopencode-agent-opencode:v0.2.0-baseline
```

Then configure your Agent to use this baseline image.

---

### Step 3: Session Database Migration

If your Agent uses session persistence (`spec.persistence.session.enabled: true`):
- OpenCode 2.x uses SQLite for session storage, just like OpenCode 1.x.
- On startup, the OpenCode 2.x server inspects the existing database in the persistent volume (`/data/sessions/opencode.db`) and automatically migrates schemas.
- **No manual database migration is required.**

> [!NOTE]
> Database migration is forward-only. If you rollback an agent from 2.x to 1.x while preserving the PVC, OpenCode 1.x may not recognize the newer database schema. If rolling back, backup your session PVC first.

---

## Keeping Existing OpenCode 1.x Agents

If you wish to keep specific Agents running OpenCode 1.x after upgrading the KubeOpenCode cluster to v0.2.0+:

1. **Pin the image tag**: Explicitly set `agentImage` to a 1.x version:
   ```yaml
   spec:
     agentImage: ghcr.io/kubeopencode/kubeopencode-agent-opencode:v0.1.9
   ```
2. **Custom 1.x Builds**: You can continue to build custom OpenCode 1.x images using the `agents/Makefile`:
   ```bash
   make -C agents AGENT=opencode OPENCODE_VERSION=1.17.11 build \
     IMG=ghcr.io/my-org/kubeopencode-agent-opencode:v1.17.11
   ```

KubeOpenCode will automatically interact with this agent using the legacy 1.x endpoints and flags.

---

## Verifying Migration

After updating an Agent to OpenCode 2.x:

1. **Check Agent Status**:
   ```bash
   kubectl get agent my-agent
   ```
   Ensure `READY` is `True` and `URL` is populated.

2. **Verify Server Logs**:
   ```bash
   kubectl logs deployment/my-agent-server -c opencode-server
   ```
   You should see:
   ```text
   server listening on http://0.0.0.0:4096
   ```

3. **Run a Test Task**:
   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: kubeopencode.io/v1alpha1
   kind: Task
   metadata:
     name: test-migration
   spec:
     agentRef:
       name: my-agent
     description: "Echo test"
   EOF
   ```
   Wait for completion and verify task status:
   ```bash
   kubectl get task test-migration
   ```
