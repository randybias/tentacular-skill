---
name: tentacular
description: Build, test, and deploy TypeScript workflow DAGs on Kubernetes using the tntc CLI. Use when creating new agentic workflows, deploying or managing existing workflows, writing or debugging workflow nodes, running tests, checking workflow status, viewing logs, or working with Tentacular secrets and contracts. Covers the full lifecycle: scaffold -> validate -> test -> build -> deploy -> run -> monitor.
---

# Tentacular

## Common Mistakes -- Never Do This

| # | Mistake | What Happens | Fix |
|---|---------|--------------|-----|
| 1 | Using `source:` instead of `path:` in workflow.yaml nodes | Schema validation fails | Use `path: ./nodes/foo.ts` |
| 2 | Putting `depends_on:` inside node blocks instead of top-level `edges:` | Silently ignored, DAG has no edges | Use top-level `edges:` list |
| 3 | Node returns `{ status: "ok" }` or `{}` (performative node) | Downstream nodes receive empty data | Return actual result data |
| 4 | Skipping the contract -- writing nodes before declaring dependencies | Runtime throws "not declared in contract" | Write contract first, then nodes |
| 5 | Writing code before user confirms the DAG design | Rework when design changes | STOP after DAG design, wait for confirmation |
| 6 | Adding `tentacular-*` deps without calling `exo_status` first | Deploy fails if exoskeleton is disabled | Always check `exo_status` first |
| 7 | Adding `host`/`port`/`auth` fields to a `tentacular-*` dependency | MCP server overwrites them during provisioning | Only set `protocol:` for managed deps |
| 8 | Deploying without `tntc test --pipeline` passing | Broken DAG in production | Run full pipeline test before deploy |
| 9 | Ignoring auth failures (401/403) in node tests | Auth fails silently in production | Treat every 401/403 as a blocker |
| 10 | Passing literal `~` in KUBECONFIG paths instead of expanding | Path resolution fails | Expand to full absolute path |
| 11 | Using `ctx.fetch()` or `ctx.secrets` instead of `ctx.dependency()` | Legacy API, flagged as contract violation | Use `ctx.dependency(name)` |
| 12 | Deploying with `exo_status.auth_enabled=true` without running `tntc login` | Deploy rejected with auth error | Run `tntc login` before deploying |
| 13 | Fixture `expected: {}` -- test passes even when node returns nothing | False green tests | Set meaningful expected values |
| 14 | Skipping cluster profile before workflow design | Wrong assumptions about cluster capabilities | Always run Phase 3 first |

---

## CLI vs MCP Routing

Building or developing? **CLI.** Querying or operating the cluster? **MCP tools.**

| Situation | Route | Tool |
|-----------|-------|------|
| Scaffold / validate / test / build / deploy | CLI | `tntc <command>` |
| Local dev server | CLI | `tntc dev` |
| CI/CD pipeline | CLI | `tntc <command>` |
| Read/write local files | CLI | `tntc <command>` |
| List/inspect deployed workflows | MCP | `wf_list`, `wf_describe` |
| Trigger workflow run | MCP | `wf_run` |
| View logs / pods / events / jobs | MCP | `wf_logs`, `wf_pods`, `wf_events`, `wf_jobs` |
| Cluster health and security audit | MCP | `health_*`, `audit_*` |
| Namespace management | MCP | `ns_*` |
| Install MCP server | Helm | `helm install` |
| Workflow needs backing services? | MCP | Check `exo_status` first |

---

## MCP Authentication

When `mcp_endpoint` is configured for an environment, the MCP server may require
OIDC authentication. Check and handle auth before any MCP operations:

| Auth Mode | How to Detect | Login Required? |
|-----------|---------------|-----------------|
| Bearer-token only | `mcp_token_path` is set, no OIDC config | No -- token file is sufficient |
| OIDC enabled | `exo_status` returns `auth_enabled: true` | Yes -- run `tntc login --env <env>` |

**Before MCP operations on an OIDC-enabled server:**

1. Run `tntc whoami --env <env>` to check authentication status
2. If not authenticated, run `tntc login --env <env>` (browser-based SSO flow)
3. OIDC tokens last 12 hours -- re-authentication is infrequent
4. If a 401/403 occurs mid-session, the token has expired -- re-run `tntc login --env <env>`

