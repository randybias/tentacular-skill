# Phase 05: Test and Deploy

**Prerequisites: phases 01–04 complete. All nodes tested individually.**

For fixture format, mock context, and advanced test patterns: `references/testing-guide.md`.

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

### Gate 5: Live test against dev (required before any production deploy)

```bash
tntc test --live --env dev
```

No production deploy without a passing live test. No exceptions.

If dev environment is unavailable: stop and resolve that before deploying to prod.

---

## Deploy

Only after all 5 gates pass:

```bash
tntc build --push
tntc deploy --env <target>
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

If the live output is wrong or empty after a successful deploy: the workflow has
a data-flow bug that tests didn't catch. Return to `phases/04-build.md`, trace
the data flow, and find the performative or broken node.

---

## Querying Deployed Workflows

`tntc list`, `tntc status`, `tntc logs`, and `tntc run` do not support `--env`.
You must resolve the actual kubeconfig path and namespace from config first:

```bash
# Step 1: read the actual values from config
cat ~/.tentacular/config.yaml
# Find your environment block, e.g.:
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

Do not use `<env.kubeconfig>` literally. Do not pass `~` unexpanded.

---

## Pre-Deploy Checklist

- [ ] Cluster profile exists and is < 7 days old (phases/03-profile.md)
- [ ] Contract section present and complete in workflow.yaml
- [ ] `tntc validate` passes
- [ ] All individual node tests pass
- [ ] `tntc test --pipeline` passes with correct final output
- [ ] `tntc test --live --env dev` passes
- [ ] `workflow-diagram.md` and `contract-summary.md` generated: `tntc visualize --rich --write`
- [ ] `tntc run` post-deploy output verified (not empty, not a status string)

---

## Known Issues

### tntc build --push requires Docker

`tntc build --push` requires a running Docker daemon. If Docker is unavailable,
deploy using the pre-existing engine image:

```bash
tntc deploy --env <target> --skip-live-test
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

Removes: Deployment, Service, Secret, ConfigMap, CronJobs. Verify with `tntc list`.
