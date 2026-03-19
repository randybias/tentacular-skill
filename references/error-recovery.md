# Error Recovery Playbooks

Symptom-to-fix playbooks for common failure scenarios. For health triage
methodology, see the G/A/R Health Model in `references/mcp-tools.md`.

## Deploy Failures

### ImagePullBackOff

Symptom: Pod stuck in ImagePullBackOff phase.

Diagnosis: `wf_pods` shows ImagePullBackOff. Check image name, registry
auth, and image existence.

Fix:
- Correct the image reference in workflow config.
- If using the pre-built engine image, verify
  `ghcr.io/randybias/tentacular-engine:latest` is accessible from the
  cluster.
- If using a private registry, ensure an imagePullSecret is configured in
  the namespace.

### CrashLoopBackOff

Symptom: Pod repeatedly crashes, restart count climbing.

Diagnosis: Run `wf_logs` to see the error output. Common causes:
- Module resolution timeout (first deploy with jsr/npm deps)
- Missing secrets
- Node throws unhandled exception at startup

Fix:
- Module timeout: wait for the next pod restart (normal for first deploy).
  Check `proxy_status` if the problem persists beyond two restarts.
- Missing secrets: run `tntc secrets check`, then provision any missing
  values in `.secrets.yaml` and redeploy.
- Code error: read `wf_logs` to find the exception, fix the node code,
  retest with `tntc test`, redeploy.

### PSA Violation (Forbidden)

Symptom: Pod rejected with "violates PodSecurity" admission message.

Diagnosis: The namespace has restricted Pod Security Admission enforcement.
The MCP server auto-injects PSA-compliant security contexts into `wf_apply`
manifests, but custom manifests submitted outside `wf_apply` may lack them.

Fix: Ensure containers set:
- `runAsNonRoot: true`
- `capabilities.drop: [ALL]`
- `readOnlyRootFilesystem: true`
- `seccompProfile.type: RuntimeDefault`

The `wf_apply` tool handles this automatically. If using manual manifests,
add these fields to the container security context.

### NetworkPolicy Blocks Traffic

Symptom: Workflow pod cannot reach an external dependency. Connection refused
or timeout at runtime.

Diagnosis: Check `contract.dependencies` for the target host. Run
`audit_netpol` to see current policies in the namespace.

Fix: Ensure the dependency is declared in `contract.dependencies`.
NetworkPolicy egress rules are auto-generated from the contract. If the
dependency was added after the last deploy, redeploy the workflow to
regenerate the NetworkPolicy.

### gVisor RuntimeClass Not Found

Symptom: `unknown RuntimeClass "gvisor"` error during pod scheduling.

Diagnosis: Run `gvisor_check` to verify whether the gVisor RuntimeClass is
installed in the cluster.

Fix:
- Install gVisor (see cluster prerequisites in `phases/01-install.md`), or
- Set `runtime_class: ""` in the environment config to disable gVisor
  sandboxing for this deploy.

## Runtime Errors

### Auth Failures (401/403)

Symptom: Node test or live run returns HTTP 401 or 403.

Diagnosis: The dependency's auth configuration is incorrect or the secret
value is stale/expired.

Fix:
1. Check the dependency's `auth.type` and `auth.secret` in `workflow.yaml`.
2. Update `.secrets.yaml` with the correct credential value.
3. Re-run the individual node test with `tntc test` before the pipeline test.
4. If the dependency uses a short-lived token, provision a fresh one.

### Module Resolution Timeout

Symptom: Pod fails on first start with a module import error or timeout.

Diagnosis: First-time deploy with jsr/npm dependencies. The module proxy
cache is not yet warm.

Fix:
- Wait for the pod to restart automatically (Kubernetes restarts the pod).
- By the second attempt, the cache should be warm and the pod will start.
- If the problem persists beyond two restarts, check `proxy_status` to
  verify the module proxy is running and healthy.
- If the proxy is down, the Helm chart reconciler should auto-recover it.

### Dependency Connection Refused

Symptom: Node throws ECONNREFUSED or connection timeout when calling
`ctx.dependency()` methods.

Diagnosis: The target service is unreachable from the cluster pod.

Fix:
1. Verify DNS resolution for the dependency host from within the cluster.
2. Check NetworkPolicy egress rules with `audit_netpol`.
3. Confirm the dependency `host` and `port` are correct in `workflow.yaml`.
4. Confirm the target service is running and accepting connections.

### ctx.dependency() Throws "not declared in contract"

Symptom: Node code calls `ctx.dependency("foo")` but the engine throws
"not declared in contract" at runtime.

Fix:
1. Add the dependency to `workflow.yaml` under `contract.dependencies`.
2. Re-run `tntc validate` to confirm the contract is valid.
3. Redeploy with `tntc deploy`.

## Health Status Triage

### Red: Pod Not Ready

Diagnosis: `wf_health` returns red with "0/N replicas ready."

Steps:
1. `wf_pods` -- check pod phase and restart count.
2. `wf_logs` -- check error output from the pod.
3. `wf_events` -- check for scheduling, resource, or admission issues.

Fix: Address the underlying pod failure (see Deploy Failures above).

### Red: Health Endpoint Unreachable

Diagnosis: Pod is running but the `/health` endpoint returns an error or
times out.

Steps:
1. `wf_status detail=true` -- check resource status.
2. `wf_logs` -- check for engine startup errors.

Fix: The engine may not have started correctly. Check for module resolution
errors, missing config, or a code exception in the root node initialization.

### Amber: Last Execution Failed

Diagnosis: `wf_health detail=true` shows `lastRunFailed: true`.

Steps:
1. `wf_logs` -- find the failed run's error message.
2. Identify whether the failure is transient (network) or persistent (code).

Fix:
- Transient: run `wf_run` to re-trigger and clear the amber status.
- Persistent: fix the node code, redeploy, re-run. Amber clears
  automatically on the next successful run.

### Amber: Execution In Flight

Diagnosis: `wf_health detail=true` shows `inFlight > 0`.

This is informational -- a run is currently executing. No action is needed
unless the run has been in flight longer than the expected timeout (default
120 seconds, max 600 seconds).

Fix: If the run appears stuck beyond the configured timeout, check `wf_logs`
for signs of a hung operation, then consider `wf_restart` to recover.

## Common tntc Command Failures

### "tntc version" fails

Fix: Re-run the install script. See `phases/01-install.md`.

### "unknown command" for cluster profile or other commands

Cause: The installed binary is too old and does not include this command.

Fix: Update tntc by re-running the install script:
```bash
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
```

### "tntc logs" returns empty output

Cause: Pod may still be initializing, or the pod has already terminated and
logs have been garbage-collected.

Fix: Check pod status with `wf_pods`. If the pod is Running, try again in a
moment. If the pod is Terminated, logs may no longer be available -- use
`wf_events` to find clues from the last pod lifecycle.

### KUBECONFIG path errors

Cause: A literal `~` was passed in a KUBECONFIG path instead of the expanded
absolute path.

Fix: Always expand `~` to the full home directory path (e.g.,
`/Users/name/.kube/config`). The tntc CLI does not perform shell expansion
on path values read from config files.

See also `references/deployment-ops.md` for environment configuration
patterns that avoid this issue.
