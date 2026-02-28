---
name: tentacular
description: Build, test, and deploy TypeScript workflow DAGs on Kubernetes using the tntc CLI. Use when creating new agentic workflows, deploying or managing existing workflows, writing or debugging workflow nodes, running tests, checking workflow status, viewing logs, or working with Tentacular secrets and contracts. Covers the full lifecycle: scaffold → validate → test → build → deploy → run → monitor.
---

# Tentacular

## Prerequisites

Before using any `tntc` commands, verify the CLI is installed:

```bash
which tntc
```

If that fails, install it now:

```bash
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
```

After install, confirm it works:

```bash
tntc version
```

If `tntc version` reports `dev (commit none, built unknown)`, the binary was built from source (no release tag yet) — this is expected until the first tagged release. All commands are fully functional.

> **Minimum required version:** Check `tntc version` output. If the skill specifies a minimum version and the installed binary reports an older tag, re-run the install script with `TNTC_VERSION=<tag>`.

### MCP Server Bootstrap

Before deploying workflows, the MCP server must be
installed in the target cluster. This is a one-time
bootstrap step:

```bash
tntc cluster install
```

This deploys the tentacular-mcp server and module proxy
into the `tentacular-system` namespace, generates a
bearer token, and saves the MCP endpoint and token to
`~/.tentacular/config.yaml`. All subsequent `tntc`
commands automatically route through MCP.

`tntc cluster install` is the **only** command that
communicates directly with the Kubernetes API. All other
cluster operations go through the MCP server.

## Architecture

Tentacular is a *secure* workflow build and execution
system designed for AI agents. Its purpose is to allow
an AI agent (you) to easily build repeatable and durable
workflows (pipelines). Arbitrary workflows can be built
dynamically by an agent, reused or replaced as necessary,
and deployed onto Kubernetes clusters to be triggered
manually or by various hooks (cron, webhook, etc).

