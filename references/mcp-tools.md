# MCP Tools Reference

For parameter schemas, use MCP `tools/list` at runtime. This file covers
tool behavior, response semantics, and usage patterns. The SKILL.md safety
classification table tells you the risk level of each tool.

The tentacular MCP server exposes 38 tools organized into 14 groups. Agents
can discover all tools and their full parameter schemas via the MCP
`tools/list` method -- no `tntc` CLI or `KUBECONFIG` needed.

## Workflow Discovery

### wf_list

Lists all tentacular-managed workflow deployments. Filters by label selector
`app.kubernetes.io/managed-by=tentacular`. Can filter by namespace, owner
annotation, or tag annotation. Requires namespace Read to list tentacles
in a namespace.

Returns an array of workflow entries, each with: `name`, `namespace`,
`version`, `owner` (from `tentacular.io/owner-email`), `group`, `environment`, `ready`, `age`.

### wf_describe

Returns detailed information about a single workflow deployment, including
metadata annotations, replica status, node list, trigger configuration,
and authorization info.

Returns: `name`, `namespace`, `version`, `owner` (from
`tentacular.io/owner-email`), `group`, `tags`, `environment`, `mode`,
`ready`, `replicas`, `ready_replicas`, `image`, `age`, `nodes`,
`triggers`, `annotations` (all `tentacular.io/*` annotations).

Node names and trigger descriptions are enriched from the workflow ConfigMap
(`<name>-code`) when available.

## Workflow Execution

### wf_run

Triggers a deployed workflow by POSTing directly to the workflow's `/run`
endpoint via HTTP. The MCP server in tentacular-system connects directly to
the workflow service; NetworkPolicy allows ingress from tentacular-system via
namespaceSelector. Returns the JSON output. No ephemeral pods are created.

Returns: `name`, `namespace`, `output` (structured JSON), `duration_ms`.

Default timeout: 120 seconds, max 600 seconds.

## Workflow Lifecycle

### wf_apply

Apply a set of Kubernetes manifests as a named deployment in a namespace.
Uses release labels for tracking and garbage collection. Includes garbage
collection of stale resources from previous deployments. Creating a new
tentacle requires namespace Write; updating an existing tentacle requires
tentacle Write.

Allowed manifest kinds: Deployment, Service, PersistentVolumeClaim,
NetworkPolicy, ConfigMap, Secret, Job, CronJob, Ingress.

Returns: `name`, `namespace`, `created` count, `updated` count, `deleted`
count (garbage-collected resources no longer in the manifest set).

### wf_remove

Remove all resources belonging to a named deployment in a namespace. When
exoskeleton cleanup is enabled (`cleanup_on_undeploy: true`), also drops
backing-service data (Postgres schema, RustFS objects, NATS artifacts).
This is destructive and permanent -- confirm with the user before calling.

### wf_status

Get status of all resources belonging to a named deployment in a namespace.
Read-only. Use `detail=true` to include resource-level readiness.

### wf_restart

Perform a rollout restart of a deployment in a managed namespace by patching
the pod template with a `tentacular.io/restartedAt` annotation (same
mechanism as `kubectl rollout restart`).

Common use cases:
- ConfigMap/Secret changes: Kubernetes does not auto-restart pods when
  mounted ConfigMaps or Secrets change.
- Stuck/degraded recovery: pods in CrashLoopBackOff or degraded state are
  replaced gracefully.
- Post-credential rotation: after `cred_rotate` invalidates prior
  ServiceAccount tokens, running deployments need a restart to obtain fresh
  tokens.
- gVisor enablement: after `gvisor_annotate_ns` annotates a namespace,
  existing pods continue on the old runtime. A restart forces new pods onto
  gVisor.

## Workflow Observability

### wf_logs

Get pod logs from a namespace. Returns tail lines (default 100).
Requires namespace Read when no tentacle name is specified.

Tip: Use `wf_pods` first to find the pod name, then pass it to `wf_logs`.

### wf_pods

List pods in a namespace with phase, readiness, restart count, images,
and age. Requires namespace Read when no tentacle name is specified.

### wf_events

List events in a namespace sorted by most recent first (default limit 100).
Requires namespace Read when no tentacle name is specified.

### wf_jobs

List Jobs and CronJobs in a namespace. Requires namespace Read when no
tentacle name is specified.

