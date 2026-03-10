# Tentacular Skill — Agent Instructions

Agent skill definition that teaches AI agents how to use the Tentacular CLI (`tntc`) and MCP tools to build, test, and deploy secure TypeScript workflow DAGs on Kubernetes. Part of the Tentacular platform — a security-first, agent-centric, DAG-based workflow builder and runner for Kubernetes.

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [tentacular](https://github.com/randybias/tentacular) | Go CLI (`tntc`) + Deno workflow engine |
| [tentacular-mcp](https://github.com/randybias/tentacular-mcp) | In-cluster MCP server (Go, Helm chart) |
| [tentacular-skill](https://github.com/randybias/tentacular-skill) | Agent skill definition (this repo) |
| [tentacular-catalog](https://github.com/randybias/tentacular-catalog) | Workflow template catalog (TypeScript/Deno) |

## Project Structure

This is a documentation-only repo with no build or test infrastructure.

- `SKILL.md` — skill entry point, loaded by agents
- `phases/` — step-by-step workflow phases:
  - `01-install.md` — installation
  - `02-configure.md` — setup and configuration
  - `03-profile.md` — cluster understanding
  - `04-build.md` — workflow development
  - `05-test-and-deploy.md` — testing and deployment
- `references/` — detailed reference documentation:
  - `agent-workflow.md` — agent interaction patterns
  - `cli.md` — CLI command reference
  - `contract.md` — node contract specification
  - `deployment-guide.md` — deployment procedures
  - `node-data-flow.md` — data flow between nodes
  - `node-development.md` — writing workflow nodes
  - `testing-guide.md` — testing workflows
  - `workflow-spec.md` — workflow.yaml specification

## When to Update

This skill must stay in sync with the CLI and MCP server. Update when:

- **New CLI commands** are added or changed in `tentacular`
- **New MCP tools** are added or changed in `tentacular-mcp`
- **Node contract changes** in `tentacular/engine/`
- **Workflow spec changes** in `tentacular/pkg/spec/`
- **Security model changes** that affect how agents interact with clusters

## Versioning

Skill versions track CLI versions. Skill `v0.X.Y` documents CLI `v0.X.Y` features.

All four repos use **lockstep versioning** — they are tagged with the same version number for every release, even if a repo has no changes. Tags use semantic versioning: `vMAJOR.MINOR.PATCH`.

## Commit Messages

All repos use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
feat: document new catalog commands
fix: correct node contract examples
docs: update deployment guide for gVisor changes
```

## Temporary Files

Use `scratch/` for all temporary files, experiments, and throwaway work. This directory is gitignored. Never place temp files in the project root or alongside source code.

## License

MIT (Mirantis, Inc.)
