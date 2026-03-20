## ADDED Requirements

### Requirement: SKILL.md includes authz summary
SKILL.md SHALL include a brief section explaining the POSIX-like authorization model (owner/group/mode) and how it affects deploy and operations.

#### Scenario: Agent reads SKILL.md and understands authz
- **WHEN** an agent reads SKILL.md
- **THEN** the agent SHALL find a summary explaining that tentacles have owner, group, and mode permissions
- **THEN** the summary SHALL mention bearer-token bypass and OIDC identity requirement

### Requirement: Phase 05 includes permissions workflow
The `phases/05-test-and-deploy.md` document SHALL include instructions for managing permissions during and after deployment.

#### Scenario: Deploy with group flag documented
- **WHEN** an agent reads the deploy instructions in phase 05
- **THEN** the instructions SHALL show `tntc deploy --group <group>` and `tntc deploy --share` flags

#### Scenario: Post-deploy permissions management documented
- **WHEN** an agent reads phase 05
- **THEN** the instructions SHALL show how to use `tntc permissions get`, `tntc permissions chmod`, and `tntc permissions chgrp`
