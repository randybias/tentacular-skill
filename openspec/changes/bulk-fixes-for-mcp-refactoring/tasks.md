## 1. Dependency Preflight Guidance

- [ ] 1.1 Add dependency preflight validation section to SKILL.md deployment phase
- [ ] 1.2 Document how to check external service connectivity before deploy

## 2. Environment References

- [ ] 2.1 Find and fix all hardcoded `--env dev` references
- [ ] 2.2 Replace with environment-agnostic patterns and explain `--env` flag

## 3. Trigger Pod NetworkPolicy

- [ ] 3.1 Document that CronJob trigger pods need egress NetworkPolicy
- [ ] 3.2 Include example NetworkPolicy manifest for reference

## 4. MCP Tools Documentation

- [ ] 4.1 Update MCP tools reference to current signatures
- [ ] 4.2 Remove references to deprecated tools
- [ ] 4.3 Add documentation for new tools from the refactoring

## 5. K8s API References

- [ ] 5.1 Find stale K8s API references
- [ ] 5.2 Update to current API endpoints and resource names

## 6. Verification

- [ ] 6.1 Review all documentation changes for accuracy
- [ ] 6.2 Cross-reference with current CLI and MCP server behavior
