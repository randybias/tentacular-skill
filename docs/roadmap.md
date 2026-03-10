# Tentacular Skill Roadmap

Last updated: 2026-03-10

## Active

### P1 — Next Up

Items planned for the near term.

| Item | Description | Dependencies | Target |
|------|-------------|--------------|--------|
| NATS SPIFFE mode documentation | Document NATS SPIFFE mTLS configuration and usage for workflows. | NATS TLS infra active (tentacular-mcp P0) | TBD |
| Vault integration documentation | Document Vault-based credential management for workflows. | Vault integration (tentacular-mcp P1) | TBD |
| Provenance query documentation | Document audit API usage and provenance querying for deployers and agents. | Audit API (tentacular-mcp P1) | TBD |

### P2 — Planned

Items planned but not yet scheduled.

| Item | Description | Dependencies |
|------|-------------|--------------|
| Istio ambient mode documentation | Document Istio ambient mode integration and zero-config mTLS for workflows. | Istio support (tentacular-mcp P2) |

## Archive

Completed items, most recent first.

### 2026-03-10 — Exoskeleton Phase 1

| Item | Completed | Notes |
|------|-----------|-------|
| Exoskeleton Services section in SKILL.md | 2026-03-10 | Full agent reference for exoskeleton service integration |
| Authentication/SSO guidance | 2026-03-10 | CLI login/logout flow, token management, dual auth model |
| Deployer provenance documentation | 2026-03-10 | Annotation-based provenance tracking for deployments |
| CLI login/logout/whoami command reference | 2026-03-10 | Device auth flow, token storage, identity display |
| Known limitations and prerequisites | 2026-03-10 | NATS SPIFFE mode, RustFS alpha quirks, Keycloak azp |
| Contract model tentacular-* prefix docs | 2026-03-10 | How workflows declare exoskeleton dependencies |
