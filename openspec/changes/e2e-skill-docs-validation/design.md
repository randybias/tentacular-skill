## Context

The Skill repo contains SKILL.md (the primary agent-facing documentation), reference docs, and phase guides. The health monitoring documentation was added in the skill-health-docs change and covers wf_health/wf_health_ns tools, G/A/R model, standard reports, and progressive disclosure. These docs were written to match the MCP tool implementation, but there is no ongoing validation that they stay in sync.

## Goals / Non-Goals

**Goals:**
- Verify documented parameter names and types match actual MCP tool JSON schemas
- Verify standard report templates cover all G/A/R states with correct indicators
- Verify cross-references between SKILL.md and reference docs are consistent
- Provide a repeatable validation checklist that can be run after any doc update

**Non-Goals:**
- Automated schema extraction from the running MCP server (manual comparison is sufficient for this phase)
- Testing the MCP tools themselves (covered by the MCP repo E2E change)
- Validating prose quality or completeness of narrative documentation
- CI integration for doc validation (future work)

## Decisions

### 1. Validation approach: manual checklist vs. automated parsing

**Decision**: Use a structured validation checklist in the tasks file that a human or agent can execute by comparing SKILL.md content against the MCP tool source code. Include grep-based spot checks where possible.

**Rationale**: The Skill repo is pure documentation (Markdown). Automated parsing of Markdown tables is brittle and over-engineered for the current scale. A structured checklist is more maintainable and directly useful for the agent that maintains the docs.

### 2. Scope: health tools only, not all MCP tools

**Decision**: Focus validation on wf_health, wf_health_ns, and the G/A/R model. Other MCP tools are out of scope for this change.

**Rationale**: These are the new tools added in the telemetry feature. Existing tools have been validated through prior usage. Keeping scope narrow makes the validation actionable.

## Risks / Trade-offs

- [Manual validation may miss subtle schema differences] -> Mitigated by including specific field-by-field comparison tasks
- [Docs may drift again after validation] -> Future work: add a CI check that extracts tool schemas from MCP binary and compares to SKILL.md