## Workflow Health

### wf_health

Get G/A/R health status of a single workflow deployment. Checks pod
readiness and probes the engine `/health` endpoint. With `detail=true`,
includes execution telemetry from `/health?detail=1`.

Returns: `name`, `namespace`, `status` (green/amber/red), `reason`,
`pod_ready`, `detail` (when requested).

### wf_health_ns

Aggregate G/A/R health status for all tentacular workflow deployments in a
namespace. Requires namespace Read. Returns per-workflow status and a summary
with green/amber/red counts.

Returns: `namespace`, `summary` (green/amber/red counts), `workflows` (array
of name/status/reason entries), `truncated`, `total`.

## G/A/R Health Model

Health status uses a three-level classification:

| Status | Meaning | Conditions |
|--------|---------|------------|
| Green | Healthy | Pod ready, health endpoint reachable, no failure signals |
| Amber | Degraded | Pod ready but last execution failed or execution in flight |
| Red | Unhealthy | Pod not ready or health endpoint unreachable |

## Standard Reports

**Workflow Listing Report** (from `wf_health_ns`):

```
Namespace: production
Workflows: 5 total | 3 green | 1 amber | 1 red

NAME                      VERSION  REPLICAS  HEALTH  LAST RUN         DURATION
uptime-prober             1.0      1/1       green   2m ago (ok)      1.2s
slack-notifier            1.0      1/1       amber   5m ago (failed)  0.8s
data-collector            1.0      0/1       red     --               --
```

**Workflow Detail Report** (from `wf_health` with `detail=true`):

```
Workflow: slack-notifier
Namespace: production
Health: AMBER -- last execution failed
Last Run: 5m ago | Duration: 0.8s
Totals: 142 runs | 140 succeeded | 2 failed
Recommended next steps:
  - Check logs: tntc logs slack-notifier -n production
  - Re-run: tntc run slack-notifier -n production
```

## Progressive Disclosure

Use health checks in a layered approach:

1. **Quick scan** -- `wf_health_ns` for namespace-wide overview. If all
   green, no further action needed.
2. **Drill down** -- `wf_health` (without detail) for any amber or red
   workflows to get the reason.
3. **Deep dive** -- `wf_health` with `detail=true` for execution telemetry,
   then `wf_logs` for pod logs.

## Namespace Management

### ns_create

Create a new managed namespace with network policies, resource quotas, and
workflow RBAC. Quota preset options: `small`, `medium`, `large`.

When OIDC auth is active, the caller becomes the namespace owner. Accepts
`group` (group name), `mode` (raw rwx string), and `share` (preset name)
parameters for namespace-level permissions. Also accepts `default_mode` and
`default_group` to set inheritance defaults for new tentacles in the namespace.

### ns_delete

Delete a managed namespace. Only namespaces with the tentacular managed-by
label can be deleted. This is destructive.

### ns_get

Get details for a namespace including labels, status, quota summary, and
limit range summary.

### ns_update

Update labels, annotations, and/or resource quota preset on a
tentacular-managed namespace. Rejects changes to the `managed-by` label.
Requires at least one update field.

### ns_list

List all namespaces managed by tentacular. No parameters required.

## Credentials

### cred_issue_token

Issue a short-lived token for the tentacular-workflow service account in a
namespace. Token lifetime: 10-1440 minutes. Each call creates a new token
(not idempotent).

### cred_kubeconfig

Generate a kubeconfig for the tentacular-workflow service account in a
namespace. Token lifetime: 10-1440 minutes. Each call creates a new
kubeconfig (not idempotent).

### cred_rotate

Rotate the workflow service account in a namespace, invalidating all existing
tokens. This is destructive -- all running pods that rely on the current
service account token will lose access until restarted.

## Cluster Operations

### cluster_preflight

Run preflight checks for a namespace: API reachability, namespace existence,
RBAC, and gVisor availability.

### cluster_profile

Profile the cluster: Kubernetes version, distribution, nodes, runtime
classes, CNI, storage, and extensions. Optionally includes quota and limit
range details for a specific namespace.

## Cluster Health

### health_nodes

List nodes with readiness, capacity, allocatable resources, kubelet version,
and unhealthy conditions.

### health_ns_usage

Compare namespace resource usage against ResourceQuota limits and return
utilization percentages.

