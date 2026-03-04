# Design: Update skill docs for CLI/MCP separation

## Affected Files

### SKILL.md

The main skill definition file. Search for and update:
- References to `tntc cluster install` -> replace with Helm instructions
- References to `tntc cluster profile` -> note it requires running MCP server
- Environment configuration examples -> add per-env MCP config fields
- Add agent promotion workflow pattern (deploy dev -> verify health -> deploy prod)

### phases/ directory

Check each phase document for references to changed commands:
- Cluster setup phases that mention `tntc cluster install`
- Deployment phases that should teach the agent promotion workflow pattern
- Configuration phases that show environment setup

### references/ directory

If command reference docs exist, update them to reflect:
- Removed: `tntc cluster install`
- Changed: `tntc cluster profile` (now routes through MCP)
- Changed: per-env MCP config in `config.yaml`
- New concept: agent promotion workflow (deploy dev -> verify -> deploy prod)

## Content Changes

### Installation guidance

Old:
```
tntc cluster install --namespace tentacular-system
```

New:
```
helm install tentacular-mcp charts/tentacular-mcp \
  --namespace tentacular-system --create-namespace
```

### Environment config example

Old:
```yaml
environments:
  dev:
    kubeconfig: ~/dev-secrets/kubeconfigs/dev.kubeconfig
    namespace: tentacular-dev-wf
```

New:
```yaml
default_env: dev
environments:
  dev:
    kubeconfig: ~/dev-secrets/kubeconfigs/dev.kubeconfig
    namespace: tentacular-dev-wf
    mcp_endpoint: http://tentacular-mcp.tentacular-system.svc.cluster.local:8080
    mcp_token_path: ~/.tentacular/tokens/dev-mcp-token
```

### Agent Promotion Workflow Pattern

Add to deployment/promotion phase guidance for the agent:
```
# Promotion is a workflow pattern, not a CLI command.
# The agent follows this sequence:

# 1. Deploy to dev
tntc deploy --env dev

# 2. Verify health in dev
# (agent checks wf_health via MCP, confirms GREEN status)

# 3. Deploy to prod
tntc deploy --env prod --verify
```
