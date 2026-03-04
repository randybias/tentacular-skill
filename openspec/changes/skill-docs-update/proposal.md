# Update skill phases and references for CLI/MCP separation

## Why

The tentacular-skill repo contains SKILL.md and phase documentation that
references CLI commands and architecture. Several changes from the CLI/MCP
separation refactor affect the skill documentation:

1. `tntc cluster install` is removed -- the skill should reference Helm
   installation instead.
2. Promotion is an agent workflow concept -- the skill should teach the agent
   the promotion pattern: deploy to dev, verify health, then deploy to prod.
   There is no `tntc promote` CLI command; the agent uses existing deploy flow.
3. Per-environment MCP config changes how environments are configured -- the
   skill's environment setup guidance needs updating.
4. `tntc cluster profile` now routes through MCP -- the skill should reflect
   that profiling requires a running MCP server.

## What Changes

- **Update SKILL.md** to remove references to `tntc cluster install` and replace
  with Helm-based installation instructions.
- **Add agent promotion workflow guidance** to the skill's deployment phase:
  deploy to dev first, verify health via `wf_health`, then deploy to prod.
  This is an agent workflow pattern, not a CLI command.
- **Update environment configuration guidance** to reflect per-env MCP config
  with `default_env`, `mcp_endpoint`, and `mcp_token_path` fields.
- **Update `tntc cluster profile` references** to note it requires a running MCP
  server.
- **Update any phase docs** in `phases/` that reference removed or changed
  commands.
- **Update `references/`** if there are command reference docs.

## Acceptance Criteria

- No references to `tntc cluster install` remain in SKILL.md or phase docs.
- Agent promotion workflow pattern is documented (deploy dev -> verify -> deploy prod).
- Environment config examples show per-env MCP settings.
- All phase docs reflect the current CLI command set.

## Non-goals

- Rewriting the skill from scratch -- only update affected sections.
- Adding new skill phases or capabilities beyond what changed.