### health_cluster_summary

Aggregate cluster-wide CPU, memory, and pod counts across all nodes.

## Security Audit

### audit_rbac

Audit RBAC in a namespace: scan for wildcard verbs/resources, sensitive
access, privilege escalation verbs (bind, escalate, impersonate), and
ClusterRoleBindings targeting namespace service accounts. All findings include
actionable remediation suggestions.

### audit_netpol

Audit network policies in a namespace: check for default-deny policy, missing
egress restrictions, overly broad allow rules, cross-namespace ingress via
empty namespaceSelector, and list all policies. All findings include
actionable remediation suggestions.

### audit_psa

Audit Pod Security Admission labels on a namespace: check enforce/audit/warn
levels, distinguish privileged from baseline, detect level mismatches, and
flag non-restricted or missing configuration. All findings include actionable
remediation suggestions.

## gVisor Runtime Sandbox

### gvisor_check

Check whether a gVisor RuntimeClass is available in the cluster. Read-only.
No parameters.

### gvisor_annotate_ns

Annotate a managed namespace with the gVisor runtime class annotation.
Existing pods continue on the old runtime; use `wf_restart` to force new
pods onto gVisor. Idempotent.

### gvisor_verify

Verify gVisor sandboxing by creating an ephemeral verification pod with the
gVisor runtime class and checking kernel identity. The pod is created and
deleted in a single operation (net effect is zero).

## Exoskeleton

### exo_status

Check exoskeleton service availability on the cluster. Returns the
enabled/disabled state of each exoskeleton service (Postgres, NATS, RustFS)
and whether the exoskeleton is enabled overall. Also reports auth/SSO status.

Call this before adding `tentacular-*` dependencies to a workflow contract.

Returns: `enabled`, `cleanup_on_undeploy`, `postgres_available`,
`nats_available`, `rustfs_available`, `spire_available`,
`nats_spiffe_enabled`, `auth_enabled`, `auth_issuer`.

### exo_registration

Check the exoskeleton registration state of a deployed workflow. Shows which
services the workflow is registered with and the current credential/scope
status.

Returns: `found`, `namespace`, `name`, `data` (map of Secret key/value
pairs, sensitive values redacted).

### exo_list

List all workflows with exoskeleton registrations by scanning Secrets across
all namespaces. Read-only. No parameters.

## Permissions

### Tentacle Permissions

### permissions_get

Get owner, group, and mode for a deployed workflow. Read-only.

Parameters: `namespace`, `name`.

Returns: `namespace`, `name`, `owner_sub`, `owner_email`, `owner_name`,
`group`, `mode` (rwx string, e.g., `rwxr-x---`), `preset` (matching
preset name or empty), `auth_provider`.

### permissions_set

Set group or share preset for a deployed workflow. Only the workflow
owner can call this tool. When authenticated via bearer token (no OIDC
identity), authz is bypassed entirely. At least one of `group` or
`share` must be provided.

Parameters: `namespace`, `name`, `group` (optional, new group name),
`share` (optional, preset name: `private`, `group-read`, `group-run`,
`group-edit`, `public-read`).

Returns: `namespace`, `name`, `group`, `mode` (rwx string), `preset`.

### Namespace Permissions

### ns_permissions_get

Get owner, group, and mode for a namespace. Read-only.

Parameters: `namespace`.

Returns: `namespace`, `owner_sub`, `owner_email`, `owner_name`, `group`,
`mode` (rwx string), `preset`, `default_mode`, `default_group`.

### ns_permissions_set

Set group, mode, or share preset for a namespace. Only the namespace
owner can call this tool. Bearer-token requests bypass authz. At least
one of `group`, `mode`, or `share` must be provided.

Parameters: `namespace`, `group` (optional), `mode` (optional, raw rwx
string), `share` (optional, preset name).

Returns: `namespace`, `group`, `mode` (rwx string), `preset`.

## Module Proxy

### proxy_status

Check the installation and readiness status of the module proxy (esm.sh).

Returns: `installed`, `ready`, `namespace`, `image`, `storage`.

## Allowed Manifest Kinds

`wf_apply` accepts: Deployment, Service, PersistentVolumeClaim,
NetworkPolicy, ConfigMap, Secret, Job, CronJob, Ingress.
