# tentacular-skill

Agent skill for [Tentacular](https://github.com/randybias/tentacular) — teaches AI agents (Claude Code, Codex, Gemini) to build, test, and deploy secure TypeScript workflow DAGs on Kubernetes.

## Documentation

Full documentation: **[randybias.github.io/tentacular-docs](https://randybias.github.io/tentacular-docs)** — see [Agent Skill](https://randybias.github.io/tentacular-docs/concepts/agent-skill/) for an overview of what the skill does and how agents use it.

`SKILL.md` in this repo is the canonical agent instruction set and is linked from, not duplicated in, the documentation site.

## What this is

This skill teaches an AI agent how to use the Tentacular CLI and MCP tools to build, test, and deploy secure TypeScript workflow DAGs on Kubernetes.

## Install

Manually copy into your OpenClaw workspace:

```sh
cp -r . ~/.openclaw/workspace/skills/tentacular
```

## Requirements

- [tntc](https://github.com/randybias/tentacular) CLI installed (`which tntc`)
- If not installed, the skill will run the installer automatically:
  ```sh
  curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
  ```

## Contents

- [`SKILL.md`](SKILL.md) — skill entrypoint loaded by OpenClaw
- [`references/`](references/) — detailed documentation on CLI, contracts, deployment, testing, and more

## Versioning

Skill versions track CLI versions. Skill `v0.1.x` documents CLI `v0.1.x` features.

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [tentacular](https://github.com/randybias/tentacular) | Go CLI (`tntc`) + Deno workflow engine |
| [tentacular-mcp](https://github.com/randybias/tentacular-mcp) | In-cluster MCP server (Helm chart, 32 tools) |
| [tentacular-catalog](https://github.com/randybias/tentacular-catalog) | Workflow template catalog |

## License

Copyright (c) 2025-2026 Mirantis, Inc. All rights reserved. See [LICENSE](LICENSE).
