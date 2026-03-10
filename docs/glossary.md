# Glossary

Working glossary of Tentacular-specific terms. Will be unified into a central glossary when documentation moves to a dedicated repo.

| Term | Definition |
|------|-----------|
| **Tentacle** | A deployed workflow managed by Tentacular. |
| **Skill** | The instruction set that teaches AI agents how to use `tntc` and MCP tools to build, test, and deploy tentacles. |
| **Exoskeleton** | Optional per-tentacle workspace provisioning for Postgres, NATS, RustFS, and SPIRE. |
| **Workspace** | The scoped backing-service resources for a tentacle. Declared via `tentacular-*` contract dependencies. |
| **MCP tool** | A tool exposed by the MCP server via the Model Context Protocol. Agents call these to manage clusters and workflows. |
| **Contract** | Workflow dependency declarations that drive NetworkPolicy, Deno permissions, and exoskeleton provisioning. |
| **Deploy gate** | Pre-deployment checks (contract validation, secret availability, drift detection) before `tntc deploy` proceeds. |
| **Deployer provenance** | Annotations on deployed resources identifying who deployed and when. Requires SSO authentication. |
