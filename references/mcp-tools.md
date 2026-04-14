# MCP Tools Reference

For parameter schemas, use MCP `tools/list` at runtime. This file covers
tool behavior, response semantics, and usage patterns. The SKILL.md safety
classification table tells you the risk level of each tool.

The tentacular MCP server exposes 27 tools organized into 10 groups. Agents
can discover all tools and their full parameter schemas via the MCP
`tools/list` method -- no `tntc` CLI or `KUBECONFIG` needed.

## Workflow Discovery

### wf_list

Lists all tentacular-managed workflow deployments. Filters by label selector
`app.kubernetes.io/managed-by=tentacular`. Can filter by enclave, owner
annotation, or tag annotation. Requires enclave Read to list tentacles
in an enclave.

Returns an array of workflow entries, each with: `name`, `namespace`,
`version`, `owner` (from `tentacular.io/owner-email`), `mode`, `environment`, `ready`, `age`.

### wf_describe

Returns detailed information about a single workflow deployment, including
metadata annotations, replica status, node list, trigger configuration,
and authorization info.

Returns: `name`, `namespace`, `version`, `owner` (from
`tentacular.io/owner-email`), `tags`, `environment`, `mode`,
`ready`, `replicas`, `ready_replicas`, `image`, `age`, `nodes`,
`node_descriptions`, `triggers`, `annotations` (all `tentacular.io/*`
annotations).

Node names and trigger descriptions are enriched from the workflow ConfigMap
(`<name>-code`) when available. `node_descriptions` is an array of
`{name, description}` objects read from the `node_descriptions` key in the
metadata ConfigMap. For workflows deployed before the description requirement,
this array is empty.

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

Apply a set of Kubernetes manifests as a named deployment in an enclave.
Uses release labels for tracking and garbage collection. Includes garbage
collection of stale resources from previous deployments. Creating a new
tentacle requires enclave Write; updating an existing tentacle requires
tentacle Write.

Allowed manifest kinds: Deployment, Service, PersistentVolumeClaim,
NetworkPolicy, ConfigMap, Secret, Job, CronJob, Ingress.

Returns: `name`, `namespace`, `created` count, `updated` count, `deleted`
count (garbage-collected resources no longer in the manifest set).

### wf_remove

Remove all resources belonging to a named deployment in an enclave. When
exoskeleton cleanup is enabled (`cleanup_on_undeploy: true`), also drops
backing-service data (Postgres schema, RustFS objects, NATS artifacts).
This is destructive and permanent -- confirm with the user before calling.

### wf_status

Get status of all resources belonging to a named deployment in an enclave.
Read-only. Use `detail=true` to include resource-level readiness.

### wf_restart

Perform a rollout restart of a deployment in a managed enclave by patching
the pod template with a `tentacular.io/restartedAt` annotation (same
mechanism as `kubectl rollout restart`).

Common use cases:
- ConfigMap/Secret changes: Kubernetes does not auto-restart pods when
  mounted ConfigMaps or Secrets change.
- Stuck/degraded recovery: pods in CrashLoopBackOff or degraded state are
  replaced gracefully.
- Runtime class changes: after enclave annotations are updated, existing
  pods continue on the old runtime. A restart forces new pods onto the
  configured runtime class.

## Workflow Observability

### wf_logs

Get pod logs from an enclave. Returns tail lines (default 100).
Requires enclave Read when no tentacle name is specified.

Tip: Use `wf_pods` first to find the pod name, then pass it to `wf_logs`.

### wf_pods

List pods in an enclave with phase, readiness, restart count, images,
and age. Requires enclave Read when no tentacle name is specified.

### wf_events

List events in an enclave sorted by most recent first (default limit 100).
Requires enclave Read when no tentacle name is specified.

### wf_jobs

List Jobs and CronJobs in an enclave. Requires enclave Read when no
tentacle name is specified.

## Workflow Health

### wf_health

