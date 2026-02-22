# tentacular-skill

OpenClaw agent skill for [Tentacular](https://github.com/randybias/tentacular) — a security-first, agent-centric, DAG-based workflow builder and runner for Kubernetes.

## What this is

This skill teaches an AI agent (running inside [OpenClaw](https://openclaw.ai)) how to use the `tntc` CLI to build, test, and deploy TypeScript workflow DAGs on Kubernetes.

## Install

```sh
clawhub install tentacular-skill
```

Or manually copy into your OpenClaw workspace:

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

## License

MIT — Copyright (c) 2026 Mirantis, Inc. See [LICENSE](LICENSE).
