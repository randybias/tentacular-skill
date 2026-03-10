# Exoskeleton Phase 1 - Tasks (tentacular-skill)

## Step 1: Add Exoskeleton Services Section
- [x] New section after Contract Model explaining exoskeleton concept
- [x] Detection via exo_status tool
- [x] When to use tentacular-* vs manual deps
- [x] Coexistence examples (mixed deps in one contract)
- [x] Example with exoskeleton enabled
- [x] Example without exoskeleton (manual Postgres)

## Step 2: Update CLI vs MCP Decision Tree
- [x] Add decision branch for database/messaging/storage needs
- [x] Route through exo_status check

## Step 3: Update Deploy Prerequisites
- [x] Add exo_status verification step for tentacular-* deps

## Step 4: Update MCP Tools Reference
- [x] Add exo_status tool entry with description and params
- [x] Add exo_registration tool entry with description and params

## Step 5: Update Contract Model Section
- [x] Document tentacular-* prefix convention
- [x] Only protocol required for tentacular-* deps
- [x] MCP rejects if service is disabled

## Step 6: Add Anti-confusion Guidance
- [x] Never add tentacular-* without exo_status
- [x] Never add host/port/auth to tentacular-* deps
- [x] Ask user about exo vs self-managed when ambiguous
- [x] tentacular-* deps fail without exoskeleton (by design)

## Step 7: Update References
- [x] Update references/contract.md with tentacular-* protocol info
- [x] Update references/agent-workflow.md with exo_status pre-check

## Step 8: SPIRE and NATS SPIFFE Documentation Updates
- [x] Update Known Limitations: replace shared-token caveat with SPIFFE mode note
- [x] Update exo_status return fields: add `spire_available`, `nats_spiffe_enabled`
- [x] Update cleanup behavior: NATS cleanup in SPIFFE mode removes authz ConfigMap entry
- [x] Add note that NATS SPIFFE mode requires cluster-level NATS TLS configuration
- [x] Update references/contract.md if NATS shared token mentioned as limitation
- [x] Update references/agent-workflow.md if NATS shared token mentioned as limitation
