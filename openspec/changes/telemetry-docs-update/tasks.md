## 1. SKILL.md Updates

- [ ] 1.1 Update Architecture section (line ~87) to mention `GET /health?detail=1` telemetry endpoint in engine description
- [ ] 1.2 Update MCP tool count from 29 to 31 (line ~201)
- [ ] 1.3 Add "Workflow Health" subsection after "Workflow Observability" (after line ~340) with wf_health parameter table, return types, and G/A/R status definitions
- [ ] 1.4 Add wf_health_ns documentation with parameter table, return types, truncation note (max 20 workflows)

## 2. Deployment Guide Updates

- [ ] 2.1 Add "Health Monitoring" subsection to `references/deployment-guide.md` after post-deploy verification section (after line ~121) with MCP tool usage examples and G/A/R definitions

## 3. Phase 05 Updates

- [ ] 3.1 Add "Post-deploy health check" step to `phases/05-test-and-deploy.md` after post-deploy verification (after line ~121) instructing agents to use wf_health and verify green status

## 4. Verification

- [ ] 4.1 Review all changes for consistent G/A/R terminology and formatting
- [ ] 4.2 Verify no broken markdown links or table formatting issues