Get G/A/R health status of a single workflow deployment. Checks pod
readiness and probes the engine `/health` endpoint. With `detail=true`,
includes execution telemetry from `/health?detail=1`.

Returns: `name`, `namespace`, `status` (green/amber/red), `reason`,
`pod_ready`, `detail` (when requested).

### wf_health_enclave

Aggregate G/A/R health status for all tentacular workflow deployments in an
enclave. Requires enclave Read. Returns per-workflow status and a summary
with green/amber/red counts.

Returns: `enclave`, `summary` (green/amber/red counts), `workflows` (array
of name/status/reason entries), `truncated`, `total`.

## G/A/R Health Model

Health status uses a three-level classification:

| Status | Meaning | Conditions |
|--------|---------|------------|
| Green | Healthy | Pod ready, health endpoint reachable, no failure signals |
| Amber | Degraded | Pod ready but last execution failed or execution in flight |
| Red | Unhealthy | Pod not ready or health endpoint unreachable |

## Standard Reports

**Workflow Listing Report** (from `wf_health_enclave`):

```
Enclave: production
Workflows: 5 total | 3 green | 1 amber | 1 red

NAME                      VERSION  REPLICAS  HEALTH  LAST RUN         DURATION
uptime-prober             1.0      1/1       green   2m ago (ok)      1.2s
slack-notifier            1.0      1/1       amber   5m ago (failed)  0.8s
data-collector            1.0      0/1       red     --               --
```

**Workflow Detail Report** (from `wf_health` with `detail=true`):

```
Workflow: slack-notifier
Enclave: production
Health: AMBER -- last execution failed
Last Run: 5m ago | Duration: 0.8s
Totals: 142 runs | 140 succeeded | 2 failed
Recommended next steps:
  - Check logs: tntc logs slack-notifier -n production
  - Re-run: tntc run slack-notifier -n production
```

## Progressive Disclosure

Use health checks in a layered approach:

1. **Quick scan** -- `wf_health_enclave` for enclave-wide overview. If all
   green, no further action needed.
2. **Drill down** -- `wf_health` (without detail) for any amber or red
   workflows to get the reason.
3. **Deep dive** -- `wf_health` with `detail=true` for execution telemetry,
   then `wf_logs` for pod logs.

## Cluster Operations

### enclave_preflight

Run preflight checks for an enclave: API reachability, enclave existence,
RBAC, and gVisor availability.

### cluster_profile

Profile the cluster: Kubernetes version, distribution, nodes, runtime
classes, CNI, storage, and extensions. Optionally includes quota and limit
range details for a specific enclave.

## Cluster Health

### health_nodes

List nodes with readiness, capacity, allocatable resources, kubelet version,
and unhealthy conditions.

### health_enclave_usage

Compare enclave resource usage against ResourceQuota limits and return
utilization percentages.

### health_cluster_summary

Aggregate cluster-wide CPU, memory, and pod counts across all nodes.

## Security Audit

### audit_rbac

Audit RBAC in an enclave: scan for wildcard verbs/resources, sensitive
access, privilege escalation verbs (bind, escalate, impersonate), and
ClusterRoleBindings targeting enclave service accounts. All findings include
actionable remediation suggestions.

### audit_netpol

Audit network policies in an enclave: check for default-deny policy, missing
egress restrictions, overly broad allow rules, cross-enclave ingress via
empty namespaceSelector, and list all policies. All findings include
actionable remediation suggestions.

### audit_psa

Audit Pod Security Admission labels on an enclave: check enforce/audit/warn
levels, distinguish privileged from baseline, detect level mismatches, and
flag non-restricted or missing configuration. All findings include actionable
remediation suggestions.

## Module Proxy

### proxy_status

Check the installation and readiness status of the module proxy (esm.sh).

Returns: `installed`, `ready`, `namespace`, `image`, `storage`.

## Allowed Manifest Kinds

`wf_apply` accepts: Deployment, Service, PersistentVolumeClaim,
NetworkPolicy, ConfigMap, Secret, Job, CronJob, Ingress.
