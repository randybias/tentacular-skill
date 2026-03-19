## ADDED Requirements

### Requirement: Error recovery index in SKILL.md
SKILL.md SHALL contain a compact error recovery index table that maps error symptoms to quick-fix actions, serving as a triage tool before the agent reads the full playbooks.

#### Scenario: Index covers deployment errors
- **WHEN** the agent reads the error recovery index
- **THEN** it SHALL find rows for: ImagePullBackOff, CrashLoopBackOff, health probe timeout, MCP unreachable

#### Scenario: Index covers workflow execution errors
- **WHEN** the agent reads the error recovery index
- **THEN** it SHALL find rows for: node execution failed, dependency not found, contract validation failed, workflow stuck in pending

#### Scenario: Index covers operational errors
- **WHEN** the agent reads the error recovery index
- **THEN** it SHALL find rows for: namespace quota exceeded, RBAC permission denied, NetworkPolicy blocking, gVisor runtime class missing

#### Scenario: Each row has symptom and fix
- **WHEN** the agent reads any row in the error recovery index
- **THEN** it SHALL find a symptom description and a concise fix action (1-2 sentences)

### Requirement: Full playbooks in references/error-recovery.md
`references/error-recovery.md` SHALL contain detailed recovery playbooks for each error in the SKILL.md index, plus additional edge cases.

#### Scenario: ImagePullBackOff playbook
- **WHEN** the agent reads the ImagePullBackOff playbook
- **THEN** it SHALL find: (1) diagnostic: check pod events with wf_events, (2) common causes: wrong image tag, private registry without credentials, image does not exist, (3) recovery: fix image reference in workflow.yaml and re-deploy with wf_apply

#### Scenario: CrashLoopBackOff playbook
- **WHEN** the agent reads the CrashLoopBackOff playbook
- **THEN** it SHALL find: (1) diagnostic: check logs with wf_logs, check pod status with wf_pods, (2) common causes: missing env vars, bad config, dependency crash, (3) recovery: fix the root cause and restart with wf_restart

#### Scenario: MCP connection error playbook
- **WHEN** the agent reads the MCP connection error playbook
- **THEN** it SHALL find: (1) diagnostic: check proxy_status, verify MCP endpoint config, (2) common causes: MCP server not installed, wrong endpoint, auth token expired, (3) recovery: verify Helm install, update config, rotate token with cred_rotate

#### Scenario: Contract validation failure playbook
- **WHEN** the agent reads the contract validation failure playbook
- **THEN** it SHALL find: (1) diagnostic: run tntc validate, check contract fields, (2) common causes: missing required fields, invalid dependency declaration, conflicting network rules, (3) recovery: fix contract and re-validate before deploy

#### Scenario: Playbook cross-references MCP tools
- **WHEN** a playbook references diagnostic steps
- **THEN** it SHALL reference specific MCP tools by name (e.g., wf_logs, wf_events, wf_pods, wf_health) for the agent to invoke
