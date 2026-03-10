# Exoskeleton Phase 1 - Design (tentacular-skill)

## SKILL.md Changes

### New Section: "Exoskeleton Services" (after Contract Model, ~line 806)
- What the exoskeleton is
- How to detect it (exo_status gate)
- When to use tentacular-* deps vs manual deps
- They can coexist in one contract
- Examples with and without exoskeleton

### Updated: CLI vs MCP Decision Tree (~line 108)
- New branch: "Workflow needs database/messaging/storage?" -> exo_status check

### Updated: Deploy Prerequisites (Deployment Flow, ~line 886)
- If workflow uses tentacular-* deps, confirm via exo_status

### Updated: MCP Tools Reference (~line 208)
- Add exo_status and exo_registration to tools table

### Updated: Contract Model (~line 794)
- tentacular-* prefix convention explanation
- Only protocol required, host/port/auth auto-filled

### Anti-confusion Guidance
- Never add tentacular-* deps without exo_status check
- Never add host/port/auth to tentacular-* deps
- Always ask user about exo vs self-managed when ambiguous
- tentacular-* deps fail on clusters without exoskeleton (by design)
