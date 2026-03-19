## Why

The current SKILL.md is a monolithic ~1,400-line file where 95% of content is irrelevant to any given agent task. There is no progressive disclosure, no error recovery guidance, and anti-patterns are scattered throughout. This directly hurts agent accuracy -- the agent wastes context window on architecture docs when it needs tool syntax, or reads deployment ops when it needs workflow specs. Restructuring into a lean core with on-demand references enables precise, task-relevant context loading.

## What Changes

- **BREAKING** Rewrite SKILL.md from ~1,400 lines to ~280 lines as a lean dispatch core with section summaries and `@references/` pointers
- Extract existing content into 7 reference files under `references/`:
  - `references/architecture.md` -- system architecture overview (from lines 6-210)
  - `references/mcp-tools.md` -- MCP tool catalog with safety classifications (from lines 212-713, trimmed)
  - `references/node-contract.md` -- workflow node behavioral contract (from lines 715-774 + new content)
  - `references/workflow-spec.md` -- workflow specification and patterns (from lines 776-831, 1127-1174)
  - `references/contract-model.md` -- contract model and validation (from lines 833-1097)
  - `references/deployment-ops.md` -- deployment and operations guide (from lines 1176-1365)
  - `references/error-recovery.md` -- entirely new error recovery playbooks
- Wire in existing `phases/` directory from the core SKILL.md
- Add new content: error recovery playbooks, workflow creation inversion pattern, tool safety classification table, schema introspection hints
- Update AGENTS.md with new directory structure listing
- Update README.md with new contents listing
- No backwards compatibility required

## Capabilities

### New Capabilities
- `lean-skill-core`: Rewrite SKILL.md as a ~280-line dispatch core with progressive disclosure via @reference pointers
- `reference-extraction`: Extract monolithic content into 7 focused reference files under references/
- `error-recovery`: New error recovery playbooks covering common failure modes and retry patterns
- `tool-safety-classification`: Tool safety classification table in mcp-tools.md aligned with MCP ToolAnnotations
- `workflow-inversion`: Document workflow creation inversion pattern (schema-first, not template-first)
- `schema-introspection`: Add schema introspection hints so agents can discover tool capabilities dynamically

### Modified Capabilities
<!-- No existing OpenSpec capabilities are being modified -->

## Impact

- **SKILL.md** -- complete rewrite, all existing content relocated to references/
- **references/** -- new directory with 7 reference files
- **phases/** -- no changes, but newly wired into core SKILL.md
- **AGENTS.md** -- updated to reflect new file structure
- **README.md** -- updated contents listing
- **Downstream consumers** -- any agent loading SKILL.md gets a fundamentally different (smaller) document; agents must follow @reference pointers for detail
- **tentacular-mcp dependency** -- the tool safety classification table in references/mcp-tools.md will reference MCP ToolAnnotations being added in the tentacular-mcp repo (separate change: mcp-tool-annotations)
