## ADDED Requirements

### Requirement: SKILL.md is a lean dispatch core
SKILL.md SHALL be no more than 350 lines (target ~280) and SHALL serve as a dispatch core with summary sections and reference pointers rather than containing full documentation inline.

#### Scenario: Line count within budget
- **WHEN** SKILL.md is measured with `wc -l`
- **THEN** it SHALL report no more than 350 lines

#### Scenario: Frontmatter preserved
- **WHEN** the agent reads SKILL.md
- **THEN** the YAML frontmatter block (name, description) SHALL be present at the top of the file

### Requirement: Common mistakes table
SKILL.md SHALL include a "Common Mistakes" table near the top that lists the most frequent agent errors and their fixes.

#### Scenario: Table present and populated
- **WHEN** the agent reads the Common Mistakes section
- **THEN** it SHALL find a table with at least 5 rows, each containing a mistake description and a fix/correction

### Requirement: CLI vs MCP routing section
SKILL.md SHALL include a concise CLI vs MCP routing section that tells the agent when to use `tntc` CLI commands vs MCP tools.

#### Scenario: Routing guidance present
- **WHEN** the agent reads the CLI vs MCP section
- **THEN** it SHALL find clear rules for when to use CLI (local operations: scaffold, validate, dev, test, build) vs MCP (cluster operations: deploy, status, logs, health)

### Requirement: Tool safety classification table
SKILL.md SHALL include a tool safety classification table with three tiers: read-only, state-changing, and destructive.

#### Scenario: All tiers present
- **WHEN** the agent reads the tool safety table
- **THEN** it SHALL find three categories (read-only, state-changing, destructive) with tool names listed under each

#### Scenario: Every MCP tool classified
- **WHEN** the agent cross-references the safety table against the MCP tools in references/mcp-tools.md
- **THEN** every tool in mcp-tools.md SHALL appear in exactly one safety tier

### Requirement: Schema introspection hint
SKILL.md SHALL include a short section (no more than 10 lines) advising agents to use wf_describe and cluster_profile for dynamic capability discovery.

#### Scenario: Introspection hint present
- **WHEN** the agent reads SKILL.md
- **THEN** it SHALL find guidance to use wf_describe for workflow schema discovery and cluster_profile for cluster capability discovery

### Requirement: Pipeline phases with gates
SKILL.md SHALL list all 5 pipeline phases (01-install through 05-test-and-deploy) with a one-line summary and a verifiable gate condition for each.

#### Scenario: All phases listed
- **WHEN** the agent reads the Pipeline Phases section
- **THEN** it SHALL find entries for phases 01 through 05 with file paths to phases/*.md

#### Scenario: Gates are verifiable
- **WHEN** the agent reads a phase gate condition
- **THEN** the gate SHALL be a concrete, verifiable condition (e.g., a command that succeeds, a file that exists, a tool call that returns expected output)

### Requirement: Workflow creation inversion pattern
SKILL.md SHALL document the workflow creation order as: (1) define contract, (2) derive workflow.yaml, (3) implement nodes. It SHALL explicitly state that writing nodes before the contract is an anti-pattern.

#### Scenario: Inversion pattern documented
- **WHEN** the agent reads the Workflow Creation section
- **THEN** it SHALL find the three-step order with the contract-first requirement explicit

#### Scenario: Anti-pattern called out
- **WHEN** the agent reads the Workflow Creation section
- **THEN** it SHALL find an explicit warning against implementing nodes before defining the contract

### Requirement: Error recovery index
SKILL.md SHALL include a compact error recovery index as a table mapping symptoms to fixes, with a pointer to references/error-recovery.md for full playbooks.

#### Scenario: Index table present
- **WHEN** the agent reads the Error Recovery section
- **THEN** it SHALL find a table with at least 8 rows mapping error symptoms to quick fixes

#### Scenario: Reference pointer present
- **WHEN** the agent reads the Error Recovery section
- **THEN** it SHALL find a pointer to references/error-recovery.md for detailed playbooks

### Requirement: Summary sections with Read triggers
Each major section in SKILL.md SHALL have a 3-5 line summary followed by a reference pointer in the format `> Read references/<file>.md for full details`.

#### Scenario: Reference pointers for all 7 files
- **WHEN** the agent reads SKILL.md
- **THEN** it SHALL find Read triggers pointing to all 7 reference files: architecture.md, mcp-tools.md, node-contract.md, workflow-spec.md, contract-model.md, deployment-ops.md, error-recovery.md

### Requirement: References index at end
SKILL.md SHALL end with a references index listing all reference files and phase files with one-line descriptions.

#### Scenario: Index lists all files
- **WHEN** the agent reads the References section at the end of SKILL.md
- **THEN** it SHALL find entries for all 7 reference files and all 5 phase files with descriptions
