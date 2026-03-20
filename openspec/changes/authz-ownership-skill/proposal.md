## Why

The agent skill (SKILL.md and reference docs) teaches AI agents how to use Tentacular. With the new authz-ownership feature, agents need to understand permissions, new MCP tools, new CLI commands, and the annotation migration. Without skill updates, agents will use outdated annotation names and won't know about permissions management.

## What Changes

- Update `references/workflow-spec.md` with new annotation namespace and authz annotations
- Update `references/contract-model.md` if it references annotations
- Update `references/mcp-tools.md` with permissions_get and permissions_set tools
- Update `phases/05-test-and-deploy.md` with --group and --share deploy flags and permissions commands
- Update `SKILL.md` with authz awareness (permission model summary, new commands)
- **BREAKING**: All `tentacular.dev/*` annotation references change to `tentacular.io/*`

## Capabilities

### New Capabilities
- `skill-authz-awareness`: Updates to SKILL.md and phase docs teaching agents about the authorization model, permissions commands, and deploy flags
- `skill-tool-reference`: Updates to MCP tool reference adding permissions_get/permissions_set
- `skill-annotation-migration`: Update all annotation references from tentacular.dev to tentacular.io across skill docs

### Modified Capabilities
<!-- No existing OpenSpec capabilities to modify -->

## Impact

- **SKILL.md**: Main skill file gains authz section
- **references/mcp-tools.md**: New permissions tool group
- **references/workflow-spec.md**: Annotation namespace changes
- **phases/05-test-and-deploy.md**: New deploy flags and permissions workflow
- **Cross-repo dependency**: Content depends on finalized tool schemas and CLI flags from tentacular-mcp and tentacular
