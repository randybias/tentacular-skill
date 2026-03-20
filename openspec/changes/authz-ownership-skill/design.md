## Context

The tentacular-skill repo contains the agent skill definition -- markdown files that teach AI agents how to build, test, and deploy tentacles. The skill is structured as phases (01-install through 05-test-and-deploy) and references (architecture, contract-model, mcp-tools, workflow-spec). SKILL.md is the main entry point.

## Goals / Non-Goals

**Goals:**
- Update all annotation references to use tentacular.io namespace
- Add permissions awareness to the deploy phase
- Add new MCP tools to the tool reference
- Give agents enough context to manage permissions when deploying tentacles

**Non-Goals:**
- Teaching agents the internals of the authz evaluation algorithm
- Creating a standalone authz phase (permissions management is part of deploy)
- Documenting IdP configuration in the skill

## Decisions

### 1. Add authz to phase 05 rather than creating a new phase

**Rationale:** Permissions management is part of the deploy and operate lifecycle. Adding it to phase 05 (test-and-deploy) keeps the phase count stable and locates authz instructions where agents encounter them naturally.

### 2. Brief authz summary in SKILL.md, details in references

**Rationale:** SKILL.md is the quick-reference. A 3-4 line summary of the permission model with a pointer to the full reference keeps SKILL.md concise.

## Risks / Trade-offs

- **[Docs written before implementation]** Tool schemas and flag names may change during implementation. -> Mitigation: Update skill docs after implementation stabilizes.
- **[Agent confusion during migration]** Agents trained on old skill may use tentacular.dev annotations. -> Mitigation: Include explicit migration note in SKILL.md.
