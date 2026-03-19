## ADDED Requirements

### Requirement: references/ directory with 7 files
The `references/` directory SHALL contain exactly 7 Markdown files: architecture.md, mcp-tools.md, node-contract.md, workflow-spec.md, contract-model.md, deployment-ops.md, error-recovery.md.

#### Scenario: All files present
- **WHEN** the references/ directory is listed
- **THEN** it SHALL contain exactly these 7 files and no others

### Requirement: architecture.md content
`references/architecture.md` SHALL contain the system architecture overview extracted from SKILL.md lines ~57-107, covering the three components (Go CLI, MCP Server, Deno Engine), key benefits, and security model overview.

#### Scenario: Three components documented
- **WHEN** the agent reads references/architecture.md
- **THEN** it SHALL find documentation for the Go CLI (data plane), MCP Server (control plane), and Deno/TypeScript Engine

#### Scenario: Security overview present
- **WHEN** the agent reads references/architecture.md
- **THEN** it SHALL find gVisor sandboxing and NetworkPolicy generation described

### Requirement: mcp-tools.md content
`references/mcp-tools.md` SHALL contain the full MCP tool catalog extracted from SKILL.md, organized by category, with parameter tables, return value descriptions, and tool safety classifications aligned with MCP ToolAnnotations.

#### Scenario: All tool categories present
- **WHEN** the agent reads references/mcp-tools.md
- **THEN** it SHALL find sections for: Workflow Discovery, Workflow Execution, Workflow Lifecycle, Workflow Observability, Workflow Health, Namespace Management, Credentials, Cluster Operations, Cluster Health, Security Audit, gVisor Runtime Sandbox, Exoskeleton

#### Scenario: Tool safety annotations present
- **WHEN** the agent reads a tool entry in mcp-tools.md
- **THEN** each tool SHALL have a safety classification label (read-only, state-changing, or destructive)

#### Scenario: Parameter tables preserved
- **WHEN** the agent reads any tool entry
- **THEN** it SHALL find a parameter table listing parameter name, type, required/optional status, and description

### Requirement: node-contract.md content
`references/node-contract.md` SHALL contain the workflow node behavioral contract, including the Context API, dependency access patterns, auth patterns, and node interface specification.

#### Scenario: Context API documented
- **WHEN** the agent reads references/node-contract.md
- **THEN** it SHALL find the Context object API with methods for dependency resolution, config, secrets, fetch, and logging

#### Scenario: Node interface specification present
- **WHEN** the agent reads references/node-contract.md
- **THEN** it SHALL find the TypeScript interface that nodes must implement (default export function signature)

### Requirement: workflow-spec.md content
`references/workflow-spec.md` SHALL contain the complete workflow.yaml specification including all fields, config block format, minimal workflow example, and trigger types.

#### Scenario: Config block documented
- **WHEN** the agent reads references/workflow-spec.md
- **THEN** it SHALL find the config block format with all supported fields

#### Scenario: Minimal example present
- **WHEN** the agent reads references/workflow-spec.md
- **THEN** it SHALL find a minimal workflow.yaml example that is valid and self-contained

### Requirement: contract-model.md content
`references/contract-model.md` SHALL contain the contract model and validation rules, including the tentacular dependency system, anti-confusion rules, deploy decision tree, and known limitations.

#### Scenario: Dependency types documented
- **WHEN** the agent reads references/contract-model.md
- **THEN** it SHALL find documentation for tentacular-* dependencies vs manual dependencies, with clear rules for when to use each

#### Scenario: Deploy decision tree present
- **WHEN** the agent reads references/contract-model.md
- **THEN** it SHALL find the deploy decision tree that guides whether to use tntc deploy vs manual kubectl

### Requirement: deployment-ops.md content
`references/deployment-ops.md` SHALL contain deployment and operations guidance including common workflow, deployment flow, environment promotion, dependency preflight, environment configuration, and namespace conventions.

#### Scenario: Promotion steps documented
- **WHEN** the agent reads references/deployment-ops.md
- **THEN** it SHALL find environment promotion steps (dev -> prod) with health verification at each stage

#### Scenario: MCP configuration documented
- **WHEN** the agent reads references/deployment-ops.md
- **THEN** it SHALL find MCP endpoint configuration format and namespace convention guidance

### Requirement: error-recovery.md content (new)
`references/error-recovery.md` SHALL contain error recovery playbooks covering common failure modes with diagnostic steps, root causes, and recovery procedures.

#### Scenario: Playbooks cover deployment failures
- **WHEN** the agent reads references/error-recovery.md
- **THEN** it SHALL find playbooks for: image pull errors, pod crash loops, health probe failures, MCP connection errors

#### Scenario: Playbooks cover workflow execution failures
- **WHEN** the agent reads references/error-recovery.md
- **THEN** it SHALL find playbooks for: node execution errors, dependency resolution failures, timeout errors, contract validation failures

#### Scenario: Each playbook has consistent structure
- **WHEN** the agent reads any playbook in error-recovery.md
- **THEN** it SHALL find: Symptom, Diagnostic Steps, Root Cause, Recovery Procedure

## MODIFIED Requirements

### Requirement: AGENTS.md references listing updated
AGENTS.md SHALL list the 7 new reference files under the `references/` directory in its Project Structure section, replacing the previous aspirational listing.

#### Scenario: New files listed
- **WHEN** the agent reads AGENTS.md Project Structure
- **THEN** the references/ listing SHALL show: architecture.md, mcp-tools.md, node-contract.md, workflow-spec.md, contract-model.md, deployment-ops.md, error-recovery.md

#### Scenario: Old files removed
- **WHEN** the agent reads AGENTS.md Project Structure
- **THEN** it SHALL NOT list the previously planned files (agent-workflow.md, cli.md, contract.md, deployment-guide.md, node-data-flow.md, node-development.md, testing-guide.md)

### Requirement: README.md updated
README.md SHALL reflect the new directory structure with the 7 reference files listed.

#### Scenario: References listed in README
- **WHEN** the agent reads README.md
- **THEN** it SHALL find the 7 reference files listed with one-line descriptions
