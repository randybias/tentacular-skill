# Enclaves Reference

An enclave is the primary organizational unit in Tentacular. It binds a Slack
channel, a Kubernetes namespace, shared exoskeleton services (Postgres, RustFS,
and optionally NATS and SPIRE), and team membership into a single governed space.
Every tentacle lives inside exactly one enclave.

## What Is an Enclave

```
Enclave = {
  name:         string    // DNS-1123 slug (set at creation, immutable)
  owner:        string    // OIDC email of Slack channel owner
  members:      string[]  // OIDC emails of registered members (max 100)
  platform:     string    // "slack" (discord/teams: future)
  channel_id:   string    // platform-specific ID (e.g., "C08XXXXXXX")
  channel_name: string    // current Slack channel name
  namespace:    string    // K8s namespace (= enclave name)
  exo_services: object    // provisioned backing services
  status:       string    // "active" | "frozen"
}
```

All enclave metadata is stored as Kubernetes annotations on the namespace — the
namespace IS the enclave's backing resource. The Kraken syncs Slack membership
into these annotations and reconciles drift on startup and at regular intervals.

## Enclave Lifecycle

```
provision → active → frozen → deprovision
                  ↕
               (unfreeze)
```

| Phase | Trigger | What Happens |
|-------|---------|--------------|
| **provision** | Channel owner asks Kraken to initialize the channel | Namespace created, exo baseline provisioned (Postgres + RustFS), RBAC and NetworkPolicy applied, quota preset applied |
| **active** | Immediately after provision | All operations allowed: deploy, run, read |
| **frozen** | Slack channel archived | Existing tentacles keep running; cron triggers paused; no new deployments allowed |
| **deprovision** | Explicit request by enclave owner | All tentacles removed, exo cleaned up, namespace deleted, channel reverts to regular mode |

Channel rename: `channel_name` annotation updated via `enclave_sync`. K8s
namespace name does not change (K8s does not support namespace rename). The
enclave `name` slug is always the original slug.

## MCP Tools

Use `tools/list` for current parameter schemas. This section covers semantics.

### enclave_provision (Write)

Creates an enclave from a Slack channel. Called by The Kraken after verifying
the requester is the channel owner.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | yes | DNS-1123 slug (slugified from channel name) |
| `owner_email` | string | yes | OIDC email of the channel owner |
| `owner_sub` | string | yes | OIDC subject (sub claim) of the owner |
| `platform` | string | yes | `"slack"` (only supported value currently) |
| `channel_id` | string | yes | Platform-specific channel ID |
| `channel_name` | string | yes | Human-readable channel name |
| `members` | array of strings | no | Initial member OIDC emails |
| `quota_preset` | string | no | `small`, `medium` (default), or `large` |

Returns: enclave info object (same shape as `enclave_info`).

Quota presets:

| Preset | CPU | Memory | Storage | Typical use |
|--------|-----|--------|---------|-------------|
| `small` | 1 | 2Gi | 10Gi | API integrations, text processing |
| `medium` | 2 | 4Gi | 50Gi | File processing, moderate databases (default) |
| `large` | 4 | 8Gi | 100Gi | Large datasets, media processing, many tentacles |

### enclave_info (Read-only)

Returns full enclave details: owner, members, exo service status, tentacle
count, quota usage, and current permission mode.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | yes | Enclave name (DNS-1123 slug) |

Returns: `name`, `owner`, `owner_sub`, `members` (array),
`platform`, `channel_id`, `channel_name`, `status`, `quota_preset`, `tentacle_count`,
`exo_services` (per-service available flags), `created_at`, `updated_at`.

### enclave_list (Read-only)

Lists enclaves the caller has read access to. With OIDC auth, returns only
enclaves where the caller is owner or member (or has other-read permission).
With bearer token, returns all enclaves.

No required parameters. Optional: `caller_email` filter (set automatically for
OIDC callers).

Returns: array of enclave summary objects (name, owner, status, platform,
channel_name, created_at, members).

### enclave_sync (Write)

Updates enclave membership, owner, channel name, or status. Called by The
Kraken when Slack events occur (member join/leave, channel rename, archive).
Can also be called by the enclave owner directly.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | yes | Enclave name |
| `add_members` | array of strings | no | OIDC emails to add as members |
| `remove_members` | array of strings | no | OIDC emails to remove |
| `new_owner` | string | no | New owner email (must be a current member) |
| `new_channel_name` | string | no | Updated display name |
| `new_status` | string | no | `"active"` or `"frozen"` |

Only the enclave owner (or bearer token) can call this tool. Returns updated
enclave info object.

### enclave_deprovision (Destructive)

Deletes the enclave and everything in it: all tentacles removed, exoskeleton
services cleaned up (`cleanup_on_undeploy: true` semantics), namespace deleted.
This is permanent and cannot be undone.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | yes | Enclave name |

Only the enclave owner (or bearer token) can call this tool. Confirmation is
handled CLI-side (--confirm flag), not as a tool parameter.

Always confirm with the user before calling. Returns `{ name, deleted, tentacles_removed }`.

## Permission Model

Enclaves use the same POSIX-like owner/member/other model as namespaces and
tentacles, with one key change: **"group" is replaced by "enclave membership."**
Instead of checking IdP group claims in the JWT, the evaluator checks the
`tentacular.io/enclave-members` annotation on the namespace.

### Three Principal Classes

