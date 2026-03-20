# Contract Model Reference

## Overview

The `contract` section declares every external dependency a workflow needs.
Dependencies are the single primitive: secrets, NetworkPolicy, connection
config, and validation are all derived from the dependency list. There is no
separate `secrets` or `networkPolicy` section to author.

For the complete contract specification, including structure, dependency
protocols, auth types, enforcement modes, drift detection, dynamic-target
dependencies, label-scoped ingress, and NetworkPolicy overrides, read the
[Node Contract](https://randybias.github.io/tentacular-docs/reference/node-contract/)
and [Security](https://randybias.github.io/tentacular-docs/concepts/security/) docs.

## Dependency Declaration

Dependencies named with the `tentacular-` prefix are auto-provisioned by the
MCP server's exoskeleton control plane. Only `protocol` is required -- host,
port, database, user, and auth are filled automatically at deploy time. The
MCP server rejects deployment if the requested exoskeleton service is
disabled on the cluster.

Known `tentacular-*` dependencies:

| Name | Protocol | Service |
|------|----------|---------|
| `tentacular-postgres` | `postgresql` | Scoped Postgres schema and role |
| `tentacular-nats` | `nats` | Scoped NATS subjects and credentials |
| `tentacular-rustfs` | `s3` | Scoped RustFS prefix and credentials |

## Exoskeleton Services

The exoskeleton is an optional backing-service bundle (Postgres, NATS,
RustFS) managed by the MCP server. When enabled, it provides deterministic,
per-workflow scoped access to these services without manual credential
provisioning.

### Detection

Always call `exo_status` before building a workflow that needs database,
messaging, or object storage. The response indicates which exoskeleton
services are enabled on the cluster. It also reports `auth_enabled` and
`auth_issuer` -- use these to determine whether SSO login is required before
deploying.

## Authentication and SSO

When `exo_status` returns `auth_enabled: true`, or when `mcp_endpoint` is
configured for an environment, the cluster requires OIDC authentication. The
user must configure SSO and authenticate before deploying.

### SSO Setup

Configure OIDC credentials for an environment using `tntc configure`. This
can be done non-interactively (agent-safe) or via guided prompts:

```bash
# Non-interactive (all values via flags)
tntc configure -e <env> \
  --oidc-issuer <issuer-url> \
  --oidc-client-id <client-id> \
  --oidc-client-secret <client-secret> \
  --mcp-endpoint <mcp-url>

# Interactive guided setup (prompts for missing values)
tntc configure --sso -e <env>
```

When `oidc_client_secret` is present, the config file is written with
restricted permissions (0600).

### Login Flow

1. Check auth status: if `exo_status` shows `auth_enabled: true`, instruct
   the user to run `tntc login` before deploying.
2. `tntc login -e <env>` initiates a browser-based login via Google SSO
   through Keycloak. The CLI opens the browser automatically or prints a URL
   if browser launch fails. Google SSO is restricted to an
   administrator-configured domain. Only Google accounts from the allowed
   domain can authenticate.
3. `tntc whoami` confirms the authenticated identity.
4. Once logged in, subsequent `tntc deploy` commands include the OIDC token
   automatically.
5. `tntc logout` clears the local auth token.

Token refresh is automatic. If the refresh token expires, the CLI prompts
the user to run `tntc login` again.

## Deployer Provenance and Authorization

When SSO auth is active, the MCP server annotates Deployment manifests with
deployer identity and authorization metadata:

**Provenance annotations:**
- `tentacular.io/deployed-by`: deployer email (legacy alias for owner-email)
- `tentacular.io/deployed-at`: deployment timestamp
- `tentacular.io/deployed-via`: agent type (cli, etc.)

**Authorization annotations (stamped on CREATE, preserved on UPDATE):**
- `tentacular.io/owner-sub`: owner's OIDC subject identifier
- `tentacular.io/owner-email`: owner's email address
- `tentacular.io/owner-name`: owner's display name
- `tentacular.io/group`: group assignment (from `--group` flag or empty)
- `tentacular.io/mode`: permission string (e.g., `rwxr-x---`)
- `tentacular.io/auth-provider`: authentication provider type (e.g., `keycloak`, `bearer-token`)
- `tentacular.io/created-at`: creation timestamp (set once on first deploy)

**Audit annotations (stamped on UPDATE):**
- `tentacular.io/updated-at`: last update timestamp
- `tentacular.io/updated-by-sub`: last updater's OIDC subject
- `tentacular.io/updated-by-email`: last updater's email

These annotations are visible in `wf_describe` output. Use `permissions_get`
to check the effective owner, group, and mode. Only the owner can modify
permissions via `permissions_set`.

**Permission presets:**

| Preset | Mode | Meaning |
|--------|------|---------|
| `private` | `rwx------` | Owner only |
| `group-read` | `rwxr-x---` | Owner full, group can read and execute (default) |
| `group-run` | `rwx--x---` | Owner full, group can execute only |
| `group-edit` | `rwxrwx---` | Owner and group have full access |
| `public-read` | `rwxr--r--` | Owner full, everyone can read |

Bearer-token deploys bypass authorization entirely. To disable authz
server-wide, set `TENTACULAR_AUTHZ_ENABLED=false`.

## When to Use Managed vs Manual Dependencies

Use `tentacular-*` dependency names when `exo_status` shows the target
service is enabled. The MCP server provisions scoped credentials and injects
them automatically.

Use manually configured dependencies (with explicit host, port, auth) when:

- The exoskeleton is disabled on the cluster.
- The service is external (not managed by the exoskeleton).
- The user requires a specific external instance.

## Mixed Dependencies Example

A single contract can combine exoskeleton-managed and manually configured
dependencies. Node code is identical either way -- `ctx.dependency()` returns
a `DependencyConnection` with the same fields regardless of how the
dependency was provisioned.

**With exoskeleton (Postgres auto-provisioned, GitHub manual):**

```yaml
contract:
  version: "1"
  dependencies:
    tentacular-postgres:
      protocol: postgresql
    github-api:
      protocol: https
      host: api.github.com
      auth:
        type: bearer-token
        secret: github-api.token
```

**Without exoskeleton (both manually configured):**

```yaml
contract:
  version: "1"
  dependencies:
    my-postgres:
      protocol: postgresql
      host: my-db.example.com
      port: 5432
      database: mydb
      user: myuser
      auth:
        type: password
        secret: my-postgres.password
    github-api:
      protocol: https
      host: api.github.com
      auth:
        type: bearer-token
        secret: github-api.token
```

Node code for either case:

```typescript
// With exoskeleton:
const pg = ctx.dependency("tentacular-postgres");
// Without exoskeleton:
const pg = ctx.dependency("my-postgres");
// Both return: pg.host, pg.port, pg.database, pg.user, pg.secret
```

## Anti-Confusion Rules

- NEVER add `tentacular-*` dependencies without checking `exo_status` first.
- NEVER add `host`, `port`, `database`, or `auth` fields to a `tentacular-*`
  dependency -- the MCP server overwrites them during provisioning.
- ALWAYS ask the user whether they want exoskeleton-managed or self-managed
  services when the choice is ambiguous.
- A workflow with `tentacular-*` dependencies WILL FAIL on a cluster without
  the exoskeleton. This is by design, not a bug. The error message is
  explicit.
- If `exo_status` returns `auth_enabled: true`, instruct the user to run
  `tntc login` BEFORE deploying. A deploy without a valid OIDC token will
  fail with an authentication error.

## Deploy Decision Tree

Before deploying a workflow with exoskeleton dependencies:

1. Call `exo_status` to check service availability.
2. If `auth_enabled` is `true`:
   a. Ask the user to run `tntc whoami` to check login status.
   b. If not logged in, instruct: `tntc login`.
   c. Confirm identity with `tntc whoami`.
3. Verify required services are available (e.g., `postgres_available`,
   `nats_available`).
4. Proceed with `tntc deploy`.

## NATS Isolation and Known Limitations

When SPIFFE mode is enabled (`TENTACULAR_NATS_SPIFFE_ENABLED=true`), NATS
subject isolation is cryptographically enforced via mTLS with SPIRE SVIDs
and per-tentacle authorization rules. This is the recommended configuration.

The NATS server TLS certificate is managed by cert-manager via an internal
CA (`tentacular-internal-ca`). The server cert has 1-year validity and is
auto-renewed 30 days before expiry. NATS trusts both the cert-manager CA
(server cert chain) and the SPIRE CA (client SVIDs) via a combined trust
bundle.

Token mode is available as a fallback for clusters without SPIRE. In token
mode, subject isolation is convention-only -- any workflow with NATS access
can publish/subscribe to any subject.

Remaining manual step: When SPIRE rotates its CA, the combined trust bundle
in the `nats-spire-ca` Secret must be refreshed manually. A future sidecar
or CronJob will automate this sync.

## Prerequisites

The exoskeleton admin credentials configured on the MCP server must meet
these requirements:

- **Postgres:** The admin user must have `CREATEROLE` privilege. `SUPERUSER`
  is NOT required.
- **RustFS:** The `tentacular` bucket must exist before workflows can use
  `tentacular-rustfs`. The MCP server attempts to create it on startup, but
  the admin user must have bucket-creation permissions.
- **SPIRE:** The MCP service account's ClusterRole must include permissions
  for `spire.spiffe.io` resources (e.g., `clusterspiffeids`). The Helm chart
  includes this by default, but verify on live clusters.

## Cleanup Behavior

`cleanup_on_undeploy` defaults to `false`. When cleanup is disabled (the
default), `wf_remove` deletes only Kubernetes resources. All backing-service
data -- Postgres schemas, RustFS objects, NATS artifacts -- is preserved. A
subsequent deploy of the same workflow reconnects to the existing data.

When `cleanup_on_undeploy` is `true`, `wf_remove` drops exoskeleton resources
for the workflow:

- **Postgres:** Drops the workflow's schema and role
  (`DROP SCHEMA ... CASCADE`).
- **RustFS:** Deletes the workflow's objects, service user, and access
  policies.
- **NATS:** Revokes credentials and auth artifacts. In SPIFFE mode, removes
  the authorization ConfigMap entry.

This is destructive and permanent. Cleanup runs best-effort for all
configured services, not only those the workflow declared.

**Agent decision before calling `wf_remove`:** When removing a workflow that
has exoskeleton dependencies, first call `exo_status` and check
`cleanup_on_undeploy`. If it is `true`, warn the user explicitly before
proceeding:

> Removing this workflow will permanently delete its Postgres schema, RustFS
> objects, and NATS artifacts. This cannot be undone. Do you want to proceed?

Only call `wf_remove` after the user confirms. If `cleanup_on_undeploy` is
`false`, no data-loss warning is needed -- only Kubernetes resources will be
removed.
