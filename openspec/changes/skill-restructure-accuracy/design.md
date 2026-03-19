## Context

SKILL.md is a monolithic ~1,400-line file that agents must load in full, wasting context window on content irrelevant to the current task. There is no progressive disclosure -- an agent needing tool syntax must wade through architecture docs, deployment ops, and contract specs. This restructure creates a lean ~280-line dispatch core with 7 focused reference files under `references/`, enabling agents to load only what they need via `@references/` pointers.

The existing `references/` directory listed in AGENTS.md does not exist on disk. The files listed there (agent-workflow.md, cli.md, contract.md, etc.) were planned but never created. This change replaces that plan with a new set of reference files derived from the actual SKILL.md content.

## Goals / Non-Goals

**Goals:**
- Reduce SKILL.md from ~1,400 lines to ~280 lines as a dispatch core
- Extract content into 7 reference files that can be loaded on demand
- Add new content: error recovery playbooks, tool safety classification, workflow creation inversion pattern, schema introspection hints
- Wire existing `phases/01-05` into the core via imperative gates
- Update AGENTS.md and README.md to reflect the new structure

**Non-Goals:**
- Rewriting the content of phases/01-05 (they stay as-is, just get wired in)
- Changing the docs site at randybias.github.io/tentacular-docs
- Modifying tentacular-mcp tool implementations
- Adding CI/automation for doc validation

## Decisions

### 1. SKILL.md structure: summary sections with Read triggers

**Decision**: Each major section in the new SKILL.md gets a 3-5 line summary followed by a `> Read references/<file>.md for full details` pointer. The agent reads the lean core first and follows pointers only when the task requires detail.

**Rationale**: This is progressive disclosure. The agent gets routing information (which reference to read) without paying the context cost of the full content. The summary is enough to determine relevance.

### 2. Reference file decomposition: 7 files aligned to task domains

**Decision**: Extract into these 7 files, each aligned to a distinct agent task domain:

| File | Source lines (approx) | Task domain |
|------|----------------------|-------------|
| `architecture.md` | 57-107 | Understanding system design |
| `mcp-tools.md` | 212-700 | Tool discovery and invocation |
| `node-contract.md` | 715-774 + contract model sections | Writing workflow nodes |
| `workflow-spec.md` | 776-831, 1127-1174 | Authoring workflow.yaml |
| `contract-model.md` | 833-1097 | Security and contract validation |
| `deployment-ops.md` | 1176-1365 | Deploying and operating workflows |
| `error-recovery.md` | New content | Diagnosing and recovering from failures |

**Rationale**: Each file maps to a single "what am I trying to do" question. An agent building a node reads node-contract.md. An agent deploying reads deployment-ops.md. No file serves two unrelated purposes.

### 3. Tool safety classification: 3-tier model in mcp-tools.md

**Decision**: Classify all MCP tools into three safety tiers in `references/mcp-tools.md`:

- **read-only** -- tools that only query state (wf_list, wf_describe, wf_status, wf_logs, wf_pods, wf_events, wf_jobs, wf_health, wf_health_ns, ns_get, ns_list, health_*, audit_*, gvisor_check, gvisor_verify, exo_status, cluster_preflight, cluster_profile, proxy_status)
- **state-changing** -- tools that modify cluster state reversibly (wf_apply, wf_restart, wf_run, ns_create, ns_update, gvisor_annotate_ns, exo_registration, cred_issue_token, cred_kubeconfig)
- **destructive** -- tools that delete or irreversibly modify (wf_remove, ns_delete, cred_rotate)

**Rationale**: Aligns with MCP ToolAnnotations (readOnlyHint, destructiveHint) being added in tentacular-mcp. Gives agents a quick lookup before invoking a tool.

### 4. Pipeline phases with imperative gates

**Decision**: The new SKILL.md includes a "Pipeline Phases" section that lists phases/01-05 with a one-line summary each, plus a gate condition that must be true before advancing. Example: "Gate: `tntc version` succeeds" for phase 01.

**Rationale**: Agents skip phases or execute them out of order. Gates make progression explicit and verifiable. The phase files themselves are unchanged.

### 5. Workflow creation inversion pattern

**Decision**: Add a "Workflow Creation" section to SKILL.md that enforces schema-first workflow creation: (1) define contract, (2) derive workflow.yaml from contract, (3) implement nodes. Not: write nodes first, then figure out the contract.

**Rationale**: The current SKILL.md buries this in the agent workflow guide. Making it a first-class section in the core prevents the most common agent mistake: writing code before defining the contract.

### 6. Error recovery index in core, playbooks in reference

**Decision**: SKILL.md contains a compact error recovery index (symptom -> fix table, ~15 rows). `references/error-recovery.md` contains the full playbooks with diagnostic steps, root causes, and recovery procedures.

**Rationale**: Quick lookup in the core for common errors; deep dive available on demand. The index is small enough to keep in the ~280-line budget.

### 7. Schema introspection hint

**Decision**: Add a 5-line section to SKILL.md noting that agents can use `wf_describe` and `cluster_profile` to discover capabilities dynamically rather than relying solely on documentation.

**Rationale**: Reduces doc staleness risk. If a tool parameter changes, the agent can verify by introspecting rather than trusting possibly-outdated docs.

### 8. AGENTS.md references/ listing replacement

**Decision**: Replace the current AGENTS.md `references/` listing (8 files that do not exist) with the 7 new files from this change.

**Rationale**: The current listing is aspirational and misleading. Replacing it with actually-existing files prevents confusion.

## Risks / Trade-offs

- **Agents that hard-code SKILL.md section paths will break**: Any agent referencing specific line ranges or section headers in the current SKILL.md will need updating. Mitigation: the proposal explicitly states no backwards compatibility is needed.
- **Two-hop context loading increases latency**: Agents must read SKILL.md then follow a pointer to a reference file, adding a tool call. Mitigation: the summaries in SKILL.md are sufficient for many tasks without following the pointer.
- **error-recovery.md is entirely new content**: Unlike other references that are extracted from existing SKILL.md, this file must be authored from scratch. Mitigation: the error recovery index in SKILL.md scopes what the playbooks must cover; implementation can start with the most common failure modes and expand.
- **mcp-tools.md tool safety classifications may drift from MCP server**: As new tools are added or classifications change in tentacular-mcp. Mitigation: the schema introspection hint encourages agents to verify dynamically; the e2e-skill-docs-validation change provides a validation checklist pattern.
