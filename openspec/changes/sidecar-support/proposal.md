## Why

AI agents using Tentacular need to know how to declare and use sidecar containers in workflows. The skill documentation is the primary reference agents consult when building workflows. Without sidecar documentation in the skill, agents will not know the feature exists or how to use it correctly.

## What Changes

### SKILL.md Pointer Section

Add a 3-5 line sidecar section to SKILL.md following the progressive disclosure pattern. The section introduces sidecars and points to the full reference file.

### references/sidecars.md

Create a full sidecar reference covering:
- What sidecars are and when to use them
- SidecarSpec YAML schema with all fields
- Field validation rules (port range, name format, protocol values)
- Communication pattern: `fetch("http://localhost:PORT/path")` from node code
- Shared volume usage (`/shared/input/`, `/shared/output/`)
- Security model: all containers share gVisor sandbox, identical SecurityContext
- Example workflow YAML with sidecars declared
- Example node code calling a sidecar

## Requirements

1. SKILL.md sidecar section must be 3-5 lines max, following existing progressive disclosure pattern
2. SKILL.md must include "Read `references/sidecars.md` when:" pointer with 2-3 bullet points
3. `references/sidecars.md` must document the complete SidecarSpec schema
4. Reference must include field validation rules so agents can produce valid YAML
5. Reference must show the `fetch()` communication pattern for node code
6. Reference must explain shared volume semantics
7. Reference must note security implications (gVisor covers all containers)

## Acceptance Criteria

- [ ] SKILL.md has a "Sidecars" section of 3-5 lines
- [ ] SKILL.md section includes "Read `references/sidecars.md` when:" pointer
- [ ] `references/sidecars.md` exists with complete SidecarSpec schema
- [ ] Reference documents all field validation rules (name, port, protocol)
- [ ] Reference includes example workflow YAML with `sidecars:` block
- [ ] Reference includes example node TypeScript calling `fetch("http://localhost:PORT/...")`
- [ ] Reference explains `/shared` volume usage pattern
- [ ] Reference notes gVisor/SecurityContext security model for sidecars
- [ ] No implementation details or configuration specifics are inlined in SKILL.md

## Scope

### In Scope

- SKILL.md pointer section for sidecars
- `references/sidecars.md` full reference document
- Update tool safety classification tables if sidecar-related tools are added (none expected)

### Out of Scope

- Changes to other SKILL.md sections
- Phase documentation updates
- Tool count updates (no new MCP tools for sidecars)

## Dependencies

- `tentacular/openspec/changes/sidecar-support/` -- spec schema must be finalized before documenting it
