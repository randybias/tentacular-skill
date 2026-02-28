## ADDED Requirements

### Requirement: SKILL.md architecture section mentions telemetry
The SKILL.md Architecture section SHALL mention the engine's `GET /health?detail=1` telemetry endpoint alongside existing HTTP triggers.

#### Scenario: Architecture description includes telemetry
- **WHEN** an agent reads the Architecture section of SKILL.md
- **THEN** the engine description SHALL mention `GET /health?detail=1` for telemetry alongside `POST /run` and `GET /health`

### Requirement: SKILL.md documents wf_health tool
The SKILL.md MCP Tools Reference section SHALL include a "Workflow Health" subsection documenting the `wf_health` tool with a parameter table and return type description.

#### Scenario: wf_health parameter table present
- **WHEN** an agent reads the Workflow Health subsection of SKILL.md
- **THEN** the documentation SHALL include a parameter table with `namespace` (string, required) and `name` (string, required) parameters

#### Scenario: wf_health return type documented
- **WHEN** an agent reads the wf_health documentation
- **THEN** the returns description SHALL list: `name`, `namespace`, `status` (green/amber/red), `ready`, `total_runs`, `in_flight`, `recent_runs`, `error`

#### Scenario: G/A/R status values defined
- **WHEN** an agent reads the wf_health documentation
- **THEN** the documentation SHALL define Green (ready + last run succeeded or no runs), Amber (last run failed or run in-flight >5min), and Red (0 replicas or unreachable)

### Requirement: SKILL.md documents wf_health_ns tool
The SKILL.md MCP Tools Reference section SHALL include documentation for the `wf_health_ns` tool with a parameter table and return type description.

#### Scenario: wf_health_ns parameter table present
- **WHEN** an agent reads the wf_health_ns documentation
- **THEN** the documentation SHALL include a parameter table with `namespace` (string, required)

#### Scenario: wf_health_ns return type documented
- **WHEN** an agent reads the wf_health_ns documentation
- **THEN** the returns description SHALL list: `namespace`, `workflows` (array of name + status + ready), `truncated`, `limit`, and note the max 20 workflows per call

### Requirement: SKILL.md tool count updated
The SKILL.md MCP Tools Reference introduction SHALL reflect the updated tool count including the two new health tools.

#### Scenario: Tool count is 31
- **WHEN** an agent reads the MCP Tools Reference introduction
- **THEN** the text SHALL state 31 tools (updated from 29)

### Requirement: Deployment guide includes health monitoring
The `references/deployment-guide.md` SHALL include a "Health Monitoring" subsection after the post-deploy verification section documenting how to use wf_health and wf_health_ns for post-deploy monitoring.

#### Scenario: Health monitoring section present
- **WHEN** an agent reads the deployment guide
- **THEN** a "Health Monitoring" subsection SHALL exist after post-deploy verification with MCP tool usage examples and G/A/R status definitions

### Requirement: Phase 05 includes post-deploy health check
The `phases/05-test-and-deploy.md` SHALL include a "Post-deploy health check" step after the post-deploy verification section instructing agents to verify workflow health status.

#### Scenario: Health check step present
- **WHEN** an agent reads phase 05
- **THEN** a "Post-deploy health check" step SHALL exist instructing the agent to use `wf_health` and verify the status is "green"
