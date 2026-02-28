## ADDED Requirements

### Requirement: Workflow Listing Report format
SKILL.md SHALL define a standard Workflow Listing Report format that includes a table with columns: Name, Namespace, Status (emoji + text), and Reason. The status column SHALL use green_circle/yellow_circle/red_circle emoji alongside GREEN/AMBER/RED text.

#### Scenario: Listing report format documented
- **WHEN** the agent reads the standard reports section
- **THEN** it SHALL find a Workflow Listing Report template with Name, Namespace, Status, and Reason columns

#### Scenario: Emoji indicators defined
- **WHEN** the agent reads the report format
- **THEN** it SHALL find that GREEN uses green_circle, AMBER uses yellow_circle, and RED uses red_circle emoji

### Requirement: Workflow Detail Report format
SKILL.md SHALL define a standard Workflow Detail Report format for single-workflow deep dives that includes: workflow name, namespace, G/A/R status with emoji, reason (if non-green), pod readiness, and optionally telemetry detail (event counts, error rate, uptime, last error).

#### Scenario: Detail report format documented
- **WHEN** the agent reads the standard reports section
- **THEN** it SHALL find a Workflow Detail Report template with all required fields

#### Scenario: Telemetry detail included when available
- **WHEN** the detail report is generated from a wf_health call with detail=true
- **THEN** the report SHALL include telemetry fields: totalEvents, errorCount, errorRate, uptimeMs, lastError
