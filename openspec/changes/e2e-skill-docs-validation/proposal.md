## Why

The SKILL.md documentation for wf_health and wf_health_ns tools, the G/A/R health model, standard report formats, and progressive disclosure workflow were added alongside the MCP tool implementation. However, there is no validation that the documented parameter tables, response formats, and report templates match the actual MCP tool behavior. Documentation drift causes agents to make incorrect tool calls and misinterpret results, breaking the health monitoring workflow.

## What Changes

- Add documentation accuracy tests: verify SKILL.md wf_health and wf_health_ns parameter tables match actual MCP tool schemas
- Add standard report format validation: verify documented listing and detail report templates produce correct output given actual tool response structures
- Add progressive disclosure workflow validation: verify the documented layered approach (namespace scan -> single check -> detail mode) produces coherent results
- Add cross-reference validation: verify deployment-guide.md and phases/05-test-and-deploy.md health monitoring sections reference correct tool names and parameters

## Capabilities

### New Capabilities
- `e2e-skill-health-docs`: Validation suite ensuring SKILL.md health tool documentation accuracy against actual MCP tool schemas and response formats
- `e2e-standard-reports`: Validation suite ensuring standard report templates produce correct output for workflow listing and detail views

### Modified Capabilities
<!-- None -->

## Impact

- Test scripts or validation checks comparing SKILL.md parameter tables against MCP tool JSON schemas
- Test validation for standard report format completeness (all G/A/R states represented, emoji indicators correct)
- Cross-reference checks between SKILL.md, deployment-guide.md, and phases/05-test-and-deploy.md
