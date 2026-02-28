## ADDED Requirements

### Requirement: wf_health tool documentation in SKILL.md
SKILL.md SHALL document the wf_health MCP tool with its parameter table (namespace, name, detail), return value description, G/A/R classification model, and at least one usage example.

#### Scenario: Agent discovers wf_health tool
- **WHEN** the agent reads SKILL.md
- **THEN** it SHALL find documentation for wf_health including parameters, return format, and G/A/R status meanings

#### Scenario: Parameter table present
- **WHEN** the agent reads the wf_health documentation
- **THEN** it SHALL find a table listing namespace (required), name (required), and detail (optional, default false) parameters

### Requirement: wf_health_ns tool documentation in SKILL.md
SKILL.md SHALL document the wf_health_ns MCP tool with its parameter table (namespace, limit), return value description including summary counts and truncation behavior, and at least one usage example.

#### Scenario: Agent discovers wf_health_ns tool
- **WHEN** the agent reads SKILL.md
- **THEN** it SHALL find documentation for wf_health_ns including parameters, return format, and summary count fields

#### Scenario: Limit parameter documented
- **WHEN** the agent reads the wf_health_ns documentation
- **THEN** it SHALL find that the limit parameter defaults to 20 and controls maximum workflows checked

### Requirement: G/A/R health model documentation
SKILL.md SHALL include a reference section explaining the Green/Amber/Red health classification model with clear definitions of each status level and the conditions that trigger them.

#### Scenario: G/A/R definitions present
- **WHEN** the agent reads the health model reference
- **THEN** it SHALL find: GREEN = pod ready + health endpoint OK + no AMBER signals, AMBER = last execution failed or execution in flight, RED = pod not ready or health endpoint unreachable

### Requirement: Progressive disclosure guidance
SKILL.md SHALL include a workflow pattern for health monitoring that guides the agent through progressive escalation: single workflow check, namespace scan, detail mode for debugging.

#### Scenario: Workflow pattern documented
- **WHEN** the agent reads the progressive disclosure section
- **THEN** it SHALL find a step-by-step pattern: (1) wf_health for single workflow, (2) wf_health_ns for namespace overview, (3) wf_health with detail=true for deep investigation

### Requirement: Deployment guide health monitoring section
references/deployment-guide.md SHALL include a health monitoring section covering how to verify workflow health after deployment using wf_health.

#### Scenario: Post-deploy health check documented
- **WHEN** the agent reads the deployment guide
- **THEN** it SHALL find a section on verifying deployment health using wf_health after tntc deploy

### Requirement: CLI reference health commands
references/cli.md SHALL document health-related MCP tool invocations alongside existing CLI commands.

#### Scenario: Health tools in CLI reference
- **WHEN** the agent reads the CLI reference
- **THEN** it SHALL find wf_health and wf_health_ns listed as MCP tools available for health monitoring

### Requirement: Phase 05 health verification step
phases/05-test-and-deploy.md SHALL include a health verification step after the deploy step, instructing the agent to use wf_health to confirm the deployed workflow is healthy.

#### Scenario: Health check in deploy phase
- **WHEN** the agent follows the phase 05 workflow
- **THEN** it SHALL find a step after deployment to run wf_health and verify GREEN status
