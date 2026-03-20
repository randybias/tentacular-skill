## ADDED Requirements

### Requirement: MCP tools reference includes permissions tools
The `references/mcp-tools.md` document SHALL include permissions_get and permissions_set in a Permissions tool group.

#### Scenario: permissions_get documented in skill
- **WHEN** an agent reads the MCP tools reference
- **THEN** permissions_get SHALL be listed with its parameters (namespace, name) and return fields (owner, group, mode)

#### Scenario: permissions_set documented in skill
- **WHEN** an agent reads the MCP tools reference
- **THEN** permissions_set SHALL be listed with its parameters (namespace, name, mode, group) and noted as owner-only
