## Why

The engine now supports telemetry (BasicSink/NoopSink) and the MCP server adds `wf_health`/`wf_health_ns` tools, but the skill documentation does not reference these capabilities. Agents using the Tentacular skill have no way to discover or use the new health monitoring tools. The SKILL.md, deployment guide, CLI reference, and phase 05 docs all need updates to reflect the new telemetry and health monitoring features.

## What Changes

- Update SKILL.md Architecture section to mention `GET /health?detail=1` telemetry endpoint
- Add `wf_health` and `wf_health_ns` tool documentation to SKILL.md MCP Tools Reference (parameter tables, return types, status values)
- Update SKILL.md tool count from 29 to 31
- Add "Health Monitoring" subsection to `references/deployment-guide.md` after post-deploy verification
- Add "Post-deploy health check" step to `phases/05-test-and-deploy.md`
- Document G/A/R status values (Green/Amber/Red) and their meanings across all updated docs

## Capabilities

### New Capabilities

- `health-monitoring-docs`: Documentation for wf_health and wf_health_ns MCP tools in SKILL.md, deployment guide, and phase 05 docs

### Modified Capabilities

<!-- None - this is a docs-only repo with no existing specs -->

## Impact

- `SKILL.md`: Architecture section (line ~87), MCP Tools Reference section (after line ~340), tool count (line ~201)
- `references/deployment-guide.md`: New "Health Monitoring" subsection after post-deploy verification (after line ~121)
- `phases/05-test-and-deploy.md`: New "Post-deploy health check" step after post-deploy verification (after line ~121)
- `references/cli.md`: No changes needed (wf_health is MCP-only, not a tntc command)
- No code changes, documentation only
