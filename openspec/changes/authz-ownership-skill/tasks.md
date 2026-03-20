## 1. Annotation Migration

- [ ] 1.1 Update `references/workflow-spec.md`: replace all `tentacular.dev/` with `tentacular.io/`
- [ ] 1.2 Update `references/mcp-tools.md`: replace all `tentacular.dev/` with `tentacular.io/`
- [ ] 1.3 Update `references/contract-model.md`: replace any `tentacular.dev/` references
- [ ] 1.4 Update `SKILL.md`: replace all `tentacular.dev/` with `tentacular.io/`, add deprecation note
- [ ] 1.5 Search all remaining files for `tentacular.dev/` and update

## 2. MCP Tools Reference

- [ ] 2.1 Add Permissions tool group to `references/mcp-tools.md` with permissions_get (parameters, return fields)
- [ ] 2.2 Add permissions_set to the Permissions group (parameters, owner-only constraint)

## 3. Phase 05 Update

- [ ] 3.1 Add --group and --share deploy flags to deploy instructions in `phases/05-test-and-deploy.md`
- [ ] 3.2 Add post-deploy permissions management section (permissions get, chmod, chgrp examples)

## 4. SKILL.md Update

- [ ] 4.1 Add authz summary section to SKILL.md (owner/group/mode model, bearer bypass, OIDC requirement)
- [ ] 4.2 Add new permissions CLI commands to the command reference in SKILL.md
