## Context

The tentacular-skill repository contains the SKILL documentation that guides Claude Code agents through the workflow development lifecycle. After the MCP server refactoring, several documentation sections reference outdated patterns, causing agents to generate incorrect deployment configurations or miss required steps.

## Goals / Non-Goals

**Goals:**
- Ensure all deployment documentation matches the current MCP-based flow
- Add missing guidance for dependency preflight validation
- Document NetworkPolicy requirements for trigger pods
- Update MCP tools documentation to current signatures
- Remove or update stale K8s API references

**Non-Goals:**
- Restructuring the SKILL documentation
- Adding new SKILL phases
- Changing the SKILL workflow process itself

## Decisions

### Dependency preflight as explicit step
Add dependency preflight validation as an explicit step in the deployment phase. This catches connectivity issues (missing secrets, wrong endpoints, DNS failures) before the workflow is deployed.

### Environment-agnostic examples
Replace `--env dev` with `--env <environment>` in examples, with a note explaining that the environment flag selects the target namespace and configuration.

### Trigger NetworkPolicy documentation
Add a section explaining that CronJob trigger pods need an egress NetworkPolicy to reach the workflow Service. Include the generated NetworkPolicy as a reference example.

### MCP tools reference update
Update the MCP tools documentation to match current tool signatures. Remove references to deprecated tools and add new tools introduced during the refactoring.

## Risks / Trade-offs

- **Documentation churn**: Frequent updates to SKILL docs may cause confusion if agents have cached older versions. Mitigated by clear version dating in the docs.
