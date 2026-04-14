# Authorization Reference

POSIX-like owner/member/other permissions for tentacles, enforced at the
MCP server layer. Enclaves act as directories; tentacles act as files.
Membership is evaluated from the `tentacular.io/enclave-members` annotation
(comma-separated emails), not IdP group claims.

## Permission Model

Every tentacle has three permission scopes, each with read/write/execute bits:

| Scope | Check | Example |
|-------|-------|---------|
| Owner | `deployer.Subject == tentacle.owner-sub` | Creator of the tentacle |
| Member | `deployer.Email in enclave.enclave-members` | Registered enclave member |
| Other | Everyone else | Any authenticated OIDC user |

Permission bits map to operations:

| Bit | Operations |
|-----|-----------|
| Read (`r`) | `wf_list`, `wf_status`, `wf_describe`, `wf_health`, `wf_logs`, `wf_pods`, `wf_events`, `wf_jobs` |
| Write (`w`) | `wf_apply` (update), `wf_remove` |
| Execute (`x`) | `wf_run`, `wf_restart` |

## Enclave Permissions

Enclaves are directories; tentacles are files. Both layers use the same
owner/member/other model, and both must pass for an operation to succeed.

### Directory/File Analogy

| Concept | POSIX | Tentacular |
|---------|-------|------------|
| Directory | `/home/team/` | Enclave `team-prod` |
| File | `/home/team/report.sh` | Tentacle `team-prod/report-gen` |
| `ls` a directory | Requires Read on the directory | `wf_list` requires Read on the enclave |
| Create a file | Requires Write on the directory | `wf_apply` (create) requires Write on the enclave |
| Read a file | Requires Read on the file | `wf_describe` requires Read on the tentacle |
| Execute a file | Requires Execute on the file | `wf_run` requires Execute on the tentacle |

### Two-Layer Check

Every operation checks permissions in order:

1. **Enclave check** — does the caller have the required bit on the enclave?
2. **Tentacle check** — does the caller have the required bit on the tentacle?

If either check fails, the request is denied. Exception: the enclave owner
bypasses the tentacle check entirely (superuser).

### Enclave Permission Bits

| Bit | Enclave Operations |
|-----|-------------------|
| Read (`r`) | `wf_list`, `wf_health_enclave`, `wf_pods`, `wf_logs`, `wf_events`, `wf_jobs` (when no tentacle name is specified) |
| Write (`w`) | `wf_apply` (create a new tentacle in the enclave) |
| Execute (`x`) | Reserved for future use |

### Enclave Ownership

`enclave_provision` stamps ownership annotations on the namespace:

- `tentacular.io/enclave-owner` — OIDC email of the channel owner
- `tentacular.io/enclave-owner-sub` — OIDC subject of the owner
- `tentacular.io/enclave-members` — comma-separated OIDC emails of members
- `tentacular.io/mode` — from `quota` preset or default `rwxrwx---`

### Default Inheritance

Enclaves can specify a default mode for new tentacles via the
`tentacular.io/enclave-default-mode` annotation. When a deployer does not
pass an explicit `--mode` flag, this default is used. If unset, new tentacles
inherit the enclave's own `tentacular.io/mode`.

## Enclave Membership Lifecycle

Membership is driven by Slack channel events and Kraken owner commands.
The `tentacular.io/enclave-members` annotation stores a comma-separated list
of OIDC emails. IdP groups are not consulted.

### Channel Event Mapping

| Slack Event | Enclave Effect |
|-------------|---------------|
| `member_joined_channel` | User becomes a **visitor** until the owner explicitly adds them |
| `member_left_channel` | User is removed from members; their tentacles transfer to the enclave owner |
| `channel_archive` | Enclave status set to **frozen** (no new deploys, cron paused) |
| `channel_unarchive` | Enclave status set to **active** |
| `channel_rename` | `channel_name` annotation updated; enclave slug is immutable |

### Kraken Commands

| Command | Who Can Run | Effect |
|---------|------------|--------|
| `@kraken add @user` | Owner only | Adds the mentioned user as an enclave member (resolves Slack email to OIDC email) |
| `@kraken remove @user` | Owner only | Removes the user from enclave members; transfers their tentacles to the enclave owner |
| `@kraken set mode <preset>` | Owner only | Updates the enclave permission mode via `enclave_sync` `new_mode` parameter |
| `@kraken members` | Anyone | Lists enclave owner, members, and current access level in plain language |
| `@kraken whoami` | Anyone | Reports the caller's role: owner, member, or visitor |

Valid mode presets for `set mode`: `private`, `team` (alias for `member-edit`),
`shared` (alias for `team`), `open-read`, `open-run`. Raw rwx strings
(e.g., `rwxrwx---`) are also accepted.

