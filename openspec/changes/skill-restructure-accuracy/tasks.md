## 1. Create references/ directory and extract content

- [ ] 1.1 Create `references/` directory
- [ ] 1.2 Create `references/architecture.md` -- extract system architecture from SKILL.md (lines ~57-107): three components, key benefits, security overview
- [ ] 1.3 Create `references/mcp-tools.md` -- extract full MCP tool catalog from SKILL.md (lines ~212-700): all tool categories with parameter tables, return values, usage examples. Add safety classification label (read-only / state-changing / destructive) to each tool entry
- [ ] 1.4 Create `references/node-contract.md` -- extract node behavioral contract from SKILL.md (lines ~715-774): Context API, dependency access, auth patterns, node interface specification
- [ ] 1.5 Create `references/workflow-spec.md` -- extract workflow specification from SKILL.md (lines ~776-831, 1127-1174): workflow.yaml format, config block, minimal example, trigger types
- [ ] 1.6 Create `references/contract-model.md` -- extract contract model from SKILL.md (lines ~833-1097): tentacular dependencies, anti-confusion rules, deploy decision tree, known limitations
- [ ] 1.7 Create `references/deployment-ops.md` -- extract deployment and operations from SKILL.md (lines ~1176-1365): common workflow, deployment flow, environment promotion, dependency preflight, env config, namespace conventions, MCP config
- [ ] 1.8 Create `references/error-recovery.md` -- author new error recovery playbooks covering: ImagePullBackOff, CrashLoopBackOff, health probe failures, MCP connection errors, node execution failures, dependency resolution failures, contract validation failures, namespace quota, RBAC denied, NetworkPolicy blocking, gVisor runtime class missing. Each playbook: Symptom, Diagnostic Steps (with MCP tool names), Root Cause, Recovery Procedure

## 2. Rewrite SKILL.md as lean dispatch core

- [ ] 2.1 Preserve YAML frontmatter block (name, description)
- [ ] 2.2 Write Prerequisites section (keep existing install/verify commands, keep MCP server prerequisite -- compact to essentials)
- [ ] 2.3 Write Common Mistakes table (at least 5 rows: mistake -> fix)
- [ ] 2.4 Write CLI vs MCP routing section (compact version of existing, ~10-15 lines)
- [ ] 2.5 Write Tool Safety Classification table (3-tier: read-only, state-changing, destructive -- every MCP tool classified)
- [ ] 2.6 Write Schema Introspection hint (5-10 lines: use wf_describe and cluster_profile for dynamic discovery)
- [ ] 2.7 Write Pipeline Phases section with gates (list phases/01-05 with one-line summary and verifiable gate each)
- [ ] 2.8 Write Workflow Creation section with inversion pattern (contract-first, explicit anti-pattern warning)
- [ ] 2.9 Write Architecture summary (3-5 lines) with `> Read references/architecture.md` pointer
- [ ] 2.10 Write MCP Tools summary (3-5 lines) with `> Read references/mcp-tools.md` pointer
- [ ] 2.11 Write Node Contract summary (3-5 lines) with `> Read references/node-contract.md` pointer
- [ ] 2.12 Write Workflow Spec summary (3-5 lines) with `> Read references/workflow-spec.md` pointer
- [ ] 2.13 Write Contract Model summary (3-5 lines) with `> Read references/contract-model.md` pointer
- [ ] 2.14 Write Deployment & Ops summary (3-5 lines) with `> Read references/deployment-ops.md` pointer
- [ ] 2.15 Write Error Recovery index table (at least 8 rows: symptom -> fix) with `> Read references/error-recovery.md` pointer
- [ ] 2.16 Write References index at end of file (list all 7 reference files and 5 phase files with one-line descriptions)
- [ ] 2.17 Verify SKILL.md is under 350 lines with `wc -l`

## 3. Update supporting files

- [ ] 3.1 Update AGENTS.md: replace `references/` listing (remove old 8 files, add new 7 files with descriptions)
- [ ] 3.2 Update README.md: update contents/structure listing to reflect new reference files

## 4. Validation

- [ ] 4.1 Verify all 7 reference files exist in references/
- [ ] 4.2 Verify SKILL.md contains Read triggers pointing to all 7 reference files
- [ ] 4.3 Verify SKILL.md tool safety table covers every tool documented in references/mcp-tools.md
- [ ] 4.4 Verify SKILL.md error recovery index covers at least 8 error symptoms
- [ ] 4.5 Verify SKILL.md pipeline phases section lists all 5 phases with gates
- [ ] 4.6 Verify no content was lost -- spot check that key sections from old SKILL.md appear in the appropriate reference file (architecture components, tool parameter tables, contract model rules, deployment flow steps)
- [ ] 4.7 Verify AGENTS.md references/ listing matches actual files on disk