| Class | Who | How determined |
|-------|-----|----------------|
| **Owner** | Enclave owner (Slack channel owner) | `tentacular.io/enclave-owner` annotation |
| **Member** | Registered enclave members | `tentacular.io/enclave-members` comma-separated list |
| **Other** | Authenticated but not owner or member | Anyone else with a valid OIDC token |

The enclave owner is a **superuser** within their enclave: they bypass
per-tentacle permission checks and can modify or remove any tentacle regardless
of who deployed it.

### Permission Presets (Enclave Defaults)

| Preset | Mode | Owner | Member | Other | Use case |
|--------|------|-------|--------|-------|----------|
| `private` | `rwx------` | Full | None | None | Personal enclave, sensitive work |
| `member-read` | `rwxr-x---` | Full | Read + Run | None | Members can view and run; only owner deploys |
| `member-edit` | `rwxrwx---` | Full | Full | None | **DEFAULT** — full team collaboration |
| `open-read` | `rwxrwxr--` | Full | Full | Read-only | Leadership/stakeholders can view status |
| `open-run` | `rwxrwxr-x` | Full | Full | Read + Run | Open triggers for external consumers |

New enclaves default to `member-edit` (`rwxrwx---`). Members can do everything;
non-members see nothing.

### Two-Layer Authorization

Every operation on a tentacle checks both layers. Both must pass:

1. **Enclave check** — does the caller have the required bit on the enclave?
2. **Tentacle check** — does the caller have the required bit on the tentacle?

Exception: the enclave owner bypasses the tentacle check entirely (superuser).

### Updated Evaluator Flow

```
1. Evaluator disabled? → Allow
2. Bearer-token caller? → Allow (platform operators only)
3. No enclave annotation? → Deny (unmanaged resource)
4. Caller is enclave owner? → Allow (superuser — bypasses tentacle check)
5. Caller is tentacle owner? → Check owner bits (positions 0-2)
6. Caller in enclave-members? → Check member bits (positions 3-5)
7. Otherwise → Check other bits (positions 6-8)
```

Step 6 uses `tentacular.io/enclave-members` annotation, not IdP group claims.

### Permission Scenarios

| Actor | Enclave Mode | Action | Result |
|-------|-------------|--------|--------|
| Owner | `rwxrwx---` | Any action on any tentacle | Allow (superuser) |
| Member | `rwxrwx---` | Deploy new tentacle | Allow (member w=yes) |
| Member | `rwxrwx---` | Modify someone else's tentacle | Depends on tentacle mode |
| Visitor | `rwxrwx---` | List tentacles | Deny (other bits: `---`) |
| Visitor | `rwxrwxr--` | View status | Allow (other r=yes) |
| Visitor | `rwxrwxr--` | Deploy tentacle | Deny (other w=no) |
| Visitor | `rwxrwxr-x` | Trigger a run | Allow (other x=yes) |
| Platform operator | Any | Anything | Allow (bearer-token bypass) |

## Common Workflows

### Create an enclave for your team

The Kraken handles this end-to-end. Tell it: "Set up this channel for our
team." It will:
1. Verify you are the Slack channel owner
2. Ask two sizing questions (data volume, number of automations)
3. Call `enclave_provision` with the appropriate quota preset
4. Walk current channel members through OIDC sign-in
5. Confirm when the enclave is ready

### Add a member to an enclave

When a user joins the Slack channel, The Kraken handles it automatically:
prompts OIDC sign-in, then calls `enclave_sync` with `add_members`. As an
agent operating directly on MCP tools, call `enclave_sync` with `add_members`
after verifying the member has completed OIDC authentication.

### Transfer tentacle ownership

Only the enclave owner can do this. Use `enclave_sync` once `enclave_chown`
is available, or use the bearer-token kubectl path as a break-glass measure.
The Kraken exposes this via: "Transfer [tentacle] to [user]."

### Freeze an enclave

Call `enclave_sync` with `new_status: "frozen"`. Existing tentacles keep running;
cron triggers are paused; `wf_apply` will be rejected until unfrozen. This
happens automatically when a Slack channel is archived.

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Checking enclave membership via Keycloak groups | Groups are no longer the authority — Keycloak is identity-only | Check `enclave-members` annotation via `enclave_info` |
| Assuming the namespace name matches the channel display name | Channel rename updates the display name but not the namespace slug | Use `enclave_info` to get the current `name` slug |
| Operating on a frozen enclave | `wf_apply` rejected, cron paused | Call `enclave_sync` with `status: "active"` to unfreeze |
| Using bearer token as a regular user | Bypasses authz — resources become unowned | Bearer token is for platform operators only; users must use OIDC |

## Annotations Reference

All enclave metadata is stored on the K8s namespace:

```
tentacular.io/enclave               # enclave name slug
tentacular.io/enclave-owner         # OIDC email of owner
tentacular.io/enclave-owner-sub     # OIDC subject of owner
tentacular.io/enclave-members       # comma-separated OIDC emails
tentacular.io/enclave-platform      # "slack"
tentacular.io/enclave-channel-id    # platform channel ID
tentacular.io/enclave-channel-name  # current display name
tentacular.io/enclave-status        # "active" | "frozen"
tentacular.io/enclave-created-at    # RFC3339 timestamp
tentacular.io/enclave-updated-at    # RFC3339 timestamp
```

Tentacles within an enclave retain their own `tentacular.io/owner`,
`tentacular.io/mode`, etc. annotations. The enclave annotations live on the
namespace; tentacle annotations live on the Deployment.
