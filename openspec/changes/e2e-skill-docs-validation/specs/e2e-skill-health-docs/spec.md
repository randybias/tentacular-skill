## ADDED Requirements

### Requirement: wf_health parameter table accuracy
The validation SHALL verify that the SKILL.md wf_health parameter table matches the actual MCP tool schema.

#### Scenario: All parameters documented
- **WHEN** the wf_health tool JSON schema defines parameters (namespace, name, detail)
- **THEN** SKILL.md SHALL list all three parameters with matching names, types, and required/optional status

#### Scenario: Parameter descriptions match tool behavior
- **WHEN** the wf_health tool detail parameter defaults to false
- **THEN** SKILL.md SHALL document the default value as false and describe the parameter as optional

### Requirement: wf_health_ns parameter table accuracy
The validation SHALL verify that the SKILL.md wf_health_ns parameter table matches the actual MCP tool schema.

#### Scenario: All parameters documented
- **WHEN** the wf_health_ns tool JSON schema defines parameters (namespace, limit)
- **THEN** SKILL.md SHALL list both parameters with matching names, types, and required/optional status

#### Scenario: Default limit value documented
- **WHEN** the wf_health_ns tool defaults limit to 20
- **THEN** SKILL.md SHALL document the default limit as 20

### Requirement: G/A/R model documentation accuracy
The validation SHALL verify that the documented G/A/R classification rules match the MCP tool implementation.

#### Scenario: GREEN classification conditions match
- **WHEN** the MCP classifyFromDetail function returns GREEN for pod ready + no failure signals
- **THEN** SKILL.md SHALL document GREEN as "Pod ready, health endpoint reachable, no failure signals"

#### Scenario: AMBER classification conditions match
- **WHEN** the MCP classifyFromDetail function returns AMBER for last_status failed or in_flight true
- **THEN** SKILL.md SHALL document both AMBER trigger conditions

#### Scenario: RED classification conditions match
- **WHEN** the MCP handleWfHealth function returns RED for pod not ready or health probe failure
- **THEN** SKILL.md SHALL document both RED trigger conditions

### Requirement: Cross-reference consistency
The validation SHALL verify that health monitoring references across docs are consistent.

#### Scenario: Deployment guide references correct tool names
- **WHEN** references/deployment-guide.md mentions health monitoring
- **THEN** it SHALL reference wf_health and wf_health_ns by their exact tool names

#### Scenario: Phase 05 references correct health check steps
- **WHEN** phases/05-test-and-deploy.md includes health verification
- **THEN** it SHALL reference the correct tool name and at minimum the namespace parameter
