## ADDED Requirements

### Requirement: All annotation references use tentacular.io namespace
All skill documents SHALL use `tentacular.io/*` annotation prefix instead of `tentacular.dev/*`.

#### Scenario: workflow-spec.md uses new annotations
- **WHEN** an agent reads `references/workflow-spec.md`
- **THEN** all annotation references SHALL use `tentacular.io/` prefix

#### Scenario: mcp-tools.md uses new annotations
- **WHEN** an agent reads `references/mcp-tools.md`
- **THEN** all annotation references SHALL use `tentacular.io/` prefix

#### Scenario: SKILL.md uses new annotations
- **WHEN** an agent reads SKILL.md
- **THEN** all annotation references SHALL use `tentacular.io/` prefix

### Requirement: Migration note in SKILL.md
SKILL.md SHALL include a note that `tentacular.dev/*` annotations are deprecated and replaced by `tentacular.io/*`.

#### Scenario: Agent aware of migration
- **WHEN** an agent reads SKILL.md
- **THEN** the agent SHALL find a note warning that old tentacular.dev annotations are no longer used
