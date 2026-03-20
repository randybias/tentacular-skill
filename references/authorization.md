# Authorization Reference

POSIX-like owner/group/mode permissions for tentacles, enforced at the
MCP server layer. Namespaces act as directories; tentacles act as files.

## Permission Model

Every tentacle has three permission scopes, each with read/write/execute bits:

| Scope | Check | Example |
|-------|-------|---------|
| Owner | `deployer.Subject == tentacle.owner-sub` | Creator of the tentacle |
| Group | `tentacle.group in deployer.Groups` (from JWT) | IdP group assigned to the tentacle |
| Others | Everyone else | Any authenticated OIDC user |

Permission bits map to operations:

| Bit | Operations |
|-----|-----------|
| Read (`r`) | `wf_list`, `wf_status`, `wf_describe`, `wf_health`, `wf_logs`, `wf_pods`, `wf_events`, `wf_jobs` |
| Write (`w`) | `wf_apply` (update), `wf_remove` |
| Execute (`x`) | `wf_run`, `wf_restart` |

## Namespace Permissions

Namespaces are directories; tentacles are files. Both layers use the same
owner/group/mode model, and both must pass for an operation to succeed.

### Directory/File Analogy

| Concept | POSIX | Tentacular |
|---------|-------|------------|
| Directory | `/home/team/` | Namespace `team-prod` |
| File | `/home/team/report.sh` | Tentacle `team-prod/report-gen` |
| `ls` a directory | Requires Read on the directory | `wf_list` requires Read on the namespace |
| Create a file | Requires Write on the directory | `wf_apply` (create) requires Write on the namespace |
| Read a file | Requires Read on the file | `wf_describe` requires Read on the tentacle |
| Execute a file | Requires Execute on the file | `wf_run` requires Execute on the tentacle |

### Two-Layer Check

Every operation checks permissions in order:

1. **Namespace check** — does the caller have the required bit on the namespace?
2. **Tentacle check** — does the caller have the required bit on the tentacle?

If either check fails, the request is denied.

### Namespace Permission Bits

| Bit | Namespace Operations |
|-----|---------------------|
| Read (`r`) | `wf_list`, `wf_health_ns`, `wf_pods`, `wf_logs`, `wf_events`, `wf_jobs` (when no tentacle name is specified) |
| Write (`w`) | `wf_apply` (create a new tentacle in the namespace), `ns_update`, `ns_delete` |
| Execute (`x`) | Reserved for future use |

### Namespace Ownership

`ns_create` stamps the same ownership annotations on the namespace as
`wf_apply` stamps on tentacles:

- `tentacular.io/owner-sub`, `owner-email`, `owner-name` — set from OIDC identity
- `tentacular.io/group` — from `--group` flag or empty
- `tentacular.io/mode` — from `--share`/`--mode` flag or default `rwxr-x---`

### Default Inheritance

Namespaces can specify defaults that new tentacles inherit when the deployer
does not pass explicit `--group` or `--share` flags:

- `tentacular.io/default-mode`: default mode for new tentacles (e.g., `rwxrwx---`)
- `tentacular.io/default-group`: default group for new tentacles (e.g., `platform-eng`)

## Presets

| Name | Mode | Meaning |
|------|------|---------|
| `private` | `rwx------` | Owner only |
| `group-read` | `rwxr-x---` | Owner full, group read+execute (default) |
| `group-run` | `rwx--x---` | Owner full, group execute only |
| `group-edit` | `rwxrwx---` | Owner and group full access |
| `public-read` | `rwxr--r--` | Owner full, everyone can read |

## Evaluator Flow

```
1. Bearer-token caller? → ALLOW (full trust bypass)
2. No owner-sub annotation? → DENY (unowned resource; use bearer-token to adopt)
3. Caller is owner? → check owner bits (positions 0-2)
4. Tentacle group in caller's JWT groups? → check group bits (positions 3-5)
5. Otherwise → check others bits (positions 6-8)
```

## CLI Commands

