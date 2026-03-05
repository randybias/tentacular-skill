# CLI Reference

Complete reference for `tntc` commands, flags, and configuration.

## Architecture

The CLI has no direct Kubernetes API access. All
cluster-facing commands (`deploy`, `run`, `list`,
`status`, `logs`, `undeploy`, `audit`, `cluster check`,
`cluster profile`) route through the MCP
server. The MCP server is installed separately via its
Helm chart. MCP connection details are configured
per-environment in `~/.tentacular/config.yaml` or via
`TNTC_MCP_ENDPOINT` / `TNTC_MCP_TOKEN` environment
variables.

## Command Reference

| Command | Usage | Key Flags | Description |
|---------|-------|-----------|-------------|
| `init` | `tntc init <name>` | | Scaffold a new workflow directory with workflow.yaml, example node, test fixture, .secrets.yaml.example |
| `validate` | `tntc validate [dir]` | | Parse and validate workflow.yaml (name, version, triggers, nodes, edges, DAG acyclicity, contract structure, derived artifacts) |
| `dev` | `tntc dev [dir]` | `-p` port (default 8080) | Start Deno engine locally with hot-reload (`--watch`). POST /run triggers execution |
| `test` | `tntc test [dir][/<node>]` | `--pipeline`, `--live`, `--env`, `--keep`, `--timeout`, `--warn` | Run node-level tests from fixtures, full pipeline test with `--pipeline`, or live cluster test with `--live`. `--warn` downgrades contract violations to warnings. |
| `build` | `tntc build [dir]` | `-t` tag, `-r` registry, `--push`, `--platform` | Generate Dockerfile (distroless Deno base), build container image via `docker build` |
| `deploy` | `tntc deploy [dir]` | `-n` namespace, `--env`, `--image`, `--runtime-class`, `--force`, `--verify`, `--warn` | Generate K8s manifests and apply via MCP. `--env` targets a named environment (context, namespace, image). Auto-gates on live test if dev env configured; `--force` skips. `--warn` downgrades contract violations to warnings. Namespace resolves: CLI > env > workflow.yaml > config > default |
| `audit` | `tntc audit <workflow-dir>` | `-n` namespace, `-o` json | Audit deployed resources against contract expectations via MCP |
| `configure` | `tntc configure` | `--registry`, `--namespace`, `--runtime-class`, `--project` | Set default config (user-level or project-level) |
| `secrets check` | `tntc secrets check [dir]` | | Check secrets provisioning against node requirements |
| `secrets init` | `tntc secrets init [dir]` | `--force` | Initialize .secrets.yaml from .secrets.yaml.example |
| `status` | `tntc status <name>` | `-n` namespace, `-o` json, `--detail` | Check deployment status via MCP; `--detail` shows pods, events |
| `run` | `tntc run <name>` | `-n` namespace, `--timeout` | Trigger a deployed workflow via MCP and return JSON result |
| `logs` | `tntc logs <name>` | `-n` namespace, `--tail` | View workflow pod logs via MCP (snapshot only; `--follow` not supported through MCP, use `kubectl logs -f`) |
| `list` | `tntc list` | `-n` namespace, `-o` json | List all deployed workflows via MCP with version, status, and age |
| `undeploy` | `tntc undeploy <name>` | `-n` namespace, `--yes`/`-y` | Remove a deployed workflow via MCP. Use `-y` to skip confirmation. |
| `cluster check` | `tntc cluster check` | `-n` namespace, `-o` json | Preflight validation of cluster readiness via MCP |
| `visualize` | `tntc visualize [dir]` | `--rich`, `--write` | Generate Mermaid diagram of the workflow DAG. `--rich` adds dependency graph, derived secrets, and network intent. `--write` writes `workflow-diagram.md` and `contract-summary.md` to the workflow directory. |

## Global Flags

- `-n`/`--namespace` (default "default")
- `-r`/`--registry`
- `-o`/`--output` (text|json)

## Namespace Resolution

