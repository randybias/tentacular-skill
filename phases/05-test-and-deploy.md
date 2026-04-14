# Phase 05: Test and Deploy

**Prerequisites: phases 01–04 complete. All nodes tested individually.**

For fixture format, mock context, and advanced test patterns: [Testing Guide](https://randybias.github.io/tentacular-docs/guides/testing/).

## Testing Sequence

Run in order. Each gate must pass before the next step.

### Gate 1: Spec validation

```bash
tntc validate
```

Must exit 0. Fix any errors before proceeding.

### Gate 2: Secrets check

```bash
tntc secrets check
```

All referenced secrets must be provisioned. If missing: `tntc secrets init`, then fill
in `.secrets.yaml`. Do not deploy with unresolved secrets.

### Gate 3: Node tests (all nodes)

```bash
tntc test
```

Every node must pass. If any fail: fix the node. Do not move to pipeline test
while individual node tests are failing.

### Gate 4: Pipeline test

```bash
tntc test --pipeline
```

This runs the full DAG end-to-end. After it passes:

**Verify data flowed correctly.** Inspect the pipeline test output:
- Final output must contain meaningful end-to-end data — not `{}`, not a status string
- If any intermediate node returned empty data, the pipeline will either fail or
  produce wrong final output — both are bugs

If pipeline output is empty or wrong: go back to `phases/04-build.md` and find
the node that broke the chain.

### Gate 5: Live test (required before any production deploy)

```bash
tntc test --live --cluster <your-cluster>
```

Replace `<your-cluster>` with the target cluster name (e.g., `dev`, `staging`). No production deploy without a passing live test. No exceptions.

If the target cluster is unavailable: stop and resolve that before deploying to prod.

---

## Cron Workflow Testing

Cron workflows fire on a schedule managed by the MCP server's internal
scheduler. The schedule is stored in a `tentacular.io/cron-schedule` annotation
on the Deployment -- no CronJob resources are created. **You must manually
trigger cron workflows for testing.** Never wait for the schedule to fire to
verify a fresh deploy.

### Step 1: Deploy first, then trigger manually

```bash
tntc deploy --cluster <target>
tntc status <workflow-name>   # wait for ready
```

If deploying with new `jsr:` or `npm:` module dependencies for the first time,
the MCP server pre-warms the module proxy cache in the background. There is a
brief race window where the workflow pod may fail on its first start with a
module resolution timeout. Wait for the pod to restart (one restart is normal)
and confirm readiness before proceeding:

```bash
wf_pods enclave=<enclave>   # check for 1 restart, then ready
```

Then trigger manually:

```bash
tntc run <workflow-name>      # manual trigger — fires the workflow immediately
```

### Step 2: Check logs — do not undeploy immediately

After `tntc run`, the workflow pod runs to completion and its logs persist briefly. **Keep the workflow deployed** until you have confirmed the output is correct.

```bash
KUBECONFIG=/full/path/to/kubeconfig tntc logs <workflow-name> -n <enclave>
```

If `tntc logs` returns nothing yet, the pod may still be running. Poll until output appears:

```bash
# Check pod status directly if tntc logs is empty:
KUBECONFIG=/full/path/to/kubeconfig kubectl get pods -n <namespace> -l workflow=<workflow-name>
```

### Step 3: Verify output before trusting the schedule

The manual trigger output must contain correct end-to-end data — not `{}`, not a status string. Only after confirming real output should you trust the scheduled run.

**Do not mark a cron workflow as verified by the schedule alone.** The first scheduled fire can happen hours later and may silently fail if the deploy had issues.

---

## Deploy

Only after all 5 gates pass:

```bash
tntc build --push
tntc deploy --cluster <target>

# Optional: deploy to a specific enclave
tntc deploy --cluster <target> --enclave <enclave-name>

# Optional: set permission mode at deploy time
tntc deploy --cluster <target> --mode member-read
```

### Post-deploy verification

```bash
tntc status <workflow-name>
```

Wait for status to show ready. Then trigger the workflow:

```bash
tntc run <workflow-name>
```

Inspect the output. It must contain the correct end-to-end data — not just `{"status":"ok"}`.

### Post-deploy health check

After verifying the run output, confirm the workflow
reports healthy via the MCP health tools:

1. Use `wf_health` (MCP tool) to check the workflow's
   G/A/R status. It should report **green**.
2. If amber or red, use `wf_health` with `detail=true`
   to get execution telemetry, then `wf_logs` for
   pod-level diagnostics.
3. For enclave-wide checks, use `wf_health_enclave` to
   confirm all workflows in the enclave are green.

A green health status after a successful `tntc run`
confirms the workflow is fully operational.

If the live output is wrong or empty after a successful deploy: the workflow has
a data-flow bug that tests didn't catch. Return to `phases/04-build.md`, trace
the data flow, and find the performative or broken node.

### Post-deploy permissions management

When deploying with OIDC authentication, the workflow is automatically
assigned owner, member, and mode permissions. To adjust permissions after
deploy, use `enclave_sync` (MCP tool) or the `@kraken set mode` command.
Only the enclave owner can change the enclave permission mode.

---

## Querying Deployed Workflows

`tntc list`, `tntc status`, `tntc logs`, and `tntc run` do not support `--cluster`.
You must resolve the actual kubeconfig path and namespace from config first:

```bash
# Step 1: read the actual values from config
cat ~/.tentacular/config.yaml
# Find your cluster block, e.g.:
#   prod:
#     kubeconfig: ~/secrets/prod.kubeconfig   ← this is the path
#     namespace: tentacular-prod              ← this is the namespace

# Step 2: expand ~ manually — do NOT pass literal ~
# e.g. ~/secrets/prod.kubeconfig → /Users/yourname/secrets/prod.kubeconfig

# Step 3: run with resolved values
KUBECONFIG=/Users/yourname/secrets/prod.kubeconfig tntc list -n tentacular-prod
KUBECONFIG=/Users/yourname/secrets/prod.kubeconfig tntc status <name> -n tentacular-prod
KUBECONFIG=/Users/yourname/secrets/prod.kubeconfig tntc logs <name> -n tentacular-prod
```

Do not use `<cluster.kubeconfig>` literally. Do not pass `~` unexpanded.

---

## Pre-Deploy Checklist

- [ ] Cluster profile exists and is < 7 days old (phases/03-profile.md)
- [ ] Contract section present and complete in workflow.yaml
- [ ] `tntc validate` passes
- [ ] All individual node tests pass
- [ ] `tntc test --pipeline` passes with correct final output
- [ ] `tntc test --live --cluster <your-cluster>` passes
- [ ] `workflow-diagram.md` and `contract-summary.md` generated: `tntc visualize --rich --write`
- [ ] `tntc run` post-deploy output verified (not empty, not a status string)

---

## Known Issues

### tntc build --push requires Docker

`tntc build --push` requires a running Docker daemon. If Docker is unavailable,
deploy using the pre-existing engine image:

```bash
tntc deploy --cluster <target> --skip-live-test
```

The engine image (`ghcr.io/randybias/tentacular-engine:latest`) is shared across
all workflows — it does not need to be rebuilt per workflow.

---

## Exposing Workflows to External Traffic

For workflows with `queue` or `webhook` triggers that need to receive traffic from
outside the cluster (e.g. GitHub webhooks, external event sources):

```bash
# Apply the ingress + service from the tentacular repo:
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/webhook/ingress.yaml
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/webhook/service.yaml
```

Check the ingress for the required DNS name and TLS configuration. The ingress
assumes cert-manager with a `letsencrypt-prod` cluster-issuer is present.

---

## Undeploy

```bash
tntc undeploy <workflow-name>
```

Removes: Deployment (and cron annotation), Service, Secret, ConfigMap,
NetworkPolicy. No CronJob resources are created, so none need cleanup. The MCP
server drops cron entries automatically when the Deployment is removed. Verify
with `tntc list`.
