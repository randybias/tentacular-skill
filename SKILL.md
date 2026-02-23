---
name: tentacular
description: Use when installing the tntc CLI, configuring target Kubernetes
  environments, generating cluster environment profiles, creating or modifying
  TypeScript DAG workflows, testing workflow nodes, or deploying workflows to a
  cluster. Also use when a workflow is broken, nodes are not passing data, or a
  cluster profile needs refreshing.
---

# Tentacular

Tentacular runs TypeScript DAG workflows in Kubernetes with gVisor sandboxing,
NetworkPolicy egress contracts, and secret isolation. The `tntc` CLI manages the
full lifecycle.

**Find your phase below. Read only the phase file you need.**

## Lifecycle

```
Is tntc installed and working? (run: tntc version)
  NO  → REQUIRED: phases/01-install.md
  YES ↓

Environment(s) configured in ~/.tentacular/config.yaml?
  NO  → REQUIRED: phases/02-configure.md
  YES ↓

Fresh cluster profile for target env? (generatedAt < 7 days)
  NO  → REQUIRED: phases/03-profile.md
  YES ↓

Building or modifying a workflow?
  YES → REQUIRED: phases/04-build.md

Testing or deploying an existing workflow?
  YES → REQUIRED: phases/05-test-and-deploy.md
```

## Iron Laws

```
1. tntc must be installed and verified before any other work.
2. A fresh cluster profile must exist before designing any workflow.
3. Every node must receive data, transform it, and return it — no performative nodes, ever.
4. Every node must be individually tested and its output verified before writing the next.
```

Violating any Iron Law means starting that phase over, not patching around it.

## References (load only when needed)

| Topic | File |
|-------|------|
| CLI flags and commands | references/cli.md |
| Contract model and dependencies | references/contract.md |
| Workflow YAML spec | references/workflow-spec.md |
| Node Context API | references/node-development.md |
| Testing fixtures and patterns | references/testing-guide.md |
| Build, deploy, environment config | references/deployment-guide.md |
| Agent planning and E2E cycle | references/agent-workflow.md |
| Performative node definition and data flow | references/node-data-flow.md |