Namespace resolution order: CLI `-n` flag >
`workflow.yaml deployment.namespace` > config file
default > `default`. When `--env` is used (deploy,
test --live), environment namespace is inserted after
CLI `-n`.

## Config Files

- `~/.tentacular/config.yaml` (user-level)
- `.tentacular/config.yaml` (project-level)
- Project overrides user.

### Recommended Config

Install the MCP server via Helm, then configure the
CLI with the MCP endpoint and token:

```yaml
registry: ghcr.io/randybias
namespace: default
runtime_class: gvisor
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
    image: ghcr.io/randybias/tentacular-engine:latest
    runtime_class: gvisor
    mcp_endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
    mcp_token_path: ~/.tentacular/prod-mcp-token
```

For this repository, the canonical public engine image is:
`ghcr.io/randybias/tentacular-engine:latest`.

Without an explicit environment `image`, deploy/test-live may
fall back to `<workflow-dir>/.tentacular/base-image.txt` or
the internal default `tentacular-engine:latest`.

### MCP Configuration

MCP connection is resolved per-environment:

| Config Field | Env Var | Description |
|-------------|---------|-------------|
| `environments.<name>.mcp_endpoint` | -- | Per-env MCP server URL |
| `environments.<name>.mcp_token_path` | -- | Per-env path to bearer token file |
| `mcp.endpoint` | `TNTC_MCP_ENDPOINT` | Global MCP server URL (fallback) |
| `mcp.token_path` | -- | Global path to bearer token file |
| -- | `TNTC_MCP_TOKEN` | Bearer token value (overrides token_path) |

Resolution order: active environment config >
global config > environment variables.

### Per-Environment MCP Access

Each environment can specify its own MCP endpoint and
token, enabling the CLI to work with multiple clusters
without direct kubeconfig access:

| Field | Description |
|-------|-------------|
| `mcp_endpoint` | MCP server URL for this environment |
| `mcp_token_path` | Path to bearer token file. `~` is expanded to home directory. |

```yaml
default_env: dev

environments:
  dev:
    namespace: tentacular-dev
    mcp_endpoint: http://localhost:8080
    mcp_token_path: ~/.tentacular/mcp-token
  staging:
    namespace: staging-workflows
    mcp_endpoint: http://staging-mcp:8080
    mcp_token_path: ~/.tentacular/staging-token
  prod:
    namespace: tentacular-prod
    mcp_endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
    mcp_token_path: ~/.tentacular/prod-token
```

## Visualization Reference

`tntc visualize` generates workflow diagrams. The
`--rich` flag adds contract-derived metadata.

### Basic Mode

```bash
tntc visualize sep-tracker
```

Produces a Mermaid diagram of the DAG topology
(nodes and edges only).

### Rich Mode

```bash
tntc visualize --rich sep-tracker
```

Rich output includes:

- **DAG topology**: nodes and edges as in basic mode
- **Dependency nodes**: external services shown with
  protocol and host labels
- **Derived secret inventory**: all secret key refs
  collected from `dep.auth.secret` across dependencies
- **Network intent summary**: derived egress and
  ingress rules from contract + triggers

Rich visualization output is deterministic and stable
for diffs.

### Write Mode

```bash
tntc visualize --rich --write sep-tracker
```

The `--write` flag writes artifacts to the workflow
directory instead of printing to stdout:

- **`workflow-diagram.md`** -- Mermaid diagram content
  (with code fence markers), ready for rendering
- **`contract-summary.md`** -- Derived secrets
  inventory, egress rules, and ingress rules as
  markdown tables

These files are co-resident with the workflow for
PR review. Output is deterministic and stable for
diffs. Agents MUST use `--write` during the pre-build
review gate to persist artifacts for commit.

### Example: sep-tracker Workflow Diagram

<!-- sep-tracker workflow diagram available after: tntc catalog init sep-tracker && tntc visualize --rich --write sep-tracker -->

The sep-tracker workflow demonstrates a full contract
with four external dependencies (GitHub API, Postgres,
Azure Blob Storage, Slack webhook). The rich
visualization shows how each dependency connects to
the nodes that use it, along with derived secrets and
network policy rules.
