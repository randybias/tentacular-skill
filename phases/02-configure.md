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

`tntc configure` automatically profiles all reachable environments and writes
profiles to `~/.tentacular/envprofiles/<env>.md`. If auto-profiling was
skipped (cluster unreachable), go to `phases/03-profile.md` next.

## Gate Verification

Before leaving this phase, confirm:

- [ ] Config file exists (`~/.tentacular/config.yaml` or `.tentacular/config.yaml`)
- [ ] At least one environment is defined under `environments:`
- [ ] `tntc cluster check` passes for the target environment (use `--env <name>`)

```bash
tntc cluster check --env <name>
```

If cluster check fails: resolve the issue before proceeding. Do not build workflows
against a cluster that fails preflight.

Only then: proceed to `phases/03-profile.md`.
