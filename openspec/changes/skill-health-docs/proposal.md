## Why

The new wf_health and wf_health_ns MCP tools need documentation in SKILL.md so the AI agent knows how to use them. The deployment guide, CLI reference, and phase 05 docs also need updates to cover the health monitoring workflow. Without documentation, the agent will not discover or use the health tools.

## What Changes

- Add wf_health and wf_health_ns tool documentation to SKILL.md with parameter tables, G/A/R health model explanation, and usage examples
- Add standard report formats: Workflow Listing Report (with health status column) and Workflow Detail Report (with G/A/R emoji indicators)
- Add progressive disclosure guidance for health check workflow (single workflow -> namespace scan -> detail mode)
- Update references/deployment-guide.md with health monitoring section
- Update references/cli.md with health-related commands
- Update phases/05-test-and-deploy.md with health check steps post-deploy

## Capabilities

### New Capabilities
- `skill-health-tools`: SKILL.md documentation for wf_health and wf_health_ns tools including parameter tables, G/A/R model, and progressive disclosure
- `standard-reports`: Standard report formats for workflow listing and detail views with colored health status indicators

### Modified Capabilities
<!-- None -->

## Impact

- `SKILL.md`: Add MCP Tools section for wf_health and wf_health_ns, G/A/R model reference, standard report formats, progressive disclosure
- `references/deployment-guide.md`: Add health monitoring section covering post-deploy health checks
- `references/cli.md`: Add health check commands and flags
- `phases/05-test-and-deploy.md`: Add health verification step after deployment
