## Context

SKILL.md is the primary instruction document the AI agent reads when using tentacular. It needs to document the new wf_health and wf_health_ns MCP tools so the agent can discover and use them. The deployment guide and phase docs also need health monitoring coverage.

## Goals / Non-Goals

**Goals:**
- Document wf_health and wf_health_ns tools with parameter tables and return value descriptions
- Explain the G/A/R (Green/Amber/Red) health model clearly for the AI agent
- Define standard report formats with colored emoji indicators for health status
- Provide progressive disclosure guidance: start with single workflow health, then namespace scan, then detail mode
- Update deployment guide and phase 05 with health verification steps

**Non-Goals:**
- Documenting engine internals (telemetry sink implementation details)
- API reference for the engine health endpoint
- Documenting NetworkPolicy changes

## Decisions

### G/A/R emoji indicators
Use traffic-light emoji convention: green_circle for GREEN, yellow_circle for AMBER, red_circle for RED. This aligns with common status conventions and is visually scannable in terminal output.

### Progressive disclosure in SKILL.md
Document health tools in a workflow pattern: (1) check single workflow health, (2) scan namespace, (3) use detail mode for debugging. This matches how an operator naturally escalates investigation.

### Standard report formats
Define two report templates: Workflow Listing Report (table with name, namespace, status emoji, reason) and Workflow Detail Report (single workflow deep dive with telemetry data). These give the agent consistent output patterns.

## Risks / Trade-offs

- **SKILL.md size**: Adding health tool docs increases SKILL.md by approximately 100-150 lines. Mitigation: use concise parameter tables and keep examples minimal.
- **Emoji rendering**: Some terminals may not render emoji correctly. Mitigation: always include text status alongside emoji (e.g., "GREEN" not just the circle).