```bash
# Deploy with permissions
tntc deploy --group platform-team --share group-read

# --- Tentacle permissions (2 positional args) ---

# Check permissions on a tentacle
tntc permissions get <namespace> <name>

# Change mode (accepts preset names or raw rwx strings)
tntc permissions set <namespace> <name> --mode group-edit
tntc permissions set <namespace> <name> --mode rwxrwx---

# Change group
tntc permissions set <namespace> <name> --group dev-team

# Shortcuts
tntc chmod <mode-or-preset> <namespace>/<name>
tntc chgrp <group> <namespace>/<name>

# --- Namespace permissions (1 positional arg) ---

# Check permissions on a namespace
tntc permissions get <namespace>

# Change namespace mode
tntc permissions set <namespace> --mode group-edit

# Change namespace group
tntc permissions set <namespace> --group platform-eng

# Shortcuts for namespaces
tntc chmod <mode-or-preset> <namespace>
tntc chgrp <group> <namespace>
```

## MCP Tools

**Tentacle permissions:**
- `permissions_get(namespace, name)` — returns owner, group, mode, preset. Requires Read.
- `permissions_set(namespace, name, group?, mode?, share?)` — changes group and/or mode. Owner-only (plus bearer-token bypass). `mode` accepts raw strings, `share` accepts preset names.

**Namespace permissions:**
- `ns_permissions_get(namespace)` — returns namespace owner, group, mode, preset.
- `ns_permissions_set(namespace, group?, mode?, share?)` — changes namespace group and/or mode. Namespace-owner-only (plus bearer-token bypass).

## Annotations

See `references/contract-model.md` section "Deployer Provenance and Authorization"
for the full annotation schema and create-vs-update stamping behavior.

## Key Behaviors

- **Create path**: deployer becomes owner, annotations stamped from OIDC identity + flags
- **Update path**: ownership preserved, only provenance/audit annotations updated. Owner-only can change group/mode via `--group`/`--share` flags on redeploy.
- **Bearer-token**: bypasses all authz checks, owner fields left empty
- **Unowned resources denied**: resources without `owner-sub` annotation are denied to OIDC callers. Use bearer-token to adopt unowned resources.
- **Group membership**: evaluated live from JWT claims at request time, never stored as annotations
- **Annotation migration**: `tentacular.dev/*` annotations are read with fallback but all writes use `tentacular.io/*`

## Kubernetes Administrator Guide

MCP-layer authorization enforces permissions through annotations. Kubernetes
administrators with `kubectl` access can directly manage these annotations as
a break-glass mechanism or for bulk operations.

### Trust Boundary

kubectl access to annotations bypasses MCP-layer authz entirely. This is by
design — Kubernetes RBAC is the outer security perimeter. MCP authz protects
against unauthorized access through the MCP/CLI interface, not against cluster
administrators.

### Adopting Unowned Resources

Resources without `tentacular.io/owner-sub` are denied to OIDC callers.
To stamp ownership on unowned resources:

```bash
# Stamp ownership on a namespace
kubectl annotate ns tent-dev \
  tentacular.io/owner-sub=<user-uuid> \
  tentacular.io/owner-email=user@example.com \
  tentacular.io/owner-name="User Name" \
  tentacular.io/group=platform-team \
  tentacular.io/mode=rwxr-x---

# Stamp ownership on a tentacle deployment
kubectl annotate deploy -n tent-dev my-tentacle \
  tentacular.io/owner-sub=<user-uuid> \
  tentacular.io/owner-email=user@example.com \
  tentacular.io/owner-name="User Name" \
  tentacular.io/group=platform-team \
  tentacular.io/mode=rwxr-x---
```

Find user UUIDs via `tntc whoami` (Subject field) or the Keycloak admin console.

### Transferring Ownership

No `chown` command exists yet. Transfer ownership via kubectl:

```bash
kubectl annotate deploy -n tent-dev my-tentacle \
  tentacular.io/owner-sub=<new-user-uuid> \
  tentacular.io/owner-email=new-user@example.com \
  tentacular.io/owner-name="New User" \
  --overwrite
```

### Auditing Permissions

```bash
# List all tentacle permissions in a namespace
kubectl get deploy -n tent-dev -o custom-columns=\
  NAME:.metadata.name,\
  OWNER:.metadata.annotations.tentacular\.io/owner-email,\
  GROUP:.metadata.annotations.tentacular\.io/group,\
  MODE:.metadata.annotations.tentacular\.io/mode

# Check namespace permissions
kubectl get ns tent-dev -o jsonpath='{.metadata.annotations}' | jq
```
