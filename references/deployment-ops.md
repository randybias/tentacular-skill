# Deployment and Operations Reference

Deploy flow, environment promotion, dependency preflight, and post-deploy
verification. For step-by-step deploy procedures, see `phases/05-test-and-deploy.md`.

## Quick Command Reference

```bash
# MCP server install (one-time per cluster, via Helm)
# helm install tentacular-mcp charts/tentacular-mcp ...
tntc configure --registry reg.io   # one-time registry setup + MCP endpoint

# Workflow development
tntc init my-workflow              # scaffold directory
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

The recommended agentic deployment flow validates workflows through six steps:

```bash
tntc validate -o json               # 1. Validate spec + contract
tntc visualize --rich --write       # 2. Persist contract artifacts
tntc test -o json                   # 3. Mock tests + drift detection
tntc test --live --env <target> -o json  # 4. Live test against target env
tntc deploy -o json                 # 5. Deploy (auto-gates on live)
tntc run <name> -o json             # 6. Post-deploy verification
```

For step details, deploy gate behavior, structured output format, and the
pre-build review gate, read the
[Deploy a Tentacle](https://randybias.github.io/tentacular-docs/cookbook/deploy-tentacle/)
cookbook.

## Pre-Deploy Checklist

- [ ] Contract section present in workflow.yaml
- [ ] `tntc validate` passes (contract + spec)
- [ ] `tntc visualize --rich --write` reviewed with user
- [ ] Dependency hosts and ports confirmed
- [ ] Secret refs match `.secrets.yaml` keys
- [ ] Derived NetworkPolicy matches expected access
- [ ] `tntc test` passes with zero drift
- [ ] `workflow-diagram.md` and `contract-summary.md` committed to repo
- [ ] If workflow uses `tentacular-*` dependencies, confirm exoskeleton
      services are available via `exo_status`

## Dependency Preflight

Before deploying a workflow, verify that all infrastructure dependencies are
in place. Deploying without these checks leads to pods that start but fail
at runtime.

Run these checks after cluster profiling and before build/deploy:

- **Namespace exists** -- use `ns_get` (MCP) or `tntc cluster check -n
  <namespace>` to confirm the target namespace is created. If missing,
  create it with `ns_create` (MCP) or ask the cluster admin.
- **Database exists** -- if the workflow declares a `postgresql` dependency
  in `contract.dependencies`, verify the database is provisioned and
  reachable before deploying. Mock tests do not catch a missing database.
- **External services reachable** -- if the workflow declares `https`
  dependencies, confirm the target hosts are reachable from the cluster.
  NetworkPolicy allows egress, but DNS or firewall issues can still block
  traffic.
- **Module proxy healthy** -- use `proxy_status` (MCP) to verify the
  in-cluster esm.sh proxy is running. The engine needs it to resolve
  TypeScript imports at startup.

These checks are fast and prevent the most common class of
deploy-then-debug failures.

## Environment Promotion

Promotion is an **agent workflow pattern**, not a CLI command. The agent
deploys to dev first, verifies health, then deploys to prod using the same
CLI commands with different `--env` targets.

```bash
# 1. Deploy to dev
tntc deploy my-workflow --env dev

# 2. Verify health in dev (via MCP)
#    Use wf_health MCP tool or:
tntc status my-workflow --env dev --detail

# 3. Run in dev to confirm
tntc run my-workflow --env dev

# 4. Deploy to prod (same source, different env)
tntc deploy my-workflow --env prod

# 5. Verify health in prod
tntc status my-workflow --env prod --detail
```

Agent promotion rules:

- **Health gate:** The dev deployment MUST show GREEN health status before
  promoting to prod. Check with `wf_health` (MCP tool) or `tntc status`.
- **Same source:** Both environments deploy from the same local workflow
  source. Per-env settings (namespace, image, runtime_class, MCP endpoint)
  come from the environment config.
- **Secrets are separate:** Each environment has its own secrets. Deploying
  to prod does NOT copy dev secrets. Provision prod secrets separately before
  deploying.
- **No automatic rollback:** If prod deploy fails, diagnose via `tntc logs`
  and `wf_health` before retrying.

## Environment Configuration

Named environments (`dev`, `staging`, `production`) extend the config cascade
with cluster-specific settings. Define them in `~/.tentacular/config.yaml`
or `.tentacular/config.yaml`. Each environment can specify `namespace`,
`image`, `runtime_class`, `mcp_endpoint`, `mcp_token_path`,
`config_overrides`, and `secrets_source`. The `default_env` field sets which
environment is used when no `--env` flag or `TENTACULAR_ENV` variable is set.

Resolution order: CLI flags > environment config > project config > user
config > defaults.

See the [Deploy a Tentacle](https://randybias.github.io/tentacular-docs/cookbook/deploy-tentacle/)
cookbook for full environment configuration details.

## Post-Deploy Health Check

After every deployment:

1. Run `wf_health <name> -n <namespace>` -- check for green status.
2. If amber or red, run `wf_logs` to find the error.
3. Run the workflow once with `wf_run` to confirm end-to-end execution.
4. Check `wf_health detail=true` to verify execution telemetry.

A deployment is not complete until `wf_health` returns green AND a live
`wf_run` completes successfully.

## Agent Workflow Guide

Agents must plan before coding. The planning loop confirms user intent,
authors the contract first, derives secrets and network intent,
pre-validates the environment, and defines dev-to-prod promotion gates.
Do not start implementation until the plan is explicitly confirmed by the
user.

For the full agent workflow guide including planning steps, E2E cycle, image
tag cascade, deploy flags, and common gotchas, read the
[Agent Skill](https://randybias.github.io/tentacular-docs/concepts/agent-skill/) docs.

## External References

- [CLI Reference](https://randybias.github.io/tentacular-docs/reference/cli/)
  -- full command table, global flags, namespace resolution, config files
- [Node Contract](https://randybias.github.io/tentacular-docs/reference/node-contract/)
  -- Context API, dependency access, auth patterns, testing fixtures
- [Workflow Specification](https://randybias.github.io/tentacular-docs/reference/workflow-spec/)
  -- complete workflow.yaml format, all fields, trigger types, validation rules
- [Security Model](https://randybias.github.io/tentacular-docs/concepts/security/)
  -- contract-driven zero-trust, five defense layers, NetworkPolicy, drift
  detection
- [Deploy a Tentacle](https://randybias.github.io/tentacular-docs/cookbook/deploy-tentacle/)
  -- build, deploy, verify, troubleshoot
- [Testing Guide](https://randybias.github.io/tentacular-docs/guides/testing/)
  -- fixture format, mock context, node and pipeline testing
- [Cluster Configuration](https://randybias.github.io/tentacular-docs/guides/cluster-configuration/)
  -- environment config, OIDC/SSO, MCP resolution