Skipping login on an OIDC-enabled server causes all MCP tool calls to fail
with authentication errors. See `references/contract-model.md` for OIDC
configuration details.

---

## Tool Safety Classification

### Read-Only Tools (safe to call at any time)

| Tool | Description |
|------|-------------|
| `ns_get` | Get namespace details, quota, limit range |
| `ns_list` | List managed namespaces |
| `wf_list` | List deployed workflows |
| `wf_describe` | Describe a single workflow |
| `wf_status` | Get resource status for a deployment |
| `wf_pods` | List pods in a namespace |
| `wf_logs` | Get pod logs |
| `wf_events` | List namespace events |
| `wf_jobs` | List Jobs and CronJobs |
| `wf_health` | Single workflow G/A/R health |
| `wf_health_ns` | Namespace-wide G/A/R health |
| `cluster_preflight` | Run preflight checks |
| `cluster_profile` | Profile cluster capabilities |
| `health_nodes` | Node readiness and capacity |
| `health_ns_usage` | Namespace resource utilization |
| `health_cluster_summary` | Cluster-wide resource summary |
| `audit_rbac` | Audit RBAC findings |
| `audit_netpol` | Audit network policies |
| `audit_psa` | Audit Pod Security Admission |
| `gvisor_check` | Check gVisor availability |
| `exo_status` | Exoskeleton service status |
| `exo_registration` | Workflow exo registration details |
| `exo_list` | List all exo registrations |
| `proxy_status` | Module proxy readiness |
| `permissions_get` | Get owner, group, and mode for a workflow |
| `ns_permissions_get` | Get owner, group, and mode for a namespace |

### Write Tools (create or modify resources)

| Tool | Description |
|------|-------------|
| `ns_create` | Create namespace with policies, quotas, RBAC |
| `ns_update` | Update namespace labels, annotations, quota |
| `wf_apply` | Apply K8s manifests as a named deployment |
| `wf_run` | Trigger a workflow execution |
| `wf_restart` | Rollout restart a deployment |
| `permissions_set` | Set group or mode for a workflow (owner-only) |
| `ns_permissions_set` | Set group or mode for a namespace (owner-only) |

### Destructive Tools (data loss possible -- confirm with user)

| Tool | Description |
|------|-------------|
| `ns_delete` | Delete a managed namespace and all contents |
| `wf_remove` | Remove all resources for a deployment |

---

## Schema Introspection

Do NOT memorize tool parameter tables. Use `tools/list` (MCP protocol method)
to get the current JSON Schema for any tool's parameters at runtime. This is
always authoritative. The safety classification above tells you WHAT each
tool does; `tools/list` tells you HOW to call it.

---

## Pipeline Phases

Execute in order. Each gate MUST pass before proceeding.

### Phase 1: Install tntc

Read `phases/01-install.md`. Gate: `tntc version` exits 0 AND
`~/.tentacular/engine/main.ts` exists.

### Phase 2: Configure Environments

Read `phases/02-configure.md`. Gate: config file exists with at least one
environment AND `tntc cluster check` passes.

### Phase 3: Cluster Profile

Read `phases/03-profile.md`. Gate: profile exists, `generatedAt` < 7 days,
Agent Guidance section read.

### Phase 4: Build the Workflow

Read `phases/04-build.md`. Gate: all individual node tests pass, all auth
dependencies verified, user has confirmed the DAG design.

### Phase 5: Test and Deploy

Read `phases/05-test-and-deploy.md`. Gate: validate, secrets check, node
tests, pipeline test, live test all pass. Post-deploy: run + verify output
+ `wf_health` green.

### Drift Triggers -- Re-run Phase 3 when:

- Deploy fails with `unknown RuntimeClass`
- NetworkPolicy rejects expected traffic
- PVC fails to bind
- Profile `generatedAt` > 7 days
- User mentions cluster upgrade or infrastructure change

---

## Creating a New Workflow

### Step 0: Search for Scaffolds (ALWAYS do this first)

Before writing any code, search for existing scaffolds:

```
tntc scaffold search "<relevant terms from user request>"
```