### Ownership Transfer on Member Removal

When a member is removed (via `@kraken remove` or `member_left_channel`),
`enclave_sync` with `remove_members` automatically transfers all tentacles
owned by the departing member to the enclave owner. The response includes a
`transfers` array with `{tentacle_name, from_owner, to_owner, success, error}`
entries for each affected tentacle.

### Drift Detection

The Kraken runs periodic drift detection (default: every 5 minutes) to
reconcile Slack channel membership with enclave annotations. Drift detection:
- Removes annotation members who are no longer in the Slack channel
- Never auto-adds members (channel join alone does not grant membership)
- Never removes the enclave owner
- Skips frozen enclaves
- Processes enclaves in round-robin batches

## Presets

Used with `enclave_provision`, `enclave_sync`, and `@kraken set mode`.
"Member" is checked against `tentacular.io/enclave-members` annotation on
the namespace — not IdP claims.

| Name | Mode | Meaning |
|------|------|---------|
| `private` | `rwx------` | Owner only |
| `member-read` | `rwxr-x---` | Owner full, members read+run |
| `member-edit` | `rwxrwx---` | Owner and members full access (default for enclaves) |
| `open-read` | `rwxrwxr--` | Owner and members full; visitors read-only |
| `open-run` | `rwxrwxr-x` | Owner and members full; visitors read+run |

## Evaluator Flow: CheckEnclave

Called for all enclave namespaces (has `tentacular.io/enclave` annotation).

```
1. Evaluator disabled? → Allow
2. No enclave annotation? → Deny (not an enclave namespace)
3. Caller is enclave owner? → Allow (superuser — bypasses tentacle check)
4. Caller is tentacle owner? → check owner bits (positions 0-2)
5. Caller email in enclave-members? → check member bits (positions 3-5)
6. Otherwise → check other bits (positions 6-8)
```

Step 6 reads `tentacular.io/enclave-members` from the namespace. IdP groups
are not consulted.

## CLI Commands

```bash
# Deploy with explicit mode
tntc deploy --mode member-read

# Deploy to a specific enclave
tntc deploy --enclave marketing

# --- Enclave management ---

# List enclaves you have access to
tntc enclave list

# Get enclave details
tntc enclave info <enclave>

# Provision a new enclave
tntc enclave provision <name> --owner-email user@example.com

# Update enclave membership or status
tntc enclave sync <name> --add-members user@example.com

# Update enclave permission mode
tntc enclave sync <name> --mode rwxrwx---
```

## Annotations

See `references/contract-model.md` section "Deployer Provenance and Authorization"
for the full annotation schema and create-vs-update stamping behavior.

## Key Behaviors

- **Create path**: deployer becomes owner, annotations stamped from OIDC identity + flags
- **Update path**: ownership preserved, only provenance/audit annotations updated. Owner can change mode via `--mode` flag on redeploy.
- **Unowned resources denied**: resources without `owner-sub` annotation are denied to OIDC callers. Use kubectl annotation patching to adopt unowned resources (see Kubernetes Administrator Guide below).
- **Member membership**: evaluated live from the `enclave-members` annotation (comma-separated emails) at request time, never from JWT group claims
- **Ownership transfer**: when a member is removed, their tentacles are automatically transferred to the enclave owner
- **No group annotation**: `tentacular.io/group` is not written. Authorization uses enclave membership only.
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
# Stamp ownership on an enclave namespace
kubectl annotate ns team-prod \
  tentacular.io/enclave-owner=user@example.com \
  tentacular.io/enclave-owner-sub=<user-uuid> \
  tentacular.io/mode=rwxrwx---

# Stamp ownership on a tentacle deployment
kubectl annotate deploy -n team-prod my-tentacle \
  tentacular.io/owner=user@example.com \
  tentacular.io/owner-sub=<user-uuid> \
  tentacular.io/mode=rwxrwx---
```

Find user UUIDs via `tntc whoami` (Subject field) or the Keycloak admin console.

### Transferring Ownership

No `chown` command exists yet. Transfer ownership via kubectl:

```bash
kubectl annotate deploy -n team-prod my-tentacle \
  tentacular.io/owner=new-user@example.com \
  tentacular.io/owner-sub=<new-user-uuid> \
  --overwrite
```

### Auditing Permissions

```bash
# List all tentacle permissions in an enclave
kubectl get deploy -n team-prod -o custom-columns=\
  NAME:.metadata.name,\
  OWNER:.metadata.annotations.tentacular\.io/owner-email,\
  MODE:.metadata.annotations.tentacular\.io/mode

# Check enclave permissions
kubectl get ns team-prod -o jsonpath='{.metadata.annotations}' | jq
```
