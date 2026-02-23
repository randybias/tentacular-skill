# Phase 05: Test and Deploy

**Prerequisites: phases 01–04 complete. All nodes tested individually.**

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
Pass KUBECONFIG and namespace explicitly:

```bash
KUBECONFIG=<env.kubeconfig> tntc list -n <env.namespace>
KUBECONFIG=<env.kubeconfig> tntc status <name> -n <env.namespace>
KUBECONFIG=<env.kubeconfig> tntc logs <name> -n <env.namespace>
```

Find kubeconfig and namespace values in `~/.tentacular/config.yaml`.

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

## Undeploy

```bash
tntc undeploy <workflow-name>
```

Removes: Deployment, Service, Secret, ConfigMap, CronJobs. Verify with `tntc list`.
