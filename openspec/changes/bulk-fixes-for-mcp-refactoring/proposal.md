## Why

The SKILL documentation references pre-MCP patterns that no longer match the current deployment flow. Hardcoded `--env dev` references, missing dependency preflight guidance, undocumented trigger pod NetworkPolicy requirements, stale K8s API references, and outdated MCP tools documentation all cause confusion and deployment failures for users following the SKILL guide.

## What Changes

- Add dependency preflight guidance: document how to validate external service connectivity before deployment
- Fix hardcoded `--env dev` references to use environment-agnostic patterns or explain the `--env` flag properly
- Document trigger pod NetworkPolicy requirement: explain that CronJob trigger pods need egress NetworkPolicy to reach the workflow Service
- Update MCP tools documentation to reflect current tool signatures and behavior
- Fix stale K8s API references that point to removed or renamed API endpoints

## Capabilities

### New Capabilities
- `dependency-preflight-docs`: Document dependency preflight validation process

### Modified Capabilities
- `skill-documentation`: Update deployment guidance, MCP tools reference, and K8s API references

## Impact

- `SKILL.md`: Primary documentation file -- update deployment sections, MCP tools reference, dependency guidance
- `phases/`: Phase documentation may need updates for environment references
- `references/`: Reference documentation for MCP tools and K8s APIs
