# tentacular-skill

Agent skill for [Tentacular](https://github.com/randybias/tentacular) — teaches AI agents (Claude Code, Codex, Gemini) to build, test, and deploy secure TypeScript workflow DAGs on Kubernetes.

## Documentation

Full documentation: **[randybias.github.io/tentacular-docs](https://randybias.github.io/tentacular-docs)** — see [Agent Skill](https://randybias.github.io/tentacular-docs/concepts/agent-skill/) for an overview of what the skill does and how agents use it.

`SKILL.md` in this repo is the canonical agent instruction set and is linked from, not duplicated in, the documentation site.

## What This Is

This skill teaches AI agents how to use the Tentacular CLI (`tntc`) and MCP tools to build, test, and deploy secure TypeScript workflow DAGs on Kubernetes. It covers the full lifecycle: scaffold, validate, test, build, deploy, run, monitor.

## Contents

- [`SKILL.md`](SKILL.md) — skill entrypoint, loaded by agents on activation
- `phases/` — step-by-step workflow phases:
  - `01-install.md` — install tntc CLI and engine
  - `02-configure.md` — configure environments and MCP
  - `03-profile.md` — cluster profiling and Agent Guidance
  - `04-build.md` — design and build workflow nodes
  - `05-test-and-deploy.md` — testing gates and deployment
- `references/` — detailed reference documentation:
  - `architecture.md` — system architecture and components
  - `mcp-tools.md` — tool details, health model, reports
  - `node-contract.md` — node signature, Context API
  - `workflow-spec.md` — workflow.yaml schema and triggers
  - `contract-model.md` — contract deps, exoskeleton, SSO
  - `deployment-ops.md` — deploy flow, promotion, env config
  - `error-recovery.md` — error playbooks and triage

## Requirements

- [tntc](https://github.com/randybias/tentacular) CLI installed (`which tntc`)
- If not installed, the skill will run the installer automatically:
  ```sh
  curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
  ```

## Versioning

Skill versions track CLI versions. Skill `v0.1.x` documents CLI `v0.1.x` features.

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [tentacular](https://github.com/randybias/tentacular) | Go CLI (`tntc`) + Deno workflow engine |
| [tentacular-mcp](https://github.com/randybias/tentacular-mcp) | In-cluster MCP server (Helm chart, 36 tools) |
| [tentacular-scaffolds](https://github.com/randybias/tentacular-scaffolds) | [Scaffold quickstart library](https://randybias.github.io/tentacular-scaffolds) |
| [tentacular-docs](https://github.com/randybias/tentacular-docs) | [Documentation site](https://randybias.github.io/tentacular-docs) |

## License

Copyright (c) 2025-2026 Mirantis, Inc. All rights reserved. See [LICENSE](LICENSE).
