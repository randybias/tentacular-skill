## ADDED Requirements

### Requirement: Workflow listing report completeness
The validation SHALL verify that the documented workflow listing report template covers all G/A/R states.

#### Scenario: All three status values represented
- **WHEN** the standard Workflow Listing Report format is documented in SKILL.md
- **THEN** the report template SHALL include examples or references for GREEN, AMBER, and RED status values

#### Scenario: Listing report fields match tool response
- **WHEN** wf_health_ns returns entries with name, namespace, status, and reason fields
- **THEN** the documented listing report template SHALL include columns for all four fields

### Requirement: Workflow detail report completeness
The validation SHALL verify that the documented workflow detail report template covers the full wf_health response.

#### Scenario: Detail report includes G/A/R emoji indicators
- **WHEN** the standard Workflow Detail Report is documented in SKILL.md
- **THEN** it SHALL use emoji indicators that match the documented G/A/R model (green circle for GREEN, yellow circle for AMBER, red circle for RED)

#### Scenario: Detail report includes optional telemetry fields
- **WHEN** wf_health is called with detail=true and returns execution telemetry
- **THEN** the documented detail report template SHALL include a section for telemetry data (event counts, error rate, uptime)

### Requirement: Progressive disclosure workflow documented
The validation SHALL verify that the progressive disclosure pattern is documented with correct tool call sequence.

#### Scenario: Three-tier disclosure documented
- **WHEN** the progressive disclosure section is present in SKILL.md
- **THEN** it SHALL document the three tiers: namespace scan (wf_health_ns), single check (wf_health without detail), deep dive (wf_health with detail=true)

#### Scenario: Tool call sequence uses correct parameters
- **WHEN** each tier of the progressive disclosure is documented
- **THEN** each tier SHALL reference the correct tool name and list the minimum required parameters