Every tentacular agentic workflow is deployed in a tightly
constrained sandbox, without a tool chain or other
ancillary systems. Only what is needed is deployed into a
production runtime ([gVisor](https://gvisor.dev/)). Each
workflow's networking and other access requirements are
declared in the workflow contract, which drives automatic
generation of Kubernetes NetworkPolicy to lockdown the
workflow's production network access.

It is composed of three key components:

- **Go CLI** (`cmd/tntc/`, `pkg/`) -- the data plane.
  Manages the workflow lifecycle: scaffold, validate,
  dev, test, build, deploy. After bootstrap, all cluster
  operations route through the MCP server.
- **MCP Server** (`tentacular-mcp`) -- the control plane.
  An in-cluster MCP server that proxies all K8s
  operations through scoped RBAC. The CLI communicates
  with the cluster exclusively through MCP after the
  initial `tntc cluster install` bootstrap.
- **Deno/TypeScript Engine** (`engine/`) -- executes
  workflows as DAGs. Compiles workflow.yaml into
  topologically sorted stages, loads TypeScript node
  modules, runs them with a Context providing dependency
  resolution, fetch, logging, config, and secrets.
  Exposes HTTP triggers (`POST /run`, `GET /health`).

The key benefits of Tentacular are:

- Hardened execution environment for production workloads
  - Attack surface is absolute minimum as there is no
    non-essential code
  - Runs on [gVisor](https://gvisor.dev/) by default
- Agent-friendly
  - Build workflows using plain english,
    but in a repeatable and manageable manner
  - Full tentacular SKILL to help manage the workflow
    end-to-end
  - Testing is baked into the workflow development process
    prior to pushing to production

## CLI vs MCP: When to Use Which

Both the CLI and MCP tools require `tntc cluster install`
to have been run first (except for purely local commands
like `validate` and `test`).

**Use `tntc` CLI** for the full lifecycle and
terminal/CI operations:
- Scaffold, validate, test, build, deploy
- Local development (`tntc dev`)
- CI/CD pipelines
- Any operation that reads/writes local files

**Use MCP tools directly** from an AI agent session for
discovery, execution, and observability without shelling
out:
- List and inspect deployed workflows
- Trigger workflow runs
- View logs, pods, events, jobs
- Check cluster health and security posture
- Manage namespaces and credentials

**Decision tree:**
- Building or developing a workflow? **CLI**
- Querying or operating on the cluster? **MCP tools**
- Bootstrap or initial setup? **CLI**
- Agent session needing cluster info? **MCP tools**

---

## Tentacles: Production Workflow Best Practice

Production workflows (called **tentacles**) MUST be stored
outside the tentacular repository. The repo contains only
the tool (CLI + engine), and example workflows.
Real workflows with secrets, credentials, and
production-specific configuration belong in a separate
local directory.

**Canonical location: `~/workspace/tentacles/`**

```
~/workspace/tentacles/
  ai-news-roundup/
    workflow.yaml
    .secrets.yaml        # NEVER committed to any repo
    .secrets.yaml.example
    nodes/
    tests/
  uptime-prober/
    ...
```

**Why separate?**

- The tentacular repo is public — secrets and production
  configs must never end up there
- Workflows are operational artifacts, not source code
  for the tool itself
- `example-workflows/` in the repo are reference
  implementations only — copy them to your tentacles
  directory to customize and deploy

**When building a new workflow**, always create it in the
tentacles directory, not in the repo. Use
`example-workflows/` as templates if helpful, but the
working copy lives in tentacles.

> ⚠️ **NEVER deploy from `example-workflows/`** — those
> are read-only reference implementations. If a workflow
> was accidentally deployed from there, move the source to
> `~/workspace/tentacles/<name>/`, verify `.secrets.yaml`
> is present there, and redeploy from the new location.

## Querying Running Workflows

All cluster operations route through the MCP server.
Once the MCP server is installed (`tntc cluster install`),
commands like `list`, `status`, `logs`, `run`, and
`undeploy` work without `KUBECONFIG` or direct K8s access:

```bash
tntc list -n <namespace>
tntc status <workflow-name> -n <namespace>
tntc logs <workflow-name> -n <namespace>
tntc run <workflow-name> -n <namespace>
```

The MCP endpoint and token are auto-configured by
`tntc cluster install` and saved to
`~/.tentacular/config.yaml`. No per-command MCP URL
flags are needed.

> **Note:** `tntc logs --follow` is not supported through
> MCP (snapshot only). Use `kubectl logs -f` for streaming.

## MCP Tools Reference

The tentacular MCP server exposes 29 tools organized into
11 groups. These tools are available directly from an AI
agent session -- no `tntc` CLI or `KUBECONFIG` needed.

Agents can discover all tools and their full parameter
schemas via the MCP `tools/list` method. The sections
below provide detailed parameter tables for every tool.

### Workflow Discovery

#### wf_list

Lists all tentacular-managed workflow deployments. Filters
by label selector `app.kubernetes.io/managed-by=tentacular`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | No | Namespace to filter. Empty = all tentacular namespaces. |
| `owner` | string | No | Filter by `tentacular.dev/owner` annotation. |
| `tag` | string | No | Filter by tag in `tentacular.dev/tags` annotation. |

Returns an array of workflow entries, each with:
`name`, `namespace`, `version`, `owner`, `team`,
`environment`, `ready`, `age`.

#### wf_describe

Returns detailed information about a single workflow
deployment, including metadata annotations, replica
status, node list, and trigger configuration.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace of the workflow. |
| `name` | string | Yes | Workflow deployment name. |

Returns: `name`, `namespace`, `version`, `owner`, `team`,
`tags`, `environment`, `ready`, `replicas`,
`ready_replicas`, `image`, `age`, `nodes`, `triggers`,
`annotations` (all `tentacular.dev/*` annotations).

Node names and trigger descriptions are enriched from the
workflow ConfigMap (`<name>-code`) when available.

### Workflow Execution

#### wf_run

Triggers a deployed workflow by creating an ephemeral
in-cluster curl pod that POSTs to the workflow's `/run`
endpoint. Returns the JSON output.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace of the workflow. |
| `name` | string | Yes | Workflow deployment name. |
| `input` | byte[] | No | Optional JSON input payload (byte array). |
| `timeout_seconds` | int | No | Timeout in seconds (default 120, max 600). |

Returns: `name`, `namespace`, `output` (raw JSON),
`duration_ms`, `pod_name`.

### Workflow Lifecycle

#### wf_apply

Apply a set of Kubernetes manifests as a named deployment
in a namespace. Uses release labels for tracking and
garbage collection.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Target namespace for the workflow. |
| `name` | string | Yes | Deployment name for tracking resources. |
| `manifests` | array | Yes | List of Kubernetes manifest objects to apply. |

Allowed manifest kinds: Deployment, Service,
PersistentVolumeClaim, NetworkPolicy, ConfigMap,
Secret, Job, CronJob, Ingress.

Returns: `name`, `namespace`, `created` count,
`updated` count, `deleted` count (garbage-collected
resources no longer in the manifest set).

#### wf_remove

Remove all resources belonging to a named deployment
in a namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace containing the workflow resources. |
| `name` | string | Yes | Deployment name to remove. |

#### wf_status

Get status of all resources belonging to a named
deployment in a namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace containing the workflow resources. |
| `name` | string | Yes | Deployment name to check status for. |

### Workflow Observability

#### wf_logs

Get pod logs from a namespace. Returns tail lines
(default 100).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace of the pod. |
| `pod` | string | Yes | Name of the pod to get logs from. |
| `container` | string | No | Container name (defaults to first container). |
| `tail_lines` | int | No | Number of log lines to return (default 100). |

**Tip:** Use `wf_pods` first to find the pod name, then
pass it to `wf_logs`.

#### wf_pods

List pods in a namespace with phase, readiness, restart
count, images, and age.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to list pods in. |

#### wf_events

List events in a namespace sorted by most recent first.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to list events in. |
| `limit` | int | No | Maximum number of events to return (default 100). |

#### wf_jobs

List Jobs and CronJobs in a namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to list jobs in. |

### Namespace Management

#### ns_create

Create a new managed namespace with network policies,
resource quotas, and workflow RBAC.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Name of the namespace to create. |
| `quota_preset` | string | Yes | Resource quota preset: `small`, `medium`, or `large`. |

#### ns_delete

Delete a managed namespace. Only namespaces with the
tentacular managed-by label can be deleted.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Name of the namespace to delete. |

#### ns_get

Get details for a namespace including labels, status,
quota summary, and limit range summary.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Name of the namespace to get. |

#### ns_list

List all namespaces managed by tentacular.

No parameters.

### Credentials

#### cred_issue_token

Issue a short-lived token for the tentacular-workflow
service account in a namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to issue the token for. |
| `ttl_minutes` | int | Yes | Token lifetime in minutes (10-1440). |

#### cred_kubeconfig

Generate a kubeconfig for the tentacular-workflow
service account in a namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace for the kubeconfig context. |
| `ttl_minutes` | int | Yes | Token lifetime in minutes (10-1440). |

#### cred_rotate

Rotate the workflow service account in a namespace,
invalidating all existing tokens.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace whose workflow service account should be rotated. |

### Cluster Operations

#### cluster_preflight

Run preflight checks for a namespace: API reachability,
namespace existence, RBAC, and gVisor availability.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to run preflight checks against. |

#### cluster_profile

Profile the cluster: K8s version, distribution, nodes,
runtime classes, CNI, storage, and extensions.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | No | Optional namespace to include quota and limit range details. |

### Cluster Health

#### health_nodes

List nodes with readiness, capacity, allocatable
resources, kubelet version, and unhealthy conditions.

No parameters.

#### health_ns_usage

Compare namespace resource usage against ResourceQuota
limits and return utilization percentages.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to check resource usage for. |

#### health_cluster_summary

Aggregate cluster-wide CPU, memory, and pod counts
across all nodes.

No parameters.

### Security Audit

#### audit_rbac

Audit RBAC in a namespace: scan for wildcard
verbs/resources, sensitive access, and
ClusterRoleBindings targeting namespace service accounts.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to audit RBAC in. |

#### audit_netpol

Audit network policies in a namespace: check for
default-deny policy, missing egress restrictions, and
list all policies.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to audit network policies in. |

#### audit_psa

Audit Pod Security Admission labels on a namespace:
check enforce/audit/warn levels and flag non-restricted
or missing configuration.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to audit Pod Security Admission configuration in. |

### gVisor Runtime Sandbox

#### gvisor_check

Check whether a gVisor RuntimeClass is available in the
cluster.

No parameters.

#### gvisor_apply

Apply the gVisor runtime class annotation to a managed
namespace.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace to apply gVisor runtime class annotation to. |

#### gvisor_verify

Verify gVisor sandboxing by creating an ephemeral pod
with the gVisor runtime class and checking kernel
identity.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `namespace` | string | Yes | Namespace in which to create the verification pod. |

### Module Proxy

#### proxy_status

Check the installation and readiness status of the
module proxy (esm.sh).

No parameters.

Returns: `installed`, `ready`, `namespace`, `image`,
`storage`.

---

For full CLI reference, read [references/cli.md](references/cli.md).

## Node Contract

Every node is a TypeScript file with a single default
export:

```typescript
import type { Context } from "tentacular";

export default async function run(
  ctx: Context,
  input: unknown
): Promise<unknown> {
  // input: output from upstream node(s) via edges
  // return value: passed to downstream node(s)
  ctx.log.info("processing");
  return { result: "done" };
}
```

### Context API

| Member | Type | Description |
|--------|------|-------------|
| `ctx.dependency(name)` | `(string) => DependencyConnection` | **Primary API** for external services. Returns connection metadata (host, port, protocol, authType, protocol-specific fields) and resolved secret value. HTTPS deps also get a `fetch(path, init?)` convenience method that builds the URL (no auth injection -- nodes handle auth explicitly). Throws if the dependency is not declared in the contract. |
| `ctx.log` | `Logger` | Structured logging with `info`, `warn`, `error`, `debug` methods. All output prefixed with `[nodeId]`. |
| `ctx.config` | `Record<string, unknown>` | Workflow-level config from `config:` in workflow.yaml. Use for business-logic parameters only (e.g., `target_repo`). |
| `ctx.fetch(service, path, init?)` | `(string, string, RequestInit?) => Promise<Response>` | **Legacy.** HTTP request with auto URL construction. Flagged as contract violation when a contract is present. Use `ctx.dependency()` instead. |
| `ctx.secrets` | `Record<string, Record<string, string>>` | **Legacy.** Direct secret access. Flagged as contract violation when a contract is present. Use `ctx.dependency()` instead. |

### Using ctx.dependency()

Nodes access external service connection info through
the contract dependency API:

```typescript
const pg = ctx.dependency("postgres");
// pg.protocol, pg.host, pg.port, pg.database, pg.user
// pg.authType -- "password", pg.secret -- resolved value

const gh = ctx.dependency("github-api");
// gh.fetch(path, init?) -- URL builder (no auth injection)
const resp = await gh.fetch!("/repos/org/repo", {
  headers: { "Authorization": `Bearer ${gh.secret}` },
});
```

**Auth is explicit.** `dep.fetch()` builds the URL
(`https://<host>:<port><path>`) but does not inject
auth headers. Nodes set auth headers themselves using
`dep.secret` and `dep.authType`.

In mock context (during `tntc test`), `ctx.dependency()`
returns mock values and records access for drift
detection.

**Migration note:** Replace `ctx.config` + `ctx.secrets`
assembly with `ctx.dependency()`. Connection metadata
and secret references belong in `contract.dependencies`.
The `config` section should contain only business-logic
parameters (e.g., `target_repo`, `sep_label`).

## Trigger Types

| Type | Mechanism | Required Fields | K8s Resources | Status |
|------|-----------|----------------|---------------|--------|
| `manual` | HTTP POST `/run` | none | -- | Implemented |
| `cron` | K8s CronJob -> curl POST `/run` | `schedule`, optional `name` | CronJob | Implemented |
| `queue` | NATS subscription -> execute | `subject` | -- | Implemented |
| `webhook` | Future: gateway -> NATS bridge | `path` | -- | Roadmap |

### Trigger Name Field

Triggers can have an optional `name` field (must match
`[a-z][a-z0-9_-]*`, unique within workflow). Named cron
triggers POST `{"trigger": "<name>"}` to `/run`. Root
nodes receive this as `input.trigger` to branch behavior.

### Cron Trigger Lifecycle

1. Define in workflow.yaml: `type: cron`,
   `schedule: "0 9 * * *"`, optional `name`
2. `tntc deploy` generates CronJob manifest(s) alongside
   Deployment and Service
3. CronJob naming: `{wf}-cron` (single) or
   `{wf}-cron-0`, `{wf}-cron-1` (multiple)
4. CronJob curls
   `http://{wf}.{ns}.svc.cluster.local:8080/run` with
   trigger payload
5. `tntc undeploy` deletes CronJobs by label selector
   (automatic cleanup)

### Queue Trigger (NATS)

Queue triggers subscribe to NATS subjects. Config:
`config.nats_url`, auth: `secrets.nats.token`. If
either is missing, the engine warns and skips NATS
(graceful degradation). Messages are parsed as JSON
and passed as input to root nodes.

## Contract Model

The `contract` section declares every external dependency
a workflow needs. Dependencies are the single primitive:
secrets, NetworkPolicy, connection config, and validation
are all derived from the dependency list. There is no
separate `secrets` or `networkPolicy` section to author.

For the complete contract specification, including
structure, dependency protocols, auth types, enforcement
modes, drift detection, dynamic-target dependencies,
label-scoped ingress, and NetworkPolicy overrides, read
[references/contract.md](references/contract.md).

## Config Block

The `config:` block is **open** -- it accepts arbitrary
keys alongside `timeout` and `retries`. Custom keys flow
through to `ctx.config` in nodes. This is the standard
mechanism for non-secret workflow configuration.

```yaml
config:
  timeout: 30s
  retries: 2
  nats_url: "nats.ospo-dev.miralabs.dev:18453"
  custom_setting: "value"
```

In Go, extra keys are stored in
`WorkflowConfig.Extras` (via `yaml:",inline"`). Use
`ToMap()` to get a flat merged map.

## Minimal workflow.yaml

```yaml
name: my-workflow
version: "1.0"
description: "A minimal workflow"

# Optional: metadata for MCP discovery (wf_list / wf_describe)
# metadata:
#   owner: my-team
#   tags: [dev]

triggers:
  - type: manual

contract:
  version: "1"
  dependencies: {}

nodes:
  hello:
    path: ./nodes/hello.ts

edges: []

config:
  timeout: 30s
  retries: 0
```

## Common Workflow

```
# Bootstrap (one-time per cluster)
tntc cluster install               # deploy MCP server + module proxy
tntc configure --registry reg.io   # one-time registry setup

# Workflow development
tntc init my-workflow              # scaffold directory
cd my-workflow
# edit nodes/*.ts and workflow.yaml
tntc validate                      # check spec validity
tntc secrets check                 # verify secrets
tntc secrets init                  # create .secrets.yaml
tntc dev                           # local dev server
tntc test                          # run node tests
tntc test --pipeline               # run full DAG e2e
tntc build                         # build container image

# Deploy and operate (all via MCP)
tntc cluster check                 # verify K8s cluster via MCP
tntc deploy                        # deploy to cluster via MCP
tntc status my-workflow            # check deploy status
tntc list                          # list all workflows
tntc run my-workflow               # trigger workflow
tntc logs my-workflow              # view pod logs
tntc undeploy my-workflow          # remove from cluster
```

## Deployment Flow

The recommended agentic deployment flow validates
workflows through six steps:

```
tntc validate -o json               # 1. Validate spec + contract
tntc visualize --rich --write       # 2. Persist contract artifacts
tntc test -o json                   # 3. Mock tests + drift detection
tntc test --live --env <target> -o json  # 4. Live test against target env
tntc deploy -o json                 # 5. Deploy (auto-gates on live)
tntc run <name> -o json             # 6. Post-deploy verification
```

**Required checklist before build/deploy:**

- [ ] Contract section present in workflow.yaml
- [ ] `tntc validate` passes (contract + spec)
- [ ] `tntc visualize --rich --write` reviewed with user
- [ ] Dependency hosts and ports confirmed
- [ ] Secret refs match `.secrets.yaml` keys
- [ ] Derived NetworkPolicy matches expected access
- [ ] `tntc test` passes with zero drift
- [ ] `workflow-diagram.md` and `contract-summary.md` committed to repo

For step details, deploy gate behavior, structured
output format, and the pre-build review gate, read
[references/deployment-guide.md](references/deployment-guide.md).

## Dependency Preflight

Before deploying a workflow, verify that all
infrastructure dependencies are in place. Deploying
without these checks leads to pods that start but
fail at runtime.

**Run these checks after cluster profiling and before
build/deploy:**

- **Namespace exists** -- use `ns_get` (MCP) or
  `tntc cluster check -n <namespace>` to confirm the
  target namespace is created. If missing, create it
  with `ns_create` (MCP) or ask the cluster admin.
- **Database exists** -- if the workflow declares a
  `postgresql` dependency in `contract.dependencies`,
  verify the database is provisioned and reachable
  before deploying. Mock tests do not catch a missing
  database.
- **External services reachable** -- if the workflow
  declares `https` dependencies, confirm the target
  hosts are reachable from the cluster. NetworkPolicy
  allows egress, but DNS or firewall issues can still
  block traffic.
- **Module proxy healthy** -- use `proxy_status` (MCP)
  to verify the in-cluster esm.sh proxy is running.
  The engine needs it to resolve TypeScript imports at
  startup.

These checks are fast and prevent the most common
class of deploy-then-debug failures.

## Environment Configuration

Named environments (`dev`, `staging`, `production`)
extend the config cascade with cluster-specific
settings. Define them in `~/.tentacular/config.yaml`
or `.tentacular/config.yaml`. Each environment can
specify `context`, `namespace`, `image`,
`runtime_class`, `config_overrides`, and
`secrets_source`. Resolution order: CLI flags >
environment config > project config > user config >
defaults. See
[references/deployment-guide.md](references/deployment-guide.md)
for full environment configuration details.

### MCP Configuration

The MCP server connection is configured automatically
by `tntc cluster install`. Manual configuration:

```yaml
# ~/.tentacular/config.yaml
mcp:
  endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
  token_path: ~/.tentacular/mcp-token
```

Environment variables (override config file):
- `TNTC_MCP_ENDPOINT` -- MCP server URL
- `TNTC_MCP_TOKEN` -- bearer token value

## Agent Workflow Guide

Agents must plan before coding. The planning loop
confirms user intent, authors the contract first,
derives secrets and network intent, pre-validates
the environment, and defines dev-to-prod promotion
gates. Do not start implementation until the plan
is explicitly confirmed by the user.

For the full agent workflow guide including planning
steps, E2E cycle, image tag cascade, deploy flags,
and common gotchas, read
[references/agent-workflow.md](references/agent-workflow.md).

## References

For detailed documentation on specific topics:

- [CLI Reference](references/cli.md)
  -- full command table, global flags, namespace
  resolution, config files, visualization reference
- [Contract Model](references/contract.md)
  -- contract structure, dependency protocols, auth
  types, enforcement, drift detection, dynamic targets,
  label-scoped ingress, NetworkPolicy overrides
- [Agent Workflow Guide](references/agent-workflow.md)
  -- planning steps, E2E cycle, image tag cascade,
  deploy flags, common gotchas
- [Workflow Specification](references/workflow-spec.md)
  -- complete workflow.yaml format, all fields, trigger
  types, validation rules
- [Node Development](references/node-development.md)
  -- Context API details, data passing between nodes,
  patterns
- [Testing Guide](references/testing-guide.md)
  -- fixture format, mock context, node and pipeline
  testing, live testing
- [Deployment Guide](references/deployment-guide.md)
  -- build, deploy, cluster check, security model,
  secrets, environment config, deploy gate
