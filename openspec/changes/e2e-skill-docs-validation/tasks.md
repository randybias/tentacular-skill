## 1. wf_health Documentation Validation

- [ ] 1.1 Compare SKILL.md wf_health parameter table against WfHealthParams struct in tentacular-mcp pkg/tools/wf_health.go: verify namespace, name, detail fields match in name, type, and required/optional status
- [ ] 1.2 Verify SKILL.md documents detail parameter default as false
- [ ] 1.3 Compare SKILL.md wf_health_ns parameter table against WfHealthNsParams struct: verify namespace, limit fields match
- [ ] 1.4 Verify SKILL.md documents default limit as 20

## 2. G/A/R Model Validation

- [ ] 2.1 Compare SKILL.md GREEN classification description against classifyFromDetail logic in wf_health.go
- [ ] 2.2 Compare SKILL.md AMBER classification triggers (last_status failed, in_flight true) against classifyFromDetail substring matching
- [ ] 2.3 Compare SKILL.md RED classification triggers (pod not ready, endpoint unreachable) against handleWfHealth logic
- [ ] 2.4 Verify SKILL.md G/A/R table has all three rows with accurate descriptions

## 3. Standard Report Format Validation

- [ ] 3.1 Verify Workflow Listing Report template includes Name, Namespace, Status, Reason columns matching WfHealthNsEntry fields
- [ ] 3.2 Verify Workflow Listing Report shows examples for all three G/A/R states
- [ ] 3.3 Verify Workflow Detail Report uses correct emoji indicators (green/yellow/red circles)
- [ ] 3.4 Verify Workflow Detail Report includes optional telemetry section for detail=true responses

## 4. Progressive Disclosure Validation

- [ ] 4.1 Verify SKILL.md documents three-tier progressive disclosure: wf_health_ns -> wf_health -> wf_health with detail=true
- [ ] 4.2 Verify each tier references the correct tool name
- [ ] 4.3 Verify each tier lists the minimum required parameters

## 5. Cross-Reference Validation

- [ ] 5.1 Verify references/deployment-guide.md references wf_health and wf_health_ns by exact tool names
- [ ] 5.2 Verify phases/05-test-and-deploy.md includes health verification step with correct tool reference
- [ ] 5.3 Verify no stale tool names or deprecated parameter names exist across all documentation files
