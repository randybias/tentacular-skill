# Phase 02: Configure Environments

**Gate: at least one environment must be defined before any workflow work.**

## Check

```bash
cat ~/.tentacular/config.yaml 2>/dev/null || cat .tentacular/config.yaml 2>/dev/null
```

If the file exists and contains at least one entry under `environments:`, you are done
with this phase. Proceed to `phases/03-profile.md`.

## Create or Update Config

If no config exists, bootstrap a stub with:

```bash
tntc configure --registry <registry-url>   # e.g. ghcr.io/username
```

**`tntc configure` creates a stub — it sets `registry`, `namespace`, and `runtime_class` but does NOT add environment definitions.** You must edit the file to add them. Open `~/.tentacular/config.yaml` (user-level) or `.tentacular/config.yaml` (project-level) and add at minimum one environment block:

```yaml
registry: ghcr.io/your-org
namespace: default
runtime_class: gvisor

environments:
  dev:
    context: kind-local          # kubeconfig context name
    namespace: tentacular-dev
    runtime_class: ""            # no gVisor for kind
  prod:
    kubeconfig: ~/secrets/prod.kubeconfig
    context: prod-admin
    namespace: tentacular-prod
    runtime_class: gvisor
```

Config resolution: CLI flags → project `.tentacular/config.yaml` → user `~/.tentacular/config.yaml`.

For full environment config options: `references/deployment-guide.md`.

## Auto-Profile on Configure

`tntc configure` automatically profiles all reachable environments in newer binary
versions (>= the release that includes `cluster profile`). If auto-profiling was
skipped or the command isn't available, proceed to `phases/03-profile.md` to
generate the profile manually.

## Gate Verification

Before leaving this phase, confirm:

- [ ] Config file exists (`~/.tentacular/config.yaml` or `.tentacular/config.yaml`)
- [ ] At least one environment is defined under `environments:`
- [ ] `tntc cluster check` passes for the target environment

`tntc cluster check` does not support `--env`. Resolve the kubeconfig and namespace
from the config file and pass them explicitly:

```bash
# Read your environment's kubeconfig + namespace from config:
cat ~/.tentacular/config.yaml

# Then run (expand ~ manually to the full path):
KUBECONFIG=/full/path/to/kubeconfig tntc cluster check -n <namespace>

# If the namespace or RBAC doesn't exist yet, use --fix to create them:
KUBECONFIG=/full/path/to/kubeconfig tntc cluster check -n <namespace> --fix
```

| cluster check result | Action |
|----------------------|--------|
| All checks pass ✓ | Proceed to phase 03 |
| `gVisor RuntimeClass` fails | See **gVisor not installed** below |
| `Namespace` fails | Re-run with `--fix`, or create namespace manually |
| `RBAC permissions` fails | Re-run with `--fix` |
| K8s API unreachable | Verify kubeconfig path and cluster connectivity |

### gVisor not installed

If `tntc cluster check` reports `✗ gVisor RuntimeClass`, the cluster is missing the
gVisor sandbox runtime. This blocks all workflow deploys that use `runtime_class: gvisor`.

**Tell the user their cluster does not have gVisor installed.** They need to run the
install script from the tentacular repository on each cluster node (requires root):

```bash
# Run on each worker node that will execute workflows:
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/install.sh | sudo bash

# Then apply the RuntimeClass to the cluster (once per cluster):
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/runtimeclass.yaml

# Verify gVisor is working:
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/test-pod.yaml
kubectl wait --for=condition=Ready pod/gvisor-test --timeout=60s
kubectl logs gvisor-test   # should contain gVisor kernel messages
kubectl delete pod gvisor-test
```

After installation, re-run `tntc cluster check` to confirm `✓ gVisor RuntimeClass`.

**Alternative:** If gVisor cannot be installed on this cluster (managed cloud, no node
access), set `runtime_class: ""` in the environment config. Workflows will run without
sandboxing — note the security implications and confirm with the user before proceeding.

Only then: proceed to `phases/03-profile.md`.
