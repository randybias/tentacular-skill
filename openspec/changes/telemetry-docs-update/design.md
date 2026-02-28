## Context

The Tentacular skill documentation (SKILL.md, deployment guide, phase docs) serves as the primary reference for AI agents operating Tentacular workflows. The engine and MCP server are adding telemetry and health monitoring capabilities (`wf_health`, `wf_health_ns`). Without corresponding skill documentation, agents cannot discover or use these tools effectively.

The skill repo contains three files that need updates:
- `SKILL.md` (main skill definition, ~858 lines)
- `references/deployment-guide.md` (deployment and operations guide, ~207 lines)
- `phases/05-test-and-deploy.md` (testing and deployment phase, ~207 lines)

## Goals / Non-Goals

**Goals:**
- Document wf_health and wf_health_ns tools with full parameter tables and return type descriptions
- Explain G/A/R status semantics so agents can interpret and act on health results
- Add health monitoring to the post-deploy workflow so agents check health after deployment
- Keep documentation consistent with existing style and formatting patterns

**Non-Goals:**
- Documenting engine-internal telemetry implementation (that belongs in the engine repo)
- Adding new CLI commands (wf_health is MCP-only)
- Changing the deployment workflow or gating behavior

## Decisions

### 1. Place health tools after Workflow Observability section in SKILL.md

The new "Workflow Health" subsection goes after the existing "Workflow Observability" section (wf_logs, wf_pods, wf_events, wf_jobs) at line ~340. This groups health tools with related observability tools while keeping them distinct.

Alternative: Place under a new top-level section. Rejected because the health tools are MCP tools like the others and fit the existing hierarchy.

### 2. Use same parameter table format as existing tools

All MCP tool docs in SKILL.md use a consistent `| Parameter | Type | Required | Description |` table format. The new tools follow this exactly for consistency.

### 3. Inline G/A/R definitions rather than a separate reference page

The Green/Amber/Red status definitions are included inline in each doc where they appear rather than creating a separate reference document. The definitions are short (3 bullet points) and repeating them where needed is clearer than cross-referencing.

## Risks / Trade-offs

- **Documentation out of sync with implementation**: If the engine or MCP implementation changes status semantics, docs need manual updates. Mitigation: the G/A/R rules are simple and stable for v1.
- **Tool count update fragility**: SKILL.md hardcodes the tool count ("29 tools"). Updating to 31 requires finding and changing this number. Mitigation: this is a single line edit with a clear grep target.
