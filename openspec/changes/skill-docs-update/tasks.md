# Tasks

## Implementation

- [ ] Update SKILL.md to remove `tntc cluster install` references
- [ ] Add Helm installation instructions to SKILL.md
- [ ] Add agent promotion workflow pattern to SKILL.md (deploy dev -> verify health -> deploy prod)
- [ ] Update environment configuration examples with per-env MCP config and `default_env`
- [ ] Update `tntc cluster profile` references to note MCP requirement
- [ ] Review and update all files in `phases/` directory for changed commands
- [ ] Review and update all files in `references/` directory if present

## Verification

- [ ] No references to `tntc cluster install` remain in any skill docs
- [ ] Agent promotion workflow pattern is documented (deploy dev -> verify -> deploy prod)
- [ ] Environment config examples include `mcp_endpoint`, `mcp_token_path`, `default_env`
- [ ] All phase docs reflect the current command set