Evaluate results:
- **Exact match found:** Use Lifecycle B -- scaffold as-is, fill parameters.
  See [Parameter Negotiation Protocol](#parameter-negotiation-protocol-lifecycle-b).
- **Close match found:** Use Lifecycle C -- scaffold as starting point, modify.
  See [Scaffold Modification Protocol](#scaffold-modification-protocol-lifecycle-c).
- **No match:** Use Lifecycle A -- build from scratch with `tntc init`.

Always search scaffolds first, even if the user does not mention them. Private
scaffolds may contain org-specific patterns that are a better starting point
than building from scratch. Communicate your decision to the human before
proceeding.

### Step 1: Answer these questions with the user

1. **Where does it live?** `~/tentacles/<name>/`
   Create with `tntc init <name>` (from scratch) or
   `tntc scaffold init <scaffold> <name>` (from a scaffold).

2. **What triggers it?** manual | cron | queue
   (See `references/workflow-spec.md` for trigger details.)

3. **What external dependencies does it need?**
   List every API, database, queue, storage.
   Check `exo_status` for managed services.

4. **What is the data flow?**
   For each node, define: Input -> Action -> Output -> Edge.
   No performative nodes (output must carry data to the next node).

5. **Write the contract FIRST.** Declare all dependencies in `workflow.yaml`
   under `contract.dependencies` before writing any nodes.

6. **Get user confirmation.** Present the DAG, data flow, and contract.
   STOP. Do not write code until the user explicitly confirms.

---

## Scaffold Lifecycle Protocols

Four lifecycles cover every path from idea to deployed tentacle. Read
`references/scaffold-lifecycle.md` for the full reference including diagrams,
CLI commands, and extraction heuristics.

### Parameter Negotiation Protocol (Lifecycle B)

When a scaffold matches the user's request as-is:

1. `tntc scaffold init <scaffold> <name> --no-params`
2. Read `params.schema.yaml` from the new tentacle directory
3. For each parameter where `required: true`:
   - Present the parameter name, description, type, and example to the human
   - Collect the value
4. For each parameter where `required: false`:
   - Present the parameter with its default value
   - Ask if the human wants to change it
5. Edit `workflow.yaml` directly, replacing example values with real values
   at the paths indicated by the schema
6. `tntc scaffold params validate` -- verify all parameters have real values
7. `tntc validate`

**Alternative:** If all parameter values are known upfront, write a
`params.yaml` file and use `tntc scaffold init <scaffold> <name> --params-file
params.yaml` to apply them in one step. Same result, fewer edits.

### Scaffold Modification Protocol (Lifecycle C)

When a scaffold is close but needs structural changes:

1. `tntc scaffold init <scaffold> <name> --no-params`
2. Read the scaffold's `workflow.yaml` and node code to understand the pattern
3. Present a modification plan to the human (what changes, what stays)
4. After human approval, make all changes directly to the tentacle files
5. Delete or rewrite `params.schema.yaml` as appropriate (delete if structure
   diverged significantly, rewrite if extraction is anticipated)
6. Set `modified: true` in `tentacle.yaml` under the `scaffold` section
7. `tntc validate`

### Scaffold Extraction Protocol (Lifecycle D)

When extracting a reusable scaffold from a working tentacle:

1. `tntc scaffold extract --json` -- get the analysis without generating files
2. Review proposed parameters against extraction heuristics
   (see `references/scaffold-lifecycle.md`)
3. Present the parameterization plan to the human
4. After human approval, `tntc scaffold extract` to generate files
5. Default output: private scaffold at `~/.tentacular/scaffolds/<name>/`
6. To publish: re-run with `--public`, review generated files, PR to
   `tentacular-scaffolds`

### Post-Deployment Prompt

After a successful `tntc deploy`, if the tentacle was created from a modified
scaffold (Lifecycle C) or from scratch (Lifecycle A), ask:

> "This tentacle is working well. Would you like me to extract it as a
> reusable scaffold for future use?"

This encourages the scaffold library to grow organically from real work.

---

## Error Recovery

For diagnosis and remediation playbooks covering:
- Deploy failures (ImagePull, CrashLoopBackOff, NetworkPolicy, PSA
  violations, gVisor)
- Runtime errors (auth failures, module resolution, dependency timeouts)
- Health status triage (amber/red diagnosis)
- Common tntc command failures

Read `references/error-recovery.md`.

---

## Architecture

Tentacular is a secure workflow build and execution system for AI agents.
Three components: Go CLI (data plane), MCP Server (control plane),
Deno/TypeScript Engine (execution). The CLI has no direct Kubernetes API
access -- all cluster operations route through MCP.

Read `references/architecture.md` when:
- First time using Tentacular
- Need to understand component responsibilities
- Explaining the system to someone

## MCP Tools

34 tools organized into 12 groups: namespace management,
workflow lifecycle, execution, discovery, observability, health, cluster
ops, audit, exoskeleton, permissions, deploy, and module proxy. Use the safety
classification table above for risk assessment and `tools/list` for
parameter schemas.

Read `references/mcp-tools.md` when:
- Need tool behavior details beyond the safety table
- Need to understand response formats
- Working with G/A/R health model or standard reports

## Node Contract

Nodes are TypeScript files: `export default async function run(ctx: Context,
input: unknown): Promise<unknown>`. Use `ctx.dependency(name)` to access
external services. Auth is explicit (node sets headers using `dep.secret`).

Read `references/node-contract.md` when:
- Writing or debugging node code
- Setting up `ctx.dependency()` calls
- Migrating from legacy `ctx.fetch`/`ctx.secrets`

## Workflow Specification

workflow.yaml defines: name, version, triggers, contract, nodes (with
`path:` not `source:`), edges (top-level list), and config. The config block
is open -- custom keys flow to `ctx.config`.

Read `references/workflow-spec.md` when:
- Writing or editing workflow.yaml
- Adding triggers (cron, queue, webhook)
- Configuring metadata for `wf_list`/`wf_describe`

## Contract Model

The contract declares all external dependencies. Secrets, NetworkPolicy,
connection config, and validation are all derived from the dependency list.
Exoskeleton-managed deps use `tentacular-*` prefix; manual deps specify
full connection info.

Read `references/contract-model.md` when:
- Declaring dependencies in a workflow
- Working with exoskeleton services
- Configuring SSO/auth for deploy

## Authorization

Tentacles use POSIX-like owner/group/mode permissions enforced at the
MCP layer. Namespaces are directories; tentacles are files.

Read `references/authorization.md` when:
- Deploying with `--group` or `--share` flags
- Managing permissions (`permissions_get`, `permissions_set`, `chmod`, `chgrp`)
- Troubleshooting access denied errors

## Scaffold Lifecycle

Scaffolds are reusable starting structures for building tentacles. They come
from private scaffolds (`~/.tentacular/scaffolds/`), public quickstarts
(`~/.tentacular/quickstarts/`), or are created fresh (`tntc init`). Scaffolds
are temporary -- they accelerate tentacle creation but do not constrain it.

Read `references/scaffold-lifecycle.md` when:
- Creating a tentacle from a scaffold
- Negotiating parameters with the user
- Extracting a scaffold from a working tentacle
- Understanding workspace layout (`~/tentacles/`, `~/.tentacular/`)

## Deployment and Operations

Deployment flow: validate -> visualize -> test -> live test -> deploy ->
verify -> health check. Environment promotion is an agent pattern, not a CLI
command.

Read `references/deployment-ops.md` when:
- Deploying a workflow
- Promoting between environments
- Configuring environment settings
- Running dependency preflight checks

---

## References Index

| File | Content |
|------|---------|
| `phases/01-install.md` | Install tntc CLI and engine |
| `phases/02-configure.md` | Configure environments and MCP |
| `phases/03-profile.md` | Cluster profiling and Agent Guidance |
| `phases/04-build.md` | Design and build workflow nodes |
| `phases/05-test-and-deploy.md` | Testing gates and deployment |
| `references/architecture.md` | System architecture and components |
| `references/mcp-tools.md` | Tool details, health model, reports |
| `references/node-contract.md` | Node signature, Context API |
| `references/workflow-spec.md` | workflow.yaml schema and triggers |
| `references/contract-model.md` | Contract deps, exoskeleton, SSO |
| `references/deployment-ops.md` | Deploy flow, promotion, env config |
| `references/authorization.md` | Permission model, presets, CLI/MCP tools |
| `references/scaffold-lifecycle.md` | Scaffold lifecycle, CLI reference, extraction heuristics |
| `references/error-recovery.md` | Error playbooks and triage |
