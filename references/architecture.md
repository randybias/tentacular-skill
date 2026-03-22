# Architecture Reference

System components, directory layout, namespace conventions, and MCP
configuration. For installation steps, see `phases/01-install.md`.
For environment configuration, see `phases/02-configure.md`.

## System Components

Tentacular is a secure workflow build and execution system designed for AI
agents. Its purpose is to allow an AI agent to build repeatable and durable
workflows (pipelines). Arbitrary workflows can be built dynamically by an
agent, reused or replaced as necessary, and deployed onto Kubernetes clusters
to be triggered manually or by various hooks (cron, webhook, etc.).

Every tentacular agentic workflow is deployed in a tightly constrained
sandbox, without a tool chain or other ancillary systems. Only what is needed
is deployed into a production runtime (gVisor). Each workflow's networking
and other access requirements are declared in the workflow contract, which
drives automatic generation of Kubernetes NetworkPolicy to lock down the
workflow's production network access.

Three key components:

- **Go CLI** (`cmd/tntc/`, `pkg/`) -- the data plane. Manages the workflow
  lifecycle: scaffold, validate, dev, test, build, deploy. All cluster
  operations route through the MCP server. The CLI has no direct Kubernetes
  API access.
- **MCP Server** (`tentacular-mcp`) -- the control plane. An in-cluster MCP
  server that proxies all Kubernetes operations through scoped RBAC. The CLI
  communicates with the cluster exclusively through MCP.
- **Deno/TypeScript Engine** (`engine/`) -- executes workflows as DAGs.
  Compiles workflow.yaml into topologically sorted stages, loads TypeScript
  node modules, runs them with a Context providing dependency resolution,
  fetch, logging, config, and secrets. Exposes HTTP triggers (`POST /run`,
  `GET /health`, `GET /health?detail=1` for telemetry snapshots).

Key benefits:

- Hardened execution environment for production workloads: attack surface is
  absolute minimum, runs on gVisor by default.
- Agent-friendly: build workflows using plain English in a repeatable and
  manageable manner. Full skill to help manage the workflow end-to-end.
  Testing is baked into the development process prior to pushing to production.

## Directory Layout

Production workflows (called **tentacles**) MUST be stored outside the
tentacular repository. The repo contains only the tool (CLI + engine) and
example workflows. Real workflows with secrets, credentials, and
production-specific configuration belong in a separate local directory.

Canonical location: `~/workspace/tentacles/`

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

Why separate:

- The tentacular repo is public -- secrets and production configs must never
  end up there.
- Workflows are operational artifacts, not source code for the tool itself.
- Production-ready scaffolds are available via tentacular-scaffolds --
  use `tntc scaffold init` to scaffold from them.

When building a new workflow, use `tntc init <name>` for a blank scaffold or
`tntc scaffold init <scaffold> <name>` to start from a production scaffold.

```bash
# Browse available scaffolds
tntc scaffold list
tntc scaffold search monitoring

# Scaffold from a quickstart
tntc scaffold init hn-digest my-news-digest
```

## Querying Deployed Workflows

All cluster operations route through the MCP server. Once the MCP server is
installed (via Helm) and the CLI is configured with the MCP endpoint,
commands like `list`, `status`, `logs`, `run`, and `undeploy` work without
`KUBECONFIG` or direct Kubernetes access:

```bash
tntc list -n <namespace>
tntc status <workflow-name> -n <namespace>
tntc logs <workflow-name> -n <namespace>
tntc run <workflow-name> -n <namespace>
```

The MCP endpoint and token are configured in `~/.tentacular/config.yaml`
(per-environment or global). No per-command MCP URL flags are needed.

Note: `tntc logs --follow` is not supported through MCP (snapshot only). Use
`kubectl logs -f` for streaming.

## Namespace Convention

Tentacular uses three categories of namespaces:

| Namespace | Purpose |
|-----------|---------|
| `tentacular-system` | MCP server and secure control plane. Protected -- no workflow tools can target this namespace. |
| `tentacular-support` | Secure support systems. Hosts the esm.sh module proxy. Protected from workflow operations. |
| All others with `app.kubernetes.io/managed-by=tentacular` | Workflow namespaces. Created by `ns_create`, managed by workflow tools. |

## MCP Configuration

MCP connection is configured per-environment or globally:

```yaml
# ~/.tentacular/config.yaml
default_env: dev

mcp:
  endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
  token_path: ~/.tentacular/mcp-token

environments:
  dev:
    namespace: tentacular-dev
    mcp_endpoint: http://localhost:8080
    mcp_token_path: ~/.tentacular/mcp-token
  prod:
    namespace: tentacular-prod
    mcp_endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
    mcp_token_path: ~/.tentacular/prod-mcp-token
```

MCP resolution cascade:
1. Active environment's `mcp_endpoint` / `mcp_token_path`
   (`--env` > `TENTACULAR_ENV` > `default_env`)
2. Global `mcp.endpoint` / `mcp.token_path`
3. `TNTC_MCP_ENDPOINT` / `TNTC_MCP_TOKEN` env vars
