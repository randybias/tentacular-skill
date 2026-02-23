# Phase 03: Cluster Environment Profile

**Gate: a fresh profile must exist and be read before any workflow design.**

A profile tells you what the cluster can actually do — CNI, storage, extensions,
security posture, and explicit Agent Guidance strings. Designing without it
produces workflows that fail silently in production.

## Check

```bash
cat ~/.tentacular/envprofiles/<env>.md | head -3
```

Check `generatedAt`. If it is less than 7 days old, skip to **Read the Profile** below.

## Generate or Refresh

```bash
# Generate for one environment
tntc cluster profile --env <name> --save

# Regenerate all environments
tntc cluster profile --all --save

# Force rebuild (ignore freshness guard)
tntc cluster profile --env <name> --save --force
```

Profiles are saved to:
- `~/.tentacular/envprofiles/<env>.md` (user-level config)
- `.tentacular/envprofiles/<env>.md` (project-level config)

## Read the Profile

Read the full profile for the target environment. Pay particular attention to:

**Agent Guidance section** — apply every item before writing any workflow code:

| Guidance string | What to do |
|-----------------|------------|
| `Use runtime_class: gvisor` | Set in workflow.yaml |
| `gVisor not available` | Omit or set `runtime_class: ""` |
| `kind cluster detected` | Set `runtime_class: ""`, `imagePullPolicy: IfNotPresent` |
| `Istio detected: NetworkPolicy egress...` | Add `namespaceSelector: istio-system` to all egress rules |
| `Namespace enforces restricted PodSecurity` | Containers must be non-root, no privilege escalation |
| `RWX storage (inferred) via <name>` | Use that StorageClass; verify RWX with cluster admin before relying on it |
| `No RWX-capable StorageClass inferred` | No shared PVCs across replicas |
| `ResourceQuota active` | Size resource requests/limits within quota |
| `cert-manager available` | Auto-TLS available for HTTPS dependencies |
| `WARNING: CNI could not be detected` | Verify NetworkPolicy support manually before writing egress rules |
| `SECURITY NOTE: Node labels...` | Treat profile file as sensitive; review before committing to shared repo |

## Drift Triggers — Re-run this phase when:

- Deploy fails with `unknown RuntimeClass`
- NetworkPolicy rejects traffic the profile said is allowed
- PVC fails to bind but profile shows provisioner is available
- `cluster check` passes but deploy has unexpected resource errors
- Profile `generatedAt` > 7 days
- Cluster version changed since last profile
- User mentions cluster upgrade, new add-on, or infrastructure change

## Gate Verification

Before leaving this phase, confirm:

- [ ] Profile file exists for the target environment
- [ ] `generatedAt` is within 7 days (or just regenerated)
- [ ] Agent Guidance section has been read and all items noted
- [ ] Any `WARNING:` guidance items have been resolved or escalated

Only then: proceed to `phases/04-build.md` or `phases/05-test-and-deploy.md`.
